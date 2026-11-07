# 04 — Bounded Contexts et modèle métier

## 1. Objectif

Ce document décrit le modèle métier de chacun des sept bounded contexts de TCCTDD.

Pour chaque contexte, il précise :

- Sa responsabilité.
- Ses concepts métier.
- Ses agrégats et entités.
- Les principales règles de gestion.
- Les données dont il est propriétaire.
- Ses interactions avec les autres contextes.

L'objectif est de maintenir une séparation claire entre les responsabilités métier et d'éviter les dépendances implicites.

Les détails de persistance PostgreSQL sont décrits dans `06-DICTIONNAIRE-DONNEES-POSTGRESQL.md`. Les contrats synchrones entre contextes, les échanges avec le service Python et les événements d'intégration sont décrits dans `07-CONTRATS-INTERSERVICES.md`.

## 2. Companion

### 2.1 Responsabilité

Le contexte Companion gère les conversations entre l'utilisateur et le compagnon IA.

Il est propriétaire de l'historique conversationnel, de l'ordre des messages et du cycle de vie des générations.

Il ne possède ni la mémoire personnelle, ni les exercices, ni les documents, ni les modèles IA.

### 2.2 Agrégats

**Conversation**

Racine d'agrégat représentant une conversation.

Attributs conceptuels :

- Identifiant.
- Identifiant utilisateur.
- Titre.
- Type : générale ou thématique.
- Statut : active, archivée ou supprimée.
- Dernier numéro de séquence.
- Dates de création et de modification.

À défaut de titre saisi, le titre est dérivé des premiers mots du premier message, sans appel au modèle. Il reste modifiable par l'utilisateur.

Une conversation archivée peut être restaurée.

Conversation et Message sont deux agrégats distincts. La conversation porte le compteur de séquence et attribue de façon atomique le numéro de chaque nouveau message, ce qui garantit l'unicité et l'ordre de la séquence. Elle ne charge pas l'ensemble de ses messages.

**Message**

Message est sa propre racine d'agrégat, rattachée à sa conversation par identifiant.

Un message représente une intervention de l'utilisateur, du compagnon ou un message système interne.

Chaque message possède :

- Un identifiant.
- Un identifiant de conversation.
- Un numéro de séquence.
- Un rôle.
- Un statut.
- Un contenu lorsque celui-ci est stocké directement.
- Une référence vers la tentative de génération sélectionnée, si applicable.

Les messages utilisateur sont immuables après envoi. Une correction est effectuée par l'envoi d'un nouveau message.

Les messages de rôle USER et SYSTEM sont créés directement avec le statut COMPLETED. Seuls les messages assistant suivent le cycle de la génération.

**GenerationAttempt**

Une tentative représente une exécution de génération associée à un message assistant.

Elle possède :

- Un identifiant.
- Un message associé.
- Un numéro de tentative.
- Un statut.
- Le contenu généré.
- Les informations techniques de génération.
- Une éventuelle erreur.
- La release de configuration IA utilisée (`ai_release`), propre à chaque tentative.
- Les dates et durées d'exécution.

Plusieurs tentatives peuvent exister pour un même message assistant.

Une tentative est dite active lorsque son statut est PENDING ou STREAMING. Une seule tentative peut être active à un instant donné.

Une régénération ou une relance crée une nouvelle tentative qui référence la release active au moment de sa création. Deux tentatives d'un même message peuvent donc référencer des releases différentes.

### 2.3 Règles métier

- Chaque conversation appartient à un utilisateur.
- Les messages sont ordonnés par une séquence croissante.
- Une séquence est unique au sein d'une conversation.
- La création du message et l'incrément de séquence sont atomiques.
- Une régénération crée une nouvelle tentative pour le même message assistant.
- Les tentatives précédentes sont conservées selon la politique de rétention.
- Une tentative terminée est immuable.
- Une génération interrompue n'est pas présentée comme une réponse complète.
- Une tentative interrompue ne doit pas être automatiquement réinjectée dans le contexte conversationnel.
- Une nouvelle tentative repart du contexte conversationnel valide.
- Le contenu partiel est conservé temporairement, au maximum sept jours.
- Une génération de conversation est toujours interactive : elle n'est jamais préemptée.
- Si le créneau d'inférence est déjà occupé par une autre requête interactive, la nouvelle requête reçoit `ENGINE_BUSY` : la tentative échoue (FAILED) avec ce code d'erreur, sans relance automatique, et l'utilisateur peut régénérer la réponse.
- La suppression d'une conversation déclenche également le traitement des données dérivées qui lui sont associées.

