# Mecabot

**Serveur MCP de diagnostic automobile OBD-II / UDS, générique et en lecture seule.**

Mecabot expose le diagnostic d'un véhicule à un agent IA via le
[Model Context Protocol](https://modelcontextprotocol.io) : lire les codes
défaut de tous les modules, suivre des paramètres temps réel, récupérer les
données figées, corréler tout ça dans le temps. Multiplateforme — macOS, Linux,
Windows — parce que la motivation d'origine est qu'un très bon outil du domaine,
FORScan, ne tourne que sous Windows.

**Écriture exclue par construction.** Mecabot ne programme pas de module, ne
lance pas de routine, ne force pas de régénération. Ce n'est pas une option
désactivée par défaut : les services d'écriture n'existent pas dans l'encodeur.

---

## ⚠️ État du projet : préconception

**Aucune ligne de code applicatif n'est écrite.** Ce dépôt contient
aujourd'hui un dossier de préconception et sa chaîne d'intégration continue.

Ce n'est pas un accident de calendrier, c'est la méthode : le domaine touche à
du matériel qui peut abîmer un véhicule, et une partie du travail consiste à
établir ce qui est vérifié avant de construire dessus. Le dossier distingue
systématiquement quatre niveaux de confiance — `[VÉRIFIÉ]`, `[COMMUNAUTÉ]`,
`[NON VÉRIFIÉ]`, `[INTROUVABLE]` — et cette distinction survit dans le code
sous forme de champ `provenance` sur chaque donnée véhicule.

**Le dossier est en français.** Le code, quand il viendra, sera en anglais.

---

## Le dossier

Il se lit dans cet ordre. `00` suffit pour comprendre le projet ; les autres
documents sont là pour qui veut vérifier un raisonnement.

| # | Document | Ce qu'on y trouve |
|---|----------|-------------------|
| [`00`](docs/00-synthese.md) | Synthèse | Le projet en un document. **Commencer ici.** |
| [`01`](docs/01-acces-donnees-et-securite.md) | Accès aux données et sécurité | Ce qui est lisible, ce qui est verrouillé, et pourquoi |
| [`02`](docs/02-profil-vehicule.md) | Profil véhicule | Le véhicule de référence n°1, comme cas d'étude |
| [`03`](docs/03-architecture-mcp.md) | Architecture MCP | Première passe sur la forme du serveur |
| [`04`](docs/04-budget-tokens.md) | Budget de contexte | Pourquoi un outil de diagnostic doit compter ses jetons |
| [`05`](docs/05-materiel-et-stack.md) | Matériel et pile technique | Adaptateurs, puces, ce qui tient la route |
| [`06`](docs/06-critique-du-document-initial.md) | Critique du document initial | Ce que la première version avait faux |
| [`07`](docs/07-verifications-a-faire.md) | Vérifications à faire | La liste ouverte — ce qui reste à confirmer sur un vrai véhicule |
| [`08`](docs/08-architecture-forscan-bridge.md) | Pont FORScan *(écarté)* | Une direction explorée puis abandonnée, conservée pour la trace |
| [`09`](docs/09-architecture-cross-platform.md) | **Architecture** | Le document de référence : plateformes, transports, paliers, déploiement |
| [`10`](docs/10-conception-serveur-mcp.md) | Conception du serveur MCP | Outils, état, erreurs, feuille de route de mise en œuvre |
| [`11`](docs/11-modele-de-connaissance-vehicule.md) | Modèle de connaissance véhicule | Les cinq couches : comment l'app reste générique |
| [`12`](docs/12-standards-de-developpement.md) | Standards de développement | Ce qui est emprunté à punt-kit, et ce qui ne se transpose pas |

---

## Les interdits par construction

Ce sont des décisions verrouillées, pas des réglages. Elles sont vérifiées à la
relecture de chaque *pull request* par la flotte d'agents, et pour partie en CI.

- **Aucune commande brute.** Pas de `send_command`, pas de passe-plat AT, sur
  aucune surface — ni MCP, ni CLI, ni bibliothèque, même en développement. Un
  seul chemin de ce genre annule toutes les garanties ci-dessous. Le débogage
  se fait au terminal série, hors du serveur.
- **Énumération fermée de services UDS**, appliquée dans l'encodeur :
  `$22` (lire par identifiant), `$19` (lire les DTC), `$3E` (maintien de
  session), `$10` (contrôle de session), plus les modes OBD-II normalisés.
  Rien d'autre n'est encodable.
- **Jamais de balayage d'identifiants de service.** Balayer des DID avec `$22`
  est sans danger. Balayer des services ne l'est pas : `$11` réinitialise un
  module, `$2F` actionne des sorties physiques, `$28` peut couper la
  communication d'un module, `$31` lance des routines parfois destructrices.
- **La découverte de DID est explicitement autorisée par profil**
  (`discovery_allowed`, faux par défaut) et sous préconditions : véhicule
  immobile, stationné, frein serré, contact mis, mainteneur de batterie
  branché. Jamais en roulant.
- **Aucune donnée d'exemplaire ne quitte la machine.** VIN, As-Built,
  CALID/CVN, odomètre, historique d'entretien. Une empreinte est journalisable,
  la valeur en clair non. Une trace série contient le VIN dès qu'elle inclut un
  Mode `$09` ou un `$22 F190` — c'est pourquoi `traces/` est ignoré par git.
- **Aucune dépendance GPL** tant que la diffusion n'est pas décidée. Appliqué
  par [`deny.toml`](deny.toml), en liste d'autorisation : la GPL est exclue par
  omission, pas par une règle qu'on pourrait oublier de tenir à jour.
- **Aucun fait sur un véhicule particulier écrit dans le code.** La
  connaissance véhicule vit en données. Une constante propre à une marque dans
  le code est un défaut, pas un raccourci.

### La frontière avec FORScan

FORScan reste le meilleur outil de sa catégorie et Mecabot ne cherche pas à le
remplacer là où il est irremplaçable — notamment tout ce qui écrit. S'en servir
sur son propre véhicule pour recouper ses propres décodages est un usage
normal. **Extraire ou redistribuer sa base de PID ne l'est pas.** La frontière
est tenue par le schéma de données : une donnée marquée
`provenance: proprietary_crosscheck` est exclue de tout export de profil.

---

## Relecture

Il n'y a pas de porte de relecture humaine sur le diff. Elle est remplacée par
une flotte d'agents d'angles différents, ce qui n'est défendable que parce que
la flotte existe réellement :

| Relecteur | Rôle |
|-----------|------|
| Gemini Code Review | Relecture générale |
| CodeRabbit | Relecture générale, [configurée](.coderabbit.yaml) avec les invariants du projet |
| Claude Code Review | [Relecture ciblée](.github/workflows/claude-review.yml) sur les huit invariants ci-dessus |
| `cargo-deny` | Licences, avis de sécurité, provenance des sources |
| `markdownlint-cli2` | Le dossier lui-même |

Les *pull requests* issues de forks ne sont pas relues par Claude : le
déclencheur est `pull_request` et non `pull_request_target`, donc le secret
d'API n'est jamais exposé à du code de fork. Elles restent couvertes par les
autres relecteurs.

---

## Licence

**Aucune licence n'est accordée pour l'instant.** Le dépôt est public pour être
lisible et critiquable, pas pour être réutilisé : en l'absence de fichier
`LICENSE`, tous droits sont réservés.

Le choix de licence est une décision ouverte, liée à celle de la diffusion
(`09` §1). C'est aussi la raison pour laquelle aucune dépendance GPL n'entre
avant que cette décision soit prise — elle contraindrait le choix.

## Avertissement

Diagnostiquer un véhicule avec un outil qu'on écrit soi-même engage sa propre
responsabilité. Mecabot lit ; il ne dit pas si un véhicule est sûr à conduire.
Aucune garantie, d'aucune sorte.
