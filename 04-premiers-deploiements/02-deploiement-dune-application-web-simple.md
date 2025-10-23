🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 4.2 Déploiement d'une Application Web Simple

## Introduction

Maintenant que vous comprenez l'anatomie d'un manifeste YAML, il est temps de passer à la pratique ! Dans cette section, nous allons déployer notre première application web sur MicroK8s. Nous utiliserons **Nginx**, un serveur web léger et populaire, comme exemple.

Nous allons découvrir deux approches :
1. **Le déploiement d'un Pod simple** (pour comprendre les bases)
2. **Le déploiement avec un Deployment** (la méthode recommandée en production)

## Pourquoi Nginx ?

Nginx est un excellent choix pour débuter car :
- ✅ Image Docker légère et fiable
- ✅ Démarrage rapide
- ✅ Page web par défaut visible immédiatement
- ✅ Largement utilisé en production
- ✅ Parfait pour comprendre les concepts

## Méthode 1 : Déployer un Pod Simple

### Comprendre le Pod

Un **Pod** est l'unité de déploiement la plus petite dans Kubernetes. Il contient un ou plusieurs conteneurs qui partagent :
- La même adresse IP
- Le même espace de stockage
- Le même namespace réseau

**Analogie :** Pensez à un Pod comme une "enveloppe" qui contient vos conteneurs.

### Créer le manifeste du Pod

Créez un fichier nommé `nginx-pod.yaml` :

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx-pod
  labels:
    app: nginx
    type: webserver
spec:
  containers:
  - name: nginx
    image: nginx:1.21-alpine
    ports:
    - containerPort: 80
      name: http
```

**Décortiquons ce manifeste :**

| Champ | Valeur | Explication |
|-------|--------|-------------|
| `apiVersion: v1` | Version de l'API | Pods sont dans l'API v1 stable |
| `kind: Pod` | Type d'objet | Nous créons un Pod |
| `name: nginx-pod` | Nom unique | Identifiant de notre Pod |
| `labels.app` | nginx | Label pour identifier l'application |
| `containers[0].name` | nginx | Nom du conteneur dans le Pod |
| `containers[0].image` | nginx:1.21-alpine | Image Docker à télécharger |
| `containerPort` | 80 | Port sur lequel Nginx écoute |

**💡 Note sur l'image :**
- `nginx:1.21-alpine` utilise Alpine Linux (version très légère)
- La taille est d'environ 23 MB au lieu de 133 MB pour la version standard
- Parfait pour un lab ou un environnement de développement

### Déployer le Pod

```bash
# Appliquer le manifeste
microk8s kubectl apply -f nginx-pod.yaml

# Vous devriez voir :
# pod/nginx-pod created
```

**Que se passe-t-il en arrière-plan ?**

1. kubectl envoie le manifeste au serveur API Kubernetes
2. Le serveur API valide le manifeste
3. L'objet Pod est stocké dans etcd (base de données Kubernetes)
4. Le scheduler assigne le Pod à un nœud
5. Le kubelet sur ce nœud télécharge l'image Docker
6. Le conteneur démarre

### Vérifier le déploiement

```bash
# Lister les Pods
microk8s kubectl get pods

# Sortie attendue :
# NAME        READY   STATUS    RESTARTS   AGE
# nginx-pod   1/1     Running   0          30s
```

**Comprendre la sortie :**

- `NAME` : Nom du Pod
- `READY` : 1/1 signifie 1 conteneur prêt sur 1 total
- `STATUS` : État actuel (Running, Pending, Error, etc.)
- `RESTARTS` : Nombre de redémarrages
- `AGE` : Temps depuis la création

### Obtenir plus de détails

```bash
# Informations détaillées du Pod
microk8s kubectl describe pod nginx-pod
```

Cette commande affiche :
- **Events** : Historique des événements (création, téléchargement image, démarrage)
- **Conditions** : État de santé du Pod
- **IP** : Adresse IP interne du Pod
- **Conteneurs** : Détails de chaque conteneur
- **Volumes** : Volumes montés (s'il y en a)

### Accéder aux logs

```bash
# Voir les logs du conteneur
microk8s kubectl logs nginx-pod

