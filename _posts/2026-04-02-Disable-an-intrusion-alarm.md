---
title: Comment neutraliser une alarme anti-intrusion ? 
date: 2026-04-02
categories: [IoT]
tags: [IoT, Home Automation, Radio Frequency]
---

# Comment neutraliser une alarme anti-intrusion ? 

## Introduction

Cet article cible les alarmes anti-intrusion sans fil pour particuliers, de l'entrée de gamme à la moyenne gamme, qui communiquent sur la bande `ISM 433 MHz`.

Les deux attaques présentées (replay et jamming) fonctionnent sur la majorité de ces systèmes. Les alarmes haut de gamme intègrent des contre-mesures (rolling code, saut de fréquence, détection de brouillage) qui rendent ces attaques plus difficiles, sans pour autant les rendre impossibles.

## Cadre légal

Avant d'aller plus loin, un petit rappel légal :

S'introduire dans un système informatique de traitement de données automatisé sans l'autorisation du propriétaire est illégal.

![alt text](/assets/img/posts/neutraliser-alarme/article_321-3.png){: width="950" .center}

Ceci s'applique quel que soit le mobile et quel que soit le système. Les démonstrations présentées ici ont été réalisées sur du matériel dont je suis le propriétaire.

Ne reproduisez pas ces opérations sans une autorisation écrite du propriétaire.

### Fonctionnement d'une alarme anti-intrusion

La quasi-totalité des alarmes sans fil pour particuliers reposent sur le même modèle :

![alt text](/assets/img/posts/neutraliser-alarme/schema-alarme-fonctionnement.png){: width="950" .center}

Le principe est simple. Lorsqu'un capteur de présence détecte un mouvement ou qu'un capteur d'ouverture détecte une porte ou fenêtre ouverte, un signal radio est envoyé à la passerelle (centrale).

- Si l'alarme est armée, elle se déclenche.
- Si elle est désarmée, la centrale reçoit le signal mais ne réagit pas.

La télécommande ou le clavier déporté envoie un signal radio à la centrale pour l'armer ou la désarmer. Ce signal n'est pas toujours chiffré, et c'est ce qui va nous intéresser dans la suite de cet article.

Les technologies de communication entre périphériques et centrale varient :
- RF 433 MHz
- RF 868 MHz
- Zigbee
- Bluetooth

Nous nous concentrons ici sur le `433 MHz`, la technologie la plus répandue dans le segment grand public et celle qui présente le plus de vulnérabilités.

## Repérer une alarme anti-intrusion

Il est relativement simple de repérer une alarme `433 MHz`. Il suffit de placer un récepteur radio à proximité du domicile et d'observer le spectre pendant un à deux jours. Si une alarme est présente, de nombreux signaux apparaîtront aux moments où les habitants sont actifs.

### Implant de surveillance
La bande `433 MHz` ne nécessite que peu de matériel pour être analysée. Deux options :

- SDR + Raspberry Pi
- Flipper Zero

![alt text](/assets/img/posts/neutraliser-alarme/implants.png){: width="950" .center}

**SDR + Raspberry Pi**

Une `SDR` (Software Defined Radio) est un récepteur radio piloté par logiciel. Dans ce cas, cette `SDR` (`NESDR Smart`) fait uniquement récepteur. Elle coûte environ 50 euros. Si vous avez besoin d'un récepteur et d'un émetteur, le HackRF répondra à votre besoin mais coûtera un peu plus cher (600 euros).

Cet implant a une portée d'environ 10 à 15 mètres selon l'antenne, ce qui est souvent bien suffisant si vous positionnez l'implant dans un jardin à proximité de la maison.

Il faudra tout de même penser à relier l'implant à une batterie externe pour l'alimenter

Sur la raspberry pi. Il vous suffit juste d'installer rtl_433 disponible sur github ou peu même s'installer avec la commande `sudo apt install rtl-433`

Puis de lancer un cron au démarrage de la raspberry pi pour qui aura pour mission de lancer `rtl_433`

```
crontab -e
@reboot sleep 300 && /usr/bin/rtl_433 -f 433920000 -S all
```

A chaque nouvel signal détecté sur la fréquence `433,92`, rtl_433 le sauvegardera dans un nouveau fichier

