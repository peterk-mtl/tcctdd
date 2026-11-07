# 06 — Dictionnaire de données PostgreSQL

## 1. Objectif

Ce document définit le modèle de persistance PostgreSQL de TCCTDD.

Il précise :

- Les schémas PostgreSQL.
- Les tables et leurs colonnes.
- Les types de données.
- Les clés primaires et étrangères.
- Les contraintes d'intégrité.
- Les index.
- Les relations entre les données.
- La séparation des responsabilités entre Laravel et Python.

Le modèle de données respecte les sept bounded contexts décrits dans `04-BOUNDED-CONTEXTS.md`.

## 2. Principes généraux

### 2.1 Identifiants

- UUID v7 comme identifiant primaire des entités métier.
- UUID générés côté application via `ramsey/uuid`, manipulés sous forme de chaîne (`string`).
- Pas de dépendance aux séquences numériques PostgreSQL pour les identifiants métier.

### 2.2 Dates

- Toutes les dates persistées sont stockées en UTC.
- Type PostgreSQL : `TIMESTAMPTZ`.
- Les conversions vers le fuseau utilisateur sont effectuées au niveau applicatif.

### 2.3 Types

- `UUID` pour les identifiants.
- `TEXT` pour les contenus textuels longs.
- `VARCHAR` pour les chaînes à longueur contrôlée.
- `BIGINT` pour les compteurs et séquences.
- `BOOLEAN` pour les valeurs binaires.
- `JSONB` pour les structures réellement variables.
- `NUMERIC` pour les valeurs numériques nécessitant une précision exacte.
- `CHAR(64)` pour les checksums SHA-256.
- `TIMESTAMPTZ` pour les dates.

Les enums métier sont représentés par des enums PHP et des contraintes SQL adaptées.

### 2.4 JSONB

JSONB est réservé aux structures variables ou configurables.

Les données relationnelles importantes, les références et les états métier doivent rester dans des colonnes explicites.

Les questions et blocs d'exercices ne doivent pas être regroupés dans un unique document JSONB : leur structure est normalisée.

### 2.5 Suppression

La suppression physique ou logique dépend du cycle de vie métier.

Les suppressions doivent être explicites et cohérentes avec les règles de conservation et de suppression des données dérivées.

Le soft delete n'est pas systématique.

### 2.6 Contraintes

Les contraintes importantes doivent être garanties au niveau de la base lorsque cela est possible :

- Unicité.
- Non-nullité.
- Intégrité référentielle.
- Cohérence des états.
- Absence de références invalides.
- Unicité conditionnelle.

Les invariants métier complexes restent également protégés dans le domaine.

## 3. Organisation des schémas PostgreSQL

Les schémas prévus sont :

| Schéma | Propriétaire |
|---|---|
| identity | Infrastructure d'authentification |
| companion | Companion |
| personal_context | PersonalContext |
| tcc_content | TccContent |
| user_practice | UserPractice |
| knowledge_base | KnowledgeBase |
| ai_governance | AiGovernance |
| platform | PlatformOperations et infrastructure |
| ai_engine | Service Python |

Laravel possède les schémas métier et les données de référence nécessaires.

Python possède exclusivement les données techniques du schéma `ai_engine`.

Les rôles SQL, les schémas et les extensions (dont `pgvector`) sont créés par le script d'initialisation PostgreSQL. Les migrations Laravel et Alembic se limitent aux tables, index et contraintes de leurs schémas respectifs.

Les rôles SQL doivent respecter le principe du moindre privilège.

Le service Python n'a **aucun droit** (lecture ni écriture) sur les schémas métier Laravel. Il possède exclusivement le schéma `ai_engine`.

## 4. Schéma identity

### 4.1 users

Table utilisateur compatible avec les mécanismes natifs Laravel.

| Colonne | Type | Contraintes |
|---|---|---|
| id | UUID | PK |
| name | VARCHAR(255) | NOT NULL |
| email | VARCHAR(255) | NOT NULL, UNIQUE |
| email_verified_at | TIMESTAMPTZ | NULL |
| password | VARCHAR(255) | NOT NULL |
| remember_token | VARCHAR(100) | NULL |
| role | VARCHAR(20) | NOT NULL, DEFAULT 'USER' |
| is_active | BOOLEAN | NOT NULL, DEFAULT TRUE |
| created_at | TIMESTAMPTZ | NOT NULL |
| updated_at | TIMESTAMPTZ | NOT NULL |

Valeurs de `role` : `ADMIN`, `USER`. Les autorisations reposent sur les Policies Laravel.

### 4.2 password_reset_tokens

Table compatible avec le Password Broker Laravel.

| Colonne | Type | Contraintes |
|---|---|---|
| email | VARCHAR(255) | PK |
| token | VARCHAR(255) | NOT NULL |
| created_at | TIMESTAMPTZ | NULL |

### 4.3 sessions

Table de sessions Laravel.

Les colonnes doivent rester compatibles avec le driver de session sélectionné. En particulier, `user_id` est de type `UUID NULL`.

### 4.4 session_activity

Table complémentaire permettant de gérer le verrouillage après inactivité.

| Colonne | Type | Contraintes |
|---|---|---|
| id | UUID | PK |
| user_id | UUID | FK users |
| session_id | VARCHAR(255) | NOT NULL |
| last_interaction_at | TIMESTAMPTZ | NOT NULL |
| locked_at | TIMESTAMPTZ | NULL |
| created_at | TIMESTAMPTZ | NOT NULL |
| updated_at | TIMESTAMPTZ | NOT NULL |

Unicité sur `session_id`.

### 4.5 user_preferences

| Colonne | Type | Contraintes |
|---|---|---|
| id | UUID | PK |
| user_id | UUID | FK users, UNIQUE |
| locale | VARCHAR(10) | NOT NULL |
| timezone | VARCHAR(100) | NOT NULL |
| country_code | CHAR(2) | NOT NULL, DEFAULT 'FR' |
| ui_preferences | JSONB | NOT NULL, DEFAULT '{}' |
| privacy_preferences | JSONB | NOT NULL, DEFAULT '{}' |
| created_at | TIMESTAMPTZ | NOT NULL |
| updated_at | TIMESTAMPTZ | NOT NULL |

`country_code` détermine les ressources d'urgence présentées (France par défaut).

Les préférences spécifiques au compagnon et à la pratique restent dans leurs contextes respectifs.

## 5. Schéma companion

### 5.1 conversations

| Colonne | Type | Contraintes |
|---|---|---|
| id | UUID | PK |
| user_id | UUID | NOT NULL |
| title | VARCHAR(255) | NULL |
| conversation_type | VARCHAR(20) | NOT NULL |
| status | VARCHAR(20) | NOT NULL |
| last_message_sequence | BIGINT | NOT NULL, DEFAULT 0 |
| created_at | TIMESTAMPTZ | NOT NULL |
| updated_at | TIMESTAMPTZ | NOT NULL |
| archived_at | TIMESTAMPTZ | NULL |
| deleted_at | TIMESTAMPTZ | NULL |

Valeurs de `conversation_type` : GENERAL, THEMATIC.

Valeurs de `status` : ACTIVE, ARCHIVED, DELETED.

Index :
- `(user_id, status, updated_at DESC)`.
- `(user_id, created_at DESC)`.

### 5.2 messages

| Colonne | Type | Contraintes |
|---|---|---|
| id | UUID | PK |
| conversation_id | UUID | FK conversations |
| sequence | BIGINT | NOT NULL |
| role | VARCHAR(20) | NOT NULL |
| status | VARCHAR(20) | NOT NULL |
| content | TEXT | NULL |
| selected_attempt_id | UUID | NULL |
| created_at | TIMESTAMPTZ | NOT NULL |
| updated_at | TIMESTAMPTZ | NOT NULL |
| deleted_at | TIMESTAMPTZ | NULL |

