# 10 — Conception du serveur MCP

**Date :** 2026-08-24
**Objet :** la conception détaillée de la surface MCP, dont `09` §8 ne donnait qu'un tableau d'intentions.
**Préalables :** `03` (état réel de la spec MCP), `09` (architecture retenue), `04` (budget de tokens).

Ce document traite ce qui décide si le serveur est utilisable ou pénible : le cycle de vie de la connexion, la concurrence, la taxonomie d'erreurs, et le contrat de chaque outil.

> **Révisé le 2026-08-25 pour le rescope multi-marques** (`11`). L'essentiel de ce document était **déjà indépendant du véhicule** — §1 (état), §2 (concurrence), §3 (classes d'outils), §8 (ce qu'il ne faut pas construire) et §9 (test) n'ont pas bougé. Ce qui change : **§6 est recasté** en mécanismes pilotés par le profil, **§4 gagne deux conditions d'erreur**, **§5 scinde un champ qui en cachait deux**, et §7 précise deux contrats.

---

## 1. La difficulté centrale : MCP est sans état, l'OBD est avec état

C'est le problème de conception le plus important du projet côté logiciel, et il ne se voit pas dans un tableau d'outils.

Un appel d'outil MCP est une transaction isolée. Une session OBD, elle, porte de l'état à **quatre niveaux empilés** :

| Niveau | État porté |
|---|---|
| Port série | ouvert / fermé, et **exclusif** — un seul détenteur, **y compris entre processus** (`09` §11.2) |
| Adaptateur | initialisé, protocole choisi, en-têtes posés (`ATSH`), **et surtout : quel bus** (HS-CAN ou MS-CAN) |
| Session UDS, par module | défaut ou étendue, avec un **chronomètre de `$3E TesterPresent`** qui court |
| Véhicule | contact mis ou coupé, moteur tournant ou arrêté, immobile ou en mouvement |

Entre deux appels d'outil, il s'écoule le temps de réflexion du modèle : quelques secondes à quelques minutes, parfois davantage si l'utilisateur part manger. **Tout état de protocole que l'agent devrait entretenir se désynchronisera.**

### Deux options, et pourquoi je tranche

**Option A — outils `connect` / `disconnect` explicites.** L'agent ouvre, travaille, ferme. C'est ce que font `shoka-jp/obd2-mcp` et `serial-mcp-server` (`05` §4).

Le problème n'est pas théorique : l'agent oubliera de fermer. Il n'y a aucune garantie qu'un agent conversationnel exécute une étape de nettoyage — il peut être interrompu, changer de sujet, ou décider que la tâche est finie. Le résultat est **un port série retenu et une session morte**, et l'échec est silencieux. En plus, ça charge le contexte de l'agent de comptabilité de protocole qui ne lui apporte rien.

**Option B — connexion implicite, gérée par le serveur.** Aucun outil de connexion. Le premier outil qui a besoin du véhicule ouvre le port, initialise l'adaptateur, et met l'état en cache. Les appels suivants réutilisent. Un chien de garde ferme après un délai d'inactivité. Le choix du bus se déduit du module adressé.

**→ Option B.** L'agent ne doit jamais voir l'adaptateur. Le critère de décision est simple : *tout état que l'agent doit maintenir entre deux tours de conversation finira désynchronisé*, et le mode de défaillance de A est silencieux alors que celui de B est un simple délai de réouverture.

### Le coût de l'option B, à nommer

**La commutation de bus est chère.** Passer de HS-CAN à MS-CAN demande une commande STN, un délai, et probablement une réinitialisation de session. Si l'ordre des appels de l'agent pilote la commutation, on va faire du va-et-vient coûteux.

**Règle de conception :** un outil qui touche plusieurs modules **ordonne son travail par bus** — tout le HS-CAN, puis une commutation, puis tout le MS-CAN. `scan_modules` et `read_all_dtc` fonctionnent ainsi. L'ordre d'appel de l'agent ne doit jamais déterminer l'ordre de commutation.

⚠️ [NON VÉRIFIÉ] Le délai réel de commutation et la nécessité de réinitialiser sont l'item 4bis de `07`. Cette section est un plan qui suppose des chiffres non mesurés.

> ### ✅ Partiellement levé le 2026-08-28 — la **nécessité** de réinitialiser n'est plus une hypothèse
>
> `[VÉRIFIÉ]` sur le manuel du fabricant (*OBDLink Family Reference and Programming Manual*, rév. F, 2025-08-29, §8.6) : l'adaptateur n'a **qu'un seul périphérique CAN**, mappé sur différentes broches **par logiciel**, donc **un seul bus actif à la fois**. La commutation est un changement de préréglage — `STP 33` pour HS-CAN, **`STP 53` pour le MS-CAN Ford**. Le périphérique étant **remappé**, une session UDS ouverte sur l'autre bus est perdue **par construction** : le *« probablement »* ci-dessus tombe.
>
> **Ce qui reste `[NON VÉRIFIÉ]`, et c'est le chiffre qui décide de tout :** le **délai**. Aucune documentation ne le publie. S'il est de 50 ms, la règle d'ordonnancement par bus ci-dessus est un confort ; s'il est de 2 s, elle est structurante. Mesuré au bloc D de `13`, trois fois dans chaque sens.
>
> ⚠️ **Et il y a un second chien de garde, indépendant du chronomètre `$3E`.** L'adaptateur a sa propre fonction de basse consommation, qui s'annonce par `ACT ALERT` (une minute) puis `LP ALERT` (deux secondes), avec un délai d'inactivité UART par défaut de 1200 s. `STSLCS` en imprime toute la configuration en une commande. Deux minuteurs à des échelles différentes, à ne pas confondre dans la conception — surtout pour le rôle enregistreur de `09` §11, qui tourne des heures.

---

## 2. Concurrence : le port série est une ressource exclusive

Un agent peut émettre **plusieurs appels d'outils en parallèle** dans un même tour. C'est un comportement normal et souhaitable — sauf que deux lectures simultanées sur un port série unique produisent des octets entrelacés, donc du décodage faux, pas une erreur propre.

**Conception :** un **acteur unique propriétaire du port**, avec une file de requêtes. Les gestionnaires d'outils soumettent et attendent. Ce n'est pas une optimisation, c'est la différence entre un serveur qui marche et un serveur qui corrompt silencieusement ses données.

Conséquences à assumer :

- **Un seul outil touchant le véhicule s'exécute à la fois.** Les outils hors-ligne (§3) ne passent pas par la file et restent parallélisables.
- **Un outil long bloque la file.** `read_live` sur 60 s, `probe_dids` sur 40 minutes. Il faut donc que les outils longs soient **annulables** et rapportent leur avancement (`03` §5, `notifications/progress`).
- **Un appel qui attend derrière un balayage de DID doit le savoir.** Renvoyer « en attente derrière une opération de N minutes, annulable » plutôt que de laisser expirer un délai d'attente sans explication.

### ⚠️ L'exclusivité vaut aussi entre processus — décidé le 2026-08-26

Cette section réglait la concurrence **à l'intérieur** du serveur. Elle laissait passer un cas que `09` §11.2 traite maintenant : **`mecabot serve` et `mecabot log` sont deux processus, et l'acteur unique de ce paragraphe n'en protège qu'un.** Deux processus qui ouvrent le même port produisent exactement la corruption silencieuse que cette section existe pour empêcher, un cran plus bas.

**Règle retenue :** un seul processus détient le port par machine ; l'autre rôle devient son client par `--remote`. Côté conception du serveur, la conséquence est mince mais réelle :

- L'acteur propriétaire du port reste **le seul** composant qui ouvre le périphérique, et il le fait avec un verrou explicite plutôt qu'en supposant l'exclusivité.
- L'échec d'ouverture parce qu'**un autre processus Mecabot détient déjà le port** est une condition d'erreur nommée, distincte de « aucun adaptateur sur aucun port série » (§4). L'action à conseiller n'est pas la même : brancher l'adaptateur, ou arrêter l'autre processus.
- La file d'attente de §2 devient le point de service de **deux** sources de requêtes — les outils MCP et le rôle enregistreur distant. Rien à changer dans sa conception : une file avec un propriétaire unique ne se soucie pas de qui soumet.

---

## 3. Deux classes d'outils, et l'agent doit pouvoir les distinguer

Environ la moitié de la surface ne demande aucun véhicule. C'est une propriété précieuse : le camion n'est pas toujours dans l'entrée.

| Hors-ligne — base de données et corpus | En ligne — exige adaptateur + véhicule |
|---|---|
| `list_sessions` | `read_all_dtc` |
| `summarize_session` | `read_freeze_frame` |
| `compare_operating_point` | `read_live` |
| `render_signals` | `scan_modules` |
| `search_service_docs` | `read_calibration` |
| `lookup_recall_tsb` | `probe_dids` |
| `get_vehicle_profile` | |

Cette distinction doit apparaître **dans la description de chaque outil**, pour que l'agent ne propose pas une lecture live quand le véhicule est à 40 km. Et une ressource `mecabot://status` doit répondre bon marché à « l'adaptateur est-il là, le véhicule répond-il », afin que l'agent puisse vérifier avant de proposer plutôt qu'échouer pour apprendre.

---

## 4. Taxonomie d'erreurs — la partie qui décide si l'agent raisonne juste

C'est la section la plus sous-estimée d'une conception de serveur MCP. Si tous les échecs remontent comme « erreur », l'agent produit des conclusions fausses avec assurance.

**Chaque condition ci-dessous doit être un état distinct et nommé, avec ce que l'agent doit en conclure :**

| Condition | Ce que l'agent doit conclure |
|---|---|
| Aucun adaptateur sur aucun port série | Matériel absent. **Les outils hors-ligne restent utilisables** — le dire. |
| **Port détenu par un autre processus Mecabot** | ⚠️ **Ajouté le 2026-08-26.** Condition distincte de la précédente, et l'action humaine n'est pas la même : arrêter l'autre processus, ou le viser par `--remote`. ⛔ **Jamais confondue avec « matériel absent »** — ça enverrait l'humain vérifier un câble qui est branché (§2, `09` §11.2). |
| Adaptateur présent, aucune réponse du véhicule | Contact coupé, ou adaptateur non branché au camion. Actionnable par l'humain. |
| Module silencieux sur le bus interrogé | Module absent, **ou mauvais bus, ou endormi**. ⚠️ **Jamais « module en panne ».** |
| `7F .. 31` requestOutOfRange | **Ce DID n'existe pas sur ce module.** Résultat **normal**, voir ci-dessous. |
| `7F .. 33` securityAccessDenied | Frontière du palier 4 (`09` §4). Attendu. **Ne pas réessayer.** |
| `7F .. 22` conditionsNotCorrect | État véhicule inadéquat (moteur tournant / arrêté). Actionnable par l'humain. |
| `7F .. 78` responsePending | Normal, attendre. Ne pas remonter à l'agent. |
| Délai dépassé en cours de session | Possible expiration de `$3E`. Le serveur réessaie **une** fois en transparence ; si ça se reproduit, remonter. |
| Valeur lue mais **invariante** | ⚠️ **Non validée**, quelle que soit la cause. Cinquième méthode de validation, `09` §4 et §6.2. |
| **Refus par une passerelle sécurisée** | ⚠️ **Ajouté le 2026-08-25.** Condition distincte de `7F .. 33` — ce n'est pas le même niveau du réseau, et l'agent ne doit pas en conclure la même chose. Le palier 1 devrait rester accessible ; le palier 3 probablement pas. Aucun contournement n'existe pour ce projet (`11` §6.1). |
| **Aucun profil ne correspond au véhicule** | ⚠️ **Ajouté le 2026-08-25.** N'est **pas** une erreur : **palier 1 seulement, et le dire.** ⛔ **Jamais « appliquer le profil le plus proche »** — ça produirait des grandeurs plausibles et fausses, le pire mode de défaillance du dossier (`11` §10). |

### ⚠️ Il manque un étage — ajouté le 2026-08-28

Le tableau ci-dessus est construit sur les conditions **véhicule** et **UDS**. Or l'adaptateur a **sa propre couche d'erreurs**, avec un catalogue fermé de chaînes documentées (*OBDLink Family Reference and Programming Manual*, rév. F, 2025-08-29, §9.0), et l'action humaine n'y est pas la même. Les confondre avec les `7F` d'UDS fait raisonner faux — exactement ce que cette section combat.

| Chaîne de l'adaptateur | Ce que l'agent doit conclure |
|---|---|
| `NO DATA` | ⛔ **Jamais « module absent ».** Le manuel donne **quatre** causes : requête non supportée, requête bloquée par un filtre, **timeout `STPTO` trop court**, protocole non supporté. La troisième est celle qu'on se cause soi-même |
| `CAN ERROR` | **Signature du mauvais bus** ou du mauvais débit. Attendue si on interroge un module MS-CAN sans avoir commuté (§1) |
| `FC RX TIMEOUT` | Trame de contrôle de flux non reçue : bus chargé, trame filtrée, **ou paire d'adresses absente**. Les paires automatiques supposent `TxID = RxID − 8`, ce qui ne vaut pas pour tous les modules de carrosserie |
| `UNABLE TO CONNECT` | **Contact coupé**, pas d'alimentation au connecteur, ou câblage. Vérifier le contact **avant** de soupçonner le logiciel |
| `BUS BUSY`, `BUS ERROR`, `FB ERROR` | Câblage, ou **un second outil qui émet en parallèle** |
| `BUFFER FULL`, `OUT OF MEMORY` | Bornes de tampon atteintes. Actionnable : débit UART, `ATH0`/`ATS0`, filtres |
| `STOPPED` | ⚠️ **Un caractère reçu sur l'UART a interrompu une commande en cours.** C'est la **preuve matérielle** de l'exclusivité de §2 : deux écrivains ne produisent pas une erreur propre |
| `LV RESET` | ⚠️ **Réinitialisation par sous-tension** — la puce redémarre et **perd tout son état**. C'est la **preuve matérielle** de la précondition « mainteneur de batterie branché » de `probe_dids` (§7). Un balayage de 40 minutes contact mis, moteur arrêté, sans mainteneur, finit là |
| `ACT ALERT`, `LP ALERT` | Basse consommation imminente. **Second chien de garde**, indépendant de `$3E` — voir §1 |
| `<DATA ERROR`, `<RX ERROR` | Échec d'intégrité : connecteur oxydé, bruit, **ou mauvais débit** |
| `?` | Syntaxe invalide **ou commande inappropriée au contexte**. ⚠️ Ne dit pas laquelle des deux |

**Trois de ces chaînes valident des invariants que le dossier posait par prudence** — `STOPPED` pour le propriétaire unique du port, `LV RESET` pour le mainteneur de batterie, `CAN ERROR` pour la commutation de bus. Ce n'étaient pas des précautions de principe : ce sont des mécanismes nommés dans le manuel du fabricant.

### La règle qui compte le plus

**`7F .. 31` n'est pas une erreur, c'est de l'information.** Lors d'un balayage de DID, environ 65 000 réponses sur 65 536 seront exactement ça. Si cette condition remonte comme erreur d'outil, l'agent conclura que l'outil est cassé — et il aura raison de le croire.

Le balayage doit donc renvoyer **un résumé de ce qui a répondu positivement**, jamais la liste des refus.

---

## 5. Provenance obligatoire sur chaque valeur

`04` §6 établit que des nombres sans attribution acquièrent une autorité qu'ils n'ont pas, et que c'est un des cinq modes de défaillance par hallucination. Sur ce projet, le risque est aggravé : une bonne partie des DID seront décodés par inférence, pas par documentation.

**Enveloppe commune à toute grandeur renvoyée par un outil :**

| Champ | Contenu |
|---|---|
| `name` | nom canonique |
| `value`, `unit` | la grandeur |
| `source` | `j1979` (PID normalisé) · `did_validated` · `did_unvalidated` |
| `validation_method` | **comment on sait que le décodage est juste** — corrélation Mode 01, vérité terrain physique, comportement caractéristique, recoupement outil tiers, ou **invariance** au négatif (les cinq de `09` §4) |
| `provenance` | **d'où vient le décodage, et peut-il être redistribué** — `iso_standard` · `sae_standard` · `own_observation` · `community_published` · `odx_import` · `proprietary_crosscheck` · `unknown` |
| `confidence` | qualitatif, dérivé de `validation_method` seul, jamais un pourcentage inventé |

> ⚠️ **Correction du 2026-08-25 : ce tableau avait un seul champ `validation` là où il en fallait deux.** « Méthode ayant établi le décodage » et « source » sont des questions **orthogonales** — un décodage peut être solidement validé et non redistribuable, ou de provenance publique et mal validé. Le mono-véhicule masquait la confusion parce que la réponse à la seconde était toujours « moi, localement, non publié ». Raisonnement complet en `11` §5.
>
> `confidence` ne dérive que de `validation_method`. **La provenance ne dit rien sur la justesse** — c'est précisément pourquoi il faut deux champs.

Toutes les sorties sont typées via `outputSchema` et renvoyées en `structuredContent` (`03` §6). **Une grandeur `did_unvalidated` ne doit jamais apparaître dans une conclusion sans que sa nature soit dite.**

---

## 6. Deux mécanismes pilotés par le profil

> **Recasté le 2026-08-25.** Cette section s'intitulait « deux précautions propres à ce véhicule » et décrivait des faits sur un Transit *deleted*. Le rescope multi-marques (`11`) impose de la réécrire — **et la réécriture a révélé que les deux précautions n'étaient pas de la même nature.** L'une est bien propre à un exemplaire ; l'autre ne l'a jamais été.

Rappel de la règle de `11` §1 : **aucun fait sur un véhicule particulier n'est écrit dans le code.** Ces deux points restent donc des mécanismes du serveur, mais ce qui les déclenche vient des données de profil.

### 6.1 Avertissement d'absence de preuve, déclenché par le profil d'exemplaire

**Le mécanisme général :** un profil d'exemplaire (couche 4, `11` §4) peut déclarer des **familles de DTC supprimées, avec un motif**. Dès qu'il en existe au moins une, `read_all_dtc` accompagne sa sortie de la mention explicite que **l'absence de code dans ces familles ne prouve rien**.

Sans ça, l'agent conclut « aucun défaut » à partir d'un silence fabriqué. **C'est le seul endroit du dossier où un outil parfaitement correct produirait un raisonnement faux**, et c'est pour ça que l'avertissement est dans l'outil et non dans la documentation.

Trois motifs produisent exactement le même silence trompeur, et le mécanisme les sert tous :

| Motif | Exemple |
|---|---|
| **Calibration modifiée** | Le Transit de référence : *deleted*, donc les familles P24xx, P20xx, P046x sont muettes par construction |
| **Option non montée en usine** | Un module attendu par le profil de modèle mais absent de cet exemplaire ne lèvera jamais ses codes |
| **Variante de marché** | Un équipement réglementaire présent dans une géographie et pas dans l'autre |

**Ce qui rend le recast meilleur que l'original :** le mécanisme sert désormais les véhicules d'origine, pas seulement les véhicules modifiés. Une option non montée est bien plus fréquente qu'un delete, et elle piégeait l'agent de la même façon.

### 6.2 L'invariance n'était pas une précaution locale, c'était une méthode de validation

**Correction.** Je présentais « un DID invariant est suspect, pas validé » comme une conséquence du delete — un DID survivant dans la calibration tout en lisant un capteur physiquement retiré. **C'était mal rangé : la règle est générale et le delete n'en était qu'une cause parmi d'autres.**

Un DID qui ne varie sur **aucune** session, à travers des points de fonctionnement différents, n'est pas validé — quelle qu'en soit la raison : capteur retiré, option non montée, DID orphelin hérité d'une autre variante de marché, ou décodage simplement faux.

**→ Promue en cinquième méthode de validation, négative, dans la méthodologie générale de `09` §4**, où elle s'applique à toute marque. Elle est la moins chère du lot et la seule qui protège contre une valeur *plausible* et fausse — le mode de défaillance qui compte, puisqu'une valeur absurde se repère et qu'une valeur plausible ne se repère pas.

Côté serveur, la conséquence est inchangée : une telle grandeur est marquée suspecte et **jamais promue en `did_validated`**.

---

## 7. Contrat des outils

Chaque outil déclare une **borne de sortie** et tronque de façon déterministe en disant ce qu'il a laissé tomber. Aucun outil ne renvoie de texte non borné (`04` §1).

### En ligne

| Outil | Entrées | Sort | Borne |
|---|---|---|---|
| `read_freeze_frame` | — | Instantané Mode `$02` — **premier réflexe de toute session** | 60-120 tokens |
| `read_all_dtc` | `modules?` | DTC de tous les modules, ordonné par bus, **avec l'avertissement de §6** | 100-500 tokens |
| `read_live` | `signals[]`, `duration_s`, `hz?` | Échantillonne puis renvoie **un résumé statistique**, pas la série | 100-300 tokens |
| `scan_modules` | — | Inventaire par bus des modules qui répondent | 100-200 tokens |
| `read_calibration` | — | CALID + CVN + bits de moniteurs + Mode `$06` (`07` item 16) | 200-400 tokens |
| `probe_dids` | `module`, `range?` | **Résumé des DID ayant répondu**, jamais les refus. Annulable, avec avancement. ⛔ **Exige `discovery_allowed: true` dans le profil de marque — défaut `false`** (`11` §6.2) | résumé seul |

`read_live` ne renvoie **jamais** la série brute : 60 s à 15 signaux et 1 Hz font 900 points, soit des milliers de tokens pour une information que trois statistiques et une détection d'événement portent mieux (`04` §2). Pour voir la forme, il y a `render_signals`.

### Hors-ligne

| Outil | Sort | Borne |
|---|---|---|
| `get_vehicle_profile` | Marque/modèle/année, **quelles couches de connaissance sont disponibles et donc quels paliers sont atteignables**, modifications déclarées, familles de DTC supprimées, rappels, ligne de base d'entretien | ~200 tokens, `ttlMs` long |
| `list_sessions` | Inventaire daté | ~20 tokens/session |
| `summarize_session` | Plages, casiers couverts, événements, anomalies classées | 300-600 tokens |
| `compare_operating_point` | Même signal, même casier, N sessions — **le cœur du longitudinal** | 100-300 tokens |
| `render_signals` | `resource_link` vers un PNG (`plotters`) | image, pas de texte |
| `search_service_docs` | Extraits du corpus, **citation obligatoire** | 200-800 tokens |
| `lookup_recall_tsb` | Rappels et TSB pertinents | 100-400 tokens |

**`get_vehicle_profile` change de nature avec le multi-marques, et en mieux.** Il ne rend plus une fiche de faits sur un camion connu ; il rend **ce qui est connaissable sur ce véhicule** — quelles couches de profil sont présentes, donc quels paliers de capacité sont atteignables, donc ce que l'agent a le droit de proposer. C'est l'outil que l'agent doit appeler en premier sur un véhicule qu'il ne connaît pas, et c'est ce qui l'empêche de promettre une lecture de charge de suie sur un véhicule sans profil de couche 3.

`compare_operating_point` porte la valeur du projet et **doit refuser de répondre** si les sessions comparées ne partagent pas de casier commun suffisamment peuplé. Comparer des moyennes de session produit du bruit convaincant (`09` §7A) : l'outil doit dire « pas de recouvrement exploitable » plutôt que de produire un nombre.

### Ressources et prompts

**Ressources :** profil du véhicule · `mecabot://status` (adaptateur, véhicule, file d'attente) · état de la base de DID · historique des DTC · ligne de base d'entretien.

**Prompts** — ils portent des règles qu'il ne faut pas espérer que l'agent redécouvre :

| Prompt | Règle qu'il encode |
|---|---|
| « analyse cette session » | Anomalie **relative à l'enveloppe attendue**, jamais à une valeur absolue de mémoire |
| « qu'est-ce qui a changé depuis la dernière fois » | **Conditionnement par casier de point de fonctionnement**, obligatoire |
| « prépare l'intervention pour X » | Citation obligatoire · préconditions · avertissements de sûreté · l'humain reste l'actionneur |

`ttlMs` / `cacheScope` (`03` §9) sur ce qui est stable : profil véhicule, définitions de DID, corpus documentaire. Pas sur les DTC ni les données live.

---

## 8. Ce qu'il ne faut surtout pas construire

Cette section vaut les précédentes, parce que c'est là qu'on perd la garantie de lecture seule.

**⛔ Aucun outil de commande brute.** Pas de `send_command`, pas de `send_raw`, pas de passe-plat AT. **Un tel outil annule à lui seul toute l'énumération fermée de services de `09` §6** : l'agent pourrait émettre `$2F` en écrivant de l'hexadécimal, et rien dans le code ne l'en empêcherait. C'est exactement le genre d'outil de commodité qu'on ajoute pour déboguer et qu'on oublie de retirer. **Il ne doit jamais exister**, même derrière un drapeau, même en développement — un drapeau se laisse activer.

Le débogage se fait au terminal série, hors du serveur.

**⛔ Aucun `clear_dtcs`.** `shoka-jp/obd2-mcp` en a un, derrière confirmation utilisateur (`05` §4). Ici c'est `$14`, un service d'écriture, exclu par construction. Et effacer des codes détruit précisément la donnée longitudinale que le projet existe pour accumuler. Si tu veux effacer, FORScan le fait, sous ton contrôle direct.

**⛔ Aucune dépendance à l'elicitation pour la sûreté.** `03` §3 documente l'elicitation MRTR, et `07` item 12 s'inquiétait de son support par les clients. **Sans écriture, la question tombe** : il n'y a aucune action dangereuse à faire confirmer. La sûreté est structurelle, pas conversationnelle — et c'est bien plus solide. L'item 12 de `07` peut être déclassé.

**⛔ Aucune annotation d'outil comme garde-fou.** `readOnlyHint` reste utile comme indication d'intention pour le client, mais la spec dit explicitement que les clients doivent traiter les annotations comme non fiables (`03` §6). Elles décrivent, elles ne protègent pas.

---

## 9. Développer sans le camion — le trou qu'il fallait combler

Ce document dit en §11 que les deux premières étapes de construction ne produisent **aucune fonctionnalité visible** et sont les plus importantes. Ça pose une question qu'aucun document du dossier ne traitait : **comment sait-on qu'elles sont justes ?** Le camion n'est pas dans l'entrée en permanence, et on ne débogue pas une machine à états d'adaptateur en allant s'asseoir dehors à chaque itération.

La réponse est déjà dans l'architecture, il suffit de la voir : **le `trait Transport` de `09` §2, introduit pour régler le multiplateforme, règle aussi le test.**

```text
                    trait Transport
                          │
        ┌─────────────────┼─────────────────┬──────────────────┐
        │                 │                 │                  │
   serialport        Web Serial        REJEU              SIMULATION
   (natif)           (WASM)            journal capturé    réponses fabriquées
                                       → tests            → cas limites
```

Deux implémentations de test, pour deux usages distincts :

| Implémentation | Usage |
|---|---|
| **Rejeu** — relit un journal série réel, horodaté | Fidélité. Développer le décodage et la machine à états contre ce que le véhicule a *réellement* répondu, y compris ses bizarreries. |
| **Simulation** — fabrique des réponses | Cas limites qu'on ne peut pas provoquer à volonté : `7F .. 31` en rafale, délai dépassé au milieu d'une trame, réponse tronquée, module qui s'endort. |

### La conséquence pratique, et elle change ce qu'il faut faire ensuite

**L'item 4bis de `07` doit être journalisé.** Ce n'était présenté que comme « caractériser l'adaptateur » — quelques commandes au terminal pour noter des réponses. Il faut en plus **capturer la session complète dans un fichier**, parce que ce journal devient **le premier jeu de test du projet**.

Autrement dit, la séance au terminal série ne sert pas seulement à répondre à quatre questions : elle produit l'artefact contre lequel les étapes 1 et 2 se développent. Une séance de 45 minutes avec le camion peut couvrir des semaines de développement à l'intérieur.

À capturer largement, tant qu'on y est : l'initialisation, une commutation de bus, quelques `$22` qui réussissent, **quelques-uns qui échouent**, un `$19`, et une session laissée volontairement expirer pour voir ce que fait `ATST`.

`rmcp` fournit par ailleurs un **transport in-process** (`09` §3), qui permet de piloter le serveur MCP depuis un test sans lancer de sous-processus. Combiné au transport de rejeu, ça donne une chaîne testable de bout en bout, sans véhicule et sans adaptateur.

**Un point qui ne se teste pas ainsi :** la concurrence de §2. Une file avec un propriétaire unique se vérifie par des tests qui lancent des appels réellement simultanés — c'est du test de logique, pas de protocole, et ça n'a pas besoin de transport du tout.

`ELM327-emulator` (`05` §4) reste utile comme oracle indépendant, mais c'est du Python : à considérer comme un outil de validation ponctuelle, pas comme la base de la suite de tests.

---

## 10. Ordre de construction

Révision de `09` §9 côté serveur, à la lumière de ce document :

| # | Contenu | Pourquoi dans cet ordre |
|---|---|---|
| 1 | Acteur port série + file d'attente + machine à états d'adaptateur | **Tout le reste en dépend.** Le construire après, c'est le reconstruire. |
| 2 | Taxonomie d'erreurs et enveloppe de provenance | Deux structures à figer avant d'écrire un seul outil, sinon on les rétro-ajuste partout |
| 3 | `read_freeze_frame`, `read_all_dtc`, `read_calibration` | Les trois outils à plus forte densité, tous en palier 1 |
| 4 | `read_live` + résumé statistique | Introduit la logique de compression de `04` |
| 5 | `scan_modules` + commutation MS-CAN → **palier 2** | Le jalon qui sort de la VM |
| 6 | SQLite + outils hors-ligne | Rend le serveur utile sans véhicule |
| 7 | `probe_dids` + annulation + avancement | Le seul outil long ; a besoin de 1 et 2 solides |
| 8 | Casiers, CUSUM, `render_signals`, `compare_operating_point` | Demande des sessions accumulées |

**Les étapes 1 et 2 ne produisent aucune fonctionnalité visible et sont les plus importantes.** C'est la tentation classique de commencer par un outil qui affiche des DTC ; le coût se paie en réécriture. Elles se développent contre le transport de rejeu de §9, pas contre le véhicule.

---

## 11. Ce que ce document ne spécifie pas encore

Distinguer ce qui est **vérifié**, ce qui reste **inconnu**, et ce qui est simplement **pas encore décidé** — les trois ont des conséquences très différentes sur la question « peut-on commencer ? ».

### ✅ Levé le 2026-08-25

**`rmcp` 3.1.4, Apache 2.0**, révision `2026-07-28` : `outputSchema` / `structuredContent`, `notifications/progress`, stdio, Streamable HTTP, transport in-process — tous couverts. Détail en `09` §3. **Aucune couche de protocole à écrire soi-même.** Seul résidu : `resource_link` non confirmé, ce qui ne concerne que `render_signals`.

### ⚠️ Encore inconnu — et tout se lève dans la même séance de terminal série

- **Le délai de commutation de bus et la nécessité de réinitialiser.** Toute la §1 en dépend.
- **Le comportement de `ATST` et le besoin réel de `$3E`** sur session longue. Décide si le chien de garde de §1 est un détail ou un mécanisme central.
- **La taille maximale de réponse réassemblée par le STN2120.** Borne ce qu'un seul appel peut rapporter.
- **Le format exact des lignes de réponse** et le comportement sur `7F`. Le décodeur en dépend directement.

C'est l'item 4bis de `07`, **et c'est le seul véritable préalable à l'écriture de code** — d'autant plus depuis §9, puisque cette séance produit aussi le jeu de test.

### 🔲 Pas encore décidé — mais rien de bloquant

Ces choix se prennent dans la première heure de développement, pas avant. Ils sont listés pour que « le dossier est complet » ne soit pas surinterprété :

| Sujet | Ce qui manque |
|---|---|
| Découpage en crates | `09` §2 implique la séparation cœur / transport / MCP / stockage, sans la spécifier |
| ~~Nommage canonique des signaux~~ | ✅ **Décidé le 2026-08-25 : anglais, `snake_case`**, aligné J1979 et conventions Rust. Descriptions et sorties vers l'agent en français. C'était le point de cette liste dont le report coûtait le plus cher, puisque le nom se retrouve dans la base et dans chaque sortie d'outil. |
| Schéma SQLite | `09` §7 le décrit en entités, pas en DDL avec types, clés et index. Gagne deux entités le 2026-08-25 (`vehicle_profile`, `vehicle_instance`) et un champ scindé sur `did_definition`. |
| **Schéma de profil de couche 3** | 🕓 **Différé volontairement, et c'est une décision, pas un oubli.** `11` §8 pose la règle des deux instances : aucun champ n'est généralisé depuis un seul véhicule observé. Le concevoir maintenant serait exactement l'erreur d'abstraction prématurée que le rescope multi-marques rend possible. |
| Format de sérialisation des profils | TOML, JSON ou SQLite. Faible conséquence, à trancher en écrivant le chargeur. |
| Licence | Non décidée. Une seule contrainte en vigueur : **aucune dépendance GPL** (`11` §7). |
| Corpus documentaire | Étape 8 de la feuille de route ; dépend d'un abonnement pas encore pris |
