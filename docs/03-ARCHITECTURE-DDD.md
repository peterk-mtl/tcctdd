# TCCTDD — Dictionnaire de données PostgreSQL

**Version :** 1.0  
**Statut :** Référence logique pour la conception des migrations  
**Dépendance :** `03-ARCHITECTURE-DDD.md`

## 1. Conventions générales

### 1.1. Identifiants

- Identifiants principaux : UUID.
- Génération des UUID côté application.
- Pas d'identifiant numérique auto-incrémenté pour les agrégats métier.
- Les clés composites sont autorisées pour les tables de liaison.

### 1.2. Dates

- Toutes les dates techniques sont stockées en UTC.
- Type PostgreSQL : `TIMESTAMPTZ`.
- Les dates de calendrier propres à l'utilisateur sont interprétées avec son fuseau horaire.

### 1.3. Types de données

- `TEXT` pour les contenus textuels de longueur variable.
- `VARCHAR` pour les statuts et valeurs catégorielles.
- `JSONB` pour les structures réellement variables ou configurables.
- `NUMERIC` pour les valeurs décimales nécessitant une précision contrôlée.
- `BIGINT` pour les compteurs et séquences.
- `BOOLEAN` pour les indicateurs binaires.
- `CHAR(64)` pour les empreintes SHA-256 hexadécimales.
- `VECTOR(n)` pour les embeddings, avec dimension définie par le modèle retenu.

Les types JSONB ne doivent pas remplacer systématiquement des tables relationnelles lorsque les données possèdent une identité, un cycle de vie ou des relations propres.

### 1.4. Statuts

Les statuts sont stockés sous forme de `VARCHAR`, avec validation par enums PHP et contraintes SQL `CHECK` lorsque cela est pertinent.

Les enums PostgreSQL natifs ne sont pas utilisés par défaut, afin de faciliter l'évolution des statuts.

### 1.5. Suppression et historique

La suppression logique est réservée aux objets ayant un intérêt métier à être archivés ou restaurés.

Les données supprimées doivent respecter la politique de conservation propre à leur contexte.

Les versions publiées et historiques importantes sont conservées selon les règles définies dans les documents fonctionnels.

### 1.6. Séparation des schémas

Les données sont réparties dans les schémas suivants :

- `identity`
- `companion`
- `personal_context`
- `tcc_content`
- `user_practice`
- `knowledge_base`
- `ai_governance`
- `platform`
- `ai_engine`

Chaque contexte reste propriétaire de ses tables.

---

## 2. Schéma `identity`

Ce schéma contient l'identité du compte et les préférences générales de l'application.

### 2.1. `users`

Compte utilisateur.

| Colonne | Type | Règle |
|---|---|---|
| id | UUID | PK |
| name | VARCHAR | NOT NULL |
| email | VARCHAR | UNIQUE, NOT NULL |
| email_verified_at | TIMESTAMPTZ | NULL |
| password | VARCHAR | NOT NULL |
| remember_token | VARCHAR | NULL |
| is_active | BOOLEAN | NOT NULL, DEFAULT TRUE |
| created_at | TIMESTAMPTZ | NOT NULL |
| updated_at | TIMESTAMPTZ | NOT NULL |

Compatible avec le système d'authentification Laravel, avec identifiant UUID.

### 2.2. `password_reset_tokens`

Jetons de réinitialisation de mot de passe compatibles avec Laravel.

| Colonne | Type | Règle |
|---|---|---|
| email | VARCHAR | PK |
| token | VARCHAR | NOT NULL |
| created_at | TIMESTAMPTZ | NULL |

### 2.3. `sessions`

Table de sessions conforme au driver de session Laravel retenu.

La structure standard Laravel doit être conservée. Ne pas y ajouter arbitrairement des colonnes qui nécessiteraient un handler personnalisé.

### 2.4. `session_activity`

Suivi de l'activité réelle permettant le verrouillage automatique.

| Colonne | Type | Règle |
|---|---|---|
| session_id | VARCHAR | PK, référence session |
| last_interaction_at | TIMESTAMPTZ | NOT NULL |
| locked_at | TIMESTAMPTZ | NULL |

Les simples requêtes de polling ou opérations techniques ne doivent pas nécessairement remettre à zéro le délai d'inactivité.

Le verrouillage intervient après 15 minutes d'inactivité, durée configurable.

### 2.5. `user_preferences`

Préférences générales de compte.

| Colonne | Type | Règle |
|---|---|---|
| user_id | UUID | PK, FK users |
| locale | VARCHAR | NOT NULL |
| timezone | VARCHAR | NOT NULL |
| ui_preferences | JSONB | NOT NULL |
| privacy_preferences | JSONB | NOT NULL |
| created_at | TIMESTAMPTZ | NOT NULL |
| updated_at | TIMESTAMPTZ | NOT NULL |

Les préférences métier spécifiques restent dans leurs contextes respectifs.

---

## 3. Schéma `companion`

### 3.1. `conversations`

Agrégat Conversation.

| Colonne | Type | Règle |
|---|---|---|
| id | UUID | PK |
| user_id | UUID | propriétaire |
| title | VARCHAR | NULL |
| conversation_type | VARCHAR | GENERAL / THEMATIC |
| status | VARCHAR | ACTIVE / ARCHIVED / DELETED |
| last_message_sequence | BIGINT | NOT NULL, DEFAULT 0 |
| created_at | TIMESTAMPTZ | NOT NULL |
| updated_at | TIMESTAMPTZ | NOT NULL |
| archived_at | TIMESTAMPTZ | NULL |
| deleted_at | TIMESTAMPTZ | NULL |

Index : `(user_id, status, updated_at)`.

`last_message_sequence` est incrémenté sous verrou transactionnel lors de l'ajout d'un message.

### 3.2. `messages`

Entité Message.

| Colonne | Type | Règle |
|---|---|---|
| id | UUID | PK |
| conversation_id | UUID | FK conversations |
| sequence | BIGINT | NOT NULL |
| role | VARCHAR | USER / ASSISTANT / SYSTEM |
| status | VARCHAR | statut de traitement |
| content | TEXT | contenu direct, nullable |
| selected_attempt_id | UUID | nullable |
| created_at | TIMESTAMPTZ | NOT NULL |
| updated_at | TIMESTAMPTZ | NOT NULL |
| deleted_at | TIMESTAMPTZ | NULL |

Contrainte unique : `(conversation_id, sequence)`.

Règles :
- le contenu des messages utilisateur est conservé ici ;
- le contenu généré par l'assistant est conservé dans `generation_attempts` ;
- `selected_attempt_id` désigne la tentative retenue pour l'affichage ;
- un message utilisateur terminé est immuable ;
- les messages assistant peuvent avoir plusieurs tentatives.

Le statut doit notamment distinguer les messages en génération, terminés, interrompus, échoués, annulés et supprimés.

### 3.3. `generation_attempts`

Agrégat GenerationAttempt.

