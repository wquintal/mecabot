# 12 — Standards de développement : ce qui est repris de punt-kit, et ce qui ne l'est pas

**Date :** 2026-08-26
**Objet :** structurer le développement de Mecabot en s'appuyant sur **punt-kit** (`github.com/punt-labs/punt-kit`), sans en adopter les parties qui présupposent une organisation.
**Préalables :** `09` (architecture retenue), `10` (conception du serveur MCP).

Toutes les citations de ce document proviennent des standards épinglés à la révision **`278e9cc`**. Le dépôt est actif ; **le contenu peut avoir changé** — les citations sont datées, pas éternelles.

---

## 1. Ce que punt-kit est, et le cadre d'emprunt

punt-kit remplit trois fonctions à la fois : **autorité de standards** (28 documents dans `standards/`), **échafaudeur** (`punt init`) et **auditeur** (`punt audit`, lecture seule). Licence MIT.

⚠️ **Point de cadrage important.** punt-labs appartient au dirigeant de l'entreprise de William, **pas à lui**. Trois conséquences pratiques :

| Fait | Conséquence |
|---|---|
| MIT, dépôt public | L'usage des standards et de l'outillage ne pose aucun problème. |
| Le dépôt n'est pas sous notre contrôle | Les standards bougeront sans nous. **Épingler la révision** consultée plutôt que suivre `main` aveuglément. |
| Combler un manque en amont serait du travail non rémunéré pour l'organisation d'un tiers | Écrire un `rust.md` amont est **une décision de William**, pas une conséquence technique. Le fossé de §2 se contourne localement sans rien contribuer. |

Le mode d'emploi retenu est donc l'**emprunt sélectif et cité**, pas l'adoption. Chaque section ci-dessous dit ce qui est pris, ce qui est refusé, et pourquoi.

---

## 2. Le fossé Rust : les standards se transposent, l'outillage est inopérant

**⛔ [VÉRIFIÉ] Aucun fichier du dépôt ne mentionne Rust.** Les standards de langage couvrent Python, Go, C, Swift et Pharo. `punt init` détecte Python, Node.js, Swift et les projets de documentation — **pas Cargo**.

Ce n'est pas un manque cosmétique, et il faut le voir précisément parce qu'il décide de ce qui est utilisable :

| Composant punt-kit | État pour Mecabot |
|---|---|
| `standards/architecture.md`, `cli.md`, `logging.md`, `filesystem.md`, `permissions.md` | ✅ **Transposables** — ce sont des règles de conception, indépendantes du langage |
| `punt doctor` | ⛔ **Inopérant** — vérifie Python, `uv`, `ruff`, `mypy`, `pyright` |
| `punt release` (pipeline en 11 phases) | ⛔ **Inopérant** — synchronise `pyproject.toml`, `src/<pkg>/__init__.py`, `install.sh`, publie sur **PyPI** via une porte TestPyPI |
| Gabarits CI (`github.md`) | ⛔ **Aucune ligne Rust** dans le tableau des vérifications obligatoires — pas de `cargo fmt`, `clippy`, `test` |
| `templates/claude-md.md.j2`, configurations markdownlint | ✅ **Utilisables tels quels** |
| `CommandResult` (`cli.md`) | ⚠️ **Le patron oui, l'artefact non** — c'est une `@dataclass(frozen=True, slots=True)` Python ; en Rust c'est une `struct` et un `enum`, transposition triviale |

**Formulation retenue : le corps de doctrine est bon et se prend ; la moitié outillage ne tourne pas sur un projet Cargo.** Aucune des règles adoptées ci-dessous ne dépend de l'outillage.

---

## 3. Architecture et CLI — convergence, et un vrai ajout de périmètre

### 3.1 Ce que le standard confirme

`architecture.md` impose un modèle **un moteur, plusieurs clients minces** :

> *« Every project is an **engine** fronted by thin clients. The engine is the core — it holds the logic, the state, and the authority. »*
> *« Library, CLI, MCP, REST — none reimplements or forks the engine logic. »*

