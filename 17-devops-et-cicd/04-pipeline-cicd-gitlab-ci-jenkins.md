🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 17.4 Pipeline CI/CD (GitLab CI/Jenkins)

## Introduction

Vous avez maintenant vos manifestes Kubernetes dans Git et un registry privé pour stocker vos images Docker. Il est temps d'automatiser le processus qui relie ces deux éléments : le **pipeline CI/CD** (Continuous Integration / Continuous Deployment).

Un pipeline CI/CD automatise toutes les étapes entre le moment où vous écrivez du code et celui où il est déployé en production. Fini les déploiements manuels et les erreurs humaines : la machine s'occupe de tout, de manière fiable et reproductible.

Dans ce chapitre, nous allons explorer deux des solutions les plus populaires : **GitLab CI** et **Jenkins**, en comprenant leurs forces respectives et comment les mettre en œuvre sur votre infrastructure MicroK8s.

---

## Qu'est-ce que CI/CD ?

### Continuous Integration (CI)

L'**Intégration Continue** est une pratique qui consiste à intégrer régulièrement (plusieurs fois par jour) le code de tous les développeurs dans une branche commune.

**Processus typique de CI** :

1. Un développeur pousse du code vers Git
2. Le système détecte automatiquement le changement
3. Le code est récupéré et testé
4. Les tests automatisés sont exécutés
5. Si tout passe, l'application est construite (build)
6. L'image Docker est créée
7. L'image est poussée vers le registry
8. Une notification est envoyée

**Objectifs de la CI** :
- Détecter rapidement les bugs
- Réduire les conflits d'intégration
- Valider chaque changement automatiquement
- Maintenir la qualité du code

### Continuous Deployment/Delivery (CD)

Le **Déploiement Continu** va plus loin en automatisant aussi le déploiement.

**Distinction importante** :

- **Continuous Delivery** : Le code est toujours prêt à être déployé, mais le déploiement nécessite une action manuelle
- **Continuous Deployment** : Tout code validé est automatiquement déployé en production

**Processus typique de CD** :

1. L'image est disponible dans le registry (sortie de la CI)
2. Les manifestes Kubernetes sont mis à jour
3. Le déploiement vers l'environnement cible est effectué
4. Des tests post-déploiement sont exécutés
5. Le monitoring vérifie la santé de l'application
6. En cas de problème, un rollback automatique peut être déclenché

### Le pipeline complet

Un pipeline CI/CD complet ressemble à ceci :

```
┌─────────┐    ┌──────┐    ┌───────┐    ┌──────────┐    ┌────────┐    ┌──────────┐
│ Git     │ -> │ Test │ -> │ Build │ -> │ Push     │ -> │ Deploy │ -> │ Monitor  │
│ Push    │    │      │    │ Image │    │ Registry │    │ K8s    │    │          │
└─────────┘    └──────┘    └───────┘    └──────────┘    └────────┘    └──────────┘
     ↓             ↓            ↓             ↓            ↓              ↓
  Webhook      Unit tests   docker build   docker push  kubectl apply  Prometheus
              Integration   tag version                  ArgoCD sync
              Linting
```

---

## GitLab CI : La solution intégrée

### Présentation

**GitLab CI/CD** est la solution d'intégration continue et de déploiement continu intégrée nativement à GitLab. C'est une des raisons pour lesquelles GitLab est si populaire : tout est inclus dans une seule plateforme.

**Caractéristiques** :
- Configuration via un fichier `.gitlab-ci.yml` dans le repository
- Runners pour exécuter les jobs
- Interface web intégrée
- Variables et secrets sécurisés
- Artefacts et caches
- Pipelines complexes avec stages et jobs
- Gratuit pour les projets open source et privés

### Architecture GitLab CI

GitLab CI fonctionne avec des **runners** qui exécutent vos jobs :