Valeurs de `role` : USER, ASSISTANT, SYSTEM.

Index et contraintes :
- UNIQUE `(conversation_id, sequence)`.
- Index `(conversation_id, sequence)`.
- Index `(conversation_id, status)`.

Le contenu utilisateur est stocké directement dans `content`.

Pour un message assistant, le contenu généré est stocké dans GenerationAttempt.

### 5.3 generation_attempts

| Colonne | Type | Contraintes |
|---|---|---|
| id | UUID | PK |
| message_id | UUID | FK messages |
| attempt_number | INTEGER | NOT NULL |
| status | VARCHAR(20) | NOT NULL |
| generated_content | TEXT | NULL |
| error_code | VARCHAR(100) | NULL |
| ai_release_id | UUID | NOT NULL |
| model_version | VARCHAR(255) | NULL |
| generation_parameters | JSONB | NULL |
| duration_ms | BIGINT | NULL |
| input_tokens | BIGINT | NULL |
| output_tokens | BIGINT | NULL |
| created_at | TIMESTAMPTZ | NOT NULL |
| started_at | TIMESTAMPTZ | NULL |
| completed_at | TIMESTAMPTZ | NULL |

Valeurs de `status` : PENDING, STREAMING, COMPLETED, INTERRUPTED, FAILED, CANCELLED.

Une génération de conversation n'est jamais préemptée. Si le créneau est déjà occupé par une autre requête interactive, la tentative passe `FAILED` avec `error_code = ENGINE_BUSY`.

Contraintes :
- UNIQUE `(message_id, attempt_number)`.
- UNIQUE `(id, message_id)` pour la référence composite depuis messages.
- Un index unique partiel sur `(message_id)` WHERE `status IN ('PENDING', 'STREAMING')` garantit une seule tentative active par message.

`ai_release_id` référence la release de configuration IA utilisée pour cette tentative (voir `ai_governance.ai_configuration_releases`). Deux tentatives d'un même message peuvent référencer des releases différentes.

`generated_content` est vidé sept jours après qu'une tentative a atteint le statut `INTERRUPTED`, `CANCELLED` ou `FAILED`.

La référence `messages.selected_attempt_id` doit être protégée par une clé étrangère composite :

`(selected_attempt_id, id)` vers `(generation_attempts.id, generation_attempts.message_id)`.

Elle garantit que la tentative sélectionnée appartient bien au message concerné.

Les traitements IA de fond (extraction, consolidation, bilans, propositions de parcours, évaluations) ne sont pas stockés dans `generation_attempts`. Leur suivi, y compris le statut terminal `PREEMPTED`, relève de `platform.async_operations` (paragraphe 12.4).

### 5.4 Statuts des messages

Les messages de rôle USER et SYSTEM sont créés directement avec le statut COMPLETED.

Les messages assistant suivent le cycle de génération. Leurs statuts possibles sont :

- GENERATING.
- COMPLETED.
- INTERRUPTED.
- FAILED.
- CANCELLED.
- DELETED.

Le statut doit rester cohérent avec les tentatives de génération.

## 6. Schéma personal_context

### 6.1 memories

| Colonne | Type | Contraintes |
|---|---|---|
| id | UUID | PK |
| user_id | UUID | NOT NULL |
| category | VARCHAR(40) | NOT NULL |
| content | JSONB | NOT NULL |
| processing_status | VARCHAR(20) | NOT NULL |
| epistemic_status | VARCHAR(20) | NOT NULL |
| confidence | NUMERIC(4,3) | NULL |
| superseded_by_id | UUID | NULL |
| created_at | TIMESTAMPTZ | NOT NULL |
| updated_at | TIMESTAMPTZ | NOT NULL |
| archived_at | TIMESTAMPTZ | NULL |

Catégories :

PERSONAL_CONTEXT, RECURRING_DIFFICULTY, TRIGGER, THOUGHT_PATTERN, EMOTIONAL_PATTERN, GOAL, HELPFUL_STRATEGY, RESOURCE, PROGRESS.

Statuts de traitement :

PROPOSED, ACTIVE, REJECTED, ARCHIVED, SUPERSEDED.

Statuts épistémiques :

HYPOTHESIS, DECLARED, CONFIRMED, CONTESTED.

Contraintes :
- `confidence` compris entre 0 et 1 si renseigné.
- Une mémoire SUPERSEDED doit référencer une autre mémoire.
- Une mémoire ne peut pas se référencer elle-même.

Index :
- `(user_id, processing_status)`.
- `(user_id, category)`.
- `(user_id, epistemic_status)`.

### 6.2 memory_sources

| Colonne | Type | Contraintes |
|---|---|---|
| id | UUID | PK |
| memory_id | UUID | FK memories |
| source_type | VARCHAR(40) | NOT NULL |
| source_id | UUID | NOT NULL |
| source_version | VARCHAR(100) | NULL |
| created_at | TIMESTAMPTZ | NOT NULL |

Une source externe à ce schéma est référencée par identifiant et type, sans FK intercontexte.

Une mémoire active doit posséder au moins une source admissible.

### 6.3 memory_revisions

| Colonne | Type | Contraintes |
|---|---|---|
| id | UUID | PK |
| memory_id | UUID | FK memories |
| revision_number | INTEGER | NOT NULL |
| content | JSONB | NOT NULL |
| epistemic_status | VARCHAR(20) | NOT NULL |
| change_reason | VARCHAR(255) | NULL |
| created_at | TIMESTAMPTZ | NOT NULL |

UNIQUE `(memory_id, revision_number)`.

### 6.4 memory_relations

| Colonne | Type | Contraintes |
|---|---|---|
| id | UUID | PK |
| user_id | UUID | NOT NULL |
| source_memory_id | UUID | FK memories |
| target_memory_id | UUID | FK memories |
| relation_type | VARCHAR(40) | NOT NULL |
| status | VARCHAR(20) | NOT NULL |
| epistemic_status | VARCHAR(20) | NOT NULL |
| created_at | TIMESTAMPTZ | NOT NULL |
| updated_at | TIMESTAMPTZ | NOT NULL |

Types : TRIGGERED_BY, ASSOCIATED_WITH, SUPPORTED_BY, CONTRADICTS, REINFORCES, MITIGATED_BY, CONTRIBUTES_TO, RELATED_TO.

Contraintes :
- Pas de relation vers soi-même.
- Même utilisateur pour les deux mémoires.
- Pas de doublon actif de même type et direction.

### 6.5 Contradictions

Il n'existe pas de tables `memory_conflicts` ni `memory_conflict_items`.

Une contradiction potentielle est représentée uniquement par une `memory_relations` de type `CONTRADICTS` entre deux mémoires.

### 6.6 memory_consolidations

| Colonne | Type | Contraintes |
|---|---|---|
| id | UUID | PK |
| user_id | UUID | NOT NULL |
| consolidation_type | VARCHAR(30) | NOT NULL |
| status | VARCHAR(20) | NOT NULL |
| proposal | JSONB | NOT NULL |
| created_at | TIMESTAMPTZ | NOT NULL |
| reviewed_at | TIMESTAMPTZ | NULL |

Types : ATTACH_SOURCE, MERGE, REVISE.

Statuts : PROPOSED, AUTO_APPLIED, ACCEPTED, REJECTED, EXPIRED.

### 6.7 memory_consolidation_items

| Colonne | Type | Contraintes |
|---|---|---|
| consolidation_id | UUID | FK memory_consolidations |
| memory_id | UUID | FK memories |
| role | VARCHAR(20) | NOT NULL |

PK `(consolidation_id, memory_id)`.

### 6.8 companion_preferences

| Colonne | Type | Contraintes |
|---|---|---|
| id | UUID | PK |
| user_id | UUID | UNIQUE |
| preferences | JSONB | NOT NULL |
| inference_enabled | BOOLEAN | NOT NULL, DEFAULT TRUE |
| created_at | TIMESTAMPTZ | NOT NULL |
| updated_at | TIMESTAMPTZ | NOT NULL |

