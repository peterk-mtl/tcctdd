# 08 — Brief de planification pour Claude

## 1. Rôle et objectif

Tu interviens comme architecte logiciel et planificateur technique du projet TCCTDD.

Ton rôle est d'analyser le dossier de conception existant et de produire un plan d'implémentation détaillé, destiné à être exécuté ensuite par Composer 2 dans Cursor.

**Tu ne dois pas implémenter le projet.** Tu ne dois pas générer les classes, les migrations, les composants frontend ou les fichiers applicatifs. Cette étape sera réalisée séparément par Composer 2.

Ton livrable doit transformer la conception fonctionnelle et technique en étapes de développement concrètes, vérifiables et ordonnées.

## 2. Lecture et analyse préalables

Avant de rédiger le plan :

1. Parcours tous les documents du dossier de conception (`docs/00` à `docs/09`, et les renvois vers `10-PROTOCOLE-SECURITE.md` lorsque ce document existera).
2. Identifie leur sujet, leur version et leur statut.
3. Repère les décisions métier, techniques, architecturales et de sécurité, notamment dans `00-REGISTRE-DECISIONS.md`.
4. Identifie les dépendances entre les modules.
5. Détecte les contradictions, doublons et informations manquantes.
6. Vérifie que les choix décrits sont cohérents entre la vision, le cahier des charges, l'architecture DDD, les bounded contexts, le dictionnaire de données, les cas d'utilisation et les contrats interservices.

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

Évite de lui redemander des choix déjà tranchés dans `00-REGISTRE-DECISIONS.md`.

## 4. Architecture à respecter

Le projet repose sur :

- Laravel pour le cœur métier, à la racine du dépôt ;
- une architecture DDD organisée en monolithe modulaire ;
- sept bounded contexts clairement séparés ;
- Python dans `ai-engine/` pour les traitements documentaires et l'orchestration IA ;
- contrats versionnés dans `contracts/` (créés en phase 1) ;
- Ollama pour l'inférence locale ;
- PostgreSQL avec pgvector et recherche plein texte ;
- Vue 3, TypeScript, Inertia et Tailwind ;
- Filament pour l'administration ;
- Docker Compose pour l'environnement ;
- Pest pour les tests PHP ;
- PHP 8.4, sans aucune déclaration `strict_types` ;
- DTO natifs `final readonly` (pas de `spatie/laravel-data` par défaut) ;
- UUID v7 via `ramsey/uuid`, manipulés comme `string` ;
- pas de Redis au MVP : files, cache et sessions dans PostgreSQL.

Le service Python ne possède aucune donnée métier Laravel et n'a aucun droit sur les schémas métier.

Les contextes ne doivent pas accéder directement aux tables métier des autres contextes. Les dépendances passent uniquement par `Contracts/`.

Le domaine ne doit dépendre ni de Laravel, ni d'Eloquent, ni de Python.

Les règles validées incluent notamment :

- événements via Outbox ;
- idempotence des consommateurs ;
- autorisations contrôlées par Laravel (rôles `ADMIN` et `USER`, Policies) ;
- absence de contenu personnel dans les logs techniques ;
- SSE relayé par Laravel : le navigateur ne communique jamais avec Python ;
- une seule génération IA à la fois ; conversations interactives jamais préemptées ; traitements de fond préemptables.

Les versions exactes des frameworks et packages doivent être vérifiées au moment de la planification, sans supposer une compatibilité non vérifiée.

## 5. Contraintes produit et infrastructure

Le MVP doit être complet et cohérent, mais ne doit pas implémenter prématurément toutes les évolutions possibles.

Contraintes importantes :

