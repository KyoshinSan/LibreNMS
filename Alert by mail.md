# Créer des alertes par mail sur LibreNMS

Tout d'abord, il est important de savoir que les alertes de LibreNMS ce compose de 3 choses :

- Pour commencer, vous avez d’abord besoin de **règles d’alerte** qui réagissent aux modifications apportées à vos appareils avant de déclencher une alerte.
- Après cela, vous devez également indiquer à LibreNMS comment vous avertir lorsqu'une alerte est déclenchée, ceci est effectué à l'aide du **transport d'alertes**.
- La dernière étape n’est pas strictement requise, mais la plupart des gens la trouvent utile. La création de **modèles d'alerte** personnalisés vous aidera à tirer parti du système d'alerte en général. Bien que nous incluions un modèle par défaut, il est limité dans les données que vous recevrez dans les alertes.

## Transport d'alertes

Dans un premier temps, nous allons configurer le transporteur d'alertes. LibreNMS propose beaucoup de [solutions](https://docs.librenms.org/Alerting/Transports/) dans ce domaine, mais nous allons utiliser les mails. Pour ce faire il faut configurer un **serveur de mail** (postfix,...), puis modifier ce fichier dans LibreNMS :

```
vi /opt/librenms/vendor/phpmailer/phpmailer/src/PHPMailer.php

public $From = 'librenms@exemple.fr';
public $FromName = 'LibreNMS';
```

Ensuite rendez-vous sur l'interface web de LibreNMS :

![Screenshot_2](https://raw.githubusercontent.com/KyoshinSan/LibreNMS/master/Doc%20librenms/Screenshot_2.png)

Compléter les informations suivantes et enregistrer :

![Screenshot_3](https://raw.githubusercontent.com/KyoshinSan/LibreNMS/master/Doc%20librenms/Screenshot_3.png)

Vous pouvez tester si le mail marche bien :

![Screenshot_4](https://raw.githubusercontent.com/KyoshinSan/LibreNMS/master/Doc%20librenms/Screenshot_4.png)

## Règles d'alertes

Maintenant, nous allons configurer les règles d'alertes. Rendez-vous sur l'interface web de LibreNMS et créer une alerte :

![Screenshot_5](https://raw.githubusercontent.com/KyoshinSan/LibreNMS/master/Doc%20librenms/Screenshot_5.png)

Compléter les champs suivants. Pour les règles, utiliser les [**`Entités`**](https://docs.librenms.org/Alerting/Entities/) de la documentation de LibreNMS en fonction vos besoins.
> Ici, on configure une règle pour savoir si la machine répond au ping

![Screenshot_6](https://raw.githubusercontent.com/KyoshinSan/LibreNMS/master/Doc%20librenms/Screenshot_6.png)


## Modèles d'alertes

Enfin, nous allons configurer les modèles d'alertes. Cela permet d'avoir des mails avec nos propres messages d'alertes. Sur l'interface web crée un nouveau modèle :

![Screenshot_7](https://raw.githubusercontent.com/KyoshinSan/LibreNMS/master/Doc%20librenms/Screenshot_7.png)

Compléter en fonction de vos [besoins](https://docs.librenms.org/Alerting/Templates/) et **attacher le modèle à une règle** :

![Screenshot_8](https://raw.githubusercontent.com/KyoshinSan/LibreNMS/master/Doc%20librenms/Screenshot_8.png)

Enfin tester votre alerte !
