---
title: "An open Redis port is a shell: unauthenticated RCE and the trail it leaves"
summary: "Why an exposed Redis instance with no password hands an attacker code execution, the two paths that get there, and what each one looks like in host and network telemetry."
attack: [T1190, T1059, T1053]
---

Redis is built to be fast, which historically meant it trusted whoever could reach it. An instance bound to a public interface with no password is not a weak configuration, it is an open door, and getting from that door to code execution takes a few lines.

I ran into this more than once in offensive practice, and it is a clean example of a class of bug that still shows up constantly: a backend service that assumes the network is trusted, exposed where the network is anything but. Here is how the two main paths work, reconstructed in a lab, and then the telemetry that gives each one away.

## The starting condition

The whole thing depends on one thing: an attacker can talk to the Redis port and run commands without authentication. You can confirm it in one line:

```bash
redis-cli -h target ping
```

If that answers `PONG` with no `AUTH` required, every command below is available. The reason it is dangerous is that Redis commands can write files, and some can load code.

## Path one: writing a file through the database

The oldest technique abuses the fact that Redis can be told where to save its database and under what name. Point it at a sensitive directory, put your payload in a key, and let the save write it to disk.

The classic target is an SSH key. The attacker sets a key whose value is their public key padded with newlines, then redirects the dump:

```bash
redis-cli -h target config set dir /root/.ssh
redis-cli -h target config set dbfilename authorized_keys
redis-cli -h target set payload "\n\nssh-rsa AAAA... attacker\n\n"
redis-cli -h target save
```

If Redis is running as root and `/root/.ssh` is writable, the saved database file is now a valid `authorized_keys`, and the attacker logs in over SSH. The same primitive writes a cron job into `/var/spool/cron` or a web shell into a served directory.

## Path two: loading a malicious module

Newer Redis supports loadable modules, and that turns a file write into direct command execution. The attacker uses the same `dir` and `dbfilename` trick to drop a compiled module onto disk, then loads it:

```bash
redis-cli -h target module load /tmp/exp.so
redis-cli -h target system.exec "id"
```

The module exports a command that runs shell input inside the Redis process. No SSH, no cron wait, just execution with the privileges of the Redis daemon. This is the path most public tooling automates, and it is the fast one.

## Why it works

As with most of these, there is no memory corruption and no exploit against Redis itself. Every command is legitimate. The failure is that an unauthenticated service with file-writing and module-loading capability was reachable by an attacker at all. The network was treated as a trust boundary, and it was not one.

## What it looks like from the defender's side

Redis compromise is noisy if you are watching the right layers, and the two paths look different.

At the network layer, the giveaway is `CONFIG SET` traffic to a Redis port from anywhere unexpected. Legitimate clients issue `GET`, `SET` and the like. A client that sends `CONFIG SET dir`, `CONFIG SET dbfilename` or `MODULE LOAD` is almost never an application doing its job. If you have visibility into Redis command traffic, those verbs alert on their own.

On the host, path one shows up as a write to a sensitive file by the Redis process. An `authorized_keys`, a cron file or a web-root file whose writing process is `redis-server` is the signal, and file-integrity monitoring on those locations catches it:

```kql
DeviceFileEvents
| where InitiatingProcessFileName =~ "redis-server"
| where FolderPath has_any (".ssh", "/var/spool/cron", "/etc/cron", "authorized_keys")
| project Timestamp, DeviceName, FolderPath, FileName, InitiatingProcessAccountName
```

Path two shows up as process creation: a child of `redis-server` that is a shell or any command the operator would not expect a cache to run.

```kql
DeviceProcessEvents
| where InitiatingProcessFileName =~ "redis-server"
| where FileName in~ ("sh", "bash", "dash", "id", "whoami", "curl", "wget")
| project Timestamp, DeviceName, FileName, ProcessCommandLine, InitiatingProcessAccountName
```

A cache server spawning a shell is one of those events that needs no tuning. If it happens, something is wrong.

## Closing the door

Redis should never be reachable from an untrusted network. Bind it to localhost or a private interface, require a strong password with `requirepass`, and put it behind a firewall that only lets application hosts in. For anything exposed, enable protected mode and rename or disable the dangerous commands, `CONFIG`, `MODULE`, `SAVE` and friends, through the command-renaming feature. Run the daemon as a dedicated unprivileged user so that even a successful write lands somewhere harmless.

The detections above matter because exposure happens by accident, through a misconfigured container or a cloud security group opened too wide. When it does, a `CONFIG SET` from the internet or a shell under `redis-server` turns a silent breach into an alert, and that is the difference between finding out now and finding out from the incident report.
