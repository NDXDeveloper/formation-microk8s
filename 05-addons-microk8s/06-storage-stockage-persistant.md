🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 5.6 Storage (stockage persistant - microk8s enable hostpath-storage)

## Introduction

Le **Storage** (stockage persistant) est un addon absolument crucial pour toute application sérieuse dans Kubernetes. Sans stockage persistant, toutes vos données disparaissent quand un pod redémarre. Imaginez une base de données qui perd toutes ses données à chaque redémarrage : c'est inutilisable ! L'addon Storage résout ce problème en permettant à vos données de survivre aux redémarrages des pods.

## Qu'est-ce que le stockage persistant ?

### Le problème : Conteneurs éphémères

Par défaut, les conteneurs Docker et les pods Kubernetes sont **éphémères** (temporaires). Cela signifie que :

1. Quand un pod démarre, il a un système de fichiers vide
2. Vous pouvez écrire des fichiers pendant que le pod tourne
3. **Quand le pod s'arrête ou redémarre, tout est effacé**
4. Le nouveau pod repart de zéro

C'est parfait pour des applications **sans état** (stateless) comme des serveurs web qui ne font que servir du contenu. Mais c'est catastrophique pour des applications **avec état** (stateful) comme des bases de données.

### La solution : Volumes persistants

Les **volumes persistants** sont des espaces de stockage qui :
- Existent **indépendamment** des pods
- **Survivent** aux redémarrages et suppressions de pods
- Peuvent être **rattachés** à différents pods
- Stockent les données de manière **durable**

### Analogie simple

Imaginez que vous travaillez dans un bureau partagé (coworking) :

**Sans stockage persistant (éphémère) :**
- Vous arrivez le matin, on vous assigne un bureau vide
- Vous travaillez, créez des documents
- Le soir, vous partez, et tout est jeté à la poubelle
- Le lendemain, nouveau bureau vide, vous recommencez de zéro
- **Impossible de faire un travail sérieux !**

**Avec stockage persistant :**
- Vous avez un casier personnel avec clé
- Chaque jour, vous pouvez changer de bureau (peu importe)
- Mais vos documents importants restent dans votre casier
- Vous pouvez toujours retrouver votre travail
- **Vous pouvez progresser et construire quelque chose !**

Le stockage persistant dans Kubernetes, c'est votre "casier" où vos données importantes sont en sécurité.

## Pourquoi le stockage persistant est-il indispensable ?

### 1. Bases de données

Une base de données **doit** conserver ses données. Imaginez :
- MySQL qui perd toutes vos tables à chaque redémarrage
- PostgreSQL qui oublie vos utilisateurs
- MongoDB qui efface vos collections

C'est évidemment inacceptable. Le stockage persistant est **obligatoire** pour les bases de données.

### 2. Fichiers uploadés

Si votre application permet aux utilisateurs d'uploader des fichiers (photos, documents, vidéos), ces fichiers doivent être conservés quelque part de manière permanente.

### 3. Logs et données applicatives

Les logs de vos applications, les caches, les sessions utilisateurs, toutes ces données doivent souvent être persistées.

### 4. Configuration dynamique

Certaines applications créent ou modifient leur configuration à l'exécution. Sans stockage persistant, ces modifications sont perdues au redémarrage.

### 5. État de l'application

Pour certaines applications complexes (systèmes de queue, traitement de données), l'état de progression doit être sauvegardé.

## Les concepts de stockage dans Kubernetes

Kubernetes utilise plusieurs concepts pour gérer le stockage. Ne vous inquiétez pas si cela semble complexe au début, nous allons les expliquer simplement.

### 1. Volume

Un **Volume** est un répertoire accessible à un ou plusieurs conteneurs dans un pod. Il existe de nombreux types de volumes, mais le concept de base est simple : c'est un espace de stockage.

**Types de volumes simples :**
- **emptyDir** : Volume éphémère, vide au démarrage, partagé entre conteneurs d'un même pod
- **hostPath** : Monte un répertoire de la machine hôte dans le pod
- **configMap / secret** : Monte des configurations ou secrets comme fichiers

**Types de volumes persistants :**
- **persistentVolumeClaim** : Demande d'accès à un volume persistant (le plus utilisé)

### 2. PersistentVolume (PV)

