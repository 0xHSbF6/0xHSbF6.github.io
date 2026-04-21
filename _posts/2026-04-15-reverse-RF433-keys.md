---
title: Reverse engineering télécommandes RF433
date: 2026-04-02
categories: [IoT]
tags: [IoT, Home Automation, Radio Frequency]
---

# Reverse engineering télécommandes RF433

## Introduction

Beaucoup de télécommandes pour ouvrir les portails de garage ou pour activer/désactiver des alarmes anti-intrusion d'entrée de gamme utilisent une porteuse à `433,92 MHz`.

L'objectif de cet article est d'analyser en détail les messages envoyés par ces télécommandes, les décoder et ensuite pouvoir générer nos propres signaux afin de tester la sécurité de nos équipements domotiques.

Nous verrons dans cet article que bien souvent les signaux envoyés ne sont pas chiffrés et sont statiques. 

Si vous capturez un signal de désactivation d'alarme ou d'ouverture d'une porte de garage et qu'il n'y a pas de rolling code, vous serez en mesure de faire une attaque par rejeu et donc désactiver l'alarme ou ouvrir la porte.

Sur ce bloc, j'utilise cette attaque dans l'article "Comment neutraliser une alarme anti-intrusion ?"

Nous verrons également qu'on peut aller plus loin en déduisant le signal d'ouverture à partir du signal de fermeture, et dans le dernier cas, qu'un espace combinatoire insuffisant permet un bruteforce complet en dizaines de minutes. 

## Matériel et logiciel

### Hardware

- **Flipper Zero** 

Émetteur/récepteur très pratique car il tient dans la poche. Il est capable de décoder la plupart des signaux `RF 433 MHz` et de générer de nouveaux signaux. Utile pour une validation rapide sur le terrain, mais insuffisant pour une analyse approfondie. Environ 200€.

- **HackRF One** 

Émetteur/récepteur couvrant de `1 MHz` à `6 GHz`. Nécessite un PC. Très puissant pour la génération de signaux de tout type, pas uniquement RF 433. Environ 400€ avec les antennes.
- **RTL-SDR** 
Récepteur uniquement. Peu coûteux (~40€). Doit être relié à un PC. Peut aller jusqu'à `1,7 GHz`. 

### Software

- **URH (Universal Radio Hacker)** 
- 
Permet de visualiser, analyser et décoder des signaux radio capturés. 

<a href="https://github.com/jopohl/urh">Universal Radio Hacker</a>

- **rtl_433** 

Outil de décodage automatique de signaux `RF 433 MHz`. Supporte des centaines de protocoles et permet d'identifier rapidement l'encodage utilisé.

<a href="https://github.com/merbanan/rtl_433">rtl_433 </a>

## A - Télécommande RF433 classique

### Reverse

Cette télécommande permet d'ouvrir et fermer une porte de garage à distance.

Quand nous appuyons sur un bouton, nous pouvons connaître la fréquence de communication grâce au spectre.

![alt text](/assets/img/posts/Reverse-engineering-télécommande-RF433/spectre-urh-telecommande1.png){: width="950" .center}


Il est possible aussi de voir un peu plus en détail le signal radio.

![alt text](/assets/img/posts/Reverse-engineering-télécommande-RF433/capture-urh-telecommande1.png){: width="950" .center}

URH nous permet d'extraire du signal radio les bits du signal envoyé par la télécommande.

![alt text](/assets/img/posts/Reverse-engineering-télécommande-RF433/extraction-signal-urh-telecommande1.png){: width="950" .center}

Voici la photo du `PCB` de cette télécommande.

![alt text](/assets/img/posts/Reverse-engineering-télécommande-RF433/pcb-telecommande1.png){: width="450" .center}

En ouvrant le `PCB` de la télécommande, nous pouvons récupérer le nom du CI qui gère l'encodage des signaux radio. On peut lire `STC1527`. Une rapide recherche sur internet nous amène sur la datasheet du CI http://www.sc-tech.cn/en/SCT1527.pdf

