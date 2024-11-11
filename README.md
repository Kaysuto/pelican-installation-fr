# 🦅 Installation complète de Pelican | Version FR - [![Discord](https://img.shields.io/discord/1027968386640117770?label=&logo=discord&logoColor=ffffff&color=7389D8&labelColor=6A7EC2)](https://discord.gg/EYzUxYd9Pk)

🖐️ Ces étapes décrivent l'installation du panel de gestion Pelican, une solution open-source.

- 📗 Avant de commencer, veuillez lire attentivement la [documentation](https://pelican.dev/docs/).
- ☕ Une connaissance de base de Linux est également recommandée !
- 🚧 Pelican est actuellement en version bêta, certaines fonctionnalités peuvent donc changer ou être cassées entre les versions.

___

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

### 📌 Dépendances
Pour le panel, vous devez installer PHP 8.3 (**recommandé**) ou 8.2, avec les extensions suivantes : 

`gd, mysql, mbstring, bcmath, xml, curl, zip, intl, sqlite3 et fpm.`

Vous aurez également besoin d'un serveur web. **Actuellement, Apache, NGINX ou Caddy sont supportés.**

Si vous souhaitez utiliser **MySQL** ou **MariaDB** pour la base de données du panel, assurez-vous d'installer soit **MySQL 8+** soit **MariaDB 10.3+**. (**client et serveur !**)

Enfin, pour certaines commandes pendant l'installation, vous aurez besoin de **curl**, **tar** et **unzip**.

```
# Installer PHP 8.3 et les extensions requises
apt-get update && apt-get install -y software-properties-common && add-apt-repository -y ppa:ondrej/php && apt-get update && apt-get install -y php8.3 php8.3-fpm php8.3-gd php8.3-mysql php8.3-mbstring php8.3-bcmath php8.3-xml php8.3-curl php8.3-zip php8.3-intl php8.3-sqlite3 curl tar unzip

# Installer un serveur web (dans cet exemple, NGINX)
apt-get install -y nginx

# Installer MySQL 8+ ou MariaDB 10.3+ (facultatif)
apt-get install -y mysql-server
# Ou
apt-get install -y mariadb-server
```

#### ⚠️ **Veuillez vous assurer d'avoir installé toutes les dépendances avant de continuer !**

___

### 🌐 Configuration de Cloudflare
Avant d'installer **Pelican**, vous devez configurer un tunnel Cloudflare "**Zero Trust**" pour sécuriser l'accès à votre panel :

1. Créez un compte [Cloudflare](https://dash.cloudflare.com/sign-up) si vous n'en avez pas déjà un.
2. Une fois le compte Cloudflare créé, rendez-vous dans l'onglet "Domains" de votre compte Cloudflare et ajoutez votre nom de domaine, voici un [tuto vidéo pour le réaliser](https://www.youtube.com/watch?v=mKki2xuD_k4).
4. Configurez un **nouveau tunnel** Cloudflare en suivant la [documentation officielle](https://developers.cloudflare.com/cloudflare-one/connections/connect-networks/get-started/create-remote-tunnel/).
5. Configurez votre domaine pour qu'il pointe vers le tunnel Cloudflare (`Public hostname`) :
   - Créez une première redirection pour votre nom de domaine principal, par exemple : `pelican.votreserveur.fr`, qui doit pointer vers votre **IP locale**.
   - Créez une deuxième redirection pour le nœud, par exemple : `node.votreserveur.fr` avec le port `8080`.
   - Dans ces deux redirections, ajoutez les paramètres supplémentaires d'application (`Additional application settings`) :
     - Sous **TLS** - **Origin Server Name**, entrez votre **IP** publique.
     - Cochez l'option "**No TLS Verify**".
6. **Assurez-vous que votre serveur web est configuré pour accepter les connexions SSL de Cloudflare.** 👉 [Configuration SSL](https://github.com/Kaysuto/pelican-installation-fr/blob/main/README.md#-configuration-du-serveur-web)

Une fois ces étapes de configuration Cloudflare terminées, vous pourrez procéder à **__l'installation de Pelican__** et accéder à votre panel via le tunnel "Zero Trust".

### 📁 Créer des Répertoires & Télécharger des Fichiers
La première étape de ce processus est de **créer** le dossier où le panel sera installé, puis de nous **déplacer** dans ce dossier nouvellement créé.

```
mkdir -p /var/www/pelican
cd /var/www/pelican
```

Une fois que vous avez créé un nouveau répertoire et que vous vous y êtes déplacé, vous devez télécharger les fichiers du panel. 
Cela se fait simplement en utilisant **curl** pour télécharger la dernière version.

```
curl -L https://github.com/pelican-dev/panel/releases/latest/download/panel.tar.gz | tar -xzv
```

### ⚙️ Installer Composer
Ensuite, nous allons configurer Composer avec les dépendances requises.

```
curl -sS https://getcomposer.org/installer | php -- --install-dir=/usr/local/bin --filename=composer
composer install --no-dev --optimize-autoloader
```

### 🔧 Configuration du Serveur Web
⚠️ Lorsque vous utilisez la configuration SSL (https) avec Cloudflare, vous **DEVEZ** configurer votre serveur pour accepter la connexion SSL.

☝️ **Configuration SSL**

```
mkdir -p /etc/certs
cd /etc/certs
openssl req -new -newkey rsa:4096 -days 3650 -nodes -x509 -subj "/C=NA/ST=NA/L=NA/O=NA/CN=Generic SSL Certificate" -keyout privkey.pem -out fullchain.pem
```

### 📝 Configuration de Nginx

**Tout d'abord, supprimez la configuration par défaut de NGINX.**

```
rm /etc/nginx/sites-enabled/default
```

Ensuite, créez un fichier appelé `pelican.conf` dans `/etc/nginx/sites-available/` et collez-y le contenu suivant, en remplaçant `<ip>` par votre adresse IP local :

```
nano /etc/nginx/sites-available/ pelican.conf
```

```
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
ln -s /etc/nginx/sites-available/pelican.conf /etc/nginx/sites-enabled/pelican.conf
systemctl restart nginx
```

### 🪧 Configuration du Panel
L'environnement principal est facilement configuré à l'aide d'une seule commande CLI et de l'installateur web intégré à l'application.

Cette étape couvrira la configuration des sessions, du cache, des identifiants de base de données et de l'envoi d'e-mails.

Retournez dans le dossier `cd /var/www/pelican`.

⚠️ **Exécuter `php artisan p:environment:setup` créera automatiquement le fichier `.env` requis et générera une `APP_KEY` si elle n'existe pas.**

**Vous souhaitez utiliser MySQL/MariaDB ?**
Assurez-vous de lire le [guide MySQL](https://pelican.dev/docs/panel/advanced/mysql) d'abord si vous souhaitez utiliser MySQL/MariaDB au lieu de SQLite !


**SAUVEGARDEZ APP_KEY !**

👉 Sauvegardez votre clé de chiffrement (APP_KEY dans le fichier `.env`). 

Cela est utilisé comme clé de chiffrement pour toutes les données qui doivent être stockées en toute sécurité (par exemple, les clés API). Rangez-la quelque part en sécurité - pas seulement sur votre serveur. Si vous la perdez, toutes les données chiffrées sont irrécupérables, même si vous avez des sauvegardes de la base de données.

### 🤚 Définir les Permissions

L'étape suivante du processus d'installation consiste à définir les bonnes permissions sur les fichiers du panel afin que le serveur web puisse les utiliser correctement.

```
chmod -R 755 storage/* bootstrap/cache/
chown -R www-data:www-data /var/www/pelican
```

### 🕰️ Configuration de Crontab (facultatif)

Nous devons créer une nouvelle tâche cron qui s'exécute chaque minute pour traiter des tâches spécifiques, telles que le nettoyage des sessions et les tâches planifiées.

```
crontab -e -u www-data
* * * * * php /var/www/pelican/artisan schedule:run >> /dev/null 2>&1
```

### 🪛 Configuration du Service de Queue

Une fois que vous avez installé le panel et configuré le cron, la dernière étape est de configurer le service de queue. Cela peut être fait avec la commande ci-dessous.

```
php artisan p:environment:queue-service
```

### 🖥️ Installateur Web
Une fois que vous avez défini les permissions appropriées et créé le Cron & le Worker de Queue, continuez l'installation via l'interface web à l'adresse `<domaine>/installer` ou `<ip>/installer`.

___

### 🚀 Installation de Wings

### 🧩 Prérequis système
- ⚠️ Veuillez noter que certains hébergeurs installent un noyau modifié qui ne prend pas en charge certaines fonctionnalités de Docker requises pour le bon fonctionnement de Wings. Vérifiez votre noyau en exécutant `uname -r`. Si votre noyau se termine par `-xxxx-grs-ipv6-64` ou `-xxxx-mod-std-ipv6-64`, vous utilisez probablement un noyau non pris en charge. Contactez votre hébergeur et demandez un noyau non modifié.

Pour exécuter **Wings**, vous aurez besoin d'un système Linux capable d'exécuter des conteneurs Docker. La plupart des VPS et presque tous les serveurs dédiés devraient être en mesure d'exécuter Docker, mais il existe des cas particuliers.

Lorsque votre fournisseur utilise Virtuozzo, OpenVZ (ou OVZ) ou LXC, vous ne pourrez probablement pas exécuter Wings. Certains fournisseurs ont apporté les modifications nécessaires pour la virtualisation imbriquée afin de prendre en charge Docker. Demandez à l'équipe d'assistance de votre fournisseur pour vous en assurer. **KVM est garanti de fonctionner.**

La façon la plus simple de vérifier est de taper `systemd-detect-virt`. Si le résultat ne contient pas `OpenVZ` ou `LXC`, cela devrait fonctionner. Le résultat de `none` apparaîtra lors de l'exécution sur du matériel dédié sans aucune virtualisation.

### 🧫 Installation de Docker
Pour une installation rapide de Docker CE, vous pouvez utiliser la commande ci-dessous :

```
curl -sSL https://get.docker.com/ | CHANNEL=stable sh
```

Si la commande ci-dessus ne fonctionne pas, veuillez vous référer à la [documentation officielle](https://docs.docker.com/) de Docker pour savoir comment installer [Docker CE](https://docs.docker.com/engine/install/) sur votre serveur.

### 🧪 Démarrer Docker au démarrage
Si vous utilisez un système d'exploitation avec systemd (Ubuntu 16+, Debian 8+, CentOS 7+), exécutez la commande ci-dessous pour que Docker démarre lorsque vous démarrez votre machine.

```
systemctl enable --now docker
```

### 🍃 Activation du swap

Sur la plupart des systèmes, Docker ne pourra pas configurer l'espace d'échange (swap) par défaut. Vous pouvez le confirmer en exécutant `docker info` et en recherchant la sortie de `WARNING: No swap limit support` près du bas.

🛑 L'activation du swap est entièrement facultative, mais nous vous recommandons de le faire si vous hébergerez pour d'autres personnes et pour éviter les erreurs OOM.

Pour activer le swap, ouvrez `/etc/default/grub` en tant qu'utilisateur root et trouvez la ligne commençant par `GRUB_CMDLINE_LINUX_DEFAULT`. Assurez-vous que la ligne inclut `swapaccount=1` quelque part à l'intérieur des guillemets.

Ensuite, exécutez `update-grub` suivi de `reboot` pour redémarrer le serveur et avoir le swap activé. 

Voici un exemple de ce à quoi la ligne devrait ressembler, ne copiez pas cette ligne textuellement. Elle comporte souvent des paramètres spécifiques au système d'exploitation.

```
GRUB_CMDLINE_LINUX_DEFAULT="swapaccount=1"
```

### 🍗 Installation de Wings

La première étape pour installer Wings est de vous assurer que nous avons la structure de répertoires requise. Pour ce faire, exécutez les commandes ci-dessous, qui créeront le répertoire de base et téléchargeront l'exécutable Wings.

```
mkdir -p /etc/pelican /var/run/wings
curl -L -o /usr/local/bin/wings "https://github.com/pelican-dev/wings/releases/latest/download/wings_linux_$([[ "$(uname -m)" == "x86_64" ]] && echo "amd64" || echo "arm64")"
chmod u+x /usr/local/bin/wings
```

### 🔧 Configuration
Une fois que vous avez installé Wings et les composants requis, l'étape suivante consiste à créer un nœud sur votre panel  installé. Accédez à la vue administrative de votre panel, sélectionnez Nodes dans la barre latérale, et sur le côté droit, cliquez sur le bouton Créer un nouveau.

![alt text](https://i.imgur.com/MggPXgj.png)

Après avoir créé un nœud avec le port 8080, cliquez dessus et il y aura un onglet appelé "**Configuration File**". Copiez le contenu du bloc de code, collez-le dans un nouveau fichier appelé `config.yml` dans `/etc/pelican` et enregistrez-le.

⚠️ N'oubliez pas de changer **cert** et **key** !

```
nano /etc/pelican/config.yml
```

Exemple :
```
debug: false
uuid: 9e7fXXX595b-XXXX-4f6e-8adc-XXXX
token_id: m9Gmhp8BXXXXXs
token: XXXtXXXX
api:
  host: 0.0.0.0
  port: 8080
  ssl:
    enabled: true
    cert: /etc/certs/fullchain.pem
    key: /etc/certs/privkey.pem
  upload_limit: 1024
system:
  data: /var/lib/pelican/volumes
  sftp:
    bind_port: 2022
allowed_mounts: []
remote: 'https://pelican.votreserveur.fr'
```

N'oubliez pas de changer le port **8080** à **443** via l'interface web avant de passer à l'étape suivante ! Laissez le port **8080** dans le fichier config.yml, sinon cela ne fonctionnera pas avec le tunnel de Cloudflare. 😉

![alt text](https://i.imgur.com/BHeXa7F.png)

### 🚀 Démarrage de Wings

Pour démarrer Wings, exécutez simplement la commande ci-dessous, qui le démarrera en mode débogage. Une fois que vous aurez confirmé qu'il fonctionne sans erreurs, utilisez CTRL+C pour terminer le processus et le mettre en arrière-plan en suivant les instructions ci-dessous.

```
wings --debug
```
Vous pouvez éventuellement ajouter le drapeau `--debug` pour exécuter Wings en mode débogage.

Vous devrez normalement avoir ceci dans <domaine>/admin/nodes :
![alt text](https://i.imgur.com/jdkTHRB.png)


### 🌓 Mise en arrière-plan (avec systemd)

Exécuter Wings en arrière-plan est une tâche simple, assurez-vous simplement qu'il fonctionne sans erreurs avant de faire cela. 

Placez le contenu ci-dessous dans un fichier appelé `wings.service` dans le répertoire `/etc/systemd/system`.

```
[Unit]
Description=Wings Daemon
After=docker.service
Requires=docker.service
PartOf=docker.service

[Service]
User=root
WorkingDirectory=/etc/pelican
LimitNOFILE=4096
PIDFile=/var/run/wings/daemon.pid
ExecStart=/usr/local/bin/wings
Restart=on-failure
StartLimitInterval=180
StartLimitBurst=30
RestartSec=5s

[Install]
WantedBy=multi-user.target
```

Ensuite, exécutez les commandes ci-dessous pour recharger **systemd** et démarrer **Wings**.

```
systemctl enable --now wings
```

___

### 🛠️ Configuration optionnelle

Vous pouvez également **protéger** l'accès à une URL de votre panel, par exemple, si vous souhaitez protéger la **partie admin** (`pelican.votreserveur.fr/admin`), vous pouvez utiliser l'**Access** de CloudFlare "Zero Trust".

Exemple :

![alt text](https://i.imgur.com/nFfb6B0.png)

Pour ce faire, dirigez-vous dans la catégorie "**Access**" et "**Applications**", puis faites "**Add an application**". Sélectionnez "**Self-hosted**", mettez un nom à votre application, par exemple "**Pelican Secure**". 

Dans la partie "**Application domain**", mettez votre sous-domaine (`pelican`), le nom de domaine (`votreserveur.fr`) et le path (`admin`). Descendez et cliquez sur "**Next**".

Entrez un nom dans la partie "**Policy Name**", et dans "**Create additional rules**", ajoutez "**Ce que vous souhaitez**" comme moyen de connexion. Par exemple, choisissez "**Emails**" et indiquez vos mails qui seront autorisés à accéder au panel admin. Ensuite, cliquez sur "**Next**" et descendez encore, puis cliquez sur "**Add application**".

🔒 **Maintenant, seuls les mails que vous avez indiqués pourront se connecter au panel admin.**


Si vous rencontrez des problèmes, consultez la documentation de dépannage :
 - https://pelican.dev/docs/troubleshooting
 - https://github.com/pelican-dev/panel
 - Ou rejoignez mon serveur Discord : https://discord.gg/EYzUxYd9Pk
 - Ou le serveur Discord officiel de Pelican : https://discord.gg/pelican-panel

**N'hésitez pas à me faire savoir si vous avez besoin d'autres modifications !**