### 6.9 context_snapshots

| Colonne | Type | Contraintes |
|---|---|---|
| id | UUID | PK |
| user_id | UUID | NOT NULL |
| conversation_id | UUID | NULL |
| assembly_version | VARCHAR(50) | NOT NULL |
| references | JSONB | NOT NULL |
| created_at | TIMESTAMPTZ | NOT NULL |
| expires_at | TIMESTAMPTZ | NULL |

Le snapshot conserve des références et versions, pas une copie complète des données personnelles.

## 7. Schéma tcc_content

### 7.1 exercises

| Colonne | Type | Contraintes |
|---|---|---|
| id | UUID | PK |
| title | VARCHAR(255) | NOT NULL |
| slug | VARCHAR(255) | UNIQUE |
| description | TEXT | NULL |
| status | VARCHAR(30) | NOT NULL |
| current_version_id | UUID | NULL |
| created_at | TIMESTAMPTZ | NOT NULL |
| updated_at | TIMESTAMPTZ | NOT NULL |

### 7.2 exercise_versions

| Colonne | Type | Contraintes |
|---|---|---|
| id | UUID | PK |
| exercise_id | UUID | FK exercises |
| version_number | INTEGER | NOT NULL |
| status | VARCHAR(30) | NOT NULL |
| introduction | TEXT | NOT NULL |
| objectives | TEXT | NOT NULL |
| instructions | TEXT | NOT NULL |
| conclusion | TEXT | NOT NULL |
| precautions | TEXT | NULL |
| rationale | TEXT | NOT NULL |
| methodological_basis | TEXT | NOT NULL |
| evidence_level | VARCHAR(40) | NOT NULL |
| limitations | TEXT | NULL |
| author_id | UUID | NOT NULL |
| reviewer_id | UUID | NULL |
| reviewed_at | TIMESTAMPTZ | NULL |
| created_at | TIMESTAMPTZ | NOT NULL |
| published_at | TIMESTAMPTZ | NULL |

UNIQUE `(exercise_id, version_number)`.

`author_id` et `reviewer_id` référencent des utilisateurs disposant du rôle `ADMIN`. La validation est réservée à ce rôle.

### 7.3 exercise_version_sections

| Colonne | Type | Contraintes |
|---|---|---|
| id | UUID | PK |
| exercise_version_id | UUID | FK exercise_versions |
| section_type | VARCHAR(40) | NOT NULL |
| title | VARCHAR(255) | NULL |
| position | INTEGER | NOT NULL |

### 7.4 exercise_content_blocks

| Colonne | Type | Contraintes |
|---|---|---|
| id | UUID | PK |
| section_id | UUID | FK exercise_version_sections |
| block_type | VARCHAR(40) | NOT NULL |
| content | JSONB | NOT NULL |
| condition | JSONB | NULL |
| position | INTEGER | NOT NULL |

Le contenu JSONB est limité à la configuration du bloc. Les questions et leurs identifiants restent dans des tables dédiées.

### 7.5 exercise_questions

| Colonne | Type | Contraintes |
|---|---|---|
| id | UUID | PK |
| exercise_version_id | UUID | FK exercise_versions |
| block_id | UUID | FK exercise_content_blocks |
| question_key | VARCHAR(100) | NOT NULL |
| question_type | VARCHAR(40) | NOT NULL |
| label | TEXT | NOT NULL |
| configuration | JSONB | NOT NULL |
| required | BOOLEAN | NOT NULL, DEFAULT FALSE |
| position | INTEGER | NOT NULL |

UNIQUE `(exercise_version_id, question_key)`.

`question_key` est stable au sein d'une version d'exercice, afin de permettre le référencement des réponses indépendamment du bloc.

### 7.6 exercise_categories

| Colonne | Type | Contraintes |
|---|---|---|
| id | UUID | PK |
| parent_id | UUID | FK exercise_categories |
| name | VARCHAR(150) | NOT NULL |
| slug | VARCHAR(150) | UNIQUE |
| status | VARCHAR(20) | NOT NULL |

### 7.7 exercise_tags

| Colonne | Type | Contraintes |
|---|---|---|
| id | UUID | PK |
| name | VARCHAR(100) | NOT NULL |
| slug | VARCHAR(100) | UNIQUE |
| status | VARCHAR(20) | NOT NULL |

### 7.8 exercise_category_assignments

| Colonne | Type | Contraintes |
|---|---|---|
| exercise_id | UUID | FK exercises |
| category_id | UUID | FK exercise_categories |
| is_primary | BOOLEAN | NOT NULL, DEFAULT FALSE |

PK `(exercise_id, category_id)`.

Un index unique partiel garantit une seule catégorie principale par exercice.

### 7.9 exercise_tag_assignments

| Colonne | Type | Contraintes |
|---|---|---|
| exercise_id | UUID | FK exercises |
| tag_id | UUID | FK exercise_tags |

PK `(exercise_id, tag_id)`.

### 7.10 bibliographic_references

| Colonne | Type | Contraintes |
|---|---|---|
| id | UUID | PK |
| title | TEXT | NOT NULL |
| authors | JSONB | NOT NULL |
| publication_year | INTEGER | NULL |
| publication | TEXT | NULL |
| doi | VARCHAR(255) | NULL |
| url | TEXT | NULL |
| reference_metadata | JSONB | NOT NULL, DEFAULT '{}' |

### 7.11 content_evidence

| Colonne | Type | Contraintes |
|---|---|---|
| id | UUID | PK |
| exercise_version_id | UUID | NULL |
| editorial_pathway_version_id | UUID | NULL |
| bibliographic_reference_id | UUID | FK bibliographic_references |
| support_type | VARCHAR(40) | NOT NULL |
| rationale | TEXT | NOT NULL |

Une contrainte garantit qu'une preuve est associée à une version d'exercice ou à une version de parcours.

### 7.12 editorial_pathways

| Colonne | Type | Contraintes |
|---|---|---|
| id | UUID | PK |
| title | VARCHAR(255) | NOT NULL |
| description | TEXT | NULL |
| status | VARCHAR(30) | NOT NULL |
| current_version_id | UUID | NULL |
| created_at | TIMESTAMPTZ | NOT NULL |
| updated_at | TIMESTAMPTZ | NOT NULL |

### 7.13 editorial_pathway_versions

| Colonne | Type | Contraintes |
|---|---|---|
| id | UUID | PK |
| pathway_id | UUID | FK editorial_pathways |
| version_number | INTEGER | NOT NULL |
| status | VARCHAR(30) | NOT NULL |
| rationale | TEXT | NOT NULL |
| methodological_basis | TEXT | NOT NULL |
| evidence_level | VARCHAR(40) | NOT NULL |
| limitations | TEXT | NULL |
| author_id | UUID | NOT NULL |
| reviewer_id | UUID | NULL |
| reviewed_at | TIMESTAMPTZ | NULL |
| created_at | TIMESTAMPTZ | NOT NULL |
| published_at | TIMESTAMPTZ | NULL |

UNIQUE `(pathway_id, version_number)`.

### 7.14 editorial_pathway_steps

| Colonne | Type | Contraintes |
|---|---|---|
| id | UUID | PK |
| pathway_version_id | UUID | FK editorial_pathway_versions |
| step_type | VARCHAR(30) | NOT NULL |
| referenced_content_id | UUID | NULL |
| referenced_content_version_id | UUID | NULL |
| title | VARCHAR(255) | NULL |
| position | INTEGER | NOT NULL |
| optional | BOOLEAN | NOT NULL, DEFAULT FALSE |

Les références exactes aux contenus sont validées par l'application, car elles peuvent concerner plusieurs types d'agrégats.

