# 11 — Modèle de connaissance véhicule : les cinq couches

**Date :** 2026-08-25
**Objet :** le document qui rend l'application multi-marques. Il répond à une question que le dossier n'avait jamais posée, parce qu'il supposait un seul véhicule : **de quoi le logiciel a-t-il besoin de *savoir* sur un véhicule, et d'où vient chaque morceau ?**
**Préalables :** `09` (architecture), `10` (conception du serveur), `01` (frontières d'accès aux données).

> ## Décision de périmètre du 2026-08-25
>
> **Multi-marques dès la conception.** L'application n'est plus « un outil pour un Transit 2016 », c'est **un outil OBD-II/UDS générique dont la connaissance véhicule vit en données**. Le Transit devient le **profil n°1** et le sujet de validation, pas le périmètre.
>
> Tu avais identifié le risque de cette voie dans l'énoncé du choix : **« risque d'abstraire avant de savoir »**. Il est réel, et c'est le principal danger de ce document. Il est traité en §8 — pas contourné.

---

## 1. Ce que « multi-marques » exige réellement, et ce qu'il n'exige pas

Il faut cadrer, sinon le périmètre devient infini.

**Ça n'exige pas** de supporter chaque véhicule au même niveau de profondeur. C'est l'erreur qui rendrait le projet impossible : personne, pas même les éditeurs à 50 000 $/an, n'a une couverture profonde universelle.

**Ça exige** une propriété plus modeste et beaucoup plus atteignable : **aucun fait sur un véhicule particulier n'est écrit dans le code.** Le code sait parler OBD-II et UDS ; il ne sait rien sur un Transit. Ce qu'il sait sur un Transit, il le lit dans un profil.

La conséquence est une **dégradation gracieuse** plutôt qu'un support binaire :

| Sur un véhicule… | Ce que Mecabot fait |
|---|---|
| …quelconque, conforme OBD-II | Palier 1 complet, immédiatement, **sans aucun profil** |
| …dont la marque a un profil de couche 2 | + inventaire de modules et DTC multi-modules (palier 2) |
| …dont le modèle a un profil de couche 3 | + grandeurs étendues (palier 3) |
| …qui est le tien, avec un historique | + l'analyse longitudinale, la seule chose que rien d'autre ne fait |

C'est un meilleur produit que le mono-véhicule sur un point précis : **il est utile le premier jour, sur n'importe quel véhicule, avant qu'aucun travail de découverte n'ait été fait.**

---

## 2. Les cinq couches

C'est l'apport central de ce document. Chaque morceau de connaissance véhicule appartient à exactement une couche, et **chaque couche a un coût d'obtention, une stabilité et une partageabilité différents**.

| # | Couche | Contenu | Origine | Coût | Partageable ? |
|---|---|---|---|---|---|
| **0** | **Norme** | Modes J1979, DTC génériques J2012, DID normalisés `F180-F1FF`, ISO 15765-2 | Normes publiques | **nul** | universelle, rien à partager |
| **1** | **Transport** | Protocole OBD, débit, 11 vs 29 bits, bus secondaires et leur brochage | **Auto-détecté**, sauf les bus secondaires | quasi nul | trivial |
| **2** | **Marque** | Conventions d'adressage de diagnostic, comportement de session, famille d'algorithme `$27`, **présence d'une passerelle sécurisée** | Observation + communauté | faible, stable dans le temps | oui, et c'est peu volumineux |
| **3** | **Modèle / année / plateforme** | Inventaire des modules et leurs adresses, topologie de bus, **carte des DID étendus** | Découverte incrémentale, ou source sous licence | **élevé — c'est le vrai travail** | oui en principe, ⚠️ champ de mines de provenance |
| **4** | **Exemplaire (ce VIN)** | As-Built, identité de calibration (CALID/CVN), ce que la calibration supprime, options montées, **modifications**, historique d'entretien, mesures accumulées | Le véhicule lui-même | nul — c'est ton camion | **jamais.** Contient le VIN. |

### Les deux conséquences qui comptent

**Première conséquence — la règle de dépendance.** Le code ne dépend que des couches 0 et 1. Les couches 2, 3 et 4 sont de l'**enrichissement optionnel**. Un profil absent dégrade la capacité ; il ne casse rien. C'est ce qui rend le multi-marques abordable : il n'y a pas de matrice de support à remplir avant de livrer.

**Deuxième conséquence — le coût et la partageabilité sont inversés.** La couche 4 est gratuite à obtenir et impossible à partager. La couche 3 est chère à obtenir et légalement délicate à partager. C'est exactement l'inverse de l'intuition, et ça détermine où va l'effort.

### Ce que le modèle explique rétrospectivement

Les quatre **paliers de capacité** de `09` §4 avaient été conçus pour un seul véhicule. Ils se révèlent être la projection des couches sur l'axe des fonctionnalités :

| Palier de `09` §4 | Couche nécessaire |
|---|---|
| 1 — OBD-II normalisé | couche 0 seule |
| 2 — inventaire modules + DTC | couche 2 (adressage) |
| 3 — DID étendus | couche 3 |
| 4 — écritures, `$27` | clés inobtenables — hors de toute couche |

**Bonne nouvelle non anticipée : la structure en paliers était déjà prête pour le multi-marques.** Elle ne décrivait pas des étapes propres au Transit, elle décrivait des besoins de connaissance. Le rescope ne l'invalide pas, il l'explique. C'est ce qui limite l'ampleur de la réécriture.

---

## 3. Art antérieur — vérifié le 2026-08-25

Avant d'inventer un format de profil, il fallait regarder qui a déjà résolu « la connaissance de diagnostic véhicule sous forme de données ». La réponse est instructive et converge partout vers la même asymétrie.

### ODX / ISO 22901 — la norme qui existe déjà pour ça

**ODX** (*Open Diagnostic data eXchange*), normalisé **ISO 22901** et maintenu par **ASAM e.V.**, est le format XML normalisé décrivant les capacités de diagnostic d'un calculateur : services, DID, structures de données, facteurs d'échelle, unités, textes. **PDX** en est la forme empaquetée. C'est littéralement le format dont la couche 3 a besoin.

**`odxtools`** [VÉRIFIÉ 2026-08-25] — publié par **Mercedes-Benz**, **licence MIT**, version **11.5.4 publiée le 2026-08-24** (la veille de ce document), 1 559 commits, Python ≥3.10 testé jusqu'à 3.14. Il parse ODX/PDX, encode des requêtes, décode des réponses, décode une session live, et compare deux versions de base. Outillage libre, mûr, manifestement très vivant.

⛔ **Et voici le résultat qui compte, et il est négatif.** La documentation d'`odxtools` **ne dit nulle part où obtenir des fichiers ODX/PDX de véhicules réels.** Le dépôt ne livre qu'un fichier de démonstration inventé, `somersault.pdx`.

**C'est exactement la forme du problème FORScan, transposée.** Le format est ouvert, l'outillage est ouvert, sous licence permissive, de qualité industrielle — **et la donnée n'est nulle part.** Les constructeurs produisent de l'ODX pour leurs propres chaînes d'outils et ne le publient pas.

> ⚠️ **[NON VÉRIFIÉ], et ça reste la meilleure piste de recherche du dossier.** `09` §5 soulevait l'hypothèse d'une obligation réglementaire européenne de publier de l'ODX au titre de l'accès à l'information de réparation. Le présent constat ne la réfute pas : il montre seulement qu'aucune source ouverte ne s'est matérialisée dans l'écosystème `odxtools`. Si l'obligation existe, la donnée est derrière un portail constructeur payant, pas sur GitHub. **Le multi-marques augmente la valeur de cette piste** — une obligation réglementaire vaudrait pour tous les constructeurs vendant en Europe, pas seulement pour Ford.
>
> Le texte de la norme ISO 22901-1 lui-même n'a pas pu être consulté (iso.org renvoie 403 à une récupération automatisée). Les normes ISO sont payantes, de l'ordre de 150-200 CHF. **[NON VÉRIFIÉ]** sur le prix exact et l'édition courante.

### OpenDBC — le bon modèle d'organisation, la mauvaise donnée

**OpenDBC** de comma.ai [VÉRIFIÉ 2026-08-25] — **licence MIT**, 2 880 commits, manifestement actif, couvre les véhicules ~2016+ dotés de LKAS/ACC.

**Ce qu'il n'est pas :** une source de connaissance de diagnostic. Il contient **des fichiers DBC seulement — des définitions de messages CAN de contrôle et d'état, aucune donnée UDS, OBD-II ou diagnostic.** L'objet déclaré est l'interface ADAS : direction, accélérateur, freins, vitesse, angle de volant. La couche 3 de Mecabot n'y trouvera rien.

**Ce qu'il est, en revanche, et c'est précieux :** la démonstration à grande échelle que **le modèle « connaissance véhicule en fichiers de données par marque, licence permissive, contributions communautaires » fonctionne.** C'est le modèle d'organisation que le registre de profils doit copier. Sur l'axe organisationnel, OpenDBC est le précédent ; sur l'axe de la donnée, il est hors sujet.

### CaringCaribou — l'art antérieur direct de `probe_dids`, et un avertissement

**CaringCaribou** [VÉRIFIÉ 2026-08-25], issu du projet de recherche HEAVENS, **licence GPL-3.0**, 550 commits. Modules : découverte UDS, **énumération de services**, identification de sous-services, `dump_dids`, `security_seed` (collecte de graines), **`UDS_Fuzz` qui teste l'aléa des graines par réinitialisations de calculateur en rafale**, plus DoIP et XCP avec dump mémoire.

Deux lectures, opposées :

1. **`dump_dids` est l'art antérieur direct de `probe_dids`.** Le balayage de DID par `$22` est une technique établie, publiée, pas une idée risquée que j'aurais inventée. Ça conforte le palier 3.
2. **Le reste de l'outil fait précisément ce que `09` §6 interdit par construction** : énumérer des identifiants de service, collecter des graines `$27`, réinitialiser des calculateurs en boucle. Ce n'est pas un défaut de CaringCaribou — c'est un outil d'**exploration en sécurité offensive**, dont ce sont les fonctions légitimes. C'est un outil d'une autre catégorie que Mecabot.

⚠️ **Le détail qui vaut d'être noté :** la page du projet ne porte **aucun avertissement explicite sur les risques pour le véhicule**, alors que ses modules réinitialisent des calculateurs et énumèrent des services. Constat factuel, pas reproche. Mais il confirme que l'énumération fermée de services de `09` §6 et l'interdiction de `probe_dids` en roulant de `09` §11.7 ne sont pas de la prudence excessive : **c'est la position minoritaire dans cet écosystème, et c'est celle qui convient à un outil qu'un agent IA pilote.**

⛔ **Licence GPL-3.0 : inutilisable en dépendance**, même raison que `ecu-diagnostics` (`09` §3). Utilisable comme **référence de conception et oracle de validation ponctuelle**, comme `ELM327-emulator`.

### Le fil conducteur des trois

| Axe | État de l'écosystème |
|---|---|
| **Formats** | Ouverts et normalisés — ODX/ISO 22901, DBC |
| **Outillage** | Ouvert, mûr, permissif — `odxtools` MIT, OpenDBC MIT |
| **Données de diagnostic réelles** | **Fermées partout**, sans exception trouvée |

**Conséquence de conception :** ça ne sert à rien d'attendre une source de couche 3. Le registre de profils doit être conçu pour être **rempli incrémentalement par l'observation**, avec l'importation ODX comme chemin opportuniste si une source légitime apparaît un jour. `odxtools` étant en Python et Mecabot en Rust, l'import serait de toute façon un utilitaire hors ligne et ponctuel — pas une dépendance d'exécution. Ça tombe bien.

---

## 4. Ce qu'un profil contient

Un profil n'est pas un fichier, c'est une **résolution en cascade sur les couches 1 à 3**, plus un enregistrement local pour la couche 4.

### Couche 2 — profil de marque

Petit, stable, et c'est celui qui débloque le palier 2 :

| Champ | Rôle |
|---|---|
| Plages d'adresses de diagnostic | Où sonder pour découvrir des modules, au lieu de balayer aveuglément |
| Bus secondaires et brochage J1962 | Le seul fait de couche 1 non auto-détectable. Ford : MS-CAN sur 3/11 à 125 kbit/s. |
| Comportement de session | Faut-il `$10` étendu avant `$22` ? Le module s'endort-il ? `$3E` est-il nécessaire ? |
| **`gateway`** | `absent` · `present` · `unknown` — voir §6, c'est un champ de sûreté |
| **`discovery_allowed`** | Le balayage de DID est-il autorisé sur cette marque ? **Défaut : non.** Voir §6. |
| Famille d'algorithme `$27` | Purement documentaire — pour dire *pourquoi* le palier 4 est fermé, jamais pour l'ouvrir |

### Couche 3 — profil de modèle

| Champ | Rôle |
|---|---|
| Inventaire de modules attendus | Adresse, nom, bus. Permet de dire « le module X n'a pas répondu » plutôt que « rien trouvé ». |
| Définitions de DID | Module, DID, nom canonique **en anglais**, formule, unité — et les deux champs de §5 |
| Familles de DTC connues | Codes non génériques rencontrés et leur signification |

### Couche 4 — enregistrement d'exemplaire, local, jamais publié

VIN, As-Built, CALID/CVN de référence, options, **modifications déclarées**, familles de DTC supprimées et pourquoi, historique d'entretien saisi à la main, et toutes les mesures.

⛔ **Cette couche ne quitte jamais la machine.** Si le projet est publié un jour, la séparation couche 3 / couche 4 est ce qui rend la publication possible sans divulguer un VIN, un odomètre ou un historique. **C'est bon marché à respecter maintenant et coûteux à rétro-ajuster** — donc c'est acté maintenant, même si la décision de diffusion reste ouverte (§7).

---

## 5. La correction que le rescope a révélée : provenance ≠ validation

`10` §5 impose une enveloppe de provenance sur chaque grandeur, avec un champ `validation` portant « la méthode ayant établi le décodage ». `09` §7 demande de même que `did_definition` porte « la méthode de validation et la source ».

**J'ai conflué deux questions différentes**, et le mono-véhicule le masquait parce que la réponse à la seconde était toujours « moi, localement, non publié ».

| Question | Champ | Valeurs |
|---|---|---|
| **Comment sait-on que ce décodage est juste ?** | `validation_method` | corrélation Mode 01 · vérité terrain physique · comportement caractéristique · recoupement outil tiers · **invariance (négatif)** |
| **D'où vient ce décodage, et ai-je le droit de le redistribuer ?** | `provenance` | `iso_standard` · `sae_standard` · `own_observation` · `community_published` · `odx_import` · **`proprietary_crosscheck`** |

Ce sont des axes **orthogonaux**. Un décodage peut être solidement validé et non redistribuable ; il peut être de provenance publique et mal validé.

**Règle qui découle :** toute entrée de couche 3 dont la `provenance` est `proprietary_crosscheck` est marquée **non redistribuable** et exclue par construction de tout export de profil. C'est la version applicable par le schéma de la frontière que `01` §2 posait en prose — *utiliser FORScan sur son propre véhicule pour valider ses propres décodages est un usage normal ; extraire ou redistribuer sa base de PID ne l'est pas.*

En mono-véhicule c'était une discipline personnelle. En multi-marques avec publication possible, **c'est une contrainte de schéma**, et le fait de la mettre dans le schéma est ce qui la rend fiable.

> Ceci vaut identiquement pour toute source communautaire : une définition de PID publiée sur un forum par son auteur est `community_published` et redistribuable dans l'esprit de sa publication ; une définition extraite d'une base d'un outil commercial ne l'est pas, quel que soit le chemin par lequel elle est arrivée sur le forum. Le champ enregistre ce qu'on sait, et `unknown` est une valeur honnête.

---

## 6. La surface de sûreté que le multi-marques ajoute

Le mono-véhicule permettait de raisonner sur un cas concret : pas de passerelle, adressage connu, 500 kbit/s, tolérance au balayage constatée. **Aucune de ces prémisses ne tient sur un véhicule arbitraire**, et c'est le vrai coût du rescope. Quatre points nouveaux.

### 6.1 Passerelles sécurisées — le fait nouveau

Le Transit 2016 n'en a pas (`01` §2), ce qui était « ton meilleur coup de chance ». **Sur le parc en général, c'est l'exception qui devient la règle.** FCA a introduit sa passerelle sécurisée en MY2018 (Wrangler JL, Ram 1500) et d'autres constructeurs ont suivi.

**AutoAuth** [VÉRIFIÉ 2026-08-25] est le service industriel qui authentifie un technicien et son outil auprès de ces passerelles. Le tarif est étonnamment bas : **5 $/mois en offre Standard** (facturée à l'année), 17,50 $ en PLUS.

⛔ **Et c'est inaccessible quand même**, pour une raison qui n'est pas le prix : l'accès passe par des **« certified AutoAuth Tool Partners »**, un programme de certification d'éditeurs d'outils. Un projet personnel en Rust n'est pas un partenaire certifié. Le site invite explicitement les fabricants d'outils indépendants à le devenir — c'est une porte de constructeur d'outils, pas de particulier.

**[NON VÉRIFIÉ] et important :** le site ne distingue pas explicitement ce qui exige l'authentification. Il parle de « unlock protected vehicle systems » et de « protected functions » — ce qui suggère que **la lecture de diagnostic normalisée reste probablement accessible sans authentification même derrière une passerelle**, et que seules les fonctions protégées (écriture, bidirectionnel) sont bloquées. **Si c'est exact, l'impact sur Mecabot est faible**, puisque Mecabot est en lecture seule par construction. À vérifier avant d'affirmer quoi que ce soit à un utilisateur.

**Conséquence de conception, indépendante de cette vérification :**

- Le profil de marque porte `gateway: absent | present | unknown`.
- Sur `present` ou `unknown`, **le serveur annonce l'incertitude plutôt que d'échouer par surprise.** L'agent doit pouvoir dire « ce véhicule a une passerelle, le palier 1 devrait marcher, le palier 3 probablement pas » avant d'essayer.
- La taxonomie d'erreurs de `10` §4 gagne une condition distincte : **refus par la passerelle**, à ne pas confondre avec `7F .. 33 securityAccessDenied` d'un module. Ce ne sont pas les mêmes couches, et l'agent ne doit pas conclure la même chose.

### 6.2 La découverte devient opt-in par profil

`09` §11.7 interdit `probe_dids` en roulant. Ça ne suffit plus.

Sur le Transit, je pouvais raisonner : véhicule connu, pas de passerelle, palier 2 constaté, communauté FORScan active. **Sur un véhicule arbitraire, je ne peux rien affirmer** sur ce que fait un module inconnu face à 65 536 requêtes `$22`.

**Règle :** `probe_dids` exige que le profil de marque porte `discovery_allowed: true`, et **le défaut est `false`**. Il faut donc que quelqu'un ait affirmé, par observation, que cette marque tolère un balayage — l'affirmation étant elle-même de la donnée de couche 2, avec sa provenance.

Ça inverse la charge : la découverte n'est pas une capacité générale que certains cas restreignent, c'est une capacité fermée que des profils ouvrent.

### 6.3 Variantes de transport hors périmètre

À déclarer explicitement plutôt qu'à découvrir en production :

| Variante | Statut |
|---|---|
| ISO 15765-4 CAN 11 bits 500 kbit/s | ✅ le cas nominal |
| CAN 29 bits, 250 kbit/s | ✅ auto-détecté par l'adaptateur, sans travail |
| Protocoles pré-CAN (ISO 9141-2, KWP2000, J1850 PWM/VPW) | ⚠️ gérés par le firmware STN, **jamais testés ici**. Véhicules nord-américains avant ~2008. |
| **J1939 — poids lourds** | ⛔ **hors périmètre.** Autre couche applicative, autres PGN, autre monde. À dire, pas à tenter. |
| DoIP — diagnostic sur IP | ⛔ hors périmètre. Véhicules récents, autre transport. |

### 6.4 Véhicules électriques et hybrides

La **lecture** ne pose aucun problème nouveau. Mais la famille d'analyse E de `09` §7 — préparation d'intervention — devient sensible : une procédure touchant un circuit haute tension n'est pas du même ordre qu'un changement de filtre. **Règle :** sur un profil marqué haute tension, `search_service_docs` et le prompt de préparation d'intervention doivent faire remonter l'avertissement de sûreté de la source **avant** la procédure, jamais après. C'est une règle de présentation, pas une restriction de capacité.

---

## 7. Les décisions du 2026-08-25

| # | Décision | Portée |
|---|---|---|
| 1 | **Aucun code** — inchangé, réaffirmé | Tout le dossier reste de la préconception |
| 2 | **Le code vivra dans ce répertoire** (`bascanada/mecabot`) | ⚠️ Ce n'est **pas encore un dépôt git**. À initialiser avant la première ligne, pas au moment de l'écrire. |
| 3 | **Noms canoniques en anglais**, `snake_case` | Clôt un point ouvert de `10` §11. Aligné J1979 et conventions Rust. Descriptions et sorties vers l'agent en français. |
| 4 | **Tous les clients MCP visés** | Voir ci-dessous — ça a une conséquence réelle |
| 5 | **Pi embarqué repoussé en fin de parcours** | `09` §11 reste valide, descend en priorité |
| 6 | **Multi-marques dès la conception** | Ce document |
| 7 | **Diffusion non décidée** | Voir ci-dessous — ça a une conséquence immédiate malgré l'indécision |

**Sur le point 4 — viser tous les clients MCP est une contrainte, pas une absence de contrainte.** Ça veut dire s'en tenir au noyau largement implémenté — outils, ressources, prompts, `outputSchema` — et traiter tout le reste en **amélioration progressive** : la fonctionnalité doit rester correcte quand le client ne la gère pas. Ça renforce deux décisions déjà prises pour d'autres raisons : ne dépendre de l'elicitation pour rien (`10` §8), et ne jamais compter sur les annotations comme garde-fou (`03` §6). Le seul résidu à surveiller est `resource_link` côté `rmcp`, qui ne concerne que `render_signals` (`09` §3).

**Sur le point 7 — l'indécision sur la diffusion ne dispense pas de deux disciplines**, parce qu'elles coûtent peu maintenant et cher plus tard :

1. **La séparation couche 3 / couche 4** (§4). C'est ce qui permettra de publier des profils sans publier un VIN.
2. **Le champ `provenance`** (§5). Rétro-attribuer une provenance à des centaines de définitions de DID accumulées est infaisable ; le noter à la création est gratuit.

⚠️ **Un piège de l'indécision, à nommer :** repousser le choix de licence est sans risque **sauf** si une dépendance contaminante entre entre-temps. C'est déjà la raison pour laquelle `ecu-diagnostics` et CaringCaribou sont écartés. **Règle simple qui préserve toutes les options sans rien décider : n'introduire aucune dépendance GPL.** Les crates visés en `09` §3 sont tous permissifs, donc la règle ne coûte rien aujourd'hui.

---

## 8. Le risque que tu as identifié : abstraire avant de savoir

Il faut le traiter, parce que c'est le mode de défaillance principal de cette voie et que le dossier a déjà un précédent : `08` a été écrit en entier sur une prémisse fausse.

**La forme que prendrait l'échec ici :** concevoir un schéma de profil élaboré, générique, paramétré — dérivé d'**un seul** véhicule observé, et faux dès le deuxième. On aurait alors payé le coût de l'abstraction sans en obtenir la généralité, ce qui est strictement pire que le mono-véhicule.

Quatre garde-fous.

**1. Le modèle en couches réduit la surface risquée à une seule couche.** Les couches 0 et 1 sont normalisées — il n'y a rien à abstraire, seulement à implémenter. La couche 4 est un enregistrement local, sans généralité à trouver. La couche 2 tient en une poignée de champs. **Tout le risque d'abstraction prématurée est concentré dans le schéma de DID de couche 3** — et par chance c'est la couche qu'on peut différer le plus longtemps, puisqu'elle n'arrive qu'au palier 3.

**2. La règle des deux instances.** Aucun champ de profil n'est généralisé à partir d'un seul véhicule observé. Tant qu'il n'existe qu'un profil, **les champs de couche 2 et 3 sont écrits en dur dans le profil Ford** et non promus en schéma général. Le schéma se dégage au deuxième profil, quand on voit ce qui varie. Ça retarde l'abstraction jusqu'au moment où on a l'information de la faire.

**3. Le profil est versionné, et son évolution est prévue.** Un profil porte un numéro de version de schéma. Il est admis dès maintenant que les premières versions seront fausses, et le chargeur doit refuser un profil de version inconnue plutôt que l'interpréter au mieux.

**4. Aucun coût sur le chemin critique.** La feuille de route de `09` §9 ne change pas d'ordre. Les étapes 1 à 5 — pilote série, serveur MCP, palier 1, palier 2, stockage — sont de la **couche 0-1-2**, donc quasi indifférentes au multi-marques. Le multi-marques ne coûte que dans le nommage et le rangement, pas en travail supplémentaire.

**Ce que ça donne concrètement :** on écrit le même logiciel qu'avant, on ne code aucun fait sur le Transit dans le code, et on ne conçoit le schéma de couche 3 que le jour où un deuxième véhicule le rend possible. **Le multi-marques est une contrainte de discipline maintenant, et une capacité plus tard.** C'est le seul ordre viable.

---

## 9. Effet sur le reste du dossier

| Document | Effet |
|---|---|
| `00` | En-tête à reformuler : le véhicule cible devient le **véhicule de référence n°1**. Index et ordre de lecture à mettre à jour. |
| `01` | **Valide, et sa portée s'étend.** §2 (UDS `$27`) explique la frontière du palier 4 sur toute marque. §4 (coûts) et §5 (CASIS) restent Ford/Canada — à lire comme un **cas d'étude d'accès aux données**, désormais, pas comme la seule géographie. |
| `02` | Devient le **profil de référence n°1** : couche 3 Ford V363 + couche 4 de cet exemplaire. Reste entièrement valide, change de statut. Ce n'est plus la colonne vertébrale du dossier. |
| `03` | Valide, inchangé. La décision 4 (tous les clients) en renforce §6. |
| `04` | Valide, inchangé et indépendant du véhicule. |
| `05` | Valide. §4 (art antérieur) est complété par §3 ci-dessus. |
| `06` | Valide. |
| `07` | **Les groupes de questions sur le véhicule sont garés, pas abandonnés** — ce sont désormais des tâches de remplissage du profil n°1, pas des blocages de conception. **L'item 4bis reste le seul véritable préalable à l'écriture de code**, et il ne dépend pas du rescope. |
| `08` | Supersédé, inchangé. |
| `09` | §1 (décisions) et §4 (paliers) à mettre à jour. Le reste — cœur sans E/S, `trait Transport`, STN, sûreté, stockage, analyse — était **déjà indépendant du véhicule**. |
| `10` | §6 à recaster : les « deux précautions propres à ce véhicule » deviennent des mécanismes pilotés par le profil. Voir ci-dessous. |

### Le cas de `10` §6, qui mérite un mot

Cette section portait deux précautions présentées comme spécifiques au Transit *deleted*. Le rescope révèle qu'elles ne sont pas de la même nature :

1. **« `read_all_dtc` doit avertir sur les familles de DTC supprimées »** — réellement spécifique à l'exemplaire, donc **couche 4**. Se généralise proprement : un profil d'exemplaire peut déclarer des familles de DTC supprimées, avec un motif (calibration modifiée, variante de marché, option non montée), et l'outil émet l'avertissement d'absence de preuve dès qu'il en existe. Le Transit devient une instance de ce mécanisme, et le mécanisme sert aussi les véhicules d'origine — une option non montée produit exactement le même silence trompeur.

2. **« un DID invariant est suspect, pas validé »** — **ça n'a jamais été spécifique à ce véhicule.** Un DID qui ne varie sur aucune session à travers des points de fonctionnement différents n'est pas validé, quelle que soit la raison. Le delete en était une cause possible parmi d'autres. À sortir des précautions locales et à promouvoir en **cinquième méthode de validation, négative**, dans la méthodologie générale de `09` §4.

Le second point est un petit gain de justesse obtenu gratuitement par le rescope : une règle générale était rangée comme un cas particulier.

---

## 10. Ce que ce document ne décide pas

- **Le schéma concret de couche 3.** Volontairement différé — c'est la règle des deux instances de §8. Le décider maintenant serait précisément l'erreur que §8 cherche à éviter.
- **Le format de sérialisation des profils** (TOML, JSON, SQLite). Choix de faible conséquence, à prendre au moment d'écrire le chargeur.
- **La licence.** Non décidée, avec la seule contrainte de §7 : aucune dépendance GPL.
- **La résolution profil → véhicule.** Le décodage de VIN est gratuit par l'API vPIC de la NHTSA (`00`, `01` §4) et donne marque, modèle et année. Reste à décider ce que fait le serveur quand aucun profil ne correspond — **la réponse par défaut doit être « palier 1 seulement, et le dire », jamais « appliquer le profil le plus proche »**. Appliquer un profil au mauvais véhicule produirait des grandeurs plausibles et fausses, ce qui est le pire mode de défaillance du dossier. La règle est décidée ; son implémentation ne l'est pas.
