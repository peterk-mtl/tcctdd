# TCCTDD — Contrats intercontextes et architecture événementielle

**Version :** 1.0  
**Statut :** Référence d'intégration entre les contextes métier  
**Dépendances :** Architecture DDD, dictionnaire PostgreSQL et catalogue des cas d'utilisation.

## 1. Principes fondamentaux

TCCTDD est un **monolithe modulaire Laravel**, accompagné d'un service Python spécialisé dans le traitement documentaire et l'orchestration IA.

Les sept contextes métier restent des modules internes au même système Laravel. Ils ne constituent pas sept microservices.

Le service Python est un composant technique séparé, pas un propriétaire des données métier.

Les communications entre contextes reposent sur trois mécanismes :

1. **Contrats synchrones** pour obtenir immédiatement une information ou exécuter une opération nécessaire.
2. **Événements métier** pour notifier un changement et déclencher des réactions indépendantes.
3. **Files de travaux** pour les opérations longues, différées ou coûteuses.

Aucun contexte ne doit modifier directement les tables métier d'un autre contexte.

## 2. Responsabilités et propriété des données

| Contexte | Données possédées | Responsabilité principale |
|---|---|---|
| Companion | Conversations, messages, tentatives | Cycle de vie des échanges |
| PersonalContext | Mémoires, relations, conflits | Mémoire personnelle |
| TccContent | Exercices, versions, parcours éditoriaux, ressources | Contenu TCC validé |
| UserPractice | Réalisations, réponses, journal, objectifs, parcours personnels | Pratique personnelle |
| KnowledgeBase | Documents, versions, ingestion, indexation métier | Bibliothèque documentaire |
| AiGovernance | Modèles, prompts, politiques, releases, évaluations | Gouvernance IA |
| PlatformOperations | Sauvegardes, exports, suppressions, notifications, maintenance | Exploitation |

Le service Python possède exclusivement les données techniques du schéma `ai_engine` :
- extractions ;
- sections ;
- chunks ;
- embeddings ;
- index techniques ;
- traces de recherche.

Il ne possède aucune donnée métier relative aux utilisateurs.

## 3. Contrats synchrones intercontextes

Les contrats sont définis par le contexte fournisseur et exposés sous une interface stable.

Le consommateur ne connaît pas les détails de persistance du fournisseur.

### 3.1. Companion et PersonalContext

`AssembleCompanionContext`

Entrée :
- userId ;
- conversationId ;
- messageId ;
- active AI release ;
- token budget ;
- stratégie de contextualisation.

Sortie :
- contextSnapshotId ;
- références et versions des mémoires retenues ;
- relations pertinentes ;
- conflits pertinents ;
- préférences comportementales ;
- références de pratique autorisées ;
- références documentaires éventuellement disponibles ;
- estimation du budget utilisé ;
- version de l'assembleur.

Règles :
- sélectionner uniquement les informations utiles ;
- vérifier les droits d'accès ;
- respecter les exclusions explicites ;
- ne pas charger toutes les mémoires ;
- ne pas copier de contenu privé dans les traces techniques.

### 3.2. Companion et UserPractice

`GetAuthorizedPracticeContext`

Entrée :
- userId ;
- références des réalisations ou entrées demandées ;
- usage prévu.

Sortie :
- données strictement autorisées ;
- identifiants et versions des sources ;
- indicateurs d'autorisation ;
- exclusions éventuelles.

Une entrée de journal n'est accessible que si sa politique effective l'autorise.

`CanAccessJournalEntry`

Entrée :
- userId ;
- journalEntryId ;
- requestedUsage.

Sortie :
- allowed ;
- effectivePolicy ;
- reasonCode.

`CanExtractMemoryFromSource`

Contrat distinct de la consultation IA. Une autorisation de consultation ne donne jamais automatiquement l'autorisation d'extraire une mémoire durable.

### 3.3. Companion et TccContent

`GetPublishedExerciseVersion`

Entrée :
- exerciseId ;
- éventuellement exerciseVersionId.

Sortie :
- version publiée ;
- contenu structuré ;
- questions ;
- conditions ;
- précautions ;
- niveau de preuve ;
- références associées.

Une ancienne réalisation conserve toujours sa version d'origine.