Un **PersistentVolume** est une ressource de stockage dans le cluster. C'est l'espace de stockage **réel**, physique.

Pensez au PV comme à un **disque dur** ou une **partition** disponible dans votre cluster.

**Caractéristiques d'un PV :**
- A une **taille** (10Gi, 50Gi, etc.)
- A un **mode d'accès** (lecture seule, lecture-écriture, etc.)
- Existe **indépendamment** des pods
- Peut être **recyclé** et réutilisé

### 3. PersistentVolumeClaim (PVC)

Un **PersistentVolumeClaim** est une **demande** de stockage faite par un utilisateur ou une application.

Pensez au PVC comme à une **commande** ou une **réservation** : "J'ai besoin de 5Gi de stockage en lecture-écriture".

**Le workflow :**
1. L'administrateur (ou un système automatique) crée des PV (espace disponible)
2. L'utilisateur crée un PVC (demande d'espace)
3. Kubernetes **lie** automatiquement le PVC à un PV compatible
4. Le pod utilise le PVC pour accéder au stockage

**Analogie :**
- **PV** = Les chambres d'hôtel disponibles
- **PVC** = Votre réservation de chambre
- **Pod** = Vous, le client qui séjourne dans la chambre

### 4. StorageClass

Une **StorageClass** décrit les "types" ou "qualités" de stockage disponibles.

Par exemple :
- **fast** : SSD rapide, cher
- **standard** : HDD normal, économique
- **backup** : Stockage lent mais fiable pour les sauvegardes

La StorageClass permet le **provisionnement dynamique** : quand vous créez un PVC, Kubernetes peut automatiquement créer un PV correspondant.

**Analogie :**
- **StorageClass "économique"** = Auberge de jeunesse
- **StorageClass "standard"** = Hôtel 3 étoiles
- **StorageClass "premium"** = Hôtel 5 étoiles

Vous choisissez le type selon vos besoins et votre budget.

## L'addon hostpath-storage de MicroK8s

MicroK8s propose l'addon **hostpath-storage** qui implémente un système de stockage simple et efficace pour un environnement de développement ou un lab personnel.

### Qu'est-ce que hostpath-storage ?

**Hostpath** signifie "chemin sur l'hôte". Cet addon :
- Stocke les données dans un répertoire sur votre machine hôte
- Crée automatiquement des PV quand vous créez des PVC (provisionnement dynamique)
- Fournit une StorageClass nommée `microk8s-hostpath`

**Où sont stockées les données ?**

Par défaut, dans `/var/snap/microk8s/common/default-storage/` sur votre machine hôte.

### Limitations de hostpath-storage

**Important à comprendre :**

Hostpath-storage est **parfait** pour :
- ✅ Développement local
- ✅ Labs personnels
- ✅ Apprentissage
- ✅ Tests
- ✅ Clusters single-node (un seul serveur)

Hostpath-storage est **inadéquat** pour :
- ❌ Production multi-node
- ❌ Haute disponibilité
- ❌ Stockage partagé entre plusieurs serveurs

**Pourquoi cette limitation ?**

Hostpath stocke les données sur la machine hôte locale. Dans un cluster multi-node :
- Le PV est créé sur **un seul** node
- Si le pod démarre sur un **autre** node, il ne peut pas accéder aux données
- Il n'y a pas de réplication ou de partage

Pour un cluster multi-node en production, vous utiliseriez des solutions comme :
- NFS (Network File System)
- Ceph
- GlusterFS
- Cloud storage (AWS EBS, Google Persistent Disks, etc.)

**Mais pour votre lab MicroK8s, hostpath-storage est parfait !**

## Activation du Storage

L'activation est très simple.

### Commande d'activation

```bash
microk8s enable hostpath-storage
```

Ou, si vous préférez, l'alias plus court :

```bash
microk8s enable storage
```

Les deux commandes font exactement la même chose.

### Ce qui se passe lors de l'activation

Quand vous activez l'addon, MicroK8s :

1. **Crée le namespace** `kube-system` s'il n'existe pas
2. **Déploie le provisioner** hostpath : un pod qui gère la création automatique des PV
3. **Crée une StorageClass** nommée `microk8s-hostpath`
4. **Définit cette StorageClass comme default** : vos PVC l'utiliseront automatiquement