C'est **exactement** le cœur sans E/S de `09` §2 avec son `trait Transport`. Et l'invariant *« Server-Side State Authority »* — *« the engine holds these authoritatively… Keep clients thin »* — **confirme indépendamment l'option B de `10` §1** : connexion implicite, l'état de protocole détenu par le serveur, l'agent qui ne voit jamais l'adaptateur.

Deux conceptions menées séparément ont abouti à la même règle. **C'est le meilleur signal de validation du dossier à ce jour**, et c'est la raison principale de prendre ce standard au sérieux.

### 3.2 Ce qu'il coûte : « The CLI is the complete product »

⚠️ **Conflit réel avec une décision verrouillée.** `cli.md` :

> *« The CLI is the complete product. »* / *« Every engine capability must be reachable from the terminal; features unavailable in the CLI are considered deliberate omissions requiring documented justification. »*
> *« Every CLI command has an MCP tool equivalent »* — sauf exception documentée.

`09` §1 disait *« Serveur MCP seul — l'agent est l'interface. Aucune UI à construire. Le plus court chemin vers l'utile. »* Sous ce standard, Mecabot **doit** exposer toutes ses capacités en ligne de commande. C'est du périmètre en plus, et il faut l'assumer comme tel.

**Ce qui le rend acceptable**, en trois points :

1. Le standard fournit le patron qui le rend bon marché — **Humble Object Commands** : une opération = un objet appelable rendant un `CommandResult`, et les quatre surfaces appellent la même instance. Le CLI n'est pas une seconde implémentation, c'est un adaptateur mince.
2. Ça sert directement le test. `10` §9 développe contre un transport de rejeu ; un CLI le pilote beaucoup plus simplement qu'un client MCP.
3. Ça règle la porte de démonstration de §6 — voir plus bas.

**Verbes d'administration obligatoires :** `version`, `doctor`, `status`, `install`, plus `serve`/`mcp` pour un projet MCP. Mecabot a `serve`. 💡 **`doctor` tombe particulièrement bien** : « l'adaptateur est-il là, le véhicule répond-il » est mot pour mot la ressource `mecabot://status` de `10` §3. Une seule commande de moteur, deux surfaces.

⚠️ **`mecabot log` ne respecte pas la convention nom-verbe** du standard pour un outil à plusieurs noms. À renommer en écrivant le CLI ; sans conséquence puisque rien n'existe encore.

💡 **Le drapeau `--remote <url>`** est exigé par le standard dès que `serve` existe. Il est **la pièce qui manquait** à la décision de propriétaire unique du port (`09` §11.2) et au mode Pi de `09` §11.3 — trois questions, une seule réponse.

### 3.3 Le modèle d'identité, et pourquoi il ne transfère pas

`permissions.md` et `cli.md` posent la même règle :

> *« The engine has no admin path exposed on a client surface. »*
> *« Admin verbs stay on the CLI. Process supervision, install, uninstall, enable, disable, and doctor never appear on MCP, REST, or the library surface. »*
> *« Exposing an admin verb on MCP recreates the superuser MCP surface the identity model forbids. »*

C'est **le même raisonnement que `10` §8** sur l'interdiction des commandes brutes, atteint indépendamment. Mais il faut voir où il s'arrête :

> ⛔ **punt-kit traite « CLI seulement » comme une frontière de privilège. Pour Mecabot, ce n'en est pas une.**
>
> Le raisonnement du standard suppose que l'humain pilote le CLI et que l'agent pilote MCP. **Sur cette machine, l'agent a accès à Bash** — il peut donc invoquer n'importe quelle sous-commande. Toute sûreté qui reposerait sur la séparation CLI/MCP serait fictive ici.

**Ce qui sauve Mecabot est que sa garantie n'a jamais reposé là.** `09` §6 place l'énumération fermée de services **dans l'encodeur**, donc en amont de toute surface : les deux chemins tapent dans la même barrière, et il n'existe aucun chemin de code capable d'émettre `$2F`. Rien à changer au dossier.

À noter que `permissions.md` **sait** que Bash est une échappatoire : il refuse `Bash(bash:*)` (*« permits any command at all »*), `Bash(sed:*)` (*« edits any file in place, bypassing Edit(path) rules »*), `sudo`, `su`, et tout le réseau (`curl`, `wget`, `ssh`, `nc`, `socat`). Un refus sur une sous-commande brute serait donc un mécanisme disponible — **mais c'est un fichier de réglages modifiable, pas une garantie structurelle.** Défense en profondeur, jamais le contrôle principal.