![alt text](/assets/img/posts/neutraliser-alarme/saving-image-rtl_433.png){: width="950" .center}

Il faudra ensuite décoder les signaux. Pour cela, je vous envoi vers mon article sur le reverse de clef RF433.

![alt text](/assets/img/posts/neutraliser-alarme/urh-signal.png){: width="950" .center}

<a href="https://0xhsbf6.github.io/posts/reverse-RF433-keys/">Reverse de télécommande RF433</a>

**Flipper Zero**

Encore plus simple, utilisons le pingouin préféré des hackers (lol). C'est un petit couteau suisse qui dépanne bien. Par rapport au système précédent, il est plus compact et son autonomie est très bonne, sans avoir besoin d'une batterie externe.

Inconvénient : la portée est limitée à environ 5 mètres sur `433 MHz`.

**Analyse des résultats**

Après quelques jours d'analyse, nous récupérons notre implant afin d'analyser les signaux qu'il a interceptés.

Par exemple, si votre implant était un Flipper Zero, voici un exemple de trame que vous verrez :

<video controls width="400">
  <source src="/assets/vid/neutraliser-alarme/flipper-zero-implant.webm" type="video/mp4">
</video>

Nous pouvons voir tous les détails sur les signaux interceptés.

![alt text](/assets/img/posts/neutraliser-alarme/enable_alarm.png){: width="400" .center}

Attention avec le flipper zéro en utilisant le mode Read, le signal n'est pas directement utilisable pour une attaque par replay contrairement au signaux que nous avons capturé avec notre SDR. C'est pour cette raison que je préfère la SDR pour faire un implant. 

Heureseument, il est assez facile de créer un signal valide à partir des élements que nous avons récolté et l'article que j'ai écris sur le reverse de télécommande 433MHz. 

<a href="https://0xhsbf6.github.io/posts/reverse-RF433-keys/">Reverse de télécommande RF433</a>

D'une manière générale, la difficulté des implants réside dans le placement de celui-ci à proximité de l'alarme, des capteurs. Sans que celui-ci ne soit trouvé par votre target. Le choix de la cachette est laissé à votre imagination.

### Cartographie passive

Grâce à notre implant, nous pouvons cartographier l'ensemble des équipements radio du système d'alarme en analysant les signaux sur la durée.

Avec tout ces indicateurs, il est très souvent possible avec un peu de déduction de faire une cartographie passive du système d'alarme de notre target. 

**- Capteurs de présence**

Lorsque les habitants sont chez eux et ne sont pas en train de dormir, le signal du capteur de présence est émis fréquemment. En l'absence de rolling code, ce signal est identique à chaque émission et non chiffré.

**- Télécommande / clavier déporté**

Un autre signal apparaît seulement une à deux fois par jour, aux heures de départ et d'arrivée des habitants c'est le signal d'armement ou de désarmement de l'alarme.

Les signaux d'armement et de désarmement se ressemblent fortement, à un ou deux octets près. C'est un pattern caractéristique qui permet d'identifier un signal de télécommande ou de clavier déporté. Nous les décoderons dans partie suivante. 

Quand vous tapez votre code sur un clavier déporté, le clavier vérifie si le code est correct. Si c'est le cas, il envoie un signal de la même forme à la centrale. Quand vous activez votre alarme, le fonctionnement est identique. Un clavier déporté est donc une sorte de télécommande.

**- Capteurs de porte**

Le capteur de porte sera activé systématiquement juste avant ou après le signal de télécommande. On active par exemple l'alarme et on sort de la maison (armement d'alarme puis capteur de porte) ou inversement quand vous rentrez chez vous et que vous désarmer votre alarme sur votre clavier déporté (après avoir ouvert votre porte).

**- Capteurs de fenêtre**

Les capteurs de fenêtre sont plus aléatoires et peuvent ne pas apparaître du tout pendant la durée de la capture. Tout le monde n'ouvre pas ses fenêtres tous jours et encore moins à la même heure.

### Encodage des signaux

La plupart des alarmes entrée/moyenne gamme utilisent le même schéma d'encodage pour leurs signaux radio : les 20 premiers bits encodent le numéro de série de l'équipement, les 4 derniers bits identifient le bouton ou la fonction.
Exemple avec les signaux capturés de la télécommande

