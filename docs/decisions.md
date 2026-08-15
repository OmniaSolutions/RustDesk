# Decision Log — OmniaRustDesk

Registro delle decisioni tecniche. Formato per voce: **data · decisione · motivazione · alternative scartate**.

---

## 2026-08-14 — Distribuzione build Windows via SSH invece di artifact/Release GitHub

**Decisione.** Il workflow `build-windows.yml` carica l'installer (`rustdesk-1.4.6-x86_64.exe`)
direttamente sul server Omnia tramite `scp`, usando una chiave SSH conservata nei GitHub Secrets
(`DEPLOY_HOST`, `DEPLOY_USER`, `DEPLOY_PATH`, `DEPLOY_SSH_KEY`, opzionali `DEPLOY_PORT` e
`DEPLOY_KNOWN_HOSTS`). L'upload dell'artifact resta solo come fallback, attivo unicamente finché i
Secret di deploy non sono configurati. Nessuna GitHub Release viene pubblicata.

**Motivazione.** Su un repo pubblico gli artifact e i log dei run sono scaricabili da chiunque veda
il run: si vuole invece che il download resti "non evidente", disponibile solo sul dominio
`omniasolutions.eu/support` controllato da noi. Il deploy diretto tiene il binario fuori da GitHub.
La chiave privata è scritta a runtime da un Secret e cancellata a fine step, quindi nel repo non
resta nulla di segreto.

**Alternative scartate.**
- *GitHub Releases* — pubbliche, indicizzate da Google, mostrate nella sidebar del repo: massima
  visibilità, l'opposto dell'obiettivo.
- *Artifact pubblico come canale primario* — scaricabile da chiunque sul repo pubblico durante la
  retention; declassato a solo-fallback.
- *Repo privato + self-hosted runner* — nasconderebbe anche gli artifact, ma richiede infrastruttura
  dedicata; non necessario visto che il sorgente non contiene segreti (vedi decisione seguente).

---

## 2026-08-14 — Repository reso pubblico (con soli Secret privati)

**Decisione.** Rendere pubblico `github.com/OmniaSolutions/RustDesk` per usare i runner GitHub
gratuiti (illimitati sui repo pubblici) al posto di quelli a consumo. Le uniche informazioni riservate
(chiave SSH di deploy, eventuale token GitHub) restano nei Secret / fuori dal repo.

**Motivazione.** Il job Actions era bloccato per fatturazione ("recent account payments have failed or
your spending limit needs to be increased"). Su repo pubblico i runner Linux/Windows/macOS sono
gratuiti, sbloccando la CI senza toccare la fatturazione. Verifica effettuata: nel sorgente non ci
sono chiavi private, token o password reali. `RENDEZVOUS_SERVERS = help.omniasolutions.eu` e
`RS_PUB_KEY` (`config.rs`) sono per definizione **pubblici** — presenti in ogni binario distribuito
ed estraibili con `strings`; la vera chiave privata del server risiede solo su `hbbs`, non nel client.

**Alternative scartate.**
- *Aumentare lo spending limit / sistemare il pagamento sull'org* — costo ricorrente per minuti
  Actions (Windows conta 2×, macOS 10×); evitabile dato che il sorgente può essere pubblico.
- *Repo privato + self-hosted runner* — CI gratis mantenendo il sorgente nascosto, ma richiede
  gestire macchine (una VM Windows in particolare); complessità non giustificata.
- *Build solo in locale* — Linux e Android si compilano già via Docker in locale, ma Windows richiede
  MSVC/Visual Studio: non cross-compilabile da Linux, quindi serve comunque un runner Windows.

---

## 2026-08-14 — Nessun tentativo di cross-compilare Windows/macOS da Linux

**Decisione.** Windows continua a essere compilato su un runner Windows (Actions o self-hosted);
macOS resta fuori scope. Da Linux si compilano solo Linux e Android.

**Motivazione.** Flutter desktop Windows richiede la toolchain MSVC/Visual Studio, disponibile solo
su Windows (non esiste cross-compile del target `x86_64-pc-windows-msvc` con Flutter). macOS richiede
Xcode su hardware Apple, vietato anche dalla licenza Apple. Android invece si costruisce nativamente
su Linux (NDK + Flutter), come già fatto in locale.

**Alternative scartate.**
- *Cross-compile Windows con `x86_64-pc-windows-gnu` + mingw* — copre il core Rust ma non la parte
  Flutter desktop; build incompleta.
- *macOS su cloud (Scaleway Mac mini / EC2 Mac / MacinCloud)* — tecnicamente possibile ma fuori dallo
  scope attuale (target Linux/Android/Windows).
