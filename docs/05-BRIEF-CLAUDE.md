# TCCTDD — Brief de planification pour Claude

## 1. Rôle et objectif

Tu interviens comme architecte logiciel et planificateur technique du projet TCCTDD.

Ton rôle est d'analyser le dossier de conception existant et de produire un plan d'implémentation détaillé, destiné à être exécuté ensuite par Composer 2 dans Cursor.

**Tu ne dois pas implémenter le projet.** Tu ne dois pas générer les classes, les migrations, les composants frontend ou les fichiers applicatifs. Cette étape sera réalisée séparément par Composer 2.

Ton livrable doit transformer la conception fonctionnelle et technique en étapes de développement concrètes, vérifiables et ordonnées.

## 2. Lecture et analyse préalables

Avant de rédiger le plan :

1. Parcours tous les documents du dossier de conception.
2. Identifie leur sujet, leur version et leur statut.
3. Repère les décisions métier, techniques, architecturales et de sécurité.
4. Identifie les dépendances entre les modules.
5. Détecte les contradictions, doublons et informations manquantes.
6. Vérifie que les choix décrits sont cohérents entre le cahier des charges, le DDD, le dictionnaire de données, les cas d'utilisation et les contrats API.

Ne commence pas le plan avant d'avoir terminé cette analyse.

## 3. Autorité des décisions

Les décisions explicitement validées dans les documents constituent les exigences du projet.

Ne les remplace pas par tes préférences techniques.

En cas de contradiction :
- identifie précisément les documents concernés ;
- privilégie une décision explicite et validée plutôt qu'une proposition ancienne ;
- signale les contradictions qui ne peuvent pas être résolues sans décision produit ;
- ne modifie jamais silencieusement une règle métier ou de confidentialité.

Tu peux prendre les décisions techniques de faible impact nécessaires à l'implémentation, à condition de les justifier et de les documenter.

Ne sollicite l'utilisateur que pour les décisions qui relèvent réellement de sa vision produit, de la confidentialité, de la sécurité ou du comportement de l'application.

Évite de lui redemander des choix déjà tranchés.

## 4. Architecture à respecter

Le projet repose sur :

- Laravel pour le cœur métier ;
- une architecture DDD organisée en monolithe modulaire ;
- sept bounded contexts clairement séparés ;
- Python pour les traitements documentaires et l'orchestration IA ;
- Ollama pour l'inférence locale ;
- PostgreSQL avec pgvector et recherche full-text ;
- Vue 3, TypeScript, Inertia et Tailwind ;
- Filament pour l'administration ;
- Docker Compose pour l'environnement ;
- Pest pour les tests PHP.

Le service Python ne possède pas les données métier Laravel.

Les contextes ne doivent pas accéder directement aux tables métier des autres contextes.

Le domaine ne doit dépendre ni de Laravel, ni d'Eloquent, ni de Python.

Les règles validées incluent notamment :
- pas de `strict_types` ;
- DTO avec spatie/laravel-data ;
- identifiants UUID ;
- dates UTC ;
- événements via Outbox ;
- idempotence des consommateurs ;
- autorisations contrôlées par Laravel ;
- absence de contenu personnel dans les logs techniques.

Les versions exactes des frameworks et packages doivent être vérifiées au moment de la planification, sans supposer une compatibilité non vérifiée.

## 5. Contraintes produit et infrastructure

Le MVP doit être complet et cohérent, mais ne doit pas implémenter prématurément toutes les évolutions possibles.

Contraintes importantes :

- application local-first ;
- fonctionnement sans service externe obligatoire ;
- IA locale via Ollama ;
- machine cible limitée, notamment environ 8 Go de RAM et sans GPU dédié ;
- les fonctionnalités non IA doivent rester utilisables si Ollama ou Python est indisponible ;
- pas de fallback automatique vers un autre modèle ;
- pas de synchronisation cloud automatique ;
- pas de téléversement documentaire par l'utilisateur final dans la KnowledgeBase ;
- pas d'OCR ni d'analyse d'images dans le MVP ;
- bibliothèque documentaire administrée et validée ;
- sécurité et confidentialité prioritaires ;
- aucune fonctionnalité clinique ou médicale revendiquée.

Le plan doit tenir compte des contraintes réelles de ressources et éviter de multiplier les modèles chargés simultanément.

## 6. Construction du plan

Produis un plan progressif, structuré en phases et en tâches suffisamment petites pour être confiées à Composer 2.

Pour chaque phase, indique :

- objectif ;
- périmètre ;
- dépendances ;
- modules concernés ;
- tâches ordonnées ;
- fichiers ou zones du projet concernés, lorsque cela peut être déterminé ;
- migrations nécessaires ;
- contrats à définir ;
- tests à écrire ;
- critères d'acceptation ;
- conditions de fin de phase.

Chaque tâche doit être assez précise pour que Composer 2 puisse l'implémenter sans devoir réinterpréter toute l'architecture.

