🔝 Retour au [Sommaire](/SOMMAIRE.md)

# Annexe B : Exemples de Pipelines CI/CD

## Introduction au CI/CD

### Qu'est-ce que le CI/CD ?

**CI/CD** signifie **Continuous Integration / Continuous Deployment** (Intégration Continue / Déploiement Continu).

**CI (Continuous Integration)** :
- 🔍 Tester automatiquement le code à chaque commit
- 🔨 Compiler et builder l'application
- ✅ Vérifier la qualité du code
- 🐛 Détecter les bugs rapidement

**CD (Continuous Deployment)** :
- 📦 Créer des images Docker automatiquement
- 🚀 Déployer sur Kubernetes/MicroK8s
- 🔄 Mettre à jour les applications sans intervention manuelle
- 🎯 Livrer rapidement de nouvelles fonctionnalités

### Pourquoi utiliser le CI/CD ?

**Sans CI/CD** :
```bash
# Processus manuel, long et sujet aux erreurs
git pull
npm test
docker build -t mon-app:latest .
docker push mon-app:latest
kubectl set image deployment/mon-app app=mon-app:latest
# Oups, j'ai oublié de tester !
# Oups, mauvaise version déployée !
```

**Avec CI/CD** :
```bash
# Simple et automatique
git commit -m "Nouvelle fonctionnalité"
git push
# ✨ Le pipeline fait tout automatiquement !
```

### Pipeline CI/CD Typique

```
┌──────────────┐
│  Git Push    │
└──────┬───────┘
       │
       ▼
┌──────────────┐
│ 1. Tests     │  ← Exécuter les tests unitaires
└──────┬───────┘
       │
       ▼
┌──────────────┐
│ 2. Build     │  ← Compiler l'application
└──────┬───────┘
       │
       ▼
┌──────────────┐
│ 3. Docker    │  ← Créer l'image Docker
└──────┬───────┘
       │
       ▼
┌──────────────┐
│ 4. Push      │  ← Pousser vers le registry
└──────┬───────┘
       │
       ▼
┌──────────────┐
│ 5. Deploy    │  ← Déployer sur Kubernetes
└──────────────┘
```

---

## 1. Pipeline GitLab CI

GitLab CI est intégré directement dans GitLab et utilise des fichiers `.gitlab-ci.yml`.

### 1.1 Pipeline Simple - Application Node.js

```yaml
# .gitlab-ci.yml

# Définition des étapes du pipeline
stages:
  - test       # Étape 1: Tests
  - build      # Étape 2: Build
  - deploy     # Étape 3: Déploiement

# Variables globales
variables:
  # Nom de l'application
  APP_NAME: mon-app-nodejs
  # Version Docker
  DOCKER_VERSION: "20.10"
  # Registry Docker
  DOCKER_REGISTRY: registry.gitlab.com/mon-groupe/mon-projet
  # Tag de l'image basé sur le commit
  IMAGE_TAG: $CI_COMMIT_SHORT_SHA

# Template pour éviter la répétition
.docker_template: &docker_config
  image: docker:${DOCKER_VERSION}
  services:
    - docker:${DOCKER_VERSION}-dind
  before_script:
    - docker login -u $CI_REGISTRY_USER -p $CI_REGISTRY_PASSWORD $CI_REGISTRY

# ========================================
# ÉTAPE 1: TESTS
# ========================================

# Tests unitaires
test:unit:
  stage: test
  image: node:18-alpine
  script:
    # Installer les dépendances
    - npm ci
    # Exécuter les tests
    - npm run test
    # Générer le rapport de couverture
    - npm run test:coverage
  # Conserver les rapports de tests
  artifacts:
    reports:
      coverage_report:
        coverage_format: cobertura
        path: coverage/cobertura-coverage.xml
    paths:
      - coverage/
    expire_in: 1 week
  # Exécuter seulement sur les merge requests et la branche main
  only:
    - merge_requests
    - main

# Tests de linting (qualité du code)
test:lint:
  stage: test
  image: node:18-alpine
  script:
    - npm ci
    - npm run lint
  allow_failure: true  # Ne pas bloquer le pipeline si le linting échoue
  only:
    - merge_requests
    - main

# ========================================
# ÉTAPE 2: BUILD
# ========================================

# Build de l'image Docker
build:docker:
  <<: *docker_config
  stage: build
  script:
    # Afficher les informations
    - echo "Building Docker image for commit $CI_COMMIT_SHORT_SHA"

    # Builder l'image avec plusieurs tags
    - docker build
      --build-arg NODE_ENV=production
      --build-arg BUILD_DATE=$(date -u +"%Y-%m-%dT%H:%M:%SZ")
      --build-arg VCS_REF=$CI_COMMIT_SHORT_SHA
      -t $DOCKER_REGISTRY/$APP_NAME:$IMAGE_TAG
      -t $DOCKER_REGISTRY/$APP_NAME:latest
      .

    # Pousser l'image vers le registry
    - docker push $DOCKER_REGISTRY/$APP_NAME:$IMAGE_TAG
    - docker push $DOCKER_REGISTRY/$APP_NAME:latest

    # Afficher les informations de l'image
    - docker images | grep $APP_NAME

  # Exécuter seulement si les tests passent
  needs:
    - test:unit
  only:
    - main
    - tags

# ========================================
# ÉTAPE 3: DÉPLOIEMENT
# ========================================

# Déploiement sur l'environnement de développement
deploy:dev:
  stage: deploy
  image: alpine/k8s:1.28.0
  script:
    # Configurer kubectl avec les credentials
    - kubectl config set-cluster k8s --server="$KUBE_URL" --insecure-skip-tls-verify=true
    - kubectl config set-credentials gitlab --token="$KUBE_TOKEN"
    - kubectl config set-context default --cluster=k8s --user=gitlab
    - kubectl config use-context default

    # Créer le namespace s'il n'existe pas
    - kubectl create namespace dev --dry-run=client -o yaml | kubectl apply -f -

    # Mettre à jour l'image dans le déploiement
    - kubectl set image deployment/$APP_NAME
      $APP_NAME=$DOCKER_REGISTRY/$APP_NAME:$IMAGE_TAG
      -n dev

    # Attendre que le déploiement soit terminé
    - kubectl rollout status deployment/$APP_NAME -n dev --timeout=5m

    # Afficher les informations du déploiement
    - kubectl get pods -n dev -l app=$APP_NAME

  environment:
    name: development
    url: https://dev.example.com

  needs:
    - build:docker

  only:
    - main

# Déploiement sur l'environnement de production
deploy:production:
  stage: deploy
  image: alpine/k8s:1.28.0
  script:
    # Configuration kubectl
    - kubectl config set-cluster k8s --server="$KUBE_URL" --insecure-skip-tls-verify=true
    - kubectl config set-credentials gitlab --token="$KUBE_TOKEN_PROD"
    - kubectl config set-context default --cluster=k8s --user=gitlab
    - kubectl config use-context default

    # Déployer avec Helm
    - helm upgrade --install $APP_NAME ./helm-chart
      --namespace production
      --create-namespace
      --set image.tag=$IMAGE_TAG
      --set image.repository=$DOCKER_REGISTRY/$APP_NAME
      --wait
      --timeout 10m

    # Vérifier le déploiement
    - kubectl get all -n production -l app.kubernetes.io/name=$APP_NAME

  environment:
    name: production
    url: https://app.example.com

  needs:
    - build:docker

  # Déploiement manuel en production
  when: manual

  only:
    - tags  # Seulement sur les tags (releases)
```

