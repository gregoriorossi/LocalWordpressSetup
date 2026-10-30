# Migrare un sito WordPress da Aruba a Docker in locale

Questa guida descrive come creare una copia locale di un sito WordPress ospitato su Aruba, usando Docker Desktop su Windows.

## 1. Ambiente di partenza

In base alle informazioni disponibili, il sito online utilizza:

- WordPress **6.7.11**
- PHP **8.2**
- MySQL **5.5**

L'obiettivo è ottenere una copia locale raggiungibile da `http://localhost:8080`, mantenendo i file del sito e il database separati in modo da poter fare test e aggiornamenti senza intervenire sulla produzione.

> **Nota sulla compatibilità:** MySQL 5.5 è obsoleto e non più supportato. L'immagine `mysql:5.5` potrebbe non essere disponibile o non funzionare correttamente su tutti i sistemi. Non aggiornare il database alla cieca: conserva sempre il dump originale e prova il ripristino su una copia. La compatibilità va verificata prima di scegliere un motore database alternativo.
>
> **Nota sulla versione WordPress:** per mantenere WordPress 6.7.11, usa i file del core scaricati da Aruba. Non lasciare che una configurazione Docker installi o sostituisca automaticamente il core con una versione differente.

## 2. Prerequisiti

1. Un PC Windows.
2. Docker Desktop installato e avviato.
3. Accesso FTP/SFTP al tuo hosting Aruba.
4. Accesso al database MySQL tramite il pannello Aruba/phpMyAdmin.
5. Spazio disco sufficiente per una copia dei file e del database.