**Sortie attendue :**
```
Infer repository core for addon hostpath-storage
Enabling default storage class
deployment.apps/hostpath-provisioner created
storageclass.storage.k8s.io/microk8s-hostpath created
serviceaccount/microk8s-hostpath created
clusterrole.rbac.authorization.k8s.io/microk8s-hostpath created
clusterrolebinding.rbac.authorization.k8s.io/microk8s-hostpath created
Storage will be available soon
```

### Temps d'activation

L'activation prend environ **10 à 30 secondes**.

### Vérification de l'activation

Pour vérifier que le Storage est bien activé :

```bash
microk8s status
```

Vous devriez voir :
```
enabled:
  hostpath-storage     # (core) Storage class; allocates storage from host directory
```

### Vérifier le provisioner

```bash
microk8s kubectl get pods -n kube-system -l k8s-app=hostpath-provisioner
```

Sortie attendue :
```
NAME                                    READY   STATUS    RESTARTS   AGE
hostpath-provisioner-xxxxxxxxxx-xxxxx   1/1     Running   0          1m
```

### Vérifier la StorageClass

```bash
microk8s kubectl get storageclass
```

Sortie attendue :
```
NAME                          PROVISIONER            RECLAIMPOLICY   VOLUMEBINDINGMODE   ALLOWVOLUMEEXPANSION   AGE
microk8s-hostpath (default)   microk8s.io/hostpath   Delete          Immediate           false                  1m
```

Le `(default)` indique que c'est la StorageClass par défaut. Tout PVC sans StorageClass explicite utilisera celle-ci.

## Utiliser le stockage : Premier exemple

Maintenant que le storage est activé, voyons comment l'utiliser dans un déploiement réel.

### Exemple : Base de données PostgreSQL

Déployons une base de données PostgreSQL avec stockage persistant.

**Fichier `postgres-deployment.yaml` :**
```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: postgres-pvc
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
  name: postgres
spec:
  replicas: 1
  selector:
    matchLabels:
      app: postgres
  template:
    metadata:
      labels:
        app: postgres
    spec:
      containers:
      - name: postgres
        image: postgres:15-alpine
        env:
        - name: POSTGRES_PASSWORD
          value: "mysecretpassword"
        - name: POSTGRES_DB
          value: "myapp"
        ports:
        - containerPort: 5432
        volumeMounts:
        - name: postgres-storage
          mountPath: /var/lib/postgresql/data
      volumes:
      - name: postgres-storage
        persistentVolumeClaim:
          claimName: postgres-pvc
---
apiVersion: v1
kind: Service
metadata:
  name: postgres
spec:
  selector:
    app: postgres
  ports:
  - port: 5432
    targetPort: 5432
```

### Décortiquons ce fichier

**1. PersistentVolumeClaim (demande de stockage) :**
```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: postgres-pvc  # Nom de la demande
spec:
  accessModes:
    - ReadWriteOnce  # Un seul pod peut lire/écrire à la fois
  resources:
    requests:
      storage: 5Gi  # On demande 5 Go d'espace
```

**Modes d'accès possibles :**
- **ReadWriteOnce (RWO)** : Un seul pod peut lire et écrire (le plus courant)
- **ReadOnlyMany (ROX)** : Plusieurs pods peuvent lire, personne ne peut écrire
- **ReadWriteMany (RWX)** : Plusieurs pods peuvent lire et écrire (nécessite un stockage partagé, pas supporté par hostpath)

**2. Volume dans le Deployment :**
```yaml
volumes:
- name: postgres-storage  # Nom local du volume
  persistentVolumeClaim:
    claimName: postgres-pvc  # Référence au PVC créé plus haut
```

**3. VolumeMount dans le conteneur :**
```yaml
volumeMounts:
- name: postgres-storage  # Nom du volume (doit correspondre)
  mountPath: /var/lib/postgresql/data  # Où monter dans le conteneur
```

`/var/lib/postgresql/data` est le répertoire où PostgreSQL stocke ses données. En montant notre volume persistant à cet emplacement, les données PostgreSQL sont sauvegardées.

### Déployer PostgreSQL

```bash
microk8s kubectl apply -f postgres-deployment.yaml
```

**Sortie attendue :**
```
persistentvolumeclaim/postgres-pvc created
deployment.apps/postgres created
service/postgres created
```

### Vérifier le PVC