Le contenu utilisateur est stocké sur le message. Le contenu généré par le compagnon est conservé dans les tentatives de génération. La réponse affichée correspond à la tentative sélectionnée.

## 3. PersonalContext

### 3.1 Responsabilité

PersonalContext gère la mémoire personnelle structurée utilisée pour maintenir la continuité de l'accompagnement.

Il est propriétaire des mémoires, de leur provenance, de leurs révisions, de leurs relations et des propositions de consolidation.

### 3.2 Agrégat Memory

Memory est la racine d'agrégat principale.

Une mémoire représente une information atomique ou un élément de contexte personnel suffisamment cohérent pour être réutilisé indépendamment.

Catégories possibles :

- PERSONAL_CONTEXT.
- RECURRING_DIFFICULTY.
- TRIGGER.
- THOUGHT_PATTERN.
- EMOTIONAL_PATTERN.
- GOAL.
- HELPFUL_STRATEGY.
- RESOURCE.
- PROGRESS.

Chaque mémoire possède deux dimensions de statut indépendantes.

**Statut de traitement :**

- PROPOSED.
- ACTIVE.
- REJECTED.
- ARCHIVED.
- SUPERSEDED.

**Statut épistémique :**

- HYPOTHESIS.
- DECLARED.
- CONFIRMED.
- CONTESTED.

Le statut de traitement décrit le cycle de vie de la mémoire.

Le statut épistémique décrit la nature de l'information et le niveau de validation associé.

Une information saisie directement par l'utilisateur dans sa mémoire est créée ACTIVE et DECLARED. Une interprétation proposée par l'IA peut être PROPOSED et HYPOTHESIS.

Toute mémoire extraite par l'IA, notamment depuis une conversation, est créée avec le statut de traitement PROPOSED, y compris lorsqu'elle reprend une déclaration explicite de l'utilisateur (statut épistémique DECLARED). Elle ne devient ACTIVE qu'après validation explicite de l'utilisateur.

L'extraction de mémoires et les propositions de consolidation sont des traitements IA de fond, soumis aux règles de préemption décrites au paragraphe 9.6.

### 3.3 Provenance et révisions

Chaque mémoire doit conserver sa provenance.

Les sources peuvent être des messages, des exercices, des entrées de journal autorisées ou d'autres éléments explicitement admissibles.

Une source doit être référencée sans recopier systématiquement son contenu sensible dans la mémoire.

Les révisions conservent l'historique des modifications significatives.

### 3.4 Contradictions

Les mémoires potentiellement contradictoires peuvent coexister.

Le système doit pouvoir signaler une contradiction potentielle sans décider automatiquement quelle information est vraie.

La résolution appartient à l'utilisateur ou à une démarche de clarification explicite.

Une contradiction est représentée par une MemoryRelation de type CONTRADICTS entre deux mémoires. Il n'existe pas d'agrégat dédié aux conflits de mémoire.

### 3.5 Consolidation

La consolidation permet de proposer :

- L'ajout d'une nouvelle source.
- La fusion de mémoires proches.
- La révision d'une mémoire existante.

Une simple association de source, sans changement de sens, peut être appliquée automatiquement lorsqu'elle est suffisamment fiable.

Une fusion ou une modification d'interprétation nécessite une validation utilisateur.

Les anciennes mémoires sont conservées et peuvent être marquées SUPERSEDED après validation d'une consolidation.

### 3.6 Relations entre mémoires

MemoryRelation représente une relation entre deux mémoires.

Types possibles :

- TRIGGERED_BY.
- ASSOCIATED_WITH.
- SUPPORTED_BY.
- CONTRADICTS.
- REINFORCES.
- MITIGATED_BY.
- CONTRIBUTES_TO.
- RELATED_TO.

Les relations proposées par l'IA restent des hypothèses jusqu'à validation.

Une relation ne peut pas relier une mémoire à elle-même.

### 3.7 Assemblage du contexte

Le ContextAssembler sélectionne les éléments utiles pour une interaction donnée :

- Messages récents.
- Mémoires pertinentes.
- Relations pertinentes.
- Contradictions éventuelles.
- Préférences du compagnon.
- Informations provenant des autres contextes lorsque leur utilisation est autorisée.

