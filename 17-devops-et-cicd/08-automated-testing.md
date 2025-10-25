🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 17.8 Automated testing

## Introduction

Vous avez appris à créer des applications Kubernetes, à les packager avec Helm, et à les déployer automatiquement avec des pipelines CI/CD. Mais comment vous assurer que ce qui est déployé **fonctionne réellement** ? Comment garantir qu'une modification ne casse pas quelque chose qui marchait auparavant ?

C'est là qu'interviennent les **tests automatisés**. Dans ce chapitre, nous allons explorer les différents types de tests applicables à Kubernetes, les outils disponibles, et comment intégrer ces tests dans votre workflow DevOps pour déployer avec confiance.

Les tests automatisés sont votre **filet de sécurité** : ils détectent les problèmes avant qu'ils n'atteignent la production, réduisent le stress des déploiements, et permettent d'itérer rapidement en toute sérénité.

---

## Pourquoi tester dans Kubernetes ?

### Les défis spécifiques à Kubernetes

Kubernetes ajoute des couches de complexité par rapport aux applications traditionnelles :

**Manifestes YAML complexes** :
- Syntaxe YAML fragile (indentation, types)
- Nombreuses ressources interconnectées
- Configurations imbriquées

**Comportements dynamiques** :
- Orchestration automatique
- Scaling et self-healing
- Networking complexe

**Environnements multiples** :
- Dev, staging, production
- Différentes configurations
- Variations de versions

### Ce qui peut mal tourner

Sans tests, des problèmes courants passent inaperçus :

❌ **Erreurs de syntaxe YAML** :
```yaml
# Indentation incorrecte
spec:
containers:  # Manque 2 espaces
- name: app
```

❌ **Références invalides** :
```yaml
# ConfigMap référencé qui n'existe pas
envFrom:
- configMapRef:
    name: config-typo  # Devrait être "config"
```

❌ **Limites de ressources inadéquates** :
```yaml
# Trop peu de mémoire
resources:
  limits:
    memory: 64Mi  # L'app a besoin de 256Mi minimum
```

❌ **Problèmes de permissions RBAC** :
```yaml
# ServiceAccount sans les droits nécessaires
apiVersion: v1
kind: ServiceAccount
# Mais pas de Role/RoleBinding associé
```

❌ **Incompatibilités de versions** :
```yaml
# API dépréciée
apiVersion: extensions/v1beta1  # Supprimée dans K8s 1.22+
kind: Ingress
```

### Bénéfices des tests automatisés

✅ **Détection précoce** : Les bugs sont trouvés avant le déploiement

✅ **Confiance** : Déployer sans stress, sachant que tout a été validé

✅ **Refactoring sûr** : Modifier le code sans crainte de casser quelque chose

✅ **Documentation vivante** : Les tests documentent le comportement attendu

✅ **Qualité** : Maintenir un haut niveau de qualité dans le temps

✅ **Rapidité** : Feedback immédiat sur les changements

---

## La pyramide des tests pour Kubernetes

### Vue d'ensemble

Comme pour le développement logiciel traditionnel, les tests Kubernetes suivent une pyramide :

```
                    ┌─────────────┐
                    │   E2E Tests │  Moins nombreux
                    │   (Lents)   │  Plus réalistes
                    └─────────────┘
                 ┌──────────────────┐
                 │ Integration Tests│  Moyennement nombreux
                 │    (Moyens)      │  Vérifient l'interaction
                 └──────────────────┘
            ┌──────────────────────────┐
            │    Contract Tests        │
            │  (Validation manifestes) │
            └──────────────────────────┘
       ┌──────────────────────────────────┐
       │         Unit Tests               │  Très nombreux
       │     (Validation YAML)            │  Très rapides
       └──────────────────────────────────┘
```

### Types de tests

**1. Unit Tests (Tests unitaires)** :
- Validation de la syntaxe YAML
- Vérification de la structure des manifestes
- Tests des templates Helm
- Très rapides (millisecondes)

**2. Contract Tests (Tests de contrat)** :
- Validation contre le schéma Kubernetes
- Vérification des API versions
- Conformité aux bonnes pratiques
- Rapides (secondes)

