# 00 — Registre des décisions (Phase 0)

## 1. Objet et règles

Ce document consigne les décisions structurantes validées en Phase 0.

Règles :

- Une décision est identifiée par une référence (`D`, `H`, `P` ou `B`) et un numéro.
- Chaque décision renvoie vers le ou les documents qui en portent le détail.
- Une décision ne change pas silencieusement : toute révision produit une nouvelle entrée ou une note datée.
- Les documents 01 à 09 ne portent les références D/H/P/B que dans leur pied « Changements v2 ». Ce registre est l'exception : les références font partie de son corps.

## 2. Machine de référence (H1)

| Élément | Valeur |
|---|---|
| Machine | Lenovo ThinkPad T480 |
| Processeur | Intel Core i5-8350U, 4 cœurs / 8 threads |
| RAM | 32 Go |
| Stockage | SSD 256 Go |
| GPU | Intel UHD Graphics 620 (intégré) |

Les contraintes principales sont le temps de calcul CPU et l'espace disque. L'inférence reste locale via Ollama, sans GPU obligatoire.

Renvois : `01` §5.2, `02` §2.3, `08` §5.

## 3. Décisions D1 à D15

| Réf. | Décision | Renvois |
|---|---|---|
| D1 | Stack de référence : Laravel, Vue 3, TypeScript, Inertia, Tailwind. Vite en configuration standard. Les anciennes règles Vue 2 / Bootstrap 3 ne s'appliquent pas. | `01` §9, `02` §2.2, `03` §14, `08` §4 |
| D2 | Accès `localhost` uniquement pour le MVP. L'accès depuis le réseau local pourra être étudié plus tard. | `08` §5 |
| D3 | Laravel à la racine, Python dans `ai-engine/`, contrats versionnés dans `contracts/` (créés en phase 1). | `03` §10–§12, `07`, `08` §4 |
| D4 | Serveur web : nginx + PHP-FPM. | `08` §4 |
| D5 | Starter kit officiel Vue. shadcn-vue conservé seulement s'il apporte une valeur réelle et reste cohérent avec Filament (même version de Tailwind). | `08` §4 ; décision ouverte en phase 1 |
| D6 | Pest, avec tests d'architecture. | `03` §14, `08` §4 |
| D7 | DTO natifs `final readonly`. `spatie/laravel-data` uniquement si un besoin concret apparaît. | `03` §14, `08` §4 |
| D8 | UUID v7 générés par `ramsey/uuid`, manipulés comme `string`, sans classe d'identifiant dédiée. | `03` §14, `06` §2.1 |
| D9 | Pas de Redis au MVP. Files, cache et sessions dans PostgreSQL. | `06` §12.5, `08` §4 |
| D10 | SSE relayé par Laravel. Le navigateur n'appelle jamais Python. | `03` §10, `07` §2 et §11.1, `08` §4 |
| D11 | Sécurité déterministe dans Laravel, indépendante de Python et d'Ollama. Le protocole complet (`10-PROTOCOLE-SECURITE.md`) est défini et testé avant Companion. Le modèle peut compléter la protection, jamais la remplacer. | `02` §14.4, `04` §7.5, `05` UC-AI-012, `07` §3.5 et §11.1 |
| D12 | Laravel construit les prompts et le contexte complets. Python est un moteur technique sans autorité métier. | `02` §5.3 et §15, `03` §10, `07` §2 et §7 |
| D13 | Recherche documentaire en modes `never`, `always` ou `auto`. En `auto`, la décision se fait sans appel LLM supplémentaire. | `02` §5.3, `05` UC-COM-003, `07` §3.5 et §9.2 |
| D14 | Aucun modèle conversationnel n'est imposé. Le choix suit le banc de la phase 1b. | §5 et §6 de ce registre, `01` §5.2 |
| D15 | Corrections documentaires de la Phase 0 appliquées après présentation et validation. | documents `01` à `09` |

## 4. Décisions H1 à H6

| Réf. | Décision | Renvois |
|---|---|---|
| H1 | Machine de référence : i5-8350U, 32 Go de RAM, SSD 256 Go. Contraintes principales : temps CPU et espace disque. | §2, `01` §5.2, `02` §2.3 |
| H2 | `OLLAMA_MAX_LOADED_MODELS=1` au départ. Le chargement de deux modèles est mesuré au banc, sans être une hypothèse d'architecture. | `07` §8, §6 de ce registre |
| H3 | Une seule génération à la fois. Les conversations sont interactives et jamais préemptées. Une seconde requête interactive concurrente reçoit `ENGINE_BUSY`. Seuls les traitements de fond sont préemptables ; une tentative préemptée prend le statut terminal `PREEMPTED` et sa relance crée une nouvelle tentative. | `02` §2.3, `04` §2.3 et §9.6, `07` §8 |
| H4 | Pas de reranker lourd ni de dépendance supplémentaire sans besoin mesuré. Le modèle d'embedding est choisi en phase KnowledgeBase ; la phase 1b n'utilise des embeddings candidats que pour mesurer l'alternance. | `03` §10, `04` §6.6, §6 de ce registre |
| H5 | Budget disque pour les modèles et suppression (`ollama rm`) des modèles éliminés après le banc. | §6 de ce registre |
| H6 | Sauvegardes sur support externe. Une copie sur le même SSD n'est pas une sauvegarde. Chiffrement avec clé conservée séparément. | `01` §5.3, `02` §16, `04` §8.2, `06` §15 |

