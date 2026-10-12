# roving101.github.io

Sito personale di Andrea Toscano, pubblicato con GitHub Pages e Jekyll. GitHub ricompila il sito in automatico a ogni modifica del branch `main`, di solito in uno o due minuti.

## Pubblicare un write-up

Copia `templates/writeup.md` nella cartella `_writeups/`, rinominalo (per esempio `_writeups/htb-forensics-challenge.md`), compila l'intestazione e scrivi il contenuto in Markdown. Compare nella pagina Writeups e tra gli ultimi contenuti in homepage.

## Pubblicare un articolo

Copia `templates/post.md` nella cartella `_posts/` con il nome nel formato `AAAA-MM-GG-titolo.md` (per esempio `_posts/2026-10-20-hunting-lolbins.md`). La data nel nome del file è la data di pubblicazione.

## Aggiungere un progetto

Aggiungi una voce in `_data/projects.yml` seguendo l'esempio commentato nel file e cancella la riga `[]`. La sezione Projects compare in homepage appena c'è almeno un progetto.

## Aggiornare esperienza e certificazioni

Esperienze e formazione sono in `_data/experience.yml`, le certificazioni in `_data/certifications.yml`. Il testo di presentazione è in `index.html`.

## Note

Le immagini dei write-up vanno in `assets/img/` e si richiamano con `![descrizione](/assets/img/nome.png)`. Prima di pubblicare un write-up controlla che non contenga dati di clienti o dettagli coperti da riservatezza.
