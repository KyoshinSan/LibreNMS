# Créer des alertes par mail sur LibreNMS

Tout d'abord, il est important de savoir que les alertes de LibreNMS ce compose de 3 choses :

- Pour commencer, vous avez d’abord besoin de **règles d’alerte** qui réagissent aux modifications apportées à vos appareils avant de déclencher une alerte.
- Après cela, vous devez également indiquer à LibreNMS comment vous avertir lorsqu'une alerte est déclenchée, ceci est effectué à l'aide du **transport d'alertes**.
- La dernière étape n’est pas strictement requise, mais la plupart des gens la trouvent utile. La création de **modèles d'alerte** personnalisés vous aidera à tirer parti du système d'alerte en général. Bien que nous incluions un modèle par défaut, il est limité dans les données que vous recevrez dans les alertes.

## Transport d'alertes

Dans un premier temps, nous allons configurer le transporteur d'alertes. LibreNMS propose beaucoup de [solutions](https://docs.librenms.org/Alerting/Transports/) dans ce domaine, mais nous allons utilisé les mails. Pour ce faire il faut configurer un **serveur de mail** (postfix,...), puis modifier ce fichier dans LibreNMS :

```
vi /opt/librenms/vendor/phpmailer/phpmailer/src/PHPMailer.php

public $From = 'librenms@exemple.fr';
public $FromName = 'LibreNMS';
```

Ensuite rendez vous sur l'interface web de LibreNMS :

![Sceenshot_2](https://raw.githubusercontent.com/KyoshinSan/LibreNMS/master/Doc%20librenms/Screenshot_2.png)

Compléter les informations suivantes et enregistrer :

![Sceenshot_3](https://raw.githubusercontent.com/KyoshinSan/LibreNMS/master/Doc%20librenms/Screenshot_3.png)