La sélection tient compte du budget de contexte défini par AiGovernance.

Le système ne doit jamais charger l'intégralité de la mémoire sans nécessité.

Un ContextSnapshot conserve les références et versions utilisées lors de l'assemblage, sans constituer une copie intégrale permanente de toutes les données personnelles.

## 4. TccContent

### 4.1 Responsabilité

TccContent est propriétaire des contenus TCC éditoriaux et pédagogiques.

Il définit les exercices, les ressources et les parcours éditoriaux disponibles dans l'application.

Il ne gère pas les sessions réalisées par l'utilisateur.

### 4.2 Exercices

**Exercise**

Racine représentant un exercice éditorial.

Il contient notamment :

- Un titre.
- Une description.
- Une classification.
- Un statut éditorial.
- Une référence vers sa version courante.

Les catégories et tags appartiennent à l'exercice, et non à chaque version.

Un exercice peut être associé à plusieurs catégories et tags. Une seule catégorie peut être principale.

Un exercice publié doit posséder au moins une catégorie.

**ExerciseVersion**

Une version publiée ou en préparation est immuable après publication.

Elle contient une structure commune :

- Introduction.
- Objectifs.
- Instructions.
- Conclusion.
- Précautions.

Elle peut également contenir des blocs de contenu modulaires et ordonnés.

Les blocs peuvent représenter du texte, des questions, des explications ou d'autres éléments pédagogiques.

Les questions possèdent une clé (`question_key`) unique au sein de leur version d'exercice, afin de permettre le référencement stable des réponses.

Chaque version conserve la trace de son auteur et de son validateur.

La validation est réservée au rôle `ADMIN`. Par défaut, le MVP personnel fonctionne en mode auteur-validateur unique : le même compte `ADMIN` peut rédiger et valider une version. Une séparation stricte entre auteur et validateur peut être activée par configuration.

### 4.3 Conditions d'affichage

Les blocs et questions peuvent être conditionnels.

Pour le MVP :

- Une condition par bloc.
- Un opérateur typé.
- Une référence vers une question de la même version.
- Une valeur de comparaison adaptée au type de réponse.

Les expressions complexes AND/OR, boucles et branchements multiples sont exclus.

Une condition ne peut pas dépendre d'une question située après le bloc concerné.

Une absence de réponse ne déclenche pas implicitement une condition.

L'évaluation des conditions appartient à UserPractice, tandis que la définition de la condition appartient à TccContent.

Une réponse masquée reste conservée, mais elle est exclue de la validation et du contexte IA tant que le bloc reste masqué.

### 4.4 Parcours éditoriaux

EditorialPathway définit un parcours pédagogique réutilisable.

EditorialPathwayVersion représente une version immuable de ce parcours.

Les étapes peuvent référencer :

- Un exercice et sa version exacte.
- Une ressource.
- Une réflexion.
- Un bilan intermédiaire.

Les prérequis peuvent former un graphe orienté sans cycle.

Selon son type, une étape peut exister sans charge utile de contenu (par exemple une réflexion ou un bilan intermédiaire sans exercice ni ressource associés). Seules les étapes dont le type l'exige doivent référencer un contenu.

Une modification d'exercice ne modifie pas silencieusement un parcours déjà publié.

### 4.5 Ressources

Resource représente une ressource pédagogique autonome.

ResourceVersion représente une version précise de cette ressource.

Les ressources originales sont conservées dans un stockage persistant privé.

TccContent publie les ressources. L'activation de leur utilisation par l'IA relève de KnowledgeBase et reste indépendante de leur publication éditoriale.

### 4.6 Justification scientifique et méthodologique

Chaque version d'exercice ou de parcours doit comporter une justification méthodologique.

Les niveaux de preuve éditoriaux sont :

- ESTABLISHED_METHOD.
- PEDAGOGICAL_ADAPTATION.
- EXPERIMENTAL_PROPOSAL.

Les adaptations pédagogiques doivent être identifiées comme telles.

Les propositions expérimentales ne doivent jamais être présentées comme des méthodes validées.

Les références bibliographiques sont associées à la version et au type de soutien qu'elles apportent.

Une référence bibliographique ne constitue pas automatiquement une validation de l'intégralité du contenu.

La validation éditoriale n'est pas une certification médicale ou scientifique.

## 5. UserPractice

### 5.1 Responsabilité