![alt text](/assets/img/posts/Reverse-engineering-télécommande-RF433/sct1527-datasheet-1.png){: width="900" .center}

Le décodage est assez facile et peut se faire à la main. Grâce à la datasheet, nous savons que :

```
0 = L = 1000
1 = H = 1110
```

![alt text](/assets/img/posts/Reverse-engineering-télécommande-RF433/sct1527-datasheet-2.png){: width="900" .center}

Analysons le signal. Celui-ci est répété 14 fois quand on appuie sur le bouton. De cette manière, même s'il y a une interférence au moment de l'appui, le signal sera quand même compréhensible par le récepteur.

```
100010001000111011101110100010001000100011101000111011101000111011101000111010001110100010001000
```

Ce qui nous donne

```
1000 1000 1000 1110 1110 1110 1000 1000 1000 1000 1110 1000 1110 1110 1000 1110 1110 1000 1110 1000 1110 1000 1000 1000
```

Oublions le `1` qui se glisse entre chaque trame. C'est le préambule. 

En suivant la doc, on obtient :

```
0001 1100 0010 1101 1010 1000 --> 1C2DA8.
```

Même sans la datasheet, il est possible de lire le signal visuellement en zoomant dans `URH` :


![alt text](/assets/img/posts/Reverse-engineering-télécommande-RF433/analyse-urh-telecommande-1.png){: width="950" .center}

Ceci est un `0`

![alt text](/assets/img/posts/Reverse-engineering-télécommande-RF433/0-urh-telecommande1.png){: width="950" .center}

Ceci est un `1`

![alt text](/assets/img/posts/Reverse-engineering-télécommande-RF433/1-urh-telecommande1.png){: width="950" .center}

La doc indique que le signal est composé de deux parties (hormis le préambule). Les premiers octets `1C2DA8` encodent le `Device ID`. Ils identifient la télécommande. Le dernier octet encode le bouton.

Si nous avons configuré notre alarme pour l'activer à partir du bouton 1 `1C2DA8` et la désactiver avec le bouton 2 `1C2DA4`

Vous pouvez aussi le décoder automoatiquement avec le flipper zéro mais c'est moins drôle.

![alt text](/assets/img/posts/Reverse-engineering-télécommande-RF433/flipper-1C2DA8.png){: width="450" .center}

Bonne nouvelle, nous retrouvons les mêmes valeurs que lors de notre analyse manuelle. 

Ou avec `rtl_433`: 

```
rtl_433 -M hires -M level
```

![alt text](/assets/img/posts/Reverse-engineering-télécommande-RF433/rtl_433-1C2DA8.png){: width="950" .center}

Attention : le nom `Akhan-100F14` est inexact car `rtl_433` déduit le nom de l'équipement à partir du signal décodé. Parfois, il se trompe.

En revanche il a bien trouvé le device ID `0x1C2DA` et le bouton `0x8`.

### Creér une nouvelle clef

Maintenant que nous avons compris comment est encodé le signal, nous pouvons le modifier pour générer un signal compatible avec le récepteur. 

Imaginons que nous n'ayons capturé que le signal de fermeture de la porte. Nous voulons l'ouvrir.

Nous savons que le signal de fermeture est:

```
1000 1000 1000 1110 1110 1110 1000 1000 1000 1000 1110 1000 1110 1110 1000 1110 1110 1000 1110 1000 1110 1000 1000 1000 ==> 1C2DA8
```

Il suffit de modifier le dernier octet pour obtenir le signal d'ouverture :

```
1000 1000 1000 1110 1110 1110 1000 1000 1000 1000 1110 1000 1110 1110 1000 1110 1110 1000 1110 1000 1000 1110 1000 1000 ==> 1C2DA4
```

Appliquons cette modification sur urh. Depuis la fenêtre générateur, nous pouvons modifier directement les `0` et les `1`. 

Attention à appliquer la modification partout car le signal est répété plusieurs fois.

![alt text](/assets/img/posts/Reverse-engineering-télécommande-RF433/create-new-key.png){: width="950" .center}

Appuyons sur start. Le flipper zéro sera l'arbitre et affichera le signal.

