🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 22.4 Snapshots de volumes

## Introduction

Les **snapshots de volumes** sont des copies instantanées de vos volumes persistants à un moment précis. Imaginez qu'ils fonctionnent comme une "photo" de l'état d'un disque dur à un instant T. Cette section explore comment créer, gérer et restaurer des snapshots de volumes dans le contexte de MicroK8s.

### Analogie pour comprendre les snapshots

Pensez à un snapshot comme à une **machine à voyager dans le temps** pour vos données :

- **Photo traditionnelle** (backup par fichier) : Vous prenez en photo chaque objet de votre maison, un par un. Cela prend du temps, mais vous pouvez choisir exactement ce que vous photographiez.

- **Snapshot** : Vous figez instantanément toute la maison dans le temps. En un clic, tout est capturé simultanément. C'est beaucoup plus rapide, mais vous capturez tout ou rien.

Les snapshots sont particulièrement utiles pour les applications avec beaucoup de données, comme les bases de données, où la vitesse et la cohérence sont cruciales.

## Différence entre snapshots et backups par fichier

Comprendre la différence est essentiel pour choisir la bonne approche.

### Snapshots de volumes (natifs)

**Comment ça fonctionne** :
- Le système de stockage sous-jacent crée une "image" du volume
- C'est quasi-instantané (quelques secondes)
- Utilise des techniques comme le Copy-on-Write (CoW)
- Géré au niveau du système de stockage, pas au niveau fichier

**Avantages** :
- ⚡ Extrêmement rapide (secondes, même pour des To)
- 🎯 Cohérence garantie (tout est capturé au même instant)
- 💾 Économe en espace (seuls les changements sont stockés)
- 🔄 Restauration très rapide

**Inconvénients** :
- 🔒 Nécessite un support au niveau du stockage
- 💰 Souvent limité aux solutions cloud (AWS EBS, GCP Disk, Azure)
- 🏠 Difficile à implémenter sur du stockage local simple

### Backups par fichier (Restic/Kopia)

**Comment ça fonctionne** :
- Copie fichier par fichier le contenu du volume
- Peut prendre plusieurs minutes à heures selon la taille
- Fonctionne avec n'importe quel type de stockage