Une contrainte `CHECK` impose que les références soient renseignées ou nulles selon `step_type` : seules les étapes dont le type l'exige doivent référencer un contenu.

### 7.15 editorial_pathway_prerequisites

| Colonne | Type | Contraintes |
|---|---|---|
| step_id | UUID | FK editorial_pathway_steps |
| prerequisite_step_id | UUID | FK editorial_pathway_steps |

PK `(step_id, prerequisite_step_id)`.

Les cycles sont interdits par validation applicative.

### 7.16 resources

| Colonne | Type | Contraintes |
|---|---|---|
| id | UUID | PK |
| title | VARCHAR(255) | NOT NULL |
| description | TEXT | NULL |
| status | VARCHAR(30) | NOT NULL |
| current_version_id | UUID | NULL |
| created_at | TIMESTAMPTZ | NOT NULL |
| updated_at | TIMESTAMPTZ | NOT NULL |

### 7.17 resource_versions

| Colonne | Type | Contraintes |
|---|---|---|
| id | UUID | PK |
| resource_id | UUID | FK resources |
| version_number | INTEGER | NOT NULL |
| storage_key | TEXT | NOT NULL |
| original_filename | VARCHAR(255) | NOT NULL |
| mime_type | VARCHAR(150) | NOT NULL |
| file_size | BIGINT | NOT NULL |
| checksum | CHAR(64) | NOT NULL |
| created_at | TIMESTAMPTZ | NOT NULL |

UNIQUE `(resource_id, version_number)`.

L'activation pour la recherche IA ne figure pas sur cette table. Elle est portée exclusivement par `knowledge_base.knowledge_documents.ai_retrieval_enabled`, y compris lorsque le document a pour source une ressource TCC (`source_type = TCC_RESOURCE`).

## 8. Schéma user_practice

### 8.1 exercise_runs

| Colonne | Type | Contraintes |
|---|---|---|
| id | UUID | PK |
| user_id | UUID | NOT NULL |
| exercise_id | UUID | NOT NULL |
| exercise_version_id | UUID | NOT NULL |
| status | VARCHAR(20) | NOT NULL |
| started_at | TIMESTAMPTZ | NOT NULL |
| completed_at | TIMESTAMPTZ | NULL |
| abandoned_at | TIMESTAMPTZ | NULL |
| version | BIGINT | NOT NULL, DEFAULT 1 |
| created_at | TIMESTAMPTZ | NOT NULL |
| updated_at | TIMESTAMPTZ | NOT NULL |

Les identifiants de contenu sont des références vers TccContent, sans FK interschéma obligatoire.

### 8.2 exercise_answers

| Colonne | Type | Contraintes |
|---|---|---|
| id | UUID | PK |
| exercise_run_id | UUID | FK exercise_runs |
| question_id | UUID | NOT NULL |
| answer | JSONB | NOT NULL |
| visibility_state | VARCHAR(20) | NOT NULL |
| created_at | TIMESTAMPTZ | NOT NULL |
| updated_at | TIMESTAMPTZ | NOT NULL |

UNIQUE `(exercise_run_id, question_id)`.

### 8.3 journal_entries

| Colonne | Type | Contraintes |
|---|---|---|
| id | UUID | PK |
| user_id | UUID | NOT NULL |
| title | VARCHAR(255) | NULL |
| current_version_id | UUID | NULL |
| status | VARCHAR(20) | NOT NULL, DEFAULT 'ACTIVE' |
| ai_usage_policy | VARCHAR(20) | NOT NULL |
| created_at | TIMESTAMPTZ | NOT NULL |
| updated_at | TIMESTAMPTZ | NOT NULL |
| archived_at | TIMESTAMPTZ | NULL |
| deleted_at | TIMESTAMPTZ | NULL |

Valeurs de `status` : ACTIVE, ARCHIVED, DELETED.

Une entrée archivée peut être restaurée. Une entrée `DELETED` correspond à une suppression logique immédiate ; la suppression physique est asynchrone.

### 8.4 journal_entry_versions

| Colonne | Type | Contraintes |
|---|---|---|
| id | UUID | PK |
| journal_entry_id | UUID | FK journal_entries |
| version_number | INTEGER | NOT NULL |
| content | TEXT | NOT NULL |
| created_at | TIMESTAMPTZ | NOT NULL |

UNIQUE `(journal_entry_id, version_number)`.

### 8.5 practice_preferences

| Colonne | Type | Contraintes |
|---|---|---|
| id | UUID | PK |
| user_id | UUID | UNIQUE |
| default_journal_ai_policy | VARCHAR(20) | NOT NULL, DEFAULT 'DENIED' |
| preferences | JSONB | NOT NULL, DEFAULT '{}' |
| created_at | TIMESTAMPTZ | NOT NULL |
| updated_at | TIMESTAMPTZ | NOT NULL |

### 8.6 personal_pathways

| Colonne | Type | Contraintes |
|---|---|---|
| id | UUID | PK |
| user_id | UUID | NOT NULL |
| title | VARCHAR(255) | NOT NULL |
| description | TEXT | NULL |
| origin | VARCHAR(30) | NOT NULL |
| status | VARCHAR(20) | NOT NULL |
| source_editorial_pathway_id | UUID | NULL |
| source_editorial_pathway_version_id | UUID | NULL |
| created_at | TIMESTAMPTZ | NOT NULL |
| updated_at | TIMESTAMPTZ | NOT NULL |

### 8.7 pathway_steps

| Colonne | Type | Contraintes |
|---|---|---|
| id | UUID | PK |
| personal_pathway_id | UUID | FK personal_pathways |
| step_type | VARCHAR(30) | NOT NULL |
| referenced_content_id | UUID | NULL |
| referenced_content_version_id | UUID | NULL |
| title | VARCHAR(255) | NULL |
| position | INTEGER | NOT NULL |
| operational_status | VARCHAR(20) | NOT NULL |
| optional | BOOLEAN | NOT NULL, DEFAULT FALSE |
| created_at | TIMESTAMPTZ | NOT NULL |
| updated_at | TIMESTAMPTZ | NOT NULL |

Une contrainte `CHECK` impose que les références soient renseignées ou nulles selon `step_type`.

### 8.8 pathway_prerequisites

| Colonne | Type | Contraintes |
|---|---|---|
| step_id | UUID | FK pathway_steps |
| prerequisite_step_id | UUID | FK pathway_steps |

PK `(step_id, prerequisite_step_id)`.

### 8.9 personal_goals

| Colonne | Type | Contraintes |
|---|---|---|
| id | UUID | PK |
| user_id | UUID | NOT NULL |
| title | VARCHAR(255) | NOT NULL |
| description | TEXT | NULL |
| status | VARCHAR(20) | NOT NULL |
| target_date | DATE | NULL |
| achieved_at | TIMESTAMPTZ | NULL |
| created_at | TIMESTAMPTZ | NOT NULL |
| updated_at | TIMESTAMPTZ | NOT NULL |

### 8.10 goal_pathway_links

| Colonne | Type | Contraintes |
|---|---|---|
| goal_id | UUID | FK personal_goals |
| pathway_id | UUID | FK personal_pathways |

PK `(goal_id, pathway_id)`.

### 8.11 progress_reviews

| Colonne | Type | Contraintes |
|---|---|---|
| id | UUID | PK |
| user_id | UUID | NOT NULL |
| status | VARCHAR(20) | NOT NULL |
| scope | JSONB | NOT NULL |
| generated_content | TEXT | NULL |
| ai_release_id | UUID | NULL |
| validated_at | TIMESTAMPTZ | NULL |
| created_at | TIMESTAMPTZ | NOT NULL |
| updated_at | TIMESTAMPTZ | NOT NULL |

`ai_release_id` conserve la release de configuration IA ayant servi à la génération. Pendant une préemption, le bilan reste `GENERATING` ; la tentative préemptée est suivie dans `platform.async_operations`.

