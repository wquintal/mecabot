# 04 — Budget de tokens et compression de la télémétrie

Le document initial avait raison sur l'intuition centrale : il ne faut jamais injecter du flux brut dans un LLM. Il se trompait sur l'ampleur du problème et sur les moyens.

---

## 1. L'arithmétique, faite

### 1.1 Limites physiques du bus

Une trame CAN 2.0A porte 47 bits de surcharge plus jusqu'à 64 bits de données, et peut atteindre ~132 bits avec le bit stuffing pire cas [VÉRIFIÉ, spécification CAN].

| Bus | Débit | Trames/s théoriques max |
|---|---|---|
| HS-CAN | 500 kbit/s | ~3 788 |
| MS-CAN | 125 kbit/s | ~947 |

À 40 % d'occupation du HS-CAN : ~1 514 trames/s, soit ~12 kB/s de charge utile, soit **~728 kB/min**.

Journalisé naïvement en hexadécimal textuel (`ID:0x0CF11E0F DATA:0x0000F00401000000` ≈ 45 caractères ≈ 15 tokens) : **~1,36 million de tokens par minute.** Infaisable, et pas d'un facteur 2 — d'un facteur mille.

### 1.2 Mais ce n'est pas le vrai chiffre

Le point que le document initial rate : **tu n'auras jamais ce débit.** Un adaptateur OBD interrogé en mode requête/réponse est limité par le protocole, pas par le bus.

| Contrainte | Valeur |
|---|---|
| Charge utile par réponse en trame simple | 7 octets |
| PID max par requête (extension multi-PID Mode 01) | 6 |
| Intervalle inter-requêtes recommandé | 300-500 ms |
| Surcharge de traitement ELM327 | 50-120 ms par PID |
| Surcharge STN1110/STN2120 | ~20-30 ms (≈3× plus rapide) |

Débits réalistes :

- **ELM327, PID unique** : ~2-6 PID/s
- **ELM327, lot de 6** : 12-20 PID/s en pratique

Volumes en tokens, texte non compressé :

| Scénario | Échantillons/min | Tokens/min | Fenêtre 128k remplie en |
|---|---|---|---|
| 15 PID à 1 Hz | 900 | **~7 200** | ~18 min |
| 10 PID à 5 Hz | 3 000 | **~24 000** | **~5,3 min** |

Une heure de roulage à 15 PID / 1 Hz ≈ **432 000 tokens**, soit 3,4 fois une fenêtre de 128k.

**Le problème est donc réel, mais son ordre de grandeur est ~10⁴ tokens/min, pas ~10⁶.** Ça change les solutions : on n'a pas besoin d'une pipeline d'ingénierie de données industrielle. Un bon résumé et de l'échantillonnage événementiel suffisent largement.

---

## 2. Techniques de réduction, avec leurs propriétés

| Technique | Préserve | Perd | Ratio typique auto | Coût CPU |
|---|---|---|---|---|
| **Bande morte / delta** | Changements significatifs, écarts soutenus | Oscillations sous seuil ; référence absolue si le point initial est jeté | Température moteur stabilisée 20:1 à 50:1 ; MAP en accélération 2:1 à 4:1 ; régime en croisière 5:1 à 10:1 | O(1), négligeable |
| **Swinging Door Trending** | Rampes linéaires, marches | Courbure dans la tolérance | 5:1 à 20:1 (4:1 à 50:1 en process industriel) | O(1) |
| **Douglas-Peucker** | Points d'inflexion, extrema | Points à moins de ε du modèle | 5:1 à 20:1 ; jusqu'à 30:1 sur signaux très lisses | O(n log n), **non natif au flux** |
| **PIP** | Extrema, retournements de tendance | Oscillations intermédiaires | 3:1 à 15:1, sortie ordonnée par importance | modéré |
| **SAX** | Formes, motifs, tendances relatives | **Magnitude exacte et échelle absolue** (normalisation volontaire) | 256:1 en octets ; **5-10× en tokens** | faible |
| **CUSUM** | Instant et amplitude des changements de régime | Variation intra-régime | Autoroute stable 20:1 à 100:1 ; ville 5:1 à 20:1 | O(1), quasi nul |
| **BOCPD** | Idem, plus sensible aux changements subtils | Idem | idem | O(t) naïf, O(1) amorti avec fenêtre d'oubli |