### 1.2 Pipeline avec Tests d'Intégration et Sécurité

```yaml
# .gitlab-ci.yml

stages:
  - test
  - security
  - build
  - deploy

variables:
  APP_NAME: mon-app-secure
  DOCKER_REGISTRY: registry.gitlab.com/mon-groupe/mon-projet

# ========================================
# TESTS
# ========================================

test:unit:
  stage: test
  image: node:18-alpine
  script:
    - npm ci
    - npm run test:unit
  coverage: '/All files[^|]*\|[^|]*\s+([\d\.]+)/'
  artifacts:
    reports:
      junit: junit.xml
      coverage_report:
        coverage_format: cobertura
        path: coverage/cobertura-coverage.xml

test:integration:
  stage: test
  image: node:18-alpine
  services:
    # Base de données PostgreSQL pour les tests
    - postgres:15-alpine
  variables:
    POSTGRES_DB: test_db
    POSTGRES_USER: test_user
    POSTGRES_PASSWORD: test_password
    DATABASE_URL: "postgresql://test_user:test_password@postgres:5432/test_db"
  script:
    - npm ci
    # Attendre que PostgreSQL soit prêt
    - sleep 5
    # Exécuter les migrations
    - npm run migrate
    # Exécuter les tests d'intégration
    - npm run test:integration
  only:
    - merge_requests
    - main

# ========================================
# SÉCURITÉ
# ========================================

security:sast:
  stage: security
  # Analyse statique de sécurité
  image: returntocorp/semgrep
  script:
    - semgrep --config=auto --json --output=sast-report.json .
  artifacts:
    reports:
      sast: sast-report.json
  allow_failure: true

security:dependencies:
  stage: security
  image: node:18-alpine
  script:
    - npm ci
    # Audit des dépendances npm
    - npm audit --audit-level=high
    # Vérifier les licences
    - npx license-checker --summary
  allow_failure: true

security:secrets:
  stage: security
  # Détecter les secrets dans le code
  image: trufflesecurity/trufflehog:latest
  script:
    - trufflehog filesystem . --json > secrets-report.json
  artifacts:
    paths:
      - secrets-report.json
    expire_in: 1 week
  allow_failure: true

# ========================================
# BUILD
# ========================================

build:docker:
  stage: build
  image: docker:20.10
  services:
    - docker:20.10-dind
  before_script:
    - docker login -u $CI_REGISTRY_USER -p $CI_REGISTRY_PASSWORD $CI_REGISTRY
  script:
    - |
      docker build \
        --build-arg BUILD_DATE=$(date -u +"%Y-%m-%dT%H:%M:%SZ") \
        --build-arg VCS_REF=$CI_COMMIT_SHORT_SHA \
        --label "org.opencontainers.image.created=$(date -u +"%Y-%m-%dT%H:%M:%SZ")" \
        --label "org.opencontainers.image.revision=$CI_COMMIT_SHORT_SHA" \
        --label "org.opencontainers.image.version=$CI_COMMIT_TAG" \
        -t $DOCKER_REGISTRY/$APP_NAME:$CI_COMMIT_SHORT_SHA \
        -t $DOCKER_REGISTRY/$APP_NAME:latest \
        .
    - docker push $DOCKER_REGISTRY/$APP_NAME:$CI_COMMIT_SHORT_SHA
    - docker push $DOCKER_REGISTRY/$APP_NAME:latest

# Scanner l'image Docker pour les vulnérabilités
security:container:
  stage: build
  image: aquasec/trivy:latest
  script:
    # Scanner l'image
    - trivy image --severity HIGH,CRITICAL
      --format json
      --output trivy-report.json
      $DOCKER_REGISTRY/$APP_NAME:$CI_COMMIT_SHORT_SHA
    # Afficher le résumé
    - trivy image --severity HIGH,CRITICAL $DOCKER_REGISTRY/$APP_NAME:$CI_COMMIT_SHORT_SHA
  artifacts:
    reports:
      container_scanning: trivy-report.json
  dependencies:
    - build:docker
  allow_failure: true

# ========================================
# DÉPLOIEMENT
# ========================================

deploy:staging:
  stage: deploy
  image: alpine/helm:3.12.0
  script:
    - kubectl config set-cluster k8s --server="$KUBE_URL" --insecure-skip-tls-verify=true
    - kubectl config set-credentials gitlab --token="$KUBE_TOKEN"
    - kubectl config set-context default --cluster=k8s --user=gitlab
    - kubectl config use-context default

    - helm upgrade --install $APP_NAME ./helm-chart
      --namespace staging
      --create-namespace
      --set image.tag=$CI_COMMIT_SHORT_SHA
      --set image.repository=$DOCKER_REGISTRY/$APP_NAME
      --set ingress.hosts[0].host=staging.example.com
      --wait
      --timeout 5m
  environment:
    name: staging
    url: https://staging.example.com
    on_stop: stop:staging
  only:
    - main

# Nettoyer l'environnement staging
stop:staging:
  stage: deploy
  image: alpine/helm:3.12.0
  script:
    - kubectl config set-cluster k8s --server="$KUBE_URL" --insecure-skip-tls-verify=true
    - kubectl config set-credentials gitlab --token="$KUBE_TOKEN"
    - kubectl config set-context default --cluster=k8s --user=gitlab
    - kubectl config use-context default
    - helm uninstall $APP_NAME --namespace staging
  environment:
    name: staging
    action: stop
  when: manual
  only:
    - main

deploy:production:
  stage: deploy
  image: alpine/helm:3.12.0
  script:
    - kubectl config set-cluster k8s --server="$KUBE_URL_PROD" --insecure-skip-tls-verify=true
    - kubectl config set-credentials gitlab --token="$KUBE_TOKEN_PROD"
    - kubectl config set-context default --cluster=k8s --user=gitlab
    - kubectl config use-context default

    # Backup avant déploiement
    - helm get values $APP_NAME -n production > backup-values-$(date +%Y%m%d-%H%M%S).yaml

    # Déploiement avec stratégie blue-green
    - helm upgrade --install $APP_NAME ./helm-chart
      --namespace production
      --create-namespace
      --set image.tag=$CI_COMMIT_TAG
      --set image.repository=$DOCKER_REGISTRY/$APP_NAME
      --set replicaCount=3
      --set resources.limits.cpu=1000m
      --set resources.limits.memory=1Gi
      --wait
      --timeout 10m

    # Vérifier la santé après déploiement
    - kubectl wait --for=condition=available --timeout=300s deployment/$APP_NAME -n production
  environment:
    name: production
    url: https://app.example.com
  when: manual
  only:
    - tags
```

---

## 2. Pipeline GitHub Actions

GitHub Actions utilise des fichiers YAML dans `.github/workflows/`.

### 2.1 Pipeline Complet avec Déploiement

