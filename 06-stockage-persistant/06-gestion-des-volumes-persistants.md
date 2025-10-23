🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 6.6 Gestion des volumes persistants

## Introduction

Nous avons maintenant une compréhension solide des concepts fondamentaux du stockage dans Kubernetes : les volumes, les PersistentVolumes (PV), les PersistentVolumeClaims (PVC) et les StorageClasses. Mais comprendre les concepts n'est que la première étape. Dans un environnement réel, vous devez être capable de **gérer** activement ces ressources au quotidien.

Cette section couvre les aspects opérationnels et pratiques de la gestion du stockage persistant :
- Comment surveiller l'utilisation du stockage
- Comment effectuer des sauvegardes et restaurations
- Comment redimensionner les volumes
- Comment migrer des données
- Comment nettoyer et maintenir votre infrastructure de stockage
- Comment diagnostiquer et résoudre les problèmes courants

Que vous gériez un environnement de développement local avec MicroK8s ou un cluster de production, ces compétences sont essentielles.

## Vue d'ensemble de la gestion du stockage

### Le cycle de vie complet

```
┌──────────────────────────────────────────────────────────┐
│                  CYCLE DE VIE DU STOCKAGE                │
├──────────────────────────────────────────────────────────┤
│                                                          │
│  1. PROVISIONNEMENT                                      │
│     ├─ Création de StorageClass                          │
│     ├─ Création de PVC                                   │
│     └─ Provisionnement automatique du PV                 │
│                                                          │
│  2. UTILISATION                                          │
│     ├─ Montage dans les pods                             │
│     ├─ Lecture/écriture des données                      │
│     └─ Monitoring de l'utilisation                       │
│                                                          │
│  3. MAINTENANCE                                          │
│     ├─ Surveillance de la capacité                       │
│     ├─ Expansion si nécessaire                           │
│     ├─ Sauvegardes régulières                            │
│     └─ Vérification de l'intégrité                       │
│                                                          │
│  4. MIGRATION (si nécessaire)                            │
│     ├─ Copie des données                                 │
│     ├─ Changement de classe de stockage                  │
│     └─ Déplacement entre namespaces                      │
│                                                          │
│  5. DÉCOMMISSIONNEMENT                                   │
│     ├─ Sauvegarde finale                                 │
│     ├─ Suppression du PVC                                │
│     ├─ Récupération selon la politique                   │
│     └─ Nettoyage du stockage physique                    │
│                                                          │
└──────────────────────────────────────────────────────────┘
```

### Responsabilités de gestion

**Tâches quotidiennes** :
- Monitoring de l'utilisation de l'espace
- Vérification des alertes
- Réponse aux demandes de stockage

**Tâches hebdomadaires** :
- Revue de l'utilisation globale
- Vérification des sauvegardes
- Nettoyage des ressources inutilisées

**Tâches mensuelles** :
- Analyse des tendances de croissance
- Planification de la capacité
- Audit de sécurité du stockage

**Tâches trimestrielles** :
- Tests de restauration
- Révision des politiques
- Optimisation des coûts

## Monitoring et observation

### Surveiller l'utilisation du stockage

#### 1. État global des PVC

```bash
# Lister tous les PVC
microk8s kubectl get pvc --all-namespaces

# Avec plus de détails
microk8s kubectl get pvc --all-namespaces -o wide

# Format personnalisé
microk8s kubectl get pvc --all-namespaces -o custom-columns=\
NAMESPACE:.metadata.namespace,\
NAME:.metadata.name,\
STATUS:.status.phase,\
CAPACITY:.status.capacity.storage,\
STORAGECLASS:.spec.storageClassName,\
AGE:.metadata.creationTimestamp
```

**Résultat exemple** :
```
NAMESPACE    NAME           STATUS   CAPACITY   STORAGECLASS       AGE
production   postgres-data  Bound    20Gi       microk8s-hostpath  30d
production   redis-cache    Bound    5Gi        microk8s-hostpath  30d
staging      app-data       Bound    10Gi       microk8s-hostpath  15d
```

#### 2. État des PV

```bash
# Lister tous les PV
microk8s kubectl get pv

# Avec détails sur le claim
microk8s kubectl get pv -o custom-columns=\
NAME:.metadata.name,\
CAPACITY:.spec.capacity.storage,\
STATUS:.status.phase,\
CLAIM:.spec.claimRef.name,\
STORAGECLASS:.spec.storageClassName,\
AGE:.metadata.creationTimestamp
```

#### 3. Utilisation réelle de l'espace disque

Pour MicroK8s avec hostpath-storage, vérifier directement sur le nœud :

```bash
# Voir l'utilisation du répertoire de stockage par défaut
du -sh /var/snap/microk8s/common/default-storage/*

# Version détaillée
du -h --max-depth=1 /var/snap/microk8s/common/default-storage/ | sort -hr
```

**Résultat exemple** :
```
15G     /var/snap/microk8s/common/default-storage/production-postgres-data-pvc-abc123
3.2G    /var/snap/microk8s/common/default-storage/production-redis-cache-pvc-def456
1.5G    /var/snap/microk8s/common/default-storage/staging-app-data-pvc-ghi789
```

#### 4. Script de monitoring complet

Créons un script bash pour surveiller l'utilisation :