Installa Docker Desktop dal [sito ufficiale Docker](https://www.docker.com/products/docker-desktop/). Se utilizzi WSL 2, verifica che sia operativo.

Apri PowerShell e controlla:

```powershell
docker --version
docker compose version
docker version
```

`docker version` deve mostrare sia le informazioni del client sia quelle del server. Se compare un errore relativo a `//./pipe/dockerDesktopLinuxEngine`, avvia Docker Desktop, attendi che il motore sia pronto e riprova. Puoi inoltre verificare il contesto:

```powershell
docker context ls
```

Se `desktop-linux` è presente, selezionalo:

```powershell
docker context use desktop-linux
```

Se necessario, controlla WSL:

```powershell
wsl --status
wsl --list --verbose
```

## 3. Scaricare i file WordPress da Aruba

1. Accedi al pannello Aruba e recupera i dati FTP/SFTP.
2. Collegati con un client come FileZilla.
3. Individua la cartella in cui è installato WordPress.
4. Scarica **tutti** i file e le cartelle, compresi `wp-admin`, `wp-includes`, `wp-content`, `wp-config.php`, `.htaccess` e gli eventuali file nascosti.
5. Conserva il backup originale senza modificarlo.

La cartella `wp-content` contiene in particolare temi, plugin, media e altri file caricati. Non basta scaricare soltanto questa cartella: per una copia fedele serve l'intera installazione.

## 4. Esportare il database

1. Apri lo strumento di gestione MySQL disponibile nel pannello Aruba.
2. Individua il database utilizzato da WordPress.
3. Esportalo in formato SQL e salva il file come `database.sql`.
4. Conserva una copia non modificata del dump.

Se non conosci il nome del database, apri il `wp-config.php` scaricato da Aruba e cerca:

```php
define( 'DB_NAME', 'nome_database' );
define( 'DB_USER', 'utente_database' );
define( 'DB_PASSWORD', 'password_database' );
define( 'DB_HOST', 'server_database' );
```

Annota anche il prefisso delle tabelle, per esempio:

```php
$table_prefix = 'wp_';
```

Il prefisso può essere diverso da `wp_`: in seguito va mantenuto quello originale.

## 5. Creare la struttura del progetto

Crea, per esempio, questa cartella:

```text
C:\docker\testedipietra\
├── compose.yaml
├── Dockerfile
├── .env
├── database.sql
└── wordpress\
    ├── wp-admin\
    ├── wp-content\
    ├── wp-includes\
    ├── wp-config.php
    └── ...
```

- `wordpress\` contiene tutti i file scaricati da Aruba.
- `database.sql` è il dump del database.
- `Dockerfile`, `compose.yaml` e `.env` verranno creati nei passaggi seguenti.

**Attenzione a Windows:** assicurati che il file si chiami davvero `Dockerfile`, non `Dockerfile.txt`. In Esplora file abilita la visualizzazione delle estensioni oppure verifica da PowerShell:

```powershell
cd C:\docker\testedipietra
Get-ChildItem -Force
```

Il `Dockerfile` deve stare nella cartella principale, allo stesso livello di `compose.yaml`, non dentro `wordpress`.

## 6. Creare il Dockerfile per PHP 8.2

Crea `Dockerfile` con questo contenuto:

```dockerfile
FROM wordpress:php8.2-apache

RUN a2enmod rewrite
```

Questa immagine fornisce PHP 8.2 con Apache. Il volume del passaggio successivo farà utilizzare a WordPress i file già scaricati.

> Il tag `wordpress:php8.2-apache` non fissa necessariamente la versione 6.7.11 del core contenuto nell'immagine. Poiché monteremo la cartella locale completa sopra `/var/www/html`, i file del sito scaricati da Aruba saranno quelli effettivamente utilizzati. Non copiare un nuovo core sopra la tua copia senza aver prima fatto un backup.

## 7. Creare il file `.env`

Nella cartella principale crea `.env`:

```dotenv
MYSQL_ROOT_PASSWORD=imposta_una_password_locale_sicura
MYSQL_DATABASE=wordpress
MYSQL_USER=wordpress
MYSQL_PASSWORD=imposta_un_altra_password_locale
```

Sostituisci i valori di esempio con password locali. Non usare queste credenziali per un database esposto a Internet e non condividere il file `.env`.

## 8. Creare `compose.yaml`

Crea `compose.yaml` con la configurazione seguente:

```yaml
services:
  wordpress:
    build:
      context: .
      dockerfile: Dockerfile
    container_name: testedipietra-wordpress
    restart: unless-stopped
    ports:
      - "8080:80"
    environment:
      WORDPRESS_DB_HOST: db:3306
      WORDPRESS_DB_NAME: ${MYSQL_DATABASE}
      WORDPRESS_DB_USER: ${MYSQL_USER}
      WORDPRESS_DB_PASSWORD: ${MYSQL_PASSWORD}
    volumes:
      - ./wordpress:/var/www/html
    depends_on:
      - db

  db:
    image: mysql:5.5
    container_name: testedipietra-db
    restart: unless-stopped
    platform: linux/amd64
    environment:
      MYSQL_ROOT_PASSWORD: ${MYSQL_ROOT_PASSWORD}
      MYSQL_DATABASE: ${MYSQL_DATABASE}
      MYSQL_USER: ${MYSQL_USER}
      MYSQL_PASSWORD: ${MYSQL_PASSWORD}
    volumes:
      - db_data:/var/lib/mysql

  phpmyadmin:
    image: phpmyadmin:latest
    container_name: testedipietra-phpmyadmin
    restart: unless-stopped
    ports:
      - "127.0.0.1:8081:80"
    environment:
      PMA_HOST: db
      PMA_PORT: 3306
    depends_on:
      - db

volumes:
  db_data:
```

### Cosa fa la configurazione

- WordPress è raggiungibile su `http://localhost:8080`.
- Il database è raggiungibile **dai container** all'indirizzo `db:3306`; non è pubblicato direttamente sulla rete del PC.
- phpMyAdmin è raggiungibile localmente su `http://localhost:8081`.
- I file del sito si trovano nella cartella `wordpress` del PC.
- I dati del database sono conservati nel volume Docker `db_data`.

`depends_on` determina l'ordine di avvio, ma non garantisce che MySQL sia già pronto ad accettare connessioni. Se WordPress parte prima del database, attendi e controlla i log.

### Possibile problema con MySQL 5.5

Se `mysql:5.5` non può essere scaricato o non si avvia, non cambiare subito versione e non cancellare il volume. Conserva il dump SQL e verifica prima la disponibilità dell'immagine legacy e la compatibilità del dump. Un database più recente può richiedere adattamenti per ripristinare un dump MySQL 5.5.

## 9. Avviare i container

Apri PowerShell nella cartella del progetto:

```powershell
cd C:\docker\testedipietra
```

Controlla la configurazione Compose:

```powershell
docker compose config
```

Se non ci sono errori, avvia:

```powershell
docker compose up -d --build
```

Controlla lo stato:

```powershell
docker compose ps
```

Visualizza i log:

```powershell
docker compose logs --tail=100
```

Per seguire i log in tempo reale:

```powershell
docker compose logs -f
```

Per uscire dalla visualizzazione dei log premi `Ctrl+C`.

Se compare `failed to read dockerfile: open dockerfile: no such file or directory`, verifica che:
- il file esista davvero e si chiami `Dockerfile`;
- sia nella stessa cartella di `compose.yaml`;
- `build.context` sia `.` e `dockerfile` sia `Dockerfile`;
- il comando venga eseguito dalla cartella corretta.

Se compare `open //./pipe/dockerDesktopLinuxEngine: The system cannot find the file specified`, Docker Desktop o il suo motore Linux non sono pronti o il contesto Docker è errato: risolvi prima questo problema.

## 10. Correggere la configurazione del database in `wp-config.php`

Apri `wordpress/wp-config.php` e aggiorna le definizioni del database in modo che corrispondano al file `.env`:

```php
define( 'DB_NAME', 'wordpress' );
define( 'DB_USER', 'wordpress' );
define( 'DB_PASSWORD', 'imposta_un_altra_password_locale' );
define( 'DB_HOST', 'db:3306' );
```

Sostituisci la password con il valore esatto di `MYSQL_PASSWORD` in `.env`.

Non modificare `$table_prefix`: deve rimanere uguale a quello del database originale.

Le variabili `WORDPRESS_DB_*` dell'immagine ufficiale vengono usate principalmente per generare la configurazione quando `wp-config.php` non esiste. Dato che stai montando il file originale, è importante aggiornare direttamente le definizioni in `wp-config.php`.

## 11. Importare il database SQL

Attendi che il database sia avviato:

```powershell
docker compose ps
docker compose logs --tail=100 db
```

Apri `http://localhost:8081`. Accedi a phpMyAdmin con:
- Server: `db`
- Utente: `root`
- Password: il valore `MYSQL_ROOT_PASSWORD` nel file `.env`

Seleziona il database `wordpress`. Se non esiste, crealo con codifica e collation compatibili con il dump. Poi apri **Importa**, seleziona `database.sql` e avvia l'importazione.

Se l'importazione fallisce, leggi il messaggio di errore: può trattarsi di un limite di dimensione, di un problema di codifica o di incompatibilità tra versioni. Non continuare finché il database non è stato importato correttamente.

Per dump grandi, puoi usare il client MySQL nel container. Un metodo che evita di inserire la password nella riga di comando è copiare il dump nel container e usare una sessione interattiva:

```powershell
docker cp .\database.sql testedipietra-db:/tmp/database.sql
docker compose exec db mysql -u root -p wordpress
```

Quando richiesto, inserisci la password root del database. Esci dalla console MySQL con `exit;`. Per importare il file, esegui invece il comando seguente e inserisci la password quando richiesta:

```powershell
docker compose exec db sh -c "mysql -u root -p wordpress < /tmp/database.sql"
```

Se il dump contiene istruzioni per creare o selezionare un database con un altro nome, controlla il file prima di importarlo. Dopo l'importazione puoi rimuovere il file temporaneo dal container:

```powershell
docker compose exec db rm -f /tmp/database.sql
```

## 12. Impostare l'URL locale di WordPress

Il database importato contiene probabilmente il dominio pubblico Aruba. Per evitare reindirizzamenti al sito online, aggiorna `home` e `siteurl` nel database locale.

In phpMyAdmin seleziona il database, apri la scheda SQL ed esegui la query seguente, sostituendo `wp_options` se il prefisso originale è diverso:

```sql
UPDATE wp_options
SET option_value = 'http://localhost:8080'
WHERE option_name IN ('home', 'siteurl');
```

Questa operazione va eseguita **solo sul database locale**, non su quello di Aruba.

Altri URL possono essere presenti nei contenuti, nei widget, nelle opzioni dei temi e nei dati serializzati dei plugin. Per sostituire il dominio in modo sicuro, utilizza uno strumento compatibile con i dati serializzati, per esempio WP-CLI o un plugin di search-and-replace. Esegui prima un backup e una simulazione (*dry run*). Non effettuare una sostituzione globale con query SQL semplici.

## 13. Aprire e verificare il sito

Apri:

- Sito: `http://localhost:8080`
- Amministrazione: `http://localhost:8080/wp-admin`
- phpMyAdmin: `http://localhost:8081`

Accedi con le credenziali WordPress esistenti.

Se il sito non funziona, consulta:

```powershell
docker compose logs --tail=200 wordpress
docker compose logs --tail=200 db
```

Problemi frequenti:

| Sintomo | Controlli |
|---|---|
| Errore di connessione al database | `DB_HOST`, credenziali, database importato e log del servizio `db` |
| Reindirizzamento verso Aruba | Valori `home` e `siteurl` e URL residui nel database |
| Errore 404 nelle pagine interne | Presenza di `.htaccess`, modulo Apache `rewrite`, impostazioni Permalink |
| Pagina bianca o errore PHP | Log del container, plugin e tema incompatibili |
| Immagini mancanti | Cartella `wp-content/uploads` e URL memorizzati nel database |
| Aggiornamenti che chiedono FTP | Permessi di scrittura dei file nel container e valore di `FS_METHOD` |

Dopo aver verificato che il sito si apra, vai in **Impostazioni → Permalink** nell'amministrazione e salva le impostazioni se le pagine interne restituiscono 404.

## 14. Aggiornamenti WordPress, plugin e temi

WordPress può chiedere credenziali FTP se PHP non ha i permessi necessari per scrivere nei file. In un ambiente locale si può provare a usare il metodo diretto, purché i permessi siano corretti.

Nel `wp-config.php`, prima del commento finale se presente, puoi aggiungere:

```php
define( 'FS_METHOD', 'direct' );
```

Riavvia WordPress:

```powershell
docker compose restart wordpress
```

Questa impostazione non risolve da sola i problemi di permessi. Se WordPress non riesce a scrivere nella cartella montata, controlla i permessi e la proprietà dei file. Prima di aggiornare, conserva un backup dei file e del database.

Se vuoi comunque un vero server FTP per collegarti con FileZilla, aggiungi un servizio FTP dedicato e configura correttamente porte passive, credenziali e cartella condivisa. Per i soli aggiornamenti di WordPress, però, è generalmente preferibile risolvere i permessi di scrittura anziché introdurre un server FTP. Non esporre un FTP non cifrato su una rete pubblica.

## 15. Comandi Docker utili

Avviare l'ambiente:

```powershell
docker compose up -d
```

Arrestare i container senza eliminarli:

```powershell
docker compose stop
```

Riavviare WordPress:

```powershell
docker compose restart wordpress
```

Visualizzare lo stato:

```powershell
docker compose ps
```

Seguire i log:

```powershell
docker compose logs -f
```

Ricostruire l'immagine dopo aver modificato il Dockerfile:

```powershell
docker compose up -d --build
```

Rimuovere i container e la rete del progetto, mantenendo normalmente il volume nominato:

```powershell
docker compose down
```

> **Non eseguire `docker compose down -v` se vuoi conservare il database locale:** l'opzione `-v` elimina anche i volumi associati.

## 16. Backup e sicurezza

- Conserva sempre una copia intatta dei file Aruba e del dump SQL.
- Esegui backup periodici sia dei file sia del database locale.
- Non pubblicare su Internet le porte di phpMyAdmin o di un server FTP.
- Non usare password reali di produzione nel file `.env`.
- Effettua aggiornamenti e test soltanto sulla copia locale.
- Prima di aggiornare PHP, WordPress o il database, verifica la compatibilità di temi e plugin e procedi un componente alla volta.

## Riferimenti ufficiali

- [Docker Desktop](https://www.docker.com/products/docker-desktop/)
- [Immagine ufficiale WordPress su Docker Hub](https://hub.docker.com/_/wordpress)
- [Requisiti WordPress](https://wordpress.org/about/requirements/)
- [Backup WordPress](https://developer.wordpress.org/advanced-administration/security/backup/)
- [Migrare WordPress](https://developer.wordpress.org/advanced-administration/upgrade/migrating/)

---

## Checklist finale

- [ ] Docker Desktop è avviato e `docker version` mostra client e server.
- [ ] I file completi del sito Aruba sono in `wordpress/`.
- [ ] Il dump `database.sql` è stato salvato.
- [ ] `Dockerfile`, `compose.yaml` e `.env` sono nella cartella principale.
- [ ] I container sono avviati.
- [ ] `wp-config.php` punta al servizio `db:3306` e alle credenziali locali.
- [ ] Il database è stato importato correttamente.
- [ ] `home` e `siteurl` puntano a `http://localhost:8080`.
- [ ] Il sito e `/wp-admin` si aprono.
- [ ] Pagine, immagini, temi e plugin sono stati verificati.
- [ ] È disponibile un backup prima di provare gli aggiornamenti.