Évite les tâches vagues comme « implémenter le module Companion ».

Préfère des étapes limitées, par exemple :
- créer le squelette du contexte ;
- définir les entités et invariants ;
- définir les interfaces de repository ;
- créer les migrations ;
- implémenter un cas d'utilisation ;
- ajouter les tests unitaires ;
- ajouter les tests d'intégration PostgreSQL ;
- exposer le contrat HTTP ;
- vérifier les autorisations.

## 7. Ordre de développement

Propose un ordre qui respecte les dépendances réelles.

Le premier objectif est une tranche verticale fonctionnelle :

1. authentification ;
2. consultation du catalogue d'exercices ;
3. démarrage d'une réalisation ;
4. sauvegarde automatique des réponses ;
5. reprise d'une réalisation ;
6. finalisation ;
7. consultation de l'historique.

Cette première tranche doit fonctionner sans IA.

L'IA, le RAG et les fonctionnalités de mémoire seront intégrés progressivement, après stabilisation des fondations métier.

Ne cherche pas à générer des centaines de fichiers dès le départ.

## 8. Format des tâches destinées à Composer 2

Pour chaque tâche, prépare une instruction autonome comprenant :

**Contexte :** ce qui existe déjà et pourquoi la tâche est nécessaire.

**Objectif :** résultat concret attendu.

**Périmètre :** ce qui doit être modifié et ce qui ne doit pas l'être.

**Contraintes :** règles DDD, conventions, sécurité et décisions produit applicables.

**Implémentation attendue :** éléments précis à créer ou modifier.

**Tests :** comportements à couvrir.

**Critères d'acceptation :** conditions objectives permettant de vérifier le résultat.

**Vérification :** commandes de tests, d'analyse statique ou de validation à exécuter.

Les tâches doivent être indépendantes autant que possible, mais leurs dépendances doivent être explicites.

Prévois des points de contrôle après chaque étape importante.

## 9. Qualité et sécurité

Le plan doit intégrer dès le départ :

- tests unitaires du domaine ;
- tests applicatifs ;
- tests d'intégration avec PostgreSQL réel ;
- tests d'API ;
- tests d'autorisation ;
- tests de confidentialité ;
- tests d'idempotence ;
- tests de reprise après erreur ;
- analyse statique PHP ;
- formatage ;
- tests frontend ;
- tests de contrats Python ;
- tests de restauration des sauvegardes.

SQLite seul ne suffit pas pour valider les comportements PostgreSQL spécifiques.

Les critères de sécurité et de confidentialité non négociables doivent être traités comme des critères bloquants, pas comme des améliorations facultatives.

## 10. Risques et points à vérifier

Identifie notamment :

- compatibilité des versions Laravel/PHP/packages ;
- faisabilité de pgvector dans l'environnement retenu ;
- compatibilité des modèles d'embeddings ;
- qualité réelle des modèles Ollama sur les usages TCC ;
- consommation mémoire et concurrence ;
- temps de génération et streaming ;
- sécurité des communications Laravel/Python ;
- gestion des interruptions ;
- cohérence des sauvegardes et restaurations ;
- complexité de l'indexation documentaire.

Distingue clairement :
- les décisions déjà prises ;
- les vérifications techniques à effectuer ;
- les décisions encore ouvertes ;
- les risques nécessitant un prototype.

Ne transforme pas une hypothèse technique en décision validée.

## 11. Livrables attendus

Produis les documents suivants :

### `ANALYSE-COHERENCE.md`

- inventaire des documents lus ;
- décisions structurantes identifiées ;
- contradictions détectées ;
- éléments manquants ;
- risques et hypothèses ;
- vérifications techniques nécessaires.

### `PLAN-IMPLEMENTATION.md`

- phases de développement ;
- ordre des modules ;
- tâches détaillées ;
- dépendances ;
- tests ;
- critères d'acceptation ;
- points de contrôle.

### `TACHES-COMPOSER.md`

- liste ordonnée des tâches prêtes à être confiées à Composer 2 ;
- instructions autonomes ;
- périmètre strict de chaque tâche ;
- critères de validation.

### `DECISIONS-RESTANTES.md`

Uniquement si de véritables décisions restent à prendre. Ne crée pas artificiellement des questions pour remplir ce document.

## 12. Règles finales

- Ne code pas.
- Ne lance pas l'implémentation.
- Ne réécris pas toute la conception.
- Ne supprime aucune exigence validée.
- Ne simplifie pas les règles de confidentialité pour gagner du temps.
- Ne transforme pas le monolithe modulaire en microservices.
- Ne crée pas de dépendances inutiles.
- Ne prévois pas une architecture surdimensionnée pour le MVP.
- Ne demande pas de validation à chaque décision technique mineure.
- Signale les incertitudes plutôt que de les masquer.

Le résultat doit être un plan réaliste, progressif, vérifiable et directement exploitable dans Cursor avec Composer 2.