```
┌─────────────────────────────────────────────────────────┐
│                    GitLab Server                        │
│  ┌──────────────────────────────────────────────────┐   │
│  │  Repository + .gitlab-ci.yml                     │   │
│  └──────────────────────────────────────────────────┘   │
│                         ↓                               │
│  ┌──────────────────────────────────────────────────┐   │
│  │  GitLab CI/CD Engine                             │   │
│  │  - Détecte les push                              │   │
│  │  - Parse le .gitlab-ci.yml                       │   │
│  │  - Distribue les jobs aux runners                │   │
│  └──────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────┘
                         ↓
        ┌────────────────┴──────────────────┐
        ↓                                   ↓
┌───────────────────┐            ┌──────────────────────┐
│  GitLab Runner 1  │            │  GitLab Runner 2     │
│  (Docker)         │            │  (Kubernetes)        │
│  Exécute les jobs │            │  Exécute les jobs    │
└───────────────────┘            └──────────────────────┘
```

### Installation d'un GitLab Runner sur MicroK8s

Pour utiliser GitLab CI, vous avez besoin d'un **runner** qui exécutera vos pipelines.

#### Option 1 : Runner Docker (le plus simple)

Si vous avez Docker sur votre machine hôte :

```bash
# Installer le runner
curl -L https://packages.gitlab.com/install/repositories/runner/gitlab-runner/script.deb.sh | sudo bash
sudo apt-get install gitlab-runner

# Enregistrer le runner avec GitLab
sudo gitlab-runner register
```

Lors de l'enregistrement, vous devrez fournir :
- L'URL de votre GitLab (ex: https://gitlab.com)
- Le token de registration (disponible dans Settings > CI/CD > Runners)
- Une description du runner
- Des tags (ex: docker, microk8s)
- L'exécuteur : choisissez `docker`
- L'image Docker par défaut : `docker:latest`

#### Option 2 : Runner dans Kubernetes (avancé)

Vous pouvez déployer le runner directement dans votre cluster MicroK8s :

```bash
# Ajouter le repo Helm de GitLab
helm repo add gitlab https://charts.gitlab.io
helm repo update

# Installer le runner via Helm
helm install gitlab-runner gitlab/gitlab-runner \
  --namespace gitlab \
  --create-namespace \
  --set gitlabUrl=https://gitlab.com \
  --set runnerRegistrationToken=YOUR_TOKEN \
  --set rbac.create=true
```

### Le fichier .gitlab-ci.yml

Le fichier `.gitlab-ci.yml` à la racine de votre repository définit votre pipeline.

#### Structure de base

```yaml
# .gitlab-ci.yml

# Définition des stages (étapes)
stages:
  - test
  - build
  - deploy

# Variables globales
variables:
  DOCKER_REGISTRY: "localhost:32000"
  APP_NAME: "mon-app"

# Job de test
test_job:
  stage: test
  image: python:3.11
  script:
    - pip install -r requirements.txt
    - pytest tests/
  only:
    - branches

# Job de build
build_image:
  stage: build
  image: docker:latest
  services:
    - docker:dind
  script:
    - docker build -t $DOCKER_REGISTRY/$APP_NAME:$CI_COMMIT_SHA .
    - docker push $DOCKER_REGISTRY/$APP_NAME:$CI_COMMIT_SHA
  only:
    - main

# Job de déploiement
deploy_production:
  stage: deploy
  image: bitnami/kubectl:latest
  script:
    - kubectl set image deployment/$APP_NAME $APP_NAME=$DOCKER_REGISTRY/$APP_NAME:$CI_COMMIT_SHA
    - kubectl rollout status deployment/$APP_NAME
  only:
    - main
  when: manual
```

#### Explication des concepts

**Stages** : Les étapes du pipeline exécutées séquentiellement
```yaml
stages:
  - test      # Étape 1
  - build     # Étape 2
  - deploy    # Étape 3
```

**Jobs** : Les tâches individuelles dans chaque stage
```yaml
test_job:           # Nom du job
  stage: test       # Stage auquel il appartient
  image: python:3.11  # Image Docker à utiliser
  script:           # Commandes à exécuter
    - pip install -r requirements.txt
    - pytest tests/
```

**Variables** : Variables disponibles dans tous les jobs
```yaml
variables:
  DOCKER_REGISTRY: "localhost:32000"
  APP_NAME: "mon-app"
```

**Conditions d'exécution** :
```yaml
only:
  - main        # Seulement sur la branche main
  - tags        # Seulement sur les tags
  - branches    # Sur toutes les branches

except:
  - develop     # Sauf sur develop

when: manual    # Requiert une action manuelle
```

