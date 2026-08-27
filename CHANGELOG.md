# Changelog

Toutes les modifications notables de ce projet sont consignées ici.

Le format suit [Keep a Changelog](https://keepachangelog.com/fr/1.1.0/) et le
projet suivra le [versionnage sémantique](https://semver.org/lang/fr/) à partir
de sa première version publiée.

**Aucune version n'est publiée.** Le projet est en préconception : `docs/`
contient le dossier, aucun code applicatif n'existe. La section
`[Unreleased]` ci-dessous est donc l'intégralité de l'historique.

## [Unreleased]

### Added

- Dossier de préconception en treize documents (`docs/00` à `docs/12`) :
  synthèse, accès aux données et sûreté, profil véhicule, architecture MCP,
  budget de contexte, matériel, critique du document initial, vérifications à
  faire, architecture multiplateforme, conception du serveur MCP, modèle de
  connaissance véhicule sur cinq couches, standards de développement.
- Chaîne d'intégration continue : `docs.yml` (markdownlint-cli2, exigé de tout
  dépôt par le standard `github.md` de punt-kit) et `rust.yml` (fmt, clippy,
  test, cargo-deny), ce dernier mis en veille par une tâche `guard` qui teste la
  présence de `Cargo.toml`. Les filtres `paths` seuls n'y suffisaient pas :
  `deny.toml` et `rust.yml` y figurent, donc le commit initial l'a déclenché.
- `deny.toml` — liste d'autorisation de licences qui applique en CI la règle
  « aucune dépendance GPL » de `docs/09` §1, et interdictions ciblées sur
  `ecu-diagnostics` (GPL-3.0) et `socketcan` (Linux uniquement).
- Flotte de relecture par agents : Gemini Code Review, CodeRabbit
  (`.coderabbit.yaml`), Claude Code Review (`claude-review.yml`, huit
  invariants du projet).

### Decided

Les décisions structurantes sont datées dans les documents qu'elles touchent.
Les principales, dans l'ordre où elles ont été prises :

- L'application est **générique et multi-marques** : aucun fait sur un véhicule
  particulier n'est écrit dans le code, la connaissance véhicule vit en données
  sur cinq couches (`docs/11`). Le Ford Transit 350HD 2016 est le profil de
  référence n°1, pas le périmètre.
- **Rust, transport stdio, sur OBDLink EX**, multiplateforme macOS / Linux /
  Windows (`docs/09` §1 et §2).
- **Lecture seule, énumération fermée de services UDS dans l'encodeur.** Aucune
  commande brute sur aucune surface (`docs/10` §8).
- **Un seul processus détient le port série par machine** ; les autres rôles
  sont ses clients via `--remote` (`docs/09` §11.2).
- Les données d'exécution vivent dans les **répertoires natifs de la
  plateforme** sous le nom `mecabot`, et non sous `~/.punt-labs/`
  (`docs/09` §7).
- Emprunt **sélectif** aux standards punt-kit : les standards se transposent,
  l'outillage est inopérant en Rust (`docs/12`).
- Dépôt **public**, sans licence accordée pour l'instant.

[Unreleased]: https://github.com/wquintal/mecabot/commits/main
