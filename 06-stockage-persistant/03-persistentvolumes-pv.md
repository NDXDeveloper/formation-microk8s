🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 6.3 PersistentVolumes (PV)

## Introduction

Nous avons vu dans les sections précédentes que les volumes simples (comme `emptyDir` et `hostPath`) ont des limitations importantes : ils sont soit éphémères, soit liés à un nœud spécifique. Pour des applications nécessitant du stockage vraiment persistant et portable, Kubernetes introduit le concept de **PersistentVolume** (PV).

Un PersistentVolume est une abstraction qui représente une ressource de stockage réelle dans votre cluster, mais gérée de manière indépendante des pods qui l'utilisent. C'est un peu comme réserver un espace de stockage qui existera indépendamment de vos applications.

## Qu'est-ce qu'un PersistentVolume ?

### Définition

Un **PersistentVolume (PV)** est une ressource de stockage dans le cluster Kubernetes qui :
- A été provisionné par un administrateur **ou** créé dynamiquement
- Existe indépendamment de tout pod
- A une durée de vie propre, distincte des applications
- Représente du stockage physique réel (disque local, NFS, stockage cloud, etc.)

**Analogie** : Si vous pensez à un data center, un PV est comme un espace de stockage physique (un disque dur, un volume réseau) qui est disponible et prêt à être utilisé. Il existe qu'on l'utilise ou non.

### Pourquoi les PersistentVolumes sont-ils nécessaires ?

Les volumes standards que nous avons vus précédemment ont des limitations :

**emptyDir** :
- ❌ Disparaît avec le pod
- ❌ Ne peut pas être partagé entre pods
- ❌ Pas adapté pour des données importantes

**hostPath** :
- ❌ Lié à un nœud spécifique
- ❌ Non portable
- ❌ Problématique en multi-nœuds
- ❌ Risques de sécurité

**PersistentVolume** :
- ✅ Survit aux pods
- ✅ Portable entre nœuds
- ✅ Géré centralement
- ✅ Peut être partagé (selon le type)
- ✅ Séparation des responsabilités

## Architecture : Comment ça fonctionne ?

### Le modèle PV/PVC

Kubernetes utilise un modèle en deux parties pour le stockage persistant :

```
┌─────────────────────────────────────────────────────────┐
│                    ADMINISTRATEUR                       │
│                          │                              │
│                          ▼                              │
│              ┌─────────────────────┐                    │
│              │  PersistentVolume   │                    │
│              │   (Ressource PV)    │                    │
│              │                     │                    │
│              │ - Capacité: 10Gi    │                    │
│              │ - Type: NFS         │                    │
│              │ - Access: RWX       │                    │
│              └──────────┬──────────┘                    │
│                         │                               │
│                         │ Liaison (Binding)             │
│                         │                               │
│              ┌──────────▼──────────┐                    │
│              │PersistentVolumeClaim│                    │
│              │    (Demande PVC)    │                    │
│              │                     │                    │
│              │ - Besoin: 5Gi       │                    │
│              │ - Access: RWX       │                    │
│              └──────────┬──────────┘                    │
│                         │                               │
│                         ▼                               │
│                    ┌─────────┐                          │
│                    │   Pod   │                          │
│                    └─────────┘                          │
│                   UTILISATEUR                           │
└─────────────────────────────────────────────────────────┘
```

**Séparation des rôles** :
- **L'administrateur** crée et gère les PersistentVolumes (l'offre)
- **L'utilisateur** crée des PersistentVolumeClaims (la demande)
- **Kubernetes** fait le lien entre les deux

### Cycle de vie d'un PersistentVolume

Un PV passe par plusieurs états durant sa vie :

1. **Provisioning** (Provisionnement)
   - Le PV est créé (manuellement ou dynamiquement)

2. **Available** (Disponible)
   - Le PV existe et n'est lié à aucun PVC
   - Prêt à être utilisé

3. **Bound** (Lié)
   - Un PVC a été associé au PV
   - Le PV est maintenant réservé

4. **Released** (Libéré)
   - Le PVC a été supprimé
   - Le PV n'est plus utilisé mais contient encore des données