```bash
#!/bin/bash
# monitoring-storage.sh

echo "==================================="
echo "   RAPPORT STOCKAGE KUBERNETES"
echo "==================================="
echo ""

echo "1. Résumé des PVC par namespace"
echo "--------------------------------"
microk8s kubectl get pvc --all-namespaces --no-headers | \
  awk '{ns[$1]++} END {for (n in ns) print n": "ns[n]" PVC"}'
echo ""

echo "2. Espace total alloué"
echo "----------------------"
total=$(microk8s kubectl get pvc --all-namespaces -o json | \
  jq -r '.items[].spec.resources.requests.storage' | \
  sed 's/Gi//' | awk '{sum+=$1} END {print sum}')
echo "Total: ${total}Gi"
echo ""

echo "3. PVC par état"
echo "---------------"
microk8s kubectl get pvc --all-namespaces --no-headers | \
  awk '{status[$3]++} END {for (s in status) print s": "status[s]}'
echo ""

echo "4. Top 5 des plus gros PVC"
echo "--------------------------"
microk8s kubectl get pvc --all-namespaces -o json | \
  jq -r '.items | sort_by(.spec.resources.requests.storage) | reverse | .[:5] | .[] | "\(.metadata.namespace)/\(.metadata.name): \(.spec.resources.requests.storage)"'
echo ""

echo "5. Utilisation du disque physique (hostpath)"
echo "---------------------------------------------"
df -h /var/snap/microk8s/common/default-storage 2>/dev/null || echo "Chemin non accessible"
echo ""

echo "6. Alertes"
echo "----------"
# PVC en Pending
pending=$(microk8s kubectl get pvc --all-namespaces --no-headers | grep Pending | wc -l)
if [ $pending -gt 0 ]; then
  echo "⚠️  $pending PVC en état Pending"
  microk8s kubectl get pvc --all-namespaces | grep Pending
else
  echo "✅ Aucun PVC en attente"
fi
echo ""
```

**Utilisation** :
```bash
chmod +x monitoring-storage.sh
./monitoring-storage.sh
```

#### 5. Monitoring avec Prometheus (avancé)

Si vous avez Prometheus installé :

```bash
# Activer Prometheus sur MicroK8s
microk8s enable prometheus
```

**Métriques importantes à surveiller** :
- `kubelet_volume_stats_capacity_bytes` : Capacité totale
- `kubelet_volume_stats_used_bytes` : Espace utilisé
- `kubelet_volume_stats_available_bytes` : Espace disponible
- `kubelet_volume_stats_inodes_used` : Inodes utilisés

**Exemple de requête PromQL** :
```promql
# Pourcentage d'utilisation par PVC
(kubelet_volume_stats_used_bytes / kubelet_volume_stats_capacity_bytes) * 100
```

### Alertes et notifications

#### Définir des seuils d'alerte

**Seuils recommandés** :
- ⚠️ Warning : 70% d'utilisation
- 🚨 Critical : 85% d'utilisation
- 🔴 Emergency : 95% d'utilisation

#### Script d'alerte simple

```bash
#!/bin/bash
# alert-storage.sh

THRESHOLD=80  # Pourcentage

# Vérifier l'utilisation du disque
usage=$(df /var/snap/microk8s/common/default-storage | tail -1 | awk '{print $5}' | sed 's/%//')

if [ $usage -gt $THRESHOLD ]; then
  echo "🚨 ALERTE: Stockage à ${usage}% d'utilisation (seuil: ${THRESHOLD}%)"

  # Lister les plus gros volumes
  echo "Plus gros consommateurs:"
  du -sh /var/snap/microk8s/common/default-storage/* | sort -hr | head -5

  # Envoyer une notification (exemple avec mail)
  # echo "Stockage critique: ${usage}%" | mail -s "Alerte Stockage K8s" admin@example.com
else
  echo "✅ Stockage OK: ${usage}% d'utilisation"
fi
```

## Expansion de volumes

### Vérifier si l'expansion est supportée

```bash
# Vérifier la StorageClass
microk8s kubectl get storageclass microk8s-hostpath -o yaml | grep allowVolumeExpansion
```

**Résultat attendu** :
```yaml
allowVolumeExpansion: true
```

### Augmenter la taille d'un PVC

#### Méthode 1 : Via kubectl edit

```bash
# Éditer le PVC
microk8s kubectl edit pvc mon-pvc

# Modifier la ligne storage
# Avant:
#   storage: 10Gi
# Après:
#   storage: 20Gi
```

#### Méthode 2 : Via kubectl patch

```bash
# Augmenter la taille à 20Gi
microk8s kubectl patch pvc mon-pvc -p '{"spec":{"resources":{"requests":{"storage":"20Gi"}}}}'
```

#### Méthode 3 : Via fichier YAML

```yaml
# pvc-expanded.yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: mon-pvc
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 20Gi    # Nouvelle taille
  storageClassName: microk8s-hostpath
```

```bash
microk8s kubectl apply -f pvc-expanded.yaml
```

### Processus d'expansion

```
┌──────────────────────────────────────────────────────────┐
│              PROCESSUS D'EXPANSION DE VOLUME             │
├──────────────────────────────────────────────────────────┤
│                                                          │
│  1. Modification du PVC                                  │
│     └─ Augmentation de spec.resources.requests.storage   │
│                                                          │
│  2. Kubernetes détecte le changement                     │
│     └─ Condition PVCResizing ajoutée                     │
│                                                          │
│  3. Expansion du volume sous-jacent                      │
│     ├─ Provisioner étend le stockage physique            │
│     └─ Peut prendre quelques secondes/minutes            │
│                                                          │
│  4. Expansion du système de fichiers                     │
│     ├─ Automatique pour certains types                   │
│     ├─ Nécessite redémarrage du pod pour d'autres        │
│     └─ Condition FileSystemResizePending si besoin       │
│                                                          │
│  5. Expansion terminée                                   │
│     ├─ PVC reflète la nouvelle taille                    │
│     └─ Condition PVCResizing supprimée                   │
│                                                          │
└──────────────────────────────────────────────────────────┘
```

