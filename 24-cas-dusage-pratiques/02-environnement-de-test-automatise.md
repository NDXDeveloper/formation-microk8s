🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 24.2 Environnement de Test Automatisé

## Introduction

Un environnement de test automatisé sur MicroK8s vous permet de valider automatiquement vos applications avant leur déploiement en production. Au lieu de tester manuellement chaque modification, vous créez un système qui vérifie automatiquement que tout fonctionne correctement.

Dans ce chapitre, nous allons découvrir comment transformer votre cluster MicroK8s en une plateforme de test automatisé complète, capable de tester vos applications à différents niveaux et de vous alerter en cas de problème.

## Pourquoi Automatiser les Tests ?

### Les Avantages de l'Automatisation

**Gain de temps** : Les tests s'exécutent automatiquement à chaque modification de code, sans intervention manuelle.

**Fiabilité** : Les tests automatisés sont cohérents et reproductibles, contrairement aux tests manuels qui peuvent varier.

**Détection rapide des bugs** : Les problèmes sont identifiés immédiatement, avant d'atteindre la production.

**Confiance dans le code** : Vous pouvez modifier votre code en sachant que les tests détecteront les régressions.

**Documentation vivante** : Les tests décrivent comment votre application doit fonctionner.

### Les Différents Niveaux de Test

Dans un environnement Kubernetes, on distingue plusieurs types de tests :

1. **Tests unitaires** : Testent les fonctions individuelles de votre code (généralement exécutés avant le build)
2. **Tests d'intégration** : Vérifient que les différents composants fonctionnent ensemble
3. **Tests end-to-end (E2E)** : Simulent le comportement d'un utilisateur réel
4. **Tests de performance** : Mesurent les performances sous charge
5. **Tests de sécurité** : Détectent les vulnérabilités
6. **Tests de configuration** : Valident vos manifestes Kubernetes

## Architecture de l'Environnement de Test

Votre environnement de test automatisé comprendra :

### Composants Principaux

1. **Namespace de test isolé** : Un espace dédié pour exécuter les tests
2. **Pipeline CI/CD** : Orchestration des tests automatiques
3. **Registry de test** : Stockage des images de test
4. **Base de données de test** : Données isolées pour les tests
5. **Outils de test** : Frameworks et bibliothèques de test
6. **Système de reporting** : Collecte et visualisation des résultats

### Stratégie de Test

Nous allons mettre en place une stratégie en plusieurs étapes :

```
Code Push → Build → Tests Unitaires → Build Image →
→ Déploiement Test → Tests Intégration → Tests E2E →
→ Tests Performance → Validation → Nettoyage
```

## Configuration Système Recommandée

Pour un environnement de test confortable :

### Configuration Minimale
- **CPU** : 4 cœurs
- **RAM** : 8 GB
- **Stockage** : 50 GB

### Configuration Recommandée
- **CPU** : 6-8 cœurs
- **RAM** : 12-16 GB
- **Stockage** : 100 GB (SSD)

**Note** : Les tests peuvent consommer beaucoup de ressources temporairement. Prévoyez de la marge.

## Mise en Place - Étape par Étape

### Étape 1 : Création des Namespaces de Test

Créez des namespaces isolés pour différents types de tests :

```bash
# Namespace pour les tests d'intégration
kubectl create namespace testing

# Namespace pour les tests end-to-end
kubectl create namespace e2e-tests

# Namespace pour les tests de performance
kubectl create namespace perf-tests

# Ajouter des labels pour identification
kubectl label namespace testing environment=test
kubectl label namespace e2e-tests environment=test
kubectl label namespace perf-tests environment=test
```

### Étape 2 : Configuration des Quotas de Ressources

Pour éviter que les tests ne monopolisent toutes les ressources :

Créez un fichier `test-quotas.yaml` :

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: test-quota
  namespace: testing
spec:
  hard:
    requests.cpu: "4"
    requests.memory: 8Gi
    limits.cpu: "8"
    limits.memory: 16Gi
    persistentvolumeclaims: "10"
    pods: "50"
---
apiVersion: v1
kind: LimitRange
metadata:
  name: test-limits
  namespace: testing
spec:
  limits:
  - max:
      cpu: "2"
      memory: "2Gi"
    min:
      cpu: "100m"
      memory: "128Mi"
    default:
      cpu: "500m"
      memory: "512Mi"
    defaultRequest:
      cpu: "250m"
      memory: "256Mi"
    type: Container
