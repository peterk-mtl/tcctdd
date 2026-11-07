# 07 — Contrats interservices et événements d'intégration

## 1. Objet, périmètre et dépendances

Ce document définit :

- les contrats synchrones entre les modules Laravel (bounded contexts) ;
- les événements métier et d'intégration, l'enveloppe commune et le mécanisme d'outbox ;
- les contrats entre Laravel et le service Python (`ai-engine/`) ;
- les règles transverses : opérations asynchrones, suppression, transactions et tests obligatoires.

Il couvre :

- Les principes d'autorité et de séparation des responsabilités.
- La propriété des données par contexte.
- Les contrats synchrones intercontextes.
- Les événements, leur enveloppe, l'outbox et l'idempotence des consommateurs.
- Le versionnement de l'API interne `/v1`, le format des erreurs, l'ordonnancement des générations et les endpoints.
- Les séquences critiques (conversation, ingestion, traitement de fond, suppression).
- L'idempotence, les timeouts et les règles de transaction.

Il ne remplace pas le modèle métier détaillé des bounded contexts (`04-BOUNDED-CONTEXTS.md`) ni le dictionnaire PostgreSQL (`06-DICTIONNAIRE-DONNEES-POSTGRESQL.md`).

**Dépendances :** `00-REGISTRE-DECISIONS.md`, `02-CAHIER-DES-CHARGES.md`, `03-ARCHITECTURE-DDD.md`, `04-BOUNDED-CONTEXTS.md`, `06-DICTIONNAIRE-DONNEES-POSTGRESQL.md`.

## 2. Principes

### 2.1 Autorité et séparation

- TCCTDD est un monolithe modulaire Laravel. Les sept contextes métier sont des modules internes au même système, pas des microservices.
- Laravel est l'autorité métier : autorisations, construction des prompts et du contexte, décisions de publication et d'activation documentaire.
- Python est un moteur technique : extraction, chunking, embeddings, recherche documentaire, orchestration Ollama. Il ne possède aucune donnée métier utilisateur.
- Le service Python n'a **aucun accès** aux schémas métier PostgreSQL (lecture et écriture). Il possède exclusivement le schéma `ai_engine`.
- Les communications Laravel ↔ Python transitent sur le réseau Docker interne, authentifiées par un jeton porteur partagé.
- Le navigateur ne communique jamais avec Python. Le streaming visible côté client est toujours relayé par Laravel (SSE).
- Aucun contexte Laravel ne modifie directement les tables métier d'un autre contexte.
- Les journaux techniques ne contiennent aucun contenu personnel (messages, journaux, mémoires, documents utilisateurs).

### 2.2 Mécanismes de collaboration

Les communications entre contextes reposent sur trois mécanismes :

1. **Contrats synchrones** (`Contracts/` du contexte fournisseur) pour obtenir immédiatement une information ou exécuter une opération nécessaire.
2. **Événements** pour notifier un fait accompli et déclencher des réactions indépendantes.
3. **Files de travaux** (jobs Laravel) pour les opérations longues, différées ou coûteuses.

Un événement indique qu'un fait s'est produit. Il ne sert jamais à transmettre une commande déguisée à un autre contexte.

### 2.3 Propriété des données

| Contexte | Données possédées | Responsabilité principale |
|---|---|---|
| Companion | Conversations, messages, tentatives de génération, instantanés de contexte | Cycle de vie des échanges |
| PersonalContext | Mémoires, versions, sources, relations (dont `CONTRADICTS`) | Mémoire personnelle |
| TccContent | Exercices, versions, parcours éditoriaux, ressources | Contenu TCC validé |
| UserPractice | Réalisations, réponses, journal, objectifs, parcours personnels, bilans | Pratique personnelle |
| KnowledgeBase | Documents, versions, ingestions, activation pour la recherche (`ai_retrieval_enabled`) | Bibliothèque documentaire |
| AiGovernance | Modèles, prompts, politiques, releases, évaluations, protocole de sécurité | Gouvernance IA |
| PlatformOperations | Sauvegardes, exports, suppressions et registre des suppressions, notifications, maintenance, outbox | Exploitation |

