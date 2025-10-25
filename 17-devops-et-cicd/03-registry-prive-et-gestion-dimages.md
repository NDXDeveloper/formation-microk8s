🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 17.3 Registry privé et gestion d'images

## Introduction

Maintenant que vos manifestes Kubernetes sont versionnés dans Git, il est temps de s'occuper de l'autre composant essentiel de votre pipeline : les **images Docker**. Dans ce chapitre, nous allons découvrir comment stocker et gérer vos images de conteneur de manière sécurisée et efficiente grâce à un **registry privé**.

MicroK8s facilite grandement cette tâche avec son addon `registry` intégré, qui vous permet de mettre en place un registry Docker privé en une seule commande. Nous verrons également comment construire, taguer et pousser vos images, ainsi que les bonnes pratiques pour les gérer.

---

## Qu'est-ce qu'un Registry Docker ?

### Définition simple

Un **registry Docker** (ou registre d'images) est un **serveur de stockage** pour les images de conteneurs Docker. C'est l'équivalent de ce que GitHub est pour le code : un endroit centralisé où vous pouvez :

- **Stocker** vos images Docker
- **Versionner** vos images avec des tags
- **Partager** vos images entre différents environnements
- **Déployer** vos applications depuis un emplacement fiable

### Registries publics vs privés

#### Registries publics

Les plus connus sont :
- **Docker Hub** : le registry par défaut de Docker (hub.docker.com)
- **GitHub Container Registry (ghcr.io)** : intégré à GitHub
- **Quay.io** : registry de Red Hat

**Avantages** :
- Gratuits pour les images publiques
- Faciles à utiliser
- Haute disponibilité

**Inconvénients** :
- Images publiques visibles par tous
- Limites de téléchargement (rate limiting)
- Dépendance à un service externe

#### Registries privés

Un registry **privé** est hébergé par vous-même. C'est ce que nous allons mettre en place avec MicroK8s.

**Avantages** :
- **Contrôle total** : vos images ne quittent jamais votre infrastructure
- **Pas de limites** : téléchargements illimités
- **Rapidité** : réseau local ultra-rapide
- **Sécurité** : images propriétaires protégées
- **Conformité** : respect des politiques de l'entreprise

**Inconvénients** :
- Vous devez le gérer et le maintenir
- Stockage sur votre infrastructure
- Sauvegarde de votre responsabilité

### Comment ça fonctionne ?

Le workflow avec un registry est simple :

```
1. BUILD    →  2. TAG      →  3. PUSH           →  4. PULL      →  5. RUN
   Docker       Nommage         Envoi au             Récup depuis    Déploiement
   build        de l'image      registry             le registry     dans K8s
```

Détaillons chaque étape :

1. **BUILD** : Vous construisez une image Docker depuis un Dockerfile
2. **TAG** : Vous donnez un nom et une version à votre image
3. **PUSH** : Vous envoyez l'image vers le registry
4. **PULL** : Kubernetes télécharge l'image depuis le registry
5. **RUN** : Kubernetes crée des conteneurs à partir de l'image

---

## L'addon Registry de MicroK8s

### Présentation

MicroK8s inclut un addon `registry` qui déploie un **registry Docker complet** directement dans votre cluster. C'est une implémentation du registry Docker officiel, pré-configurée et prête à l'emploi.

**Caractéristiques** :
- Registry Docker v2
- Stockage persistant automatique
- Accessible localement
- Configuration simplifiée

### Architecture

Une fois activé, le registry est déployé comme un pod dans votre cluster :

```
┌────────────────────────────────────┐
│         MicroK8s Cluster           │
│                                    │
│  ┌──────────────────────────────┐  │
│  │   Registry Pod               │  │
│  │   - Port: 32000              │  │
│  │   - Storage: /var/snap/...   │  │
│  └──────────────────────────────┘  │
│                                    │
│  ┌──────────────────────────────┐  │
│  │   Vos Applications           │  │
│  │   - Tirent depuis localhost  │  │
│  └──────────────────────────────┘  │
└────────────────────────────────────┘
```

### Activation du registry

Activer le registry est extrêmement simple :

```bash
microk8s enable registry
```

Cette commande va :
1. Créer un namespace `container-registry`
2. Déployer le registry comme un pod
3. Créer un service exposé sur le port 32000
4. Configurer le stockage persistant
5. Rendre le registry accessible à `localhost:32000`

**Vérification de l'installation** :

```bash
# Vérifier que le registry fonctionne
microk8s kubectl get all -n container-registry

# Vous devriez voir quelque chose comme :
# NAME                            READY   STATUS    RESTARTS   AGE
# pod/registry-6c99d8f8f8-xxxxx   1/1     Running   0          2m

# NAME               TYPE       CLUSTER-IP      EXTERNAL-IP   PORT(S)          AGE
# service/registry   NodePort   10.152.183.XXX  <none>        5000:32000/TCP   2m
```

**Tester l'accessibilité** :

```bash
curl http://localhost:32000/v2/
```

Vous devriez recevoir une réponse JSON :
```json
{}
```

Cela confirme que le registry est opérationnel !

### Configuration du registry

Le registry MicroK8s est pré-configuré pour fonctionner en HTTP sur localhost. Pour la plupart des usages de lab, cette configuration est suffisante.

**Emplacement du stockage** :

Les images sont stockées dans :
```
/var/snap/microk8s/common/var/lib/registry
```

**Capacité de stockage** :

Assurez-vous d'avoir suffisamment d'espace disque pour stocker vos images. Chaque image peut faire plusieurs centaines de Mo.

```bash
# Vérifier l'espace disponible
df -h /var/snap/microk8s/common/
```

---

## Construction et gestion d'images Docker

### Rappel : Qu'est-ce qu'une image Docker ?

Une **image Docker** est un **template** qui contient :
- Le système de fichiers de votre application
- Les dépendances et bibliothèques
- Le code de votre application
- Les configurations
- Les métadonnées (variables d'environnement, ports, etc.)

C'est à partir de cette image que Kubernetes va créer des **conteneurs** (instances en exécution).

### Le Dockerfile

Un **Dockerfile** est un fichier texte qui contient les instructions pour construire une image.

**Exemple simple** - Application Python Flask :

```dockerfile
# Partir d'une image de base
FROM python:3.11-slim

# Définir le répertoire de travail
WORKDIR /app

# Copier le fichier des dépendances
COPY requirements.txt .

# Installer les dépendances
RUN pip install --no-cache-dir -r requirements.txt

# Copier le code de l'application
COPY . .

# Exposer le port
EXPOSE 5000

# Commande de démarrage
CMD ["python", "app.py"]
```

**Exemple** - Application Node.js :

```dockerfile
FROM node:18-alpine

WORKDIR /usr/src/app

# Copier package.json et package-lock.json
COPY package*.json ./

# Installer les dépendances
RUN npm ci --only=production

# Copier le code source
COPY . .

EXPOSE 3000

CMD ["node", "server.js"]
```

### Construire une image

Pour construire une image à partir d'un Dockerfile :

```bash
# Syntaxe de base
docker build -t nom-image:tag .

# Exemple concret
docker build -t mon-app:v1.0.0 .
```

**Explication des paramètres** :
- `-t` : Tag (nom) de l'image
- `mon-app` : Nom de votre application
- `v1.0.0` : Version/tag
- `.` : Contexte de build (répertoire courant)

**Exemple de construction** :

```bash
cd /chemin/vers/mon-application

# Construire l'image
docker build -t mon-app-web:1.0.0 .

# Vérifier que l'image existe
docker images | grep mon-app-web
```

### Nommage et tagging des images

Le **tag** est crucial pour identifier précisément une version de votre image.

#### Anatomie d'un nom d'image complet

```
[registry]/[namespace]/[nom]:[tag]
```

Exemples :
```
localhost:32000/production/mon-app:v1.0.0
└──────┬───────┘ └────┬────┘ └──┬──┘ └──┬──┘
   Registry      Namespace    Nom     Tag
```

**Composants** :
- **Registry** : Adresse du registry (ex: `localhost:32000`)
- **Namespace** : Organisation/projet (optionnel)
- **Nom** : Nom de l'application
- **Tag** : Version ou identifiant

#### Stratégies de tagging

**1. Versioning sémantique** (recommandé) :
```
mon-app:1.0.0
mon-app:1.0.1
mon-app:1.1.0
mon-app:2.0.0
```

**2. Tag par environnement** :
```
mon-app:dev
mon-app:staging
mon-app:prod
```

**3. Tag par commit Git** (traçabilité maximale) :
```
mon-app:git-a1b2c3d
mon-app:git-d4e5f6g
```

**4. Tag par date** :
```
mon-app:2025-10-25
mon-app:20251025-143000
```

**5. Combinaison** (le plus complet) :
```
mon-app:v1.0.0-git-a1b2c3d
mon-app:v1.0.0-20251025
```

#### Tags spéciaux

**latest** :
```bash
docker tag mon-app:1.0.0 mon-app:latest
```

⚠️ **Attention** : Évitez d'utiliser `latest` en production car il est ambigu et peut changer au fil du temps.

**stable** :
```bash
docker tag mon-app:1.0.0 mon-app:stable
```

Utilisé pour marquer la dernière version stable.

### Pousser une image vers le registry

Une fois construite, vous devez **taguer** l'image avec l'adresse du registry, puis la **pousser**.

#### Étape 1 : Taguer l'image

```bash
# Syntaxe
docker tag IMAGE_SOURCE REGISTRY/IMAGE_DESTINATION:TAG

# Exemple
docker tag mon-app:1.0.0 localhost:32000/mon-app:1.0.0
```

Vous pouvez aussi construire directement avec le bon nom :

```bash
docker build -t localhost:32000/mon-app:1.0.0 .
```

#### Étape 2 : Pousser l'image

```bash
docker push localhost:32000/mon-app:1.0.0
```

**Exemple complet** :

```bash
# Construire l'image
docker build -t localhost:32000/production/web-app:v1.2.3 .

# Pousser vers le registry
docker push localhost:32000/production/web-app:v1.2.3

# Pousser aussi avec le tag 'latest' pour faciliter les tests
docker tag localhost:32000/production/web-app:v1.2.3 \
           localhost:32000/production/web-app:latest
docker push localhost:32000/production/web-app:latest
```

### Lister les images dans le registry

Le registry expose une API HTTP pour interroger son contenu :

```bash
# Lister tous les repositories
curl http://localhost:32000/v2/_catalog

# Réponse :
# {"repositories":["mon-app","production/web-app"]}

# Lister les tags d'une image
curl http://localhost:32000/v2/mon-app/tags/list

# Réponse :
# {"name":"mon-app","tags":["1.0.0","1.0.1","latest"]}
```

### Tirer (pull) une image depuis le registry

Pour récupérer une image depuis votre registry :

```bash
docker pull localhost:32000/mon-app:1.0.0
```

Kubernetes le fera automatiquement quand vous déploierez une application.

---

## Utiliser le registry dans vos déploiements Kubernetes

### Référencer une image du registry

Dans vos manifestes Kubernetes, vous spécifiez l'image à utiliser :

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mon-app
spec:
  replicas: 3
  selector:
    matchLabels:
      app: mon-app
  template:
    metadata:
      labels:
        app: mon-app
    spec:
      containers:
      - name: app
        image: localhost:32000/mon-app:1.0.0
        ports:
        - containerPort: 8080
```

**Point important** : L'image pointe vers `localhost:32000`, votre registry privé.

### Politique de pull d'images

Kubernetes a plusieurs politiques pour télécharger les images :

```yaml
spec:
  containers:
  - name: app
    image: localhost:32000/mon-app:1.0.0
    imagePullPolicy: Always  # Politique de pull
```

**Options disponibles** :

1. **Always** (Toujours) :
   - Kubernetes télécharge l'image à chaque fois
   - Utile en développement
   - Garantit la dernière version

2. **IfNotPresent** (Si pas présente) :
   - Kubernetes télécharge seulement si l'image n'existe pas localement
   - Économise de la bande passante
   - Recommandé avec des tags de version spécifiques

3. **Never** (Jamais) :
   - Kubernetes n'essaie jamais de télécharger l'image
   - Utilise uniquement les images locales
   - Rare, sauf cas spécifiques

**Recommandations** :

```yaml
# ✅ Bon : Version spécifique + IfNotPresent
image: localhost:32000/mon-app:1.0.0
imagePullPolicy: IfNotPresent

# ❌ Éviter : latest + IfNotPresent (peut utiliser une vieille version)
image: localhost:32000/mon-app:latest
imagePullPolicy: IfNotPresent

# ✅ Acceptable : latest + Always (force le téléchargement)
image: localhost:32000/mon-app:latest
imagePullPolicy: Always
```

### Exemple de déploiement complet

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-application
  namespace: production
  labels:
    app: web
    version: v1.0.0
spec:
  replicas: 3
  selector:
    matchLabels:
      app: web
  template:
    metadata:
      labels:
        app: web
        version: v1.0.0
    spec:
      containers:
      - name: web
        image: localhost:32000/production/web-app:v1.0.0
        imagePullPolicy: IfNotPresent
        ports:
        - containerPort: 8080
          name: http
        resources:
          requests:
            memory: "128Mi"
            cpu: "100m"
          limits:
            memory: "256Mi"
            cpu: "500m"
        livenessProbe:
          httpGet:
            path: /health
            port: 8080
          initialDelaySeconds: 30
          periodSeconds: 10
        readinessProbe:
          httpGet:
            path: /ready
            port: 8080
          initialDelaySeconds: 5
          periodSeconds: 5
---
apiVersion: v1
kind: Service
metadata:
  name: web-service
  namespace: production
spec:
  selector:
    app: web
  ports:
  - port: 80
    targetPort: 8080
  type: ClusterIP
```

---

## Workflow complet : Du code à la production

Voici un workflow complet qui intègre Git, Docker et Kubernetes :

### Scénario : Déployer une nouvelle version

#### 1. Développement et commit

```bash
# Modifier le code
vim app/main.py

# Commiter dans Git
git add app/main.py
git commit -m "Ajout de la fonctionnalité de recherche"
git push origin main
```

#### 2. Construction de l'image

```bash
# Se placer dans le répertoire du projet
cd mon-projet

# Construire l'image avec le nouveau code
docker build -t localhost:32000/mon-app:v1.1.0 .

# Aussi taguer comme 'latest'
docker tag localhost:32000/mon-app:v1.1.0 \
           localhost:32000/mon-app:latest
```

#### 3. Push vers le registry

```bash
# Pousser la nouvelle version
docker push localhost:32000/mon-app:v1.1.0

# Pousser le tag latest
docker push localhost:32000/mon-app:latest
```

#### 4. Mise à jour du manifeste Kubernetes

```bash
# Éditer le deployment
vim k8s/deployment.yaml

# Changer la version de l'image
# image: localhost:32000/mon-app:v1.0.0  # Ancienne
# image: localhost:32000/mon-app:v1.1.0  # Nouvelle

# Commiter le changement
git add k8s/deployment.yaml
git commit -m "Update app to v1.1.0"
git push origin main
```

#### 5. Déploiement

```bash
# Appliquer les changements
kubectl apply -f k8s/deployment.yaml

# Surveiller le déploiement
kubectl rollout status deployment/mon-app

# Vérifier que les nouveaux pods utilisent la nouvelle image
kubectl describe pod mon-app-xxx | grep Image
```

### Script d'automatisation

Créez un script pour automatiser ce workflow :

```bash
#!/bin/bash
# scripts/build-and-deploy.sh

set -e  # Arrêt en cas d'erreur

APP_NAME="mon-app"
VERSION=$1

if [ -z "$VERSION" ]; then
    echo "Usage: ./build-and-deploy.sh <version>"
    echo "Example: ./build-and-deploy.sh v1.2.0"
    exit 1
fi

REGISTRY="localhost:32000"
IMAGE="$REGISTRY/$APP_NAME:$VERSION"

echo "🏗️  Construction de l'image $IMAGE"
docker build -t $IMAGE .

echo "🏷️  Tagging latest"
docker tag $IMAGE $REGISTRY/$APP_NAME:latest

echo "📤 Push vers le registry"
docker push $IMAGE
docker push $REGISTRY/$APP_NAME:latest

echo "📝 Mise à jour du manifeste"
sed -i "s|image: $REGISTRY/$APP_NAME:.*|image: $IMAGE|" k8s/deployment.yaml

echo "📦 Commit Git"
git add k8s/deployment.yaml
git commit -m "Deploy $APP_NAME $VERSION"
git push origin main

echo "🚀 Déploiement dans Kubernetes"
kubectl apply -f k8s/deployment.yaml

echo "⏳ Attente du rollout"
kubectl rollout status deployment/$APP_NAME

echo "✅ Déploiement de $VERSION terminé avec succès!"
```

Utilisation :
```bash
./scripts/build-and-deploy.sh v1.2.0
```

---

## Bonnes pratiques pour les images Docker

### 1. Images légères

**Privilégiez les images de base minimales** :

❌ **Évitez** :
```dockerfile
FROM ubuntu:latest  # 77 MB
```

✅ **Préférez** :
```dockerfile
FROM alpine:latest  # 5 MB
# ou
FROM python:3.11-slim  # 45 MB vs python:3.11 (883 MB)
```

**Avantages** :
- Téléchargement plus rapide
- Moins de vulnérabilités
- Consommation moindre de stockage

### 2. Builds multi-étapes

Réduisez la taille finale en séparant le build de l'exécution :

```dockerfile
# Étape 1 : Build
FROM node:18 AS builder
WORKDIR /app
COPY package*.json ./
RUN npm install
COPY . .
RUN npm run build

# Étape 2 : Production (image finale)
FROM node:18-alpine
WORKDIR /app
COPY --from=builder /app/dist ./dist
COPY --from=builder /app/node_modules ./node_modules
CMD ["node", "dist/server.js"]
```

L'image finale ne contient que le strict nécessaire.

### 3. Cache des layers

Optimisez l'ordre des instructions pour maximiser le cache Docker :

✅ **Bon ordre** :
```dockerfile
FROM python:3.11-slim

# 1. Ce qui change rarement en premier
WORKDIR /app

# 2. Dépendances (cache réutilisé si pas de changement)
COPY requirements.txt .
RUN pip install -r requirements.txt

# 3. Code (change souvent) en dernier
COPY . .

CMD ["python", "app.py"]
```

### 4. Fichier .dockerignore

Créez un `.dockerignore` pour exclure les fichiers inutiles :

```
# .dockerignore
.git
.gitignore
README.md
.vscode
.idea
node_modules
*.log
*.tmp
.env
.DS_Store
*.md
Dockerfile
docker-compose.yml
```

Cela réduit le contexte de build et accélère la construction.

### 5. Pas de secrets dans les images

❌ **JAMAIS de secrets dans le Dockerfile** :

```dockerfile
# ❌ NE FAITES JAMAIS ÇA
ENV API_KEY=sk_live_123456789
ENV DB_PASSWORD=secretpassword
```

✅ **Utilisez les ConfigMaps et Secrets Kubernetes** :

```dockerfile
# Dockerfile propre
ENV API_KEY=""
ENV DB_PASSWORD=""
```

```yaml
# Deployment avec secrets injectés
spec:
  containers:
  - name: app
    image: localhost:32000/mon-app:1.0.0
    env:
    - name: API_KEY
      valueFrom:
        secretKeyRef:
          name: app-secrets
          key: api-key
    - name: DB_PASSWORD
      valueFrom:
        secretKeyRef:
          name: app-secrets
          key: db-password
```

### 6. Utilisateur non-root

Ne pas exécuter en tant que root améliore la sécurité :

```dockerfile
FROM python:3.11-slim

# Créer un utilisateur non-root
RUN useradd -m -u 1000 appuser

WORKDIR /app
COPY . .

# Changer de propriétaire
RUN chown -R appuser:appuser /app

# Basculer vers l'utilisateur
USER appuser

CMD ["python", "app.py"]
```

### 7. Labels et métadonnées

Ajoutez des métadonnées à vos images :

```dockerfile
FROM python:3.11-slim

LABEL maintainer="jean.dupont@example.com"
LABEL version="1.0.0"
LABEL description="Application web de gestion"
LABEL org.opencontainers.image.source="https://github.com/user/repo"

# ... reste du Dockerfile
```

### 8. Health checks

Intégrez des health checks dans votre image :

```dockerfile
FROM nginx:alpine

COPY nginx.conf /etc/nginx/nginx.conf
COPY app/ /usr/share/nginx/html/

HEALTHCHECK --interval=30s --timeout=3s \
  CMD wget --quiet --tries=1 --spider http://localhost/ || exit 1

EXPOSE 80
CMD ["nginx", "-g", "daemon off;"]
```

### 9. Versioning cohérent

Adoptez une stratégie de versioning claire :

```bash
# Build avec plusieurs tags
VERSION="1.2.3"
docker build -t localhost:32000/mon-app:$VERSION .
docker build -t localhost:32000/mon-app:1.2 .
docker build -t localhost:32000/mon-app:1 .
docker build -t localhost:32000/mon-app:latest .

# Push tous les tags
docker push localhost:32000/mon-app:$VERSION
docker push localhost:32000/mon-app:1.2
docker push localhost:32000/mon-app:1
docker push localhost:32000/mon-app:latest
```

---

## Gestion du cycle de vie des images

### Nettoyage régulier

Les images s'accumulent et consomment de l'espace disque.

#### Sur votre machine locale

```bash
# Supprimer les images non utilisées
docker image prune

# Supprimer toutes les images non utilisées (pas seulement les dangling)
docker image prune -a

# Supprimer les images plus anciennes que 24h
docker image prune -a --filter "until=24h"
```

#### Dans le registry

Le registry MicroK8s ne supprime pas automatiquement les anciennes images. Vous devez gérer manuellement le nettoyage.

**Script de nettoyage** :

```bash
#!/bin/bash
# scripts/clean-registry.sh

REGISTRY="localhost:32000"
REPO="mon-app"

echo "🗑️  Nettoyage du registry pour $REPO"

# Récupérer tous les tags
TAGS=$(curl -s http://$REGISTRY/v2/$REPO/tags/list | jq -r '.tags[]')

# Garder seulement les 5 derniers
KEEP_COUNT=5
TAG_COUNT=$(echo "$TAGS" | wc -l)

if [ $TAG_COUNT -gt $KEEP_COUNT ]; then
    TAGS_TO_DELETE=$(echo "$TAGS" | head -n -$KEEP_COUNT)

    for TAG in $TAGS_TO_DELETE; do
        echo "Suppression de $REPO:$TAG"
        # Récupérer le digest
        DIGEST=$(curl -I -s -H "Accept: application/vnd.docker.distribution.manifest.v2+json" \
                 http://$REGISTRY/v2/$REPO/manifests/$TAG | \
                 grep -i Docker-Content-Digest | awk '{print $2}' | tr -d '\r')

        # Supprimer
        curl -X DELETE http://$REGISTRY/v2/$REPO/manifests/$DIGEST
    done

    echo "✅ Nettoyage terminé"
else
    echo "ℹ️  Rien à nettoyer ($TAG_COUNT images)"
fi
```

### Scan de vulnérabilités

Scannez régulièrement vos images pour détecter les vulnérabilités :

```bash
# Avec Trivy (outil de scan populaire)
# Installation
sudo apt-get install trivy

# Scanner une image locale
trivy image localhost:32000/mon-app:1.0.0

# Scanner avec sévérité minimale
trivy image --severity HIGH,CRITICAL localhost:32000/mon-app:1.0.0

# Générer un rapport
trivy image -f json -o report.json localhost:32000/mon-app:1.0.0
```

### Signatures d'images

Pour garantir l'authenticité et l'intégrité de vos images, vous pouvez les signer :

```bash
# Avec cosign (outil de signature d'images)
cosign sign localhost:32000/mon-app:1.0.0

# Vérifier une signature
cosign verify localhost:32000/mon-app:1.0.0
```

---

## Registry haute disponibilité (avancé)

Pour un usage plus poussé, vous pouvez configurer un registry avec :

### Stockage externe

Au lieu du stockage local, utilisez un stockage objet (S3, MinIO) :

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: registry-config
data:
  config.yml: |
    version: 0.1
    storage:
      s3:
        accesskey: AKIAIOSFODNN7EXAMPLE
        secretkey: wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY
        region: us-east-1
        bucket: my-registry-bucket
```

### Authentification

Ajoutez une authentification pour sécuriser l'accès :

```bash
# Créer un fichier htpasswd
htpasswd -Bbn username password > htpasswd

# Créer un secret Kubernetes
kubectl create secret generic registry-auth \
  --from-file=htpasswd=htpasswd \
  -n container-registry
```

### Interface web

Ajoutez une interface web pour naviguer dans votre registry :

```bash
# Déployer Docker Registry UI
kubectl create deployment registry-ui \
  --image=joxit/docker-registry-ui:latest \
  -n container-registry
```

---

## Intégration avec les pipelines CI/CD

Dans les prochaines sections, vous verrez comment intégrer le registry dans vos pipelines automatisés :

```
Git Push → CI Build → Push Registry → GitOps Deploy → K8s Pull
```

Le registry devient le **point de passage obligatoire** entre votre CI (Continuous Integration) et votre CD (Continuous Deployment).

---

## Récapitulatif

### Points clés

**Le registry privé, c'est** :
- Un stockage sécurisé pour vos images Docker
- Un contrôle total sur vos artifacts
- Une rapidité maximale en réseau local
- L'indépendance vis-à-vis des services externes

**Avec MicroK8s** :
- Activation en une commande : `microk8s enable registry`
- Accessible sur `localhost:32000`
- Stockage persistant automatique
- Prêt pour la production

**Workflow complet** :
1. Écrire le code et le Dockerfile
2. Construire l'image : `docker build`
3. Taguer l'image : `docker tag`
4. Pousser au registry : `docker push`
5. Référencer dans K8s : `image: localhost:32000/...`
6. Déployer : `kubectl apply`

**Bonnes pratiques** :
- Images légères (Alpine, slim)
- Versioning sémantique
- Pas de secrets dans les images
- Utilisateur non-root
- Nettoyage régulier
- Scan de vulnérabilités

### Bénéfices immédiats

Avec votre registry privé :
- ✅ Vos images restent chez vous (sécurité)
- ✅ Déploiements ultra-rapides (réseau local)
- ✅ Pas de limites de téléchargement
- ✅ Traçabilité complète des versions
- ✅ Base solide pour l'automatisation

---

## Prochaines étapes

Maintenant que vous maîtrisez :
- La gestion de vos manifestes avec Git (17.2)
- Le stockage de vos images avec un registry privé (17.3)

Vous êtes prêt pour :

1. **Section 17.4** : Mettre en place un pipeline CI/CD complet avec GitLab CI ou Jenkins
2. **Section 17.5** : Déployer ArgoCD pour automatiser la synchronisation Git → Kubernetes
3. **Section 17.6** : Créer des Helm Charts pour packager vos applications

Le triptyque **Git + Registry + Kubernetes** est maintenant en place. Il ne reste plus qu'à automatiser tout ça !

**Prêt à construire votre premier pipeline CI/CD ?** Passons à la section 17.4 : Pipeline CI/CD !

⏭️ [Pipeline CI/CD (GitLab CI/Jenkins)](/17-devops-et-cicd/04-pipeline-cicd-gitlab-ci-jenkins.md)
