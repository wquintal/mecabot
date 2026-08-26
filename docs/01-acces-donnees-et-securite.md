# 01 — Accès aux données et barrières de sécurité

Ce document répond directement aux trois inquiétudes : données propriétaires, éléments sécurisés, API inaccessibles.

**Convention de fiabilité** utilisée dans tout le dossier :

- **[VÉRIFIÉ]** — source primaire consultée pendant la recherche (page officielle, dépôt Git, API interrogée)
- **[COMMUNAUTÉ]** — forum ou dépôt tiers, avec numéro de fil quand disponible
- **[NON VÉRIFIÉ]** — plausible, cohérent avec le reste, mais aucune source primaire atteinte
- **[INTROUVABLE]** — cherché explicitement, rien d'autoritaire trouvé. À ne pas confondre avec « faux ».

---

## 1. Passerelles de sécurité : le Transit 2016 est ouvert

### 1.1 Pas de Secure Gateway chez Ford sur cette génération

Le Transit 2016 (V363, MY2015+ en Amérique du Nord) **n'a aucun module passerelle qui bloque l'écriture depuis le port OBD-II**.

Preuve la plus solide : une recherche plein texte sur `forum.forscan.org` retourne **zéro résultat** pour « gateway », « secure gateway » et « FNV3 » [VÉRIFIÉ, recherche effectuée]. Sur n'importe quel forum Jeep ou Ram, le SGW est le sujet dominant depuis 2018. Une absence totale de discussion sur un forum de 1,7 M de vues par fil est un signal négatif fort.

Comparaison des trois constructeurs :

| Constructeur | Mécanisme | Années | Écriture OBD bloquée matériellement ? |
|---|---|---|---|
| **Ford** (FNV2, ~2013-2020, inclut Transit 2015-2019) | Passerelle de **routage** entre HS-CAN et MS-CAN, pas de filtrage sécurité | — | **Non** |
| **Ford** (FNV3, 2021+) | Protocoles plus complexes ; FORScan annonce un « ensemble limité de fonctions » pour 2021-2024MY [VÉRIFIÉ, `forscan.org`] | 2021+ | Non — HP Tuners et SCT fonctionnent sans dongle de contournement |
| **FCA / Stellantis** | Secure Gateway Module : pare-feu CAN matériel, tout trafic d'écriture non authentifié est jeté | MY2018+ (Wrangler JL, Ram 1500 DT d'abord) [NON VÉRIFIÉ sur la date exacte — forums inaccessibles] | **Oui** — dongle de contournement ou API abonnée obligatoires |
| **GM** | Pas de passerelle ; SecurityAccess ($27) au niveau de chaque module + crédentiels TechLink pour certaines opérations | — | Non, mais restriction par crédentiels |

L'attribution du Transit 2016 à la génération FNV2 est **[NON VÉRIFIÉ]** : les assignations FNV de Ford sont de la documentation interne, jamais publiée [INTROUVABLE]. Aucun signal contradictoire n'a été trouvé.

### 1.2 Ford n'a pas bloqué les outils tiers sur cette génération

FORScan annonce le support Ford 1996-2024MY sans exclusion pour 2015-2019 [VÉRIFIÉ, `forscan.org/faq`]. Aucune mise à jour de firmware ayant cassé l'accès tiers sur cette génération n'est documentée où que ce soit [INTROUVABLE].

Les restrictions qui existent sont **logicielles ou par crédentiels, jamais des blocages OBD matériels** :

- les calibrations PCM exigent un compte Ford TIS2Web + IDS/FDRS ;
- certaines opérations PATS exigent des jetons de sécurité Ford via le portail VSIG de la NASTF ;
- Ford ne bloque techniquement ni J2534 Pass-Thru ni l'accès OBD-II standard.

### 1.3 FEPS : sans objet sur ton véhicule

Le FEPS (Ford Enhanced Power Supply) est un ~18 V sur la broche 13 du connecteur OBD, que les PCM de l'ère **EEC-V** (1996 à ~2004) exigeaient pour déverrouiller la NVM en écriture [VÉRIFIÉ, spécification OBD-II ; autoenginuity.com].