Le service Python possède exclusivement les données techniques du schéma `ai_engine` : extractions, sections, chunks, embeddings, index techniques et traces de recherche techniques. Il ne possède aucune donnée métier relative aux utilisateurs.

## 3. Contrats synchrones intercontextes

Les contrats sont définis par le contexte fournisseur dans son répertoire `Contracts/` et exposés sous une interface stable. Les données échangées sont des DTO typés et immuables.

Le consommateur ne connaît pas les détails de persistance du fournisseur et n'appelle jamais ses repositories.

Les noms ci-dessous constituent l'inventaire initial. Les signatures exactes sont fixées à l'implémentation, sans en retirer les garanties décrites.

### 3.1 Companion et PersonalContext

**`AssembleCompanionContext`**

Entrée :

- identifiant utilisateur ;
- identifiant de conversation ;
- identifiant du message ;
- release IA active ;
- budget de tokens ;
- stratégie de contextualisation.

Sortie :

- identifiant de l'instantané de contexte ;
- références et versions des mémoires retenues ;
- relations pertinentes, dont les contradictions (`CONTRADICTS`) ;
- préférences comportementales ;
- références de pratique autorisées ;
- références documentaires éventuellement disponibles ;
- estimation du budget utilisé ;
- version de l'assembleur.

Règles :

- sélectionner uniquement les informations utiles ;
- ne retenir que des mémoires `ACTIVE` ; une mémoire `PROPOSED` n'est jamais utilisée ;
- vérifier les droits d'accès et respecter les exclusions explicites ;
- ne pas charger toutes les mémoires ;
- ne pas copier de contenu privé dans les traces techniques.

### 3.2 Companion et UserPractice

**`GetAuthorizedPracticeContext`**

- Entrée : identifiant utilisateur, références des réalisations ou entrées demandées, usage prévu.
- Sortie : données strictement autorisées, identifiants et versions des sources, indicateurs d'autorisation, exclusions éventuelles.

Une entrée de journal n'est accessible que si sa politique effective l'autorise. Une entrée interdite à l'IA n'est jamais récupérée, même indirectement.

**`CanAccessJournalEntry`**

- Entrée : identifiant utilisateur, identifiant de l'entrée, usage demandé.
- Sortie : autorisation (`allowed`), politique effective, code de raison.

**`CanExtractMemoryFromSource`**

Contrat distinct de la consultation IA. Une autorisation de consultation ne donne jamais automatiquement l'autorisation d'extraire une mémoire durable.

### 3.3 Companion et TccContent

**`GetPublishedExerciseVersion`**

- Entrée : identifiant d'exercice, éventuellement identifiant de version.
- Sortie : version publiée, contenu structuré, questions, conditions, précautions, niveau de preuve, références associées.

Une ancienne réalisation conserve toujours sa version d'origine.

### 3.4 Companion et KnowledgeBase

**`GetAuthorizedKnowledgeDocuments`**

- Entrée : identifiant utilisateur, usage, filtres éventuels.
- Sortie : `allowed_document_version_ids` (versions actives dont `ai_retrieval_enabled` est vrai) et métadonnées nécessaires à la recherche et à la citation.

Le service Python ne décide jamais lui-même des autorisations métier : il reçoit cette liste et filtre la recherche en conséquence.

### 3.5 Companion et AiGovernance

**`GetActiveAIConfiguration`**

Sortie : release active, versions exactes des composants, configuration modèle par usage, prompts applicables, politiques, limites de ressources, version des règles de sécurité.

**`AssessSafetySignals`**

- Entrée : message ou signal structuré, contexte minimal nécessaire, version de politique applicable.
- Sortie : niveau de risque, catégories de signaux, protocole à déclencher, nécessité d'une orientation immédiate.

Règles :

- L'évaluation est déterministe et exécutée dans Laravel, avant tout appel au modèle.
- En cas de danger imminent, la réponse du protocole est produite sans appel au modèle.
- Une analyse assistée par le modèle peut compléter l'évaluation ; elle ne peut jamais abaisser le niveau déterminé par les règles déterministes.

