# 03 — Architecture DDD et organisation du domaine

## 1. Objectif

Ce document définit l'architecture métier et logicielle de TCCTDD selon les principes du Domain-Driven Design (DDD).

Il précise :

- Les bounded contexts.
- Les responsabilités de chaque contexte.
- Les frontières entre les domaines.
- L'organisation en couches.
- Les règles de dépendance.
- La structure des namespaces et répertoires.
- Les principes de collaboration entre contextes.

Le but est de construire un monolithe modulaire cohérent, maintenable et extensible, sans introduire prématurément une architecture distribuée complexe.

## 2. Principes architecturaux

### 2.1 Monolithe modulaire

TCCTDD est organisé comme un monolithe modulaire Laravel, accompagné d'un service Python indépendant pour les traitements techniques liés à l'IA et aux documents.

Les modules métier Laravel sont séparés logiquement, mais partagent le même déploiement applicatif.

Cette approche permet :

- Une séparation claire des responsabilités.
- Des transactions métier locales.
- Des tests indépendants par contexte.
- Une évolution progressive des modules.
- Une limitation de la complexité opérationnelle.

Le découpage en microservices métier n'est pas prévu pour le MVP.

### 2.2 Domain-Driven Design

Chaque bounded context possède un modèle métier cohérent et un langage explicite.

Les règles métier doivent être portées par le domaine et non dispersées dans les contrôleurs, les composants d'interface ou les modèles de persistance.

Les concepts métier sont représentés par des agrégats, entités, objets valeur, services de domaine et événements lorsque cela est pertinent.

### 2.3 Séparation des responsabilités

L'architecture distingue :

- Le domaine métier.
- Les cas d'utilisation et l'orchestration applicative.
- Les mécanismes techniques et la persistance.
- Les interfaces d'entrée et de sortie.

Le domaine ne dépend jamais de Laravel, d'Eloquent, de Filament, de PostgreSQL ou d'un service externe.

## 3. Bounded contexts

L'application est découpée en sept bounded contexts.

| Contexte | Responsabilité principale |
|---|---|
| Companion | Conversations et interactions avec le compagnon |
| PersonalContext | Mémoire personnelle et contexte évolutif |
| TccContent | Catalogue et gouvernance éditoriale des contenus TCC |
| UserPractice | Pratique personnelle, exercices, journal, parcours et objectifs |
| KnowledgeBase | Documents, versions et gestion de la base documentaire |
| AiGovernance | Modèles, prompts, politiques et évaluation de l'IA |
| PlatformOperations | Sauvegardes, exports, suppressions et opérations techniques |

Ces contextes constituent les frontières principales du modèle métier.

## 4. Responsabilités des bounded contexts

### 4.1 Companion

Responsabilité : gérer les conversations entre l'utilisateur et le compagnon IA.

Concepts métier principaux :

- Conversation.
- Message.
- GenerationAttempt.

Le contexte gère :

- Le cycle de vie des conversations.
- L'historique des messages.
- L'ordre des messages.
- Les tentatives de génération.
- Les interruptions et régénérations.
- La sélection de la réponse effectivement produite.

Il conserve l'état métier des conversations, mais ne réalise pas lui-même l'inférence du modèle.

Il délègue les traitements IA au service Python et utilise les contrats applicatifs appropriés.

### 4.2 PersonalContext

Responsabilité : gérer la mémoire personnelle structurée et les éléments utiles à la continuité de l'accompagnement.

Concepts métier principaux :

- Memory.
- MemorySource.
- MemoryRevision.
- MemoryRelation.
- MemoryConsolidation.
- ContextSnapshot.

Le contexte gère :

- La création et le cycle de vie des mémoires.
- Leur provenance.
- Leur statut épistémique.
- Les révisions.
- Les contradictions potentielles.
- Les relations entre mémoires.
- Les propositions de consolidation.
- La préparation du contexte personnel destiné au compagnon.

Une hypothèse produite par l'IA ne devient pas automatiquement un fait confirmé.

### 4.3 TccContent

Responsabilité : gérer les contenus éditoriaux et méthodologiques TCC.

Concepts métier principaux :

- Exercise.
- ExerciseVersion.
- EditorialPathway.
- EditorialPathwayVersion.
- Resource.
- ResourceVersion.
- BibliographicReference.
- ContentEvidence.

Le contexte gère :

