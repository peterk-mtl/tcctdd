# TCCTDD — Index maître du dossier de conception

**Version :** 1.0  
**Statut :** Référentiel de navigation et de cohérence  
**Objectif :** Centraliser les documents à transmettre à Claude avant la génération du plan d'implémentation pour Composer 2.

## 1. Organisation recommandée

Créer un dossier `docs/conception/` dans le projet, avec cette organisation :

```text
docs/
└── conception/
    ├── 00-INDEX-MAITRE.md
    ├── 01-VISION-ET-CAHIER-DES-CHARGES.md
    ├── 02-ARCHITECTURE-DDD.md
    ├── 03-BOUNDED-CONTEXTS.md
    ├── 04-DICTIONNAIRE-DONNEES-POSTGRESQL.md
    ├── 05-CATALOGUE-CAS-UTILISATION.md
    ├── 06-CONTRATS-INTERCONTEXTES-EVENEMENTS.md
    ├── 07-CONTRATS-API-ET-ECHANGES-PYTHON.md
    ├── 08-PLAN-IMPLEMENTATION-PHASES.md
    ├── 09-BRIEF-PLANIFICATION-CLAUDE.md
    └── annexes/
        ├── DECISIONS-TECHNIQUES.md
        ├── DECISIONS-PRODUIT.md
        └── RISQUES-ET-VERIFICATIONS.md
```

Les noms sont des noms de fichiers recommandés. Ils ne supposent pas que les documents correspondants ont déjà été exportés sous ces noms.

## 2. Rôle de chaque document

| Document | Contenu | Autorité |
|---|---|---|
| 01 — Vision et cahier des charges | Objectifs, MVP, fonctionnalités, exclusions, expérience utilisateur | Référence produit |
| 02 — Architecture DDD | Principes, couches, dépendances, conventions | Référence architecturale |
| 03 — Bounded Contexts | Responsabilités et frontières des sept contextes | Référence de découpage métier |
| 04 — Dictionnaire PostgreSQL | Schémas, tables, relations, contraintes et index | Référence de persistance |
| 05 — Cas d'utilisation | Commandes, requêtes, flux, invariants et résultats | Référence applicative |
| 06 — Contrats intercontextes | Interfaces, événements, Outbox, idempotence | Référence de collaboration |
| 07 — Contrats API | Endpoints, streaming, échanges Laravel/Python, erreurs | Référence d'intégration |
| 08 — Plan d'implémentation | Phases, dépendances et ordre de développement | Référence de séquencement |
| 09 — Brief Claude | Méthode d'analyse et livrables attendus | Consigne de planification |

## 3. Décisions produit non négociables

Claude doit retrouver et respecter notamment les décisions suivantes :

- TCCTDD est une application personnelle d'accompagnement TCC, pas un dispositif médical.
- Le MVP est fonctionnel et cohérent, sans chercher à intégrer toutes les évolutions futures.
- Le framework méthodologique est exclusivement la TCC.
- Le compagnon conserve une posture bienveillante, sans jugement ni infantilisation.
- L'utilisateur conserve le contrôle de ses mémoires et de ses données.
- Les faits explicitement déclarés peuvent être mémorisés ; les interprétations IA doivent être proposées puis validées.
- Les contradictions entre mémoires sont conservées et signalées, pas arbitrées automatiquement.
- Les réponses de journal sont protégées par une politique d'accès IA configurable.
- La consultation IA d'un journal n'autorise pas automatiquement l'extraction de mémoire.
- Les bilans de progression sont déclenchés à la demande et validés par l'utilisateur.
- La KnowledgeBase est administrée ; l'utilisateur final ne téléverse pas de documents dans celle-ci.
- Les documents documentaires sont validés avant publication et indexation.
- Aucun OCR ni analyse d'image dans le MVP.
- Aucun contenu personnel dans les logs techniques.
- Les fonctionnalités non IA restent utilisables sans Ollama.

## 4. Décisions architecturales non négociables

- Monolithe modulaire Laravel, pas sept microservices.
- Sept bounded contexts métier.
- Python est un service technique spécialisé.
- Laravel reste propriétaire des données métier et des autorisations.
- PostgreSQL avec pgvector et recherche full-text.
- Les modules ne modifient jamais directement les tables métier des autres modules.
- Les événements utilisent une Outbox transactionnelle.
- Les consommateurs sont idempotents.
- Les transactions restent locales et courtes.
- Pas de transaction distribuée.
- Les identifiants sont des UUID.
- Les dates sont stockées en UTC.
- Les entités du domaine ne dépendent pas d'Eloquent.
- Les DTO utilisent spatie/laravel-data.
- Ne pas utiliser `strict_types`.
- Les migrations et tests doivent tenir compte des spécificités PostgreSQL.

## 5. Règles de priorité en cas de contradiction

Claude doit appliquer l'ordre suivant :

1. Décision explicite et validée par l'utilisateur.
2. Document consolidé le plus récent portant sur le sujet.
3. Document de conception initial.
4. Hypothèse technique ou proposition non validée.

Une décision explicite ne peut pas être remplacée par une préférence technique de Claude.

Toute contradiction non résoluble doit être inscrite dans `ANALYSE-COHERENCE.md`.

## 6. Vérifications avant transmission à Claude

- [ ] Tous les documents existent sous forme de fichiers.
- [ ] Les documents sont complets et non tronqués.
- [ ] Les versions consolidées remplacent les brouillons contradictoires.
- [ ] Les décisions produit et techniques sont présentes.
- [ ] Les contrats API et intercontextes sont inclus.
- [ ] Le brief de planification Claude est inclus.
- [ ] Aucun secret, mot de passe, jeton ou donnée personnelle réelle n'est présent.
- [ ] Les documents sont accessibles depuis le dossier transmis à Claude.

## 7. Workflow prévu

1. Enregistrer les documents dans `docs/conception/`.
2. Ouvrir le projet dans Cursor.
3. Demander à Claude de lire l'ensemble du dossier.
4. Lui faire produire `ANALYSE-COHERENCE.md`, `PLAN-IMPLEMENTATION.md` et `TACHES-COMPOSER.md`.
5. Examiner les éventuelles contradictions ou décisions réellement ouvertes.
6. Faire exécuter les tâches une par une par Composer 2.
7. Vérifier les tests et critères d'acceptation après chaque étape.

## 8. Règle de maintenance

Les documents de conception restent la référence du projet.

Toute évolution importante doit être répercutée dans les documents concernés. Le plan d'implémentation ne doit jamais devenir une source de vérité contradictoire avec la conception validée.

Les décisions nouvelles doivent être consignées avec :
- date ;
- décision ;
- justification ;
- documents affectés ;
- conséquences éventuelles sur les tâches déjà planifiées.