### Pipeline complet pour une application Python Flask

Voici un exemple complet et réaliste :

```yaml
# .gitlab-ci.yml

image: docker:latest

services:
  - docker:dind

stages:
  - lint
  - test
  - build
  - deploy

variables:
  DOCKER_REGISTRY: "localhost:32000"
  APP_NAME: "flask-app"
  DOCKER_DRIVER: overlay2
  DOCKER_TLS_CERTDIR: ""

# Cache pour accélérer les builds
cache:
  paths:
    - .pip-cache/

# ═══════════════════════════════════════════════════════
# STAGE: LINT
# ═══════════════════════════════════════════════════════

lint:code:
  stage: lint
  image: python:3.11
  before_script:
    - pip install flake8 black
  script:
    - flake8 app/ --max-line-length=120
    - black --check app/
  allow_failure: true  # Le pipeline continue même si le linting échoue

# ═══════════════════════════════════════════════════════
# STAGE: TEST
# ═══════════════════════════════════════════════════════

test:unit:
  stage: test
  image: python:3.11
  before_script:
    - pip install -r requirements.txt -r requirements-dev.txt
  script:
    - pytest tests/unit/ --cov=app --cov-report=term
  coverage: '/TOTAL.*\s+(\d+%)$/'

test:integration:
  stage: test
  image: python:3.11
  services:
    - postgres:14
  variables:
    POSTGRES_DB: test_db
    POSTGRES_USER: test_user
    POSTGRES_PASSWORD: test_pass
    DATABASE_URL: postgresql://test_user:test_pass@postgres:5432/test_db
  before_script:
    - pip install -r requirements.txt -r requirements-dev.txt
  script:
    - pytest tests/integration/

# ═══════════════════════════════════════════════════════
# STAGE: BUILD
# ═══════════════════════════════════════════════════════

build:image:
  stage: build
  before_script:
    - docker login -u $REGISTRY_USER -p $REGISTRY_PASSWORD $DOCKER_REGISTRY
  script:
    # Build avec plusieurs tags
    - docker build -t $DOCKER_REGISTRY/$APP_NAME:$CI_COMMIT_SHA .
    - docker build -t $DOCKER_REGISTRY/$APP_NAME:$CI_COMMIT_REF_NAME .
    - docker build -t $DOCKER_REGISTRY/$APP_NAME:latest .

    # Push vers le registry
    - docker push $DOCKER_REGISTRY/$APP_NAME:$CI_COMMIT_SHA
    - docker push $DOCKER_REGISTRY/$APP_NAME:$CI_COMMIT_REF_NAME
    - docker push $DOCKER_REGISTRY/$APP_NAME:latest
  only:
    - main
    - tags

# Scan de sécurité de l'image
build:scan:
  stage: build
  image: aquasec/trivy:latest
  script:
    - trivy image --severity HIGH,CRITICAL $DOCKER_REGISTRY/$APP_NAME:$CI_COMMIT_SHA
  allow_failure: true
  only:
    - main

# ═══════════════════════════════════════════════════════
# STAGE: DEPLOY
# ═══════════════════════════════════════════════════════

deploy:staging:
  stage: deploy
  image: bitnami/kubectl:latest
  environment:
    name: staging
    url: https://staging.example.com
  before_script:
    - kubectl config use-context staging
  script:
    # Mise à jour de l'image dans le deployment
    - kubectl set image deployment/$APP_NAME
        $APP_NAME=$DOCKER_REGISTRY/$APP_NAME:$CI_COMMIT_SHA
        -n staging

    # Attendre que le rollout soit terminé
    - kubectl rollout status deployment/$APP_NAME -n staging

    # Vérifier que les pods sont prêts
    - kubectl wait --for=condition=ready pod
        -l app=$APP_NAME -n staging --timeout=300s
  only:
    - main

deploy:production:
  stage: deploy
  image: bitnami/kubectl:latest
  environment:
    name: production
    url: https://app.example.com
  before_script:
    - kubectl config use-context production
  script:
    # Mise à jour de l'image
    - kubectl set image deployment/$APP_NAME
        $APP_NAME=$DOCKER_REGISTRY/$APP_NAME:$CI_COMMIT_SHA
        -n production

    # Rollout avec surveillance
    - kubectl rollout status deployment/$APP_NAME -n production

    # Tests post-déploiement
    - curl -f https://app.example.com/health || exit 1
  only:
    - main
  when: manual  # Déploiement manuel en production

# Rollback en cas de problème
rollback:production:
  stage: deploy
  image: bitnami/kubectl:latest
  environment:
    name: production
  script:
    - kubectl rollout undo deployment/$APP_NAME -n production
    - kubectl rollout status deployment/$APP_NAME -n production
  when: manual
  only:
    - main
```