⚠️ **Non adopté :** la liste d'autorisations MCP de `permissions.md` est du tout-ou-rien (*« all Punt Labs plugins or none »*) et tire beadle, biff, dungeon, ethos, lux, quarry, vox, z-spec. Couplage injustifié pour un projet personnel.

---

## 4. Journalisation — le standard le plus directement utile du lot

`logging.md` est celui qui rapporte le plus, et il résout un problème que j'avais signalé comme un conflit.

### 4.1 Il désamorce le piège stdio

> *« For MCP servers, hook subprocesses: **stderr is discarded by the host**; the file handler is the only survivable sink. »*
> *« The file handler is the system of record. stderr is not. »*

`09` §8 utilise le transport **stdio**, où **stdout porte le JSON-RPC**. Toute sortie humaine égarée sur stdout corrompt la session — c'est le faux pas classique d'un serveur MCP. Le standard l'évite en ne journalisant ni sur stdout ni sur stderr, mais **dans un fichier durable**. Le conflit apparent avec le `--json` de `cli.md` tombe de lui-même.

### 4.2 Il fournit le mécanisme qui manquait à la couche 4

Le dossier posait « aucune donnée d'exemplaire ne quitte la machine » (`11` §4) **sans moyen d'application**. Le standard est explicite :

| Jamais journalisé | Sûr à journaliser |
|---|---|
| Contenu transmis, corps de messages | *« Content-derived hashes »* |
| Clés, jetons, identifiants | Compteurs, noms d'objets |
| URL portant des données utilisateur en paramètres | Chemins sous le répertoire personnel |

**Transposition pour Mecabot :** ⛔ le VIN ne se journalise pas — **son empreinte, oui**. As-Built, odomètre, historique d'entretien : jamais. C'est la règle de couche 4, rendue applicable.

S'y ajoutent deux exigences reprises telles quelles :

- **Fichiers de journal en `0600`**, y compris les rotations. Le standard signale que le `RotatingFileHandler` de la bibliothèque standard Python ouvre en `0666 & ~umask` et ne resserre jamais les fichiers existants — le piège est réel et le même en Rust si on ne fixe pas le mode à la création.
- **Protection contre l'injection dans les journaux** : *« Never put an untrusted value into a log message raw — and that includes eager formatting. »* Échapper les caractères de contrôle C0, DEL, `U+0085`, `U+2028`, `U+2029`. 💡 **Particulièrement pertinent ici :** les octets journalisés viennent d'un bus véhicule et sont **non fiables par définition**.

### 4.3 Il a mis au jour le problème du propriétaire unique

> *« RotatingFileHandler is not safe when more than one process writes the file. »*

Mecabot a exactement cette configuration à plusieurs écrivains — `serve`, `log`, et les invocations CLI. Mais **la version dure du problème n'est pas le fichier, c'est le port série**, et le dossier ne la traitait pas. Le patron du standard — *« One process owns the file; others ship records over existing transport »* — se généralise mot pour mot.

**→ D'où la décision de `09` §11.2 :** un seul propriétaire du port par machine, les autres rôles en clients par `--remote`.

Pour le fichier de journal lui-même, le standard offre trois patrons ; retenir l'**écriture atomique en `O_APPEND`** de lignes entières, qui ne demande aucune dépendance et dont le noyau garantit l'atomicité.

**Format retenu :** `timestamp [LEVEL] module.path: message`, sans paires clé-valeur. Rotation 5 Mo × 5. Niveau `stderr` supplémentaire à `WARNING` pour le CLI, jamais pour `serve` en stdio.

---

## 5. Système de fichiers — la seule divergence assumée

`filesystem.md` impose :

> *« All Punt Labs tools store user data under `~/.punt-labs/<tool>/`. No tool creates its own top-level dot-directory. »*

**⛔ Non adopté.** Décision et raisonnement complets en **`09` §7** : répertoires natifs de la plateforme sous le nom `mecabot`, parce que le chemin punt-kit n'a pas de sens sous Windows et parce que `~/.punt-labs/` appartient à l'outillage punt-labs employé dans **estran-studio**.

