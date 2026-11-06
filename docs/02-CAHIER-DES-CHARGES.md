# TCCTDD — Cahier des charges fonctionnel

**Version :** 1.0  
**Statut :** Document de référence  
**Dépendance :** `01-VISION-PRODUIT.md`

## 1. Principes fonctionnels généraux

### 1.1. Propriété des données

Toutes les données personnelles sont rattachées à un utilisateur.

Chaque opération doit vérifier les droits d'accès côté serveur. Il ne suffit jamais de masquer une fonctionnalité dans l'interface.

### 1.2. Fonctionnement sans IA

Les fonctionnalités métier indépendantes de l'IA doivent rester disponibles lorsque le moteur IA est arrêté ou indisponible.

Une panne IA ne doit pas empêcher :
- la consultation du journal ;
- la consultation des exercices ;
- la sauvegarde des réponses ;
- la consultation des parcours ;
- la gestion des objectifs ;
- l'accès aux préférences ;
- l'export des données.

### 1.3. Traçabilité

Les opérations importantes doivent pouvoir être retracées techniquement sans conserver de contenu personnel dans les journaux techniques.

Les versions des exercices, documents, prompts, modèles et configurations IA doivent être identifiables.

### 1.4. Sauvegarde progressive

Les formulaires importants doivent sauvegarder leur progression afin d'éviter la perte de données en cas de fermeture ou d'interruption.

---

## 2. Authentification et préférences

### 2.1. Authentification

L'application utilise le système d'authentification natif de Laravel.

Fonctionnalités :
- connexion par identifiant et mot de passe ;
- déconnexion ;
- gestion de session ;
- réinitialisation du mot de passe via les mécanismes Laravel ;
- limitation des tentatives de connexion ;
- protection contre les accès non autorisés.

Aucun fournisseur d'identité externe n'est obligatoire.

### 2.2. Verrouillage automatique

Après 15 minutes d'inactivité, l'interface doit être verrouillée.

Le verrouillage :
- ne détruit pas la session ;
- ne supprime aucune donnée ;
- demande une nouvelle authentification ;
- doit être appliqué côté serveur, pas uniquement par un minuteur JavaScript.

La durée doit être configurable.

### 2.3. Préférences générales

L'utilisateur peut configurer :
- langue ;
- fuseau horaire ;
- préférences d'interface ;
- préférences générales de confidentialité ;
- préférences de comportement du compagnon ;
- politique d'accès IA par défaut au journal ;
- rappels et notifications facultatifs.

Les préférences déduites par le compagnon doivent être proposées ou consultables et corrigibles par l'utilisateur.

---

## 3. Compagnon conversationnel

### 3.1. Gestion des conversations

L'utilisateur peut :
- créer une conversation ;
- consulter la liste de ses conversations ;
- ouvrir une conversation ;
- renommer une conversation ;
- archiver une conversation ;
- restaurer une conversation archivée ;
- supprimer une conversation.

Une conversation peut être générale ou thématique.

L'historique doit conserver l'ordre exact des messages.

### 3.2. Gestion des messages

L'utilisateur peut envoyer un message texte et consulter les réponses du compagnon.

Règles :
- un message utilisateur est conservé dès son envoi ;
- un message envoyé est immuable ;
- une correction se fait par l'envoi d'un nouveau message ;
- une réponse assistant peut être régénérée ;
- les nouvelles tentatives sont conservées techniquement ;
- une seule génération active est autorisée par message assistant.

### 3.3. Génération progressive

La réponse doit apparaître progressivement pendant sa génération.

L'utilisateur peut explicitement interrompre une génération.

En cas d'interruption :
- le texte partiel est conservé temporairement dans la tentative ;
- il est clairement identifié comme incomplet ;
- il n'est pas traité comme une réponse terminée ;
- il n'est pas automatiquement réinjecté dans le contexte d'une nouvelle génération ;
- il est supprimé après une durée maximale de sept jours.

Une nouvelle tentative crée une nouvelle tentative de génération, sans écraser l'historique précédent.

### 3.4. Gestion des erreurs

Si le modèle configuré est indisponible :
- l'utilisateur reçoit un message compréhensible ;
- l'application conserve les messages déjà envoyés ;
- les fonctions indépendantes de l'IA restent disponibles ;
- aucun modèle de remplacement n'est activé automatiquement.

### 3.5. Comportement conversationnel

Le compagnon doit :
- écouter avant de proposer une solution ;
- adapter ses réponses au contexte ;
- éviter de transformer chaque échange en exercice ;
- pouvoir simplement accompagner une réflexion ;
- proposer des exercices lorsqu'ils sont pertinents ;
- faire des liens avec des éléments antérieurs lorsqu'ils sont utiles ;
- reconnaître ses incertitudes ;
- ne pas inventer d'interprétation psychologique.