### Vérifier le statut de l'expansion

```bash
# Voir l'état du PVC
microk8s kubectl get pvc mon-pvc

# Voir les conditions détaillées
microk8s kubectl describe pvc mon-pvc
```

**Rechercher** :
```
Conditions:
  Type                      Status
  ----                      ------
  FileSystemResizePending   True    # Redémarrage du pod nécessaire
  Resizing                  True    # Expansion en cours
```

### Redémarrer le pod si nécessaire

Pour certains types de stockage, le pod doit être redémarré :

```bash
# Supprimer le pod (sera recréé par le Deployment/StatefulSet)
microk8s kubectl delete pod <nom-pod>

# Ou faire un rollout restart
microk8s kubectl rollout restart deployment/<nom-deployment>
```

### Limitations de l'expansion

**Points importants** :
- ❌ On ne peut jamais **réduire** la taille
- ✅ On peut seulement **augmenter**
- ⚠️ Certains systèmes nécessitent un redémarrage du pod
- ⚠️ L'expansion peut prendre du temps selon le type de stockage
- ⚠️ Tous les provisioners ne supportent pas l'expansion

## Sauvegardes et restaurations

### Stratégies de sauvegarde

#### Stratégie 1 : Sauvegarde au niveau du pod

Copier les données depuis un pod vers un emplacement externe :

```bash
# Créer une archive des données
microk8s kubectl exec <pod-name> -- tar czf /tmp/backup.tar.gz /data

# Copier l'archive localement
microk8s kubectl cp <pod-name>:/tmp/backup.tar.gz ./backup-$(date +%Y%m%d).tar.gz

# Nettoyer
microk8s kubectl exec <pod-name> -- rm /tmp/backup.tar.gz
```

#### Stratégie 2 : Sauvegarde au niveau du nœud

Accéder directement au stockage sur le nœud :

```bash
# Pour MicroK8s avec hostpath
# Identifier le répertoire du PVC
PVC_DIR=$(microk8s kubectl get pv <pv-name> -o jsonpath='{.spec.hostPath.path}')

# Créer une archive
sudo tar czf backup-$(date +%Y%m%d).tar.gz -C $PVC_DIR .

# Copier vers un emplacement sûr
sudo mv backup-*.tar.gz /backup/location/
```

#### Stratégie 3 : Snapshot de volume (avancé)

Certains provisioners supportent les snapshots :

```yaml
apiVersion: snapshot.storage.k8s.io/v1
kind: VolumeSnapshot
metadata:
  name: mon-pvc-snapshot
spec:
  volumeSnapshotClassName: csi-snapclass
  source:
    persistentVolumeClaimName: mon-pvc
```

**Note** : Nécessite un driver CSI supportant les snapshots.

### Script de sauvegarde automatisé

```bash
#!/bin/bash
# backup-pvc.sh

# Configuration
NAMESPACE=$1
PVC_NAME=$2
BACKUP_DIR="/backup/k8s-volumes"
DATE=$(date +%Y%m%d-%H%M%S)

if [ -z "$NAMESPACE" ] || [ -z "$PVC_NAME" ]; then
  echo "Usage: $0 <namespace> <pvc-name>"
  exit 1
fi

echo "🔵 Démarrage de la sauvegarde: $NAMESPACE/$PVC_NAME"

# Trouver un pod utilisant ce PVC
POD=$(microk8s kubectl get pods -n $NAMESPACE -o json | \
  jq -r ".items[] | select(.spec.volumes[]?.persistentVolumeClaim.claimName==\"$PVC_NAME\") | .metadata.name" | head -1)

if [ -z "$POD" ]; then
  echo "❌ Aucun pod trouvé utilisant ce PVC"
  exit 1
fi

echo "📦 Pod trouvé: $POD"

# Trouver le point de montage
MOUNT_PATH=$(microk8s kubectl get pod $POD -n $NAMESPACE -o json | \
  jq -r ".spec.containers[0].volumeMounts[] | select(.name==(\"$PVC_NAME\" | split(\"-\") | .[0])) | .mountPath")

if [ -z "$MOUNT_PATH" ]; then
  MOUNT_PATH="/data"  # Défaut
fi

echo "📂 Chemin de montage: $MOUNT_PATH"

# Créer le répertoire de backup
mkdir -p "$BACKUP_DIR"

# Créer l'archive
BACKUP_FILE="$BACKUP_DIR/${NAMESPACE}-${PVC_NAME}-${DATE}.tar.gz"
echo "💾 Création de l'archive..."

microk8s kubectl exec $POD -n $NAMESPACE -- tar czf - $MOUNT_PATH | cat > $BACKUP_FILE

if [ $? -eq 0 ]; then
  SIZE=$(du -h $BACKUP_FILE | cut -f1)
  echo "✅ Sauvegarde terminée: $BACKUP_FILE ($SIZE)"
else
  echo "❌ Erreur lors de la sauvegarde"
  exit 1
fi

# Afficher les 5 dernières sauvegardes
echo ""
echo "📋 Dernières sauvegardes de ${NAMESPACE}/${PVC_NAME}:"
ls -lht "$BACKUP_DIR/${NAMESPACE}-${PVC_NAME}"* 2>/dev/null | head -5
```

