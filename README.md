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

## Ajouter un équipement

**SNMP** est un protocole réseau qui permet le **monitoring** et la **supervision** d'éléments systèmes et réseau. Généralement, un serveur de supervision utilise SNMP pour connaitre rapidement l'état du parc informatique (switch, routeur, serveurs) comme l'occupation RAM, CPU, Disque, etc.

### Switch et routeur

Dans le cas des switchs et des routeurs le snmp est à activé dans les configuration. Une fois configurer, ajouter les via l'interface Web de LibreNMS.

![Screenshot_1](https://raw.githubusercontent.com/KyoshinSan/LibreNMS/master/Doc%20librenms/Screenshot_1.png)

### CentOS 7

Dans cette partie, nous allons voir comment installer SNMP sous CentOS 7, notre machine avec LibreNMS pourra alors interrogé la machine en SNMP.

### Installation de SNMPD

Il faut bien entendu commencer par installer le service SNMP sur notre machine Linux. Le service se mettra alors en écoute sur le port 161 en UDP afin d'être prêt à répondre aux sollicitations extérieures.

```
yum install net-snmp net-snmpd-utils
```

Net-snmp va alors s'installer, ainsi que ses dépendances.

Il faudra également activer le démarrage du service SNMP au démarrage du serveur avec la commande suivante :

```
systemctl enable snmpd
```

### Mise en route

Démarrer le service :

```
systemctl start snmpd
```

Enfin, une fois celui-ci démarré, nous verrons qu'il est actif sur le port UDP 161, cela avec la commande **`ss`** qui permet de lister les ports en écoute :

```
ss -ulnp
```

Pour détailler cette commande :

-   l'option **`u`** permet de lister les ports UDP uniquement
-   l'option **`l`** permet de lister les ports en écoute (**Listening**)
-   l'option **`n`** permet d'afficher les numéros de ports et non leur correspondance (exemple : afficher **22** plutôt que **SSH**)
-   l'option **`p`** permet d'afficher les processus correspondants aux ports en écoute

Dans tous les cas, nous devrions avoir quelque chose qui ressemble à cela :

```
UNCONN     0      0                                   *:161                                             *:*                   
users:(("snmpd",pid=21551,fd=6))
```

### Configuration

Nous allons maintenant effectuer quelques configurations basiques, notre service SNMP écoute maintenant les sollicitations extérieures, mais il reste quelques changements à faire dans la configuration pour qu'il soit pleinement opérationnel.

Dans le fichier **/etc/snmp/snmpd.conf** à la ligne 41, nous allons changer la "**community**" . La community est une sorte de "**mot de passe**" qui va permettre de restreindre l'accès aux informations fournis par les serveurs SNMP. Par défaut, cette **community** est généralement "**public**" , c'est pourquoi il est recommandé de changer, autrement, n'importe qui dans votre réseau pourra alors questionner vos serveurs sur leurs états de santé :

```
# First, map the community name "public" into a "security name"

#       sec.name  source          community
com2sec notConfigUser  default       test
```

Dans le bloc ci-dessus, j'ai remplacé "**public**" , par "**test**" pour vous donner un exemple.

Il est également nécessaire de corriger les lignes 55 et 56 pour qu'elles ressemblent à cela :

```
view   systemview   included   .1.3.6.1.2.1
view   systemview   included   .1.3.6.1.2.1.25.1
```

Enfin, afin d'ouvrir certaines informations et de permettre qu'elles soient transmises via SNMP, nous allons décommenter les lignes suivantes :

-   lignes 85
-   lignes 122 à 147
-   lignes 151

Aux lignes 162 et 163, il peut être utile de mettre les valeurs propres à votre serveur, par exemple :

```
# It is also possible to set the sysContact and sysLocation system
# variables through the snmpd.conf file:
syslocation Serveur Test 01 
syscontact root@devnull.fr
```

Une fois tout cela fait, on pourra redémarrer SNMP pour recharger la configuration modifiée ("**systemctl restart snmpd**" ), puis passer à la phase de test de notre service SNMP.

### Test de communication

Nous allons maintenant effectuer quelques tests pour vérifier que notre service SNMP est fonctionnel. Étant donné que nous avons installé la partie "**client**" de SNMP, on peut tester cela avec la machine LibreNMS, en utilisant la commande "**snmpwalk**" :

```
snmpwalk -v1 ip-de-la-machine -c test
```
> Ajouter l'**ip** de la machine **cliente** ainsi que son **community**
 
 On devrait avoir cela en **output** :
```
...

SNMPv2-MIB::sysORLastChange.0 = Timeticks: (5) 0:00:00.05
SNMPv2-MIB::sysORID.1 = OID: SNMP-MPD-MIB::snmpMPDCompliance
SNMPv2-MIB::sysORID.2 = OID: SNMP-USER-BASED-SM-MIB::usmMIBCompliance
SNMPv2-MIB::sysORID.3 = OID: SNMP-FRAMEWORK-MIB::snmpFrameworkMIBCompliance
SNMPv2-MIB::sysORID.4 = OID: SNMPv2-MIB::snmpMIB
SNMPv2-MIB::sysORID.5 = OID: TCP-MIB::tcpMIB
SNMPv2-MIB::sysORID.6 = OID: IP-MIB::ip
SNMPv2-MIB::sysORID.7 = OID: UDP-MIB::udpMIB
SNMPv2-MIB::sysORID.8 = OID: SNMP-VIEW-BASED-ACM-MIB::vacmBasicGroup
SNMPv2-MIB::sysORID.9 = OID: SNMP-NOTIFICATION-MIB::snmpNotifyFullCompliance
SNMPv2-MIB::sysORID.10 = OID: NOTIFICATION-LOG-MIB::notificationLogMIB
SNMPv2-MIB::sysORDescr.1 = STRING: The MIB for Message Processing and Dispatching.
SNMPv2-MIB::sysORDescr.2 = STRING: The management information definitions for the SNMP User-based Security Model.
SNMPv2-MIB::sysORDescr.3 = STRING: The SNMP Management Architecture MIB.
SNMPv2-MIB::sysORDescr.4 = STRING: The MIB module for SNMPv2 entities
SNMPv2-MIB::sysORDescr.5 = STRING: The MIB module for managing TCP implementations
SNMPv2-MIB::sysORDescr.6 = STRING: The MIB module for managing IP and ICMP implementations
SNMPv2-MIB::sysORDescr.7 = STRING: The MIB module for managing UDP implementations
SNMPv2-MIB::sysORDescr.8 = STRING: View-based Access Control Model for SNMP.
SNMPv2-MIB::sysORDescr.9 = STRING: The MIB modules for managing SNMP Notification, plus filtering.
SNMPv2-MIB::sysORDescr.10 = STRING: The MIB module for logging SNMP Notifications.
SNMPv2-MIB::sysORUpTime.1 = Timeticks: (4) 0:00:00.04
SNMPv2-MIB::sysORUpTime.2 = Timeticks: (4) 0:00:00.04
SNMPv2-MIB::sysORUpTime.3 = Timeticks: (4) 0:00:00.04
SNMPv2-MIB::sysORUpTime.4 = Timeticks: (4) 0:00:00.04
SNMPv2-MIB::sysORUpTime.5 = Timeticks: (5) 0:00:00.05
SNMPv2-MIB::sysORUpTime.6 = Timeticks: (5) 0:00:00.05
SNMPv2-MIB::sysORUpTime.7 = Timeticks: (5) 0:00:00.05
SNMPv2-MIB::sysORUpTime.8 = Timeticks: (5) 0:00:00.05
SNMPv2-MIB::sysORUpTime.9 = Timeticks: (5) 0:00:00.05
SNMPv2-MIB::sysORUpTime.10 = Timeticks: (5) 0:00:00.05

...
```

Vous pouvez maintenant l'ajouter à LibreNMS.