### Variables et Secrets dans GitLab CI

GitLab permet de définir des variables sécurisées dans l'interface web.

**Dans GitLab** : Settings > CI/CD > Variables

Créez des variables :
- `DOCKER_REGISTRY` : localhost:32000
- `REGISTRY_USER` : votre username
- `REGISTRY_PASSWORD` : votre mot de passe (marqué comme "masked" et "protected")
- `KUBE_CONFIG` : contenu de votre kubeconfig (type: file)

**Utilisation dans le pipeline** :

```yaml
deploy:
  script:
    - echo $REGISTRY_PASSWORD | docker login -u $REGISTRY_USER --password-stdin $DOCKER_REGISTRY
    - kubectl --kubeconfig=$KUBE_CONFIG apply -f k8s/
```

### Artefacts et Cache

**Artefacts** : Fichiers générés qui persistent entre stages

```yaml
build:
  script:
    - npm run build
  artifacts:
    paths:
      - dist/
    expire_in: 1 week

deploy:
  script:
    - ls dist/  # Les fichiers sont disponibles
```

**Cache** : Accélère les builds en réutilisant des dépendances

```yaml
test:
  cache:
    key: ${CI_COMMIT_REF_SLUG}
    paths:
      - node_modules/
      - .pip-cache/
  script:
    - npm install
    - npm test
```

---

## Jenkins : Le vétéran flexible

### Présentation

**Jenkins** est un serveur d'automatisation open-source, considéré comme le grand-père des outils CI/CD. Créé en 2011, il reste extrêmement populaire grâce à sa flexibilité et son écosystème de plugins.

**Caractéristiques** :
- Très extensible (1000+ plugins)
- Configuration via l'interface web ou Jenkinsfile
- Pipelines déclaratifs ou scriptés
- Écosystème mature
- Large communauté
- Auto-hébergé (pas de service cloud obligatoire)

### Installation de Jenkins sur MicroK8s

#### Méthode 1 : Via Helm (recommandé)

```bash
# Ajouter le repo Helm Jenkins
helm repo add jenkins https://charts.jenkins.io
helm repo update

# Créer un namespace
kubectl create namespace jenkins

# Installer Jenkins
helm install jenkins jenkins/jenkins \
  --namespace jenkins \
  --set controller.servicePort=8080 \
  --set controller.serviceType=NodePort \
  --set persistence.enabled=true \
  --set persistence.size=10Gi
```

**Récupérer le mot de passe admin** :

```bash
kubectl exec --namespace jenkins -it svc/jenkins -c jenkins -- \
  /bin/cat /run/secrets/additional/chart-admin-password && echo
```

**Accéder à Jenkins** :

```bash
# Récupérer le NodePort
kubectl get svc -n jenkins

# Accéder via http://localhost:PORT
```

#### Méthode 2 : Via manifestes YAML

```yaml
# jenkins-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: jenkins
  namespace: jenkins
spec:
  replicas: 1
  selector:
    matchLabels:
      app: jenkins
  template:
    metadata:
      labels:
        app: jenkins
    spec:
      containers:
      - name: jenkins
        image: jenkins/jenkins:lts
        ports:
        - containerPort: 8080
          name: web
        - containerPort: 50000
          name: agent
        volumeMounts:
        - name: jenkins-home
          mountPath: /var/jenkins_home
        env:
        - name: JAVA_OPTS
          value: "-Djenkins.install.runSetupWizard=false"
      volumes:
      - name: jenkins-home
        persistentVolumeClaim:
          claimName: jenkins-pvc
---
apiVersion: v1
kind: Service
metadata:
  name: jenkins
  namespace: jenkins
spec:
  type: NodePort
  selector:
    app: jenkins
  ports:
  - name: web
    port: 8080
    targetPort: 8080
    nodePort: 32080
  - name: agent
    port: 50000
    targetPort: 50000
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: jenkins-pvc
  namespace: jenkins
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 10Gi
```