- La définition des exercices.
- Le versionnement des exercices.
- Les blocs de contenu et les questions.
- Les catégories et tags.
- Les parcours éditoriaux.
- Les ressources pédagogiques.
- Les justifications méthodologiques.
- Les références bibliographiques.
- Le cycle de validation éditoriale.

Il définit les contenus disponibles, mais ne gère pas leur utilisation effective par l'utilisateur.

### 4.4 UserPractice

Responsabilité : gérer la pratique personnelle de l'utilisateur.

Concepts métier principaux :

- ExerciseRun.
- ExerciseAnswer.
- JournalEntry.
- JournalEntryVersion.
- PersonalPathway.
- PathwayStep.
- PersonalGoal.
- ProgressReview.
- UserBookmark.

Le contexte gère :

- Les sessions d'exercices.
- Les réponses aux exercices.
- Le journal personnel.
- Les versions des entrées de journal.
- Les parcours personnels.
- Les étapes de parcours.
- Les objectifs.
- Les bilans de progression.
- Les favoris.

Il référence les versions exactes des contenus TCC utilisés, sans en devenir propriétaire.

### 4.5 KnowledgeBase

Responsabilité : gérer la base documentaire utilisée pour la recherche et l'enrichissement des réponses IA.

Concepts métier principaux :

- KnowledgeDocument.
- KnowledgeDocumentVersion.
- DocumentIngestion.
- DocumentIndexation.

Le contexte gère :

- Les documents importés.
- Les versions documentaires.
- Les états de validation.
- L'activation de l'utilisation par l'IA.
- Le cycle d'ingestion et d'indexation.
- La traçabilité des sources.
- La disponibilité documentaire pour la recherche.

Les traitements techniques d'extraction, de segmentation, d'embedding et de recherche sont exécutés par Python.

Les index de recherche et les traces de recherche relèvent du service Python (`ai_engine`) : ce ne sont pas des concepts du domaine Laravel. Laravel conserve uniquement l'état fonctionnel des documents, des versions, des ingestions et des indexations.

### 4.6 AiGovernance

Responsabilité : encadrer le comportement, les configurations et les évaluations de l'intelligence artificielle.

Concepts métier principaux :

- AIModel.
- AIModelConfiguration.
- PromptDefinition.
- PromptVersion.
- GovernancePolicy.
- SafetyPolicy.
- EmergencyResource.
- SafetyIntervention.
- EvaluationSuite.
- EvaluationRun.
- AIConfigurationRelease.

Le contexte gère :

- Les modèles disponibles.
- Les configurations par usage.
- Les versions de prompts.
- Les politiques de gouvernance.
- Les règles de sécurité configurables.
- Les ressources d'urgence par pays.
- Les interventions de sécurité déclenchées (traces sensibles).
- Les jeux de tests.
- Les évaluations.
- Les releases de configuration.
- L'activation et le retour arrière.

Les garde-fous critiques restent protégés par le code et ne peuvent pas être neutralisés par une simple configuration éditoriale.

### 4.7 PlatformOperations

Responsabilité : gérer les opérations transversales de maintenance et de cycle de vie des données.

Concepts métier principaux :

- Backup.
- RestoreOperation.
- ExportRequest.
- DeletionRequest.
- Notification.
- NotificationPreference.
- MaintenanceOperation.
- TelemetryConfiguration.

Le contexte gère :

- Les sauvegardes.
- Les restaurations.
- Les exports.
- Les demandes de suppression.
- La coordination des suppressions entre contextes.
- Les notifications internes.
- Les préférences de notification.
- Les opérations de maintenance.
- La configuration de la télémétrie locale.

Il orchestre les opérations transversales sans devenir propriétaire des données métier des autres contextes.

## 5. Organisation en couches

Chaque bounded context Laravel suit une architecture en quatre couches.

### 5.1 Domain

Contient le modèle métier et les règles qui lui sont propres.

Éléments possibles :

- Aggregates.
- Entities.
- ValueObjects.
- DomainServices.
- DomainEvents.
- Repository interfaces.
- Domain exceptions.
- Specifications.

Règles :

- Aucune dépendance vers Laravel.
- Aucune dépendance vers Eloquent.
- Aucune dépendance vers l'infrastructure.
- Aucune dépendance vers les contrôleurs ou l'interface.
- Les invariants métier sont protégés par les agrégats.