Il peut prendre une initiative discrète lorsqu'une évolution ou une répétition semble pertinente, sans interrompre systématiquement l'utilisateur.

---

## 4. Mémoire personnelle

### 4.1. Création

La mémoire est composée d'éléments atomiques catégorisés.

Catégories initiales :
- contexte personnel ;
- difficultés récurrentes ;
- déclencheurs ;
- schémas de pensée ;
- schémas émotionnels ;
- objectifs ;
- stratégies utiles ;
- ressources ;
- progrès.

Les faits explicitement déclarés peuvent être enregistrés automatiquement comme informations déclarées.

Les interprétations produites par l'IA doivent être proposées à l'utilisateur avant de devenir des mémoires actives.

### 4.2. Validation et modification

L'utilisateur peut :
- consulter les mémoires ;
- accepter une proposition ;
- refuser une proposition ;
- corriger une information ;
- consulter son historique ;
- archiver une mémoire ;
- restaurer une mémoire lorsque cela est permis.

Chaque mémoire conserve sa provenance.

### 4.3. Contradictions

Lorsque deux mémoires semblent contradictoires :
- elles sont conservées ;
- le système signale le conflit potentiel ;
- le compagnon peut demander une clarification ;
- l'utilisateur peut reconnaître, résoudre ou écarter le conflit.

Le système ne choisit jamais automatiquement quelle information est vraie.

### 4.4. Consolidation

L'IA peut proposer :
- une fusion de mémoires similaires ;
- une révision d'une mémoire ;
- l'ajout d'une nouvelle source à une mémoire existante.

L'ajout d'une source sans modification de sens peut être automatique lorsque sa fiabilité est suffisante.

Une fusion ou une modification de sens nécessite une validation utilisateur.

Les anciennes mémoires restent conservées dans l'historique et peuvent être marquées comme remplacées.

---

## 5. Catalogue d'exercices TCC

### 5.1. Consultation

L'utilisateur peut :
- consulter le catalogue ;
- rechercher un exercice ;
- filtrer par catégorie ou étiquette ;
- consulter une fiche détaillée ;
- consulter les précautions et limites ;
- ajouter ou retirer un favori.

### 5.2. Structure des exercices

Chaque exercice comprend :
- une introduction ;
- des objectifs ;
- des instructions ;
- un ensemble de blocs de contenu ordonnés ;
- une conclusion ;
- des précautions ;
- une justification pédagogique et méthodologique ;
- un niveau de preuve ou de justification ;
- des références bibliographiques lorsque pertinentes.

Les blocs peuvent contenir des explications, des questions, des exemples, des tableaux ou des consignes.

Certains blocs peuvent dépendre d'une réponse précédente.

Les conditions restent simples et déclaratives dans le MVP.

### 5.3. Réalisation

L'utilisateur peut :
- démarrer un exercice ;
- répondre progressivement ;
- reprendre une réalisation en cours ;
- terminer un exercice ;
- abandonner une réalisation ;
- consulter les réalisations précédentes.

Chaque réalisation référence exactement la version de l'exercice utilisée.

Les réponses sont sauvegardées automatiquement, avec temporisation adaptée pour les champs textuels.

Les réponses restent modifiables tant que la réalisation est en cours.

Une réalisation terminée est immuable. Une correction nécessite une nouvelle réalisation.

### 5.4. Questions conditionnelles

Les blocs conditionnels ne s'affichent que lorsque leur condition est satisfaite.

Les réponses des blocs temporairement masqués sont conservées, mais ne sont pas transmises à l'IA tant que les blocs restent masqués.

Les conditions ne doivent pas permettre de créer des dépendances circulaires.

---

## 6. Parcours d'accompagnement

### 6.1. Parcours éditoriaux

L'utilisateur peut consulter les parcours publiés.

Chaque parcours comprend des étapes ordonnées pouvant référencer :
- un exercice ;
- une ressource ;
- une réflexion ;
- un bilan intermédiaire.

Les prérequis déterminent les étapes disponibles.

Une nouvelle version d'un exercice ne modifie pas rétroactivement un parcours déjà publié.

### 6.2. Parcours personnels

L'utilisateur peut :
- créer un parcours manuellement ;
- accepter un parcours proposé par l'IA ;
- créer un parcours avec assistance IA ;
- modifier ses étapes ;
- activer ou mettre en pause le parcours ;
- terminer ou archiver le parcours ;
- suivre sa progression.

Un parcours proposé par l'IA reste à l'état de brouillon tant qu'il n'a pas été accepté.

### 6.3. Prérequis

Les dépendances entre étapes doivent former un graphe sans cycle.

Les états disponibles ou verrouillés sont calculés à partir des prérequis.

Les étapes facultatives peuvent être ignorées.

---

## 7. Journal personnel

### 7.1. Gestion des entrées