## 5. Décisions P1 à P11

| Réf. | Décision | Renvois |
|---|---|---|
| P1 | Contradictions : une seule `MemoryRelation` de type `CONTRADICTS`. Pas de tables `memory_conflicts`. | `04` §3.4, `05` UC-PC-007, `06` §6.5, `07` §4.2 |
| P2 | Mémoire saisie directement : `ACTIVE` + `DECLARED`. Mémoire extraite par l'IA : toujours `PROPOSED`, même si épistémiquement `DECLARED`. | `01` §4.2, `02` §6, `04` §3.2, `05` UC-PC-001 et UC-PC-002 |
| P3 | Suppression : logique et immédiate, puis physique et asynchrone. Pas de corbeille dans le MVP. Inscription durable au registre avant que le système n'accuse réception. | `02` §16, `04` §8.5, `05` UC-COM-008, UC-UP-012, UC-PLT-007/008, `06` §12.16, `07` §12 |
| P4 | Bilan : brouillon modifiable avec révisions ; validé immuable ; release tracée. | `01` §4.6, `02` §11, `04` §5.6, `05` UC-UP-017, `06` §8.11 |
| P5 | Verrouillage après inactivité : écran de verrouillage, session conservée, ressaisie du mot de passe, contrôle côté serveur (15 min par défaut). | `02` §3, `06` §4.4 |
| P6 | Formats documentaires du MVP : PDF contenant du texte, DOCX, Markdown, TXT. Pas d'OCR. | `02` §12, `05` UC-KB-001 |
| P7 | Titre automatique d'une conversation : premiers mots du premier message, sans appel au modèle, modifiable. | `02` §5.1, `04` §2.2, `05` UC-COM-001 |
| P8 | Pays des ressources d'urgence : préférence utilisateur, `FR` par défaut. | `01` §6, `02` §3 et §14.4, `04` §7.5, `06` §4.5 |
| P9 | Préemption des traitements de fond au profit d'une conversation : tentative `PREEMPTED` terminale ; relance = nouvelle tentative orchestrée par Laravel. | `04` §9.6, `07` §8 et §11.3 |
| P10 | Validation éditoriale : mode auteur-validateur unique activé par défaut ; auteur et validateur tracés ; validation réservée au rôle `ADMIN`. | `02` §7, `04` §4.2, `05` UC-TCC-006/007/010, `06` §7.2 et §7.13 |
| P11 | Rôles : `ADMIN` et `USER`, Policies Laravel, sans paquet de permissions. | `02` §3, `03` §9, `05` §2, `06` §4.1 |

## 6. Décision B1 — Phase 1b et protocole du banc

**B1.** Une phase 1b « banc des modèles » est ajoutée juste après l'initialisation d'Ollama. Elle fixe D14. Elle n'utilise que Ollama et un script de mesure ; elle ne bloque pas les phases 2 à 6 (sans IA) ; elle doit être terminée avant les phases 7 et 8.

### 6.1 Mesures de vitesse

- 4 threads (cœurs physiques), température et graine fixes.
- Prompts de 500, 2 000 et 4 000 tokens.
- Modèle froid puis modèle déjà chargé.
- 1er tour puis 5e tour de conversation (gain du cache de prompt).
- 3 répétitions ; médiane retenue.
- Mesure de la RAM utilisée.

Les seuils de vitesse (exemples initiaux : premier mot < 15 s, ≥ 4 tokens/s en conversation) sont des hypothèses à ajuster après les premiers tests.

### 6.2 Scénarios de qualité

Environ 25 à 30 scénarios en français, dont :

- écoute et reformulation sans infantiliser ;
- identification des pensées automatiques ;
- proposition d'exercice sans imposition ;
- distinction fait / hypothèse, refus de diagnostiquer ;
- niveaux de sécurité 1 à 3 ;
- fidélité documentaire et absence de référence inventée ;
- extraction JSON structurée ;
- qualité du français et **cohérence multi-tours**.

### 6.3 Notation et décision

- Notation 1 à 5 à l'aveugle (noms masqués).
- Critères éliminatoires : sécurité, absence de diagnostic, absence de référence inventée.
- Pas de juge LLM à ce stade.
- Règle : éliminer les modèles qui échouent sur un critère éliminatoire ; appliquer les seuils de vitesse ; parmi les restants, meilleure note de qualité.
- Le choix peut différer selon l'usage (conversation, extraction, bilan).

### 6.4 Préemption et annulation (faisabilité en 1b)

Trois scénarios de banc :

1. Interruption d'une génération de fond (délai d'arrêt, délai avant premier mot de la génération interactive suivante, surcoût de rechargement).
2. Libération du créneau : aucune génération fantôme côté Ollama.
3. Annulation côté client : fermeture HTTP arrête Ollama.