### 8.12 progress_review_snapshots

| Colonne | Type | Contraintes |
|---|---|---|
| id | UUID | PK |
| progress_review_id | UUID | FK progress_reviews |
| source_type | VARCHAR(40) | NOT NULL |
| source_id | UUID | NOT NULL |
| source_version | VARCHAR(100) | NULL |
| created_at | TIMESTAMPTZ | NOT NULL |

### 8.13 progress_review_revisions

| Colonne | Type | Contraintes |
|---|---|---|
| id | UUID | PK |
| progress_review_id | UUID | FK progress_reviews |
| revision_number | INTEGER | NOT NULL |
| content | TEXT | NOT NULL |
| created_at | TIMESTAMPTZ | NOT NULL |

UNIQUE `(progress_review_id, revision_number)`.

### 8.14 user_bookmarks

| Colonne | Type | Contraintes |
|---|---|---|
| id | UUID | PK |
| user_id | UUID | NOT NULL |
| target_type | VARCHAR(40) | NOT NULL |
| target_id | UUID | NOT NULL |
| created_at | TIMESTAMPTZ | NOT NULL |

UNIQUE `(user_id, target_type, target_id)`.

## 9. Schéma knowledge_base

### 9.1 knowledge_documents

| Colonne | Type | Contraintes |
|---|---|---|
| id | UUID | PK |
| title | VARCHAR(255) | NOT NULL |
| description | TEXT | NULL |
| document_type | VARCHAR(40) | NOT NULL |
| source_type | VARCHAR(30) | NOT NULL |
| source_id | UUID | NULL |
| status | VARCHAR(30) | NOT NULL |
| latest_version_id | UUID | NULL |
| active_version_id | UUID | NULL |
| ai_retrieval_enabled | BOOLEAN | NOT NULL, DEFAULT FALSE |
| created_by | UUID | NULL |
| created_at | TIMESTAMPTZ | NOT NULL |
| updated_at | TIMESTAMPTZ | NOT NULL |

Valeurs possibles de `source_type` : IMPORTED, TCC_RESOURCE.

`ai_retrieval_enabled` est la seule source de vérité pour l'activation de la recherche IA, y compris lorsque `source_type = TCC_RESOURCE`. Dans ce cas, `source_id` référence l'identifiant de la ressource TCC correspondante (`tcc_content.resources.id`), sans clé étrangère interschéma.

### 9.2 knowledge_document_versions

| Colonne | Type | Contraintes |
|---|---|---|
| id | UUID | PK |
| document_id | UUID | FK knowledge_documents |
| version_number | INTEGER | NOT NULL |
| storage_key | TEXT | NOT NULL |
| original_filename | VARCHAR(255) | NOT NULL |
| mime_type | VARCHAR(150) | NOT NULL |
| file_size | BIGINT | NOT NULL |
| checksum | CHAR(64) | NOT NULL |
| extraction_status | VARCHAR(30) | NOT NULL |
| approved_at | TIMESTAMPTZ | NULL |
| created_at | TIMESTAMPTZ | NOT NULL |

UNIQUE `(document_id, version_number)`.

### 9.3 document_ingestions

| Colonne | Type | Contraintes |
|---|---|---|
| id | UUID | PK |
| document_version_id | UUID | FK knowledge_document_versions |
| status | VARCHAR(30) | NOT NULL |
| requested_at | TIMESTAMPTZ | NOT NULL |
| completed_at | TIMESTAMPTZ | NULL |

Valeurs de `status` : PENDING, PROCESSING, COMPLETED, PARTIAL, FAILED, UNSUPPORTED.

Une extraction `PARTIAL` doit être signalée à l'administrateur.

### 9.4 document_ingestion_attempts

| Colonne | Type | Contraintes |
|---|---|---|
| id | UUID | PK |
| ingestion_id | UUID | FK document_ingestions |
| attempt_number | INTEGER | NOT NULL |
| status | VARCHAR(30) | NOT NULL |
| error_code | VARCHAR(100) | NULL |
| started_at | TIMESTAMPTZ | NULL |
| completed_at | TIMESTAMPTZ | NULL |

UNIQUE `(ingestion_id, attempt_number)`.

### 9.5 document_indexations

| Colonne | Type | Contraintes |
|---|---|---|
| id | UUID | PK |
| document_version_id | UUID | FK knowledge_document_versions |
| status | VARCHAR(30) | NOT NULL |
| index_version | VARCHAR(100) | NULL |
| indexed_at | TIMESTAMPTZ | NULL |
| error_code | VARCHAR(100) | NULL |

### 9.6 knowledge_categories

| Colonne | Type | Contraintes |
|---|---|---|
| id | UUID | PK |
| parent_id | UUID | FK knowledge_categories |
| name | VARCHAR(150) | NOT NULL |
| slug | VARCHAR(150) | UNIQUE |
| status | VARCHAR(20) | NOT NULL |

### 9.7 knowledge_tags

| Colonne | Type | Contraintes |
|---|---|---|
| id | UUID | PK |
| name | VARCHAR(100) | NOT NULL |
| slug | VARCHAR(100) | UNIQUE |
| status | VARCHAR(20) | NOT NULL |

### 9.8 knowledge_document_categories

| Colonne | Type | Contraintes |
|---|---|---|
| document_id | UUID | FK knowledge_documents |
| category_id | UUID | FK knowledge_categories |

PK `(document_id, category_id)`.

### 9.9 knowledge_document_tags

| Colonne | Type | Contraintes |
|---|---|---|
| document_id | UUID | FK knowledge_documents |
| tag_id | UUID | FK knowledge_tags |

PK `(document_id, tag_id)`.

## 10. Schéma ai_engine — Propriété Python

Ce schéma est exclusivement administré par le service Python.

Laravel ne modifie pas directement ses tables.

### 10.1 document_extractions

| Colonne | Type | Contraintes |
|---|---|---|
| id | UUID | PK |
| document_version_id | UUID | NOT NULL |
| status | VARCHAR(30) | NOT NULL |
| extraction_engine | VARCHAR(100) | NULL |
| extraction_version | VARCHAR(100) | NULL |
| error_code | VARCHAR(100) | NULL |
| created_at | TIMESTAMPTZ | NOT NULL |
| completed_at | TIMESTAMPTZ | NULL |

### 10.2 document_sections

| Colonne | Type | Contraintes |
|---|---|---|
| id | UUID | PK |
| document_version_id | UUID | NOT NULL |
| parent_section_id | UUID | NULL |
| title | TEXT | NULL |
| section_path | TEXT | NULL |
| position | INTEGER | NOT NULL |
| page_start | INTEGER | NULL |
| page_end | INTEGER | NULL |
| created_at | TIMESTAMPTZ | NOT NULL |

### 10.3 document_chunks

| Colonne | Type | Contraintes |
|---|---|---|
| id | UUID | PK |
| document_version_id | UUID | NOT NULL |
| section_id | UUID | NULL |
| parent_chunk_id | UUID | NULL |
| content | TEXT | NOT NULL |
| sequence | INTEGER | NOT NULL |
| page_start | INTEGER | NULL |
| page_end | INTEGER | NULL |
| token_count | INTEGER | NULL |
| metadata | JSONB | NOT NULL, DEFAULT '{}' |
| created_at | TIMESTAMPTZ | NOT NULL |

### 10.4 chunk_embeddings

| Colonne | Type | Contraintes |
|---|---|---|
| chunk_id | UUID | PK, FK document_chunks |
| embedding | VECTOR(N) | NOT NULL |
| embedding_model | VARCHAR(150) | NOT NULL |
| created_at | TIMESTAMPTZ | NOT NULL |

N représente la dimension du modèle d'embedding retenu.

Une seule configuration d'embedding est utilisée dans le MVP.

### 10.5 retrieval_indexes