| Colonne | Type | Règle |
|---|---|---|
| id | UUID | PK |
| message_id | UUID | FK messages |
| attempt_number | INTEGER | NOT NULL |
| status | VARCHAR | PENDING / STREAMING / COMPLETED / INTERRUPTED / FAILED / CANCELLED |
| ai_release_id | UUID | release utilisée |
| generated_content | TEXT | contenu de la tentative |
| error_code | VARCHAR | NULL |
| started_at | TIMESTAMPTZ | NULL |
| completed_at | TIMESTAMPTZ | NULL |
| duration_ms | BIGINT | NULL |
| input_tokens | BIGINT | NULL |
| output_tokens | BIGINT | NULL |
| created_at | TIMESTAMPTZ | NOT NULL |
| updated_at | TIMESTAMPTZ | NOT NULL |

Contraintes :
- unique `(message_id, attempt_number)` ;
- un seul attempt actif par message, via index unique partiel ;
- unique `(id, message_id)` pour permettre la contrainte composite de sélection.

La relation `messages(selected_attempt_id, id)` doit référencer `generation_attempts(id, message_id)`. Cela garantit qu'une tentative sélectionnée appartient bien au message concerné.

Les tentatives terminées sont immuables.

Les contenus interrompus ou échoués sont conservés temporairement, avec suppression automatique au plus tard après sept jours lorsque leur conservation n'est plus nécessaire.

---

## 4. Schéma `personal_context`

### 4.1. `memories`

Agrégat Memory.

| Colonne | Type | Règle |
|---|---|---|
| id | UUID | PK |
| user_id | UUID | propriétaire |
| category | VARCHAR | catégorie mémoire |
| content | JSONB | contenu structuré |
| status | VARCHAR | PROPOSED / ACTIVE / REJECTED / ARCHIVED / SUPERSEDED |
| epistemic_status | VARCHAR | HYPOTHESIS / DECLARED / CONFIRMED / CONTESTED |
| confidence | NUMERIC(4,3) | NULL, entre 0 et 1 |
| superseded_by_memory_id | UUID | NULL |
| created_at | TIMESTAMPTZ | NOT NULL |
| updated_at | TIMESTAMPTZ | NOT NULL |
| archived_at | TIMESTAMPTZ | NULL |

Catégories initiales :
- PERSONAL_CONTEXT
- RECURRING_DIFFICULTY
- TRIGGER
- THOUGHT_PATTERN
- EMOTIONAL_PATTERN
- GOAL
- HELPFUL_STRATEGY
- RESOURCE
- PROGRESS

Le statut de traitement et le statut épistémique sont indépendants.

Une mémoire peut être ACTIVE et DECLARED.

Une mémoire proposée par l'IA peut être PROPOSED et HYPOTHESIS.

Une mémoire SUPERSEDED doit référencer sa remplaçante.

### 4.2. `memory_sources`

Provenance des mémoires.

| Colonne | Type | Règle |
|---|---|---|
| id | UUID | PK |
| memory_id | UUID | FK memories |
| source_type | VARCHAR | type de source |
| source_id | UUID | référence externe |
| source_version | VARCHAR | NULL |
| source_excerpt_reference | JSONB | référence structurée facultative |
| created_at | TIMESTAMPTZ | NOT NULL |

Contrainte d'unicité sur la combinaison mémoire, type de source, identifiant et version.

Ne pas conserver de copie brute de l'extrait personnel dans cette table.

### 4.3. `memory_revisions`

Historique des révisions.

| Colonne | Type | Règle |
|---|---|---|
| id | UUID | PK |
| memory_id | UUID | FK memories |
| revision_number | INTEGER | NOT NULL |
| content | JSONB | NOT NULL |
| status | VARCHAR | NOT NULL |
| epistemic_status | VARCHAR | NOT NULL |
| change_reason | TEXT | NULL |
| changed_by_type | VARCHAR | USER / AI / SYSTEM |
| created_at | TIMESTAMPTZ | NOT NULL |

Unique `(memory_id, revision_number)`.

### 4.4. `memory_relations`

Agrégat MemoryRelation.

| Colonne | Type | Règle |
|---|---|---|
| id | UUID | PK |
| user_id | UUID | propriétaire |
| source_memory_id | UUID | référence mémoire |
| target_memory_id | UUID | référence mémoire |
| relation_type | VARCHAR | type de relation |
| status | VARCHAR | statut |
| epistemic_status | VARCHAR | statut épistémique |
| description | TEXT | NULL |
| created_at | TIMESTAMPTZ | NOT NULL |
| updated_at | TIMESTAMPTZ | NOT NULL |

Types initiaux :
- TRIGGERED_BY
- ASSOCIATED_WITH
- SUPPORTED_BY
- CONTRADICTS
- REINFORCES
- MITIGATED_BY
- CONTRIBUTES_TO
- RELATED_TO

Pas de relation vers soi-même.

Les deux mémoires doivent appartenir au même utilisateur.

Une relation active ne doit pas être dupliquée avec la même direction et le même type.

Les relations proposées par l'IA restent des hypothèses tant qu'elles ne sont pas acceptées.

### 4.5. `memory_conflicts`

Agrégat MemoryConflict.

| Colonne | Type | Règle |
|---|---|---|
| id | UUID | PK |
| user_id | UUID | propriétaire |
| status | VARCHAR | DETECTED / PRESENTED / ACKNOWLEDGED / RESOLVED / DISMISSED |
| description | TEXT | NULL |
| detected_at | TIMESTAMPTZ | NOT NULL |
| acknowledged_at | TIMESTAMPTZ | NULL |
| resolved_at | TIMESTAMPTZ | NULL |
| created_at | TIMESTAMPTZ | NOT NULL |
| updated_at | TIMESTAMPTZ | NOT NULL |

### 4.6. `memory_conflict_items`

Association entre un conflit et les mémoires concernées.

| Colonne | Type | Règle |
|---|---|---|
| conflict_id | UUID | FK memory_conflicts |
| memory_id | UUID | référence mémoire |
| position | INTEGER | NOT NULL |
| role | VARCHAR | NOT NULL |

PK composite `(conflict_id, memory_id)`.

Les mémoires associées doivent appartenir au même utilisateur que le conflit.

### 4.7. `memory_consolidations`

Agrégat MemoryConsolidation.

| Colonne | Type | Règle |
|---|---|---|
| id | UUID | PK |
| user_id | UUID | propriétaire |
| type | VARCHAR | ATTACH_SOURCE / MERGE / REVISE |
| status | VARCHAR | PROPOSED / AUTO_APPLIED / ACCEPTED / REJECTED / EXPIRED |
| proposed_content | JSONB | NULL |
| reason | TEXT | NULL |
| proposed_at | TIMESTAMPTZ | NOT NULL |
| decided_at | TIMESTAMPTZ | NULL |
| decided_by_type | VARCHAR | NULL |
| created_at | TIMESTAMPTZ | NOT NULL |

### 4.8. `memory_consolidation_items`

| Colonne | Type | Règle |
|---|---|---|
| consolidation_id | UUID | FK memory_consolidations |
| memory_id | UUID | référence mémoire |
| role | VARCHAR | SOURCE / TARGET / RESULT |

PK composite `(consolidation_id, memory_id, role)`.

### 4.9. `companion_preferences`

Préférences comportementales du compagnon.