**Le piège de SAX pour ce projet :** SAX normalise à moyenne nulle et variance unitaire, donc **détruit l'échelle absolue**. C'est parfait pour reconnaître une forme (« le régime a fait ce motif »), et **inutilisable pour du diagnostic à seuil** (« la correction court terme a dépassé +10 % »). Or le diagnostic automobile est massivement à seuil. SAX est donc un outil de niche ici, pas la solution générale que sa réputation suggère.

**Recommandation :** bande morte + CUSUM sur les signaux clés, plus des instantanés horodatés. C'est trivial à implémenter, O(1), et ça préserve exactement ce dont le diagnostic a besoin. Le reste est de la sophistication prématurée.

---

## 3. Ce que la recherche dit sur les LLM et les séries temporelles

Trois résultats qui devraient recadrer l'ambition du document initial.

**LLMTIME** (Gruver et al., arXiv:2310.07820, NeurIPS 2023) : les LLM encodent des séries temporelles comme chaînes de chiffres et atteignent une prévision zero-shot comparable à des modèles dédiés. Détail révélateur : **GPT-4 fait pire que GPT-3** sur les séries temporelles, à cause de sa tokenisation des nombres et d'une calibration d'incertitude dégradée par le RLHF.

**« Are LLMs Useful for Time Series Forecasting? »** (Tan et al., arXiv:2406.16964, **Spotlight NeurIPS 2024**) : trois méthodes populaires de prévision par LLM ont été ablatées. Conclusion : « retirer le composant LLM ou le remplacer par une couche d'attention basique ne dégrade pas la performance — dans la plupart des cas, les résultats s'améliorent même. » Le bénéfice observé dans les travaux antérieurs venait de la structure de patching et d'attention, **pas de la connaissance linguistique**.

**Goulot de la tokenisation** (arXiv:2606.18986) : le BPE fragmente les valeurs numériques continues en « tokens instables dont les embeddings manquent de structure métrique, ce qui entraîne la perte d'information de magnitude, d'échelle et de tendance ». `"123.45"` peut se tokeniser `["12","3",".","4","5"]` ou `["123",".45"]` selon le contexte — la structure arithmétique est détruite.

**Ce que ça implique pour Mecabot :** ne jamais demander au LLM de faire de l'analyse numérique fine sur des séries. Le calcul se fait en code déterministe ; le LLM reçoit des conclusions et des grandeurs peu nombreuses. Cette conclusion rejoint celle du document initial (§3.2), mais pour une raison plus forte que « c'est inefficace » : **c'est incorrect**.

### 3.1 Une piste que le document initial ignorait complètement : le visuel

**arXiv:2608.07427** compare l'encodage textuel de séries temporelles à leur rendu en graphique 2D soumis à un modèle vision-langage :

- **réduction de 3,6× à 10,4× des tokens d'entrée** (Llama-3.2-90B, Qwen2.5-VL-72B, Pixtral-12B)
- **1,8× à 2,5× de réduction d'énergie d'inférence mesurée**
- **+220,7 % de précision** (Llama-3.2-90B-Vision affiné vs son équivalent texte seul)
- constat clé : **à 24 indicateurs, la représentation textuelle dépasse la fenêtre de 128k** ; l'encodage visuel reste dedans

Concrètement : rendre 60 secondes de tracés (régime, pression rampe, EGT, position EGR) en un PNG de 400×200 et le soumettre à un modèle multimodal est à la fois **moins cher et plus juste** que de sérialiser les valeurs. Les VLM conservent les relations spatiales — pics, creux, tendances — que la tokenisation BPE détruit.

C'est probablement le levier le plus intéressant, et le plus contre-intuitif, de tout ce dossier.

---

## 4. Ce qui est réellement détectable depuis l'OBD-II

### Détectable

- **Correction court terme (STFT) au-delà de ±10 %** : mélange riche/pauvre actif — fuite d'admission, injecteur qui bave, MAF encrassé, sonde lambda fatiguée
- **Dérive de la correction long terme (LTFT)** : la condition a persisté assez pour que l'ECU l'apprenne
- **Écart MAF vs charge calculée** : recoupement entre débit attendu (MAP/TPS) et lecture MAF ; attrape un MAF contaminé ou une fuite en aval
- **Motif de réponse de la sonde lambda** : fréquence de commutation (lente = vieillissement), plage de tension (0,1-0,9 V normal), biais permanent
- **Température moteur vs temps de fonctionnement** : fonction du thermostat (88-92 °C en 5-8 min en charge)
- **Tension batterie aux événements de contact** : chute au démarrage, régulation alternateur au ralenti vs en charge
- **Statut des monitors embarqués** (Mode 01 PID 0x01) : indique si un P0420/P0430 est imminent