**Utilisation** :
```bash
chmod +x backup-pvc.sh
./backup-pvc.sh production postgres-data
```

### Restauration depuis une sauvegarde

#### Méthode 1 : Restauration dans un pod existant

```bash
# 1. Arrêter l'application (si nécessaire)
microk8s kubectl scale deployment/mon-app --replicas=0

# 2. Copier l'archive dans le pod temporaire
cat backup-20240101.tar.gz | microk8s kubectl exec -i temp-pod -- tar xzf - -C /data

# 3. Redémarrer l'application
microk8s kubectl scale deployment/mon-app --replicas=1
```

#### Méthode 2 : Restauration au niveau du nœud

```bash
# 1. Identifier le répertoire PV
PVC_DIR=$(microk8s kubectl get pv <pv-name> -o jsonpath='{.spec.hostPath.path}')

# 2. Vider le répertoire actuel (ATTENTION !)
sudo rm -rf $PVC_DIR/*

# 3. Extraire la sauvegarde
sudo tar xzf backup-20240101.tar.gz -C $PVC_DIR

# 4. Vérifier les permissions
sudo chown -R 1000:1000 $PVC_DIR  # Ajuster selon les besoins
```

#### Méthode 3 : Restauration dans un nouveau PVC

```yaml
# 1. Créer un nouveau PVC
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: restored-pvc
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 20Gi
  storageClassName: microk8s-hostpath
---
# 2. Pod temporaire pour restaurer
apiVersion: v1
kind: Pod
metadata:
  name: restore-pod
spec:
  containers:
  - name: restore
    image: busybox:1.35
    command: ['sleep', '3600']
    volumeMounts:
    - name: data
      mountPath: /data
  volumes:
  - name: data
    persistentVolumeClaim:
      claimName: restored-pvc
```

```bash
# 3. Appliquer
microk8s kubectl apply -f restore-pod.yaml

# 4. Copier l'archive dans le pod
microk8s kubectl cp backup-20240101.tar.gz restore-pod:/tmp/

# 5. Extraire dans le volume
microk8s kubectl exec restore-pod -- tar xzf /tmp/backup-20240101.tar.gz -C /data

# 6. Vérifier
microk8s kubectl exec restore-pod -- ls -la /data

# 7. Nettoyer le pod temporaire
microk8s kubectl delete pod restore-pod
```

### Planification automatique des sauvegardes

Utiliser un CronJob Kubernetes :

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: backup-postgres
spec:
  schedule: "0 2 * * *"    # Tous les jours à 2h du matin
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: backup
            image: busybox:1.35
            command:
            - /bin/sh
            - -c
            - |
              echo "Backup starting at $(date)"
              tar czf /backup/postgres-$(date +%Y%m%d).tar.gz -C /data .
              echo "Backup completed"
              # Nettoyer les backups > 7 jours
              find /backup -name "postgres-*.tar.gz" -mtime +7 -delete
            volumeMounts:
            - name: data
              mountPath: /data
              readOnly: true
            - name: backup
              mountPath: /backup
          restartPolicy: OnFailure
          volumes:
          - name: data
            persistentVolumeClaim:
              claimName: postgres-data
          - name: backup
            hostPath:
              path: /backup/kubernetes
              type: DirectoryOrCreate
```

## Migration de données

### Cas d'usage de migration

**Scénarios courants** :
1. Changement de StorageClass (ex: standard → fast)
2. Migration entre namespaces
3. Redimensionnement non supporté par expansion
4. Changement de type de stockage (hostPath → NFS)
5. Migration entre clusters

### Migration basique : Nouveau PVC dans le même namespace

#### Étape 1 : Créer le nouveau PVC

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: new-pvc
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 30Gi    # Nouvelle taille
  storageClassName: fast    # Nouvelle classe
```

```bash
microk8s kubectl apply -f new-pvc.yaml
```

#### Étape 2 : Pod de migration

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: migration-pod
spec:
  containers:
  - name: migrator
    image: busybox:1.35
    command: ['sleep', '3600']
    volumeMounts:
    - name: source
      mountPath: /source
      readOnly: true
    - name: destination
      mountPath: /destination
  volumes:
  - name: source
    persistentVolumeClaim:
      claimName: old-pvc
  - name: destination
    persistentVolumeClaim:
      claimName: new-pvc
```

```bash
microk8s kubectl apply -f migration-pod.yaml
```

#### Étape 3 : Copier les données

```bash
# Copier avec tar (préserve les permissions)
microk8s kubectl exec migration-pod -- sh -c 'tar cf - -C /source . | tar xf - -C /destination'

# Ou avec cp (plus simple mais peut perdre certains attributs)
microk8s kubectl exec migration-pod -- cp -av /source/. /destination/

# Vérifier
microk8s kubectl exec migration-pod -- du -sh /source
microk8s kubectl exec migration-pod -- du -sh /destination
```

#### Étape 4 : Mettre à jour l'application

```bash
# Arrêter l'ancienne application
microk8s kubectl scale deployment/mon-app --replicas=0

# Modifier le deployment pour utiliser le nouveau PVC
microk8s kubectl patch deployment mon-app -p '
{
  "spec": {
    "template": {
      "spec": {
        "volumes": [{
          "name": "data",
          "persistentVolumeClaim": {
            "claimName": "new-pvc"
          }
        }]
      }
    }
  }
}'

# Redémarrer
microk8s kubectl scale deployment/mon-app --replicas=1

