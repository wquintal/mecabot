# 03 — Architecture MCP : ce que la spécification permet réellement

Le document initial décrivait plusieurs mécanismes MCP qui n'existent pas. Ce document établit l'état réel de la spécification, vérifié contre `modelcontextprotocol.io` et le `schema.ts` du dépôt de spécification.

---

## 1. Révisions de la spécification

| Révision | Apports majeurs |
|---|---|
| 2024-11-05 | Version initiale. Handshake `initialize`, transport HTTP+SSE, sampling et roots initiés serveur. |
| 2025-03-26 | Transport Streamable HTTP (SSE déprécié), OAuth 2.1, **annotations d'outils**, contenu audio. |
| 2025-06-18 | **Elicitation** (mode formulaire), **sortie structurée** (`outputSchema` / `structuredContent`), liens de ressources dans les résultats d'outils. |
| 2025-11-25 | Elicitation en mode URL, extension Tasks (expérimentale), `toolChoice` dans le sampling. |
| **2026-07-28** | **Révision courante.** Protocole **sans état** (plus d'`initialize`), motif MRTR, `subscriptions/listen`, `server/discover`, **dépréciation de Sampling / Roots / Logging**, propagation du contexte de trace OTel. |

[VÉRIFIÉ — toutes les URL de révision consultées]

La bascule 2026-07-28 est structurante : **le serveur n'initie plus jamais de requête JSON-RPC.** Toute conception antérieure qui reposait sur des requêtes serveur→client doit être repensée.

---

## 2. Primitives disponibles

**Côté serveur :**

- **Tools** — fonctions exécutables, contrôlées par le modèle. `tools/list` + `tools/call`. Paginé, cacheable.
- **Resources** — données contextuelles en lecture, contrôlées par l'application. `resources/list`, `resources/read`, `resources/templates/list`.
- **Prompts** — gabarits contrôlés par l'utilisateur. `prompts/list`, `prompts/get`.

**Côté client :**

- **Elicitation** — active et centrale (voir §3).
- **Sampling** — ⚠️ **déprécié** au 2026-07-28 (SEP-2577), suppression pas avant la première révision postérieure au 2027-07-28. Migration : appeler directement l'API du fournisseur de LLM.
- **Roots** — ⚠️ **déprécié** au 2026-07-28. Migration : passer les répertoires en paramètres d'outils ou en configuration serveur.

Le découpage que le document initial proposait — ressources pour la télémétrie, outils pour les actions, prompts pour le profil véhicule — reste valide et correspond bien à l'intention de la spec.

---

## 3. Elicitation : le bon mécanisme pour le human-in-the-loop

C'est exactement le cas d'usage prévu par la spec, et la réponse à la question « comment obtenir une confirmation humaine avant d'agir sur un véhicule ».

**Mécanisme courant (2026-07-28) — Multi Round-Trip Requests :** pendant un `tools/call`, un serveur qui a besoin d'une entrée utilisateur retourne un `InputRequiredResult` (`"resultType": "input_required"`). Le client affiche l'interface de confirmation, puis **relance la requête d'origine** avec `inputResponses`.

```json
// Réponse du serveur à tools/call — suspend l'exécution
{
  "jsonrpc": "2.0", "id": 2,
  "result": {
    "resultType": "input_required",
    "inputRequests": {
      "confirm": {
        "method": "elicitation/create",
        "params": {
          "mode": "form",
          "message": "Lancer une régénération DPF forcée ? Véhicule dehors, échappement dégagé, extincteur à portée ?",
          "requestedSchema": {
            "type": "object",
            "properties": { "confirmed": { "type": "boolean", "title": "Je confirme" } },
            "required": ["confirmed"]
          }
        }
      }
    },
    "requestState": "<jeton d'état opaque côté serveur>"
  }
}
```

Le client relance avec `inputResponses: { confirm: { action: "accept", content: { confirmed: true } } }`. `action` vaut `accept`, `decline` ou `cancel` — **le serveur doit gérer les trois**.

Contraintes du `requestedSchema` : objets **plats** de propriétés primitives (string, number, integer, boolean, enum). Pas d'objets imbriqués — c'est volontaire, pour que le client puisse générer une interface.

**Réserve d'adoption :** la forme MRTR du 2026-07-28 est récente. La forme antérieure (requête initiée serveur, 2025-06-18) avait des implémentations client réelles ; le support de la nouvelle par chaque client est [NON VÉRIFIÉ]. La négociation passe par `server/discover`. À tester tôt sur le client visé plutôt qu'à supposer.

---

## 4. Notifications asynchrones : ce qui est possible et ce qui ne l'est pas

C'est le point où le document initial se trompait le plus, avec sa section sur les « moniteurs asynchrones » qui réveillent l'agent.

### Notifications serveur→client existantes (2026-07-28)

| Notification | Canal |
|---|---|
| `notifications/tools/list_changed` | flux `subscriptions/listen` |
| `notifications/prompts/list_changed` | flux `subscriptions/listen` |
| `notifications/resources/list_changed` | flux `subscriptions/listen` |
| `notifications/resources/updated` | flux `subscriptions/listen` |
| `notifications/progress` | flux de réponse de la requête concernée |
| `notifications/message` (log) | flux de réponse de la requête concernée |
| `notifications/subscriptions/acknowledged` | flux `subscriptions/listen` |
| `notifications/cancelled` | — |

Les anciens RPC `resources/subscribe` / `resources/unsubscribe` **ont disparu** au 2026-07-28. Ils sont remplacés par `subscriptions/listen`, un POST long que le client maintient ouvert en déclarant ce qui l'intéresse :

```json
{ "method": "subscriptions/listen",
  "params": { "notifications": { "resourceSubscriptions": ["vehicle://pid/dpf_soot_load"] } } }
```

### La réponse à « peut-on réveiller l'agent quand un seuil est franchi ? »

**Oui, mais seulement si un flux d'abonnement est déjà ouvert.** Le serveur peut alors pousser `notifications/resources/updated` à tout moment ; le client va relire la ressource.

**Non, si le client est au repos sans abonnement.** Le protocole 2026-07-28 est explicitement sans état, les serveurs n'initient jamais de requête, et **il n'existe aucun mécanisme pour injecter un nouveau message dans la conversation du LLM depuis l'extérieur.** La spec le dit sans détour dans sa vue d'ensemble des transports : aucune autre direction de message n'existe.

Ce que le document initial décrivait — « l'interaction avec le LLM est suspendue, libérant le contexte, et le pont réveille l'agent quand la condition est satisfaite » — **n'est pas réalisable via MCP seul**. La partie « réveiller l'agent » relève de l'application hôte, pas du protocole.

### Contournements réalistes

1. **Flux d'abonnement ouvert en début de session** : le client s'abonne aux ressources de seuil ; le serveur pousse quand ça franchit. Ce qui déclenche ensuite une réintervention du LLM est la responsabilité de l'application, pas du protocole.
2. **Polling côté client** : simple, pas du push, mais coûte des tokens.
3. **Extension Tasks** (`io.modelcontextprotocol/tasks`, 2026-07-28) : l'outil rend une poignée de tâche ; le client interroge `tasks/get`. Pas du push, mais découple le travail long de la requête synchrone.
4. **Hors bande** : le backend écrit dans une file ou appelle un webhook ; un processus séparé réveille l'application hôte. Entièrement en dehors de MCP.

**Conséquence pour Mecabot :** la surveillance longue durée (« préviens-moi si la sonde lambda oscille au-delà de 0,45 V ») se conçoit comme un **service local qui journalise, et dont l'agent lit le résumé à la session suivante** — pas comme un réveil temps réel. Ce qui, pour du suivi de dérive sur des semaines, est de toute façon le bon modèle.

---

## 5. Opérations longues : progression et annulation

`notifications/progress` existe depuis la 2024-11-05 et n'a pas changé. Le client fournit un `progressToken` dans le `_meta` de la requête ; le serveur émet des notifications avant la réponse finale :

```json
{ "method": "notifications/progress",
  "params": { "progressToken": "regen-42", "progress": 45, "total": 100,
              "message": "EGT post-DPF 612 °C, suie 3,1 g/L" } }
```

`progress` doit croître de façon monotone ; `total` est optionnel.

**Annulation :** en stdio, le client envoie `notifications/cancelled` avec le `requestId`. En Streamable HTTP, il ferme le flux SSE et le serveur **doit** traiter la déconnexion comme une annulation. Les implémentations peuvent réarmer le compteur de timeout à chaque notification de progression, mais **doivent** imposer un maximum absolu.

C'est le mécanisme correct pour toute opération qui dure — un cycle de mesure de plusieurs minutes, une capture de données en roulant.

---

## 6. Sortie structurée, liens de ressources, annotations

**Sortie structurée** (depuis 2025-06-18) : un outil déclare un `outputSchema` optionnel (JSON Schema 2020-12 par défaut) ; le résultat porte `structuredContent`. Si `outputSchema` est présent, le serveur **doit** s'y conformer et le client **devrait** valider. Par compatibilité, il est recommandé de retourner aussi le JSON sérialisé dans un bloc `TextContent`.

C'est directement utile : les relevés de PID sortent comme objets typés, le modèle n'a pas à parser du texte.

**Liens de ressources dans les résultats d'outils** (depuis 2025-06-18) :

```json
{ "type": "resource_link", "uri": "vehicle://log/session-2026-08-19",
  "name": "Journal de session", "mimeType": "application/json" }
```

Ces URI n'apparaissent pas forcément dans `resources/list`. **C'est le mécanisme recommandé par la spec pour les gros volumes** : l'outil rend un lien, le contenu n'entre en contexte que si le client le lit explicitement.

**Annotations d'outils** (depuis 2025-03-26, inchangées au 2026-07-28, extraites du `schema.ts` faisant autorité) :

```typescript
interface ToolAnnotations {
  title?: string;
  readOnlyHint?: boolean;      // défaut false — l'outil ne modifie pas son environnement
  destructiveHint?: boolean;   // défaut true  — l'outil PEUT être destructeur
  idempotentHint?: boolean;    // défaut false
  openWorldHint?: boolean;     // défaut true
}
```

⚠️ Deux réserves importantes. La spec précise que les clients **doivent** considérer ces annotations comme non fiables sauf provenance de serveurs de confiance. Et elle **n'impose aucune action** au client sur la base de ces indices. Ce que Claude Code ou tout autre client en fait dans son interface est une décision d'implémentation, pas une garantie de protocole.

**Conséquence :** `readOnlyHint` est une bonne hygiène de déclaration, mais **ne constitue pas un garde-fou**. La sécurité doit être structurelle — un outil qui n'a pas de chemin de code vers l'écriture, plutôt qu'un outil marqué comme non destructeur.

---

## 7. OpenTelemetry : la correction la plus nette

**`notifications/otel/trace` n'existe pas.** Le document initial le présentait comme un mécanisme de la spec ; c'est une invention.

Ce qui existe réellement, par **SEP-414** (statut Final, créé 2025-04-25) et le changelog 2026-07-28 : la propagation du contexte de trace W3C se fait en plaçant les clés directement dans le `_meta` de n'importe quelle requête.

```json
{ "method": "tools/call",
  "params": { "_meta": {
      "traceparent": "00-0af7651916cd43dd8448eb211c80319c-00f067aa0ba902b7-01",
      "tracestate": "rojo=00f067aa0ba902b7" },
    "name": "read_pids", "arguments": {} } }
```

C'est une **documentation d'une pratique existante** (déjà implémentée dans les SDK C# et Python, Logfire, Envoy AI Gateway, OpenInference, ToolHive), pas un nouveau type de message. C'est une exception délibérée à la convention de préfixe DNS des clés `_meta`, pour préserver l'interopérabilité W3C/OTel.

L'objectif d'observabilité du document initial reste valide et souhaitable. Le mécanisme décrit était faux.

---

## 8. Transport : stdio

Trois transports définis :

1. **stdio** — JSON-RPC délimité par saut de ligne sur stdin/stdout d'un sous-processus lancé par le client. **C'est le bon choix pour un serveur attaché à du matériel local :** pas d'authentification à configurer, pas de pile réseau, isolation de processus héritée. La spec précise que les transports personnalisés sur socket Unix ou TCP **devraient** réutiliser le cadrage stdio.
2. **Streamable HTTP** — un POST par message vers un point de terminaison unique ; réponse en JSON ou en flux SSE. Pertinent pour un serveur partagé ou distant. Authentification : OAuth 2.1, jetons porteurs **par requête** (le protocole étant sans état, il n'y a plus d'authentification au niveau session). Le serveur MCP est classé comme Resource Server OAuth ; découverte par RFC 9728 ; enregistrement client par Client ID Metadata Documents (le Dynamic Client Registration est déprécié au 2026-07-28).
3. **HTTP+SSE (ancien)** — déprécié depuis 2025-03-26. À ne pas utiliser.

Pour Mecabot : **stdio**, sans hésitation. Le serveur tourne sur la machine physiquement reliée au véhicule.

---

## 9. Efficacité de contexte : ce que la spec offre

- **Pagination** par curseur sur `tools/list`, `resources/list`, `prompts/list`, `resources/templates/list`.
- **Cache** (nouveau au 2026-07-28, SEP-2549) : tous les résultats de liste et de lecture portent `ttlMs` (indice de fraîcheur) et `cacheScope` (`public` / `private`). Les clients **peuvent** cacher. Les serveurs **devraient** retourner les outils dans un ordre déterministe, pour améliorer les taux de succès du cache de prompt du LLM.
- **Pas de recherche d'outils dans la spec.** Aucun mécanisme de type RAG-sur-outils. Il n'y a ni `tools/search` ni manifeste partiel. Un serveur à forte surface doit soit tout paginer, soit concevoir sa granularité d'outils correctement.

Recommandations que la spec formule pour les sources à fort volume, et qui s'appliquent directement :

- rendre des `resource_link` plutôt qu'incorporer les données brutes — le gros payload est différé jusqu'à lecture explicite ;
- utiliser `outputSchema` / `structuredContent` pour que le modèle n'ait pas à parser du texte ;
- utiliser `ttlMs` / `cacheScope` pour que les données stables ne soient pas refetchées à chaque tour ;
- pour les données en flux ou par tranches, la section « Stateful Tools » recommande de rendre des **poignées opaques** depuis l'outil de création, et d'accepter ces poignées aux appels suivants.

Aucune limite de taille de résultat d'outil n'est imposée par la spec. La discipline est du côté de la conception.

---

## 10. Conséquences pour la conception de Mecabot

Ce que la spec valide du document initial :

- outils de haut niveau plutôt qu'accès UDS brut au modèle ✅
- ressources pour la télémétrie en lecture, prompts pour le profil véhicule ✅
- confirmation humaine avant action — et il y a un mécanisme formel pour ça (elicitation) ✅
- observabilité par traces — objectif valide, mécanisme à corriger (SEP-414, pas de notification) ✅

Ce que la spec invalide :

- le réveil asynchrone de l'agent par le serveur ❌ — impossible sans flux préouvert, et impossible d'injecter dans la conversation
- `notifications/otel/trace` ❌ — inventé
- l'idée que les annotations d'outils constituent un garde-fou ❌ — ce sont des indices non fiables par conception

Ce que la spec ajoute et que le document initial ignorait :

- `resource_link` comme réponse au problème de volume — probablement le levier le plus important de tout le dossier côté MCP
- `outputSchema` pour de la sortie typée validée
- `ttlMs` / `cacheScope` pour ne pas refetcher le stable
- la dépréciation de Sampling et Roots, à ne pas construire dessus