```

Appliquez la configuration :

```bash
kubectl apply -f test-quotas.yaml
```

### Étape 3 : Déploiement d'une Base de Données de Test

Créez un fichier `test-database.yaml` :

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: postgres-test-pvc
  namespace: testing
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 5Gi
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: postgres-test
  namespace: testing
  labels:
    app: postgres-test
spec:
  replicas: 1
  selector:
    matchLabels:
      app: postgres-test
  template:
    metadata:
      labels:
        app: postgres-test
    spec:
      containers:
      - name: postgres
        image: postgres:15
        env:
        - name: POSTGRES_DB
          value: testdb
        - name: POSTGRES_USER
          value: testuser
        - name: POSTGRES_PASSWORD
          value: testpassword
        - name: POSTGRES_HOST_AUTH_METHOD
          value: trust
        ports:
        - containerPort: 5432
          name: postgres
        volumeMounts:
        - name: postgres-storage
          mountPath: /var/lib/postgresql/data
        resources:
          requests:
            memory: "256Mi"
            cpu: "250m"
          limits:
            memory: "512Mi"
            cpu: "500m"
      volumes:
      - name: postgres-storage
        persistentVolumeClaim:
          claimName: postgres-test-pvc
---
apiVersion: v1
kind: Service
metadata:
  name: postgres-test
  namespace: testing
spec:
  selector:
    app: postgres-test
  ports:
  - protocol: TCP
    port: 5432
    targetPort: 5432
  type: ClusterIP
```

Déployez la base de données :

```bash
kubectl apply -f test-database.yaml

# Vérifier que le pod est prêt
kubectl wait --for=condition=ready pod -l app=postgres-test -n testing --timeout=300s
```

### Étape 4 : ConfigMap pour les Configurations de Test

Créez un fichier `test-config.yaml` :

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: test-config
  namespace: testing
data:
  DATABASE_URL: "postgresql://testuser:testpassword@postgres-test:5432/testdb"
  TEST_ENV: "kubernetes"
  LOG_LEVEL: "debug"
  TEST_TIMEOUT: "300"
  CLEANUP_AFTER_TEST: "true"
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: test-scripts
  namespace: testing
data:
  setup.sh: |
    #!/bin/bash
    echo "Setting up test environment..."
    # Initialiser la base de données
    psql $DATABASE_URL -c "CREATE TABLE IF NOT EXISTS test_data (id SERIAL PRIMARY KEY, name VARCHAR(100));"
    echo "Test environment ready!"

  cleanup.sh: |
    #!/bin/bash
    echo "Cleaning up test environment..."
    # Nettoyer les données de test
    psql $DATABASE_URL -c "TRUNCATE TABLE test_data CASCADE;"
    echo "Cleanup complete!"

  health-check.sh: |
    #!/bin/bash
    # Vérifier que tous les services sont prêts
    curl -f http://my-app:8080/health || exit 1
    psql $DATABASE_URL -c "SELECT 1" || exit 1
    echo "All services healthy!"
```

Appliquez la configuration :

```bash
kubectl apply -f test-config.yaml
```

## Types de Tests et Leur Implémentation

### 1. Tests d'Intégration

Les tests d'intégration vérifient que votre application communique correctement avec ses dépendances (base de données, cache, API externes).

#### Exemple de Job de Test d'Intégration

Créez un fichier `integration-test-job.yaml` :

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: integration-test
  namespace: testing
spec:
  template:
    metadata:
      labels:
        test-type: integration
    spec:
      restartPolicy: Never
      initContainers:
      - name: wait-for-db
        image: postgres:15
        command: ['sh', '-c']
        args:
        - |
          until pg_isready -h postgres-test -p 5432; do
            echo "Waiting for database..."
            sleep 2
          done
          echo "Database is ready!"
      containers:
      - name: test-runner
        image: localhost:32000/my-app:test
        command: ['npm', 'run', 'test:integration']
        env:
        - name: DATABASE_URL
          valueFrom:
            configMapKeyRef:
              name: test-config
              key: DATABASE_URL
        - name: TEST_ENV
          valueFrom:
            configMapKeyRef:
              name: test-config
              key: TEST_ENV
        resources:
          requests:
            memory: "256Mi"
            cpu: "250m"
          limits:
            memory: "512Mi"
            cpu: "500m"
  backoffLimit: 1
  ttlSecondsAfterFinished: 3600
```

Exécutez le test :

```bash
kubectl apply -f integration-test-job.yaml

# Suivre les logs du test
kubectl logs -f -n testing job/integration-test

# Vérifier le résultat
kubectl get job integration-test -n testing
```