# Vérifier
microk8s kubectl get pods
microk8s kubectl logs -f deployment/mon-app
```

#### Étape 5 : Nettoyer

```bash
# Supprimer le pod de migration
microk8s kubectl delete pod migration-pod

# Après vérification que tout fonctionne, supprimer l'ancien PVC
microk8s kubectl delete pvc old-pvc
```

### Migration entre namespaces

```bash
#!/bin/bash
# migrate-pvc-namespace.sh

SOURCE_NS="staging"
DEST_NS="production"
PVC_NAME="app-data"

echo "Migration: $SOURCE_NS/$PVC_NAME → $DEST_NS/$PVC_NAME"

# 1. Créer le namespace destination si nécessaire
microk8s kubectl create namespace $DEST_NS --dry-run=client -o yaml | microk8s kubectl apply -f -

# 2. Exporter la définition du PVC
microk8s kubectl get pvc $PVC_NAME -n $SOURCE_NS -o yaml | \
  sed "s/namespace: $SOURCE_NS/namespace: $DEST_NS/" | \
  sed '/uid:/d' | sed '/resourceVersion:/d' | sed '/selfLink:/d' > temp-pvc.yaml

# 3. Créer le nouveau PVC
microk8s kubectl apply -f temp-pvc.yaml

# 4. Créer un pod de migration
cat <<EOF | microk8s kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: migration-pod
  namespace: $SOURCE_NS
spec:
  containers:
  - name: migrator
    image: busybox:1.35
    command: ['sleep', '3600']
    volumeMounts:
    - name: source
      mountPath: /source
  volumes:
  - name: source
    persistentVolumeClaim:
      claimName: $PVC_NAME
EOF

# Attendre que le pod soit prêt
microk8s kubectl wait --for=condition=ready pod/migration-pod -n $SOURCE_NS --timeout=60s

# 5. Créer une archive
microk8s kubectl exec migration-pod -n $SOURCE_NS -- tar czf /tmp/data.tar.gz -C /source .

# 6. Copier localement
microk8s kubectl cp $SOURCE_NS/migration-pod:/tmp/data.tar.gz ./migration-data.tar.gz

# 7. Créer un pod dans le namespace destination
cat <<EOF | microk8s kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: restore-pod
  namespace: $DEST_NS
spec:
  containers:
  - name: restore
    image: busybox:1.35
    command: ['sleep', '3600']
    volumeMounts:
    - name: dest
      mountPath: /dest
  volumes:
  - name: dest
    persistentVolumeClaim:
      claimName: $PVC_NAME
EOF

# Attendre
microk8s kubectl wait --for=condition=ready pod/restore-pod -n $DEST_NS --timeout=60s

# 8. Copier et extraire
microk8s kubectl cp ./migration-data.tar.gz $DEST_NS/restore-pod:/tmp/data.tar.gz
microk8s kubectl exec restore-pod -n $DEST_NS -- tar xzf /tmp/data.tar.gz -C /dest

# 9. Vérifier
echo "Source:"
microk8s kubectl exec migration-pod -n $SOURCE_NS -- du -sh /source
echo "Destination:"
microk8s kubectl exec restore-pod -n $DEST_NS -- du -sh /dest

# 10. Nettoyer
microk8s kubectl delete pod migration-pod -n $SOURCE_NS
microk8s kubectl delete pod restore-pod -n $DEST_NS
rm -f migration-data.tar.gz temp-pvc.yaml

echo "✅ Migration terminée"
```

## Nettoyage et maintenance

### Identifier les ressources inutilisées

#### PVC non utilisés par aucun pod

```bash
#!/bin/bash
# find-unused-pvcs.sh

echo "PVC non utilisés par aucun pod:"
echo "================================"

for ns in $(microk8s kubectl get ns -o jsonpath='{.items[*].metadata.name}'); do
  pvcs=$(microk8s kubectl get pvc -n $ns -o jsonpath='{.items[*].metadata.name}')

  for pvc in $pvcs; do
    # Chercher des pods utilisant ce PVC
    used=$(microk8s kubectl get pods -n $ns -o json | \
      jq -r ".items[] | select(.spec.volumes[]?.persistentVolumeClaim.claimName==\"$pvc\") | .metadata.name" | wc -l)

    if [ $used -eq 0 ]; then
      size=$(microk8s kubectl get pvc $pvc -n $ns -o jsonpath='{.spec.resources.requests.storage}')
      age=$(microk8s kubectl get pvc $pvc -n $ns -o jsonpath='{.metadata.creationTimestamp}')
      echo "  $ns/$pvc ($size) - Créé: $age"
    fi
  done
done
```

#### PV en statut Released

```bash
# Lister les PV Released
microk8s kubectl get pv | grep Released

# Avec détails
microk8s kubectl get pv -o json | \
  jq -r '.items[] | select(.status.phase=="Released") | "\(.metadata.name) - \(.spec.capacity.storage)"'
```

### Nettoyer les PV Released

```bash
#!/bin/bash
# cleanup-released-pv.sh

echo "Nettoyage des PV en statut Released"
echo "===================================="

# Lister les PV Released
released_pvs=$(microk8s kubectl get pv -o json | \
  jq -r '.items[] | select(.status.phase=="Released") | .metadata.name')

if [ -z "$released_pvs" ]; then
  echo "✅ Aucun PV à nettoyer"
  exit 0
fi

echo "PV à nettoyer:"
echo "$released_pvs"
echo ""

read -p "Voulez-vous continuer? (oui/non) " -r
if [[ ! $REPLY =~ ^[Oo]ui$ ]]; then
  echo "Annulé"
  exit 0