**`DetermineRetrievalStrategy`**

- Entrée : mode de recherche configuré (`never`, `always`, `auto`), type de demande, indicateurs structurés.
- Sortie : `NOT_NEEDED`, `RECOMMENDED` ou `REQUIRED`, avec une justification technique.

Règles :

- `never` donne toujours `NOT_NEEDED` ; `always` donne toujours `REQUIRED`.
- En mode `auto`, la décision repose sur des règles et indicateurs structurés, sans appel LLM supplémentaire.
- L'arbitrage final appartient à la politique de gouvernance, pas au modèle.

### 3.6 UserPractice et TccContent

**`GetExerciseVersionForRun`**

- Entrée : identifiant d'exercice, identifiant de version.
- Sortie : version exacte, questions, types de réponses, configuration, conditions d'affichage.

**`ValidateExerciseAnswer`**

- Entrée : identifiant de version, clé de question (stable au sein de la version), valeur de réponse, contexte de réalisation.
- Sortie : résultat de validation, erreurs éventuelles, normalisation éventuelle.

La validation s'appuie sur le contenu publié et non sur une logique dupliquée dans le frontend.

## 4. Événements

### 4.1 Événements métier et événements d'intégration

- Un **événement métier** est nommé au passé en PascalCase dans le code du contexte producteur (par exemple `ConversationDeleted`).
- Un événement métier devient un **événement d'intégration** lorsqu'un autre contexte doit y réagir. Il est alors enregistré dans `platform.outbox_events` sous le type `<contexte>.<fait_au_passé>` (par exemple `companion.conversation_deleted`).
- Seuls les événements ayant au moins un consommateur sont publiés dans l'outbox. Les autres restent internes au contexte.
- Aucun événement ne transporte de contenu personnel : messages, entrées de journal, réponses, mémoires et documents ne figurent jamais dans le payload, seulement des identifiants.

### 4.2 Événements par contexte

Inventaire initial des événements métier. Les événements réellement publiés sont identifiés à l'implémentation, selon les consommateurs effectifs.

**Companion :** `ConversationCreated`, `MessageSubmitted`, `GenerationStarted`, `GenerationCompleted`, `GenerationInterrupted`, `GenerationFailed`, `ConversationArchived`, `ConversationRestored`, `ConversationDeleted`.

**PersonalContext :** `MemoryProposed`, `MemoryCreated`, `MemoryValidated`, `MemoryRejected`, `MemoryRevised`, `MemoryArchived`, `MemoryRelationProposed` (notamment une contradiction `CONTRADICTS`), `MemoryConsolidationProposed`, `MemoryConsolidationCompleted`.

**TccContent :** `ExercisePublished`, `ExerciseArchived`, `EditorialPathwayPublished`, `ResourceVersionPublished`.

L'activation d'une ressource pour la recherche IA ne relève pas de TccContent : elle est portée par KnowledgeBase (`DocumentRetrievalEnabled`, `DocumentRetrievalDisabled`).

**UserPractice :** `ExerciseCompleted`, `ExerciseAbandoned`, `ExerciseRunDeleted`, `JournalEntryCreated`, `JournalEntryUpdated`, `JournalEntryArchived`, `JournalEntryRestored`, `JournalEntryDeleted`, `PersonalGoalUpdated`, `PathwayCompleted`, `ProgressReviewRequested`, `ProgressReviewValidated`.

Les événements liés au journal ne transportent jamais le contenu de l'entrée.

**KnowledgeBase :** `KnowledgeDocumentApproved`, `KnowledgeDocumentVersionCreated`, `DocumentRetrievalEnabled`, `DocumentRetrievalDisabled`, `DocumentIndexationRequested`, `DocumentVersionActivated`, `DocumentVersionArchived`.

**AiGovernance :** `AIConfigurationReleaseCreated`, `AIConfigurationReleaseEvaluated`, `AIConfigurationReleaseApproved`, `AIConfigurationReleaseActivated`, `AIConfigurationReleaseRolledBack`, `SafetyProtocolTriggered`.

`SafetyProtocolTriggered` reste un événement interne à AiGovernance : les interventions de sécurité sont des données sensibles et ne sont jamais publiées comme événements d'intégration.

