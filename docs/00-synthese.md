# Mecabot — Synthèse de préconception

**Date :** 2026-08-19 · **dernière révision de périmètre : 2026-08-25** · **standards de développement : 2026-08-26**
**Objet :** un serveur MCP de diagnostic automobile **OBD-II / UDS générique, en lecture seule**, multiplateforme.
**Véhicule de référence n°1 :** Ford Transit 350HD 2016 (plateforme V363, marché nord-américain) — **3,2 L Power Stroke I5 diesel, après-traitement retiré (*deleted*), calibration PCM modifiée**. C'est le sujet de validation, **pas le périmètre**.
**Statut :** préconception. Aucun code écrit. Aucune décision d'architecture gelée.

> ⚠️ **Périmètre, à lire avant le reste.** L'objectif est **l'équivalent multiplateforme de FORScan, en lecture seule, exposé à un agent IA via MCP** — motivation concrète : FORScan ne tourne que sous Windows. Architecture retenue : **serveur MCP en Rust, transport stdio, sur OBDLink EX**. Voir `09`, et `09` §11 pour le déploiement embarqué (repoussé en fin de parcours).
>
> ## 🔄 Rescope du 2026-08-25 — l'application est multi-marques
>
> Ce document était organisé autour d'un véhicule ; il ne l'est plus. **La règle est désormais qu'aucun fait sur un véhicule particulier n'est écrit dans le code** : la connaissance véhicule vit en **données**, sur cinq couches (`11`). Le Transit devient le profil de référence n°1.
>
> **Ce que ça change moins qu'il n'y paraît :** les quatre paliers de capacité de `09` §4 se révèlent être la projection de ces cinq couches sur l'axe des fonctionnalités. Ils ne décrivaient pas des étapes propres au Transit mais des besoins de connaissance — donc la structure du dossier tient, et la feuille de route ne change pas d'ordre.
>
> **Ce que ça change vraiment :** l'application devient utile le premier jour sur **n'importe quel** véhicule conforme OBD-II, avant qu'aucun travail de découverte n'ait été fait, avec une dégradation gracieuse au lieu d'un support binaire. En échange, elle hérite d'une surface de sûreté nouvelle — passerelles sécurisées, découverte à autoriser par profil, variantes de transport hors périmètre (`11` §6).
>
> **Conséquence sur ce document :** le « mur de parité FORScan » décrit plus bas **existe toujours, mais il n'est plus binaire.** Il se franchit par paliers — l'OBD-II normalisé et l'inventaire de DTC multi-modules ne demandent aucune donnée propriétaire ; seuls les DID étendus exigent un travail de découverte incrémental. Les sections 1 à 3 ci-dessous restent valides, en lisant « Ford » comme **un cas d'étude d'accès aux données** plutôt que comme le périmètre.
>
> *(Une interprétation intermédiaire du projet — « FORScan en amont, Mecabot lit ses fichiers » — a été explorée en `08` puis écartée : c'était un malentendu de ma part sur l'intention. `08` est conservé avec un avertissement, pour la trace. C'est le précédent qui justifie la prudence de `11` §8 sur l'abstraction prématurée.)*

---

## La question qui compte

Tu as posé une inquiétude en trois parties : les **données propriétaires**, les **éléments sécurisés**, et les **API inaccessibles**. La recherche donne une réponse nuancée mais globalement rassurante, à condition d'abandonner l'ambition « niveau OEM universel » du document initial.

### 1. Les données propriétaires : accessibles, mais reverse-engineerées, pas documentées

Ford n'a jamais publié ses PID étendus. La base de données des PID non standard de Ford est détenue par l'Equipment and Tool Institute (ETI) et se licencie à environ **50 000 $US/an** — c'est hors de portée et ça le restera.

Mais ce n'est pas la seule voie. Ce que tu peux obtenir :