### 3.4. Companion et KnowledgeBase

`GetAuthorizedKnowledgeDocuments`

Entrée :
- userId ;
- usage ;
- filtres éventuels.

Sortie :
- identifiants des documents autorisés ;
- versions actives ;
- métadonnées nécessaires à la recherche.

Le service Python ne décide jamais lui-même des autorisations métier.

### 3.5. Companion et AiGovernance

`GetActiveAIConfiguration`

Sortie :
- release active ;
- versions exactes des composants ;
- configuration modèle par usage ;
- prompts applicables ;
- politiques ;
- limites de ressources ;
- version des règles de sécurité.

`AssessSafetySignals`

Entrée :
- signal structuré ;
- contexte minimal nécessaire ;
- version de politique applicable.

Sortie :
- niveau de risque proposé ;
- catégories de signaux ;
- protocole à déclencher ;
- niveau de certitude ;
- nécessité d'une orientation immédiate.

Les règles déterministes de sécurité ne doivent pas dépendre uniquement de cette évaluation.

`DetermineRetrievalStrategy`

Entrée :
- type de demande ;
- contexte d'utilisation ;
- indicateurs structurés.

Sortie :
- NOT_NEEDED ;
- RECOMMENDED ;
- REQUIRED ;
- justification technique.

L'arbitrage final appartient à la politique de gouvernance, pas au seul modèle.

### 3.6. UserPractice et TccContent

`GetExerciseVersionForRun`

Entrée :
- exerciseId ;
- exerciseVersionId.

Sortie :
- version exacte ;
- questions ;
- types de réponses ;
- configuration ;
- conditions d'affichage.

`ValidateExerciseAnswer`

Entrée :
- exerciseVersionId ;
- questionId ;
- answerValue ;
- contexte de réalisation.

Sortie :
- résultat de validation ;
- erreurs éventuelles ;
- normalisation éventuelle.

La validation doit s'appuyer sur le contenu publié et non sur une logique dupliquée dans le frontend.

## 4. Événements métier

Un événement indique qu'un fait métier s'est produit.

Il ne doit pas servir à transmettre arbitrairement une commande à un autre contexte.

### 4.1. Événements Companion

- `ConversationCreated`
- `MessageSubmitted`
- `GenerationStarted`
- `GenerationCompleted`
- `GenerationInterrupted`
- `GenerationFailed`
- `ConversationArchived`
- `ConversationDeleted`

### 4.2. Événements PersonalContext

- `MemoryProposed`
- `MemoryCreated`
- `MemoryValidated`
- `MemoryRejected`
- `MemoryRevised`
- `MemoryArchived`
- `MemoryConflictDetected`
- `MemoryConsolidationProposed`
- `MemoryConsolidationCompleted`

### 4.3. Événements TccContent

- `ExercisePublished`
- `ExerciseArchived`
- `EditorialPathwayPublished`
- `ResourcePublished`
- `ResourceRetrievalEnabled`
- `ResourceRetrievalDisabled`

### 4.4. Événements UserPractice

- `ExerciseCompleted`
- `ExerciseAbandoned`
- `JournalEntryCreated`
- `JournalEntryUpdated`
- `JournalEntryDeleted`
- `PersonalGoalUpdated`
- `PathwayCompleted`
- `ProgressReviewRequested`
- `ProgressReviewValidated`

Les événements liés au journal ne transportent jamais le contenu de l'entrée.

### 4.5. Événements KnowledgeBase

- `KnowledgeDocumentApproved`
- `KnowledgeDocumentVersionCreated`
- `DocumentRetrievalEnabled`
- `DocumentRetrievalDisabled`
- `DocumentIndexationRequested`
- `DocumentVersionActivated`

### 4.6. Événements AiGovernance

- `AIConfigurationReleaseCreated`
- `AIConfigurationReleaseEvaluated`
- `AIConfigurationReleaseApproved`
- `AIConfigurationReleaseActivated`
- `AIConfigurationReleaseRolledBack`
- `SafetyProtocolTriggered`

### 4.7. Événements PlatformOperations

- `BackupCompleted`
- `BackupFailed`
- `RestoreCompleted`
- `ExportRequested`
- `ExportCompleted`
- `DeletionRequested`
- `DeletionCompleted`
- `NotificationScheduled`
- `MaintenanceCompleted`