Les objets du domaine doivent rester indépendants du framework.

### 5.2 Application

Contient les cas d'utilisation et l'orchestration métier.

Éléments possibles :

- Commands.
- Queries.
- Handlers.
- ApplicationServices.
- DTO.
- Repository ports.
- Integration event handlers.
- Application exceptions.

Responsabilités :

- Charger les agrégats nécessaires.
- Exécuter les règles métier via le domaine.
- Coordonner les opérations.
- Gérer les transactions applicatives.
- Appeler les ports techniques.
- Publier les événements nécessaires après validation de la transaction.

Les handlers sont appelés directement, sans bus de commandes : un contrôleur ou une commande console injecte le handler concerné et l'invoque.

Les lectures passent par des queries qui retournent des DTO, jamais des modèles Eloquent ni des agrégats.

Les handlers ne doivent pas devenir des conteneurs de logique métier.

Une règle métier doit être placée dans le domaine lorsque sa responsabilité appartient à un agrégat ou à un service de domaine.

### 5.3 Infrastructure

Contient les implémentations techniques des abstractions définies par le domaine et l'application.

Éléments possibles :

- Eloquent models.
- Repository implementations.
- Database mappings.
- Event publishers.
- External service clients.
- Queue adapters.
- Storage adapters.
- Laravel-specific implementations.

Les modèles Eloquent sont des objets de persistance. Ils ne constituent pas les agrégats métier.

La conversion entre les objets de persistance et les objets du domaine est explicite.

### 5.4 Presentation

Contient les points d'entrée applicatifs.

Éléments possibles :

- HTTP Controllers.
- Form Requests.
- API Resources.
- Filament Resources.
- Console commands.
- Stream endpoints.

Les contrôleurs doivent rester minces.

Ils valident les entrées, déclenchent les cas d'utilisation et transforment les résultats en réponses adaptées.

Aucune règle métier importante ne doit être implémentée directement dans cette couche.

Exception Filament : les ressources Filament peuvent lire les données via Eloquent pour les listes et les formulaires d'administration. Toute écriture passe en revanche par les handlers de la couche Application, afin de préserver les invariants du domaine.

## 6. Règles de dépendance

Les dépendances internes suivent la règle suivante :

```text
Presentation
     |
     v
Application
     |
     v
Domain

Infrastructure ---> Application / Domain
```

Le domaine ne connaît jamais les couches externes.

Les interfaces de repository et les ports nécessaires à l'application sont définis côté domaine ou application selon leur responsabilité.

L'infrastructure les implémente.

Les dépendances entre bounded contexts ne doivent pas prendre la forme d'importations directes et incontrôlées de leurs modèles internes.

Les échanges passent par des contrats explicites, des interfaces publiques ou des événements d'intégration.

Les dépendances entre modules ne sont autorisées que via le répertoire `Contracts/` du module concerné. Un module ne référence jamais les classes `Domain`, `Application`, `Infrastructure` ou `Presentation` d'un autre module.

Des tests d'architecture automatisés font respecter cette règle.

## 7. Frontières entre les contextes

### 7.1 Propriété des données

Chaque contexte est propriétaire de ses données métier.

Un contexte ne doit pas modifier directement les tables métier d'un autre contexte.

Il peut consulter des informations exposées par un contrat public ou demander une opération au contexte propriétaire.

### 7.2 Références intercontextes

Les références entre contextes utilisent des identifiants stables, généralement des UUID.

Une référence externe ne signifie pas que le contexte référencé est une entité interne de l'agrégat.

Les vérifications d'existence, d'appartenance et d'autorisation sont effectuées par les mécanismes applicatifs appropriés.

### 7.3 Événements

Les événements d'intégration permettent de notifier les autres contextes d'un changement métier.

Ils doivent :

- Être versionnés.
- Être documentés.
- Être indépendants des modèles Eloquent.
- Ne contenir que les données nécessaires à leur traitement.
- Ne jamais contenir de texte personnel sensible.

La publication doit être fiable et compatible avec une livraison au moins une fois.

Les consommateurs doivent donc être idempotents.

Le mécanisme d'outbox est utilisé pour éviter de perdre un événement entre la transaction métier et sa publication.

L'inventaire des contrats synchrones intercontextes, le catalogue des événements, leur enveloppe et les règles de l'outbox sont décrits dans `07-CONTRATS-INTERSERVICES.md`.

