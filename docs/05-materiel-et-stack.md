# 05 — Matériel, bibliothèques et contraintes macOS

> ✅ **Revalidé le 2026-08-24.** L'avertissement « partiellement caduc » qui figurait ici reposait sur un malentendu, corrigé en `09-architecture-cross-platform.md`. Ce document est de nouveau valide dans son ensemble. Deux nuances à lire avec lui :
>
> - **Le pessimisme multiplateforme de §2 ne concerne QUE les interfaces CAN brutes.** SocketCAN, `can-utils` et `gs_usb` sont bien Linux-only. Mais un **OBDLink EX est un périphérique USB-série (CDC-ACM)** : `COM3` sur Windows, `/dev/tty.usbmodem*` sur macOS, `/dev/ttyACM0` sur Linux. `pyserial` et le crate Rust `serialport` gèrent les trois sans effort. **Le multiplateforme est facile dès qu'on standardise sur un adaptateur STN en série** — c'est FORScan qui est mono-plateforme, pas le matériel.
> - **Le verdict de §3.2 sur Rust est à nuancer.** Il supposait qu'il fallait implémenter ISO-TP soi-même. Avec un adaptateur STN, **le firmware gère la segmentation multi-trames et le contrôle de flux** ; il ne reste qu'un pilote série et un encodeur UDS. Voir `09` §3.
>
> **§6 (montage minimal) :** l'OBDLink EX est **déjà en main et fonctionnel sur macOS**. Le panda reste optionnel. La VM Windows n'est plus nécessaire pour faire tourner Mecabot — seulement comme oracle de validation ponctuel.

Tu développes sur macOS. C'est la contrainte transversale la plus sous-estimée de tout le projet : **la majeure partie de l'outillage automobile sérieux est Linux-only ou Windows-only.**

---

## 1. Adaptateurs

### 1.1 Le point décisif : la commutation MS-CAN

Rappel de `02-profil-vehicule.md` : la moitié des modules qui t'intéressent (BCM, IPC, APIM, SCCM, portes, climatisation) sont sur **MS-CAN, broches 3/11**, une affectation propriétaire Ford. Un adaptateur qui ne sait pas y basculer ne verra **aucun** de ces modules — pas des données dégradées, zéro réponse.

Il faut une commutation **électronique** (pilotée par logiciel), pas un interrupteur physique, pour permettre un basculement rapide en tranches de temps.

### 1.2 Comparatif

| Adaptateur | Puce | Prix | MS-CAN | macOS | Verdict Transit |
|---|---|---|---|---|---|
| **OBDLink EX (USB)** | STN2120 | 69,95 $US | ✅ **électronique** | ✅ CDC-ACM natif, aucun pilote | **Le bon choix.** Recommandé officiellement par FORScan. |
| OBDLink MX+ (BLE) | STN2120 | ~99-109 $ | ✅ électronique | ⚠️ BLE ajoute 5-15 ms de gigue | Bon pour mobile, moins bon pour développer |
| OBDLink CX (BLE) | STN | 79,95 $ | ❌ **non mentionné** | — | **À éviter.** Ciblé BMW/BimmerCode. Ne pas supposer le MS-CAN. |
| ELS27 | ELM327 modifié | ~40 $ | ✅ mais **interrupteur physique** | ✅ série | Fonctionne, moins pratique |
| vLinker FS | — | ~35-50 $ | ✅ annoncé | ✅ | Alternative économique annoncée compatible FORScan |
| Clone ELM327 générique | ELM327 (ou contrefaçon) | 10-25 $ | ❌ | ✅ série | **Inutilisable** pour ce projet |

Le STN2120 est la puce de génération actuelle d'OBD Solutions, remplaçant l'ELM327 : horloge plus rapide, tampon interne plus grand, commutation multi-bus native. Les chiffres exacts de tampon ne sont pas publiés, mais les retours communautaires situent le débit utile à **~300-500 trames/s** en conditions réelles [NON VÉRIFIÉ], contre 20-50 PID/s pour un ELM327.

