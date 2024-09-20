# 🦅 Installation de Pterodactyl | Version FR

[![Discord](https://img.shields.io/discord/1027968386640117770?label=&logo=discord&logoColor=ffffff&color=7389D8&labelColor=6A7EC2)](https://discord.gg/EYzUxYd9Pk)

☕ Le panel **Pelican** est conçu pour fonctionner sur votre serveur web.

- ⚠️ **Vous êtes encouragé à lire la [documentation](https://pelican.dev/docs/).**

- ⚠️ **Vous devez avoir une familiarité de base avec Linux avant de continuer !**

- ⚠️ **Pelican est actuellement en version bêta ! Certaines choses peuvent changer ou se casser entre les versions bêta !**

### 💽 Choisir un Système d'Exploitation (OS)
**Pelican** fonctionne sur une large gamme de systèmes d'exploitation, donc choisissez celui avec lequel vous êtes le plus à l'aise. 

🗒️ Note : D'autres systèmes d'exploitation non listés ci-dessous pourraient également fonctionner.

⚠️ **OpenVZ, à moins d'être spécifiquement configuré, ne fonctionnera pas avec Pelican.**

| Système d'Exploitation | Version | Supporté | Notes |
|------------------------|---------|----------|-------|
| Ubuntu                 | 20.04   | ⚠️       | Pas de support SQLite, EoL d'Ubuntu 20.04 en avril 2025, non recommandé |
|                        | 22.04   | ✅       |       |
|                        | 24.04   | ✅       | Documentation écrite en supposant Ubuntu 24.04 comme OS de base. |
| Rocky Linux            | 9       | ✅       |       |
| Debian                 | 11      | ⚠️       | Pas de support SQLite |
|                        | 12      | ✅       |       |

Le support SQLite dépend de `libsqlite3-0_3.35+` étant présent sur le système hôte. 

**Ubuntu 20.04 et Debian 11 ne répondent pas à cette exigence.**

### 📌 Dépendances
Pour le panel, vous devez installer PHP 8.3 (**recommandé**) ou 8.2, avec les extensions suivantes : 

`gd, mysql, mbstring, bcmath, xml, curl, zip, intl, sqlite3 et fpm.`

Vous aurez également besoin d'un serveur web. **Actuellement, Apache, NGINX ou Caddy sont supportés.**

Si vous souhaitez utiliser **MySQL** ou **MariaDB** pour la base de données du panel, assurez-vous d'installer soit **MySQL 8+** soit **MariaDB 10.3+**. (**client et serveur !**)

Enfin, pour certaines commandes pendant l'installation, vous aurez besoin de **curl**, **tar** et **unzip**.

## ⚠️ **Veuillez vous assurer d'avoir installé toutes les dépendances nécessaires avant de continuer !**

### 📁 Créer des Répertoires & Télécharger des Fichiers
La première étape de ce processus est de **créer** le dossier où le panel sera installé, puis de nous **déplacer** dans ce dossier nouvellement créé.

```
mkdir -p /var/www/pelican
cd /var/www/pelican
```

Une fois que vous avez créé un nouveau répertoire et que vous vous y êtes déplacé, vous devez télécharger les fichiers du panel. 
Cela se fait simplement en utilisant **curl** pour télécharger la dernière version.

```
curl -L https://github.com/pelican-dev/panel/releases/latest/download/panel.tar.gz | sudo tar -xzv
```

### ⚙️ Installer Composer
Ensuite, nous allons configurer Composer avec les dépendances requises.
```
curl -sS https://getcomposer.org/installer | sudo php -- --install-dir=/usr/local/bin --filename=composer
sudo composer install --no-dev --optimize-autoloader
```
### 🔧 Configuration du Serveur Web
⚠️ Lorsque vous utilisez la configuration SSL (https), vous **DEVEZ** créer des certificats SSL, sinon votre serveur web ne pourra pas démarrer. 

Consultez la **page de documentation** sur la [création de certificats SSL](https://pelican.dev/docs/guides/ssl) pour apprendre comment créer ces certificats avant de continuer.

Nous utiliserons Nginx.

Tout d'abord, supprimez la configuration par défaut de NGINX.

```
rm /etc/nginx/sites-enabled/default
```

Maintenant, vous devez coller le contenu du fichier ci-dessous, en remplaçant `<domain>` par votre domaine ou votre IP, dans un fichier appelé `pelican.conf` et placez ce fichier dans `/etc/nginx/sites-available/`.

### Configuration HTTPS
⚠️ Note : Les IP ne peuvent pas être utilisées avec SSL.

```
/etc/nginx/sites-available/pelican.conf
server_tokens off;

server {
    listen 80;
    server_name <domain>;
    return 301 https://$server_name$request_uri;
}

server {
    listen 443 ssl http2;
    server_name <domain>;

    root /var/www/pelican/public;
    index index.php;

    access_log /var/log/nginx/pelican.app-access.log;
    error_log  /var/log/nginx/pelican.app-error.log error;

    client_max_body_size 100m;
    client_body_timeout 120s;

    sendfile off;

    ssl_certificate /etc/letsencrypt/live/<domain>/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/<domain>/privkey.pem;
    ssl_session_cache shared:SSL:10m;
    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_ciphers "ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384:ECDHE-ECDSA-CHACHA20-POLY1305:ECDHE-RSA-CHACHA20-POLY1305:DHE-RSA-AES128-GCM-SHA256:DHE-RSA-AES256-GCM-SHA384";
    ssl_prefer_server_ciphers on;

    add_header X-Content-Type-Options nosniff;
    add_header X-XSS-Protection "1; mode=block";
    add_header X-Robots-Tag none;
    add_header Content-Security-Policy "frame-ancestors 'self'";
    add_header X-Frame-Options DENY;
    add_header Referrer-Policy same-origin;

    location / {
        try_files $uri $uri/ /index.php?$query_string;
    }

    location ~ \.php$ {
        fastcgi_split_path_info ^(.+\.php)(/.+)$;
        fastcgi_pass unix:/run/php/php8.3-fpm.sock;
        fastcgi_index index.php;
        include fastcgi_params;
        fastcgi_param PHP_VALUE "upload_max_filesize = 100M \n post_max_size=100M";
        fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
        fastcgi_param HTTP_PROXY "";
        fastcgi_intercept_errors off;
        fastcgi_buffer_size 16k;
        fastcgi_buffers 4 16k;
        fastcgi_connect_timeout 300;
        fastcgi_send_timeout 300;
        fastcgi_read_timeout 300;
        include /etc/nginx/fastcgi_params;
    }

    location ~ /\.ht {
        deny all;
    }
}
```

### ✨ Activer la Configuration
La dernière étape consiste à activer votre configuration **NGINX** et à la redémarrer.

```
sudo ln -s /etc/nginx/sites-available/pelican.conf /etc/nginx/sites-enabled/pelican.conf
```
### 🪧 Configuration du Panel
L'environnement principal est facilement configuré à l'aide d'une seule commande CLI et de l'installateur web intégré à l'application.

Cette étape couvrira la configuration des sessions, du cache, des identifiants de base de données et de l'envoi d'e-mails.

Exécuter `php artisan p:environment:setup` créera automatiquement le fichier `.env` requis et générera une `APP_KEY` si elle n'existe pas.

**Vous souhaitez utiliser MySQL/MariaDB ?**
Assurez-vous de lire le guide MySQL d'abord si vous souhaitez utiliser MySQL/MariaDB au lieu de SQLite !

```
php artisan p:environment:setup
```

**SAUVEGARDEZ APP_KEY !**
Sauvegardez votre clé de chiffrement (APP_KEY dans le fichier `.env`). Cela est utilisé comme clé de chiffrement pour toutes les données qui doivent être stockées en toute sécurité (par exemple, les clés API). Rangez-la quelque part en sécurité - pas seulement sur votre serveur. Si vous la perdez, toutes les données chiffrées sont irrécupérables, même si vous avez des sauvegardes de la base de données.

### 🤚 Définir les Permissions
L'étape suivante du processus d'installation consiste à définir les bonnes permissions sur les fichiers du panel afin que le serveur web puisse les utiliser correctement.

```
chmod -R 755 storage/* bootstrap/cache/
chown -R www-data:www-data /var/www/pelican
```

### 🕰️ Configuration de Crontab (facultatif)
Nous devons créer une nouvelle tâche cron qui s'exécute chaque minute pour traiter des tâches spécifiques, telles que le nettoyage des sessions et les tâches planifiées. Vous voudrez ouvrir votre crontab.

```
sudo crontab -e -u www-data
* * * * * php /var/www/pelican/artisan schedule:run >> /dev/null 2>&1
```

### 🪛 Configuration du Service de Queue
Une fois que vous avez installé le panel et configuré le cron, la dernière étape est de configurer le service de queue. Cela peut être fait avec la commande ci-dessous.

```
sudo php artisan p:environment:queue-service
```

### 🖥️ Installateur Web
Une fois que vous avez défini les permissions appropriées et créé le Cron & le Worker de Queue, continuez l'installation via l'interface web à l'adresse `<domain>/installer` ou `<ip>/installer`.

**N'hésitez pas à me faire savoir si vous avez besoin d'autres modifications !**
