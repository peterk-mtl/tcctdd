# 02 — Cahier des charges fonctionnel et technique

## 1. Objet du document

Ce document décrit les exigences fonctionnelles et techniques du MVP de TCCTDD.

Il complète `01-VISION-PRODUIT.md` en détaillant les fonctionnalités attendues, leurs règles de gestion, les contraintes d'utilisation et les critères d'acceptation.

Il constitue une référence pour la conception, le développement et la validation du produit.

En cas de contradiction, les décisions explicites validées dans les documents de conception spécialisés prévalent sur les descriptions générales de ce cahier des charges.

## 2. Périmètre général

### 2.1 Utilisation

- Un utilisateur actif initialement.
- Authentification privée obligatoire.
- Aucune inscription publique.
- Données cloisonnées par utilisateur.
- Interface responsive.
- Langue initiale : français.
- Architecture compatible avec une future internationalisation.

### 2.2 Environnement technique

- Application web locale.
- Docker Compose.
- Backend Laravel.
- Service Python indépendant pour les traitements IA et documentaires.
- PostgreSQL avec pgvector.
- Frontend Vue 3, TypeScript, Inertia et Tailwind.
- Administration via Filament.
- Inférence locale via Ollama.

### 2.3 Contraintes matérielles

L'application doit pouvoir fonctionner sur une machine sans GPU dédié. La machine de référence est de type ThinkPad T480 :

- Processeur Intel Core i5-8350U (4 cœurs, 8 threads).
- 32 Go de RAM.
- SSD de 256 Go.

Les contraintes principales sont le temps de calcul CPU et l'espace disque. Le choix des modèles sera fondé sur un benchmark réalisé sur cette machine (phase 1b), l'inférence restant locale via Ollama.

Les traitements IA doivent être contrôlés afin d'éviter une saturation des ressources :

- Une seule génération à la fois.
- Limitation des traitements simultanés.
- Gestion du chargement et du déchargement des modèles.
- Priorisation des opérations : une génération de conversation est toujours interactive et n'est jamais préemptée ; un traitement IA de fond peut être préempté au profit d'une conversation, puis relancé.
- Possibilité de différer les traitements lourds.
- Disponibilité des fonctionnalités non IA lorsque le moteur IA est indisponible.

## 3. Authentification et gestion des accès

### Exigences

- Authentification par identifiant et mot de passe.
- Sessions sécurisées.
- Protection contre les tentatives répétées de connexion.
- Déconnexion explicite.
- Verrouillage automatique après une période d'inactivité configurable, avec 15 minutes par défaut.
- Après inactivité, affichage d'un écran de verrouillage et nouvelle saisie du mot de passe, sans perte de la session en cours.
- Contrôle du verrouillage côté serveur : il ne repose pas uniquement sur du JavaScript dans le navigateur.
- Réinitialisation du mot de passe via les mécanismes Laravel natifs.
- Contrôle d'accès côté serveur à chaque opération.

### Règles

- Aucune donnée utilisateur ne doit être accessible sans authentification.
- Toute ressource métier doit être vérifiée par rapport à son propriétaire.
- Les identifiants UUID ne constituent pas un mécanisme d'autorisation.
- Les opérations administratives sont réservées au rôle `ADMIN`.

### Rôles

Deux rôles sont prévus : `ADMIN` et `USER`. Les autorisations sont contrôlées par les Policies Laravel, sans paquet de gestion des permissions.

Le rôle `ADMIN` est seul autorisé à administrer les contenus et à valider les contenus éditoriaux. Dans le MVP personnel, le compte initial peut disposer du rôle `ADMIN`.

### Préférences générales

L'utilisateur peut configurer :

- La langue.
- Le fuseau horaire.
- Le pays utilisé pour les ressources d'urgence (France par défaut).
- Les préférences d'interface.
- Les préférences générales de confidentialité.
- Les préférences de comportement du compagnon.
- La politique d'accès IA par défaut au journal.
- Les rappels et notifications facultatifs.

Les préférences déduites par le compagnon doivent être proposées à l'utilisateur, ou rester consultables et corrigibles par lui.

## 4. Tableau de bord

Le tableau de bord constitue le point d'entrée après authentification.

Il doit présenter de manière synthétique :