Déployer :
```bash
kubectl create namespace jenkins
kubectl apply -f jenkins-deployment.yaml
```

### Configuration initiale de Jenkins

1. **Accéder à Jenkins** : http://localhost:32080
2. **Débloquer Jenkins** avec le mot de passe initial
3. **Installer les plugins suggérés**
4. **Créer le premier utilisateur admin**

**Plugins essentiels à installer** :
- Docker Pipeline
- Kubernetes
- Git
- Pipeline
- Blue Ocean (interface moderne)
- Credentials Binding

### Le Jenkinsfile

Jenkins utilise un **Jenkinsfile** pour définir les pipelines (équivalent du `.gitlab-ci.yml`).

#### Pipeline déclaratif (recommandé)

```groovy
// Jenkinsfile

pipeline {
    agent any

    environment {
        DOCKER_REGISTRY = 'localhost:32000'
        APP_NAME = 'mon-app'
        IMAGE_TAG = "${env.BUILD_NUMBER}"
    }

    stages {
        stage('Checkout') {
            steps {
                git branch: 'main',
                    url: 'https://github.com/user/repo.git'
            }
        }

        stage('Test') {
            steps {
                sh 'pip install -r requirements.txt'
                sh 'pytest tests/'
            }
        }

        stage('Build') {
            steps {
                script {
                    docker.build("${DOCKER_REGISTRY}/${APP_NAME}:${IMAGE_TAG}")
                }
            }
        }

        stage('Push') {
            steps {
                script {
                    docker.withRegistry("http://${DOCKER_REGISTRY}") {
                        docker.image("${DOCKER_REGISTRY}/${APP_NAME}:${IMAGE_TAG}").push()
                        docker.image("${DOCKER_REGISTRY}/${APP_NAME}:${IMAGE_TAG}").push('latest')
                    }
                }
            }
        }

        stage('Deploy') {
            steps {
                sh """
                    kubectl set image deployment/${APP_NAME} \
                        ${APP_NAME}=${DOCKER_REGISTRY}/${APP_NAME}:${IMAGE_TAG}
                    kubectl rollout status deployment/${APP_NAME}
                """
            }
        }
    }

    post {
        success {
            echo 'Pipeline réussi !'
        }
        failure {
            echo 'Pipeline échoué !'
        }
        always {
            cleanWs()
        }
    }
}
```

#### Pipeline complet avec tous les stages

