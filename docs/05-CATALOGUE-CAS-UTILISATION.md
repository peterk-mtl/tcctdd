# 05 — Catalogue des cas d'utilisation

## 1. Objectif

Ce document recense les cas d'utilisation fonctionnels de TCCTDD.

Chaque cas d'utilisation décrit une capacité observable de l'application, indépendamment de son implémentation technique.

Les cas d'utilisation sont regroupés par bounded context :

1. Companion
2. PersonalContext
3. TccContent
4. UserPractice
5. KnowledgeBase
6. AiGovernance
7. PlatformOperations

Les règles métier détaillées sont définies dans `04-BOUNDED-CONTEXTS.md`.

## 2. Acteurs

### Utilisateur

Utilisateur authentifié de l'application, disposant du rôle `USER`.

Il peut gérer ses conversations, ses mémoires, ses exercices, son journal, ses parcours, ses objectifs et ses données personnelles.

### Administrateur

Compte disposant du rôle `ADMIN`, autorisé à gérer les contenus éditoriaux, les documents, les configurations IA et les opérations de maintenance. Seul le rôle `ADMIN` peut valider et publier les contenus éditoriaux.

Dans le MVP personnel, le compte initial peut disposer du rôle `ADMIN`.

Dans ce catalogue, les mentions « Administrateur », « Auteur » et « Validateur » désignent toutes un compte disposant du rôle `ADMIN`.

### Service IA

Ensemble des traitements techniques exécutés par le service Python et les modèles locaux via Ollama.

Le service IA intervient à la demande de l'application. Il ne possède pas l'autorité métier sur les données utilisateur.

### Services internes

Composants applicatifs et workers responsables des traitements asynchrones, de la publication des événements et des opérations de maintenance.

---

# 3. Companion — Conversations

## UC-COM-001 — Créer une conversation

**Acteur :** Utilisateur

**Objectif :** démarrer une nouvelle conversation avec le compagnon.

**Préconditions :**
- L'utilisateur est authentifié.

**Scénario nominal :**
1. L'utilisateur demande la création d'une conversation.
2. Il peut préciser un titre ; à défaut, le titre est dérivé des premiers mots du premier message, sans appel au modèle.
3. Il peut choisir un type général ou thématique.
4. Le système crée la conversation.
5. La conversation devient accessible dans l'historique.

**Règles :**
- La conversation est créée sans message ; la séquence des messages démarre avec le premier message envoyé.

**Résultat :** une conversation active est disponible.

## UC-COM-002 — Consulter les conversations

**Acteur :** Utilisateur

**Objectif :** retrouver une conversation existante.

**Scénario nominal :**
1. L'utilisateur ouvre l'historique.
2. Le système affiche ses conversations.
3. L'utilisateur sélectionne une conversation.
4. Le système affiche ses messages dans l'ordre chronologique.

**Règles :**
- Seules les conversations appartenant à l'utilisateur sont accessibles.
- Les conversations archivées peuvent être consultées.
- Les conversations supprimées ne sont plus accessibles normalement.

## UC-COM-003 — Envoyer un message

**Acteur :** Utilisateur

**Objectif :** échanger avec le compagnon.

**Scénario nominal :**
1. L'utilisateur saisit un message.
2. Il l'envoie.
3. Le système crée le message utilisateur avec le statut COMPLETED.
4. Les règles de sécurité déterministes de Laravel analysent le message.
5. Le système prépare le contexte autorisé.
6. Il sélectionne la release de configuration IA active et applique le mode de récupération documentaire (`never`, `always` ou `auto`, sans appel LLM supplémentaire en mode `auto`).
7. Laravel construit le prompt complet et crée une tentative de génération qui référence la release utilisée.
8. Il transmet la demande au service Python.
9. Le compagnon génère une réponse.
10. La réponse est affichée progressivement.
11. La génération terminée est enregistrée.

**Alternatives :**
- Le modèle est indisponible : le système signale l'indisponibilité.
- Le service Python échoue : le message utilisateur reste conservé et la génération est signalée en échec.
- La connexion est interrompue : la tentative est marquée interrompue.
- Un signal de risque grave apparaît : le protocole de sécurité prend le relais.
- Le créneau d'inférence est occupé par une autre requête interactive : le service répond `ENGINE_BUSY`, la tentative échoue avec ce code et l'utilisateur peut régénérer la réponse. Une génération de conversation n'est jamais préemptée.

