# 08 — Architecture « FORScan en amont » — ❌ PRÉMISSE ERRONÉE, DOCUMENT SUPERSÉDÉ

> ## ⛔ Ce document repose sur un malentendu — ne pas l'utiliser comme référence
>
> **2026-08-24 :** j'avais interprété le rescope du 2026-08-19 comme « FORScan devient la couche d'acquisition et Mecabot lit ses fichiers de sortie ». **C'est faux.** L'intention réelle est de construire **l'équivalent multiplateforme de FORScan**, avec l'agent IA branché directement dessus — FORScan étant justement trop limitant sur les plateformes où il peut tourner (Windows).
>
> Ce qui reste valide de ce document, et qui a été repris dans `09` :
>
> - §4 (pipeline d'ingestion, stockage SQLite) — l'étage de stockage et de dérivation est le même
> - §5 (surface MCP proposée) — les outils restent globalement bons
> - §6 (les cinq familles d'analyse) — **entièrement valide**, notamment le conditionnement sur le point de fonctionnement
> - §7 (dépendance à un outil fermé, frontière juridique) — partiellement, voir `09`
>
> Ce qui est faux : §1 (« ce que le rescope dissout » — rien n'est dissous, les obstacles reviennent), §2 (format des sorties FORScan — sans objet), §3 (modes de couplage par fichier), §8 (ordre de réalisation), §9 (effet sur le dossier).
>
> **→ Voir `09-architecture-cross-platform.md`.**

**Date du rescope :** 2026-08-19
**Formulation mal interprétée :** « FORScan mais avec les données directement connectées dans l'AI agent de mon choix. »

Ce document remplace l'ambition « middleware de diagnostic OEM » par une architecture beaucoup plus étroite et beaucoup plus faisable. Il rend caduques des pans entiers du dossier — signalés en §9.

---

## 1. Ce que le rescope dissout

FORScan devient la **couche d'acquisition**. Mecabot devient une **couche d'analyse et de mémoire** qui lit ce que FORScan produit. Rien ne parle au véhicule sauf FORScan.

| Problème identifié dans le dossier | Statut après rescope |
|---|---|
| Base de PID/DID étendus Ford non réplicable (`00`, `02` §3) | **Dissous.** FORScan la fournit, légitimement, sur ton véhicule. |
| UDS $27 SecurityAccess publiquement inconnu (`01` §2) | **Sans objet.** Aucune session UDS n'est ouverte par Mecabot. |
| Commutation MS-CAN qu'aucune bibliothèque ne gère (`05` §3.1) | **Dissous.** C'était « le premier vrai morceau d'ingénierie du projet ». FORScan le fait. |
| SocketCAN / `can-utils` / J2534 absents de macOS (`05` §2) | **Dissous.** Plus aucune pile CAN à faire tourner sur macOS. |
| Lecture seule imposée par impossibilité technique (`01` §2.4) | **Devient structurel plutôt que subi.** Mecabot lit des fichiers. Il n'a aucun chemin de code vers le véhicule — c'est exactement le garde-fou structurel que `03` §6 réclamait, et il est gratuit. |
| Exposition DMCA (`01` §2.1) | **Fortement réduite.** Aucun reverse engineering. Tu lis des fichiers qu'un outil sous licence a écrits sur ta machine, à propos de ton véhicule. |
| Réveil asynchrone de l'agent impossible (`03` §4) | **Sans objet.** Le modèle était de toute façon « journaliser, l'agent lit ensuite ». C'est maintenant le modèle nominal, pas un contournement. |

La VM Windows reste nécessaire — mais elle l'était déjà (`05` §2), et elle héberge maintenant la partie utile plutôt qu'un pis-aller.

**Ce qui ne change pas :** il faut toujours un **OBDLink EX** (69,95 $). FORScan a exactement le même besoin de commutation MS-CAN électronique. Le §1 de `05-materiel-et-stack.md` reste valide intégralement.

---

## 2. Ce que FORScan produit — **la nouvelle inconnue bloquante**

Toute l'architecture dépend de la forme exacte des sorties de FORScan. Je sais qu'elles existent ; je ne connais pas leur format en détail, et je ne vais pas le supposer.

| Sortie | Ce que j'en sais | Fiabilité |
|---|---|---|
| **Journal CSV du Data Logger** | FORScan journalise les PID sélectionnés en CSV horodaté. C'est la base de tous les fils de forum « poste ton log ». | [COMMUNAUTÉ] — existence certaine, **format non vérifié** |
| **Rapport de scan / DTC** | FORScan sait sauvegarder un rapport du scan multi-modules avec les DTC de chaque module | [COMMUNAUTÉ] — format non vérifié |
| **Données As-Built** | Lecture confirmée sur Transit 2016 (`01` §3) ; exportables | [COMMUNAUTÉ] — format non vérifié |
| **Journaux applicatifs** | FORScan écrit ses propres logs de session | [NON VÉRIFIÉ] |
| **API / CLI / scripting** | Aucune trace. Je pars du principe qu'il n'y en a pas. | [NON VÉRIFIÉ] |

### Les questions de format qui changent la conception

Dans l'ordre d'importance :

1. **Échantillonnage simultané ou tour de rôle ?** Avec 20 PID sélectionnés, FORScan ne peut pas les interroger en même temps — la contrainte requête/réponse de `04` §1.2 s'applique à lui aussi. Deux possibilités : soit une ligne CSV contient des valeurs acquises jusqu'à une seconde d'écart, soit les colonnes ont des trous. **Ça détermine si on peut corréler deux signaux à un instant donné.** C'est le point qui pourrait silencieusement corrompre toute analyse croisée, donc c'est la première chose à caractériser.
2. **Cadence réelle atteinte** en fonction du nombre de PID sélectionnés. Détermine combien de PID on peut demander sans dégrader la résolution temporelle en dessous de l'utile.
3. **Délimiteur et séparateur décimal.** ⚠️ Sur un Windows en locale française, un CSV peut sortir avec `;` en délimiteur et `,` en décimal. À vérifier plutôt qu'à découvrir après avoir parsé six mois de journaux de travers.
4. **Stabilité des noms de PID** entre versions de FORScan. Si un nom change à une mise à jour, l'historique longitudinal casse. → **exigence de conception : une couche de normalisation vers des noms canoniques internes**, pour qu'un renommage côté FORScan n'invalide pas deux ans de mesures.
5. **Unités présentes ou implicites** dans l'en-tête.
6. **Profils de PID sauvegardables ?** Si FORScan permet d'enregistrer des jeux de PID réutilisables, ça règle en grande partie la friction décrite en §7.

**Méthode :** une seule session de journalisation de cinq minutes répond aux six questions. C'est l'item à ajouter en tête du niveau 1 de `07-verifications-a-faire.md`.

---

## 3. Deux modes de couplage, un seul pipeline

C'est le point élégant de cette architecture : le mode post-mortem et le mode quasi-live sont **le même code de lecture**, à un détail près — quand on lit.

**Mode A — post-session (le mode nominal).** Tu roules, FORScan journalise, tu sauvegardes. Mecabot ingère le fichier ensuite. Latence : ton flux de travail.

**Mode B — quasi-live.** FORScan écrit le CSV **pendant** qu'il journalise. Un lecteur incrémental (sémantique `tail`) peut suivre le fichier en cours d'écriture et publier les nouvelles lignes au fur et à mesure. Latence de quelques secondes, sans aucune coopération de FORScan, sans API, sans capture d'écran.

⚠️ [NON VÉRIFIÉ] : ça suppose que FORScan écrit et vide son tampon en continu plutôt qu'à la fermeture du fichier. À tester en regardant si le fichier grossit pendant la journalisation. Si c'est le cas, tu obtiens du diagnostic assisté **pendant que tu roules ou pendant que le moteur tourne au ralenti**, ce qui est proche de ce que « connecté directement à l'agent » suggère.

Le mode B ne change rien à l'analyse : mêmes parseurs, même stockage, même surface MCP. Il ne fait qu'avancer le moment où les données arrivent.

---

## 4. Pipeline d'ingestion

Cinq étapes, toutes déterministes, toutes hors LLM.

```text
Dossier partagé VM Windows ↔ macOS
        │
        ▼
1. SURVEILLANCE    détecte les nouveaux fichiers / la croissance d'un fichier
        │
        ▼
2. ANALYSE         CSV du Data Logger, rapports de DTC, As-Built
        │
        ▼
3. NORMALISATION   noms de PID FORScan → noms canoniques internes
                   unités → unités SI de référence
                   horodatages → instants absolus datés
        │
        ▼
4. STOCKAGE        SQLite, un seul fichier
        │
        ▼
5. DÉRIVATION      détection d'événements (CUSUM), regroupement par point
                   de fonctionnement, statistiques par session, deltas
                   entre sessions
```

**Le stockage :** SQLite. Un fichier, pas de serveur, et l'échelle est triviale — une session de 30 min à 15 PID/1 Hz fait ~27 000 échantillons ; une année de sessions mensuelles fait quelques centaines de milliers de lignes. Aucune base de séries temporelles n'est justifiée.

Structure descriptive (design, pas schéma d'implémentation) :

| Entité | Contenu |
|---|---|
| `session` | date, durée, version de FORScan, PID présents, odomètre, contexte libre (« bruit à froid depuis 2 semaines ») |
| `mesure` | session, nom canonique, instant, valeur |
| `point_de_fonctionnement` | session, instant, casier (régime × charge × température) — dérivé |
| `evenement` | session, instant, type, signaux impliqués, amplitude — dérivé |
| `dtc` | code, module, session de première et de dernière observation, compteur, statut |
| `intervention` | date, ce qui a été fait, pièces, kilométrage — **saisi à la main** |
| `pid_alias` | nom FORScan observé → nom canonique, version — la couche anti-renommage de §2 |

Les deux dernières entités portent une bonne part de la valeur. `intervention` est ce qui permet de dire « la dérive s'est arrêtée quand tu as changé le filtre » ; aucune donnée du véhicule ne le contient.

---

## 5. Surface MCP proposée

Toutes les opérations sont en lecture. Aucune ne touche au véhicule. Aucune ne rend de données brutes au modèle.

### Outils

| Outil | Rend | Ordre de grandeur |
|---|---|---|
| `list_sessions` | inventaire daté : durée, PID présents, DTC actifs, odomètre | ~20 tokens/session |
| `summarize_session` | plages par signal, points de fonctionnement couverts, événements détectés, anomalies classées | **300-600 tokens** |
| `get_session_events` | liste d'événements : « t=412 s, pression rampe −300 bar en 2 s à régime stable » | 200-500 tokens |
| `get_dtc_history` | tous les DTC jamais vus, première/dernière occurrence, fréquence, corrélation avec les interventions | 100-300 tokens |
| `compare_operating_point` | même signal, même casier de fonctionnement, N sessions — **le cœur du suivi longitudinal** | 100-300 tokens |
| `render_signals` | `resource_link` vers un PNG de tracés | image, pas de texte |
| `search_service_docs` | extraits du corpus RAG, **avec citation obligatoire** | 200-800 tokens |
| `lookup_recall_tsb` | rappels et TSB pertinents pour un symptôme ou un code | 100-400 tokens |
| `prepare_intervention` | procédure, source citée, préconditions, avertissements de sûreté, **ce qu'il faut faire dans FORScan** | 300-800 tokens |

`render_signals` applique directement le résultat de `04` §3.1 : rendre les séries en graphique pour un modèle multimodal coûte 3,6 à 10,4× moins de tokens **et** donne une meilleure précision qu'une sérialisation numérique. Sous cette architecture, c'est facile — les données sont déjà au repos dans SQLite.

Toutes les sorties sont typées via `outputSchema` (`03` §6). Les gros volumes sortent en `resource_link`, jamais incorporés.

### Ressources

- profil du véhicule (VIN décodé, moteur, options, configuration As-Built)
- ligne de base d'entretien et historique des interventions
- historique des DTC
- inventaire du corpus documentaire

### Prompts

- « analyse cette session »
- « qu'est-ce qui a changé depuis la dernière fois »
- « prépare l'intervention pour X »
- « qu'est-ce que je devrais journaliser la prochaine fois » ← celui-ci ferme la boucle avec §7

---

## 6. Ce que l'analyse produit réellement

C'est la question posée. Cinq familles, par valeur décroissante.

### A. Suivi de dérive longitudinal — la raison d'être

Le seul chose que FORScan ne peut structurellement pas faire : il n'a pas de mémoire entre les sessions.

**La subtilité qui décide de la validité :** on ne compare pas des moyennes de session. Une pression de rampe moyenne dépend entièrement de la façon dont tu as conduit ce jour-là. Il faut **conditionner sur le point de fonctionnement** — regrouper les échantillons par casier (régime × charge × température moteur) et ne comparer qu'à l'intérieur d'un même casier, entre sessions.

Fait naïvement, ce module produit du bruit convaincant. Fait correctement, il produit :

| Signal, à point de fonctionnement comparable | Ce qu'une tendance révèle |
|---|---|
| Correction de débit par injecteur (×5) | Une valeur qui s'écarte des quatre autres = injecteur identifié **avant** le raté |
| Écart EGR commandé/réel | Encrassement progressif, avant le P0401 |
| Charge de suie DPF et rythme de chargement | DPF qui se charge tôt, avant le voyant |
| Delta du compteur de régénérations | Changement de profil d'usage, ou DPF qui se dégrade |
| Réponse des aubes de turbo à la mise en charge | Collage naissant |
| Consommation DEF par 1 000 km (si SCR) | Hausse = dosage bloqué ouvert ; baisse = injecteur mort |
| Delta température de démarrage vs ambiante | Santé du thermostat |
| Tension au démarrage | État de santé de la batterie, sur une ligne de base réelle |

### B. Triage de session

Une session brute fait ~27 000 échantillons, soit ~200 000 tokens en texte. Après triage : 300 à 600 tokens qui répondent à « qu'est-ce qui sort de l'ordinaire ». La comparaison se fait contre l'enveloppe attendue — issue du manuel, de tes propres sessions antérieures, ou de valeurs communautaires, **la source étant toujours indiquée**.

### C. Détection d'événements dans la session

CUSUM sur les signaux clés (`04` §2), en code déterministe. L'agent reçoit « à t=412 s, la pression de rampe a chuté de 300 bar en 2 s alors que le régime était stable » au lieu de 27 000 lignes. C'est la compression qui compte, et elle est triviale à implémenter.

### D. Corrélation documentaire

Symptôme ou DTC → TSB, rappel, procédure, avec citation vers le fragment source. Le corpus est celui de `04` §6.5 : quelques centaines de pages achetées, pour **un** véhicule. Découpage hiérarchique (94,1 % contre 86,2 % pour un découpage naïf, et **33 points d'écart sur les questions dépendant de tableaux** — or les couples de serrage et les brochages sont tous tabulaires).

Règle non négociable de `04` §6.2 : **une réponse sans citation à une question de spécification est traitée comme hallucinée.**

### E. Préparation d'intervention

Exactement ce que `01` §2.4 recommandait, et qui devient naturel ici : Mecabot identifie la procédure, cite la source, liste les préconditions et les avertissements de sûreté, et te dit quoi faire dans FORScan. L'humain reste l'actionneur, avec l'outil qui a le droit et la capacité.

Les avertissements de `02` §8 s'appliquent intégralement — au premier chef la régénération DPF forcée : 550-700 °C à l'échappement, extérieur obligatoire, et **jamais** si une dilution d'huile par le carburant est suspectée.

**Le principe qui traverse les cinq familles :** tout calcul numérique se fait en code déterministe ; le LLM reçoit des conclusions et peu de grandeurs. Ce n'est pas une optimisation de tokens, c'est une condition de justesse — `04` §3 établit que les LLM perdent l'information de magnitude et d'échelle à la tokenisation, et que retirer le LLM de trois méthodes de prévision par LLM **améliore** souvent le résultat.

---

## 7. Ce que ça coûte

Quatre limites réelles, à assumer.

**1. La sélection des PID précède tout.** L'agent ne peut pas demander « lis-moi ce PID maintenant » — il voit ce que tu as journalisé. S'il lui manque un signal, c'est une autre session.

*Atténuation :* définir des jeux de PID standards par type d'enquête (« ligne de base diesel », « surveillance DPF », « surveillance injecteurs »), configurés une fois. Le prompt « qu'est-ce que je devrais journaliser la prochaine fois » transforme cette limite en boucle d'itération explicite plutôt qu'en frustration.

**2. Tu es le goulot d'acquisition.** Les sessions arrivent quand tu branches. Une année de sessions mensuelles, c'est douze points de mesure — suffisant pour de la dérive lente, insuffisant pour autre chose. **C'est un instrument dont la valeur se construit sur des années, pas sur des semaines.** Autant le savoir maintenant.

**3. Pas de temps réel au sens strict.** Le mode B de §3 donne quelques secondes de latence si FORScan vide son tampon en continu. Ce n'est pas de la surveillance embarquée. Mais `03` §4 avait déjà établi qu'aucun réveil temps réel de l'agent n'était possible via MCP — donc cette capacité était de toute façon fictive.

**4. Dépendance à un outil fermé.** Si FORScan change de format de sortie, le parseur casse. La couche `pid_alias` de §4 amortit les renommages ; un changement de format de fichier demanderait un vrai correctif. Risque modéré et détectable — les mises à jour de FORScan sont manuelles.

### La frontière à tenir

Utiliser FORScan sur ton véhicule et lire les fichiers qu'il écrit, c'est de l'usage normal. **Extraire ou redistribuer sa base de correspondance PID, c'est autre chose.** Les règles du forum FORScan interdisent explicitement de discuter numéros de PID et reverse engineering, et Ford a déjà employé le DMCA contre des travaux de ce genre (`01` §2.1).

Conséquence concrète : la table `pid_alias` est un artefact local, dérivé de tes propres journaux, **qui ne se publie pas**. C'est aussi ce qui protège le projet.

---

## 8. Ordre de réalisation suggéré

| Étape | Contenu | Ce qui la débloque |
|---|---|---|
| 0 | Les 4 vérifications gratuites de `07` niveau 0 | rien — à faire d'abord |
| 1 | OBDLink EX + FORScan gratuit en VM. Une session de journalisation de 5 min. | 70 $ |
| 2 | **Caractériser le CSV** — les six questions de §2 | étape 1 |
| 3 | Parseur + normalisation + SQLite. Rejouer les journaux existants. | étape 2 |
| 4 | Serveur MCP en stdio, trois outils : `list_sessions`, `summarize_session`, `get_dtc_history` | étape 3 |
| 5 | Détection d'événements et casiers de point de fonctionnement | quelques sessions accumulées |
| 6 | `render_signals` (graphiques pour lecture multimodale) | étape 5 |
| 7 | Corpus RAG depuis l'abonnement documentaire | ~30-115 $ |
| 8 | `compare_operating_point` — le suivi longitudinal | **plusieurs mois de sessions** |

L'étape 8 est celle qui porte la valeur, et c'est la seule qu'aucun raccourci n'accélère : elle demande du temps calendaire. Les étapes 3 et 4 sont modestes et donnent déjà quelque chose d'utilisable.

---

## 9. Effet sur le reste du dossier

| Document | Statut |
|---|---|
| `00-synthese.md` | Le « mur de parité FORScan » est **résolu** par le rescope, pas contourné. Le repositionnement en « couche d'analyse et de mémoire » qui y était recommandé devient l'architecture entière. |
| `01-acces-donnees-et-securite.md` | **Reste valide et central.** §4 (coûts) et §5 (CASIS) inchangés. §2 (UDS $27) devient une explication de *pourquoi* FORScan est en amont, plutôt qu'un obstacle. |
| `02-profil-vehicule.md` | **Intégralement valide.** C'est la carte de ce qu'il faut journaliser et surveiller. |
| `03-architecture-mcp.md` | Valide. §6 (`resource_link`, `outputSchema`, annotations) et §8 (stdio) directement applicables. §3 (elicitation) devient **moins critique** : sans écriture véhicule, il n'y a plus d'action dangereuse à confirmer. §4 (réveil asynchrone) devient sans objet. |
| `04-budget-tokens.md` | **Valide, et c'est le document le plus utile du dossier sous cette architecture.** §2 (compression), §3.1 (encodage visuel), §5 (freeze frame, Mode 06), §6 (RAG) s'appliquent tous directement. §1 (arithmétique du bus) devient du contexte historique. |
| `05-materiel-et-stack.md` | **§1 (adaptateurs) intégralement valide** — FORScan a le même besoin d'OBDLink EX. **§2, §3.1, §3.2, §3.3, §5 deviennent du matériel de phase 2** : la pile python-can / can-isotp / udsoncan, le mur SocketCAN, opendbc, SavvyCAN ne concernent qu'un accès direct au bus que cette architecture n'a plus. §4 (art antérieur) reste instructif, surtout `shoka-jp/obd2-mcp` pour la frontière d'outils en lecture seule. |
| `06-critique-du-document-initial.md` | Valide. Le rescope confirme plusieurs de ses conclusions par un autre chemin. |
| `07-verifications-a-faire.md` | **Niveau 0 inchangé et toujours prioritaire.** Ajouter les six questions de format de §2 en tête du niveau 1. Les items 12-14 (elicitation MRTR, débit STN, commutation MS-CAN) deviennent **sans objet ou de phase 2**. |

### Phase 2, si le besoin apparaît

Rien n'interdit d'ajouter plus tard un accès direct en lecture aux PID **standards** (Mode 01/02/03/06) avec l'OBDLink EX, en parallèle de FORScan : ça donnerait du live sur une quinzaine de signaux normalisés, FORScan restant la source des PID étendus. C'est là que `05` §3.1 redeviendrait pertinent. À ne pas construire avant d'en avoir constaté le besoin.