5. **Failed** (Échoué)
   - Une erreur s'est produite lors de la récupération

## Anatomie d'un PersistentVolume

Voyons la structure d'un manifeste PV :

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: mon-pv
  labels:
    type: local
spec:
  capacity:
    storage: 10Gi
  volumeMode: Filesystem
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  storageClassName: manual
  hostPath:
    path: /mnt/data
```

Décomposons chaque partie :

### metadata : Identification du PV

```yaml
metadata:
  name: mon-pv          # Nom unique du PV
  labels:               # Labels pour organiser et sélectionner
    type: local
    environment: dev
```

**Points importants** :
- Le nom doit être unique dans le cluster
- Les labels sont optionnels mais très utiles pour l'organisation
- Pas de `namespace` : les PV sont des ressources au niveau du cluster

### capacity : Capacité de stockage

```yaml
spec:
  capacity:
    storage: 10Gi       # Taille du volume
```

**Unités acceptées** :
- `Ki`, `Mi`, `Gi`, `Ti` (kilo, mega, giga, tera - base 1024)
- `K`, `M`, `G`, `T` (base 1000, moins courant)

**Exemple** :
- `1Gi` = 1024 Mi = 1 073 741 824 bytes
- `1G` = 1000 M = 1 000 000 000 bytes

### volumeMode : Mode de volume

```yaml
spec:
  volumeMode: Filesystem    # ou Block
```

**Deux modes possibles** :

**Filesystem** (par défaut) :
- Le volume est monté comme un système de fichiers
- Le plus courant et le plus simple
- Vous pouvez créer des fichiers et des répertoires
- Utilisé par la majorité des applications

**Block** :
- Le volume est exposé comme un périphérique bloc brut
- Utilisé par des applications ayant besoin d'un accès direct au disque
- Plus rare, réservé aux cas spécialisés (certaines bases de données)

### accessModes : Modes d'accès

```yaml
spec:
  accessModes:
    - ReadWriteOnce
```

Les modes d'accès définissent comment le volume peut être monté :

#### ReadWriteOnce (RWO)
- **Le plus courant**
- Le volume peut être monté en lecture-écriture par **un seul nœud**
- Plusieurs pods sur le même nœud peuvent l'utiliser
- Parfait pour : bases de données, applications avec un seul pod

**Exemple** :
```yaml
accessModes:
  - ReadWriteOnce
```

#### ReadOnlyMany (ROX)
- Le volume peut être monté en lecture seule par **plusieurs nœuds**
- Idéal pour partager des données statiques
- Parfait pour : assets statiques, configuration en lecture seule

**Exemple** :
```yaml
accessModes:
  - ReadOnlyMany
```

#### ReadWriteMany (RWX)
- Le volume peut être monté en lecture-écriture par **plusieurs nœuds**
- Nécessite un système de fichiers distribué (NFS, CephFS, etc.)
- Plus complexe à configurer
- Parfait pour : applications nécessitant un partage d'écriture

**Exemple** :
```yaml
accessModes:
  - ReadWriteMany
```

#### ReadWriteOncePod (RWOP)
- Nouveau mode introduit récemment
- Le volume peut être monté par **un seul pod** uniquement
- Plus restrictif que RWO
- Garantit qu'un seul pod peut utiliser le volume

**Tableau récapitulatif** :

| Mode | Lecture | Écriture | Nœuds | Pods |
|------|---------|----------|-------|------|
| RWO | ✅ | ✅ | 1 | Plusieurs (même nœud) |
| ROX | ✅ | ❌ | Plusieurs | Plusieurs |
| RWX | ✅ | ✅ | Plusieurs | Plusieurs |
| RWOP | ✅ | ✅ | 1 | 1 |

**Important** : Tous les types de stockage ne supportent pas tous les modes. Par exemple :
- hostPath : RWO uniquement
- NFS : RWO, ROX, RWX
- Stockage cloud : varie selon le fournisseur

### persistentVolumeReclaimPolicy : Politique de récupération

```yaml
spec:
  persistentVolumeReclaimPolicy: Retain