> ⚠️ **Deux réserves ajoutées le 2026-08-28, sur le manuel du fabricant** (*OBDLink Family Reference and Programming Manual*, rév. F, 2025-08-29) :
>
> 1. **La puce de l'OBDLink EX actuel est un STN2232, pas un STN2120.** Le FRPM range le **STN2120** parmi les **circuits interpréteurs OBD-vers-UART vendus seuls**, et le donne **obsolète** ; l'EX est listé en STN2230 / STN2231 (hors production) puis **STN2232** (actif). Tout le dossier écrit STN2120 — à corriger partout **une fois relevé sur l'appareil**, ce que `STI` fait en une commande (`13` §0.1). Ça compte, parce que le manuel donne les tailles de tampon **par appareil**.
> 2. **Ce qui est publié est une taille de *message*, pas une taille de tampon** — la phrase ci-dessus reste donc exacte, et la nuance compte : `[VÉRIFIÉ]` **4 ko de taille maximale de message** pour l'OBDLink EX, contre 2 ko pour un SX ou un STN1110. C'est la borne du **réassemblage ISO-TP** (elle recoupe le plafond de 4095 octets du champ de longueur sur 12 bits de l'ISO 15765-2), et **non** la capacité du tampon interne, qui reste `[INTROUVABLE]`. Ne pas dériver l'une de l'autre : un tampon plus petit qu'un message maximal se gère par flux, et rien ne dit lequel des deux borne le débit. La fourchette de **300-500 trames/s** reste `[COMMUNAUTÉ]` — le manuel ne publie pas de débit utile. Le bloc G de `13` la mesure sans la promouvoir.

### 1.3 Pourquoi les clones ELM327 ne conviennent pas

FORScan a une position officielle : « nous ne recommandons plus aucun ELM327 » [VÉRIFIÉ, `forscan.org`].

- **Versions falsifiées** : la puce ELM327 d'Elm Electronics n'a pas été mise à jour depuis plus d'une décennie. Les étiquettes « v1.5 », « v2.0 », « v2.1 » sont des numéros fabriqués.
- **Latence des commandes AT** : chaque requête de PID exige au moins deux allers-retours série (ATSH pour l'en-tête, puis la requête). À 115200 bauds : 3-8 ms de surcharge de cadrage avant toute transaction CAN. À 38400 bauds (défaut de certains clones) : plus de 20 ms.
- **Tampon d'une seule trame.** Sur un HS-CAN chargé, l'ELM327 perd les trames qu'il ne peut pas tamponner.
- **Aucun MS-CAN** : le jeu de commandes AT n'a aucun mécanisme pour basculer le transceiver entre broches 6/14 et 3/11.

Un clone ELM327 permet de lire les Modes 01/02/03/04 sur HS-CAN, et rien d'autre. Ni Mode 22, ni MS-CAN, ni ISO-TP multi-trames fiable.

### 1.4 J2534 : à écarter sur macOS

**J2534 est une spécification de DLL Windows 32 bits.** Il n'existe aucun runtime J2534 sur macOS, à une exception près.

| Appareil | Prix | Note |
|---|---|---|
| Ford VCM II (authentique) | ~1 500-2 000 $ (fin de série) | Clones 60-150 $, compatibilité FDRS peu fiable en 2024+ |
| Ford VCM3 | ~1 200-1 500 $ | Détection de clones par FDRS, succès variable |
| Drew Technologies Mongoose-Plus Ford | ~250-400 $ historiquement | Vendu via Bosch/OTC désormais ; J2534-1 et -2 (canal MS-CAN) |
| Tactrix Openport 2.0 | ~169 $ | **Fournit un runtime compatible macOS** — inhabituel dans ce domaine. Mais pas d'extension J2534-2, donc **HS-CAN seulement** : aucun avantage sur l'OBDLink EX pour du Ford. |
| Clones Godiag / GD801 / FC200 | 30-80 $ | J2534-1 nominal ; support J2534-2 MS-CAN peu fiable |

**Rappel utile de `01` :** FDRS n'exige pas de VCM3 — depuis 2021-2022, tout adaptateur J2534 conforme fonctionne. Mais FDRS lui-même est Windows, et de toute façon hors périmètre puisque le reflashing n'est pas l'objectif.

**Conclusion :** J2534 ne devient pertinent que pour la reprogrammation de modules via FDRS, ce qui exige une VM Windows. Pour tout le reste, OBDLink EX en série suffit.

### 1.5 Interfaces CAN brutes, pour le reverse engineering

Utile en second temps, pour capturer le trafic de diffusion et découvrir des DID.

| Interface | Prix | macOS | Note |
|---|---|---|---|
| **comma.ai panda (Red)** | 99 $ | ✅ **le meilleur sur macOS** — bibliothèque Python via libusb, sans extension noyau | STM32H725, 3 bus CAN + 1 multiplexé, CAN FD. Utilisé par openpilot sur F-150, Mach-E. Pas d'API SocketCAN. Atteindre le MS-CAN exige de câbler les broches 3/11 à la main. |
| **CANable / candleLight** | 20-40 $ (clones 10-15 $) | ⚠️ dépend du firmware | Firmware **candleLight** = pilote noyau Linux `gs_usb`, **hostile à macOS**. Firmware **slcan** = série USB, fonctionne sur macOS via python-can. ~2 000 trames/s théoriques, ~20 % de surcharge d'encodage. |
| PEAK PCAN-USB | ~90-120 € | ✅ pilote macOS propriétaire | SJA1000 + PCA82C251. L'option « professionnelle » la plus mûre sur macOS. Backend `pcan` dans python-can. |
| Kvaser Leaf Light HS v2 | ~300-350 $ | ✅ SDK Kvaser | Excessif pour un usage individuel |

---

## 2. Le mur macOS : SocketCAN

**SocketCAN est un sous-système du noyau Linux** (`net/can`). Mûr, stable, standard sur Linux. Il n'existe pas sur macOS, et rien ne le portera.

Ce qui tombe avec lui :

| Élément | Problème | Contournement |
|---|---|---|
| SocketCAN / `gs_usb` | Pilote noyau Linux | Firmware SLCAN sur CANable, ou API Python de panda |
| `can-utils` (`candump`, `cansend`, `isotpsend`, `isotprecv`) | Linux only, aucun port macOS | Réimplémenter en Python avec python-can + can-isotp |
| Module noyau `can-isotp` | Fonctionnalité du noyau Linux | Mode Python pur de `can-isotp` (`isotp.CanStack`, pas `isotp.socket`) |
| Backend SocketCAN de python-can | Linux only | Backends SLCAN ou PCAN |
| Crate Rust `socketcan` (v3.6.2) | **Explicitement Linux only** | Passer par Python |
| API J2534 | DLL Windows 32 bits | Non nécessaire hors reflashing |
| **FORScan** | Windows en primaire ; un fork Docker+Wine existe mais fragile | **VM Parallels/VMware** pour les sessions de découverte |

Le dernier point mérite d'être digéré : **tu auras besoin d'une VM Windows de toute façon**, pour FORScan. Ce n'est pas un détail d'implémentation, c'est une contrainte de flux de travail. Autant l'assumer tôt.

---

## 3. Bibliothèques

### 3.1 Python — la pile de référence

| Bibliothèque | Version / licence | Rôle | Limites |
|---|---|---|---|
| **python-can** | 4.6.1 (août 2025), MIT, Python ≥3.9 | Abstraction matérielle CAN. Backends : SocketCAN, SLCAN, PCAN, Kvaser, Ixxat, Vector, gs_usb… CAN FD. Journaux ASC, BLF, MF4, TRC, CSV, SQLite. | ❌ **Aucun backend ELM327/STN.** Pas conçu pour les commandes AT. |
| **cantools** | ~2 300 étoiles, actif | Parse DBC, KCD, SYM, ARXML 3/4, CDD. Encode/décode signaux avec multiplexage. Génération de code C. | Outil de codec CAN, **pas de pile diagnostique**. Aucun DBC Ford embarqué. |
| **udsoncan** | MIT, ~430 étoiles | Client UDS ISO 14229 pur Python. 25+ services : $22 ReadDataByIdentifier, $2E, $27 SecurityAccess, $19 ReadDTCInformation, $31 RoutineControl, $14… | Aucune base de DID Ford. **Aucun algorithme seed/key Ford** (cf. `01` §2). |
| **can-isotp** | MIT, v2.x | ISO 15765-2. Deux modes : Python pur (**macOS OK**) ou wrapper du module noyau Linux. | Le mode Python pur suffit pour des sessions de diagnostic ; moins adapté à de la surveillance de charge de bus. |
| **python-OBD** | GPL v2, ~1 300 étoiles | Client ELM327 pour J1979 Modes 01/02/03 | ⚠️ **À ne pas utiliser ici.** ELM327 uniquement, **aucun Mode 22**, et maintenance en panne : `pint` épinglé en 0.20.x alors que l'écosystème est en 0.24.x, 86 tickets ouverts, pas de commit substantiel récent. |

**La pile recommandée :**

```text
pyserial        → contrôle STN de l'OBDLink EX (commutation MS-CAN)
python-can      → abstraction du bus (backend SLCAN ou panda)
can-isotp       → ISO 15765-2, mode Python pur (macOS)
udsoncan        → UDS $22 / $19 par-dessus
cantools        → décodage DBC du trafic de diffusion
ELM327-emulator → tests sans véhicule (Python, ~670 étoiles)
```

**⚠️ Le trou à combler soi-même :** ni python-can ni udsoncan n'ont la moindre notion de commutation de bus au connecteur OBD. Il faudra envoyer la commande STN de sélection de bus via le port série **avant** d'émettre les requêtes ISO-TP, puis router les trames diagnostiques à travers cette même connexion série. **Aucune bibliothèque ne fait ça pour Ford aujourd'hui.** C'est le premier vrai morceau d'ingénierie du projet.

> ⛔ **Corrigé le 2026-08-28 sur le manuel du fabricant.** Ce paragraphe disait *« les commandes STN spécifiques (`ATSW`/`ATMSCAN` et pilotage du relais) »*. **Ces deux commandes n'existent pas, et il n'y a pas de relais.**
>
> `[VÉRIFIÉ]` *OBDLink Family Reference and Programming Manual*, rév. F (2025-08-29), §8.6 : l'appareil n'a **qu'un seul périphérique CAN**, mappé sur différentes broches **par logiciel**, donc **un seul bus actif à la fois**. La commutation est un **changement de préréglage de protocole** : `STP 33` pour HS-CAN (broches 6/14, 500 kbps), **`STP 53` pour le MS-CAN Ford** (broches 3/11, 125 kbps), encadré au besoin de `STPC` / `STPO`.
>
> Conséquence qui n'était pas visible sous l'hypothèse du relais : le périphérique étant **remappé**, toute session UDS ouverte sur l'autre bus est perdue par construction. Détail et protocole de mesure en `13` §0.1 et §2 bloc D.

### 3.2 Rust — pas encore

| Crate | Version | État |
|---|---|---|
| `socketcan` | 3.6.2 | **Linux only, explicitement** |
| `ecu-diagnostics` | 0.107.4, GPL-3.0 | UDS, KWP2000, OBD2 ; API matérielle J2534/SocketCAN/SLCAN/PCAN ; FFI C/C++. ⚠️ **build docs.rs en échec** depuis la 0.101.0. Licence GPL-3.0 contraignante. |
| `OpenVehicleDiag` | 1.0.5 (mai 2021), GPL-3.0 | **Abandonné.** Orienté fichiers CBF Mercedes. |

**Verdict d'origine :** l'écosystème Rust automobile a 3-4 ans de retard sur Python en maturité et couverture. Le crate central est Linux-only. Le document initial recommandait Rust ou C++ « pour éliminer la latence du garbage collector » — argument sans objet ici : ta contrainte est de 2-20 PID/s imposée par le protocole OBD, pas par le langage.

> ❌ **Verdict renversé le 2026-08-24. La conclusion « Python est le bon choix » ne tient plus.** Deux raisons, et aucune ne contredit l'analyse d'écosystème ci-dessus :
>
> 1. **Les crates manquants ne sont pas nécessaires.** Le tableau ci-dessus est pertinent si on implémente ISO-TP soi-même. Avec un adaptateur STN, **le firmware s'en charge** — il ne reste qu'un pilote série (`serialport`, mûr et réellement multiplateforme) et un encodeur UDS pour quatre services. Voir `09` §3.
> 2. **La cible Raspberry Pi Zero embarquée tranche.** Un binaire croisé de quelques mégaoctets, déployé par `scp`, contre un interpréteur plus un environnement virtuel plus des roues natives sur 512 Mo de RAM. Voir `09` §11.5.
>
> **Décision retenue : Rust.** La pile Python de §3.1 reste une référence utile — `udsoncan` documente la sémantique des services, et `ELM327-emulator` sert au prototypage sans véhicule.

### 3.3 opendbc : peu utile

Fichiers Ford présents : `FORD_CADS.dbc`, `FORD_CADS_64.dbc`, `ford_cgea1_2_bodycan_2011.dbc`, `ford_cgea1_2_ptcan_2011.dbc`, `ford_fusion_2018_adas.dbc`, `ford_fusion_2018_pt.dbc`, `ford_lincoln_base_pt.dbc`.

**Aucun Transit.** Les modèles Ford supportés sont Bronco Sport, Escape, Expedition, Explorer, F-150, Focus, Kuga, Maverick, Mach-E, Ranger.

Deux limites de fond :

1. Les fichiers `ford_cgea1_2_*` (plateforme CGEA 1.2 de 2011) sont l'équivalent architectural le plus proche, mais **les adresses de signaux diffèrent**. Utiles comme gabarit de reverse engineering, pas en plug-and-play.
2. Surtout : opendbc couvre le **CAN de diffusion** (messages périodiques : vitesse, angle de volant, papillon, rapport engagé) — ce dont openpilot a besoin pour l'ADAS. Il **ne couvre pas** les sessions de diagnostic (DID UDS, lecture de DTC). Pour la couche qui t'intéresse, c'est hors sujet.

---

## 4. Art antérieur à étudier

**`shoka-jp/obd2-mcp`** (MIT, v0.1) — le plus proche de ton projet.

Serveur MCP exposant des outils de diagnostic OBD-II : DTC, données live (régime, températures, corrections, lambda), freeze frames, monitors de conformité, VIN. Enveloppe le protocole AT série d'ELM327.

Architecture notable : **couche outils séparée de la couche connaissance** (un greffon compagnon `automotive-ai-skills`). Périmètre déclaré explicitement : « lecture seule ; aucune écriture ECU, aucun codage, aucun test d'actionneur ». `clear_dtcs` est la seule opération d'écriture, derrière une confirmation utilisateur explicite.

- **À reprendre :** la frontière d'outils (lecture seule par défaut, portes de confirmation pour les écritures), la déclaration de périmètre nette, la séparation outils/connaissance, le mode simulé pour tester sans matériel.
- **À ne pas reprendre :** la dépendance ELM327, qui le limite au HS-CAN et aux PID standards. Ni Mode 22, ni MS-CAN. Rien de spécifique à Ford.
- **Volume de tokens :** non traité dans la documentation. Les réponses par outil sont compactes (données structurées, pas des trames brutes), mais aucun mécanisme de tranches ou de flux.

**`serial-mcp-server`** (Rust, MIT, v0.1.0, août 2025) — pont série générique. Cinq outils : lister les ports, connecter, envoyer, recevoir, fermer. Encodages UTF-8/hex/binaire. Windows, Linux, macOS.

Aucune pertinence automobile directe : ni OBD, ni gestion AT, ni parsing de protocole. Comme brique de base pour un pilote OBDLink EX (qui est du CDC-ACM), la couche transport MCP est propre. Mais aucun cadrage, aucune logique de reconnexion, aucune machine à états — tout resterait à construire au-dessus.

**Autres projets utiles :**

| Projet | Étoiles | Intérêt |
|---|---|---|
| `ddt4all` | ~1 800 | Diagnostic multi-marques Python/ELM327 avec interface. **Bonne référence pour structurer une base de DID par constructeur.** |
| `ELM327-emulator` | ~670 | Simulateur multi-ECU Python. **Essentiel pour développer sans accès au véhicule.** |
| `odxtools` | ~310 (auteur Mercedes-Benz) | Utilitaires ODX. Montre le bon niveau d'abstraction pour une pile diagnostique sérieuse. |
| `caringcaribou` | Apache 2.0 | Outil de recherche sécurité : scan CAN, énumération UDS, découverte XCP. **Utile pour découvrir des DID inconnus.** ⚠️ Dépôt inaccessible pendant la recherche (404) [INTROUVABLE]. |

**Constat :** aucun serveur MCP dédié à Ford, ni outil de diagnostic Ford intégrant un LLM, n'existe sur GitHub à ce jour. Le créneau est libre.

---

## 5. Outillage de reverse engineering

| Outil | macOS | Rôle |
|---|---|---|
| **FORScan** | Windows primaire ; fork Docker+Wine fragile | Journalisation CSV horodatée de tout le CAN de diffusion. **Le point de départ le plus pratique** pour un inventaire de signaux Transit. Base de PID étendus propriétaire. |
| **SavvyCAN** | ✅ binaires macOS | Capture et reverse engineering Qt/C++. Interfaces : SocketCAN (Linux), Qt SerialBus (PCAN-USB, Vector, J2534), Macchina M2/Teensy, GVRET/ESP32RET. Décodage par DBC, graphes, **scan et décodage UDS**. Compatible Vector ASC, PCAN .trc, Microchip. **L'outil macOS principal**, avec PCAN-USB ou panda. |
| Wireshark | ⚠️ complexe | Dissecteurs SocketCAN pour CAN, CAN FD, CAN XL. Exige un backend de capture ; PCAN-USB avec le pilote PEAK peut alimenter pcap. Utile pour valider le protocole (flux ISO-TP, codes de réponse UDS), pas pour capturer. |
| `can-utils` | ❌ Linux only | Perdus sur macOS, y compris `isotpsend`/`isotprecv` qui sont précisément les plus utiles pour tester une pile UDS contre un vrai ECU. À réimplémenter en Python. |

---

## 6. Montage minimal recommandé

**Matériel — deux appareils, ~170 $US :**

1. **OBDLink EX USB (69,95 $)** — diagnostic et commutation MS-CAN/HS-CAN. Seul adaptateur recommandé officiellement par FORScan qui commute électroniquement ; USB-série natif sur macOS ; STN2120 suffisant pour des sessions UDS ; l'option capable la moins chère. Apparaît en `/dev/tty.usbmodemXXXX`.
2. **comma.ai panda (99 $)** — journalisation CAN brute et reverse engineering. Meilleure expérience macOS (bibliothèque Python multiplateforme), 3 bus, peut surveiller le trafic de diffusion HS-CAN **en parallèle** d'une session diagnostique sur l'OBDLink EX. Alternative économique : CANable en firmware slcan (20-40 $), un seul bus.

**Logiciel :**

- Pile Python de §3.1
- `ELM327-emulator` pour développer sans véhicule
- SavvyCAN pour la capture et le décodage UDS sur macOS
- **VM Windows (Parallels/VMware) pour FORScan** — non négociable, et de toute façon nécessaire comme outil d'écriture (cf. `01` §2.4)

**Ordre d'acquisition suggéré :** OBDLink EX + FORScan gratuit en VM d'abord. Ça coûte 70 $, ça lève la plupart des inconnues de `07-verifications-a-faire.md`, et ça dira si le projet vaut le reste de l'investissement. Le panda et l'abonnement documentaire viennent après.
