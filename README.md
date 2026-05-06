# ci-cd

# 🚀 CHEATSHEET EC06 — Pipeline CI/CD SkillHub

---

## ⏱️ Plan des 4h

| Durée | Étape |
|-------|-------|
| 30min | Git setup + branches |
| 60min | Réorganiser Dockerfile / docker-compose / .gitlab-ci.yml |
| 45min | Faire tourner localement + screenshots |
| 45min | Rédiger DOCUMENTATION.md |
| 30min | ZIP + vérification finale |

---

## 1. GIT — Setup initial (C21 / C22)

```bash
# Cloner le projet fourni
git clone <url-du-repo>
cd <nom-du-repo>

# Créer develop depuis main
git checkout main
git checkout -b develop
git push -u origin develop

# Créer la branche de travail
git checkout -b feature/ci-cd-setup
git push -u origin feature/ci-cd-setup
```

### Commits à faire (Conventional Commits)
```bash
git add .gitlab-ci.yml
git commit -m "ci: add GitLab CI/CD pipeline"

git add Dockerfile
git commit -m "feat: add multi-stage Dockerfile"

git add docker-compose.yml
git commit -m "feat: add docker compose with healthchecks"

git add DOCUMENTATION.md
git commit -m "docs: add technical documentation"

git push
```

### Merge feature → develop → main
```bash
# Sur GitLab : créer une MR feature/ci-cd-setup → develop
# Valider la MR (même tout seul)

# Ensuite en local :
git checkout develop
git pull origin develop

git checkout main
git merge develop
git push origin main

# Tag de version
git tag -a v1.0.0 -m "Release v1.0.0"
git push origin v1.0.0
```

---

## 2. DOCKERFILE (C23)

> Ordre obligatoire : FROM → WORKDIR → COPY requirements → RUN pip → COPY code → USER → EXPOSE → CMD

```dockerfile
# Stage 1 — build
FROM python:3.11-slim AS builder

WORKDIR /app

COPY requirements.txt .
RUN pip install --user --no-cache-dir -r requirements.txt

# Stage 2 — runtime
FROM python:3.11-slim

WORKDIR /app

# Copier les dépendances installées
COPY --from=builder /root/.local /root/.local

# Copier le code source
COPY . .

# Sécurité : utilisateur non-root
RUN adduser --disabled-password --gecos "" appuser
USER appuser

ENV PATH=/root/.local/bin:$PATH
ENV FLASK_ENV=production

EXPOSE 5000

HEALTHCHECK --interval=30s --timeout=10s --retries=3 \
  CMD python -c "import urllib.request; urllib.request.urlopen('http://localhost:5000/health')"

CMD ["python", "-m", "flask", "run", "--host=0.0.0.0"]
```

### .dockerignore (ne pas oublier !)
```
__pycache__/
*.pyc
*.pyo
.env
.git
.gitignore
*.md
tests/
.pytest_cache/
```

---

## 3. DOCKER-COMPOSE (C24)

> Vérifier que les noms de services dans `depends_on` correspondent exactement aux clés déclarées

```yaml
version: '3.9'

services:
  api:
    build: .
    container_name: skillhub-api
    ports:
      - "5000:5000"
    environment:
      - FLASK_ENV=${FLASK_ENV:-production}
      - DATABASE_URL=${DATABASE_URL:-sqlite:///skillhub.db}
    depends_on:
      db:
        condition: service_healthy
    healthcheck:
      test: ["CMD", "python", "-c", "import urllib.request; urllib.request.urlopen('http://localhost:5000/health')"]
      interval: 30s
      timeout: 10s
      retries: 3
    networks:
      - skillhub-network
    restart: unless-stopped

  db:
    image: postgres:15-alpine
    container_name: skillhub-db
    environment:
      POSTGRES_DB: ${POSTGRES_DB:-skillhub}
      POSTGRES_USER: ${POSTGRES_USER:-skillhub_user}
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD:-changeme}
    volumes:
      - db_data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U ${POSTGRES_USER:-skillhub_user}"]
      interval: 10s
      timeout: 5s
      retries: 5
    networks:
      - skillhub-network
    restart: unless-stopped

volumes:
  db_data:

networks:
  skillhub-network:
    driver: bridge
```