Sur un Transit 2016, tout est CAN + UDS. La broche 13 ne porte aucune signification. **Aucune procédure ne nécessite le FEPS.** Les adaptateurs orientés FORScan (OBDLink EX, ELS27, vLinker FS) ne l'implémentent pas, et n'en ont pas besoin.

---

## 2. UDS $27 SecurityAccess : la vraie limite

C'est ici qu'il faut être franc : **c'est le point où ton middleware ne pourra pas égaler FORScan, et il n'y a pas de contournement propre.**

### 2.1 L'algorithme Ford 2015-2019 n'est publiquement connu de personne

Preuve la plus nette : le dépôt `jglim/UnlockECU` est la base seed/key ouverte la plus complète qui existe — **3 038 entrées** couvrant Daimler, VW, Honda, Subaru, Marquardt. Interrogation directe de son `db.json` : **exactement 0 entrée Ford** [VÉRIFIÉ].

Pourquoi ce vide :

- les bases européennes dérivent de fichiers SMR-D/ODX ; l'équivalent Ford n'est pas public ;
- Ford a fait retirer des travaux antérieurs par DMCA (le parseur SMR d'OpenVehicleDiag a disparu pour cette raison) ;
- aucun article académique, CVE ou présentation de conférence sécurité n'analyse spécifiquement le $27 Ford 2015-2019 [INTROUVABLE].

### 2.2 Ce qui est public : le FG Falcon australien, et c'est tout

Le dépôt `jakka351/FG-Falcon` [COMMUNAUTÉ] documente le Ford Falcon FG australien (2008-2014), pas le marché nord-américain. Il révèle deux mécanismes, instructifs sur la philosophie Ford de l'époque :

**Clés statiques** — certains modules acceptent une réponse codée en dur quel que soit le seed :

| Adresse | Module | Niveau | Clé | ASCII |
|---|---|---|---|---|
| 0x720 | IPC | 0x01 | `43 4F 4C 49 4E` | « COLIN » |
| 0x727 | body | 0x01 | `4A 61 6E 69 73` | « Janis » |
| 0x7A6 | FDIM | 0x01 | `42 72 61 64 57` | « BradW » |
| 0x767 | ABS | 0x01 | `8A 78 90 34 F7` | — |

Des prénoms de développeurs comme clés de sécurité. Les clés sont **par module et par niveau**, pas partagées.

**Algorithme LFSR calculé** (session 0xFA SystemSupplierSpecific) : état initial `0xC541A9`, polynôme XOR `0x109028`, constante `0x123456`, 64 itérations, sortie 3 octets.

Rien de tout cela ne s'applique à ton véhicule. C'est utile uniquement pour comprendre la forme du problème.

### 2.3 Comment FORScan y arrive

Les règles du forum FORScan **interdisent explicitement** de discuter « protocole, numéros de PID » et « les questions de reverse engineering sur le protocole et l'application » [VÉRIFIÉ, page de règles]. Un administrateur déclare que FORScan utilise « le protocole de service propriétaire utilisé par les scanners dealer et professionnels » [COMMUNAUTÉ].

Le changelog enregistre un bug « impossible d'obtenir l'accès sécurité PATS sur certains nouveaux BCM » corrigé en v2.3.71 [VÉRIFIÉ, `forscan.org/download`] — confirmation d'une implémentation active par module.

Inférence la mieux étayée : FORScan maintient une base interne de clés statiques par numéro de pièce/calibration, plus des algorithmes calculés avec des paramètres spécifiques par module, obtenus par reverse engineering des DLL de Ford IDS. Tout tourne **hors ligne**, côté client, après activation de licence. Aucun contact serveur Ford n'est requis [VÉRIFIÉ, `forscan.org/faq`].

### 2.4 Conséquence de conception, à assumer

**Mecabot sera en lecture seule sur les modules, point.** Pas par prudence architecturale — par impossibilité technique et juridique de reproduire le $27.

Cela invalide directement plusieurs pans du document initial : le service $2F (contrôle d'actionneurs), le $2E (écriture de DID), le $31 (routines), et toute la machinerie FSM séquençant seed/key → écriture → reset. Ces sections décrivent des capacités que le projet n'aura pas.

**Le repositionnement :** quand une action d'écriture est nécessaire, Mecabot ne l'exécute pas — il **prépare l'intervention** : identifie la procédure, cite la source, liste les préconditions, et te dit exactement quoi faire dans FORScan. L'humain reste l'actionneur, avec l'outil qui a le droit et la capacité de le faire.

---

## 3. Ce que FORScan sait faire sur un Transit 2016

Répartition officielle gratuit / Extended License [COMMUNAUTÉ — déclaration officielle de l'équipe FORScan, fil t=18697, 1,69 M de vues] :

| Fonction | Gratuit | Extended |
|---|---|---|
| Lecture/effacement DTC, tous modules | ✅ | — |
| Données live / PID, tous modules | ✅ | — |
| Output Control (tests d'actionneurs) | ✅ | — |
| PID de surveillance DPF | ✅ | — |
| Reset KAM / adaptations | ✅ | — |
| **Lecture** des données As-Built | ✅ | — |
| **Écriture** des données As-Built | ❌ | requis |
| Tous les sous-menus Configuration et Programming | ❌ | requis |
| Programmation de clés PATS | ❌ | requis |
| Reflashing de firmware | **indisponible dans toutes les licences** | — |

Confirmations spécifiques au Transit :

- **Scan multi-modules confirmé** sur Transit : PCM, BCM II, ABS, APIM, SASM, RCM, PAM, HCM, IPC [COMMUNAUTÉ, t=28335, t=30712]
- **PATS** : le fil « PATS Q&A » liste explicitement « Transit Custom 2016 » comme supporté [COMMUNAUTÉ, t=17250]. Le modèle 2016 utilise le PATS classique BCM-PCM, non affecté par le « PATS distribué » introduit en 2019.75MY.
- **As-Built IPC et BCM** confirmés accessibles sur Transit 2016 [COMMUNAUTÉ, t=30843]. Certains blocs PCM posent problème (« cannot find a solution »). L'As-Built ABS est **absent** sur Transit MK7 (2006-2013) [COMMUNAUTÉ, t=30788] ; statut inconnu sur V363.
- **Reflashing retiré** : « les mauvais utilisateurs créaient trop de problèmes… Ford a verrouillé la plupart des fichiers de firmware » [COMMUNAUTÉ, t=30683, t=30761].

### 3.1 Contradiction non résolue, à trancher empiriquement

Deux volets de recherche affirment que FORScan permet la **régénération DPF forcée** et le **codage IQA d'injecteurs** sur le 3,2 L Power Stroke. Un troisième volet, qui a fait une recherche directe sur le forum FORScan, ne trouve **aucune trace de support pour le 3,2 L** et signale [INTROUVABLE] :

- le changelog FORScan documente la régénération statique uniquement pour le **Ford Focus II 1.6 TDCi** (ECU Siemens SID-206) — aucune variante Transit [VÉRIFIÉ, `forscan.org/download`] ;
- la fonction « pilot injection relearn » existe sur Transit 2.2 TDCi européen mais a échoué sur un Transit 2016 2.2 TDI 155 ch avec l'erreur « engine speed too high » — fil t=28667 resté sans réponse [COMMUNAUTÉ] ;
- le 3,2 L nord-américain a une calibration PCM spécifique aux États-Unis ; l'extrapolation depuis les Transit européens n'est pas légitime.

Les deux volets affirmatifs s'appuyaient sur des connaissances générales, non sur des sources primaires. **Je penche pour la version négative**, mais c'est une question qui se règle en installant FORScan et en regardant le menu Service, pas en cherchant davantage.

Fonctions par ailleurs confirmées **absentes** : test d'équilibre des cylindres, test individuel de bougies de préchauffage, contribution individuelle par injecteur [COMMUNAUTÉ, t=314, sur 6.0L Power Stroke]. Fonctions DEF/SCR (reset qualité DEF, adaptation sonde NOx) : aucune trace.

---

## 4. Coût réel d'accès aux données

### 4.1 Ford officiel

Le portail Ford pour les indépendants est **`motorcraftservice.com`**, accessible aux Canadiens (le Canada est sélectionnable à l'inscription). Le portail CASIS canadien `oemrepairinfo.ca` y renvoie explicitement [VÉRIFIÉ].

| Accès | Coût approximatif | Contenu |
|---|---|---|
| Passe 72 h | ~26-30 $US | Manuel d'atelier, schémas, TSB texte intégral, calibrations |
| 1 mois | ~35-45 $US | idem |
| 1 an | ~100-130 $US | idem |
| **Données As-Built par VIN** | **gratuit** | Configuration usine de tous les modules programmables |

⚠️ Les tarifs sont **[NON VÉRIFIÉ]** : le certificat SSL de `motorcraftservice.com` a bloqué la vérification directe pendant la recherche. Les ordres de grandeur sont cohérents avec la documentation publique antérieure, mais à confirmer sur le portail.

**FDRS** (Ford Diagnosis and Repair System) n'exige **pas** de VCM3 : depuis 2021-2022, Ford accepte tout adaptateur J2534 conforme. Le VCM3 n'est que l'implémentation J2534 de Ford. En revanche FDRS est un abonnement distinct de l'info technique, et le reflashing complet exige en plus un statut de compte approuvé.

### 4.2 Agrégateurs tiers

| Source | Coût | API ? |
|---|---|---|
| **ALLDATA DIY** | 19,99 $/mois · 59,99 $/an · 129,99 $/3 ans [VÉRIFIÉ, alldata.com] | Aucune |
| ALLDATA Repair (pro) | 209 $/mois · 2 508 $/an [VÉRIFIÉ] | Aucune |
| Identifix Direct-Hit DIY | 29,99 $/mois · ~270 $/an [VÉRIFIÉ, store.solera.com] | Aucune |
| Identifix Pro | ~288 $/mois · ~2 390 $/an [VÉRIFIÉ] | Aucune |
| Mitchell1 ProDemand | tarifs non publics, sur devis | Aucune |
| **MOTOR Information Systems** | contrat entreprise sur devis | **B2B strict** — pas de palier individuel, pas d'inscription en libre-service, pas de tarif public |
| Haynes (manuel Transit 2015-2019) | ~30-50 $ | Aucune |

ALLDATA DIY couvre les millésimes 1982-2021, donc ton 2016 [VÉRIFIÉ]. C'est le chemin le moins cher vers de la donnée OEM complète, disponible au Canada.

**Verdict sur les API : la section 4 du document initial est à jeter.** Aucun de ces fournisseurs n'expose d'API à un individu. MOTOR n'est pas seulement cher — ce n'est pas une porte conçue pour des individus.

### 4.3 Ce qui est réellement gratuit et machine-readable

C'est court, mais c'est de la vraie API sans clé :

| Endpoint | Contenu | Testé |
|---|---|---|
| `vpic.nhtsa.dot.gov/api/` | Décodage VIN : marque, modèle, moteur, cylindrée, puissance, classe GVWR, empattement, usine | ✅ retour JSON confirmé |
| `api.nhtsa.gov/recalls/recallsByVehicle` | **Texte intégral** des rappels : composant, résumé du défaut, conséquence, remède | ✅ 21 rappels pour Transit 2016 |
| `api.nhtsa.gov/complaints/complaintsByVehicle` | Narratifs complets de plaintes, drapeaux incendie/collision, blessures | ✅ 153 plaintes pour Transit 2016 |
| `api.nhtsa.gov/TSBs/tsbsByVehicle` | — | ❌ **HTTP 403** — endpoint apparemment mort |
| `motorcraftservice.com/AsBuilt` | Config usine par VIN | gratuit, sans abonnement |
| `opendbc` (comma.ai, MIT) | DBC Ford : Fusion 2018, F-150, CGEA 1.2 2011, Lincoln | ✅ **aucun fichier Transit** |

À noter : même quand l'API TSB de la NHTSA fonctionnait, elle ne fournissait que **numéros et titres**, jamais le texte. **Il n'existe aucune source gratuite du texte intégral des TSB.** Le Transit 2016 en compte environ 331. C'est le seul argument fort en faveur de l'abonnement Motorcraft ou ALLDATA.

`opendbc` ne couvre pas le Transit, et de toute façon décrit le CAN de diffusion (vitesse, angle de volant, pédale) utilisé par openpilot — pas les sessions de diagnostic. Utile comme gabarit de reverse engineering, inutile pour la couche UDS.

---

## 5. Cadre légal : CASIS au Canada

Le **CASIS** (Canadian Automotive Service Information Standard) est un **accord volontaire**, pas une loi, signé en septembre 2009 entre la CVMA (Ford, GM, Stellantis), Global Automakers of Canada, la NATA et l'AIA [VÉRIFIÉ, cvma.ca, natacanada.ca].

Il engage les constructeurs à partager l'information de service « à un niveau équivalent à celui des concessionnaires autorisés ». Le portail central est `oemrepairinfo.ca`, qui renvoie vers 27 sites constructeurs — Ford vers `motorcraftservice.com`.

**Toujours en vigueur en 2026** [NON VÉRIFIÉ au-delà de mi-2025], mais la NATA reconnaît publiquement qu'il « pourrait nécessiter une mise à jour » pour couvrir la télématique moderne. Un groupe de travail CASIS examine la question. Aucun remplacement législatif n'a été adopté.

### Limites qu'il faut connaître

- Le CASIS cible les « professionnels indépendants » et « ateliers de réparation ». Il **ne couvre pas explicitement le propriétaire-exploitant** agissant comme son propre mécanicien. En pratique, `motorcraftservice.com` ne demande aucune preuve d'être un atelier — l'inscription individuelle fonctionne. Mais la garantie juridique, elle, est cadrée sur les entreprises.
- Le CASIS **ne couvre pas** : les données PID en format machine-readable, les flux télématiques infonuagiques, les crédentiels de sécurité / antidémarreur (programme « Vehicle Security Professional » distinct).
- **Angle Québec** : aucune loi provinciale sur le droit à la réparation automobile en 2026. La LPC et le Code civil donnent des droits généraux de garantie contre les vices, rien de spécifique à l'accès aux données. Le Québec suit le cadre CASIS national sans disposition supplémentaire.

Côté américain, le cadre qui a créé cette infrastructure : règles EPA du Clean Air Act (40 CFR Part 86), loi Right to Repair du Massachusetts (2012), et le MOU national de 2014 signé par Ford [VÉRIFIÉ]. Ces textes n'ont pas force de loi au Canada mais ont produit le portail que le CASIS pointe ensuite.

---

## 6. Récapitulatif : ce qui est ouvert, ce qui est fermé

**Ouvert, sans authentification, sur ton véhicule :**

- DTC de tous les modules, y compris codes propriétaires Ford
- Données live, tous modules (HS-CAN et MS-CAN)
- PID étendus Ford via UDS $22 ReadDataByIdentifier
- Freeze frame (Mode 02), monitors embarqués (Mode 06)
- Tests d'actionneurs de base
- Données As-Built en lecture
- Rappels, plaintes, décodage VIN par API gratuite

**Payant mais accessible (~80-150 $/an au total) :**

- Manuels d'atelier, schémas électriques, texte intégral des TSB
- PID étendus dans FORScan (Extended License, ~15-20 $/an — ⚠️ les ventes de nouvelles licences étaient **temporairement suspendues** au moment de la recherche [VÉRIFIÉ, `forscan.org/download`] ; un essai gratuit de 2 mois reste disponible)

**Fermé, définitivement, pour un individu :**

- Algorithmes SecurityAccess $27 → aucune écriture module par tes propres moyens
- Reflashing / recalibration PCM → IDS/FDRS + TIS2Web uniquement
- Base de PID non standard de Ford → licence ETI à ~50 000 $US/an
- API de données de réparation (MOTOR, ALLDATA, Mitchell) → B2B, contrats entreprise
- Texte intégral des TSB gratuitement → n'existe pas