# Suivre les logs en temps réel
microk8s kubectl logs -f nginx-pod
```

Les logs montrent les requêtes HTTP reçues par Nginx et d'éventuelles erreurs.

### Tester l'application

Pour l'instant, le Pod n'est accessible qu'à l'intérieur du cluster. Nous allons utiliser le **port-forwarding** pour y accéder depuis notre machine locale :

```bash
# Créer un tunnel entre votre machine et le Pod
microk8s kubectl port-forward nginx-pod 8080:80
```

**Que fait cette commande ?**
- Transfère le port **8080 de votre machine** vers le **port 80 du Pod**
- Le processus reste actif (Ctrl+C pour arrêter)

Ouvrez votre navigateur et accédez à : **http://localhost:8080**

Vous devriez voir la page d'accueil par défaut de Nginx : "Welcome to nginx!"

### Limitations d'un Pod simple

Déployer un Pod directement présente plusieurs inconvénients :

❌ **Pas de résilience** : Si le Pod crash, il ne redémarre pas automatiquement
❌ **Pas de mise à l'échelle** : Impossible d'avoir plusieurs répliques facilement
❌ **Pas de rolling updates** : Pas de mise à jour progressive sans interruption
❌ **Gestion manuelle** : Vous devez recréer le Pod manuellement en cas de problème

**C'est pourquoi nous utilisons des Deployments en production !**

### Supprimer le Pod

```bash
# Supprimer le Pod
microk8s kubectl delete pod nginx-pod

# Ou via le fichier
microk8s kubectl delete -f nginx-pod.yaml
```

## Méthode 2 : Déployer avec un Deployment (Recommandé)

### Pourquoi utiliser un Deployment ?

Un **Deployment** est un contrôleur qui gère des Pods pour vous. Il offre :

✅ **Auto-réparation** : Redémarre automatiquement les Pods défaillants
✅ **Réplication** : Maintient le nombre de répliques souhaité
✅ **Mises à jour progressives** : Rolling updates sans interruption de service
✅ **Rollback** : Retour à une version précédente en cas de problème
✅ **Déclaratif** : Vous déclarez l'état souhaité, Kubernetes s'en charge

**Analogie :** Si un Pod est un employé, un Deployment est le manager qui s'assure qu'il y a toujours le bon nombre d'employés opérationnels.

### Créer le manifeste du Deployment

Créez un fichier nommé `nginx-deployment.yaml` :

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  labels:
    app: nginx
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.21-alpine
        ports:
        - containerPort: 80
          name: http
        resources:
          requests:
            memory: "64Mi"
            cpu: "100m"
          limits:
            memory: "128Mi"
            cpu: "200m"
```

**Décortiquons les nouveaux éléments :**

#### replicas: 3
```yaml
replicas: 3
```
Le Deployment maintiendra **3 répliques** (copies) du Pod en permanence. Si un Pod meurt, un nouveau sera créé automatiquement.

#### selector
```yaml
selector:
  matchLabels:
    app: nginx
```
Le sélecteur indique au Deployment **quels Pods il doit gérer**. Il cherche les Pods avec le label `app: nginx`.

**⚠️ Important :** Les labels du `selector` doivent correspondre aux labels dans `template.metadata.labels`.

#### template
```yaml
template:
  metadata:
    labels:
      app: nginx
  spec:
    # ... spécifications du Pod
```
Le `template` est le **modèle de Pod** que le Deployment utilisera pour créer les répliques. C'est exactement la même structure que notre Pod simple, mais imbriquée dans le Deployment.

#### resources
```yaml
resources:
  requests:
    memory: "64Mi"
    cpu: "100m"
  limits:
    memory: "128Mi"
    cpu: "200m"
```

Les ressources définissent les besoins CPU/RAM :

