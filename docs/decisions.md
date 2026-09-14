# Log delle decisioni

Formato: data — titolo, poi Perché / Come / Note.

---

## 2026 — Isolamento di Traefik in uno stack dedicato

**Perché**: separare il reverse proxy dagli stack applicativi (inizialmente Traefik viveva dentro lo stack `webfeste`), per poterlo riusare come reverse proxy condiviso anche per nuovi stack (es. `paglias-website`) senza accoppiarli tra loro.

**Come**: creata una rete Docker esterna condivisa chiamata `proxy`; Traefik spostato in uno stack separato che la referenzia; gli altri stack (es. `paglias-website`) si collegano alla stessa rete `proxy` per essere raggiungibili da Traefik, mantenendo una rete `internal` separata per la comunicazione privata (es. WordPress ↔ DB).

**Note**: lo stack `webfeste` mantiene comunque la propria rete storica `webfeste_default` per i suoi servizi interni (adminer, api, db, php, seq).

---

## 2026 — Migrazione del sito da TopHost a container Docker su Ionos

**Perché**: consolidare hosting web sulla VPS Ionos già in uso per altri servizi, evitare di pagare hosting su più provider.

**Come**: nuovo stack Docker `paglias-website` (WordPress + MariaDB) su Ionos, backup del sito precedente fatto con Akeeba Backup for WordPress prima della migrazione, dominio e DNS spostati da TopHost a Ionos, Traefik configurato per servire `paglias.org` (con redirect HTTP→HTTPS e www→apex).

**Note**: piano hosting TopHost da dismettere dopo verifica — vedi [domini-hosting.md](./domini-hosting.md).

---

## 2026-09-14 — Bind mount invece di volume named per WordPress

**Perché**: serve un accesso comodo via WinSCP/SFTP per scaricare e ricaricare i backup generati da Akeeba Backup; un volume Docker named non è facilmente navigabile da fuori (i file finiscono sotto `/var/lib/docker/volumes/...`, accessibile solo con permessi elevati e path poco pratico).

**Come**:
- Copiati i dati dal volume named `paglias-website_wp_data` alla cartella `/opt/paglias-site/wordpress` tramite un container Alpine temporaneo (`docker run --rm -v paglias-website_wp_data:/source -v /opt/paglias-site/wordpress:/dest alpine cp -av /source/. /dest/`)
- Aggiornato il `docker-compose.yml` dello stack `paglias-website`: il servizio `wordpress` monta ora `/opt/paglias-site/wordpress:/var/www/html` invece del volume named
- Permessi impostati a `33:33` (www-data) sulla cartella
- Container ricreato con `docker compose up -d --force-recreate wordpress`

**Note**:
- Il volume Docker originale `paglias-website_wp_data` mantenuto temporaneamente come rete di sicurezza, da rimuovere solo dopo verifica di stabilità
- Durante il processo creato per errore un volume vuoto chiamato `wp_data` (senza prefisso progetto) — non contiene dati, da rimuovere quando comodo
- Il volume del database (`paglias-website_db_data`) è rimasto named, non necessita di accesso diretto via SFTP
- Dettagli completi della procedura in [backup-restore.md](./backup-restore.md)

---

## 2026-09 — Isolamento di Traefik in uno stack dedicato + rete 'proxy' (attuazione)

**Perché**: Traefik viveva dentro lo stack `webfeste`, accoppiando il ciclo di vita del reverse proxy a quello degli altri servizi applicativi. Un redeploy di `webfeste` per un bug in api/db rischiava di riavviare anche Traefik e portare giù tutti i siti serviti.

