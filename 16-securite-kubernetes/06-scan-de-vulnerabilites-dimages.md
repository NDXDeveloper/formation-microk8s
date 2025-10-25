🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 16.6 Scan de Vulnérabilités d'Images

## Introduction

Le scan de vulnérabilités d'images est le processus d'analyse des images de conteneurs pour détecter les failles de sécurité connues. C'est une étape cruciale dans la sécurisation de vos applications Kubernetes, car vos conteneurs sont construits à partir d'images qui peuvent contenir des logiciels vulnérables.

**Analogie simple :** Imaginez que vous achetez une maison. Avant de l'acheter, vous faites inspecter la structure, la plomberie, l'électricité pour détecter les problèmes. Le scan de vulnérabilités, c'est l'inspection de vos images Docker avant de les déployer dans votre cluster.

## Le Problème : Images Vulnérables

### Sources de Vulnérabilités

Les images de conteneurs peuvent contenir des vulnérabilités provenant de plusieurs sources :

```
┌─────────────────────────────────────────────┐
│         Image Docker                        │
│                                             │
│  ┌────────────────────────────────────────┐ │
│  │  Image de Base (ex: ubuntu:20.04)      │ │
│  │  ⚠️ Vulnérabilités du système d'OS     │ │
│  └────────────────────────────────────────┘ │
│                │                            │
│                ▼                            │
│  ┌────────────────────────────────────────┐ │
│  │  Dépendances Système (libc, openssl)   │ │
│  │  ⚠️ Vulnérabilités des bibliothèques   │ │
│  └────────────────────────────────────────┘ │
│                │                            │
│                ▼                            │
│  ┌────────────────────────────────────────┐ │
│  │  Runtime (Python, Node.js, Java)       │ │
│  │  ⚠️ Vulnérabilités du runtime          │ │
│  └────────────────────────────────────────┘ │
│                │                            │
│                ▼                            │
│  ┌────────────────────────────────────────┐ │
│  │  Dépendances Applicatives              │ │
│  │  (npm packages, pip packages, etc.)    │ │
│  │  ⚠️ Vulnérabilités des dépendances     │ │
│  └────────────────────────────────────────┘ │
│                │                            │
│                ▼                            │
│  ┌────────────────────────────────────────┐ │
│  │  Votre Code Application                │ │
│  │  ⚠️ Vulnérabilités du code custom      │ │
│  └────────────────────────────────────────┘ │
└─────────────────────────────────────────────┘
```

### Exemples de Vulnérabilités Courantes

1. **CVE-2021-44228 (Log4Shell)**
   - Bibliothèque Java Log4j
   - Exécution de code à distance
   - Impact : Critique

2. **CVE-2022-0492**
   - Kernel Linux
   - Escalade de privilèges depuis un conteneur
   - Impact : Élevé

3. **CVE-2021-3156 (Sudo Baron Samedit)**
   - Outil sudo
   - Escalade de privilèges locale
   - Impact : Élevé

### Conséquences d'une Image Vulnérable

```
Image Vulnérable Déployée
         │
         ▼
Attaquant Exploite la Vulnérabilité
         │
         ▼
┌────────────────────────────────┐
│  Scénarios Possibles :         │
│                                │
│  • Exécution de code arbitraire│
│  • Vol de données sensibles    │
│  • Escalade de privilèges      │
│  • Mouvement latéral           │
│  • Déni de service             │
│  • Cryptominage                │
└────────────────────────────────┘
```

## Comprendre les CVE

### Qu'est-ce qu'un CVE ?

**CVE (Common Vulnerabilities and Exposures)** : Identifiant unique pour une vulnérabilité de sécurité connue publiquement.

**Format :** `CVE-ANNÉE-NUMÉRO`
- Exemple : `CVE-2024-1234`

### Bases de Données CVE

Plusieurs organisations maintiennent des bases de données de vulnérabilités :

- **NVD** (National Vulnerability Database) - NIST
- **CVE.org** - MITRE Corporation
- **Vendor Security Advisories** - Red Hat, Ubuntu, Debian, etc.

### Scoring CVSS

**CVSS (Common Vulnerability Scoring System)** : Système standardisé pour évaluer la gravité d'une vulnérabilité.

**Score de 0 à 10 :**

