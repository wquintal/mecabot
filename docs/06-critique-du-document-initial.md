# 06 — Critique du document initial

Verdict point par point sur le document Gemini « Conception et Architecture Logicielle d'un Middleware MCP pour le Diagnostic Automobile OEM ».

Précision de cadrage : le document est **bien construit et globalement bien informé**. Ses intuitions structurantes sont justes. Ses erreurs sont concentrées dans deux zones précises — les mécanismes MCP qu'il décrit, et les capacités qu'il suppose accessibles — et elles ont pour cause commune de ne s'être appuyé sur **aucune vérification de source primaire**. C'est un excellent document de cadrage et un mauvais document de spécification.

---

## 1. Ce qui était juste, et qui tient après vérification

| Affirmation | Verdict | Confirmé par |
|---|---|---|
| Ne jamais injecter de trames CAN brutes dans le contexte du LLM | ✅ **Juste, et pour une raison plus forte qu'il ne le dit** | `04` §1, §3 |
| Exposer des outils de haut niveau (`get_dtc_summary`) plutôt qu'un accès UDS brut | ✅ Juste, et conforme à l'intention de la spec MCP | `03` §10 |
| Machine à états finis pour séquencer les sessions diagnostiques | ✅ Juste — UDS est intrinsèquement à états (session, TesterPresent $3E, timers) | ISO 14229 |
| Humain dans la boucle avant toute action sur le véhicule | ✅ **Juste, et il existe un mécanisme formel pour ça** (elicitation) | `03` §3 |
| Découpage Tools / Resources / Prompts (actions / télémétrie / profil véhicule) | ✅ Juste, correspond bien aux rôles définis par la spec | `03` §2 |
| Objectif d'observabilité par traces OpenTelemetry | ✅ Objectif valide — mécanisme faux (voir §3) | `03` §7 |
| Compression par delta / bande morte de la télémétrie | ✅ Juste, et c'est effectivement le meilleur rapport valeur/effort | `04` §2 |
| BYOK et souveraineté des données du client | ✅ Juste, et sans objet ici : mono-véhicule, mono-utilisateur, serveur local | — |
| Cadre Right to Repair comme fondement de légitimité | ✅ Juste sur le principe. Le cadre canadien applicable est le **CASIS**, un accord volontaire, pas une loi | `01` §5 |
| PWA + Web Serial / Web Bluetooth pour le déploiement | ✅ Techniquement correct, mais hors périmètre — un serveur MCP local sur stdio suffit | `03` §8 |

Le document a par ailleurs eu raison sur un point que je juge important : il a compris que **le problème central est la consommation de contexte, pas la capacité de raisonnement**. C'est le bon diagnostic. Il se trompe sur l'ampleur, pas sur la nature.

---

## 2. Ce qui était faux

### 2.1 La détection de ratés d'allumage par LSTM embarqué

**Faux comme formulé.** L'affirmation : un modèle LSTM à la périphérie détecte « la signature caractéristique d'un raté » depuis le signal de vitesse du vilebrequin.

L'arithmétique la contredit. L'ECU surveille le capteur CKP à 360-720 événements par tour, soit ~36 000 événements/s à 3 000 tr/min. Le port OBD n'expose le régime qu'à 1-10 Hz, et seulement le **verdict pré-calculé** de l'ECU. La fluctuation causée par un raté à 2 000 tr/min dure ~33 ms : à 2 Hz d'interrogation, tu vois au mieux un échantillon, sans résolution temporelle.

Les travaux qui revendiquent ce résultat font en réalité l'une de trois choses : ils lisent le compteur de ratés de l'ECU (lecture de registre, pas détection), ils exploitent des symptômes secondaires peu spécifiques, ou ils travaillent sur du **trafic CAN propriétaire brut** hors du port de diagnostic.

Détail à noter : la revue MDPI citée en appui dans le document initial **n'a pas pu être consultée** (HTTP 403, plusieurs tentatives) [INTROUVABLE]. Le verdict repose sur l'arithmétique d'échantillonnage et sur la littérature de détection sur CAN, elle consultée. Voir `04` §4.1.