```yaml
# .github/workflows/ci-cd.yml

name: CI/CD Pipeline

# Déclencheurs
on:
  # Sur chaque push
  push:
    branches:
      - main
      - develop
  # Sur chaque pull request
  pull_request:
    branches:
      - main
  # Sur les tags
  push:
    tags:
      - 'v*'

# Variables d'environnement
env:
  APP_NAME: mon-app
  DOCKER_REGISTRY: ghcr.io
  DOCKER_IMAGE: ghcr.io/${{ github.repository_owner }}/mon-app

# Jobs du pipeline
jobs:

  # ========================================
  # JOB 1: TESTS
  # ========================================

  test:
    name: Tests Unitaires et Linting
    runs-on: ubuntu-latest

    strategy:
      matrix:
        node-version: [16.x, 18.x, 20.x]

    steps:
      # Récupérer le code
      - name: Checkout code
        uses: actions/checkout@v4

      # Configurer Node.js
      - name: Setup Node.js ${{ matrix.node-version }}
        uses: actions/setup-node@v4
        with:
          node-version: ${{ matrix.node-version }}
          cache: 'npm'

      # Installer les dépendances
      - name: Install dependencies
        run: npm ci

      # Linting
      - name: Run linter
        run: npm run lint

      # Tests unitaires
      - name: Run unit tests
        run: npm run test:unit

      # Générer le rapport de couverture
      - name: Generate coverage report
        run: npm run test:coverage

      # Upload du rapport de couverture vers Codecov
      - name: Upload coverage to Codecov
        uses: codecov/codecov-action@v3
        with:
          files: ./coverage/lcov.info
          flags: unittests
          name: codecov-${{ matrix.node-version }}

  # ========================================
  # JOB 2: TESTS D'INTÉGRATION
  # ========================================

  integration-tests:
    name: Tests d'Intégration
    runs-on: ubuntu-latest

    services:
      # PostgreSQL pour les tests
      postgres:
        image: postgres:15
        env:
          POSTGRES_DB: test_db
          POSTGRES_USER: test_user
          POSTGRES_PASSWORD: test_password
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5
        ports:
          - 5432:5432

      # Redis pour les tests
      redis:
        image: redis:7-alpine
        options: >-
          --health-cmd "redis-cli ping"
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5
        ports:
          - 6379:6379

    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: '18.x'
          cache: 'npm'

      - name: Install dependencies
        run: npm ci

      - name: Run database migrations
        env:
          DATABASE_URL: postgresql://test_user:test_password@localhost:5432/test_db
        run: npm run migrate

      - name: Run integration tests
        env:
          DATABASE_URL: postgresql://test_user:test_password@localhost:5432/test_db
          REDIS_URL: redis://localhost:6379
        run: npm run test:integration

  # ========================================
  # JOB 3: ANALYSE DE SÉCURITÉ
  # ========================================

  security:
    name: Analyse de Sécurité
    runs-on: ubuntu-latest

    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: '18.x'
          cache: 'npm'

      - name: Install dependencies
        run: npm ci

      # Audit des dépendances
      - name: Run npm audit
        run: npm audit --audit-level=moderate
        continue-on-error: true

      # Analyse CodeQL (GitHub Security)
      - name: Initialize CodeQL
        uses: github/codeql-action/init@v2
        with:
          languages: javascript

      - name: Perform CodeQL Analysis
        uses: github/codeql-action/analyze@v2

      # Scan des secrets
      - name: TruffleHog Secret Scanning
        uses: trufflesecurity/trufflehog@main
        with:
          path: ./
          base: ${{ github.event.repository.default_branch }}
          head: HEAD

  # ========================================
  # JOB 4: BUILD DOCKER
  # ========================================

  build:
    name: Build Docker Image
    runs-on: ubuntu-latest
    needs: [test, integration-tests]

    # Exécuter seulement sur main et les tags
    if: github.ref == 'refs/heads/main' || startsWith(github.ref, 'refs/tags/')

    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      # Configurer Docker Buildx
      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3

      # Login au GitHub Container Registry
      - name: Login to GitHub Container Registry
        uses: docker/login-action@v3
        with:
          registry: ghcr.io
          username: ${{ github.repository_owner }}
          password: ${{ secrets.GITHUB_TOKEN }}

      # Extraire les métadonnées pour Docker
      - name: Extract Docker metadata
        id: meta
        uses: docker/metadata-action@v5
        with:
          images: ${{ env.DOCKER_IMAGE }}
          tags: |
            type=ref,event=branch
            type=ref,event=pr
            type=semver,pattern={{version}}
            type=semver,pattern={{major}}.{{minor}}
            type=sha,prefix={{branch}}-

      # Build et push de l'image
      - name: Build and push Docker image
        uses: docker/build-push-action@v5
        with:
          context: .
          push: true
          tags: ${{ steps.meta.outputs.tags }}
          labels: ${{ steps.meta.outputs.labels }}
          cache-from: type=gha
          cache-to: type=gha,mode=max
          build-args: |
            BUILD_DATE=${{ github.event.head_commit.timestamp }}
            VCS_REF=${{ github.sha }}
            VERSION=${{ github.ref_name }}

      # Scanner l'image avec Trivy
      - name: Run Trivy vulnerability scanner
        uses: aquasecurity/trivy-action@master
        with:
          image-ref: ${{ env.DOCKER_IMAGE }}:${{ github.sha }}
          format: 'sarif'
          output: 'trivy-results.sarif'

      # Upload des résultats de scan
      - name: Upload Trivy results to GitHub Security
        uses: github/codeql-action/upload-sarif@v2
        with:
          sarif_file: 'trivy-results.sarif'

  # ========================================
  # JOB 5: DÉPLOIEMENT DÉVELOPPEMENT
  # ========================================

  deploy-dev:
    name: Deploy to Development
    runs-on: ubuntu-latest
    needs: [build]
    if: github.ref == 'refs/heads/main'

    environment:
      name: development
      url: https://dev.example.com

    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      # Configurer kubectl
      - name: Setup kubectl
        uses: azure/setup-kubectl@v3
        with:
          version: 'v1.28.0'

      # Configurer kubeconfig
      - name: Configure kubectl
        run: |
          mkdir -p $HOME/.kube
          echo "${{ secrets.KUBE_CONFIG_DEV }}" | base64 -d > $HOME/.kube/config
          chmod 600 $HOME/.kube/config

      # Déployer sur Kubernetes
      - name: Deploy to Kubernetes
        run: |
          kubectl set image deployment/${{ env.APP_NAME }} \
            ${{ env.APP_NAME }}=${{ env.DOCKER_IMAGE }}:${{ github.sha }} \
            -n development

          kubectl rollout status deployment/${{ env.APP_NAME }} \
            -n development \
            --timeout=5m

      # Vérifier le déploiement
      - name: Verify deployment
        run: |
          kubectl get pods -n development -l app=${{ env.APP_NAME }}
          kubectl get svc -n development -l app=${{ env.APP_NAME }}

  # ========================================
  # JOB 6: DÉPLOIEMENT PRODUCTION
  # ========================================

  deploy-prod:
    name: Deploy to Production
    runs-on: ubuntu-latest
    needs: [build]
    if: startsWith(github.ref, 'refs/tags/v')

    environment:
      name: production
      url: https://app.example.com

    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      # Configurer Helm
      - name: Setup Helm
        uses: azure/setup-helm@v3
        with:
          version: '3.12.0'

      # Configurer kubectl
      - name: Setup kubectl
        uses: azure/setup-kubectl@v3
        with:
          version: 'v1.28.0'

      # Configurer kubeconfig
      - name: Configure kubectl
        run: |
          mkdir -p $HOME/.kube
          echo "${{ secrets.KUBE_CONFIG_PROD }}" | base64 -d > $HOME/.kube/config
          chmod 600 $HOME/.kube/config

      # Extraire la version du tag
      - name: Get version from tag
        id: version
        run: echo "VERSION=${GITHUB_REF#refs/tags/v}" >> $GITHUB_OUTPUT

      # Déployer avec Helm
      - name: Deploy with Helm
        run: |
          helm upgrade --install ${{ env.APP_NAME }} ./helm-chart \
            --namespace production \
            --create-namespace \
            --set image.tag=${{ steps.version.outputs.VERSION }} \
            --set image.repository=${{ env.DOCKER_IMAGE }} \
            --set replicaCount=5 \
            --set ingress.enabled=true \
            --set ingress.hosts[0].host=app.example.com \
            --wait \
            --timeout 10m

      # Créer une release GitHub
      - name: Create GitHub Release
        uses: softprops/action-gh-release@v1
        with:
          generate_release_notes: true
          body: |
            ## Déploiement Production

            Version déployée: ${{ steps.version.outputs.VERSION }}
            Image Docker: ${{ env.DOCKER_IMAGE }}:${{ steps.version.outputs.VERSION }}

            ### Changements
            ${{ github.event.head_commit.message }}
```