UserPractice gère l'utilisation personnelle des contenus, les exercices réalisés, le journal, les parcours et les objectifs.

Il est propriétaire de toutes les données résultant de la pratique personnelle.

### 5.2 ExerciseRun

ExerciseRun représente une session d'exercice.

Chaque démarrage crée une nouvelle session indépendante.

La session référence une version précise d'ExerciseVersion.

Statuts :

- IN_PROGRESS.
- COMPLETED.
- ABANDONED.

ExerciseAnswer représente une réponse à une question ou un élément de la session.

Règles :

- Les réponses sont sauvegardées régulièrement.
- Les champs textuels peuvent utiliser un mécanisme de sauvegarde différée.
- Une session en cours reste modifiable.
- Une session terminée est immuable.
- Une correction après terminaison nécessite une nouvelle session.
- Les réponses conditionnellement masquées sont conservées, mais exclues du contexte IA tant que leur bloc est masqué.

### 5.3 Journal

JournalEntry représente une entrée personnelle.

JournalEntryVersion conserve les versions successives.

Une version est créée lors d'une sauvegarde effective, et non à chaque frappe.

Chaque entrée possède une politique d'utilisation par l'IA :

- DEFAULT.
- ALLOWED.
- DENIED.

La préférence générale de l'utilisateur est DENIED par défaut.

La politique est vérifiée à chaque consultation. Une entrée interdite n'est jamais récupérée par la recherche documentaire ni injectée dans un contexte conversationnel.

La consultation du journal ne constitue jamais une autorisation implicite d'extraction de mémoire.

Une entrée peut être archivée puis restaurée. Sa suppression est d'abord logique et immédiate, puis physique et asynchrone (voir paragraphe 8.5).

### 5.4 Parcours personnels

PersonalPathway représente un parcours appartenant à l'utilisateur.

Il peut être :

- Créé manuellement.
- Proposé par l'IA.
- Construit avec l'aide de l'IA.

Une proposition de l'IA reste un brouillon jusqu'à acceptation. La génération d'une proposition de parcours est un traitement IA de fond, soumis aux règles de préemption du paragraphe 9.6.

Statuts :

- DRAFT.
- ACTIVE.
- PAUSED.
- COMPLETED.
- ARCHIVED.

PathwayStep représente une étape du parcours.

Selon son type, une étape peut exister sans charge utile de contenu (par exemple une réflexion libre ou un bilan intermédiaire). Seules les étapes dont le type l'exige référencent un exercice ou une ressource.

Les états LOCKED et AVAILABLE sont calculés à partir des prérequis.

Les états IN_PROGRESS, COMPLETED et SKIPPED sont des états opérationnels conservés.

### 5.5 Objectifs

PersonalGoal représente un objectif personnel.

Statuts :

- ACTIVE.
- ON_HOLD.
- ACHIEVED.
- ABANDONED.

Un objectif peut exister indépendamment d'un parcours et être associé à plusieurs parcours.

### 5.6 Bilans de progression

ProgressReview représente un bilan demandé par l'utilisateur.

Statuts :

- REQUESTED.
- GENERATING.
- DRAFT.
- VALIDATED.
- CANCELLED.
- FAILED.

Le bilan conserve les références et versions des sources utilisées.

Un bilan au statut DRAFT reste modifiable par l'utilisateur ; chaque modification enregistre une révision. Une fois VALIDATED, le bilan est immuable. La release de configuration IA ayant servi à la génération est tracée.

La génération d'un bilan est un traitement IA de fond (paragraphe 9.6). Si elle est préemptée, le bilan reste GENERATING : la tentative préemptée prend le statut terminal PREEMPTED et Laravel crée une nouvelle tentative.

Le journal est exclu lorsque sa politique est DENIED.

Les conversations ne sont pas incluses automatiquement.

Les hypothèses doivent être distinguées des faits déclarés.

L'absence de données ne doit jamais être interprétée comme une absence de progrès.

### 5.7 Favoris

UserBookmark représente un favori générique vers un élément d'un autre contexte.

Il conserve une référence et un type de cible.

Le contexte propriétaire reste responsable de vérifier l'existence et l'autorisation d'accès à la cible.

## 6. KnowledgeBase

### 6.1 Responsabilité

KnowledgeBase gère les documents utilisables par le système de recherche documentaire.

Il est propriétaire des métadonnées documentaires, des versions, de leur validation et de leur disponibilité pour la recherche.