fi

for pv in $released_pvs; do
  echo "Traitement de $pv..."

  # Récupérer le chemin hostPath (si applicable)
  path=$(microk8s kubectl get pv $pv -o jsonpath='{.spec.hostPath.path}')

  if [ ! -z "$path" ]; then
    echo "  Chemin: $path"

    # Sauvegarder si nécessaire
    read -p "  Sauvegarder les données? (oui/non) " -r
    if [[ $REPLY =~ ^[Oo]ui$ ]]; then
      backup_file="backup-$pv-$(date +%Y%m%d).tar.gz"
      sudo tar czf $backup_file -C $path .
      echo "  ✅ Sauvegarde: $backup_file"
    fi

    # Nettoyer le répertoire
    read -p "  Supprimer les données? (oui/non) " -r
    if [[ $REPLY =~ ^[Oo]ui$ ]]; then
      sudo rm -rf $path/*
      echo "  ✅ Données supprimées"
    fi
  fi

  # Supprimer le PV
  microk8s kubectl delete pv $pv
  echo "  ✅ PV supprimé"
  echo ""
done

echo "✅ Nettoyage terminé"
```

### Optimisation de l'espace disque

#### Supprimer les fichiers temporaires dans les volumes

```bash
#!/bin/bash
# cleanup-temp-files.sh

NAMESPACE=$1
PVC_NAME=$2

if [ -z "$NAMESPACE" ] || [ -z "$PVC_NAME" ]; then
  echo "Usage: $0 <namespace> <pvc-name>"
  exit 1
fi

# Trouver un pod utilisant le PVC
POD=$(microk8s kubectl get pods -n $NAMESPACE -o json | \
  jq -r ".items[] | select(.spec.volumes[]?.persistentVolumeClaim.claimName==\"$PVC_NAME\") | .metadata.name" | head -1)

if [ -z "$POD" ]; then
  echo "❌ Aucun pod trouvé"
  exit 1
fi

echo "Nettoyage des fichiers temporaires dans $NAMESPACE/$PVC_NAME"

# Supprimer les fichiers temporaires courants
microk8s kubectl exec $POD -n $NAMESPACE -- find /data -type f \( -name "*.tmp" -o -name "*.temp" -o -name "*.log.old" \) -delete

# Supprimer les vieux logs (>30 jours)
microk8s kubectl exec $POD -n $NAMESPACE -- find /data -type f -name "*.log" -mtime +30 -delete

echo "✅ Nettoyage terminé"
```

### Rotation et compression des logs

Si vos applications génèrent beaucoup de logs dans les volumes :

```bash
#!/bin/bash
# compress-old-logs.sh

NAMESPACE=$1
PVC_NAME=$2
LOG_DIR="/data/logs"

POD=$(microk8s kubectl get pods -n $NAMESPACE -o json | \
  jq -r ".items[] | select(.spec.volumes[]?.persistentVolumeClaim.claimName==\"$PVC_NAME\") | .metadata.name" | head -1)

# Compresser les logs > 7 jours
microk8s kubectl exec $POD -n $NAMESPACE -- \
  find $LOG_DIR -type f -name "*.log" -mtime +7 ! -name "*.gz" -exec gzip {} \;

# Supprimer les logs compressés > 90 jours
microk8s kubectl exec $POD -n $NAMESPACE -- \
  find $LOG_DIR -type f -name "*.log.gz" -mtime +90 -delete

echo "✅ Rotation des logs terminée"
```

## Diagnostic et dépannage

### Vérifier l'intégrité des données

#### Test de lecture/écriture

```bash
# Créer un pod de test
cat <<EOF | microk8s kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: volume-test
spec:
  containers:
  - name: test
    image: busybox:1.35
    command: ['sleep', '3600']
    volumeMounts:
    - name: test-vol
      mountPath: /test
  volumes:
  - name: test-vol
    persistentVolumeClaim:
      claimName: mon-pvc
EOF

# Test d'écriture
microk8s kubectl exec volume-test -- sh -c 'echo "test" > /test/testfile.txt'

# Test de lecture
microk8s kubectl exec volume-test -- cat /test/testfile.txt

# Vérifier les permissions
microk8s kubectl exec volume-test -- ls -la /test

# Test de performance (écriture)
microk8s kubectl exec volume-test -- dd if=/dev/zero of=/test/testfile bs=1M count=100

# Nettoyer
microk8s kubectl exec volume-test -- rm /test/testfile*
microk8s kubectl delete pod volume-test
```

### Problèmes courants et solutions

#### Problème 1 : "No space left on device"

**Symptômes** :
```
Error: no space left on device
```

**Diagnostic** :
```bash
# Vérifier l'utilisation du disque
df -h /var/snap/microk8s/common/default-storage

# Identifier les gros fichiers
du -sh /var/snap/microk8s/common/default-storage/* | sort -hr | head -10
```

**Solutions** :
1. Nettoyer les fichiers inutiles
2. Étendre le PVC si possible
3. Ajouter de l'espace disque au nœud
4. Migrer vers un volume plus grand

#### Problème 2 : "Permission denied"

**Symptômes** :
```
Error: permission denied writing to /data
```

**Diagnostic** :
```bash
# Vérifier les permissions dans le volume
microk8s kubectl exec <pod> -- ls -la /data
```

**Solutions** :
```bash
# Option 1: Ajuster les permissions via un init container
apiVersion: v1
kind: Pod
metadata:
  name: mon-pod
spec:
  initContainers:
  - name: fix-permissions
    image: busybox:1.35
    command: ['sh', '-c', 'chmod -R 777 /data']
    volumeMounts:
    - name: data
      mountPath: /data
  containers:
  - name: app
    image: mon-app
    volumeMounts:
    - name: data
      mountPath: /data
  volumes:
  - name: data
    persistentVolumeClaim:
      claimName: mon-pvc
```

```bash
# Option 2: Ajuster directement sur le nœud
sudo chown -R 1000:1000 /var/snap/microk8s/common/default-storage/<pv-dir>
```

#### Problème 3 : PVC bloqué en "Terminating"

**Symptômes** :
```bash
$ microk8s kubectl delete pvc mon-pvc
# La commande ne se termine pas
$ microk8s kubectl get pvc
NAME      STATUS        VOLUME   CAPACITY   ACCESS MODES
mon-pvc   Terminating   pvc-xxx  10Gi       RWO
```

**Diagnostic** :
```bash
# Trouver les pods utilisant le PVC
microk8s kubectl get pods -o json | \
  jq -r '.items[] | select(.spec.volumes[]?.persistentVolumeClaim.claimName=="mon-pvc") | .metadata.name'
```

**Solutions** :
```bash
# 1. Supprimer les pods utilisant le PVC
microk8s kubectl delete pod <pod-utilisant-pvc>

# 2. Si ça ne suffit pas, supprimer les finalizers
microk8s kubectl patch pvc mon-pvc -p '{"metadata":{"finalizers":null}}'
```

#### Problème 4 : Corruption de données

**Symptômes** :
- Erreurs de lecture incohérentes
- Application qui crash de manière aléatoire
- Fichiers corrompus

**Diagnostic** :
```bash
# Vérifier le système de fichiers (nécessite d'arrêter le pod)
# Pour hostPath sur le nœud
sudo fsck /dev/<device>

# Vérifier les logs système
sudo journalctl -u snap.microk8s.daemon-kubelite | grep -i error
```

**Solutions** :
1. Restaurer depuis une sauvegarde
2. Recréer le PVC avec de nouvelles données
3. Vérifier le hardware sous-jacent

## Automatisation avec des outils

### Utiliser kubectl plugins

#### krew : Gestionnaire de plugins kubectl

```bash
# Installation de krew
(
  set -x; cd "$(mktemp -d)" &&
  OS="$(uname | tr '[:upper:]' '[:lower:]')" &&
  ARCH="$(uname -m | sed -e 's/x86_64/amd64/' -e 's/\(arm\)\(64\)\?.*/\1\2/' -e 's/aarch64$/arm64/')" &&
  KREW="krew-${OS}_${ARCH}" &&
  curl -fsSLO "https://github.com/kubernetes-sigs/krew/releases/latest/download/${KREW}.tar.gz" &&
  tar zxvf "${KREW}.tar.gz" &&
  ./"${KREW}" install krew
)