| Colonne | Type | Contraintes |
|---|---|---|
| id | UUID | PK |
| index_name | VARCHAR(150) | NOT NULL |
| index_version | VARCHAR(100) | NOT NULL |
| embedding_model | VARCHAR(150) | NOT NULL |
| status | VARCHAR(30) | NOT NULL |
| created_at | TIMESTAMPTZ | NOT NULL |
| activated_at | TIMESTAMPTZ | NULL |

### 10.6 retrieval_traces

| Colonne | Type | Contraintes |
|---|---|---|
| id | UUID | PK |
| request_id | UUID | NOT NULL |
| correlation_id | UUID | NOT NULL |
| retrieval_strategy | VARCHAR(50) | NOT NULL |
| duration_ms | BIGINT | NULL |
| result_count | INTEGER | NOT NULL |
| created_at | TIMESTAMPTZ | NOT NULL |

Les traces ne doivent pas stocker le texte des requêtes personnelles.

### 10.7 retrieval_trace_results

| Colonne | Type | Contraintes |
|---|---|---|
| id | UUID | PK |
| retrieval_trace_id | UUID | FK retrieval_traces |
| chunk_id | UUID | NOT NULL |
| rank | INTEGER | NOT NULL |
| relevance_score | NUMERIC(8,6) | NULL |
| created_at | TIMESTAMPTZ | NOT NULL |

## 11. Schéma ai_governance

### 11.1 ai_models

| Colonne | Type | Contraintes |
|---|---|---|
| id | UUID | PK |
| runtime | VARCHAR(30) | NOT NULL |
| model_name | VARCHAR(255) | NOT NULL |
| model_tag | VARCHAR(100) | NULL |
| checksum | VARCHAR(255) | NULL |
| capabilities | JSONB | NOT NULL |
| resource_requirements | JSONB | NOT NULL |
| status | VARCHAR(20) | NOT NULL |
| created_at | TIMESTAMPTZ | NOT NULL |
| updated_at | TIMESTAMPTZ | NOT NULL |

### 11.2 ai_model_configurations

| Colonne | Type | Contraintes |
|---|---|---|
| id | UUID | PK |
| name | VARCHAR(255) | NOT NULL |
| usage_type | VARCHAR(50) | NOT NULL |
| current_version_id | UUID | NULL |
| status | VARCHAR(20) | NOT NULL |
| created_at | TIMESTAMPTZ | NOT NULL |
| updated_at | TIMESTAMPTZ | NOT NULL |

### 11.3 ai_model_configuration_versions

| Colonne | Type | Contraintes |
|---|---|---|
| id | UUID | PK |
| configuration_id | UUID | FK ai_model_configurations |
| version_number | INTEGER | NOT NULL |
| model_id | UUID | FK ai_models |
| parameters | JSONB | NOT NULL |
| resource_limits | JSONB | NOT NULL |
| created_at | TIMESTAMPTZ | NOT NULL |

UNIQUE `(configuration_id, version_number)`.

### 11.4 prompt_definitions

| Colonne | Type | Contraintes |
|---|---|---|
| id | UUID | PK |
| name | VARCHAR(255) | NOT NULL |
| usage_type | VARCHAR(50) | NOT NULL |
| description | TEXT | NULL |
| current_version_id | UUID | NULL |
| status | VARCHAR(20) | NOT NULL |

### 11.5 prompt_versions

| Colonne | Type | Contraintes |
|---|---|---|
| id | UUID | PK |
| prompt_definition_id | UUID | FK prompt_definitions |
| version_number | INTEGER | NOT NULL |
| content | TEXT | NOT NULL |
| variables | JSONB | NOT NULL |
| created_at | TIMESTAMPTZ | NOT NULL |

UNIQUE `(prompt_definition_id, version_number)`.

### 11.6 governance_policies

| Colonne | Type | Contraintes |
|---|---|---|
| id | UUID | PK |
| name | VARCHAR(255) | NOT NULL |
| policy_type | VARCHAR(50) | NOT NULL |
| current_version_id | UUID | NULL |
| status | VARCHAR(20) | NOT NULL |

### 11.7 governance_policy_versions

| Colonne | Type | Contraintes |
|---|---|---|
| id | UUID | PK |
| policy_id | UUID | FK governance_policies |
| version_number | INTEGER | NOT NULL |
| configuration | JSONB | NOT NULL |
| created_at | TIMESTAMPTZ | NOT NULL |

UNIQUE `(policy_id, version_number)`.

### 11.8 safety_policy_versions

| Colonne | Type | Contraintes |
|---|---|---|
| id | UUID | PK |
| version_number | INTEGER | NOT NULL |
| configuration | JSONB | NOT NULL |
| created_at | TIMESTAMPTZ | NOT NULL |

Les règles critiques protégées dans le code ne sont pas modifiables par cette table.

### 11.9 emergency_resources

| Colonne | Type | Contraintes |
|---|---|---|
| id | UUID | PK |
| country_code | CHAR(2) | NOT NULL |
| resource_type | VARCHAR(40) | NOT NULL |
| name | VARCHAR(255) | NOT NULL |
| phone | VARCHAR(50) | NULL |
| description | TEXT | NULL |
| url | TEXT | NULL |
| availability | VARCHAR(100) | NULL |
| verified_at | TIMESTAMPTZ | NULL |
| status | VARCHAR(20) | NOT NULL |

### 11.10 safety_interventions

| Colonne | Type | Contraintes |
|---|---|---|
| id | UUID | PK |
| user_id | UUID | NOT NULL |
| conversation_id | UUID | NULL |
| safety_level | SMALLINT | NOT NULL |
| intervention_type | VARCHAR(50) | NOT NULL |
| policy_version_id | UUID | NULL |
| created_at | TIMESTAMPTZ | NOT NULL |

Aucun contenu conversationnel sensible n'est stocké dans cette table.

Les interventions de sécurité sont des données de santé sensibles : elles restent cloisonnées par utilisateur, n'apparaissent jamais dans les logs techniques ni dans les événements d'intégration, et sont incluses dans l'export et la suppression des données de l'utilisateur.

### 11.11 evaluation_suites

| Colonne | Type | Contraintes |
|---|---|---|
| id | UUID | PK |
| name | VARCHAR(255) | NOT NULL |
| description | TEXT | NULL |
| current_version_id | UUID | NULL |
| status | VARCHAR(20) | NOT NULL |

### 11.12 evaluation_suite_versions

| Colonne | Type | Contraintes |
|---|---|---|
| id | UUID | PK |
| suite_id | UUID | FK evaluation_suites |
| version_number | INTEGER | NOT NULL |
| created_at | TIMESTAMPTZ | NOT NULL |

UNIQUE `(suite_id, version_number)`.

### 11.13 evaluation_scenarios

| Colonne | Type | Contraintes |
|---|---|---|
| id | UUID | PK |
| suite_version_id | UUID | FK evaluation_suite_versions |
| scenario_key | VARCHAR(100) | NOT NULL |
| category | VARCHAR(50) | NOT NULL |
| input_payload | JSONB | NOT NULL |
| expected_behavior | JSONB | NOT NULL |
| critical | BOOLEAN | NOT NULL, DEFAULT FALSE |

### 11.14 evaluation_runs

| Colonne | Type | Contraintes |
|---|---|---|
| id | UUID | PK |
| suite_version_id | UUID | FK evaluation_suite_versions |
| status | VARCHAR(30) | NOT NULL |
| release_id | UUID | NULL |
| started_at | TIMESTAMPTZ | NOT NULL |
| completed_at | TIMESTAMPTZ | NULL |

### 11.15 evaluation_results

| Colonne | Type | Contraintes |
|---|---|---|
| id | UUID | PK |
| evaluation_run_id | UUID | FK evaluation_runs |
| scenario_id | UUID | FK evaluation_scenarios |
| status | VARCHAR(20) | NOT NULL |
| score | NUMERIC(8,4) | NULL |
| details | JSONB | NOT NULL |
| created_at | TIMESTAMPTZ | NOT NULL |

