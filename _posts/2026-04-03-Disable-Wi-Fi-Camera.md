---
title: Comment neutraliser une caméra de surveillance ? 
date: 2026-04-03
categories: [IoT]
tags: [IoT, Home Automation, Radio Frequency]
---

# Comment neutraliser une caméra de surveillance ? 

## Context

Vous avez acheté une caméra de surveillance Wi-Fi afin de sécuriser votre domicile. Grâce à cela, vous pouvez vérifier si votre chat va bien Si quelqu'un rentre chez vous, il sera filmé et grâce à l'ère l'intelligence artificielle, vous pouvez même recevoir une notification sur votre smartphone qui vous dira qu'un humain vient d'être repéré dans votre couloir. 

Vous vous sentez en sécurité. Mais est-il possible de neutraliser cette caméra sans avoir besoin de rentrer dans votre réseau Wi-Fi ? Dans le but de s'introduire chez vous, puis la réactiver sans laisser de trace ? Êtes-vous certain que votre caméra Wi-Fi assurera sa fonction quoi qu'il arrive ? 

Dans cet article, nous verrons comment repérer passivement une caméra de surveillance Wi-Fi pour particulier, puis comment la neutraliser sans avoir besoin de se connecter au réseau Wi-Fi. 

Cette attaque fonctionne relativement bien sur les caméras de surveillance pour particulier d'entrée de gamme car très souvent connectées au réseau uniquement en Wi-Fi et rarement compatible PMF

Avant d'aller plus loin un petit rappel légal:

S'introduire dans un système informatique de traitement de données automatisé sans l'autorisation du propriétaire est illégal.

![alt text](/assets/img/posts/neutraliser-camera/article_321-3.png){: width="950" .center}

Ceci s'applique quel que soit le mobile et quel que soit le système. Les démonstrations présentées ici ont été réalisées sur du matériel dont je suis le propriétaire. 

Ne reproduisez pas ces opérations sans une autorisation écrite

## Repérer une caméra Wi-Fi

Imaginons le scénario suivant : vous devez savoir si une caméra de surveillance est active sur un réseau Wi-Fi dont vous ne connaissez que le SSID, pas le mot de passe. Pas de connexion possible, donc pas de scan de ports.

Ce n'est pas un problème : nous pouvons sniffer passivement les trames Wi-Fi.

**Pourquoi c'est possible**

Dans le standard 802.11, les points d'accès diffusent périodiquement (plusieurs fois par secondes) des beacon frames en broadcast. Ces trames contiennent le SSID, le BSSID (adresse MAC de l'AP), le canal et certains paramètres réseau. Elles sont transmises en clair, indépendamment du chiffrement des données (WPA2, WPA3).

De la même manière, les en-têtes des trames de données 802.11 contiennent les adresses MAC source, destination et le BSSID, en clair. Un adaptateur Wi-Fi en mode monitor comme fameuse carte Alpha peut donc observer toutes les stations associées à un point d'accès sans connaître la clé du réseau.

**Identifier le point d'accès**

Dans un premier temps, nous avons besoin de lister les points d'accès Wi-Fi autour de nous afin d'identifier le canal et le BSSID du réseau Wi-Fi cible

```
sudo airmon-ng start wlan0
sudo airodump-ng wlan0mon
```

![alt text](/assets/img/posts/neutraliser-camera/list-AP.png){: width="950" .center}

Le réseau cible apparaît dans la liste. Récupérons le BSSID et le canal de communication 

**Lister les stations associées**

Nous pouvons maintenant lister les équipements connectés à ce point d'accès :

```
sudo airodump-ng --bssid <BSSID_CIBLE> --channel <CH> wlan0mon
```

![alt text](/assets/img/posts/neutraliser-camera/station-connected.png){: width="950" .center}

Cela permet d'identifier deux stations

La première a pour adresse MAC : 1C:4D:89:XX:XX:XX
La seconde : 3A:3B:7D:XX:XX:XX

**Identifier laquelle est la caméra**

Les 24 premiers bits d'une adresse MAC constituent l'OUI (Organizationally Unique Identifier), Une recherche dans la base IEEE permet de remonter au fabricant.

Avant cela, il faut vérifier que l'adresse n'est pas randomisée.
Si le deuxième bit de poids faible du premier octet est à 0 alors l'adresse MAC n'est pas randomisée et il est utile de faire une recherche dans les bases IEEE

Dans notre cas 

`1C:4D:89:XX:XX:XX` — `0x1C` = `00011100`, le bit de randomisation = 0. Donc l'adresse MAC n'est pas randominisé

`3A:3B:7D:XX:XX:XX` — `0x3A` = `00111010`. L'adresse MAC est randominisée.

Maintenant qu'on sait que l'adresse MAC n'est pas randomisée. On peut trouver des informations sur son fabricant

https://macaddresslookup.io/fr?search=1C-4D-89

![alt text](/assets/img/posts/neutraliser-camera/adress-lockup.png){: width="950" .center}

Si l'OUI renvoyait vers un fabricant généraliste comme TP-Link (Tapo), qui commercialise aussi des prises connectées et des routeurs, le constructeur seul ne suffirait pas à conclure. 

