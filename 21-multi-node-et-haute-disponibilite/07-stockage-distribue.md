🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 21.7 Stockage Distribué

## Introduction

Vous avez maintenant un cluster multi-node avec load balancing. Mais il reste un élément crucial à maîtriser : le **stockage distribué**. Sans lui, vos données sont liées à un seul nœud, ce qui limite la résilience et la flexibilité de votre cluster.

Dans ce chapitre, nous allons découvrir :
- Pourquoi le stockage distribué est essentiel dans un cluster multi-node
- Les différentes solutions de stockage disponibles
- Comment configurer du stockage distribué avec MicroK8s
- Les concepts de PersistentVolumes, PersistentVolumeClaims et StorageClasses
- Les bonnes pratiques pour gérer les données persistantes

Le stockage distribué permet à vos applications de :
- **Survivre aux pannes** de nœuds (les données sont répliquées)
- **Se déplacer librement** entre nœuds (les pods peuvent accéder aux données depuis n'importe quel nœud)
- **Scaler horizontalement** (ajouter de la capacité en ajoutant des nœuds)
- **Garantir la durabilité** des données critiques

## Le Problème du Stockage Local

### Stockage Local : Comment ça Fonctionne

Par défaut, avec l'addon `hostpath-storage` de MicroK8s, le stockage est **local** à chaque nœud.

**Schéma du stockage local :**
```
┌───────────────────────────────────────────────────────┐
│                    CLUSTER                            │
│                                                       │
│  ┌─────────────┐   ┌─────────────┐   ┌─────────────┐  │
│  │   Node 1    │   │   Node 2    │   │   Node 3    │  │
│  │             │   │             │   │             │  │
│  │  Pod A      │   │  Pod B      │   │  Pod C      │  │
│  │  │          │   │  │          │   │  │          │  │
│  │  ▼          │   │  ▼          │   │  ▼          │  │
│  │ /data       │   │ /data       │   │ /data       │  │
│  │ (local)     │   │ (local)     │   │ (local)     │  │
│  └─────────────┘   └─────────────┘   └─────────────┘  │
│       ▲                 ▲                 ▲           │
│       │                 │                 │           │
│   Données sur       Données sur       Données sur     │
│   le disque         le disque         le disque       │
│   de Node 1         de Node 2         de Node 3       │
└───────────────────────────────────────────────────────┘
```

**Chaque pod stocke ses données sur le disque local du nœud où il s'exécute.**

### Les Problèmes du Stockage Local

**Problème 1 : Perte de données en cas de panne**

Si Node 2 tombe en panne :
```
Node 1 : ✅ Données accessibles
Node 2 : ❌ PANNE → Données inaccessibles
Node 3 : ✅ Données accessibles
```

Pod B sera redémarré sur Node 1 ou Node 3, mais **ses données sont perdues** car elles étaient sur Node 2.

**Problème 2 : Pod lié à un nœud spécifique**

Si un pod utilise du stockage local, Kubernetes **ne peut pas** le déplacer sur un autre nœud (car ses données ne sont pas accessibles ailleurs).

**Exemple concret :**
```bash
# Pod avec base de données sur Node 2
microk8s kubectl get pod postgres-pod -o wide
```
```
NAME            READY   STATUS    NODE
postgres-pod    1/1     Running   node2
```

Si vous drainez Node 2 pour maintenance :
```bash
microk8s kubectl drain node2 --ignore-daemonsets
```

Le pod postgres-pod **ne peut pas** être évacué car ses données sont liées à Node 2. Il reste bloqué.

**Problème 3 : Pas de réplication**

Les données existent en un seul exemplaire. Pas de backup automatique, pas de redondance.

**Problème 4 : Capacité limitée**

La capacité de stockage est limitée au disque du nœud. Vous ne pouvez pas facilement ajouter de la capacité.

### Quand le Stockage Local Suffit

**Le stockage local est acceptable pour :**
- Données temporaires (cache, logs)
- Applications stateless (sans état)
- Environnements de développement/test
- Clusters single-node

**Le stockage local N'EST PAS acceptable pour :**
- Bases de données en production
- Applications stateful critiques
- Données qui doivent survivre aux pannes
- Clusters multi-node en production

## Qu'est-ce que le Stockage Distribué ?

### Définition

Le **stockage distribué** est un système où les données sont réparties et répliquées sur plusieurs nœuds du cluster. Chaque nœud peut accéder aux données, peu importe où elles sont physiquement stockées.

**Analogie du cloud personnel :**
C'est comme avoir Google Drive ou Dropbox dans votre cluster :
- Les fichiers sont répliqués sur plusieurs serveurs
- Vous pouvez y accéder depuis n'importe quelle machine
- Si un serveur tombe, les données restent accessibles via les autres

**Schéma du stockage distribué :**
```
┌───────────────────────────────────────────────────────┐
│                    CLUSTER                            │
│                                                       │
│  ┌─────────────┐   ┌─────────────┐   ┌─────────────┐  │
│  │   Node 1    │   │   Node 2    │   │   Node 3    │  │
│  │             │   │             │   │             │  │
│  │  Pod A      │   │  Pod B      │   │  Pod C      │  │
│  │  │          │   │  │          │   │  │          │  │
│  │  ▼          │   │  ▼          │   │  ▼          │  │
│  └──┼──────────┘   └──┼──────────┘   └──┼──────────┘  │
│     │                 │                 │             │
│     └─────────────────┼─────────────────┘             │
│                       │                               │
│           ┌───────────▼──────────────────────────┐    │
│           │  STOCKAGE DISTRIBUÉ                  │    │
│           │  (Répliqué 3x)                       │    │
│           │                                      │    │
│           │  Replica 1  Replica 2   Replica 3    │    │
│           │  (Node 1)   (Node 2)   (Node 3)      │    │
│           └──────────────────────────────────────┘    │
└───────────────────────────────────────────────────────┘
```

**Tous les pods peuvent accéder aux mêmes données, peu importe leur nœud.**

### Avantages du Stockage Distribué

**1. Résilience aux pannes**
- Données répliquées sur plusieurs nœuds (3 copies par défaut)
- Si un nœud tombe, les données restent accessibles

**2. Mobilité des pods**
- Un pod peut être déplacé d'un nœud à un autre
- Il retrouve ses données automatiquement

**3. Haute disponibilité**
- Applications stateful peuvent avoir de la HA
- Pas de downtime lors des maintenances

**4. Scalabilité**
- Ajouter de la capacité en ajoutant des nœuds
- Distribution automatique des données

**5. Performance**
- Lectures parallèles possibles
- Load balancing du stockage

### Concepts Clés

**Réplication :**
Chaque donnée existe en plusieurs exemplaires (replicas). Par défaut : 3 copies.

**Consistency (Cohérence) :**
Toutes les copies des données sont synchronisées. Si vous écrivez sur un nœud, la modification est propagée aux autres.

**Fault Tolerance (Tolérance aux pannes) :**
Le système continue de fonctionner même si un ou plusieurs nœuds tombent.

**Self-Healing (Auto-réparation) :**
Si un nœud tombe, le système recrée automatiquement les replicas manquants sur d'autres nœuds.

## Solutions de Stockage Distribué pour Kubernetes

Il existe plusieurs solutions de stockage distribué. Voici les principales :

### 1. Ceph (avec Rook)

**Description :**
- Solution de stockage distribué très mature
- Utilisée par les grandes entreprises
- Gérée par Rook (orchestrateur Kubernetes pour Ceph)

**Avantages :**
- Très robuste et éprouvée
- Supporte block, file et object storage
- Excellente performance
- Très scalable (jusqu'à des milliers de nœuds)

**Inconvénients :**
- Complexe à configurer
- Consomme beaucoup de ressources (RAM, CPU)
- Overkill pour un petit cluster

**Recommandé pour :**
- Grandes entreprises
- Clusters de 10+ nœuds
- Production à grande échelle

### 2. Longhorn

**Description :**
- Développé par Rancher (SUSE)
- Conçu pour être simple et léger
- Interface web intégrée

**Avantages :**
- **Simple à installer et utiliser**
- Interface web intuitive
- Léger en ressources
- Backups intégrés
- Parfait pour les petits/moyens clusters

**Inconvénients :**
- Moins mature que Ceph
- Performance moyenne (suffisante pour la plupart des cas)

**Recommandé pour :**
- **Homelabs et PME** ← NOTRE CAS
- Clusters de 3-20 nœuds
- Applications standard

### 3. OpenEBS

**Description :**
- Stockage distribué cloud-native
- Plusieurs "storage engines" disponibles
- CNCF Sandbox project

**Avantages :**
- Flexible (plusieurs backends)
- Intégration Kubernetes native
- Bonnes performances

**Inconvénients :**
- Configuration parfois complexe
- Documentation touffue

**Recommandé pour :**
- Utilisateurs avancés
- Besoins spécifiques de performance

### 4. NFS (Network File System)

**Description :**
- Protocole de partage de fichiers réseau
- Ancien mais éprouvé
- Nécessite un serveur NFS externe

**Avantages :**
- Très simple
- Compatible avec tout
- Pas besoin de ressources cluster

**Inconvénients :**
- **Serveur NFS = Single Point of Failure**
- Pas de réplication intégrée
- Performances variables
- Pas vraiment "distribué"

**Recommandé pour :**
- Prototypes rapides
- Si vous avez déjà un NAS/serveur NFS
- Environnements de test

### 5. GlusterFS

**Description :**
- Système de fichiers distribué
- Alternative à Ceph

**Avantages :**
- Mature
- Pas d'architecture client-serveur (peer-to-peer)

**Inconvénients :**
- Configuration complexe
- Moins activement développé qu'avant

**Recommandé pour :**
- Cas spécifiques
- Utilisateurs expérimentés avec GlusterFS

## Choisir la Bonne Solution

### Tableau Comparatif

| Solution | Complexité | Ressources | Performance | HA | Recommandé pour |
|----------|-----------|------------|-------------|----|-----------------|
| **Longhorn** | ⭐ Faible | ⭐ Faible | ⭐⭐⭐ Bonne | ✅ Oui | **Homelabs, PME** |
| Ceph+Rook | ⭐⭐⭐ Élevée | ⭐⭐⭐ Élevée | ⭐⭐⭐⭐ Excellente | ✅ Oui | Grandes entreprises |
| OpenEBS | ⭐⭐ Moyenne | ⭐⭐ Moyenne | ⭐⭐⭐ Bonne | ✅ Oui | Utilisateurs avancés |
| NFS | ⭐ Faible | ⭐ Très faible | ⭐⭐ Moyenne | ❌ Non | Test/Dev |
| GlusterFS | ⭐⭐⭐ Élevée | ⭐⭐ Moyenne | ⭐⭐⭐ Bonne | ✅ Oui | Cas spécifiques |

### Recommandation pour Votre Cluster

**Pour un cluster MicroK8s 3-10 nœuds (homelab, PME) :**

**→ Longhorn** est le choix idéal :
- Installation simple
- Interface web claire
- Ressources raisonnables
- Parfait pour apprendre et produire

**Pour un cluster 10+ nœuds (production enterprise) :**

**→ Ceph avec Rook** pour :
- Performance maximale
- Scalabilité illimitée
- Support enterprise

**Pour du prototypage rapide :**

**→ NFS** si vous avez déjà un NAS

## Installer Longhorn sur MicroK8s

Nous allons installer **Longhorn** car c'est la solution la plus adaptée pour un cluster MicroK8s multi-node.

### Prérequis

**Ressources minimales :**
- 3 nœuds (pour la réplication)
- 2 GB RAM par nœud (en plus de ce qui tourne déjà)
- 10 GB d'espace disque libre par nœud

**Paquets système nécessaires :**

Sur **chaque nœud** du cluster :
```bash
# Installer les dépendances
sudo apt update
sudo apt install -y open-iscsi nfs-common

# Démarrer et activer iSCSI (protocole utilisé par Longhorn)
sudo systemctl enable --now iscsid
sudo systemctl status iscsid

# Vérifier que le module kernel est chargé
lsmod | grep iscsi
```

**Vérifier l'espace disque disponible :**
```bash
df -h /var/lib/longhorn
```

### Installation de Longhorn

**Méthode 1 : Via Helm (Recommandée)**

```bash
# 1. Activer Helm (si pas déjà fait)
microk8s enable helm3

# 2. Ajouter le repo Longhorn
microk8s helm3 repo add longhorn https://charts.longhorn.io
microk8s helm3 repo update

# 3. Installer Longhorn
microk8s helm3 install longhorn longhorn/longhorn \
  --namespace longhorn-system \
  --create-namespace

# 4. Attendre que tous les pods soient prêts (peut prendre 2-5 minutes)
microk8s kubectl get pods -n longhorn-system -w
```

**Méthode 2 : Via YAML**

```bash
# 1. Télécharger le manifest
kubectl apply -f https://raw.githubusercontent.com/longhorn/longhorn/v1.5.3/deploy/longhorn.yaml

# 2. Attendre le déploiement
microk8s kubectl get pods -n longhorn-system -w
```

### Vérifier l'Installation

**Tous les pods doivent être en Running :**
```bash
microk8s kubectl get pods -n longhorn-system
```

**Sortie attendue :**
```
NAME                                        READY   STATUS
csi-attacher-xxx                            1/1     Running
csi-provisioner-xxx                         1/1     Running
csi-resizer-xxx                             1/1     Running
csi-snapshotter-xxx                         1/1     Running
engine-image-ei-xxx                         1/1     Running
instance-manager-e-xxx                      1/1     Running
instance-manager-r-xxx                      1/1     Running
longhorn-csi-plugin-xxx                     2/2     Running
longhorn-driver-deployer-xxx                1/1     Running
longhorn-manager-xxx                        1/1     Running
longhorn-ui-xxx                             1/1     Running
```

**Vérifier la StorageClass créée :**
```bash
microk8s kubectl get storageclass
```

```
NAME                 PROVISIONER          RECLAIMPOLICY
longhorn (default)   driver.longhorn.io   Delete
```

**🎉 Longhorn est installé et prêt !**

### Accéder à l'Interface Web Longhorn

Longhorn fournit une interface web pour gérer le stockage.

**Option 1 : Port-forward (Simple)**
```bash
microk8s kubectl port-forward -n longhorn-system svc/longhorn-frontend 8080:80
```

Ouvrez votre navigateur : `http://localhost:8080`

**Option 2 : Service LoadBalancer (Avec MetalLB)**
```bash
# Éditer le service
microk8s kubectl edit svc longhorn-frontend -n longhorn-system

# Changer "type: ClusterIP" en "type: LoadBalancer"
# Sauvegarder et quitter

# Obtenir l'IP
microk8s kubectl get svc -n longhorn-system longhorn-frontend
```

Accédez via l'IP externe attribuée par MetalLB.

**Option 3 : Ingress (Recommandé pour production)**
```yaml
# longhorn-ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: longhorn-ingress
  namespace: longhorn-system
  annotations:
    nginx.ingress.kubernetes.io/auth-type: basic
    nginx.ingress.kubernetes.io/auth-secret: basic-auth
spec:
  rules:
  - host: longhorn.yourdomain.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: longhorn-frontend
            port:
              number: 80
```

**Créer un mot de passe :**
```bash
# Créer un fichier auth
sudo apt install -y apache2-utils
htpasswd -c auth admin

# Créer le secret
microk8s kubectl -n longhorn-system create secret generic basic-auth --from-file=auth

# Appliquer l'Ingress
microk8s kubectl apply -f longhorn-ingress.yaml
```

Accédez via `http://longhorn.yourdomain.com` (configurez le DNS).

## Utiliser le Stockage Distribué

### Concepts Kubernetes

Avant de créer des volumes, comprenons les concepts :

**PersistentVolume (PV) :**
- Représente un morceau de stockage dans le cluster
- Créé par l'administrateur ou dynamiquement par une StorageClass
- Indépendant du cycle de vie d'un pod

**PersistentVolumeClaim (PVC) :**
- Demande de stockage par un utilisateur/application
- "Je veux 10 GB de stockage"
- Lié automatiquement à un PV disponible

**StorageClass :**
- Définit le "type" de stockage (performance, réplication, etc.)
- Permet le provisioning dynamique
- Longhorn crée automatiquement une StorageClass

**Schéma des relations :**
```
┌──────────────┐         ┌──────────────┐         ┌──────────────┐
│     Pod      │────────►│     PVC      │────────►│     PV       │
│              │  utilise│  (Demande)   │   lié   │  (Stockage)  │
└──────────────┘         └──────────────┘         └──────────────┘
                                                          │
                                                          │ créé par
                                                          ▼
                                                   ┌──────────────┐
                                                   │ StorageClass │
                                                   │  (Longhorn)  │
                                                   └──────────────┘
```

### Créer un PVC et l'Utiliser

**Exemple : Base de données PostgreSQL avec stockage distribué**

**1. Créer un PVC :**
```yaml
# postgres-pvc.yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: postgres-pvc
spec:
  accessModes:
    - ReadWriteOnce  # Un seul pod peut écrire à la fois
  storageClassName: longhorn  # Utiliser Longhorn
  resources:
    requests:
      storage: 10Gi  # Demander 10 GB
```

```bash
microk8s kubectl apply -f postgres-pvc.yaml
```

**Vérifier le PVC :**
```bash
microk8s kubectl get pvc
```

```
NAME           STATUS   VOLUME                                     CAPACITY   STORAGECLASS
postgres-pvc   Bound    pvc-abc123-def456-ghi789                   10Gi       longhorn
```

**STATUS = Bound** → Le PVC est lié à un PV créé automatiquement par Longhorn ! ✅

**2. Créer le Deployment PostgreSQL :**
```yaml
# postgres-deployment.yaml
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
          value: "monmotdepasse"
        - name: PGDATA
          value: /var/lib/postgresql/data/pgdata
        ports:
        - containerPort: 5432
        volumeMounts:
        - name: postgres-storage
          mountPath: /var/lib/postgresql/data
      volumes:
      - name: postgres-storage
        persistentVolumeClaim:
          claimName: postgres-pvc  # ← Utiliser le PVC créé
```

```bash
microk8s kubectl apply -f postgres-deployment.yaml
```

**Vérifier le pod :**
```bash
microk8s kubectl get pods
```

```
NAME                        READY   STATUS    RESTARTS   AGE
postgres-xxxxxxxxxx-xxxxx   1/1     Running   0          30s
```

**3. Tester la résilience :**

**Créer une table dans la base :**
```bash
# Obtenir le nom du pod
POD_NAME=$(microk8s kubectl get pod -l app=postgres -o jsonpath="{.items[0].metadata.name}")

# Se connecter à PostgreSQL
microk8s kubectl exec -it $POD_NAME -- psql -U postgres

# Dans psql:
CREATE TABLE test (id SERIAL PRIMARY KEY, data TEXT);
INSERT INTO test (data) VALUES ('Mon premier enregistrement');
SELECT * FROM test;
\q
```

**Simuler une panne : supprimer le pod**
```bash
microk8s kubectl delete pod $POD_NAME
```

Kubernetes recrée automatiquement le pod (Deployment avec replicas=1).

**Vérifier que les données sont toujours là :**
```bash
# Attendre que le nouveau pod soit prêt
microk8s kubectl get pods -w

# Nouveau nom de pod
POD_NAME=$(microk8s kubectl get pod -l app=postgres -o jsonpath="{.items[0].metadata.name}")

# Se connecter
microk8s kubectl exec -it $POD_NAME -- psql -U postgres

# Dans psql:
SELECT * FROM test;
```

**Le résultat est toujours là !** 🎉

Les données ont survécu à la suppression du pod grâce au stockage distribué.

**4. Tester la mobilité :**

**Voir sur quel nœud le pod tourne :**
```bash
microk8s kubectl get pod $POD_NAME -o wide
```

```
NAME        NODE     STATUS
postgres-xxx node2   Running
```

**Drainer Node 2 (forcer le pod à aller ailleurs) :**
```bash
microk8s kubectl drain node2 --ignore-daemonsets --delete-emptydir-data
```

Le pod est recréé sur node1 ou node3.

**Vérifier les données :**
```bash
# Nouveau pod sur un autre nœud
microk8s kubectl get pod -l app=postgres -o wide

# Se connecter et vérifier
microk8s kubectl exec -it <nouveau-pod> -- psql -U postgres -c "SELECT * FROM test;"
```

**Les données sont toujours accessibles !** Le pod a "suivi" ses données sur le nouveau nœud.

**5. Libérer les ressources :**
```bash
# Supprimer le deployment
microk8s kubectl delete deployment postgres

# Supprimer le PVC (et donc le PV)
microk8s kubectl delete pvc postgres-pvc
```

### Modes d'Accès (Access Modes)

Quand vous créez un PVC, vous devez spécifier le **mode d'accès** :

**1. ReadWriteOnce (RWO) - Le plus courant**
- Volume monté en lecture/écriture par **un seul nœud** à la fois
- Parfait pour : bases de données, applications stateful

**2. ReadOnlyMany (ROX)**
- Volume monté en lecture seule par **plusieurs nœuds** simultanément
- Parfait pour : fichiers de configuration partagés, assets statiques

**3. ReadWriteMany (RWX)**
- Volume monté en lecture/écriture par **plusieurs nœuds** simultanément
- Parfait pour : applications qui partagent des fichiers (CMS, etc.)
- **Note :** Longhorn supporte RWX via NFS

**Tableau de support Longhorn :**
| Mode | Supporté | Usage |
|------|----------|-------|
| RWO | ✅ Oui | Bases de données, apps standard |
| ROX | ✅ Oui | Config partagée en lecture |
| RWX | ✅ Oui (via NFS) | Partage de fichiers |

**Exemple RWX :**
```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: shared-storage
spec:
  accessModes:
    - ReadWriteMany  # ← Plusieurs nœuds peuvent écrire
  storageClassName: longhorn
  resources:
    requests:
      storage: 20Gi
```

## Configuration Avancée de Longhorn

### Réplication et Haute Disponibilité

Par défaut, Longhorn crée **3 replicas** de chaque volume (si vous avez 3+ nœuds).

**Vérifier dans l'interface web Longhorn :**
- Onglet "Volume" → Cliquez sur un volume
- Section "Replicas" : vous verrez 3 replicas sur 3 nœuds différents

**Modifier le nombre de replicas (via la StorageClass) :**
```yaml
# storageclass-longhorn-2replicas.yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: longhorn-2replicas
provisioner: driver.longhorn.io
allowVolumeExpansion: true
parameters:
  numberOfReplicas: "2"  # ← 2 copies au lieu de 3
  staleReplicaTimeout: "2880"
```

```bash
microk8s kubectl apply -f storageclass-longhorn-2replicas.yaml
```

**Utiliser cette StorageClass :**
```yaml
spec:
  storageClassName: longhorn-2replicas
```

**Quand utiliser 2 replicas ?**
- Cluster de 2 nœuds (pas le choix)
- Économiser de l'espace disque
- Données moins critiques

**Quand utiliser 3+ replicas ?**
- Production (recommandé)
- Données critiques
- Cluster de 3+ nœuds

### Snapshots et Backups

**Longhorn supporte les snapshots (instantanés) et backups.**

**Créer un snapshot manuel :**
```bash
# Via l'interface web : Volume → Actions → Take Snapshot

# Ou via kubectl
microk8s kubectl apply -f - <<EOF
apiVersion: longhorn.io/v1beta2
kind: VolumeSnapshot
metadata:
  name: postgres-snapshot-1
  namespace: longhorn-system
spec:
  volumeName: pvc-abc123-def456-ghi789
EOF
```

**Restaurer depuis un snapshot :**
Dans l'interface web Longhorn :
1. Onglet "Volume"
2. Sélectionner le volume
3. Onglet "Snapshot"
4. Cliquer sur "Revert" sur le snapshot souhaité

**Configurer des backups automatiques :**
Longhorn peut sauvegarder vers S3, NFS, etc.

```yaml
# Exemple : Backup vers NFS
apiVersion: v1
kind: Secret
metadata:
  name: backup-target
  namespace: longhorn-system
stringData:
  AWS_ACCESS_KEY_ID: ""
  AWS_SECRET_ACCESS_KEY: ""
  AWS_ENDPOINTS: ""
  BACKUP_TARGET: "nfs://192.168.1.100:/exports/backups"
```

Puis dans les settings Longhorn (interface web) :
- Backup Target : `nfs://192.168.1.100:/exports/backups`

**Créer un backup :**
Interface web → Volume → Actions → Create Backup

### Performances et Tuning

**Optimisations possibles :**

**1. Utiliser des disques SSD**
Les performances de Longhorn dépendent des disques sous-jacents. Des SSD donnent de bien meilleures performances que des HDD.

**2. Éviter l'overcommit**
Ne pas remplir les disques à plus de 80%.

**3. Ajuster les ressources Longhorn**
```bash
# Éditer les deployments Longhorn pour augmenter les ressources
microk8s kubectl edit deployment longhorn-manager -n longhorn-system

# Augmenter CPU/RAM si besoin
resources:
  limits:
    cpu: "1"
    memory: "1Gi"
  requests:
    cpu: "500m"
    memory: "512Mi"
```

**4. Désactiver la réplication pour les données temporaires**
Pour des caches ou logs, utilisez `numberOfReplicas: "1"`.

### Monitoring Longhorn

**Métriques Prometheus :**
Longhorn expose des métriques Prometheus automatiquement.

**Si Prometheus est activé :**
```bash
microk8s enable prometheus
```

Longhorn sera automatiquement scrappé.

**Métriques utiles :**
- `longhorn_volume_actual_size_bytes` : Taille réelle des volumes
- `longhorn_volume_usage_bytes` : Utilisation des volumes
- `longhorn_volume_robustness` : Santé des volumes
- `longhorn_node_storage_capacity_bytes` : Capacité totale par nœud

**Dashboard Grafana pour Longhorn :**
ID: 13032 (sur grafana.com)

## Alternatives et Comparaisons

### Ceph avec Rook

**Si vous avez besoin de plus de puissance que Longhorn :**

```bash
# Installation Rook-Ceph (plus complexe)
microk8s kubectl apply -f https://raw.githubusercontent.com/rook/rook/release-1.12/deploy/examples/crds.yaml
microk8s kubectl apply -f https://raw.githubusercontent.com/rook/rook/release-1.12/deploy/examples/operator.yaml
microk8s kubectl apply -f https://raw.githubusercontent.com/rook/rook/release-1.12/deploy/examples/cluster.yaml
```

**Avantages sur Longhorn :**
- Meilleures performances
- Plus scalable
- Support object storage (S3-compatible)

**Inconvénients :**
- Beaucoup plus complexe
- Consomme beaucoup plus de ressources
- Configuration longue

### NFS Simple

**Si vous voulez du prototypage rapide :**

**1. Configurer un serveur NFS (sur une machine séparée ou un NAS) :**
```bash
# Sur le serveur NFS
sudo apt install nfs-kernel-server
sudo mkdir -p /exports/k8s
sudo chown nobody:nogroup /exports/k8s
sudo chmod 777 /exports/k8s

# Éditer /etc/exports
echo "/exports/k8s 192.168.1.0/24(rw,sync,no_subtree_check,no_root_squash)" | sudo tee -a /etc/exports

# Appliquer
sudo exportfs -ra
```

**2. Installer le provisioner NFS dans Kubernetes :**
```bash
microk8s enable community
microk8s enable nfs
```

**3. Créer une StorageClass NFS :**
```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: nfs
provisioner: nfs.csi.k8s.io
parameters:
  server: 192.168.1.100  # IP du serveur NFS
  share: /exports/k8s
```

**Limitation :** Serveur NFS = SPOF (single point of failure). Pas de vraie HA.

## Bonnes Pratiques

### 1. Toujours Utiliser des PVC en Production

**❌ Mauvais (stockage éphémère) :**
```yaml
volumes:
- name: data
  emptyDir: {}
```

Données perdues à chaque redémarrage du pod.

**✅ Bon (stockage persistant) :**
```yaml
volumes:
- name: data
  persistentVolumeClaim:
    claimName: my-pvc
```

### 2. Définir des Resource Requests/Limits

**Sur le PVC :**
```yaml
resources:
  requests:
    storage: 10Gi  # Quantité demandée
```

**Sur les pods Longhorn :**
Assurez-vous que les pods Longhorn ont assez de ressources.

### 3. Surveiller l'Utilisation du Stockage

```bash
# Voir l'utilisation des PVC
microk8s kubectl get pvc

# Détails d'un PVC
microk8s kubectl describe pvc postgres-pvc
```

Dans l'interface Longhorn :
- Onglet "Node" → Voir l'espace disque disponible sur chaque nœud
- Configurer des alertes quand < 20% d'espace libre

### 4. Faire des Backups Réguliers

Même avec la réplication, faire des backups hors cluster :
- Snapshots Longhorn réguliers
- Backups vers stockage externe (S3, NFS)
- Tests de restauration

**Automatiser avec CronJobs :**
```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: longhorn-backup
spec:
  schedule: "0 2 * * *"  # Tous les jours à 2h du matin
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: backup
            image: longhornio/longhorn-manager
            command:
            - /bin/sh
            - -c
            - |
              # Script de backup
              kubectl -n longhorn-system exec deploy/longhorn-manager -- \
                longhorn-manager backup create --volume pvc-abc123
          restartPolicy: OnFailure
```

### 5. Nettoyer les PVC Inutilisés

**Lister les PVC non utilisés :**
```bash
# PVC sans pod associé
microk8s kubectl get pvc --all-namespaces
```

**Supprimer un PVC :**
```bash
microk8s kubectl delete pvc <nom-du-pvc>
```

**Attention :** La suppression d'un PVC supprime aussi le PV et donc **les données** (si `reclaimPolicy: Delete`).

### 6. Utiliser des Labels et Annotations

**Organiser vos PVC :**
```yaml
metadata:
  name: postgres-pvc
  labels:
    app: postgres
    environment: production
    backup: enabled
  annotations:
    description: "Stockage pour PostgreSQL production"
```

## Dépannage

### Problème : PVC reste en "Pending"

**Symptôme :**
```bash
microk8s kubectl get pvc
```
```
NAME           STATUS    VOLUME   CAPACITY
postgres-pvc   Pending
```

**Causes possibles :**

**1. Longhorn pas installé ou pods pas prêts**
```bash
microk8s kubectl get pods -n longhorn-system
```

Vérifier que tous les pods sont en `Running`.

**2. StorageClass incorrecte**
```bash
microk8s kubectl get storageclass
```

Vérifier que la StorageClass existe.

**3. Pas assez d'espace disque**
```bash
df -h /var/lib/longhorn
```

Libérer de l'espace si nécessaire.

**4. Logs du provisioner**
```bash
microk8s kubectl logs -n longhorn-system -l app=longhorn-manager
```

### Problème : Volume en "Degraded"

**Symptôme :**
Dans l'interface Longhorn, un volume apparaît en rouge/orange "Degraded".

**Cause :**
Un ou plusieurs replicas ne sont pas sains.

**Solution :**
1. Interface Longhorn → Volume → Voir les replicas
2. Identifier le replica défaillant
3. Supprimer le replica défaillant (il sera recréé automatiquement)
4. Ou redémarrer le nœud problématique

### Problème : Performances Lentes

**Diagnostic :**

**1. Vérifier les disques sous-jacents**
```bash
# Test de performance I/O
sudo apt install fio
fio --name=randwrite --ioengine=libaio --iodepth=16 --rw=randwrite --bs=4k --direct=1 --size=1G --numjobs=4 --runtime=60 --group_reporting --filename=/var/lib/longhorn/test
```

**2. Vérifier la latence réseau**
```bash
ping -c 100 <autre-noeud>
```

Si > 10ms de latence moyenne ou > 1% de paquets perdus → problème réseau.

**3. Vérifier les ressources Longhorn**
```bash
microk8s kubectl top pods -n longhorn-system
```

Si CPU/RAM saturés, augmenter les ressources.

**Solutions :**
- Utiliser des SSD au lieu de HDD
- Optimiser le réseau (switch Gigabit, cables Cat6)
- Réduire le nombre de replicas (de 3 à 2) pour les données moins critiques

### Problème : Nœud Plein

**Symptôme :**
```
Events:
  Warning  FailedScheduling  pod/xxx: 0/3 nodes available: insufficient storage
```

**Solution :**

**1. Voir l'espace par nœud (interface Longhorn)**
Onglet "Node"

**2. Nettoyer les anciens snapshots**
Interface Longhorn → Volume → Snapshots → Supprimer les vieux

**3. Augmenter la capacité**
- Ajouter un disque supplémentaire sur un nœud
- Ajouter un nouveau nœud

**4. Configurer l'overprovisioning**
Par défaut, Longhorn réserve 25% d'espace libre. Vous pouvez le réduire (non recommandé en production).

Settings → `Storage Over Provisioning Percentage`

## Points Clés à Retenir

1. **Stockage local** = lié à un nœud, pas de résilience
2. **Stockage distribué** = données répliquées, accessibles depuis n'importe quel nœud
3. **Longhorn** = solution idéale pour homelabs et PME (simple, léger, efficace)
4. **3 replicas** par défaut = haute disponibilité
5. **PVC** = demande de stockage, **PV** = stockage réel
6. **RWO** = un pod à la fois (DB), **RWX** = plusieurs pods (partage de fichiers)
7. **Interface web** Longhorn = gestion facile, snapshots, backups
8. **Monitoring** = surveiller l'espace disque et les volumes degraded
9. **Backups** = même avec réplication, toujours faire des backups externes
10. **SSD recommandé** pour de bonnes performances

## Prochaines Étapes

Vous maîtrisez maintenant le stockage distribué ! Les prochains chapitres couvriront :

- **21.8** : Backup du control plane et stratégies de sauvegarde complètes
- **21.9** : Stratégies de redondance avancées
- **21.10** : Tests de failover et validation de la résilience

Avec Longhorn, vos applications stateful peuvent maintenant profiter de la haute disponibilité et de la résilience du cluster multi-node ! 🚀

---

**Félicitations !** Vous savez maintenant configurer et gérer du stockage distribué dans votre cluster Kubernetes. C'est la dernière pièce du puzzle pour avoir un cluster multi-node véritablement résilient et prêt pour la production !

⏭️ [Backup du control plane](/21-multi-node-et-haute-disponibilite/08-backup-du-control-plane.md)