**PlatformOperations :** `BackupCompleted`, `BackupFailed`, `RestoreCompleted`, `ExportRequested`, `ExportCompleted`, `UserDataDeletionRequested`, `UserDataDeletedInContext`, `DeletionCompleted`, `NotificationScheduled`, `MaintenanceCompleted`.

### 4.3 Catalogue initial des événements d'intégration

| Type | Émetteur | Intention |
|---|---|---|
| `companion.conversation_deleted` | Companion | Purger dérivés liés à une conversation |
| `user_practice.journal_entry_deleted` | UserPractice | Purger dérivés liés à une entrée de journal |
| `user_practice.exercise_run_deleted` | UserPractice | Purger dérivés liés à une réalisation |
| `tcc_content.resource_version_published` | TccContent | Permettre à KnowledgeBase d'indexer une ressource publiée |
| `knowledge_base.document_version_archived` | KnowledgeBase | Purger index techniques associés |
| `platform.user_data_deletion_requested` | PlatformOperations | Déclencher la suppression transverse |
| `platform.user_data_deleted_in_context` | Chaque contexte | Accuser la suppression locale terminée |

Tout nouvel événement d'intégration est ajouté à ce catalogue avec son émetteur, son intention et ses consommateurs.

### 4.4 Enveloppe

L'enveloppe correspond aux colonnes de `platform.outbox_events` :

| Champ | Type | Description |
|---|---|---|
| `event_id` | UUID | Identifiant unique de l'événement |
| `event_type` | VARCHAR | Type `<contexte>.<fait_au_passé>` |
| `event_version` | INTEGER | Version du schéma de payload |
| `producer` | VARCHAR | Contexte producteur |
| `aggregate_type` | VARCHAR | Type d'agrégat |
| `aggregate_id` | UUID | Identifiant de l'agrégat |
| `aggregate_version` | BIGINT | Version métier de l'agrégat, si applicable |
| `occurred_at` | TIMESTAMPTZ | Date du fait |
| `correlation_id` | UUID | Traçabilité de l'opération |
| `causation_id` | UUID | Événement ou commande déclencheur, si applicable |
| `payload` | JSONB | Données minimales nécessaires (identifiants) |
| `metadata` | JSONB | Métadonnées techniques |

Les noms de propriétés des payloads sont cohérents et versionnés. Un changement incompatible de payload incrémente `event_version`.

Exemple conceptuel, `user_practice.exercise_completed` :

- `producer` = UserPractice ;
- `aggregate_type` = ExerciseRun ;
- `payload` : identifiant utilisateur, identifiant d'exercice, identifiant de version d'exercice, identifiant de réalisation, date d'achèvement.

Aucune réponse personnelle n'est incluse.

## 5. Outbox et consommateurs

### 5.1 Écriture transactionnelle

Lorsqu'une opération métier produit un événement d'intégration :

1. l'état métier est modifié ;
2. l'événement est enregistré dans `platform.outbox_events` ;
3. les deux écritures sont validées dans la même transaction PostgreSQL.

Un événement ne peut donc pas être publié si la transaction métier échoue.

### 5.2 Dispatcher

Un processus dédié :

- récupère les événements non publiés ;
- les publie vers les consommateurs internes ;
- marque leur publication (`published_at`) ;
- réessaie en cas d'erreur temporaire ;
- conserve les erreurs techniques nécessaires au diagnostic, sans contenu privé.

La publication peut se produire plusieurs fois : les consommateurs doivent être idempotents.

Le dispatcher reste simple et interne au monolithe : aucun bus événementiel externe ni infrastructure de microservices n'est introduit.

### 5.3 Idempotence des consommateurs

Chaque consommateur utilise `platform.processed_events`, de clé `(consumer, event_id)`.

Le traitement et l'enregistrement de la consommation sont validés dans la même transaction. Un événement déjà traité par un consommateur ne provoque jamais une seconde mutation métier.

### 5.4 Erreurs