Afin de nous assurer qu'il s'agisse bien d'une caméra, nous allons analyser la fréquence des trames envoyée et reçue par l'équipement 

**Analyse du volume de trames**

![alt text](/assets/img/posts/neutraliser-camera/statistiques-network.png){: width="950" .center}

Une caméra de surveillance qui envoie en continu un flux de vidéo enverra un nombre très important de paquets. En revanche un aspirateur robot ou une ampoule connectée enverra ou recevra beaucoup moins de paquets.


Vu la fréquence des trames observées sur la station `1C:4D:89:XX:XX:XX`, nous pouvons être quasi certains qu'il s'agit d'une caméra.

## Neutraliser la caméra

Il est maintenant temps d'empêcher la caméra de transmettre son flux. Pour cela, nous allons injecter en boucle des trames de désauthentification 802.11.

**Pourquoi ça fonctionne**

Dans le standard 802.11, les trames de désauthentification sont des notifications: toute station qui en reçoit une doit s'y conformer et se déconnecter. Ces trames de gestion ne sont ni chiffrées ni authentifiées. N'importe qui peut donc en envoyer en direction d'un réseau Wi-Fi sans y être connecté. 

En revanche si le PMF (Protected Management Frames) est activé sur le routeur. Cette attaque ne fonctionnera pas. L'avantage dans notre cas, la plupart des réseaux domestiques n'ont pas ce paramètre activé. De plus il existe peu de caméras grand publique compatible avec ce paramètre. 

La caméra maintient une connexion persistante (WebSocket ou MQTT) vers le serveur cloud du constructeur. C'est par ce canal que transitent le flux vidéo en direct, les notifications de détection et les commandes de contrôle (pan/tilt, activation, etc.). Le smartphone se connecte au même cloud via l'application mobile.
Quand la caméra est désauthentifiée, elle perd sa connexion Wi-Fi au routeur. Toute la chaîne est rompue :

- Le flux vidéo n'est plus transmis au cloud, donc plus visible dans l'application.
- Les notifications (détection de personne, mouvement) ne peuvent plus être envoyées.
- La caméra n'est plus contrôlable à distance.
- Même si la caméra détecte localement un intrus, elle ne peut pas transmettre l'alerte.

Allons-y déconnectons la caméra du réseau et empêchons la de s'y reconnecter

```
sudo aireplay-ng --deauth 0 -a <BSSID_AP> -c <MAC_CAMERA> wlan0mon
```

Le paramètre `--deauth 0` envoie des trames en boucle infinie. La caméra se déconnecte du réseau et ne parvient pas à se réassocier tant que l'injection est maintenue.

<video controls width="100%">
  <source src="/assets/vid/neutraliser-camera/neutraliser-camera.mp4" type="video/mp4">
</video>

J'ai volontairement flouté la partie gauche de la vidéo. Celle qui correspond à mon PC. Pour une raison simple, la commande leak des informations sur mon réseau Wi-Fi (BSSID et SSID) qui pourrait permettre avec une API comme Wigle de retrouver mon adresse. Le caviardage sur cette vidéo n'est pas efficace donc je préfère utiliser la manière forte, tout flouter. 

A droite, vous avez la vue vers l'application mobile de la caméra. Avant l'attaque tout va bien, la caméra envoie son flux, nous pouvons la contrôler depuis le smartphone.

Dès que je déclenche l'attaque, la vue caméra se fige, on voit d'ailleurs l'horloge de la caméra se figer car elle n'envoie plus les flux vidéo. 

Une fois l'injection arrêtée, la caméra se réassocie automatiquement au point d'accès et le fonctionnement reprend comme si de rien n'était

## Mitigations

**802.11w — Protected Management Frames (PMF)**

L'amendement IEEE 802.11w introduit la protection des trames de gestion. Quand cette protection est négociée entre l'AP et le client, les trames de désauthentification spoofées sont rejetées. Ce paramètre s'active sur un point en WPA2, sur le WPA3, ce paramètre est activé par défaut. 


Beaucoup de box pour particulier proposent ce paramètre mais la plupart des caméras d'entrée de gamme pour particulier ne sont pas compatibles.

Pour activer la protection, il faut que le PMF soit activé sur la caméra et le routeur. 

**Enregistrement local sur carte SD**

Si l'enregistrement local est activé (carte micro-SD), le flux vidéo continue d'être stocké même sans connectivité Wi-Fi. L'intrus sera filmé, mais le propriétaire ne recevra aucune alerte en temps réel. 

**Connexion Ethernet / PoE**

L'attaque par désauthentification est spécifique au protocole 802.11. Une caméra câblée en Ethernet y est totalement immunisée. Des attaques sont possibles pour bloquer le flux d'une caméra connectée en Ethernet mais elles demandent d'être dans le réseau. 


**Détection des trames de désauthentification**

Un WIDS (Wireless Intrusion Detection System) peut monitorer les trames de gestion et alerter en cas de flood de désauthentification. La détection n'empêche pas l'attaque mais permet de prévenir le propriétaire que quelque d'anormal se passe. 