- Un accès à la conversation avec le compagnon.
- Les exercices en cours.
- Les parcours actifs.
- Les derniers éléments du journal.
- Les objectifs personnels.
- Des indicateurs simples d'activité et de progression.

L'interface ne doit pas transformer la progression en obligation ou générer de culpabilité en cas d'inactivité.

Les suggestions éventuelles doivent rester discrètes et pertinentes.

## 5. Compagnon conversationnel

### 5.1 Conversations

L'utilisateur doit pouvoir :

- Créer une conversation.
- Consulter ses conversations précédentes.
- Renommer une conversation.
- Archiver une conversation.
- Restaurer une conversation archivée.
- Reprendre une conversation active.
- Supprimer une conversation selon la politique de suppression définie.

Les conversations peuvent être générales ou thématiques.

À défaut de titre saisi, le titre est dérivé des premiers mots du premier message, sans appel au modèle. Il reste modifiable.

### 5.2 Messages

L'utilisateur doit pouvoir envoyer des messages textuels et consulter les réponses du compagnon.

Les réponses doivent pouvoir être affichées progressivement lorsque le modèle le permet.

Chaque message utilisateur est conservé après envoi. Il n'est pas modifiable : une correction passe par un nouveau message.

Les messages assistant sont associés à une tentative de génération. Une régénération crée une nouvelle tentative sans écraser l'historique technique des précédentes.

### 5.3 Génération

Le système doit :

- Construire le contexte utile à partir de l'historique et des mémoires autorisées.
- Appliquer les préférences et règles de gouvernance.
- Déterminer si une recherche documentaire est pertinente, selon le mode de récupération documentaire configuré.
- Construire le prompt complet côté Laravel, puis transmettre la requête au service Python.
- Afficher progressivement la réponse.
- Enregistrer le résultat final.
- Gérer les erreurs, interruptions et annulations.

Modes de récupération documentaire :

- `never` : aucune recherche documentaire.
- `always` : recherche documentaire systématique.
- `auto` : Laravel décide selon des règles déterministes, sans appel supplémentaire à un modèle de langage.

Laravel construit le prompt (historique, mémoires autorisées, extraits documentaires, release de configuration active). Le service Python ne compose pas le prompt métier.

Une génération interrompue ne doit jamais être présentée comme une réponse terminée.

Le texte partiel est conservé uniquement dans le cadre technique prévu, avec une durée de rétention limitée.

### 5.4 Posture conversationnelle

Le compagnon doit :

- Être bienveillant, empathique et non jugeant.
- Éviter les réponses infantilisantes ou artificiellement positives.
- Respecter le rythme de l'utilisateur.
- Ne pas imposer systématiquement des exercices.
- Distinguer observation, hypothèse et fait déclaré.
- Reconnaître ses limites et son incertitude.
- Rester cohérent avec le cadre TCC.

## 6. Mémoire personnelle

### Fonctionnalités

Le système doit pouvoir proposer et enregistrer des informations utiles à la continuité de l'accompagnement.

Les mémoires peuvent concerner :

- Le contexte personnel.
- Les difficultés récurrentes.
- Les déclencheurs.
- Les pensées et schémas identifiés.
- Les réactions émotionnelles.
- Les objectifs.
- Les stratégies utiles.
- Les ressources.
- Les progrès.

### Règles de gestion

- Une information saisie directement par l'utilisateur dans sa mémoire est enregistrée comme active et déclarée (`ACTIVE` + `DECLARED`).
- Une information extraite d'une conversation par l'IA est toujours créée comme proposition (`PROPOSED`), y compris lorsqu'elle reprend une déclaration explicite de l'utilisateur. Elle ne devient active qu'après validation par l'utilisateur.
- Une interprétation de l'IA reste une hypothèse tant qu'elle n'est pas validée.
- Une mémoire doit conserver sa provenance.
- Les contradictions potentielles doivent être signalées sans suppression automatique.
- Les consolidations qui modifient le sens des informations nécessitent une validation.
- L'utilisateur peut consulter, corriger, rejeter ou archiver les mémoires.
- Une mémoire ne doit pas être créée uniquement à partir d'une interprétation non vérifiée.

## 7. Catalogue d'exercices TCC

### Fonctionnalités

Le système doit proposer un catalogue administrable d'exercices.

L'utilisateur doit pouvoir :