<video controls width="100%">
  <source src="/assets/vid/Reverse-engineering-télécommande-RF433/play-new-signal.mp4" type="video/mp4">
</video>

Techniquement, le signal est bon et face à un vrai récepteur radio, la porte de garage s'ouvrirait. A défaut, il faudra aussi encoder le signal `1C2DA2` et `1C2DA1`.

## B - Télécommande RF433 vulnérable

On va maintenant reverser le signal emis par une serrure connectée vulnérable disponible sur Amazon sous différentes marques (rebranding)

**Capture et analyse du signal**

On capture le signal avec le RTL-SDR pendant qu'on appuie sur le bouton de la télécommande. En ouvrant le fichier `cu8` dans `URH`, on observe la structure suivante :

![alt text](/assets/img/posts/Reverse-engineering-télécommande-RF433/image-29.png){: width="950" .center}

Une longue impulsion de synchronisation à gauche (~5400µs), suivie d'une série de pulses réguliers avec des espacements variables. 

Pour encoder un 0 ou un 1 dans un signal `ASK/OOK`, il n'existe que quelques méthodes courantes :

- **PWM (Pulse Width Modulation)** : 

c'est la largeur du pulse qui encode le bit 
- un pulse court = `0`, 
- un pulse long = `1`. 
  
C'est ce qu'utilise la première télécommande avec le `STC1527`

- **PPM (Pulse Position Modulation)** : 

c'est la durée du silence après un pulse fixe qui encode le bit

Sur notre signal, les pulses sont tous identiques (`~400µs`). En revanche on observe clairement deux catégories de gaps : des silences courts (`~400µs`) et des silences longs (`~800µs`). Ce qui est une signature de la modulation `ASK/PPM`

On peut donc poser l'hypothèse suivante

```
Gap court  : ~400µs → bit 0
Gap long   : ~800µs → bit 1
```

Le signal est répété entre 10 et 32 fois selon la durée d'appui sur la télécommande. 

En suivant ces règles, nous pouvons lire

```
1101 0101 0000 0110 0000 0000 0000 001 ===> 0xD5060002
```

Nous avons vu sur les deux premières télécommandes que les signaux provenant du bouton 1 et du bouton 2 d'une même télécommande se ressemblent beaucoup. Seul 4 bits changent (hormis la partie chiffrée sur le rolling code).

Si nous comparons avec cette méthode deux signaux provenant de cette télécommande, nous trouvons `0xD5040002`

![alt text](/assets/img/posts/Reverse-engineering-télécommande-RF433/seconde-clef-signal-urh.png){: width="950" .center}

Ce qui confirme que notre analyse est correcte.

En analysant plusieurs télécommandes de la même gamme, on observe rapidement un pattern dans les payloads :

```
0xAAA60002  ===>  fermeture
0xAAA40002  ===>  ouverture
0xD5060002  ===>  fermeture
0xD5040002  ===>  ouverture
0x56C60002  ===>  fermeture
0x56C40002  ===>  ouverture
```

```
0x[device_id][action]0002
```

- **device_id** : 12 bits, identifiant unique de la télécommande
- **action**: 0x6 = fermeture  ou  0x4 = ouverture
- **0x0002** : suffixe fixe du protocole


**La faille combinatoire**

Le device_id est encodé sur `12 bits`, ce qui donne seulement 4096 combinaisons possibles (`0x000` à `0xFFF`). C'est catastrophique. La première télécommande que nous avons analyser avait `16⁵` device_id différent. 

L'attaque par bruteforce est donc largement possible sur cette nouvelle serrure. En fonction de la durée d'émission et notre chance, une quarantaine de minutes suffisent pour trouver une clé valide. 

Ce script permet de généréer un signal valide. 