| Score | Sévérité | Description |
|-------|----------|-------------|
| 0.0 | None | Aucune vulnérabilité |
| 0.1 - 3.9 | **Low** | Impact faible, difficile à exploiter |
| 4.0 - 6.9 | **Medium** | Impact modéré |
| 7.0 - 8.9 | **High** | Impact important, relativement facile à exploiter |
| 9.0 - 10.0 | **Critical** | Impact critique, facile à exploiter |

**Exemple de rapport :**
```
CVE-2024-1234
Sévérité : HIGH (8.8)
Package : openssl 1.1.1k
Description : Buffer overflow dans la validation des certificats
```

### Vecteur d'Attaque

Le score CVSS prend en compte plusieurs facteurs :

- **Attack Vector (AV)** : Network, Adjacent, Local, Physical
- **Attack Complexity (AC)** : Low, High
- **Privileges Required (PR)** : None, Low, High
- **User Interaction (UI)** : None, Required
- **Scope (S)** : Unchanged, Changed
- **Impact (CIA)** : Confidentiality, Integrity, Availability

## Outils de Scan

### 1. Trivy (Recommandé)

**Trivy** est l'outil le plus populaire et le plus complet pour scanner les images.

**Avantages :**
- ✅ Open source et gratuit
- ✅ Facile à utiliser
- ✅ Rapide
- ✅ Détecte : vulnérabilités OS, dépendances, secrets, IaC
- ✅ Mis à jour régulièrement
- ✅ Supporte de nombreux formats (Docker, OCI, etc.)

**Installation :**

```bash
# Linux
wget https://github.com/aquasecurity/trivy/releases/download/v0.48.0/trivy_0.48.0_Linux-64bit.tar.gz
tar zxvf trivy_0.48.0_Linux-64bit.tar.gz
sudo mv trivy /usr/local/bin/

# macOS
brew install trivy

# Avec Docker (pas d'installation)
docker run aquasec/trivy:latest
```

**Utilisation de base :**

```bash
# Scanner une image
trivy image nginx:latest

# Scanner avec sévérité minimale
trivy image --severity HIGH,CRITICAL nginx:latest

# Format JSON
trivy image -f json -o results.json nginx:latest

# Scanner une image locale
trivy image my-app:latest

# Scanner un fichier tar
trivy image --input image.tar

# Scanner avec cache mis à jour
trivy image --download-db-only
trivy image nginx:latest
```

**Exemple de sortie :**

```
nginx:latest (debian 11.6)

Total: 45 (UNKNOWN: 0, LOW: 15, MEDIUM: 20, HIGH: 8, CRITICAL: 2)

┌────────────────┬────────────────┬──────────┬────────────┬──────────────────┬─────────────────┐
│   Library      │ Vulnerability  │ Severity │  Installed │  Fixed Version   │     Title       │
│                │      ID        │          │  Version   │                  │                 │
├────────────────┼────────────────┼──────────┼────────────┼──────────────────┼─────────────────┤
│ libssl1.1      │ CVE-2023-0286  │ CRITICAL │ 1.1.1n-0   │ 1.1.1n-0+deb11u4 │ OpenSSL: X.509  │
│                │                │          │ +deb11u3   │                  │ Email Address   │
│                │                │          │            │                  │ Buffer Overflow │
├────────────────┼────────────────┼──────────┼────────────┼──────────────────┼─────────────────┤
│ libcurl4       │ CVE-2023-28322 │ HIGH     │ 7.74.0-1.3 │ 7.74.0-1.3       │ curl: more POST │
│                │                │          │ +deb11u7   │ +deb11u8         │ locking horror  │
└────────────────┴────────────────┴──────────┴────────────┴──────────────────┴─────────────────┘
```

### 2. Grype

**Grype** est développé par Anchore, simple et rapide.

**Installation :**

```bash
# Linux/macOS
curl -sSfL https://raw.githubusercontent.com/anchore/grype/main/install.sh | sh -s -- -b /usr/local/bin

# macOS
brew install grype
```

**Utilisation :**

```bash
# Scanner une image
grype nginx:latest

# Avec sévérité minimale
grype nginx:latest --fail-on high

# Format JSON
grype nginx:latest -o json > results.json

# Scanner un répertoire
grype dir:/path/to/project
```

### 3. Clair

**Clair** est un scanner open source développé par Red Hat/CoreOS.