**Repris en revanche**, et c'est ce qui rend la divergence sans risque :

- Les noms conventionnels de sous-répertoires — `logs/`, `cache/`, `data/`.
- *« Each tool defines a single root constant or function in one central module. All subdirectory paths derive from it. »* **Un seul point de résolution de chemin.** C'est ce qui rend la décision révisable au prix d'une constante.
- *« No automatic migration. The `install` command creates the new directory. »* Un déplacement futur ne lit ni ne déplace l'ancien répertoire.

---

## 6. Processus — le conflit lourd, et un avertissement

`workflow.md`, `git.md`, `github.md` et `pr-review.md` décrivent un processus conçu pour une entreprise dotée d'un opérateur, de flottes d'agents et de Copilot activé au niveau organisation.

### 6.1 Ce qui est pris

| Règle | Citation |
|---|---|
| `make check` avant **chaque** commit | *« A commit that fails `make check` is a broken commit. »* |
| TDD sur les correctifs | *« a failing test reproduces the defect first »* |
| Commits incrémentaux | *« one commit per logical step… Do not accumulate more than 30 minutes of uncommitted changes »* |
| `main` protégé, pas de poussée directe | *« main has branch protection in every repo »* |
| Jamais de réécriture d'historique sur une PR ouverte | *« Never `git rebase` or `git push --force`… A force-push rewrites commit SHAs »* — les commentaires de relecture y sont ancrés |
| Actions épinglées au SHA complet, pas aux étiquettes | `github.md` |
| Documentation dans la même PR que le comportement | `workflow.md` |
| `docs.yml` + markdownlint dans **tous** les dépôts | *« All repos also run a `docs.yml` workflow »* |

### 6.2 Ce qui est refusé

| Règle | Motif du refus |
|---|---|
| *« Beads are mandatory »* (`bd`, `bd sync`) | Machinerie de suivi lourde pour un projet solo |
| Contrats de mission typés T1–T3 (rôles, write-set, budget) | Présuppose des agents délégués et un opérateur distinct |
| Trailers `Mission:` / `Delegation:` | Liés au système `ethos` de l'organisation |
| Rulesets à *« zero bypass actors »* | Interdirait toute poussée sur `main` de son propre projet personnel. Protection de branche : oui ; sans échappatoire : non. |
| `punt release`, `punt doctor` | Inopérants (§2) |

### 6.3 ⛔ L'avertissement : l'adoption partielle est ici pire que l'abstention

> *« There is no human code-review gate — the operator may read the diff in the IDE, but that inspection is non-gating. »*
> *« Review of code is agent-owned end to end »* — Copilot et Bugbot possèdent le diff.

**Cette règle n'est sûre que parce que la flotte de bots remplace l'humain.** Elle présuppose *« Every repository must enable GitHub Copilot as a code reviewer »*, activé à l'échelle de l'organisation.

Sur un projet personnel sans cette flotte, supprimer la porte de relecture humaine ne laisse **aucune** relecture. C'est le seul endroit de tout punt-kit où reprendre le standard à moitié crée un risque réel, et il faut le nommer : **soit la flotte de relecture est en place, soit la relecture humaine reste la porte.** Pas d'entre-deux.