| Colonne | Type | Règle |
|---|---|---|
| id | UUID | PK |
| user_id | UUID | UNIQUE |
| explicit_preferences | JSONB | NOT NULL |
| inferred_preferences | JSONB | NOT NULL |
| adaptive_enabled | BOOLEAN | DEFAULT TRUE |
| updated_at | TIMESTAMPTZ | NOT NULL |

Les préférences inférées doivent rester consultables et corrigibles.

### 4.10. `context_snapshots`

Trace de contextualisation.

| Colonne | Type | Règle |
|---|---|---|
| id | UUID | PK |
| user_id | UUID | propriétaire |
| conversation_id | UUID | NULL |
| assembly_version | VARCHAR | NOT NULL |
| selected_references | JSONB | NOT NULL |
| created_at | TIMESTAMPTZ | NOT NULL |

Cette table conserve les références et versions utilisées, pas une copie complète des contenus personnels injectés.

---

## 5. Schéma `user_practice`

### 5.1. `exercise_runs`

Agrégat ExerciseRun.

| Colonne | Type | Règle |
|---|---|---|
| id | UUID | PK |
| user_id | UUID | propriétaire |
| exercise_id | UUID | référence TccContent |
| exercise_version_id | UUID | version exacte |
| status | VARCHAR | IN_PROGRESS / COMPLETED / ABANDONED |
| started_at | TIMESTAMPTZ | NOT NULL |
| completed_at | TIMESTAMPTZ | NULL |
| abandoned_at | TIMESTAMPTZ | NULL |
| version | BIGINT | NOT NULL, DEFAULT 1 |
| created_at | TIMESTAMPTZ | NOT NULL |
| updated_at | TIMESTAMPTZ | NOT NULL |

La version est utilisée pour le contrôle de concurrence optimiste.

### 5.2. `exercise_answers`

Réponses associées à une réalisation.

| Colonne | Type | Règle |
|---|---|---|
| id | UUID | PK |
| exercise_run_id | UUID | FK exercise_runs |
| question_id | UUID | référence question versionnée |
| answer_value | JSONB | NOT NULL |
| answered_at | TIMESTAMPTZ | NOT NULL |
| updated_at | TIMESTAMPTZ | NOT NULL |

Unique `(exercise_run_id, question_id)`.

La validité de la réponse est vérifiée selon le type de question et la version exacte de l'exercice.

Les réponses des blocs conditionnels masqués sont conservées, mais exclues du contexte IA tant que le bloc reste masqué.

Une réalisation terminée rend ses réponses immuables.

### 5.3. `journal_entries`

Agrégat JournalEntry.

| Colonne | Type | Règle |
|---|---|---|
| id | UUID | PK |
| user_id | UUID | propriétaire |
| title | VARCHAR | NULL |
| entry_type | VARCHAR | catégorie |
| event_at | TIMESTAMPTZ | NULL |
| ai_access_policy | VARCHAR | DEFAULT / ALLOWED / DENIED |
| status | VARCHAR | ACTIVE / ARCHIVED / DELETED |
| current_version | INTEGER | NOT NULL |
| created_at | TIMESTAMPTZ | NOT NULL |
| updated_at | TIMESTAMPTZ | NOT NULL |
| deleted_at | TIMESTAMPTZ | NULL |

La politique DEFAULT hérite de la préférence générale.

### 5.4. `journal_entry_versions`

| Colonne | Type | Règle |
|---|---|---|
| id | UUID | PK |
| journal_entry_id | UUID | FK journal_entries |
| version_number | INTEGER | NOT NULL |
| content | TEXT | NOT NULL |
| created_at | TIMESTAMPTZ | NOT NULL |
| created_by_type | VARCHAR | USER / AI / SYSTEM |

Unique `(journal_entry_id, version_number)`.

Une version est créée lors d'une sauvegarde effective, pas à chaque frappe clavier.

La mise à jour du pointeur `current_version` et la création de la version doivent être transactionnelles.

### 5.5. `practice_preferences`

| Colonne | Type | Règle |
|---|---|---|
| user_id | UUID | PK |
| default_journal_ai_access | BOOLEAN | DEFAULT FALSE |
| autosave_enabled | BOOLEAN | DEFAULT TRUE |
| updated_at | TIMESTAMPTZ | NOT NULL |

### 5.6. `personal_pathways`

Agrégat PersonalPathway.

| Colonne | Type | Règle |
|---|---|---|
| id | UUID | PK |
| user_id | UUID | propriétaire |
| title | VARCHAR | NOT NULL |
| description | TEXT | NULL |
| creation_mode | VARCHAR | MANUAL / AI_PROPOSED / AI_ASSISTED |
| status | VARCHAR | DRAFT / ACTIVE / PAUSED / COMPLETED / ARCHIVED |
| created_at | TIMESTAMPTZ | NOT NULL |
| updated_at | TIMESTAMPTZ | NOT NULL |

### 5.7. `pathway_steps`

| Colonne | Type | Règle |
|---|---|---|
| id | UUID | PK |
| pathway_id | UUID | FK personal_pathways |
| position | INTEGER | NOT NULL |
| title | VARCHAR | NOT NULL |
| description | TEXT | NULL |
| step_type | VARCHAR | NOT NULL |
| reference_id | UUID | NULL |
| reference_version_id | UUID | NULL |
| status | VARCHAR | IN_PROGRESS / COMPLETED / SKIPPED |
| is_optional | BOOLEAN | DEFAULT FALSE |

Les états LOCKED et AVAILABLE sont calculés à partir des prérequis.

### 5.8. `pathway_prerequisites`

| Colonne | Type | Règle |
|---|---|---|
| pathway_step_id | UUID | FK pathway_steps |
| prerequisite_step_id | UUID | FK pathway_steps |

PK composite.

Les dépendances doivent appartenir au même parcours et former un graphe sans cycle.

### 5.9. `personal_goals`

Agrégat PersonalGoal.

| Colonne | Type | Règle |
|---|---|---|
| id | UUID | PK |
| user_id | UUID | propriétaire |
| title | VARCHAR | NOT NULL |
| description | TEXT | NULL |
| domain | VARCHAR | NULL |
| status | VARCHAR | ACTIVE / ON_HOLD / ACHIEVED / ABANDONED |
| target_date | DATE | NULL |
| created_at | TIMESTAMPTZ | NOT NULL |
| updated_at | TIMESTAMPTZ | NOT NULL |

### 5.10. `goal_pathway_links`

| Colonne | Type | Règle |
|---|---|---|
| goal_id | UUID | référence goal |
| pathway_id | UUID | référence pathway |
| created_at | TIMESTAMPTZ | NOT NULL |

PK composite `(goal_id, pathway_id)`.

Les deux objets doivent appartenir au même utilisateur.

### 5.11. `progress_reviews`

Agrégat ProgressReview.