### 11.16 ai_configuration_releases

| Colonne | Type | Contraintes |
|---|---|---|
| id | UUID | PK |
| release_number | INTEGER | NOT NULL, UNIQUE |
| status | VARCHAR(20) | NOT NULL |
| created_at | TIMESTAMPTZ | NOT NULL |
| evaluated_at | TIMESTAMPTZ | NULL |
| approved_at | TIMESTAMPTZ | NULL |

### 11.17 ai_configuration_release_components

| Colonne | Type | Contraintes |
|---|---|---|
| id | UUID | PK |
| release_id | UUID | FK ai_configuration_releases |
| component_type | VARCHAR(40) | NOT NULL |
| component_id | UUID | NOT NULL |
| component_version_id | UUID | NOT NULL |

### 11.18 active_ai_configuration

| Colonne | Type | Contraintes |
|---|---|---|
| id | SMALLINT | PK, valeur unique 1 |
| release_id | UUID | FK ai_configuration_releases |
| activated_at | TIMESTAMPTZ | NOT NULL |

### 11.19 ai_release_activations

| Colonne | Type | Contraintes |
|---|---|---|
| id | UUID | PK |
| release_id | UUID | FK ai_configuration_releases |
| activated_at | TIMESTAMPTZ | NOT NULL |
| deactivated_at | TIMESTAMPTZ | NULL |
| activation_reason | VARCHAR(255) | NULL |

La référence vers la release utilisée par chaque génération conversationnelle est conservée dans `companion.generation_attempts.ai_release_id`. Pour les bilans, elle est conservée dans `user_practice.progress_reviews.ai_release_id`.

## 12. Schéma platform

### 12.1 outbox_events

| Colonne | Type | Contraintes |
|---|---|---|
| id | UUID | PK |
| event_id | UUID | UNIQUE |
| event_type | VARCHAR(150) | NOT NULL |
| event_version | INTEGER | NOT NULL |
| producer | VARCHAR(100) | NOT NULL |
| aggregate_type | VARCHAR(100) | NOT NULL |
| aggregate_id | UUID | NOT NULL |
| aggregate_version | BIGINT | NULL |
| occurred_at | TIMESTAMPTZ | NOT NULL |
| correlation_id | UUID | NOT NULL |
| causation_id | UUID | NULL |
| payload | JSONB | NOT NULL |
| metadata | JSONB | NOT NULL |
| published_at | TIMESTAMPTZ | NULL |
| created_at | TIMESTAMPTZ | NOT NULL |

Index sur `(published_at, created_at)`.

Le payload ne doit contenir aucune donnée personnelle sensible.

### 12.2 processed_events

| Colonne | Type | Contraintes |
|---|---|---|
| consumer | VARCHAR(150) | NOT NULL |
| event_id | UUID | NOT NULL |
| processed_at | TIMESTAMPTZ | NOT NULL |

PK `(consumer, event_id)`.

### 12.3 idempotency_keys

| Colonne | Type | Contraintes |
|---|---|---|
| id | UUID | PK |
| scope | VARCHAR(100) | NOT NULL |
| idempotency_key | VARCHAR(255) | NOT NULL |
| request_hash | CHAR(64) | NOT NULL |
| status | VARCHAR(20) | NOT NULL |
| response_reference | UUID | NULL |
| created_at | TIMESTAMPTZ | NOT NULL |
| expires_at | TIMESTAMPTZ | NOT NULL |

UNIQUE `(scope, idempotency_key)`.

### 12.4 async_operations

| Colonne | Type | Contraintes |
|---|---|---|
| id | UUID | PK |
| operation_type | VARCHAR(100) | NOT NULL |
| owner_context | VARCHAR(100) | NOT NULL |
| owner_id | UUID | NULL |
| status | VARCHAR(30) | NOT NULL |
| progress | SMALLINT | NULL |
| generation_id | UUID | NULL |
| correlation_id | UUID | NOT NULL |
| error_code | VARCHAR(100) | NULL |
| created_at | TIMESTAMPTZ | NOT NULL |
| started_at | TIMESTAMPTZ | NULL |
| completed_at | TIMESTAMPTZ | NULL |

Cette table suit les opérations techniques transversales. Elle ne remplace pas les statuts métier des agrégats.

Pour les traitements IA de fond, chaque appel au moteur constitue une tentative suivie ici : `generation_id` est l'identifiant transmis à Python ; une tentative préemptée prend le statut terminal `PREEMPTED` (ou `error_code = PREEMPTED`) et n'est jamais reprise ; la relance crée une nouvelle ligne avec un nouveau `generation_id`.

### 12.5 Tables techniques Laravel

Les tables standards suivantes sont créées dans le schéma `platform` (ou le schéma public selon la configuration Laravel), sans duplication métier :

- `migrations`
- `cache`
- `cache_locks`
- `jobs`
- `job_batches`
- `failed_jobs`

Le driver de files PostgreSQL est retenu pour le MVP. Les jobs ne transportent que des identifiants, jamais de contenu personnel.

Aucune table technique redondante ne doit être créée pour remplacer ces mécanismes.

### 12.6 backups

| Colonne | Type | Contraintes |
|---|---|---|
| id | UUID | PK |
| status | VARCHAR(30) | NOT NULL |
| storage_key | TEXT | NOT NULL |
| checksum | CHAR(64) | NULL |
| size_bytes | BIGINT | NULL |
| deletion_ledger_checkpoint | BIGINT | NULL |
| started_at | TIMESTAMPTZ | NOT NULL |
| completed_at | TIMESTAMPTZ | NULL |

`deletion_ledger_checkpoint` conserve le numéro de séquence du registre des suppressions pris sous verrou avant le début de la capture.

### 12.7 backup_components

| Colonne | Type | Contraintes |
|---|---|---|
| id | UUID | PK |
| backup_id | UUID | FK backups |
| component_type | VARCHAR(50) | NOT NULL |
| storage_key | TEXT | NOT NULL |
| checksum | CHAR(64) | NULL |
| size_bytes | BIGINT | NULL |

### 12.8 restore_operations

| Colonne | Type | Contraintes |
|---|---|---|
| id | UUID | PK |
| backup_id | UUID | FK backups |
| status | VARCHAR(30) | NOT NULL |
| started_at | TIMESTAMPTZ | NOT NULL |
| completed_at | TIMESTAMPTZ | NULL |
| error_code | VARCHAR(100) | NULL |

### 12.9 export_requests

| Colonne | Type | Contraintes |
|---|---|---|
| id | UUID | PK |
| user_id | UUID | NOT NULL |
| status | VARCHAR(30) | NOT NULL |
| storage_key | TEXT | NULL |
| expires_at | TIMESTAMPTZ | NULL |
| created_at | TIMESTAMPTZ | NOT NULL |
| completed_at | TIMESTAMPTZ | NULL |

### 12.10 deletion_requests

| Colonne | Type | Contraintes |
|---|---|---|
| id | UUID | PK |
| user_id | UUID | NOT NULL |
| status | VARCHAR(30) | NOT NULL |
| requested_at | TIMESTAMPTZ | NOT NULL |
| confirmed_at | TIMESTAMPTZ | NULL |
| completed_at | TIMESTAMPTZ | NULL |

### 12.11 deletion_tasks

| Colonne | Type | Contraintes |
|---|---|---|
| id | UUID | PK |
| deletion_request_id | UUID | FK deletion_requests |
| target_context | VARCHAR(100) | NOT NULL |
| status | VARCHAR(30) | NOT NULL |
| attempts | INTEGER | NOT NULL, DEFAULT 0 |
| error_code | VARCHAR(100) | NULL |
| completed_at | TIMESTAMPTZ | NULL |

### 12.12 user_notifications