| Ressource | Coût | Nature |
|---|---|---|
| Données As-Built par VIN (config usine de chaque module) | **gratuit** | `motorcraftservice.com/AsBuilt`, sans abonnement |
| Manuel d'atelier + schémas électriques + TSB texte intégral | ~27 $ (3 jours) à ~115 $ (1 an) | `motorcraftservice.com`, ouvert aux Canadiens |
| Alternative moins cher, couvre 2016 | 19,99 $/mois ou 59,99 $/an | ALLDATA DIY |
| Rappels + plaintes NHTSA, texte intégral | **gratuit, API JSON sans clé** | `api.nhtsa.gov` |
| Décodage VIN complet | **gratuit, API sans clé** | vPIC NHTSA |
| PID étendus Ford (DPF, EGT, rail, EGR, turbo…) | ~15-20 $/an | FORScan Extended License |

La conclusion importante : **la porte n'est pas fermée, elle est payante et pas machine-readable.** Il n'existe aucune API. Tu paies un abonnement et tu lis des PDF. Pour un projet mono-véhicule, ~80 $/an couvre 95 % du besoin documentaire.

### 2. Les éléments sécurisés : c'est ton meilleur coup de chance

**Le Transit 2016 n'a pas de Secure Gateway.** Ford n'a jamais déployé de pare-feu CAN matériel équivalent à celui de FCA (introduit MY2018 sur Wrangler JL / Ram 1500). La recherche l'a vérifié par un signal négatif fort : une recherche plein texte du forum FORScan retourne **zéro résultat** pour « gateway » / « secure gateway ». Sur n'importe quel forum Jeep, le SGW sature les discussions. Le silence est concluant.

Ce qui existe quand même, et qu'il faut regarder en face :

- **UDS $27 SecurityAccess est bien présent, par module**, et l'algorithme Ford 2015-2019 n'est **publiquement connu de personne**. Le dépôt `jglim/UnlockECU`, la base seed/key ouverte la plus complète (3 038 entrées : Daimler, VW, Honda, Subaru…), contient **exactement 0 entrée Ford**. Ford a par ailleurs fait retirer des travaux antérieurs par DMCA.
- Conséquence directe : **tu ne peux pas écrire dans les modules par tes propres moyens.** FORScan y arrive parce qu'il embarque une base propriétaire de clés extraites des DLL de Ford IDS. Tu ne reproduiras pas ça, et essayer t'exposerait juridiquement.
- **Le reflashing n'existe plus nulle part hors dealer** : FORScan a retiré la fonction de toutes ses licences ; Ford a verrouillé les fichiers de firmware. Il faut IDS/FDRS + un compte TIS2Web.

**La bonne nouvelle : rien de tout ça ne bloque la lecture.** Les DTC de tous les modules, les PID étendus via UDS $22, les données live, le freeze frame, les tests d'actionneurs de base — tout ça est accessible sans authentification, sur un véhicule 2016, avec un adaptateur à 70 $.

### 3. Les API inaccessibles : confirmé, et ça ne bloque rien

MOTOR Information Systems, ALLDATA, Mitchell : **aucun ne fournit d'API à un individu**. MOTOR est du B2B pur avec contrat sur devis (ordre de grandeur : plusieurs milliers de dollars par mois). Cette section du document initial était de la fiction.

Mais le besoin qu'elle prétendait couvrir — donner du contexte de réparation fiable au LLM — se règle autrement : tu achètes l'abonnement Motorcraft ou ALLDATA DIY, tu télécharges les PDF des sections qui concernent **ton** véhicule, et tu construis ton propre index RAG local. Un seul véhicule, ce n'est pas un corpus de 40 000 modèles ; c'est quelques centaines de pages.

---

## Le vrai mur, celui dont le document initial ne parle pas

Ce n'est ni la sécurité ni les données. C'est **l'absence de recouvrement entre ce que FORScan sait faire et ce que ton middleware pourra faire.**

FORScan est le point de comparaison implicite de tout le document. Or FORScan, sur ton véhicule, apporte deux choses que tu ne peux pas répliquer :