- Parcourir les exercices.
- Consulter leur description et leurs objectifs.
- Démarrer une session.
- Reprendre une session en cours.
- Enregistrer ses réponses.
- Terminer ou abandonner une session.
- Consulter les sessions précédentes.

### Exigences de contenu

Chaque exercice doit comporter une structure commune et des blocs de contenu modulaires.

Les questions peuvent être conditionnelles selon les réponses précédentes.

Les conditions du MVP doivent rester simples et déclaratives.

Les exercices publiés doivent être versionnés et conserver leur historique éditorial.

Une session utilise une version précise de l'exercice. Une modification ultérieure du contenu ne modifie pas les sessions déjà commencées.

### Administration

L'administration doit permettre :

- La création et la modification des exercices.
- La gestion des versions.
- La classification par catégories et tags.
- La gestion des références bibliographiques.
- La validation éditoriale.
- La publication et l'archivage.

La validation éditoriale est réservée au rôle `ADMIN`. L'auteur et le validateur de chaque version sont tracés. Par défaut, le MVP personnel fonctionne en mode auteur-validateur unique ; une séparation stricte entre auteur et validateur peut être activée par configuration.

La validation éditoriale ne doit pas être présentée comme une certification scientifique ou médicale.

## 8. Journal personnel

### Fonctionnalités

L'utilisateur doit pouvoir :

- Créer une entrée.
- Consulter ses entrées.
- Modifier une entrée.
- Consulter les versions précédentes.
- Archiver une entrée.
- Restaurer une entrée archivée.
- Supprimer une entrée.

### Confidentialité

Chaque entrée dispose d'une règle d'utilisation par l'IA :

- Utilisation autorisée.
- Utilisation interdite.
- Utilisation selon la préférence générale de l'utilisateur.

La préférence générale est désactivée par défaut.

Les droits sont vérifiés au moment de chaque accès par l'IA. Une entrée interdite ne doit jamais être récupérée par la recherche documentaire ni injectée dans un contexte conversationnel.

La consultation d'une entrée par l'utilisateur ne vaut pas autorisation de l'utiliser dans une extraction de mémoire.

Les versions du journal sont créées lors des sauvegardes effectives, et non à chaque frappe.

## 9. Parcours personnalisés

### Fonctionnalités

L'utilisateur doit pouvoir :

- Créer un parcours.
- Ajouter des étapes.
- Réorganiser les étapes.
- Mettre un parcours en pause.
- Reprendre un parcours.
- Terminer ou archiver un parcours.

Les étapes peuvent être des exercices, des ressources, des réflexions ou des bilans intermédiaires.

L'IA peut proposer un parcours, mais celui-ci reste un brouillon jusqu'à validation de l'utilisateur.

Les dépendances entre étapes doivent être explicites et ne pas permettre de cycles.

## 10. Objectifs personnels

L'utilisateur doit pouvoir :

- Créer un objectif.
- Modifier sa description.
- Le mettre en pause.
- Le marquer comme atteint.
- L'abandonner.
- Le relier à un ou plusieurs parcours.

Un objectif peut exister indépendamment d'un parcours.

Aucun objectif ne doit être imposé automatiquement par le compagnon.

## 11. Bilans de progression

Les bilans sont déclenchés à la demande de l'utilisateur.

Ils peuvent exploiter les sources autorisées :

- Exercices réalisés.
- Objectifs personnels.
- Mémoires validées ou déclarées.
- Conversations selon le périmètre explicitement autorisé.
- Journal uniquement si son utilisation est autorisée.

Le bilan doit :

- Présenter les éléments observés.
- Identifier les évolutions possibles.
- Distinguer faits et hypothèses.
- Signaler les informations manquantes.
- Éviter les scores psychologiques globaux.
- Conserver les références aux données utilisées.

Un bilan généré est un brouillon. L'utilisateur peut le modifier, chaque modification étant enregistrée comme révision, puis le valider ou le supprimer. Un bilan validé est immuable. La release de configuration IA ayant servi à la génération est tracée.

La génération d'un bilan est un traitement IA de fond : elle peut être préemptée au profit d'une conversation, puis relancée automatiquement.

L'absence de données ne doit jamais être interprétée comme une absence de progrès.

## 12. Base documentaire et recherche RAG