**Règles :**
- Le contenu de l'utilisateur est conservé après envoi.
- Le navigateur ne communique jamais directement avec le service Python.
- Les réponses sont associées à une tentative de génération.
- Les données personnelles ne doivent pas apparaître dans les logs techniques.

## UC-COM-004 — Interrompre une génération

**Acteur :** Utilisateur

**Objectif :** arrêter une réponse en cours.

**Scénario nominal :**
1. L'utilisateur demande l'arrêt.
2. Le système transmet une demande d'annulation.
3. La génération est arrêtée si possible.
4. La tentative est marquée CANCELLED ou INTERRUPTED selon la cause.

**Règles :**
- Le contenu partiel n'est pas présenté comme une réponse terminée.
- Il n'est pas automatiquement réinjecté dans la conversation.
- La tentative est conservée selon la politique de rétention.

## UC-COM-005 — Régénérer une réponse

**Acteur :** Utilisateur

**Objectif :** demander une nouvelle réponse à un message donné.

**Scénario nominal :**
1. L'utilisateur demande une régénération.
2. Le système conserve la tentative précédente.
3. Il crée une nouvelle tentative pour le même message assistant.
4. Il relance la génération.
5. La nouvelle tentative devient la réponse sélectionnée si elle aboutit.

**Règles :**
- Les tentatives précédentes ne sont pas écrasées.
- Une seule tentative est active à la fois.
- Une tentative terminée reste immuable.

## UC-COM-006 — Renommer une conversation

**Acteur :** Utilisateur

**Scénario nominal :**
1. L'utilisateur choisit de modifier le titre.
2. Il saisit le nouveau titre.
3. Le système enregistre le changement.

## UC-COM-007 — Archiver ou restaurer une conversation

**Acteur :** Utilisateur

**Scénario nominal :**
1. L'utilisateur demande l'archivage.
2. Le système modifie le statut de la conversation.
3. La conversation disparaît de la liste des conversations actives.
4. Elle reste consultable dans les archives.

**Alternative :**
- L'utilisateur restaure une conversation archivée : elle redevient active et réapparaît dans la liste des conversations actives.

## UC-COM-008 — Supprimer une conversation

**Acteur :** Utilisateur

**Scénario nominal :**
1. L'utilisateur demande la suppression.
2. Le système présente les conséquences de l'opération.
3. L'utilisateur confirme.
4. La conversation devient immédiatement inaccessible (suppression logique).
5. Les traitements de suppression physique et des données dérivées sont déclenchés de manière asynchrone.

**Règles :**
- La suppression est inscrite durablement dans le registre des suppressions avant que le système n'accuse réception de la suppression.

---

# 4. PersonalContext — Mémoire personnelle

## UC-PC-001 — Proposer une mémoire

**Acteur :** Service IA

**Objectif :** identifier une information potentiellement utile à la continuité de l'accompagnement.

**Scénario nominal :**
1. Le système analyse une source autorisée.
2. Il identifie une information potentiellement mémorisable.
3. Il détermine sa catégorie et sa nature épistémique.
4. Il crée une proposition de mémoire.
5. La proposition est présentée à l'utilisateur.

**Règles :**
- Toute mémoire extraite par l'IA, notamment depuis une conversation, est créée avec le statut PROPOSED, y compris lorsqu'elle reprend une déclaration explicite de l'utilisateur (statut épistémique DECLARED).
- Elle ne devient ACTIVE qu'après validation par l'utilisateur (UC-PC-003).
- Les interprétations restent des hypothèses.
- La confiance du modèle ne remplace pas la validation utilisateur.
- Les sources interdites ne sont pas exploitées.
- L'extraction est un traitement IA de fond : si elle est préemptée, la tentative prend le statut PREEMPTED et Laravel crée une nouvelle tentative.

## UC-PC-002 — Enregistrer une information déclarée

**Acteur :** Utilisateur

**Objectif :** permettre à l'utilisateur de saisir directement une information dans sa mémoire.

**Scénario nominal :**
1. L'utilisateur ouvre sa mémoire et saisit une information personnelle.
2. Il choisit ou confirme sa catégorie.
3. Le système crée une mémoire avec le statut de traitement ACTIVE et le statut épistémique DECLARED.
4. Il enregistre la saisie directe par l'utilisateur comme provenance.
5. La mémoire devient disponible selon les règles de sélection du contexte.