### 2. Tests End-to-End (E2E)

Les tests E2E simulent le comportement d'un utilisateur réel en interagissant avec l'application complète.

#### Déploiement d'un Environnement E2E Complet

Créez un fichier `e2e-environment.yaml` :

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app-under-test
  namespace: e2e-tests
spec:
  replicas: 1
  selector:
    matchLabels:
      app: app-under-test
  template:
    metadata:
      labels:
        app: app-under-test
    spec:
      containers:
      - name: app
        image: localhost:32000/my-app:latest
        ports:
        - containerPort: 8080
        env:
        - name: DATABASE_URL
          value: "postgresql://testuser:testpassword@postgres-test.testing:5432/testdb"
        resources:
          requests:
            memory: "128Mi"
            cpu: "100m"
          limits:
            memory: "256Mi"
            cpu: "200m"
---
apiVersion: v1
kind: Service
metadata:
  name: app-under-test
  namespace: e2e-tests
spec:
  selector:
    app: app-under-test
  ports:
  - protocol: TCP
    port: 8080
    targetPort: 8080
---
apiVersion: batch/v1
kind: Job
metadata:
  name: e2e-test
  namespace: e2e-tests
spec:
  template:
    spec:
      restartPolicy: Never
      initContainers:
      - name: wait-for-app
        image: busybox
        command: ['sh', '-c']
        args:
        - |
          until wget -q --spider http://app-under-test:8080/health; do
            echo "Waiting for application..."
            sleep 5
          done
          echo "Application is ready!"
      containers:
      - name: test-runner
        image: cypress/included:12.0.0
        command: ['cypress', 'run']
        workingDir: /e2e
        env:
        - name: CYPRESS_BASE_URL
          value: "http://app-under-test:8080"
        volumeMounts:
        - name: test-specs
          mountPath: /e2e
      volumes:
      - name: test-specs
        configMap:
          name: e2e-test-specs
  backoffLimit: 2
```

### 3. Tests de Performance

Les tests de performance vérifient que votre application peut gérer la charge attendue.

#### Test de Charge avec K6

Créez un fichier `performance-test.yaml` :

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: k6-test-script
  namespace: perf-tests
data:
  load-test.js: |
    import http from 'k6/http';
    import { check, sleep } from 'k6';

    export const options = {
      stages: [
        { duration: '1m', target: 10 },   // Montée progressive
        { duration: '3m', target: 50 },   // Charge normale
        { duration: '1m', target: 100 },  // Pic de charge
        { duration: '2m', target: 50 },   // Retour à la normale
        { duration: '1m', target: 0 },    // Descente
      ],
      thresholds: {
        http_req_duration: ['p(95)<500'],  // 95% des requêtes < 500ms
        http_req_failed: ['rate<0.01'],     // Taux d'erreur < 1%
      },
    };

    export default function () {
      const res = http.get('http://app-under-test.e2e-tests:8080/api/users');
      check(res, {
        'status is 200': (r) => r.status === 200,
        'response time < 500ms': (r) => r.timings.duration < 500,
      });
      sleep(1);
    }
---
apiVersion: batch/v1
kind: Job
metadata:
  name: performance-test
  namespace: perf-tests
spec:
  template:
    spec:
      restartPolicy: Never
      containers:
      - name: k6
        image: grafana/k6:latest
        command: ['k6', 'run', '/scripts/load-test.js']
        volumeMounts:
        - name: test-script
          mountPath: /scripts
        resources:
          requests:
            memory: "512Mi"
            cpu: "500m"
          limits:
            memory: "1Gi"
            cpu: "1000m"
      volumes:
      - name: test-script
        configMap:
          name: k6-test-script
```

Exécutez le test de performance :

```bash
kubectl apply -f performance-test.yaml

# Suivre l'exécution
kubectl logs -f -n perf-tests job/performance-test
```

### 4. Tests de Configuration Kubernetes

Validez vos manifestes Kubernetes avant de les déployer avec des outils comme `kubeval` ou `conftest`.

#### Validation avec Kubeval

Créez un fichier `validate-manifests-job.yaml` :

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: validate-manifests
  namespace: testing
