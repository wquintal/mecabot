# Mecabot — contexte pour l'agent

## Ce qu'est ce dépôt

Un serveur MCP de diagnostic automobile **OBD-II / UDS générique, en lecture
seule**, en Rust, multiplateforme.

**État : préconception.** `docs/` contient quatorze documents de préconception en
français. **Aucun code applicatif n'existe encore, et il ne s'en écrit pas sans
demande explicite.** Configurer le dépôt et la CI est autorisé ; écrire
l'application ne l'est pas tant que ce n'est pas demandé.

## Langue

**Le dossier `docs/` est en français, et le reste.** Les nouveaux documents, les
sections ajoutées et les réponses dans ce dépôt sont en français. Le code, les
identifiants et les messages de commit, quand ils viendront, seront en anglais.

## Où lire quoi

Ne pas relire les quatorze documents pour une question ponctuelle. Chacun a un
rôle :

| Doc | Autorité sur |
|-----|--------------|
| `docs/00-synthese.md` | Le projet en un document, et l'index du reste |
| `docs/09-architecture-cross-platform.md` | **L'architecture de référence.** §1 = table des décisions verrouillées |
| `docs/10-conception-serveur-mcp.md` | Outils MCP, état, taxonomie d'erreurs, feuille de route |
| `docs/11-modele-de-connaissance-vehicule.md` | Les cinq couches de connaissance véhicule, `provenance`, `validation_method` |
| `docs/12-standards-de-developpement.md` | Ce qui est emprunté à punt-kit et ce qui ne se transpose pas |
| `docs/07-verifications-a-faire.md` | La liste ouverte de ce qui reste à confirmer sur un vrai véhicule |
| `docs/13-protocole-seance-terminal.md` | **La séance au terminal série (item 4bis)** : commandes AT/ST, format de trace, anonymisation. ⛔ Ce n'est pas la spec d'un passe-plat — c'est une séance humaine, hors du serveur |
| `docs/08-architecture-forscan-bridge.md` | ⚠️ **Direction écartée**, conservée pour la trace. Ne pas s'en servir comme référence |

**`docs/09` §1 est la table des décisions verrouillées.** Une décision qui y
figure ne se rouvre pas sans que l'utilisateur le demande.

## Étiquettes de confiance

Le dossier distingue quatre niveaux, et cette distinction est structurante — pas
décorative :

- `[VÉRIFIÉ]` — confirmé sur une source primaire ou observé directement
- `[COMMUNAUTÉ]` — rapporté de façon convergente par des forums, non confirmé
- `[NON VÉRIFIÉ]` — plausible, jamais confirmé
- `[INTROUVABLE]` — cherché sans résultat

**Ne jamais promouvoir une affirmation d'un niveau au suivant sans nouvelle
preuve**, et ne jamais écrire une affirmation nue là où le dossier en attend une
étiquette. C'est la même distinction que le champ `provenance` du modèle de
données.

## Les interdits par construction

Ce ne sont pas des préférences de style. Un patch qui en franchit un est à
refuser, pas à discuter.

1. ⛔ **Aucune commande brute.** Pas de `send_command`, `send_raw`, ni passe-plat
   AT — sur **aucune** surface : ni MCP, ni CLI, ni bibliothèque, pas même
   derrière un drapeau de développement. Un seul chemin de ce genre annule
   l'énumération fermée. Le débogage se fait au terminal série, hors du serveur.
   (`docs/10` §8)
2. ⛔ **Énumération fermée de services UDS, appliquée dans l'encodeur** et non
   dans une surface : `$22`, `$19`, `$3E`, `$10`, plus les modes OBD-II
   normalisés. Rien d'autre n'est encodable. (`docs/09` §6)
3. ⛔ **Jamais de balayage d'identifiants de service.** Balayer des DID avec
   `$22` est sans danger ; balayer des services ne l'est pas — `$11`
   réinitialise, `$2F` actionne des sorties physiques, `$28` peut couper la
   communication d'un module, `$31` lance des routines parfois destructrices.
4. ⛔ **`probe_dids` exige `discovery_allowed: true`** dans le profil de marque,
   faux par défaut. Préconditions : véhicule immobile, stationné, frein serré,
   contact mis, mainteneur de batterie branché. Jamais en roulant. Pas compilé
   dans le rôle enregistreur embarqué.