**Règles :**
- Ce cas couvre uniquement la saisie directe par l'utilisateur.
- Une information extraite d'une conversation par l'IA relève de UC-PC-001 et reste PROPOSED jusqu'à validation.

## UC-PC-003 — Valider ou rejeter une proposition

**Acteur :** Utilisateur

**Scénario nominal :**
1. L'utilisateur consulte une proposition.
2. Il peut la valider ou la rejeter.
3. Le système met à jour son statut.
4. L'historique de traitement est conservé.

## UC-PC-004 — Consulter les mémoires

**Acteur :** Utilisateur

L'utilisateur peut consulter les mémoires, leurs catégories, leur statut, leur provenance et leurs éventuelles relations.

Il peut distinguer les informations déclarées des hypothèses.

## UC-PC-005 — Corriger une mémoire

**Acteur :** Utilisateur

**Scénario nominal :**
1. L'utilisateur sélectionne une mémoire.
2. Il modifie son contenu.
3. Le système enregistre une nouvelle révision.
4. La provenance et l'historique sont conservés.

## UC-PC-006 — Archiver une mémoire

**Acteur :** Utilisateur

L'utilisateur peut archiver une mémoire devenue inutile sans supprimer son historique.

## UC-PC-007 — Signaler une contradiction

**Acteur :** Système IA / Utilisateur

**Scénario nominal :**
1. Le système détecte deux mémoires potentiellement contradictoires.
2. Il propose une MemoryRelation de type CONTRADICTS entre les deux mémoires.
3. Il présente les éléments concernés.
4. L'utilisateur peut confirmer, contester ou laisser la proposition en attente.

**Règles :**
- La contradiction est portée uniquement par la relation CONTRADICTS ; il n'existe pas d'agrégat de conflit distinct.
- Aucune mémoire n'est automatiquement supprimée ou remplacée.

## UC-PC-008 — Consolider des mémoires

**Acteur :** Service IA / Utilisateur

**Scénario nominal :**
1. Le système identifie des mémoires proches.
2. Il propose une consolidation.
3. Il précise les changements envisagés.
4. L'utilisateur accepte ou refuse.
5. En cas d'acceptation, les mémoires sont fusionnées ou révisées.
6. Les anciennes mémoires sont conservées et marquées SUPERSEDED lorsque nécessaire.

Une simple association de source sans changement de sens peut être automatisée si elle respecte les règles de fiabilité.

La préparation d'une proposition de consolidation est un traitement IA de fond, préemptible au profit d'une conversation.

## UC-PC-009 — Gérer les relations entre mémoires

**Acteur :** Utilisateur / Service IA

L'utilisateur peut consulter et valider les relations proposées.

Les relations doivent rester traçables et ne pas créer de cycles ou de liens incohérents lorsque leur nature impose des contraintes.

## UC-PC-010 — Assembler le contexte personnel

**Acteur :** Système applicatif

**Objectif :** préparer les informations utiles à une génération IA.

**Scénario nominal :**
1. Le système identifie l'usage demandé.
2. Il récupère les messages récents.
3. Il sélectionne les mémoires pertinentes.
4. Il vérifie les autorisations et exclusions.
5. Il prend en compte les relations et contradictions utiles.
6. Il applique les limites de contexte.
7. Il produit un contexte structuré avec les références et versions utilisées.

**Règles :**
- Le système ne charge pas toute la mémoire par défaut.
- Le journal n'est pas consulté automatiquement.
- Les informations interdites sont exclues.

---

# 5. TccContent — Catalogue éditorial TCC

## UC-TCC-001 — Créer un exercice

**Acteur :** Administrateur

L'administrateur crée un exercice avec son titre, sa description, ses catégories, ses tags et sa structure initiale.

## UC-TCC-002 — Modifier un exercice

**Acteur :** Administrateur

L'administrateur modifie le contenu d'un exercice en préparation.

Une modification d'un contenu publié nécessite une nouvelle version.

## UC-TCC-003 — Structurer un exercice

**Acteur :** Administrateur

L'administrateur peut organiser le contenu en sections et blocs modulaires.

Il peut ajouter :