**Caractéristiques :**
- Service avec API
- Utilisé par Quay.io
- Nécessite une base de données PostgreSQL

**Déploiement :**

```bash
# Via Docker Compose
docker-compose up -d

# Scanner avec clairctl
clairctl analyze nginx:latest
```

### 4. Snyk

**Snyk** est une solution commerciale avec version gratuite limitée.

**Avantages :**
- Interface web conviviale
- Intégration Git et CI/CD
- Suggestions de correction automatiques

**Utilisation :**

```bash
# Installation
npm install -g snyk

# Authentification
snyk auth

# Scanner une image
snyk container test nginx:latest

# Scanner avec correctifs suggérés
snyk container test nginx:latest --file=Dockerfile
```

### 5. Docker Scout

**Docker Scout** est intégré dans Docker Desktop (nouveauté).

**Utilisation :**

```bash
# Scanner une image
docker scout cves nginx:latest

# Comparer avec une version de base
docker scout compare --to nginx:latest nginx:1.24

# Voir les recommandations
docker scout recommendations nginx:latest
```

### Comparaison des Outils

| Outil | Gratuit | Facilité | Performance | Formats | CI/CD |
|-------|---------|----------|-------------|---------|-------|
| **Trivy** | ✅ | ⭐⭐⭐ | ⭐⭐⭐ | Image, Filesystem, Git | ✅ |
| **Grype** | ✅ | ⭐⭐⭐ | ⭐⭐⭐ | Image, Filesystem | ✅ |
| **Clair** | ✅ | ⭐⭐ | ⭐⭐ | Image (API) | ✅ |
| **Snyk** | 🆓 Limité | ⭐⭐⭐ | ⭐⭐ | Multiple | ✅ |
| **Docker Scout** | ✅ | ⭐⭐⭐ | ⭐⭐ | Image Docker | ⚠️ |

**Recommandation :** Commencez avec **Trivy** pour sa simplicité et sa complétude.

## Utilisation Avancée de Trivy

### Scanner Différents Targets

```bash
# Image Docker
trivy image nginx:latest

# Système de fichiers
trivy fs /path/to/project

# Repository Git
trivy repo https://github.com/user/repo

# Fichier SBOM
trivy sbom result.json

# Image distroless
trivy image gcr.io/distroless/static:latest

# Image privée (avec authentification Docker)
docker login registry.example.com
trivy image registry.example.com/private/app:latest
```

### Filtrer les Résultats

```bash
# Seulement HIGH et CRITICAL
trivy image --severity HIGH,CRITICAL nginx:latest

# Ignorer les non-fixées
trivy image --ignore-unfixed nginx:latest

# Par type de vulnérabilité
trivy image --vuln-type os nginx:latest
trivy image --vuln-type library nginx:latest

# Combiner les filtres
trivy image --severity CRITICAL --ignore-unfixed nginx:latest
```

### Formats de Sortie

```bash
# Table (par défaut)
trivy image nginx:latest

# JSON complet
trivy image -f json nginx:latest

# JSON pour CI/CD
trivy image -f json --severity HIGH,CRITICAL nginx:latest | jq .

# Sarif (pour GitHub)
trivy image -f sarif -o results.sarif nginx:latest

# Template personnalisé
trivy image --format template --template "@contrib/html.tpl" -o report.html nginx:latest

# CycloneDX (SBOM)
trivy image -f cyclonedx nginx:latest

# SPDX (SBOM)
trivy image -f spdx-json nginx:latest
```

### Politique de Vulnérabilités

Créer un fichier de configuration `.trivyignore` :

```
# Ignorer des CVE spécifiques (avec justification)
CVE-2023-1234  # Fixed in next release, low risk
CVE-2023-5678  # False positive for our use case

# Ignorer par pattern
CVE-2023-*     # Toutes les CVE de 2023 (pas recommandé)
```

Ou utiliser un fichier de politique YAML :

```yaml
# trivy-policy.yaml
ignore:
  - id: CVE-2023-1234
    reason: "Fixé dans la prochaine version, risque faible"
    expires: "2024-12-31"
  - id: CVE-2023-5678
    reason: "Faux positif pour notre cas d'usage"
    package: "openssl"
```

Utiliser la politique :

```bash
trivy image --ignorefile .trivyignore nginx:latest
```