| Colonne | Type | Règle |
|---|---|---|
| id | UUID | PK |
| user_id | UUID | propriétaire |
| period_start | DATE | NOT NULL |
| period_end | DATE | NOT NULL |
| scope | JSONB | NOT NULL |
| status | VARCHAR | REQUESTED / GENERATING / DRAFT / VALIDATED / CANCELLED / FAILED |
| content | JSONB | NULL |
| requested_at | TIMESTAMPTZ | NOT NULL |
| generated_at | TIMESTAMPTZ | NULL |
| validated_at | TIMESTAMPTZ | NULL |
| created_at | TIMESTAMPTZ | NOT NULL |
| updated_at | TIMESTAMPTZ | NOT NULL |

### 5.12. `progress_review_snapshots`

| Colonne | Type | Règle |
|---|---|---|
| id | UUID | PK |
| progress_review_id | UUID | FK progress_reviews |
| source_type | VARCHAR | NOT NULL |
| source_id | UUID | NOT NULL |
| source_version | VARCHAR | NULL |
| included_at | TIMESTAMPTZ | NOT NULL |

Conserve les références exactes aux données utilisées, sans recopier leur contenu.

### 5.13. `progress_review_revisions`

| Colonne | Type | Règle |
|---|---|---|
| id | UUID | PK |
| progress_review_id | UUID | FK progress_reviews |
| revision_number | INTEGER | NOT NULL |
| content | JSONB | NOT NULL |
| created_at | TIMESTAMPTZ | NOT NULL |
| created_by_type | VARCHAR | USER / AI / SYSTEM |

Unique `(progress_review_id, revision_number)`.

### 5.14. `user_bookmarks`

Favoris transversaux.

| Colonne | Type | Règle |
|---|---|---|
| id | UUID | PK |
| user_id | UUID | propriétaire |
| target_type | VARCHAR | type d'objet |
| target_id | UUID | identifiant objet |
| created_at | TIMESTAMPTZ | NOT NULL |

Unique `(user_id, target_type, target_id)`.

Les références sont vérifiées par le contexte applicatif propriétaire.

---

## 6. Schéma `tcc_content`

### 6.1. `exercises`

Agrégat Exercise.

| Colonne | Type | Règle |
|---|---|---|
| id | UUID | PK |
| slug | VARCHAR | UNIQUE |
| status | VARCHAR | DRAFT / UNDER_REVIEW / PUBLISHED / ARCHIVED |
| primary_category_id | UUID | NULL |
| created_by | UUID | auteur |
| created_at | TIMESTAMPTZ | NOT NULL |
| updated_at | TIMESTAMPTZ | NOT NULL |
| archived_at | TIMESTAMPTZ | NULL |

Les catégories et étiquettes sont associées à l'exercice, et non à une version particulière.

### 6.2. `exercise_versions`

Versions immuables des exercices.

| Colonne | Type | Règle |
|---|---|---|
| id | UUID | PK |
| exercise_id | UUID | FK exercises |
| version_number | INTEGER | NOT NULL |
| status | VARCHAR | DRAFT / UNDER_REVIEW / APPROVED / PUBLISHED / ARCHIVED |
| pedagogical_rationale | TEXT | NOT NULL |
| methodological_basis | TEXT | NOT NULL |
| evidence_level | VARCHAR | niveau de justification |
| limitations | JSONB | NOT NULL |
| author_id | UUID | NOT NULL |
| reviewer_id | UUID | NULL |
| approved_at | TIMESTAMPTZ | NULL |
| published_at | TIMESTAMPTZ | NULL |
| created_at | TIMESTAMPTZ | NOT NULL |

Unique `(exercise_id, version_number)`.

Niveaux :
- ESTABLISHED_METHOD
- PEDAGOGICAL_ADAPTATION
- EXPERIMENTAL_PROPOSAL

### 6.3. `exercise_version_sections`

Sections communes de l'exercice.

| Colonne | Type | Règle |
|---|---|---|
| id | UUID | PK |
| exercise_version_id | UUID | FK exercise_versions |
| section_type | VARCHAR | INTRODUCTION / OBJECTIVES / INSTRUCTIONS / CONCLUSION / PRECAUTIONS |
| content | TEXT | NOT NULL |
| position | INTEGER | NOT NULL |

Contrainte unique `(exercise_version_id, section_type)`.

Chaque version publiée doit posséder les sections obligatoires.

### 6.4. `exercise_content_blocks`

Blocs modulaires de contenu.

| Colonne | Type | Règle |
|---|---|---|
| id | UUID | PK |
| exercise_version_id | UUID | FK exercise_versions |
| block_key | VARCHAR | identifiant stable dans la version |
| position | INTEGER | NOT NULL |
| block_type | VARCHAR | type de bloc |
| content | JSONB | NOT NULL |
| condition | JSONB | NULL |

Les conditions sont déclaratives et simples.

Elles ne peuvent référencer que des questions compatibles de la même version.

### 6.5. `exercise_questions`

Questions auxquelles l'utilisateur peut répondre.

| Colonne | Type | Règle |
|---|---|---|
| id | UUID | PK |
| exercise_version_id | UUID | FK exercise_versions |
| block_id | UUID | FK exercise_content_blocks |
| question_key | VARCHAR | NOT NULL |
| position | INTEGER | NOT NULL |
| question_type | VARCHAR | NOT NULL |
| label | TEXT | NOT NULL |
| configuration | JSONB | NOT NULL |
| is_required | BOOLEAN | DEFAULT FALSE |

Les questions possèdent des identifiants stables à l'intérieur de leur version.

Leur type détermine la structure JSONB autorisée dans `exercise_answers`.

La validation des réponses est définie par le contenu publié, pas par une interprétation libre du frontend.

### 6.6. `exercise_categories`

Taxonomie hiérarchique.

| Colonne | Type | Règle |
|---|---|---|
| id | UUID | PK |
| parent_id | UUID | NULL |
| slug | VARCHAR | UNIQUE |
| name | VARCHAR | NOT NULL |
| description | TEXT | NULL |
| status | VARCHAR | ACTIVE / ARCHIVED |
| position | INTEGER | NOT NULL |

Une catégorie ne peut pas être son propre parent ni créer un cycle.

### 6.7. `exercise_category_assignments`

| Colonne | Type | Règle |
|---|---|---|
| exercise_id | UUID | FK exercises |
| category_id | UUID | FK exercise_categories |
| created_at | TIMESTAMPTZ | NOT NULL |

PK composite.

Un exercice peut avoir plusieurs catégories et au maximum une catégorie principale.

Un exercice publié doit posséder au moins une catégorie.

### 6.8. `exercise_tags`

| Colonne | Type | Règle |
|---|---|---|
| id | UUID | PK |
| slug | VARCHAR | UNIQUE |
| name | VARCHAR | NOT NULL |
| status | VARCHAR | ACTIVE / ARCHIVED |

### 6.9. `exercise_tag_assignments`

| Colonne | Type | Règle |
|---|---|---|
| exercise_id | UUID | FK exercises |
| tag_id | UUID | FK exercise_tags |

PK composite.

### 6.10. `editorial_pathways`

Agrégat EditorialPathway.

| Colonne | Type | Règle |
|---|---|---|
| id | UUID | PK |
| slug | VARCHAR | UNIQUE |
| status | VARCHAR | DRAFT / PUBLISHED / ARCHIVED |
| created_at | TIMESTAMPTZ | NOT NULL |
| updated_at | TIMESTAMPTZ | NOT NULL |