### Non détectable — nécessite des signaux internes à l'ECU

- **Détection de ratés par vitesse vilebrequin** : l'ECU surveille le capteur CKP à 360-720 événements par tour, soit ~36 000 événements/s à 3 000 tr/min. **Ces données n'apparaissent nulle part dans les modes OBD-II standards.** L'OBD n'expose que le verdict pré-calculé.
- Contribution en puissance par cylindre sans PID propriétaires
- Largeur d'impulsion et équilibre d'injecteurs (parfois via Mode 22 étendu, pas standardisé)
- Différence de régime turbine/arbre d'entrée de boîte pour la détection de glissement

### 4.1 Verdict sur la détection de ratés par LSTM

**La revendication du document initial est fausse comme formulée.**

L'affirmation était : un modèle LSTM embarqué détecte « la signature caractéristique d'un raté d'allumage » à partir du signal de vitesse du vilebrequin. Trois raisons pour lesquelles ça ne tient pas :

1. **Le régime tel qu'exposé par l'OBD est échantillonné à 1-10 Hz.** La fluctuation de vitesse vilebrequin causée par un raté à 2 000 tr/min se produit sur ~33 ms. À 2 Hz d'interrogation, tu vois au plus un échantillon pendant l'événement, sans aucune résolution temporelle. L'inertie de la transmission à vitesse de croisière amortit encore le signal.
2. **Les articles qui revendiquent ce résultat** soit (a) lisent le compteur de ratés déjà calculé par l'ECU — ce qui est une lecture de registre, pas une détection —, soit (b) utilisent des symptômes secondaires (instabilité de régime, perturbation des corrections, pointes de température catalyseur) dont la spécificité est faible.
3. **Les articles de détection sur bus CAN** (CANShield, CAN-BERT et similaires) travaillent sur du **trafic CAN propriétaire brut**, avec des identifiants de gestion moteur non standardisés — inapplicable à un montage ELM327/STN sur le port de diagnostic.

⚠️ La revue MDPI « A Review of OBD-II-Based Machine Learning Applications », citée dans le document initial, **n'a pas pu être consultée** (HTTP 403 sur MDPI, plusieurs tentatives d'URL) [INTROUVABLE]. Le verdict ci-dessus s'appuie sur la littérature de détection sur CAN qui, elle, a été consultée, et sur l'arithmétique d'échantillonnage.

**Ce qui reste vrai et utile :** un modèle embarqué peut détecter les symptômes métaboliques d'un raté **soutenu** (dérive STFT/LTFT, asymétrie lambda) avec une spécificité modérée. Et surtout, sur un diesel, la **dérive lente** des corrections d'injecteur, de la position EGR, ou de la charge de suie est parfaitement observable — mais par de la statistique simple sur des semaines, pas par un LSTM sur des millisecondes.

---

## 5. Les deux artefacts OBD à plus forte densité diagnostique

### Freeze frame (Mode 02)

Quand un DTC se pose, l'ECU enregistre un instantané de tous les PID Mode 01 à cet instant. Le Mode 02 expose cet instantané avec les mêmes numéros de PID. Contenu typique : régime, charge calculée, température moteur, STFT/LTFT bancs 1 et 2, MAP/MAF, papillon, tensions lambda, temps de fonctionnement, vitesse — soit 12 à 20 PID.

- Octets bruts : ~20-40
- En contexte LLM structuré : **~60-120 tokens**
- **Densité diagnostique : très élevée.** C'est l'artefact le plus informatif de tout l'OBD-II — l'enregistreur de vol au moment de la panne.

**Toute session sur un véhicule avec DTC stocké devrait commencer là.** Le document initial ne le mentionne pas une seule fois, ce qui est l'omission la plus coûteuse de sa section sur les tokens.

### Mode 06 — résultats des monitors embarqués