## 5. Enveloppe standard d'un événement

Chaque événement publié respecte une enveloppe commune.

| Champ | Type | Description |
|---|---|---|
| eventId | UUID | Identifiant unique |
| eventType | VARCHAR | Type d'événement |
| eventVersion | INTEGER | Version du contrat |
| producer | VARCHAR | Contexte producteur |
| aggregateType | VARCHAR | Type d'agrégat |
| aggregateId | UUID | Identifiant de l'agrégat |
| aggregateVersion | BIGINT | Version métier |
| occurredAt | TIMESTAMPTZ | Date de production |
| correlationId | UUID | Traçabilité de l'opération |
| causationId | UUID | Événement ou commande déclencheur |
| payload | JSONB | Données minimales nécessaires |
| metadata | JSONB | Métadonnées techniques |

Les noms de propriétés des payloads sont définis de manière cohérente et versionnée.

### Exemple conceptuel

`ExerciseCompleted` :

- eventId ;
- eventType ;
- eventVersion ;
- producer = UserPractice ;
- aggregateType = ExerciseRun ;
- aggregateId ;
- aggregateVersion ;
- occurredAt ;
- correlationId ;
- causationId ;
- payload :
  - userId ;
  - exerciseId ;
  - exerciseVersionId ;
  - exerciseRunId ;
  - completedAt.

Aucune réponse personnelle n'est incluse dans cet événement.

## 6. Outbox pattern

La publication d'un événement doit être fiable.

### 6.1. Écriture transactionnelle

Lorsqu'une opération métier produit un événement :

1. modifier l'état métier ;
2. enregistrer l'événement dans `platform.outbox_events` ;
3. valider la même transaction PostgreSQL.

L'événement ne peut donc pas être publié si la transaction métier échoue.

### 6.2. Dispatcher

Un processus dédié :
- récupère les événements non publiés ;
- les publie vers les consommateurs internes ;
- marque leur publication ;
- réessaie en cas d'erreur temporaire ;
- conserve les erreurs techniques nécessaires au diagnostic.

La publication peut se produire plusieurs fois. Les consommateurs doivent être idempotents.

### 6.3. Idempotence des consommateurs

Chaque consommateur utilise `platform.processed_events`.

Clé logique :
- consumer_name ;
- event_id.

Le traitement et l'enregistrement de la consommation doivent être transactionnels.

Un événement déjà traité par ce consommateur ne doit pas provoquer une seconde mutation métier.

### 6.4. Erreurs

Distinguer :
- erreur temporaire : nouvelle tentative avec backoff ;
- erreur permanente : mise en échec et intervention ;
- refus métier : pas de retry automatique ;
- événement invalide : rejet et journalisation technique expurgée.

Les erreurs ne doivent jamais exposer de contenu privé.

## 7. Échanges Laravel vers Python

Le service Python est accessible uniquement sur le réseau interne Docker.

Le navigateur ne communique jamais directement avec lui.

Laravel :
- authentifie l'utilisateur ;
- vérifie les droits ;
- assemble les données autorisées ;
- transmet une requête minimale ;
- persiste l'état métier ;
- expose le résultat au frontend.

Python :
- exécute l'orchestration IA ;
- utilise Ollama ;
- réalise la recherche RAG ;
- traite les documents ;
- renvoie les résultats techniques.

### 7.1. Enveloppe de requête interne

Chaque requête comporte :

- requestId ;
- correlationId ;
- operationId ;
- contractVersion ;
- payload.

### 7.2. Enveloppe de réponse

Chaque réponse comporte :

- requestId ;
- status ;
- result ;
- error ;
- correlationId.

Statuts possibles :
- ACCEPTED ;
- COMPLETED ;
- REJECTED ;
- FAILED.

Un statut ACCEPTED indique que le traitement a été accepté, pas qu'il est terminé.

### 7.3. Sécurité

- authentification de service à service ;
- réseau Docker privé ;
- validation stricte des contrats ;
- limites de taille ;
- timeouts ;
- idempotence ;
- aucune confiance accordée aux identifiants fournis par Python sans validation Laravel.