| Nature | Traitement |
|---|---|
| Erreur temporaire | Nouvelle tentative avec backoff |
| Erreur permanente | Mise en échec et intervention |
| Refus métier | Pas de nouvelle tentative automatique |
| Événement invalide | Rejet et journalisation technique expurgée |

Les erreurs n'exposent jamais de contenu privé.

## 6. Opérations asynchrones

`platform.async_operations` assure le suivi transversal des opérations longues : ingestion documentaire, indexation, génération de bilan, export, suppression de compte, restauration.

Elle fournit : identifiant, contexte propriétaire, type d'opération, ressource associée, statut global, progression technique, code d'erreur normalisé et identifiant de corrélation.

Elle ne remplace jamais les statuts métier propres aux processus. Par exemple, l'ingestion conserve son propre statut dans KnowledgeBase et un bilan conserve son statut dans UserPractice ; l'opération asynchrone ne fait que présenter le suivi global.

## 7. API interne Laravel ↔ Python

### 7.1 Rôles

Laravel :

- authentifie l'utilisateur et vérifie les droits ;
- assemble les données autorisées et construit le prompt ;
- transmet une requête minimale ;
- persiste l'état métier ;
- expose le résultat au frontend.

Python :

- orchestre les appels à Ollama ;
- réalise la recherche documentaire dans les versions autorisées ;
- traite les documents ;
- renvoie des résultats techniques.

Python ne reçoit jamais de privilège lui permettant d'élargir les droits d'accès.

### 7.2 Versionnement `/v1`

- Le préfixe `/v1` est stable.
- Les évolutions compatibles se limitent à des ajouts de champs **optionnels**.
- Les champs inconnus sont ignorés par le récepteur.
- Tout changement incompatible nécessite `/v2`.

Source de vérité des contrats HTTP :

- Modèles Pydantic du service Python.
- Instantané OpenAPI exporté dans `contracts/v1/` (créé en phase 1).
- Fixtures JSON dans `contracts/v1/fixtures/`.

Les fixtures sont vérifiées des deux côtés :

- Tests Python : acceptation des payloads valides, rejet des invalides, absence de dérive OpenAPI.
- Tests Laravel : sérialisation et lecture des DTO correspondants.

### 7.3 En-têtes et statuts

| En-tête | Rôle |
|---|---|
| `X-Request-Id` | Identifiant unique de la requête HTTP |
| `X-Correlation-Id` | Corrélation bout en bout (conversation, ingestion, suppression) |
| `Idempotency-Key` | Clé d'idempotence lorsque l'opération le requiert |

Les statuts HTTP portent l'issue de la requête : `202` indique qu'un traitement long est accepté, pas qu'il est terminé ; `200` indique un résultat disponible ; les erreurs suivent le format du paragraphe 7.4.

### 7.4 Format d'erreur

```json
{
  "error": {
    "code": "ENGINE_BUSY",
    "message": "An inference slot is already occupied by an interactive request.",
    "retryable": true,
    "details": {}
  }
}
```

`message` et `details` ne doivent contenir aucune donnée personnelle.

### 7.5 Codes d'erreur

| Code | Signification |
|---|---|
| `INVALID_REQUEST` | Payload invalide |
| `UNSUPPORTED_CONTRACT_VERSION` | Version de contrat non supportée |
| `UNAUTHORIZED` | Jeton manquant ou invalide |
| `NOT_FOUND` | Ressource technique introuvable |
| `MODEL_UNAVAILABLE` | Modèle Ollama demandé indisponible |
| `ENGINE_UNAVAILABLE` | Service Python ou Ollama indisponible |
| `ENGINE_BUSY` | Créneau d'inférence occupé par une autre requête interactive |
| `PREEMPTED` | Traitement de fond interrompu au profit d'une requête interactive |
| `GENERATION_CANCELLED` | Annulation explicite |
| `TIMEOUT` | Délai dépassé |
| `UNSUPPORTED_FORMAT` | Format de document non supporté |
| `DOCUMENT_UNREADABLE` | Document illisible ou extraction impossible |
| `CHECKSUM_MISMATCH` | Checksum du fichier différent de celui attendu |
| `INVALID_STORAGE_KEY` | Clé de stockage invalide ou hors racine autorisée |
| `INTERNAL_ERROR` | Erreur interne non classifiée |