### 6.11. `editorial_pathway_versions`

| Colonne | Type | Règle |
|---|---|---|
| id | UUID | PK |
| pathway_id | UUID | FK editorial_pathways |
| version_number | INTEGER | NOT NULL |
| title | VARCHAR | NOT NULL |
| description | TEXT | NULL |
| pedagogical_rationale | TEXT | NOT NULL |
| methodological_basis | TEXT | NOT NULL |
| evidence_level | VARCHAR | NOT NULL |
| limitations | JSONB | NOT NULL |
| author_id | UUID | NOT NULL |
| reviewer_id | UUID | NULL |
| published_at | TIMESTAMPTZ | NULL |
| created_at | TIMESTAMPTZ | NOT NULL |

Unique `(pathway_id, version_number)`.

### 6.12. `editorial_pathway_steps`

| Colonne | Type | Règle |
|---|---|---|
| id | UUID | PK |
| pathway_version_id | UUID | FK editorial_pathway_versions |
| position | INTEGER | NOT NULL |
| step_type | VARCHAR | EXERCISE / RESOURCE / REFLECTION / INTERMEDIATE_REVIEW |
| reference_id | UUID | référence contenu |
| reference_version_id | UUID | version exacte |
| is_optional | BOOLEAN | DEFAULT FALSE |
| description | TEXT | NULL |

### 6.13. `editorial_pathway_prerequisites`

| Colonne | Type | Règle |
|---|---|---|
| pathway_step_id | UUID | FK editorial_pathway_steps |
| prerequisite_step_id | UUID | FK editorial_pathway_steps |

PK composite.

Les dépendances forment un graphe sans cycle.

### 6.14. `resources`

Agrégat Resource.

| Colonne | Type | Règle |
|---|---|---|
| id | UUID | PK |
| slug | VARCHAR | UNIQUE |
| status | VARCHAR | DRAFT / UNDER_REVIEW / PUBLISHED / ARCHIVED |
| ai_retrieval_enabled | BOOLEAN | DEFAULT FALSE |
| created_at | TIMESTAMPTZ | NOT NULL |
| updated_at | TIMESTAMPTZ | NOT NULL |

La publication et l'autorisation d'indexation sont deux décisions distinctes.

### 6.15. `resource_versions`

| Colonne | Type | Règle |
|---|---|---|
| id | UUID | PK |
| resource_id | UUID | FK resources |
| version_number | INTEGER | NOT NULL |
| title | VARCHAR | NOT NULL |
| description | TEXT | NULL |
| storage_key | VARCHAR | NOT NULL |
| mime_type | VARCHAR | NOT NULL |
| file_size | BIGINT | NOT NULL |
| checksum | CHAR(64) | NOT NULL |
| created_at | TIMESTAMPTZ | NOT NULL |
| published_at | TIMESTAMPTZ | NULL |

Unique `(resource_id, version_number)`.

Le fichier est conservé dans le stockage privé. PostgreSQL ne stocke que ses métadonnées et sa référence de stockage.

### 6.16. `bibliographic_references`

Références bibliographiques.

| Colonne | Type | Règle |
|---|---|---|
| id | UUID | PK |
| title | TEXT | NOT NULL |
| authors | JSONB | NOT NULL |
| publication_year | INTEGER | NULL |
| publication_type | VARCHAR | NULL |
| publisher | VARCHAR | NULL |
| doi | VARCHAR | NULL |
| url | TEXT | NULL |
| citation_data | JSONB | NULL |
| created_at | TIMESTAMPTZ | NOT NULL |

### 6.17. `content_evidence`

Association entre un contenu versionné et une référence.

| Colonne | Type | Règle |
|---|---|---|
| id | UUID | PK |
| content_type | VARCHAR | EXERCISE_VERSION / PATHWAY_VERSION |
| content_version_id | UUID | référence version |
| bibliographic_reference_id | UUID | FK bibliographic_references |
| rationale | TEXT | NOT NULL |
| support_type | VARCHAR | NOT NULL |
| created_at | TIMESTAMPTZ | NOT NULL |

Une référence bibliographique n'est pas automatiquement une preuve de validité de l'ensemble d'un exercice.

La pertinence du lien entre référence et contenu doit être explicitée.

---

## 7. Schéma `knowledge_base`

### 7.1. `knowledge_documents`

Agrégat KnowledgeDocument.

| Colonne | Type | Règle |
|---|---|---|
| id | UUID | PK |
| title | VARCHAR | NOT NULL |
| description | TEXT | NULL |
| document_type | VARCHAR | NOT NULL |
| source_type | VARCHAR | IMPORTED / TCC_RESOURCE |
| source_id | UUID | NULL |
| status | VARCHAR | DRAFT / UNDER_REVIEW / APPROVED / ARCHIVED |
| latest_version_id | UUID | NULL |
| active_version_id | UUID | NULL |
| ai_retrieval_enabled | BOOLEAN | DEFAULT FALSE |
| created_by | UUID | NOT NULL |
| created_at | TIMESTAMPTZ | NOT NULL |
| updated_at | TIMESTAMPTZ | NOT NULL |
| archived_at | TIMESTAMPTZ | NULL |

`latest_version_id` désigne la version la plus récente connue.

`active_version_id` désigne la version actuellement exploitable dans le RAG.

Ces deux références ne doivent pas être confondues.

### 7.2. `knowledge_document_versions`

| Colonne | Type | Règle |
|---|---|---|
| id | UUID | PK |
| document_id | UUID | FK knowledge_documents |
| version_number | INTEGER | NOT NULL |
| storage_key | VARCHAR | NOT NULL |
| original_filename | VARCHAR | NOT NULL |
| mime_type | VARCHAR | NOT NULL |
| file_size | BIGINT | NOT NULL |
| checksum | CHAR(64) | NOT NULL |
| extraction_status | VARCHAR | NOT NULL |
| approved_at | TIMESTAMPTZ | NULL |
| created_at | TIMESTAMPTZ | NOT NULL |

Unique `(document_id, version_number)`.

Les versions sont immuables.

### 7.3. `document_ingestions`

Processus d'extraction et de préparation documentaire.

| Colonne | Type | Règle |
|---|---|---|
| id | UUID | PK |
| document_version_id | UUID | FK knowledge_document_versions |
| pipeline_version | VARCHAR | NOT NULL |
| status | VARCHAR | PENDING / PROCESSING / COMPLETED / PARTIAL / FAILED / UNSUPPORTED |
| attempt_count | INTEGER | DEFAULT 0 |
| started_at | TIMESTAMPTZ | NULL |
| finished_at | TIMESTAMPTZ | NULL |
| error_code | VARCHAR | NULL |
| created_at | TIMESTAMPTZ | NOT NULL |
| updated_at | TIMESTAMPTZ | NOT NULL |

### 7.4. `document_ingestion_attempts`

| Colonne | Type | Règle |
|---|---|---|
| id | UUID | PK |
| ingestion_id | UUID | FK document_ingestions |
| attempt_number | INTEGER | NOT NULL |
| status | VARCHAR | NOT NULL |
| started_at | TIMESTAMPTZ | NOT NULL |
| finished_at | TIMESTAMPTZ | NULL |
| error_code | VARCHAR | NULL |
| technical_details | JSONB | NULL |