### 2.2 Pipeline avec Environnements Multiples

```yaml
# .github/workflows/multi-env.yml

name: Multi-Environment Deployment

on:
  push:
    branches:
      - main
      - develop
      - 'feature/*'

env:
  REGISTRY: ghcr.io
  IMAGE_NAME: ${{ github.repository }}

jobs:

  # ========================================
  # BUILD
  # ========================================

  build:
    runs-on: ubuntu-latest
    outputs:
      image-tag: ${{ steps.meta.outputs.tags }}

    steps:
      - uses: actions/checkout@v4

      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3

      - name: Log in to Container Registry
        uses: docker/login-action@v3
        with:
          registry: ${{ env.REGISTRY }}
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}

      - name: Extract metadata
        id: meta
        uses: docker/metadata-action@v5
        with:
          images: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}

      - name: Build and push
        uses: docker/build-push-action@v5
        with:
          context: .
          push: true
          tags: ${{ steps.meta.outputs.tags }}
          labels: ${{ steps.meta.outputs.labels }}

  # ========================================
  # DÉPLOIEMENT FEATURE
  # ========================================

  deploy-feature:
    if: startsWith(github.ref, 'refs/heads/feature/')
    needs: build
    runs-on: ubuntu-latest

    steps:
      - uses: actions/checkout@v4

      - name: Extract branch name
        id: extract_branch
        run: echo "BRANCH_NAME=${GITHUB_REF#refs/heads/feature/}" >> $GITHUB_OUTPUT

      - name: Setup kubectl
        uses: azure/setup-kubectl@v3

      - name: Configure kubectl
        run: |
          echo "${{ secrets.KUBE_CONFIG }}" | base64 -d > $HOME/.kube/config

      - name: Create feature namespace
        run: |
          kubectl create namespace feature-${{ steps.extract_branch.outputs.BRANCH_NAME }} \
            --dry-run=client -o yaml | kubectl apply -f -

      - name: Deploy to feature environment
        run: |
          kubectl set image deployment/app \
            app=${{ needs.build.outputs.image-tag }} \
            -n feature-${{ steps.extract_branch.outputs.BRANCH_NAME }}

      - name: Comment PR with URL
        uses: actions/github-script@v7
        if: github.event_name == 'pull_request'
        with:
          script: |
            github.rest.issues.createComment({
              issue_number: context.issue.number,
              owner: context.repo.owner,
              repo: context.repo.repo,
              body: '🚀 Feature deployed at: https://feature-${{ steps.extract_branch.outputs.BRANCH_NAME }}.example.com'
            })

  # ========================================
  # DÉPLOIEMENT STAGING (develop branch)
  # ========================================

  deploy-staging:
    if: github.ref == 'refs/heads/develop'
    needs: build
    runs-on: ubuntu-latest
    environment:
      name: staging
      url: https://staging.example.com

    steps:
      - uses: actions/checkout@v4

      - name: Setup kubectl
        uses: azure/setup-kubectl@v3

      - name: Configure kubectl
        run: echo "${{ secrets.KUBE_CONFIG }}" | base64 -d > $HOME/.kube/config

      - name: Deploy to staging
        run: |
          kubectl set image deployment/app \
            app=${{ needs.build.outputs.image-tag }} \
            -n staging
          kubectl rollout status deployment/app -n staging

  # ========================================
  # DÉPLOIEMENT PRODUCTION (main branch)
  # ========================================

  deploy-production:
    if: github.ref == 'refs/heads/main'
    needs: build
    runs-on: ubuntu-latest
    environment:
      name: production
      url: https://app.example.com

    steps:
      - uses: actions/checkout@v4

      - name: Setup Helm
        uses: azure/setup-helm@v3

      - name: Setup kubectl
        uses: azure/setup-kubectl@v3

      - name: Configure kubectl
        run: echo "${{ secrets.KUBE_CONFIG_PROD }}" | base64 -d > $HOME/.kube/config

      - name: Deploy to production
        run: |
          helm upgrade --install app ./helm-chart \
            --namespace production \
            --set image.tag=${{ github.sha }} \
            --wait
```

---

## 3. Pipeline Jenkins

Jenkins utilise des Jenkinsfiles pour définir les pipelines.

### 3.1 Pipeline Déclaratif