- Des textes.
- Des instructions.
- Des questions.
- Des explications.
- Des blocs conditionnels.

Les conditions respectent les contraintes du MVP.

## UC-TCC-004 — Gérer les catégories et tags

**Acteur :** Administrateur

L'administrateur peut créer, modifier, organiser et archiver les éléments de classification.

Les catégories sont hiérarchiques et les tags transversaux.

L'IA peut proposer des classifications, mais ne peut pas créer ou modifier directement la taxonomie.

## UC-TCC-005 — Ajouter une justification méthodologique

**Acteur :** Administrateur

L'administrateur associe à une version :

- Une justification.
- Une base méthodologique.
- Un niveau de preuve éditorial.
- Des limites.
- Des références bibliographiques lorsque pertinentes.

## UC-TCC-006 — Soumettre un exercice à validation

**Acteur :** Administrateur (rôle `ADMIN`)

L'exercice passe en revue éditoriale.

L'auteur et le validateur sont tracés sur la version.

Par défaut, le MVP personnel fonctionne en mode auteur-validateur unique : le même compte `ADMIN` peut rédiger et valider. Une séparation stricte entre auteur et validateur peut être activée par configuration.

## UC-TCC-007 — Publier un exercice

**Acteur :** Administrateur (rôle `ADMIN`)

Après validation, une version est publiée et devient disponible pour les sessions utilisateur.

Une version publiée est immuable.

## UC-TCC-008 — Archiver un exercice

**Acteur :** Administrateur

L'exercice est retiré du catalogue actif sans modifier les sessions historiques.

## UC-TCC-009 — Créer un parcours éditorial

**Acteur :** Administrateur

L'administrateur crée un parcours composé d'étapes et de prérequis.

Chaque étape référence une version précise du contenu concerné.

## UC-TCC-010 — Publier un parcours éditorial

**Acteur :** Administrateur (rôle `ADMIN`)

Le parcours est vérifié puis publié sous forme de version immuable.

## UC-TCC-011 — Gérer les ressources pédagogiques

**Acteur :** Administrateur

L'administrateur importe, versionne, valide, publie et archive des ressources autonomes.

TccContent publie les ressources. L'activation de leur utilisation par l'IA relève de KnowledgeBase (UC-KB-006) et reste indépendante de leur publication.

---

# 6. UserPractice — Pratique personnelle

## UC-UP-001 — Parcourir le catalogue d'exercices

**Acteur :** Utilisateur

L'utilisateur consulte les exercices publiés, leurs descriptions, leurs objectifs et leurs précautions.

Il peut filtrer par catégorie ou tag.

## UC-UP-002 — Démarrer un exercice

**Acteur :** Utilisateur

**Scénario nominal :**
1. L'utilisateur sélectionne un exercice.
2. Le système identifie sa version publiée.
3. Il crée une nouvelle session.
4. La session commence avec le statut IN_PROGRESS.

Chaque démarrage crée une nouvelle session indépendante.

## UC-UP-003 — Répondre à un exercice

**Acteur :** Utilisateur

L'utilisateur répond aux questions et progresse dans les blocs.

Le système évalue les conditions d'affichage à partir des réponses déjà enregistrées.

Les réponses sont sauvegardées régulièrement.

## UC-UP-004 — Reprendre une session

**Acteur :** Utilisateur

L'utilisateur peut retrouver une session IN_PROGRESS et reprendre à partir de son état sauvegardé.

## UC-UP-005 — Terminer un exercice

**Acteur :** Utilisateur

L'utilisateur termine sa session.

Le système vérifie les contraintes nécessaires et passe la session au statut COMPLETED.

Les réponses deviennent immuables.

## UC-UP-006 — Abandonner une session

**Acteur :** Utilisateur

L'utilisateur peut abandonner une session en cours.

La session est conservée avec le statut ABANDONED.

## UC-UP-007 — Consulter l'historique des exercices

**Acteur :** Utilisateur

L'utilisateur consulte ses sessions passées, leur date, leur statut et leurs réponses selon les règles de conservation.

## UC-UP-008 — Créer une entrée de journal

**Acteur :** Utilisateur

L'utilisateur rédige et sauvegarde une entrée personnelle.

Le système applique la politique d'utilisation IA correspondante.

## UC-UP-009 — Modifier une entrée de journal

