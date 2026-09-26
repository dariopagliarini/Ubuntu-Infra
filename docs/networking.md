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

## Firewall Ionos

Il firewall sta **davanti** alla VPS, si configura dal pannello Ionos ("My firewall policy") e vale per il traffico in entrata. Se sulla macchina ci sia anche un `ufw` attivo non è stato verificato: da controllare con `sudo ufw status` la prossima volta che si entra, perché due filtri in serie che non si sanno l'uno dell'altro fanno perdere ore.

Dal 26 settembre 2026 le porte aperte sono tre:

| Porta | Cosa | Perché è aperta |
|---|---|---|
| 22 | SSH | amministrazione, WinSCP, e i tunnel verso i servizi non pubblicati |
| 80 | Traefik | redirect a HTTPS e challenge HTTP di Let's Encrypt |
| 443 | Traefik | tutti i siti: `paglias.org`, `festewebapp.ddnsfree.com` |

**Tutto il resto passa da 80/443 attraverso Traefik, o da un tunnel SSH.** Una porta nuova nel firewall si apre solo se un servizio deve stare su internet per forza: non è il caso di nessun pannello di amministrazione.

### Cosa è stato tolto, e perché

Fino al 26 settembre 2026 erano aperte anche 8443, 8447, 9443 (Portainer), 23306 (MySQL), 9090 (Prometheus), 9100 (Node Exporter) e 5341 (Seq). Provandole dall'esterno una per una, **solo la 9443 rispondeva**: le altre sei erano residui dello stack `webfeste`, spento quando è arrivato `feste_online`. Porte aperte sul nulla, che sarebbero tornate pericolose il giorno in cui un container avesse pubblicato per sbaglio quella stessa porta.

La 23306 in particolare esponeva il MySQL del vecchio stack. Il database di `feste_online` **non pubblica nessuna porta** (`deploy/online/compose.yml` nel repo `Feste`: vive solo sulla rete Docker interna, raggiungibile da `online` e `adminer`), quindi quella regola non serviva più a niente.

### Portainer, dopo la chiusura della 9443

Portainer continua ad ascoltare sulla 9443 dell'host: non è più raggiungibile da internet, ma lo è da dentro un tunnel SSH.

```bash
ssh -N -L 9443:127.0.0.1:9443 <utente>@festewebapp.ddnsfree.com
```

Poi nel browser `https://localhost:9443` (l'avviso sul certificato self-signed è lo stesso di prima). Con una voce in `~/.ssh/config`:

```
Host vps
  HostName festewebapp.ddnsfree.com
  User <utente>
  LocalForward 9443 127.0.0.1:9443
```

basta `ssh vps`. Sul PC di sviluppo c'è un `Portainer.cmd` sul Desktop che apre il tunnel (se non c'è già), aspetta che la porta risponda e apre il browser.

Nel browser si può usare un nome più leggibile di `localhost`: qualsiasi nome che finisce in `.localhost` — per esempio `https://portainer.localhost:9443` — viene risolto a 127.0.0.1 dai browser, senza toccare il file `hosts`. Fuori dal browser (curl, DBeaver) quel nome non esiste: lì si usa `localhost`.

Lo stesso schema vale per qualsiasi altro servizio che non deve stare su internet: si lascia senza porta pubblicata, o pubblicata solo su `127.0.0.1`, e ci si arriva in tunnel.

### Punti ancora aperti

- **Adminer è pubblico** su `https://festewebapp.ddnsfree.com/db`: chiuso il firewall, è rimasto l'unico ingresso al database dall'esterno, protetto solo dalla password. Da mettere dietro una basic auth di Traefik, o da togliere dal routing pubblico e raggiungere in tunnel come Portainer.
- **SSH sulla 22 è aperta a tutti**: accettabile con l'accesso a sola chiave. Da controllare che in `/etc/ssh/sshd_config` ci sia `PasswordAuthentication no`.

## Accesso SSH

L'host è raggiungibile via SSH (usato anche come base per l'accesso SFTP/WinSCP ai file di WordPress, per i comandi diagnostici Docker/curl usati nel troubleshooting di Traefik — vedi [backup-restore.md](./backup-restore.md) — e per i tunnel verso i servizi non pubblicati, vedi sopra).

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
