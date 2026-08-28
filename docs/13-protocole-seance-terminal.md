# 13 — Protocole de la séance au terminal série (item 4bis)

**Date :** 2026-08-28
**Objet :** la séance qui caractérise l'adaptateur, produit la première trace, et
lève le seul véritable préalable à l'écriture de code.
**Préalables :** `07` §4bis (l'item), `10` §1 et §4 (ce qui dépend des mesures),
`10` §9 (le transport de rejeu qui consommera la trace), `09` §3 (ce que le
firmware fait à ta place).

Ce document est **opérationnel** : il se lit avec le camion à côté, pas dans un
fauteuil. `07` §4bis dit *ce qu'il faut savoir* ; celui-ci dit *quoi taper, dans
quel ordre, et quoi noter*.

> ⛔ **Ce document n'est pas la spécification d'un passe-plat.** Il décrit une
> séance humaine au terminal série, ce que `10` §8 désigne explicitement comme
> le lieu du débogage : *« Le débogage se fait au terminal série, hors du
> serveur. »* Aucune des commandes ci-dessous ne doit apparaître dans une
> surface MCP, CLI ou bibliothèque. L'interdit n°1 du projet est intact.

---

## 0. Ce que la recherche a levé avant de toucher au véhicule

**La séance a rétréci.** Le manuel primaire du constructeur de la puce répond sur
papier à une bonne moitié de ce que `07` §4bis présentait comme à mesurer — et
il **contredit** deux affirmations du dossier. Autant le savoir avant de brancher
quoi que ce soit.

**Source :** *OBDLink® Family Reference and Programming Manual (FRPM)*,
**révision F, 2025-08-29**, OBD Solutions. Source primaire du fabricant, d'où
les étiquettes `[VÉRIFIÉ]` ci-dessous. Les mesures, elles, restent à faire :
un manuel dit ce que le firmware *prétend* faire.

### 0.1 Deux corrections au dossier

> ### ⛔ La commutation MS-CAN n'est pas un relais, et il n'y a pas de commande `ATSW`
>
> `05` §3.1 écrit qu'il faudra *« envoyer les commandes STN spécifiques
> (`ATSW`/`ATMSCAN`) et le pilotage du relais »*. **Ces commandes n'existent
> pas**, et il n'y a pas de relais.
>
> `[VÉRIFIÉ]` FRPM §8.6 : *« OBDLink ICs support a maximum of three physical CAN
> channels… Internally, the OBDLink has only one CAN peripheral that can be
> mapped to different IC pins under software control. This means that **only one
> CAN channel can be active at a time**. »*
>
> **La commutation de bus est donc un changement de préréglage de protocole**, et
> rien d'autre :
>
> | Préréglage `STP p` | Bus | Broches J1962 | Protocole |
> |---|---|---|---|
> | `33` | HS-CAN | 6 / 14 | ISO 15765, Tx 11 bits, 500 kbps, DLC=8 |
> | `34` | HS-CAN | 6 / 14 | ISO 15765, Tx 29 bits, 500 kbps, DLC=8 |
> | **`53`** | **MS-CAN** | **3 / 11** | **ISO 15765, Tx 11 bits, 125 kbps, DLC=8** |
> | `54` | MS-CAN | 3 / 11 | ISO 15765, Tx 29 bits, 125 kbps, DLC=8 |
> | `51` / `52` | MS-CAN | 3 / 11 | ISO 11898 **brut** — pas d'ISO-TP automatique |
>
> Le MS-CAN Ford est à **125 kbps**, ce que le préréglage porte déjà : pas besoin
> de `STPBR`. Et comme le périphérique CAN est **remappé**, toute session UDS
> ouverte sur l'autre bus est perdue par construction — la question de `10` §1
> *« et probablement une réinitialisation de session »* n'est donc plus une
> hypothèse, c'est une conséquence de l'architecture de la puce.
>
> ---
>
> ### ⚠️ L'OBDLink EX n'est probablement pas un STN2120
>
> Tout le dossier écrit **STN2120** (`05` §1.2, `09` §3, mémoire de projet). Le
> FRPM rév. F range les pièces autrement :
>
> | Référence | Ce que c'est | État |
> |---|---|---|
> | **STN2232** | **la puce de l'OBDLink EX actuel** | actif |
> | STN2230 / STN2231 | OBDLink EX, révisions antérieures | hors production |
> | STN2120 | **circuit interpréteur OBD-vers-UART**, vendu seul | **obsolète** |
>
> Ce n'est pas un détail cosmétique : les tailles de tampon et le débit utile
> sont donnés **par appareil** dans le manuel, et une caractéristique attribuée
> à la mauvaise puce est une caractéristique fausse. **La première commande de
> la séance tranche la question** — `STI` imprime la chaîne réelle, par exemple
> `STN2232 v5.10.3`. Tant qu'elle n'est pas relevée, l'étiquette du dossier
> reste `[NON VÉRIFIÉ]`.

### 0.2 Le piège d'identité : l'EX se présente comme un ELM327

`[VÉRIFIÉ]` FRPM §6.0 — au démarrage, l'appareil imprime :

```text
ELM327 v1.4b
>
```

Et `ATI` renvoie la même chose. **C'est de la compatibilité descendante, pas un
clone.** `05` §1.3 explique longuement que les étiquettes « ELM327 v1.5 / v2.1 »
sont des numéros fabriqués — l'ironie est qu'un **vrai** OBDLink dit `v1.4b` en
s'annonçant. Ne pas conclure de `ATI` qu'on a acheté un clone.

**Ce qui identifie réellement l'appareil :** `STI` (firmware), `STDI` (matériel),
`STMFR` (fabricant), `STSN` (numéro de série). Un clone ELM327 répond `?` à
toutes les quatre — c'est d'ailleurs le test de clone le plus rapide qui existe.

### 0.3 Ce qui est déjà répondu, et ce qui reste

| Question de `07` §4bis | Réponse documentaire | Reste à mesurer |
|---|---|---|
| Débit série, cadrage | `[VÉRIFIÉ]` **115200 bps, 8N1**, pour les OBDLink USB (FRPM §6.0). Modifiable par `STSBR`, jusqu'à **2 Mbps** sur l'EX **avec écho coupé** (`ATE0`) — 1 Mbps avec écho | Que le débit relevé tienne sur CDC-ACM côté macOS |
| Commande de commutation MS-CAN | `[VÉRIFIÉ]` `STP 53` — voir §0.1 | **Le délai**, qui n'est documenté nulle part |
| Taille max de réponse réassemblée | `[VÉRIFIÉ]` **4 ko** de taille de message pour l'OBDLink EX (FRPM §8.6, tableau par appareil). Recoupe la borne de l'ISO 15765-2 classique, dont le champ de longueur sur 12 bits plafonne à **4095 octets** — deux sources indépendantes qui tombent au même endroit | Que la **réception** réassemblée tienne jusque-là, le tableau du manuel portant sur la taille de message |
| Format des lignes de réponse | `[VÉRIFIÉ]` documenté et exemplifié — voir §0.4 | Confirmation sur le véhicule, et le format sous `ATH1` |
| Comportement du timeout | `[VÉRIFIÉ]` `ATST` est **déprécié** sur STN au profit de **`STPTO ms`**, en millisecondes directes (1 à 65535 ; 0 = infini). **Défaut : 102 ms** | Ce qui se passe *après* expiration, et si `$3E` est nécessaire |
| Besoin de `$3E TesterPresent` | — | **Entier.** Se mesure, ne se lit pas |
| Débit en requêtes/seconde | — | **Entier.** Dépend du véhicule autant que de l'adaptateur |

### 0.4 Le format de réponse, tel que documenté

`[VÉRIFIÉ]` FRPM §7.2, sous `ATCAF 1` (formatage automatique CAN, **actif par
défaut**), l'appareil : génère l'octet PCI des requêtes, **retire** les PCI des
réponses, retire le remplissage, ignore les trames RTR, et pour une réponse
multi-trames **imprime la longueur sur une ligne à part puis préfixe chaque
trame de son numéro de séquence suivi de deux-points**. Exemple du manuel :

```text
>0902
014
0: 49 02 01 31 47 31
1: 4A 43 35 34 34 34 52
2: 37 32 35 32 33 36 37
>
```

Trois choses à en tirer, et la troisième est la plus importante :

1. `014` est la longueur **en hexadécimal** — 20 octets. Un décodeur qui la lit
   en décimal se trompe d'un facteur.
2. `ATH1` **écrase une bonne partie** de ce formatage tout en continuant à
   générer le PCI des requêtes. Deux formats à supporter, donc, et un réglage
   d'adaptateur qui change la grammaire de sortie : c'est exactement le genre
   d'état que `10` §1 dit de ne jamais laisser à l'agent.
3. ⛔ **Cette réponse-là est un VIN.** `49 02 01` puis l'ASCII de
   `1G1JC5444R7252367`. L'exemple d'ouverture du manuel du fabricant est une
   donnée de couche 4. Retiens-le pour §4 : ce n'est pas une hypothèse
   théorique, c'est la sortie de la commande la plus banale du protocole.

### 0.5 Quatre défauts par défaut, qui mordront si on les ignore

Le firmware fait beaucoup, mais **pas dans sa configuration de sortie d'usine**.
Chacun des quatre points ci-dessous est un `[VÉRIFIÉ]` du FRPM, et chacun
invalide une hypothèse implicite du dossier.

| Réglage | Défaut | Pourquoi ça mord |
|---|---|---|
| `ATAL` / `ATNL` | **`ATNL`** — la limite J1979 de **7 octets de données est appliquée en réception** | `09` §3 suppose qu'on envoie `2201A0` et qu'on récupère la réponse assemblée. Sans `ATAL`, une réponse UDS longue est refusée avant même d'être réassemblée |
| `STCSEGR` / `STCSEGT` | **`0` — segmentation CAN désactivée** | Nuance importante sur *« ISO-TP délégué au firmware »* de `09` §3 : la délégation vient de `ATCAF1` pour les préréglages **ISO 15765** (33/34/53/54). Sur les préréglages **ISO 11898 bruts** (51/52), rien n'est réassemblé tant que `STCSEGR 1` n'est pas posé. **Un choix de préréglage décide donc si la couche ISO-TP existe** |
| Paires de contrôle de flux | **automatiques**, `TxID = RxID − 8` | Convention qui tient pour le groupe motopropulseur (`7E8 → 7E0`). Un module MS-CAN dont l'identifiant ne suit pas la convention **ne recevra jamais sa trame de contrôle de flux** : `FC RX TIMEOUT`, et la réponse multi-trames échoue. Il faut alors `STCFCPA` |
| `STCFCPA` | — | ⚠️ **Effet de bord majeur :** *« When using STCFCPA, automatic flow control pair generation is disabled, and only the manually added pairs will be used. »* Ajouter **une** paire coupe l'automatisme pour **toutes** les autres. Un pilote qui ajoute des paires à la demande casse ce qui marchait |

**Conséquence de conception, à porter en `09` §3 :** la formule *« le firmware
gère ISO-TP »* est vraie **sous conditions**, et les conditions sont un
préréglage ISO 15765, `ATCAF1`, `ATAL`, et des paires de contrôle de flux
cohérentes. La machine à états d'adaptateur de `10` §1 doit tenir ces quatre
réglages comme de l'état, pas comme une initialisation qu'on oublie.

### 0.6 Les erreurs de l'adaptateur sont une couche à part

Découverte qui vaut à elle seule la recherche : le FRPM §9.0 donne un
**catalogue fermé de chaînes d'erreur**, et cette couche est **distincte** des
réponses négatives UDS `7F` que `10` §4 énumère. Confondre les deux fait
raisonner faux — c'est précisément le mode de défaillance que `10` §4 combat.

| Chaîne | Ce que ça veut dire | Ce que l'humain doit faire |
|---|---|---|
| `?` | Syntaxe invalide, **ou commande inappropriée au contexte** | Corriger. ⚠️ Un `?` ne dit pas laquelle des deux |
| `NO DATA` | Aucune réponse avant expiration. **Quatre causes :** requête non supportée, requête bloquée par un filtre, `STPTO` trop court, protocole non supporté | ⛔ **Jamais conclure « module absent ».** C'est le pendant, côté adaptateur, du *« jamais module en panne »* de `10` §4 |
| `CAN ERROR` | Problème d'émission/réception : non connecté au bus, **mauvais protocole ou mauvais débit CAN**, câblage | **C'est la signature du mauvais bus.** Attendue si on interroge un module MS-CAN en `STP 33` |
| `FC RX TIMEOUT` | Aucune trame de contrôle de flux reçue à temps : bus chargé, trame filtrée, **ou paire FC absente** | `STCFCPA` — voir §0.5 |
| `BUS BUSY` | Expiration avant d'avoir détecté un bus au repos | *« In the majority of cases, this error indicates a wiring problem »* |
| `BUS ERROR`, `FB ERROR` | Tension de bus qui ne varie pas comme attendu ; incohérence émetteur/récepteur | Câblage, court-circuit, ou **un second outil qui émet en parallèle** |
| `BUFFER FULL` | Mémoire de réception épuisée | Monter le débit UART, `ATH0`, `ATS0`, ou filtrer |
| `OUT OF MEMORY` | RAM insuffisante pour l'opération | Réponse attendue si on demande un message plus grand que le tampon — **c'est le test de borne du §2, bloc C** |
| `UART RX OVERFLOW` | Débordement du tampon UART | Surtout sous ISO 9141/14230, hors périmètre ici |
| `STOPPED` | **Un caractère reçu sur l'UART a interrompu une commande OBD en cours** | ⚠️ **La justification matérielle de l'interdit n°9** (`09` §11.2) : deux écrivains sur le port ne produisent pas une erreur propre, ils produisent `STOPPED` au mieux, des octets entrelacés au pire |
| `LV RESET` | **Réinitialisation par sous-tension** — la puce redémarre et perd tout son état | ⚠️ **La justification électrique de l'interdit n°4.** Un balayage de DID de 40 minutes contact mis, moteur arrêté, sans mainteneur de batterie, finit là. Ce n'est pas une précaution de principe : c'est un mécanisme nommé dans le manuel du fabricant |
| `ACT ALERT`, `LP ALERT` | Passage en basse consommation imminent (1 minute / 2 secondes) | ⚠️ **Un second chien de garde**, indépendant de `$3E`. Voir §2 bloc F |
| `UNABLE TO CONNECT` | Protocole OBD indétectable : véhicule non conforme, **contact coupé**, pas d'alimentation au connecteur, câblage | Vérifier le contact **avant** de soupçonner le logiciel |
| `<DATA ERROR`, `<RX ERROR` | Échec de contrôle d'intégrité ; erreur détectée par le périphérique CAN | Connecteur oxydé, bruit, câble trop long, **ou mauvais débit** |

**À porter en `10` §4 :** la taxonomie d'erreurs y est construite autour des
conditions véhicule et UDS. Il lui manque **un étage** — les chaînes ci-dessus,
qui viennent de l'adaptateur et dont l'action humaine diffère. Trois d'entre
elles (`STOPPED`, `LV RESET`, `CAN ERROR`) sont des preuves matérielles
d'invariants que le dossier posait par prudence.

---

## 1. Préconditions

**Deux blocs sont faisables au bureau, sans véhicule.** C'est le sous-item non
bloquant de l'ajout du 2026-08-26 de `07` : reste à savoir si l'EX démarre sur l'alimentation USB
seule. Le protocole ci-dessous **répond à la question en passant** — bloc A0.

### Au bureau

- Adaptateur branché au Mac, **rien d'autre**.
- Un terminal série. `screen /dev/tty.usbmodem* 115200` suffit ; un outil qui
  **journalise** est nettement préférable — voir §3.
- ⛔ **Un seul programme ouvre le port.** Fermer FORScan, la VM Windows, toute
  application OBD. Deux écrivains produisent `STOPPED` ou du silence, pas une
  erreur lisible (§0.6).

### Au véhicule

| Précondition | Pourquoi |
|---|---|
| Véhicule **immobile, stationné, frein de stationnement serré** | Interdit n°4. Non négociable |
| **Contact mis, moteur arrêté** pour l'essentiel de la séance | Les modules sont réveillés sans consommer de carburant. `UNABLE TO CONNECT` est la réponse au contact coupé |
| **Mainteneur de batterie branché** | `LV RESET` (§0.6). Contact mis moteur arrêté pendant 45 minutes tire sur la batterie, et une puce qui redémarre en plein réassemblage donne une trace inexploitable en plus d'une mesure fausse |
| Une passe **moteur tournant**, courte, en fin de séance | Certains DID répondent `7F .. 22 conditionsNotCorrect` moteur arrêté. Une seule passe suffit à le documenter |
| 45 à 60 minutes sans être pressé | La partie utile est la partie qu'on ne bâcle pas |

### ⛔ Ce que la séance interdit, et il faut le dire explicitement

**Au terminal, l'énumération fermée de services n'existe pas.** L'interdit n°2
est appliqué *dans l'encodeur*, et il n'y a pas d'encodeur ici : le clavier parle
directement à la puce. **L'humain est le seul garde-fou.**

Ne jamais taper, à aucun moment, pour aucune raison :

| Service | Ce qu'il fait |
|---|---|
| `$11` | Réinitialise un module (`ECUReset`) |
| `$2F` | **Actionne des sorties physiques** (`InputOutputControlByIdentifier`) |
| `$31` | Lance des routines, parfois destructrices (`RoutineControl`) |
| `$28` | Peut couper la communication d'un module (`CommunicationControl`) |
| `$14` | Effacer des DTC — détruit l'information qu'on vient chercher |
| `$2E` | Écriture de DID (`WriteDataByIdentifier`) |
| `$27` | `SecurityAccess` — hors périmètre, palier 4 (`09` §4) |
| `$85` | `ControlDTCSetting` |
| Mode `$04` | Effacement OBD-II normalisé |

Et ⛔ **jamais de balayage d'identifiants de service** — interdit n°3. Balayer
des DID avec `$22` est sans danger ; balayer des services ne l'est pas.

⚠️ **`STPX` est à éviter à cette séance.** C'est la commande d'émission de message
arbitraire du STN, avec en-tête, longueur et répétitions libres. Elle est
puissante et elle n'a aucun garde-fou : la même frappe qui envoie un `$22`
envoie un `$2F`. Les blocs ci-dessous n'en ont pas besoin.

---

## 2. Le protocole

Chaque bloc donne les commandes **dans l'ordre**, ce qu'on relève, et ce que
signifie un échec. Les réponses observées se notent telles quelles, **y compris
les bizarreries** — surtout les bizarreries.

### Bloc A0 — l'adaptateur seul, sur USB, sans véhicule

**Répond au sous-item « alimentation USB » de `07`** (ajout du 2026-08-26), et le répond gratuitement : si l'invite `>`
apparaît, l'EX vit sur l'alimentation USB seule et la moitié de cette séance se
fait au bureau. Sinon, il faut la broche 16 — et un fusible près de la source.

```text
(brancher, observer le message de démarrage)
ATZ
ATE0
STI
STDI
STIX
STMFR
STSN
STVR
STSLCS
```

| Commande | Ce qu'on relève |
|---|---|
| *message de démarrage* | Attendu `ELM327 v1.4b` — voir §0.2, ce n'est pas un clone |
| `ATZ` | Réinitialisation logicielle. **Chronométrer** le temps jusqu'à l'invite : c'est le coût d'une réinitialisation, dont `10` §1 a besoin |
| `ATE0` | Écho coupé. À faire tôt : l'écho double le volume de la trace et brouille le parsing |
| `STI` | **La chaîne d'identité réelle.** `STN2232 v…` ou `STN2120 v…` — tranche §0.1 |
| `STDI`, `STIX`, `STMFR`, `STSN` | Identité matérielle, firmware étendu, fabricant, série. ⚠️ **Le numéro de série est une donnée identifiante** — voir §4 |
| `STVR` | Tension lue. Sur USB seul, elle sera basse ou absente ; au véhicule, c'est la ligne de base du mainteneur de batterie |
| `STSLCS` | **Toute la configuration PowerSave en un coup.** Relever `UART SLEEP`, `VL SLEEP`, `PA SLEEP` et leurs délais. Défauts documentés : déclencheur de veille UART **désactivé**, délai d'inactivité 1200 s. C'est ce qui décide si l'adaptateur peut s'endormir sous un enregistreur de plusieurs heures (`09` §11) |

⛔ **Si les quatre commandes `ST…` répondent `?`, ce n'est pas un STN.** Arrêter
la séance et vérifier le matériel : tout le reste du dossier suppose une puce STN.

### Bloc A — initialisation au véhicule, HS-CAN

```text
ATZ
ATE0
ATS0
ATH1
ATSP0
0100
ATDP
ATDPN
STPR
STPRS
ATRV
```

| Étape | Ce qu'on relève |
|---|---|
| `ATS0` | Espaces coupés. Volume de trace divisé ; **le décodeur doit gérer les deux** |
| `ATH1` | En-têtes **affichés**. ⚠️ Change la grammaire de sortie (§0.4). On les veut pour la séance — savoir *quel module* a répondu est le point — mais chaque bloc doit noter sous quel réglage il tourne |
| `ATSP0` puis `0100` | Détection automatique. La requête est ce qui **déclenche** la recherche ; noter `SEARCHING…` et sa durée |
| `ATDP`, `ATDPN`, `STPR`, `STPRS` | Le protocole retenu, en clair et en numéro, vu par l'AT **et** par le ST. Attendu : ISO 15765 11 bits 500 kbps, soit `33` en préréglage STN |
| `ATRV` | Tension au connecteur. À recouper avec `STVR` |

⚠️ **`ATSP0` est pour cette séance, pas pour le pilote.** La détection
automatique coûte du temps et se conclut par un protocole qu'on connaît déjà.
Une fois le numéro relevé, le pilote posera `STP 33` directement — c'est une
décision de conception que ce bloc alimente.

### Bloc B — premier contact et sonde normalisée

```text
ATSH 7E0
0902
2201 F1 90      ← noter tel quel : voir la remarque ci-dessous
0100
0300
```

**Sonde recommandée par `07` §4bis : `$22 F190`.** Un DID normalisé par
l'ISO 14229, donc réponse prévisible, et il valide d'un coup la chaîne complète
série → UDS → décodage. La forme réellement tapée est `22F190` sans espaces ;
l'écrire espacé ici sert la lisibilité, pas la frappe.

| Étape | Ce qu'on relève |
|---|---|
| `0902` | VIN par le mode OBD-II normalisé. **Le format multi-trames de §0.4 se confirme ici** — longueur en hexadécimal sur sa ligne, puis `n:` par trame |
| `22F190` | Le même VIN par UDS. **Deux chemins vers la même valeur** : si les deux concordent, la chaîne est bonne de bout en bout |
| `0100` | PID supportés — la carte de ce qui est lisible au palier 1 |
| `0300` | DTC stockés. Réponse vide = aucun code, ce qui **n'est pas** une erreur |

⛔ **Les deux premières réponses de ce bloc contiennent le VIN en clair.** À
partir d'ici, la trace est une donnée de couche 4. Voir §4, et **ne pas
`git add`**.

### Bloc C — bornes du réassemblage

C'est le bloc qui mesure ce que `09` §3 laisse en `[NON VÉRIFIÉ]` : *« jusqu'à
quelle taille de réponse le STN2120 assemble correctement »*.

```text
ATAL            ← lever la limite de 7 octets ; voir §0.5
22F190
09 02
(une requête $22 multi-DID, si le module l'accepte : 22 F190 F188 F18C)
ATNL            ← remettre la limite, et refaire la plus longue des requêtes
```

| Ce qu'on cherche | Comment on le lit |
|---|---|
| La plus longue réponse obtenue | Compter les octets de la ligne de longueur. Comparer aux **4 ko** documentés et aux **4095 octets** de l'ISO 15765-2 |
| Le comportement **à la borne** | `OUT OF MEMORY`, `BUFFER FULL`, ou troncature silencieuse ? ⚠️ **La troncature silencieuse est le pire cas** et c'est celui qu'il faut savoir détecter : une réponse tronquée décode en valeurs plausibles et fausses |
| L'effet de `ATNL` | Si la même requête réussit sous `ATAL` et échoue sous `ATNL`, la nécessité de `ATAL` est démontrée, pas supposée. **C'est une mesure à deux points, pas une observation** |
| `$22` multi-DID | Combien de DID par requête le module accepte. Conditionne directement la durée d'un balayage et les volumes de `04` §1.2 |

### Bloc D — commutation de bus, chronométrée, dans les deux sens

**Le bloc le plus important de la séance.** Toute la §1 de `10` en dépend, et le
chiffre n'existe dans aucune documentation.

```text
STPR            ← état de départ, pour la trace
STPC            ← fermer le protocole courant
STP 53          ← MS-CAN, ISO 15765 11 bits 125 kbps
STPO            ← ouvrir
STPR
STPRS
ATSH 726        ← en-tête d'un module de carrosserie, selon le profil (02)
22F190
(retour)
STPC
STP 33
STPO
ATSH 7E0
22F190
```

| Mesure | Méthode |
|---|---|
| **Délai de commutation** | Horodatage de la trace, de l'envoi de `STPC` à la première réponse valide sur le nouveau bus. **Trois fois dans chaque sens** — une mesure unique ne dit pas si c'est stable |
| **Faut-il `STPC`/`STPO` ?** | Essayer aussi `STP 53` seul, sans fermer/ouvrir. Si ça marche, la machine à états est plus simple ; si ça donne `?` ou `CAN ERROR`, la séquence complète est obligatoire |
| **La session UDS survit-elle ?** | Ouvrir une session étendue (`1003`) sur HS-CAN, commuter, revenir, et interroger sans réouvrir. **Attendu : perdue** — le périphérique CAN est remappé (§0.1). À confirmer, parce que c'est ce qui rend la commutation coûteuse |
| **Coût d'un aller-retour** | Le total mesuré est ce qui justifie la règle *« un outil qui touche plusieurs modules ordonne son travail par bus »* de `10` §1. Si c'est 50 ms, la règle est un confort. Si c'est 2 s, c'est structurant |

⚠️ **`ATSH 726` est un exemple, pas une valeur du dossier.** L'identifiant du
module à interroger vient du profil de `02`, et **un fait sur un véhicule
particulier n'entre pas dans le code** (interdit n°8) — il n'entre pas non plus
dans ce protocole comme une constante. Prendre l'identifiant dans le profil.

### Bloc E — les échecs, délibérément

`10` §4 pose que `7F .. 31` est de **l'information**, pas une erreur, et que
65 000 réponses sur 65 536 le seront pendant un balayage. **Le décodeur doit être
développé contre des échecs réels**, donc il faut en capturer.

```text
22 FF FF        ← un DID qui n'existe très probablement pas → 7F 22 31 attendu
22 F1 A0        ← un DID plausible mais non garanti
(interroger un module MS-CAN en restant sur STP 33)   → CAN ERROR / NO DATA attendu
1003            ← session étendue ; noter si elle est accordée ou refusée
```

| Échec provoqué | Ce qu'on veut voir |
|---|---|
| `7F 22 31` requestOutOfRange | **Le format exact de la réponse négative** : trois octets, avec ou sans en-tête selon `ATH`. C'est la ligne que le décodeur verra 65 000 fois |
| Mauvais bus | La distinction entre `CAN ERROR` et `NO DATA` en pratique. Deux causes différentes, deux actions humaines différentes (§0.6) |
| `1003` | Si `7F 10 xx` revient, la frontière du palier est documentée. Si la session est accordée, noter que le chronomètre `$3E` **démarre** |
| Tout `7F` inattendu | Le noter tel quel. `10` §4 a une table de conditions ; une condition qui n'y figure pas est une découverte, pas une anomalie |

### Bloc F — la session longue et les deux chiens de garde

C'est le bloc qu'on ne peut pas raccourcir, parce qu'il **mesure une attente**.
`07` pose la question ainsi : *« Une session interactive dure dix minutes ;
un enregistreur tourne des heures. »*

```text
STPTO 102       ← poser explicitement le défaut, pour que la trace le dise
1003            ← ouvrir une session étendue
22F190          ← confirmer qu'elle répond
(attendre 2 s, 5 s, 10 s, 30 s, 60 s — sans rien envoyer)
22F190          ← après chaque attente
(refaire, en envoyant 3E00 toutes les 2 s pendant l'attente)
STPTO 1000      ← puis un timeout long, et refaire la plus lente des requêtes
```

| Mesure | Ce qu'elle décide |
|---|---|
| **À quelle attente la session tombe** | Le S3 du module, mesuré. `10` §1 en fait *« un chronomètre de `$3E` qui court »* sans en connaître la valeur |
| **Ce que « tomber » produit** | `NO DATA` ? `7F .. 7F`? Une réponse en session par défaut, silencieusement ? ⚠️ **La dernière possibilité est la dangereuse** : un DID de palier 3 qui répond en session par défaut avec une valeur différente ne s'annonce pas |
| **`3E00` suffit-il** | Si oui, le chien de garde de `10` §1 est un mécanisme central et pas un détail. Noter la **période** nécessaire, pas seulement le fait |
| **Effet de `STPTO`** | Un timeout court fait remonter `NO DATA` sur une réponse simplement lente. C'est l'une des quatre causes de `NO DATA` (§0.6), et la seule qu'on se cause soi-même |
| **`ACT ALERT` / `LP ALERT`** | S'ils apparaissent, l'adaptateur a son **propre** minuteur, indépendant du S3 du module. Deux chiens de garde à des échelles différentes, à distinguer dans la conception |

### Bloc G — débit

```text
(boucler 22F190 une centaine de fois, en notant l'horodatage)
(boucler un PID unique : 0105)
(boucler un PID en lot : 01 05 0C 0D 10)
```

Ce qu'on relève : **requêtes par seconde**, à trois régimes. `05` §1.2 situe le
STN2120 à **300–500 trames/s** `[NON VÉRIFIÉ, communauté]` contre 20–50 PID/s
pour un ELM327. Cette mesure remplace une étiquette communautaire par une mesure
— **sur cet appareil et ce véhicule**, ce qui ne se généralise pas.

⚠️ **Ne pas promouvoir la valeur communautaire.** Si la mesure donne 400/s, ça ne
`[VÉRIFIE]` pas la fourchette publiée : ça donne un point mesuré dans cette
fourchette. La distinction est celle du champ `provenance` de `11`.

### Bloc H — clôture

```text
STPC
ATZ
(débrancher)
```

Et **immédiatement** : sauver la trace, l'horodater, remplir la fiche de §5
pendant que la séance est fraîche. ⛔ Ne pas la déplacer hors de la machine.

---

## 3. Le format de trace

La trace n'est pas un souvenir de séance, c'est **un artefact que du code va
consommer** : le transport de rejeu de `10` §9. Ça impose deux exigences que le
défilement d'un terminal ne satisfait pas.

### 3.1 Ce que le rejeu exige

| Exigence | Pourquoi | Conséquence pratique |
|---|---|---|
| **Horodatage à la milliseconde, par ligne** | Le rejeu doit reproduire des **délais**, pas seulement des octets. Le bloc D ne mesure rien sans ça | `screen` seul ne suffit pas |
| **Sens marqué** (envoyé / reçu) | Sans lui, on ne sait pas reconstituer les paires requête-réponse | Un préfixe par ligne |
| **Brut, non nettoyé** | *« Les bizarreries sont précisément ce qui a de la valeur »* (`07` §4bis) | Ne pas corriger les fautes de frappe : les garder, et garder le `?` qu'elles ont produit |
| **Réglages d'adaptateur en cours** | Le format de sortie dépend de `ATH`, `ATS`, `ATCAF`, `ATAL` (§0.4, §0.5). Une trace sans ces réglages est ambiguë | Les journaliser comme des événements, à chaque changement |

### 3.2 Forme retenue

**JSON Lines**, une ligne par événement. Choisi pour trois raisons : ça se lit à
l'œil, ça se filtre à la ligne pendant l'anonymisation (§4), et `serde_json` est
déjà au tableau des dépendances de `09` §3 — aucune dépendance nouvelle.

```json
{"t":0,"dir":"meta","note":"séance 4bis, adaptateur seul sur USB"}
{"t":0,"dir":"cfg","echo":true,"spaces":true,"headers":false,"caf":true,"al":false}
{"t":142,"dir":"tx","raw":"ATZ"}
{"t":1183,"dir":"rx","raw":"ELM327 v1.4b"}
{"t":1184,"dir":"rx","raw":">"}
{"t":2011,"dir":"tx","raw":"22F190"}
{"t":2094,"dir":"rx","raw":"014"}
{"t":2096,"dir":"rx","raw":"0: 62 F1 90 XX XX XX"}
{"t":2412,"dir":"note","note":"commutation MS-CAN : STPC→première réponse = 318 ms"}
```

- `t` — millisecondes depuis le début de la séance. **Relatif**, pas absolu :
  une heure murale est une donnée de contexte en moins à anonymiser.
- `dir` — `tx`, `rx`, `cfg`, `meta`, `note`.
- `raw` — la ligne telle qu'elle est passée, sans interprétation.
- `note` — les observations humaines, dans le fil. Ce sont elles qui rendent la
  trace relisible dans six mois.

⚠️ **Le format n'est pas figé.** Il est proposé ici pour que la séance produise
quelque chose d'exploitable ; le transport de rejeu de `10` §9 est libre de le
raffiner. Ce qui **est** figé, c'est les quatre exigences de §3.1 — une trace qui
en manque une est à refaire, et refaire veut dire retourner au camion.

### 3.3 Emplacement

⛔ `traces/` — **ignoré par git**, avec `*.trace` et `*.serial.log`, déjà en
place dans `.gitignore`. Nommage suggéré :
`traces/2026-08-28-4bis-<bloc>.jsonl`.

---

## 4. Anonymisation — obligatoire avant tout versement

> ⛔ **Interdit n°5, et c'est le plus facile à enfreindre par distraction.**
> Aucune donnée de couche 4 ne quitte la machine ni n'entre dans un journal :
> VIN, As-Built, CALID/CVN, odomètre, historique d'entretien. **Une empreinte
> dérivée est acceptable, la valeur en clair non.** (`docs/11` §4, `docs/12` §4.2)

**Une trace de cette séance contient le VIN, et pas par accident : par
construction.** Le bloc B le lit deux fois, délibérément, parce que `$22 F190`
est la meilleure sonde de premier contact qui existe. C'est le prix de la bonne
sonde.

### 4.1 Ce qu'il faut retirer

| Donnée | Où elle apparaît dans cette séance |
|---|---|
| **VIN** | Mode `0902` (bloc B), `22F190` (blocs B, C, D, F) — **la requête la plus répétée du protocole** |
| **Numéro de série de l'adaptateur** | `STSN` (bloc A0). Identifie le matériel, donc l'acheteur |
| **Identité matérielle** | `STDI`, `STIX` — moins sensibles, mais à relire |
| **CALID / CVN** | Mode `0904`, et les DID de calibration si le bloc E en touche |
| **Odomètre** | Selon les DID interrogés |
| **As-Built** | Hors périmètre de cette séance, mais à surveiller si elle dérive |
| **Horodatage absolu, position** | N'entrent pas dans le format de §3.2, et c'est voulu |

### 4.2 Procédure

1. **Relire la trace en entier, à l'œil.** Pas de `grep` en premier : on ne sait
   pas encore ce qu'on cherche. Une réponse inattendue du bloc E peut contenir
   n'importe quoi.
2. **Remplacer, ne pas supprimer.** Une valeur retirée change les longueurs et
   casse le rejeu. Substituer octet pour octet — `XX`, ou un VIN factice de
   longueur exacte. **La structure est ce qui a de la valeur, pas le contenu.**
3. **Vérifier la ligne de longueur.** Si le remplacement change le nombre
   d'octets, la ligne `014` de §0.4 devient fausse et le décodeur se développera
   contre une incohérence.
4. **Empreinte plutôt que valeur, si le lien importe.** Un hachage tronqué du VIN
   permet de dire « même véhicule » sans dire lequel — c'est ce que `11` §4
   autorise explicitement.
5. **Second passage automatisé, après le passage humain.** Chercher toute
   séquence de 17 caractères alphanumériques, tout ASCII imprimable long dans une
   réponse, le numéro de série relevé au bloc A0. **En second**, parce qu'un
   filtre qui passe donne une fausse assurance.
6. **Verser à la main.** ⛔ **Jamais `git add .`** — nommer le fichier
   explicitement. `traces/` étant ignoré, il faudra un `git add -f`, et **ce
   `-f` est la dernière porte** : s'il faut forcer, c'est le moment de se
   demander pourquoi.
7. **Garder l'original non anonymisé hors de git**, dans `traces/`. Il sert à
   revérifier une mesure sans retourner au camion.

⚠️ **La trace anonymisée et la trace originale ne sont pas le même artefact.**
La première est un jeu de test versionnable ; la seconde est une donnée
personnelle qui reste sur la machine. Ne pas écraser l'une par l'autre.

---

## 5. Fiche de relevé

À remplir pendant la séance, pas après. Les cases vides sont des mesures
manquantes, et une mesure manquante veut dire retourner au camion.

| # | Mesure | Bloc | Valeur | Étiquette |
|---|---|---|---|---|
| 1 | Chaîne `STI` — la puce réelle | A0 | | |
| 2 | L'EX démarre-t-il sur USB seul ? | A0 | | |
| 3 | Configuration PowerSave (`STSLCS`) | A0 | | |
| 4 | Durée d'un `ATZ` jusqu'à l'invite | A0 / A | | |
| 5 | Protocole détecté par `ATSP0`, et durée du `SEARCHING` | A | | |
| 6 | Le VIN concorde-t-il entre `0902` et `22F190` ? | B | | |
| 7 | Format exact d'une réponse multi-trames, sous `ATH1` | B | | |
| 8 | Plus longue réponse réassemblée obtenue | C | | |
| 9 | Comportement à la borne : erreur nommée ou troncature ? | C | | |
| 10 | `ATAL` est-il nécessaire ? (mesure à deux points) | C | | |
| 11 | Nombre de DID par requête `$22` accepté | C | | |
| 12 | **Délai de commutation HS→MS**, 3 mesures | D | | |
| 13 | **Délai de commutation MS→HS**, 3 mesures | D | | |
| 14 | `STPC`/`STPO` obligatoires, ou `STP` seul suffit ? | D | | |
| 15 | La session UDS survit-elle à une commutation ? | D | | |
| 16 | Un module MS-CAN a-t-il besoin de `STCFCPA` ? | D | | |
| 17 | Format exact de `7F 22 31` | E | | |
| 18 | `CAN ERROR` vs `NO DATA` sur mauvais bus | E | | |
| 19 | La session étendue `1003` est-elle accordée ? | E | | |
| 20 | **À quelle attente la session tombe** | F | | |
| 21 | **Ce que produit la chute** — erreur, ou silence ? | F | | |
| 22 | `3E00` suffit-il, et à quelle période ? | F | | |
| 23 | `ACT ALERT` / `LP ALERT` observés ? | F | | |
| 24 | Débit : requêtes/s, PID unique et en lot | G | | |
| 25 | Trace journalisée, aux quatre exigences de §3.1 | toute | | |
| 26 | Trace anonymisée selon §4.2 | après | | |

**L'étiquette est une colonne, pas une formalité.** Une mesure faite une fois est
`[VÉRIFIÉ]` pour cet appareil et ce véhicule — pas pour le matériel en général.
Un comportement supposé d'après le manuel et non observé reste `[NON VÉRIFIÉ]`,
même si le manuel est une source primaire.

---

## 6. Ce que la séance ne peut pas lever

Nommé pour que « 4bis est clos » ne soit pas surinterprété.

- **La généralisation à d'autres adaptateurs.** Un STN2232 mesuré ne dit rien
  d'un STN1170 ni d'un MX+. La conception doit rester à un cran d'abstraction
  au-dessus des chiffres relevés.
- **La généralisation à d'autres véhicules.** Le S3 du bloc F, les identifiants
  du bloc D, les DID du bloc E sont **du véhicule**, et vont en couche 3 du
  modèle de `11` — jamais dans le code (interdit n°8).
- **Le comportement d'une passerelle sécurisée.** `11` §6.1 laisse ouvert si une
  passerelle peut bloquer une **lecture** normalisée. Un véhicule qui n'en a pas
  ne répond pas à la question.
- **Le refus au palier 4.** `7F .. 33 securityAccessDenied` est **attendu** et ne
  se contourne pas (`09` §4). L'observer confirme la frontière, ne la déplace pas.
- **Le comportement en roulant.** Hors périmètre de cette séance, et interdit n°4
  pour `probe_dids`. Certains PID ne bougent qu'en charge : c'est une séance
  distincte, avec un passager qui tient le clavier.
- **La tenue sur des heures.** Le bloc F mesure des dizaines de secondes. Le rôle
  enregistreur embarqué de `09` §11 tourne des heures, et rien ici ne le couvre.

---

## 7. Ce que la séance change en amont

À porter dans les documents concernés **une fois les valeurs relevées**, et pas
avant — c'est la règle de promotion d'étiquette du dossier.

| Document | Ce qui change |
|---|---|
| `05` §1.2, §3.1 | ⛔ **Déjà faux, indépendamment de la séance :** `ATSW`/`ATMSCAN` et le « pilotage du relais » n'existent pas (§0.1), et la puce de l'EX est à revérifier (§0.1). **À corriger sur la foi du FRPM**, sans attendre le camion |
| `09` §3 | Le `[NON VÉRIFIÉ]` sur la taille de réassemblage se remplace par la mesure du bloc C. Et *« ISO-TP délégué au firmware »* gagne ses **quatre conditions** (§0.5) |
| `09` §1 | Rien à rouvrir. Aucune décision verrouillée ne dépend de ces chiffres — c'est ce qui rend la séance sûre à faire |
| `10` §1 | Les `[NON VÉRIFIÉ]` sur le délai de commutation et la réinitialisation de session tombent (bloc D). Le chien de garde `$3E` devient un mécanisme dimensionné (bloc F) |
| `10` §4 | Gagne **l'étage adaptateur** de la taxonomie (§0.6), avec `STOPPED`, `LV RESET`, `CAN ERROR` et `FC RX TIMEOUT` comme conditions nommées et distinctes |
| `10` §9, §10 | Le transport de rejeu a enfin son entrée. Les étapes 1 et 2 de l'ordre de construction deviennent développables sans véhicule |
| `07` §4bis, et le sous-item « alimentation USB » | Se closent — le second gratuitement, par le bloc A0 |
| `11` couche 3 | Reçoit les identifiants de module et les DID confirmés, **en données** |

---

## 8. Résumé en une page

**Ce qu'il faut retenir si on ne lit qu'un encadré :**

1. **La commutation MS-CAN est `STP 53`.** Pas de relais, pas de `ATSW`. Un seul
   périphérique CAN, remappé par logiciel, **un seul bus à la fois**.
2. **`ATST` est déprécié, `STPTO ms` le remplace**, défaut 102 ms.
3. **Quatre défauts d'usine mordent** : `ATNL` limite la réception à 7 octets,
   `STCSEGR` est à 0, les paires de contrôle de flux supposent `RxID − 8`, et une
   paire manuelle coupe l'automatisme pour toutes les autres.
4. **`STI`, pas `ATI`.** L'EX s'annonce `ELM327 v1.4b` par compatibilité.
5. **Les erreurs de l'adaptateur sont une couche à part** des `7F` d'UDS.
   `LV RESET` et `STOPPED` sont les preuves matérielles de deux invariants que le
   dossier posait par prudence.
6. **Journalise tout, horodaté à la milliseconde, sens marqué, réglages inclus.**
   Une trace qui manque une de ces quatre choses est à refaire au camion.
7. ⛔ **La trace contient le VIN.** Anonymiser avant de verser, à l'œil puis au
   filtre, en remplaçant octet pour octet. **Jamais `git add .`**
8. ⛔ **Au terminal, l'encodeur n'existe pas.** L'énumération fermée de services
   est appliquée dans du code qui n'est pas là. **L'humain est le garde-fou** :
   `$11`, `$2F`, `$31`, `$28`, `$14`, `$2E`, `$27`, `$85`, Mode `$04` ne se
   tapent pas.
