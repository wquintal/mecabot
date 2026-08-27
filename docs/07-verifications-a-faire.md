# 07 — Vérifications à faire, par ordre de priorité

Tout ce dossier repose sur des inconnues qui se lèvent en grande partie sans dépenser un dollar. Cette liste est ordonnée par **ce qui débloque le plus de décisions par minute investie**.

Les items 1 à 4 sont gratuits et prennent moins d'une heure au total. Ils devraient être faits avant toute autre chose.

---

> ## 🔄 Reclassement du 2026-08-25 — un seul item bloque encore le code
>
> Le rescope multi-marques (`11`) change le **statut** de la plupart de ces items, sans les rendre inutiles.
>
> | Catégorie | Nouveau statut |
> |---|---|
> | **Items sur le véhicule** (moteur, rappels, calibration, PID diesel, configuration actuelle) | 🅿️ **Garés, pas abandonnés.** Ce sont désormais des tâches de **remplissage du profil de référence n°1** (couches 3 et 4 de `11` §4), pas des préalables d'architecture. Le logiciel se conçoit et se code sans leurs réponses. |
> | **Item 4bis — séance au terminal série** | 🔴 **Le seul véritable préalable à l'écriture de code**, et il est **indépendant du véhicule** : il caractérise l'**adaptateur STN**, pas le camion. Il produit en plus le premier jeu de test du projet (`10` §9). |
> | **Item 4ter — recherche ODX + PID communautaires** | ⚠️ **Partiellement avancé, et sa valeur augmente.** L'outillage ODX est vérifié (`11` §3 : `odxtools` MIT, 11.5.4) mais **aucune source de données ouverte n'existe**. Une obligation réglementaire européenne vaudrait pour **tous** les constructeurs, plus seulement Ford — donc la piste vaut plus cher qu'avant. |
> | **Item 15 — Pi embarqué** | ⏬ **Descendu en fin de parcours** par décision du 2026-08-25. Reste valide. |
>
> **Nouveaux items apportés par le rescope :** vérifier si une passerelle sécurisée bloque réellement la **lecture** normalisée ou seulement les fonctions protégées (`11` §6.1 — [NON VÉRIFIÉ], et c'est ce qui décide si le multi-marques a un vrai plafond) ; et **initialiser un dépôt git dans ce répertoire**, qui n'en est pas un.

---

> ## 🔧 Ajout du 2026-08-26 — les standards de développement remontent la valeur de l'item 4bis
>
> `12` adopte une partie des standards punt-kit, dont la porte de démonstration de `workflow.md` : *« the feature must be driven through its real entry point… demonstrated to the operator before the PR opens »*.
>
> **Le vrai point d'entrée de Mecabot est un camion.** Franchir cette porte à chaque changement voudrait dire sortir le véhicule à chaque fois — contact mis, mainteneur de batterie branché, du temps. Pour `probe_dids`, une heure de stationnement par démonstration.
>
> **→ L'item 4bis n'est donc plus seulement le préalable au code, c'est aussi le préalable au processus.** Sans trace enregistrée, le transport de rejeu de `10` §9 n'existe pas, et la porte de démonstration devient impayable. C'est le troisième motif indépendant qui pointe vers la même séance de 45 minutes.
>
> **Deux items non bloquants ajoutés**, à faire avant la première ligne de code et sans le véhicule :
>
> - **Vérifier si l'OBDLink EX se contente de l'alimentation USB** ou s'il exige la broche 16 du J1962. ⚠️ [NON VÉRIFIÉ] — test de trente secondes, et la réponse décide si l'item 4bis a une moitié faisable au bureau, adaptateur branché au Mac seul.
> - ~~**Mettre en place `docs.yml` + markdownlint**~~ ✅ **Clos le 2026-08-26.** Fait sur les **seize fichiers Markdown suivis** — les treize documents, plus `README.md`, `CHANGELOG.md` et `CLAUDE.md`, le lint tournant sur `**/*.md`. Dossier propre. L'ajustement de règles anticipé ici n'a pas eu lieu pour la raison prévue : **une seule règle a été désactivée en réaction à la mesure** (`MD060`, cosmétique — `MD013` et `MD036` l'étaient d'avance, comme choix de rédaction), et **aucun des vingt signalements restants ne portait sur le français**. Détail mesuré en `12` §7.

---

> ## ✅ Mise à jour 2026-08-24 — les items 1, 2 et 6 sont clos
>
> **Moteur : 3,2 L Power Stroke I5 diesel. Et le véhicule est *deleted*** — après-traitement retiré, donc calibration PCM modifiée. Voir l'encadré en tête de `02`.
>
> | Item | Nouveau statut |
> |---|---|
> | **1 — quel moteur** | ✅ **Résolu.** Diesel 3,2 L. |
> | **2 — DEF/SCR présent** | ✅ **Sans objet.** S'il y en avait, c'est parti avec le reste. Le bridage à ~8 km/h ne s'applique pas. |
> | **6 — menu Service FORScan / régénération DPF** | ✅ **Abandonné.** Plus de DPF : la contradiction la plus nette du dossier n'a plus besoin d'être tranchée. |
> | **7 — inventaire des PID** | ⚠️ **Change de nature.** La cible n'est plus le DPF mais le circuit haute pression, le turbo et la 6R80. Voir `02` §6 révisé. |
> | **16 — caractériser la calibration** | 🆕 **Ajouté ci-dessous.** Nouvelle priorité, gratuite et normalisée. |
>
> Les items 3 et 4 restent entiers et non faits. **Le 16V618000 s'applique bel et bien** puisqu'il est spécifique au diesel — et le delete n'y change rien, il ne touche pas le circuit d'alimentation.

---

## Niveau 0 — Gratuit, immédiat, débloque presque tout

### 1. ~~Quel moteur ?~~ — ✅ **RÉSOLU : 3,2 L Power Stroke I5 diesel, deleted**

*Conservé pour la trace du raisonnement. La méthode ci-dessous reste valide si tu veux confirmer par le VIN — et elle a un intérêt résiduel : `vPIC` donne aussi la classe GVWR, l'empattement et l'usine, utiles comme référence documentaire.*

**Pourquoi ça bloque :** les trois motorisations n'ont rien en commun diagnostiquement. Le diesel ajoute DPF, EGR refroidi par eau, quatre sondes EGT, turbo à géométrie variable, injection à ~1 800 bar, cinq bougies de préchauffage, et peut-être un système SCR. L'essentiel de `02-profil-vehicule.md` et de `04-budget-tokens.md` change selon la réponse.

**Méthode :** le **8ᵉ caractère du VIN**. Ou, plus simple encore : soulever le capot et regarder — un 5 cylindres en ligne diesel ne ressemble à aucun V6.

**Recoupement gratuit :** `https://vpic.nhtsa.dot.gov/api/vehicles/decodevin/<VIN>?format=json` retourne marque, modèle, moteur, cylindrée, puissance, classe GVWR, empattement et usine. API vérifiée fonctionnelle, sans clé.

**Ce que ça détermine :** la liste des PID à cibler, la présence d'après-traitement, le profil de pannes probables, l'applicabilité du rappel 16V618000, et la pertinence de la moitié du dossier.

---

### 2. Y a-t-il du DEF / SCR ?

**Pourquoi ça bloque :** deux volets de recherche affirment que le 3,2 L Power Stroke nord-américain comporte un système SCR à urée. **Aucune source primaire ne l'a confirmé** [NON VÉRIFIÉ]. Si c'est présent, ça ajoute le dosage DEF, des sondes NOx, et surtout une **stratégie de bridage progressif jusqu'à ~8 km/h** en cas de défaut persistant — ce qui peut immobiliser le véhicule dans la circulation, et qui est donc un élément de sûreté, pas un détail de configuration.

**Méthode :** ouvrir la trappe de remplissage de carburant et regarder s'il y a un second bouchon (généralement bleu). Dix secondes. Alternativement, chercher un réservoir DEF sous le véhicule ou une jauge DEF au tableau de bord.

**Ce que ça détermine :** environ six PID de surveillance, une famille de DTC (P20EE, P2BAD, P2002), et un avertissement de sûreté à documenter.

---

### 3. État des rappels — trois qui comptent vraiment

**Pourquoi ça bloque :** ce ne sont pas des considérations de conception, ce sont des considérations de sûreté sur un véhicule que tu utilises.

| Rappel | Défaut | Enjeu |
|---|---|---|
| **16V618000** | Débris métalliques de pompe HP dans le circuit d'alimentation diesel | Détruit injecteurs, vanne de dosage et rampe. Démarrage difficile, calage en roulant. Des plaintes décrivent des défaillances identiques **hors** de la fenêtre du rappel. |
| **17V408000** | Accouplement flexible d'arbre de transmission | Rupture **sans préavis et sans DTC**, souvent avant 160 000 km. Rupture de conduites de frein et de carburant documentée en dommage collatéral. |
| **19V767000** | Idem, supersède le 17V | Ajoute le risque de véhicule roulant librement (le frein de stationnement agit sur l'essieu, pas sur la transmission). |

À vérifier aussi, moins urgents mais gratuits dans le même geste : 16V188000 (airbags rideaux — **peut ne générer aucun DTC**, inspection physique uniquement), 17V668000 et 18V275000 (faisceau du module d'attelage, risque d'incendie, deux campagnes pour le même défaut), 21V631000 (câble de frein de stationnement), 25V572000 (caméra de recul).

**Méthode :** `ford.ca/support/recalls/` par VIN — c'est la seule source qui dit si les rappels ont été **effectués sur ton véhicule**. L'API NHTSA (`api.nhtsa.gov/recalls/recallsByVehicle`) donne le texte intégral des 21 rappels applicables au modèle, mais pas le statut par VIN.

**Ce que ça détermine :** si le véhicule est fiable pour du transport sur lequel tu comptes. Et une entrée dans la mémoire longue de Mecabot, s'il en a une un jour.

---

### 4. Kilométrage et historique d'entretien réels

**Pourquoi ça bloque :** un suivi longitudinal a besoin d'une ligne de base. Sans savoir quand le filtre à carburant a été changé pour la dernière fois (intervalle ~48 000 km, **critique** — l'eau détruit injecteurs et pompe HP), ni où on est dans la vie du DPF (service cendres ~240 000 km), ni quand le fluide 6R80 a été fait (~96 000 km en usage sévère, et un usage commercial *est* un usage sévère), aucune tendance n'est interprétable.

**Méthode :** relevé de l'odomètre, plus ce que tu as en factures. Reconstitution imparfaite acceptée — l'important est d'avoir un point de départ daté.

---

## Niveau 1 — ~70 $US, une soirée, lève l'essentiel du reste

> ✅ **Mise à jour 2026-08-24 : l'investissement matériel est déjà fait.** L'**OBDLink EX est en main et fonctionne sur macOS**, et **FORScan tourne déjà dans une VM Windows**. Ce niveau ne coûte donc plus rien — seulement du temps. Les mentions de « 70 $ » ci-dessous sont historiques.

### 4bis. Caractériser l'adaptateur et le premier contact — **priorité technique**

**Pourquoi ça bloque :** c'est le socle de l'étape 1 de la feuille de route de `09` §9. Rien ne se construit avant de savoir comment l'adaptateur se comporte réellement sur ce véhicule.

**Méthode :** un terminal série sur `/dev/tty.usbmodem*`, à la main, avant d'écrire une ligne de Rust. Quelques commandes AT/ST et quelques requêtes.

> ### ⚠️ **Journalise la séance — c'est le point ajouté le 2026-08-25, et il change la valeur de l'item**
>
> Cet item ne sert pas seulement à répondre aux six questions du tableau ci-dessous. **Le journal de la séance devient le premier jeu de test du projet** : `10` §9 établit qu'un troisième `Transport` de rejeu permet de développer la machine à états et le décodeur contre ce que le véhicule a *réellement* répondu, sans avoir le camion sous la main.
>
> **Une séance de 45 minutes avec le camion peut donc couvrir des semaines de développement.** Capture large, tant que tu y es :
>
> - l'initialisation complète, depuis la mise sous tension
> - une commutation de bus, dans les deux sens
> - quelques `$22` qui **réussissent** — dont `F190` (VIN), `F188`/`F18C`
> - quelques `$22` qui **échouent** (`7F .. 31`) — au moins aussi importants que les succès, `10` §4
> - un `$19` sur un module
> - **une session laissée volontairement expirer**, pour voir ce que fait `ATST`
> - une réponse longue, pour borner le réassemblage du STN
>
> Enregistre le brut, horodaté, sans nettoyer. Les bizarreries sont précisément ce qui a de la valeur.

| À caractériser | Pourquoi |
|---|---|
| Séquence d'initialisation qui fonctionne (débit, `ATZ`, `ATE0`, `ATSP`, protocole détecté) | Le pilote série en dépend directement |
| **Commande exacte de commutation MS-CAN** sur le STN2120, et son délai | Sans ça, aucun module de carrosserie n'est joignable (`05` §1.1) |
| **Taille maximale de réponse réassemblée** par l'adaptateur | `09` §3 délègue ISO-TP au firmware — il faut savoir jusqu'où ça tient |
| Débit réel en requêtes/seconde, PID unique et en lot | Conditionne la durée d'un balayage de DID et les volumes de `04` §1.2 |
| Format exact des lignes de réponse, et comportement sur réponse négative (`7F`) | Le décodeur en dépend |
| Comportement du timeout `ATST` et nécessité de `$3E TesterPresent` | Évite de perdre la session en pleine acquisition |

**Sonde de premier contact recommandée :** `$22 F190` (VIN) — DID normalisé par ISO 14229, donc réponse prévisible, et ça valide d'un coup la chaîne complète série → UDS → décodage.

### 16. Caractériser la calibration modifiée — **nouvelle priorité, gratuite et normalisée**

**Pourquoi c'est important :** le véhicule roule sur une calibration PCM qui n'est pas celle de Ford (`02`, encadré de tête). Tout ce que Mecabot conclura dépend de savoir **ce que cette calibration a désactivé**. Et il y a un piège : une calibration de delete supprime des familles de DTC, donc **un scan silencieux ne prouve pas qu'il n'y a pas de problème.**

Bonne nouvelle : les quatre sondes utiles sont de l'**OBD-II normalisé**, donc palier 1, aucune donnée propriétaire, faisable dès le premier contact.

| Sonde | Ce qu'on en tire |
|---|---|
| **Mode `$09` InfoType `04` — CALID** | L'identifiant de calibration. À **archiver** : c'est la ligne de base. |
| **Mode `$09` InfoType `06` — CVN** | Somme de contrôle de la calibration. **Sa raison d'être réglementaire est de détecter une reprogrammation.** À archiver aussi : s'il change un jour, quelqu'un a reflashé le PCM. |
| **Mode `$01` PID `01`** | Bits de support et d'achèvement des moniteurs. Les moniteurs d'après-traitement désactivés se voient ici. |
| **Mode `$06`** | Résultats des moniteurs non continus — **l'empreinte la plus fine de ce que le tune a réellement coupé**, pour ~600 tokens (`04` §5). |

**À vérifier aussi, physiquement, en même temps :**

- **L'EGR a-t-il été supprimé, ou seulement le DPF ?** Ça détermine si les signaux de position EGR ont un sens. Un regard sur le collecteur d'admission tranche.
- **Les sondes EGT sont-elles encore en place ?** Si elles ont disparu avec la ligne d'échappement, les DID correspondants liront n'importe quoi — et il ne faut surtout pas construire une ligne de base dessus.
- **Qui a fait le tune, et existe-t-il une documentation ?** Certains préparateurs publient ce qu'ils modifient. Ça vaut la question, même si la réponse est souvent « aucune idée » sur un véhicule d'occasion.

**Conséquence de conception à retenir :** `read_all_dtc` doit signaler explicitement que les familles d'après-traitement sont potentiellement muettes sur ce véhicule, plutôt que de laisser l'agent conclure à l'absence de défaut. C'est le genre de précaution qui distingue un outil utile d'un outil trompeur.

### 4ter. Recherche ODX européenne — **la piste au meilleur rendement**

**Pourquoi c'est prioritaire :** le cadre européen d'accès à l'information de réparation est **du règlement, pas un accord volontaire comme le CASIS** (`01` §5), et il comporte une obligation de fournir les données de diagnostic dans un format normalisé — en pratique ODX. Le Transit est un véhicule conçu en Europe. Si le portail technique Ford européen expose de l'ODX pour le V363, **ça pourrait raccourcir le palier 3 de plusieurs mois** en fournissant la base de DID par une voie parfaitement légitime.

⚠️ **[NON VÉRIFIÉ]** — ni le texte réglementaire exact, ni ce que le portail Ford européen offre réellement. C'est une piste, pas un acquis.

**À faire dans le même mouvement :** recenser les **définitions de PID personnalisés publiées par la communauté** FORScan. L'outil permet de définir ses propres PID (module, service, DID, formule, unité) et les utilisateurs publient ces définitions depuis des années — provenance volontaire de leurs auteurs, radicalement différente d'une extraction de base interne. [NON VÉRIFIÉ] sur l'ampleur de ce qui existe pour le Transit.

Cette vérification est **indépendante de tout le reste** et peut se faire à n'importe quel moment. À ne pas repousser : elle change le coût de l'étape la plus lourde du projet.

### 5. Scan de modules complet

**Ce que ça résout :**

- **Le TCM est-il séparé du PCM ?** Deux volets affirment l'intégration au PCM sur le 6R80, un troisième liste un TCM distinct sur HS-CAN. Détermine s'il faut adresser un ECU supplémentaire.
- **La cartographie module→bus.** Toute la répartition HS-CAN / MS-CAN de `02` §2 est partiellement [NON VÉRIFIÉ]. La source autoritaire est le diagramme de topologie du manuel d'atelier, derrière abonnement ; le scan la confirme empiriquement et gratuitement.
- **Y a-t-il un module d'après-traitement séparé ?** L'hypothèse est que non, et que toute la logique DPF/EGR réside dans le PCM (contrairement au 6.7 L Power Stroke des F-Series). Détermine à qui adresser les requêtes.
- **Confirmation qu'aucune passerelle ne filtre.** Si le scan atteint les modules MS-CAN sans obstacle, la conclusion « pas de Secure Gateway » de `01` §1 est vérifiée sur ton véhicule et pas seulement inférée.

Un scan réellement observé sur Transit rapporte : PCM, BCM II, ABS, APIM, SASM, RCM, PAM, HCM, IPC [COMMUNAUTÉ, t=28335].

### 6. Ouvrir le menu Service de FORScan — la contradiction à trancher

**Pourquoi ça bloque :** c'est la contradiction la plus nette du dossier, documentée en `01` §3.1 et `02`. Deux volets de recherche affirment que FORScan permet la **régénération DPF forcée** et le **codage IQA d'injecteurs** sur le 3,2 L Power Stroke, en s'appuyant sur des connaissances générales. Un troisième, qui a cherché directement sur le forum FORScan, ne trouve aucune trace de support pour ce moteur : le changelog documente la régénération statique uniquement pour le Focus II 1.6 TDCi (ECU Siemens SID-206), et la fonction « pilot injection relearn » a échoué sur un Transit 2016 2.2 TDCi avec « engine speed too high » (fil t=28667, resté sans réponse).

**Je penche pour la version négative.** Mais cette question ne se règle pas en cherchant davantage : elle se règle en ouvrant le menu et en lisant ce qui s'y trouve.

**Ce que ça détermine :** si l'un des outils les plus utiles du diagnostic diesel existe pour toi, et donc si Mecabot doit préparer cette intervention ou l'exclure. Rappel de `02` §8 : une régénération forcée produit 550-700 °C à l'échappement, exige l'extérieur, et **ne doit jamais être lancée si une dilution d'huile par le carburant est suspectée**.

### 7. Capturer la liste réelle des PID disponibles

**Pourquoi ça bloque :** tous les noms de PID de `02` §6 sont [NON VÉRIFIÉ]. Les seuls PID DPF confirmés le sont sur le Transit 2.2 Duratorq **européen** (`DPF_LOAD`, `DPF_DP`, `CATEMP12`, `EGT12`, fil t=6089), et le 3,2 L nord-américain a une calibration PCM spécifique aux États-Unis — l'extrapolation depuis l'Europe n'est pas légitime.

**Méthode :** FORScan expose la liste des PID qu'il connaît pour le véhicule connecté, et sait journaliser en CSV horodaté. C'est la façon la plus directe d'obtenir un inventaire réel.

**⚠️ Réserve juridique et éthique :** les règles du forum FORScan interdisent explicitement de discuter « protocole, numéros de PID » et les questions de reverse engineering [VÉRIFIÉ]. Observer quels PID sont disponibles pour ton propre véhicule et journaliser leurs valeurs est de l'usage normal de l'outil. Extraire ou redistribuer la base de correspondance PID de FORScan est autre chose. La frontière à tenir : **utiliser l'outil sur ton véhicule, oui ; en exfiltrer la propriété intellectuelle, non.** C'est aussi ce qui protège le projet — Ford a déjà utilisé le DMCA contre des travaux de ce genre (`01` §2.1).

### 8. Lire les données As-Built

Gratuit et sans abonnement, sur `motorcraftservice.com/AsBuilt` par VIN. Donne la configuration usine de tous les modules programmables. FORScan gratuit sait les lire depuis le véhicule aussi ; l'écriture exige l'Extended License.

**Ce que ça détermine :** la configuration réelle du véhicule (options, variantes de modules), utile comme référence documentaire et comme point de comparaison si un module est un jour remplacé. As-Built IPC et BCM sont confirmés accessibles sur Transit 2016 [COMMUNAUTÉ, t=30843] ; certains blocs PCM retournent « cannot find a solution ».

---

## Niveau 2 — Décisions d'achat, une fois le niveau 1 fait

### 9. Confirmer les tarifs de `motorcraftservice.com`

**Pourquoi ça bloque :** c'est le chiffre le plus structurant de tout le dossier côté budget, et il est **[NON VÉRIFIÉ]** — le certificat SSL du site a bloqué la vérification directe pendant la recherche. Les ordres de grandeur retenus (~26-30 $US pour 72 h, ~35-45 $ le mois, ~100-130 $ l'année) sont cohérents avec la documentation publique antérieure, mais à confirmer sur le portail.