> ### ✅ Tranché le 2026-08-26 — la flotte existe
>
> L'alternative ci-dessus est levée par le premier terme : **la flotte est en place, donc la règle devient adoptable.** Elle compte **trois relecteurs de code** et **une porte de dépendances** — ce ne sont pas la même chose, et les confondre gonfle le décompte :
>
> | Outil | Angle | Configuration | Actif |
> |---|---|---|---|
> | Gemini Code Review | Relecture générale de code | Déjà en place avant cette session | ✅ |
> | CodeRabbit | Relecture générale de code, plus les invariants du projet | `.coderabbit.yaml`, `path_instructions` par type de fichier | ✅ application installée le 2026-08-26 |
> | Claude Code Review | Relecture de code ciblée sur les huit interdits par construction | `.github/workflows/claude-review.yml` | ✅ secret posé le 2026-08-26 |
> | `cargo-deny` | **Pas un relecteur de code** — licences, avis de sécurité, provenance des sources | `deny.toml` | ⏸️ en veille jusqu'au premier `Cargo.toml` |
>
> **Le décompte qui vaut, à la date du 2026-08-26 :** **trois relecteurs de code actifs** sur une *pull request* interne, **deux** sur une *pull request* issue d'un fork (voir ci-dessous), et **zéro** porte de dépendances tant qu'il n'y a pas de `Cargo.toml`.
>
> **La colonne « Actif » est la seule qui compte.** Un fichier de configuration présent au dépôt ne prouve rien : `.coderabbit.yaml` sans l'application installée et `claude-review.yml` sans le secret `ANTHROPIC_API_KEY` sont des relecteurs qui ne relisent pas. Ces deux activations sont hors du dépôt — elles ne se lisent pas dans le diff, donc elles se vérifient sur une vraie *pull request*, pas en relisant les fichiers.
>
> **Ce qui rend la substitution honnête, et non un habillage :** les trois relecteurs de code sont indépendants — modèles différents, éditeurs différents, points de défaillance différents. Une flotte d'un seul agent n'aurait pas remplacé la porte humaine, elle l'aurait supprimée en la nommant autrement.
>
> ⚠️ **Ce que la flotte ne couvre pas, et qu'il faut savoir :** les *pull requests* issues de forks ne sont pas relues par Claude. Le déclencheur est `pull_request` et non `pull_request_target`, précisément pour que le secret d'API ne soit jamais exposé à du code de fork. Sur un dépôt public, c'est le bon sens de la marche — mais ça veut dire qu'une contribution externe est relue par **deux** relecteurs de code au lieu de trois.
>
> ⛔ **Corollaire à ne pas retourner.** Cette règle n'est citable que tant que la flotte tourne réellement. Si les relecteurs sont désactivés, retirés du dépôt ou laissés en échec silencieux, **la porte humaine redevient obligatoire** — l'ordre logique va de la flotte vers la règle, jamais l'inverse. **Le plancher est de trois relecteurs de code indépendants sur une *pull request* interne**, ce qui est exactement l'état actuel : il n'y a aucune marge.
>
> **L'échec silencieux est le mode de défaillance à surveiller**, plus que la désactivation volontaire. Une clé d'API expirée, un quota dépassé, une application désinstallée par accident : la *pull request* fusionne, la vérification obligatoire `docs` est verte, et personne n'a relu. Rien dans l'outillage ne signale l'absence d'un relecteur — c'est une lacune connue de ce montage, pas un oubli de rédaction.
>
> ⛔ **Un cas de cet échec silencieux est déjà acquis, pas hypothétique : l'exclusion des forks.** Sur une *pull request* de fork, le job `claude-review` n'échoue pas — il est *skipped*, et **un job sauté par un `if:` de niveau job compte comme réussi** dans les vérifications de protection de branche. Le jour où `claude-review` deviendrait une vérification obligatoire, la protection afficherait donc vert sur précisément les *pull requests* que Claude n'a pas relues. C'est le seul mode de défaillance du montage qui soit déjà constitué plutôt que redouté ; il ne se corrige pas en rendant la vérification obligatoire, mais en sachant qu'une contribution externe se lit à deux relecteurs.

### 6.4 Deux constats plus favorables

✅ **La porte la plus lourde du standard est déjà franchie.** `workflow.md` exige qu'une mission de conception soit **ratifiée par l'opérateur avant implémentation**, matérialisée par des entrées ADR dans `DESIGN.md`. C'est littéralement ce que sont `09` et `11` : des décisions de conception ratifiées, avec alternatives et raisonnement. Le découpage en fichiers diffère, la fonction est identique.

⚠️ **Mais la porte de démonstration est physiquement coûteuse ici.** Le standard exige que *« the feature must be driven through its real entry point, verified against expected outcomes, and demonstrated to the operator before the PR opens »*. **Le vrai point d'entrée de Mecabot est un camion** — contact mis, mainteneur de batterie branché, du temps. Pour `probe_dids`, c'est une heure de stationnement par démonstration.