**3. Integration Tests (Tests d'intégration)** :
- Déploiement sur un cluster de test
- Vérification des interactions entre composants
- Tests de bout en bout limités
- Moyennement rapides (minutes)

**4. End-to-End Tests (Tests E2E)** :
- Tests complets sur un environnement similaire à la production
- Scénarios utilisateur complets
- Tests de charge et de performance
- Lents (dizaines de minutes)

---

## Tests de validation YAML

### Validation de la syntaxe YAML

Le premier niveau de test : s'assurer que vos YAML sont valides.

#### Avec yamllint

**Installation** :

```bash
# Ubuntu/Debian
sudo apt-get install yamllint

# Avec pip
pip install yamllint

# macOS
brew install yamllint
```

**Utilisation** :

```bash
# Valider un fichier
yamllint deployment.yaml

# Valider un dossier
yamllint k8s/

# Avec configuration personnalisée
yamllint -c .yamllint.yaml k8s/
```

**Configuration .yamllint.yaml** :

```yaml
extends: default

rules:
  line-length:
    max: 120
    level: warning
  indentation:
    spaces: 2
    indent-sequences: true
  comments:
    min-spaces-from-content: 1
  document-start: disable
  truthy:
    allowed-values: ['true', 'false', 'yes', 'no']
```

#### Avec yq (pour les requêtes complexes)

**Installation** :

```bash
# Linux
wget https://github.com/mikefarah/yq/releases/latest/download/yq_linux_amd64 -O yq
chmod +x yq
sudo mv yq /usr/local/bin/

# macOS
brew install yq
```

**Exemples d'utilisation** :

```bash
# Extraire des valeurs
yq '.spec.replicas' deployment.yaml

# Valider la présence de champs obligatoires
if [ $(yq '.spec.template.spec.containers[0].resources' deployment.yaml) == "null" ]; then
  echo "❌ Resources limits manquantes"
  exit 1
fi

# Vérifier les versions d'images
yq '.spec.template.spec.containers[].image' deployment.yaml | grep -v latest || echo "✅ Pas de tag latest"
```

---

## Tests de conformité Kubernetes

### Avec kubeval

**kubeval** valide vos manifestes contre les schémas Kubernetes officiels.

**Installation** :

```bash
# Linux
wget https://github.com/instrumenta/kubeval/releases/latest/download/kubeval-linux-amd64.tar.gz
tar xf kubeval-linux-amd64.tar.gz
sudo mv kubeval /usr/local/bin/

# macOS
brew install kubeval
```

**Utilisation** :

```bash
# Valider un fichier
kubeval deployment.yaml

# Valider un dossier
kubeval k8s/*.yaml

# Spécifier la version de Kubernetes
kubeval --kubernetes-version 1.28.0 deployment.yaml

# Ignorer les CRDs (Custom Resource Definitions)
kubeval --ignore-missing-schemas deployment.yaml

# Format de sortie JSON
kubeval --output json deployment.yaml
```

**Exemple de sortie** :

```
PASS - deployment.yaml contains a valid Deployment (web-app)
WARN - service.yaml contains an invalid Service (web-service) - spec.type: unsupported value "LoadBalanser" (must be one of ClusterIP, NodePort, LoadBalancer, ExternalName)
```

### Avec kubeconform

**kubeconform** est une alternative plus moderne et rapide à kubeval.

**Installation** :

```bash
# Linux
wget https://github.com/yannh/kubeconform/releases/latest/download/kubeconform-linux-amd64.tar.gz
tar xf kubeconform-linux-amd64.tar.gz
sudo mv kubeconform /usr/local/bin/

# macOS
brew install kubeconform
```

**Utilisation** :

```bash
# Valider des fichiers
kubeconform k8s/*.yaml

# Avec plusieurs threads (plus rapide)
kubeconform -n 8 k8s/*.yaml

# Strict mode (échoue sur warnings)
kubeconform -strict k8s/*.yaml

# Version Kubernetes spécifique
kubeconform -kubernetes-version 1.28.0 k8s/*.yaml
```

### Avec kubectl dry-run

La validation la plus fiable : demander au cluster Kubernetes lui-même.

**Validation client** (sans accès au cluster) :

```bash
# Syntaxe de base
kubectl apply --dry-run=client -f deployment.yaml

# Pour tous les fichiers
kubectl apply --dry-run=client -f k8s/

# Voir ce qui serait créé
kubectl apply --dry-run=client -f k8s/ -o yaml
```

**Validation serveur** (nécessite un cluster) :

```bash
# Validation complète incluant admission controllers
kubectl apply --dry-run=server -f deployment.yaml

# Vérifie aussi les quotas, RBAC, policies
kubectl apply --dry-run=server -f k8s/
```

---

## Tests des Helm Charts

### Tests Helm natifs

Helm intègre un système de tests basé sur des pods.

**Créer un test** :

```yaml
# templates/tests/test-connection.yaml
apiVersion: v1
kind: Pod
metadata:
  name: "{{ include "mon-app.fullname" . }}-test-connection"
  labels:
    {{- include "mon-app.labels" . | nindent 4 }}
  annotations:
    "helm.sh/hook": test
    "helm.sh/hook-delete-policy": hook-succeeded
spec:
  containers:
  - name: wget
    image: busybox
    command: ['wget']
    args: ['{{ include "mon-app.fullname" . }}:{{ .Values.service.port }}']
  restartPolicy: Never
```

**Exécuter les tests** :

```bash
# Installer le chart
helm install my-app ./mon-app

# Lancer les tests
helm test my-app

# Avec logs détaillés
helm test my-app --logs
```

### Tests avec helm unittest

**helm-unittest** permet de tester les templates Helm sans cluster.

**Installation** :

```bash
helm plugin install https://github.com/helm-unittest/helm-unittest
```

**Créer des tests** :

```yaml
# tests/deployment_test.yaml
suite: test deployment
templates:
  - deployment.yaml
tests:
  - it: should create a deployment with correct name
    asserts:
      - isKind:
          of: Deployment
      - equal:
          path: metadata.name
          value: RELEASE-NAME-mon-app

  - it: should use correct image
    set:
      image.repository: nginx
      image.tag: "1.25"
    asserts:
      - equal:
          path: spec.template.spec.containers[0].image
          value: nginx:1.25

  - it: should have correct replica count
    set:
      replicaCount: 3
    asserts:
      - equal:
          path: spec.replicas
          value: 3

  - it: should set resources when provided
    set:
      resources:
        limits:
          cpu: 500m
          memory: 512Mi
    asserts:
      - equal:
          path: spec.template.spec.containers[0].resources.limits.cpu
          value: 500m

  - it: should not create ingress when disabled
    set:
      ingress.enabled: false
    template: ingress.yaml
    asserts:
      - hasDocuments:
          count: 0
```

**Exécuter les tests** :

```bash
# Tous les tests
helm unittest ./mon-app

# Tests spécifiques
helm unittest -f 'tests/deployment_test.yaml' ./mon-app

# Avec code coverage
helm unittest -c ./mon-app

# Format JUnit pour CI/CD
helm unittest -t junit -o test-results.xml ./mon-app
```

### Validation avec helm lint

```bash
# Linter le chart
helm lint ./mon-app

# Strict mode
helm lint --strict ./mon-app

# Avec des values personnalisées
helm lint ./mon-app -f values-production.yaml
```

---

## Tests de sécurité

### Avec Trivy

**Trivy** scanne les vulnérabilités dans les images et les configurations.

**Installation** :

```bash
# Linux
wget https://github.com/aquasecurity/trivy/releases/latest/download/trivy_Linux-64bit.tar.gz
tar zxvf trivy_Linux-64bit.tar.gz
sudo mv trivy /usr/local/bin/

# macOS
brew install trivy
```

**Scanner les images** :

```bash
# Scanner une image
trivy image nginx:1.25

# Seulement les vulnérabilités critiques et high
trivy image --severity HIGH,CRITICAL nginx:1.25

# Format JSON
trivy image -f json -o results.json nginx:1.25
```

**Scanner les manifestes Kubernetes** :

```bash
# Scanner les configurations
trivy config k8s/

# Scanner avec des policies spécifiques
trivy config --policy policy/ k8s/
```

### Avec Checkov

**Checkov** analyse les configurations pour détecter les problèmes de sécurité.

**Installation** :

```bash
pip install checkov
```

**Utilisation** :

```bash
# Scanner un dossier
checkov -d k8s/

# Format de sortie
checkov -d k8s/ -o json
checkov -d k8s/ -o junit-xml

# Ignorer certains checks
checkov -d k8s/ --skip-check CKV_K8S_8,CKV_K8S_9

# Seulement certains checks
checkov -d k8s/ --check CKV_K8S_*
```

**Exemples de checks** :
- `CKV_K8S_8` : Vérifie les limites de CPU
- `CKV_K8S_9` : Vérifie les limites de mémoire
- `CKV_K8S_10` : Vérifie runAsNonRoot
- `CKV_K8S_11` : Vérifie readOnlyRootFilesystem
- `CKV_K8S_12` : Vérifie allowPrivilegeEscalation

### Avec kubesec

**kubesec** évalue la sécurité des configurations Kubernetes.

**Installation** :

```bash
# Via Docker
docker pull kubesec/kubesec:v2

# Ou téléchargement direct
wget https://github.com/controlplaneio/kubesec/releases/latest/download/kubesec_linux_amd64
chmod +x kubesec_linux_amd64
sudo mv kubesec_linux_amd64 /usr/local/bin/kubesec
```

**Utilisation** :

```bash
# Scanner un fichier
kubesec scan deployment.yaml

# Via stdin
cat deployment.yaml | kubesec scan -

# Via Docker
docker run -i kubesec/kubesec:v2 scan /dev/stdin < deployment.yaml
```

**Exemple de sortie** :

```json
[
  {
    "object": "Deployment/web-app",
    "valid": true,
    "message": "Passed with a score of 7 points",
    "score": 7,
    "scoring": {
      "advise": [
        {
          "selector": "containers[] .securityContext .runAsNonRoot == true",
          "reason": "Force running as non-root",
          "points": 1
        }
      ]
    }
  }
]
```

### Avec Polaris

**Polaris** audite les clusters pour les bonnes pratiques.

**Installation** :

```bash
# Via Helm
helm repo add fairwinds-stable https://charts.fairwinds.com/stable
helm install polaris fairwinds-stable/polaris --namespace polaris --create-namespace

# CLI
wget https://github.com/FairwindsOps/polaris/releases/latest/download/polaris_linux_amd64.tar.gz
tar -xzf polaris_linux_amd64.tar.gz
sudo mv polaris /usr/local/bin/
```

**Utilisation** :

```bash
# Auditer un cluster (dashboard)
polaris dashboard --port 8080

# Auditer des fichiers locaux
polaris audit --audit-path k8s/

# Format JSON
polaris audit --audit-path k8s/ --format json
```

---

## Tests d'intégration

### Avec Kind (Kubernetes in Docker)

**Kind** permet de créer des clusters Kubernetes éphémères pour les tests.

**Installation** :

```bash
# Linux
curl -Lo ./kind https://kind.sigs.k8s.io/dl/latest/kind-linux-amd64
chmod +x ./kind
sudo mv ./kind /usr/local/bin/kind

# macOS
brew install kind
```

**Créer un cluster de test** :

```bash
# Cluster simple
kind create cluster --name test

# Avec configuration
cat <<EOF | kind create cluster --name test --config=-
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
- role: control-plane
- role: worker
- role: worker
EOF

# Vérifier
kubectl cluster-info --context kind-test
```

**Script de test d'intégration** :

```bash
#!/bin/bash
# test-integration.sh

set -e

echo "🚀 Création du cluster de test..."
kind create cluster --name test-cluster

echo "📦 Déploiement de l'application..."
kubectl apply -f k8s/

echo "⏳ Attente que les pods soient prêts..."
kubectl wait --for=condition=ready pod -l app=mon-app --timeout=300s

echo "🧪 Tests de connectivité..."
kubectl run test-pod --rm -i --restart=Never --image=curlimages/curl -- \
  curl -f http://mon-app-service/health

echo "✅ Tests réussis!"

echo "🧹 Nettoyage..."
kind delete cluster --name test-cluster
```

### Avec k3d (K3s in Docker)

**k3d** est une alternative légère basée sur K3s.

**Installation** :

```bash
curl -s https://raw.githubusercontent.com/k3d-io/k3d/main/install.sh | bash
```

**Utilisation** :

```bash
# Créer un cluster
k3d cluster create test-cluster

# Avec registry local
k3d cluster create test-cluster --registry-create test-registry:0.0.0.0:5000

# Déployer et tester
kubectl apply -f k8s/
kubectl wait --for=condition=ready pod -l app=mon-app --timeout=300s

# Nettoyer
k3d cluster delete test-cluster
```

### Tests avec Bats (Bash Automated Testing System)

**Installation** :

```bash
# Ubuntu/Debian
sudo apt-get install bats

# macOS
brew install bats-core
```

**Créer des tests** :

```bash
# tests/integration.bats
#!/usr/bin/env bats

setup() {
  kubectl create namespace test-ns
  kubectl apply -f k8s/ -n test-ns
}

teardown() {
  kubectl delete namespace test-ns
}

@test "deployment should be created" {
  run kubectl get deployment mon-app -n test-ns
  [ "$status" -eq 0 ]
}

@test "service should be created" {
  run kubectl get service mon-app -n test-ns
  [ "$status" -eq 0 ]
}

@test "pods should become ready" {
  run kubectl wait --for=condition=ready pod -l app=mon-app -n test-ns --timeout=300s
  [ "$status" -eq 0 ]
}

@test "application should respond to health check" {
  POD=$(kubectl get pod -l app=mon-app -n test-ns -o jsonpath='{.items[0].metadata.name}')
  run kubectl exec -n test-ns $POD -- wget -qO- http://localhost:8080/health
  [ "$status" -eq 0 ]
  [[ "$output" == *"healthy"* ]]
}
```

**Exécuter les tests** :

```bash
bats tests/integration.bats
```

---

## Tests de performance et de charge

### Avec k6

**k6** est un outil moderne de test de charge.

**Installation** :

```bash
# Linux
sudo gpg -k
sudo gpg --no-default-keyring --keyring /usr/share/keyrings/k6-archive-keyring.gpg --keyserver hkp://keyserver.ubuntu.com:80 --recv-keys C5AD17C747E3415A3642D57D77C6C491D6AC1D69
echo "deb [signed-by=/usr/share/keyrings/k6-archive-keyring.gpg] https://dl.k6.io/deb stable main" | sudo tee /etc/apt/sources.list.d/k6.list
sudo apt-get update
sudo apt-get install k6

# macOS
brew install k6
```

**Script de test** :

```javascript
// load-test.js
import http from 'k6/http';
import { check, sleep } from 'k6';

export const options = {
  stages: [
    { duration: '30s', target: 10 },  // Ramp up
    { duration: '1m', target: 50 },   // Stay at 50 users
    { duration: '30s', target: 100 }, // Spike
    { duration: '1m', target: 50 },   // Back down
    { duration: '30s', target: 0 },   // Ramp down
  ],
  thresholds: {
    http_req_duration: ['p(95)<500'], // 95% des requêtes < 500ms
    http_req_failed: ['rate<0.01'],   // Moins de 1% d'échecs
  },
};

export default function () {
  const res = http.get('http://mon-app.example.com/api/items');

  check(res, {
    'status is 200': (r) => r.status === 200,
    'response time < 500ms': (r) => r.timings.duration < 500,
  });

  sleep(1);
}
```

**Exécuter le test** :

```bash
# Test de charge
k6 run load-test.js

# Avec résultats dans InfluxDB
k6 run --out influxdb=http://localhost:8086/k6 load-test.js
```

### Avec Apache Bench

```bash
# Test simple
ab -n 1000 -c 10 http://mon-app.example.com/

# Avec keepalive
ab -n 10000 -c 100 -k http://mon-app.example.com/api/items
```

### Avec wrk

```bash
# Installation
sudo apt-get install wrk

# Test de charge
wrk -t4 -c100 -d30s http://mon-app.example.com/

# Avec script Lua
wrk -t4 -c100 -d30s -s script.lua http://mon-app.example.com/
```

---

## Intégration dans les pipelines CI/CD

### Pipeline GitLab CI complet avec tests

```yaml
# .gitlab-ci.yml

stages:
  - validate
  - test
  - build
  - integration
  - deploy

# ═══════════════════════════════════════════════════════
# STAGE: VALIDATE
# ═══════════════════════════════════════════════════════

validate:yaml:
  stage: validate
  image: alpine:latest
  before_script:
    - apk add --no-cache yamllint
  script:
    - yamllint k8s/
  only:
    - merge_requests
    - main

validate:kubeval:
  stage: validate
  image: alpine:latest
  before_script:
    - wget -qO- https://github.com/instrumenta/kubeval/releases/latest/download/kubeval-linux-amd64.tar.gz | tar xz
    - mv kubeval /usr/local/bin/
  script:
    - kubeval --strict k8s/*.yaml
  only:
    - merge_requests
    - main

validate:helm-lint:
  stage: validate
  image: alpine/helm:latest
  script:
    - helm lint ./charts/mon-app
    - helm lint ./charts/mon-app -f ./charts/mon-app/values-production.yaml
  only:
    - merge_requests
    - main

# ═══════════════════════════════════════════════════════
# STAGE: TEST
# ═══════════════════════════════════════════════════════

test:helm-unittest:
  stage: test
  image: alpine/helm:latest
  before_script:
    - helm plugin install https://github.com/helm-unittest/helm-unittest
  script:
    - helm unittest ./charts/mon-app
  artifacts:
    reports:
      junit: test-results.xml
  only:
    - merge_requests
    - main

test:security-trivy:
  stage: test
  image: aquasec/trivy:latest
  script:
    - trivy config --severity HIGH,CRITICAL k8s/
    - trivy image --severity HIGH,CRITICAL $CI_REGISTRY_IMAGE:$CI_COMMIT_SHA
  allow_failure: true
  only:
    - merge_requests
    - main

test:security-checkov:
  stage: test
  image: bridgecrew/checkov:latest
  script:
    - checkov -d k8s/ -o junitxml > checkov-report.xml
  artifacts:
    reports:
      junit: checkov-report.xml
  allow_failure: true
  only:
    - merge_requests
    - main

# ═══════════════════════════════════════════════════════
# STAGE: BUILD
# ═══════════════════════════════════════════════════════

build:image:
  stage: build
  image: docker:latest
  services:
    - docker:dind
  script:
    - docker build -t $CI_REGISTRY_IMAGE:$CI_COMMIT_SHA .
    - docker push $CI_REGISTRY_IMAGE:$CI_COMMIT_SHA
  only:
    - main

# ═══════════════════════════════════════════════════════
# STAGE: INTEGRATION
# ═══════════════════════════════════════════════════════

test:integration:
  stage: integration
  image: alpine:latest
  before_script:
    - apk add --no-cache curl bash
    - curl -Lo ./kind https://kind.sigs.k8s.io/dl/latest/kind-linux-amd64
    - chmod +x ./kind
    - mv ./kind /usr/local/bin/kind
    - curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
    - chmod +x kubectl
    - mv kubectl /usr/local/bin/
  script:
    - kind create cluster --name test
    - kubectl apply -f k8s/
    - kubectl wait --for=condition=ready pod -l app=mon-app --timeout=300s
    - kubectl run test --rm -i --restart=Never --image=curlimages/curl -- curl -f http://mon-app-service/health
  after_script:
    - kind delete cluster --name test
  only:
    - main

# ═══════════════════════════════════════════════════════
# STAGE: DEPLOY
# ═══════════════════════════════════════════════════════

deploy:staging:
  stage: deploy
  image: alpine/helm:latest
  script:
    - helm upgrade --install mon-app ./charts/mon-app \
        --namespace staging \
        --set image.tag=$CI_COMMIT_SHA \
        -f ./charts/mon-app/values-staging.yaml
    - kubectl rollout status deployment/mon-app -n staging
  environment:
    name: staging
  only:
    - main

deploy:production:
  stage: deploy
  image: alpine/helm:latest
  script:
    - helm upgrade --install mon-app ./charts/mon-app \
        --namespace production \
        --set image.tag=$CI_COMMIT_SHA \
        -f ./charts/mon-app/values-production.yaml
    - kubectl rollout status deployment/mon-app -n production
  environment:
    name: production
  when: manual
  only:
    - main
```

### Pipeline Jenkins avec tests

```groovy
// Jenkinsfile

pipeline {
    agent any

    environment {
        KUBECONFIG = credentials('kubeconfig')
        IMAGE_NAME = 'mon-app'
        IMAGE_TAG = "${env.BUILD_NUMBER}"
    }

    stages {
        stage('Validate YAML') {
            steps {
                sh '''
                    yamllint k8s/
                    kubeval --strict k8s/*.yaml
                '''
            }
        }

        stage('Lint Helm Chart') {
            steps {
                sh 'helm lint ./charts/mon-app'
            }
        }

        stage('Unit Tests Helm') {
            steps {
                sh 'helm unittest ./charts/mon-app'
            }
        }

        stage('Security Scan') {
            parallel {
                stage('Trivy Config') {
                    steps {
                        sh 'trivy config k8s/'
                    }
                }
                stage('Checkov') {
                    steps {
                        sh 'checkov -d k8s/'
                    }
                }
            }
        }

        stage('Build Image') {
            steps {
                sh '''
                    docker build -t ${IMAGE_NAME}:${IMAGE_TAG} .
                    docker tag ${IMAGE_NAME}:${IMAGE_TAG} ${IMAGE_NAME}:latest
                '''
            }
        }

        stage('Scan Image') {
            steps {
                sh 'trivy image ${IMAGE_NAME}:${IMAGE_TAG}'
            }
        }

        stage('Integration Tests') {
            steps {
                sh '''
                    kind create cluster --name test-${BUILD_NUMBER}
                    kubectl apply -f k8s/
                    kubectl wait --for=condition=ready pod -l app=mon-app --timeout=300s
                    ./scripts/run-integration-tests.sh
                '''
            }
            post {
                always {
                    sh 'kind delete cluster --name test-${BUILD_NUMBER}'
                }
            }
        }

        stage('Deploy to Staging') {
            steps {
                sh '''
                    helm upgrade --install mon-app ./charts/mon-app \
                        --namespace staging \
                        --set image.tag=${IMAGE_TAG} \
                        -f values-staging.yaml
                '''
            }
        }

        stage('Smoke Tests') {
            steps {
                sh './scripts/smoke-tests.sh staging'
            }
        }

        stage('Deploy to Production') {
            when {
                branch 'main'
            }
            steps {
                input message: 'Déployer en production?'
                sh '''
                    helm upgrade --install mon-app ./charts/mon-app \
                        --namespace production \
                        --set image.tag=${IMAGE_TAG} \
                        -f values-production.yaml
                '''
            }
        }
    }

    post {
        always {
            junit 'test-results/*.xml'
        }
        success {
            echo 'Pipeline réussi!'
        }
        failure {
            echo 'Pipeline échoué!'
        }
    }
}
```

---

## Tests de smoke (tests de fumée)

Les **smoke tests** sont des tests rapides après déploiement pour vérifier que l'essentiel fonctionne.

```bash
#!/bin/bash
# scripts/smoke-tests.sh

ENVIRONMENT=$1
BASE_URL="https://mon-app-${ENVIRONMENT}.example.com"

echo "🔥 Smoke tests pour ${ENVIRONMENT}..."

# Test 1: Health check
echo "Testing health endpoint..."
response=$(curl -s -o /dev/null -w "%{http_code}" ${BASE_URL}/health)
if [ $response -eq 200 ]; then
    echo "✅ Health check OK"
else
    echo "❌ Health check failed (HTTP ${response})"
    exit 1
fi

# Test 2: API disponible
echo "Testing API endpoint..."
response=$(curl -s ${BASE_URL}/api/version)
if [[ $response == *"version"* ]]; then
    echo "✅ API OK"
else
    echo "❌ API failed"
    exit 1
fi

# Test 3: Base de données accessible
echo "Testing database connectivity..."
response=$(curl -s ${BASE_URL}/api/db-status)
if [[ $response == *"connected"* ]]; then
    echo "✅ Database OK"
else
    echo "❌ Database failed"
    exit 1
fi

# Test 4: Vérifier les métriques
echo "Testing metrics endpoint..."
response=$(curl -s -o /dev/null -w "%{http_code}" ${BASE_URL}/metrics)
if [ $response -eq 200 ]; then
    echo "✅ Metrics OK"
else
    echo "⚠️  Metrics unavailable (non-critical)"
fi

echo "🎉 Tous les smoke tests réussis!"
```

---

## Monitoring continu et alertes

### Tests synthétiques avec Blackbox Exporter

**Blackbox Exporter** permet de surveiller continuellement vos endpoints.

```yaml
# blackbox-config.yaml
modules:
  http_2xx:
    prober: http
    timeout: 5s
    http:
      valid_http_versions: ["HTTP/1.1", "HTTP/2.0"]
      valid_status_codes: [200]
      method: GET

  http_post_2xx:
    prober: http
    http:
      method: POST
      headers:
        Content-Type: application/json
      body: '{"test": true}'
```

**Prometheus configuration** :

```yaml
# prometheus-config.yaml
scrape_configs:
  - job_name: 'blackbox'
    metrics_path: /probe
    params:
      module: [http_2xx]
    static_configs:
      - targets:
        - https://mon-app.example.com/health
        - https://mon-app.example.com/api/status
    relabel_configs:
      - source_labels: [__address__]
        target_label: __param_target
      - source_labels: [__param_target]
        target_label: instance
      - target_label: __address__
        replacement: blackbox-exporter:9115
```

### Alertes Prometheus

```yaml
# alerts.yaml
groups:
  - name: application
    interval: 30s
    rules:
      - alert: ApplicationDown
        expr: probe_success{job="blackbox"} == 0
        for: 5m
        labels:
          severity: critical
        annotations:
          summary: "Application {{ $labels.instance }} is down"
          description: "The application has been down for more than 5 minutes."

      - alert: HighLatency
        expr: probe_http_duration_seconds > 1
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "High latency on {{ $labels.instance }}"
          description: "Response time is above 1 second."

      - alert: PodCrashLooping
        expr: rate(kube_pod_container_status_restarts_total[15m]) > 0
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "Pod {{ $labels.pod }} is crash looping"
```

---

## Bonnes pratiques de tests

### 1. Pyramide des tests équilibrée

```
E2E (10%)         ←  Moins nombreux mais critiques
Integration (20%) ←  Scénarios clés
Contract (30%)    ←  Validation exhaustive
Unit (40%)        ←  Maximum de couverture
```

### 2. Tests rapides et fiables

✅ **Bon** :
- Tests < 30 secondes pour feedback rapide
- Tests déterministes (pas de flakiness)
- Parallélisation quand possible

❌ **À éviter** :
- Tests lents qui bloquent le développement
- Tests instables qui échouent aléatoirement
- Dépendances externes non mockées

### 3. Test en isolation

```bash
# Bon : cluster éphémère par test
kind create cluster --name test-${BUILD_ID}
# ... tests
kind delete cluster --name test-${BUILD_ID}

# Mauvais : réutiliser le même cluster
# → risque de conflits entre tests
```

### 4. Fail fast

```yaml
# Dans CI/CD : arrêter au premier échec critique
stages:
  - validate    # Si échec → stop
  - test       # Si échec → stop
  - build      # Si échec → stop
  - deploy     # Seulement si tout OK
```

### 5. Tests comme documentation

```yaml
# test_deployment.yaml
suite: test deployment configuration
tests:
  - it: should deploy with at least 2 replicas for high availability
    set:
      environment: production
    asserts:
      - matchRegex:
          path: spec.replicas
          pattern: "^[2-9]|[1-9][0-9]+$"
```

### 6. Environnements de test dédiés

- **Dev** : Tests manuels, expérimentation
- **CI** : Tests automatisés sur clusters éphémères
- **Staging** : Tests d'intégration continus
- **Production** : Smoke tests et monitoring

### 7. Versioning des tests

```
tests/
├── v1.0/
│   └── integration_test.sh
├── v2.0/
│   └── integration_test.sh
└── current -> v2.0/
```

### 8. Métriques de qualité

Suivez ces métriques :
- **Code coverage** : % de code testé
- **Test success rate** : Taux de réussite
- **Test duration** : Temps d'exécution
- **Flakiness rate** : Tests instables

### 9. Tests de régression

```bash
# Sauvegarder une baseline
kubectl get all -n production -o yaml > baseline.yaml

# Après déploiement, comparer
kubectl get all -n production -o yaml > current.yaml
diff baseline.yaml current.yaml
```

### 10. Documentation des tests

```markdown
# Tests

## Tests unitaires
- Localisation : `tests/unit/`
- Commande : `helm unittest ./charts/mon-app`
- Durée : ~5 secondes

## Tests d'intégration
- Localisation : `tests/integration/`
- Commande : `./scripts/integration-tests.sh`
- Durée : ~5 minutes
- Prérequis : Kind, kubectl

## Tests de charge
- Localisation : `tests/load/`
- Commande : `k6 run load-test.js`
- Durée : ~10 minutes
```

---

## Outils récapitulatifs

### Validation et linting

| Outil | Usage | Type |
|-------|-------|------|
| yamllint | Validation syntaxe YAML | Linter |
| kubeval | Validation schémas K8s | Validator |
| kubeconform | Alternative kubeval | Validator |
| kubectl dry-run | Validation par K8s | Validator |
| helm lint | Validation Charts Helm | Linter |

### Tests

| Outil | Usage | Type |
|-------|-------|------|
| helm-unittest | Tests unitaires Helm | Unit |
| Bats | Tests shell script | Integration |
| Kind | Clusters K8s éphémères | Integration |
| k3d | Alternative Kind | Integration |

### Sécurité

| Outil | Usage | Type |
|-------|-------|------|
| Trivy | Scan vulnérabilités | Security |
| Checkov | Analyse config | Security |
| Kubesec | Score sécurité | Security |
| Polaris | Audit bonnes pratiques | Security |

### Performance

| Outil | Usage | Type |
|-------|-------|------|
| k6 | Tests de charge modernes | Load |
| wrk | Tests de charge rapides | Load |
| Apache Bench | Tests HTTP simples | Load |

### Monitoring

| Outil | Usage | Type |
|-------|-------|------|
| Blackbox Exporter | Tests synthétiques | Monitoring |
| Prometheus | Métriques et alertes | Monitoring |

---

## Récapitulatif

### Points clés

**Les tests automatisés, c'est** :
- La confiance pour déployer sans stress
- La détection précoce des problèmes
- La documentation vivante du comportement
- L'assurance qualité continue

**Types de tests** :
1. **Unit** : Validation YAML et templates
2. **Contract** : Conformité Kubernetes
3. **Integration** : Déploiement sur cluster test
4. **E2E** : Scénarios utilisateur complets
5. **Security** : Vulnérabilités et configuration
6. **Performance** : Charge et latence

**Outils essentiels** :
- yamllint / kubeval : Validation
- helm-unittest : Tests Helm
- Kind : Clusters éphémères
- Trivy : Sécurité
- k6 : Performance

**Intégration CI/CD** :
- Tests à chaque étape du pipeline
- Fail fast sur les erreurs critiques
- Parallélisation pour rapidité
- Artefacts de tests conservés

### Bénéfices

Avec des tests automatisés :
- ✅ Confiance totale dans les déploiements
- ✅ Bugs détectés avant production
- ✅ Refactoring sans crainte
- ✅ Qualité maintenue dans le temps
- ✅ Documentation automatique
- ✅ Feedback immédiat

---

## Conclusion

Les tests automatisés ne sont pas un luxe, mais une **nécessité** pour opérer Kubernetes avec sérénité. Ils forment un filet de sécurité qui vous permet d'itérer rapidement tout en maintenant la qualité.

**La progression idéale** :

1. **Commencez simple** : yamllint + kubeval
2. **Ajoutez la sécurité** : Trivy
3. **Tests Helm** : helm-unittest
4. **Intégration** : Kind + tests basiques
5. **Performance** : k6 pour les APIs critiques
6. **Monitoring** : Blackbox Exporter

Vous n'avez pas besoin de tout implémenter d'un coup. Commencez par les tests de base et ajoutez progressivement des couches selon vos besoins.

**Le workflow DevOps complet** :

```
┌──────┐   ┌──────┐   ┌──────┐   ┌───────┐   ┌──────┐   ┌───────┐
│ Code │ → │ Test │ → │Build │ → │ Test  │ → │Deploy│ → │Monitor│
│      │   │ Unit │   │Image │   │ Integ │   │      │   │ Cont. │
└──────┘   └──────┘   └──────┘   └───────┘   └──────┘   └───────┘
```

Chaque étape est validée automatiquement, vous donnant la **confiance** nécessaire pour déployer en production plusieurs fois par jour.

**Vous avez maintenant** :
- ✅ Git pour le versioning
- ✅ Registry pour les images
- ✅ CI/CD pour l'automatisation
- ✅ ArgoCD pour GitOps
- ✅ Helm pour le packaging
- ✅ Tests pour la qualité

**Félicitations !** Vous disposez d'une infrastructure DevOps complète, moderne et professionnelle sur MicroK8s. Vous pouvez maintenant déployer vos applications avec confiance, sachant que chaque changement est validé, testé et surveillé automatiquement.

**La prochaine étape ?** Les stratégies de déploiement avancées (Blue-Green, Canary) pour minimiser les risques lors des mises à jour !

⏭️ [Stratégies de déploiement](/17-devops-et-cicd/09-strategies-de-deploiement.md)