```groovy
// Jenkinsfile

// Pipeline déclaratif
pipeline {
    // Agent: où le pipeline s'exécute
    agent any

    // Variables d'environnement
    environment {
        APP_NAME = 'mon-app'
        DOCKER_REGISTRY = 'registry.example.com'
        DOCKER_IMAGE = "${DOCKER_REGISTRY}/${APP_NAME}"
        KUBE_NAMESPACE = 'production'
        // Credentials Jenkins
        DOCKER_CREDENTIALS = credentials('docker-registry-credentials')
        KUBE_CONFIG = credentials('kubeconfig')
    }

    // Paramètres du build
    parameters {
        choice(
            name: 'ENVIRONMENT',
            choices: ['dev', 'staging', 'production'],
            description: 'Environnement de déploiement'
        )
        booleanParam(
            name: 'RUN_TESTS',
            defaultValue: true,
            description: 'Exécuter les tests ?'
        )
        string(
            name: 'IMAGE_TAG',
            defaultValue: 'latest',
            description: 'Tag de l\'image Docker'
        )
    }

    // Déclencheurs
    triggers {
        // Poll SCM toutes les 5 minutes
        pollSCM('H/5 * * * *')
        // Build périodique
        cron('H 2 * * *')
    }

    // Options
    options {
        // Conserver les 10 derniers builds
        buildDiscarder(logRotator(numToKeepStr: '10'))
        // Timeout du pipeline
        timeout(time: 30, unit: 'MINUTES')
        // Pas de builds concurrents
        disableConcurrentBuilds()
        // Timestamps dans les logs
        timestamps()
    }

    // Étapes du pipeline
    stages {

        // ========================================
        // ÉTAPE 1: CHECKOUT
        // ========================================

        stage('Checkout') {
            steps {
                echo 'Récupération du code source...'
                checkout scm

                // Afficher les informations du commit
                script {
                    env.GIT_COMMIT_MSG = sh(
                        script: 'git log -1 --pretty=%B',
                        returnStdout: true
                    ).trim()
                    env.GIT_AUTHOR = sh(
                        script: 'git log -1 --pretty=%an',
                        returnStdout: true
                    ).trim()
                }

                echo "Commit: ${env.GIT_COMMIT_MSG}"
                echo "Auteur: ${env.GIT_AUTHOR}"
            }
        }

        // ========================================
        // ÉTAPE 2: TESTS
        // ========================================

        stage('Tests') {
            when {
                expression { params.RUN_TESTS == true }
            }

            parallel {
                // Tests unitaires
                stage('Unit Tests') {
                    agent {
                        docker {
                            image 'node:18-alpine'
                            reuseNode true
                        }
                    }
                    steps {
                        echo 'Exécution des tests unitaires...'
                        sh 'npm ci'
                        sh 'npm run test:unit'
                    }
                    post {
                        always {
                            // Publier les résultats des tests
                            junit 'test-results/*.xml'
                        }
                    }
                }

                // Linting
                stage('Linting') {
                    agent {
                        docker {
                            image 'node:18-alpine'
                            reuseNode true
                        }
                    }
                    steps {
                        echo 'Analyse du code...'
                        sh 'npm ci'
                        sh 'npm run lint'
                    }
                }

                // Analyse de sécurité
                stage('Security Scan') {
                    steps {
                        echo 'Scan de sécurité...'
                        sh 'npm audit --audit-level=moderate || true'
                    }
                }
            }
        }

        // ========================================
        // ÉTAPE 3: BUILD
        // ========================================

        stage('Build Docker Image') {
            steps {
                echo 'Construction de l\'image Docker...'
                script {
                    // Définir le tag de l'image
                    def imageTag = params.IMAGE_TAG == 'latest' ?
                        env.BUILD_NUMBER : params.IMAGE_TAG

                    env.FULL_IMAGE_TAG = "${DOCKER_IMAGE}:${imageTag}"

                    // Builder l'image
                    docker.build(
                        env.FULL_IMAGE_TAG,
                        "--build-arg BUILD_NUMBER=${env.BUILD_NUMBER} " +
                        "--build-arg GIT_COMMIT=${env.GIT_COMMIT} ."
                    )
                }
            }
        }

        // ========================================
        // ÉTAPE 4: SCAN DE L'IMAGE
        // ========================================

        stage('Scan Docker Image') {
            steps {
                echo 'Scan de vulnérabilités...'
                script {
                    // Scanner avec Trivy
                    sh """
                        docker run --rm \
                            -v /var/run/docker.sock:/var/run/docker.sock \
                            aquasec/trivy:latest image \
                            --severity HIGH,CRITICAL \
                            --exit-code 0 \
                            ${env.FULL_IMAGE_TAG}
                    """
                }
            }
        }

        // ========================================
        // ÉTAPE 5: PUSH DE L'IMAGE
        // ========================================

        stage('Push Docker Image') {
            steps {
                echo 'Push de l\'image vers le registry...'
                script {
                    docker.withRegistry(
                        "https://${DOCKER_REGISTRY}",
                        'docker-registry-credentials'
                    ) {
                        docker.image(env.FULL_IMAGE_TAG).push()
                        docker.image(env.FULL_IMAGE_TAG).push('latest')
                    }
                }
            }
        }

        // ========================================
        // ÉTAPE 6: DÉPLOIEMENT
        // ========================================

        stage('Deploy to Kubernetes') {
            steps {
                echo "Déploiement sur ${params.ENVIRONMENT}..."
                script {
                    // Configurer kubectl
                    sh """
                        mkdir -p ~/.kube
                        echo "${KUBE_CONFIG}" > ~/.kube/config
                        chmod 600 ~/.kube/config
                    """

                    // Déployer selon l'environnement
                    if (params.ENVIRONMENT == 'production') {
                        // Demander confirmation pour la production
                        input message: 'Déployer en production ?',
                              ok: 'Déployer'
                    }

                    // Mise à jour du déploiement
                    sh """
                        kubectl set image deployment/${APP_NAME} \
                            ${APP_NAME}=${env.FULL_IMAGE_TAG} \
                            -n ${params.ENVIRONMENT}

                        kubectl rollout status deployment/${APP_NAME} \
                            -n ${params.ENVIRONMENT} \
                            --timeout=5m
                    """
                }
            }
        }

        // ========================================
        // ÉTAPE 7: TESTS DE SMOKE
        // ========================================

        stage('Smoke Tests') {
            steps {
                echo 'Exécution des tests de smoke...'
                script {
                    // Obtenir l'URL de l'application
                    def appUrl = sh(
                        script: """
                            kubectl get ingress ${APP_NAME} \
                                -n ${params.ENVIRONMENT} \
                                -o jsonpath='{.spec.rules[0].host}'
                        """,
                        returnStdout: true
                    ).trim()

                    // Tester l'application
                    sh """
                        curl -f https://${appUrl}/health || exit 1
                        echo "✅ Application accessible"
                    """
                }
            }
        }
    }

    // Actions post-build
    post {
        // Toujours exécuté
        always {
            echo 'Nettoyage...'
            // Nettoyer les images Docker locales
            sh "docker rmi ${env.FULL_IMAGE_TAG} || true"

            // Archiver les artifacts
            archiveArtifacts artifacts: '**/target/*.jar',
                            allowEmptyArchive: true
        }

        // En cas de succès
        success {
            echo '✅ Pipeline réussi !'

            // Notification Slack
            slackSend(
                color: 'good',
                message: """
                    ✅ Build réussi !
                    Job: ${env.JOB_NAME}
                    Build: ${env.BUILD_NUMBER}
                    Environment: ${params.ENVIRONMENT}
                    Auteur: ${env.GIT_AUTHOR}
                """
            )
        }

        // En cas d'échec
        failure {
            echo '❌ Pipeline échoué !'

            // Notification Slack
            slackSend(
                color: 'danger',
                message: """
                    ❌ Build échoué !
                    Job: ${env.JOB_NAME}
                    Build: ${env.BUILD_NUMBER}
                    Voir: ${env.BUILD_URL}
                """
            )
        }

        // Si instable
        unstable {
            echo '⚠️ Build instable'
        }

        // Toujours nettoyer le workspace
        cleanup {
            cleanWs()
        }
    }
}
```

### 3.2 Pipeline Scripted (Avancé)

