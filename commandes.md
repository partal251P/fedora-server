# Commandes utiles:

Avoir l'ip locale (l'ip publique peut se trouver avec internet):

```bash
ip a
```

Vérifier l'état de la syncro du raid:

```bash
cat /proc/mdstat
```

Vérifier l'état des disques:

```bash
mdadm --detail /dev/md0
```


Faire la mise a jour du système:

```bash
dnf upgrade -y
sudo dnf autoremove -y
sudo dnf clean all
```

Faire un speedtest

```bash
python3 speedtest-cli
```

Voir tout les timers/services actifs:

```bash
systemctl list-timers --all
```
<br></br>
<br></br>
<br></br>
<br></br>

# Server selfhost Davidscloud.live



# Table des matières

- [Commandes utiles](#commandes-utiles)
- [Server selfhost Davidscloud.live](#server-selfhost-davidscloudlive)
  - [1. Installation](#1-installation)
  - [2. Créer le RAID](#2-créer-le-raid)
  - [3. Après installation](#3-après-installation)
  - [4. Vérifier que le RAID est OK](#4-vérifier-que-le-raid-est-ok)
  - [5. Installation Nextcloud](#5-installation-nextcloud)
  - [6. Correction des messages d'erreur dans Nextcloud](#6-correction-des-messages-derreur-dans-nextcloud)
  - [7. Acheter un domaine pour le serveur](#7-acheter-un-domaine-pour-le-serveur)
  - [8. Speedtest](#8-speedtest)
  - [9. Faire en sorte que le serveur idle a des heures précises](#9-faire-en-sorte-que-le-serveur-idle-a-des-heures-précises)









## 1. Installation
### a. Wipe toutes les partitions des disques durs dans l'installateur fedora

**Ctrl + Alt + F2** pour afficher le terminal dans l'installateur

```bash
sudo wipefs -a /dev/sdb
sudo wipefs -a /dev/sdc
sudo sgdisk --zap-all /dev/sdb
sudo sgdisk --zap-all /dev/sdc
```
Puis **Ctrl + Alt + F6** pour revenir a l'installateur

## 2. Créer le RAID
Créer les partitions RAID sur `sda` et `sdb`:

```bash
fdisk /dev/sda
```
* -g (pour créer une table GPT)

* -n (nouvelle partition)

* -[ENTER] (partition 1)

* -[ENTER] (début par défaut)

* -[ENTER] (fin par défaut, donc tout le disque)

* -t → fd (type Linux RAID)

* -w pour écrire

Même chose pour **/dev/sdb**

Créer l'ensemble RAID:

```bash
mdadm --create --verbose /dev/md0 --level=1 --raid-devices=2 /dev/sda1 /dev/sdb1
```

Formater le RAID EN **ext4**:

```bash
mkfs.ext4 /dev/md0
```

Puis on va créer un point de montage:

```bash
sudo mkdir /mnt/raid
sudo mount /dev/md0 /mnt/raid
```
Rendre le montage persistant:

```bash
blkid /dev/md0
```

Il faut copier le UUID (ex : UUID=xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx) puis:
 
```bash
nano /etc/fstab
```

Ajoute la ligne dans le fichier ouvert:

```bash
UUID=le_fameux_uuid /mnt/raid ext4 defaults 0 2
```

Sauvegarder la config RAID pour qu'elle survive au redémarrage:

```bash
sudo mdadm --detail --scan >> /etc/mdadm.conf
```
<span style="color:#fc2c03"> Attention ça peut varier

## 3. Après installation

```bash
dnf upgrade -y
```
(Optionnel: Nettoyage après mise a jour)

```bash
sudo dnf autoremove -y
sudo dnf clean all
```
## 4. Vérifier que le RAID est OK
```bash
cat /proc/mdstat
```
Normalement, le résultat devrait être comme:

```java
md0 : active raid1 sda1[0] sdb1[1]
      3906887488 blocks super 1.2 [2/2] [UU]
```
* **md0** : Nom du volume RAID
* **[2/2]** : 2 disques sur 2 sont actifs
* **[UU]** : Les deux disques sont **Up**

On vérifie les détails avec mdadm:

```bash
mdadm --detail /dev/md0
```
Cela doit afficher:

* **State : clean**
* **Active Devices : 2**
* **Working Devices : 2**
* **Failed Devices : 0**
* **Spare Devices : 0**

On vérifie le point de montage:

```bash
df -h
```
On doit voir **/dev/md0** par exemple:

```bash
/dev/md0        3.6T  1.2G  3.4T   1% /mnt/raid
```

Et on **reboot** 

<span style="color:#fc2c03"> Attention ! </span>
Juste après l'installation, fedora nous laisse pas shutdown, reboot ou idle, **c'est normal**! Il faut que le RAID se syncronise (et ça prend du temps, beaucoup de temps. Quelques heures perso)
On peut vérifier ce qui bloque en faisant:

```bash
systemd-inhibit --list
```

On peut suivre la progression avec:

```bash
cat /proc/mdstat
```

## 5. Installation Nextcloud

On va installer Apache, PHP et MariaDB:

```bash
dnf install -y httpd mariadb-server php php-mysqlnd php-gd php-xml php-mbstring php-json php-curl php-zip php-bcmath php-intl php-cli unzip wget
```

On active et démarre les services:

```bash
systemctl enable --now httpd mariadb
mysql_secure_installation
```

Puis suivre les instructions pour:

* Définir un mot de passe root
* Supprimer les utilisateurs anonymes (oui)
* Désactiver l'accès root distant
* Supprimer la base de test (oui)
* Recharger les privilèges

On crée une base de données pour Nextcloud:

```bash
mysql -u root -p
```

Puis entrer:

```sql
CREATE DATABASE nextcloud CHARACTER SET utf8mb4 COLLATE utf8mb4_general_ci;
CREATE USER 'nextclouduser'@'localhost' IDENTIFIED BY 'motdepassefortquetuvaschangerbiensur';
GRANT ALL PRIVILEGES ON nextcloud.* TO 'nextclouduser'@'localhost';
FLUSH PRIVILEGES;
EXIT;
```
*Remplacer **motdepassefortquetuvaschangerbiensur** par un vrai mot de passe*

Télécharger et installer Nextcloud sur la machine:

```bash
cd /var/www
wget https://download.nextcloud.com/server/releases/latest.zip
unzip latest.zip
chown -R apache:apache nextcloud
```

Déplacer le dossier **data** sur le RAID:

```bash
mkdir -p /mnt/raid/nextcloud-data
chown -R apache:apache /mnt/raid/nextcloud-data
```

Configuration d'Apache:

```bash
nano /etc/httpd/conf.d/nextcloud.conf
```

Et ajouter dans ce même fichier:

```apache
<VirtualHost *:80>
    DocumentRoot /var/www/nextcloud
    ServerName ton-domaine-ou-ip

    <Directory /var/www/nextcloud/>
        Require all granted
        AllowOverride All
        Options FollowSymLinks MultiViews
    </Directory>
</VirtualHost>
```
<span style="color:#fc2c03"> Remplacer **ton-domaine-ou-ip** par l'ip local</span>


On active les modules Apache nécessaire:

```bash
dnf install -y mod_ssl
setsebool -P httpd_can_network_connect_db 1
systemctl restart httpd
```

On ouvre les ports du pare-feu:

```bash
firewall-cmd --permanent --add-service=http
firewall-cmd --permanent --add-service=https
firewall-cmd --reload
```

Normalement, la page en local est bien load mais il y a un message d'erreur qui nous dit "**Can't write into config directory! This can usually be fixed by giving the webserver write access to the config directory.**". Cela veut dire que le serveur web n'a pas les droits suffisants pour modifier les fichiers de configuration de Nextcloud

Pour fix ça il faut:

### Etape 1: Localiser le dossier Nextcloud

```bash
cd /var/www/nextcloud
```

<span style="color:#fc2c03"> Attention, le chemin peut être différent comme par exemple /var/www/html/nextcloud ! </span>

### Etape 2: Donner les droits au serveur web (Apache)

```bash
chown -R apache:apache /var/www/nextcloud
chmod -R 755 /var/www/nextcloud
chown -R apache:apache /var/www/nextcloud/config
```

### Etape 3 (si SELinux est activé)
Fedora utilise souvent **SELinux**, qui peut bloquer l’accès même si les permissions sont bonnes.

```bash
chcon -R -t httpd_sys_rw_content_t /var/www/nextcloud
```

Puis restart Apache:

```bash
systemctl restart httpd
```

Il se peut qu'il y ai une erreur du genre *"**Cannot write into "apps" directory. This can usually be fixed by giving the web server write access to the apps directory or disabling the App Store in the config file.**"*

On vérifie les **droits SELinux**:

```bash
ls -lZ /var/www/nextcloud/config
```

Et ça doit donner un truc du genre:

```arduino
drwxr-xr-x. apache apache system_u:object_r:httpd_sys_rw_content_t:s0 config
```

Si c'est pas le cas, il faut faire:

```bash
chcon -R -t httpd_sys_rw_content_t /var/www/nextcloud
sudo systemctl restart httpd
```

<span style="color:#fc2c03"> Monter le RAID comme dossier de données</span>
On a déjà fait le RAID donc maintenant il faut juste modifier les permission et déplacer le dossier:

```bash
chown -R apache:apache /mnt/raid
chcon -R -t httpd_sys_rw_content_t /mnt/raid
```

On déplace le dossier des données:

```bash
systemctl stop httpd
rsync -av /var/www/nextcloud/data/ /mnt/raid/data/
```

Il faut modifier le fichier config:

```bash
nano /var/www/nextcloud/config/config.php
```

Trouve la ligne suivante:

```php
'datadirectory' => '/var/www/nextcloud/data',
```

Puis change en:

```php
'datadirectory' => '/mnt/raid/data',
```

Et finalement on revérifie les permissions du nouveau dossier:

```bash
chown -R apache:apache /mnt/raid/data
chcon -R -t httpd_sys_rw_content_t /mnt/raid/data
```

## 6. Correction des messages d'erreur dans Nextcloud
### 1. Installer le module PHP manquant : `posix`

Cela crée une erreur fatale, il faut l'installer:

```bash
dnf install php-posix
systemctl restart httpd
```

### 2. Augmenter la mémoire PHP

Il faut éditer ce fichier:

```bash
nano /etc/php.ini
```

Il faut touver la ligne:

```ini
memory_limit = 128M
```

Et la **remplacer** par:

```ini
memory_limit = 512M
```

Puis on restart sytemctl

### 3. Protéger le dossier `/data`

Il faut vérifier que le dossier `/var/www/nextcloud/data` **n'est pas accessible depuis le web**.
On place un `.htaccess` qui contient

```apache
Deny from all
```
### 4. Corriger OPcache

On va éditer `/etc/php.d/10-opcache.ini` et on change le 8 par le 16

```ini
opcache.interned_strings_buffer=16
```

### 5. Activer les `.js.map` et `.mjs` dans Apache

On tape

```bash
nano /etc/mime.types
```

On ajoute ces lignes si elles ne sont pas déjà présentes:

```apache
application/javascript           js mjs
application/json                 map
```

Et on redémarre Apache:

```bash
systemctl restart apache2      # Debian/Ubuntu
systemctl restart httpd        # CentOS/RHEL
```


On peut **tester** pour voir si tout est OK:

```bash
curl -I http://ton-serveur/nextcloud/apps/files/js/filelist.js.map
```

On devrait avoir:

```pgsql
Content-Type: application/json
```

### 6. Problème d'accès à la data directory

```bash
nano /var/www/html/nextcloud/config/config.php
```

Et il faut **s'assurer qu'il y a quelque chose comme ça**:

```php
'trusted_domains' =>
array (
  0 => 'davidscloud.live',
  1 => 'localhost',
  2 => '127.0.0.1',
),
'overwrite.cli.url' => 'https://davidscloud.live',
```

Puis on redémarre:

```bash
systemctl restart apache2
```

## 7. Acheter un domaine pour le serveur

Perso j'ai acheter sur Namecheap donc je vais expliquer en fonction de ça (mais chaque fournisseur dois avoir la même chose)
### 1. Retier n'importe quelle redirection
Exemple:

```arduino
davidscloud.live → http://www.davidscloud.live
```
Il faut le retirer

Il faut se rendre dans la partie **Advanced DNS**, dans la secttion **Host Records** et ajouter

| Type    | Host | Value (adresse IP)    | TTL |
| -------- | ------- | -------- | ------- |
| A  | @  | `123.123.123.123` *(→ remplace par l’IP publique de ton serveur)*  | Automatic    | 
| A | www | `123.123.123.123` *(→ remplace par l’IP publique de ton serveur)*  | Automatic    | 

<span style="color:#fc2c03">Ne pas oublier d'ouvrir les ports 80 et 443</span>

Si vous voulez verifier vos ports, [voici un site](https://www.yougetsignal.com/tools/open-ports/)

### 2. Tunnel Cloudflare

Il faut créer un compte chez eux avant toute chose, ajouter le domaine et installer cloudflared sur le server. Attention ! Il faut ajouter dans **Domain**, les **Nameserver** que Cloudlfare vous donne !

```bash
wget https://github.com/cloudflare/cloudflared/releases/latest/download/cloudflared-linux-amd64
```

On le rend exécutable et on vérifie que tout fonctionne:

```bash
chmod +x cloudflared-linux-amd64
cloudflared --version
```

Il faut ensuite se **connecter** sur le serveur, **créer** le tunnel, associer un **sous-domaine**, puis **démarrer** le tunnel:

```bash
cloudflared tunnel login
cloudflared tunnel create davidscloud
cloudflared tunnel route dns davidscloud davidscloud.live
cloudflared tunnel run davidscloud
```

(pour moi ca n'a pas marché de faire un tunnel donc j'ai juste laissé les nameserveurs et cloudflare donc j'utilise toujours cloudflare mais pas pour les ports)

### 3. Activer le https

```bash
dnf install certbot python3-certbot-apache
certbot --apache
```

On va aussi renouveller automaticement le certificat:

```bash
systemctl list-timers | grep certbot
```

Et on peut **forcer un test**:

```bash
certbot renew --dry-run
systemctl restart httpd
```

## 8. Speedtest

Si vous avez pas déjà installé **python**:

```bash
dnf install python3 -y
```

On va download les fichiers, on rend les fichiers exécutable, puis on fait le speedtest

```bash
wget -O speedtest-cli https://raw.githubusercontent.com/sivel/speedtest-cli/master/speedtest.py
chmod +x speedtest-cli
python3 speedtest-cli
```
***En sachant que la vitesse de l'upload est la vitesse à laquelle les utilisateurs pourront upload les fichiers sur nextcloud (ex: upload -> 30Mbit/s => vitesse d'upload d'une image sur le serveur bloquée à 30 Mbit/s)***

[Voici la source](https://utho.com/docs/linux/fedora/how-to-test-internet-speed-on-fedora/)


## 9. Faire en sorte que le serveur idle a des heures précises

On va devoir créer 3 fichiers: `suspend-with-wake.sh`, `suspend-at-night.service` et ``suspend-at-night.timer``

Le fichier `suspend-with-wake.sh` nous permet de mettre en veille le serveur et le réveiller 8h après (on peux modifier en fonction des besoins):

```bash
sudo nano /usr/local/bin/suspend-with-wake.sh
```


```bash
#!/bin/bash

# Nombre de secondes avant le réveil (par exemple 8h = 8*3600 = 28800)
WAKE_TIME=$((8 * 3600))

# Mettre la machine en veille avec réveil automatique
/usr/sbin/rtcwake -m mem -s $WAKE_TIME
```

Puis on le rend éxecutable:

```bash
sudo chmod +x /usr/local/bin/suspend-with-wake.sh
```

*On peut vérifier que le fichier est bien un éxecutable avec la commande  ``ls -l /usr/local/bin/suspend-with-wake.sh``*
*Il faut que ça affiche ``-rwxr-xr-x``*



Le fichier `suspend-at-night.service` nous permet d'éxecuter le ficher sh:

```bash
sudo nano /etc/systemd/system/suspend-at-night.service
```

```bash
[Unit]
Description=Suspend and wake service
After=network.target

[Service]
Type=oneshot
ExecStart=/usr/local/bin/suspend-wake.sh
```

Finalement, le fichier `suspend-at-night.timer` permet d'éxecuter le fichier service a une heure spéciale:

```bash
sudo nano /etc/systemd/system/suspend-at-night.timer
```


```bash
[Unit]
Description=Timer for suspend and wake

[Timer]
OnCalendar=*-*-* 23:00:00
Persistent=true

[Install]
WantedBy=timers.target
```

Il faut pour tout mettre en place, recharger les timers:


```bash
sudo systemctl daemon-reexec
sudo systemctl daemon-reload
sudo systemctl enable --now suspend-at-night.timer
```

On peut vérifier tout les timers:


```bash
systemctl list-timers --all
```

## 10. Ajouter un outil de monitoring:

### 1. Installer Docker:

Docker sert a créer des cellules (un peu comme une machine virtuelle) pour chaque service qu'on veut installer. Dans mon cas, je vais commencer par installer Kuma qui permet de ping le serveur et de m'avertir si le server c'est en ligne ou pas.

On installe Docker:

```bash
sudo dnf install -y dnf-plugins-core
sudo curl -o /etc/yum.repos.d/docker-ce.repo https://download.docker.com/linux/fedora/docker-ce.repo
sudo dnf install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin -y
sudo systemctl enable --now docker
```

On va ensuite tester si Docker marche ou pas:

```bash
docker run hello-world
```

### 2. Installer Uptime Kuma:

```bash
docker volume create uptime-kuma

docker run -d \
  --restart=always \
  -p 3001:3001 \
  -v uptime-kuma:/app/data \
  --name uptime-kuma \
  louislam/uptime-kuma
```

(Optionnel: on peut rajouter une fonction qui shutdown le serveur a disatance grave a pyhton):

```python                                                                                
from flask import Flask, request, abort
import subprocess
import os

app = Flask(__name__)

SECRET_TOKEN = "e5367ac5aa4a894363b3e3bd410bf6fb425e79544edd801c29c9f74ec2adc8b7"

@app.route('/shutdown', methods=['POST'])
def shutdown():
    data = request.get_json()
    if not data or 'token' not in data:
        return jsonify({"error": "Token manquant"}), 400
    if data['token'] != SECRET_TOKEN:
        return jsonify({"error": "Token invalide"}), 403
    os.system("shutdown now")
    return jsonify({"message" : "Shutdown command sent"}), 200


if __name__ == '__main__':
    app.run(host='0.0.0.0', port=5005)
```

### 3. Configurer le sous domaine (dans cloudflare):

Dans cloudflare, on va créer un nouvel enregistrement DNS de type A:

* Nom : status

* Cible : IP publique ([On peut obtenir avec ce site](https://www.monippublique.com/) )

* Proxied (nuage orange) : Activé

### 4. Installation de Caddy:
Ca nous permet de plus facilement déployer en HTTPS des sous-domaine dans notre cas. On va donc l'installer:

```bash
dnf install 'dnf-command(copr)' -y
dnf copr enable @caddy/caddy -y
dnf install caddy -y
```

On va créer un fichier de configuration:

```bash
sudo nano /etc/caddy/Caddyfile
```

Et on va supprimer le bloc "http:// { ...} et mettre:

```caddy
status.lenomdedomaine (dans mon cas: status.davidscloud.live) {
    reverse_proxy 127.0.0.1:3001 {
        header_up Host {host}
        header_up X-Real-IP {remote}
        header_up X-Forwarded-For {remote}
        header_up X-Forwarded-Proto {scheme}
    }
}

lenomdedomaine (dans mon cas: davidscloud.live){
    root * /var/www/nextcloud
    php_fastcgi unix//run/php-fpm/www.sock
    file_server
    header {
      X-Forwarded-For {remote_host}
      X-Forwarded-Proto {scheme}
    }
}

```
Attention: Si on ajoute pas le deuxieme bloc, le sous-domaine marche mais pas nextcloud (il devrait y avoir l'erreur "SSL handshake failed Error code 525").

Ensuite on va configurer Nextcloud pour faire confiance à Caddy:

```bash
sudo nano /var/www/nextcloud/config/config.php
```

Et on va ajouter 2 lignes:

```php
<?php
$CONFIG = array (
  ........
  'trusted_proxies' => ['127.0.0.1'],
  'forwarded_for_headers' => ['HTTP_X_FORWARDED_FOR'],
);
```


Et on peut vérifier que le fichier est valide avec et si tout est bon, on peut restart caddy et php:

```bash
sudo caddy validate --config /etc/caddy/Caddyfile
sudo systemctl restart caddy
sudo systemctl restart php-fpm
```

C'est pas fini! Il faut maintenant modifier le fichier www.conf pour dire que caddy peut acceder au socket. Ca veut dire quand caddy essaie d'envoyer les requêtes PHP via ce socket, il n’a pas les droits nécessaires → 502 Bad Gateway.

bash
sudo nano /etc/php-fpm.d/www.conf
```

Il faut chercher les lignes:

```ini
listen.owner = caddy
listen.group = caddy
listen.mode = 0660
.......
listen.acl_users = apache,nginx,caddy
listen.acl_groups = caddy
listen.mode = 0660
```

Attention: Il faut enlever les ; au début des lignes

On restart pour appliquer les modifications:

```bash
systemctl restart php-fpm
```

Normalement tout marche !

## 11. Installtion de glances:

Glances nous permet d'avoir la température du gpu, l'état des disques,...

```bash
sudo nano /etc/caddy/Caddyfile
```


```caddy
monitor.mondomaine.com {
    encode gzip
    reverse_proxy 127.0.0.1:61208
}

```
Ajouter monitor sur cloudflare comme pour Kuma


On va créer un fichier pour que ca se lance automatiquement au démarrage:

```bash
sudo nano /etc/systemd/system/glances.service
```

On écrit:

```ini
[Unit]
Description=Glances
After=network.target

[Service]
ExecStart=/usr/local/bin/glances -w
Restart=always

[Install]
WantedBy=multi-user.target
```

Puis:

```bash
sudo systemctl daemon-reexec
sudo systemctl daemon-reload
sudo systemctl enable --now glances
```







ET VOIR SI JE NE PEUX PAS INTEGRER STATUS ET MONITOR DIRECTEMENT SUR NEXTCLOUD

## 12. Ajout d'un "bouton" shutdown disponible sur kuma:

L'idée est d'ajouter une notification qui va nous permetre de shutdown le serveur a distance depuis notre téléphone par exemple.

On va installer fail2ban, pip (s'il n'est pas déjà installé) et flask:

```bash
sudo dnf install fail2ban -y
sudo systemctl enable --now fail2ban
sudo dnf install python3-pip -y
pip3 install flask
```

### 1. On va créer un script shell pour éteindre le serveur:

```bash
sudo nano /usr/local/bin/shutdown-server.sh
```

Et on écrit:

```bash
#!/bin/bash
/sbin/shutdown now
```
On le rend éxécutable:

```bash
sudo chmod +x /usr/local/bin/shutdown-server.sh
```

### 2. Création d'un petit serveur web local avec flask:

```bash
sudo nano /opt/shutdown_api.py
```

On écrit:

```python
from flask import Flask, request, jsonify
import os

app = Flask(__name__)


SECRET_TOKEN = "abc123"  # remplace par un vrai mot de passe sécurisé

@app.route('/shutdown', methods=['POST'])
def shutdown():
    data = request.get_json()
    if not data or 'token' not in data:
        return jsonify({"error": "Token manquant"}), 400

    if data['token'] != SECRET_TOKEN:
        return jsonify({"error": "Token invalide"}), 403

    os.system("shutdown now")
    return jsonify({"message": "Shutdown command sent"}), 200

if __name__ == '__main__':
    app.run(host='0.0.0.0', port=5005)

```

ATTENTION: Il faut remplacer "ton_super_secret" par un token. Comment? avec la commande: `openssl rand -hex 32`

### 3. Création d'un service systemd pour l'API flask:

On va créer un service nommé shutdown-api.service:

```bash
sudo nano /etc/systemd/system/shutdown-api.service
```

On y met:
```ini
[Unit]
Description=Shutdown API Flask service
After=network.target

[Service]
ExecStart=/usr/bin/python3 /opt/shutdown_api.py
Restart=on-failure
User=root
WorkingDirectory=/opt

[Install]
WantedBy=multi-user.target
```

On va ensuite activer puis démarrer le service:

```bash
sudo systemctl daemon-reexec
sudo systemctl daemon-reload
sudo systemctl enable --now shutdown-api.service
```

On peut vérifier qu'il tourne bien:

```bash
sudo systemctl status shutdown-api.service
```

### 4. Modification du fichier config.php:

Si on va regarder sur les sous domaine, on a une erreur de nextcloud qui nous dit "Accès à partir d'un domaine non approuvé".

On va aller modifier:

```bash
sudo nano /var/www/nextcloud/config/config.php
```

Et on va ajouter:

```php
'trusted_domains' => 
array (
  0 => 'localhost',
  1 => '192.168.1.100',
  2 => 'davidscloud.live',
  3 => 'www.davidscloud.live',
  4 => 'status.davidscloud.live',
  4 => 'monitor.davidscloud.live',
),
```

Puis on redémarre:

```bash
sudo systemctl restart php-fpm
sudo systemctl reload caddy
```

ATTENTION:  S'il y a une erreur sur les sous domaine, ca se peut qu'il y a un problème avec le port 80:
On peut tester avec:

```bash
sudo systemctl stop httpd nginx
```

On accède au site et on voit si tout marche, si oui alors on peut faire `sudo systemctl disable httpd nginx`


### 5. Ajout de la notification sur Kuma:

Aller sur la sonde de votre site, clicker sur modifier, puis ajouter une notification:
* Nom: Shutdown
* Type de notification: webhook
* Post URL: http://192.168.129.48:5005/shutdown
* Coprs de la requete: Application/json:
```json
{
  "token": "abc123"
}
```
* En tête supplémentaire:
```json
{
  "Content-Type": "application/json"
}
```

Et normalement si on appuies sur tester, ça devrait shutdown le serveur

REGARDER DANS LE FICHIER CADDY DANS STATUS SI J'AI CHANGÉ LE FICHIER, REGARDER SI DANS LE FICHIER PYTHON C'EST 5005 OU 5050 ET AJOUTER DANS MD sudo firewall-cmd --add-port=5050/tcp --permanent