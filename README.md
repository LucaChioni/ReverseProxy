# Reverse Proxy

Reverse proxy centralizzato basato su **Nginx** e **Certbot**, utilizzato per gestire HTTPS e instradare il traffico verso più applicazioni Docker presenti sullo stesso server.

## Funzionalità

* Gestione delle porte HTTP `80` e HTTPS `443`
* Terminazione SSL tramite certificati Let's Encrypt
* Redirect automatico da HTTP a HTTPS
* Redirect da `www` al dominio principale
* Proxy delle richieste API verso i container applicativi
* Condivisione di una rete Docker esterna con gli altri progetti
* Rinnovo automatico dei certificati tramite cron

## Struttura del progetto

```text
reverse-proxy/
├── docker-compose.yml
├── nginx/
│   └── default.conf
├── cron_certificate_renew.txt
└── README.md
```

## Architettura

Il reverse proxy espone pubblicamente le porte:

```text
80
443
```

Le applicazioni non espongono direttamente le proprie porte verso Internet. I loro container vengono collegati alla rete Docker condivisa:

```text
reverse-proxy
```

Nginx può quindi raggiungere i servizi attraverso il nome del servizio Docker.

Esempio:

```nginx
proxy_pass http://backend:8000;
```

## Requisiti

* Docker
* Docker Compose
* Un dominio che punti all'indirizzo IP del server
* Porte `80` e `443` aperte nel firewall
* Un indirizzo email valido per Let's Encrypt

## Prima configurazione del certificato SSL

Prima di generare il certificato, Nginx deve essere avviato temporaneamente solo in HTTP, senza riferimenti a certificati ancora inesistenti.

Configurazione HTTP temporanea:

```nginx
server {
    listen 80;
    server_name lucachioni.com www.lucachioni.com;

    location /.well-known/acme-challenge/ {
        root /var/www/certbot;
    }

    location / {
        root /srv;
        try_files $uri $uri/ =404;
    }
}
```

Avviare il reverse proxy:

```bash
docker compose up -d
```

Generare il certificato:

```bash
docker compose run --rm certbot certonly \
  --webroot \
  --webroot-path=/var/www/certbot \
  --email EMAIL@example.com \
  --agree-tos \
  --no-eff-email \
  -d lucachioni.com \
  -d www.lucachioni.com
```

Dopo la generazione del certificato, ripristinare la configurazione HTTPS definitiva e ricreare Nginx:

```bash
docker compose up -d --force-recreate nginx
```

## Configurazione Nginx

Esempio di configurazione per `lucachioni.com`:

```nginx
server {
    listen 80;
    server_name lucachioni.com www.lucachioni.com;

    location /.well-known/acme-challenge/ {
        root /var/www/certbot;
    }

    location / {
        return 301 https://lucachioni.com$request_uri;
    }
}

server {
    listen 443 ssl;
    server_name www.lucachioni.com;

    ssl_certificate /etc/letsencrypt/live/lucachioni.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/lucachioni.com/privkey.pem;

    return 301 https://lucachioni.com$request_uri;
}

server {
    listen 443 ssl;
    server_name lucachioni.com;

    ssl_certificate /etc/letsencrypt/live/lucachioni.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/lucachioni.com/privkey.pem;

    root /srv;
    index index.html;

    location /api/ {
        proxy_pass http://backend:8000;

        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }

    location / {
        try_files $uri $uri/ =404;
    }

    error_page 404 /error.html;

    location = /error.html {
        root /srv;
        internal;
    }
}
```

## Rinnovo automatico dei certificati

Per verificare manualmente il rinnovo:

```bash
cd /opt/reverse-proxy
docker compose run --rm certbot renew --dry-run
```

Configurazione consigliata per il crontab:

```cron
0 3 * * * (cd /opt/reverse-proxy && docker compose run --rm certbot renew --quiet && docker compose exec -T nginx nginx -t && docker compose exec -T nginx nginx -s reload) >> /var/log/certbot-renew.log 2>&1
```

Il comando viene eseguito ogni giorno alle `03:00`.

Certbot rinnova il certificato solamente quando è vicino alla scadenza.

Per modificare il crontab:

```bash
crontab -e
```

Per controllare il log:

```bash
tail -f /var/log/certbot-renew.log
```

## Aggiungere un nuovo sito

Per aggiungere una nuova applicazione:

1. collegare il container applicativo alla rete `reverse-proxy`;
2. aggiungere un nuovo blocco `server` nella configurazione Nginx;
3. generare o estendere il certificato Let's Encrypt;
4. validare e ricaricare Nginx.