**Acteur :** Utilisateur

L'utilisateur modifie une entrée.

Une nouvelle version est créée lors de la sauvegarde effective.

## UC-UP-010 — Consulter le journal

**Acteur :** Utilisateur

L'utilisateur consulte ses entrées et leur historique de versions.

La consultation n'autorise pas automatiquement l'exploitation de leur contenu par l'IA.

## UC-UP-011 — Contrôler l'utilisation IA du journal

**Acteur :** Utilisateur

L'utilisateur peut :

- Définir une préférence générale.
- Autoriser une entrée particulière.
- Interdire une entrée particulière.
- Revenir à la préférence générale.

La préférence générale est DENIED par défaut.

## UC-UP-012 — Supprimer une entrée de journal

**Acteur :** Utilisateur

**Scénario nominal :**
1. L'utilisateur demande la suppression d'une entrée.
2. Le système présente les conséquences de l'opération.
3. L'utilisateur confirme.
4. L'entrée et ses versions deviennent immédiatement inaccessibles et sont exclues de tout traitement IA (suppression logique).
5. La suppression physique et celle des données dérivées (mémoires sourcées, références de bilans, index) sont déclenchées de manière asynchrone.

**Règles :**
- La suppression est inscrite durablement dans le registre des suppressions avant que le système n'accuse réception de la suppression.
- Les données dérivées sont traitées selon la politique de suppression.

## UC-UP-013 — Créer un parcours personnel

**Acteur :** Utilisateur

L'utilisateur crée un parcours et lui ajoute des étapes.

Il peut également accepter une proposition de parcours générée par l'IA. La génération de cette proposition est un traitement IA de fond, préemptible au profit d'une conversation.

## UC-UP-014 — Organiser un parcours

**Acteur :** Utilisateur

L'utilisateur peut :

- Ajouter une étape.
- Supprimer une étape.
- Réorganiser les étapes.
- Définir des prérequis.
- Mettre le parcours en pause.
- Le reprendre.

Les prérequis doivent former un graphe sans cycle.

## UC-UP-015 — Suivre les étapes d'un parcours

**Acteur :** Utilisateur

L'utilisateur démarre et termine les étapes disponibles.

Les états LOCKED et AVAILABLE sont calculés selon les prérequis.

Les états opérationnels sont conservés.

## UC-UP-016 — Gérer un objectif personnel

**Acteur :** Utilisateur

L'utilisateur peut créer, modifier, suspendre, atteindre ou abandonner un objectif.

Un objectif peut être lié à plusieurs parcours.

## UC-UP-017 — Demander un bilan de progression

**Acteur :** Utilisateur

**Scénario nominal :**
1. L'utilisateur demande un bilan.
2. Il sélectionne le périmètre souhaité.
3. Le système identifie les sources autorisées.
4. Il collecte les références et versions pertinentes.
5. Il demande une génération IA.
6. Le résultat est enregistré comme brouillon, avec la release de configuration IA utilisée.
7. L'utilisateur peut modifier le brouillon (chaque modification crée une révision), le valider ou le supprimer.
8. Une fois validé, le bilan devient immuable.

**Alternative :**
- La génération est préemptée au profit d'une conversation : le bilan reste GENERATING, la tentative préemptée prend le statut terminal PREEMPTED et Laravel crée une nouvelle tentative avec délai.

**Règles :**
- Les bilans ne sont pas générés automatiquement.
- Le journal DENIED est exclu.
- Les hypothèses sont identifiées.
- L'absence de données ne signifie pas absence de progrès.

## UC-UP-018 — Gérer les favoris

**Acteur :** Utilisateur

L'utilisateur peut ajouter ou retirer des favoris vers les contenus et éléments disponibles dans les autres contextes.

## UC-UP-019 — Archiver ou restaurer une entrée de journal

**Acteur :** Utilisateur

L'utilisateur archive une entrée de journal : elle disparaît de la liste des entrées actives, reste consultable dans les archives et conserve son historique de versions et sa politique d'utilisation IA.

L'utilisateur peut restaurer une entrée archivée : elle redevient active.

---

# 7. KnowledgeBase — Base documentaire

## UC-KB-001 — Importer un document

**Acteur :** Administrateur

L'administrateur importe un document dans la bibliothèque.

