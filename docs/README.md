# Documentazione Infrastruttura — Ubuntu-Infra

Documentazione delle configurazioni e delle decisioni relative al server Ubuntu (VPS Ionos), agli stack Docker e ai servizi collegati (dominio, hosting, PEC).

## Panoramica architettura

- **Host**: VPS Ionos, Ubuntu 24.04, con Docker e Portainer
- **Gestione container**: Portainer (deploy stack col metodo "Repository", puntando al repository GitHub `Ubuntu-Infra` per gli stack di infrastruttura pura, e al repository `WebFeste` per lo stack applicativo)
- **Reverse proxy**: Traefik, in uno stack dedicato separato (`traefik`), isolato dagli altri stack tramite la rete Docker esterna condivisa `proxy`
- **Certificati SSL**: gestiti da Traefik tramite Let's Encrypt (certresolver `le`)
- **DNS dinamico**: DynuDNS associa `festewebapp.ddnsfree.com` (e, per i test, `paglias-test.xubi.org`) all'IP del server Ionos
- **Dominio principale**: `paglias.org`, registrazione e DNS gestiti su Ionos (trasferiti da TopHost)

## Stack attivi

| Stack | Contenuto | Note |
|---|---|---|
| `traefik` | Reverse proxy | Stack dedicato, deploy da repo `Ubuntu-Infra`, rete condivisa `proxy` |
| `webfeste` | adminer, api, db, php, seq | Rete `proxy` (esterna) + `webfeste_internal` (dedicata al db); Traefik non è più incluso in questo stack |
| `wordpress-site` (o `paglias`) | WordPress + MariaDB (dominio paglias.org) | Deploy da repo `Ubuntu-Infra`; db isolato su rete `internal` dedicata — vedi [backup-restore.md](./backup-restore.md) |

## Repository di codice

- **Ubuntu-Infra** (GitHub): infrastruttura pura — stack `traefik`, stack `wordpress-site`. Nessun codice applicativo, solo configurazione (docker-compose, `.env.example`). Contiene anche questa documentazione.
- **WebFeste** (GitHub): stack `webfeste` (contiene i build context per api/php) + codice applicativo WebFeste.Api, sviluppato con Visual Studio.
- Entrambi i repository tenuti in locale in `S:\Progetti\Privato`.

## Indice documenti

- [networking.md](./networking.md) — reti Docker, Traefik, DNS, diagnostica routing/certificati
- [backup-restore.md](./backup-restore.md) — backup e restore di WordPress (Akeeba), accesso via WinSCP, procedura di ripristino in Docker
- [domini-hosting.md](./domini-hosting.md) — stato del consolidamento dominio / hosting / PEC
- [decisions.md](./decisions.md) — log delle decisioni tecniche con motivazione

## Dominio paglias.org

Dominio e sito completamente migrati da TopHost a Ionos: registrazione e DNS su Ionos, sito ospitato come container Docker (stack `wordpress-site`) sulla stessa VPS che ospita gli altri stack. Nessun servizio email attivo sul dominio (scelta deliberata: le vecchie caselle TopHost non erano usate da nessuno).

## Come aggiornare questa documentazione

Ogni volta che si prende una decisione di configurazione non banale (cambio di volume, nuova rete, nuovo servizio, migrazione), aggiungere una voce in `decisions.md` con data, motivazione e comando/procedura usata. Aggiornare anche il documento specifico pertinente (networking, backup, domini) se lo stato cambia.