L'utilisateur peut :
- créer une entrée ;
- modifier son contenu ;
- consulter son historique ;
- archiver une entrée ;
- restaurer une entrée archivée ;
- supprimer une entrée.

Les versions précédentes restent disponibles selon la politique de conservation.

### 7.2. Accès de l'IA

Trois politiques sont disponibles :
- `DEFAULT` : hérite de la préférence générale ;
- `ALLOWED` : autorise explicitement l'accès ;
- `DENIED` : interdit l'accès.

La préférence générale est désactivée par défaut.

Toute consultation par l'IA doit vérifier les droits au moment de l'accès.

Une entrée interdite ne doit jamais être récupérée par le RAG ni injectée dans un contexte conversationnel.

L'autorisation de consulter ne vaut pas autorisation d'extraire une mémoire durable.

---

## 8. Objectifs et progression

### 8.1. Objectifs personnels

L'utilisateur peut :
- créer un objectif ;
- le modifier ;
- le mettre en pause ;
- le marquer comme atteint ;
- l'abandonner ;
- l'associer à un ou plusieurs parcours.

Un objectif peut exister sans parcours associé.

### 8.2. Bilans de progression

L'utilisateur peut demander un bilan sur une période et un périmètre donnés.

Le système :
1. identifie les sources autorisées ;
2. enregistre leurs références et versions ;
3. génère un brouillon ;
4. présente le résultat à l'utilisateur ;
5. attend sa validation avant conservation définitive.

Les entrées de journal interdites sont exclues.

Les conversations ne sont pas incluses automatiquement.

Les hypothèses doivent être identifiées comme telles.

L'absence de données ne doit jamais être interprétée comme une absence de progrès.

Aucun score psychologique global n'est calculé.

---

## 9. Base documentaire et RAG

### 9.1. Importation

Seuls les administrateurs peuvent importer des documents.

Chaque document suit un cycle éditorial :
`DRAFT → UNDER_REVIEW → APPROVED → ARCHIVED`.

Les documents doivent être validés avant d'être accessibles au moteur de recherche.

### 9.2. Extraction

Le système extrait le texte natif des documents.

Les images, illustrations et contenus sans couche textuelle exploitable ne sont pas analysés.

Les états d'extraction doivent distinguer :
- `COMPLETED` ;
- `PARTIAL` ;
- `FAILED` ;
- `UNSUPPORTED`.

Une extraction partielle doit être signalée.

### 9.3. Indexation

L'indexation comprend :
- extraction ;
- normalisation ;
- découpage structurel et sémantique ;
- génération des embeddings ;
- indexation textuelle ;
- contrôles de cohérence.

La version précédemment active reste disponible jusqu'à l'activation complète de la nouvelle version.

Le basculement de version doit être atomique.

### 9.4. Recherche

Le RAG est contextuel.

Il est recommandé ou obligatoire lorsque la demande porte notamment sur :
- une technique TCC ;
- une notion théorique ;
- une justification méthodologique ;
- des références ou études ;
- une question nécessitant une documentation spécifique.

Une conversation informelle ou une simple reformulation ne nécessite pas systématiquement une recherche documentaire.

### 9.5. Références et divergences

Les réponses documentées doivent permettre d'identifier les documents, versions et passages utilisés.

Les références ne doivent jamais être inventées.

Lorsque les sources divergent, le compagnon présente les différences de manière nuancée en tenant compte :
- des populations étudiées ;
- des méthodes ;
- du niveau de preuve ;
- des limites des documents.

Lorsque la base ne contient pas de source pertinente, le compagnon peut utiliser ses connaissances générales, mais doit le signaler explicitement.

---

## 10. Gouvernance IA

### 10.1. Modèles

Plusieurs modèles peuvent être configurés selon les usages.

Une configuration par usage est sélectionnée explicitement.

Aucun remplacement automatique de modèle n'est autorisé.

### 10.2. Prompts

Les prompts sont spécialisés et composables :
- socle comportemental commun ;
- conversation générale ;
- accompagnement TCC ;
- analyse du journal ;
- exercices ;
- bilans de progression ;
- extraction de mémoire.

Les versions sont historisées.

### 10.3. Releases

Une release représente une configuration cohérente de l'ensemble du système IA.

Elle référence les versions exactes de ses composants.

Une release activée est immuable.

Toute modification crée une nouvelle release.

Le retour arrière réactive une ancienne release sans la modifier.

### 10.4. Évaluations

Toute nouvelle configuration doit être évaluée avant activation.

Les critères comprennent :
- sécurité ;
- confidentialité ;
- pertinence ;
- qualité conversationnelle ;
- cohérence TCC ;
- exactitude des références ;
- qualité de la mémoire ;
- performances.

Un échec critique de sécurité bloque l'activation indépendamment du score global.

---