Le debug Ollama est désactivé en fonctionnement normal.

### 7.6 Sécurité des échanges

- Authentification de service à service par jeton porteur.
- Réseau Docker privé.
- Validation stricte des contrats.
- Limites de taille des requêtes et des réponses.
- Timeouts (paragraphe 10).
- Idempotence (paragraphe 10).
- Aucune confiance accordée aux identifiants renvoyés par Python sans validation par Laravel.

## 8. Ordonnancement des générations

- Un seul créneau d'inférence est disponible.
- Deux priorités : `interactive` et `background`.
- Une génération de conversation est toujours `interactive` et n'est **jamais** préemptée.
- Une requête `interactive` qui arrive alors qu'une autre requête `interactive` occupe le créneau reçoit `ENGINE_BUSY`. Côté Companion, la tentative échoue avec ce code, sans relance automatique ; l'utilisateur peut régénérer.
- Seuls les traitements de fond sont `background` et préemptables : extraction et consolidation de mémoires (PersonalContext), génération de bilans et de propositions de parcours (UserPractice), évaluations (AiGovernance). La liste de référence figure dans `04-BOUNDED-CONTEXTS.md`.
- Une requête `interactive` préempte un traitement `background` en cours. Python répond `PREEMPTED`.
- Une tentative préemptée prend le statut terminal `PREEMPTED`. Elle n'est jamais reprise : la relance crée une **nouvelle tentative**, avec un nouveau `generation_id`, orchestrée par Laravel.
- L'agrégat métier concerné conserve son propre statut pendant la relance (par exemple, un bilan reste `GENERATING`).
- `OLLAMA_MAX_LOADED_MODELS=1` est la configuration de départ ; toute autre stratégie d'alternance est mesurée au banc (phase 1b), pas présumée.

## 9. Endpoints v1

| Méthode | Chemin | Rôle |
|---|---|---|
| `GET` | `/v1/health` | Santé du service et d'Ollama |
| `GET` | `/v1/models` | Modèles disponibles |
| `POST` | `/v1/generations/stream` | Génération en flux NDJSON |
| `POST` | `/v1/generations/{id}/cancel` | Annulation d'une génération |
| `POST` | `/v1/completions` | Complétion synchrone (`response_format` éventuel) |
| `POST` | `/v1/retrieval/search` | Recherche documentaire |
| `POST` | `/v1/ingestions` | Démarrage d'ingestion (202) |
| `GET` | `/v1/ingestions/{id}` | État d'une ingestion |
| `DELETE` | `/v1/document-versions/{id}` | Purge technique d'une version documentaire |
| `GET` | `/v1/chunks/{id}` | Lecture d'un chunk technique |

Il n'existe pas d'endpoint d'activation documentaire ni d'évaluation métier dédié. Laravel décide des versions autorisées et transmet `allowed_document_version_ids` à la recherche.

### 9.1 Flux NDJSON de génération

Événements possibles : `start`, `ping` (toutes les 15 s), `delta`, `end` (avec métriques), `error`.

Ces événements de transport ne remplacent pas les événements métier persistés.

Règles de relais :

- Laravel relaie le flux sans le mettre intégralement en mémoire.
- Les fragments générés sont associés à la tentative en cours.
- La finalisation est explicite : `COMPLETED` (réponse terminée et sélectionnable), `INTERRUPTED` (texte incomplet, jamais présenté comme réponse achevée), `FAILED` (génération échouée), `CANCELLED` (annulation confirmée).
- Une reconnexion du navigateur ne crée jamais une seconde génération involontaire.

### 9.2 Recherche documentaire

- Entrée : requête, `allowed_document_version_ids`, paramètres techniques.
- La requête utilisateur n'est ni stockée ni journalisée côté Python.
- Modes métier côté Laravel : `never`, `always`, `auto` (paragraphe 3.5). En mode `auto`, la décision n'implique pas d'appel LLM supplémentaire.

## 10. Idempotence et timeouts