- **requests** : Ressources garanties au conteneur
  - `100m` = 0,1 CPU (10% d'un cœur)
  - `64Mi` = 64 mégaoctets de RAM

- **limits** : Ressources maximales que le conteneur peut utiliser
  - `200m` = 0,2 CPU (20% d'un cœur)
  - `128Mi` = 128 mégaoctets de RAM

**Pourquoi définir des ressources ?**
- Évite qu'un conteneur monopolise toutes les ressources
- Aide le scheduler à placer les Pods efficacement
- Améliore la stabilité du cluster

### Déployer le Deployment

```bash
# Appliquer le manifeste
microk8s kubectl apply -f nginx-deployment.yaml

# Sortie :
# deployment.apps/nginx-deployment created
```

### Vérifier le Deployment

```bash
# Voir les Deployments
microk8s kubectl get deployments

# Sortie attendue :
# NAME               READY   UP-TO-DATE   AVAILABLE   AGE
# nginx-deployment   3/3     3            3           1m
```

**Comprendre la sortie :**

- `READY` : 3/3 = 3 Pods prêts sur 3 souhaités
- `UP-TO-DATE` : Nombre de Pods à jour avec la dernière configuration
- `AVAILABLE` : Nombre de Pods disponibles pour servir du trafic
- `AGE` : Temps depuis la création

```bash
# Voir les Pods créés par le Deployment
microk8s kubectl get pods

# Sortie attendue :
# NAME                                READY   STATUS    RESTARTS   AGE
# nginx-deployment-5d59d67564-7k9xm   1/1     Running   0          1m
# nginx-deployment-5d59d67564-h2p4l   1/1     Running   0          1m
# nginx-deployment-5d59d67564-tz8wq   1/1     Running   0          1m
```

**Remarquez :**
- Les Pods ont des noms générés automatiquement
- Format : `<deployment-name>-<replicaset-id>-<random-id>`
- Il y a bien 3 Pods comme demandé

### Tester la résilience

L'un des grands avantages d'un Deployment est la **résilience automatique**. Testons-la :

```bash
# Supprimer un Pod manuellement
microk8s kubectl delete pod nginx-deployment-5d59d67564-7k9xm

# Immédiatement après, lister les Pods
microk8s kubectl get pods
```

**Observation :** Un nouveau Pod est créé automatiquement pour remplacer celui supprimé ! Le Deployment maintient toujours 3 répliques.

```
NAME                                READY   STATUS              RESTARTS   AGE
nginx-deployment-5d59d67564-h2p4l   1/1     Running             0          5m
nginx-deployment-5d59d67564-tz8wq   1/1     Running             0          5m
nginx-deployment-5d59d67564-x9n4k   0/1     ContainerCreating   0          2s
```

Le nouveau Pod `x9n4k` remplace automatiquement celui que nous avons supprimé.

### Mise à l'échelle (Scaling)

Changer le nombre de répliques est très simple :

#### Méthode 1 : Via kubectl

```bash
# Passer à 5 répliques
microk8s kubectl scale deployment nginx-deployment --replicas=5

# Vérifier
microk8s kubectl get pods
```

Vous verrez maintenant 5 Pods au lieu de 3.

#### Méthode 2 : Modifier le manifeste

Modifiez `nginx-deployment.yaml` :

```yaml
spec:
  replicas: 2  # Changé de 3 à 2
```

Puis appliquez :

```bash
microk8s kubectl apply -f nginx-deployment.yaml
```

Kubernetes ajustera automatiquement pour avoir exactement 2 répliques (suppression d'un Pod).

**💡 Bonne pratique :** Préférez modifier le fichier YAML et appliquer les changements. Cela maintient vos fichiers comme source de vérité et facilite le versioning avec Git.

### Mettre à jour l'application

Imaginons que nous voulons passer à Nginx 1.22. C'est très simple avec un Deployment.

#### Mise à jour de l'image

Modifiez `nginx-deployment.yaml` :

```yaml
containers:
- name: nginx
  image: nginx:1.22-alpine  # Changé de 1.21 à 1.22
```

Appliquez la modification :

```bash
microk8s kubectl apply -f nginx-deployment.yaml
```

#### Observer le rolling update

```bash
# Voir le statut de la mise à jour
microk8s kubectl rollout status deployment/nginx-deployment

# Sortie :
# Waiting for deployment "nginx-deployment" rollout to finish: 1 out of 3 new replicas have been updated...
# Waiting for deployment "nginx-deployment" rollout to finish: 2 out of 3 new replicas have been updated...
# deployment "nginx-deployment" successfully rolled out
```

**Que se passe-t-il ?**

1. Kubernetes crée un nouveau Pod avec la nouvelle image
2. Attend qu'il soit prêt
3. Supprime un ancien Pod
4. Répète jusqu'à ce que tous les Pods soient à jour

**Avantage :** Aucune interruption de service ! Des Pods sont toujours disponibles pendant la mise à jour.

#### Vérifier l'historique

```bash
# Voir l'historique des déploiements
microk8s kubectl rollout history deployment/nginx-deployment

# Sortie :
# REVISION  CHANGE-CAUSE
# 1         <none>
# 2         <none>
```

### Rollback : Revenir en arrière

Si la nouvelle version pose problème, vous pouvez revenir à la version précédente instantanément :

```bash
# Revenir à la révision précédente
microk8s kubectl rollout undo deployment/nginx-deployment

# Revenir à une révision spécifique
microk8s kubectl rollout undo deployment/nginx-deployment --to-revision=1
```

Kubernetes effectuera un rolling update inverse, restaurant progressivement l'ancienne version.

### Obtenir des informations détaillées

```bash
# Description complète du Deployment
microk8s kubectl describe deployment nginx-deployment
```

Cette commande affiche :
- Informations de configuration
- Stratégie de déploiement
- État des répliques
- Événements récents
- Conditions

```bash
# Voir les logs d'un Pod spécifique
microk8s kubectl logs nginx-deployment-5d59d67564-h2p4l

# Voir les logs de tous les Pods du Deployment
microk8s kubectl logs -l app=nginx

# Suivre les logs en temps réel
microk8s kubectl logs -l app=nginx -f
```

### Stratégies de déploiement

Kubernetes supporte plusieurs stratégies pour mettre à jour un Deployment :

#### RollingUpdate (par défaut)

```yaml
spec:
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 1
      maxSurge: 1
```

- `maxUnavailable` : Nombre maximum de Pods indisponibles pendant la mise à jour
- `maxSurge` : Nombre maximum de Pods supplémentaires créés temporairement

**Comportement :** Mise à jour progressive, zéro downtime.

#### Recreate

```yaml
spec:
  strategy:
    type: Recreate
```

**Comportement :** Supprime tous les Pods existants avant de créer les nouveaux. Provoque un downtime mais utile pour les applications qui ne supportent pas plusieurs versions simultanées.

## Commandes Essentielles - Récapitulatif

### Gestion des Deployments

```bash
# Créer/mettre à jour depuis un fichier
microk8s kubectl apply -f deployment.yaml

# Lister les Deployments
microk8s kubectl get deployments

# Détails d'un Deployment
microk8s kubectl describe deployment nginx-deployment

# Supprimer un Deployment
microk8s kubectl delete deployment nginx-deployment
```

### Gestion des Pods

```bash
# Lister les Pods
microk8s kubectl get pods

# Lister avec plus d'informations (IP, Node)
microk8s kubectl get pods -o wide

# Détails d'un Pod
microk8s kubectl describe pod <pod-name>

# Logs d'un Pod
microk8s kubectl logs <pod-name>

# Logs en temps réel
microk8s kubectl logs -f <pod-name>

# Accéder au shell d'un conteneur
microk8s kubectl exec -it <pod-name> -- /bin/sh
```

### Scaling et Mises à jour

```bash
# Changer le nombre de répliques
microk8s kubectl scale deployment nginx-deployment --replicas=5

# Mettre à jour l'image
microk8s kubectl set image deployment/nginx-deployment nginx=nginx:1.22-alpine

# Statut du rollout
microk8s kubectl rollout status deployment/nginx-deployment

# Historique
microk8s kubectl rollout history deployment/nginx-deployment

# Rollback
microk8s kubectl rollout undo deployment/nginx-deployment
```

### Débogage

```bash
# Événements du cluster
microk8s kubectl get events

# Événements triés par date
microk8s kubectl get events --sort-by='.lastTimestamp'

# Utiliser port-forward pour tester
microk8s kubectl port-forward deployment/nginx-deployment 8080:80
```

## Comparaison Pod vs Deployment

| Aspect | Pod Simple | Deployment |
|--------|------------|------------|
| **Résilience** | ❌ Aucune | ✅ Auto-réparation |
| **Réplication** | ❌ Manuelle | ✅ Automatique |
| **Mises à jour** | ❌ Manuelles avec downtime | ✅ Rolling updates |
| **Rollback** | ❌ Impossible | ✅ Facile |
| **Use case** | Tests rapides, jobs ponctuels | Production, applications |

**Recommandation :** Utilisez toujours des Deployments pour vos applications, sauf cas très spécifiques.

## Bonnes Pratiques

### 1. Toujours utiliser des labels

```yaml
metadata:
  labels:
    app: nginx
    version: "1.21"
    environment: production
```

Les labels facilitent la gestion et le filtrage des ressources.

### 2. Définir les ressources

```yaml
resources:
  requests:
    memory: "64Mi"
    cpu: "100m"
  limits:
    memory: "128Mi"
    cpu: "200m"
```

Aide Kubernetes à mieux gérer les ressources du cluster.

### 3. Utiliser des versions d'images spécifiques

❌ **Éviter :**
```yaml
image: nginx:latest
```

✅ **Préférer :**
```yaml
image: nginx:1.21-alpine
```

Les tags `latest` peuvent changer et causer des comportements imprévisibles.

### 4. Configurer les health checks

```yaml
livenessProbe:
  httpGet:
    path: /
    port: 80
  initialDelaySeconds: 30
  periodSeconds: 10

readinessProbe:
  httpGet:
    path: /
    port: 80
  initialDelaySeconds: 5
  periodSeconds: 5
```

Les probes permettent à Kubernetes de détecter si votre application fonctionne correctement.

### 5. Organiser vos fichiers

Créez une structure de dossiers claire :

```
mon-projet/
├── deployments/
│   └── nginx-deployment.yaml
├── services/
│   └── nginx-service.yaml
└── configmaps/
    └── nginx-config.yaml
```

### 6. Versionner avec Git

Stockez tous vos manifestes YAML dans Git pour :
- Historique des changements
- Collaboration en équipe
- Faciliter les rollbacks
- Documentation automatique

### 7. Utiliser des namespaces

```yaml
metadata:
  name: nginx-deployment
  namespace: production
```

Les namespaces permettent d'isoler les environnements (dev, staging, production).

## Résolution de Problèmes Courants

### Le Pod reste en "Pending"

**Causes possibles :**
- Ressources insuffisantes sur le cluster
- Problèmes de volumes
- Image registry inaccessible

**Diagnostic :**
```bash
microk8s kubectl describe pod <pod-name>
```

Regardez la section **Events** en bas pour identifier le problème.

### Le Pod est en "CrashLoopBackOff"

Le conteneur démarre puis crash en boucle.

**Diagnostic :**
```bash
# Voir les logs
microk8s kubectl logs <pod-name>

# Logs du conteneur précédent (avant crash)
microk8s kubectl logs <pod-name> --previous
```

**Causes courantes :**
- Erreur dans l'application
- Configuration incorrecte
- Dépendances manquantes

### Le Pod est en "ImagePullBackOff"

Kubernetes ne peut pas télécharger l'image Docker.

**Causes :**
- Image n'existe pas
- Nom d'image incorrect
- Registry privé nécessite une authentification

**Vérification :**
```bash
microk8s kubectl describe pod <pod-name>
```

Recherchez les messages d'erreur dans les Events.

### Le conteneur démarre mais l'application ne répond pas

**Diagnostic :**
```bash
# Accéder au shell du conteneur
microk8s kubectl exec -it <pod-name> -- /bin/sh

# Tester depuis l'intérieur
curl localhost:80
```

## Prochaines Étapes

Maintenant que votre application est déployée, les prochaines étapes logiques sont :

1. **Exposer l'application** : Créer un Service pour rendre l'application accessible (section 4.3)
2. **Configurer l'application** : Utiliser ConfigMaps et Secrets (sections suivantes)
3. **Persister les données** : Ajouter des volumes pour les données permanentes
4. **Monitorer** : Mettre en place des health checks et du monitoring

## Récapitulatif

Dans cette section, vous avez appris à :

✅ Déployer un Pod simple avec Nginx
✅ Créer un Deployment pour gérer des répliques
✅ Utiliser les commandes kubectl essentielles
✅ Mettre à l'échelle une application
✅ Effectuer des mises à jour progressives (rolling updates)
✅ Revenir à une version précédente (rollback)
✅ Déboguer des problèmes courants

**Points clés :**

- Les **Deployments** sont préférables aux Pods simples en production
- Kubernetes gère automatiquement la **résilience** et la **réplication**
- Les **rolling updates** permettent des mises à jour sans interruption
- Toujours définir des **ressources** et utiliser des **labels**
- Les **manifestes YAML** doivent être versionnés dans Git

Vous avez maintenant une application web fonctionnelle sur votre cluster MicroK8s ! La prochaine étape sera de l'exposer pour y accéder plus facilement.

---

**Prochaine section :** 4.3 Exposition d'une application

⏭️ [Exposition d'une application](/04-premiers-deploiements/03-exposition-dune-application.md)