```py
import numpy as np

SAMPLE_RATE = 2_000_000
FREQ_HZ     = 433_840_000

SYNC_US    = 5_380
PULSE_US   =   400
GAP0_US    =   400  # bit 0
GAP1_US    =   805  # bit 1

def segment(on, duration_us):
    n = int(round(duration_us * SAMPLE_RATE / 1_000_000))
    i_val = np.uint8(127) if on else np.uint8(0)
    return np.column_stack([np.full(n, i_val), np.zeros(n, np.uint8)]).flatten()

def generate(device_id, action, n_repeat=27):
    payload = (device_id << 20) | (action << 16) | 0x0002
    bits    = format(payload, '032b')
    frames  = [segment(False, 10_000)]
    for _ in range(n_repeat):
        frames.append(segment(True,  SYNC_US))
        frames.append(segment(False, GAP1_US))
        for bit in bits:
            frames.append(segment(True,  PULSE_US))
            frames.append(segment(False, GAP1_US if bit == '1' else GAP0_US))
    frames.append(segment(False, 10_000))
    return np.concatenate(frames)

signal = generate(device_id=0xD50, action=0x4, n_repeat=30)
signal.tofile("signal_ouverture.cu8")
```

Pour envoyer le signal avec le hackrf

```
hackrf_transfer -t signal_ouverture.cu8 -f 433840000 -s 2000000 -a 1 -x 47 -R
```

Il est ensuite facile de générer l'ensemble des signaux possibles afin d'ouvrir cette fameuse serrure par bruteforce. 

Lors de mes essais de bruteforce, je me suis apperçu que si un `device_id[n]` était valide `devide_id[n+1]` aussi
par exemple:

```
0x56C40002  ===>  ouverture
0x56C60002  ===>  fermeture
0x56D40002  ===>  ouverture
0x56D60002  ===>  fermeture
```

Ceci est vrai quelque soit le device_id. 


## C - Clef RF433 Rolling code

### Introduction au rolling


Nous allons maintenant analyser une seconde clef avec du rolling code qui fonctionne elle aussi sur `433 MHz`. Elle est censée être plus sécurisée que les précédentes.

**Fonctionnement du rolling code**

Le principe : à chaque appui bouton, un compteur s'incrémente côté émetteur. Ce compteur est chiffré avec une clé partagée entre l'émetteur et le récepteur. Le code transmis change donc à chaque utilisation, rendant l'enregistrement et le rejeu d'un code capturé inutile. Le récepteur déchiffre le message, vérifie que le compteur est dans une fenêtre acceptable, et exécute la commande si tout est cohérent.

Un des algorithmes les plus courants dans les appareils domotiques à rolling code est `KeeLoq (Microchip)`. 

Il existe en version "Classic" et en version "Ultimate" qui utilise `AES-128` à la place du chiffrement `KeeLoq` d'origine qui utilise une clé de `clé de 64 bits`.

La clé de chiffrement de chaque télécommande (`device key`) est dérivée en combinant cette clé fabricant avec le numéro de série de l'émetteur. 

Bien évidemmeent des clés ont fabricant plusieurs fois vu leur clé leaker, ce qui permet de déchiffrer les trames de tous les appareils de la marque concernée.

<a href="https://www.youtube.com/watch?v=Zr_NCoSH2cg">Unlocking KeeLoq: A Reverse Engineering Story - Rogan Dawes</a>

L'attaque `rolljam` qui permet sur le papier de récupérer un code valide. Dans les faits sauf si c'est un fake Tiktok ou que vous avez les planètes alignées. Cette attaque est difficile à réaliser dans la pratique. 

Même si le rolling code est intéressant, il n'est pas toujours bien implémenté ce qui permet de l'attaquer sans avoir besoin de faire une attaque rolljam

![alt text](/assets/img/posts/Reverse-engineering-télécommande-RF433/setup-analyse.png){: width="850" .center}

Ceci est un récepteur que j'ai testé qui permet une attaque par rejeu à cause d'une faille dans la réassociation de la clef... 

Normalement nous ne pouvons pas réassocier une clef avec un compteur très inférieur au compteur du récepteur... 

La réassociation se fait très bien quand le compteur de la télécommande est en avance sur le compteur du récepteur. L'inverse pose problème... car à ce moment c'est du rejeu...

Je dis ça je dis rien...

### Reverse cle RF433 Keeloq