| Opération | Règle |
|---|---|
| Ingestion | `ingestion_id` est la clé d'idempotence |
| Génération | `generation_id` à usage unique ; un rejeu avec le même id produit 409 |
| Completions | Sans état côté Python |

Timeouts initiaux :

| Opération | Timeout |
|---|---|
| Santé | 2 s |
| Recherche | 30 s |
| Completions | 600 s |
| Flux sans événement | 60 s (les pings comptent comme événements) |

## 11. Séquences

### 11.1 Conversation

1. Laravel reçoit le message utilisateur et persiste le message.
2. Sécurité déterministe dans Laravel (`AssessSafetySignals`) : en cas de danger imminent, réponse du protocole de sécurité **sans** appel au modèle. Le protocole détaillé figure dans `10-PROTOCOLE-SECURITE.md` (phase 7).
3. Si la génération est autorisée : recherche documentaire selon le mode (`always` ou `auto` le cas échéant).
4. Laravel construit le prompt et le contexte (historique, mémoires autorisées, extraits documentaires, release active).
5. Laravel appelle `POST /v1/generations/stream` en priorité `interactive` et relaie en SSE vers le navigateur. Si le créneau est occupé par une autre requête interactive, la tentative échoue avec `ENGINE_BUSY`.
6. Déconnexion du navigateur : détectée à chaque écriture SSE, pings compris. Laravel ferme le flux, appelle `cancel`, marque la tentative `INTERRUPTED`.
7. Si le flux HTTP Laravel ↔ Python se ferme sans `cancel` explicite, Python interrompt aussi Ollama. Aucune génération orpheline.

### 11.2 Ingestion

Laravel fournit une `storage_key` relative, un checksum et les métadonnées nécessaires. Le volume documentaire est monté en lecture seule dans Python.

Résolution de `storage_key` :

- Seule la racine autorisée vient de la configuration.
- La clé est relative ; un chemin absolu est rejeté (`INVALID_STORAGE_KEY`).
- Chaque composant du chemin, répertoires intermédiaires compris, est parcouru depuis le descripteur de la racine **sans suivre aucun lien symbolique**, à aucun niveau.
- Rejet `INVALID_STORAGE_KEY` pour : composant vide, `.`, `..`, lien symbolique, fichier final non régulier, chemin hors racine.
- Type, taille, checksum, lecture et extraction s'effectuent sur le **même descripteur**, sans réouverture par chemin (protection TOCTOU).
- Le mécanisme système exact (par exemple `openat` / `O_NOFOLLOW`) est laissé à l'implémentation, sous réserve du respect de ces invariants.

L'état métier de l'ingestion (`PENDING`, `PROCESSING`, `COMPLETED`, `PARTIAL`, `FAILED`, `UNSUPPORTED`) est tenu par KnowledgeBase à partir des états techniques renvoyés par Python.

### 11.3 Traitement de fond

1. Laravel crée une tentative et appelle Python en priorité `background`, avec un `generation_id` neuf.
2. Si une requête interactive arrive, Python interrompt le traitement et répond `PREEMPTED`.
3. Laravel marque la tentative `PREEMPTED` (statut terminal).
4. Laravel planifie, avec un délai, une nouvelle tentative portant un nouveau `generation_id`, sans perdre l'idempotence métier de l'opération.

### 11.4 Suppression d'un élément

Après suppression logique métier, Laravel orchestre la purge des dérivés techniques Python via `DELETE /v1/document-versions/{id}` lorsque applicable, et via les événements d'intégration pour les autres contextes.

## 12. Suppression transverse

La suppression des données d'un utilisateur illustre la coordination entre contextes. PlatformOperations :

1. inscrit durablement la demande confirmée dans le registre des suppressions ;
2. accuse réception de la suppression — jamais avant l'étape 1 ;
3. bloque les nouvelles opérations et rend les données inaccessibles (suppression logique) ;
4. crée une tâche de suppression par contexte (`deletion_tasks`) et publie `platform.user_data_deletion_requested` ;
5. déclenche les traitements idempotents ; chaque contexte supprime ses propres données et publie `platform.user_data_deleted_in_context` ;
6. vérifie les résultats, inscrit l'achèvement dans le registre et clôture la demande.

