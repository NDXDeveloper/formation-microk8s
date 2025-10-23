🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 5.5 Registry (registre d'images privé - microk8s enable registry)

## Introduction

Le **Registry** est un addon qui vous permet d'héberger vos propres images Docker localement dans votre cluster MicroK8s. Si vous avez déjà utilisé Docker, vous connaissez probablement Docker Hub, le registre public où se trouvent des millions d'images. Le Registry MicroK8s vous permet de créer votre propre "mini Docker Hub" privé directement dans votre cluster.

## Qu'est-ce qu'un Registry Docker ?

### Dans le monde Docker

Un registre Docker (ou Docker Registry) est un **dépôt d'images de conteneurs**. C'est comme une bibliothèque où sont stockées toutes les images Docker que vous pouvez télécharger et utiliser.

Quand vous faites :
```bash
docker pull nginx
```

Docker télécharge l'image `nginx` depuis Docker Hub, le registre public par défaut.

### Le Registry dans MicroK8s

Le Registry addon crée un registre Docker **privé** qui tourne **à l'intérieur** de votre cluster MicroK8s. Vous pouvez y stocker vos propres images et les utiliser dans vos déploiements sans avoir besoin de Docker Hub ou d'un autre service externe.

### Analogie simple

Imaginez que vous êtes photographe :

- **Docker Hub (registre public)** = Getty Images ou Unsplash
  - Vous y trouvez des millions de photos professionnelles
  - Tout le monde peut les utiliser (dans certaines conditions)
  - Parfait pour des images standards et publiques

- **Registry privé** = Votre disque dur personnel
  - Vous y stockez VOS propres photos
  - Seul vous (et votre cluster) y avez accès
  - Parfait pour vos créations personnelles et projets en cours

Le Registry MicroK8s est votre "photothèque" personnelle pour les images Docker.

## Pourquoi utiliser un Registry privé ?

### 1. Développement local

Quand vous développez une application :
1. Vous modifiez le code
2. Vous construisez une nouvelle image Docker
3. Vous voulez la tester immédiatement dans Kubernetes

**Sans Registry privé :**
- Vous devez pousser l'image sur Docker Hub (lent, nécessite Internet)
- Ou configurer un registre externe compliqué
- Ou jongler avec des fichiers tar d'images

**Avec Registry privé :**
- Vous poussez l'image localement (rapide, pas d'Internet nécessaire)
- Elle est immédiatement disponible dans votre cluster
- Cycle de développement ultra-rapide

### 2. Images personnalisées

Vous voulez créer des images spécifiques à vos besoins :
- Une image avec vos outils de développement préférés
- Une image avec votre stack applicative préconfigurée
- Une image pour vos tests automatisés

Ces images ne sont pas destinées à être publiques. Un registre privé est idéal.

### 3. Confidentialité