```
C2C411 : 1100 0010 1100 0100 0001 | 0001  → Bouton 1 (armer)
C2C412 : 1100 0010 1100 0100 0001 | 0010  → Bouton 2 (désarmer)
C2C414 : 1100 0010 1100 0100 0001 | 0100  → Bouton 3
C2C418 : 1100 0010 1100 0100 0001 | 1000  → Bouton 4
```
Même si nous n'avons pas intercepté de signal de désarmement, il est souvent facile de déduire le signal d'armement.

Les capteurs (porte, fenêtre, présence) n'ont pas de boutons. Le constructeur attribue vraisemblablement un identifiant complet sur 24 bits au capteur. C'est un code fixe unique par capteur, généré de manière plus ou moins aléatoire. Par exemple, le capteur de présence de l'alarme que j'ai testée envoie le code `B472AC`.

Attention, les stations météo pour particulier émettent souvent sur la même fréquence, mais elles sont très faciles à repérer :
- elles n'ont pas la même taille de trame
- elles émettent toutes les minutes, 24h/24

**Signal d'une station météo**

![alt text](/assets/img/posts/neutraliser-alarme/temperature_sensor.png){: width="400" .center}

Pour plus de détail, je vous renvoi vers mon article sur le reverse et l'analyse des communications radios sur la bande `ISF 433 MHz`. 

## Attaque 1: Replay 

### Le problème fondamental

Ces équipements envoient le même signal radio à chaque action. Bien souvent sur des modèles d'entrée ou milieu de gamme, il n'y a pas de chiffrement ni de rolling code. 

Le signal de désarmement émis par la télécommande est strictement identique d'une utilisation à l'autre. 

Puisque nous avons intercepté les signaux lors de la phase de cartographie, il suffit de les rejouer tels quels pour usurper la télécommande et désarmer l'alarme.

### Démonstration d'attaque par rejeu

Le HackRF est une radio SDR émetteur/récepteur (~350-600 €). Son avantage principal est sa portée : selon l'antenne utilisée, il peut recevoir et émettre sur plusieurs dizaines de mètres.

Vous avez placé un implant devant la maison de Toto et capturé tous les signaux radio pendant plusieurs jours. Comme expliqué précédemment, vous êtes en mesure de déterminer si le signal a été émis par un capteur ou par une télécommande.

Vous allez maintenant pouvoir rejouer un signal de désactivation d'alarme. Ici j'utilise un Flipper Zero pour désactiver l'alarme une fois celle-ci déclenchée.

Dans les faits, il est préférable de désarmer l'alarme avant d'entrer...

<video controls width="100%">
  <source src="/assets/vid/neutraliser-alarme/disable-alarme.mp4" type="video/mp4">
</video>

L'attaque par rejeu fonctionne car le signal est statique et qu'il n'y a pas de compteur de trame sur cette alarme

## Attaque 2: Jamming

### Pourquoi ne pas simplement rejouer le signal de désarmement ?

L'attaque par replay pour désarmer l'alarme fonctionne, mais il présente deux inconvénients: 
- le propriétaire reçoit une notification indiquant que son alarme a été désarmée.
- le désarmement apparaît dans les logs de l'application mobile. 

![alt text](/assets/img/posts/neutraliser-alarme/log-alarme.png){: width="400" .center}

D'où l'intérêt de neutraliser l'alarme sans la désarmer : en empêchant les capteurs de communiquer avec la centrale.

### Principe 

Le brouillage consiste à émettre des interférences sur la fréquence `433 MHz` afin de noyer les signaux des capteurs. La centrale ne reçoit plus les signaux d'intrusion (capteur de présence, capteur de porte), l'alarme ne se déclenche pas, et rien n'apparaît dans les logs.

### Mise en œuvre

Le HackRF émet un signal parasite sur `433 MHz`. La centrale reçoit plus de bruit que de signaux légitimes : les signaux des capteurs sont noyés dans la masse.

<!-- screenshot du spectre montrant le signal brouillé -->

<video controls width="100%">
  <source src="/assets/vid/neutraliser-alarme/jamming-alarme.mp4" type="video/mp4">
</video>

Dans cette vidéo, les capteurs continuent d'envoyer des signaux radio à la centrale mais celle-ci est incable de les voir car ils sont complétement noyés dans la masse. L'alarme ne se déclenche pas malgré le passage devant le capteur de présence.