- application local-first, accessible en `localhost` uniquement pour le MVP ;
- fonctionnement sans service externe obligatoire ;
- IA locale via Ollama ;
- machine de référence : ThinkPad T480, i5-8350U (4 cœurs / 8 threads), 32 Go de RAM, SSD 256 Go, sans GPU dédié ; les contraintes principales sont le temps CPU et l'espace disque ;
- choix des modèles fondé sur le banc de la phase 1b, avant toute sélection définitive ;
- `OLLAMA_MAX_LOADED_MODELS=1` au départ ;
- les fonctionnalités non IA doivent rester utilisables si Ollama ou Python est indisponible ;
- pas de fallback automatique vers un autre modèle ;
- pas de synchronisation cloud automatique ;
- pas de téléversement documentaire par l'utilisateur final dans la KnowledgeBase ;
- pas d'OCR ni d'analyse d'images dans le MVP ;
- bibliothèque documentaire administrée et validée ;
- sauvegardes sur support externe, chiffrées, clé conservée séparément ;
- sécurité et confidentialité prioritaires ;
- aucune fonctionnalité clinique ou médicale revendiquée ;
- accessibilité WCAG 2.2 AA pour les parcours essentiels.

Le plan doit tenir compte des contraintes réelles de ressources et éviter de multiplier les modèles chargés simultanément.

## 6. Construction du plan

Produis un plan progressif, structuré en phases et en tâches suffisamment petites pour être confiées à Composer 2.

Respecte la roadmap de `00-REGISTRE-DECISIONS.md` (phases 0 à 14, dont 1b).

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

Propose un ordre qui respecte les dépendances réelles et la roadmap du registre.

Le premier objectif est une tranche verticale fonctionnelle sans IA :

1. authentification ;
2. consultation du catalogue d'exercices ;
3. démarrage d'une réalisation ;
4. sauvegarde automatique des réponses ;
5. reprise d'une réalisation ;
6. finalisation ;
7. consultation de l'historique.

L'IA, le RAG et les fonctionnalités de mémoire seront intégrés progressivement, après stabilisation des fondations métier.

Le protocole de sécurité (`10-PROTOCOLE-SECURITE.md`) doit être défini et testé avant Companion.

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
- tests d'architecture (dépendances entre modules via `Contracts/` uniquement) ;
- analyse statique PHP ;
- formatage ;
- tests frontend ;
- tests de contrats Python (fixtures de `contracts/v1/fixtures/` vérifiées des deux côtés) ;
- tests de restauration des sauvegardes et de rejeu du registre des suppressions.

SQLite seul ne suffit pas pour valider les comportements PostgreSQL spécifiques.

Les critères de sécurité et de confidentialité non négociables doivent être traités comme des critères bloquants, pas comme des améliorations facultatives.

## 10. Risques et points à vérifier

Identifie notamment :

- compatibilité des versions Laravel/PHP/packages ;
- faisabilité de pgvector dans l'environnement retenu ;
- compatibilité des modèles d'embeddings ;
- qualité réelle des modèles Ollama sur les usages TCC (banc 1b) ;
- consommation mémoire et concurrence ;
- temps de génération et streaming ;
- sécurité des communications Laravel/Python ;
- gestion des interruptions, de la préemption et des déconnexions ;
- cohérence des sauvegardes et restaurations ;
- complexité de l'indexation documentaire.

Distingue clairement :

- les décisions déjà prises ;
- les vérifications techniques à effectuer ;
- les décisions encore ouvertes (voir `00-REGISTRE-DECISIONS.md`) ;
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

---

**Statut :** brief de planification — version de travail à valider.

**Dépendances :** `00-REGISTRE-DECISIONS.md`, `01-VISION-PRODUIT.md`, `02-CAHIER-DES-CHARGES.md`, `03-ARCHITECTURE-DDD.md`, `04-BOUNDED-CONTEXTS.md`, `05-CATALOGUE-CAS-UTILISATION.md`, `06-DICTIONNAIRE-DONNEES-POSTGRESQL.md`, `07-CONTRATS-INTERSERVICES.md` et `09-INDEX.md`.

**Documents liés :** `10-PROTOCOLE-SECURITE.md` (à venir, phase 7).

**Changements v2 :** migration depuis `05-BRIEF-CLAUDE.md` ; stack et contraintes alignées sur la Phase 0 (32 Go, DTO natifs, UUID v7, pas de Redis, SSE relayé, une génération à la fois, banc 1b, rôles `ADMIN`/`USER`) ; renvois vers 00 à 09 et 10 futur (Phase 0).