> 💡 **Conséquence qui remonte la valeur de l'item 4bis.** Le transport de rejeu de `10` §9 n'est plus un confort : **c'est ce qui rend cette porte payable.** Une trace enregistrée permet de la franchir au bureau. Le seul blocage restant du dossier est donc aussi la clé de l'adoption du processus — raison de plus de le lever tôt.

---

## 7. Ce qui est adoptable aujourd'hui, sans écrire de code

Le constat de séquencement le plus utile du dossier sur ce sujet : **Mecabot est aujourd'hui littéralement un projet de documentation**, et c'est un type de projet que punt-kit gère pleinement. Toute cette colonne s'applique sans toucher à la contrainte « pas de code » :

| Action | Origine | État |
|---|---|---|
| `git init`, branche `main`, protection de branche | `git.md`, `github.md` | ✅ fait le 2026-08-26 |
| `docs.yml` avec markdownlint-cli2 sur les seize fichiers Markdown suivis | `github.md` — exigé de **tous** les dépôts | ✅ fait |
| `.markdownlint.jsonc` + `.markdownlint-cli2.jsonc` | `github.md`, gabarits fournis | ✅ fait |
| Actions épinglées au SHA complet, délais de tâche | `github.md` | ✅ fait, cinq SHA résolus par `gh api` |
| `CHANGELOG.md` avec une section `[Unreleased]` | `release-requirements.md` | ✅ fait |
| Secret scanning, push protection, alertes Dependabot | `github.md` | ✅ fait |

> **Ce que le passage au lint a réellement coûté, mesuré plutôt que supposé.** L'avertissement écrit ici était que des documents **en français** en prose longue feraient probablement dérailler quelques règles. Résultat sur les seize fichiers : **554 signalements, dont 534 pour une seule règle cosmétique** (`MD060`, remplissage des barres verticales de tableau), désactivée. Des vingt restants, **aucun ne portait sur la langue ou la longueur des lignes** — c'étaient de vrais petits défauts de structure : cinq blocs de code sans langage, deux sauts de niveau de titre, une cellule de tableau manquante, une liste sans ligne vide, une ponctuation en fin de titre.
>
> **La leçon inverse de celle attendue :** le lint n'a pas puni le français, il a trouvé une ligne de tableau qui perdait sa donnée (`06` §1) — un défaut qu'aucune relecture humaine n'avait attrapé en une semaine. Les onze occurrences de `forscan.org` signalées n'étaient pas des fautes de casse mais des noms de domaine sans mise en forme ; les mettre entre chevrons inverses est la correction juste, et garde la règle capable d'attraper un vrai `forscan` en minuscules écrit à la place de « FORScan » dans la prose.
>
> ⚠️ **Une seule règle a été désactivée pour convenance**, `MD060`, et le motif est écrit dans `.markdownlint.jsonc` : elle déduit un style de la première ligne de séparation rencontrée puis l'exige partout, pour un rendu GitHub identique. `MD056` (nombre de colonnes), qui attrape de vrais défauts, reste active.

---

## 8. Ce que punt-kit ne couvre pas — où le dossier est devant

`agent-engineering.md` ne contient pas ce que son nom suggère : ce sont des règles d'ingénierie générales numérotées. **Aucune mention de budget de tokens, de sortie structurée, de déterminisme, ni de conception de serveur MCP.**

**Donc `04` (budget de tokens) et `10` (conception du serveur MCP) n'ont aucun équivalent en amont.** Ce sont les deux documents que Mecabot pourrait contribuer — décision de William, pas conséquence technique (§1).

Quelques règles générales de ce standard méritent malgré tout d'être retenues, parce qu'elles recoupent le dossier :

| Règle | Recoupement |
|---|---|
| *« Errors should fail loudly and specifically — not silently, not generically »* | ✅ La taxonomie d'erreurs de `10` §4, mot pour mot |
| *« Logging, metrics, and error handling are part of the feature, not afterthoughts »* | ✅ `10` §10 place la taxonomie en étape 2, avant tout outil |
| *« Don't claim a fix works until you've run it »* | ⚠️ Sain rappel : ce dossier entier n'est pas vérifié par le matériel |
| *« The simplest solution that meets requirements is almost always the right one »* | ⚠️ Pointe contre l'abstraction en cinq couches de `11`. À lire comme un renfort de **la règle des deux instances** (`11` §8), pas comme une objection |
| *« Names should describe intent, not implementation »* | ✅ Nommage canonique anglais `snake_case` (`09` §1) |

