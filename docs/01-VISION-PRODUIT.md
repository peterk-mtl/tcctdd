# 01 — Vision produit TCCTDD

## 1. Présentation du projet

**TCCTDD** est une application personnelle d'accompagnement fondée sur les thérapies cognitivo-comportementales (TCC), intégrant un compagnon conversationnel utilisant une intelligence artificielle locale.

L'application vise à aider l'utilisateur à mieux comprendre ses difficultés, identifier ses schémas de pensée et ses réactions émotionnelles, expérimenter des stratégies adaptées et suivre son évolution dans le temps.

Elle combine plusieurs dimensions :

- Un compagnon conversationnel bienveillant et non jugeant.
- Un catalogue structuré d'exercices et de ressources TCC.
- Un journal personnel.
- Une mémoire évolutive du contexte personnel de l'utilisateur.
- Des parcours d'accompagnement personnalisables.
- Un suivi de la pratique et des progrès.
- Une base documentaire permettant de fonder les réponses sur des ressources sélectionnées.

Le projet est conçu en priorité pour un usage personnel, tout en reposant sur une architecture suffisamment structurée pour permettre une éventuelle ouverture à d'autres utilisateurs.

## 2. Vision et philosophie

L'objectif n'est pas de créer un chatbot généraliste auquel on aurait simplement ajouté quelques exercices psychologiques.

TCCTDD doit constituer un environnement cohérent d'accompagnement TCC, capable de conserver le contexte utile au fil du temps, de proposer des interventions adaptées et de respecter le rythme de l'utilisateur.

Les principes fondamentaux sont :

- **Bienveillance :** adopter une posture chaleureuse, empathique et non jugeante.
- **Respect du rythme :** ne pas imposer d'exercice, de méthode ou de progression.
- **Continuité :** conserver une mémoire structurée et évolutive des éléments utiles.
- **Cohérence méthodologique :** rester dans le cadre des TCC pour les interventions thérapeutiques.
- **Transparence :** distinguer les faits déclarés, les hypothèses et les interprétations.
- **Autonomie :** laisser l'utilisateur contrôler ses données, ses préférences et son parcours.
- **Prudence :** ne pas présenter l'application comme un professionnel de santé ou un substitut à une thérapie.

L'application doit pouvoir écouter et accompagner une conversation sans chercher systématiquement à transformer chaque échange en exercice ou en intervention.

## 3. Utilisateurs et positionnement

### 3.1 Utilisateur initial

Le produit est développé initialement pour un utilisateur unique.

L'architecture doit néanmoins prévoir une séparation stricte des données par utilisateur (`user_id`), une authentification privée et des mécanismes d'autorisation côté serveur.

### 3.2 Positionnement futur

Le produit doit rester techniquement extensible vers une utilisation par plusieurs personnes, sans que cela implique de développer immédiatement une plateforme publique.

Le MVP ne prévoit pas :

- D'inscription publique.
- De fonctionnalités sociales ou communautaires.
- De paiement ou d'abonnement.
- De marketplace.
- De mise en relation avec des thérapeutes.

## 4. Fonctionnalités principales

### 4.1 Compagnon conversationnel

Le compagnon est le point d'entrée principal de l'application.

Il doit pouvoir :

- Entretenir des conversations générales ou thématiques.
- Écouter et reformuler les difficultés exprimées.
- Aider à identifier les pensées automatiques et les schémas récurrents.
- Proposer des pistes de réflexion adaptées.
- Suggérer des exercices TCC lorsque cela est pertinent.
- S'appuyer sur le contexte personnel mémorisé.
- Utiliser les ressources documentaires disponibles lorsque nécessaire.
- Conserver l'historique des conversations.
- Permettre de relancer une génération interrompue.

Le compagnon doit adopter une posture bienveillante, naturelle et non infantilisante. Il peut encourager une remise en question douce, sans jugement ni pression.

### 4.2 Mémoire personnelle évolutive