Les formats acceptés sont le PDF contenant du texte, le DOCX, le Markdown et le TXT. Le système ne réalise pas d'OCR : un PDF sans texte exploitable est refusé ou signalé.

Le système enregistre le fichier original et ses métadonnées.

## UC-KB-002 — Consulter un document

**Acteur :** Administrateur

L'administrateur consulte les métadonnées, versions, états de traitement et paramètres d'utilisation IA.

## UC-KB-003 — Créer une nouvelle version documentaire

**Acteur :** Administrateur

Une nouvelle version est importée sans remplacer immédiatement la version active.

## UC-KB-004 — Lancer l'ingestion

**Acteur :** Service interne / Administrateur

Le système déclenche :

1. L'extraction du texte.
2. La détection des sections.
3. La segmentation.
4. La préparation des passages.
5. La génération des embeddings.
6. L'indexation plein texte et vectorielle.
7. Les vérifications techniques.

Les traitements longs sont asynchrones.

L'ingestion passe par les états PENDING, PROCESSING, puis COMPLETED, PARTIAL, FAILED ou UNSUPPORTED. Une extraction partielle est signalée à l'administrateur.

## UC-KB-005 — Valider un document

**Acteur :** Administrateur

L'administrateur vérifie le contenu, les métadonnées et l'état de traitement avant approbation.

## UC-KB-006 — Activer l'utilisation IA

**Acteur :** Administrateur

L'administrateur autorise l'utilisation du document dans les recherches IA.

Seuls les documents approuvés peuvent être activés.

Laravel ajoute la version active du document à la liste des versions documentaires autorisées, transmise à chaque recherche. Il n'existe pas d'endpoint d'activation côté Python.

## UC-KB-007 — Désactiver l'utilisation IA

**Acteur :** Administrateur

Le document est immédiatement exclu des nouvelles recherches : Laravel retire ses versions de la liste des versions documentaires autorisées.

Le nettoyage technique de l'index peut se poursuivre en arrière-plan.

## UC-KB-008 — Activer une nouvelle version

**Acteur :** Système interne

Après traitement complet et validation, le système active atomiquement la nouvelle version.

L'ancienne version reste active en cas d'échec.

## UC-KB-009 — Rechercher dans la base documentaire

**Acteur :** Service IA

**Scénario nominal :**
1. Une demande nécessite une recherche documentaire.
2. Le système identifie les documents approuvés et activés.
3. Il effectue une recherche hybride.
4. Il fusionne et classe les résultats.
5. Il diversifie les passages pertinents.
6. Il retourne les passages avec leur provenance.

**Règles :**
- Une recherche insuffisante est signalée.
- Lorsque les sources divergent, les différences sont présentées sans fabriquer de consensus.
- Si le compagnon répond sans source pertinente, il signale explicitement qu'il s'appuie sur ses connaissances générales.

## UC-KB-010 — Consulter la provenance d'une source

**Acteur :** Utilisateur / Administrateur

L'application peut présenter le document, sa version, la section et la page à l'origine d'une information.

## UC-KB-011 — Archiver un document

**Acteur :** Administrateur

Le document est retiré du catalogue actif et exclu des recherches futures.

---

# 8. AiGovernance — Gouvernance IA

## UC-AI-001 — Enregistrer un modèle

**Acteur :** Administrateur

L'administrateur référence un modèle local disponible via Ollama.

Il renseigne ses capacités, sa version et ses contraintes matérielles.

## UC-AI-002 — Configurer un usage IA

**Acteur :** Administrateur

L'administrateur associe un modèle à un usage et configure ses paramètres.

## UC-AI-003 — Gérer les prompts

**Acteur :** Administrateur

L'administrateur crée, modifie, versionne et désactive des prompts.

Chaque modification crée une nouvelle version.

## UC-AI-004 — Gérer les politiques de gouvernance

**Acteur :** Administrateur

L'administrateur peut modifier les paramètres autorisés des politiques.

Les garde-fous critiques protégés dans le code ne sont pas modifiables depuis l'administration.

## UC-AI-005 — Gérer les ressources d'urgence

**Acteur :** Administrateur

L'administrateur peut gérer les ressources d'orientation adaptées aux pays concernés.

Les informations doivent être vérifiées et maintenues à jour.

## UC-AI-006 — Créer une suite d'évaluation

**Acteur :** Administrateur

