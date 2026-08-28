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

- Dossier de préconception en quatorze documents (`docs/00` à `docs/13`) :
  synthèse, accès aux données et sûreté, profil véhicule, architecture MCP,
  budget de contexte, matériel, critique du document initial, vérifications à
  faire, architecture multiplateforme, conception du serveur MCP, modèle de
  connaissance véhicule sur cinq couches, standards de développement, protocole
  de la séance au terminal série.
- `docs/13-protocole-seance-terminal.md` — l'exécution de l'item 4bis de
  `docs/07`, le seul préalable réel à l'écriture de code : huit blocs de
  commandes AT/ST dans l'ordre, fiche de relevé de vingt-six mesures, format de
  trace horodatée que consommera le transport de rejeu de `docs/10` §9, et
  procédure d'anonymisation obligatoire avant tout versement — une trace
  contient le VIN dès le premier contact, `$22 F190` étant la meilleure sonde
  qui existe.
- Chaîne d'intégration continue : `docs.yml` (markdownlint-cli2, exigé de tout
  dépôt par le standard `github.md` de punt-kit) et `rust.yml` (fmt, clippy,
  test, cargo-deny), ce dernier mis en veille par une tâche `guard` qui teste la
  présence de `Cargo.toml`. Les filtres `paths` seuls n'y suffisaient pas :
  `deny.toml` et `rust.yml` y figurent, donc le commit initial l'a déclenché.
- `deny.toml` — liste d'autorisation de licences qui exprime la règle « aucune
  dépendance GPL » de `docs/09` §1, et interdictions ciblées sur
  `ecu-diagnostics` (GPL-3.0) et `socketcan` (Linux uniquement). **Porte écrite
  mais pas encore exercée** : `cargo-deny` dort derrière `guard` tant qu'il n'y
  a pas de `Cargo.toml`.
- Relecture par agent : **CodeRabbit** (`.coderabbit.yaml`), configuré avec les
  invariants du projet par `path_instructions`. Il **s'ajoute** à la lecture
  humaine du diff au lieu de la remplacer.

### Changed

- ⛔ **`docs/05` §3.1 corrigé sur source primaire** : la commutation MS-CAN n'est
  ni un relais ni une commande `ATSW`/`ATMSCAN` — ces commandes n'existent pas.
  C'est un changement de préréglage de protocole (`STP 53`), l'adaptateur n'ayant
  qu'un seul périphérique CAN remappé par logiciel, donc **un seul bus actif à la
  fois**. Source : *OBDLink Family Reference and Programming Manual*, rév. F
  (2025-08-29), §8.6.
- ⚠️ **`docs/05` §1.2 : la puce de l'OBDLink EX est à revérifier.** Le manuel
  donne le STN2120 pour un circuit interpréteur vendu seul et **obsolète** ;
  l'EX actuel est un **STN2232**. Le dossier écrit STN2120 partout. La correction
  attend le relevé de `STI` sur l'appareil (`docs/13` §0.1).
- `docs/09` §3 : la délégation d'ISO-TP au firmware gagne ses **quatre
  conditions** (préréglage ISO 15765, `ATCAF1`, `ATAL`, paires de contrôle de
  flux) — la configuration d'usine ne les réunit **pas ensemble**, même si deux
  d'entre elles y sont actives. **La phrase
  d'origine porte désormais la condition elle-même** : l'encadré ne suffisait
  pas, un lecteur qui ne lisait que la ligne 78 en tirait le contraire.
- ⚠️ **`docs/13` §0.5 : réserve sur la portée de `ATNL`, relevée en relecture.**
  La rédaction initiale concluait qu'une réponse UDS longue est *refusée* sans
  `ATAL` — c'était une **inférence présentée comme un fait**, et l'exemple `0902`
  du manuel la contredit (20 octets réassemblés en trois trames, sans `ATAL`).
  Deux lectures restent ouvertes, le bloc C les tranche par une mesure à deux
  points, et le protocole pose `ATAL` avant toute requête longue en attendant.
- 🐛 **`docs/13` blocs B et C : ordre de `ATAL` corrigé.** `ATAL` n'était posé
  qu'au bloc C, alors que le bloc B tire `0902` et `22F190`, tous deux
  multi-trames ; et le bloc C sortait sous `ATNL`, laissant les blocs D et F
  mesurer sous un réglage différent de celui du premier contact. Un échec au
  premier contact serait arrivé au pire moment pour l'attribuer à la bonne cause.
- `docs/05` §1.2 : **taille de message ≠ taille de tampon.** Les 4 ko publiés
  bornent le réassemblage ISO-TP ; la capacité du tampon interne reste
  `[INTROUVABLE]`, et la phrase d'origine du document était donc exacte.
- `docs/07` §4bis : les consignes résiduelles renvoyant à `ATST` passent à
  `STPTO`, et la commutation MS-CAN n'est plus listée comme inconnue — seul son
  délai l'est.
- `docs/10` §1 : la **nécessité** de réinitialiser après une commutation de bus
  n'est plus une hypothèse mais une conséquence de l'architecture de la puce ;
  seul le **délai** reste à mesurer. §4 gagne l'**étage adaptateur** de la
  taxonomie d'erreurs, dont `STOPPED` et `LV RESET` — preuves matérielles de deux
  invariants que le dossier posait par prudence.

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
- **La relecture du diff reste humaine**, avec CodeRabbit en plus. Le montage à
  trois relecteurs agents décidé le 2026-08-26, qui aurait supprimé la porte
  humaine selon le standard `github.md` de punt-kit, a été **renversé le
  2026-08-27** après mesure sur une vraie *pull request* : Gemini n'était pas
  installé sur ce dépôt et `claude-review` coûtait de l'ordre de 0,60 $US par
  tour d'API. `claude-review.yml` est conservé désactivé (`docs/12` §6.3).

[Unreleased]: https://github.com/wquintal/mecabot/commits/main