### Fonctionnalités

L'administrateur doit pouvoir :

- Importer un document.
- Consulter ses métadonnées.
- Gérer ses versions.
- Valider son contenu.
- Activer ou désactiver son utilisation par l'IA.
- Consulter l'état de son indexation.
- Archiver un document.

### Traitement documentaire

Le service Python doit assurer :

- L'extraction du texte.
- La segmentation en sections et passages.
- La génération des embeddings.
- L'indexation vectorielle.
- L'indexation plein texte.
- La recherche hybride.
- Le classement et la diversification des résultats.

Les documents originaux sont conservés.

Les formats pris en charge dans le MVP sont le PDF contenant du texte, le DOCX, le Markdown et le TXT.

Le système ne réalise pas d'OCR dans le MVP : un PDF constitué d'images est refusé ou signalé comme non exploitable.

Les images, illustrations et contenus sans couche textuelle exploitable ne sont pas analysés.

Les états d'une ingestion documentaire distinguent :

- `PENDING` : ingestion demandée, pas encore commencée.
- `PROCESSING` : traitement en cours.
- `COMPLETED` : extraction complète.
- `PARTIAL` : extraction partielle, qui doit être signalée à l'administrateur.
- `FAILED` : échec du traitement.
- `UNSUPPORTED` : format ou contenu non pris en charge.

Les passages retrouvés doivent conserver leur provenance exacte : document, version, section et page lorsque disponible.

### Pertinence de la recherche

Une recherche documentaire est recommandée, voire nécessaire, lorsque la demande porte notamment sur :

- Une technique TCC.
- Une notion théorique.
- Une justification méthodologique.
- Des références ou des études.
- Une question nécessitant une documentation spécifique.

Une conversation informelle ou une simple reformulation ne nécessite pas systématiquement de recherche documentaire.

### Règles de recherche

- Seuls les documents approuvés et activés sont utilisables.
- Un document désactivé ne doit plus être proposé à la recherche.
- Une nouvelle version ne remplace l'ancienne qu'après validation complète de son indexation.
- Une recherche insuffisante doit être signalée.
- Les références bibliographiques ne doivent jamais être inventées.
- Les réponses documentées doivent permettre d'identifier les documents, versions et passages utilisés.

### Divergences et connaissances générales

Lorsque les sources divergent, le compagnon présente les différences de manière nuancée, sans fabriquer de consensus, en tenant compte :

- Des populations étudiées.
- Des méthodes.
- Du niveau de preuve.
- Des limites des documents.

Lorsque la base ne contient pas de source pertinente, le compagnon peut utiliser ses connaissances générales, mais doit le signaler explicitement.

## 13. Administration

L'interface Filament doit permettre de gérer :

- Les utilisateurs et leurs rôles (`ADMIN`, `USER`).
- Les exercices et leurs versions.
- Les catégories et tags.
- Les parcours éditoriaux.
- Les ressources documentaires.
- Les documents de la base de connaissances.
- Les modèles IA et leurs configurations.
- Les prompts.
- Les règles de gouvernance.
- Les évaluations et versions de configuration.
- Les opérations de maintenance.

Les règles critiques de sécurité ne doivent pas pouvoir être désactivées par une simple modification de configuration éditoriale.

## 14. Gouvernance de l'intelligence artificielle

### 14.1 Modèles

Les modèles sont configurables par type d'utilisation.

Aucun basculement automatique vers un service externe ou un autre modèle n'est prévu en cas d'échec.

Si le modèle nécessaire est indisponible, la fonctionnalité IA concernée doit signaler son indisponibilité. Les autres fonctions de l'application restent disponibles.

### 14.2 Prompts

Les prompts doivent être :

- Versionnés.
- Composables.
- Associés à un usage.
- Évaluables.
- Réversibles.

Une modification de configuration doit produire une nouvelle version contrôlée.

### 14.3 Évaluations

Les configurations doivent être évaluées avant activation.

Les tests de sécurité critiques sont bloquants, indépendamment du score moyen obtenu sur les autres critères.

Les évaluations doivent conserver les versions du modèle, des prompts, des politiques et du jeu de tests utilisés.

### 14.4 Sécurité psychologique

Le système doit identifier les situations de détresse et appliquer les protocoles d'orientation définis.