Les traitements techniques d'extraction et d'indexation sont réalisés par Python.

### 6.2 KnowledgeDocument

KnowledgeDocument représente un document de la base.

Il contient notamment :

- Titre.
- Description.
- Type documentaire.
- Origine.
- Statut.
- Version courante.
- Version active.
- Paramètre d'utilisation par l'IA.

Les statuts sont :

- DRAFT.
- UNDER_REVIEW.
- APPROVED.
- ARCHIVED.

Seuls les documents APPROVED et activés peuvent être utilisés dans la recherche.

### 6.3 KnowledgeDocumentVersion

Une version documentaire conserve :

- Le fichier original.
- Son nom.
- Son type MIME.
- Sa taille.
- Son checksum.
- Son numéro de version.
- Son état de traitement.

Les versions sont immuables.

### 6.4 Ingestion et indexation

DocumentIngestion représente le traitement d'import et d'extraction.

Ses états sont :

- PENDING.
- PROCESSING.
- COMPLETED.
- PARTIAL : extraction partielle, signalée à l'administrateur.
- FAILED.
- UNSUPPORTED : format ou contenu non pris en charge.

DocumentIndexation représente l'indexation d'une version.

Une nouvelle version est préparée sans désactiver la version active.

La bascule est atomique : l'ancienne version reste disponible jusqu'à validation complète de la nouvelle.

Laravel décide des versions utilisables : il transmet à chaque recherche la liste des versions documentaires autorisées (`allowed_document_version_ids`). La bascule de version consiste à modifier cette liste, sans endpoint d'activation côté Python.

En cas d'échec, l'ancienne version reste active.

La désactivation de l'utilisation IA bloque immédiatement les futures recherches, même si le nettoyage physique de l'index est asynchrone.

### 6.5 Bibliothèque documentaire

La base est alimentée exclusivement par des documents importés et validés manuellement.

Aucun flux externe automatique n'est prévu dans le MVP.

L'extraction ne réalise pas d'OCR.

Les éléments graphiques et images sans texte exploitable sont ignorés et signalés lorsqu'ils sont détectés.

Les tableaux sont conservés lorsque leur structure textuelle peut être récupérée.

La segmentation doit respecter autant que possible les titres, sections, paragraphes, listes et tableaux.

Les passages doivent conserver leur provenance exacte.

### 6.6 Recherche

La recherche combine :

- Recherche plein texte.
- Recherche vectorielle.
- Fusion des résultats.
- Reranking éventuel, introduit uniquement si un besoin est mesuré.
- Diversification des sources.

La recherche doit éviter les doublons proches issus d'une même section lorsque des passages complémentaires sont disponibles.

Les résultats doivent conserver leur document, version, section et page.

Lorsque les sources sont insuffisantes ou contradictoires, le système doit le signaler sans inventer de consensus.

## 7. AiGovernance

### 7.1 Responsabilité

AiGovernance contrôle les configurations, les comportements et les règles de sécurité applicables aux fonctionnalités IA.

Il est propriétaire des modèles configurés, des prompts, des politiques, des évaluations et des releases.

### 7.2 Modèles

AIModel représente un modèle disponible dans l'environnement local.

AIModelConfiguration définit son utilisation dans un contexte donné.

Une configuration est associée à un usage : conversation, extraction de mémoire, analyse documentaire, bilan ou autre traitement.

Plusieurs usages peuvent partager le même modèle.

Aucun fallback automatique n'est prévu.

### 7.3 Prompts

PromptDefinition représente une définition de prompt.

PromptVersion représente une version immuable.

Les prompts sont composables :

- Prompt de base.
- Prompt spécifique à une tâche.
- Instructions complémentaires selon le contexte.

Chaque génération doit pouvoir être rattachée à la configuration de prompts utilisée.

### 7.4 Gouvernance

GovernancePolicy définit les règles de comportement applicables à l'IA.

Les politiques configurables ne peuvent pas neutraliser les garde-fous critiques protégés dans le code.

Le cadre d'intervention thérapeutique reste limité aux TCC dans le MVP.

### 7.5 Sécurité psychologique

SafetyPolicy définit les paramètres et protocoles de traitement des situations préoccupantes.

Les niveaux de sécurité internes sont :

