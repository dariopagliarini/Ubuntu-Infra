# Dominio, Hosting e PEC — stato del consolidamento

Obiettivo: consolidare dominio, hosting e PEC per non pagare in tre posti diversi (in precedenza sparsi su TopHost, Ionos e Netsons).

## Stato attuale

| Servizio | Provider precedente | Provider attuale | Stato |
|---|---|---|---|
| Registrazione dominio `paglias.org` | TopHost | **Ionos** | ✅ Trasferimento completato, nameserver e DNS gestiti su Ionos |
| Hosting sito web | TopHost (piano `topwebultra`) | **Container Docker su VPS Ionos** (stack `wordpress-site`, WordPress + MariaDB) | ✅ Sito ripristinato da backup Akeeba, operativo su `https://paglias.org` |
| Email su `paglias.org` | TopHost (5 caselle, inutilizzate) | — | ✅ Nessun servizio email attivato: deciso di non sostituirlo, dato che nessuno le usava |
| PEC | Netsons | Netsons (invariato) | ⏳ Da decidere se mantenere o consolidare — indipendente dal resto |

## TopHost — da dismettere

Il dominio e il sito sono entrambi migrati su Ionos. Il piano hosting `topwebultra` su TopHost non serve più a nulla.

**Prima di disattivare, verificare:**
- Sito raggiungibile correttamente su `paglias.org` da più reti/location, con certificato HTTPS valido (Let's Encrypt via Traefik, non self-signed)
- Nessun redirect o servizio residuo ancora puntato a TopHost (es. sottodomini dimenticati)
- Che il record AAAA non sia più presente/rilevante lato TopHost (già rimosso durante la migrazione, causava problemi di validazione dei certificati)

**Come disattivare:**
- Il codice EPP per il trasferimento del dominio è già stato ottenuto e usato — questo passaggio è completato
- Resta da disdire il piano hosting: se manca poco alla scadenza naturale → non rinnovare; se si vuole chiudere subito → ticket di assistenza a TopHost richiedendo la disdetta anticipata

## DNS su Ionos (post-trasferimento)

Dopo il completamento del trasferimento, la zona DNS di Ionos parte con alcuni record di default che vanno gestiti così:

- **Record A** (`@`): sostituito con l'IP della VPS Ionos (era di default puntato alla pagina "Default Site" di Ionos)
- **Record AAAA** (`@`): **cancellato** — la VPS non ha un indirizzo IPv6 pubblico globale, un AAAA presente causa fallimento di validazione ACME/Let's Encrypt
- Record `TXT _dep_ws_mutex`, `CNAME _domainconnect`: legati al "Default Site"/Domain Connect di Ionos, innocui, lasciati come sono
- Record MX, TXT SPF, `CNAME _dmarc`, `CNAME _domainkey` (x2), `CNAME autodiscover`: configurazione email di default di Ionos — non utilizzata (vedi sopra, nessun servizio email attivo), ma non dà fastidio lasciarla; può essere rimossa per pulizia se si preferisce

## Netsons (PEC)

Ancora da decidere se mantenere la PEC su Netsons separatamente o se consolidarla altrove. Nessuna azione ancora presa.

## Prossimi passi aperti

- [ ] Disdire formalmente il piano hosting TopHost (dominio e sito già migrati)
- [ ] Decidere il destino della PEC su Netsons
- [ ] Valutare se ripulire i record email di default lasciati da Ionos (opzionale, nessuna urgenza)
