# TCCTDD — Vision produit et périmètre

**Version :** 1.0  
**Statut :** Document de référence  
**Projet :** TCCTDD

## 1. Vision générale

TCCTDD est une application personnelle d'accompagnement fondée sur les thérapies cognitivo-comportementales (TCC).

Elle associe :
- un compagnon conversationnel utilisant une IA locale ;
- une mémoire personnelle structurée et évolutive ;
- un catalogue d'exercices TCC documentés et validés ;
- des parcours d'accompagnement guidés ;
- un journal personnel ;
- une base documentaire scientifique et pédagogique exploitée par un système RAG ;
- des outils de suivi et de réflexion sur les progrès.

L'objectif est de proposer un espace personnel permettant à l'utilisateur de mieux comprendre ses difficultés, d'identifier ses schémas de pensée et de comportement, d'expérimenter des stratégies concrètes et de suivre son évolution à son rythme.

TCCTDD est un outil d'auto-accompagnement. Il ne remplace pas un psychologue, un psychiatre, un médecin ou une prise en charge professionnelle.

## 2. Principes fondamentaux

### 2.1. Une approche exclusivement TCC

La méthodologie de l'application repose sur les thérapies cognitivo-comportementales.

Le compagnon peut écouter, reformuler, questionner, aider à prendre du recul ou proposer un exercice. Il ne doit pas transformer chaque conversation en séance structurée.

Il ne doit pas introduire arbitrairement des méthodes issues d'autres approches thérapeutiques.

### 2.2. Un accompagnement humain dans le ton, rigoureux dans le fond

Le compagnon adopte une posture :
- bienveillante et empathique ;
- non jugeante ;
- respectueuse du rythme de l'utilisateur ;
- encourageante sans infantilisation ;
- capable de questionner doucement certaines pensées ou interprétations ;
- attentive à ne pas culpabiliser.

Il peut adapter la longueur, le niveau de détail et le rythme de ses réponses aux préférences de l'utilisateur.

Il ne doit pas flatter systématiquement, approuver automatiquement ou chercher à rendre chaque échange artificiellement positif.

### 2.3. Une IA locale et une confidentialité prioritaire

L'inférence IA doit fonctionner localement, via Ollama.

Aucune conversation, donnée personnelle, entrée de journal ou mémoire personnelle ne doit être automatiquement envoyée à un service IA externe.

Le fonctionnement sans IA doit rester possible pour les fonctionnalités qui ne dépendent pas de celle-ci.

L'application ne doit pas rendre obligatoire un service cloud, une télémétrie externe ou une synchronisation distante.

### 2.4. Une mémoire personnelle contrôlée

Le compagnon peut construire progressivement une compréhension structurée du contexte de l'utilisateur.

Cette mémoire doit :
- être composée d'informations atomiques et catégorisées ;
- conserver leur provenance ;
- distinguer les faits déclarés des hypothèses produites par l'IA ;
- préserver les informations contradictoires sans choisir arbitrairement laquelle est vraie ;
- permettre à l'utilisateur de consulter, corriger, accepter ou refuser les propositions de mémoire ;
- conserver l'historique des révisions pertinentes.

Une interprétation psychologique proposée par l'IA ne devient jamais automatiquement un fait confirmé.

### 2.5. Une base de connaissances maîtrisée

La base documentaire est constituée exclusivement de documents importés et validés par un administrateur.

Elle ne doit pas aspirer automatiquement des sites web, des flux externes ou des contenus non vérifiés.

Les exercices et ressources publiés doivent disposer d'une justification pédagogique et méthodologique explicite, avec des références bibliographiques lorsque cela est pertinent.

La validation éditoriale ne doit jamais être présentée comme une certification scientifique ou médicale.

## 3. Public et usage initial

Le produit est conçu initialement pour un utilisateur unique, dans un cadre personnel.

L'architecture doit néanmoins permettre une ouverture future à plusieurs utilisateurs sans devoir reconstruire les fondations.

Chaque donnée personnelle est rattachée à son propriétaire et protégée par des contrôles d'accès côté serveur.

Le MVP ne comporte pas d'inscription publique.

## 4. Périmètre fonctionnel du MVP

Le MVP vise une application complète et cohérente, plutôt qu'une démonstration technique limitée à quelques fonctionnalités.

### 4.1. Compagnon conversationnel

- Conversations générales ou thématiques.
- Historique consultable.
- Génération progressive des réponses.
- Annulation explicite d'une génération.
- Nouvelle tentative après interruption ou erreur.
- Accès contrôlé au contexte personnel.
- Utilisation contextuelle de la base documentaire.
- Réponses capables de reconnaître leurs limites et incertitudes.

### 4.2. Mémoire personnelle

- Mémoires structurées et catégorisées.
- Provenance et historique des révisions.
- Propositions de nouvelles mémoires par l'IA.
- Validation utilisateur des interprétations.
- Gestion des contradictions.
- Propositions de fusion ou de révision.
- Contrôle utilisateur sur les informations conservées.

### 4.3. Catalogue d'exercices TCC

- Catalogue initial d'environ 20 à 30 exercices documentés.
- Catégories et étiquettes transversales.
- Versions immuables des exercices publiés.
- Structure commune et blocs de contenu modulaires.
- Questions et réponses adaptées au type d'exercice.
- Conditions d'affichage simples.
- Sauvegarde automatique des réponses.
- Historique des exercices réalisés.

Le catalogue doit pouvoir évoluer sans nécessiter de modification du code pour chaque nouvel exercice.

### 4.4. Parcours d'accompagnement