1. **La base de PID/DID étendus Ford** (reverse-engineerée, propriétaire, protégée par les règles du forum qui interdisent explicitement d'en discuter).
2. **Les clés SecurityAccess** pour les opérations d'écriture.

Ce que ton middleware peut apporter que FORScan n'apporte pas : **le raisonnement, la corrélation, la mémoire longue, et la documentation contextuelle.** FORScan te montre 40 PID ; il ne te dit pas que la dérive de tes valeurs de correction d'injecteur sur six mois correspond au TSB X et au rappel 16V618000. C'est là que la valeur est, et c'est là qu'il faut viser.

**Le repositionnement que je recommande :** Mecabot n'est pas un remplaçant de FORScan. C'est une **couche d'analyse et de mémoire au-dessus** d'un accès en lecture. FORScan reste l'outil d'écriture, sous ton contrôle manuel.

> **Précision (2026-08-24) :** ce paragraphe reste la bonne analyse de la valeur, mais le mur se franchit mieux que je ne le laissais croire. Les paliers 1 et 2 de `09` §4 — OBD-II normalisé, puis inventaire de modules et DTC de tout le véhicule via `$19` — **ne demandent aucune donnée propriétaire** et couvrent déjà une grosse part de l'usage courant de FORScan. Seuls les DID étendus (charge de suie, corrections d'injecteur, EGT, pression de rampe) exigent une découverte incrémentale. FORScan reste l'outil d'écriture et devient un **oracle de validation** pendant ce travail, pas une dépendance d'exécution.

---

## Ce qu'il faut vérifier sur le véhicule de référence n°1

*Mis à jour le 2026-08-24 : les deux premières inconnues sont levées. **Reclassé le 2026-08-25.***

> **Ces items ne bloquent plus la conception.** Avec le rescope multi-marques, ce sont des tâches de **remplissage du profil n°1** (couches 3 et 4 de `11` §4), pas des préalables d'architecture. Ils restent utiles — sans un premier profil rempli, il n'y a rien à valider contre.
>
> **Le seul véritable préalable à l'écriture de code est ailleurs, et il ne dépend pas du véhicule :** c'est l'**item 4bis de `07`**, une séance au terminal série pour caractériser l'adaptateur STN — et pour en **journaliser** la trace, qui devient le premier jeu de test du projet (`10` §9).

1. ~~**Quel moteur ?**~~ ✅ **3,2 L Power Stroke I5 diesel** — et le véhicule est **deleted**, ce qui implique une calibration PCM modifiée. Ça déplace la cible du projet vers le circuit haute pression, le turbo et la 6R80, ça retire le DPF et les EGT de l'équation, et ça introduit un piège de raisonnement : **la calibration supprime des familles de DTC, donc un scan silencieux ne prouve rien.** Voir l'encadré en tête de `02`.
2. ~~**DEF/SCR présent ou non ?**~~ ✅ **Sans objet.** Corollaire important : le **bridage à ~8 km/h**, l'élément de sûreté le plus inquiétant du dossier, ne s'applique pas.
3. **Rappels 16V618000, 17V408000, 19V767000 effectués ?** — **toujours ouvert, et devenu la vérification n°1.** Le premier concerne des débris métalliques de pompe HP qui détruisent les injecteurs ; il est spécifique au diesel, donc il **s'applique**, et le delete ne touche pas le circuit d'alimentation. Les deux autres concernent l'accouplement flexible de transmission qui casse sans préavis. Vérifiable gratuitement par VIN sur `ford.ca/support/recalls` ou l'API NHTSA.
4. **Ce que la calibration a désactivé** — nouveau, gratuit, entièrement normalisé : CALID et CVN (Mode `$09` InfoTypes `04` et `06`), bits de moniteurs (Mode `$01` PID `01`), et Mode `$06`. Item 16 de `07`.

---

## Documents de ce dossier