L'application doit disposer d'une mémoire structurée permettant de conserver les informations utiles à la continuité de l'accompagnement.

Elle peut notamment contenir :

- Des éléments de contexte personnel.
- Des difficultés récurrentes.
- Des déclencheurs identifiés.
- Des schémas de pensée ou réactions émotionnelles.
- Des objectifs personnels.
- Des stratégies qui semblent aider.
- Des ressources et progrès constatés.

La mémoire ne doit pas traiter toutes les interprétations de l'IA comme des faits.

Les informations saisies directement par l'utilisateur dans sa mémoire sont enregistrées comme des informations déclarées. Les informations que l'IA extrait d'une conversation, même lorsqu'elles ont été explicitement déclarées, restent des propositions soumises à la validation de l'utilisateur. Les hypothèses et interprétations proposées par l'IA doivent rester identifiées comme telles et pouvoir être validées ou refusées.

L'utilisateur doit pouvoir consulter, corriger et contrôler les éléments mémorisés.

### 4.3 Catalogue d'exercices TCC

L'application doit proposer un catalogue d'exercices documentés et structurés.

Le MVP vise environ 20 à 30 exercices initiaux, avec une architecture permettant d'en ajouter sans modification du code applicatif.

Chaque exercice peut comporter :

- Une introduction.
- Des objectifs.
- Des consignes.
- Des questions et blocs de contenu.
- Des conditions d'affichage.
- Une conclusion.
- Des précautions d'utilisation.
- Une justification méthodologique et des références lorsque pertinentes.

Les exercices doivent être versionnés afin de préserver l'intégrité des sessions déjà réalisées.

### 4.4 Pratique personnelle et journal

L'utilisateur doit pouvoir :

- Démarrer et compléter des exercices.
- Reprendre un exercice en cours.
- Conserver l'historique de ses sessions.
- Tenir un journal personnel.
- Corriger ses entrées selon les règles de versionnement.
- Contrôler si le contenu de son journal peut être utilisé par l'IA.

La consultation du journal ne vaut pas autorisation automatique d'extraire des informations pour enrichir la mémoire personnelle.

### 4.5 Parcours personnalisés

L'application doit permettre de créer des parcours composés de plusieurs étapes :

- Exercices.
- Ressources documentaires.
- Temps de réflexion.
- Bilans intermédiaires.

Les parcours peuvent être créés manuellement ou proposés par l'IA.

Une proposition de l'IA reste un brouillon tant que l'utilisateur ne l'a pas acceptée.

### 4.6 Suivi des progrès

L'utilisateur peut demander ponctuellement un bilan de son évolution.

Ces bilans peuvent s'appuyer sur les exercices réalisés, les objectifs, les éléments de mémoire autorisés et les autres sources explicitement sélectionnées.

Ils doivent distinguer les observations des hypothèses, signaler les limites des informations disponibles et ne jamais produire de score psychologique global.

Aucun système de séries ou de récompenses culpabilisant ne doit être introduit.

Les bilans générés par l'IA restent des brouillons : l'utilisateur peut les modifier, chaque modification étant conservée comme révision, puis les valider ou les supprimer. Un bilan validé devient immuable.

### 4.7 Base documentaire et RAG

L'application dispose d'une base documentaire administrable contenant des ressources sélectionnées et importées manuellement.

Ces documents peuvent être exploités par un système de recherche hybride combinant recherche textuelle et vectorielle.

Le compagnon peut consulter cette base lorsqu'une réponse nécessite des connaissances TCC, une justification méthodologique ou des références documentaires.

Les réponses doivent préserver la provenance des informations et ne jamais inventer de sources ou de références.

Lorsque la documentation disponible est insuffisante, le compagnon doit le signaler clairement. S'il s'appuie alors sur ses connaissances générales, il doit l'indiquer explicitement.

La base documentaire est administrée : l'utilisateur final n'y importe pas ses documents personnels.

## 5. Intelligence artificielle et confidentialité

### 5.1 IA locale

L'inférence des modèles doit être réalisée localement via Ollama.

