---
title: Ne laissez pas traîner vos clefs
date: 2026-04-03
categories: [IoT]
tags: [IoT, NFC, Home Automation]
---

# Ne laissez pas traîner vos clefs

## Contexte

Votre collègue Toto a installé chez lui une serrure connectée de ce type, achetée à bas coût sur Internet. Votre mission est de vous introduire chez lui.

![alt text](/assets/img/posts/Ne-laissez-pas-traîner-vos-clefs/badge-serrure.png)

La serrure s'ouvre de 5 manières :
- code
- empreinte digitale
- badge
- clef
- depuis l'application mobile TTLock

Nous allons nous intéresser à la partie badge NFC. L'objectif sera d'ouvrir la serrure de Toto en copiant son badge.

## Rappel juridique

La loi est très claire : s'introduire dans un système informatisé sans l'autorisation du propriétaire est illégal (articles 323-1 et suivants du Code pénal).

Ce scénario est fictif et toutes les manipulations ont été effectuées sur ma propre serrure, dans mon propre lab.

Au vu des photos de mon lab de serrure connectée, vous devriez le comprendre aisément.

## Étape 1 : Récupérer les informations du badge

Dans notre scénario, Toto a laissé traîner ses clefs. Vous n'avez pas d'équipement de type Flipper Zero ou Proxmark. En revanche, vous avez votre téléphone dans la poche. En utilisant l'application NFC Tools, vous scannez le badge et obtenez les informations suivantes :

![alt text](/assets/img/posts/Ne-laissez-pas-traîner-vos-clefs/badge-table.png)

![alt text](/assets/img/posts/Ne-laissez-pas-traîner-vos-clefs/nfc-tool.png)

On y retrouve :

- Type de tag : ISO 14443-3A — NXP Mifare Classic 1K
- Numéro de série (UID) : 48:A3:D3:E3
- ATQA : 0x0004
- SAK : 0x08
- Mémoire : 1 Ko répartie en 16 secteurs de 4 blocs (16 octets par bloc)

**Pourquoi ça fonctionne avec un simple téléphone ?**
Ici, c'est possible car le récepteur de la serrure vérifie uniquement le numéro de série (UID) de la carte.
Dans la vraie vie, vous n'avez aucun moyen de le savoir avant d'essayer.

Dans certains cas, la serrure vérifie aussi le contenu de la carte en lecture.
Tout va alors dépendre de la technologie de la carte et des protections mises en place par le constructeur, notamment les mots de passe protégeant l'accès aux secteurs.

**Et si la serrure vérifie le contenu de la carte ?**
Depuis plusieurs années, les cartes de type Mifare Classic 1K sont presque toujours vulnérables.
Même si des mots de passe robustes protègent les secteurs, des techniques d'attaque cryptographique (comme les attaques darkside, nested ou hardnested) permettent dans la quasi-totalité des cas de les retrouver.

En utilisant un Flipper Zero ou un Proxmark3, quelques minutes suffisent la plupart du temps pour récupérer l'intégralité du contenu de la carte.

**En résumé** :
si Toto s'éloigne quelques secondes de son bureau ET que vous avez juste votre téléphone avec vous, ALORS croisez les doigts pour que seul le numéro de série suffise à copier la carte.

## Étape 2 : Créer une nouvelle carte

Maintenant que vous avez les informations du badge sur votre téléphone, vous pouvez tranquillement créer un clone en utilisant un Proxmark3.

![alt text](/assets/img/posts/Ne-laissez-pas-traîner-vos-clefs/proxmark.png)

J'utilise ici une Magic Card. Contrairement à une carte Mifare Classic légitime dont le bloc 0 (contenant le numéro de série) est verrouillé en écriture en usine, une Magic Card permet de modifier librement ce numéro de série.

Pour écrire l'UID sur la Magic Card, on peut utiliser la commande Proxmark3 suivante :

```
hf mf csetuid -u 48A3D3E3 -a 0004 -s 08
```

Puis vérifier les informations de la carte avec :

```
hf mf info
```