| Fichier | Contenu |
|---|---|
| `01-acces-donnees-et-securite.md` | Passerelles, UDS $27, ce que FORScan fait, coûts d'accès aux données, cadre légal (CASIS) — **à lire comme un cas d'étude Ford/Canada, désormais** |
| `02-profil-vehicule.md` | **Profil de référence n°1** : topologie réseau, modules, pannes fréquentes, rappels, PID diesel. Couches 3 et 4 du Transit V363. |
| `03-architecture-mcp.md` | État réel de la spec MCP 2026, primitives utilisables, human-in-the-loop, ce qui est impossible |
| `04-budget-tokens.md` | Arithmétique des volumes, techniques de compression, verdict sur le ML embarqué |
| `05-materiel-et-stack.md` | Adaptateurs, MS-CAN, bibliothèques, contraintes macOS, art antérieur |
| `06-critique-du-document-initial.md` | Ce qui était juste, faux, ou inventé dans le document de départ |
| `07-verifications-a-faire.md` | Liste ordonnée des inconnues à lever, avec la méthode |
| ~~`08-architecture-forscan-bridge.md`~~ | ⛔ **Supersédé** — prémisse erronée (FORScan en amont). Conservé pour la trace. |
| **`09-architecture-cross-platform.md`** | **Architecture retenue : serveur MCP Rust, paliers de capacité, découverte de DID, sûreté, feuille de route, déploiement embarqué** |
| **`10-conception-serveur-mcp.md`** | **Conception détaillée du serveur : cycle de vie de la connexion, concurrence, taxonomie d'erreurs, contrat des outils, ce qu'il ne faut pas construire, comment tester sans le véhicule** |
| **`11-modele-de-connaissance-vehicule.md`** | **Ce qui rend l'application multi-marques : les cinq couches de connaissance véhicule, l'art antérieur vérifié (ODX, OpenDBC, CaringCaribou, AutoAuth), le contenu d'un profil, la surface de sûreté ajoutée, et le traitement du risque d'abstraction prématurée** |
| **`12-standards-de-developpement.md`** | **Comment le développement est structuré : ce qui est emprunté à punt-kit et ce qui est refusé** — le fossé Rust, le CLI complet ajouté au périmètre, la journalisation qui rend applicable la règle de couche 4, la flotte de relecture qui remplace la porte humaine, et les décisions du 2026-08-26 |

**Ordre de lecture recommandé :** `09` d'abord (l'architecture retenue), puis **`11`** (ce que le logiciel doit savoir d'un véhicule, et d'où ça vient), puis `10` (la conception du serveur), puis `04` (comment compresser). **`12` se lit juste avant d'écrire la première ligne** — c'est lui qui dit sous quelles règles le code se produit. `07` liste ce qu'il faut aller vérifier — dont l'item 4bis, le seul préalable réel au code. `02` est le profil de référence n°1. `01` explique où sont les frontières d'accès aux données. `03`, `05` et `06` sont de la documentation de référence. `08` est à ignorer.

**Si tu ne lis que deux documents :** `09` pour l'architecture, `11` pour la connaissance véhicule. Ce sont les deux qui portent les décisions.

> ### 📦 Le dossier vit maintenant dans un dépôt — 2026-08-26
>
> **`github.com/wquintal/mecabot`, public.** Le dossier n'est plus un répertoire local : il est versionné, relu en intégration continue, et lisible de l'extérieur.
>
> Ce que ça change concrètement pour la suite du travail :
>
> - **`main` est protégée** et toute modification passe par une *pull request*, y compris une modification de documentation. La vérification `docs` (markdownlint) est obligatoire.
> - **Il n'y a pas de porte de relecture humaine sur le diff.** Elle est remplacée par **trois relecteurs de code** — Gemini, CodeRabbit, Claude. `cargo-deny` s'y ajoute comme porte de dépendances, mais ce n'est pas un relecteur et il dort tant qu'il n'y a pas de `Cargo.toml`. La substitution n'est valide que tant que les trois tournent réellement, et trois est le plancher. ⚠️ **Au 2026-08-27 il n'est pas tenu :** Gemini n'est pas actif sur ce dépôt, deux relecteurs répondent, donc **la relecture humaine du diff reste requise** en attendant (`12` §6.3).
> - **La règle « aucune dépendance GPL » est devenue exécutable** (`deny.toml`) : elle casse le build au lieu d'être une phrase qu'on peut oublier.
> - **La licence n'est pas accordée.** Dépôt public sans fichier `LICENSE` ⇒ tous droits réservés. C'est cohérent avec la décision de diffusion, qui reste ouverte.
> - ⛔ **`traces/` est ignoré par git.** Une trace série contient le VIN dès qu'elle inclut un Mode `$09` ou un `$22 F190` ; une trace se verse à la main après relecture, **jamais par un `git add .`**.
