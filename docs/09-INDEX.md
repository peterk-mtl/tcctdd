# 09 — Index maître du dossier de conception

**Version :** 2.0  
**Statut :** Référentiel de navigation et de cohérence  
**Objectif :** Centraliser les documents à transmettre avant la génération du plan d'implémentation pour Composer 2.

## 1. Organisation

Les documents de conception vivent dans `docs/` :

```text
docs/
├── 00-REGISTRE-DECISIONS.md
├── 01-VISION-PRODUIT.md
├── 02-CAHIER-DES-CHARGES.md
├── 03-ARCHITECTURE-DDD.md
├── 04-BOUNDED-CONTEXTS.md
├── 05-CATALOGUE-CAS-UTILISATION.md
├── 06-DICTIONNAIRE-DONNEES-POSTGRESQL.md
├── 07-CONTRATS-INTERSERVICES.md
├── 08-BRIEF-PLANIFICATION-CLAUDE.md
├── 09-INDEX.md
└── 10-PROTOCOLE-SECURITE.md   # à venir, phase 7
```

## 2. Rôle de chaque document

| Document | Contenu | Autorité |
|---|---|---|
| 00 — Registre des décisions | Décisions D/H/P/B, roadmap, banc 1b, décisions ouvertes | Référence des arbitrages |
| 01 — Vision produit | Objectifs, MVP, fonctionnalités, exclusions, expérience utilisateur | Référence produit |
| 02 — Cahier des charges | Exigences fonctionnelles et techniques détaillées | Référence d'exigences |
| 03 — Architecture DDD | Principes, couches, dépendances, conventions | Référence architecturale |
| 04 — Bounded contexts | Responsabilités et frontières des sept contextes | Référence de découpage métier |
| 05 — Cas d'utilisation | Capacités observables, scénarios, règles | Référence applicative |
| 06 — Dictionnaire PostgreSQL | Schémas, tables, relations, contraintes et index | Référence de persistance |
| 07 — Contrats interservices | Contrats synchrones, événements, Outbox, API Laravel/Python | Référence d'intégration |
| 08 — Brief de planification | Méthode d'analyse et livrables destinés à Composer 2 | Consigne de planification |
| 09 — Index | Navigation, priorités, checklist, workflow | Référentiel de navigation |
| 10 — Protocole de sécurité | Protocoles d'urgence (à venir, phase 7) | Référence de sécurité |

## 3. Décisions produit non négociables

Toute planification doit retrouver et respecter notamment :

- TCCTDD est une application personnelle d'accompagnement TCC, pas un dispositif médical.
- Le MVP est fonctionnel et cohérent, sans chercher à intégrer toutes les évolutions futures.
- Le cadre méthodologique est exclusivement la TCC.
- Le compagnon conserve une posture bienveillante, sans jugement ni infantilisation.
- L'utilisateur conserve le contrôle de ses mémoires et de ses données.
- Une information saisie directement est `ACTIVE` + `DECLARED` ; une information extraite par l'IA est toujours `PROPOSED` jusqu'à validation.
- Les contradictions entre mémoires sont représentées par une relation `CONTRADICTS`, sans arbitrage automatique.
- Le journal est protégé par une politique d'accès IA configurable (`DENIED` par défaut).
- La consultation IA d'un journal n'autorise pas automatiquement l'extraction de mémoire.
- Les bilans de progression sont déclenchés à la demande, révisables puis immuables après validation.
- La KnowledgeBase est administrée ; l'utilisateur final n'y importe pas de documents personnels.
- Aucun OCR ni analyse d'image dans le MVP.
- Aucun contenu personnel dans les logs techniques.
- Les fonctionnalités non IA restent utilisables sans Ollama.
- Les sauvegardes sont externes ; une copie sur le même SSD n'est pas une sauvegarde.

Le détail et les renvois figurent dans `00-REGISTRE-DECISIONS.md`.

## 4. Décisions architecturales non négociables