| Colonne | Type | Contraintes |
|---|---|---|
| id | UUID | PK |
| user_id | UUID | NOT NULL |
| notification_type | VARCHAR(100) | NOT NULL |
| title | VARCHAR(255) | NOT NULL |
| body | TEXT | NULL |
| read_at | TIMESTAMPTZ | NULL |
| created_at | TIMESTAMPTZ | NOT NULL |

La table s'appelle `user_notifications` afin d'éviter toute collision avec la table `notifications` éventuelle d'un canal Laravel.

### 12.13 notification_preferences

| Colonne | Type | Contraintes |
|---|---|---|
| id | UUID | PK |
| user_id | UUID | UNIQUE |
| enabled | BOOLEAN | NOT NULL, DEFAULT FALSE |
| categories | JSONB | NOT NULL, DEFAULT '[]' |
| frequency | VARCHAR(30) | NULL |
| quiet_hours | JSONB | NOT NULL, DEFAULT '{}' |
| preferences | JSONB | NOT NULL, DEFAULT '{}' |
| updated_at | TIMESTAMPTZ | NOT NULL |

`quiet_hours` décrit les périodes de silence (fuseau, plages horaires). Les notifications restent facultatives et non culpabilisantes.

### 12.14 maintenance_operations

| Colonne | Type | Contraintes |
|---|---|---|
| id | UUID | PK |
| operation_type | VARCHAR(100) | NOT NULL |
| status | VARCHAR(30) | NOT NULL |
| parameters | JSONB | NOT NULL |
| started_at | TIMESTAMPTZ | NULL |
| completed_at | TIMESTAMPTZ | NULL |
| error_code | VARCHAR(100) | NULL |

### 12.15 telemetry_configurations

| Colonne | Type | Contraintes |
|---|---|---|
| id | UUID | PK |
| enabled | BOOLEAN | NOT NULL, DEFAULT FALSE |
| configuration | JSONB | NOT NULL |
| updated_at | TIMESTAMPTZ | NOT NULL |

### 12.16 Registre des suppressions (deletion ledger)

Le registre des suppressions n'est pas une table PostgreSQL. C'est un fichier JSON Lines en ajout seul, stocké sur un volume dédié hors de la base restaurée et hors des volumes de fichiers restaurés.

Chaque ligne contient uniquement des identifiants et des données techniques minimales (type d'objet, identifiant, horodatage, numéro de séquence, état). Aucun contenu personnel n'y figure.

Règles :

1. Numérotation et écriture sérialisées sous verrou exclusif : une seule écriture à la fois.
2. Entrée « demandée » durable avant que le système n'accuse réception de la suppression : écriture, `flush` et `fsync` du fichier ; `fsync` du répertoire à la création du fichier.
3. Entrée « terminée » durable après exécution de la suppression physique.
4. Avant chaque sauvegarde, un point de référence (checkpoint) est calculé sous le même verrou, puis inscrit durablement dans le manifeste de sauvegarde (`backups.deletion_ledger_checkpoint`) avant le début de la capture de la base et des fichiers.
5. Après restauration, rejeu ordonné et idempotent des entrées postérieures au point de référence associé à la sauvegarde.
6. Le registre est recopié sur le support externe à chaque sauvegarde. Un risque résiduel demeure entre deux recopies ; il est accepté et documenté.

## 13. Indexation et performances

Les index doivent être ajoutés selon les usages réels.

Principes :

- Index sur les FK fréquemment utilisées.
- Index composites adaptés aux filtres utilisateur et statuts.
- Index uniques pour les invariants d'unicité.
- Index partiels pour les états actifs lorsque cela est pertinent.
- Index GIN pour les colonnes JSONB interrogées.
- Index plein texte PostgreSQL pour la recherche documentaire.
- Index vectoriel pgvector adapté au volume et aux performances mesurées.

Le choix entre HNSW et IVFFlat sera effectué après évaluation sur le corpus réel.

Une recherche vectorielle exacte peut suffire au départ pour un corpus limité.

Les index ne doivent pas être ajoutés systématiquement sans besoin mesuré.

## 14. Contraintes intercontextes

Les relations entre contextes utilisent des identifiants et des contrats applicatifs.

Les clés étrangères sont principalement utilisées à l'intérieur d'un même contexte.

Les références vers d'autres contextes sont validées au niveau applicatif.

Les tables Python ne disposent pas de clés étrangères vers les tables métier Laravel.

Les opérations de suppression doivent prendre en charge les références externes et les données dérivées.

## 15. Sécurité de la persistance

- Les secrets ne sont jamais stockés dans les tables de configuration en clair lorsqu'ils peuvent être évités.
- Les données sensibles ne sont pas recopiées dans les logs.
- Les rôles SQL respectent le principe du moindre privilège.
- Les sauvegardes sont chiffrées.
- Les clés de chiffrement sont séparées des sauvegardes.
- Les accès Python sont limités au schéma `ai_engine` et aux ressources techniques explicitement nécessaires.
- Les fichiers originaux sont conservés dans un stockage persistant privé.

## 16. Migrations

Les rôles, schémas et extensions sont créés par le script d'initialisation PostgreSQL.

Les migrations sont versionnées et exécutées par Laravel pour les schémas dont il est propriétaire. Elles se limitent aux tables, index et contraintes.

Le service Python gère les migrations de son propre schéma `ai_engine` via Alembic.

Les modifications de schéma doivent respecter les principes suivants :

- Évolution progressive.
- Compatibilité avec les données existantes lorsque nécessaire.
- Pas de suppression destructive sans migration de données contrôlée.
- Tests avec une instance PostgreSQL réelle.
- Vérification des contraintes et index après migration.

SQLite seul ne suffit pas pour valider les comportements PostgreSQL spécifiques, notamment JSONB, pgvector, index partiels et contraintes.

## 17. Points à finaliser avant implémentation

Les éléments suivants nécessitent une validation technique lors de la conception détaillée :

1. Dimension exacte du modèle d'embedding et type `VECTOR(N)` — rattaché à KnowledgeBase et au banc phase 1b.
2. Stratégie d'index vectoriel — rattachée à KnowledgeBase et au banc phase 1b.
3. Détails des index JSONB.
4. Politique de rétention globale des tentatives et traces — partiellement résolue pour `generated_content` (7 jours après INTERRUPTED, CANCELLED ou FAILED) ; le reste en phase 8.
5. Politique de suppression physique et logique par contexte — principes posés ; détail opérationnel en phase d'implémentation.
6. Configuration exacte du driver de session Laravel — résolu : `user_id UUID NULL`, driver compatible.
7. Contraintes composites supplémentaires sur les références versionnées.
8. Stratégie de migration initiale des schémas — résolu : script d'initialisation pour rôles, schémas et extensions ; Laravel et Alembic pour les tables.
9. Stratégie de partitionnement éventuelle si les volumes le justifient.

---

**Statut :** dictionnaire de données — version de travail à valider.

**Dépendances :** `03-ARCHITECTURE-DDD.md`, `04-BOUNDED-CONTEXTS.md`, `05-CATALOGUE-CAS-UTILISATION.md` et `07-CONTRATS-INTERSERVICES.md`.

**Changements v2 :** UUID v7, Python sans aucun droit sur les schémas métier, rôles `ADMIN`/`USER`, `country_code`, `ai_release_id` sur les tentatives et bilans, index unique partiel des tentatives actives, rétention de `generated_content`, messages USER/SYSTEM créés COMPLETED, suppression de `memory_conflicts`, clé de question par version, auteur et validateur tracés, étapes sans contenu, `ai_retrieval_enabled` retirée de `resource_versions` et conservée dans `knowledge_documents`, statuts d'ingestion et de journal, tables techniques Laravel, `user_notifications`, périodes de silence, checkpoint du registre des suppressions, registre JSONL détaillé, Alembic pour `ai_engine` (Phase 0).