spec:
  template:
    spec:
      restartPolicy: Never
      containers:
      - name: kubeval
        image: garethr/kubeval:latest
        command: ['sh', '-c']
        args:
        - |
          echo "Validating Kubernetes manifests..."
          kubeval /manifests/*.yaml
          if [ $? -eq 0 ]; then
            echo "✅ All manifests are valid!"
          else
            echo "❌ Validation failed!"
            exit 1
          fi
        volumeMounts:
        - name: manifests
          mountPath: /manifests
      volumes:
      - name: manifests
        configMap:
          name: manifests-to-validate
```

### 5. Tests de Sécurité

Scannez vos images pour détecter les vulnérabilités.

#### Scan avec Trivy

Créez un fichier `security-scan-job.yaml` :

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: security-scan
  namespace: testing
spec:
  template:
    spec:
      restartPolicy: Never
      containers:
      - name: trivy
        image: aquasec/trivy:latest
        command: ['trivy']
        args:
        - 'image'
        - '--severity'
        - 'HIGH,CRITICAL'
        - '--exit-code'
        - '1'
        - 'localhost:32000/my-app:latest'
        volumeMounts:
        - name: docker-socket
          mountPath: /var/run/docker.sock
      volumes:
      - name: docker-socket
        hostPath:
          path: /var/run/docker.sock
```

## Pipeline de Test Automatisé

### Structure du Pipeline

Un pipeline complet exécute les tests dans un ordre logique :

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: test-pipeline-script
  namespace: testing
data:
  run-pipeline.sh: |
    #!/bin/bash
    set -e

    echo "🚀 Starting automated test pipeline..."

    # Étape 1 : Validation des manifestes
    echo "📋 Step 1: Validating Kubernetes manifests..."
    kubectl apply -f validate-manifests-job.yaml
    kubectl wait --for=condition=complete job/validate-manifests -n testing --timeout=300s

    # Étape 2 : Scan de sécurité
    echo "🔒 Step 2: Security scanning..."
    kubectl apply -f security-scan-job.yaml
    kubectl wait --for=condition=complete job/security-scan -n testing --timeout=600s

    # Étape 3 : Déploiement de l'environnement de test
    echo "🏗️  Step 3: Deploying test environment..."
    kubectl apply -f e2e-environment.yaml
    kubectl wait --for=condition=ready pod -l app=app-under-test -n e2e-tests --timeout=300s

    # Étape 4 : Tests d'intégration
    echo "🔗 Step 4: Running integration tests..."
    kubectl apply -f integration-test-job.yaml
    kubectl wait --for=condition=complete job/integration-test -n testing --timeout=600s

    # Étape 5 : Tests E2E
    echo "🌐 Step 5: Running E2E tests..."
    kubectl apply -f e2e-environment.yaml
    kubectl wait --for=condition=complete job/e2e-test -n e2e-tests --timeout=900s

    # Étape 6 : Tests de performance
    echo "⚡ Step 6: Running performance tests..."
    kubectl apply -f performance-test.yaml
    kubectl wait --for=condition=complete job/performance-test -n perf-tests --timeout=900s

    # Étape 7 : Nettoyage
    echo "🧹 Step 7: Cleaning up..."
    kubectl delete job --all -n testing
    kubectl delete job --all -n e2e-tests
    kubectl delete job --all -n perf-tests
    kubectl delete deployment --all -n e2e-tests

    echo "✅ Test pipeline completed successfully!"
```

### CronJob pour Tests Périodiques

Pour exécuter automatiquement les tests à intervalles réguliers :

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: nightly-tests
  namespace: testing
spec:
  schedule: "0 2 * * *"  # Tous les jours à 2h du matin
  jobTemplate:
    spec:
      template:
        spec:
          restartPolicy: Never
          containers:
          - name: test-runner
            image: bitnami/kubectl:latest
            command: ['sh', '-c']
            args:
            - |
              echo "Starting nightly test run..."
              kubectl apply -f /tests/integration-test-job.yaml
              kubectl wait --for=condition=complete job/integration-test -n testing --timeout=600s
              echo "Nightly tests completed!"
            volumeMounts:
            - name: test-manifests
              mountPath: /tests
          volumes:
          - name: test-manifests
            configMap:
              name: test-manifests
```

## Intégration avec GitLab CI/CD

Voici un exemple de fichier `.gitlab-ci.yml` pour automatiser les tests :

```yaml
stages:
  - build
  - test
  - deploy

variables:
  IMAGE_TAG: $CI_COMMIT_SHORT_SHA
  REGISTRY: localhost:32000

build:
  stage: build
  script:
    - docker build -t $REGISTRY/my-app:$IMAGE_TAG .
    - docker push $REGISTRY/my-app:$IMAGE_TAG

test-integration:
  stage: test
  script:
    - kubectl config use-context microk8s
    - |
      cat <<EOF | kubectl apply -f -
      apiVersion: batch/v1
      kind: Job
      metadata:
        name: integration-test-$CI_COMMIT_SHORT_SHA
        namespace: testing
      spec:
        template:
          spec:
            restartPolicy: Never
            containers:
            - name: test-runner
              image: $REGISTRY/my-app:$IMAGE_TAG
              command: ['npm', 'run', 'test:integration']
      EOF
    - kubectl wait --for=condition=complete job/integration-test-$CI_COMMIT_SHORT_SHA -n testing --timeout=600s
    - kubectl logs job/integration-test-$CI_COMMIT_SHORT_SHA -n testing
  after_script:
    - kubectl delete job integration-test-$CI_COMMIT_SHORT_SHA -n testing

test-e2e:
  stage: test
  script:
    - kubectl apply -f k8s/e2e-environment.yaml
    - kubectl wait --for=condition=ready pod -l app=app-under-test -n e2e-tests --timeout=300s
    - kubectl apply -f k8s/e2e-test-job.yaml
    - kubectl wait --for=condition=complete job/e2e-test -n e2e-tests --timeout=900s
    - kubectl logs job/e2e-test -n e2e-tests
  after_script:
    - kubectl delete -f k8s/e2e-environment.yaml

deploy-staging:
  stage: deploy
  script:
    - kubectl set image deployment/my-app my-app=$REGISTRY/my-app:$IMAGE_TAG -n staging
    - kubectl rollout status deployment/my-app -n staging
  only:
    - main
```

## Intégration avec GitHub Actions

Exemple de workflow GitHub Actions (`.github/workflows/test.yml`) :

```yaml
name: Automated Tests

on:
  push:
    branches: [ main, develop ]
  pull_request:
    branches: [ main ]

jobs:
  test:
    runs-on: ubuntu-latest

    steps:
    - uses: actions/checkout@v3

    - name: Setup MicroK8s
      run: |
        sudo snap install microk8s --classic
        sudo microk8s status --wait-ready
        sudo microk8s enable dns registry storage

    - name: Build and Push Image
      run: |
        docker build -t localhost:32000/my-app:${{ github.sha }} .
        docker push localhost:32000/my-app:${{ github.sha }}

    - name: Run Integration Tests
      run: |
        kubectl apply -f k8s/test-database.yaml
        kubectl wait --for=condition=ready pod -l app=postgres-test -n testing --timeout=300s

        kubectl apply -f k8s/integration-test-job.yaml
        kubectl wait --for=condition=complete job/integration-test -n testing --timeout=600s

        kubectl logs job/integration-test -n testing

    - name: Run E2E Tests
      run: |
        kubectl apply -f k8s/e2e-environment.yaml
        kubectl wait --for=condition=ready pod -l app=app-under-test -n e2e-tests --timeout=300s

        kubectl apply -f k8s/e2e-test-job.yaml
        kubectl wait --for=condition=complete job/e2e-test -n e2e-tests --timeout=900s

    - name: Cleanup
      if: always()
      run: |
        kubectl delete job --all -n testing
        kubectl delete job --all -n e2e-tests
        kubectl delete deployment --all -n e2e-tests
```

## Collecte et Visualisation des Résultats

### Stockage des Résultats de Test

Créez un PVC pour stocker les résultats :

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: test-results-pvc
  namespace: testing
spec:
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 10Gi
```

### Job avec Export des Résultats

Modifiez vos jobs pour sauvegarder les résultats :

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: integration-test-with-results
  namespace: testing
spec:
  template:
    spec:
      restartPolicy: Never
      containers:
      - name: test-runner
        image: localhost:32000/my-app:test
        command: ['sh', '-c']
        args:
        - |
          npm run test:integration -- --reporter=json > /results/test-results.json
          npm run test:integration -- --reporter=html > /results/test-results.html
        volumeMounts:
        - name: test-results
          mountPath: /results
      volumes:
      - name: test-results
        persistentVolumeClaim:
          claimName: test-results-pvc
```

### Serveur Web pour Consulter les Résultats

Déployez un serveur nginx pour visualiser les résultats :

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: test-results-viewer
  namespace: testing
spec:
  replicas: 1
  selector:
    matchLabels:
      app: test-results-viewer
  template:
    metadata:
      labels:
        app: test-results-viewer
    spec:
      containers:
      - name: nginx
        image: nginx:alpine
        ports:
        - containerPort: 80
        volumeMounts:
        - name: test-results
          mountPath: /usr/share/nginx/html
          readOnly: true
      volumes:
      - name: test-results
        persistentVolumeClaim:
          claimName: test-results-pvc
---
apiVersion: v1
kind: Service
metadata:
  name: test-results-viewer
  namespace: testing
spec:
  selector:
    app: test-results-viewer
  ports:
  - protocol: TCP
    port: 80
    targetPort: 80
  type: LoadBalancer
```

Accédez aux résultats :

```bash
# Obtenir l'IP du service
kubectl get svc test-results-viewer -n testing

# Ou utilisez port-forward
kubectl port-forward -n testing svc/test-results-viewer 8080:80
# Accédez à http://localhost:8080
```

## Notifications de Résultats de Test

### Notification Slack

Créez un Job qui envoie des notifications :

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: notify-test-results
  namespace: testing
spec:
  template:
    spec:
      restartPolicy: Never
      containers:
      - name: notifier
        image: curlimages/curl:latest
        command: ['sh', '-c']
        args:
        - |
          if kubectl get job integration-test -n testing -o jsonpath='{.status.succeeded}' | grep -q "1"; then
            STATUS="✅ SUCCESS"
            COLOR="good"
          else
            STATUS="❌ FAILED"
            COLOR="danger"
          fi

          curl -X POST -H 'Content-type: application/json' \
            --data "{\"text\":\"Test Pipeline: $STATUS\",\"attachments\":[{\"color\":\"$COLOR\",\"text\":\"Integration tests completed\"}]}" \
            $SLACK_WEBHOOK_URL
        env:
        - name: SLACK_WEBHOOK_URL
          valueFrom:
            secretKeyRef:
              name: slack-webhook
              key: url
```

Créez le secret pour le webhook Slack :

```bash
kubectl create secret generic slack-webhook \
  --from-literal=url='https://hooks.slack.com/services/YOUR/WEBHOOK/URL' \
  -n testing
```

## Bonnes Pratiques pour les Tests Automatisés

### 1. Isolation des Tests

**Utilisez des namespaces séparés** : Chaque type de test dans son propre namespace.

**Données de test isolées** : Utilisez des bases de données et des ressources dédiées aux tests.

**Nettoyage automatique** : Supprimez les ressources après chaque test.

### 2. Tests Rapides et Fiables

**Parallélisation** : Exécutez les tests en parallèle quand c'est possible.

**Timeouts appropriés** : Définissez des timeouts réalistes mais pas trop longs.

**Tests déterministes** : Évitez les tests qui dépendent du timing ou de l'ordre d'exécution.

### 3. Gestion des Échecs

**Retry automatique** : Configurez `backoffLimit` pour les tests flaky.

**Logs détaillés** : Capturez suffisamment d'informations pour déboguer.

**Fail fast** : Arrêtez le pipeline dès qu'un test critique échoue.

Exemple avec retry :

```yaml
spec:
  backoffLimit: 3  # Réessayer jusqu'à 3 fois
  activeDeadlineSeconds: 600  # Timeout global de 10 minutes
```

### 4. Optimisation des Ressources

**Limites de ressources** : Définissez toujours requests et limits.

**TTL pour les Jobs** : Nettoyez automatiquement les anciens jobs.

```yaml
spec:
  ttlSecondsAfterFinished: 3600  # Supprimé après 1 heure
```

**Réutilisation des images** : Utilisez un cache d'images pour accélérer les builds.

### 5. Documentation des Tests

Documentez vos tests avec des annotations :

```yaml
metadata:
  annotations:
    description: "Integration tests for user authentication"
    test-type: "integration"
    estimated-duration: "5m"
    owner: "backend-team"
    documentation: "https://wiki.company.com/tests/auth"
```

## Scripts Utiles pour la Gestion des Tests

### Script de Lancement des Tests

Créez un fichier `run-tests.sh` :

```bash
#!/bin/bash

TEST_TYPE=${1:-all}
NAMESPACE=${2:-testing}

echo "🧪 Running tests: $TEST_TYPE in namespace: $NAMESPACE"

run_integration_tests() {
    echo "Running integration tests..."
    kubectl apply -f k8s/integration-test-job.yaml
    kubectl wait --for=condition=complete job/integration-test -n $NAMESPACE --timeout=600s
    kubectl logs job/integration-test -n $NAMESPACE
}

run_e2e_tests() {
    echo "Running E2E tests..."
    kubectl apply -f k8s/e2e-environment.yaml
    kubectl wait --for=condition=ready pod -l app=app-under-test -n e2e-tests --timeout=300s
    kubectl apply -f k8s/e2e-test-job.yaml
    kubectl wait --for=condition=complete job/e2e-test -n e2e-tests --timeout=900s
    kubectl logs job/e2e-test -n e2e-tests
}

run_performance_tests() {
    echo "Running performance tests..."
    kubectl apply -f k8s/performance-test.yaml
    kubectl wait --for=condition=complete job/performance-test -n perf-tests --timeout=900s
    kubectl logs job/performance-test -n perf-tests
}

cleanup() {
    echo "Cleaning up test resources..."
    kubectl delete job --all -n testing
    kubectl delete job --all -n e2e-tests
    kubectl delete job --all -n perf-tests
    kubectl delete deployment --all -n e2e-tests
}

case $TEST_TYPE in
    integration)
        run_integration_tests
        ;;
    e2e)
        run_e2e_tests
        ;;
    performance)
        run_performance_tests
        ;;
    all)
        run_integration_tests
        run_e2e_tests
        run_performance_tests
        ;;
    cleanup)
        cleanup
        ;;
    *)
        echo "Unknown test type: $TEST_TYPE"
        echo "Usage: $0 {integration|e2e|performance|all|cleanup} [namespace]"
        exit 1
        ;;
esac

echo "✅ Test execution completed!"
```

Rendez-le exécutable :

```bash
chmod +x run-tests.sh

# Exemples d'utilisation
./run-tests.sh integration          # Tests d'intégration seulement
./run-tests.sh e2e                   # Tests E2E seulement
./run-tests.sh all                   # Tous les tests
./run-tests.sh cleanup               # Nettoyage
```

### Script de Vérification des Résultats

Créez un fichier `check-test-results.sh` :

```bash
#!/bin/bash

NAMESPACE=${1:-testing}

echo "📊 Checking test results in namespace: $NAMESPACE"

# Fonction pour vérifier le statut d'un job
check_job() {
    local job_name=$1
    local namespace=$2

    if kubectl get job $job_name -n $namespace &> /dev/null; then
        succeeded=$(kubectl get job $job_name -n $namespace -o jsonpath='{.status.succeeded}')
        failed=$(kubectl get job $job_name -n $namespace -o jsonpath='{.status.failed}')

        if [ "$succeeded" == "1" ]; then
            echo "✅ $job_name: SUCCESS"
            return 0
        elif [ "$failed" != "" ] && [ "$failed" != "0" ]; then
            echo "❌ $job_name: FAILED"
            return 1
        else
            echo "⏳ $job_name: RUNNING"
            return 2
        fi
    else
        echo "❓ $job_name: NOT FOUND"
        return 3
    fi
}

# Vérifier tous les jobs de test
echo ""
echo "Integration Tests:"
check_job "integration-test" "$NAMESPACE"

echo ""
echo "E2E Tests:"
check_job "e2e-test" "e2e-tests"

echo ""
echo "Performance Tests:"
check_job "performance-test" "perf-tests"

echo ""
echo "Security Scan:"
check_job "security-scan" "$NAMESPACE"

# Résumé
echo ""
echo "================================================"
total_jobs=$(kubectl get jobs --all-namespaces -l test-type --no-headers | wc -l)
succeeded_jobs=$(kubectl get jobs --all-namespaces -l test-type -o jsonpath='{range .items[*]}{.status.succeeded}{"\n"}{end}' | grep -c "1")
failed_jobs=$(kubectl get jobs --all-namespaces -l test-type -o jsonpath='{range .items[*]}{.status.failed}{"\n"}{end}' | grep -c "1")

echo "Total test jobs: $total_jobs"
echo "Succeeded: $succeeded_jobs"
echo "Failed: $failed_jobs"
echo "================================================"
```

## Monitoring des Tests

### Métriques Prometheus pour les Tests

Créez des ServiceMonitors pour collecter les métriques de test :

```yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: test-metrics
  namespace: testing
spec:
  selector:
    matchLabels:
      app: test-runner
  endpoints:
  - port: metrics
    interval: 30s
    path: /metrics
```

### Dashboard Grafana pour les Tests

Créez un ConfigMap avec un dashboard dédié aux tests :

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: test-dashboard
  namespace: monitoring
data:
  test-dashboard.json: |
    {
      "dashboard": {
        "title": "Test Automation Dashboard",
        "panels": [
          {
            "title": "Test Success Rate",
            "targets": [
              {
                "expr": "sum(rate(test_runs_total{result=\"success\"}[5m])) / sum(rate(test_runs_total[5m])) * 100"
              }
            ]
          },
          {
            "title": "Test Duration",
            "targets": [
              {
                "expr": "histogram_quantile(0.95, rate(test_duration_seconds_bucket[5m]))"
              }
            ]
          },
          {
            "title": "Failed Tests",
            "targets": [
              {
                "expr": "sum(rate(test_runs_total{result=\"failed\"}[5m]))"
              }
            ]
          }
        ]
      }
    }
```

## Dépannage des Tests

### Problème : Les Tests Échouent Aléatoirement

**Causes possibles** :
- Tests flaky (non déterministes)
- Ressources insuffisantes
- Timeouts trop courts
- Problèmes de réseau

**Solutions** :

```bash
# Augmenter les ressources
kubectl set resources deployment/test-runner \
  --limits=cpu=1000m,memory=1Gi \
  --requests=cpu=500m,memory=512Mi \
  -n testing

# Augmenter les timeouts
kubectl patch job integration-test -n testing \
  --type='json' \
  -p='[{"op": "replace", "path": "/spec/activeDeadlineSeconds", "value": 900}]'

# Activer le retry automatique
kubectl patch job integration-test -n testing \
  --type='json' \
  -p='[{"op": "replace", "path": "/spec/backoffLimit", "value": 3}]'
```

### Problème : Les Tests Sont Trop Lents

**Solutions** :

1. **Parallélisation** : Exécutez plusieurs jobs en parallèle

```yaml
spec:
  parallelism: 3  # Exécuter 3 pods de test en parallèle
  completions: 10  # 10 tests à exécuter au total
```

2. **Optimisation des images** : Utilisez des images plus légères

```dockerfile
# Au lieu de
FROM node:18
# Utilisez
FROM node:18-alpine
```

3. **Cache des dépendances** : Pré-construisez une image avec les dépendances

### Problème : Nettoyage Incomplet

Si les ressources de test ne sont pas nettoyées correctement :

```bash
# Script de nettoyage forcé
kubectl delete jobs --all -n testing --force --grace-period=0
kubectl delete pods --field-selector status.phase=Failed -n testing
kubectl delete pods --field-selector status.phase=Succeeded -n testing

# Nettoyer les PVC non utilisés
kubectl get pvc -n testing --no-headers | awk '{print $1}' | xargs kubectl delete pvc -n testing
```

## Stratégies Avancées

### Test en Production (Canary Testing)

Déployez une version canary et testez-la en production avec du vrai trafic :

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app-canary
  namespace: production
spec:
  replicas: 1  # Seulement 10% du trafic
  selector:
    matchLabels:
      app: my-app
      version: canary
  template:
    metadata:
      labels:
        app: my-app
        version: canary
    spec:
      containers:
      - name: app
        image: localhost:32000/my-app:canary
```

### Smoke Tests en Production

Tests légers qui vérifient que l'application fonctionne après un déploiement :

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: smoke-test
  namespace: production
spec:
  template:
    spec:
      restartPolicy: Never
      containers:
      - name: smoke-test
        image: curlimages/curl:latest
        command: ['sh', '-c']
        args:
        - |
          # Vérifier la page d'accueil
          curl -f https://myapp.com || exit 1

          # Vérifier l'API
          curl -f https://myapp.com/api/health || exit 1

          # Vérifier un endpoint critique
          curl -f https://myapp.com/api/users/1 || exit 1

          echo "✅ Smoke tests passed!"
```

## Conclusion

Vous disposez maintenant d'un environnement de test automatisé complet avec MicroK8s. Cet environnement vous permet de :

✅ Exécuter automatiquement différents types de tests
✅ Détecter les problèmes avant la production
✅ Améliorer la qualité et la fiabilité de vos applications
✅ Accélérer le cycle de développement
✅ Gagner en confiance lors des déploiements

**Points clés à retenir** :

- **Isolez vos tests** dans des namespaces dédiés
- **Automatisez tout** : build, test, déploiement, nettoyage
- **Surveillez les résultats** avec des dashboards et des alertes
- **Optimisez les ressources** pour des tests rapides
- **Documentez vos tests** pour faciliter la maintenance
- **Intégrez avec CI/CD** pour des tests continus

**Prochaines étapes** : Explorez les autres chapitres pour découvrir d'autres cas d'usage pratiques avec MicroK8s et approfondir votre maîtrise de Kubernetes.

**Bons tests ! 🧪**

⏭️ [Hébergement de services personnels](/24-cas-dusage-pratiques/03-hebergement-de-services-personnels.md)