**Ce qui reste vrai :** de la statistique simple sur des semaines détecte très bien la dérive lente (corrections d'injecteur, position EGR, charge de suie). C'est utile, ce n'est pas du LSTM temps réel.

### 2.2 Les API de données de réparation

**Faux.** La section 4 du document décrit une intégration RAG s'appuyant sur des « API ALLDATA / Mitchell / MOTOR ».

Aucun de ces fournisseurs n'expose d'API à un individu. ALLDATA DIY (19,99 $/mois) et Identifix DIY (29,99 $/mois) sont des interfaces web, sans API. MOTOR Information Systems est du **B2B strict** : pas de palier individuel, pas d'inscription en libre-service, pas de tarif public, contrat sur devis à plusieurs milliers de dollars par mois. Voir `01` §4.2.

Le besoin sous-jacent est légitime, et se règle autrement : abonnement DIY, téléchargement des sections concernant **un** véhicule, index RAG local de quelques centaines de pages.

### 2.3 Le réveil asynchrone de l'agent

**Faux au niveau du protocole.** Le document décrit des « moniteurs asynchrones » : l'interaction LLM est suspendue pour libérer le contexte, et le pont réveille l'agent quand une condition est satisfaite.

Depuis la révision 2026-07-28, **le serveur MCP n'initie plus jamais de requête JSON-RPC**, et il n'existe aucun mécanisme pour injecter un message dans la conversation d'un LLM depuis l'extérieur. La spec le dit explicitement dans sa vue d'ensemble des transports.

Ce qui est possible : si un flux `subscriptions/listen` est **déjà ouvert**, le serveur peut pousser `notifications/resources/updated` à tout moment. Ce qui se passe ensuite relève de l'application hôte, pas du protocole. Voir `03` §4.

**Conséquence de conception :** la surveillance longue durée se conçoit comme un service local qui journalise, dont l'agent lit le résumé à la session suivante. Pour du suivi de dérive sur des semaines — le cœur de la valeur de Mecabot — c'est de toute façon le bon modèle.

### 2.4 Les annotations d'outils comme garde-fou

**Faux comme garantie.** Le document traite `readOnlyHint` / `destructiveHint` comme un mécanisme de sécurité.

La spec est explicite sur deux points : les clients **doivent** considérer ces annotations comme non fiables sauf provenance de serveurs de confiance, et la spec **n'impose aucune action** au client sur leur base. Ce sont des indices d'affichage.

**Ce qu'il faut faire à la place :** la sécurité doit être **structurelle**. Un outil qui n'a aucun chemin de code vers l'écriture, plutôt qu'un outil déclaré non destructeur. Voir `03` §6.

### 2.5 Les capacités d'écriture UDS

**Faux pour ce projet.** Le document construit une machine à états séquençant `$27` SecurityAccess → `$2E` WriteDataByIdentifier / `$2F` InputOutputControl / `$31` RoutineControl → reset ECU.

Aucune de ces opérations ne sera accessible. L'algorithme seed/key Ford 2015-2019 n'est publiquement connu de personne : `jglim/UnlockECU`, la base ouverte la plus complète avec 3 038 entrées, contient **exactement 0 entrée Ford**. Ford a fait retirer des travaux antérieurs par DMCA. Voir `01` §2.

**Ce n'est pas une décision de prudence, c'est une impossibilité.** Le repositionnement : quand une écriture est nécessaire, Mecabot **prépare l'intervention** (identifie la procédure, cite la source, liste les préconditions) et l'humain l'exécute dans FORScan.

### 2.6 Rust ou C++ « pour éliminer la latence du garbage collector »

**Faux comme raisonnement.** La contrainte de débit est de 2-20 PID/s, imposée par le protocole requête/réponse OBD et la surcharge de traitement de l'adaptateur (20-120 ms par PID). Le langage hôte n'y change rien : l'ordre de grandeur est de trois à quatre décades au-dessus de ce qui pourrait rendre le GC pertinent.

Accessoirement, l'écosystème Rust automobile est inutilisable ici : le crate `socketcan` est **explicitement Linux-only**, `ecu-diagnostics` a un build docs.rs en échec depuis la 0.101.0, `OpenVehicleDiag` est abandonné depuis 2021. Voir `05` §3.2.

### 2.7 L'ampleur du problème de tokens

**Faux d'un facteur ~100, dans le sens rassurant.** Le document raisonne sur la charge d'un bus HS-CAN saturé : à 40 % d'occupation, journalisé naïvement en hexadécimal, on arrive effectivement à ~1,36 million de tokens/minute.

Mais tu n'auras jamais ce débit. Un adaptateur interrogé en requête/réponse plafonne à 2-6 PID/s en PID unique, 12-20 PID/s en lot de 6. Les volumes réels :

| Scénario | Tokens/min |
|---|---|
| 15 PID à 1 Hz | ~7 200 |
| 10 PID à 5 Hz | ~24 000 |

**L'ordre de grandeur réel est 10⁴ tokens/min, pas 10⁶.** Le problème reste réel — une heure de roulage à 15 PID/1 Hz fait ~432 000 tokens, soit 3,4 fenêtres de 128k — mais il ne justifie pas la pipeline d'ingénierie de données industrielle que le document propose. Bande morte + CUSUM + instantanés horodatés suffisent. Voir `04` §1.

### 2.8 Le FEPS comme préoccupation

**Sans objet.** Le document mentionne le Ford Enhanced Power Supply (~18 V sur la broche 13). C'était une exigence des PCM de l'ère **EEC-V** (1996 à ~2004) pour déverrouiller la NVM en écriture. Sur un Transit 2016, tout est CAN + UDS ; la broche 13 ne porte aucune signification. Aucun adaptateur orienté FORScan ne l'implémente, et aucun n'en a besoin. Voir `01` §1.3.

---

## 3. Ce qui était inventé

**`notifications/otel/trace`** — cette notification **n'existe dans aucune révision de la spécification MCP**, ni actuelle ni antérieure. Le document la présente comme un mécanisme du protocole.

Le mécanisme réel, documenté par **SEP-414** (statut Final, créé 2025-04-25) : la propagation du contexte de trace W3C se fait en plaçant `traceparent` et `tracestate` directement dans le `_meta` de n'importe quelle requête. C'est une **documentation d'une pratique existante** (déjà implémentée dans les SDK C# et Python, Logfire, Envoy AI Gateway, OpenInference, ToolHive), pas un nouveau type de message. C'est aussi une exception délibérée à la convention de préfixe DNS des clés `_meta`, précisément pour préserver l'interopérabilité W3C. Voir `03` §7.

L'objectif d'observabilité est bon. Le mécanisme était fabriqué de façon plausible — c'est exactement le mode de défaillance décrit en `04` §6.2 pour les LLM sur les spécifications techniques, et une raison de plus pour la règle « toute affirmation de spécification porte sa citation ».

Je n'ai pas trouvé d'autre invention franche dans le document. Les autres erreurs sont des extrapolations excessives, pas des fabrications.

---

## 4. Ce qu'il a omis

### 4.1 Le freeze frame (Mode 02) — l'omission la plus coûteuse

Le document ne le mentionne **pas une seule fois**, dans un texte dont le sujet principal est l'économie de tokens.

Quand un DTC se pose, l'ECU enregistre un instantané de tous les PID Mode 01 à cet instant précis : régime, charge calculée, température moteur, STFT/LTFT des deux bancs, MAP/MAF, papillon, tensions lambda, vitesse. Soit 12 à 20 PID pour **~60-120 tokens** en contexte structuré.

C'est l'artefact le plus dense en information diagnostique de tout l'OBD-II — l'enregistreur de vol au moment de la panne. Toute session sur un véhicule avec DTC stocké devrait commencer là. Voir `04` §5.

Le **Mode 06** (résultats des monitors embarqués, ~600 tokens pour un dump complet) est également absent, alors qu'il permet de voir un monitor dériver **avant** qu'il ne pose un DTC.

### 4.2 L'encodage visuel des séries temporelles

Contre-intuitif, et probablement le levier le plus intéressant de tout le dossier. `arXiv:2608.07427` compare l'encodage textuel de séries temporelles à leur rendu en graphique 2D soumis à un VLM :

- **3,6× à 10,4× moins de tokens d'entrée**
- 1,8× à 2,5× de réduction d'énergie d'inférence mesurée
- **+220,7 % de précision** (Llama-3.2-90B-Vision affiné vs équivalent texte seul)
- à 24 indicateurs, la représentation textuelle dépasse la fenêtre de 128k ; l'encodage visuel reste dedans

Rendre 60 secondes de tracés (régime, pression rampe, EGT, position EGR) en un PNG de 400×200 est à la fois **moins cher et plus juste** que de sérialiser les valeurs. Les VLM conservent les relations spatiales que la tokenisation BPE détruit. Voir `04` §3.1.

### 4.3 `resource_link` — la réponse que la spec MCP donne déjà au problème de volume

Le document déploie beaucoup d'ingéniosité sur la compression sans mentionner qu'il existe depuis 2025-06-18 un mécanisme officiel : un outil retourne un `{"type": "resource_link", "uri": ...}` et le contenu n'entre en contexte que si le client le lit explicitement.

C'est probablement le levier le plus important côté MCP, et il est gratuit. Idem pour `outputSchema`/`structuredContent` (sortie typée validée, le modèle ne parse pas de texte) et `ttlMs`/`cacheScope` (SEP-2549, pour ne pas refetcher le stable). Voir `03` §6 et §9.

### 4.4 Le mur de parité FORScan

L'omission conceptuelle la plus importante. Le document ne nomme jamais son point de comparaison implicite.

FORScan, sur un Transit 2016, apporte deux choses non réplicables : la **base de PID/DID étendus Ford** (reverse-engineerée, protégée par des règles de forum qui interdisent explicitement d'en discuter ; l'équivalent licencié coûte ~50 000 $US/an chez l'ETI) et les **clés SecurityAccess**.

Un document de conception qui ne dit pas quelle part de son ambition est déjà couverte par un outil à 15-20 $/an, et quelle part ne le sera jamais, laisse le lecteur croire à une viabilité qu'il n'a pas démontrée. Voir `00-synthese.md`.

### 4.5 Le rôle réel du LLM, tel que la littérature l'établit

Le document positionne le LLM comme raisonneur diagnostique. La recherche récente dit autre chose, de façon assez unanime :

- **RAG guidé par graphe de connaissances** pour la validation HiL automobile (arXiv:2608.11277, ICSRS 2026) : 90 % de précision top-1 sur moteur essence, 94 % sur système EV. Constat explicite : l'ancrage par graphe empêche le LLM de produire des localisations de défaut plausibles mais fausses.
- **CAREP** (arXiv:2602.01155) : 29 100 DTC uniques, 474 motifs d'erreur ; la découverte causale surpasse le LLM seul.
- **« Are LLMs Useful for Time Series Forecasting? »** (arXiv:2406.16964, Spotlight NeurIPS 2024) : retirer le composant LLM de trois méthodes populaires **ne dégrade pas** la performance, et souvent l'améliore.

Le motif transversal : **le LLM n'est pas le détecteur, il est l'explicateur ancré.** Le calcul et la détection se font en code déterministe et en structures explicites ; le LLM fournit l'interprétabilité et la mise en contexte documentaire. Voir `04` §3 et §6.1.

### 4.6 Les contraintes de plate-forme

Le document ne mentionne à aucun moment le système d'exploitation. Or c'est structurant : **SocketCAN est un sous-système du noyau Linux**, `can-utils` n'existe pas sur macOS, **J2534 est une spécification de DLL Windows 32 bits**, et FORScan est une application Windows. Sur macOS, un développeur aura besoin d'une VM Windows de toute façon — ce n'est pas un détail d'implémentation, c'est une contrainte de flux de travail. Voir `05` §2.

### 4.7 La commutation MS-CAN

Le document parle de J2534 et d'ELM327 sans mentionner que **Ford place son bus carrosserie sur les broches 3/11 du connecteur OBD**, une affectation propriétaire. Un adaptateur qui ne sait pas y basculer ne voit **aucun** module de carrosserie, d'instrumentation ou d'infodivertissement — pas des données dégradées, zéro réponse. C'est le critère de sélection matérielle numéro un sur cette marque, et aucune bibliothèque ne le gère aujourd'hui. Voir `05` §1.1 et §3.1.

### 4.8 Le véhicule

Le document est écrit comme un cahier des charges de produit multi-marques « niveau OEM ». La cible réelle est **un véhicule**, dont le propriétaire a besoin qu'il roule.

Cette différence n'est pas une réduction d'ambition, c'est ce qui rend le projet faisable : un corpus RAG de quelques centaines de pages au lieu d'un corpus industriel, un jeu de PID à découvrir au lieu d'une base universelle, ~80-150 $/an de données au lieu d'une licence ETI. Et un critère de succès vérifiable : est-ce que ça a aidé à réparer le Transit.

---

## 5. Ce qu'il faut retenir du document initial

**À garder :** l'architecture générale (outils de haut niveau, FSM, humain dans la boucle), le diagnostic que le contexte est la ressource rare, la compression par delta, l'objectif d'observabilité, le cadrage Right to Repair.

**À jeter :** la section 4 sur les API de données de réparation (fiction), les capacités d'écriture UDS (impossibles), le réveil asynchrone de l'agent (non supporté), le LSTM de détection de ratés (arithmétiquement infondé), `notifications/otel/trace` (inventé), le choix Rust/C++ pour la latence (non-problème).

**À ajouter :** freeze frame Mode 02 en premier réflexe, `resource_link` et `outputSchema`, l'encodage visuel pour les formes temporelles, la commutation MS-CAN comme critère matériel, le LLM comme explicateur ancré plutôt que détecteur, et la reconnaissance explicite du mur de parité FORScan.

**Le glissement qui explique presque toutes les erreurs :** le document décrit ce qui *devrait* exister — et il le décrit bien, avec cohérence interne et un vocabulaire juste. Il ne vérifie pas ce qui existe. Un mécanisme MCP plausible, une API de données plausible, un modèle ML plausible : chacun est raisonnable en isolation, et chacun est faux. C'est le mode de défaillance documenté en `04` §6.2, appliqué à un document de conception au lieu d'une réponse de diagnostic. La contre-mesure est la même : **toute affirmation portant sur une capacité ou une spécification doit porter sa citation, sinon elle est traitée comme hallucinée.**
