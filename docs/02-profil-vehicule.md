# 02 — Profil de référence n°1 : Ford Transit 350HD 2016

> ## 🔄 Reclassé le 2026-08-25 — ce document n'est plus la colonne vertébrale du dossier
>
> Avec le rescope multi-marques (`11`), ce document change de statut sans changer de contenu : il passe de **« le véhicule du projet »** à **« le profil de référence n°1 »**, c'est-à-dire une instance des couches 3 et 4 du modèle de `11` §4 :
>
> | Couche | Ce que ce document en contient |
> |---|---|
> | **3 — modèle/plateforme** | Topologie de bus, inventaire de modules, adresses, grandeurs diesel de la plateforme V363. **Partageable en principe**, sous réserve de provenance (`11` §5). |
> | **4 — exemplaire** | État *deleted*, calibration modifiée, familles de DTC supprimées, rappels applicables à ce VIN, historique. ⛔ **Ne quitte jamais la machine.** |
>
> **Sa valeur augmente plutôt qu'elle ne diminue :** c'est le seul profil réel du dossier, donc le seul sujet contre lequel valider le logiciel. Mais **aucun fait de ce document ne doit se retrouver écrit dans le code** — c'est la règle de `11` §1, et la règle des deux instances de `11` §8 dit qu'il n'y a pas assez d'information ici pour généraliser un schéma.
>
> Une clarification héritée du reclassement, en §6 et ci-dessous : **« un DID invariant n'est pas validé » n'était pas une conséquence du *delete***, c'était une règle générale mal rangée. Elle est promue en cinquième méthode de validation dans `09` §4.
<!-- -->
> ## ✅ Inconnue n°1 levée le 2026-08-24 — et une information qui change beaucoup
>
> **Moteur : 3,2 L Power Stroke I5 diesel** (Duratorq d'origine européenne, injection Bosch). L'item 1 de `07` est clos.
>
> **Et le véhicule est *deleted* : l'après-traitement d'échappement a été retiré.** Ce n'est pas un détail de configuration, c'est le fait le plus structurant du dossier après le choix du moteur, parce qu'il implique nécessairement une **calibration PCM modifiée** — sans reprogrammation, le PCM lèverait immédiatement les défauts d'après-traitement et passerait en mode dégradé. Le véhicule roule donc sur un logiciel qui n'est pas celui de Ford.
>
> ### Ce que ça invalide dans ce document
>
> - **§6 et §7 : les grandeurs DPF, EGT et DEF sont vraisemblablement mortes.** Les capteurs peuvent être physiquement absents (pression différentielle DPF, sondes EGT), et les DID correspondants peuvent avoir été retirés de la calibration, renvoyer une valeur figée, ou renvoyer du bruit. **Un DID qui lit un capteur absent lit n'importe quoi — il ne faut surtout pas construire une ligne de base dessus.** Ces lignes sont marquées ci-dessous.
> - **§8 : la régénération DPF forcée sort du périmètre, définitivement.** C'était le risque le plus élevé de tout le diagnostic diesel de cette plateforme. Il disparaît.
> - **§1 : la contradiction DEF/SCR devient sans objet.** S'il y en avait, c'est parti. Le **bridage à ~8 km/h**, l'élément de sûreté le plus inquiétant du dossier, ne s'applique plus.
> - **§4 priorité 2 : encrassement EGR, fissure de refroidisseur EGR et sondes EGT** ne sont pertinents que si l'EGR a été conservé — à vérifier, le delete d'EGR est fréquent mais pas systématique.
>
> ### ⚠️ Ce que ça introduit comme risque diagnostique, et qui est plus sournois
>
> **Une calibration de delete supprime activement des familles de DTC** (P24xx, P20xx, P046x…). Conséquence directe sur la conception de Mecabot :
>
> **L'absence de code dans ces familles ne prouve rien sur ce véhicule.** Le silence peut signifier « tout va bien » ou « on a demandé au calculateur de se taire ». `read_all_dtc` doit le dire explicitement plutôt que de laisser l'agent conclure à l'absence de problème. C'est un cas de figure où un outil de diagnostic naïf devient activement trompeur.
>
> ### Ce que ça n'enlève pas — et rend même plus important
>
> Le delete ne touche **ni le circuit d'alimentation haute pression, ni le turbo, ni la transmission**. Or c'est là que sont les pannes coûteuses :
>
> - **Le rappel 16V618000** (débris métalliques de pompe HP détruisant les injecteurs) est spécifique au diesel : il s'applique donc bel et bien, et le delete n'y change rien.
> - **Les corrections de débit par injecteur deviennent le signal longitudinal n°1 du projet.** Elles étaient déjà en haut de la liste ; elles y sont maintenant seules.
> - Une calibration de delete **augmente généralement le débit de carburant**, ce qui charge davantage injecteurs, pompe HP, turbo et 6R80. La surveillance de ces quatre éléments a donc *plus* de valeur qu'en configuration d'origine, pas moins.
>
> ### Deux sondes nouvelles, gratuites et normalisées
>
> Ceci s'ajoute au palier 1 de `09` et ne demande **aucune donnée propriétaire** :
>
> | Sonde | Ce qu'elle donne |
> |---|---|
> | **Mode `$09` InfoType `04` — CALID** | L'identifiant de calibration du PCM. Dit quel logiciel tourne. |
> | **Mode `$09` InfoType `06` — CVN** | *Calibration Verification Number* : une somme de contrôle de la calibration. **Son rôle réglementaire est précisément de détecter une modification de programmation.** |
> | **Mode `$01` PID `01`** | Bits de support des moniteurs. Sur un véhicule tuné, les moniteurs d'après-traitement désactivés se voient. |
> | **Mode `$06`** | Résultats des moniteurs non continus. **Empreinte de ce que la calibration a réellement désactivé.** |
>
> Le CVN et le CALID sont à **relever et archiver une fois pour toutes** : ils constituent la ligne de base qui te dira si la calibration change un jour — par exemple si un atelier remet du logiciel d'origine. Mode 06 est la façon la moins coûteuse de cartographier ce que le tune a coupé, et c'est de l'OBD-II normalisé.
>
> ### Réserve réglementaire, dite une fois
>
> Le retrait d'un dispositif antipollution contrevient à la réglementation canadienne sur les émissions des véhicules routiers (LCPE 1999). Ça peut aussi avoir des conséquences à l'inspection, à la revente ou au changement d'immatriculation, et le 350HD est susceptible d'être immatriculé commercialement — **[NON VÉRIFIÉ]** sur ce que le régime québécois impose concrètement pour un véhicule de cette classe. C'est ton véhicule et ta décision ; c'est mentionné parce que c'est un facteur réel, pas pour y revenir. **Lire les données de diagnostic de son propre camion reste entièrement légitime**, et rien dans ce dossier ne change à cause de ça.

---

## 1. Motorisations possibles — **résolu : le 3,2 L diesel**

Le 350HD est la variante lourde à roues arrière jumelées (Classe 3, GVWR jusqu'à ~14 500 lb). Les trois motorisations étaient disponibles sur cette plateforme ; **la tienne est la troisième**. Les deux premières lignes sont conservées à titre de contraste.

| Moteur | Type | Puissance | Conséquences diagnostiques |
|---|---|---|---|
| 3,7 L Ti-VCT V6 | essence atmosphérique, injection indirecte | 275 ch / 260 lb-pi | PID OBD-II standards. Point faible documenté : corps de papillon. Aucune complexité d'après-traitement. |
| 3,5 L EcoBoost V6 | essence, bi-turbo, injection directe | 310 ch / 400 lb-pi | Ajoute PID de suralimentation, wastegate, intercooler. Risque de condensat dans l'air de suralimentation en décélération. Consommation de liquide de refroidissement / vase de dégazage connus sur la famille EcoBoost. |
| ✅ **3,2 L Power Stroke I5** | diesel turbo, rampe commune | 185 ch / 350 lb-pi | **Le moteur de ce véhicule.** À l'origine : DPF, EGR refroidi par eau, 4 sondes EGT, turbo à géométrie variable à actionneur électrique, 5 bougies de préchauffage, injection haute pression (~1 800 bar). ⚠️ **En configuration *deleted*, l'après-traitement n'est plus là et la calibration est modifiée** — voir l'encadré en tête de document. Restent le turbo VGT, la rampe commune, les bougies, et l'EGR *si* conservé. |

Le 3,2 L est un **Duratorq d'origine européenne**, rebadgé Power Stroke en Amérique du Nord. Injection Bosch. Ce n'est pas un moteur Navistar/International — cette relation s'est terminée avec les V8 6.0 L et 6.7 L des Super Duty. Conséquence pratique utile : **les forums Transit européens ont une connaissance bien plus profonde de ce moteur que les forums nord-américains**, parce qu'ils vivent avec depuis plus longtemps.

~~⚠️ **Contradiction non résolue**~~ — **devenue sans objet le 2026-08-24.** Deux volets de recherche affirmaient que le 3,2 L nord-américain comportait un système SCR à urée (DEF/AdBlue) en plus du DPF, sans qu'aucune source primaire le confirme [NON VÉRIFIÉ]. **Le véhicule étant *deleted*, la question ne se pose plus** : s'il y avait du SCR, il n'y en a plus. Le corollaire important est que la **stratégie de bridage progressif jusqu'à ~8 km/h** — l'élément de sûreté le plus inquiétant du dossier — ne s'applique pas à ce véhicule.

**Transmission :** 6R80 SelectShift 6 rapports sur toutes les combinaisons.

⚠️ **Deuxième contradiction** : sur le 6R80, la commande de transmission est-elle intégrée au PCM ou dans un TCM séparé ? Deux volets affirment l'intégration au PCM, un troisième liste un TCM distinct sur HS-CAN. À vérifier par un scan de modules — ça détermine s'il faut adresser un ECU supplémentaire.

---

## 2. Topologie réseau

Deux bus accessibles au connecteur OBD-II :

| Réseau | Débit | Broches OBD | Rôle |
|---|---|---|---|
| **HS-CAN** | 500 kbit/s | 6 (CAN-H) / 14 (CAN-L) | Groupe motopropulseur et sécurité. Broches normalisées SAE J1962. |
| **MS-CAN** | 125 kbit/s | 3 (CAN-H) / 11 (CAN-L) | Carrosserie, confort, infodivertissement. **Affectation de broches propriétaire Ford.** |
| LIN | 10,4-20 kbit/s | absent du connecteur OBD | Sous-bus interne pour capteurs et actionneurs terminaux |

Une passerelle interne relie HS-CAN et MS-CAN. Son rôle est le **routage**, pas le filtrage sécurité (voir `01`).

**Conséquence matérielle décisive :** un adaptateur ELM327 générique ne voit que HS-CAN et **manquera tous les modules MS-CAN**. Il faut un commutateur MS-CAN/HS-CAN **électronique** (piloté par logiciel), pas un interrupteur physique. Voir `05-materiel-et-stack.md`.

### Répartition des modules

Sur **HS-CAN** :

| Module | Nom | Note |
|---|---|---|
| PCM | Powertrain Control Module | Moteur + transmission (+ logique DPF/EGR/SCR sur diesel) |
| TCM | Transmission Control Module | ⚠️ existence séparée à confirmer |
| ABS/HCU | Antiblocage / groupe hydraulique | Contrôle de traction et stabilité (RSC) intégrés |
| RCM | Restraint Control Module | Airbags, prétensionneurs. Concerné par le rappel 16V188000. |
| PSCM | Power Steering Control Module | Selon configuration de direction |
| SOBDM | Secondary OBD Module (parfois « BCM II ») | Interface OBD / surveillance émissions |
| TPMS | Pression des pneus | |

Sur **MS-CAN** :

| Module | Nom | Note |
|---|---|---|
| BCM / GEM | Body Control Module | Éclairage, vitres, verrouillage, relais |
| IPC | Instrument Panel Cluster | Compteurs, centre de messages, odomètre |
| APIM | Accessory Protocol Interface Module | SYNC. Nombreux TSB logiciels. |
| ACM | Audio Control Module | Amplificateur, routage audio |
| SCCM | Steering Column Control Module | |
| FCIM | Front Controls Interface Module | Commandes climatisation/audio |
| DDM / PDM | Modules de porte conducteur / passager | Parfois sur LIN |
| HVAC / DATC | Climatisation | Parfois sur LIN |

Scan multi-modules réellement observé sur Transit : PCM, BCM II, ABS, APIM, SASM, RCM, PAM, HCM, IPC [COMMUNAUTÉ, forum FORScan t=28335].

**Note diesel :** sur le 3,2 L nord-américain, il n'y a probablement **pas de module d'après-traitement séparé** — la logique DPF/EGR réside dans le PCM. Cela diffère du 6.7 L Power Stroke des F-Series qui a des modules dédiés. Tous les PID DPF s'adressent donc au PCM. [NON VÉRIFIÉ, à confirmer par scan]

⚠️ La cartographie module→bus ci-dessus est en partie [NON VÉRIFIÉ]. La source autoritaire est le diagramme de topologie réseau du manuel d'atelier 2016, derrière l'abonnement Motorcraft. Le scan de modules FORScan la confirmera empiriquement.

---

## 3. Protocole

**Le Transit 2016 parle UDS (ISO 14229) sur CAN, partout.** Pas de SCP, pas d'UBP, pas d'ISO 9141 — ces protocoles Ford legacy ne sont pas présents sur cette génération [VÉRIFIÉ, `forscan.org` liste les protocoles supportés].

Deux couches à distinguer :

- **Mode $01** : PID standards mandatés par la réglementation, sur HS-CAN via ISO 15765.
- **UDS $22 ReadDataByIdentifier** : DID propriétaires Ford. C'est ce qui expose les températures de transmission, les vitesses de roue individuelles, les pressions TPMS par roue, l'état des modules de carrosserie, et tous les PID diesel intéressants.

Les numéros de DID sont internes à Ford. La base de PID non standard est détenue par l'Equipment and Tool Institute et se licencie à environ **50 000 $US/an** [VÉRIFIÉ, référencé dans la documentation publique OBD-II]. C'est précisément ce qui rend les données live de FORScan incomparablement plus riches que celles de n'importe quel lecteur générique — et c'est la barrière que Mecabot ne franchira pas seul.

---

## 4. Pannes réelles, par fréquence

Sources : API rappels et plaintes NHTSA [VÉRIFIÉ — 21 rappels, 153 plaintes pour Transit 2016], CarComplaints.com [VÉRIFIÉ — 190 plaintes documentées].

### Priorité 1

**1. Rupture de l'accouplement flexible d'arbre de transmission** — 45 plaintes NHTSA, la catégorie la plus élevée. Deux rappels : **17V408000** et **19V767000**.

L'accouplement polymère entre sortie de boîte et arbre de transmission se désagrège, souvent avant 160 000 km. Rupture **sans préavis et sans DTC**. Dommages collatéraux fréquemment rapportés : rupture de conduites de frein et de carburant. Dans le scénario du 19V767000, un arbre séparé peut laisser le véhicule rouler (frein de stationnement sur l'essieu, pas sur la transmission).

Signe précoce : vibration ou broutement de transmission à basse vitesse. Détection : inspection sur pont, recherche de débris de caoutchouc dans le tunnel. **Aucun capteur ne le voit venir** — c'est un cas où Mecabot ne peut rien, et doit le dire.

**2. Débris métalliques dans le circuit d'alimentation (diesel)** — rappel **16V618000**.

La pompe haute pression de production précoce génère des débris métalliques qui contaminent injecteurs, vanne de dosage et rampe. Conséquences : démarrage difficile, fonctionnement irrégulier, panne d'injecteur, calage en roulant. Principalement les dates de production 2015 et début 2016, mais des narratifs de plaintes décrivent des défaillances identiques sur des véhicules hors de la fenêtre du rappel.

Indicateur clé : métal dans le filtre à carburant. Un injecteur défaillant sur un seul cylindre sans autre cause doit déclencher une inspection de la pompe. DTC possibles : P0088 (pression rampe haute), P0193, associés à des codes de ratés.

**3. Usure prématurée des freins arrière** — plainte n° 1 sur CarComplaints. Défaillance moyenne à ~53 000 km, coût moyen ~900 $. Confirmée sur 3,2 L diesel et 3,7 L V6.

Les freins arrière atteignent le métal alors que les avant sont encore bons. Cause probable : répartition arrière ou étriers qui collent, aggravé par les charges d'essieu d'un 350HD à roues jumelées. Vérifier les colonnettes d'étrier arrière à chaque service. Piste de détection par les données : asymétrie des PID de vitesse de roue.

**4. Défaillance du corps de papillon (3,7 L / EcoBoost)** — défaillance moyenne à ~7 600 km sur les cas documentés. Perte totale de puissance sur autoroute, sans préavis. DTC P2104 (ralenti forcé) ou P0120/P0121.

### Priorité 2 (diesel, essentiellement)

| Panne | Symptômes | DTC | Détection anticipée |
|---|---|---|---|
| **Encrassement vanne EGR** | Perte de puissance, fumée au démarrage, mode dégradé | P0401, P0404 | Écart croissant entre position EGR commandée et réelle — **avant** que le DTC ne tombe |
| **Fissure refroidisseur EGR** | Fumée blanche en charge, perte de liquide sans fuite visible | — | Niveau de liquide + pression circuit |
| **Sonde EGT défaillante** | Régénérations qui avortent ou ne démarrent pas | P2032/P2033, P242F, P244A/B | Comparaison des 4 EGT entre elles et aux valeurs attendues |
| **Bougies de préchauffage** | Démarrage difficile sous ~4 °C | P0671-P0675 | Test d'intensité à la pince (15-25 A par bougie à froid) |
| **Aubes de turbo VGT collées** | Surpression ou sous-pression | P0234, P0299, P003A | Erreur de position d'aube soutenue à la mise en charge |
| **Système DEF** (si présent) | Bridage progressif jusqu'à ~8 km/h | P20EE, P2BAD, P2002 | Taux de consommation DEF vs kilométrage |
| **SYNC / APIM** | Écran figé, redémarrages, caméra de recul | — | Rappel **25V572000** (affichage caméra déformé/inversé) |
| **Corrosion faisceau module d'attelage** | Courts-circuits, risque d'incendie | — | Rappels **17V668000** et **18V275000** |

⚠️ Les DTC et seuils de cette section priorité 2 sont largement **[NON VÉRIFIÉ]** — issus de connaissances générales cohérentes avec la plateforme, pas de sources primaires consultées. À valider contre le manuel d'atelier.

---

## 5. Rappels applicables

| ID NHTSA | Composant | Pertinence diagnostique |
|---|---|---|
| **16V618000** | Circuit carburant diesel | Élevée — inspecter le filtre pour particules métalliques |
| **17V408000** | Arbre de transmission | Élevée — accouplement flexible |
| **19V767000** | Arbre de transmission | Élevée — supersède le 17V, ajoute le risque de roulement libre |
| 17V668000 | Faisceau module d'attelage | Modérée — risque d'incendie |
| 18V275000 | Faisceau module d'attelage | Deuxième campagne, même défaut |
| 16V188000 | Airbags rideaux | Sécurité — pliage incorrect, **peut ne pas générer de DTC** |
| 16V111000 | Ceintures arrière | Sécurité — inspection physique |
| 21V631000 | Câble de frein de stationnement | Sécurité |
| 25V572000 | Caméra de recul | Affichage déformé, inversé ou noir |

[VÉRIFIÉ — API rappels NHTSA interrogée]

**Environ 331 TSB** existent pour le Transit 2016. Titres et numéros consultables sur nhtsa.gov ; texte intégral uniquement via abonnement (voir `01`).

---

## 6. PID diesel d'intérêt

Liste des PID que la communauté rapporte comme accessibles sur le PCM via FORScan pour le 3,2 L. ⚠️ Les **noms exacts** sont [NON VÉRIFIÉ] — ils viennent de connaissances générales, pas d'une capture réelle. Les PID DPF confirmés le sont sur Transit 2.2 Duratorq **européen** (`DPF_LOAD`, `DPF_DP`, `CATEMP12`, `EGT12`) [COMMUNAUTÉ, t=6089], pas sur le 3,2 L nord-américain.

⚠️ **Colonne « statut delete » ajoutée le 2026-08-24.** Le véhicule étant *deleted* avec calibration modifiée, une bonne partie de cette liste ne s'applique plus. **Une grandeur marquée ⛔ ne doit pas servir de ligne de base** : soit le DID a disparu de la calibration, soit il lit un capteur physiquement absent, soit il renvoie une valeur figée. Les trois cas se ressemblent et aucun n'est exploitable.

| Grandeur | Unité | Statut delete | Valeur diagnostique |
|---|---|---|---|
| Correction de débit par injecteur (×5) | mm³/coup | ✅ **intact — priorité n°1** | Dérive individuelle = usure ou obstruction d'injecteur. Le meilleur signal du projet, et le seul qui recoupe le rappel 16V618000. |
| Pression rampe réelle vs demandée | bar | ✅ intact | Ralenti ~350-400, croisière ~600-900, pleine charge ~1 600-1 800 |
| Position aubes turbo réelle vs demandée | % | ✅ intact | Aubes collées = erreur soutenue à la mise en charge |
| Durée de vie d'huile (IOLM) | % | ✅ intact | Déclencheur d'entretien direct |
| Position EGR réelle vs demandée | % | ⚠️ **à vérifier** | Pertinent **seulement si l'EGR a été conservé**. Le delete d'EGR est fréquent mais pas systématique. À trancher physiquement. |
| EGT 1 à 4 | °C | ⛔ probablement mort | Sondes possiblement retirées avec la ligne d'échappement |
| Pression différentielle DPF | kPa | ⛔ mort | Capteur retiré avec le DPF |
| Charge de suie estimée | g/L | ⛔ mort | Plus de DPF à charger |
| Charge de cendres estimée | g/L | ⛔ mort | Idem |
| État / compteur de régénérations | drapeau / n | ⛔ mort | Logique de régénération supprimée par la calibration |
| Niveau et dosage DEF | % | ⛔ sans objet | Voir §1 |

**Le déplacement de cible à retenir :** le palier 3 de `09` visait principalement le DPF et les EGT. Il visera désormais **le circuit haute pression, le turbo et la 6R80** — moins de grandeurs, mais celles qui coûtent cher quand elles lâchent, et que la calibration modifiée sollicite davantage.

**Où trouver les vraies listes :** forum FORScan (section Transit, inscription gratuite requise, accès direct par URL restreint), `fordtransitusaforum.com` (sous-forum 3.2L I5 Turbodiesel, paywall anti-robot mais accessible en navigateur), et les forums Transit européens pour le Duratorq de base.

---

## 7. Entretien et ce qu'un suivi longitudinal peut apporter

Intervalles (3,2 L diesel) — ⚠️ [NON VÉRIFIÉ], à confirmer au manuel :

| Élément | Intervalle |
|---|---|
| Huile moteur | selon IOLM, max ~16 000 km. Spec Ford WSS-M2C171-F |
| **Filtre à carburant / séparateur d'eau** | ~48 000 km — **critique** : l'eau détruit injecteurs et pompe |
| Service cendres DPF | ~240 000 km, dépend fortement du cycle d'usage |
| Liquide de refroidissement | ~160 000 km — plus critique que sur essence à cause de l'EGR refroidi |
| Fluide 6R80 | ~240 000 km normal, ~96 000 km usage sévère (Mercon LV). Le start-stop commercial est un usage sévère. |
| Liquide de frein | 2 ans / 48 000 km |
| Huile de pont arrière | ~96 000 km usage sévère ; sur 350HD à roues jumelées, inspecter à chaque vidange (fuites de joints documentées) |

### Ce que le suivi dans le temps permet réellement

C'est ici, et pas dans la détection temps réel, que la valeur du projet se trouve. Aucune de ces grandeurs n'est visible dans une session FORScan isolée :

Table révisée le 2026-08-24 pour la configuration *deleted* :

| Signal suivi | Statut | Ce qu'une tendance révèle |
|---|---|---|
| **Corrections par injecteur** | ✅ | Une valeur qui s'écarte des quatre autres = injecteur en cause identifié **avant** le raté. **Le signal porteur du projet.** |
| Pression rampe réelle vs demandée, par casier de charge | ✅ | Pompe HP qui faiblit, régulateur, restriction d'alimentation |
| Réponse des aubes de turbo à la mise en charge | ✅ | Collage naissant |
| Températures de transmission 6R80 | ✅ | Sollicitée davantage par une calibration qui augmente le couple |
| Consommation calculée | ✅ | Déclin lent = dérive d'injecteurs, turbo, restriction d'admission |
| Delta température de démarrage vs ambiante | ✅ | Santé du thermostat |
| Tension batterie au démarrage | ✅ | État de santé, ligne de base — et **d'autant plus utile avec un Pi embarqué** (`09` §11.6) |
| Écart EGR commandé/réel | ⚠️ | Seulement si l'EGR est conservé |
| ~~Charge de suie DPF~~ | ⛔ | Plus de DPF |
| ~~Fréquence des régénérations~~ | ⛔ | Logique supprimée |
| ~~Consommation DEF vs kilométrage~~ | ⛔ | Sans objet |
| **CVN / CALID du PCM** | ✅ **nouveau** | Ne bouge jamais en usage normal. Un changement signifie que la calibration a été reprogrammée. À archiver comme ligne de base (voir l'encadré en tête). |

---

## 8. Éléments critiques pour la sécurité

À traiter comme tels dans toute conception ultérieure :

- ~~**Régénération DPF forcée**~~ — **hors périmètre depuis le 2026-08-24 : plus de DPF.** Conservé pour mémoire, parce que c'était le risque le plus élevé de tout le diagnostic diesel de cette plateforme (550-700 °C à l'échappement, extérieur obligatoire, jamais si une dilution d'huile par le carburant est suspectée) et parce que la contradiction non résolue sur son support par FORScan occupait une place importante dans le dossier. **Les deux disparaissent d'un coup** : l'item 6 de `07` n'a plus besoin d'être tranché.
- **Retrait de bougies de préchauffage** — rupture connue au démontage. Préchauffer, dégrippant, temps de pénétration long au-delà de 100 000 km. Suivre la procédure du TSB Ford.
- **Accouplement d'arbre de transmission** — rupture sans préavis, rupture de conduite de frein documentée après coup. Vérifier le statut des rappels avant de compter sur le véhicule pour du transport critique.
- ~~**Bridage DEF/SCR**~~ — **sans objet** (§1). C'était l'élément le plus inquiétant de la liste : un véhicule bridé à ~8 km/h dans la circulation, sans dérogation. Il ne s'applique pas à ce véhicule.
- ⚠️ **Nouveau — DTC supprimés par la calibration.** Sur un véhicule tuné, l'absence de code dans les familles d'après-traitement ne prouve rien. Ce n'est pas un risque mécanique, c'est un **risque d'erreur de raisonnement**, et il est réel dès qu'un agent conclut « aucun défaut » à partir d'un scan silencieux. Voir l'encadré en tête de document.
- **Airbags (16V188000)** — le défaut peut ne générer aucun DTC. Inspection physique uniquement. Un système qui ne rapporte « aucun défaut » ne dit rien sur ce rappel.