Les signaux de risque grave ne doivent pas être traités comme une conversation TCC ordinaire.

Les règles critiques doivent être testables et ne pas dépendre exclusivement du modèle génératif.

La sécurité déterministe est implémentée dans Laravel, indépendamment de Python et d'Ollama : elle reste donc active si le service Python ou le moteur d'inférence est indisponible ou défaillant.

Une situation de danger plausible doit déclencher une orientation immédiate vers les ressources d'urgence, sans attendre une évaluation complète.

Les ressources d'urgence sont adaptées au pays défini dans les préférences de l'utilisateur, la France étant le pays par défaut.

Les protocoles doivent éviter le jugement, ne pas poser de diagnostic et ne jamais prétendre contacter les secours à la place de l'utilisateur.

Le protocole détaillé sera défini dans `10-PROTOCOLE-SECURITE.md` (à venir, phase 7).

## 15. Service Python et communication interservices

Le service Python prend en charge les traitements techniques liés à l'IA et aux documents.

Il ne doit pas devenir propriétaire des données métier gérées par Laravel.

Les échanges doivent reposer sur des contrats explicites et versionnés, décrits dans `07-CONTRATS-INTERSERVICES.md`.

Le service Python n'est joignable que depuis le réseau Docker interne. Le navigateur ne communique jamais directement avec Python : tous les échanges passent par Laravel.

Les modes de communication sont :

- API synchrone pour les opérations courtes.
- Streaming pour les générations conversationnelles.
- Files de tâches pour les traitements longs.

Les requêtes doivent comporter les identifiants techniques nécessaires au suivi, à l'idempotence et à la corrélation.

Les erreurs doivent être normalisées et ne pas exposer de données personnelles dans les logs.

## 16. Sauvegarde, export et suppression

### Sauvegarde

Le système doit permettre :

- Des sauvegardes de la base PostgreSQL.
- Des sauvegardes des documents et fichiers.
- La sauvegarde des configurations nécessaires à la restauration.
- La vérification de leur intégrité.
- La restauration cohérente des données.
- Des tests périodiques de restauration, qui sont obligatoires.

Les sauvegardes sont écrites sur un support externe au poste (disque externe ou équivalent). Une copie sur le même SSD ne constitue pas une sauvegarde.

Les sauvegardes sont chiffrées. Leur clé de chiffrement est conservée séparément des sauvegardes.

### Export

L'utilisateur doit pouvoir demander un export complet de ses données, comprenant :

- Les données structurées.
- Les conversations.
- Les mémoires.
- Le journal et ses versions.
- Les réponses aux exercices.
- Les objectifs et parcours.
- Les préférences.
- Les fichiers personnels éventuellement associés.

L'export est généré de manière asynchrone et mis à disposition par un lien protégé et temporaire.

Formats initiaux : JSON, et JSON accompagné des fichiers.

### Suppression

La suppression doit être orchestrée entre les différents contextes.

Après la confirmation demandée à l'utilisateur, la suppression est d'abord logique et immédiate : les données deviennent inaccessibles et sont exclues de tout traitement. La suppression physique, y compris celle des données dérivées, est ensuite exécutée de façon asynchrone.

Elle doit inclure les données dérivées, index documentaires et mémoires associées selon les règles définies.

Les sauvegardes historiques suivent une politique de rétention spécifique. Lors d'une restauration, les suppressions intervenues après la date de sauvegarde doivent être réappliquées.

Pour cela, un registre des suppressions (*deletion ledger*) est tenu selon les principes suivants :

- Fichier JSONL en ajout seul (append-only), stocké hors de la base restaurée.
- Écriture sérialisée : une seule écriture à la fois.
- Entrée durable (écrite et synchronisée sur disque) avant que le système n'accuse réception de la suppression.
- Point de contrôle (checkpoint) enregistré avant la capture d'une sauvegarde.
- Rejeu idempotent après restauration : réappliquer une entrée déjà appliquée n'a aucun effet supplémentaire.

## 17. Notifications

Les rappels et notifications sont facultatifs et soumis au consentement explicite de l'utilisateur.

L'utilisateur doit pouvoir :

- Activer ou désactiver les notifications.
- Choisir les catégories.
- Définir une fréquence.
- Configurer des périodes de silence.