## 8. Contexte partagé

Un espace `Shared` limité peut contenir des éléments réellement transversaux :

- Types et conventions techniques partagés.
- Identifiants communs si nécessaire.
- Contrats techniques génériques.
- Abstractions d'infrastructure réellement communes.
- Primitives sans responsabilité métier propre.

Il ne doit pas devenir un fourre-tout.

Une classe ne doit pas être placée dans `Shared` uniquement parce qu'elle est utilisée par plusieurs modules.

Les concepts métier restent propriétaires de leur bounded context.

## 9. Authentification et identité

L'authentification repose sur les mécanismes natifs de Laravel.

Il n'existe pas de bounded context métier Identity & Access distinct dans le MVP.

Les mécanismes techniques suivants restent dans l'infrastructure applicative :

- Gestion des utilisateurs.
- Authentification.
- Sessions.
- Hachage des mots de passe.
- Réinitialisation du mot de passe.
- Middleware.
- Policies et Gates.
- Contrôle d'accès HTTP.

Deux rôles sont prévus : `ADMIN` et `USER`. Les autorisations reposent sur les Policies Laravel, sans paquet de gestion des permissions. Le rôle `ADMIN` est seul autorisé à administrer et à valider les contenus éditoriaux.

Les préférences générales et paramètres de confidentialité sont des concepts métier distincts de l'authentification.

## 10. Service Python

Le service Python est indépendant du monolithe Laravel.

Il prend en charge les responsabilités techniques suivantes :

- Extraction documentaire.
- Segmentation et préparation des documents.
- Génération des embeddings.
- Indexation.
- Recherche hybride.
- Reranking éventuel, introduit uniquement si un besoin est mesuré (pas de reranker lourd par défaut).
- Orchestration des appels aux modèles locaux.
- Traitements IA asynchrones.

Il ne devient pas propriétaire des agrégats métier Laravel.

Laravel reste propriétaire des données métier, des autorisations et des états fonctionnels. Le service Python n'a aucun droit sur les données métier : il ne lit ni n'écrit les tables des bounded contexts et ne manipule que ses propres données techniques (index, traces de recherche).

Une seule génération IA est exécutée à la fois, conformément à la machine de référence. Les générations de conversation sont interactives et ne sont jamais préemptées ; les traitements IA de fond peuvent l'être, puis sont relancés par Laravel.

Les contrats détaillés de communication sont décrits dans `07-CONTRATS-INTERSERVICES.md`.

## 11. Organisation des répertoires

L'organisation cible du backend Laravel est :

```text
app/
├── Modules/
│   ├── Companion/
│   │   ├── Domain/
│   │   ├── Contracts/
│   │   ├── Application/
│   │   ├── Infrastructure/
│   │   └── Presentation/
│   ├── PersonalContext/
│   ├── TccContent/
│   ├── UserPractice/
│   ├── KnowledgeBase/
│   ├── AiGovernance/
│   └── PlatformOperations/
│
└── Shared/
    ├── Domain/
    ├── Application/
    └── Infrastructure/
```

Chaque contexte possède sa propre organisation interne.

Exemple :

```text
app/Modules/Companion/
├── Contracts/
├── Domain/
│   ├── Aggregates/
│   ├── Entities/
│   ├── ValueObjects/
│   ├── Events/
│   ├── Exceptions/
│   └── Repositories/
│
├── Application/
│   ├── Commands/
│   ├── Queries/
│   ├── DTO/
│   ├── Handlers/
│   ├── Services/
│   └── Ports/
│
├── Infrastructure/
│   ├── Persistence/
│   ├── Repositories/
│   ├── Events/
│   └── Services/
│
└── Presentation/
    ├── Http/
    ├── Filament/
    └── Console/
```

`Contracts/` à la racine du module contient l'API publique du module, utilisable par les autres modules (interfaces, DTO et événements d'intégration exposés). `Application/Ports/` contient les ports sortants, c'est-à-dire les interfaces que l'application requiert de l'infrastructure (clients externes, stockage, file de tâches) et qui ne sont jamais consommées par d'autres modules.

Cette structure est une convention cible. Les sous-répertoires ne doivent être créés que lorsqu'ils sont réellement utiles.

## 12. Organisation du service Python

Le service Python possède sa propre architecture, indépendante des couches Laravel.

Il doit séparer :