- Monolithe modulaire Laravel, pas sept microservices.
- Sept bounded contexts métier.
- Python est un service technique spécialisé dans `ai-engine/`.
- Laravel reste propriétaire des données métier et des autorisations.
- Le service Python n'a aucun droit sur les schémas métier.
- PostgreSQL avec pgvector et recherche plein texte.
- Pas de Redis au MVP.
- Les modules ne modifient jamais directement les tables métier des autres modules.
- Les dépendances entre modules passent uniquement par `Contracts/`.
- Les événements utilisent une Outbox transactionnelle ; les consommateurs sont idempotents.
- Les transactions restent locales et courtes ; pas de transaction distribuée.
- Identifiants UUID v7 ; dates en UTC.
- DTO natifs `final readonly` ; aucune déclaration `strict_types`.
- SSE relayé par Laravel ; une seule génération à la fois.
- Les migrations et tests doivent tenir compte des spécificités PostgreSQL.

## 5. Règles de priorité en cas de contradiction

Appliquer l'ordre suivant :

1. Décision explicite et validée par l'utilisateur (registre `00`).
2. Document consolidé le plus récent portant sur le sujet.
3. Document de conception initial.
4. Hypothèse technique ou proposition non validée.

Une décision explicite ne peut pas être remplacée par une préférence technique.

Toute contradiction non résoluble doit être inscrite dans `ANALYSE-COHERENCE.md`.

## 6. Vérifications avant transmission

- [ ] Tous les documents 00 à 09 existent sous forme de fichiers.
- [ ] Les documents sont complets et non tronqués.
- [ ] Les versions consolidées remplacent les brouillons contradictoires.
- [ ] Les décisions produit et techniques sont présentes dans le registre.
- [ ] Les contrats interservices (07) sont inclus.
- [ ] Le brief de planification (08) est inclus.
- [ ] Aucun secret, mot de passe, jeton ou donnée personnelle réelle n'est présent.
- [ ] Le document 10 est référencé comme à venir s'il n'existe pas encore.

## 7. Workflow prévu

1. Conserver les documents à jour dans `docs/`.
2. Ouvrir le projet dans Cursor.
3. Demander la lecture de l'ensemble du dossier selon le brief `08`.
4. Produire `ANALYSE-COHERENCE.md`, `PLAN-IMPLEMENTATION.md` et `TACHES-COMPOSER.md`.
5. Examiner les éventuelles contradictions ou décisions réellement ouvertes.
6. Faire exécuter les tâches une par une par Composer 2.
7. Vérifier les tests et critères d'acceptation après chaque étape.

## 8. Règle de maintenance

Les documents de conception restent la référence du projet.

Toute évolution importante doit être répercutée dans les documents concernés. Le plan d'implémentation ne doit jamais devenir une source de vérité contradictoire avec la conception validée.

Les décisions nouvelles doivent être consignées dans `00-REGISTRE-DECISIONS.md` avec :

- date ;
- décision ;
- justification ;
- documents affectés ;
- conséquences éventuelles sur les tâches déjà planifiées.

## 9. Migrations documentaires

Les fichiers suivants ont été remplacés lors de la Phase 0 et ne sont plus des documents courants :

- `05-BRIEF-CLAUDE.md` → contenu migré vers `08-BRIEF-PLANIFICATION-CLAUDE.md` ;
- `06-INDEX.md` → contenu migré vers `09-INDEX.md` ;
- `04-CATALOGUE-CAS-UTILISATION.md` (ancien) → contenu utile migré vers `05-CATALOGUE-CAS-UTILISATION.md` et `07-CONTRATS-INTERSERVICES.md`.

---

**Statut :** index maître — version de travail à valider.

**Dépendances :** l'ensemble des documents `00` à `08`.

**Changements v2 :** migration depuis `06-INDEX.md` ; hiérarchie 00 à 09 ; 10 référencé comme futur ; décisions et stack alignées sur la Phase 0 ; workflow Claude → Composer 2 actualisé (Phase 0).