```

Que se passe-t-il quand le PVC lié au PV est supprimé ? C'est défini par la politique de récupération :

#### Retain (Conserver)
- **Comportement** : Le PV passe en statut "Released" mais garde les données
- **Données** : Préservées
- **État** : Le PV n'est plus disponible pour de nouveaux PVC
- **Action requise** : Un administrateur doit manuellement nettoyer et libérer le PV

**Quand l'utiliser** :
- Données critiques nécessitant une intervention manuelle
- Environnement de production
- Quand vous voulez examiner les données avant suppression

**Exemple** :
```yaml
persistentVolumeReclaimPolicy: Retain
```

#### Delete (Supprimer)
- **Comportement** : Le PV et le stockage sous-jacent sont supprimés automatiquement
- **Données** : Perdues définitivement
- **État** : Le PV n'existe plus
- **Action requise** : Aucune

**Quand l'utiliser** :
- Environnement de développement/test
- Données temporaires
- Provisionnement dynamique (c'est le défaut)

**Exemple** :
```yaml
persistentVolumeReclaimPolicy: Delete
```

#### Recycle (Recycler) - DÉPRÉCIÉ
- **Comportement** : Les données sont supprimées (`rm -rf /*`)
- **État** : Le PV redevient "Available"
- **Statut** : Déprécié, ne plus utiliser

**⚠️ Ne pas utiliser Recycle dans les nouvelles installations !**

### storageClassName : Classe de stockage

```yaml
spec:
  storageClassName: manual
```

La StorageClass définit le "type" de stockage :
- Regroupe des PV avec des caractéristiques similaires
- Permet le provisionnement dynamique
- Optionnelle mais fortement recommandée

**Cas spéciaux** :
- Si `storageClassName: ""` (chaîne vide) : le PV ne peut être lié qu'à un PVC sans classe de stockage
- Si omis : comportement par défaut du cluster

### Type de stockage : La source réelle

Le type de stockage définit où les données seront physiquement stockées. Voici les types les plus courants :

#### hostPath : Stockage local

```yaml
spec:
  hostPath:
    path: /mnt/data
    type: DirectoryOrCreate
```

**Caractéristiques** :
- Utilise un répertoire sur le nœud
- Simple à configurer
- ⚠️ Données liées au nœud
- ⚠️ Non recommandé en production multi-nœuds
- ✅ Parfait pour MicroK8s en single-node

**Types disponibles** :
- `DirectoryOrCreate` : Crée le répertoire s'il n'existe pas
- `Directory` : Le répertoire doit exister
- `FileOrCreate` : Fichier, créé si nécessaire
- `File` : Le fichier doit exister

#### nfs : Network File System

```yaml
spec:
  nfs:
    server: nfs-server.example.com
    path: /exported/path
```

**Caractéristiques** :
- Stockage réseau partagé
- ✅ Support RWX (lecture-écriture multi-nœuds)
- ✅ Portable entre nœuds
- ⚠️ Performances réseau dépendantes
- ✅ Excellent pour le partage de fichiers

**Configuration minimale requise** :
- Un serveur NFS configuré
- Le client NFS installé sur les nœuds
- Les bonnes permissions d'accès

#### iscsi : Internet Small Computer Systems Interface

```yaml
spec:
  iscsi:
    targetPortal: 10.0.0.1:3260
    iqn: iqn.2001-04.com.example:storage.disk1
    lun: 0
    fsType: ext4
    readOnly: false
```

**Caractéristiques** :
- Protocole de stockage réseau bloc
- Bonnes performances
- Plus complexe à configurer que NFS
- Principalement RWO

#### csi : Container Storage Interface

```yaml
spec:
  csi:
    driver: pd.csi.storage.gke.io
    volumeHandle: projects/my-project/zones/us-central1-a/disks/my-disk
    fsType: ext4
```

**Caractéristiques** :
- Interface standard moderne
- Support pour de nombreux fournisseurs de stockage
- Extensible et flexible
- Le futur du stockage dans Kubernetes

**Exemples de drivers CSI** :
- Cloud providers (AWS EBS, GCP Persistent Disk, Azure Disk)
- Ceph
- GlusterFS
- Portworx
- Et beaucoup d'autres...

## Exemples complets de PersistentVolumes

### Exemple 1 : PV basique avec hostPath

Le plus simple, parfait pour MicroK8s en développement :

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv-hostpath-exemple
  labels:
    type: local
    usage: dev
spec:
  capacity:
    storage: 5Gi
  volumeMode: Filesystem
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  storageClassName: manual
  hostPath:
    path: /mnt/k8s-data/pv-exemple
    type: DirectoryOrCreate
```

**Explications** :
- Nom : `pv-hostpath-exemple`
- Capacité : 5 Gi
- Mode d'accès : RWO (un seul nœud)
- Politique : Retain (données conservées)
- Classe : "manual" (pas de provisionnement dynamique)
- Type : hostPath sur `/mnt/k8s-data/pv-exemple`

### Exemple 2 : PV avec NFS

Pour un stockage partagé entre plusieurs nœuds :

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv-nfs-shared
  labels:
    type: nfs
    tier: shared-storage
spec:
  capacity:
    storage: 20Gi
  volumeMode: Filesystem
  accessModes:
    - ReadWriteMany      # Supporte plusieurs nœuds
  persistentVolumeReclaimPolicy: Retain
  storageClassName: nfs-storage
  mountOptions:
    - hard
    - nfsvers=4.1
  nfs:
    server: 192.168.1.100
    path: /exports/k8s-storage
```

**Explications** :
- Capacité : 20 Gi
- Mode d'accès : RWX (plusieurs nœuds en lecture-écriture)
- Options de montage NFS personnalisées
- Serveur NFS : 192.168.1.100
- Chemin exporté : /exports/k8s-storage

### Exemple 3 : PV avec différents modes d'accès

Un PV peut supporter plusieurs modes d'accès :

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv-multi-access
spec:
  capacity:
    storage: 10Gi
  volumeMode: Filesystem
  accessModes:
    - ReadWriteOnce      # Support RWO
    - ReadOnlyMany       # Support ROX aussi
  persistentVolumeReclaimPolicy: Delete
  storageClassName: flexible-storage
  nfs:
    server: nfs.example.com
    path: /data/flexible
```

**Note** : Le PVC qui se liera à ce PV choisira un des modes supportés.

### Exemple 4 : PV avec node affinity

Pour cibler des nœuds spécifiques :

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv-local-ssd
spec:
  capacity:
    storage: 100Gi
  volumeMode: Filesystem
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Delete
  storageClassName: local-ssd
  local:
    path: /mnt/ssd-disk
  nodeAffinity:
    required:
      nodeSelectorTerms:
      - matchExpressions:
        - key: kubernetes.io/hostname
          operator: In
          values:
          - node-with-ssd    # Ce PV ne peut être utilisé que sur ce nœud
```

**Utilité** :
- Stockage local ultra-rapide (SSD)
- Contrôle précis du placement
- Utile pour optimisation des performances

## Gestion des PersistentVolumes

### Créer un PersistentVolume

```bash
# Depuis un fichier YAML
microk8s kubectl apply -f mon-pv.yaml

# Vérification
microk8s kubectl get pv
```

**Résultat attendu** :
```
NAME       CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS      CLAIM   STORAGECLASS
mon-pv     10Gi       RWO            Retain           Available           manual
```

**Statuts possibles** :
- `Available` : Prêt à être utilisé
- `Bound` : Lié à un PVC
- `Released` : Libéré mais contenant encore des données
- `Failed` : Erreur

### Lister les PersistentVolumes

```bash
# Liste simple
microk8s kubectl get pv

# Liste détaillée
microk8s kubectl get pv -o wide

# Avec tous les détails
microk8s kubectl describe pv <nom-pv>
```

### Voir les détails d'un PV

```bash
microk8s kubectl describe pv mon-pv
```

**Informations importantes à chercher** :
- **Status** : État actuel du PV
- **Claim** : PVC lié (si bound)
- **Reclaim Policy** : Que se passe-t-il à la suppression
- **Access Modes** : Comment il peut être monté
- **Events** : Événements récents

### Modifier un PersistentVolume

⚠️ **Attention** : Certains champs ne peuvent pas être modifiés après création :
- `capacity.storage` : Ne peut pas être réduite
- `accessModes` : Ne peut pas être changé
- Le type de stockage : Ne peut pas être changé

**Modifications possibles** :
- Labels et annotations
- Reclaim Policy (avec précaution)

```bash
# Éditer directement
microk8s kubectl edit pv mon-pv

# Ou appliquer un nouveau fichier
microk8s kubectl apply -f mon-pv-modifie.yaml
```

### Supprimer un PersistentVolume

```bash
microk8s kubectl delete pv mon-pv
```

**⚠️ Important** :
- Si le PV est `Bound`, vous devez d'abord supprimer le PVC
- Si la politique est `Retain`, les données physiques restent (nettoyage manuel requis)
- Si la politique est `Delete`, le stockage sous-jacent est supprimé

### Nettoyer un PV en statut Released

Quand un PV passe en statut `Released` (après suppression du PVC avec politique Retain) :

```bash
# 1. Vérifier l'état
microk8s kubectl get pv mon-pv

# 2. Sauvegarder/récupérer les données si nécessaire
# (accès direct au stockage, selon le type)

# 3. Supprimer le PV
microk8s kubectl delete pv mon-pv

# 4. Recréer le PV (si vous voulez le réutiliser)
microk8s kubectl apply -f mon-pv.yaml
```

## Provisionnement : Statique vs Dynamique

### Provisionnement statique

**Définition** : L'administrateur crée manuellement les PV avant que les utilisateurs ne les demandent.

**Processus** :
1. L'admin crée un PV manuellement
2. L'utilisateur crée un PVC
3. Kubernetes lie automatiquement le PVC au PV compatible

**Avantages** :
- ✅ Contrôle total sur le stockage
- ✅ Prévisibilité des coûts
- ✅ Optimisation possible

**Inconvénients** :
- ❌ Travail manuel pour l'admin
- ❌ Risque de pénurie si pas assez de PV créés
- ❌ Pas scalable pour de nombreux utilisateurs

**Exemple de workflow** :

```yaml
# 1. Admin crée le PV
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv-static-01
spec:
  capacity:
    storage: 10Gi
  accessModes:
    - ReadWriteOnce
  storageClassName: manual
  hostPath:
    path: /mnt/data/pv01

---
# 2. Utilisateur crée le PVC
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: my-claim
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 10Gi
  storageClassName: manual

# 3. Kubernetes lie automatiquement les deux
```

### Provisionnement dynamique

**Définition** : Les PV sont créés automatiquement à la demande quand un PVC est créé.

**Processus** :
1. Une StorageClass est configurée avec un provisionneur
2. L'utilisateur crée un PVC qui référence cette StorageClass
3. Kubernetes crée automatiquement un PV correspondant
4. Le PVC est automatiquement lié au nouveau PV

**Avantages** :
- ✅ Automatique, pas d'intervention admin
- ✅ Scalable
- ✅ Pas de gestion manuelle
- ✅ Pas de risque de pénurie

**Inconvénients** :
- ❌ Moins de contrôle
- ❌ Coûts potentiellement moins prévisibles
- ❌ Nécessite un provisionneur configuré

**Exemple** :

```yaml
# StorageClass (configuré une fois)
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: fast
provisioner: kubernetes.io/host-path
parameters:
  type: pd-ssd

---
# PVC (l'utilisateur ne crée que ça)
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: my-dynamic-claim
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 10Gi
  storageClassName: fast    # Référence la StorageClass

# Kubernetes crée automatiquement le PV !
```

## Dans MicroK8s : Simplicité avec l'addon hostpath-storage

MicroK8s simplifie énormément la gestion du stockage grâce à son addon :

### Activation de l'addon

```bash
microk8s enable hostpath-storage
```

**Ce que ça fait** :
- ✅ Active un provisionneur de stockage dynamique
- ✅ Crée une StorageClass par défaut nommée `microk8s-hostpath`
- ✅ Les PV sont créés automatiquement à la demande
- ✅ Stockage sur le nœud dans `/var/snap/microk8s/common/default-storage`

### Utilisation

Vous n'avez plus besoin de créer des PV manuellement ! Il suffit de créer des PVC :

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: mon-pvc-simple
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 5Gi
  # Pas besoin de spécifier storageClassName,
  # la classe par défaut sera utilisée
```

Et Kubernetes créera automatiquement le PV correspondant !

## Bonnes pratiques pour les PersistentVolumes

### 1. Utiliser des labels

Les labels facilitent l'organisation et la sélection :

```yaml
metadata:
  name: pv-prod-db
  labels:
    environment: production
    type: database
    tier: storage
    app: postgres
```

### 2. Documenter avec des annotations

```yaml
metadata:
  name: pv-important-data
  annotations:
    description: "Stockage pour la base de données de production"
    created-by: "admin@example.com"
    backup-schedule: "daily"
```

### 3. Choisir la bonne Reclaim Policy

```yaml
# Production : conservez les données
persistentVolumeReclaimPolicy: Retain

# Dev/Test : nettoyage automatique
persistentVolumeReclaimPolicy: Delete
```

### 4. Préfixer les noms

```yaml
# Bonne pratique
metadata:
  name: pv-prod-mysql-01
  name: pv-dev-cache-shared
  name: pv-test-webapp-data

# À éviter
metadata:
  name: volume1
  name: data
  name: temp
```

### 5. Définir une capacité appropriée

```yaml
spec:
  capacity:
    # Prévoyez de la marge pour la croissance
    storage: 50Gi    # Si vous estimez 30Gi, prévoyez 50Gi
```

### 6. Utiliser le provisionnement dynamique quand possible

Pour les environnements modernes, préférez le provisionnement dynamique :
- Plus simple
- Plus scalable
- Moins de gestion manuelle

## Erreurs courantes et solutions

### Erreur 1 : PV reste en statut "Available" malgré un PVC

**Symptôme** : Vous créez un PVC mais le PV n'est pas lié.

**Causes possibles** :
1. **StorageClass différente**
   ```yaml
   # PV
   storageClassName: manual
   # PVC
   storageClassName: fast    # ← Différent !
   ```
   **Solution** : Assurez-vous que les StorageClass correspondent.

2. **Capacité insuffisante**
   ```yaml
   # PV
   capacity:
     storage: 5Gi
   # PVC demande
   resources:
     requests:
       storage: 10Gi    # ← Plus que disponible !
   ```
   **Solution** : Le PV doit avoir au moins la capacité demandée.

3. **Modes d'accès incompatibles**
   ```yaml
   # PV
   accessModes:
     - ReadWriteOnce
   # PVC
   accessModes:
     - ReadWriteMany    # ← Incompatible !
   ```
   **Solution** : Les modes doivent être compatibles.

### Erreur 2 : PV en statut "Released" et inutilisable

**Symptôme** : Après suppression d'un PVC, le PV est en statut `Released` et ne peut plus être utilisé.

**Cause** : Politique `Retain` - le PV conserve les données du PVC précédent.

**Solution** :
```bash
# Option 1 : Supprimer et recréer le PV
microk8s kubectl delete pv mon-pv
# Récupérer les données si nécessaire
microk8s kubectl apply -f mon-pv.yaml

# Option 2 : Modifier le PV pour enlever la référence au claim
microk8s kubectl edit pv mon-pv
# Supprimer la section spec.claimRef
```

### Erreur 3 : Données perdues après suppression du PVC

**Symptôme** : Vous supprimez un PVC et les données disparaissent.

**Cause** : Politique `Delete` active.

**Prévention** :
```yaml
# Utilisez Retain pour les données importantes
persistentVolumeReclaimPolicy: Retain
```

**Si c'est trop tard** : Vérifiez vos sauvegardes ! C'est pourquoi les backups sont essentiels.

### Erreur 4 : Permissions insuffisantes sur hostPath

**Symptôme** : Le pod ne peut pas écrire dans le volume.

**Cause** : Permissions incorrectes sur le répertoire du nœud.

**Solution** :
```bash
# Sur le nœud
sudo chmod -R 777 /mnt/k8s-data    # Large (dev)
# Ou
sudo chown -R 1000:1000 /mnt/k8s-data    # Propriétaire spécifique
```

## Commandes de diagnostic utiles

### Vérifier l'état d'un PV

```bash
# Liste
microk8s kubectl get pv

# Détails
microk8s kubectl describe pv <nom-pv>

# Format YAML
microk8s kubectl get pv <nom-pv> -o yaml
```

### Voir les événements liés au PV

```bash
microk8s kubectl get events --field-selector involvedObject.name=<nom-pv>
```

### Vérifier le lien PV-PVC

```bash
# Voir tous les PV avec leur PVC lié
microk8s kubectl get pv -o custom-columns=NAME:.metadata.name,CLAIM:.spec.claimRef.name,STATUS:.status.phase
```

### Vérifier la StorageClass

```bash
microk8s kubectl get storageclass
microk8s kubectl describe storageclass <nom-classe>
```

## Résumé visuel : Le cycle complet

```
┌─────────────────────────────────────────────────────────┐
│                  CRÉATION DU PV                         │
│                                                         │
│  1. Admin crée PV (statique)                            │
│     OU                                                  │
│     Provisionneur crée PV (dynamique)                   │
│                          │                              │
│                          ▼                              │
│                     [Available]                         │
│                          │                              │
│  2. Utilisateur crée PVC │                              │
│                          │                              │
│                          ▼                              │
│                      [Binding]                          │
│                          │                              │
│                          ▼                              │
│                      [Bound]                            │
│                          │                              │
│  3. Pod utilise le PVC   │                              │
│                          │                              │
│  4. Utilisateur          │                              │
│     supprime PVC         │                              │
│                          ▼                              │
│                  ┌──────────────┐                       │
│                  │   Policy ?   │                       │
│                  └──────┬───────┘                       │
│                         │                               │
│              ┌──────────┴──────────┐                    │
│              ▼                     ▼                    │
│         [Retain]               [Delete]                 │
│         Released            PV supprimé                 │
│   (données conservées)    (données perdues)             │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

## Prochaines étapes

Maintenant que vous comprenez les PersistentVolumes, vous êtes prêt pour :

- **Section 6.4** : PersistentVolumeClaims (PVC) - Comment les applications demandent du stockage
- **Section 6.5** : StorageClasses - Configuration du provisionnement dynamique
- **Section 6.6** : Gestion complète des volumes persistants
- **Section 6.7** : StatefulSets - Applications avec stockage persistant

## Conclusion

Les PersistentVolumes sont la fondation du stockage persistant dans Kubernetes. Ils représentent le stockage physique réel et existent indépendamment des pods et des applications.

**Points essentiels à retenir** :

- Les PV sont créés par les administrateurs ou automatiquement
- Ils ont une durée de vie indépendante des pods
- Les modes d'accès définissent comment le stockage peut être partagé
- La politique de récupération contrôle ce qui arrive aux données
- MicroK8s simplifie tout avec le provisionnement dynamique

Dans la prochaine section, nous verrons comment les utilisateurs consomment ces PV via les PersistentVolumeClaims, complétant ainsi le puzzle du stockage persistant dans Kubernetes.

---

**Points clés à retenir** :

✅ **PV** = Ressource de stockage au niveau cluster
✅ **Statut** : Available → Bound → Released/Failed
✅ **Access Modes** : RWO, ROX, RWX, RWOP
✅ **Reclaim Policy** : Retain (conservation) ou Delete (suppression)
✅ **Provisionnement** : Statique (manuel) ou Dynamique (automatique)
✅ **MicroK8s** : Addon hostpath-storage pour simplifier
✅ **Production** : Toujours utiliser Retain pour les données critiques
✅ **Backups** : Essentiels même avec stockage persistant !

⏭️ [PersistentVolumeClaims (PVC)](/06-stockage-persistant/04-persistentvolumeclaims-pvc.md)