---

## 9. Décisions prises le 2026-08-26

| Décision | Choix | Où c'est consigné |
|---|---|---|
| Propriétaire du port série | **Un seul processus par machine ; les autres rôles sont ses clients par `--remote`** | `09` §11.2, `10` §2 et §4 |
| Emplacement des données d'exécution | **Répertoires natifs de la plateforme sous le nom `mecabot`** — pas `~/.punt-labs/` | `09` §7 |
| Mode d'emploi de punt-kit | **Emprunt sélectif et cité, révision épinglée** — pas d'adoption, pas de contribution amont par défaut | §1 de ce document |
| Visibilité du dépôt | **Public, tel quel** — `wquintal/mecabot`, dossier compris | §9.1 ci-dessous |
| Relecture du diff | **Trois relecteurs de code, pas de porte humaine** — Gemini, CodeRabbit, Claude ; plus `cargo-deny` comme porte de dépendances, qui n'est pas un relecteur | §6.3, encadré du 2026-08-26 |
| Application de la règle « aucune dépendance GPL » | **Liste d'autorisation en CI**, la GPL exclue par omission | `deny.toml`, `rust.yml` tâche `deny` |

### 9.1 La publication, et la déclaration qu'elle emporte

Le dépôt est **public, dossier compris**, sans retrait ni rédaction. Ce choix a été fait en connaissance d'un point qui mérite d'être écrit ici plutôt que laissé implicite : **`02` décrit un véhicule dont le système d'après-traitement a été retiré (*deleted*), et le fait publiquement, sous un compte nominatif.** `02` §8 cite la LCPE 1999. Une déclaration publique auto-attribuée n'a pas le même statut qu'une note privée.

La décision est celle du propriétaire du dépôt, elle est prise, et ce document ne la rouvre pas. Elle est consignée parce qu'un dossier qui tait ses propres arbitrages est moins fiable qu'un dossier qui les nomme.

### 9.2 Une dérogation assumée à `github.md`

⚠️ Le standard exige *« Every repository must enable GitHub Copilot as a code reviewer »*. **Mecabot ne l'active pas.** La flotte retenue est Gemini + CodeRabbit + Claude.

**Pourquoi la dérogation est tenable :** la fonction que le standard visait — *« Review of code is agent-owned end to end »*, aucun diff fusionné sans relecture par un agent — est remplie, et par trois relecteurs indépendants plutôt qu'un. Le standard nomme un outil ; c'est la propriété qui compte, et elle est satisfaite plus largement que le minimum exigé.

**Comment elle est contenue :** la dérogation porte sur l'identité des relecteurs, pas sur l'existence de la porte. Le nombre plancher est de trois relecteurs de code indépendants — en descendre en dessous rouvre §6.3.

---

## 10. Ce que ce document ne décide pas

- **Un `rust.md` amont.** Le fossé de §2 se contourne localement. Contribuer serait du travail non rémunéré pour l'organisation d'un tiers : décision de William.
- **La liste exacte des sous-commandes CLI.** §3.2 impose la couverture complète et les cinq verbes d'administration ; le découpage se fait en écrivant le CLI.
- **Le contenu du `Makefile`.** `make check` est adopté comme porte ; ce qu'il exécute dépend de l'outillage Rust retenu (`cargo fmt --check`, `clippy -D warnings`, `test`), à figer à l'étape 1.
- **La licence.** Toujours non décidée (`09` §1). Rien dans punt-kit ne la contraint ; la règle « aucune dépendance GPL » reste la seule en vigueur.
- ~~**La relecture.**~~ ✅ **Tranché le 2026-08-26** : la flotte est en place, la porte humaine tombe (§6.3). Ce qui reste ouvert est plus étroit — au bout de quelques *pull requests* réelles, savoir si trois relecteurs généralistes produisent assez de signal utile ou surtout du bruit. Ça ne s'évalue pas à l'avance.
