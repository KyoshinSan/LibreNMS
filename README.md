# LibreNMS Centos 7
**LibreNMS** est un projet démarré en 2006 par Adam Armstrong puis ouvert à la communauté en 2013. Cet outil de supervision vous permet de superviser quasiment tout ce qui est possible et imaginable sur un réseau.  
Le développement avance relativement vite et est de plus en plus apprécié par la communauté.

**LibreNMS** est un système de supervision réseaux qui possèdent :

-   Une découverte automatique du réseau en utilisant CDP, FDP, LLDP, OSPF, BGP, SNMP et ARP.
-   Un système d’alerte flexible
-   Une interface très complète pour gérer, représenter et extraire les données (graphiques, tableaux …)
-   Des mises à jour automatique pour corriger des bugs ou ajouter de nouvelles fonctionnalités

Et de nombreuses autre possibilités.

LibreNMS est principalement codé en JavaScript et PHP, il est donc assez gourmand en ressources…

## Installation de LibreNMS

> REMARQUES : ces instructions supposent que vous êtes l'utilisateur **root**. Si vous ne l'êtes pas, ajoutez sudo aux commandes du shell (celles qui ne sont pas sous `mysql>`) ou devenez temporairement un utilisateur avec les privilèges root avec sudo -s ou sudo -i.

**Veuillez noter que la version PHP minimum prise en charge est 7.1.3**

### Installer les paquets requis

```
yum install epel-release
rpm -Uvh https://mirror.webtatic.com/yum/el7/webtatic-release.rpm
yum install composer cronie fping git httpd ImageMagick jwhois mariadb mariadb-server mtr MySQL-python net-snmp net-snmp-utils nmap php72w php72w-cli php72w-common php72w-curl php72w-gd php72w-mbstring php72w-mysqlnd php72w-process php72w-snmp php72w-xml php72w-zip python-memcached rrdtool
```

### Ajouter l'utilisateur LibreNMS

```
useradd librenms -d /opt/librenms -M -r
usermod -a -G librenms apache
```

### Installer LibreNMS

```
cd /opt
composer create-project --no-dev --keep-vcs librenms/librenms librenms dev-master
```

## Configuration de la base de données

Configurer MySQL :
```
systemctl start mariadb
systemctl enable mariadb
mysql -u root
```
Changer le champ **`'password'`**
```
CREATE DATABASE librenms CHARACTER SET utf8 COLLATE utf8_unicode_ci;
CREATE USER 'librenms'@'localhost' IDENTIFIED BY 'password';
GRANT ALL PRIVILEGES ON librenms.* TO 'librenms'@'localhost';
FLUSH PRIVILEGES;
exit
```
Editer le fichier **/etc/my.cnf** et ajouter dans la section `[mysqld]` :

```
vi /etc/my.cnf
innodb_file_per_table=1
lower_case_table_names=0
```

Relancer mysql/mariadb

```
systemctl enable mariadb
systemctl restart mariadb
```

## Configuration de Apache, PHP et SELinux

### PHP

Mettre une valeur au **timezone** php

```
vi /etc/php.ini
date.timezone = Europe/Paris
```

### Apache

Créez le le fichier librenms.conf :

```
vi /etc/httpd/conf.d/librenms.conf
```

Ajoutez la configuration suivante, éditez `ServerName` comme requis :

```
<VirtualHost *:80>
  DocumentRoot /opt/librenms/html/

  ServerName  librenms.example.com

  AllowEncodedSlashes NoDecode
  <Directory "/opt/librenms/html/">
    Require all granted
    AllowOverride All
    Options FollowSymLinks MultiViews
  </Directory>
</VirtualHost>
```

Relancer Apache

```
systemctl enable httpd
systemctl restart httpd
```

### SELinux

Installez l'outil de gestion de stratégie pour SELinux :

```
yum install policycoreutils-python
```

Configurez les contextes requis par LibreNMS :