### Limites

Le brouillage n'est pas toujours efficace. Pour brouiller de manière fiable, il faut émettre à puissance suffisante pour couvrir le signal des capteurs.

De plus, le spectre d'un brouilleur est très caractéristique et facilement détectable par un équipement de surveillance radio.

![alt text](/assets/img/posts/neutraliser-alarme/jamming-spectre.png){: width="950" .center}

Ici la fréquence `433,92 MHz` est totalement saturé.

Rappel : l'utilisation ET la détention d'un brouilleur sont illégales en France.

À des fins d'expérimentation, le brouillage a été réalisé à très basse puissance, sur quelques mètres et pendant quelques secondes, de manière à ne perturber aucun système tiers.

## Mitigations

La plupart des alarmes haut de gamme utilisent des communications chiffrées sur plusieurs canaux, ce qui rend les deux attaques précédentes plus difficiles.

Cependant, beaucoup d'alarmes d'entrée ou moyenne gamme sont très légères en matière de sécurité. Investir dans une alarme haut de gamme empêchera ou rendra plus difficiles ces attaques.

### Rolling code

Le rolling code empêche l'attaque par replay. Le principe : chaque signal émis est différent. Un compteur de trame chiffré est inclus dans la transmission. Le récepteur déchiffre le signal, vérifie le compteur et s'assure que la trame n'a jamais été reçue (typiquement en acceptant les N codes suivants attendus pour tolérer les appuis hors portée).

Certaines attaques existent, comme l'attaque par RollJam : l'attaquant brouille la réception tout en capturant le code émis par la télécommande, puis capture un second code lors de l'appui suivant et rejoue le premier. Il possède alors le prochain code valide.

Dans la réalité, cette attaque est complexe à mettre en œuvre car il faut synchroniser plusieurs SDR avec les appuis sur la télécommande de la victime. De plus il devra utiliser rapidement son code valide car il deviendra inutilisable dès que la victime aura réutiliser sa télécommande.

Cependant, le rolling code n'est pas toujours bien implémenté... Et quand ce n'est pas le cas, il peut se contourner... on en reparlera dans un prochain article...

### Communication multi-canal / saut de fréquence (FHSS)

Certaines alarmes haut de gamme (`Ajax` ou certains modèles `Daitem`) communiquent sur plusieurs canaux dans la bande 868 MHz, en sautant de fréquence à chaque transmission (`FHSS` — Frequency Hopping Spread Spectrum).

Cette approche rend l'interception des signaux plus complexe car il faut capturer simultanément sur tous les canaux possibles.

Le brouillage est également plus difficile car il faut couvrir une plage de fréquences plus large ou brouiller sur plusieurs fréquences.

### Détection de brouillage

Certains systèmes intègrent une détection de brouillage : la centrale surveille le niveau de bruit sur sa fréquence de communication. Si le plancher de bruit dépasse un seuil pendant une durée anormale, elle considère qu'un brouilleur est actif et déclenche une alerte (sirène, notification).

### Canal de secours (GSM/4G/Ethernet)

Certaines centrales disposent d'une connexion GSM/4G ou Ethernet en complément du lien radio. En cas de brouillage radio, l'alerte peut transiter par ce canal de secours. C'est une mitigation efficace, à condition que le canal de secours ne soit pas lui-même brouillé.

Comme pour les caméras Wi-Fi, la solution la plus robuste reste le filaire. Une alarme dont les capteurs sont raccordés par câble à la centrale est immunisée contre le brouillage et le replay radio.

Mais cela demande une installation beaucoup plus compliqué en raison des cable qu'il faudra dissimuler.

## Conclusion

Les vulnérabilités présentées ne sont pas spécifiques à une marque. Elles sont structurelles à la majorité des alarmes d'entrée de gamme qui communiquent en `433 MHz`.

Il n'existe pas de solution 100 % efficace contre un attaquant. Cependant, une alarme, même imparfaite, combinée à un panneau d'avertissement visible dissuadera la majorité des tentatives.

Les cambrioleurs opportunistes ne maîtrisent généralement pas ces techniques et privilégient les cibles non protégées.
Le rapport dissuasion/coût d'une alarme grand public reste favorable, à condition d'être conscient de ses limites.


