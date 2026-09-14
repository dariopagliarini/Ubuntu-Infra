# Backup & Restore — paglias.org (WordPress)

## Strumento di backup

**Akeeba Backup for WordPress**, plugin installato su WordPress. Genera archivi (`.jpa` o `.zip`) salvati (di default) in:
```
wp-content/uploads/akeebabackupwp/backup/
```
(percorso configurabile nelle impostazioni del plugin, sezione "Output Directory" — verificare che corrisponda).

## Accesso ai file: bind mount + WinSCP

Il volume di WordPress è stato convertito da **volume Docker named** a **bind mount**, per poter accedere ai file direttamente via SFTP con un client come WinSCP, senza container aggiuntivi né tunnel SSH manuali.

- **Path sull'host**: `/opt/paglias-site/wordpress`
- **Path nel container**: `/var/www/html`
- **Permessi**: proprietario `33:33` (UID/GID di `www-data`, usato dall'immagine `wordpress:php8.3-apache`)

### Configurazione WinSCP

- Protocollo: **SFTP**
- Host: hostname/IP della VPS Ionos (o `festewebapp.ddnsfree.com`)
- Utente: utente SSH esistente sull'host
- Autenticazione: chiave privata (o password) salvata nel sito WinSCP
- **Remote directory di default** (scheda Environment → Directories):
  ```
  /opt/paglias-site/wordpress/wp-content/uploads/akeebabackupwp/backup/
  ```
  così WinSCP apre già nella cartella dei backup.

## Procedura di ripristino WordPress via Akeeba (in ambiente Docker)

Procedura effettivamente usata per il ripristino di paglias.org nel container `paglias_wordpress` durante la migrazione da TopHost:

1. **Recuperare i file**: scaricare dal vecchio pannello Akeeba (o via WinSCP) l'ultimo archivio di backup (`.jpa`) e scaricare `kickstart.php` da akeeba.com (sezione download, gratuito, senza login).
2. **Portare i file sulla VPS**, via SCP dal PC Windows:
   ```powershell
   scp C:\percorso\site-backup.jpa root@IP-VPS:/tmp/
   scp C:\percorso\kickstart.php root@IP-VPS:/tmp/
   ```
3. **Copiare i file dentro il container** (nella webroot vera):
   ```bash
   docker cp /tmp/site-backup.jpa paglias_wordpress:/var/www/html/
   docker cp /tmp/kickstart.php paglias_wordpress:/var/www/html/
   ```
4. **Avviare Kickstart da browser**: aprire `https://<dominio-corrente>/kickstart.php`. Kickstart estrae l'archivio nella webroot, poi parte automaticamente **ANGIE** (installer guidato Akeeba).
5. **Durante ANGIE**: host database = `db` (nome del servizio nel docker-compose, **non** `localhost`), utente/password/nome database = quelli del file `.env` dello stack. Nell'ultimo step, impostare il **dominio finale corretto** (es. `paglias.org`) — ANGIE aggiorna automaticamente `siteurl`/`home` nel database, evitando un fix manuale successivo.
6. **Rimuovere `kickstart.php` e l'archivio `.jpa`** dalla webroot subito dopo il ripristino, per sicurezza:
   ```bash
   docker exec paglias_wordpress rm /var/www/html/kickstart.php /var/www/html/site-backup.jpa
   ```

### Passaggio da rifare dopo OGNI ripristino: fix del redirect loop dietro Traefik

Akeeba/ANGIE rigenera `wp-config.php` da zero a ogni ripristino, quindi va **riapplicata manualmente** questa modifica (altrimenti si presenta un loop `ERR_TOO_MANY_REDIRECTS`, perché WordPress non sa che la richiesta arriva già in HTTPS da Traefik):

```php
if (isset($_SERVER['HTTP_X_FORWARDED_PROTO']) && $_SERVER['HTTP_X_FORWARDED_PROTO'] === 'https') {
    $_SERVER['HTTPS'] = 'on';
}
```
Va aggiunta prima della riga `/* That's all, stop editing! */`.

**Procedura per modificarla** (il container non ha un editor di testo installato):
```bash
# 1. Estrai il file dal container alla VPS
docker cp paglias_wordpress:/var/www/html/wp-config.php /root/wp-config.php

# 2. Modificalo sulla VPS con nano/vi
nano /root/wp-config.php

# 3. Rimettilo dentro il container
docker cp /root/wp-config.php paglias_wordpress:/var/www/html/wp-config.php

# 4. Risistema il proprietario (docker cp lo rimette come root, va riallineato a www-data)
docker exec paglias_wordpress chown www-data:www-data /var/www/html/wp-config.php
```
Nessun riavvio del container necessario: Apache/PHP rileggono il file a ogni richiesta.

### Verifica siteurl/home nel database (se serve un controllo manuale)

Il client `mysql` non è presente nell'immagine `mariadb:11` — usare `mariadb` al suo posto:
```bash
docker exec -it paglias_db mariadb -u wpuser -p wordpress -e \
  "UPDATE wp_options SET option_value='https://paglias.org' WHERE option_name IN ('siteurl','home');"
```
Nota: il database di WordPress (`paglias_db`) è isolato sulla rete `internal` dedicata dello stack — non è raggiungibile da Adminer (che vive nello stack `webfeste`, su un'altra rete). Per interventi sul DB di WordPress, passare sempre da riga di comando come sopra, non da Adminer.

## Backup del database

Il database WordPress (volume Docker named `db_data` dello stack `wordpress-site`) è incluso nel backup Akeeba se il plugin è configurato per il dump integrato (comportamento di default per i backup completi).

Per un backup del DB *indipendente* da Akeeba:
```bash
docker exec paglias_db mariadb-dump -u wpuser -p wordpress > backup-db-$(date +%F).sql
```

Lo stesso vale per il database di `webfeste` (`webfeste_db`):
```bash
docker exec webfeste_db mariadb-dump -u webfeste -p webfeste > backup-webfeste-db-$(date +%F).sql
```

## Storico della migrazione (WordPress bind mount)

- Volume originale: `paglias-website_wp_data` (volume Docker named, creato dallo stack `paglias-website`)
- Convertito in bind mount `/opt/paglias-site/wordpress` per permettere accesso SFTP comodo
- Procedura di copia usata (container temporaneo Alpine):
  ```bash
  docker run --rm \
    -v paglias-website_wp_data:/source \
    -v /opt/paglias-site/wordpress:/dest \
    alpine sh -c "cp -av /source/. /dest/"
  ```
- Il volume Docker originale `paglias-website_wp_data` è stato mantenuto come rete di sicurezza dopo la migrazione, da rimuovere solo dopo aver verificato la stabilità del nuovo setup
- Attenzione: durante la migrazione è stato creato per errore un volume vuoto chiamato semplicemente `wp_data` (senza prefisso progetto) — da non confondere con quello reale, da rimuovere quando non più necessario con `docker volume rm wp_data`

## Diagnostica Traefik/certificati — comandi utili

Utili durante troubleshooting di routing, certificati o container che non rispondono (vedi anche [networking.md](./networking.md)):

```bash
# Stato e log dei container coinvolti
docker ps -a | grep paglias
docker logs paglias_wordpress --tail 50
docker logs traefik --since 5m

# Verifica reti a cui è collegato un container (utile per capire problemi di raggiungibilità)
docker inspect paglias_wordpress | grep -A 10 Networks

# Test diretto in locale sulla VPS, bypassando DNS
curl -v -H "Host: <dominio>" http://localhost/
curl -v -k -H "Host: <dominio>" https://localhost/
```