Retourne statut réussite/échec et valeurs min/max/courante pour les monitors non continus (chauffage lambda, efficacité catalyseur, EVAP, EGR). Format : Test ID (1 octet) + limite min (2) + limite max (2) + valeur courante (2) = **7 octets par test**.

Sur un Ford 2016 (ISO 15765-4 à 500 kbaud) : Mode 06 est accessible au port standard ; Ford mélange des Test ID normalisés CARB et des ID propriétaires. Un ELM327 générique retourne l'hexadécimal brut ; **l'interprétation exige les tables TID/CID Ford**.

- Réponse typique Ford : 20 à 60 résultats de test
- En contexte structuré (« chauffage lambda B1S1 : 0,42 A, min 0,10, max 0,80 — RÉUSSI ») : ~15 tokens par test, **~600 tokens pour un dump complet**

Densité par token inférieure au freeze frame, mais **valeur unique** : on voit un monitor dériver **avant** qu'il ne pose un DTC.

---

## 6. Côté RAG : ce que la littérature établit

### 6.1 Les architectures qui marchent ne laissent pas le LLM raisonner seul

**RAG guidé par graphe de connaissances pour la validation HiL automobile** (arXiv:2608.11277, ICSRS 2026) : graphe modélisant les relations capteur→localisation de défaut, plus RAG sur cas historiques, plus LLM en couche de décision et d'explication. Entrée : séries temporelles brutes de bancs HiL converties en « évidence diagnostique compacte ».

- **90 % de précision top-1** sur moteur essence
- **94 % top-1** sur système de véhicule électrique
- Constat clé : l'ancrage par graphe empêche le LLM de produire des localisations de défaut plausibles mais fausses. **Le LLM n'est pas le détecteur, il est l'explicateur.**

**CAREP** (arXiv:2602.01155, févr. 2026) : agent de découverte causale identifiant les relations DTC→motif d'erreur par modèles causaux structurels, plus raisonnement contextuel LLM. Échelle : **29 100 DTC uniques, 474 motifs d'erreur** sur jeu OEM propriétaire. Surpasse les baselines LLM seul, avec explications causales transparentes. La structure causale contraint le raisonnement.

**RADIANT-LLM** (arXiv:2604.22755, sûreté nucléaire — méthodologie transposable) : **85-98 % de précision de contexte**, taux d'hallucination « substantiellement inférieurs » au déploiement générique quand le RAG est conscient du domaine.

**Le motif transversal :** les systèmes performants n'utilisent pas les LLM comme raisonneurs autonomes. Ils les utilisent comme **générateurs d'explication ancrés** par (a) un graphe de connaissances structuré, (b) des cas historiques récupérés, ou (c) du RAG documentaire. Le LLM fournit l'interprétabilité, pas la détection.

C'est exactement le repositionnement recommandé dans `00-synthese.md`.

### 6.2 Risque d'hallucination : les cinq modes de défaillance à craindre