```
semanage fcontext -a -t httpd_sys_content_t '/opt/librenms/logs(/.*)?'
semanage fcontext -a -t httpd_sys_rw_content_t '/opt/librenms/logs(/.*)?'
restorecon -RFvv /opt/librenms/logs/
semanage fcontext -a -t httpd_sys_content_t '/opt/librenms/rrd(/.*)?'
semanage fcontext -a -t httpd_sys_rw_content_t '/opt/librenms/rrd(/.*)?'
restorecon -RFvv /opt/librenms/rrd/
semanage fcontext -a -t httpd_sys_content_t '/opt/librenms/storage(/.*)?'
semanage fcontext -a -t httpd_sys_rw_content_t '/opt/librenms/storage(/.*)?'
restorecon -RFvv /opt/librenms/storage/
semanage fcontext -a -t httpd_sys_content_t '/opt/librenms/bootstrap/cache(/.*)?'
semanage fcontext -a -t httpd_sys_rw_content_t '/opt/librenms/bootstrap/cache(/.*)?'
restorecon -RFvv /opt/librenms/bootstrap/cache/
setsebool -P httpd_can_sendmail=1
```

Crée un fichier http_fping.tt

```
vi /root/http_fping.tt
```

Et coller les instructions suivantes :

```
module http_fping 1.0; 

require { 
type httpd_t;
class capability net_raw;
class rawip_socket { getopt create setopt write read };
}

#============= httpd_t ==============
allow httpd_t self:capability net_raw;
allow httpd_t self:rawip_socket { getopt create setopt write read };
```

Ensuite exécuter ces commandes :

```
checkmodule -M -m -o http_fping.mod /root/http_fping.tt
semodule_package -o http_fping.pp -m http_fping.mod
semodule -i http_fping.pp
```

### FirewallD

Autoriser l'accès à travers le firewall :

```
firewall-cmd --zone=public --add-service=http
firewall-cmd --permanent --zone=public --add-service=http
firewall-cmd --zone=public --add-service=https
firewall-cmd --permanent --zone=public --add-service=https
```

### Configurer le snmpd

Copiez l'exemple snmpd.conf à partir de l'installation LibreNMS.

```
cp /opt/librenms/snmpd.conf.example /etc/snmp/snmpd.conf
vi /etc/snmp/snmpd.conf
```

Éditez le texte qui dit `RANDOMSTRINGGOESHERE` et définissez votre propre communauté. Puis exécuter ces commandes :

```
curl -o /usr/bin/distro https://raw.githubusercontent.com/librenms/librenms-agent/master/snmp/distro
chmod +x /usr/bin/distro
systemctl enable snmpd
systemctl restart snmpd
```

### Cron job

```
cp /opt/librenms/librenms.nonroot.cron /etc/cron.d/librenms
```

> REMARQUE: Gardez à l'esprit que cron, par défaut, n'utilise qu'un ensemble très limité de variables d'environnement. Vous devrez peut-être configurer des variables de proxy pour l'appel cron. Il est également possible d’ajouter les paramètres proxy dans config.php. Le fichier config.php sera créé lors des prochaines étapes. Regardez l'URL suivante après avoir terminé les étapes d'installation de librenms: https://docs.librenms.org/Support/Configuration/#proxy-support

### Copier la configuration de logrotate

LibreNMS conserve les journaux dans **/opt/librenms/logs**. Avec le temps, ils peuvent devenir volumineux et faire l’objet d’une rotation. Pour faire changer les anciens journaux, vous pouvez utiliser le fichier de configuration logrotate fourni:

```
cp /opt/librenms/misc/librenms.logrotate /etc/logrotate.d/librenms
```

## Installation Web

Maintenant, dirigez-vous vers l'installateur Web et suivez les instructions à l'écran.

```
http://librenms.example.com/install.php
```
> Remplacer `librenms.example.com` par l'ip de la machine

Le l'installateur Web peut vous demander de créer manuellement un fichier `config.php` à l’emplacement de votre installation **LibreNMS**, en copiant le contenu affiché à l’écran dans le fichier. Si vous devez le faire, n'oubliez pas de définir les autorisations sur `config.php` après avoir copié le contenu à l'écran dans le fichier. Exécuter :

```
vi /opt/librenms/config.php

chown librenms:librenms /opt/librenms/config.php
```

### Dépannage

Si vous rencontrez des problèmes avec votre installation, exécutez **validate.php** en tant que **root** dans le répertoire de LibreNMS :

```
cd /opt/librenms
./validate.php
```