Vos images peuvent contenir :
- Du code propriétaire
- Des configurations spécifiques à votre entreprise
- Des secrets ou informations sensibles (même si ce n'est pas recommandé)

Un registre privé garantit que ces images restent sur votre infrastructure.

### 4. Rapidité

Les images sont stockées localement, donc :
- **Pas de téléchargement depuis Internet** : Déploiement ultra-rapide
- **Pas de limite de bande passante** : Idéal pour les gros projets
- **Pas de dépendance externe** : Fonctionne même sans connexion Internet

### 5. Apprentissage

Pour un lab personnel, avoir un registre privé vous permet d'apprendre :
- Comment fonctionnent les registres Docker
- Comment gérer vos propres images
- Les workflows CI/CD complets

## Activation du Registry

L'activation du Registry avec MicroK8s est simple, mais offre quelques options de configuration.

### Commande d'activation basique

```bash
microk8s enable registry
```

Cette commande active un registre avec une configuration par défaut (stockage de 20Go).

### Commande avec configuration de la taille

Vous pouvez spécifier la taille du stockage du registre :

```bash
microk8s enable registry:size=40Gi
```

Cela crée un registre avec 40Go d'espace de stockage pour vos images.

**Recommandations de taille :**
- **Lab personnel simple** : 20Gi (défaut) est suffisant
- **Développement actif** : 40-50Gi recommandé
- **Projets multiples** : 100Gi ou plus

Une image Docker typique fait entre 100Mo et 1Go, donc 20Gi permet de stocker environ 20-200 images selon leur taille.

### Ce qui se passe lors de l'activation

Quand vous activez le Registry, MicroK8s :

1. **Crée un PersistentVolumeClaim** pour stocker les images
2. **Déploie le registre Docker** comme un pod
3. **Crée un Service** pour exposer le registre
4. **Configure** le registre pour accepter les connexions HTTP (localhost uniquement)
5. **Ajoute** le registre à la liste des registres de confiance de MicroK8s

**Sortie attendue :**
```
Infer repository core for addon registry
Addon core/storage is already enabled
Enabling the private registry
Applying registry manifest
namespace/container-registry created
persistentvolumeclaim/registry-claim created
deployment.apps/registry created
service/registry created
configmap/local-registry-hosting created
The registry is enabled
```

### Temps d'activation

L'activation du Registry prend généralement **1 à 2 minutes**, le temps de :
- Télécharger l'image du registre
- Créer le volume de stockage
- Démarrer le pod

### Vérification de l'activation

Pour vérifier que le Registry est bien activé :

```bash
microk8s status
```

Vous devriez voir :
```
enabled:
  registry             # (core) Private image registry exposed on localhost:32000
```

### Vérifier que le pod Registry fonctionne

```bash
microk8s kubectl get pods -n container-registry
```

Sortie attendue :
```
NAME                        READY   STATUS    RESTARTS   AGE
registry-xxxxxxxxxx-xxxxx   1/1     Running   0          2m
```

Le pod doit être en état `Running`.

### Vérifier le service Registry

```bash
microk8s kubectl get service -n container-registry
```

Sortie attendue :
```
NAME       TYPE       CLUSTER-IP      EXTERNAL-IP   PORT(S)          AGE
registry   NodePort   10.152.183.50   <none>        5000:32000/TCP   2m
```

Le registre est exposé sur le port **32000** de votre machine hôte.

## Accéder au Registry

Le Registry MicroK8s est accessible de deux manières :

### 1. Depuis votre machine hôte

Le registre est accessible sur :
```
localhost:32000
```

Vous pouvez utiliser cette adresse avec les commandes Docker standard.

### 2. Depuis les pods du cluster

Les pods dans le cluster peuvent accéder au registre via :
```
registry.container-registry.svc.cluster.local:5000
```

Mais généralement, vous utiliserez simplement `localhost:32000` dans les manifestes, car MicroK8s gère automatiquement la résolution.

## Utiliser le Registry : Workflow complet

Voyons comment utiliser le Registry dans un workflow réel de développement.

### Étape 1 : Créer une application

Créons une application web simple avec Node.js.

**Fichier `app.js` :**
```javascript
const http = require('http');

const server = http.createServer((req, res) => {
  res.writeHead(200, {'Content-Type': 'text/html'});
  res.end('<h1>Hello from my private registry!</h1>');
});

server.listen(3000, () => {
  console.log('Server running on port 3000');
});
```

**Fichier `package.json` :**
```json
{
  "name": "my-app",
  "version": "1.0.0",
  "description": "Test app for private registry",
  "main": "app.js",
  "scripts": {
    "start": "node app.js"
  }
}
```

### Étape 2 : Créer un Dockerfile

**Fichier `Dockerfile` :**
```dockerfile
FROM node:18-alpine

WORKDIR /app

COPY package.json .
COPY app.js .

EXPOSE 3000

CMD ["npm", "start"]
```

### Étape 3 : Construire l'image Docker

Construisez votre image Docker **avec le préfixe du registre local** :

```bash
docker build -t localhost:32000/my-app:1.0 .
```

**Explication du nom :**
- `localhost:32000` : L'adresse du registre privé
- `/my-app` : Le nom de votre image
- `:1.0` : Le tag de version

**Sortie attendue :**
```
[+] Building 5.2s (9/9) FINISHED
 => [internal] load build definition from Dockerfile
 => => transferring dockerfile: 143B
 => [internal] load .dockerignore
...
 => => naming to localhost:32000/my-app:1.0
```

### Étape 4 : Pousser l'image vers le Registry privé

Maintenant, poussez votre image vers le registre privé :

```bash
docker push localhost:32000/my-app:1.0
```

**Sortie attendue :**
```
The push refers to repository [localhost:32000/my-app]
abc123def456: Pushed
789ghi012jkl: Pushed
...
1.0: digest: sha256:abc123... size: 1234
```

Votre image est maintenant stockée dans votre registre privé !

### Étape 5 : Vérifier que l'image est dans le Registry

Vous pouvez lister les images dans votre registre avec une requête HTTP :

```bash
curl http://localhost:32000/v2/_catalog
```

**Sortie attendue :**
```json
{"repositories":["my-app"]}
```

Pour voir les tags d'une image :
```bash
curl http://localhost:32000/v2/my-app/tags/list
```

**Sortie attendue :**
```json
{"name":"my-app","tags":["1.0"]}
```

### Étape 6 : Déployer l'image dans Kubernetes

Créez un manifeste Kubernetes pour déployer votre application.

**Fichier `deployment.yaml` :**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
spec:
  replicas: 2
  selector:
    matchLabels:
      app: my-app
  template:
    metadata:
      labels:
        app: my-app
    spec:
      containers:
      - name: my-app
        image: localhost:32000/my-app:1.0
        ports:
        - containerPort: 3000
---
apiVersion: v1
kind: Service
metadata:
  name: my-app
spec:
  type: NodePort
  selector:
    app: my-app
  ports:
  - port: 3000
    targetPort: 3000
    nodePort: 30080
```

**Point important :** Notez l'image : `localhost:32000/my-app:1.0`. C'est ainsi que vous référencez une image de votre registre privé.

Déployez :
```bash
microk8s kubectl apply -f deployment.yaml
```

### Étape 7 : Vérifier le déploiement

```bash
microk8s kubectl get pods
```

Vous devriez voir vos pods en état `Running` :
```
NAME                      READY   STATUS    RESTARTS   AGE
my-app-xxxxxxxxxx-xxxxx   1/1     Running   0          30s
my-app-xxxxxxxxxx-yyyyy   1/1     Running   0          30s
```

### Étape 8 : Tester l'application

Accédez à votre application :
```bash
curl http://localhost:30080
```

Vous devriez voir :
```html
<h1>Hello from my private registry!</h1>
```

**Félicitations !** Vous avez :
1. Créé une application
2. Construit une image Docker
3. Poussé l'image dans votre registre privé
4. Déployé l'image depuis votre registre privé
5. Testé l'application qui tourne

## Workflow de mise à jour

Un des grands avantages du registre privé est la rapidité des mises à jour. Voici le workflow :

### 1. Modifier votre code

Modifiez `app.js` :
```javascript
const http = require('http');

const server = http.createServer((req, res) => {
  res.writeHead(200, {'Content-Type': 'text/html'});
  res.end('<h1>Hello from my private registry - Version 2.0!</h1>');
});

server.listen(3000, () => {
  console.log('Server running on port 3000');
});
```

### 2. Reconstruire l'image avec un nouveau tag

```bash
docker build -t localhost:32000/my-app:2.0 .
```

### 3. Pousser la nouvelle version

```bash
docker push localhost:32000/my-app:2.0
```

### 4. Mettre à jour le déploiement

Deux options :

**Option A : Modifier le YAML et réappliquer**
Changez `image: localhost:32000/my-app:1.0` en `image: localhost:32000/my-app:2.0` puis :
```bash
microk8s kubectl apply -f deployment.yaml
```

**Option B : Utiliser set image (plus rapide)**
```bash
microk8s kubectl set image deployment/my-app my-app=localhost:32000/my-app:2.0
```

### 5. Vérifier le rolling update

```bash
microk8s kubectl rollout status deployment/my-app
```

Kubernetes va progressivement remplacer les anciens pods par les nouveaux. Vous verrez :
```
Waiting for deployment "my-app" rollout to finish: 1 out of 2 new replicas have been updated...
Waiting for deployment "my-app" rollout to finish: 1 old replicas are pending termination...
deployment "my-app" successfully rolled out
```

Testez :
```bash
curl http://localhost:30080
```

Vous verrez maintenant :
```html
<h1>Hello from my private registry - Version 2.0!</h1>
```

**Le cycle complet prend moins d'une minute !** C'est la puissance du registre privé local.

## Stratégies de tagging

Le tagging des images est important pour gérer vos versions. Voici quelques stratégies :

### 1. Versioning sémantique

Utilisez des versions au format `MAJOR.MINOR.PATCH` :
```
localhost:32000/my-app:1.0.0
localhost:32000/my-app:1.0.1
localhost:32000/my-app:1.1.0
localhost:32000/my-app:2.0.0
```

### 2. Tags d'environnement

Créez des tags pour chaque environnement :
```
localhost:32000/my-app:dev
localhost:32000/my-app:staging
localhost:32000/my-app:prod
```

### 3. Combinaison version + sha

Pour la traçabilité complète :
```
localhost:32000/my-app:1.0.0-abc1234
```

Où `abc1234` est le hash du commit Git.

### 4. Le tag 'latest'

```
localhost:32000/my-app:latest
```

**⚠️ Attention :** Le tag `latest` n'est pas automatiquement la dernière version ! Vous devez le pousser explicitement :
```bash
docker build -t localhost:32000/my-app:latest .
docker push localhost:32000/my-app:latest
```

**Bonne pratique :** En production, évitez `latest`. Utilisez toujours des versions spécifiques pour la traçabilité.

## Gestion du stockage

Les images peuvent rapidement consommer de l'espace. Voici comment gérer le stockage.

### Voir l'utilisation du stockage

```bash
microk8s kubectl get pvc -n container-registry
```

Sortie :
```
NAME             STATUS   VOLUME                                     CAPACITY   ACCESS MODES
registry-claim   Bound    pvc-abc123...                              20Gi       RWO
```

### Nettoyer les anciennes images

Le registre garde toutes les images poussées. Pour libérer de l'espace, vous devez :

1. **Identifier les images à supprimer**

Listez les images :
```bash
curl http://localhost:32000/v2/_catalog
```

Listez les tags d'une image :
```bash
curl http://localhost:32000/v2/my-app/tags/list
```

2. **Supprimer une image spécifique**

La suppression d'images dans un Docker Registry nécessite de connaître le digest de l'image :

```bash
# Obtenir le digest
curl -I -H "Accept: application/vnd.docker.distribution.manifest.v2+json" \
  http://localhost:32000/v2/my-app/manifests/1.0

# Dans le header, cherchez Docker-Content-Digest: sha256:abc123...

# Supprimer avec le digest
curl -X DELETE http://localhost:32000/v2/my-app/manifests/sha256:abc123...
```

3. **Garbage collection**

Après suppression, les données restent sur le disque. Pour les supprimer définitivement :

```bash
# Se connecter au pod du registre
microk8s kubectl exec -it -n container-registry deployment/registry -- sh

# Lancer le garbage collector
registry garbage-collect /etc/docker/registry/config.yml

# Sortir
exit
```

### Augmenter la taille du stockage

Si vous manquez d'espace, vous devez :

1. Désactiver le registre
2. Le réactiver avec plus d'espace

```bash
microk8s disable registry
microk8s enable registry:size=50Gi
```

**⚠️ Attention :** Cela supprime toutes les images du registre ! Assurez-vous de les sauvegarder si nécessaire.

## Utilisation avancée

### Authentification (pour plus de sécurité)

Par défaut, le Registry MicroK8s n'a pas d'authentification. Pour un lab personnel, c'est acceptable. Mais vous pouvez ajouter une authentification basique :

1. Créer un fichier de mots de passe
2. Créer un Secret Kubernetes
3. Modifier la configuration du Registry

C'est avancé et rarement nécessaire pour un usage local, mais c'est possible.

### Accès depuis l'extérieur

Par défaut, le registre est accessible uniquement depuis `localhost`. Pour y accéder depuis d'autres machines :

**Option 1 : Port-forward**
```bash
microk8s kubectl port-forward -n container-registry service/registry 32000:5000 --address 0.0.0.0
```

Puis accédez depuis une autre machine :
```
http://[IP-de-votre-serveur]:32000
```

**Option 2 : Utiliser un Ingress**

Créez un Ingress pour exposer le registre (nécessite l'addon ingress activé) :
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: registry-ingress
  namespace: container-registry
spec:
  rules:
  - host: registry.local
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: registry
            port:
              number: 5000
```

### Miroir de Docker Hub

Vous pouvez configurer votre registre pour servir de cache/miroir pour Docker Hub. Cela accélère les pulls d'images publiques.

C'est une fonctionnalité avancée qui nécessite de modifier la ConfigMap du registre.

## Intégration avec des outils de développement

### Avec Docker Compose

Si vous utilisez Docker Compose pour le développement, vous pouvez référencer vos images du registre privé :

```yaml
version: '3'
services:
  app:
    image: localhost:32000/my-app:latest
    ports:
      - "3000:3000"
```

### Avec des outils CI/CD

Dans un pipeline CI/CD (GitLab CI, GitHub Actions, Jenkins), vous pouvez :

1. Construire l'image
2. La pousser vers votre registre privé
3. Déclencher un redéploiement dans MicroK8s

**Exemple avec un script simple :**
```bash
#!/bin/bash
# build-and-deploy.sh

VERSION=$(git describe --tags --always)
IMAGE="localhost:32000/my-app:$VERSION"

echo "Building $IMAGE..."
docker build -t $IMAGE .

echo "Pushing to registry..."
docker push $IMAGE

echo "Deploying to Kubernetes..."
microk8s kubectl set image deployment/my-app my-app=$IMAGE

echo "Done!"
```

## Dépannage

### Impossible de pousser une image

**Symptôme :** `docker push` échoue avec une erreur de connexion

**Causes possibles :**
1. Le Registry n'est pas démarré
2. Le port 32000 n'est pas accessible
3. Docker n'a pas confiance en localhost:32000

**Solutions :**

1. **Vérifier que le Registry tourne :**
```bash
microk8s kubectl get pods -n container-registry
```

2. **Vérifier que le port est ouvert :**
```bash
curl http://localhost:32000/v2/_catalog
```

Devrait retourner `{"repositories":[]}`  ou une liste d'images.

3. **Configurer Docker pour faire confiance au registre :**

Éditez `/etc/docker/daemon.json` (créez-le s'il n'existe pas) :
```json
{
  "insecure-registries": ["localhost:32000"]
}
```

Puis redémarrez Docker :
```bash
sudo systemctl restart docker
```

### Les pods ne peuvent pas pull l'image

**Symptôme :** Pod en état `ImagePullBackOff` ou `ErrImagePull`

**Diagnostic :**
```bash
microk8s kubectl describe pod <nom-du-pod>
```

Cherchez les événements d'erreur.

**Causes possibles :**
1. L'image n'existe pas dans le registre
2. Le nom de l'image est incorrect
3. Le registre n'est pas accessible depuis les pods

**Solutions :**

1. **Vérifier que l'image existe :**
```bash
curl http://localhost:32000/v2/_catalog
curl http://localhost:32000/v2/<nom-image>/tags/list
```

2. **Vérifier le nom dans le manifeste :**
Assurez-vous d'utiliser `localhost:32000/nom-image:tag`

3. **Tester l'accès depuis un pod :**
```bash
microk8s kubectl run test --image=busybox:1.28 --rm -it --restart=Never -- sh
wget -O- http://registry.container-registry.svc.cluster.local:5000/v2/_catalog
exit
```

### Le registre manque d'espace

**Symptôme :** Erreur lors du push : "no space left on device"

**Solutions :**

1. **Vérifier l'espace utilisé :**
```bash
microk8s kubectl exec -n container-registry deployment/registry -- df -h /var/lib/registry
```

2. **Nettoyer les vieilles images** (voir section précédente)

3. **Augmenter la taille :**
```bash
microk8s disable registry
microk8s enable registry:size=50Gi
```

### Le registre est lent

**Causes possibles :**
1. Disque lent (utilisation d'un HDD au lieu d'un SSD)
2. Images très volumineuses
3. Ressources système limitées

**Solutions :**
1. Utilisez un SSD pour le stockage de MicroK8s
2. Optimisez vos images Docker (multi-stage builds, layers partagés)
3. Augmentez les ressources allouées à MicroK8s

## Bonnes pratiques

### 1. Toujours taguer vos images avec des versions

**Faites :**
```bash
docker build -t localhost:32000/my-app:1.0.0 .
docker build -t localhost:32000/my-app:1.0.1 .
```

**Ne faites pas :**
```bash
docker build -t localhost:32000/my-app .  # Tag implicite 'latest'
```

### 2. Nettoyez régulièrement

Ne laissez pas des centaines d'anciennes images s'accumuler. Supprimez-les régulièrement.

### 3. Utilisez des noms cohérents

Établissez une convention de nommage :
```
localhost:32000/<projet>-<composant>:<version>

Exemples :
localhost:32000/blog-frontend:1.0.0
localhost:32000/blog-api:1.0.0
localhost:32000/blog-worker:1.0.0
```

### 4. Documentez vos images

Créez un fichier README ou une documentation listant :
- Les images disponibles dans votre registre
- À quoi elles servent
- Comment les construire
- Les dépendances

### 5. Automatisez

Créez des scripts pour automatiser :
- La construction des images
- Le push vers le registre
- Le déploiement

Cela réduit les erreurs et accélère le développement.

### 6. Sauvegardez les images importantes

Le registre utilise un volume persistant, mais en cas de problème majeur, vous pourriez perdre vos images. Pour les images importantes :

```bash
# Sauvegarder une image
docker save localhost:32000/my-app:1.0.0 > my-app-1.0.0.tar

# Restaurer
docker load < my-app-1.0.0.tar
docker push localhost:32000/my-app:1.0.0
```

## Alternatives au Registry intégré

Le Registry MicroK8s est excellent pour un usage local, mais voici des alternatives selon vos besoins :

### Harbor

Un registre d'entreprise avec :
- Authentification avancée
- Scan de vulnérabilités
- Réplication
- Interface web riche

Plus complexe, mais très puissant.

### Docker Hub

Le registre public de Docker. Gratuit pour les images publiques, payant pour les images privées.

Avantages :
- Accessible de partout
- Intégré à l'écosystème Docker
- Fiable et rapide

Inconvénients :
- Nécessite Internet
- Limites de pull pour les comptes gratuits
- Images privées payantes

### GitLab Container Registry

Si vous utilisez GitLab, il offre un registre intégré à chaque projet.

### GitHub Container Registry (ghcr.io)

Si vous utilisez GitHub, vous avez accès à un registre gratuit.

**Pour un lab personnel, le Registry MicroK8s est idéal** car il est local, rapide, et ne nécessite aucune configuration externe.

## Récapitulatif

Le Registry privé est essentiel pour :

✅ **Héberger** vos images Docker localement
✅ **Accélérer** le cycle de développement
✅ **Tester** rapidement vos modifications
✅ **Garder** vos images privées
✅ **Économiser** de la bande passante

**Commandes clés à retenir :**
```bash
# Activer le Registry
microk8s enable registry

# Construire et pousser une image
docker build -t localhost:32000/mon-app:1.0 .
docker push localhost:32000/mon-app:1.0

# Lister les images
curl http://localhost:32000/v2/_catalog

# Déployer depuis le registre privé
# Dans votre YAML : image: localhost:32000/mon-app:1.0
```

## Ce que vous avez appris

Dans cette section, vous avez découvert :
- Ce qu'est un Docker Registry et son utilité
- Comment activer le Registry dans MicroK8s
- Le workflow complet : build → push → deploy
- Comment mettre à jour vos applications rapidement
- Les stratégies de tagging des images
- La gestion du stockage du registre
- Comment résoudre les problèmes courants
- Les bonnes pratiques d'utilisation

## Prochaine étape

Avec le Dashboard pour visualiser, le DNS pour communiquer, et le Registry pour vos images, il ne manque plus qu'une pièce essentielle : le **Storage** (section 5.6).

Le Storage vous permettra de persister des données même quand vos pods redémarrent. C'est indispensable pour :
- Les bases de données
- Les fichiers uploadés par les utilisateurs
- Les logs et données applicatives
- Tout ce qui doit survivre aux redémarrages

Continuons pour compléter votre cluster avec ce dernier addon fondamental !

---

**Astuce pratique :** Créez un alias pour simplifier vos commandes Docker avec le registre privé :
```bash
alias dblr='docker build -t localhost:32000'
alias dplr='docker push localhost:32000'

# Utilisation :
dblr/my-app:1.0 .
dplr/my-app:1.0
```

⏭️ [Storage (stockage persistant - microk8s enable hostpath-storage)](/05-addons-microk8s/06-storage-stockage-persistant.md)