Règles :

- Chaque contexte supprime ses propres données ; aucun contexte ne supprime directement les tables d'un autre.
- Le service Python supprime ses données techniques dérivées sur instruction de Laravel.
- Les sauvegardes historiques suivent leur politique de rétention. Après restauration, les suppressions inscrites au registre postérieurement à la sauvegarde sont rejouées de manière idempotente avant toute remise en service.

## 13. Règles de transaction

Les transactions PostgreSQL restent courtes.

Aucune transaction n'est maintenue ouverte pendant :

- un appel HTTP au service Python ;
- une génération Ollama ;
- une recherche documentaire longue ;
- une extraction documentaire ;
- un transfert de fichier ;
- un traitement asynchrone.

La coordination repose sur des transitions d'état, des opérations idempotentes et l'outbox. Aucune transaction distribuée n'est requise.

## 14. Tests obligatoires

### 14.1 Contrats synchrones

- Validation des entrées et sorties.
- Droits d'accès.
- Absence de dépendance aux tables internes du fournisseur.
- Compatibilité des versions de contrats.

### 14.2 Événements

- Écriture atomique de l'état métier et de l'outbox.
- Publication après validation de la transaction.
- Publication répétée.
- Idempotence des consommateurs.
- Gestion des erreurs.
- Compatibilité des versions.
- Absence de contenu personnel dans les payloads.

### 14.3 Service Python

- Service indisponible.
- Timeout.
- Réponse invalide.
- Rejeu d'une requête (409 sur un `generation_id` réutilisé).
- Interruption du streaming et déconnexion du navigateur.
- Annulation.
- `ENGINE_BUSY` pour une seconde requête interactive.
- Préemption d'un traitement de fond et relance par une nouvelle tentative.
- Limites de taille.
- Résolution de `storage_key` (liens symboliques, `..`, chemins absolus).
- Fixtures de contrat vérifiées des deux côtés.
- Absence de fuite de données.

### 14.4 Suppression

- Inscription au registre avant l'accusé de réception.
- Reprise après panne.
- Double exécution.
- Suppression partielle.
- Vérification finale.
- Restauration et rejeu du registre.

## 15. Principes d'implémentation

1. Définir les interfaces de contrats dans le contexte fournisseur (`Contracts/`).
2. Définir les DTO d'échange.
3. Identifier les événements réellement nécessaires.
4. Créer l'enveloppe et les mécanismes communs.
5. Implémenter l'outbox et l'idempotence.
6. Documenter les consommateurs.
7. Tester les frontières entre contextes.
8. Ne jamais appeler les repositories d'un autre contexte.
9. Ne jamais inclure de contenu privé dans un événement.
10. Documenter toute évolution de contrat dans ce document.

Ne pas créer de bus événementiel complexe ni d'infrastructure de microservices pour un besoin traité simplement dans le monolithe modulaire. La priorité est la fiabilité, la traçabilité technique et le faible couplage.

## 16. Points ouverts

Les sujets non tranchés (signatures exactes des contrats synchrones, détail du protocole de sécurité, choix définitifs de modèles et d'embeddings, classement des embeddings d'ingestion parmi les traitements préemptables, transport des citations documentaires dans le flux de génération) sont consignés dans `00-REGISTRE-DECISIONS.md`.

---

**Changements v2 :** création du document de contrats interservices et d'événements d'intégration ; reprise des contrats intercontextes de l'ancien catalogue (mécanismes, propriété des données, contrats synchrones, événements par contexte, enveloppe, outbox, opérations asynchrones, suppression transverse, règles de transaction, tests obligatoires) adaptés aux décisions de la Phase 0 ; ordonnancement aligné sur une conversation jamais préemptée et des traitements de fond préemptables (Phase 0).

**Dépendances :** `00-REGISTRE-DECISIONS.md`, `02-CAHIER-DES-CHARGES.md`, `03-ARCHITECTURE-DDD.md`, `04-BOUNDED-CONTEXTS.md`, `06-DICTIONNAIRE-DONNEES-POSTGRESQL.md`.