Aucune transmission automatique des conversations, du journal, des souvenirs personnels ou des documents vers un service d'IA externe n'est prévue.

Les modèles doivent être configurables selon les usages : conversation, extraction de mémoire, analyse documentaire, génération de bilans ou autres traitements.

### 5.2 Ressources matérielles

L'application doit prendre en compte une configuration matérielle modeste, de type ThinkPad T480 :

- Processeur Intel Core i5-8350U (4 cœurs, 8 threads).
- 32 Go de RAM.
- SSD de 256 Go.
- Fonctionnement CPU possible, sans GPU obligatoire.

Les contraintes principales sont le temps de calcul CPU et l'espace disque (modèles, index, documents, sauvegardes).

Les règles d'exploitation en découlent :

- Une seule génération IA à la fois.
- Gestion de la concurrence et du chargement des modèles.
- Limitation des traitements simultanés, les traitements lourds pouvant être différés.
- Inférence toujours locale via Ollama.

Le choix définitif des modèles sera fondé sur un benchmark réalisé sur cette machine de référence (phase 1b), en fonction de leur qualité, de leur latence, de leur consommation de ressources et de leur capacité à respecter les exigences de prudence et de cohérence du produit.

### 5.3 Confidentialité

La confidentialité est un principe structurant.

Le produit doit notamment prévoir :

- Une authentification privée.
- Une autorisation systématique des accès aux données.
- Un cloisonnement des données par utilisateur.
- Des sauvegardes sur un support externe (une copie sur le même SSD ne constitue pas une sauvegarde).
- Un chiffrement des volumes et des sauvegardes.
- Une gestion contrôlée des exports et suppressions.
- L'absence de télémétrie externe obligatoire.
- L'absence de données personnelles ou de contenus sensibles dans les logs techniques.

## 6. Sécurité et limites de l'accompagnement

TCCTDD est un outil personnel d'accompagnement et d'auto-observation. Il ne constitue ni un dispositif médical, ni un diagnostic, ni un substitut à un psychologue, psychiatre ou médecin.

L'application doit intégrer des règles de sécurité adaptées aux situations de détresse psychologique et aux risques d'automutilation ou de suicide.

Les situations préoccupantes doivent être traitées selon un protocole défini, avec une orientation vers les ressources d'aide appropriées.

Les règles critiques de sécurité ne doivent pas dépendre uniquement de la génération du modèle d'IA.

Les situations de danger immédiat doivent entraîner une orientation prioritaire vers les ressources d'urgence appropriées, sans attendre une analyse conversationnelle exhaustive.

Les ressources d'urgence sont adaptées au pays de l'utilisateur, la France étant le pays par défaut. Pour la France, les références initiales comprennent notamment le 15, le 112 et le 3114.

L'application ne doit jamais prétendre avoir contacté les secours à la place de l'utilisateur.

L'application doit également éviter les diagnostics, les affirmations psychologiques non fondées et les interprétations présentées comme certaines.

## 7. Périmètre du MVP

Le MVP correspond à une version fonctionnelle complète intégrant les modules essentiels de manière cohérente.

Il ne s'agit pas d'une simple démonstration du compagnon conversationnel.

### Inclus dans le MVP

- Authentification privée.
- Interface responsive.
- Compagnon conversationnel avec historique.
- Inférence locale via Ollama.
- Mémoire personnelle structurée.
- Catalogue initial d'exercices TCC.
- Exécution et suivi des exercices.
- Journal personnel.
- Parcours personnalisés.
- Bilans de progression à la demande.
- Administration du contenu via Filament.
- Base documentaire et recherche RAG.
- Gestion des modèles et des prompts.
- Gouvernance et règles de sécurité.
- Sauvegarde, export et suppression des données.
- Fonctionnement hors ligne des fonctionnalités ne nécessitant pas l'IA.

### Exclus du MVP