### Scanner les Secrets

Trivy peut aussi détecter des secrets dans les images :

```bash
# Scanner les secrets
trivy image --scanners secret nginx:latest

# Scanner vulnérabilités ET secrets
trivy image --scanners vuln,secret nginx:latest

# Seulement les secrets
trivy image --scanners secret my-app:latest
```

**Détection de :**
- Clés API
- Tokens AWS
- Mots de passe
- Clés privées SSH
- Credentials database

## Scanner dans le Workflow de Développement

### Phase 1 : Développement Local

```bash
# Avant de build
trivy config Dockerfile

# Après le build
docker build -t my-app:dev .
trivy image my-app:dev

# Si des CRITICAL trouvées : corriger avant de push
```

### Phase 2 : CI/CD Pipeline

#### Exemple GitLab CI

```yaml
# .gitlab-ci.yml
stages:
  - build
  - scan
  - deploy

build:
  stage: build
  script:
    - docker build -t $CI_REGISTRY_IMAGE:$CI_COMMIT_SHA .
    - docker push $CI_REGISTRY_IMAGE:$CI_COMMIT_SHA

security-scan:
  stage: scan
  image:
    name: aquasec/trivy:latest
    entrypoint: [""]
  script:
    - trivy image --exit-code 1 --severity CRITICAL $CI_REGISTRY_IMAGE:$CI_COMMIT_SHA
    - trivy image --exit-code 0 --severity HIGH,MEDIUM,LOW $CI_REGISTRY_IMAGE:$CI_COMMIT_SHA
  allow_failure: false

deploy:
  stage: deploy
  script:
    - kubectl set image deployment/app app=$CI_REGISTRY_IMAGE:$CI_COMMIT_SHA
  only:
    - main
```

**Explications :**
- `--exit-code 1` : Échec du pipeline si des CRITICAL trouvées
- `--exit-code 0` : Afficher mais ne pas échouer pour les autres sévérités
- `allow_failure: false` : Bloquer le déploiement si scan échoue

#### Exemple GitHub Actions

```yaml
# .github/workflows/security.yml
name: Security Scan

on:
  push:
    branches: [ main ]
  pull_request:
    branches: [ main ]

jobs:
  trivy:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout code
        uses: actions/checkout@v3

      - name: Build image
        run: docker build -t my-app:${{ github.sha }} .

      - name: Run Trivy vulnerability scanner
        uses: aquasecurity/trivy-action@master
        with:
          image-ref: 'my-app:${{ github.sha }}'
          format: 'sarif'
          output: 'trivy-results.sarif'
          severity: 'CRITICAL,HIGH'

      - name: Upload Trivy results to GitHub Security
        uses: github/codeql-action/upload-sarif@v2
        with:
          sarif_file: 'trivy-results.sarif'

      - name: Fail on CRITICAL vulnerabilities
        uses: aquasecurity/trivy-action@master
        with:
          image-ref: 'my-app:${{ github.sha }}'
          exit-code: 1
          severity: 'CRITICAL'
```

#### Exemple Jenkins

```groovy
// Jenkinsfile
pipeline {
    agent any

    stages {
        stage('Build') {
            steps {
                script {
                    docker.build("my-app:${env.BUILD_ID}")
                }
            }
        }

        stage('Security Scan') {
            steps {
                script {
                    sh """
                        docker run --rm -v /var/run/docker.sock:/var/run/docker.sock \
                            aquasec/trivy:latest image \
                            --exit-code 1 \
                            --severity CRITICAL \
                            --no-progress \
                            my-app:${env.BUILD_ID}
                    """
                }
            }
        }

        stage('Deploy') {
            when {
                branch 'main'
            }
            steps {
                sh "kubectl set image deployment/app app=my-app:${env.BUILD_ID}"
            }
        }
    }
}
```

### Phase 3 : Registry Scanning

De nombreux registres d'images offrent un scan automatique :

#### Docker Hub

- Scan automatique pour les repos payants
- Via Docker Scout

#### Harbor

```yaml
# Harbor configuration
apiVersion: v1
kind: ConfigMap
metadata:
  name: harbor-scanner-trivy
data:
  SCANNER_TRIVY_SEVERITY: "CRITICAL,HIGH"
  SCANNER_TRIVY_VULN_TYPE: "os,library"
```