```groovy
// Jenkinsfile (Pipeline Scripted)

// Variables globales
def dockerImage
def imageTag
def kubeContext

// Node principal
node {

    try {
        // ========================================
        // CONFIGURATION
        // ========================================

        stage('Configuration') {
            // Définir les variables
            env.APP_NAME = 'mon-app'
            env.DOCKER_REGISTRY = 'registry.example.com'
            imageTag = "${env.BUILD_NUMBER}-${env.GIT_COMMIT[0..7]}"

            echo "Configuration: ${env.APP_NAME}:${imageTag}"
        }

        // ========================================
        // CHECKOUT
        // ========================================

        stage('Checkout') {
            checkout scm

            // Obtenir les informations du commit
            env.GIT_COMMIT_MSG = sh(
                script: 'git log -1 --pretty=%B',
                returnStdout: true
            ).trim()
        }

        // ========================================
        // TESTS EN PARALLÈLE
        // ========================================

        stage('Tests') {
            parallel(
                'Unit Tests': {
                    node('docker') {
                        sh 'npm ci && npm run test:unit'
                    }
                },
                'Integration Tests': {
                    node('docker') {
                        sh 'npm ci && npm run test:integration'
                    }
                },
                'E2E Tests': {
                    node('docker') {
                        sh 'npm ci && npm run test:e2e'
                    }
                }
            )
        }

        // ========================================
        // BUILD
        // ========================================

        stage('Build') {
            // Builder l'image Docker
            dockerImage = docker.build(
                "${env.DOCKER_REGISTRY}/${env.APP_NAME}:${imageTag}"
            )

            // Tagger comme latest
            dockerImage.tag('latest')
        }

        // ========================================
        // PUSH
        // ========================================

        stage('Push') {
            docker.withRegistry(
                "https://${env.DOCKER_REGISTRY}",
                'docker-credentials'
            ) {
                dockerImage.push(imageTag)
                dockerImage.push('latest')
            }
        }

        // ========================================
        // DÉPLOIEMENT
        // ========================================

        stage('Deploy') {
            // Sélection de l'environnement
            def environment = input(
                message: 'Déployer sur quel environnement ?',
                parameters: [
                    choice(
                        name: 'ENV',
                        choices: ['dev', 'staging', 'production'],
                        description: 'Environnement cible'
                    )
                ]
            )

            echo "Déploiement sur ${environment}..."

            // Configurer kubectl
            withCredentials([file(credentialsId: "kubeconfig-${environment}", variable: 'KUBECONFIG')]) {
                sh """
                    kubectl set image deployment/${env.APP_NAME} \
                        ${env.APP_NAME}=${env.DOCKER_REGISTRY}/${env.APP_NAME}:${imageTag} \
                        -n ${environment}

                    kubectl rollout status deployment/${env.APP_NAME} \
                        -n ${environment}
                """
            }

            // Enregistrer l'environnement déployé
            env.DEPLOYED_ENV = environment
        }

        // ========================================
        // VÉRIFICATION
        // ========================================

        stage('Verification') {
            timeout(time: 5, unit: 'MINUTES') {
                waitUntil {
                    script {
                        def response = sh(
                            script: "kubectl get pods -n ${env.DEPLOYED_ENV} -l app=${env.APP_NAME} -o jsonpath='{.items[*].status.phase}'",
                            returnStdout: true
                        ).trim()

                        return response.contains('Running')
                    }
                }
            }

            echo '✅ Déploiement vérifié avec succès'
        }

    } catch (Exception e) {
        // Gestion des erreurs
        currentBuild.result = 'FAILURE'
        echo "❌ Erreur: ${e.message}"

        // Rollback en cas d'échec
        if (env.DEPLOYED_ENV) {
            stage('Rollback') {
                echo 'Rollback en cours...'
                withCredentials([file(credentialsId: "kubeconfig-${env.DEPLOYED_ENV}", variable: 'KUBECONFIG')]) {
                    sh "kubectl rollout undo deployment/${env.APP_NAME} -n ${env.DEPLOYED_ENV}"
                }
            }
        }

        throw e
    } finally {
        // Nettoyage
        stage('Cleanup') {
            sh 'docker system prune -f'
            cleanWs()
        }

        // Notification
        if (currentBuild.result == 'FAILURE') {
            emailext(
                subject: "❌ Build Failed: ${env.JOB_NAME} #${env.BUILD_NUMBER}",
                body: """
                    Le build a échoué.

                    Job: ${env.JOB_NAME}
                    Build: ${env.BUILD_NUMBER}
                    Commit: ${env.GIT_COMMIT_MSG}

                    Voir: ${env.BUILD_URL}
                """,
                to: '${DEFAULT_RECIPIENTS}'
            )
        }
    }
}
```

---

## 4. Pipeline ArgoCD (GitOps)

ArgoCD suit le principe GitOps : l'état du cluster est défini dans Git.

### 4.1 Configuration ArgoCD Application

```yaml
# argocd/application.yaml

apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: mon-app
  namespace: argocd
  # Finalizers pour gérer la suppression
  finalizers:
    - resources-finalizer.argocd.argoproj.io
spec:
  # Projet ArgoCD
  project: default

  # Source: où se trouve le code
  source:
    # Repository Git
    repoURL: https://github.com/mon-org/mon-app.git
    # Branche ou tag
    targetRevision: main
    # Chemin vers les manifests Kubernetes ou Helm Chart
    path: k8s/

    # Si utilisation de Helm
    helm:
      # Fichier values à utiliser
      valueFiles:
        - values-production.yaml
      # Paramètres à surcharger
      parameters:
        - name: image.tag
          value: "1.2.3"
        - name: replicaCount
          value: "5"

  # Destination: où déployer
  destination:
    # Serveur Kubernetes (in-cluster ou externe)
    server: https://kubernetes.default.svc
    # Namespace cible
    namespace: production

  # Politique de synchronisation
  syncPolicy:
    # Synchronisation automatique
    automated:
      # Auto-correction si modification manuelle
      selfHeal: true
      # Supprime les ressources qui ne sont plus dans Git
      prune: true
      # Autoriser les champs vides
      allowEmpty: false

    # Options de synchronisation
    syncOptions:
      # Créer le namespace s'il n'existe pas
      - CreateNamespace=true
      # Valider les ressources avant de les créer
      - Validate=true
      # Utiliser server-side apply
      - ServerSideApply=true

    # Politique de retry
    retry:
      limit: 5
      backoff:
        duration: 5s
        factor: 2
        maxDuration: 3m

  # Configuration avancée
  ignoreDifferences:
    # Ignorer les différences sur certains champs
    - group: apps
      kind: Deployment
      jsonPointers:
        - /spec/replicas  # Ignorer si HPA modifie les replicas
```

### 4.2 Application Multi-Environnements

```yaml
# argocd/app-of-apps.yaml
# Application ArgoCD qui gère d'autres applications

apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: mon-app-all-envs
  namespace: argocd
spec:
  project: default

  source:
    repoURL: https://github.com/mon-org/mon-app.git
    targetRevision: main
    path: argocd/environments

  destination:
    server: https://kubernetes.default.svc
    namespace: argocd

  syncPolicy:
    automated:
      selfHeal: true
      prune: true
```

```yaml
# argocd/environments/dev.yaml

apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: mon-app-dev
  namespace: argocd
spec:
  project: default

  source:
    repoURL: https://github.com/mon-org/mon-app.git
    targetRevision: develop  # Branche develop pour dev
    path: helm-chart
    helm:
      valueFiles:
        - values-dev.yaml
      parameters:
        - name: image.tag
          value: develop-latest

  destination:
    server: https://kubernetes.default.svc
    namespace: development

  syncPolicy:
    automated:
      selfHeal: true
      prune: true
    syncOptions:
      - CreateNamespace=true
```