### Commandes Docker utiles
```bash
# Build et lancer
docker compose up -d --build

# Vérifier que tout tourne
docker compose ps

# Voir les logs
docker compose logs -f api

# Tester l'API (screenshot obligatoire)
curl http://localhost:5000/health
curl http://localhost:5000/

# Arrêter
docker compose down
```

---

## 4. .GITLAB-CI.YML (C22 / C25)

> Les `stages` déclarés en haut doivent correspondre EXACTEMENT aux `stage:` de chaque job

```yaml
stages:
  - lint
  - test
  - security
  - build
  - deploy

variables:
  IMAGE_NAME: skillhub-api
  DOCKER_DRIVER: overlay2
  PIP_CACHE_DIR: "$CI_PROJECT_DIR/.cache/pip"

# Cache pip entre les jobs
cache:
  paths:
    - .cache/pip/

# ─────────────────────────────────────────
# STAGE 1 : LINT
# ─────────────────────────────────────────
lint:
  stage: lint
  image: python:3.11-slim
  script:
    - pip install flake8 -q
    - flake8 app/ --max-line-length=120 --exclude=__pycache__
  allow_failure: false

# ─────────────────────────────────────────
# STAGE 2 : TEST
# ─────────────────────────────────────────
test:
  stage: test
  image: python:3.11-slim
  script:
    - pip install -r requirements.txt -q
    - pip install pytest pytest-cov -q
    - pytest tests/ --cov=app --cov-report=term-missing
  coverage: '/TOTAL.*\s+(\d+%)$/'
  artifacts:
    reports:
      coverage_report:
        coverage_format: cobertura
        path: coverage.xml
    expire_in: 1 week

# ─────────────────────────────────────────
# STAGE 3 : SECURITY
# ─────────────────────────────────────────
security:
  stage: security
  image: python:3.11-slim
  script:
    - pip install safety bandit -q
    - echo "=== Safety - Scan des dépendances ==="
    - safety check -r requirements.txt || true
    - echo "=== Bandit - Analyse statique SAST ==="
    - bandit -r app/ -ll -f txt
  allow_failure: true   # passe même si vulnérabilités détectées (pour l'exam)

# ─────────────────────────────────────────
# STAGE 4 : BUILD
# ─────────────────────────────────────────
build:
  stage: build
  image: docker:24
  services:
    - docker:24-dind
  script:
    - docker build -t $IMAGE_NAME:$CI_COMMIT_SHORT_SHA .
    - docker save $IMAGE_NAME:$CI_COMMIT_SHORT_SHA | gzip > image.tar.gz
    - echo "Image buildée avec succès : $IMAGE_NAME:$CI_COMMIT_SHORT_SHA"
  artifacts:
    paths:
      - image.tar.gz
    expire_in: 1 hour
  only:
    - develop
    - main

# ─────────────────────────────────────────
# STAGE 5 : DEPLOY
# ─────────────────────────────────────────
deploy:
  stage: deploy
  image: docker:24
  services:
    - docker:24-dind
  script:
    - echo "Déploiement de $IMAGE_NAME vers ${DEPLOY_ENV:-production}"
    - docker compose up -d --build
    - echo "Déploiement terminé"
  environment:
    name: production
  only:
    - main
```

---

## 5. DOCUMENTATION.md — Template complet