Python ne reçoit jamais de privilège lui permettant d'élargir les droits d'accès.

## 8. Streaming des générations

Le streaming est un échange continu entre Python, Laravel et le navigateur.

Événements de transport possibles :
- `generation.started`
- `generation.progress`
- `generation.token`
- `generation.citation`
- `generation.completed`
- `generation.interrupted`
- `generation.failed`
- `generation.cancelled`

Ces événements de transport ne remplacent pas les événements métier persistés.

Laravel doit pouvoir relayer le flux sans le mettre intégralement en mémoire.

Les fragments générés sont associés à la tentative en cours.

La finalisation est explicite :
- COMPLETED : réponse terminée et sélectionnable ;
- INTERRUPTED : texte incomplet, non présenté comme réponse achevée ;
- FAILED : génération échouée ;
- CANCELLED : annulation confirmée.

Une reconnexion du navigateur ne doit pas créer une deuxième génération involontaire.

## 9. Opérations asynchrones

`platform.async_operations` sert au suivi transversal des opérations longues.

Exemples :
- ingestion documentaire ;
- indexation ;
- génération de bilan ;
- export ;
- suppression de compte ;
- restauration.

Elle fournit :
- identifiant ;
- contexte propriétaire ;
- type d'opération ;
- ressource associée ;
- statut global ;
- progression technique ;
- erreur normalisée.

Elle ne remplace jamais les statuts métier propres aux processus.

Exemple : l'indexation conserve son propre statut dans KnowledgeBase. L'AsyncOperation ne fait que présenter le suivi global.

## 10. Suppression transverse

La suppression d'un compte illustre la coordination entre contextes.

PlatformOperations :
1. enregistre la demande confirmée ;
2. bloque les nouvelles opérations ;
3. crée une tâche de suppression par contexte ;
4. déclenche les traitements idempotents ;
5. vérifie les résultats ;
6. clôture la demande.

Chaque contexte supprime ses propres données.

Le service Python supprime ses données techniques dérivées sur instruction autorisée.

Les sauvegardes historiques suivent leur politique de rétention. Après restauration, les suppressions postérieures à la sauvegarde doivent être rejouées.

Aucun contexte ne doit contourner cette orchestration en supprimant directement les tables d'un autre.

## 11. Règles de transaction

Les transactions PostgreSQL doivent rester courtes.

Ne jamais garder une transaction ouverte pendant :
- un appel HTTP à Python ;
- une génération Ollama ;
- une recherche RAG longue ;
- une extraction documentaire ;
- un transfert de fichier ;
- un traitement asynchrone.

Le système utilise des transitions d'état, des opérations idempotentes et l'outbox pour coordonner les traitements.

Aucune transaction distribuée n'est requise.

## 12. Tests obligatoires

### Contrats synchrones
- validation des entrées et sorties ;
- droits d'accès ;
- absence de dépendance aux tables internes du fournisseur ;
- compatibilité des versions de contrats.

### Événements
- écriture atomique métier/outbox ;
- publication après commit ;
- publication répétée ;
- idempotence des consommateurs ;
- gestion des erreurs ;
- compatibilité des versions.

### Python
- service indisponible ;
- timeout ;
- réponse invalide ;
- rejeu d'une requête ;
- interruption du streaming ;
- annulation ;
- limites de taille ;
- absence de fuite de données.

### Suppression
- reprise après panne ;
- double exécution ;
- suppression partielle ;
- vérification finale ;
- restauration et réconciliation.

## 13. Consigne d'implémentation pour Claude

Claude doit :

1. définir les interfaces de contrats dans le contexte fournisseur ;
2. définir les DTO d'échange ;
3. identifier les événements réellement nécessaires ;
4. créer les enveloppes et mécanismes communs ;
5. implémenter l'outbox et l'idempotence ;
6. documenter les consommateurs ;
7. tester les frontières entre contextes ;
8. éviter les appels directs aux repositories d'un autre contexte ;
9. éviter les événements contenant du contenu privé ;
10. documenter toute évolution de contrat.

Ne pas créer un bus événementiel complexe ou une infrastructure de microservices pour un besoin qui peut être traité simplement dans le monolithe modulaire.

La priorité est la fiabilité, la traçabilité technique et le faible couplage.