- Niveau 0 : conversation ordinaire.
- Niveau 1 : détresse ou souffrance exprimée.
- Niveau 2 : pensées suicidaires ou d'automutilation sans danger imminent établi.
- Niveau 3 : danger imminent, tentative ou intention précise associée à un risque immédiat.

Ces niveaux constituent une classification opérationnelle interne, et non une échelle clinique validée.

Les situations de niveau 3 déclenchent une interruption du parcours TCC ordinaire et une orientation immédiate vers les secours et ressources adaptées, sans attendre une évaluation complète.

L'application ne prétend jamais avoir contacté les secours à la place de l'utilisateur.

Les règles critiques ne dépendent pas exclusivement de l'IA générative.

La détection et la réponse de sécurité déterministes sont implémentées dans Laravel, indépendamment de Python et d'Ollama. Le protocole détaillé sera défini dans `10-PROTOCOLE-SECURITE.md` (à venir, phase 7).

Les ressources d'urgence (EmergencyResource) sont gérées par pays. Le pays par défaut est la France (FR).

Les interventions de sécurité (SafetyIntervention) et leurs traces sont des données sensibles : elles ne contiennent que le minimum nécessaire, restent cloisonnées par utilisateur et n'apparaissent jamais dans les logs techniques ni dans les événements d'intégration.

### 7.6 Évaluations

EvaluationSuite définit un ensemble de scénarios de test versionnés.

EvaluationRun représente une exécution de ces tests.

Les évaluations peuvent mesurer :

- Cohérence des réponses.
- Pertinence.
- Qualité conversationnelle.
- Exactitude documentaire.
- Qualité des propositions de mémoire.
- Latence.
- Respect des règles de sécurité.

Toute défaillance critique de sécurité bloque l'activation d'une configuration.

L'exécution d'une évaluation est un traitement IA de fond, soumis aux règles de préemption du paragraphe 9.6.

### 7.7 Releases

AIConfigurationRelease représente un ensemble cohérent de versions de configuration.

Une release référence précisément les modèles, prompts et politiques utilisés.

Une release assemblée est immuable.

Une modification entraîne la création d'une nouvelle release.

L'activation est atomique à l'échelle de la configuration IA globale.

Un retour arrière réactive une release antérieure.

Chaque génération doit conserver la référence vers la release utilisée.

## 8. PlatformOperations

### 8.1 Responsabilité

PlatformOperations gère les opérations transversales de maintenance, de conservation et de cycle de vie des données.

Il ne devient pas propriétaire des données métier des autres contextes.

### 8.2 Sauvegardes

Backup représente une sauvegarde.

Le système doit prendre en charge :

- Base de données.
- Fichiers originaux.
- Métadonnées nécessaires à la restauration.
- Vérification d'intégrité.
- Politique de rétention.

Les sauvegardes sont écrites sur un support externe au poste. Une copie sur le même SSD ne constitue pas une sauvegarde.

Les sauvegardes sont chiffrées ; leur clé de chiffrement est conservée séparément.

### 8.3 Restauration

RestoreOperation représente une opération de restauration.

La restauration doit garantir la cohérence entre la base de données et les fichiers.

Les suppressions intervenues après la date d'une sauvegarde doivent être réappliquées après restauration.

Cette réapplication s'appuie sur un registre des suppressions (*deletion ledger*) : fichier JSONL en ajout seul, écriture sérialisée, entrée durable avant que le système n'accuse réception de la suppression, point de contrôle enregistré avant la capture d'une sauvegarde, rejeu idempotent. Le détail de sa structure est décrit dans `06-DICTIONNAIRE-DONNEES-POSTGRESQL.md`.

### 8.4 Exports

ExportRequest représente une demande d'export.

L'export doit être complet, structuré et portable.

Il peut inclure les données structurées et les fichiers originaux.

Il est généré de manière asynchrone. Formats initiaux : JSON, et JSON accompagné des fichiers.

Les fichiers temporaires d'export doivent être protégés et supprimés selon une durée de conservation définie.

### 8.5 Suppression

DeletionRequest représente une demande de suppression de données.

La suppression est orchestrée par étapes :

1. Vérification de la demande et confirmation.
2. Inscription durable dans le registre des suppressions, puis accusé de réception de la suppression.
3. Blocage des nouvelles opérations concernées.
4. Exécution des suppressions par contexte propriétaire.
5. Suppression des données dérivées.
6. Vérification de la suppression.
7. Confirmation finale.