Aucun rappel ne doit culpabiliser l'utilisateur pour une absence d'activité.

Les notifications ne doivent pas devenir une condition d'utilisation de l'application.

## 18. Exigences non fonctionnelles

### Sécurité

- Authentification obligatoire.
- Autorisation côté serveur.
- Protection contre les accès interutilisateurs.
- Secrets exclus du dépôt Git.
- Logs sans contenu personnel.
- Chiffrement des volumes et sauvegardes.
- Contrats interservices contrôlés.

### Fiabilité

- Opérations idempotentes lorsque nécessaire.
- Gestion explicite des erreurs.
- Reprises contrôlées des tâches asynchrones.
- Transactions locales courtes.
- Pas de transaction SQL maintenue pendant un appel HTTP ou une génération IA.

### Maintenabilité

- Architecture modulaire.
- Séparation des responsabilités.
- Tests unitaires et d'intégration.
- Contrats documentés.
- Migrations versionnées.
- Versionnement des configurations et contenus.

### Performance

Les traitements doivent être adaptés à la configuration matérielle cible.

Les opérations longues doivent être asynchrones et leur état consultable.

Les chargements de données doivent être limités au contexte utile, sans charger systématiquement tout l'historique ou toute la mémoire.

### Accessibilité

Les parcours essentiels doivent respecter les objectifs d'accessibilité WCAG 2.2 AA.

L'interface doit rester utilisable sur ordinateur, tablette et mobile.

### Disponibilité hors ligne

Les fonctionnalités ne nécessitant pas l'IA doivent rester utilisables sans accès Internet.

L'indisponibilité d'Ollama ou du service Python ne doit pas rendre l'ensemble de l'application inutilisable.

## 19. Critères d'acceptation globaux

Le MVP doit satisfaire les conditions suivantes :

1. Un utilisateur authentifié peut accéder uniquement à ses propres données.
2. Les conversations sont conservées et leur historique est consultable.
3. Les générations interrompues sont identifiées comme telles.
4. Les régénérations conservent les tentatives précédentes.
5. La mémoire distingue les informations déclarées des hypothèses.
6. Les exercices sont versionnés et leurs sessions conservent la version utilisée.
7. Les réponses d'exercices sont sauvegardées et récupérables.
8. Le journal applique les préférences de confidentialité définies.
9. Les parcours et objectifs peuvent être gérés indépendamment.
10. Les bilans sont déclenchés à la demande et validés par l'utilisateur.
11. Les réponses documentaires conservent leur provenance.
12. Les règles de sécurité sont appliquées même en cas de défaillance du modèle.
13. Les données peuvent être sauvegardées, exportées et supprimées.
14. Les fonctionnalités non IA restent accessibles sans moteur IA.
15. Les échanges Laravel/Python respectent les contrats et les frontières de responsabilité.

## 20. Critère de livraison du MVP

### 20.1 Définition de terminé

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

### 20.2 Parcours essentiels

Le MVP est livrable lorsque l'ensemble des parcours essentiels est utilisable de bout en bout :

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

---

**Statut :** cahier des charges — version de travail à valider.

**Dépendances :** `01-VISION-PRODUIT.md` et `07-CONTRATS-INTERSERVICES.md`.

**Documents liés :** `03-ARCHITECTURE-DDD.md`, `04-BOUNDED-CONTEXTS.md`, `05-CATALOGUE-CAS-UTILISATION.md`, `06-DICTIONNAIRE-DONNEES-POSTGRESQL.md` et `10-PROTOCOLE-SECURITE.md` (à venir, phase 7).

**Changements v2 :** machine de référence 32 Go (i5-8350U, SSD 256 Go), priorité des conversations et préemption des traitements de fond, rôles `ADMIN` et `USER`, préférences générales, verrouillage côté serveur, titre automatique sans appel au modèle, distinction entre mémoire saisie et mémoire extraite, validation éditoriale, archivage et restauration, bilans révisables, modes de récupération documentaire, formats et états d'ingestion, pertinence de la recherche, divergences et connaissances générales, règles d'urgence, sécurité déterministe dans Laravel, renvoi aux contrats interservices, sauvegardes externes chiffrées, export JSON, suppression logique puis physique, registre des suppressions, notifications, accessibilité et critère de livraison du MVP (Phase 0).