Unique `(ingestion_id, attempt_number)`.

### 7.5. `document_indexations`

| Colonne | Type | Règle |
|---|---|---|
| id | UUID | PK |
| document_version_id | UUID | FK knowledge_document_versions |
| index_version | VARCHAR | NOT NULL |
| status | VARCHAR | PENDING / PROCESSING / INDEXED / FAILED / OUTDATED / REMOVAL_PENDING |
| started_at | TIMESTAMPTZ | NULL |
| completed_at | TIMESTAMPTZ | NULL |
| error_code | VARCHAR | NULL |
| created_at | TIMESTAMPTZ | NOT NULL |
| updated_at | TIMESTAMPTZ | NOT NULL |

### 7.6. Taxonomie documentaire

Tables :
- `knowledge_document_categories`
- `knowledge_document_category_assignments`
- `knowledge_document_tags`
- `knowledge_document_tag_assignments`

Les catégories sont hiérarchiques.

Les tags sont transversaux.

Un document peut posséder plusieurs catégories et tags.

Les catégories et tags sont créés ou modifiés par un administrateur. L'IA peut suggérer des classifications, mais ne peut pas modifier la taxonomie officielle.

---

## 8. Schéma `ai_governance`

### 8.1. `ai_models`

Catalogue des modèles.

| Colonne | Type | Règle |
|---|---|---|
| id | UUID | PK |
| runtime | VARCHAR | OLLAMA |
| model_name | VARCHAR | NOT NULL |
| model_tag | VARCHAR | NOT NULL |
| artifact_checksum | VARCHAR | NULL |
| capabilities | JSONB | NOT NULL |
| resource_requirements | JSONB | NOT NULL |
| status | VARCHAR | ACTIVE / RETIRED |
| created_at | TIMESTAMPTZ | NOT NULL |
| updated_at | TIMESTAMPTZ | NOT NULL |

### 8.2. `ai_model_configurations`

| Colonne | Type | Règle |
|---|---|---|
| id | UUID | PK |
| slug | VARCHAR | UNIQUE |
| usage | VARCHAR | NOT NULL |
| name | VARCHAR | NOT NULL |
| created_at | TIMESTAMPTZ | NOT NULL |
| updated_at | TIMESTAMPTZ | NOT NULL |

Une configuration est définie par usage.

### 8.3. `ai_model_configuration_versions`

| Colonne | Type | Règle |
|---|---|---|
| id | UUID | PK |
| configuration_id | UUID | FK ai_model_configurations |
| version_number | INTEGER | NOT NULL |
| model_id | UUID | FK ai_models |
| parameters | JSONB | NOT NULL |
| resource_limits | JSONB | NOT NULL |
| created_by | UUID | NOT NULL |
| created_at | TIMESTAMPTZ | NOT NULL |

Unique `(configuration_id, version_number)`.

### 8.4. `prompt_definitions`

| Colonne | Type | Règle |
|---|---|---|
| id | UUID | PK |
| slug | VARCHAR | UNIQUE |
| name | VARCHAR | NOT NULL |
| usage | VARCHAR | NOT NULL |
| status | VARCHAR | NOT NULL |
| created_at | TIMESTAMPTZ | NOT NULL |
| updated_at | TIMESTAMPTZ | NOT NULL |

### 8.5. `prompt_versions`

| Colonne | Type | Règle |
|---|---|---|
| id | UUID | PK |
| prompt_definition_id | UUID | FK prompt_definitions |
| version_number | INTEGER | NOT NULL |
| content | TEXT | NOT NULL |
| variables_schema | JSONB | NOT NULL |
| checksum | CHAR(64) | NOT NULL |
| created_by | UUID | NOT NULL |
| created_at | TIMESTAMPTZ | NOT NULL |

Unique `(prompt_definition_id, version_number)`.

### 8.6. `governance_policies`

Définitions des règles de gouvernance configurables.

### 8.7. `governance_policy_versions`

Versions immuables des règles.

Champs principaux :
- id UUID ;
- policy_id UUID ;
- version_number INTEGER ;
- configuration JSONB ;
- checksum CHAR(64) ;
- created_by UUID ;
- created_at TIMESTAMPTZ.

### 8.8. `safety_policy_versions`

Versions des paramètres de sécurité configurables.

Les garde-fous fondamentaux restent dans le code et ne peuvent pas être supprimés par une modification de cette table.

### 8.9. `emergency_resources`

Ressources d'urgence.

| Colonne | Type | Règle |
|---|---|---|
| id | UUID | PK |
| country_code | VARCHAR | NOT NULL |
| locale | VARCHAR | NOT NULL |
| resource_type | VARCHAR | NOT NULL |
| name | VARCHAR | NOT NULL |
| phone | VARCHAR | NULL |
| instructions | TEXT | NULL |
| availability | VARCHAR | NULL |
| source_url | TEXT | NULL |
| verified_at | TIMESTAMPTZ | NULL |
| status | VARCHAR | NOT NULL |

### 8.10. `safety_interventions`

Trace minimale des interventions de sécurité.

Champs :
- id UUID ;
- user_id UUID ;
- conversation_id UUID nullable ;
- risk_level VARCHAR ;
- trigger_category VARCHAR ;
- protocol_version VARCHAR ;
- outcome VARCHAR ;
- created_at TIMESTAMPTZ.

Aucun contenu conversationnel n'est stocké ici.

### 8.11. `evaluation_suites`

Définitions des suites d'évaluation.

### 8.12. `evaluation_suite_versions`

Versions immuables des suites.

### 8.13. `evaluation_scenarios`

Scénarios de test versionnés.

Chaque scénario contient :
- un identifiant ;
- une catégorie ;
- un jeu d'entrée synthétique ;
- des critères attendus ;
- une sévérité BLOCKING / MAJOR / MINOR ;
- un indicateur d'activation dans la suite.

Les scénarios ne doivent pas contenir de données personnelles réelles.

### 8.14. `evaluation_runs`

Exécutions d'une suite sur une release.

Champs principaux :
- id UUID ;
- release_id UUID ;
- suite_version_id UUID ;
- status VARCHAR ;
- started_at TIMESTAMPTZ ;
- completed_at TIMESTAMPTZ ;
- overall_result VARCHAR ;
- blocking_failure BOOLEAN ;
- technical_details JSONB.

### 8.15. `evaluation_results`

Résultats individuels des scénarios.

Champs principaux :
- id UUID ;
- evaluation_run_id UUID ;
- scenario_id UUID ;
- result VARCHAR ;
- metrics JSONB ;
- failure_category VARCHAR ;
- duration_ms BIGINT ;
- technical_details JSONB.

Les résultats sont conservés de manière immuable.

### 8.16. `ai_configuration_releases`

Release globale de configuration IA.