- Les cas d'utilisation techniques.
- Les traitements documentaires.
- Les pipelines de recherche.
- L'orchestration IA.
- Les adaptateurs vers PostgreSQL et Ollama.
- Les interfaces API.
- Les tâches asynchrones.

Les contrats interservices (endpoints, en-têtes, erreurs, idempotence, flux de génération et événements d'intégration) sont définis dans `07-CONTRATS-INTERSERVICES.md`.

La structure précise du service Python sera détaillée dans le document d'architecture technique.

## 13. Règles de conception métier

### Agrégats

Un agrégat protège un ensemble cohérent d'invariants métier.

Les modifications passent par sa racine.

Les agrégats doivent rester de taille raisonnable et ne pas charger systématiquement l'ensemble de leurs relations.

### Entités et objets valeur

Une entité possède une identité et un cycle de vie.

Un objet valeur représente un concept défini par ses attributs et son égalité.

Les objets valeur doivent être introduits lorsqu'ils apportent une réelle protection ou clarté métier, sans systématisme.

### Services de domaine

Un service de domaine est utilisé lorsqu'une règle métier ne relève naturellement d'aucune entité ou racine d'agrégat.

Il ne doit pas servir de substitut à une mauvaise répartition des responsabilités.

### Événements de domaine

Les événements de domaine représentent des faits métier qui se sont produits.

Ils sont distincts des événements d'intégration destinés aux autres contextes.

### Transactions

Les transactions sont locales à un contexte et doivent rester courtes.

Aucun appel réseau, traitement IA ou opération documentaire longue ne doit être exécuté à l'intérieur d'une transaction SQL.

## 14. Conventions de développement

- PHP 8.4 comme version cible initiale, sous réserve de compatibilité des dépendances retenues.
- Laravel comme framework backend.
- Vue 3 et TypeScript pour le frontend.
- DTO natifs : classes `final readonly` explicites, sans `spatie/laravel-data` par défaut.
- Pas de typage strict global (`strict_types` n'est pas activé).
- Types explicites lorsque cela améliore la lisibilité et la sécurité.
- Enums PHP pour les états et catégories fermés.
- UUID v7 pour les identifiants métier, générés via `ramsey/uuid` et manipulés sous forme de chaîne (`string`).
- UTC pour les dates persistées.
- Pest comme framework de tests.
- Tests unitaires pour les règles métier.
- Tests d'architecture vérifiant les règles de dépendance entre couches et entre modules.
- Tests d'intégration avec PostgreSQL réel pour les comportements spécifiques à la base.
- Analyse statique et formatage automatisé.
- Pas de logique métier dans les contrôleurs ou les modèles Eloquent.
- Pas de dépendance directe entre les modèles internes de deux contextes.

Les conventions détaillées de nommage, de tests, de migrations et de qualité du code pourront être précisées dans le brief d'implémentation.

## 15. Principes à préserver

Les décisions suivantes sont structurantes :

1. Architecture en monolithe modulaire pour Laravel.
2. Sept bounded contexts clairement délimités.
3. Séparation stricte entre domaine, application, infrastructure et présentation.
4. Domaine indépendant de Laravel et d'Eloquent.
5. Propriété explicite des données par contexte.
6. Pas d'écriture directe dans les données métier d'un autre contexte.
7. Contrats et événements versionnés pour les échanges.
8. Outbox pour la publication fiable des événements.
9. Service Python indépendant pour les traitements IA et documentaires.
10. Aucun microservice métier supplémentaire sans besoin démontré.

---

**Statut :** architecture DDD — version de travail à valider.

**Dépendances :** `01-VISION-PRODUIT.md`, `02-CAHIER-DES-CHARGES.md` et `07-CONTRATS-INTERSERVICES.md`.

**Hors périmètre :** dictionnaire PostgreSQL détaillé (`06-DICTIONNAIRE-DONNEES-POSTGRESQL.md`), catalogue des cas d'utilisation et plan d'implémentation.

**Changements v2 :** KnowledgeBase recentrée (index et traces côté Python), concepts AiGovernance et PlatformOperations complétés, handlers sans bus de commandes, exception Filament, dépendances via `Contracts/`, `Application/Ports/`, conventions DTO, UUID v7 et Pest, rôles `ADMIN` et `USER`, reranking conditionné à un besoin mesuré, priorité des conversations sur les traitements de fond (Phase 0).