## 11. Sécurité psychologique

Le système applique une politique de sécurité graduée :

- Niveau 0 : conversation ordinaire.
- Niveau 1 : détresse ou souffrance émotionnelle.
- Niveau 2 : pensées suicidaires ou d'automutilation sans danger imminent établi.
- Niveau 3 : danger imminent, tentative ou intention précise associée à un risque immédiat.

Le détecteur peut combiner règles déterministes, analyse contextuelle et assistance du modèle.

Une situation de danger plausible doit déclencher une orientation immédiate, sans attendre une évaluation complète.

Les protocoles doivent :
- être adaptés au niveau de risque ;
- éviter le jugement ;
- ne pas poser de diagnostic ;
- orienter vers les ressources appropriées ;
- ne jamais prétendre contacter les secours à la place de l'utilisateur.

---

## 12. Administration et validation éditoriale

L'administration permet :
- de gérer les exercices ;
- de versionner et valider les contenus ;
- de gérer les parcours éditoriaux ;
- de gérer les ressources ;
- d'importer et valider les documents ;
- de gérer les catégories et étiquettes ;
- de gérer les modèles, prompts et règles IA ;
- d'exécuter les évaluations ;
- d'activer ou de restaurer une configuration IA.

L'auteur d'un contenu et son validateur doivent être distincts lorsque plusieurs personnes interviennent.

Un mode de validation par une seule personne est prévu pour le contexte personnel initial et doit être explicitement identifié.

---

## 13. Notifications

Les rappels sont facultatifs.

L'utilisateur doit pouvoir :
- activer ou désactiver les notifications ;
- choisir les catégories ;
- définir une fréquence ;
- configurer des périodes de silence.

Aucun rappel ne doit culpabiliser l'utilisateur pour une absence d'activité.

Les notifications ne doivent pas devenir une condition d'utilisation de l'application.

---

## 14. Export, suppression et sauvegardes

### 14.1. Export

L'utilisateur peut demander un export complet comprenant :
- données structurées ;
- conversations ;
- mémoires ;
- journal et versions ;
- réponses aux exercices ;
- objectifs et parcours ;
- préférences ;
- fichiers personnels éventuellement associés.

L'export est généré de manière asynchrone et protégé par un lien temporaire.

Formats initiaux : JSON et JSON accompagné des fichiers.

### 14.2. Suppression

La suppression du compte doit :
1. demander une confirmation explicite ;
2. bloquer les nouvelles opérations ;
3. supprimer les données dans chaque contexte ;
4. supprimer les données dérivées et indexations ;
5. vérifier la suppression ;
6. produire une confirmation.

Les sauvegardes existantes ne sont pas réécrites individuellement.

Les suppressions doivent toutefois être réappliquées lors d'une restauration si elles sont intervenues après la date de sauvegarde.

### 14.3. Sauvegardes

Les sauvegardes doivent couvrir :
- PostgreSQL ;
- documents et fichiers ;
- configurations nécessaires à la restauration.

Elles sont chiffrées et leur clé est conservée séparément.

Des tests périodiques de restauration sont obligatoires.

---

## 15. Critères transversaux de validation

Une fonctionnalité ne peut être considérée comme terminée que si les points applicables suivants sont vérifiés :

- Les droits d'accès sont contrôlés côté serveur.
- Les invariants métier sont respectés.
- Les erreurs sont gérées explicitement.
- Les opérations sensibles sont idempotentes lorsque nécessaire.
- Les données personnelles ne sont pas exposées dans les logs.
- Les tests automatisés couvrent les scénarios nominaux et les erreurs.
- Les migrations fonctionnent sur PostgreSQL.
- L'interface reste utilisable sur ordinateur, tablette et mobile.
- Les parcours essentiels respectent les objectifs d'accessibilité WCAG 2.2 AA.
- Les fonctionnalités non IA restent disponibles lorsque le moteur IA est arrêté.

## 16. Critère de livraison du MVP

Le MVP est fonctionnel lorsque l'ensemble des parcours essentiels est utilisable de bout en bout :

1. Connexion à l'application.
2. Consultation du catalogue.
3. Réalisation et sauvegarde progressive d'un exercice.
4. Consultation de l'historique des réalisations.
5. Création et consultation d'une entrée de journal.
6. Contrôle explicite de l'accès IA au journal.
7. Conversation avec le compagnon et conservation de l'historique.
8. Consultation d'une mémoire et gestion de ses propositions.
9. Création et suivi d'un objectif ou parcours.
10. Recherche documentaire et citations lorsque pertinentes.
11. Export des données.
12. Sauvegarde et restauration vérifiées.

La livraison ne doit pas être considérée comme acquise sur la seule base d'une interface visuellement fonctionnelle. Les règles métier, la sécurité, les tests et la cohérence des données font partie intégrante du produit.