On peut voir que le signal est envoyé plusieurs fois, toujours pour la même raison : garantir la réception même en cas d'interférence.

![alt text](/assets/img/posts/Reverse-engineering-télécommande-RF433/urh-rollingcode.png){: width="950" .center}

Si nous appuyons 3 fois sur le même bouton, on observe 3 signaux différents :

```
101010101010101010101 [Pause: 4325 samples]
11010011010011010011011011011011011010011010011010011011010010011010011011010011011010010011010010011011011011011011010011010010011011011010011010011011010011010011010010010010010011010011011011011

101010101010101010101 [Pause: 4325 samples]
10011011010011011010011010010010011011010010011010011011011011010011011011011011011010010010011010011011011011011011010011010010011011011010011010011011010011010011010010010010010011010011011011011

101010101010101010101 [Pause: 4334 samples]
10011010010011011011010010010010011010011010010010010010011010010011010010010011011011010010011010011011011011011011010011010010011011011010011010011011010011010011010010010010010011010011011011011
```

Nous allons suivre le même protocole de reverse que précédemment et analyser, dans un premier temps, le `PCB`

![alt text](/assets/img/posts/Reverse-engineering-télécommande-RF433/pcb-hcs300.png){: width="450" .center}

On peut lire sur le `CI` principal le code `HCS300`. La datasheet est facilement trouvable en ligne et nous fournit les informations nécessaires pour décoder le signal.

![alt text](/assets/img/posts/Reverse-engineering-télécommande-RF433/hcs300-datasheet.png){: width="700" .center}

Le signal utilise un encodage `PWM` où chaque bit est représenté par un `rapport cyclique` différent.

La datasheet nous donne également le mapping des champs :

![alt text](/assets/img/posts/Reverse-engineering-télécommande-RF433/reverse-hcs300.png){: width="850" .center}

Le mot de `66 bits` est transmis `LSB` en premier. Dans l'ordre de réception :

- **Bits 0–31** : Encrypted (32 bits chiffrés KeeLoq)
- **Bits 32–59** : Serial Number (28 bits, en clair)
- **Bits 60–63** : Boutons dans l'ordre S3, S0, S1, S2
- **Bit 64** : VLOW (batterie faible)
- **Bit 65** : RPT (repeat)

Le signal est envoyé en deux parties : un préambule puis les données.

- préambule
- data

Les trois signaux partagent le même préambule: `101010101010101010101`. Il n'est pas nécessaire de le décoder car c'est juste un signal de synchronisation. 

Donc maintenant on va s'intéresser à la partie data du premier signal

```
11010011010011010011011011011011011010011010011010011011010010011010011011010011011010010011010010011011011011011011010011010010011011011010011010011011010011010011010010010010010011010011011011011
```

La datasheet nous dit que 

```
1:110
0:100
```

donc 

```
110 100 110 100 110 100 110 110 110 110 110 110 100 110 100 110 100 110 110 100 100 110 100 110 110 100 110 110 100 100 110 100 100 110 110 110 110 110 110 100 110 100 100 110 110 110 100 110 100 110 110 100 110 100 110 100 100 100 100 100 110 100 110 110 110 11
```

Les `66 bits` décodés donnent :

```
101010111111010101100101101100100111111010011101011010100000101111
```
En suivant le mapping:
```
Bits 0-31 (Encrypted)  : 10101011111101010110010110110010
Bits 32-59 (Serial 28b): 0111111010011101011010100000
Bits 60-63 (Boutons)   : 1 0 1 1  --> le bouton 1 est appuyé
Bit 64 (VLOW)          : 1
Bit 65 (RPT)           : 1
````

En comparant les 3 signaux, la partie chiffrée change bien à chaque appui :

(Attention au `LSB` first)
```
10101011111101010110010110110010 --> 0x4DA6AFD5
01101101000110010111101111110001 --> 0x8FDE98B6
01001110000101000001001000111001 --> 0x9C482872
```

Le `serial number` reste identique :
```
0111111010011101011010100000 --> 0x056B97E
```

Dans les 3 cas c'est le même bouton qui est appuyé donc `1011`