1. **Signification des DTC spécifiques constructeur** : les LLM connaissent bien les P0xxx génériques (abondants en données d'entraînement) mais **fabriquent des descriptions plausibles** pour les codes B/C/U propriétaires. Un B10AA Ford 2016 peut être décrit comme une panne d'un tout autre système.
2. **Couples de serrage** : valeurs récupérées d'années-modèles voisines ou de plateformes similaires ; des écarts de 20-30 % sont possibles, et critiques sur des écrous de roue ou des vis de culasse.
3. **Couleurs de fils** : non normalisées entre constructeurs, ni même entre années d'un même constructeur. Les LLM n'ont aucune source fiable et **fabriquent avec assurance**.
4. **Procédures de démontage** : étapes plausibles mais omettant des précautions critiques, citant de mauvais numéros d'outils, ou inversant l'ordre d'agrafes et de boulons.
5. **Numéros de pièces** : devinés dans un format plausible.

**Règle de conception non négociable :** toute affirmation portant sur une spécification ou une procédure doit porter une citation vers le fragment source précis. **Une réponse sans citation à une question de spécification doit être traitée comme hallucinée.**

### 6.3 Découpage des manuels : les chiffres

| Approche | Précision rapportée | Source |
|---|---|---|
| Découpage hiérarchique + Docling | **94,1 %** | arXiv:2604.04948 |
| PDFLoader naïf | 86,2 % | arXiv:2604.04948 |
| Découpage adaptatif (LLM-regex + fusion) | 72 % (depuis 62-64 %) | arXiv:2603.25333 |
| Découpage dynamique piloté par intention | top-1 67 % (+5 %), 40-60 % de fragments en moins | arXiv:2602.14784 |
| Structure-aware tabulaire | MRR 0,5945 vs 0,3576 (récursif), 56 % de fragments en moins | arXiv:2605.00318 |

Enseignements applicables directement :

1. **Les questions dépendant de tableaux creusent l'écart le plus large : 33 points de pourcentage** entre découpage naïf et hiérarchique (arXiv:2604.04948). Or les couples de serrage, capacités de fluides et brochages de connecteurs sont **tous** tabulaires.
2. **Le découpage doit suivre la structure du document** (chapitre → section → étape de procédure), pas une taille fixe.
3. **L'enrichissement par métadonnées compte plus que le choix de l'outil de parsing** : chapitre, section, variante, nom de système en champs de métadonnées améliorent nettement la précision de récupération.
4. **Ne jamais couper au milieu d'une procédure.** Une purge de freins en 12 étapes découpée sur trois fragments garantit la perte de contexte.
5. **Le RAG textuel échoue complètement sur les schémas électriques** (arXiv:2603.24556, testé sur des P&ID de pétrole et gaz, directement transposable) : « les quatre méthodes se sont révélées inefficaces ». Un schéma exige du multimodal.

### 6.4 Schémas électriques : la seule voie praticable

À l'ingestion, faire passer chaque page de schéma dans un VLM avec une consigne du type : « décris tous les connecteurs, couleurs de fils, affectations de broches et chemins de circuit visibles ; liste les étiquettes de composants ». Stocker la description générée **à côté** de l'image source. À la requête, récupérer les deux : la description pour la correspondance d'embedding, l'image pour un réexamen VLM si nécessaire.

C'est imparfait — la précision des VLM sur des schémas denses est de l'ordre de **60-80 % selon la qualité du document** — mais c'est le seul chemin viable pour du RAG sur des schémas. Et ça implique de traiter toute lecture de schéma comme une piste à vérifier, jamais comme un fait.

### 6.5 Corpus

**Il n'existe aucun corpus ouvert de manuels d'atelier automobiles.** Les droits sont détenus par les constructeurs et les agrégateurs. Ce qui existe :

- jeux de données publics de signaux OBD (POLIDriving, dépôt UCI, jeux d'intrusion CAN OTIDS / Car-Hacking) — utiles pour modéliser des signaux, pas pour des procédures
- normes SAE publiques (J1979, J2012) — cadre de définition des DTC, rien de spécifique au véhicule
- données de fréquence de pannes CARB/EPA

Pour Mecabot, le corpus se construit à la main, depuis l'abonnement payant, **et pour un seul véhicule** — quelques centaines de pages, pas un corpus industriel. C'est faisable précisément parce que le périmètre est réduit à un véhicule.

---

## 7. Synthèse : la stratégie de budget recommandée

Par ordre de rapport valeur/effort :

1. **Commencer par le freeze frame et les DTC.** ~100-300 tokens pour l'artefact le plus dense qui existe. C'est le point de départ de toute session.
2. **Rendre des `resource_link` plutôt que des données.** Mécanisme officiel de la spec MCP (`03-architecture-mcp.md` §6) : l'outil rend un lien vers le journal de session ; le contenu n'entre en contexte que si nécessaire.
3. **Bande morte + CUSUM sur les signaux clés, en code déterministe.** O(1), trivial, préserve les seuils. Émettre un événement + instantané aux points de changement.
4. **Sortie structurée typée** (`outputSchema`) pour que le modèle ne parse pas de texte.
5. **Pour les formes temporelles, envisager le rendu graphique + VLM** plutôt que la sérialisation numérique : 3,6-10,4× moins de tokens et une meilleure précision.
6. **Le calcul numérique reste en code.** Le LLM reçoit des conclusions et peu de grandeurs. Ce n'est pas une optimisation, c'est une condition de justesse.
7. **Mode 06 en second rideau**, pour voir les monitors dériver avant le DTC — ~600 tokens, nécessite les tables Ford.
8. **Toute affirmation de spécification porte sa citation**, sinon elle est traitée comme hallucinée.