| Colonne | Type | Règle |
|---|---|---|
| id | UUID | PK |
| release_number | INTEGER | UNIQUE |
| name | VARCHAR | NOT NULL |
| description | TEXT | NULL |
| status | VARCHAR | DRAFT / EVALUATED / APPROVED / ACTIVE / INACTIVE / REJECTED |
| application_version | VARCHAR | NOT NULL |
| safety_code_version | VARCHAR | NOT NULL |
| created_by | UUID | NOT NULL |
| created_at | TIMESTAMPTZ | NOT NULL |

### 8.17. `ai_configuration_release_components`

Composition d'une release.

Champs :
- id UUID ;
- release_id UUID ;
- component_type VARCHAR ;
- component_id UUID ;
- component_version_id UUID ;
- usage VARCHAR nullable ;
- configuration_snapshot JSONB nullable ;
- position INTEGER.

Les références polymorphes sont validées dans la couche Application.

Une release référence les versions exactes de ses modèles, prompts et politiques.

### 8.18. `active_ai_configuration`

Pointeur unique vers la release active.

Champs :
- id SMALLINT, valeur unique ;
- release_id UUID ;
- activated_at TIMESTAMPTZ.

Une seule ligne active est autorisée.

### 8.19. `ai_release_activations`

Historique append-only des activations et retours arrière.

Champs :
- id UUID ;
- release_id UUID ;
- previous_release_id UUID nullable ;
- activation_type VARCHAR ;
- activated_by UUID ;
- reason TEXT nullable ;
- activated_at TIMESTAMPTZ.

---

## 9. Schéma `platform`

### 9.1. `outbox_events`

Événements métier à publier.

Champs :
- id UUID ;
- event_id UUID UNIQUE ;
- event_type VARCHAR ;
- event_version INTEGER ;
- producer VARCHAR ;
- aggregate_type VARCHAR ;
- aggregate_id UUID ;
- aggregate_version BIGINT ;
- occurred_at TIMESTAMPTZ ;
- correlation_id UUID ;
- causation_id UUID nullable ;
- payload JSONB ;
- metadata JSONB ;
- available_at TIMESTAMPTZ ;
- published_at TIMESTAMPTZ nullable ;
- attempt_count INTEGER ;
- last_error_code VARCHAR nullable ;
- created_at TIMESTAMPTZ.

Les payloads ne contiennent aucun contenu personnel brut.

### 9.2. `processed_events`

Idempotence des consommateurs.

- consumer_name VARCHAR ;
- event_id UUID ;
- processed_at TIMESTAMPTZ.

PK composite `(consumer_name, event_id)`.

### 9.3. `idempotency_keys`

- id UUID ;
- scope VARCHAR ;
- idempotency_key VARCHAR ;
- request_hash CHAR(64) ;
- resource_type VARCHAR nullable ;
- resource_id UUID nullable ;
- result_status VARCHAR ;
- created_at TIMESTAMPTZ ;
- expires_at TIMESTAMPTZ.

Contrainte unique `(scope, idempotency_key)`.

### 9.4. `async_operations`

Suivi transversal des opérations longues.

- id UUID ;
- user_id UUID nullable ;
- operation_type VARCHAR ;
- context_name VARCHAR ;
- resource_id UUID nullable ;
- status VARCHAR ;
- progress JSONB nullable ;
- error_code VARCHAR nullable ;
- started_at TIMESTAMPTZ nullable ;
- completed_at TIMESTAMPTZ nullable ;
- created_at TIMESTAMPTZ.

Ne remplace pas le statut métier du processus propriétaire.

### 9.5. Files de travaux Laravel

Tables techniques Laravel :
- `jobs`
- `job_batches`
- `failed_jobs`

Les payloads de jobs doivent privilégier les identifiants et métadonnées techniques, jamais le contenu privé complet.

Les exceptions enregistrées doivent être expurgées de toute donnée personnelle.

### 9.6. `backups`

- id UUID ;
- backup_type VARCHAR ;
- status VARCHAR ;
- storage_location VARCHAR ;
- storage_key VARCHAR ;
- file_size BIGINT nullable ;
- checksum CHAR(64) nullable ;
- encryption_key_reference VARCHAR ;
- started_at TIMESTAMPTZ nullable ;
- completed_at TIMESTAMPTZ nullable ;
- expires_at TIMESTAMPTZ nullable ;
- created_at TIMESTAMPTZ.

### 9.7. `backup_components`

- id UUID ;
- backup_id UUID ;
- component_type VARCHAR ;
- storage_key VARCHAR ;
- checksum CHAR(64) ;
- size BIGINT ;
- created_at TIMESTAMPTZ.

### 9.8. `restore_operations`

- id UUID ;
- backup_id UUID ;
- status VARCHAR ;
- restore_scope JSONB ;
- started_at TIMESTAMPTZ nullable ;
- completed_at TIMESTAMPTZ nullable ;
- executed_by UUID ;
- error_code VARCHAR nullable.

### 9.9. `export_requests`

- id UUID ;
- user_id UUID ;
- status VARCHAR ;
- format VARCHAR ;
- storage_key VARCHAR nullable ;
- checksum CHAR(64) nullable ;
- file_size BIGINT nullable ;
- requested_at TIMESTAMPTZ ;
- completed_at TIMESTAMPTZ nullable ;
- expires_at TIMESTAMPTZ nullable ;
- error_code VARCHAR nullable.

Formats initiaux :
- JSON ;
- JSON_AND_FILES.

### 9.10. `deletion_requests`

- id UUID ;
- user_id UUID ;
- status VARCHAR ;
- requested_at TIMESTAMPTZ ;
- confirmed_at TIMESTAMPTZ nullable ;
- completed_at TIMESTAMPTZ nullable ;
- failure_code VARCHAR nullable.

### 9.11. `deletion_tasks`

- id UUID ;
- deletion_request_id UUID ;
- context_name VARCHAR ;
- status VARCHAR ;
- attempt_count INTEGER ;
- completed_at TIMESTAMPTZ nullable ;
- error_code VARCHAR nullable.

Une tâche par contexte concerné et opération de suppression.

### 9.12. `notifications`

- id UUID ;
- user_id UUID ;
- type VARCHAR ;
- channel VARCHAR ;
- title VARCHAR ;
- content TEXT ;
- status VARCHAR ;
- scheduled_at TIMESTAMPTZ nullable ;
- sent_at TIMESTAMPTZ nullable ;
- read_at TIMESTAMPTZ nullable ;
- created_at TIMESTAMPTZ.

Canaux initiaux :
- IN_APP ;
- EMAIL.

### 9.13. `notification_preferences`

- id UUID ;
- user_id UUID ;
- type VARCHAR ;
- channel VARCHAR ;
- enabled BOOLEAN ;
- frequency VARCHAR ;
- quiet_hours JSONB ;
- updated_at TIMESTAMPTZ.

Unique `(user_id, type, channel)`.

### 9.14. `maintenance_operations`

- id UUID ;
- operation_type VARCHAR ;
- status VARCHAR ;
- component VARCHAR ;
- from_version VARCHAR nullable ;
- to_version VARCHAR nullable ;
- started_at TIMESTAMPTZ nullable ;
- completed_at TIMESTAMPTZ nullable ;
- executed_by UUID ;
- details JSONB.