# Ajouter au PATH
export PATH="${KREW_ROOT:-$HOME/.krew}/bin:$PATH"
```

#### Plugins utiles pour le stockage

```bash
# df-pv : Voir l'utilisation des PV
kubectl krew install df-pv
kubectl df-pv

# view-utilization : Voir l'utilisation des ressources
kubectl krew install view-utilization
kubectl view-utilization

# resource-capacity : Capacité par nœud
kubectl krew install resource-capacity
kubectl resource-capacity
```

### Scripts de maintenance automatisés

#### Script quotidien

```bash
#!/bin/bash
# daily-storage-maintenance.sh

LOG_FILE="/var/log/k8s-storage-maintenance.log"

log() {
  echo "[$(date '+%Y-%m-%d %H:%M:%S')] $1" | tee -a $LOG_FILE
}

log "========================================="
log "Début de la maintenance quotidienne"
log "========================================="

# 1. Vérifier l'espace disque
log "Vérification de l'espace disque..."
usage=$(df /var/snap/microk8s/common/default-storage | tail -1 | awk '{print $5}' | sed 's/%//')
log "Utilisation: ${usage}%"

if [ $usage -gt 85 ]; then
  log "⚠️  ALERTE: Espace disque critique"
  # Envoyer une notification
fi

# 2. Lister les PVC non utilisés
log "Recherche de PVC non utilisés..."
# (utiliser le script find-unused-pvcs.sh)

# 3. Nettoyer les PV Released
log "Nettoyage des PV Released..."
released=$(microk8s kubectl get pv | grep Released | wc -l)
log "PV Released trouvés: $released"

# 4. Vérifier les PVC en Pending
log "Vérification des PVC en attente..."
pending=$(microk8s kubectl get pvc --all-namespaces | grep Pending | wc -l)
if [ $pending -gt 0 ]; then
  log "⚠️  $pending PVC en état Pending"
fi

# 5. Rapport
log "Résumé:"
total_pvc=$(microk8s kubectl get pvc --all-namespaces --no-headers | wc -l)
total_pv=$(microk8s kubectl get pv --no-headers | wc -l)
log "  - Total PVC: $total_pvc"
log "  - Total PV: $total_pv"

log "========================================="
log "Fin de la maintenance quotidienne"
log "========================================="
```

## Bonnes pratiques de gestion

### 1. Documentation

Maintenez une documentation à jour :

```yaml
# Exemple d'annotation complète
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: postgres-data
  annotations:
    description: "Base de données PostgreSQL principale"
    owner: "team-backend@example.com"
    created-by: "John Doe"
    purpose: "Production database"
    backup-schedule: "daily at 2AM"
    retention-policy: "90 days"
    cost-center: "engineering"
    disaster-recovery: "RPO: 24h, RTO: 1h"