#### Google Container Registry (GCR)

```bash
# Activer Container Analysis
gcloud services enable containeranalysis.googleapis.com

# Scanner une image
gcloud container images list-tags gcr.io/my-project/my-image
```

#### Amazon ECR

```bash
# Activer le scan automatique
aws ecr put-image-scanning-configuration \
    --repository-name my-repo \
    --image-scanning-configuration scanOnPush=true

# Voir les résultats
aws ecr describe-image-scan-findings \
    --repository-name my-repo \
    --image-id imageTag=latest
```

### Phase 4 : Runtime Scanning

Scanner les images déjà déployées :

```bash
# Lister toutes les images dans le cluster
kubectl get pods --all-namespaces -o jsonpath='{range .items[*]}{.spec.containers[*].image}{"\n"}{end}' | sort -u

# Scanner chaque image
kubectl get pods --all-namespaces -o jsonpath='{range .items[*]}{.spec.containers[*].image}{"\n"}{end}' | sort -u | while read image; do
    echo "Scanning $image"
    trivy image --severity HIGH,CRITICAL "$image"
done
```

**Automatisation avec CronJob :**

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: cluster-image-scan
  namespace: security
spec:
  schedule: "0 2 * * *"  # Tous les jours à 2h
  jobTemplate:
    spec:
      template:
        spec:
          serviceAccountName: image-scanner
          containers:
          - name: scanner
            image: aquasec/trivy:latest
            command:
            - /bin/sh
            - -c
            - |
              # Récupérer toutes les images
              kubectl get pods --all-namespaces -o json | \
              jq -r '.items[].spec.containers[].image' | sort -u | \
              while read image; do
                echo "Scanning $image"
                trivy image --severity CRITICAL,HIGH "$image" || true
              done
          restartPolicy: OnFailure
```

## Interpréter les Résultats

### Exemple de Rapport Trivy

```
nginx:1.21 (debian 11.5)

Total: 156 (UNKNOWN: 0, LOW: 67, MEDIUM: 65, HIGH: 20, CRITICAL: 4)

┌────────────────┬────────────────┬──────────┬───────────────┬───────────────────┬────────────────────────────┐
│   Library      │ Vulnerability  │ Severity │   Installed   │   Fixed Version   │           Title            │
│                │      ID        │          │   Version     │                   │                            │
├────────────────┼────────────────┼──────────┼───────────────┼───────────────────┼────────────────────────────┤
│ libssl1.1      │ CVE-2023-0286  │ CRITICAL │ 1.1.1n-0      │ 1.1.1n-0+deb11u4  │ OpenSSL: X.509 email       │
│                │                │          │ +deb11u3      │                   │ address 4-byte buffer      │
│                │                │          │               │                   │ overflow                   │
├────────────────┼────────────────┼──────────┼───────────────┼───────────────────┼────────────────────────────┤
│ libssl1.1      │ CVE-2023-0465  │ CRITICAL │ 1.1.1n-0      │ 1.1.1n-0+deb11u4  │ Invalid certificate        │
│                │                │          │ +deb11u3      │                   │ policies in leaf           │
│                │                │          │               │                   │ certificates are silently  │
│                │                │          │               │                   │ ignored                    │
├────────────────┼────────────────┼──────────┼───────────────┼───────────────────┼────────────────────────────┤
│ curl           │ CVE-2023-28322 │ HIGH     │ 7.74.0-1.3    │ 7.74.0-1.3        │ curl: more POST locking    │
│                │                │          │ +deb11u7      │ +deb11u8          │ horror                     │
└────────────────┴────────────────┴──────────┴───────────────┴───────────────────┴────────────────────────────┘
```

### Que Regarder ?

1. **Sévérité** : Prioriser CRITICAL et HIGH
2. **Fixed Version** : Y a-t-il une version corrigée disponible ?
3. **Package** : Quel logiciel est affecté ?
4. **Titre** : Description de la vulnérabilité

### Décisions à Prendre

```
CRITICAL trouvé
    │
    ├─── Fixed Version disponible ?
    │    ├─── OUI → Mettre à jour immédiatement
    │    └─── NON →
    │         ├─── Exploitable dans votre contexte ?
    │         │    ├─── OUI → Ne pas déployer, chercher alternative
    │         │    └─── NON → Documenter + monitorer + workaround
    │         └─── Ajouter à .trivyignore avec justification
    │