```bash
microk8s kubectl get pvc
```

Sortie :
```
NAME           STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS        AGE
postgres-pvc   Bound    pvc-abc123-def456-ghi789-jkl012-mno345    5Gi        RWO            microk8s-hostpath   30s
```

**Status: Bound** signifie que le PVC a été lié à un PV. Kubernetes a automatiquement créé un PV pour satisfaire votre demande. C'est le provisionnement dynamique en action !

### Vérifier le PV créé automatiquement

```bash
microk8s kubectl get pv
```

Sortie :
```
NAME                                       CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM                  STORAGECLASS        AGE
pvc-abc123-def456-ghi789-jkl012-mno345    5Gi        RWO            Delete           Bound    default/postgres-pvc   microk8s-hostpath   1m
```

Kubernetes a automatiquement créé ce PV pour votre PVC.

### Vérifier le pod

```bash
microk8s kubectl get pods
```

Le pod PostgreSQL devrait être en état `Running` :
```
NAME                        READY   STATUS    RESTARTS   AGE
postgres-xxxxxxxxxx-xxxxx   1/1     Running   0          2m
```

### Tester la persistance

Vérifions que les données survivent vraiment au redémarrage du pod.

**1. Créer des données dans la base**

Connectons-nous au pod et créons une table :

```bash
microk8s kubectl exec -it deployment/postgres -- psql -U postgres -d myapp
```

Dans le shell PostgreSQL :
```sql
-- Créer une table
CREATE TABLE users (
  id SERIAL PRIMARY KEY,
  name VARCHAR(100),
  email VARCHAR(100)
);

-- Insérer des données
INSERT INTO users (name, email) VALUES
  ('Alice', 'alice@example.com'),
  ('Bob', 'bob@example.com'),
  ('Charlie', 'charlie@example.com');

-- Vérifier les données
SELECT * FROM users;

-- Sortir
\q
```

Vous devriez voir vos trois utilisateurs.

**2. Supprimer le pod (simuler un crash)**

```bash
microk8s kubectl delete pod -l app=postgres
```

Le Deployment va automatiquement recréer un nouveau pod.

**3. Vérifier que les données sont toujours là**

Attendez que le nouveau pod soit prêt :
```bash
microk8s kubectl get pods -w
```

Puis reconnectez-vous :
```bash
microk8s kubectl exec -it deployment/postgres -- psql -U postgres -d myapp
```

```sql
SELECT * FROM users;
```

**Vos données sont toujours là !** C'est la magie du stockage persistant. Le pod a été détruit et recréé, mais les données ont survécu car elles sont sur le volume persistant.

## Où sont stockées les données physiquement ?

Les données sont stockées sur votre machine hôte dans :
```
/var/snap/microk8s/common/default-storage/
```

Vous pouvez explorer ce répertoire :

```bash
sudo ls -la /var/snap/microk8s/common/default-storage/
```

Vous verrez des répertoires avec des noms correspondant aux PV. Chaque PV a son propre répertoire.

**Exemple :**
```
drwxr-xr-x  3 root root 4096 Oct 23 10:30 default-postgres-pvc-pvc-abc123...
```

À l'intérieur, vous trouverez les fichiers réels de PostgreSQL.

**Note :** Il est généralement déconseillé de modifier directement ces fichiers. Laissez Kubernetes et vos applications gérer le contenu.

## Cas d'usage pratiques

### 1. MySQL / MariaDB

Similaire à PostgreSQL :

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: mysql-pvc
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 10Gi
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mysql
spec:
  replicas: 1
  selector:
    matchLabels:
      app: mysql
  template:
    metadata:
      labels:
        app: mysql
    spec:
      containers:
      - name: mysql
        image: mysql:8
        env:
        - name: MYSQL_ROOT_PASSWORD
          value: "rootpassword"
        - name: MYSQL_DATABASE
          value: "myapp"
        volumeMounts:
        - name: mysql-storage
          mountPath: /var/lib/mysql
      volumes:
      - name: mysql-storage
        persistentVolumeClaim:
          claimName: mysql-pvc