**Come**:
- Creata una rete Docker esterna `proxy` (`docker network create proxy`), in sostituzione della rete `webfeste_default` generata automaticamente dallo stack.
- Traefik spostato in un nuovo stack dedicato `traefik`, deployato da Portainer via repository Git (non build, solo immagine pubblica `traefik:v2.11`).
- Riutilizzato il volume esistente dei certificati (`webfeste_letsencrypt` → montato nel nuovo stack come `letsencrypt`, con `external: true` e `name:` esplicito), per non perdere i certificati Let's Encrypt già emessi.
- Stack `webfeste` aggiornato: rimosso il servizio `traefik`, tutti gli altri servizi migrati sulla rete `proxy`.
- Il vecchio container `traefik` rimasto dentro webfeste dopo il primo redeploy è stato rimosso manualmente (Portainer non elimina da solo i servizi tolti dal compose, salvo l'opzione "prune").

**Note**: il certresolver di Traefik si chiama `le` (non `letsencrypt`) — dettaglio recuperato dal file compose reale, non andava indovinato.

---

## 2026-09 — Repository Ubuntu-Infra per infrastruttura pura

**Perché**: serve un unico posto per ricostruire l'intera infrastruttura da zero (Traefik, rete, WordPress) senza doverla tenere a memoria, distinto dal codice applicativo di WebFeste.

**Come**: creato un nuovo repository GitHub `Ubuntu-Infra` (locale in `S:\Progetti\Privato\Ubuntu-Infra`), con struttura:
```
Ubuntu-Infra/
├── .gitignore
├── traefik/
│   └── docker-compose.yml
└── wordpress-site/  (o paglias.org/)
    ├── docker-compose.yml
    └── .env.example
```
Gli stack `traefik` e `wordpress-site` sono deployati in Portainer col metodo "Repository", puntando a questo repo (branch `main`, path del compose specifico per ogni stack). Lo stack `webfeste` resta invece legato al repo `WebFeste` (contiene build context per api/php, non ha senso spostarlo qui).

**Note**: `.env` reali (password) NON versionati, esclusi via `.gitignore` (`.env`, `*.env`, `acme.json`, ecc., con eccezione per `*.env.example`). Per lo stack wordpress-site le variabili d'ambiente sono passate tramite il pannello "Environment variables" di Portainer in fase di deploy da Repository, senza bisogno di un file `.env` fisico sulla VPS.

---

## 2026-09 — Migrazione sito paglias.org da TopHost a container Docker (WordPress)

**Perché**: consolidare hosting su Ionos, eliminando il piano `topwebultra` su TopHost ed evitando un canone di hosting separato, visto che la VPS è già in uso per altri container.

**Come**:
- Nuovo stack `wordpress-site` (o `paglias`): WordPress (`wordpress:php8.3-apache`) + MariaDB (`mariadb:11`), database isolato su rete `internal` dedicata, WordPress su doppia rete (`proxy` + `internal`) con label `traefik.docker.network=proxy` esplicita — necessaria perché, senza questa label, Traefik tentava di raggiungere il container sull'IP della rete interna invece che su quella condivisa, causando un errore 504 Gateway Timeout.
- Test effettuato prima su un secondo hostname DDNS dedicato (`paglias-test.xubi.org` su Dynu), per evitare conflitti di routing con `festewebapp.ddnsfree.com` (già usato da `webfeste_php` con la stessa regola Host, senza distinzione di path — le due regole erano in conflitto e Traefik instradava in modo non prevedibile).
- Ripristino contenuti tramite il backup Akeeba Backup for WordPress già esistente: upload di `kickstart.php` + archivio `.jpa` nella webroot del container (via SCP sulla VPS, poi `docker cp` dentro il container), esecuzione di Kickstart/ANGIE da browser, che gestisce anche l'aggiornamento del dominio nel database durante il ripristino.
- Al termine dei test, switch DNS di `paglias.org` (record A) da TopHost verso l'IP della VPS, e aggiornamento delle label Traefik da `paglias-test.xubi.org` a `paglias.org`.

**Note**:
- Dopo ogni ripristino Akeeba va riapplicata a mano la modifica a `wp-config.php` per far sì che WordPress si fidi dell'header `X-Forwarded-Proto` inviato da Traefik, altrimenti si genera un loop di redirect infinito (`ERR_TOO_MANY_REDIRECTS`) dietro il reverse proxy. Snippet da aggiungere prima di `/* That's all, stop editing! */`:
  ```php
  if (isset($_SERVER['HTTP_X_FORWARDED_PROTO']) && $_SERVER['HTTP_X_FORWARDED_PROTO'] === 'https') {
      $_SERVER['HTTPS'] = 'on';
  }
  ```
  Procedura per applicarla (il container non ha editor testuali): estrarre con `docker cp paglias_wordpress:/var/www/html/wp-config.php ./wp-config.php`, modificare sull'host, rimettere dentro con `docker cp`, poi risistemare il proprietario con `docker exec paglias_wordpress chown www-data:www-data /var/www/html/wp-config.php` (`docker cp` rimette il file come `root`, diverso dal resto dei file `www-data`).
- `kickstart.php` e l'archivio `.jpa` vanno sempre rimossi dalla webroot subito dopo il ripristino, per non lasciarli accessibili pubblicamente.

---

## 2026-09 — Trasferimento dominio paglias.org da TopHost a Ionos

**Perché**: completare il consolidamento — un solo pannello per dominio, hosting e VPS, coerente con l'obiettivo iniziale di non pagare/gestire su più provider.

**Come**: richiesto codice EPP su TopHost e sbloccato il dominio, poi avviato il trasferimento dal pannello Ionos scegliendo l'opzione "nameserver Ionos" invece di "mantieni impostazioni attuali", per svincolare completamente la gestione DNS da TopHost (necessario perché la zona DNS di TopHost sarebbe comunque sparita alla chiusura dell'account). Dopo il completamento del trasferimento, ricreato manualmente su Ionos il record A (verso l'IP della VPS) e **cancellato il record AAAA di default di Ionos**.

**Note**: la VPS non ha un indirizzo IPv6 pubblico globale (solo un indirizzo link-local `fe80::...` sull'interfaccia `ens6`) — un record AAAA presente (sia su TopHost inizialmente, sia quello di default creato da Ionos dopo il trasferimento) puntava a un indirizzo non funzionante, causando un fallimento di validazione ACME/Let's Encrypt (`403 unauthorized`, risposta 404 sulla challenge HTTP). Diagnosticato tramite `docker logs traefik` e risolto entrambe le volte cancellando il record AAAA. I record MX/SPF/DKIM/DomainConnect creati di default da Ionos nella zona DNS sono innocui e non collegati a nessun servizio attivato — lasciati come sono, non generano costi né conflitti.

---

## 2026-09 — Decisione: nessun servizio email sul dominio

**Perché**: le 5 caselle email su TopHost (configurate a suo tempo per sé e la famiglia) risultavano completamente inutilizzate da nessuno.

**Come**: nessuna migrazione email effettuata. Nessun inoltro, nessuna casella attivata su Ionos/Zoho/altro. La posta sparisce insieme alla chiusura di TopHost, senza alcuna sostituzione.

**Note**: se in futuro servisse un indirizzo email sporadico su questo dominio, l'opzione più semplice e gratuita resta un inoltro (es. Cloudflare Email Routing) verso la casella Gmail personale già in uso, con "Invia posta come" lato Gmail per poter anche scrivere "da" quell'indirizzo — da valutare solo se e quando servirà davvero, non preventivamente.

---

## 2026-09 — Hardening dello stack webfeste

**Perché**: revisione di sicurezza e resilienza del compose esistente, emersa durante il lavoro di separazione di Traefik.

**Come**:
- Rimossa l'esposizione pubblica diretta delle porte di `mariadb` (`3306:3306`) e `seq` (`5341:80`): entrambi raggiungibili solo tramite rete Docker interna o tramite Traefik (Seq instradato su `/logs`, con middleware `stripprefix` per rimuovere il prefisso prima di passare la richiesta al container).
- Creata rete `internal` dedicata (nome esplicito `webfeste_internal`, per non dipendere dal nome dello stack in Portainer) per isolare `mariadb`, non più raggiungibile dalla rete condivisa `proxy`; `api`, `adminer`, `phpapp` collegati sia a `proxy` che a `internal`, con label `traefik.docker.network=proxy` per evitare ambiguità di routing (stesso problema incontrato e risolto per WordPress).
- Middleware `redirect` ridefinito in modo identico sulle label di ogni servizio che lo usa (non solo su `api` come in origine), per non dipendere dalla disponibilità di un singolo container — un riavvio/crash di `api` aveva già causato la sparizione temporanea del redirect per tutti gli altri router.
- Aggiunto healthcheck a `mariadb` (`healthcheck.sh --connect --innodb_initialized`); `api`/`phpapp` ora attendono `condition: service_healthy` su mariadb invece di `service_started`, per evitare tentativi di connessione prematuri.
- Tag immagine fissati a versione major esplicita invece di `latest`: `mariadb:11`, `datalust/seq:2025.2` — evita che un futuro `docker pull`/redeploy sposti inavvertitamente a una major version successiva con formato dati incompatibile.

**Note**: se in futuro serve accedere al database da uno strumento esterno (es. MySQL Workbench), preferire un tunnel SSH temporaneo invece di riesporre la porta 3306 pubblicamente.

---

## Template per nuove voci

```markdown
## AAAA-MM-GG — Titolo della decisione

**Perché**: ...

**Come**: ...

**Note**: ...
```