```yaml
# argocd/environments/staging.yaml

apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: mon-app-staging
  namespace: argocd
spec:
  project: default

  source:
    repoURL: https://github.com/mon-org/mon-app.git
    targetRevision: main
    path: helm-chart
    helm:
      valueFiles:
        - values-staging.yaml

  destination:
    server: https://kubernetes.default.svc
    namespace: staging

  syncPolicy:
    automated:
      selfHeal: true
      prune: true
    syncOptions:
      - CreateNamespace=true
```

```yaml
# argocd/environments/production.yaml

apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: mon-app-production
  namespace: argocd
spec:
  project: default

  source:
    repoURL: https://github.com/mon-org/mon-app.git
    targetRevision: v1.2.3  # Tag pour production
    path: helm-chart
    helm:
      valueFiles:
        - values-production.yaml
      parameters:
        - name: replicaCount
          value: "10"

  destination:
    server: https://kubernetes.default.svc
    namespace: production

  # Synchronisation manuelle en production
  syncPolicy:
    syncOptions:
      - CreateNamespace=true
    retry:
      limit: 3
      backoff:
        duration: 5s
        maxDuration: 3m
```

### 4.3 Hooks ArgoCD pour Migrations

```yaml
# k8s/db-migration-job.yaml

apiVersion: batch/v1
kind: Job
metadata:
  name: db-migration
  namespace: production
  annotations:
    # Hook ArgoCD: exécuter avant la synchronisation
    argocd.argoproj.io/hook: PreSync
    # Supprimer le Job après succès
    argocd.argoproj.io/hook-delete-policy: HookSucceeded
    # Poids d'exécution (ordre)
    argocd.argoproj.io/sync-wave: "1"
spec:
  template:
    spec:
      restartPolicy: OnFailure
      containers:
      - name: migration
        image: mon-app-migrations:latest
        command:
        - sh
        - -c
        - |
          echo "Exécution des migrations..."
          flyway migrate
          echo "Migrations terminées"
        env:
        - name: DB_URL
          valueFrom:
            secretKeyRef:
              name: db-credentials
              key: url
```

```yaml
# k8s/post-deploy-test.yaml

apiVersion: batch/v1
kind: Job
metadata:
  name: post-deploy-test
  namespace: production
  annotations:
    # Hook ArgoCD: exécuter après la synchronisation
    argocd.argoproj.io/hook: PostSync
    argocd.argoproj.io/hook-delete-policy: HookSucceeded
    argocd.argoproj.io/sync-wave: "2"
spec:
  template:
    spec:
      restartPolicy: Never
      containers:
      - name: test
        image: curlimages/curl:latest
        command:
        - sh
        - -c
        - |
          echo "Test de l'application..."
          curl -f http://mon-app/health || exit 1
          echo "✅ Application fonctionnelle"
```

---

## 5. Pipeline CircleCI

CircleCI utilise des fichiers `.circleci/config.yml`.

### 5.1 Configuration CircleCI

```yaml
# .circleci/config.yml

version: 2.1

# Orbs: packages réutilisables
orbs:
  # Orb Kubernetes
  kubernetes: circleci/kubernetes@1.3
  # Orb Helm
  helm: circleci/helm@2.0
  # Orb Docker
  docker: circleci/docker@2.2

# Executors: environnements d'exécution
executors:
  node-executor:
    docker:
      - image: cimg/node:18.16
    working_directory: ~/project

  docker-executor:
    docker:
      - image: cimg/base:stable
    working_directory: ~/project

# Jobs réutilisables
jobs:

  # ========================================
  # JOB: TESTS
  # ========================================

  test:
    executor: node-executor
    steps:
      # Checkout du code
      - checkout

      # Restaurer le cache npm
      - restore_cache:
          keys:
            - npm-deps-{{ checksum "package-lock.json" }}
            - npm-deps-

      # Installer les dépendances
      - run:
          name: Install Dependencies
          command: npm ci

      # Sauvegarder le cache
      - save_cache:
          key: npm-deps-{{ checksum "package-lock.json" }}
          paths:
            - node_modules

      # Linting
      - run:
          name: Run Linter
          command: npm run lint

      # Tests unitaires
      - run:
          name: Run Unit Tests
          command: npm run test:unit

      # Générer le rapport de couverture
      - run:
          name: Generate Coverage Report
          command: npm run test:coverage

      # Sauvegarder les résultats
      - store_test_results:
          path: test-results

      - store_artifacts:
          path: coverage

  # ========================================
  # JOB: BUILD DOCKER
  # ========================================

  build:
    executor: docker-executor
    steps:
      - checkout

      - setup_remote_docker:
          version: 20.10.14
          docker_layer_caching: true

      # Login au registry
      - run:
          name: Docker Login
          command: |
            echo "$DOCKER_PASSWORD" | docker login \
              -u "$DOCKER_USERNAME" \
              --password-stdin \
              $DOCKER_REGISTRY

      # Build de l'image
      - run:
          name: Build Docker Image
          command: |
            docker build \
              --build-arg BUILD_DATE=$(date -u +"%Y-%m-%dT%H:%M:%SZ") \
              --build-arg VCS_REF=$CIRCLE_SHA1 \
              -t $DOCKER_REGISTRY/mon-app:$CIRCLE_SHA1 \
              -t $DOCKER_REGISTRY/mon-app:latest \
              .

      # Push de l'image
      - run:
          name: Push Docker Image
          command: |
            docker push $DOCKER_REGISTRY/mon-app:$CIRCLE_SHA1
            docker push $DOCKER_REGISTRY/mon-app:latest

      # Scanner l'image
      - run:
          name: Scan Image
          command: |
            docker run --rm \
              -v /var/run/docker.sock:/var/run/docker.sock \
              aquasec/trivy:latest image \
              --severity HIGH,CRITICAL \
              $DOCKER_REGISTRY/mon-app:$CIRCLE_SHA1

  # ========================================
  # JOB: DÉPLOIEMENT
  # ========================================

  deploy:
    executor: docker-executor
    parameters:
      environment:
        type: string
        default: "development"
    steps:
      - checkout

      # Installer kubectl
      - kubernetes/install-kubectl

      # Installer Helm
      - helm/install-helm-client:
          version: v3.12.0

      # Configurer kubectl
      - run:
          name: Configure kubectl
          command: |
            echo "$KUBE_CONFIG" | base64 -d > ~/.kube/config
            kubectl config use-context << parameters.environment >>

      # Déployer avec Helm
      - run:
          name: Deploy with Helm
          command: |
            helm upgrade --install mon-app ./helm-chart \
              --namespace << parameters.environment >> \
              --create-namespace \
              --set image.tag=$CIRCLE_SHA1 \
              --set image.repository=$DOCKER_REGISTRY/mon-app \
              --wait \
              --timeout 5m

      # Vérifier le déploiement
      - run:
          name: Verify Deployment
          command: |
            kubectl rollout status deployment/mon-app \
              -n << parameters.environment >>
            kubectl get pods -n << parameters.environment >>

# Workflows: orchestration des jobs
workflows:
  version: 2

  # Workflow principal
  build-test-deploy:
    jobs:
      # Tests sur toutes les branches
      - test

      # Build seulement sur main et tags
      - build:
          requires:
            - test
          filters:
            branches:
              only:
                - main
                - develop
            tags:
              only: /^v.*/

      # Déploiement dev automatique
      - deploy:
          name: deploy-dev
          environment: development
          requires:
            - build
          filters:
            branches:
              only: develop

      # Déploiement staging automatique
      - deploy:
          name: deploy-staging
          environment: staging
          requires:
            - build
          filters:
            branches:
              only: main

      # Déploiement production avec approbation
      - hold-production:
          type: approval
          requires:
            - build
          filters:
            tags:
              only: /^v.*/
            branches:
              ignore: /.*/

      - deploy:
          name: deploy-production
          environment: production
          requires:
            - hold-production
          filters:
            tags:
              only: /^v.*/
            branches:
              ignore: /.*/
```