```

### 2. MongoDB

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: mongo-pvc
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 10Gi
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mongodb
spec:
  replicas: 1
  selector:
    matchLabels:
      app: mongodb
  template:
    metadata:
      labels:
        app: mongodb
    spec:
      containers:
      - name: mongodb
        image: mongo:7
        env:
        - name: MONGO_INITDB_ROOT_USERNAME
          value: "admin"
        - name: MONGO_INITDB_ROOT_PASSWORD
          value: "password"
        volumeMounts:
        - name: mongo-storage
          mountPath: /data/db
      volumes:
      - name: mongo-storage
        persistentVolumeClaim:
          claimName: mongo-pvc
```

### 3. Application avec uploads de fichiers

Pour une application web qui permet aux utilisateurs d'uploader des fichiers :

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: uploads-pvc
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 20Gi
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: webapp
spec:
  replicas: 1
  selector:
    matchLabels:
      app: webapp
  template:
    metadata:
      labels:
        app: webapp
    spec:
      containers:
      - name: webapp
        image: my-webapp:latest
        volumeMounts:
        - name: uploads-storage
          mountPath: /app/uploads  # Où l'app stocke les fichiers uploadés
      volumes:
      - name: uploads-storage
        persistentVolumeClaim:
          claimName: uploads-pvc
```

### 4. Redis avec persistance

Redis est souvent utilisé comme cache, mais peut aussi persister les données :

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: redis-pvc
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
  name: redis
spec:
  replicas: 1
  selector:
    matchLabels:
      app: redis
  template:
    metadata:
      labels:
        app: redis
    spec:
      containers:
      - name: redis
        image: redis:7-alpine
        command: ["redis-server", "--appendonly", "yes"]
        volumeMounts:
        - name: redis-storage
          mountPath: /data
      volumes:
      - name: redis-storage
        persistentVolumeClaim:
          claimName: redis-pvc
```

## Gérer les PVC et PV

### Lister les PVC

```bash
microk8s kubectl get pvc
```

Ou avec plus de détails :
```bash
microk8s kubectl get pvc -o wide
```

### Décrire un PVC

Pour voir tous les détails d'un PVC :
```bash
microk8s kubectl describe pvc postgres-pvc
```

Vous verrez :
- L'état (Bound, Pending, etc.)
- Le PV auquel il est lié
- La capacité allouée
- Les événements

### Lister les PV

```bash
microk8s kubectl get pv
```

### Supprimer un PVC

**⚠️ Attention :** Supprimer un PVC peut entraîner la perte des données !

```bash
microk8s kubectl delete pvc postgres-pvc
```

**Que se passe-t-il quand vous supprimez un PVC ?**

Cela dépend de la **Reclaim Policy** du PV :

- **Delete** (défaut pour hostpath-storage) : Le PV ET les données sont supprimés
- **Retain** : Le PV est marqué comme "Released" mais les données restent
- **Recycle** : Les données sont effacées, le PV est réutilisable

Pour hostpath-storage, la politique est **Delete**, donc :
1. Vous supprimez le PVC
2. Le PV est automatiquement supprimé
3. **Les données sur le disque sont effacées**

**Soyez donc très prudent !**

### Agrandir un PVC

Malheureusement, avec hostpath-storage de base, l'expansion de volume n'est pas supportée automatiquement (ALLOWVOLUMEEXPANSION = false).

Pour avoir plus d'espace, vous devriez :
1. Sauvegarder vos données
2. Créer un nouveau PVC avec une taille plus grande
3. Restaurer les données

Ou utiliser une StorageClass qui supporte l'expansion.

## Volumes dans plusieurs conteneurs d'un même pod

Un cas d'usage intéressant : partager un volume entre plusieurs conteneurs d'un même pod.

**Exemple : Application web + Sidecar pour les logs**

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: app-logs-pvc
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
  name: webapp-with-sidecar
spec:
  replicas: 1
  selector:
    matchLabels:
      app: webapp
  template:
    metadata:
      labels:
        app: webapp
    spec:
      containers:
      # Conteneur principal : l'application
      - name: webapp
        image: nginx
        volumeMounts:
        - name: logs
          mountPath: /var/log/nginx

      # Conteneur sidecar : traitement des logs
      - name: log-processor
        image: busybox
        command: ["sh", "-c", "while true; do echo 'Processing logs...'; sleep 60; done"]
        volumeMounts:
        - name: logs
          mountPath: /logs

      volumes:
      - name: logs
        persistentVolumeClaim:
          claimName: app-logs-pvc