L'administrateur définit un ensemble de scénarios de test versionnés.

## UC-AI-007 — Exécuter une évaluation

**Acteur :** Administrateur / Service interne

Le système exécute les scénarios et conserve les résultats associés aux versions testées.

L'exécution est un traitement IA de fond : si elle est préemptée, la tentative prend le statut PREEMPTED et Laravel crée une nouvelle tentative.

## UC-AI-008 — Préparer une release de configuration

**Acteur :** Administrateur

Une release rassemble les versions exactes des modèles, prompts et politiques nécessaires au fonctionnement de l'IA.

## UC-AI-009 — Évaluer une release

**Acteur :** Administrateur

La release est évaluée avant activation.

Une défaillance critique de sécurité bloque obligatoirement son activation.

## UC-AI-010 — Activer une release

**Acteur :** Administrateur

Après validation, la release est activée atomiquement.

Toutes les générations suivantes utilisent la configuration active.

## UC-AI-011 — Revenir à une release précédente

**Acteur :** Administrateur

L'administrateur peut réactiver une release précédemment validée.

Les générations déjà réalisées conservent leur référence à la release d'origine.

## UC-AI-012 — Appliquer le protocole de sécurité

**Acteur :** Système IA / Application

La détection et la réponse de sécurité sont déterministes et exécutées par Laravel, indépendamment du service Python et d'Ollama. Le protocole détaillé sera défini dans `10-PROTOCOLE-SECURITE.md` (à venir, phase 7).

Lorsqu'un signal préoccupant est détecté :

1. Le système évalue le niveau de risque selon les règles définies.
2. Il interrompt le déroulement TCC ordinaire si nécessaire.
3. Il fournit une réponse adaptée.
4. Il oriente vers les ressources d'urgence du pays concerné (France par défaut).
5. Il conserve uniquement les traces minimales nécessaires, considérées comme sensibles.

Les situations de danger imminent doivent déclencher une orientation immédiate vers les secours appropriés, sans attendre une évaluation complète.

Le système ne prétend jamais avoir contacté les secours à la place de l'utilisateur.

---

# 9. PlatformOperations — Opérations de plateforme

## UC-PLT-001 — Configurer les sauvegardes

**Acteur :** Administrateur

L'administrateur définit la fréquence, la rétention et les paramètres des sauvegardes.

## UC-PLT-002 — Lancer une sauvegarde

**Acteur :** Administrateur / Service interne

Le système sauvegarde la base de données et les fichiers nécessaires à la restauration.

La sauvegarde est écrite sur un support externe au poste : une copie sur le même SSD ne constitue pas une sauvegarde.

Un point de contrôle du registre des suppressions est enregistré avant la capture de la sauvegarde.

## UC-PLT-003 — Vérifier une sauvegarde

**Acteur :** Service interne

Le système vérifie l'intégrité des composants sauvegardés.

## UC-PLT-004 — Restaurer une sauvegarde

**Acteur :** Administrateur

L'administrateur sélectionne une sauvegarde et confirme la restauration.

Le système restaure de manière cohérente la base et les fichiers.

Les suppressions intervenues depuis la sauvegarde sont réappliquées par rejeu idempotent du registre des suppressions, à partir du point de contrôle associé à la sauvegarde.

## UC-PLT-005 — Demander un export

**Acteur :** Utilisateur

L'utilisateur demande un export complet de ses données.

Le système prépare de manière asynchrone un fichier structuré et les fichiers originaux associés.

Formats initiaux : JSON, et JSON accompagné des fichiers.

## UC-PLT-006 — Télécharger un export

**Acteur :** Utilisateur

L'utilisateur récupère son export via un accès protégé et temporaire.

## UC-PLT-007 — Demander la suppression de ses données

**Acteur :** Utilisateur

**Scénario nominal :**
1. L'utilisateur demande la suppression de ses données.
2. Le système présente les conséquences de l'opération.
3. L'utilisateur confirme explicitement.
4. Le système inscrit durablement la demande dans le registre des suppressions.
5. Il accuse réception de la suppression.
6. Les données concernées deviennent immédiatement inaccessibles et les nouvelles opérations sont bloquées (suppression logique).
7. L'orchestration de la suppression physique est déclenchée (UC-PLT-008).