![alt text](/assets/img/posts/Ne-laissez-pas-traîner-vos-clefs/pm3-info.png)

On constate que l'UID, l'ATQA et le SAK correspondent bien à ceux du badge original. Comme on l'a vu plus tôt, pas besoin de modifier le contenu des autres secteurs de la carte car notre serrure ne les vérifie pas.

On peut aussi utiliser le Flipper Zero en saisissant directement l'UID en hexadécimal dans le menu NFC :

![alt text](/assets/img/posts/Ne-laissez-pas-traîner-vos-clefs/emulate-card-flipper.png)

## Étape 3 : Ouvrir la porte

Maintenant que vous avez créé votre carte clone, vous pouvez l'utiliser pour ouvrir la serrure de Toto. La LED verte confirme que l'accès est accordé.

Avec la carte UID clonée :

![alt text](/assets/img/posts/Ne-laissez-pas-traîner-vos-clefs/open-door-card.png)

Avec le Flipper Zero en mode émulation :

![alt text](/assets/img/posts/Ne-laissez-pas-traîner-vos-clefs/open-door-flipper.png)

Les deux méthodes fonctionnent : que ce soit en présentant la carte physique clonée ou en utilisant le Flipper Zero qui émule le badge, la serrure s'ouvre.

## Mitigations

### Les lecteurs qui écrivent sur le badge

Certains systèmes de contrôle d'accès ne se contentent pas de lire le badge : ils y écrivent des données à chaque utilisation.
C'est le cas par exemple de certains systèmes Intratone utilisés dans les immeubles résidentiels en France.
Ces systèmes implémentent un rolling counter :
à chaque passage, le lecteur incrémente un compteur stocké sur le badge.

Si vous créez un clone du badge, le compteur de votre clone sera figé à la valeur au moment de la copie.
Dès que le badge original est utilisé, son compteur avance et le clone est désynchronisé, ce qui risque de rendre votre clone inutilisable.

Un conseil si vous voulez copier votre propre badge : effectuez une première lecture avant de créer le clone, puis une seconde lecture après. De cette manière, vous pourrez comparer les deux lectures et identifier ce qui a été modifié.

### La détection des Magic Cards

Certains lecteurs sont capables de détecter les Magic Cards et de refuser l'accès, même si la copie est parfaite au niveau des données.

**Vérification du fabricant** :
les cartes Mifare Classic authentiques sont fabriquées par NXP Semiconductors. Les Magic Cards utilisent des puces Fudan ou d'autres fabricants.
Pour ce faire, le lecteur peut examiner :

**1er octet de l'UID** :
les vrais UID NXP commencent par 0x04 (normalisé ISO/IEC 7816-6). Les puces Fudan ont leurs propres préfixes.

**Les octets 8-15 du bloc 0** :
les cartes Fudan laissent souvent la séquence `62 63 64 65 66 67 68 69` (= "bcdefghi" en ASCII).

Tout comme mon Proxmark a réussi à détecter la marque de ma Magic Card, le lecteur de badge peut également détecter les clones de cette manière.

**Sonder la commande de réveil magic (Magic Wakeup)** :
Pour comprendre cette détection, il faut garder en tête qu'un badge NFC ne fait pas que lire et écrire des données.
Il répond aussi à tout un ensemble de commandes protocolaires envoyées par le lecteur : réveil (REQA, WUPA), mise en veille (HLTA)...

Le lecteur peut envoyer la commande Magic Wakeup. Les Magic Cards y répondront par un ACK, ce qui trahit leur identité.
C'est d'ailleurs de cette manière que le Flipper Zero est capable de vérifier si le tag qui lui est présenté est une Magic Card.

Il existe d'autres méthodes de détection des Magic Cards, mais ce sont de loin les plus courantes.

## Conclusion

Ce petit exercice illustre bien la facilité avec laquelle un badge NFC bas de gamme peut être cloné. Avec un simple smartphone et quelques secondes d'accès au badge, il est possible de récupérer son UID.

Avec un Proxmark3 ou un Flipper Zero et une Magic Card à quelques euros, le clone est fonctionnel en quelques minutes.