```

Les deux conteneurs partagent le même volume. L'application web écrit les logs, le sidecar les lit et les traite.

## emptyDir : Volume éphémère partagé

Parfois, vous n'avez pas besoin de persistance au-delà du pod, mais vous voulez partager des données entre conteneurs.

**emptyDir** est parfait pour ça :

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: shared-volume-pod
spec:
  containers:
  - name: writer
    image: busybox
    command: ["sh", "-c", "while true; do date >> /data/log.txt; sleep 5; done"]
    volumeMounts:
    - name: shared-data
      mountPath: /data

  - name: reader
    image: busybox
    command: ["sh", "-c", "while true; do cat /data/log.txt; sleep 10; done"]
    volumeMounts:
    - name: shared-data
      mountPath: /data

  volumes:
  - name: shared-data
    emptyDir: {}
```

Le volume `emptyDir` :
- Est créé quand le pod démarre
- Est partagé entre tous les conteneurs du pod
- Est détruit quand le pod est supprimé
- Ne survit pas au redémarrage du pod

**Quand utiliser emptyDir ?**
- Cache temporaire
- Échange de données entre conteneurs d'un même pod
- Espace de travail temporaire
- Traitement de fichiers intermédiaires

## StatefulSets : Pour les applications stateful

Pour les applications qui nécessitent des garanties plus fortes (comme des bases de données en cluster), Kubernetes propose les **StatefulSets**.

Les StatefulSets garantissent :
- Des noms de pods prévisibles (app-0, app-1, app-2)
- Un ordre de démarrage et d'arrêt
- Des volumes persistants liés de manière stable à chaque pod

**Exemple simple de StatefulSet :**

```yaml
apiVersion: v1
kind: Service
metadata:
  name: mysql-headless
spec:
  clusterIP: None  # Service headless
  selector:
    app: mysql
  ports:
  - port: 3306
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: mysql
spec:
  serviceName: "mysql-headless"
  replicas: 3
  selector:
    matchLabels:
      app: mysql
  template:
    metadata:
      labels:
        app: mysql
    spec:
      containers:
      - name: mysql
        image: mysql:8
        env:
        - name: MYSQL_ROOT_PASSWORD
          value: "password"
        volumeMounts:
        - name: data
          mountPath: /var/lib/mysql
  volumeClaimTemplates:
  - metadata:
      name: data
    spec:
      accessModes: [ "ReadWriteOnce" ]
      resources:
        requests:
          storage: 10Gi
```

Le StatefulSet crée automatiquement un PVC pour chaque replica :
- `data-mysql-0` pour le pod `mysql-0`
- `data-mysql-1` pour le pod `mysql-1`
- `data-mysql-2` pour le pod `mysql-2`

Si un pod est supprimé et recréé, il se reconnecte au même PVC.

**Note :** Les StatefulSets sont plus avancés et nécessitent une compréhension plus approfondie. Pour débuter, les Deployments avec PVC sont suffisants.

## Sauvegarder vos données

Même si vos données sont persistantes, il est crucial de les sauvegarder régulièrement.

### Méthode 1 : Exporter depuis l'application

**Pour une base de données :**

```bash
# PostgreSQL
microk8s kubectl exec deployment/postgres -- pg_dump -U postgres myapp > backup.sql

# MySQL
microk8s kubectl exec deployment/mysql -- mysqldump -u root -prootpassword myapp > backup.sql

# MongoDB
microk8s kubectl exec deployment/mongodb -- mongodump --username admin --password password --authenticationDatabase admin --out /backup
```

### Méthode 2 : Copier directement le volume

Vous pouvez créer un pod temporaire qui monte le PVC et copie les données :

```bash
# Créer un pod de backup
microk8s kubectl run backup-pod --image=busybox --rm -it --restart=Never \
  --overrides='
{
  "spec": {
    "containers": [
      {
        "name": "backup",
        "image": "busybox",
        "command": ["tar", "czf", "/backup/data.tar.gz", "/data"],
        "volumeMounts": [
          {
            "name": "data",
            "mountPath": "/data"
          },
          {
            "name": "backup",
            "mountPath": "/backup"
          }
        ]
      }
    ],
    "volumes": [
      {
        "name": "data",
        "persistentVolumeClaim": {
          "claimName": "postgres-pvc"
        }
      },
      {
        "name": "backup",
        "hostPath": {
          "path": "/tmp/backups"
        }
      }
    ]
  }
}'
```

