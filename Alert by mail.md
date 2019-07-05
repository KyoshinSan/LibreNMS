# Créer des alertes par mail sur LibreNMS

Tout d'abord, il est important de savoir que les alertes de LibreNMS ce compose de 3 choses :

- Pour commencer, vous avez d’abord besoin de règles d’alerte qui réagissent aux modifications apportées à vos appareils avant de déclencher une alerte.
- Après cela, vous devez également indiquer à LibreNMS comment vous avertir lorsqu'une alerte est déclenchée, ceci est effectué à l'aide du `transport d'alertes`.
- La dernière étape n’est pas strictement requise, mais la plupart des gens la trouvent utile. La création de modèles d'alerte personnalisés vous aidera à tirer parti du système d'alerte en général. Bien que nous incluions un modèle par défaut, il est limité dans les données que vous recevrez dans les alertes.


![Sceenshot_2](https://raw.githubusercontent.com/KyoshinSan/LibreNMS/master/Doc%20librenms/Screenshot_2.png)
