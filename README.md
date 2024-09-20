# 🦅 Installation de Pelican | Version FR

[![Discord](https://img.shields.io/discord/1027968386640117770?label=&logo=discord&logoColor=ffffff&color=7389D8&labelColor=6A7EC2)](https://discord.gg/EYzUxYd9Pk)

☕ Le panel Pelican est conçu pour fonctionner sur votre propre serveur web.
 - ⚠️ Avant de commencer, veuillez lire attentivement la [documentation](https://pelican.dev/docs/). 
 - ⚠️ Une connaissance de base de Linux est également recommandée !
 - ⚠️ Veuillez noter que Pelican est actuellement en version bêta, ce qui signifie que certaines fonctionnalités peuvent changer ou être cassées entre les versions.

### 🌐 Configuration de Cloudflare
Avant d'installer **Pelican**, vous devez configurer un tunnel Cloudflare "**Zero Trust**" pour sécuriser l'accès à votre panel :

1. Créez un compte [Cloudflare](https://dash.cloudflare.com/sign-up) si vous n'en avez pas déjà un.
2. Configurez un **nouveau tunnel** Cloudflare en suivant la [documentation officielle](https://developers.cloudflare.com/cloudflare-one/connections/connect-apps/install-and-setup/tunnel-guide/).
3. Configurez votre domaine pour qu'il pointe vers le tunnel Cloudflare :
   - Créez une première redirection pour votre nom de domaine principal, par exemple : `pelican.votreserveur.fr`, qui doit pointer vers votre **IP**. (Vous pouvez utiliser votre IP locale)
   - Créez une deuxième redirection pour le nœud, par exemple : `node.votreserveur.fr` avec le port `8080`.
   - Dans ces deux redirections, ajoutez les paramètres supplémentaires d'application :
     - Sous **TLS** - **Origin Server Name**, entrez votre **IP** publique ou locale.
     - Cochez l'option "**No TLS Verify**".
4. Assurez-vous que votre serveur web est configuré pour accepter les connexions SSL de Cloudflare.

Une fois ces étapes de configuration Cloudflare terminées, vous pourrez procéder à l'installation de Pelican et accéder à votre panel via le tunnel "Zero Trust".


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

```
# Installer PHP 8.3 et les extensions requises
sudo apt-get update
sudo apt-get install -y software-properties-common
sudo add-apt-repository -y ppa:ondrej/php
sudo apt-get install -y php8.3 php8.3-fpm php8.3-gd php8.3-mysql php8.3-mbstring php8.3-bcmath php8.3-xml php8.3-curl php8.3-zip php8.3-intl php8.3-sqlite3

# Installer un serveur web (dans cet exemple, NGINX)
sudo apt-get install -y nginx

# Installer MySQL 8+ ou MariaDB 10.3+
sudo apt-get install -y mysql-server
# Ou
sudo apt-get install -y mariadb-server

# Installer les outils supplémentaires
sudo apt-get install -y curl tar unzip
```

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
⚠️ Lorsque vous utilisez la configuration SSL (https) avec Cloudflare, vous **DEVEZ** configurer votre serveur pour accepter la connexion SSL.

☝️ **Configuration SSL**

```
mkdir -p /etc/certs
cd /etc/certs
openssl req -new -newkey rsa:4096 -days 3650 -nodes -x509 -subj "/C=NA/ST=NA/L=NA/O=NA/CN=Generic SSL Certificate" -keyout privkey.pem -out fullchain.pem
```

📝 **Configuration de Nginx**

Tout d'abord, supprimez la configuration par défaut de NGINX.

```
rm /etc/nginx/sites-enabled/default
```

Ensuite, créez un fichier appelé `pelican.conf` dans `/etc/nginx/sites-available/` et collez-y le contenu suivant, en remplaçant `<ip>` par votre adresse IP publique ou locale :

```
/etc/nginx/sites-available/pelican.conf
server_tokens off;

server {
    listen 80;
    server_name <ip>;
    return 301 https://$server_name$request_uri;
}

server {
    listen 443 ssl http2;
    server_name <ip>;

    root /var/www/pelican/public;
    index index.php;

    access_log /var/log/nginx/pelican.app-access.log;
    error_log  /var/log/nginx/pelican.app-error.log error;

    client_max_body_size 100m;
    client_body_timeout 120s;

    sendfile off;

    ssl_certificate /etc/certs/fullchain.pem;
    ssl_certificate_key /etc/certs/privkey.pem;
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

Une fois que vous avez créé le fichier de configuration, activez-le en créant un lien symbolique :

```
sudo ln -s /etc/nginx/sites-available/pelican.conf /etc/nginx/sites-enabled/pelican.conf
systemctl restart nginx
```

### 🪧 Configuration du Panel
L'environnement principal est facilement configuré à l'aide d'une seule commande CLI et de l'installateur web intégré à l'application.

Cette étape couvrira la configuration des sessions, du cache, des identifiants de base de données et de l'envoi d'e-mails.

Exécuter `php artisan p:environment:setup` créera automatiquement le fichier `.env` requis et générera une `APP_KEY` si elle n'existe pas.

**Vous souhaitez utiliser MySQL/MariaDB ?**
Assurez-vous de lire le guide MySQL d'abord si vous souhaitez utiliser MySQL/MariaDB au lieu de SQLite !


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

Si vous rencontrez des problèmes, consultez la documentation de dépannage :
 - https://pelican.dev/docs/troubleshooting
 - Ou rejoignez mon serveur Discord : https://discord.gg/EYzUxYd9Pk
 - Ou le serveur Discord officiel de Pelican : https://discord.gg/pelican-panel

**N'hésitez pas à me faire savoir si vous avez besoin d'autres modifications !**