```groovy
// Jenkinsfile complet

pipeline {
    agent any

    environment {
        DOCKER_REGISTRY = 'localhost:32000'
        APP_NAME = 'flask-app'
        GIT_COMMIT_SHORT = sh(script: "git rev-parse --short HEAD", returnStdout: true).trim()
    }

    options {
        buildDiscarder(logRotator(numToKeepStr: '10'))
        timeout(time: 30, unit: 'MINUTES')
        timestamps()
    }

    stages {
        stage('Initialize') {
            steps {
                script {
                    echo "Build Number: ${env.BUILD_NUMBER}"
                    echo "Git Commit: ${GIT_COMMIT_SHORT}"
                }
            }
        }

        stage('Lint') {
            steps {
                sh '''
                    pip install flake8 black
                    flake8 app/ --max-line-length=120
                    black --check app/
                '''
            }
        }

        stage('Unit Tests') {
            steps {
                sh '''
                    pip install -r requirements.txt -r requirements-dev.txt
                    pytest tests/unit/ --junitxml=reports/junit.xml --cov=app --cov-report=xml
                '''
            }
            post {
                always {
                    junit 'reports/junit.xml'
                }
            }
        }

        stage('Integration Tests') {
            steps {
                sh '''
                    docker-compose -f docker-compose.test.yml up -d
                    sleep 10
                    pytest tests/integration/
                    docker-compose -f docker-compose.test.yml down
                '''
            }
        }

        stage('Build Docker Image') {
            steps {
                script {
                    def imageName = "${DOCKER_REGISTRY}/${APP_NAME}"

                    // Build avec plusieurs tags
                    sh """
                        docker build -t ${imageName}:${GIT_COMMIT_SHORT} .
                        docker tag ${imageName}:${GIT_COMMIT_SHORT} ${imageName}:${env.BUILD_NUMBER}
                        docker tag ${imageName}:${GIT_COMMIT_SHORT} ${imageName}:latest
                    """
                }
            }
        }

        stage('Security Scan') {
            steps {
                sh """
                    docker run --rm -v /var/run/docker.sock:/var/run/docker.sock \
                        aquasec/trivy:latest image \
                        --severity HIGH,CRITICAL \
                        ${DOCKER_REGISTRY}/${APP_NAME}:${GIT_COMMIT_SHORT}
                """
            }
        }

        stage('Push to Registry') {
            steps {
                script {
                    def imageName = "${DOCKER_REGISTRY}/${APP_NAME}"

                    sh """
                        docker push ${imageName}:${GIT_COMMIT_SHORT}
                        docker push ${imageName}:${env.BUILD_NUMBER}
                        docker push ${imageName}:latest
                    """
                }
            }
        }

        stage('Deploy to Staging') {
            steps {
                sh """
                    kubectl set image deployment/${APP_NAME} \
                        ${APP_NAME}=${DOCKER_REGISTRY}/${APP_NAME}:${GIT_COMMIT_SHORT} \
                        -n staging

                    kubectl rollout status deployment/${APP_NAME} -n staging --timeout=5m
                """
            }
        }

        stage('Smoke Tests') {
            steps {
                sh '''
                    sleep 10
                    curl -f http://staging.example.com/health || exit 1
                '''
            }
        }

        stage('Deploy to Production') {
            when {
                branch 'main'
            }
            steps {
                input message: 'Déployer en production ?', ok: 'Déployer'

                sh """
                    kubectl set image deployment/${APP_NAME} \
                        ${APP_NAME}=${DOCKER_REGISTRY}/${APP_NAME}:${GIT_COMMIT_SHORT} \
                        -n production

                    kubectl rollout status deployment/${APP_NAME} -n production --timeout=5m
                """
            }
        }

        stage('Verify Production') {
            when {
                branch 'main'
            }
            steps {
                sh '''
                    sleep 10
                    curl -f https://app.example.com/health || exit 1
                    echo "Déploiement en production réussi !"
                '''
            }
        }
    }

    post {
        success {
            echo '✅ Pipeline terminé avec succès'
            // Notification Slack, email, etc.
        }

        failure {
            echo '❌ Pipeline échoué'
            // Notification d'échec
        }

        always {
            // Nettoyage
            sh 'docker system prune -f'
            cleanWs()
        }
    }
}
```

### Configuration Jenkins avec Kubernetes

Jenkins peut lancer des agents dynamiques dans Kubernetes pour chaque build :

**Configuration dans Jenkins** :

1. Aller dans **Manage Jenkins > Configure System**
2. Section **Cloud** > Add > Kubernetes
3. Configurer :
   - Kubernetes URL : https://kubernetes.default
   - Credentials : utiliser le service account
   - Namespace : jenkins

**Pod Template** :

```yaml
apiVersion: v1
kind: Pod
metadata:
  labels:
    jenkins: agent
spec:
  serviceAccountName: jenkins
  containers:
  - name: jnlp
    image: jenkins/inbound-agent:latest
    resources:
      requests:
        memory: "512Mi"
        cpu: "500m"
      limits:
        memory: "1Gi"
        cpu: "1000m"
  - name: docker
    image: docker:latest
    command:
    - cat
    tty: true
    volumeMounts:
    - name: docker-sock
      mountPath: /var/run/docker.sock
  volumes:
  - name: docker-sock
    hostPath:
      path: /var/run/docker.sock
```

---

## GitLab CI vs Jenkins : Comparaison

### GitLab CI