### Méthode 3 : Accès direct au système de fichiers

Puisque hostpath-storage stocke les données localement, vous pouvez les copier directement :

```bash
sudo cp -r /var/snap/microk8s/common/default-storage/default-postgres-pvc-* /backups/
```

**Recommandation :** Automatisez vos sauvegardes avec un CronJob Kubernetes qui s'exécute régulièrement.

## Dépannage

### PVC reste en état "Pending"

**Symptôme :** Le PVC ne passe jamais à l'état "Bound"

```bash
microk8s kubectl get pvc
```
```
NAME      STATUS    VOLUME   CAPACITY   ACCESS MODES   STORAGECLASS   AGE
my-pvc    Pending                                      microk8s-hostpath   5m
```

**Causes possibles :**

1. **Le provisioner n'est pas en cours d'exécution**
```bash
microk8s kubectl get pods -n kube-system -l k8s-app=hostpath-provisioner
```

Si le pod n'existe pas ou n'est pas en "Running", réactivez l'addon :
```bash
microk8s disable storage
microk8s enable storage
```

2. **Ressources insuffisantes**

Vérifiez l'espace disque :
```bash
df -h /var/snap/microk8s/common/default-storage/
```

3. **Problème de permissions**

```bash
sudo ls -la /var/snap/microk8s/common/
```

Le répertoire doit être accessible.

**Diagnostic :**
```bash
microk8s kubectl describe pvc my-pvc
```

Regardez la section "Events" pour voir les erreurs.

### Pod ne démarre pas à cause du volume

**Symptôme :** Pod en état "ContainerCreating" indéfiniment

```bash
microk8s kubectl describe pod <nom-du-pod>
```

Cherchez les événements du type :
```
Warning  FailedMount  ... MountVolume.SetUp failed for volume "..." : ...
```

**Solutions :**

1. **Vérifier que le PVC est bound**
```bash
microk8s kubectl get pvc
```

2. **Vérifier les permissions du volume**

3. **Redémarrer le pod**
```bash
microk8s kubectl delete pod <nom-du-pod>
```

### Données corrompues

**Symptôme :** L'application signale des données corrompues ou ne démarre pas

**Causes possibles :**
- Arrêt brutal du pod pendant une écriture
- Problème de système de fichiers

**Solutions :**

1. **Restaurer depuis une sauvegarde**

2. **Pour une base de données, utiliser les outils de réparation :**

PostgreSQL :
```bash
microk8s kubectl exec -it deployment/postgres -- pg_resetwal /var/lib/postgresql/data
```

MySQL :
```bash
microk8s kubectl exec -it deployment/mysql -- mysqlcheck --all-databases
```

3. **En dernier recours, recréer le PVC** (perte de données)

### Volume plein

**Symptôme :** L'application ne peut plus écrire, erreur "no space left on device"

**Diagnostic :**

Vérifier l'utilisation dans le pod :
```bash
microk8s kubectl exec deployment/postgres -- df -h /var/lib/postgresql/data
```

**Solutions :**

1. **Nettoyer les anciennes données** (si applicable)

2. **Augmenter la taille du PVC** (si la StorageClass le permet)

3. **Créer un nouveau PVC plus grand et migrer les données**

## Bonnes pratiques

### 1. Toujours demander la bonne taille

Évaluez vos besoins en stockage et ajoutez une marge :
- Base de données légère : 5-10Gi
- Base de données moyenne : 20-50Gi
- Application avec beaucoup de fichiers : 50-100Gi+

Il vaut mieux prévoir trop que pas assez.

### 2. Utiliser des noms descriptifs

```yaml
# Bon
metadata:
  name: mysql-production-pvc

# Moins bon
metadata:
  name: pvc-1
```

### 3. Documenter vos PVC