---

## 6. Bonnes Pratiques CI/CD

### 6.1 Sécurité

✅ **Recommandations** :

```yaml
# Ne JAMAIS mettre de secrets dans le code
# ❌ MAUVAIS
env:
  DB_PASSWORD: "mon-mot-de-passe"

# ✅ BON - Utiliser les secrets du CI/CD
env:
  DB_PASSWORD: ${{ secrets.DB_PASSWORD }}
```

```yaml
# Scanner les images pour les vulnérabilités
- name: Scan image
  run: |
    trivy image --severity HIGH,CRITICAL mon-image:latest
```

```yaml
# Utiliser des images de base sûres et à jour
FROM node:18-alpine  # ✅ Image officielle légère
# Pas FROM node:latest  # ❌ Version non fixée
```

### 6.2 Performance

✅ **Recommandations** :

```yaml
# Utiliser le cache pour accélérer les builds
- uses: actions/cache@v3
  with:
    path: ~/.npm
    key: ${{ runner.os }}-node-${{ hashFiles('**/package-lock.json') }}
```

```yaml
# Builds multi-stage pour réduire la taille
FROM node:18-alpine AS builder
WORKDIR /app
COPY package*.json ./
RUN npm ci --only=production

FROM node:18-alpine
WORKDIR /app
COPY --from=builder /app/node_modules ./node_modules
COPY . .
CMD ["node", "server.js"]
```

```yaml
# Tests en parallèle
jobs:
  test:
    strategy:
      matrix:
        node: [16, 18, 20]
        os: [ubuntu-latest, macos-latest]
```

### 6.3 Qualité

✅ **Recommandations** :

```yaml
# Tests obligatoires avant merge
- name: Run Tests
  run: |
    npm run test
    npm run lint
    npm run test:coverage
  # Échouer si couverture < 80%
  coverage-threshold: 80
```

```yaml
# Validation des manifestes Kubernetes
- name: Validate Kubernetes manifests
  run: |
    kubectl apply --dry-run=server -f k8s/
```

```yaml
# Utiliser des outils de qualité de code
- name: SonarQube Scan
  uses: sonarsource/sonarqube-scan-action@master
  env:
    SONAR_TOKEN: ${{ secrets.SONAR_TOKEN }}
```

### 6.4 Déploiement

✅ **Recommandations** :

```yaml
# Déploiement progressif (Canary)
- name: Canary Deployment
  run: |
    # Déployer 10% du trafic sur la nouvelle version
    kubectl set image deployment/app app=app:new
    kubectl patch deployment app -p '{"spec":{"replicas":1}}'

    # Attendre et surveiller
    sleep 300

    # Si OK, déployer 100%
    kubectl patch deployment app -p '{"spec":{"replicas":10}}'
```

```yaml
# Rollback automatique en cas d'échec
- name: Deploy with Rollback
  run: |
    kubectl apply -f deployment.yaml
    if ! kubectl rollout status deployment/app --timeout=5m; then
      echo "Déploiement échoué, rollback..."
      kubectl rollout undo deployment/app
      exit 1
    fi
```

```yaml
# Health checks après déploiement
- name: Health Check
  run: |
    for i in {1..30}; do
      if curl -f https://app.example.com/health; then
        echo "✅ Application healthy"
        exit 0
      fi
      sleep 10
    done
    echo "❌ Health check failed"
    exit 1
```

### 6.5 Monitoring et Notifications

✅ **Recommandations** :

```yaml
# Notifications Slack
- name: Slack Notification
  if: always()
  uses: 8398a7/action-slack@v3
  with:
    status: ${{ job.status }}
    text: |
      Deployment ${{ job.status }}
      Branch: ${{ github.ref }}
      Commit: ${{ github.sha }}
    webhook_url: ${{ secrets.SLACK_WEBHOOK }}
```

```yaml
# Métriques de déploiement
- name: Record Deployment
  run: |
    curl -X POST https://metrics.example.com/deployment \
      -d "app=mon-app" \
      -d "version=${{ github.sha }}" \
      -d "environment=production" \
      -d "status=success"
```

---

## 7. Troubleshooting

### 7.1 Problèmes Communs

**Erreur: Image non trouvée**
```bash
# Vérifier que l'image a été pushée
docker images | grep mon-app

# Vérifier les credentials
docker login registry.example.com
```

**Erreur: kubectl connection refused**
```bash
# Vérifier le kubeconfig
kubectl config view
kubectl cluster-info

# Tester la connexion
kubectl get nodes
```

**Pipeline lent**
```yaml
# Utiliser le cache
- uses: actions/cache@v3
  with:
    path: ~/.npm
    key: ${{ hashFiles('package-lock.json') }}

# Docker layer caching
- uses: docker/build-push-action@v4
  with:
    cache-from: type=gha
    cache-to: type=gha,mode=max
```

### 7.2 Debug

```yaml
# Mode debug dans GitHub Actions
- name: Debug
  run: |
    echo "SHA: ${{ github.sha }}"
    echo "Ref: ${{ github.ref }}"
    env
  if: runner.debug == '1'
```

```groovy
// Debug dans Jenkins
stage('Debug') {
    steps {
        script {
            echo "Variables d'environnement:"
            sh 'env | sort'

            echo "Workspace:"
            sh 'ls -la'
        }
    }
}
```

---

## Conclusion

Les pipelines CI/CD automatisent :
- ✅ Les tests et la validation
- 🔨 La construction des images
- 🔍 L'analyse de sécurité
- 🚀 Le déploiement sur Kubernetes
- 🔄 Les rollbacks en cas de problème

**Prochaines étapes** :
1. Choisir votre outil CI/CD préféré
2. Commencer par un pipeline simple
3. Ajouter progressivement tests et checks
4. Automatiser les déploiements
5. Mettre en place le monitoring

**Ressources** :
- GitLab CI: https://docs.gitlab.com/ee/ci/
- GitHub Actions: https://docs.github.com/actions
- Jenkins: https://www.jenkins.io/doc/
- ArgoCD: https://argo-cd.readthedocs.io/
- CircleCI: https://circleci.com/docs/

Bon déploiement automatisé ! 🚀


⏭️ [Annexe C : Configuration Réseau](/annexes/annexe-c-configuration-reseau/README.md)