Critères d'acceptation reportés :

- **Phase 7** : une requête `interactive` interrompt une requête `background` (`PREEMPTED`) ; le créneau est libéré ; le délai de démarrage interactif reste dans la marge mesurée en 1b.
- **Phase 8** : fermer l'onglet pendant une génération entraîne, dans le délai borné par les pings SSE, l'arrêt côté Python et Ollama et une tentative `INTERRUPTED`.

### 6.5 Embeddings candidats (alternance uniquement)

La phase 1b utilise un ou deux modèles d'embedding candidats **uniquement pour mesurer le coût de l'alternance** avec le modèle conversationnel (déchargement, chargement, impact sur le délai avant le premier mot). Ces mesures ne valent pas sélection : le choix du modèle d'embedding, de la dimension et de l'index reste une décision de la phase KnowledgeBase (phase 9).

### 6.6 Mémoire, disque et nettoyage

- Surveiller RAM et espace disque pendant le banc.
- Conserver les résultats pour documenter le choix final.
- Exécuter `ollama rm` sur les modèles éliminés juste après les mesures (H5).
- Les scénarios de qualité seront réutilisés comme première version de l'`EvaluationSuite` en phase 13.

## 7. Roadmap (phases 0 à 14)

| Phase | Contenu | Dépend de |
|---|---|---|
| 0 | Décisions, corrections documentaires, contrats v1 documentés (`00`–`09`) | — |
| 1 | Socle technique (Docker, Laravel, Vue, PostgreSQL, Python, Filament, premier contrat) | 0 |
| 1b | Banc des modèles conversationnels (B1) | 1 (Ollama) |
| 2 | Identité et accès + sauvegarde basique externe | 1 |
| 3 | Socle partagé : UUID, horloge, transactions, outbox, idempotence, opérations asynchrones, nettoyage des logs | 1 |
| 4 | UserPractice : journal, préférences, objectifs | 2, 3 |
| 5 | TccContent : exercices, taxonomie, versions, publication, Filament | 3 |
| 6 | UserPractice : sessions d'exercices, réponses, conditions d'affichage | 4, 5 |
| 7 | Python + Ollama + AiGovernance minimal + sécurité déterministe (`10-PROTOCOLE-SECURITE.md`) + ordonnanceur (préemption) | 3, 1b |
| 8 | Companion : conversations, streaming, annulation, régénération, déconnexion de bout en bout | 7 |
| 9 | KnowledgeBase : ingestion, recherche hybride, RAG dans Companion, choix d'embedding | 7, 8 |
| 10 | PersonalContext : mémoires manuelles, propositions IA, assemblage du contexte | 4, 8 |
| 11 | Parcours personnels et éditoriaux, liens avec les objectifs | 5, 6 |
| 12 | Bilans de progression | 6, 10, 11 |
| 13 | Évaluations IA et validation obligatoire avant activation d'une release | 7, 8 |
| 14 | PlatformOperations : export, orchestration des suppressions, restauration, notifications | toutes |

Companion n'arrive pas en premier : il dépend d'AiGovernance, de la sécurité et de Python. Les phases 2 à 6 valident la chaîne DDD sans IA.

## 8. Décisions ouvertes

Aucune ne bloque la Phase 0. Chacune est rattachée à sa phase.

| Phase | Décision |
|---|---|
| 1 | Versions exactes des frameworks et dépendances, compatibilité PHP 8.4 |
| 1 | Conservation ou non de shadcn-vue, selon sa cohérence avec Filament (D5) |
| 1 | CI distante ou tests uniquement en local |
| 1b | Modèles conversationnels par usage et seuils de vitesse définitifs (D14) |
| 2 | Support externe de sauvegarde : périphérique, montage, rétention, emplacement de la clé |
| 7 | Contenu du protocole de sécurité (`10-PROTOCOLE-SECURITE.md`), rédigé avant Companion |
| 8 | Durée de conservation des tentatives de génération, au-delà de la purge à 7 jours du contenu partiel |
| 9 | Modèle d'embedding, dimension du vecteur, type d'index, seuil du mode `auto` |
| 9 | Classement des embeddings d'ingestion parmi les traitements préemptables |
| 10 | Déclenchement de l'extraction de mémoire ; conversations comme source autorisée ou non par défaut |
| Implémentation | Index JSONB, partitionnement, valeurs fines des timeouts, nommage des routes, signatures exactes des contrats synchrones |
| Implémentation | Transport des citations documentaires dans le flux NDJSON de génération |

## 9. Périmètre hors Phase 0

- Aucun code, configuration, Docker ni `README.md` modifié en Phase 0.
- Pas de dossier `contracts/` ni `docs/adr/` en Phase 0.
- `10-PROTOCOLE-SECURITE.md` : phase 7.

---

**Statut :** registre des décisions — version de travail Phase 0.

**Documents liés :** `01` à `09` ; `10-PROTOCOLE-SECURITE.md` (à venir).

**Changements v2 :** création du registre (D1–D15, H1–H6, P1–P11, B1), roadmap 0–14 dont 1b, protocole du banc, décisions ouvertes (Phase 0).