5. ⛔ **Aucune donnée de couche 4 ne quitte la machine ni n'entre dans un
   journal** : VIN, As-Built, CALID/CVN, odomètre, historique d'entretien. Une
   empreinte dérivée est acceptable, la valeur en clair non. **Une trace série
   contient le VIN dès qu'elle inclut un Mode `$09` ou un `$22 F190`** — donc
   `traces/` est ignoré par git, et une trace se verse à la main après relecture
   et anonymisation, **jamais par un `git add .`**. (`docs/11` §4, `docs/12` §4.2)
6. ⛔ **Aucune dépendance GPL** tant que la diffusion n'est pas décidée.
   Appliqué par `deny.toml` en liste d'autorisation. Ne pas y ajouter de licence
   ni d'exception sans que ce soit demandé et daté.
7. ⛔ **La régénération forcée de DPF reste hors de Mecabot, définitivement.**
   550–700 °C ; elle reste dans FORScan sous contrôle humain direct, et jamais
   si une dilution de l'huile par le carburant est suspectée.
8. ⚠️ **Aucun fait sur un véhicule particulier écrit dans le code.** La
   connaissance véhicule vit en données. Une constante Ford dans le code est un
   défaut. (`docs/11` §1)
9. ⚠️ **Un seul processus détient le port série par machine.** Les autres rôles
   sont ses clients via `--remote`. (`docs/09` §11.2)
10. ⚠️ **Les erreurs sont nommées et distinctes.** Un échec remonté en « erreur »
    générique fait raisonner l'agent faux. En particulier, `7F .. 31`
    (*requestOutOfRange*) est de l'**information** — le DID n'existe pas sur ce
    module — et non une erreur. (`docs/10` §4)

### La frontière avec FORScan

Se servir de FORScan sur son propre véhicule pour recouper ses propres décodages
est un usage normal. **Extraire ou redistribuer sa base de PID ne l'est pas.**
La frontière est tenue par le schéma : `provenance: proprietary_crosscheck` ⇒
exclu de tout export de profil.

### Câblage

Alimenter quoi que ce soit depuis la broche 16 du J1962 (positif batterie non
commuté) impose un fusible près de la source.

## Standards de développement

`docs/12` porte le cadre complet. L'essentiel :

- Les standards de [punt-kit](https://github.com/punt-labs/punt-kit) sont
  empruntés **sélectivement**, au révision épinglée `278e9cc`. Ils appartiennent
  au CEO de l'employeur de l'utilisateur, pas à l'utilisateur.
- **Aucun fichier de punt-kit ne mentionne Rust.** Formulation retenue : *les
  standards se transposent, l'outillage est inopérant*. Ne pas invoquer
  `punt doctor`, `punt release` ou les gabarits CI Python.
- ⛔ **`~/.punt-labs/` est explicitement rejeté** comme racine de données. Les
  données vivent dans les répertoires natifs de la plateforme, sous `mecabot`
  (`docs/09` §7).
- Les actions GitHub sont **épinglées au SHA complet**, jamais à une étiquette.
  Ne pas inventer un SHA : le résoudre avec `gh api`.
- **GitHub Actions ne supporte pas les ancres YAML.** Dupliquer plutôt
  qu'ancrer.
- ⛔ **La règle punt-kit « pas de porte de relecture humaine » n'est PAS adoptée.**
  Elle supposait une flotte de trois relecteurs agents ; la mesure du 2026-08-27
  l'a renversée. **La flotte est CodeRabbit seul**, et **tout diff se relit à la
  main avant fusion** — CodeRabbit vient en plus, pas à la place. Ne pas citer
  cette règle punt-kit pour se passer d'une relecture (`docs/12` §6.3).
- **Un relecteur ne compte que s'il est observé sur une vraie *pull request*.**
  Un fichier de configuration au dépôt ne prouve rien : l'activation vit hors du
  dépôt. Gemini était tenu pour acquis parce qu'il est installé sur d'autres
  dépôts — il n'a jamais rien relu ici, et **rien ne l'a signalé**. Un relecteur
  absent ressemble à un relecteur satisfait ; c'est le mode de défaillance à
  surveiller.
- `cargo-deny` est une **porte de dépendances**, pas un relecteur de code, et
  dort derrière la tâche `guard` jusqu'au premier `Cargo.toml`.

## Dérogation assumée

Le standard `github.md` de punt-kit exige GitHub Copilot comme relecteur requis.
**Mecabot ne l'utilise pas** : le relecteur agent est **CodeRabbit**, choix de
l'utilisateur. La fonction que le standard visait — qu'aucun diff ne fusionne
sans relecture par un agent — est remplie, et la porte humaine est **conservée**
par-dessus au lieu d'être supprimée. Consigné dans `docs/12` §6.3 et §9.2.