**Avantages** :
- 🌍 Universel (fonctionne partout)
- 🎯 Sélectif (peut exclure certains fichiers)
- 🔐 Chiffrement natif
- 💾 Déduplication (économie d'espace)

**Inconvénients** :
- 🐌 Plus lent (proportionnel à la taille des données)
- ⚙️ Utilise plus de ressources CPU/RAM
- ⏱️ Moins de garantie de cohérence instantanée

### Tableau comparatif

| Critère | Snapshots natifs | Restic/Kopia |
|---------|------------------|--------------|
| Vitesse de création | Secondes | Minutes à heures |
| Vitesse de restauration | Secondes à minutes | Minutes à heures |
| Espace requis | Optimisé (CoW) | Déduplication |
| Compatibilité | Limitée (cloud) | Universelle |
| Cohérence | Parfaite | Bonne (avec hooks) |
| Coût | Variable (cloud) | Gratuit |
| Complexité | Simple | Modérée |

### Quelle approche choisir ?

**Utilisez les snapshots natifs si** :
- Vous êtes sur un cloud provider (AWS, GCP, Azure)
- Vous avez de gros volumes (> 100 GB)
- La vitesse est critique
- Votre stockage supporte les snapshots

**Utilisez Restic/Kopia si** :
- Vous êtes sur MicroK8s avec stockage local (hostpath)
- Vous voulez du chiffrement de bout en bout
- Vous voulez de la déduplication
- Vous n'avez pas de support de snapshot natif

**Pour MicroK8s** : Dans la plupart des cas, vous utiliserez **Restic/Kopia** car le stockage par défaut (`hostpath-storage`) ne supporte pas les snapshots natifs.

## Comment fonctionnent les snapshots

### Principe du Copy-on-Write (CoW)

La plupart des systèmes de snapshot utilisent la technique **Copy-on-Write** :

1. **Création du snapshot** :
   - Le système marque tous les blocs actuels comme "snapshot"
   - Aucune copie réelle n'est faite (instantané !)

2. **Modifications ultérieures** :
   - Quand un bloc est modifié, l'ancienne version est copiée dans le snapshot
   - La nouvelle version est écrite normalement

3. **Résultat** :
   - Le snapshot ne contient que les différences
   - Très économe en espace

**Illustration** :

```
État initial (100 GB):
Volume: [A][B][C][D][E]...

Création snapshot (0 GB supplémentaire):
Volume:    [A][B][C][D][E]...
Snapshot:  -> pointe vers Volume

Modification de B et D (2 GB):
Volume:    [A][B'][C][D'][E]...
Snapshot:  [B][D] (stocke les anciennes versions)

Espace utilisé: 102 GB au lieu de 200 GB
```

### Snapshots incrémentaux vs complets

**Snapshots complets** :
- Capture tout l'état du volume
- Premier snapshot toujours complet
- Taille = taille des données utilisées

**Snapshots incrémentaux** :
- Ne stockent que les différences depuis le dernier snapshot
- Beaucoup plus compacts
- Dépendent du snapshot précédent

La plupart des systèmes font automatiquement de l'incrémental.

## Snapshots dans Kubernetes

Kubernetes a standardisé la gestion des snapshots avec les **VolumeSnapshots**.

### Architecture des VolumeSnapshots

Kubernetes utilise trois ressources principales :

1. **VolumeSnapshot** : La demande de snapshot (comme un PVC pour les volumes)
2. **VolumeSnapshotClass** : Le "type" de snapshot (comme StorageClass)
3. **VolumeSnapshotContent** : Le snapshot réel (comme un PV)

**Analogie** :
- **VolumeSnapshot** = "Je veux une photo de ce volume"
- **VolumeSnapshotClass** = "Utilise cet appareil photo"
- **VolumeSnapshotContent** = La photo elle-même

### Prérequis pour les snapshots Kubernetes

Pour utiliser les VolumeSnapshots natifs, il faut :

1. **Un CSI driver** qui supporte les snapshots
2. **VolumeSnapshot CRDs** installés dans le cluster
3. **Snapshot Controller** en cours d'exécution
4. **Stockage sous-jacent** qui supporte les snapshots

### Limitations avec MicroK8s hostpath

Par défaut, MicroK8s utilise `hostpath-storage` qui **ne supporte pas** les snapshots natifs car :
- Pas de CSI driver avec support snapshot
- Stockage local simple (pas de CoW)
- Pas de système de fichiers avancé (ZFS, Btrfs) configuré

**Solutions pour MicroK8s** :
1. Utiliser Restic/Kopia avec Velero (recommandé)
2. Installer un CSI driver compatible (OpenEBS, Longhorn)
3. Utiliser un système de fichiers avec snapshots (ZFS, Btrfs)

## Installer le support des snapshots sur MicroK8s

Si vous voulez vraiment utiliser des snapshots natifs avec MicroK8s, voici les options.

### Option 1 : OpenEBS (recommandé)

**OpenEBS** est une solution de stockage cloud-native qui supporte les snapshots.

#### Installation d'OpenEBS

```bash
# Activer l'addon OpenEBS (si disponible)
microk8s enable openebs

# Ou installer manuellement avec Helm
microk8s helm3 repo add openebs https://openebs.github.io/charts
microk8s helm3 repo update
microk8s helm3 install openebs openebs/openebs -n openebs --create-namespace
```

#### Installer les snapshot CRDs

```bash
# Installer les CRDs VolumeSnapshot
kubectl apply -f https://raw.githubusercontent.com/kubernetes-csi/external-snapshotter/master/client/config/crd/snapshot.storage.k8s.io_volumesnapshotclasses.yaml
kubectl apply -f https://raw.githubusercontent.com/kubernetes-csi/external-snapshotter/master/client/config/crd/snapshot.storage.k8s.io_volumesnapshotcontents.yaml
kubectl apply -f https://raw.githubusercontent.com/kubernetes-csi/external-snapshotter/master/client/config/crd/snapshot.storage.k8s.io_volumesnapshots.yaml

# Installer le Snapshot Controller
kubectl apply -f https://raw.githubusercontent.com/kubernetes-csi/external-snapshotter/master/deploy/kubernetes/snapshot-controller/rbac-snapshot-controller.yaml
kubectl apply -f https://raw.githubusercontent.com/kubernetes-csi/external-snapshotter/master/deploy/kubernetes/snapshot-controller/setup-snapshot-controller.yaml
```

#### Vérifier l'installation

```bash
# Vérifier les CRDs
kubectl get crd | grep snapshot

# Vérifier le Snapshot Controller
kubectl get pods -n kube-system | grep snapshot-controller

# Vérifier OpenEBS
kubectl get pods -n openebs
```

#### Créer une VolumeSnapshotClass pour OpenEBS

```yaml
apiVersion: snapshot.storage.k8s.io/v1
kind: VolumeSnapshotClass
metadata:
  name: openebs-snapshot-class
driver: cstor.csi.openebs.io
deletionPolicy: Delete
```

Appliquer :

```bash
kubectl apply -f openebs-snapshot-class.yaml
```

### Option 2 : Longhorn

**Longhorn** est une autre solution de stockage distribuée qui supporte les snapshots.

```bash
# Installer Longhorn avec Helm
microk8s helm3 repo add longhorn https://charts.longhorn.io
microk8s helm3 repo update
microk8s helm3 install longhorn longhorn/longhorn \
  --namespace longhorn-system \
  --create-namespace
```

Les CRDs de snapshot et le controller doivent être installés de la même manière qu'avec OpenEBS.

### Option 3 : ZFS sur le système hôte

Si vous êtes à l'aise avec Linux, vous pouvez utiliser ZFS :

```bash
# Installer ZFS (Ubuntu)
sudo apt install zfsutils-linux

# Créer un pool ZFS
sudo zpool create mypool /dev/sdX

# Créer un filesystem ZFS pour MicroK8s
sudo zfs create mypool/microk8s

# Monter pour MicroK8s
sudo zfs set mountpoint=/var/snap/microk8s/common/volumes mypool/microk8s
```

Ensuite, utilisez des scripts pour créer des snapshots ZFS manuellement.

### Option 4 : Rester avec Restic/Kopia (le plus simple)

Pour un lab MicroK8s, **la solution la plus pragmatique** reste d'utiliser Restic/Kopia via Velero, comme vu dans les sections précédentes. C'est :
- Plus simple à mettre en place
- Fonctionne immédiatement
- Ne nécessite pas de stockage spécial
- Suffisant pour la plupart des besoins de lab

## Créer des snapshots de volumes

### Avec VolumeSnapshot natif (si supporté)

Supposons que vous avez installé OpenEBS et les CRDs.

#### Créer un snapshot d'un PVC existant

```yaml
apiVersion: snapshot.storage.k8s.io/v1
kind: VolumeSnapshot
metadata:
  name: pvc-snapshot-demo
  namespace: default
spec:
  volumeSnapshotClassName: openebs-snapshot-class
  source:
    persistentVolumeClaimName: data-pvc
```

Appliquer :

```bash
kubectl apply -f volume-snapshot.yaml
```

#### Vérifier le snapshot

```bash
# Lister les snapshots
kubectl get volumesnapshot

# Détails du snapshot
kubectl describe volumesnapshot pvc-snapshot-demo
```

Résultat attendu :

```
NAME                AGE   READY   SOURCE PVC   SOURCE SNAPSHOT   SIZE
pvc-snapshot-demo   10s   true    data-pvc                       5Gi
```

#### Créer un snapshot programmatiquement

Vous pouvez automatiser avec un script :

```bash
#!/bin/bash
PVC_NAME="data-pvc"
NAMESPACE="default"
SNAPSHOT_NAME="${PVC_NAME}-$(date +%Y%m%d-%H%M%S)"

kubectl apply -f - <<EOF
apiVersion: snapshot.storage.k8s.io/v1
kind: VolumeSnapshot
metadata:
  name: ${SNAPSHOT_NAME}
  namespace: ${NAMESPACE}
spec:
  volumeSnapshotClassName: openebs-snapshot-class
  source:
    persistentVolumeClaimName: ${PVC_NAME}
EOF

echo "Snapshot créé: ${SNAPSHOT_NAME}"
```

### Avec Velero (approche Restic/Kopia)

Si vous utilisez Velero avec Restic/Kopia :

```bash
# Créer un backup incluant les volumes
velero backup create snapshot-backup \
  --include-namespaces myapp \
  --default-volumes-to-fs-backup

# Vérifier
velero backup describe snapshot-backup
```

Les "snapshots" sont en réalité des copies par fichier, mais du point de vue de l'utilisateur, le résultat est similaire.

## Restaurer depuis un snapshot

### Restauration avec VolumeSnapshot natif

Pour restaurer depuis un snapshot, vous créez un **nouveau PVC** qui utilise le snapshot comme source.

#### Créer un PVC depuis un snapshot

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: restored-pvc
  namespace: default
spec:
  dataSource:
    name: pvc-snapshot-demo
    kind: VolumeSnapshot
    apiGroup: snapshot.storage.k8s.io
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 5Gi
  storageClassName: openebs-cstor-csi
```

Appliquer :

```bash
kubectl apply -f restored-pvc.yaml
```

#### Utiliser le PVC restauré dans un Pod

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: restored-app
  namespace: default
spec:
  containers:
  - name: app
    image: nginx
    volumeMounts:
    - name: data
      mountPath: /data
  volumes:
  - name: data
    persistentVolumeClaim:
      claimName: restored-pvc
```

Le Pod utilisera les données telles qu'elles étaient au moment du snapshot.

### Restauration avec Velero

```bash
# Restaurer tout le backup (incluant les volumes)
velero restore create --from-backup snapshot-backup

# Ou restaurer seulement un namespace
velero restore create --from-backup snapshot-backup \
  --include-namespaces myapp
```

Velero restaure automatiquement les PVCs avec leurs données.

## Gestion des snapshots

### Lister les snapshots

```bash
# Tous les snapshots
kubectl get volumesnapshot --all-namespaces

# Dans un namespace spécifique
kubectl get volumesnapshot -n default

# Format détaillé
kubectl get volumesnapshot -o wide
```

### Voir les détails d'un snapshot

```bash
kubectl describe volumesnapshot pvc-snapshot-demo
```

Informations affichées :
- Taille du snapshot
- PVC source
- État (Ready/NotReady)
- VolumeSnapshotContent associé
- Erreurs éventuelles

### Supprimer un snapshot

```bash
kubectl delete volumesnapshot pvc-snapshot-demo
```

**Attention** : Selon la `deletionPolicy` de la VolumeSnapshotClass, cela peut :
- Supprimer le snapshot sous-jacent (`Delete`)
- Conserver le snapshot mais le détacher (`Retain`)

### Définir la politique de suppression

Dans la VolumeSnapshotClass :

```yaml
apiVersion: snapshot.storage.k8s.io/v1
kind: VolumeSnapshotClass
metadata:
  name: openebs-snapshot-class
driver: cstor.csi.openebs.io
deletionPolicy: Retain  # ou Delete
```

- **Delete** : Supprime le snapshot quand la ressource est supprimée
- **Retain** : Conserve le snapshot (gestion manuelle nécessaire)

## Automatisation des snapshots

### Snapshots programmés avec CronJobs

Créez un CronJob qui crée des snapshots régulièrement :

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: daily-volume-snapshot
  namespace: default
spec:
  schedule: "0 2 * * *"  # Tous les jours à 2h
  jobTemplate:
    spec:
      template:
        spec:
          serviceAccountName: snapshot-creator
          containers:
          - name: kubectl
            image: bitnami/kubectl:latest
            command:
            - /bin/bash
            - -c
            - |
              PVC_NAME="data-pvc"
              SNAPSHOT_NAME="daily-snapshot-$(date +%Y%m%d-%H%M%S)"

              kubectl apply -f - <<EOF
              apiVersion: snapshot.storage.k8s.io/v1
              kind: VolumeSnapshot
              metadata:
                name: ${SNAPSHOT_NAME}
                namespace: default
              spec:
                volumeSnapshotClassName: openebs-snapshot-class
                source:
                  persistentVolumeClaimName: ${PVC_NAME}
              EOF

              echo "Snapshot créé: ${SNAPSHOT_NAME}"
          restartPolicy: OnFailure
```

#### Créer le ServiceAccount nécessaire

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: snapshot-creator
  namespace: default
---
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: snapshot-creator-role
  namespace: default
rules:
- apiGroups: ["snapshot.storage.k8s.io"]
  resources: ["volumesnapshots"]
  verbs: ["create", "get", "list"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: snapshot-creator-binding
  namespace: default
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: snapshot-creator-role
subjects:
- kind: ServiceAccount
  name: snapshot-creator
  namespace: default
```

### Nettoyage automatique des vieux snapshots

CronJob pour supprimer les snapshots de plus de 7 jours :

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: cleanup-old-snapshots
  namespace: default
spec:
  schedule: "0 3 * * *"  # Tous les jours à 3h
  jobTemplate:
    spec:
      template:
        spec:
          serviceAccountName: snapshot-creator
          containers:
          - name: kubectl
            image: bitnami/kubectl:latest
            command:
            - /bin/bash
            - -c
            - |
              # Supprimer les snapshots de plus de 7 jours
              CUTOFF_DATE=$(date -d '7 days ago' +%s)

              kubectl get volumesnapshot -o json | \
              jq -r '.items[] |
                select(.metadata.creationTimestamp | fromdateiso8601 < '$CUTOFF_DATE') |
                .metadata.name' | \
              while read snapshot; do
                echo "Suppression de $snapshot"
                kubectl delete volumesnapshot $snapshot
              done
          restartPolicy: OnFailure
```

### Avec Velero Schedules

Si vous utilisez Velero, les schedules créent automatiquement des "snapshots" (backups) :

```bash
velero schedule create daily-snapshot \
  --schedule="0 2 * * *" \
  --ttl 168h \
  --default-volumes-to-fs-backup
```

La gestion de la rétention est automatique avec le TTL.

## Snapshots pour les bases de données

Les bases de données nécessitent une attention particulière pour garantir la cohérence.

### Principe : Freeze/Thaw

Pour garantir la cohérence :

1. **Freeze** : Figer les écritures (flush les buffers)
2. **Snapshot** : Créer le snapshot
3. **Thaw** : Reprendre les écritures

### PostgreSQL

#### Avec des hooks dans un Pod

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: postgres
  annotations:
    # Avant snapshot : checkpoint et freeze
    pre.hook.backup.velero.io/command: >-
      ["/bin/bash", "-c",
       "psql -U postgres -c 'CHECKPOINT;' &&
        psql -U postgres -c 'SELECT pg_start_backup(\"snapshot\");'"]

    # Après snapshot : reprendre
    post.hook.backup.velero.io/command: >-
      ["/bin/bash", "-c",
       "psql -U postgres -c 'SELECT pg_stop_backup();'"]
```

#### Script manuel pour snapshot

```bash
#!/bin/bash
POD_NAME="postgres-pod"
NAMESPACE="database"
PVC_NAME="postgres-data"

# Freeze PostgreSQL
kubectl exec -n $NAMESPACE $POD_NAME -- \
  psql -U postgres -c "SELECT pg_start_backup('snapshot');"

# Créer le snapshot
kubectl apply -f - <<EOF
apiVersion: snapshot.storage.k8s.io/v1
kind: VolumeSnapshot
metadata:
  name: postgres-snapshot-$(date +%Y%m%d-%H%M%S)
  namespace: $NAMESPACE
spec:
  volumeSnapshotClassName: openebs-snapshot-class
  source:
    persistentVolumeClaimName: $PVC_NAME
EOF

# Attendre que le snapshot soit prêt
sleep 5

# Thaw PostgreSQL
kubectl exec -n $NAMESPACE $POD_NAME -- \
  psql -U postgres -c "SELECT pg_stop_backup();"

echo "Snapshot PostgreSQL créé avec succès"
```

### MySQL/MariaDB

```bash
#!/bin/bash
POD_NAME="mysql-pod"
NAMESPACE="database"

# Flush et lock
kubectl exec -n $NAMESPACE $POD_NAME -- \
  mysql -u root -p$MYSQL_ROOT_PASSWORD -e "FLUSH TABLES WITH READ LOCK;"

# Créer snapshot
# ... (même logique que PostgreSQL)

# Unlock
kubectl exec -n $NAMESPACE $POD_NAME -- \
  mysql -u root -p$MYSQL_ROOT_PASSWORD -e "UNLOCK TABLES;"
```

### MongoDB

```bash
#!/bin/bash
POD_NAME="mongodb-pod"
NAMESPACE="database"

# Forcer un sync
kubectl exec -n $NAMESPACE $POD_NAME -- \
  mongo admin -u root -p$MONGO_ROOT_PASSWORD --eval "db.fsyncLock()"

# Créer snapshot
# ...

# Unlock
kubectl exec -n $NAMESPACE $POD_NAME -- \
  mongo admin -u root -p$MONGO_ROOT_PASSWORD --eval "db.fsyncUnlock()"
```

## Snapshots et clonage de volumes

Les snapshots sont parfaits pour le clonage rapide d'environnements.

### Créer un environnement de staging depuis production

#### Étape 1 : Créer un snapshot de production

```yaml
apiVersion: snapshot.storage.k8s.io/v1
kind: VolumeSnapshot
metadata:
  name: prod-snapshot
  namespace: production
spec:
  volumeSnapshotClassName: openebs-snapshot-class
  source:
    persistentVolumeClaimName: production-data
```

#### Étape 2 : Créer un PVC de staging depuis le snapshot

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: staging-data
  namespace: staging
spec:
  dataSource:
    name: prod-snapshot
    kind: VolumeSnapshot
    apiGroup: snapshot.storage.k8s.io
    # Note: Le snapshot peut être dans un autre namespace
    # mais cela dépend du CSI driver
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 10Gi
  storageClassName: openebs-cstor-csi
```

#### Étape 3 : Déployer l'application de staging

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: staging-app
  namespace: staging
spec:
  replicas: 1
  selector:
    matchLabels:
      app: myapp
      env: staging
  template:
    metadata:
      labels:
        app: myapp
        env: staging
    spec:
      containers:
      - name: app
        image: myapp:latest
        volumeMounts:
        - name: data
          mountPath: /data
      volumes:
      - name: data
        persistentVolumeClaim:
          claimName: staging-data
```

### Clonage pour les développeurs

Créer rapidement un environnement de développement :

```bash
#!/bin/bash
# Script pour cloner un environnement

SOURCE_NS="production"
SOURCE_PVC="app-data"
TARGET_NS="dev-john"
TARGET_PVC="app-data"

# Créer le namespace de dev
kubectl create namespace $TARGET_NS

# Créer snapshot
SNAPSHOT_NAME="clone-$(date +%Y%m%d-%H%M%S)"
kubectl apply -f - <<EOF
apiVersion: snapshot.storage.k8s.io/v1
kind: VolumeSnapshot
metadata:
  name: ${SNAPSHOT_NAME}
  namespace: ${SOURCE_NS}
spec:
  volumeSnapshotClassName: openebs-snapshot-class
  source:
    persistentVolumeClaimName: ${SOURCE_PVC}
EOF

# Attendre que le snapshot soit prêt
echo "Attente du snapshot..."
kubectl wait --for=condition=ready volumesnapshot/${SNAPSHOT_NAME} -n ${SOURCE_NS} --timeout=300s

# Créer PVC dans le namespace dev
kubectl apply -f - <<EOF
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: ${TARGET_PVC}
  namespace: ${TARGET_NS}
spec:
  dataSource:
    name: ${SNAPSHOT_NAME}
    kind: VolumeSnapshot
    apiGroup: snapshot.storage.k8s.io
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 10Gi
  storageClassName: openebs-cstor-csi
EOF

echo "Environnement de dev ${TARGET_NS} créé avec succès"
```

## Monitoring et métriques des snapshots

### Surveiller l'état des snapshots

Script de surveillance basique :

```bash
#!/bin/bash
# Vérifier l'état de tous les snapshots

echo "=== État des VolumeSnapshots ==="
kubectl get volumesnapshot --all-namespaces -o json | \
jq -r '.items[] |
  "\(.metadata.namespace)/\(.metadata.name): \(.status.readyToUse) (Age: \(.metadata.creationTimestamp))"'

echo ""
echo "=== Snapshots non prêts ==="
kubectl get volumesnapshot --all-namespaces -o json | \
jq -r '.items[] |
  select(.status.readyToUse != true) |
  "\(.metadata.namespace)/\(.metadata.name): \(.status)"'
```

### Métriques Prometheus

Si vous utilisez le Snapshot Controller avec Prometheus :

```yaml
apiVersion: v1
kind: Service
metadata:
  name: snapshot-controller-metrics
  namespace: kube-system
  labels:
    app: snapshot-controller
spec:
  selector:
    app: snapshot-controller
  ports:
  - name: metrics
    port: 8080
    targetPort: 8080
```

Métriques disponibles :
- `snapshot_controller_operation_total_seconds` : Durée des opérations
- `snapshot_controller_operations_total` : Nombre d'opérations
- `snapshot_controller_volumesnapshot_count` : Nombre de snapshots

### Alertes recommandées

```yaml
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: volume-snapshot-alerts
  namespace: monitoring
spec:
  groups:
  - name: snapshots
    interval: 30s
    rules:
    - alert: VolumeSnapshotFailed
      expr: |
        snapshot_controller_operations_total{status="failed"} > 0
      for: 5m
      labels:
        severity: warning
      annotations:
        summary: "Volume snapshot operation failed"
        description: "A volume snapshot operation has failed"

    - alert: VolumeSnapshotStuck
      expr: |
        time() - snapshot_controller_operation_total_seconds > 3600
      for: 10m
      labels:
        severity: critical
      annotations:
        summary: "Volume snapshot stuck"
        description: "A snapshot operation has been running for over 1 hour"
```

## Limitations et considérations

### Limitations générales

1. **Dépendance au stockage** :
   - Nécessite un CSI driver compatible
   - Pas universel comme Restic/Kopia

2. **Performance** :
   - Même si "instantané", il y a un overhead
   - Impact possible sur les performances du stockage

3. **Espace disque** :
   - Les snapshots consomment de l'espace
   - Plus il y a de modifications, plus ils prennent de place

4. **Chaîne de dépendances** :
   - Les snapshots incrémentaux dépendent des précédents
   - Supprimer un snapshot parent peut causer des problèmes

### Limitations spécifiques MicroK8s

1. **Hostpath par défaut** :
   - Pas de support natif des snapshots
   - Nécessite installation d'OpenEBS/Longhorn

2. **Ressources limitées** :
   - OpenEBS peut être gourmand en ressources pour un lab
   - Restic/Kopia est souvent plus adapté

3. **Complexité** :
   - Setup plus complexe que Velero seul
   - Courbe d'apprentissage supplémentaire

### Quand ne pas utiliser les snapshots natifs

**Évitez les snapshots natifs si** :
- Vous avez un simple lab MicroK8s sur un laptop
- Vous manquez de ressources (< 8 GB RAM)
- Vous voulez la simplicité
- Vous n'avez pas de stockage avancé

**Privilégiez Restic/Kopia si** :
- Vous débutez avec Kubernetes
- Vous voulez une solution qui marche partout
- Vous voulez du chiffrement intégré
- Vous voulez de la déduplication

## Comparaison des solutions de snapshot pour MicroK8s

### Tableau récapitulatif

| Solution | Complexité | Performance | Ressources | Universel | Recommandé pour |
|----------|-----------|-------------|------------|-----------|-----------------|
| Hostpath + Restic | ⭐ Faible | ⭐⭐ Moyenne | ⭐⭐⭐ Faible | ✅ Oui | Labs, débutants |
| OpenEBS | ⭐⭐⭐ Élevée | ⭐⭐⭐ Bonne | ⭐⭐ Moyenne | ❌ Non | Production, apprentissage avancé |
| Longhorn | ⭐⭐⭐ Élevée | ⭐⭐⭐ Bonne | ⭐⭐ Moyenne | ❌ Non | Multi-node, HA |
| ZFS manuel | ⭐⭐⭐⭐ Très élevée | ⭐⭐⭐⭐ Excellente | ⭐⭐⭐ Faible | ❌ Non | Experts Linux |
| Cloud provider | ⭐⭐ Moyenne | ⭐⭐⭐⭐ Excellente | ⭐⭐⭐ Faible | ❌ Non | Production cloud |

### Recommandation finale pour MicroK8s

Pour un **lab personnel** :
```
Hostpath + Restic/Kopia via Velero
```

Pour un **environnement de production léger** :
```
OpenEBS avec VolumeSnapshots natifs
```

Pour un **cluster multi-node** :
```
Longhorn avec snapshots et réplication
```

Pour un **apprentissage approfondi** :
```
OpenEBS ou ZFS pour comprendre les mécanismes
```

## Scripts et automatisation utiles

### Script complet de gestion des snapshots

```bash
#!/bin/bash
# snapshot-manager.sh - Outil de gestion des snapshots

ACTION=$1
PVC_NAME=$2
NAMESPACE=${3:-default}
SNAPSHOT_CLASS=${4:-openebs-snapshot-class}

case $ACTION in
  create)
    SNAPSHOT_NAME="${PVC_NAME}-$(date +%Y%m%d-%H%M%S)"
    echo "Création du snapshot: $SNAPSHOT_NAME"

    kubectl apply -f - <<EOF
apiVersion: snapshot.storage.k8s.io/v1
kind: VolumeSnapshot
metadata:
  name: ${SNAPSHOT_NAME}
  namespace: ${NAMESPACE}
spec:
  volumeSnapshotClassName: ${SNAPSHOT_CLASS}
  source:
    persistentVolumeClaimName: ${PVC_NAME}
EOF

    echo "Attente de la création..."
    kubectl wait --for=condition=ready volumesnapshot/${SNAPSHOT_NAME} \
      -n ${NAMESPACE} --timeout=300s
    echo "Snapshot créé: ${SNAPSHOT_NAME}"
    ;;

  list)
    echo "=== Snapshots dans ${NAMESPACE} ==="
    kubectl get volumesnapshot -n ${NAMESPACE}
    ;;

  restore)
    SNAPSHOT_NAME=$PVC_NAME  # Dans ce cas, c'est le nom du snapshot
    NEW_PVC="${SNAPSHOT_NAME}-restored"
    echo "Restauration depuis ${SNAPSHOT_NAME} vers ${NEW_PVC}"

    kubectl apply -f - <<EOF
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: ${NEW_PVC}
  namespace: ${NAMESPACE}
spec:
  dataSource:
    name: ${SNAPSHOT_NAME}
    kind: VolumeSnapshot
    apiGroup: snapshot.storage.k8s.io
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 10Gi
  storageClassName: openebs-cstor-csi
EOF

    echo "PVC restauré: ${NEW_PVC}"
    ;;

  delete)
    echo "Suppression du snapshot: ${PVC_NAME}"
    kubectl delete volumesnapshot ${PVC_NAME} -n ${NAMESPACE}
    ;;

  cleanup)
    DAYS=${PVC_NAME:-7}
    echo "Suppression des snapshots de plus de ${DAYS} jours..."

    CUTOFF=$(date -d "${DAYS} days ago" +%s)
    kubectl get volumesnapshot -n ${NAMESPACE} -o json | \
    jq -r ".items[] |
      select(.metadata.creationTimestamp | fromdateiso8601 < ${CUTOFF}) |
      .metadata.name" | \
    while read snap; do
      echo "Suppression: $snap"
      kubectl delete volumesnapshot $snap -n ${NAMESPACE}
    done
    ;;

  *)
    echo "Usage: $0 {create|list|restore|delete|cleanup} PVC_NAME [NAMESPACE] [SNAPSHOT_CLASS]"
    echo ""
    echo "Actions:"
    echo "  create  - Créer un snapshot d'un PVC"
    echo "  list    - Lister les snapshots"
    echo "  restore - Restaurer un PVC depuis un snapshot"
    echo "  delete  - Supprimer un snapshot"
    echo "  cleanup - Supprimer les snapshots de plus de N jours"
    exit 1
    ;;
esac
```

Utilisation :

```bash
# Créer un snapshot
./snapshot-manager.sh create my-pvc production

# Lister les snapshots
./snapshot-manager.sh list production

# Restaurer
./snapshot-manager.sh restore my-pvc-20250115-120000 production

# Nettoyer (snapshots > 7 jours)
./snapshot-manager.sh cleanup 7 production
```

## Conclusion

Les snapshots de volumes sont un outil puissant pour la protection des données dans Kubernetes, mais leur utilisation avec MicroK8s nécessite une réflexion sur vos besoins réels.

### Points clés à retenir

✅ **Snapshots natifs** : Rapides et efficaces, mais nécessitent un stockage compatible
✅ **Restic/Kopia** : Universel et simple, idéal pour MicroK8s avec hostpath
✅ **OpenEBS/Longhorn** : Solutions pour avoir des snapshots natifs sur MicroK8s
✅ **Cohérence** : Utilisez des hooks pour les bases de données
✅ **Automatisation** : CronJobs ou Velero Schedules pour régularité
✅ **Clonage** : Parfait pour créer des environnements de test
✅ **Monitoring** : Surveillez l'état et l'espace disque

### Choix recommandé selon le contexte

**Pour débuter et labs simples** :
```
→ Velero + Restic/Kopia
```

**Pour apprendre les snapshots Kubernetes** :
```
→ OpenEBS + VolumeSnapshots CRDs
```

**Pour une production MicroK8s** :
```
→ Longhorn (multi-node) ou OpenEBS (single-node)
```

### Prochaines étapes

Dans la section suivante (22.5), nous verrons comment mettre en place des tests de restauration complets et élaborer un véritable plan de reprise d'activité (DRP) pour votre cluster MicroK8s.

---

**Checklist snapshots de volumes** :

- [ ] Comprendre la différence snapshots vs backups par fichier
- [ ] Évaluer si le stockage supporte les snapshots natifs
- [ ] Décider : Restic/Kopia ou snapshots natifs ?
- [ ] Installer les composants nécessaires (OpenEBS/CRDs)
- [ ] Créer une VolumeSnapshotClass
- [ ] Tester la création d'un snapshot
- [ ] Tester la restauration depuis un snapshot
- [ ] Automatiser avec CronJobs ou Schedules
- [ ] Implémenter des hooks pour bases de données
- [ ] Mettre en place le monitoring
- [ ] Documenter la procédure

⏭️ [Export/Import de configurations](/22-sauvegarde-et-restauration/05-export-import-de-configurations.md)
