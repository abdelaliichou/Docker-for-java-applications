# Infrastructure as Code (IaC) dans le projet Tenant

## Vue d'ensemble

Le projet **tenant-backend** applique le principe d'Infrastructure as Code (IaC) : toute l'infrastructure nécessaire au fonctionnement de l'application est décrite sous forme de fichiers versionnés dans le dépôt Git. Cela garantit la reproductibilité, la traçabilité et l'automatisation complète du cycle de vie de l'application.

## Les composants IaC du projet

### 1. Base de données — Liquibase

**Emplacement :** `infrastructure/src/main/resources/db/`

```
db/
├── init/init_database.sql          # Création des rôles et utilisateurs PostgreSQL
├── init-test/init_devservices_database.sql  # Idem pour les tests
└── changelog/
    ├── db-changelog-master.xml     # Fichier maître Liquibase
    └── changes/                    # Migrations SQL versionnées (OMOD-xxx.sql, PT-xxx.sql)
```

**Fonctionnement :**
- `init_database.sql` : crée les utilisateurs PostgreSQL (admin, applicatif, liquibase, readonly) avec leurs rôles et permissions. Exécuté au premier démarrage du conteneur.
- `db-changelog-master.xml` : orchestre l'ordre d'exécution des migrations.
- `changes/*.sql` : chaque fichier correspond à un ticket JIRA et contient les modifications de schéma (CREATE TABLE, ALTER, INSERT). Liquibase les exécute séquentiellement et ne rejoue jamais une migration déjà appliquée.

**Résultat :** La base de données est identique sur tous les environnements (local, dev, QUA, prod). Aucune modification manuelle n'est nécessaire.

---

### 2. Authentification — Keycloak (Realm JSON)

**Emplacement :** `.dev-env/keycloak/realms/`

```
realms/
├── appusers.json              # Realm principal (utilisateurs applicatifs)
├── awsusers.json              # Realm plateforme AWS
├── o2s.json                   # Realm produit O2S
└── an-autoprovisioning-realm.json  # Realm auto-provisioning
```

**Fonctionnement :**
- Ces fichiers JSON décrivent intégralement la configuration Keycloak : clients, rôles, utilisateurs, flows d'authentification, mappers de tokens.
- Au démarrage local (`docker-compose up`), Keycloak importe automatiquement ces realms via `--import-realm`.
- En production, les realms sont gérés par l'équipe plateforme mais suivent la même structure.

**Résultat :** Un développeur qui clone le projet dispose immédiatement d'un serveur d'identité fonctionnel avec des utilisateurs de test (alice/alice, axel/axel, laure/laure).

---

### 3. Environnement local — Docker Compose

**Emplacement :** `docker-compose.dev-env.yml`

```yaml
services:
  postgres:    # Base de données PostgreSQL 17
  keycloak:    # Serveur d'identité Keycloak
```

**Fonctionnement :**
- Définit les services nécessaires au développement local (PostgreSQL + Keycloak).
- Monte les scripts d'initialisation SQL et les realms Keycloak en volumes.
- Configure les healthchecks pour garantir que les services sont prêts.

**Résultat :** `docker-compose up` suffit pour avoir un environnement complet.

---

### 4. Build natif — Profil Maven + Dockerfile

**Emplacement :** `pom.xml` (profil `native`) + `application/api/api-core/docker/Dockerfile.native-micro`

**Fonctionnement :**
- Le profil Maven `-Pnative` compile l'application en binaire natif via GraalVM/Mandrel.
- Le `Dockerfile.native-micro` package ce binaire dans une image Docker minimale (basée sur `quarkus-micro`).
- L'image finale ne contient que le binaire + les librairies système minimales (~180 MB au lieu de ~500 MB avec JVM).

---

### 5. CI/CD — GitLab CI + ArgoCD

**Emplacement :** `.gitlab-ci.yml` + `.gitlab-ci/variables.yml`

**Pipeline :**

```
Code Push → Unit Test → Compile Native → Docker Build → Push AWS ECR → Deploy via ArgoCD
```

**Environnements déployés automatiquement :**

| Environnement | URL | Déclencheur |
|---|---|---|
| Dev | tenant-api-k8s.aws-dev.harvest.fr | Push sur branche par défaut |
| QUA | tenant-api-k8s.aws-qua.harvest.fr | Branche protégée |
| UAT | tenant-api-uat.harvest.fr | Tag |
| Staging | tenant-api-stg.harvest.fr | Tag |
| Production | tenant-api.harvest.fr | Tag (avec approbation) |

**Déploiement GitOps (ArgoCD) :**
- Le CI ne déploie pas directement sur Kubernetes.
- Il met à jour un dépôt Git séparé (`tenant-argocd`) avec la nouvelle version de l'image.
- ArgoCD surveille ce dépôt et synchronise automatiquement le cluster Kubernetes.

**Applications déployées :**
- `tenant-api` — API REST principale
- `tenant-broker` — Listener Kafka (événements)
- `tenant-cron` — Tâches planifiées

---

## Schéma global

```
┌─────────────────────────────────────────────────────────────────┐
│                        Dépôt Git (tenant-backend)               │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  db/changelog/*.sql        → Schéma BDD (Liquibase)            │
│  db/init/*.sql             → Utilisateurs PostgreSQL           │
│  .dev-env/keycloak/*.json  → Configuration Keycloak            │
│  docker-compose.dev-env.yml → Environnement local              │
│  pom.xml (native profile)  → Build natif GraalVM              │
│  docker/Dockerfile.*       → Image de production               │
│  .gitlab-ci.yml            → Pipeline CI/CD                    │
│                                                                 │
└──────────────────────────────┬──────────────────────────────────┘
                               │
                               ▼
┌──────────────────────────────────────────────────────────────────┐
│                     GitLab CI Pipeline                            │
│  Test → Compile Native → Docker Build → Push ECR → Update ArgoCD │
└──────────────────────────────┬───────────────────────────────────┘
                               │
                               ▼
┌──────────────────────────────────────────────────────────────────┐
│                     ArgoCD (GitOps)                               │
│  Surveille tenant-argocd → Synchronise Kubernetes                │
└──────────────────────────────────────────────────────────────────┘
```

## Avantages de cette approche

1. **Reproductibilité** — Tout environnement peut être recréé à l'identique depuis le code source.
2. **Traçabilité** — Chaque changement d'infrastructure est un commit Git avec auteur, date et raison (ticket JIRA).
3. **Autonomie développeur** — Un `git clone` + `docker-compose up` suffit pour travailler.
4. **Déploiement automatisé** — Aucune intervention manuelle entre le code et la production.
5. **Cohérence** — Les mêmes migrations SQL s'exécutent en local, en dev, et en prod.
6. **Revue de code** — Les changements d'infrastructure passent par les mêmes merge requests que le code applicatif.