HIGH trouvé
    │
    ├─── Fixed Version disponible ?
    │    ├─── OUI → Planifier mise à jour
    │    └─── NON → Évaluer le risque
    │
MEDIUM/LOW trouvé
    │
    └─── Planifier mise à jour lors de la prochaine release
```

### Faux Positifs

Parfois, Trivy détecte des vulnérabilités qui ne s'appliquent pas à votre cas :

**Exemple :**
```
CVE-2023-1234 in libcurl
Description: Vulnerability when using SOCKS proxy
Votre cas: Vous n'utilisez pas de proxy SOCKS
→ Faux positif ou risque non applicable
```

**Action :**
```
# .trivyignore
CVE-2023-1234  # N'utilisons pas SOCKS proxy, reviewed 2024-01-15
```

## Corriger les Vulnérabilités

### 1. Mettre à Jour l'Image de Base

**Problème :**
```dockerfile
FROM ubuntu:20.04  # Image obsolète
```

**Solution :**
```dockerfile
FROM ubuntu:22.04  # Version plus récente et corrigée
# Ou encore mieux
FROM ubuntu:latest  # Toujours la dernière (attention aux breaking changes)
```

### 2. Mettre à Jour les Packages Système

```dockerfile
FROM ubuntu:22.04

# Mettre à jour tous les packages
RUN apt-get update && \
    apt-get upgrade -y && \
    apt-get clean && \
    rm -rf /var/lib/apt/lists/*
```

**Attention :** Cela augmente la taille de l'image.

### 3. Utiliser des Images Minimales

**Problème :** Images complètes avec beaucoup de packages inutiles.

**Solutions :**

#### Alpine Linux

```dockerfile
FROM alpine:3.18
# Image très petite (~5 MB)
# Moins de packages = moins de vulnérabilités
```

#### Distroless

```dockerfile
FROM gcr.io/distroless/static:latest
# Seulement l'application, pas de shell, pas d'utilitaires
# Surface d'attaque minimale
```

**Exemple complet avec build multi-stage :**

```dockerfile
# Stage 1: Build
FROM golang:1.21 AS builder
WORKDIR /app
COPY . .
RUN go build -o myapp

# Stage 2: Runtime
FROM gcr.io/distroless/static:latest
COPY --from=builder /app/myapp /
USER nonroot:nonroot
ENTRYPOINT ["/myapp"]
```

### 4. Mettre à Jour les Dépendances Applicatives

#### Node.js

```dockerfile
FROM node:18-alpine

WORKDIR /app
COPY package*.json ./

# Audit et correction automatique
RUN npm audit fix

# Ou forcer les mises à jour
RUN npm update

RUN npm ci --only=production
COPY . .

USER node
CMD ["node", "server.js"]
```

#### Python

```dockerfile
FROM python:3.11-slim

WORKDIR /app
COPY requirements.txt .

# Vérifier les vulnérabilités
RUN pip install safety
RUN safety check -r requirements.txt

# Installer les dépendances
RUN pip install --no-cache-dir -r requirements.txt

COPY . .
USER nobody
CMD ["python", "app.py"]
```

#### Java

```dockerfile
FROM maven:3.9-eclipse-temurin-17 AS builder
WORKDIR /app
COPY pom.xml .
COPY src ./src

# OWASP Dependency Check
RUN mvn org.owasp:dependency-check-maven:check

RUN mvn clean package

FROM eclipse-temurin:17-jre-alpine
COPY --from=builder /app/target/app.jar /app.jar
USER 1000
ENTRYPOINT ["java", "-jar", "/app.jar"]
```

### 5. Supprimer les Packages Inutiles

```dockerfile
FROM ubuntu:22.04

# Installer seulement ce qui est nécessaire
RUN apt-get update && \
    apt-get install -y --no-install-recommends \
        curl \
        ca-certificates && \
    apt-get clean && \
    rm -rf /var/lib/apt/lists/*

# Ne PAS installer de packages de développement en production
# RUN apt-get install -y build-essential  # ❌ Pas en prod
```

## Bonnes Pratiques

### 1. Scanner à Chaque Build

```bash
# Intégrer dans le script de build
docker build -t my-app:latest .
trivy image --exit-code 1 --severity CRITICAL my-app:latest
docker push my-app:latest
```

### 2. Politique de Tolérance

Définir des seuils acceptables :

```yaml
# security-policy.yaml
thresholds:
  critical: 0     # Aucune CRITICAL autorisée
  high: 5         # Maximum 5 HIGH
  medium: 20      # Maximum 20 MEDIUM
  low: unlimited  # Pas de limite pour LOW
```

**Script de validation :**

```bash
#!/bin/bash
IMAGE=$1

# Scanner l'image
trivy image -f json $IMAGE > results.json

# Compter les vulnérabilités
CRITICAL=$(jq '[.Results[].Vulnerabilities[] | select(.Severity=="CRITICAL")] | length' results.json)
HIGH=$(jq '[.Results[].Vulnerabilities[] | select(.Severity=="HIGH")] | length' results.json)

echo "CRITICAL: $CRITICAL"
echo "HIGH: $HIGH"

# Vérifier les seuils
if [ $CRITICAL -gt 0 ]; then
    echo "❌ CRITICAL vulnerabilities found. Build rejected."
    exit 1
fi

if [ $HIGH -gt 5 ]; then
    echo "❌ Too many HIGH vulnerabilities ($HIGH > 5). Build rejected."
    exit 1
fi

echo "✅ Security checks passed"
```

### 3. Scan Régulier des Images Existantes

```bash
# CronJob pour scanner toutes les images du registry
0 2 * * * /usr/local/bin/scan-registry.sh
```

```bash
#!/bin/bash
# scan-registry.sh

REGISTRY="registry.example.com"

# Lister toutes les images
IMAGES=$(curl -s https://$REGISTRY/v2/_catalog | jq -r '.repositories[]')

for IMAGE in $IMAGES; do
    # Récupérer les tags
    TAGS=$(curl -s https://$REGISTRY/v2/$IMAGE/tags/list | jq -r '.tags[]')

    for TAG in $TAGS; do
        echo "Scanning $REGISTRY/$IMAGE:$TAG"
        trivy image --severity CRITICAL,HIGH $REGISTRY/$IMAGE:$TAG
    done
done
```

### 4. Garder les Images à Jour

```dockerfile
# Mauvais : version fixe obsolète
FROM nginx:1.18

# Bon : version récente fixe
FROM nginx:1.24

# Meilleur : dernière stable (attention aux breaking changes)
FROM nginx:stable-alpine
```

**Automatisation avec Renovate/Dependabot :**

```json
// renovate.json
{
  "extends": ["config:base"],
  "dockerfile": {
    "enabled": true
  },
  "schedule": ["every weekend"]
}
```

### 5. Utiliser des Tags Spécifiques

```yaml
# ❌ Mauvais : tag 'latest' non reproductible
image: nginx:latest

# ✅ Bon : tag spécifique ou digest
image: nginx:1.24.0
# Ou encore mieux avec le digest
image: nginx@sha256:abc123...
```

### 6. Signature d'Images

Signer vos images pour garantir leur intégrité :

```bash
# Avec Cosign (Sigstore)
cosign sign --key cosign.key my-app:v1.0.0

# Vérifier la signature
cosign verify --key cosign.pub my-app:v1.0.0
```

### 7. Admission Controller

Empêcher le déploiement d'images vulnérables automatiquement :

**Avec OPA Gatekeeper :**

```yaml
apiVersion: constraints.gatekeeper.sh/v1beta1
kind: AllowedImageScan
metadata:
  name: block-vulnerable-images
spec:
  match:
    kinds:
    - apiGroups: [""]
      kinds: ["Pod"]
  parameters:
    maxCritical: 0
    maxHigh: 0
```

### 8. SBOM (Software Bill of Materials)

Générer un inventaire de tous les composants :

```bash
# Générer un SBOM avec Trivy
trivy image -f cyclonedx my-app:latest > sbom.json

# Ou avec Syft
syft my-app:latest -o json > sbom.json
```

**Avantages :**
- Traçabilité complète
- Détection rapide en cas de nouvelle CVE
- Conformité réglementaire

## Registry Sécurisé avec MicroK8s

### Activer le Registry MicroK8s

```bash
# Activer le registry intégré
microk8s enable registry

# Le registry est disponible sur localhost:32000
```

### Intégrer Trivy avec le Registry

```bash
# Build et push
docker build -t localhost:32000/my-app:latest .

# Scanner avant push
trivy image localhost:32000/my-app:latest --exit-code 1 --severity CRITICAL

# Push si le scan passe
docker push localhost:32000/my-app:latest
```

### Harbor avec MicroK8s

Harbor est un registry avec scan intégré :

```bash
# Installer Harbor (nécessite Helm)
microk8s enable helm3

# Ajouter le repo Harbor
microk8s helm3 repo add harbor https://helm.goharbor.io

# Installer Harbor avec Trivy
microk8s helm3 install harbor harbor/harbor \
  --set expose.type=nodePort \
  --set persistence.enabled=false \
  --set trivy.enabled=true
```

**Accéder à Harbor :**
- URL: `https://<node-ip>:30003`
- User: `admin`
- Password: `Harbor12345` (par défaut)

**Scan automatique :**
Harbor scanne automatiquement toutes les images poussées avec Trivy.

## Monitoring et Alertes

### Métriques Prometheus

Exposer les métriques de scan :

```yaml
# ServiceMonitor pour Prometheus
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: image-vulnerabilities
spec:
  selector:
    matchLabels:
      app: trivy-exporter
  endpoints:
  - port: metrics
```

**Dashboard Grafana :**
- Nombre de vulnérabilités par sévérité
- Images les plus vulnérables
- Évolution dans le temps
- Alertes sur seuils dépassés

### Alertes

```yaml
# PrometheusRule
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: image-vulnerability-alerts
spec:
  groups:
  - name: vulnerabilities
    rules:
    - alert: CriticalVulnerabilitiesDetected
      expr: trivy_vulnerabilities{severity="CRITICAL"} > 0
      for: 5m
      annotations:
        summary: "Image {{ $labels.image }} has CRITICAL vulnerabilities"
        description: "{{ $value }} CRITICAL vulnerabilities detected"
```

## Checklist de Sécurité des Images

Avant de déployer une image en production :

- [ ] Scanner l'image avec Trivy ou équivalent
- [ ] Aucune vulnérabilité CRITICAL
- [ ] Moins de 5 vulnérabilités HIGH (ou selon politique)
- [ ] Image de base à jour (< 3 mois)
- [ ] Utiliser une image minimale (Alpine, Distroless)
- [ ] Multi-stage build pour réduire la taille
- [ ] Pas de secrets dans l'image
- [ ] Tag spécifique (pas `latest`)
- [ ] Signature de l'image (optionnel)
- [ ] SBOM généré (optionnel)
- [ ] Documentation des exceptions
- [ ] Scan intégré dans CI/CD
- [ ] Scan runtime régulier

## Conclusion

Le scan de vulnérabilités d'images est une pratique essentielle :

1. **Scanner tôt** : Dès le développement local
2. **Scanner souvent** : À chaque build et en production
3. **Automatiser** : Intégrer dans CI/CD
4. **Prioriser** : CRITICAL d'abord, puis HIGH
5. **Corriger** : Mettre à jour régulièrement
6. **Documenter** : Justifier les exceptions

**Formule de Sécurité des Images :**
```
Image Sécurisée = Base Minimale + Mises à Jour + Scan Régulier + Surveillance
```

**Points Clés à Retenir :**

- Utilisez **Trivy** comme outil de scan principal
- Intégrez le scan dans votre **pipeline CI/CD**
- Bloquez les déploiements si des vulnérabilités **CRITICAL** sont détectées
- Utilisez des **images minimales** (Alpine, Distroless)
- Maintenez vos images **à jour**
- Scannez régulièrement les images **en production**
- Générez des **SBOM** pour la traçabilité
- Documentez les **exceptions** et faux positifs
- Utilisez des **tags spécifiques**, pas `latest`
- Signez vos images pour garantir l'intégrité

**Prochaines Étapes :**

Dans les sections suivantes, nous explorerons :
- **16.7** : Secrets management avancé
- **16.8** : Audit logging
- **16.9** : Bonnes pratiques de sécurité globales

Le scan de vulnérabilités, combiné avec les Pod Security Standards, RBAC, ServiceAccounts et Network Policies, forme un système complet de sécurité pour vos applications Kubernetes.

⏭️ [Secrets management avancé](/16-securite-kubernetes/07-secrets-management-avance.md)
