# 09 — Architecture retenue : serveur MCP Rust multiplateforme

**Date :** 2026-08-24
**Remplace :** `08-architecture-forscan-bridge.md`, dont la prémisse était erronée.

L'objectif est **l'équivalent multiplateforme de FORScan, en lecture, avec un agent IA branché directement dessus**. La motivation est concrète : FORScan ne tourne que sous Windows, ce qui impose une VM et interdit tout usage hors du poste de travail.

---

## 1. Décisions verrouillées

| Décision | Choix | Conséquence |
|---|---|---|
| Matériel | **OBDLink EX (STN2120), déjà en main et fonctionnel sur macOS** | Aucun achat. Le poste est déjà capable. |
| Interface | **Serveur MCP seul — l'agent est l'interface** | Aucune UI à construire. Le plus court chemin vers l'utile. |
| Langage | **Rust** | Binaire unique sans runtime, distribution triviale sur les trois OS. |
| Accès véhicule | **Lecture seule, par construction** | Aucun chemin de code vers un service d'écriture. Voir §6. |
| ~~Périmètre véhicule~~ | ~~**Un seul : Transit 350HD 2016**~~ | ⛔ **Révisé le 2026-08-25 — voir la ligne suivante.** |
| **Périmètre véhicule** | **Multi-marques dès la conception. Aucun fait sur un véhicule particulier n'est écrit dans le code.** | La connaissance véhicule vit en **données**, sur cinq couches. Le Transit devient le **profil de référence n°1**, pas le périmètre. Modèle complet en **`11`**. |
| **Nommage** | **Noms canoniques en anglais, `snake_case`** | Aligné J1979 et conventions Rust. Descriptions et sorties vers l'agent en français. Clôt un point ouvert de `10` §11. |
| **Clients MCP** | **Tous visés** | Contrainte, pas absence de contrainte : s'en tenir au noyau largement implémenté, tout le reste en amélioration progressive. Voir `11` §7. |
| **Emplacement du code** | **`github.com/wquintal/mecabot`, dépôt public** | ✅ Dépôt initialisé le 2026-08-26, branche `main` protégée, vérification `docs` obligatoire. Compte personnel : ni `bascanada` ni `estran-studio` ne sont des organisations pertinentes ici. |
| **Diffusion / licence** | **Non décidée** | Une seule contrainte qui préserve toutes les options : **aucune dépendance GPL**. Depuis le 2026-08-26 ce n'est plus seulement une phrase : `deny.toml` l'exprime en liste d'autorisation. Mais la porte est **écrite, pas encore exercée** — `cargo-deny` dort derrière une tâche `guard` tant qu'il n'y a pas de `Cargo.toml`, donc aucune dépendance n'a jamais été refusée par elle. Voir `11` §7 et `12` §6.3 pour la même distinction appliquée aux relecteurs. |
| ~~Relecture du diff~~ | ~~**Trois relecteurs de code, pas de porte humaine**~~ | ⛔ **Révisé le 2026-08-27 après mesure — voir la ligne suivante.** |
| **Relecture du diff** | **Un relecteur agent — CodeRabbit — et la porte de relecture humaine maintenue** | La règle punt-kit *« There is no human code-review gate »* **n'est pas adoptée** : elle supposait une flotte de trois relecteurs, qui n'a pas tenu la mesure (Gemini jamais installé ici, `claude-review` désactivé pour son coût d'API). **Tout diff se relit à la main avant fusion**, CodeRabbit venant en plus et non à la place. `cargo-deny` est une porte de dépendances, pas un relecteur, et dort jusqu'au premier `Cargo.toml`. Voir `12` §6.3. |
| **Propriété du transport** | **Un seul processus détient le port série par machine. Les autres rôles sont ses clients.** | `mecabot log` n'ouvre pas le port quand `mecabot serve` tourne : il lui parle par `--remote`. Unifie trois questions qui étaient traitées séparément — les rôles de §11.2, le mode embarqué de §11.3, et le drapeau `--remote`. Décidé le 2026-08-26, voir §11.2. |
| **Emplacement des données d'exécution** | **Répertoires natifs de la plateforme, sous le nom `mecabot`** | ⛔ **Pas `~/.punt-labs/`** — ce namespace appartient à l'outillage punt-labs réellement employé dans estran-studio, et y ranger un projet personnel le ferait mentir. Décidé le 2026-08-26, voir §7. |
| FORScan | **Oracle de développement, pas dépendance d'exécution** | Utilisé en VM pour valider des décodages. L'application n'en a jamais besoin pour tourner. |
| Déploiement | **Poste de travail d'abord, Raspberry Pi Zero embarqué ensuite** — **repoussé en fin de parcours le 2026-08-25** | Deux rôles distincts pour le même cœur, et un changement de transport MCP. §11 reste valide, descend en priorité. |

---

## 2. La tension « browser based » vs « serveur MCP », et sa résolution

Tu as demandé du web/navigateur **et** un serveur MCP. Les deux ne se combinent pas directement, et il vaut mieux savoir pourquoi maintenant.

**Le problème :** un serveur MCP en stdio est un processus local que le client (Claude Code, Claude Desktop…) lance comme sous-processus. **Une page web ne peut pas être lancée comme sous-processus.** Et à l'inverse, seule la page navigateur peut détenir un port série ouvert via l'API Web Serial. La donnée serait donc d'un côté et le protocole de l'autre.

**Deuxième problème, indépendant :** l'API **Web Serial est réservée à Chromium**. Safari et Firefox ont explicitement refusé de l'implémenter. « Browser based » sur ton Mac signifie donc Chrome ou Edge, pas Safari. [NON VÉRIFIÉ sur l'état exact en 2026 — la position d'Apple et de Mozilla est constante depuis plusieurs années, mais à revérifier avant de s'y engager.]

### La résolution : cœur compilable en deux cibles

C'est idiomatique en Rust, et ça satisfait les deux souhaits sans compromis :

```text
┌─────────────────────────────────────────────────────┐
│  CŒUR — Rust pur, aucune E/S                        │
│  encodage/décodage UDS · assemblage des réponses    │
│  base de DID · décodeurs · détection d'événements   │
│                                                      │
│  → compile en natif ET en WASM                      │
└─────────────────────────────────────────────────────┘
             │                          │
   trait Transport            trait Transport
             │                          │
┌────────────▼───────────┐  ┌───────────▼────────────┐
│ NATIF                  │  │ WASM (plus tard)       │
│ crate `serialport`     │  │ Web Serial via web-sys │
│ → macOS/Linux/Windows  │  │ → Chromium seulement   │
│ → SERVEUR MCP (stdio)  │  │ → UI navigateur        │
└────────────────────────┘  └────────────────────────┘
```

Le cœur ne fait que du calcul : il transforme des octets en grandeurs et des grandeurs en conclusions. Il ne sait pas d'où viennent les octets. Un `trait Transport` isole l'E/S, et il en existe deux implémentations.

**Ce qu'on livre maintenant :** le binaire natif, serveur MCP en stdio. C'est ce qui répond à ton besoin réel — sortir de la dépendance Windows — et c'est **plus** portable qu'une page web, puisque ça marche sans Chromium.

**Ce qu'on garde ouvert :** la cible WASM. Le jour où tu veux une UI navigateur ou du zéro-installation, le cœur est déjà prêt ; seul le transport change.

Un binaire Rust est par ailleurs le meilleur format de distribution possible ici : un fichier, pas d'interpréteur, pas d'environnement virtuel, pas de dépendance système. Trois cibles de compilation depuis la même source.

---

## 3. Ce que le STN2120 fait à ta place

Bonne nouvelle qui allège beaucoup le choix de Rust, et que `05` §3.2 laissait dans l'ombre.

Mon inquiétude sur l'immaturité de l'écosystème Rust portait surtout sur l'absence de pile ISO-TP utilisable (`ecu-diagnostics` en GPL-3.0 avec un build docs.rs cassé, `socketcan` Linux-only). **Mais avec un adaptateur STN, tu n'implémentes pas ISO-TP** : le firmware de l'adaptateur gère la segmentation multi-trames, les trames de contrôle de flux et le réassemblage. Tu envoies `2201A0` et tu récupères la réponse assemblée.

Ce qui reste à écrire est donc beaucoup plus modeste que dans le cas général :

| Couche | Effort en Rust |
|---|---|
| Pilote série + commandes AT/ST (init, `ATSH`, commutation de bus, timeouts) | modéré — c'est le vrai travail de bas niveau |
| Assemblage des lignes de réponse de l'adaptateur | faible |
| Encodage/décodage des services UDS ($22, $19, $3E, $10) | faible — quelques structures, sémantique documentée dans ISO 14229 |
| ISO-TP | **délégué à l'adaptateur** |
| Pile CAN brute | **non nécessaire** (sauf palier 3 avancé) |

⚠️ [NON VÉRIFIÉ] : jusqu'à quelle taille de réponse le STN2120 assemble correctement, et son comportement exact sur les réponses très longues. À caractériser tôt.

> ### ⚠️ Précisé le 2026-08-28 — la délégation d'ISO-TP est **conditionnelle**
>
> *« Tu envoies `2201A0` et tu récupères la réponse assemblée »* reste vrai, mais **pas dans la configuration de sortie d'usine de l'adaptateur**. Le manuel du fabricant (*OBDLink Family Reference and Programming Manual*, rév. F, 2025-08-29) donne quatre défauts qui mordent, tous `[VÉRIFIÉ]` :
>
> | Condition | Défaut d'usine | Sans elle |
> |---|---|---|
> | Un préréglage **ISO 15765** (`33`/`34`/`53`/`54`) | selon `ATSP` | Les préréglages `51`/`52` sont de l'**ISO 11898 brut** : rien n'est réassemblé tant que `STCSEGR 1` n'est pas posé |
> | `ATCAF1` — formatage automatique CAN | ✅ actif | Pas de génération de PCI, pas de retrait des PCI en réponse |
> | `ATAL` — messages longs autorisés | ⛔ **`ATNL`** | La limite J1979 de **7 octets de données est appliquée en réception** : une réponse UDS longue est refusée avant d'être réassemblée |
> | Paires de contrôle de flux cohérentes | automatiques, `TxID = RxID − 8` | Un module dont l'identifiant ne suit pas la convention ne reçoit **jamais** sa trame de contrôle de flux → `FC RX TIMEOUT`. Et ⚠️ **ajouter une paire par `STCFCPA` coupe l'automatisme pour toutes les autres** |
>
> **Conséquence sur la machine à états d'adaptateur de `10` §1 :** ces quatre réglages sont de l'**état à tenir**, pas une initialisation qu'on pose et qu'on oublie. Un choix de préréglage décide si la couche ISO-TP existe.
>
> La borne de réassemblage, elle, est **partiellement publiée** : `[VÉRIFIÉ]` **4 ko de taille de message** pour l'OBDLink EX, ce qui recoupe le plafond de **4095 octets** du champ de longueur sur 12 bits de l'ISO 15765-2 classique. Ce que le manuel ne dit pas, c'est le comportement **à** la borne — erreur nommée (`OUT OF MEMORY`, `BUFFER FULL`) ou troncature silencieuse. ⚠️ **La troncature silencieuse serait le pire cas**, parce qu'une réponse tronquée décode en valeurs plausibles et fausses. Mesuré au bloc C de `13`.

### Bibliothèques Rust visées

| Besoin | Crate | Maturité |
|---|---|---|
| Port série multiplateforme | `serialport` | mûre, multiplateforme réel |
| Serveur MCP | `rmcp` (SDK Rust officiel) | ✅ **[VÉRIFIÉ 2026-08-25] — 3.1.4, Apache 2.0.** Voir ci-dessous. |
| Stockage | `rusqlite` | mûre |
| Rendu PNG des tracés | `plotters` | mûre — sert le levier multimodal de `04` §3.1 |
| Sérialisation | `serde` / `serde_json` | mûre |
| Cible navigateur (plus tard) | `wasm-bindgen`, `web-sys` | mûres |

À **ne pas** utiliser en dépendance : `ecu-diagnostics` (GPL-3.0 contaminante, build cassé), `socketcan` (Linux-only), `OpenVehicleDiag` (abandonné 2021), **CaringCaribou** (GPL-3.0 — utile comme référence de conception et oracle, jamais liée ; voir `11` §3).

⚠️ **Règle de licence adoptée le 2026-08-25 :** la diffusion du projet n'est pas décidée, et **aucune dépendance GPL** n'est introduite tant qu'elle ne l'est pas. C'est ce qui garde toutes les options ouvertes sans rien décider — et ça ne coûte rien, tous les crates du tableau ci-dessus étant permissifs.

### `rmcp` — vérifié le 2026-08-25, et c'est une bonne nouvelle

C'était le dernier `[NON VÉRIFIÉ]` pesant sur la faisabilité du plan. Résultat :

| Point | Constat |
|---|---|
| Version | **3.1.4**, publiée le 2026-08-20. Dépôt `modelcontextprotocol/rust-sdk` — **c'est bien le SDK officiel**. ~22 M de téléchargements cumulés. |
| Licence | **Apache 2.0** — aucune contrainte de contamination, contrairement à `ecu-diagnostics` |
| Révision de spec | **Implémente `2026-07-28`**, la révision courante identifiée en `03` §1, avec compatibilité descendante jusqu'à `2025-11-25`. La **3.0.0 est datée du 2026-07-28 elle-même** : le SDK suit la spec de près. |
| `outputSchema` / `structuredContent` | ✅ Supportés, et **assouplis** : `outputSchema` accepte n'importe quel type JSON Schema, pas seulement `object` |
| `notifications/progress` | ✅ Supportées, avec message et total d'éléments — exactement ce que `probe_dids` demande (`10` §7) |
| Elicitation | ✅ Supportée, en mode formulaire et en mode URL. Sans objet ici (`10` §8), mais disponible. |
| Transport stdio | ✅ Client et serveur |
| Transport Streamable HTTP | ✅ Serveur (service Tower) et client (reqwest) — **c'est la voie du Pi embarqué** (§11.3) |
| Transport in-process / worker | ✅ **Utile pour les tests** — voir `10` §9 |
| `resource_link` | ⚠️ [NON VÉRIFIÉ] — pas mentionné explicitement dans le README. Seul résidu ; concerne `render_signals`. |

**Conséquence :** aucune couche de protocole à écrire soi-même, et le transport HTTP du palier « Pi embarqué » est déjà couvert par le même SDK. La §11.3 n'a plus d'inconnue côté bibliothèque — il reste la question de l'authentification, qui est une décision de conception, pas un manque d'outil.

---

## 4. Les quatre paliers de capacité

C'est la structure qui remplace le « mur de parité FORScan » binaire de `00`. Le mur existe toujours, mais il se franchit par étapes.

> ### ✅ Note du 2026-08-25 — les paliers étaient déjà prêts pour le multi-marques
>
> Ces quatre paliers avaient été conçus pour un seul véhicule. `11` §2 montre qu'ils sont en réalité la projection des **cinq couches de connaissance véhicule** sur l'axe des fonctionnalités : palier 1 = couche 0 (normes) seule, palier 2 = couche 2 (adressage de marque), palier 3 = couche 3 (DID de modèle), palier 4 = clés inobtenables.
>
> **Ils ne décrivaient donc pas des étapes propres au Transit, mais des besoins de connaissance.** Le rescope multi-marques ne les invalide pas, il les explique — et c'est ce qui limite l'ampleur de la réécriture. Ce qui change dans cette section : les faits Ford ci-dessous sont des données de **profil**, pas des constantes du logiciel.

### Palier 1 — OBD-II normalisé · aucune donnée propriétaire

Modes **$01** (live), **$02** (freeze frame), **$03/$07** (DTC), **$06** (monitors), **$09** (VIN). Entièrement spécifié par SAE J1979, public. 20 à 30 grandeurs utiles.

Rappel de `04` §5 : le **freeze frame est l'artefact le plus dense de tout l'OBD-II** — 12 à 20 PID capturés à l'instant exact de la panne, pour 60-120 tokens. Le document initial ne le mentionnait jamais. C'est le premier réflexe de toute session.

**Ajout du 2026-08-24 — le véhicule est *deleted*, ce qui donne à trois sondes normalisées une importance qu'elles n'avaient pas :**

| Sonde | Rôle sur ce véhicule |
|---|---|
| **`$09` InfoType `04` (CALID)** et **`06` (CVN)** | Le PCM tourne sur une calibration modifiée. Le CVN est une somme de contrôle dont **la raison d'être réglementaire est de détecter une reprogrammation** : à archiver comme ligne de base, un changement signifierait que quelqu'un a reflashé. |
| **`$01` PID `01`** | Bits de support des moniteurs — les moniteurs d'après-traitement désactivés par le tune s'y voient. |
| **`$06`** | Résultats des moniteurs non continus : **l'empreinte la plus fine de ce que la calibration a coupé**, pour ~600 tokens. |

Avec la couche d'analyse par-dessus, ce palier seul dépasse n'importe quel scanner grand public. **Et sur un véhicule tuné, il fait quelque chose qu'aucun scanner grand public ne fait : il caractérise le logiciel avant d'interpréter les données.**

### Palier 2 — inventaire de modules et DTC de tout le véhicule · toujours rien de propriétaire

**Découverte des modules :** on sonde les adresses CAN de diagnostic et on note qui répond. La sonde la plus propre est `$22 F190` (VIN) ou `$3E` — des DID et services **normalisés par ISO 14229**, donc sans surprise.

Hypothèse de départ pour les adresses, à valider : Ford utilise 0x7E0-0x7E7 pour le groupe motopropulseur, et des adresses dispersées pour la carrosserie. Le dépôt `jakka351/FG-Falcon` documente 0x720 (IPC), 0x727 (carrosserie), 0x767 (ABS), 0x7A6 (FDIM) sur Falcon australien [COMMUNAUTÉ] — conventions d'adressage Ford de la même époque, donc **point de départ plausible, pas vérité**.

**DTC de tous les modules :** `$19 ReadDTCInformation` (sous-fonction 0x02, reportDTCByStatusMask) sur chaque module trouvé, sur les deux bus. Les codes reviennent en hexadécimal. Les P0xxx génériques sont décrits par SAE J2012 (public) ; les codes Ford B/C/U s'accumulent à la main au fil des rencontres.

**Ce palier livre déjà une grosse part de ce que tu vois à l'écran dans FORScan** — et il ne dépend d'aucune donnée réservée. C'est le premier livrable qui justifie le projet.

⚠️ **Précaution obligatoire sur ce véhicule, ajoutée le 2026-08-24.** Le véhicule est *deleted*, donc sa calibration **supprime activement des familles de DTC** (P24xx, P20xx, P046x…). `read_all_dtc` doit accompagner sa sortie d'un avertissement explicite : *l'absence de code dans ces familles ne prouve rien ici*. Sans ça, l'agent conclura « aucun défaut » à partir d'un silence fabriqué. C'est le seul endroit du dossier où un outil correct produirait un raisonnement faux.

### Palier 3 — les DID étendus · le vrai travail

C'est ici que vivent les grandeurs qui comptent sur ton diesel. **Liste révisée le 2026-08-24 pour la configuration *deleted*** — la cible initiale était surtout l'après-traitement, elle se déplace vers ce qui reste et se fait solliciter :

| Cible | Statut |
|---|---|
| **Corrections de débit par injecteur (×5)** | ✅ **la cible n°1**, et le seul signal qui recoupe le rappel 16V618000 |
| Pression de rampe réelle vs demandée | ✅ |
| Position d'aubes de turbo réelle vs demandée | ✅ |
| Températures de la 6R80 | ✅ — une calibration qui augmente le couple la sollicite davantage |
| Position EGR réelle vs demandée | ⚠️ seulement si l'EGR est conservé — à vérifier physiquement |
| ~~Charge de suie, pression différentielle DPF, EGT ×4, compteurs de régénération~~ | ⛔ retirés avec l'après-traitement |

**Moins de grandeurs à découvrir qu'en configuration d'origine, donc un palier 3 plus court** — mais les grandeurs restantes sont celles qui coûtent cher quand elles lâchent. Détail complet en `02` §6 révisé.

⚠️ **Le cas du delete, qui n'était qu'une instance d'une règle générale.** Un DID peut avoir survécu dans la calibration tout en lisant un capteur physiquement absent : il renverra alors une valeur plausible et constante, ou du bruit. J'avais rangé ça comme un piège propre à ce véhicule. **C'est faux — c'est la méthode 5 ci-dessous, et elle vaut sur tout véhicule** (`11` §9). Un capteur retiré par un delete, une option non montée en usine, un DID hérité d'une autre variante de marché : les trois produisent la même valeur figée et trompeuse.

**Trouver les DID est mécanique.** `$22` prend un DID de 2 octets, soit 65 536 possibilités par module. Un module répond soit positivement (`62` + DID + données), soit par un refus (`7F 22 31`, requestOutOfRange). Même à 20 requêtes/seconde, un module complet se balaie en moins d'une heure. Les DID `F180-F1FF` sont normalisés par ISO 14229 et servent de contrôle de bon fonctionnement du balayage.

**Interpréter les octets est le problème difficile** — facteur d'échelle, offset, unité, sémantique. **Cinq méthodes**, par ordre de solidité — les quatre premières positives, la cinquième négative :

1. **Corrélation avec le Mode 01.** Tu connais déjà le régime, la charge et la température par des PID normalisés. Un DID dont la valeur suit le régime **est** le régime. Ça calibre gratuitement plusieurs dizaines de DID.
2. **Vérité terrain physique.** Thermomètre, manomètre, position de pédale, ventilateur qui s'enclenche. Le plus solide, parce que ça ne dépend d'aucune source tierce.
3. **Comportement caractéristique.** Une grandeur qui monte de 0 à 100 en suivant l'accélérateur ; quatre valeurs de température qui bougent ensemble avec un décalage (les EGT) ; un compteur qui ne fait que croître (régénérations).
4. **Contre-vérification par un outil tiers** (FORScan en VM), pour les grandeurs que les trois premières méthodes n'atteignent pas. ⚠️ **Toute entrée validée ainsi porte `provenance: proprietary_crosscheck` et est non redistribuable** (`11` §5). Usage normal sur son propre véhicule ; jamais une source à republier.
5. **Invariance — méthode négative, à appliquer avant les quatre autres.** Une grandeur qui ne varie sur **aucune** session, à travers des points de fonctionnement différents, **n'est pas validée** : elle est marquée suspecte, jamais promue. Capteur absent, option non montée, DID orphelin d'une autre variante — trois causes, un seul symptôme. C'est la méthode la moins chère du lot, et la seule qui protège contre une valeur *plausible* et fausse.

**À assumer :** c'est incrémental, c'est lent, et **certains DID ne seront jamais interprétés avec confiance**. Un DID non identifié se stocke quand même — brut et horodaté. Sa tendance dans le temps reste exploitable même sans savoir ce qu'il mesure, et son sens peut se révéler plus tard.

### Palier 4 — inaccessible, et sans importance

Tout ce qui passe par `$27 SecurityAccess` : écritures de DID, As-Built en écriture, PATS, reflashing. Inchangé depuis `01` §2 — 0 entrée Ford dans la plus grande base seed/key ouverte (3 038 entrées).

Ce n'est pas une perte : ta valeur ajoutée n'a jamais été là, et FORScan reste disponible en VM pour les rares écritures dont tu auras besoin.

---

## 5. Deux pistes de données à explorer

**L'obligation ODX européenne.** Le Transit est conçu en Europe, et le cadre européen d'accès à l'information de réparation est **du règlement, pas un accord volontaire comme le CASIS** (`01` §5). Il comporte, sous une forme à préciser, une obligation de fournir les données de diagnostic dans un format normalisé — en pratique ODX. Si le portail technique Ford européen expose de l'ODX pour le V363, ça donnerait une base de DID par une voie parfaitement légitime, et `odxtools` sait déjà lire ce format.

⚠️ **[NON VÉRIFIÉ]** — je n'ai pas vérifié le texte réglementaire ni ce que le portail Ford européen offre réellement. **C'est la piste au meilleur rapport valeur/effort de recherche du dossier**, et elle pourrait raccourcir le palier 3 de plusieurs mois. À creuser avant de lancer un balayage de DID manuel.

> ### Mise à jour du 2026-08-25 — l'outillage ODX est vérifié, la donnée reste introuvable
>
> **`odxtools`** (Mercedes-Benz, **MIT**, version **11.5.4 publiée le 2026-08-24**) parse ODX/PDX, encode des requêtes, décode des réponses et même une session live. Outillage libre, mûr, très actif. ODX est normalisé **ISO 22901**, maintenu par **ASAM e.V.**
>
> ⛔ **Mais sa documentation ne dit nulle part où obtenir des fichiers ODX/PDX de véhicules réels** — le dépôt ne livre qu'un fichier de démonstration inventé. **Le format est ouvert, l'outillage est ouvert, la donnée ne l'est pas.** C'est la forme exacte du problème FORScan, transposée.
>
> Ça ne réfute pas l'hypothèse réglementaire : ça montre qu'aucune source ouverte ne s'est matérialisée. Si l'obligation existe, la donnée est derrière un portail constructeur payant. **Le rescope multi-marques augmente la valeur de cette piste** — une obligation européenne vaudrait pour tous les constructeurs vendant en Europe. Détail et art antérieur complet en **`11` §3**.
>
> **Conséquence pratique :** ne pas attendre cette source. Le registre de profils se conçoit pour un remplissage incrémental par observation, avec l'import ODX comme chemin opportuniste. `odxtools` étant en Python, un tel import serait un utilitaire hors ligne ponctuel — jamais une dépendance d'exécution.

**Les PID personnalisés publiés par la communauté.** FORScan permet de définir ses propres PID — module, service, DID, formule de conversion, unité — et les utilisateurs **publient ces définitions** sur les forums depuis des années. C'est une provenance radicalement différente d'une extraction de base interne : ce sont des contributions volontaires de leurs auteurs, dans un format d'échange lisible. **[NON VÉRIFIÉ]** sur ce qui existe pour le Transit ; à recenser.

---

## 6. Sûreté : ce qui rend la lecture seule structurelle

`03` §6 établit que `readOnlyHint` n'est **pas** un garde-fou — la spec MCP dit explicitement que les clients doivent traiter les annotations comme non fiables. La sûreté doit donc être dans la structure du code.

**Règle d'architecture :** l'encodeur UDS n'accepte qu'une énumération fermée de services. Il n'existe aucun chemin de code capable d'émettre autre chose.

| Autorisé | Interdit — absent du code |
|---|---|
| `$01`, `$02`, `$03`, `$06`, `$07`, `$09` (OBD-II) | `$2E` WriteDataByIdentifier |
| `$22` ReadDataByIdentifier | `$2F` InputOutputControl |
| `$19` ReadDTCInformation | `$31` RoutineControl |
| `$3E` TesterPresent | `$11` ECUReset |
| `$10` DiagnosticSessionControl (defaut / extended) | `$14` ClearDiagnosticInformation |
| | `$27` SecurityAccess |
| | `$28` CommunicationControl, `$85` ControlDTCSetting |

**⚠️ Ne jamais balayer les identifiants de service.** Balayer des DID avec `$22` est sans danger — c'est un service de lecture. Balayer des **services** est dangereux : `$11` redémarre un module, `$2F` actionne des sorties physiques, `$28` peut couper la communication d'un module, `$31` lance des routines dont certaines sont destructives. La différence entre les deux est la différence entre de l'exploration et de la casse.

**Conditions d'un balayage de DID :** véhicule immobile et stationné, frein serré, contact mis moteur arrêté (ou au ralenti si les grandeurs cherchées l'exigent), et **un mainteneur de batterie** — une heure de contact mis vide une batterie. Jamais en roulant.

Les avertissements de `02` §8 restent entièrement valides, en particulier sur la régénération DPF forcée (550-700 °C, extérieur obligatoire, jamais en cas de suspicion de dilution d'huile). Cette opération n'est pas dans le périmètre de Mecabot et n'y sera jamais : elle reste dans FORScan, sous ton contrôle direct.

---

## 7. Stockage, dérivation, analyse

Cette partie est reprise de `08` §4-6, qui restait valide malgré la prémisse erronée.

**Stockage : SQLite** (`rusqlite`). Un fichier, pas de serveur. Une session de 30 min à 15 grandeurs/1 Hz fait ~27 000 échantillons ; une année de sessions mensuelles, quelques centaines de milliers de lignes. Aucune base de séries temporelles n'est justifiée.

### Où les fichiers vivent — décidé le 2026-08-26

Question que le dossier laissait ouverte, et qui se règle mal après coup parce que le chemin se retrouve dans les scripts, les sauvegardes et la documentation.

**Répertoires natifs de la plateforme, sous le nom `mecabot`**, obtenus par le crate `directories` :

| Plateforme | Données (`data/`) |
|---|---|
| macOS | `~/Library/Application Support/mecabot/` |
| Linux (dont le Pi) | `$XDG_DATA_HOME/mecabot/`, soit `~/.local/share/mecabot/` par défaut |
| Windows | `%APPDATA%\mecabot\` |

Avec trois sous-répertoires aux noms conventionnels empruntés à punt-kit, qui ne coûtent rien et sont justes : **`data/`** (la base SQLite), **`logs/`** (journal rotatif, voir `12` §4), **`cache/`** (index du corpus documentaire, reconstructible).

**⛔ Pas `~/.punt-labs/mecabot/`**, que le standard `filesystem.md` de punt-kit imposerait. Deux raisons, la seconde étant la décisive :

1. **C'est un chemin Unix qui n'a pas de sens sous Windows**, alors que le multiplateforme est une décision verrouillée de §1. Le standard punt-kit est écrit pour des outils Python sur macOS et Linux ; Mecabot vise trois OS.
2. **Ce namespace appartient à l'outillage punt-labs réellement employé dans estran-studio.** Y déposer les données d'un projet personnel ferait mentir le répertoire sur ce qui est un outil d'entreprise et ce qui ne l'est pas.

**Une chose est reprise de punt-kit, et c'est la plus utile :** *« Each tool defines a single root constant or function in one central module. All subdirectory paths derive from it — no scattered `Path.home()` calls. »* Un seul point de résolution de chemin dans tout le code. C'est aussi ce qui rend la décision ci-dessus **révisable à coût nul** : adopter plus tard un namespace `estran-studio` serait le changement d'une constante.

⛔ **Rappel de couche 4 :** le fichier de base contient `vehicle_instance`, donc du VIN et de l'As-Built. **Il ne se synchronise pas vers un service tiers.** Le transfert Pi → poste de §11.3 est un transfert local.

### Les entités

| Entité | Contenu |
|---|---|
| `session` | date, durée, odomètre, contexte libre (« bruit à froid depuis 2 semaines ») |
| `mesure` | session, nom canonique, instant, valeur |
| `did_observe` | module, DID, octets bruts, statut d'interprétation — **y compris les non identifiés** |
| `did_definition` | module, DID, nom canonique **en anglais**, formule, unité, **`validation_method` et `provenance` — deux champs distincts, voir ci-dessous** |
| `vehicle_profile` | couches 2 et 3 résolues pour ce véhicule : marque, modèle, année, adressage, bus, `gateway`, `discovery_allowed`, version de schéma (`11` §4) |
| `vehicle_instance` | **couche 4** : VIN, As-Built, CALID/CVN de référence, options, modifications déclarées, familles de DTC supprimées et motif. ⛔ **Ne quitte jamais la machine.** |
| `point_de_fonctionnement` | session, instant, casier (régime × charge × température) — dérivé |
| `evenement` | session, instant, type, signaux impliqués, amplitude — dérivé |
| `dtc` | code, module, première/dernière observation, compteur, statut |
| `intervention` | date, ce qui a été fait, pièces, kilométrage — **saisi à la main** |

Deux points importants. `did_definition` porte **la méthode de validation et la source** de chaque décodage — sans ça, tu ne sauras plus dans six mois si une valeur est solide ou devinée. Et `intervention` est ce qui permet de dire « la dérive s'est arrêtée quand tu as changé le filtre » ; aucune donnée du véhicule ne le contient.

> ### ⚠️ Correction du 2026-08-25 : « méthode de validation et source » confondait deux choses
>
> Le mono-véhicule masquait le problème, parce que la réponse à la seconde question était toujours « moi, localement, non publié ». Ce sont deux axes **orthogonaux** :
>
> | Question | Champ |
> |---|---|
> | **Comment sait-on que ce décodage est juste ?** | `validation_method` — les cinq méthodes de §4 |
> | **D'où vient-il, et ai-je le droit de le redistribuer ?** | `provenance` — `iso_standard` · `sae_standard` · `own_observation` · `community_published` · `odx_import` · `proprietary_crosscheck` · `unknown` |
>
> Un décodage peut être solidement validé et non redistribuable ; il peut être de provenance publique et mal validé. **Toute entrée en `proprietary_crosscheck` est exclue par construction de tout export de profil** — c'est la version applicable par le schéma de la frontière que `01` §2 posait en prose. Détail en `11` §5.
>
> Rétro-attribuer une provenance à des centaines de définitions accumulées est infaisable ; le noter à la création est gratuit. **C'est pourquoi ce champ existe maintenant, alors que la diffusion n'est pas décidée.**

### Les cinq familles d'analyse

**A. Suivi de dérive longitudinal — la raison d'être.** La seule chose que FORScan ne peut structurellement pas faire : il n'a aucune mémoire entre les sessions.

**Le point technique qui décide de la validité :** il faut **conditionner sur le point de fonctionnement**. Comparer des moyennes de session ne veut rien dire — une pression de rampe moyenne dépend entièrement de la façon dont tu as conduit ce jour-là. Il faut regrouper par casier (régime × charge × température) et ne comparer qu'à l'intérieur d'un même casier, entre sessions. Fait naïvement, ce module produit du bruit convaincant.

Ce qu'il révèle : l'injecteur dont la correction s'écarte des quatre autres **avant** le raté, l'écart EGR qui se creuse avant le P0401, le DPF qui se charge tôt avant le voyant, les aubes de turbo qui commencent à coller. Table complète en `02` §7.

**B. Triage de session.** 27 000 échantillons ≈ 200 000 tokens bruts → **300-600 tokens** répondant à « qu'est-ce qui sort de l'ordinaire », comparé à l'enveloppe attendue, la source étant toujours indiquée.

**C. Détection d'événements.** CUSUM sur les signaux clés (`04` §2), en code déterministe. L'agent reçoit « à t=412 s, la pression de rampe a chuté de 300 bar en 2 s à régime stable » au lieu de 27 000 lignes.

**D. Corrélation documentaire.** Symptôme ou DTC → TSB, rappel, procédure, **avec citation vers le fragment source**. Corpus de quelques centaines de pages pour un véhicule (`04` §6.5). Découpage hiérarchique : 94,1 % contre 86,2 % pour un découpage naïf, et **33 points d'écart sur les questions dépendant de tableaux** — or les couples de serrage et les brochages sont tous tabulaires.

**E. Préparation d'intervention.** Procédure, source citée, préconditions, avertissements de sûreté, et ce qu'il faut faire dans FORScan. L'humain reste l'actionneur (`01` §2.4).

**Le principe transversal :** tout calcul numérique reste en code déterministe ; le LLM reçoit des conclusions et peu de grandeurs. Ce n'est pas une optimisation de tokens, c'est une condition de justesse — `04` §3 établit que les LLM perdent l'information de magnitude et d'échelle à la tokenisation, et que retirer le composant LLM de trois méthodes de prévision par LLM **améliore** souvent le résultat.

---

## 8. Surface MCP

Transport **stdio** (`03` §8) : le serveur tourne sur la machine reliée au véhicule, pas d'authentification à configurer, isolation de processus héritée.

⚠️ **Cette phrase contient une hypothèse qui tombe dès qu'on embarque un Pi :** stdio suppose que le serveur et le client sont sur **la même machine**, puisque le client lance le serveur comme sous-processus. Un serveur qui tourne dans le véhicule est un serveur *distant*, et stdio ne peut pas l'atteindre. Voir §11.

| Outil | Rend | Ordre de grandeur |
|---|---|---|
| `scan_modules` | inventaire des modules qui répondent, par bus | ~100-200 tokens |
| `read_all_dtc` | DTC de tous les modules, avec description quand connue | 100-500 tokens |
| `read_freeze_frame` | instantané Mode 02 — **le premier réflexe** | 60-120 tokens |
| `read_live` | grandeurs demandées, échantillonnées N secondes → résumé statistique | 100-300 tokens |
| `list_sessions` | inventaire daté | ~20 tokens/session |
| `summarize_session` | plages, points de fonctionnement couverts, événements, anomalies classées | 300-600 tokens |
| `compare_operating_point` | même signal, même casier, N sessions — **le cœur du longitudinal** | 100-300 tokens |
| `render_signals` | `resource_link` vers un PNG (`plotters`) | image, pas de texte |
| `search_service_docs` | extraits du corpus RAG, **citation obligatoire** | 200-800 tokens |
| `lookup_recall_tsb` | rappels et TSB pertinents | 100-400 tokens |
| `probe_dids` | balayage `$22` sur un module, avec avancement | résumé seul |

Toutes les sorties typées via `outputSchema` (`03` §6). Les gros volumes en `resource_link`, jamais incorporés. `ttlMs`/`cacheScope` sur les données stables (profil véhicule, définitions de DID).

`notifications/progress` (`03` §5) est le bon mécanisme pour `probe_dids`, qui dure des dizaines de minutes.

**Ressources :** profil du véhicule, ligne de base d'entretien, historique des DTC, état de la base de DID.
**Prompts :** « analyse cette session », « qu'est-ce qui a changé depuis la dernière fois », « prépare l'intervention pour X ».

**Règle non négociable** (`04` §6.2) : une réponse sans citation à une question de spécification est traitée comme hallucinée.

---

## 9. Feuille de route

| Étape | Contenu | Prérequis |
|---|---|---|
| 0 | Les 4 vérifications gratuites de `07` niveau 0 — **dont quel moteur** | rien |
| 1 | Pilote série Rust + init STN + Mode 01/03/09. Premier contact. | matériel déjà en main |
| 2 | Serveur MCP stdio, 3 outils : `read_all_dtc`, `read_freeze_frame`, `read_live` | étape 1 |
| 3 | Commutation MS-CAN + `scan_modules` + `$19` sur tous les modules → **palier 2 atteint** | étape 2 |
| 4 | SQLite, sessions, historique de DTC | étape 3 |
| **4bis** | **Rôle enregistreur autonome, porté sur Pi Zero 2 W embarqué** (§11) | étape 4 |
| 5 | **Recherche ODX européenne** et recensement des PID communautaires | indépendante — à faire tôt |
| 6 | `probe_dids` + méthodologie d'interprétation → palier 3, incrémental | étapes 3-5 |
| 7 | Casiers de point de fonctionnement, CUSUM, `render_signals` | quelques sessions |
| 8 | Corpus RAG depuis l'abonnement documentaire (~30-115 $) | — |
| 9 | `compare_operating_point` — le longitudinal | **plusieurs mois de sessions**, et l'étape 4bis |

**Les étapes 1 à 3 sont le vrai jalon** : elles donnent un lecteur de DTC multi-modules multiplateforme, ce qui est déjà la fonction que tu utilises le plus dans FORScan, et ça te sort de la VM. L'étape 5 est indépendante et peut changer radicalement le coût de l'étape 6 — à ne pas repousser.

L'étape 9 porte la valeur et **aucun raccourci ne l'accélère** : elle demande du temps calendaire. **Et elle dépend de l'étape 4bis** — c'est le raisonnement de §11.1.

---

## 10. Effet sur le reste du dossier

| Document | Statut |
|---|---|
| `00-synthese.md` | Le « mur de parité FORScan » **existe toujours mais n'est plus binaire** : les paliers 1-2 le contournent, le palier 3 le grignote. Note de rescope à corriger. |
| `01` | **Valide et central.** §2 (UDS $27) explique la frontière du palier 4. §4 (coûts) et §5 (CASIS) inchangés. |
| `02` | ⚠️ **Révisé en profondeur le 2026-08-24, puis reclassé le 2026-08-25.** Moteur confirmé (3,2 L diesel) **et véhicule *deleted***. Devient le **profil de référence n°1** — couche 3 Ford V363 + couche 4 de cet exemplaire (`11` §4). Entièrement valide ; **ce n'est plus la colonne vertébrale du dossier.** |
| `03` | **Valide et directement applicable.** §5 (progression), §6 (`resource_link`, `outputSchema`, annotations non fiables), §8 (stdio), §9 (cache). |
| `04` | **Valide, le plus utile du dossier.** §2, §3.1 (encodage visuel), §5 (freeze frame, Mode 06), §6 (RAG) tous applicables. |
| `05` | **Revalidé, avec une correction importante.** L'avertissement « partiellement caduc » est retiré. Le pessimisme macOS de §2 ne concerne **que les interfaces CAN brutes** : un OBDLink EX est un périphérique USB-série, portable sans effort. §3.2 (Rust) est à nuancer — voir §3 ci-dessus. |
| `06` | Valide. |
| `07` | **Étape 1 (achat matériel) déjà accomplie.** L'item 4bis change de nature — voir la révision. Les items 13-14 redeviennent pertinents. |
| `08` | **Supersédé.** Conservé avec avertissement en tête, pour la trace. |
| `10` | **Valide.** §6 recasté le 2026-08-25 : les précautions propres au véhicule deviennent des mécanismes pilotés par le profil. Le reste — cycle de vie, concurrence, taxonomie d'erreurs, contrats — était déjà indépendant du véhicule. |
| **`11`** | **Nouveau, 2026-08-25. Le document qui rend l'application multi-marques :** les cinq couches de connaissance véhicule, l'art antérieur vérifié (ODX, OpenDBC, CaringCaribou, AutoAuth), le contenu d'un profil, la correction provenance/validation, la surface de sûreté ajoutée par le multi-marques, et le traitement du risque d'abstraction prématurée. |
| **`12`** | **Nouveau, 2026-08-26. Les standards de développement empruntés à punt-kit** — et ceux qui sont refusés. Confirme indépendamment `09` §2 et `10` §1, ajoute un CLI complet au périmètre, fournit le mécanisme applicable de la règle de couche 4, et pose l'avertissement sur la relecture. **Deux décisions y sont consignées : propriétaire unique du port série et emplacement des données.** |

---

## 11. Topologie de déploiement — du poste de travail au Pi embarqué

Cible annoncée : d'abord le serveur MCP sur le poste qui fait tourner l'agent, puis **le même logiciel sur un Raspberry Pi Zero embarqué dans le véhicule**. Ce deuxième temps n'est pas un simple portage : il change le transport MCP, il ajoute une surface d'authentification, et il introduit une contrainte électrique qui n'existait pas.

Il valide aussi, rétrospectivement, le choix de Rust — voir §11.5.

### 11.1 Ce que le Pi apporte que le poste de travail ne peut pas apporter

C'est le point le plus important, et il faut le voir avant de parler de matériel.

**L'analyse qui justifie le projet — le suivi de dérive longitudinal de §7A — a besoin de données de roulage.** Elle exige de comparer un même signal à l'intérieur d'un même casier de point de fonctionnement (régime × charge × température) entre sessions espacées de semaines. Or un poste de travail branché dans l'entrée ne produit qu'un seul casier : ralenti, à chaud, sans charge. **On ne peut pas construire une enveloppe de fonctionnement à partir de sessions au stationnement.**

Un appareil qui reste dans le véhicule et enregistre pendant que tu roules produit exactement ce qui manque : de la couverture de casiers, sur des mois, sans effort humain. Il capture aussi ce qui est **volatile** — un DTC en attente (Mode 07) et le freeze frame associé existent au moment de l'anomalie, et peuvent avoir disparu quand tu rentres.

**Conséquence de conception :** le premier rôle du Pi n'est pas « serveur MCP », c'est **enregistreur autonome**. C'est aussi le rôle le plus simple à sécuriser, parce qu'il n'expose rien.

### 11.2 Deux rôles, un seul cœur

Le cœur sans E/S de §2 se prête directement à un second axe de variation. Le premier axe était le **transport** (série natif / Web Serial). Le second est le **rôle** :

```text
                    CŒUR — Rust pur, aucune E/S
                             │
        ┌────────────────────┼────────────────────┐
        │                    │                    │
   RÔLE SERVEUR         RÔLE ENREGISTREUR    RÔLE UI (plus tard)
   MCP + agent          autonome              WASM/navigateur
        │                    │
   poste de travail      Pi embarqué
   stdio                 écrit dans SQLite
```

En pratique : **un seul binaire, deux sous-commandes** (`mecabot serve`, `mecabot log`). Le partage de code est quasi total — même pilote série, même encodeur UDS, même base de DID, même schéma SQLite. C'est bon marché précisément parce que le cœur ne fait pas d'E/S.

#### ⚠️ Décision du 2026-08-26 — un seul propriétaire du port série

Deux rôles issus du même binaire posent un problème que le dossier n'avait pas traité : **deux processus ne peuvent pas partager un port série.** L'adaptateur est une ressource exclusive du système d'exploitation. `mecabot serve` et `mecabot log` lancés sur la même machine se disputeraient `/dev/tty.usbserial-*`, et le mode de défaillance serait laid — soit un refus d'ouverture, soit, si le système le permet, des octets entrelacés donc du décodage faux.

`10` §2 avait déjà résolu ce problème **à l'intérieur d'un processus** par un acteur unique propriétaire du port. La décision l'étend d'un cran :

> **Règle : le port série a exactement un processus propriétaire par machine. Tout autre rôle qui a besoin du véhicule est un client de ce processus, jamais un second détenteur du port.**

Le canal existe déjà et n'est pas à inventer : c'est le drapeau `--remote <url>` et le transport Streamable HTTP de §11.3. Ce qui donne trois configurations, et aucune ne demande de travail supplémentaire maintenant :

| Configuration | Propriétaire du port | Le reste |
|---|---|---|
| **Poste de travail seul** — le cas courant | `mecabot serve` | Si tu veux enregistrer aussi depuis le poste : `mecabot log --remote` contre le serveur local |
| **Pi embarqué, premier temps** | `mecabot log`, seul sur la machine | Aucun `serve` sur le Pi. Inchangé par rapport à la recommandation de §11.3 — c'est la configuration la plus simple et elle reste la bonne pour commencer. |
| **Pi embarqué, mode MCP en direct** | `mecabot serve` lié à **la boucle locale seulement** | `mecabot log --remote http://127.0.0.1:…`. La boucle locale n'est pas un réseau : **la propriété « rien à authentifier » de §11.3 est préservée.** |

**Ce que cette décision unifie.** Elle fait converger trois choses que je traitais comme indépendantes : l'axe des rôles de cette section, la question de transport de §11.3, et le drapeau `--remote` exigé par le standard CLI (`12` §3). Ce n'est pas trois décisions mais une seule, et le patron vient du même endroit que la règle de journalisation multi-écrivains — *un propriétaire, les autres expédient par un transport existant*.

**Ce qu'elle ne change pas :** rien dans la feuille de route. La première configuration est le seul cas jusqu'à l'étape 4bis, et elle a un seul processus.

### 11.3 Le transport MCP change, et ce n'est pas gratuit

**stdio suppose la colocalisation.** Le client MCP lance le serveur comme sous-processus et lui parle par ses tubes standards. Il n'y a rien à authentifier parce qu'il n'y a pas de réseau. Un serveur dans le véhicule ne peut pas être un sous-processus de Claude sur ton Mac.

Si tu veux interroger le Pi en direct depuis l'agent, il faut passer au transport **Streamable HTTP** (`03` §8), et trois problèmes apparaissent d'un coup :

| Problème | Nature |
|---|---|
| **Authentification** | La spec MCP prévoit OAuth 2.1 pour les transports HTTP. Un point d'accès MCP sur un réseau est une surface exposée — la spec avertit aussi explicitement sur la validation de l'en-tête `Origin` et le DNS rebinding. [NON VÉRIFIÉ] sur ce que `rmcp` implémente réellement côté serveur HTTP. |
| **Joignabilité** | Le véhicule n'est pas toujours sur ton réseau. Trois options : le Pi en point d'accès Wi-Fi que le Mac rejoint ; un réseau overlay (Tailscale/WireGuard) ; ou pas de joignabilité du tout. |
| **Confiance réseau** | Un service qui parle à des calculateurs, sur un réseau Wi-Fi, même en lecture seule. L'énumération fermée de services de §6 devient la mesure de sûreté qui compte vraiment. |

**Recommandation :** ne pas faire du Pi un serveur MCP au premier tour. Le laisser en **enregistreur muet** qui écrit dans SQLite, et garder le serveur MCP sur le poste de travail en stdio. La synchronisation se fait par le fichier de base : le Pi rejoint le Wi-Fi de la maison quand le véhicule est stationné, et le poste récupère le fichier. Pas d'authentification à concevoir, pas de port ouvert, et l'agent voit les données de roulage.

> **Précision du 2026-08-26.** La règle du propriétaire unique (§11.2) n'invalide pas cette recommandation, elle la conforte : dans cette configuration, `mecabot log` est **seul sur le Pi** et détient donc légitimement le port. Rien à changer.
>
> Et le jour où le mode MCP embarqué devient un besoin, la même règle donne la forme la plus sûre : `serve` lié à `127.0.0.1` avec `log` en client local. **Les trois problèmes du tableau ci-dessus disparaissent tous les trois** — pas d'authentification puisqu'il n'y a pas de réseau, pas de joignabilité à résoudre puisque les deux processus sont sur la même machine, pas de confiance réseau à accorder puisqu'aucun port n'est exposé. Le transport HTTP est alors requis plus tôt que cette section ne le supposait, **mais dans sa forme la moins risquée.**

Le mode « MCP en direct dans le véhicule » reste une extension naturelle — utile le jour où tu diagnostiques quelque chose depuis le siège conducteur avec un téléphone ou un portable. Ce n'est pas le premier besoin.

### 11.4 Le Pi Zero 2 W, pas le Pi Zero W

La différence est structurante et souvent ignorée :

| | Pi Zero W | **Pi Zero 2 W** |
|---|---|---|
| SoC | BCM2835, ARM1176JZF-S mono-cœur | RP3A0 (BCM2710A1), **quatre Cortex-A53** |
| Architecture | **ARMv6** | ARMv8 — OS 64 bits disponible |
| RAM | 512 Mo | 512 Mo |
| Cible Rust | `arm-unknown-linux-gnueabihf` — **Tier 2** | `aarch64-unknown-linux-gnu` — **Tier 1** |

L'ARMv6 est le mauvais cheval : cible Rust de second rang, et surtout un écosystème binaire où beaucoup de choses n'existent qu'en `armv7`/`aarch64`. Le Zero 2 W coûte à peine plus cher et met l'architecture en Tier 1. **Prendre le 2 W.**

Sur les deux, l'OBDLink EX se branche via un adaptateur USB OTG et apparaît en `/dev/ttyACM0` grâce au pilote `cdc_acm` du noyau — aucun pilote à installer. Le port de données unique du Zero n'est pas un obstacle puisque le Wi-Fi est intégré.

**Compilation :** croiser depuis le Mac (via `cross`, qui s'appuie sur Docker) et copier le binaire. Ne pas compiler sur l'appareil — 512 Mo de RAM et rustc font mauvais ménage.

### 11.5 Pourquoi ceci tranche définitivement le débat Rust vs Python

`05` §3.2 recommandait Python, et `01`/`09` avaient déjà nuancé ce verdict. **La cible Pi Zero l'inverse franchement :**

- Un binaire Rust croisé fait quelques mégaoctets et démarre instantanément. Le déploiement est un `scp` et un service systemd.
- Python sur 512 Mo demande un interpréteur, un environnement virtuel et la résolution de roues natives (`pyserial`, `python-can`, `can-isotp`, `udsoncan`) — faisable, mais chaque mise à jour est une opération sur l'appareil, et sur ARMv6 la disponibilité des roues est franchement inégale.
- Un processus qui doit tourner en permanence dans un véhicule bénéficie directement d'une empreinte mémoire basse et prévisible.

Ce n'est plus « Rust est défendable ici », c'est **Rust est le bon outil pour cette cible**.

### 11.6 La contrainte que tout le monde découvre trop tard : l'alimentation

**La broche 16 du connecteur J1962 est le positif de batterie non commuté.** Un appareil branché sur l'OBD est alimenté en permanence, contact coupé, véhicule stationné.

Ordres de grandeur, à mesurer et non à croire [NON VÉRIFIÉ] : un Pi Zero 2 W sans tête consomme grossièrement 0,5 à 1 W, et un convertisseur 12 V → 5 V ajoute son rendement et son courant de repos. Avec ~1,5 W au total, on est autour de **0,12 A prélevé sur le 12 V, soit environ 20 Ah par semaine de stationnement**. Sur une batterie de démarrage, c'est un quart de la capacité en une semaine — et une panne de démarrage en deux ou trois, davantage en hiver québécois où la capacité chute.

Ce n'est pas un détail à régler à la fin. Trois approches, par ordre de robustesse :

1. **Détection de contact et extinction propre.** Le montage surveille la tension de bus ou l'activité CAN ; à l'arrêt du moteur, il termine la session, valide la base et s'éteint. Il faut alors un vrai coupe-circuit en aval, sinon on ne coupe que la charge logique, pas la consommation.
2. **Coupure matérielle par relais temporisé** sur détection d'absence d'activité CAN. Plus fiable, plus de câblage.
3. **Débrancher à la main entre les trajets.** Sans élégance, mais sans risque, et acceptable pour valider le concept avant d'investir dans le montage.

⚠️ **Réserve de sûreté :** alimenter un appareil depuis la broche 16 sans fusible dédié ajoute un chemin non protégé sur le circuit de la batterie. Le montage doit être fusionné au plus près de la source. C'est du câblage de véhicule, à faire soigneusement ou à faire faire.

### 11.7 Ce que le rôle enregistreur a le droit de faire en roulant

`§6` pose que le balayage de DID ne se fait **jamais** en roulant. Cette règle reste entière, mais il faut la préciser maintenant qu'un appareil interroge des calculateurs pendant que le véhicule circule.

La distinction n'est pas « lire ou pas lire en roulant » — n'importe quelle application de tableau de bord lit des PID en roulant. Elle est **entre lecture et découverte** :

| En roulant | Statut |
|---|---|
| Modes `$01`, `$02`, `$03`, `$07` (OBD-II normalisé) | ✅ autorisé — usage prévu du protocole |
| `$22` sur des **DID déjà connus et validés** | ✅ autorisé |
| `$19` lecture de DTC | ✅ autorisé |
| **`probe_dids` — balayage de DID inconnus** | ⛔ **interdit en mouvement.** Un DID inconnu peut se révéler autre chose qu'une lecture inoffensive, et un refus mal interprété par un module en pleine gestion moteur n'est pas un risque à prendre en circulation. |
| Balayage d'identifiants de **service** | ⛔ interdit, en toutes circonstances (`§6`) |

**Règle d'architecture :** le rôle enregistreur ne compile littéralement pas `probe_dids`. La découverte reste une opération du rôle serveur, véhicule stationné, mainteneur de batterie branché.

### 11.8 Usure de la carte SD

Une base SQLite écrite en continu sur microSD s'use par le **nombre de validations**, pas par le volume de données — une session de 30 min à 15 signaux/1 Hz ne fait que ~27 000 lignes, ce qui est négligeable. Ce qui tue une carte, c'est un `commit` par échantillon.

Mesures suffisantes : mode **WAL**, `synchronous=NORMAL`, **validations par lots** (quelques secondes de données par transaction), et une carte à haute endurance. [NON VÉRIFIÉ] sur la durée de vie réelle dans cet usage — à surveiller plutôt qu'à supposer réglé.

### 11.9 Ce que ça ajoute à vérifier

À reporter dans `07` :

- La consommation réelle du montage, au multimètre, avant de le laisser branché une semaine. **Le seul chiffre de cette section qui compte vraiment.**
- Que le noyau du Pi présente bien l'OBDLink EX en `/dev/ttyACM0` via OTG, et que le débit tient — recoupe l'item 4bis, qui reste à faire d'abord au terminal série sur le Mac.
- Ce que `rmcp` couvre réellement du transport Streamable HTTP et de son volet authentification, **le jour seulement** où le mode MCP embarqué devient un besoin.
- Le comportement du timeout `ATST` et la nécessité de `$3E TesterPresent` sur une session longue — beaucoup plus critique pour un enregistreur qui tourne des heures que pour une session interactive de dix minutes.