Utilisez des labels et annotations :

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: mysql-pvc
  labels:
    app: mysql
    env: production
    tier: database
  annotations:
    description: "MySQL production database storage"
    backup-schedule: "daily at 2am"
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 20Gi
```

### 4. Sauvegarder régulièrement

Ne comptez jamais uniquement sur le stockage persistant. Ayez toujours des sauvegardes :
- Automatiques (CronJob)
- Régulières (quotidiennes minimum)
- Testées (vérifiez que vous pouvez restaurer)
- Hors du cluster (si possible)

### 5. Monitorer l'utilisation

Gardez un œil sur l'espace disque utilisé. Vous ne voulez pas de surprise quand le volume est plein.

### 6. Un PVC par application stateful

Ne partagez pas un PVC entre plusieurs applications indépendantes. Chaque application stateful devrait avoir son propre PVC.

### 7. Utiliser les bons accessModes

- **ReadWriteOnce** pour la majorité des cas (bases de données, stockage applicatif)
- **ReadOnlyMany** si plusieurs pods doivent lire (rarement nécessaire)
- **ReadWriteMany** nécessite un stockage partagé (NFS, etc.), pas supporté par hostpath

### 8. Tester la restauration

Avant un problème, testez votre processus de restauration :
1. Prenez une sauvegarde
2. Supprimez volontairement les données
3. Restaurez depuis la sauvegarde
4. Vérifiez que tout fonctionne

Ainsi, vous serez prêt en cas de vraie catastrophe.

## Récapitulatif

Le stockage persistant est essentiel pour :

✅ **Conserver** les données au-delà des redémarrages de pods
✅ **Héberger** des bases de données dans Kubernetes
✅ **Stocker** les fichiers uploadés par les utilisateurs
✅ **Persister** les logs et données applicatives
✅ **Garantir** la durabilité de vos applications

**Commandes clés à retenir :**
```bash
# Activer le storage
microk8s enable storage

# Lister les PVC
microk8s kubectl get pvc

# Lister les PV
microk8s kubectl get pv

# Voir les détails d'un PVC
microk8s kubectl describe pvc <nom-pvc>

# Vérifier le provisioner
microk8s kubectl get pods -n kube-system -l k8s-app=hostpath-provisioner
```

**Structure minimale pour utiliser le stockage :**
```yaml
# 1. Créer un PVC
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: my-pvc
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 5Gi

# 2. L'utiliser dans un Deployment
spec:
  template:
    spec:
      containers:
      - name: myapp
        volumeMounts:
        - name: my-storage
          mountPath: /data
      volumes:
      - name: my-storage
        persistentVolumeClaim:
          claimName: my-pvc
```

## Ce que vous avez appris

Dans cette section, vous avez découvert :
- Ce qu'est le stockage persistant et pourquoi il est crucial
- Les concepts : Volumes, PV, PVC, StorageClass
- Comment activer l'addon hostpath-storage
- Comment créer et utiliser des PVC dans vos déploiements
- Comment tester la persistance des données
- Les cas d'usage pratiques (bases de données, fichiers, etc.)
- La gestion des PVC et PV
- Les stratégies de sauvegarde
- Comment résoudre les problèmes courants
- Les bonnes pratiques du stockage

## Conclusion du chapitre 5

Félicitations ! Vous avez maintenant activé et maîtrisé les **quatre addons essentiels** de MicroK8s :

1. ✅ **Dashboard** : Visualisation graphique de votre cluster
2. ✅ **DNS** : Communication entre services par nom
3. ✅ **Registry** : Hébergement de vos images Docker privées
4. ✅ **Storage** : Persistance des données

Votre cluster MicroK8s est maintenant **pleinement fonctionnel** pour le développement, l'apprentissage et les projets personnels !

## Prochaine étape

Le chapitre 5 vous a introduit aux addons essentiels. Il existe de nombreux autres addons disponibles (Ingress, Cert-Manager, Prometheus, MetalLB, etc.) que nous explorerons dans les chapitres suivants.

Mais avant cela, prenez le temps de :
- Expérimenter avec ce que vous avez appris
- Déployer quelques applications de test
- Vous familiariser avec ces quatre addons fondamentaux

Vous avez posé les fondations solides. La suite du tutoriel vous permettra de construire des architectures encore plus sophistiquées !

---

**Astuce pratique :** Créez un "starter kit" avec des fichiers YAML prêts à l'emploi pour PostgreSQL, MySQL, Redis avec leurs PVC. Vous gagnerez du temps à chaque nouveau projet en ayant ces templates sous la main !

⏭️ [Liste complète des addons disponibles](/05-addons-microk8s/07-liste-complete-des-addons-disponibles.md)