La suppression est d'abord logique et immédiate : dès la confirmation, les données concernées deviennent inaccessibles et sont exclues de tout traitement. La suppression physique, y compris celle des données dérivées, est ensuite exécutée de façon asynchrone.

Les opérations doivent être idempotentes et leur progression consultable.

### 8.6 Notifications et maintenance

Notification représente une notification interne.

NotificationPreference gère les préférences de notification.

MaintenanceOperation représente une opération technique contrôlée.

Les rappels sont soumis au consentement explicite de l'utilisateur et doivent rester non culpabilisants.

Les préférences de notification couvrent l'activation, les catégories, la fréquence et les périodes de silence.

## 9. Principes transversaux

### 9.1 Propriété métier

Chaque contexte possède ses agrégats et ses règles métier.

Aucun contexte ne modifie directement les tables métier d'un autre contexte.

### 9.2 Autorisation

L'authentification et les autorisations restent contrôlées côté Laravel.

Le service Python ne peut pas élargir les droits accordés par Laravel.

Il ne lit aucune donnée métier : Laravel lui transmet uniquement les éléments nécessaires à chaque requête (prompt construit, identifiants de versions documentaires autorisées).

### 9.3 Événements

Les événements d'intégration transportent uniquement les informations nécessaires.

Ils ne contiennent jamais de texte conversationnel, de contenu de journal, de mémoire personnelle ou de passage documentaire sensible.

### 9.4 Données techniques et métier

Les données métier restent sous la responsabilité de leur contexte.

Les données techniques de traitement documentaire, d'embeddings et de recherche restent sous la responsabilité du service Python.

### 9.5 Suppression

La suppression d'une donnée métier doit prendre en compte ses données dérivées dans les autres contextes et dans les traitements techniques.

### 9.6 Traitements IA de fond et préemption

Un seul créneau d'inférence est disponible. Les générations de conversation (Companion) sont interactives et prioritaires : elles ne sont jamais préemptées.

Les traitements IA de fond sont :

- L'extraction de mémoires et les propositions de consolidation (PersonalContext).
- La génération des bilans de progression et des propositions de parcours (UserPractice).
- L'exécution des évaluations (AiGovernance).

Un traitement de fond en cours peut être préempté au profit d'une requête interactive. Chaque exécution constitue une tentative : une tentative préemptée prend le statut terminal PREEMPTED, n'est jamais présentée comme un résultat, et sa relance crée une nouvelle tentative, orchestrée par Laravel avec un délai et sans perte de l'idempotence métier.

Le détail des priorités et des codes d'erreur (`PREEMPTED`, `ENGINE_BUSY`) est décrit dans `07-CONTRATS-INTERSERVICES.md`.

## 10. Hors périmètre de ce document

Ce document ne définit pas :

- Le schéma détaillé des tables PostgreSQL, ses index et ses contraintes SQL (voir `06-DICTIONNAIRE-DONNEES-POSTGRESQL.md`).
- Les endpoints API, les formats JSON des contrats et les payloads d'événements (voir `07-CONTRATS-INTERSERVICES.md`).
- Les répertoires techniques détaillés de Python.
- Le plan de développement par phase.

Ces sujets sont décrits dans leurs documents spécialisés.

---

**Statut :** modèle métier et bounded contexts — version de travail à valider.

**Dépendances :** `01-VISION-PRODUIT.md`, `02-CAHIER-DES-CHARGES.md`, `03-ARCHITECTURE-DDD.md`, `06-DICTIONNAIRE-DONNEES-POSTGRESQL.md` et `07-CONTRATS-INTERSERVICES.md`.

**Changements v2 :** Conversation et Message distincts, titre automatique sans appel au modèle, restauration des conversations et entrées archivées, release par tentative, conversation jamais préemptée (`ENGINE_BUSY`), traitements IA de fond préemptables avec statut terminal PREEMPTED, mémoires saisies ACTIVE et mémoires extraites toujours PROPOSED, validation réservée au rôle `ADMIN`, états d'ingestion, reranking conditionné, règles d'urgence, sauvegardes chiffrées, export JSON, préférences de notification, contradiction par relation CONTRADICTS, validation éditoriale et étapes sans contenu, bilans révisables puis immuables, liste des versions documentaires autorisées, sécurité déterministe dans Laravel, sauvegardes externes, registre des suppressions, suppression logique puis physique, Python sans lecture des données métier (Phase 0).