```

### 2. Naming conventions

Utilisez des conventions de nommage cohérentes :

```
<application>-<component>-<environment>-<type>

Exemples:
- webapp-uploads-prod-data
- postgres-main-prod-data
- redis-cache-staging-data
```

### 3. Labels standardisés

```yaml
metadata:
  labels:
    app: webapp
    component: database
    environment: production
    tier: data
    backup: "enabled"
    criticality: "high"
```

### 4. Monitoring proactif

- Configurer des alertes avant d'atteindre la capacité
- Surveiller les tendances de croissance
- Automatiser les rapports hebdomadaires

### 5. Sauvegardes régulières

- Sauvegardes quotidiennes des données critiques
- Sauvegardes hebdomadaires des autres données
- Tests de restauration mensuels

### 6. Planification de la capacité

```bash
#!/bin/bash
# capacity-planning.sh

# Calculer la croissance moyenne par semaine
echo "Analyse de la capacité de stockage"
echo "==================================="

# Total actuel
current_total=$(microk8s kubectl get pvc --all-namespaces -o json | \
  jq -r '.items[].spec.resources.requests.storage' | \
  sed 's/Gi//' | awk '{sum+=$1} END {print sum}')

echo "Capacité actuelle: ${current_total}Gi"

# Estimer la croissance (exemple simplifié)
# Dans un vrai environnement, analyser les données historiques
growth_rate=10  # 10% par mois
months=6

for i in $(seq 1 $months); do
  current_total=$(echo "$current_total * 1.10" | bc)
  printf "Estimation mois +$i: %.0fGi\n" $current_total
done
```

### 7. Politique de rétention

Définir des politiques claires :

```yaml
# Politique de rétention par environnement
apiVersion: v1
kind: ConfigMap
metadata:
  name: storage-retention-policy
data:
  production: |
    - Sauvegardes quotidiennes: 30 jours
    - Sauvegardes hebdomadaires: 90 jours
    - Sauvegardes mensuelles: 1 an
    - Sauvegardes annuelles: 7 ans

  staging: |
    - Sauvegardes quotidiennes: 7 jours
    - Sauvegardes hebdomadaires: 30 jours

  development: |
    - Sauvegardes hebdomadaires: 7 jours
    - Pas de rétention à long terme
```

## Checklist de gestion

### Quotidienne
- [ ] Vérifier les alertes de capacité
- [ ] Vérifier les PVC en Pending
- [ ] Vérifier les échecs de backup
- [ ] Répondre aux demandes de stockage

### Hebdomadaire
- [ ] Exécuter le rapport de monitoring
- [ ] Nettoyer les ressources inutilisées
- [ ] Vérifier l'intégrité des sauvegardes
- [ ] Réviser l'utilisation par namespace

### Mensuelle
- [ ] Test de restauration
- [ ] Analyse des tendances de croissance
- [ ] Révision des politiques de rétention
- [ ] Nettoyage des vieilles sauvegardes
- [ ] Audit de sécurité

### Trimestrielle
- [ ] Planification de la capacité
- [ ] Révision des StorageClasses
- [ ] Optimisation des coûts
- [ ] Formation de l'équipe

## Résumé des commandes essentielles

```bash
# Monitoring
microk8s kubectl get pvc --all-namespaces
microk8s kubectl get pv
microk8s kubectl describe pvc <name>
df -h /var/snap/microk8s/common/default-storage

# Expansion
microk8s kubectl patch pvc <name> -p '{"spec":{"resources":{"requests":{"storage":"20Gi"}}}}'

# Sauvegarde
microk8s kubectl exec <pod> -- tar czf - /data > backup.tar.gz

# Restauration
cat backup.tar.gz | microk8s kubectl exec -i <pod> -- tar xzf - -C /data

# Nettoyage
microk8s kubectl delete pvc <name>
microk8s kubectl delete pv <name>

# Diagnostic
microk8s kubectl get events --field-selector involvedObject.name=<pvc>
microk8s kubectl logs <pod>
```

## Conclusion

La gestion efficace des volumes persistants est un aspect critique de l'administration Kubernetes. Elle nécessite :

- **Vigilance** : Monitoring constant de l'utilisation et des performances
- **Proactivité** : Anticipation des besoins en capacité
- **Rigueur** : Sauvegardes régulières et testées
- **Organisation** : Documentation claire et conventions cohérentes
- **Automatisation** : Scripts et outils pour réduire le travail manuel

Avec MicroK8s et les pratiques décrites dans cette section, vous disposez maintenant de tous les outils nécessaires pour gérer professionnellement votre infrastructure de stockage, que ce soit pour un lab personnel ou un environnement de production.

---

**Points clés à retenir** :

✅ **Monitoring** : Surveillez proactivement l'utilisation de l'espace
✅ **Sauvegardes** : Automatisez et testez régulièrement les restaurations
✅ **Expansion** : Augmentez les volumes avant d'atteindre la capacité maximale
✅ **Migration** : Sachez déplacer les données entre PVC/namespaces/clusters
✅ **Nettoyage** : Identifiez et supprimez régulièrement les ressources inutilisées
✅ **Documentation** : Annotez et labelisez toutes vos ressources
✅ **Automatisation** : Utilisez des scripts pour les tâches répétitives
✅ **Planification** : Anticipez la croissance future du stockage

⏭️ [StatefulSets : déployer des applications stateful](/06-stockage-persistant/07-statefulsets-deployer-des-applications-stateful.md)