**Avantages** :
- ✅ Intégré nativement (pas d'installation séparée si vous utilisez GitLab)
- ✅ Configuration simple via YAML
- ✅ Interface moderne et intuitive
- ✅ Gratuit pour les projets privés
- ✅ Runners faciles à configurer
- ✅ Excellent pour GitOps

**Inconvénients** :
- ❌ Lié à GitLab (pas utilisable avec GitHub, Bitbucket)
- ❌ Moins de flexibilité que Jenkins
- ❌ Écosystème de plugins plus limité

**Idéal pour** :
- Nouvelles équipes
- Projets déjà sur GitLab
- Besoin de simplicité
- Approche GitOps

### Jenkins

**Avantages** :
- ✅ Très flexible et puissant
- ✅ Énorme écosystème de plugins
- ✅ Fonctionne avec n'importe quel VCS (Git, SVN, Mercurial)
- ✅ Grande communauté
- ✅ Mature et stable
- ✅ Peut gérer des workflows très complexes

**Inconvénients** :
- ❌ Configuration plus complexe
- ❌ Interface vieillissante (sauf Blue Ocean)
- ❌ Maintenance nécessaire
- ❌ Courbe d'apprentissage plus raide
- ❌ Nécessite de gérer les agents/runners

**Idéal pour** :
- Grandes organisations
- Workflows complexes
- Migration depuis un ancien système
- Besoin de plugins spécifiques
- Équipes expérimentées

### Tableau récapitulatif

| Critère | GitLab CI | Jenkins |
|---------|-----------|---------|
| Installation | Intégrée | Séparée |
| Configuration | YAML | Jenkinsfile (Groovy) |
| Interface | Moderne | Classique |
| Courbe d'apprentissage | Douce | Raide |
| Flexibilité | Moyenne | Très haute |
| Plugins | Limités | 1000+ |
| Coût | Gratuit | Gratuit (open-source) |
| Maintenance | Faible | Moyenne |
| GitOps | Excellent | Bon |

---

## Bonnes pratiques pour les pipelines CI/CD

### 1. Fail Fast

Exécutez les tests les plus rapides en premier :

```yaml
stages:
  - lint        # 30 secondes
  - unit-test   # 2 minutes
  - build       # 5 minutes
  - integration # 10 minutes
  - deploy
```

Si le linting échoue (30s), pas besoin d'attendre 17 minutes avant de le savoir.

### 2. Parallélisation

Exécutez les jobs indépendants en parallèle :

```yaml
# GitLab CI
test:unit:
  stage: test
  script: pytest tests/unit/

test:integration:
  stage: test
  script: pytest tests/integration/

# Ces deux jobs s'exécutent en parallèle
```

### 3. Caching intelligent

Réutilisez les dépendances entre builds :

```yaml
cache:
  key: ${CI_COMMIT_REF_SLUG}
  paths:
    - node_modules/
    - .pip-cache/
    - vendor/
```

### 4. Versioning automatique

Utilisez des tags automatiques basés sur le commit :

```bash
# Dans le pipeline
IMAGE_TAG="${CI_COMMIT_SHORT_SHA}-${CI_PIPELINE_ID}"
docker build -t myapp:$IMAGE_TAG .
```

### 5. Tests de qualité de code

Intégrez des outils de qualité :

```yaml
code_quality:
  stage: test
  script:
    - sonar-scanner
    - codecov
```

### 6. Notifications

Informez l'équipe automatiquement :

```yaml
notify:slack:
  stage: .post
  script:
    - 'curl -X POST -H "Content-type: application/json"
        --data "{\"text\":\"Build #${CI_PIPELINE_ID} succeeded!\"}"
        ${SLACK_WEBHOOK}'
  only:
    - main
  when: on_success
```

### 7. Rollback automatique

Détectez les problèmes et revenez en arrière :

```yaml
deploy:production:
  script:
    - kubectl apply -f k8s/
    - sleep 60
    - |
      if ! curl -f https://app.example.com/health; then
        kubectl rollout undo deployment/app
        exit 1
      fi
```

### 8. Environments séparés

Différenciez clairement les environnements :

```yaml
deploy:dev:
  environment:
    name: development
    url: https://dev.example.com
  only:
    - develop

deploy:prod:
  environment:
    name: production
    url: https://example.com
  only:
    - main
  when: manual
```

### 9. Secrets sécurisés

Ne jamais exposer de secrets dans les logs :

```yaml
variables:
  PASSWORD: $SECRET_PASSWORD  # Variable masquée

script:
  - echo $PASSWORD  # ❌ Ne pas faire ça
  - echo "Connexion à la base..."  # ✅ Bon
```

### 10. Documentation du pipeline

Commentez votre pipeline :

```yaml
# .gitlab-ci.yml

# Ce pipeline déploie l'application Flask vers Kubernetes
# Étapes :
# 1. Lint et test du code Python
# 2. Build de l'image Docker
# 3. Push vers le registry privé
# 4. Déploiement sur staging (automatique)
# 5. Déploiement sur production (manuel)

stages:
  - test
  - build
  - deploy
```

---

## Monitoring et observabilité du pipeline

### Métriques importantes

Suivez ces métriques pour évaluer votre pipeline :

1. **Build Duration** : Temps total du pipeline
2. **Success Rate** : Taux de réussite des builds
3. **MTTR** : Mean Time To Recovery (temps de correction)
4. **Deployment Frequency** : Fréquence de déploiement
5. **Lead Time** : Temps entre commit et production

### Alertes

Configurez des alertes pour :
- Builds échoués sur la branche principale
- Temps de build anormalement long
- Déploiements échoués
- Tests flaky (instables)

### Dashboards

Créez des dashboards pour visualiser :
- Historique des builds
- Temps par stage
- Taux d'échec par type de test
- Tendances de qualité du code

---

## Intégration avec GitOps

Le pipeline CI/CD se marie parfaitement avec GitOps :

```
┌─────────────┐       ┌──────────────┐       ┌───────────────┐
│   Git Push  │  -->  │  CI Pipeline │  -->  │  Git Update   │
│  (app code) │       │  (build/test)│       │  (manifests)  │
└─────────────┘       └──────────────┘       └───────────────┘
                              │
                              ↓
                      ┌───────────────┐
                      │ Push Registry │
                      └───────────────┘
                              │
                              ↓
                      ┌───────────────┐
                      │   ArgoCD      │
                      │ (sync K8s)    │
                      └───────────────┘
```

**Workflow GitOps** :
1. Le développeur pousse du code
2. CI/CD construit et teste
3. CI/CD crée l'image Docker
4. CI/CD met à jour le repository de manifestes avec la nouvelle version d'image
5. ArgoCD détecte le changement et synchronise Kubernetes

Nous verrons cela en détail dans la section suivante sur ArgoCD.

---

## Récapitulatif

### Points clés

**CI/CD, c'est** :
- L'automatisation complète du développement au déploiement
- La détection précoce des bugs
- Des déploiements rapides et fiables
- La réduction des erreurs humaines

**GitLab CI** :
- Configuration simple avec .gitlab-ci.yml
- Intégré à GitLab
- Idéal pour débuter
- Parfait pour GitOps

**Jenkins** :
- Très flexible et puissant
- Énorme écosystème de plugins
- Configuration via Jenkinsfile
- Convient aux workflows complexes

**Bonnes pratiques** :
- Fail fast avec tests rapides en premier
- Parallélisation des jobs indépendants
- Caching pour accélérer les builds
- Versioning automatique des images
- Notifications d'équipe
- Rollback automatique en cas de problème

### Bénéfices immédiats

Avec un pipeline CI/CD :
- ✅ Déploiements en quelques minutes au lieu d'heures
- ✅ Confiance dans la qualité du code
- ✅ Réduction des bugs en production
- ✅ Traçabilité complète des changements
- ✅ Moins de stress pour les équipes

---

## Prochaines étapes

Vous maîtrisez maintenant :
- Les principes DevOps et GitOps (17.1)
- L'intégration avec Git (17.2)
- La gestion d'images avec un registry (17.3)
- Les pipelines CI/CD (17.4)

La prochaine étape est **ArgoCD** : l'outil qui va boucler la boucle en synchronisant automatiquement vos manifestes Git avec votre cluster Kubernetes. C'est la pièce finale du puzzle GitOps !

**Prêt pour la synchronisation automatique Git → Kubernetes ?** Passons à la section 17.5 : ArgoCD pour GitOps !

⏭️ [ArgoCD pour GitOps](/17-devops-et-cicd/05-argocd-pour-gitops.md)