**Méthode :** créer un compte (le Canada est sélectionnable ; le portail CASIS `oemrepairinfo.ca` y renvoie explicitement) et lire la grille tarifaire.

**Ce que ça détermine :** l'arbitrage entre une passe courte ciblée sur les sections utiles (schémas électriques, procédures des systèmes que tu comptes toucher, TSB pertinents) et un abonnement annuel. Pour un mono-véhicule, une passe de 72 h dont on télécharge méthodiquement les bonnes sections est probablement le meilleur rapport — c'est la seule source du **texte intégral des ~331 TSB** du Transit 2016, qui n'existe gratuitement nulle part.

### 10. La FORScan Extended License est-elle achetable ?

**Pourquoi ça bloque :** les ventes de nouvelles licences étaient **temporairement suspendues** au moment de la recherche [VÉRIFIÉ, `forscan.org/download`]. Un essai gratuit de 2 mois reste disponible.

**Ce qu'elle apporte réellement** (~15-20 $/an) : l'écriture des données As-Built, tous les sous-menus Configuration et Programming, la programmation de clés PATS. **Elle n'apporte ni PID supplémentaires ni fonctions de diagnostic** — la lecture des DTC, les données live de tous les modules, l'Output Control, les PID de surveillance DPF et le reset KAM sont dans la version gratuite [COMMUNAUTÉ — déclaration officielle de l'équipe FORScan, t=18697].

**Conséquence :** pour Mecabot, qui est en lecture seule par nécessité, **l'Extended License n'est pas sur le chemin critique.** Elle est utile à toi comme humain-actionneur, pas au projet. À ne pas acheter avant d'en avoir un besoin identifié.

### 11. ALLDATA DIY ou Motorcraft ?

À décider après avoir vu la grille Motorcraft. ALLDATA DIY couvre 1982-2021, donc ton 2016, à 19,99 $/mois ou 59,99 $/an [VÉRIFIÉ, alldata.com]. Aucune API dans les deux cas — tu liras des PDF et des pages web.

Critère de choix : Motorcraft est la source OEM (schémas et TSB d'origine, As-Built) ; ALLDATA est mieux organisé et moins cher à l'année. Pour construire un index RAG local, l'accès OEM aux schémas d'origine a plus de valeur.

---

## Niveau 3 — Questions techniques à trancher par l'expérimentation

Ces items ne sont pas des inconnues sur le véhicule, mais sur la faisabilité de l'architecture. Aucun ne bloque le niveau 1.

> ✅ **Mise à jour 2026-08-24 : ce niveau est de nouveau pleinement pertinent.** L'avertissement précédent reposait sur le malentendu corrigé en `09`. Les items 13 et 14 sont **au cœur de l'étape 1** de la feuille de route (`09` §9) et recoupent l'item 4bis. L'item 12 reste secondaire — sans écriture véhicule, il n'y a plus d'action dangereuse à confirmer par elicitation.

### 12. Le client MCP visé supporte-t-il l'elicitation MRTR ?

La forme actuelle du human-in-the-loop (`InputRequiredResult`, révision 2026-07-28) est **récente**, et le support par chaque client est [NON VÉRIFIÉ]. La forme antérieure (requête initiée serveur, 2025-06-18) avait des implémentations réelles. La négociation passe par `server/discover`.

**À tester tôt** sur le client réellement visé, plutôt qu'à supposer — c'est le mécanisme sur lequel repose toute la sûreté de conception de `03` §3.

### 13. Débit réel du STN2120 sur ce véhicule

Les chiffres retenus (~300-500 trames/s en conditions réelles, ~20-30 ms de surcharge par PID) sont [NON VÉRIFIÉ] — issus de retours communautaires, pas de mesure. Les volumes de tokens de `04` §1.2 en dépendent directement.

**Méthode :** mesurer le débit effectif en interrogeant un lot de PID connus, une fois le matériel en main. Trivial, et ça remplace une estimation par un fait.

### 14. La commutation MS-CAN, en pratique

C'est le premier vrai morceau d'ingénierie du projet, et **aucune bibliothèque ne le fait pour Ford aujourd'hui** (`05` §3.1). Il faut envoyer les commandes STN spécifiques via pyserial **avant** d'émettre les requêtes ISO-TP, puis router les trames diagnostiques à travers la même connexion série.

**À valider avant tout engagement d'architecture :** que la séquence commutation → session UDS → lecture fonctionne de façon reproductible sur un module MS-CAN réel (l'IPC ou le BCM sont de bons candidats).

### 15. Le Pi embarqué — ce qu'il faut mesurer avant de le laisser branché

Cible de déploiement ajoutée le 2026-08-24 : un **Raspberry Pi Zero 2 W** dans le véhicule, en rôle enregistreur autonome (`09` §11). Quatre vérifications, dont une qui est un enjeu réel et pas une curiosité technique.

| À vérifier | Pourquoi | Quand |
|---|---|---|
| **Consommation réelle du montage, au multimètre** | **La broche 16 du J1962 est le positif de batterie non commuté** : l'appareil est alimenté véhicule éteint. L'estimation de `09` §11.6 (~0,12 A, ~20 Ah/semaine) suffirait à empêcher un démarrage en deux à trois semaines, davantage en hiver. **À mesurer avant de laisser le montage branché plus d'une nuit.** | avant tout usage prolongé |
| Que l'OBDLink EX apparaisse en `/dev/ttyACM0` via USB OTG, et que le débit tienne | Le pilote `cdc_acm` devrait le faire sans intervention, mais ça se constate. Recoupe l'item 4bis — **à faire d'abord au terminal série sur le Mac**, c'est plus simple à déboguer. | avant le portage |
| Comportement du timeout `ATST` et nécessité de `$3E TesterPresent` sur session longue | Une session interactive dure dix minutes ; un enregistreur tourne des heures. Beaucoup plus critique ici qu'en usage bureau. | avec l'item 4bis |
| Ce que `rmcp` couvre du transport Streamable HTTP **et de son volet authentification** | Uniquement si le mode « MCP en direct dans le véhicule » devient un besoin. Le rôle enregistreur n'expose rien et n'en a pas besoin. | reportable |

⚠️ **Câblage :** alimenter depuis la broche 16 ajoute un chemin sur le circuit de batterie. À fusionner au plus près de la source. C'est du câblage de véhicule — à faire soigneusement ou à faire faire.

---

## Ce qui ne se vérifiera pas, et qu'il faut accepter

Par honnêteté, la liste symétrique. Ces questions ont été cherchées et resteront ouvertes :

- **L'algorithme UDS $27 Ford 2015-2019.** Aucun article académique, CVE ou présentation de conférence ne l'analyse [INTROUVABLE]. `jglim/UnlockECU`, 3 038 entrées, **0 Ford** [VÉRIFIÉ]. Ce n'est pas une piste à creuser, c'est une frontière du projet.
- **L'assignation FNV du Transit 2016.** Les générations de réseau Ford sont de la documentation interne, jamais publiée [INTROUVABLE]. L'attribution à FNV2 est cohérente et sans signal contradictoire, mais restera une inférence.
- **La revue MDPI sur les applications ML basées OBD-II**, citée dans le document initial. HTTP 403, plusieurs tentatives d'URL [INTROUVABLE]. Le verdict de `04` §4.1 tient sans elle, par arithmétique.
- **Le dépôt `caringcaribou`** était inaccessible pendant la recherche (404) [INTROUVABLE]. Utile pour la découverte de DID inconnus s'il réapparaît.
- **Le texte intégral des TSB gratuitement.** N'existe pas. Même quand l'API TSB de la NHTSA fonctionnait (elle retourne HTTP 403 aujourd'hui), elle ne fournissait que numéros et titres.
- **Les DTC et seuils diesel de `02` §4**, priorité 2. Largement [NON VÉRIFIÉ] — cohérents avec la plateforme, mais issus de connaissances générales. Ils se valideront contre le manuel d'atelier, pas par plus de recherche web.

---

## Résumé exécutable

| # | Vérification | Coût | Temps | Débloque |
|---|---|---|---|---|
| ~~1~~ | ~~Quel moteur~~ | — | — | ✅ **Résolu : 3,2 L diesel, deleted** |
| ~~2~~ | ~~DEF/SCR présent~~ | — | — | ✅ **Sans objet** |
| **3** | **Statut des rappels par VIN** | 0 $ | 10 min | **La fiabilité du véhicule — le 16V618000 s'applique** |
| **4** | **Odomètre + historique** | 0 $ | variable | La ligne de base du suivi longitudinal |
| **4bis** | **Caractériser l'adaptateur (terminal série)** | 0 $ | 45 min | **L'étape 1 de la feuille de route** |
| **4ter** | **Recherche ODX européenne + PID communautaires** | 0 $ | 1-2 h | **Le coût du palier 3** — indépendante, à faire tôt |
| 5 | Scan de modules | 0 $ | 20 min | Topologie réseau, TCM, absence de passerelle |
| ~~6~~ | ~~Menu Service FORScan~~ | — | — | ✅ **Abandonné — plus de DPF** |
| 7 | Inventaire réel des PID | 0 $ | 30 min | `02` §6 **révisé** — cible déplacée vers rampe/turbo/6R80 |
| **16** | **CALID + CVN + Mode 01 PID 01 + Mode 06** | 0 $ | 20 min | **Ce que le tune a désactivé** — normalisé, palier 1 |
| 8 | Données As-Built | 0 $ | 15 min | Configuration usine de référence |
| 9 | Tarifs Motorcraft | 0 $ | 10 min | La décision d'abonnement |
| 10 | Extended License disponible | 0 $ | 5 min | Rien de critique — hors chemin critique |
| 12-14 | Faisabilité technique | 0 $ | recoupe 4bis | L'architecture |
| **15** | **Consommation du Pi embarqué au multimètre** | ~20 $ (Pi Zero 2 W) | 15 min | **Que le montage ne vide pas la batterie** |

**Tout ce tableau coûte maintenant zéro dollar** — le matériel et FORScan sont déjà en place. Seul l'item 15 suppose un achat, et il n'est pas sur le chemin critique : il concerne l'étape 4bis de la feuille de route, pas le premier contact.

**Ce qui reste en tête de liste, maintenant que l'item 1 est clos :** l'**item 3** (statut des rappels — le 16V618000 concerne la pompe haute pression, et le circuit d'alimentation est la partie du moteur que le delete ne touche pas), puis l'**item 16** (ce que la calibration a désactivé, sans quoi tout scan est ambigu), puis l'**item 4bis** (caractériser l'adaptateur, préalable à toute ligne de code).