- Parcours éditoriaux définis et validés par l'administration.
- Parcours personnels créés manuellement ou avec l'aide de l'IA.
- Étapes obligatoires ou facultatives.
- Prérequis simples entre les étapes.
- Suivi de l'avancement.
- Acceptation explicite des parcours proposés par l'IA.

### 4.5. Journal personnel

- Entrées libres ou structurées.
- Historique des versions.
- Archivage et suppression.
- Politique d'accès IA configurable globalement et individuellement.

Par défaut, le journal n'est pas accessible à l'IA.

Autoriser la consultation d'une entrée ne signifie pas autoriser automatiquement l'extraction de souvenirs durables à partir de celle-ci.

### 4.6. Objectifs et progression

- Objectifs personnels.
- Association éventuelle à des parcours.
- Suivi des étapes réalisées.
- Bilans de progression demandés par l'utilisateur.
- Génération assistée par IA.
- Validation du bilan par l'utilisateur avant conservation.

Aucun score psychologique global ni système de séries culpabilisant ne doit être introduit.

### 4.7. Base documentaire et RAG

- Importation et validation administrative des documents.
- Gestion des versions.
- Extraction du texte natif.
- Découpage sémantique et structurel.
- Recherche hybride textuelle et vectorielle.
- Références précises aux documents et passages utilisés.
- Présentation transparente des divergences entre sources.
- Reconnaissance explicite de l'absence de documentation pertinente.

Aucun OCR ni analyse d'images n'est prévu dans le MVP.

### 4.8. Administration

- Gestion des exercices, ressources et parcours éditoriaux.
- Gestion de la base documentaire et de ses classifications.
- Gestion des modèles et prompts IA.
- Évaluation des configurations avant activation.
- Gestion des règles de sécurité.
- Historique des activations et retours arrière.

Les règles fondamentales de sécurité et de confidentialité ne doivent pas pouvoir être désactivées par une simple configuration administrative.

### 4.9. Exploitation et données personnelles

- Authentification locale.
- Préférences utilisateur.
- Export complet des données.
- Suppression du compte et des données associées.
- Sauvegarde et restauration.
- Notifications facultatives avec consentement explicite.
- Fonctionnement sans IA pour les fonctionnalités indépendantes de celle-ci.

## 5. Sécurité et limites de l'accompagnement

TCCTDD ne pose pas de diagnostic et ne prétend pas évaluer cliniquement l'état psychologique de l'utilisateur.

Un dispositif de sécurité doit détecter les signaux de détresse et appliquer des protocoles adaptés.

Les situations de danger immédiat doivent entraîner une orientation prioritaire vers les ressources d'urgence appropriées, sans attendre une analyse conversationnelle exhaustive.

Les ressources doivent être adaptées au pays de l'utilisateur. Pour la France, les références initiales comprennent notamment le 15, le 112 et le 3114.

L'application ne doit jamais prétendre avoir contacté les secours à la place de l'utilisateur.

## 6. Périmètre explicitement exclu du MVP

Les fonctionnalités suivantes ne sont pas prévues initialement :

- Inscription publique et gestion multi-utilisateurs avancée.
- Paiement ou abonnement.
- Réseau social, communauté ou partage public.
- Application mobile native.
- PWA.
- Dictée vocale, audio ou conversation vocale.
- Synchronisation cloud automatique.
- Envoi automatique de données à des services IA externes.
- Importation de documents personnels par les utilisateurs dans la base RAG.
- Collecte automatique de documentation externe.
- OCR et analyse d'images.
- Génération automatique de rapports sans demande utilisateur.
- Notifications imposées ou rappels culpabilisants.
- Méthodes thérapeutiques autres que les TCC.

Ces exclusions ne constituent pas nécessairement des interdictions définitives pour les versions futures.

## 7. Expérience utilisateur

L'application doit être responsive et utilisable sur ordinateur, tablette et mobile.

L'interface doit privilégier :
- la simplicité ;
- la lisibilité ;
- une navigation prévisible ;
- un faible niveau de charge cognitive ;
- l'accessibilité WCAG 2.2 AA ;
- des interactions non culpabilisantes.

Le français est la langue initiale. L'architecture doit permettre une internationalisation ultérieure.

## 8. Principes techniques structurants

- Monolithe modulaire organisé selon DDD.
- Laravel comme cœur métier et propriétaire des données fonctionnelles.
- Python comme service spécialisé pour l'ingestion documentaire, le RAG et l'orchestration IA.
- PostgreSQL avec pgvector.
- Ollama pour l'inférence locale.
- Vue 3, TypeScript, Inertia et Tailwind pour l'interface.
- Filament pour l'administration.
- Docker Compose pour l'environnement.
- Séparation stricte entre domaine, application, infrastructure et présentation.
- Contrôle des accès côté Laravel.
- Aucun accès direct de Python en écriture aux données métier Laravel.

Les versions exactes des dépendances seront vérifiées au démarrage de l'implémentation.

## 9. Critère général de réussite

Le MVP est réussi lorsque l'utilisateur peut utiliser quotidiennement une application cohérente pour :
1. échanger avec son compagnon ;
2. conserver une mémoire personnelle maîtrisée ;
3. réaliser des exercices TCC et retrouver ses réponses ;
4. tenir un journal avec un contrôle explicite de l'accès IA ;
5. suivre des objectifs et parcours ;
6. obtenir des réponses documentées lorsque cela est pertinent ;
7. conserver la maîtrise de ses données ;
8. utiliser les fonctionnalités non IA même lorsque le moteur IA est indisponible.

La priorité est la fiabilité, la confidentialité, la cohérence méthodologique et la qualité de l'accompagnement, plutôt que la multiplication des fonctionnalités.