- Inscription publique et gestion multi-utilisateurs avancée.
- Paiements et abonnements.
- Communauté ou partage social.
- Application mobile native.
- PWA.
- Audio, dictée vocale et reconnaissance vocale.
- Synchronisation cloud automatique.
- Envoi automatique de données à des services d'IA externes.
- Génération automatique de rapports périodiques.
- Intégrations obligatoires avec des services externes.
- Connexion à des bases documentaires externes ou flux de contenus automatiques.
- Importation de documents personnels par l'utilisateur dans la base documentaire.
- OCR et analyse d'images de documents.
- Notifications imposées ou rappels culpabilisants.
- Méthodes thérapeutiques autres que les TCC.
- Fonctionnalités de téléconsultation.

Ces éléments pourront éventuellement être étudiés ultérieurement, sans conditionner l'architecture initiale.

## 8. Expérience utilisateur

L'application doit proposer une interface claire, calme et accessible.

Les principes UX sont :

- Une navigation simple.
- Un tableau de bord équilibré.
- Un accès rapide aux conversations, exercices, journal et parcours.
- Une présentation lisible de la progression.
- Des interactions non culpabilisantes.
- Une utilisation possible sur ordinateur, tablette et mobile.
- Un faible niveau de charge cognitive.
- Le respect de l'accessibilité WCAG 2.2 AA pour les parcours essentiels.
- Une interface initialement en français, avec une architecture compatible avec une future internationalisation.

L'onboarding peut être hybride : une première configuration guidée, complétée progressivement au fil de l'utilisation.

Les suggestions et initiatives du compagnon doivent rester discrètes et proportionnées, sans surcharge ni interruption permanente.

## 9. Principes techniques directeurs

L'application repose sur une architecture modulaire de type monolithe modulaire, organisée autour de domaines métier clairement délimités.

Les principes directeurs sont :

- Laravel comme socle métier principal.
- Python pour les traitements IA, documentaires et RAG.
- PostgreSQL avec pgvector et recherche plein texte.
- Vue 3, TypeScript, Inertia et Tailwind pour l'interface.
- Filament pour l'administration.
- Ollama pour l'inférence locale.
- Docker Compose pour l'environnement d'exécution.
- Séparation claire entre domaine, application, infrastructure et présentation, le domaine restant indépendant de Laravel et d'Eloquent.
- Un module Laravel par bounded context, les dépendances entre modules passant uniquement par leurs contrats publics (`Contracts/`).
- Propriété des données métier par Laravel : le service Python n'a aucun droit sur ces données et ne communique qu'à travers des contrats versionnés.
- Tests automatisés à chaque niveau, avec des tests d'architecture garantissant le respect des frontières.

Les versions exactes des dépendances seront vérifiées au démarrage de l'implémentation.

## 10. Critère général de réussite

Le MVP est réussi lorsque l'utilisateur peut utiliser quotidiennement une application cohérente pour :

1. Échanger avec son compagnon.
2. Conserver une mémoire personnelle maîtrisée.
3. Réaliser des exercices TCC et retrouver ses réponses.
4. Tenir un journal avec un contrôle explicite de l'accès IA.
5. Suivre des objectifs et parcours.
6. Obtenir des réponses documentées lorsque cela est pertinent.
7. Conserver la maîtrise de ses données.
8. Utiliser les fonctionnalités non IA même lorsque le moteur IA est indisponible.

La priorité est la fiabilité, la confidentialité, la cohérence méthodologique et la qualité de l'accompagnement, plutôt que la multiplication des fonctionnalités.

---

**Statut :** vision produit — version de travail à valider.

**Documents liés :** `02-CAHIER-DES-CHARGES.md` (exigences détaillées), `03-ARCHITECTURE-DDD.md` (architecture) et `07-CONTRATS-INTERSERVICES.md` (contrats interservices).

**Changements v2 :** machine de référence 32 Go, sauvegardes externes, section 9 complétée, distinction entre mémoire saisie et mémoire extraite, bilans révisables, règles d'urgence, accessibilité, exclusions du MVP et critère général de réussite réintégrés (Phase 0).