### 9.15. `telemetry_configurations`

Configuration locale de télémétrie.

- id UUID ;
- enabled BOOLEAN ;
- retention_days INTEGER ;
- collect_performance BOOLEAN ;
- collect_errors BOOLEAN ;
- collect_resource_usage BOOLEAN ;
- updated_at TIMESTAMPTZ.

La télémétrie ne contient aucun texte personnel.

---

## 10. Schéma `ai_engine`

Ce schéma contient les données techniques possédées par Python.

Laravel ne lui délègue aucune donnée métier ni autorisation.

### 10.1. `document_extractions`

- id UUID ;
- document_version_id UUID ;
- pipeline_version VARCHAR ;
- extraction_status VARCHAR ;
- extracted_at TIMESTAMPTZ ;
- technical_metadata JSONB.

Unique selon la version documentaire et la version de pipeline.

### 10.2. `document_sections`

- id UUID ;
- extraction_id UUID ;
- parent_section_id UUID nullable ;
- title TEXT nullable ;
- section_type VARCHAR ;
- position INTEGER ;
- page_start INTEGER nullable ;
- page_end INTEGER nullable ;
- metadata JSONB.

Conserve la structure hiérarchique du document.

### 10.3. `document_chunks`

- id UUID ;
- extraction_id UUID ;
- section_id UUID nullable ;
- sequence INTEGER ;
- content TEXT ;
- page_start INTEGER nullable ;
- page_end INTEGER nullable ;
- token_count INTEGER ;
- parent_chunk_id UUID nullable ;
- metadata JSONB.

Le découpage doit respecter autant que possible les titres, paragraphes, listes et tableaux.

Les tableaux doivent conserver leurs en-têtes et relations entre cellules lorsque l'extraction le permet.

### 10.4. `chunk_embeddings`

- id UUID ;
- chunk_id UUID ;
- embedding_model VARCHAR ;
- embedding_dimensions INTEGER ;
- embedding VECTOR(n) ;
- created_at TIMESTAMPTZ.

La dimension réelle est déterminée par le modèle d'embedding choisi.

Pour le MVP, une dimension principale est retenue. La coexistence de dimensions différentes nécessiterait une organisation spécifique.

### 10.5. `retrieval_indexes`

- id UUID ;
- document_version_id UUID ;
- embedding_model VARCHAR ;
- pipeline_version VARCHAR ;
- status VARCHAR ;
- indexed_at TIMESTAMPTZ nullable ;
- metadata JSONB.

### 10.6. `retrieval_traces`

- id UUID ;
- correlation_id UUID ;
- document_query_hash CHAR(64) ;
- pipeline_version VARCHAR ;
- reranker_version VARCHAR nullable ;
- status VARCHAR ;
- result_count INTEGER ;
- started_at TIMESTAMPTZ ;
- completed_at TIMESTAMPTZ nullable.

Ne jamais conserver le texte brut des requêtes personnelles dans les traces techniques.

### 10.7. `retrieval_trace_results`

- id UUID ;
- retrieval_trace_id UUID ;
- document_id UUID ;
- document_version_id UUID ;
- chunk_id UUID ;
- vector_rank INTEGER nullable ;
- text_rank INTEGER nullable ;
- fusion_rank INTEGER nullable ;
- reranker_rank INTEGER nullable ;
- vector_score NUMERIC nullable ;
- text_score NUMERIC nullable ;
- fusion_score NUMERIC nullable ;
- reranker_score NUMERIC nullable.

Ces informations permettent d'analyser la qualité technique des résultats sans recopier les contenus privés.

---

## 11. Contraintes et index transversaux

Les migrations devront notamment implémenter :

- PK UUID ;
- FK internes aux schémas lorsque cela est pertinent ;
- unicité des versions ;
- unicité des séquences de messages ;
- unicité des réponses par réalisation et question ;
- un seul attempt actif par message ;
- contrôle de la sélection d'une tentative appartenant au message ;
- contrôle de la catégorie principale ;
- interdiction des relations mémoire vers soi-même ;
- contraintes de cohérence des statuts ;
- contrôle des prérequis sans cycles ;
- contraintes de cohérence entre versions et pointeurs actifs ;
- index des recherches par utilisateur, statut et date ;
- index partiels pour les éléments actifs ou en attente ;
- index GIN pour les JSONB lorsque justifié ;
- index vectoriels adaptés au volume réel.

Les index HNSW ou IVFFlat ne doivent pas être choisis arbitrairement. Le choix sera confirmé après tests de volume, qualité et performances.

Une recherche vectorielle exacte peut suffire pour une première bibliothèque de taille limitée.

---

## 12. Règles de propriété et références intercontextes

Les FK sont privilégiées à l'intérieur d'un même contexte.

Pour les références intercontextes :
- stockage de l'identifiant de l'objet référencé ;
- validation applicative auprès du contexte propriétaire ;
- vérification de l'appartenance au même utilisateur lorsque nécessaire ;
- aucune écriture directe dans le contexte cible.

Exemples :
- `exercise_runs.exercise_version_id` référence une version TccContent ;
- `exercise_answers.question_id` référence une question TccContent ;
- `generation_attempts.ai_release_id` référence une release AiGovernance ;
- `context_snapshots` référence des objets de plusieurs contextes.

La cohérence des références doit être vérifiée au niveau applicatif.

---

## 13. Protection des données

Ne jamais stocker dans les logs techniques :
- messages utilisateur ;
- réponses du compagnon ;
- prompts composés ;
- mémoires injectées ;
- entrées de journal ;
- passages RAG ;
- texte partiel généré ;
- contenu d'export.

Les événements transversaux ne transportent que les identifiants, versions et métadonnées indispensables.

Le chiffrement au repos repose initialement sur le chiffrement du disque ou volume et des sauvegardes.

Les clés sont séparées des sauvegardes.

Ce mécanisme ne protège pas contre une compromission du système lorsque les volumes sont déverrouillés et accessibles.

---

## 14. Points à vérifier avant les migrations

Ce dictionnaire constitue la référence logique, pas une autorisation de générer immédiatement toutes les migrations sans analyse.

Claude doit vérifier :

1. La compatibilité de la version Laravel retenue avec les schémas PostgreSQL.
2. La syntaxe exacte des contraintes composites.
3. Les possibilités d'index partiels dans les migrations.
4. L'installation et la version de pgvector.
5. La dimension du modèle d'embedding retenu.
6. Les besoins réels d'indexation selon le volume documentaire.
7. Les règles de suppression et de conservation des données.
8. Les références polymorphes et leur validation applicative.
9. Les éventuelles redondances entre statuts métier et suivi technique.
10. Les besoins d'évolution du schéma sans rupture de données.

Tout écart entre le modèle logique et l'implémentation doit être documenté.

## 15. Règle de conception finale

La base de données doit représenter les invariants et les cycles de vie métier, et non simplement refléter la structure des formulaires ou des modèles ORM.

Une donnée possède une table propre lorsqu'elle a une identité, une durée de vie, des relations ou des règles métier qui justifient cette séparation.

JSONB est utilisé pour la flexibilité, pas pour masquer un modèle relationnel mal conçu.