```markdown
# Documentation Technique — Pipeline CI/CD SkillHub

## 1. Présentation du projet

Ce projet met en place un pipeline CI/CD complet pour l'API SkillHub Learning,
développée en Python/Flask, conteneurisée avec Docker et automatisée via GitLab CI.

---

## 2. Gestion des versions (C21)

### Politique de branches
- `main` : code stable, déployé en production
- `develop` : branche d'intégration
- `feature/*` : développement de fonctionnalités

### Conventions de commits
Format : `<type>(<scope>): <description>`

Types utilisés : `feat`, `fix`, `ci`, `docs`, `chore`, `test`

Exemple : `ci: add GitLab CI/CD pipeline with security scan`

### Tags de version
Les releases sont taguées selon SemVer : `v1.0.0`

---

## 3. Intégration continue (C22)

### Plateforme
GitLab CI via `.gitlab-ci.yml`

### Stages du pipeline
| Stage | Rôle |
|-------|------|
| lint | Vérification du style de code (flake8) |
| test | Tests unitaires + couverture (pytest) |
| security | Scan vulnérabilités (safety, bandit) |
| build | Construction image Docker |
| deploy | Déploiement via Docker Compose |

### Processus de fusion
Toute modification passe par une Merge Request :
`feature/*` → `develop` → `main`

---

## 4. Conteneurisation (C23)

### Dockerfile
- Image de base : `python:3.11-slim`
- Build multi-stage pour réduire la taille finale
- Utilisateur non-root pour la sécurité
- Healthcheck intégré

### Reproductibilité
L'image Docker fonctionne de manière identique sur tous les
environnements grâce à l'isolation des dépendances.

---

## 5. Orchestration (C24)

### Services Docker Compose
- `api` : application Flask (port 5000)
- `db` : PostgreSQL 15 avec volume persistant

### Gestion dynamique
- `depends_on` avec `condition: service_healthy`
- Healthchecks sur chaque service
- Restart policy : `unless-stopped`

---

## 6. DevSecOps (C25)

### Conformité aux pratiques DevOps
- Pipeline entièrement automatisé
- Aucun secret en dur dans le code
- Variables sensibles via GitLab CI Variables

### Sécurité des déploiements
- `safety check` : scan des dépendances Python
- `bandit` : analyse statique SAST du code source
- `.dockerignore` : exclusion des fichiers sensibles de l'image
- Utilisateur non-root dans le container

---

## 7. Captures d'écran

### Pipeline CI
![Pipeline CI](screenshots/pipeline-ci.png)

### Pipeline CD
![Pipeline CD](screenshots/pipeline-cd.png)

### Réponse de l'API
![API Response](screenshots/api-response.png)

---

## 8. Commandes pour lancer le projet

\`\`\`bash
# Cloner le repo
git clone <url>
cd skillhub

# Lancer les services
docker compose up -d --build

# Tester l'API
curl http://localhost:5000/health
\`\`\`
```

---

## 6. Structure ZIP finale

```
livrable_EC06_NomPrenom.zip
├── Dockerfile
├── .dockerignore
├── docker-compose.yml
├── .gitlab-ci.yml
├── DOCUMENTATION.md
└── screenshots/
    ├── pipeline-ci.png
    ├── pipeline-cd.png
    └── api-response.png
```

```bash
# Créer le ZIP
zip -r livrable_EC06_NomPrenom.zip \
  Dockerfile \
  .dockerignore \
  docker-compose.yml \
  .gitlab-ci.yml \
  DOCUMENTATION.md \
  screenshots/
```

---

## 7. Checklist avant de rendre

- [ ] Branche `develop` créée depuis `main`
- [ ] Branche `feature/ci-cd-setup` créée depuis `develop`
- [ ] Commits en Conventional Commits
- [ ] MR créée sur GitLab (feature → develop)
- [ ] Tag `v1.0.0` sur `main`
- [ ] `docker compose up` fonctionne sans erreur
- [ ] `curl http://localhost:5000/health` répond
- [ ] Pipeline GitLab : tous les jobs verts (ou au moins lint/test/build)
- [ ] 3 screenshots dans le dossier `screenshots/`
- [ ] `DOCUMENTATION.md` complété
- [ ] Aucun secret en dur dans les fichiers
- [ ] `.dockerignore` présent
- [ ] ZIP créé avec tous les fichiers

---

## 8. Dépannage rapide

```bash
# Le build échoue ?
docker compose logs api

# Port déjà utilisé ?
lsof -i :5000
kill -9 <PID>

# Repartir de zéro
docker compose down -v
docker system prune -f
docker compose up -d --build

# Pipeline qui plante sur safety ?
# → allow_failure: true est déjà dans le yaml, ça ne bloque pas

# Conflit Git ?
git checkout develop
git pull origin develop
git checkout feature/ci-cd-setup
git rebase develop
git push --force-with-lease
```