**Règles :**
- L'accusé de réception n'intervient qu'après l'inscription durable dans le registre des suppressions.

## UC-PLT-008 — Orchestrer une suppression

**Acteur :** Service interne

**Scénario nominal :**
1. Le système crée une tâche de suppression par contexte propriétaire.
2. Chaque contexte supprime physiquement ses propres données et leurs données dérivées.
3. Le service Python supprime ses données techniques dérivées sur instruction de Laravel.
4. Le système vérifie l'achèvement de chaque tâche.
5. Il inscrit l'achèvement dans le registre des suppressions.
6. Il clôture la demande et produit une confirmation finale.

**Règles :**
- Les opérations sont idempotentes et peuvent reprendre après une panne.
- Leur progression est consultable.
- Aucun contexte ne supprime directement les données d'un autre contexte.

## UC-PLT-009 — Configurer les notifications

**Acteur :** Utilisateur

L'utilisateur peut activer ou désactiver les notifications, choisir leurs catégories, définir leur fréquence et configurer des périodes de silence.

Les notifications restent facultatives et non culpabilisantes.

## UC-PLT-010 — Gérer les opérations de maintenance

**Acteur :** Administrateur

L'administrateur consulte et exécute les opérations techniques autorisées.

## UC-PLT-011 — Configurer la télémétrie locale

**Acteur :** Administrateur

L'administrateur peut consulter et modifier les paramètres de collecte technique locale.

Aucune télémétrie externe obligatoire n'est prévue.

---

# 10. Exigences transversales des cas d'utilisation

Tous les cas d'utilisation doivent respecter les principes suivants :

1. L'utilisateur ne peut accéder qu'à ses propres données.
2. Les opérations sensibles nécessitent une autorisation explicite.
3. Les opérations longues sont exécutées de manière asynchrone lorsque nécessaire.
4. Les opérations asynchrones doivent exposer un état consultable.
5. Les traitements critiques doivent être idempotents.
6. Les erreurs doivent être explicites et ne pas divulguer de contenu sensible.
7. Les événements ne transportent pas de contenu personnel.
8. Les opérations de suppression doivent traiter les données dérivées.
9. L'indisponibilité de l'IA ne doit pas bloquer les fonctions non IA.
10. Les modifications de contenus publiés doivent préserver les versions historiques.

## 11. Hors périmètre

Les cas d'utilisation suivants ne sont pas prévus dans le MVP :

- Inscription publique.
- Paiement et abonnement.
- Partage social.
- Communauté.
- Synchronisation cloud automatique.
- Application mobile native.
- PWA.
- Audio et dictée vocale.
- OCR.
- Importation de documents personnels par l'utilisateur dans la base documentaire.
- Connexion automatique à des bibliothèques documentaires externes.
- Génération automatique de bilans périodiques.
- Fallback vers une IA distante.

---

**Statut :** catalogue des cas d'utilisation — version de travail à valider.

**Dépendances :** `01-VISION-PRODUIT.md`, `02-CAHIER-DES-CHARGES.md`, `03-ARCHITECTURE-DDD.md`, `04-BOUNDED-CONTEXTS.md`, `06-DICTIONNAIRE-DONNEES-POSTGRESQL.md` et `07-CONTRATS-INTERSERVICES.md`.

**Changements v2 :** acteurs rattachés aux rôles `ADMIN` et `USER` ; UC-COM-001 (titre automatique sans appel au modèle), 003 (statuts, prompt construit par Laravel, `ENGINE_BUSY`, conversation jamais préemptée), 007 (restauration) et 008 (suppression logique puis physique) ; UC-PC-001 et 002 (mémoire extraite toujours PROPOSED, saisie directe ACTIVE et DECLARED), 007 (relation CONTRADICTS) et 008 ; UC-TCC-006, 007, 010 et 011 ; UC-UP-012 (suppression), 013, 017 (bilan révisable puis immuable, préemption) et 019 (archivage du journal, nouveau) ; UC-KB-001, 004, 006, 007 et 009 (formats, états d'ingestion, versions autorisées, connaissances générales) ; UC-AI-007 et 012 (préemption des évaluations, sécurité déterministe dans Laravel) ; UC-PLT-002, 004, 005, 007, 008 et 009 (sauvegarde externe, registre des suppressions, export JSON, suppression orchestrée, périodes de silence) (Phase 0).
