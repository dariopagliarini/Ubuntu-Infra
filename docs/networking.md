# Networking

## Reti Docker

| Rete | Tipo | Usata da | Scopo |
|---|---|---|---|
| `proxy` | esterna, condivisa (creata manualmente con `docker network create proxy`) | Traefik, `webfeste` (api, adminer, phpapp — dual-homed), `wordpress-site` (wordpress — dual-homed) | Rete condivisa per il routing di Traefik verso container di stack diversi; sostituisce la vecchia `webfeste_default` |
| `webfeste_internal` | interna, dedicata allo stack webfeste (nome esplicito) | `webfeste` (mariadb, + api/adminer/phpapp in dual-homing) | Isola il database webfeste dalla rete condivisa `proxy` e da internet |
| `internal` (stack wordpress-site) | interna, dedicata allo stack wordpress-site | wordpress, db | Isola il database WordPress, stesso principio della rete internal di webfeste |

**Nota tecnica importante**: quando un container sta contemporaneamente su più reti (es. `proxy` + una rete interna), Traefik può instradare verso l'IP sbagliato se non gli viene detto esplicitamente quale rete usare. Su tutti i container dual-homed va aggiunta la label:
```yaml
- "traefik.docker.network=proxy"
```
(riscontrato concretamente con WordPress: senza questa label, Traefik tentava di raggiungere il container sulla rete interna, causando un errore 504 Gateway Timeout).

La rete `proxy` va creata una tantum come rete esterna prima di deployare gli stack che la referenziano:
```bash
docker network create proxy
```

## Stack Traefik (separato dallo stack applicativo)

Traefik non vive più dentro lo stack `webfeste`: è stato isolato in un proprio stack dedicato `traefik`, deployato da Portainer col metodo "Repository" puntando al repo `Ubuntu-Infra`. Questo evita che un redeploy di `webfeste` (per bug in api/db) riavvii anche il reverse proxy, portando giù tutti i siti.

- Immagine: `traefik:v2.11`
- Rete: `proxy`
- Certresolver: `le` (non `letsencrypt` — nome definito nei comandi di avvio di Traefik)
- Volume certificati: riusato lo stesso volume del vecchio setup (`webfeste_letsencrypt`, montato nel nuovo stack con `external: true` e `name:` esplicito), per non perdere i certificati Let's Encrypt già emessi

## Traefik — routing paglias.org (stack wordpress-site)

Configurazione label sul container `paglias_wordpress`:

- **HTTP → HTTPS redirect**: router `paglias` su entrypoint `web`, middleware `paglias-redirect`
- **HTTPS apex**: router `paglias-secure` su entrypoint `websecure`, certresolver `le`
- **Backend**: servizio `paglias` verso la porta 80 del container
- **Label di rete esplicita**: `traefik.docker.network=proxy` (vedi nota sopra)

## Traefik — routing webfeste (aggiornato)

- `webfeste_php`: Host su `festewebapp.ddnsfree.com`, nessun path, priority 1 (catch-all a bassa priorità)
- `webfeste_api`: Host + PathPrefix `/api`
- `webfeste_adminer`: Host + PathPrefix `/db`, priority 3
- `webfeste_seq`: Host + PathPrefix `/logs`, priority 3, con middleware `stripprefix` per rimuovere `/logs` prima di passare la richiesta a Seq (Seq si aspetta di ricevere le richieste sulla radice)
- Il middleware `redirect` (HTTP→HTTPS) è ora ridefinito in modo identico sulle label di ogni servizio che lo usa, non solo su `api`, per non dipendere dalla disponibilità di un singolo container

**Attenzione ai conflitti di routing**: due router con la stessa regola Host (senza distinzione di path) sullo stesso dominio vanno in conflitto — riscontrato tra `webfeste_php` e `wordpress` quando condividevano temporaneamente `festewebapp.ddnsfree.com`. Risolto assegnando a WordPress un hostname distinto.

## DNS

- **Dominio**: `paglias.org`, registrazione e DNS trasferiti da TopHost a **Ionos** (nameserver Ionos)
- **Record A**: punta all'IP pubblico della VPS Ionos
- **Record AAAA**: assente/rimosso intenzionalmente — la VPS non ha un indirizzo IPv6 pubblico globale (solo link-local `fe80::...` sull'interfaccia `ens6`, verificabile con `ip -6 addr show scope global`). Un record AAAA presente (sia lato TopHost che di default lato Ionos) causava fallimento di validazione ACME per Let's Encrypt (errore 403/404 sulla challenge HTTP), diagnosticato tramite `docker logs traefik`
- **DynuDNS**: servizio di DNS dinamico che associa `festewebapp.ddnsfree.com` all'IP corrente del server Ionos
- **Hostname DDNS aggiuntivo** (Dynu): `paglias-test.xubi.org`, stesso IP della VPS, creato per isolare i test di WordPress prima dello switch definitivo al dominio vero `paglias.org` (evita conflitti di routing con `festewebapp.ddnsfree.com`, già usato da webfeste)

## Accesso SSH

L'host è raggiungibile via SSH (usato anche come base per l'accesso SFTP/WinSCP ai file di WordPress, e per i comandi diagnostici Docker/curl usati nel troubleshooting di Traefik — vedi [backup-restore.md](./backup-restore.md)).

## Diagnostica Traefik/certificati — comandi utili

```bash
# Testare il routing/redirect in locale, bypassando DNS
curl -v -H "Host: <dominio>" http://localhost/
curl -v -k -H "Host: <dominio>" https://localhost/

# Verificare quale certificato viene servito (self-signed Traefik vs Let's Encrypt reale)
# guardare la riga "subject:" nell'output del comando sopra

# Verificare presenza di un certificato reale nello storage
docker exec traefik cat /letsencrypt/acme.json | grep -A 3 '"main": "<dominio>"'

# Log recenti di Traefik per errori ACME
docker logs traefik --since 15m | grep -i acme
```
