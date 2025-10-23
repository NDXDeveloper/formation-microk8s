🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 6.5 StorageClasses

## Introduction

Dans les sections précédentes, nous avons exploré les PersistentVolumes (PV) et les PersistentVolumeClaims (PVC). Nous avons vu comment un utilisateur peut demander du stockage via un PVC et comment Kubernetes lie ce PVC à un PV disponible. Mais il reste une question importante : **qui crée les PV et comment ?**

C'est là qu'interviennent les **StorageClasses** (Classes de Stockage). Elles sont le mécanisme qui permet le **provisionnement dynamique** du stockage dans Kubernetes, éliminant le besoin pour les administrateurs de créer manuellement des PV pour chaque demande.

Une StorageClass est essentiellement un "modèle" ou un "type" de stockage que vous pouvez proposer aux utilisateurs. Elle décrit les caractéristiques du stockage et comment le créer automatiquement.

## Qu'est-ce qu'une StorageClass ?

### Définition

Une **StorageClass** (SC) est une ressource Kubernetes qui :
- Définit un "type" ou une "classe" de stockage disponible dans le cluster
- Décrit comment provisionner dynamiquement des volumes de ce type
- Spécifie les paramètres et caractéristiques du stockage
- Permet aux utilisateurs de demander du stockage sans connaître les détails techniques

**Analogie** : Imaginez un restaurant avec différentes catégories de plats : "Menu Végétarien", "Menu Express", "Menu Gastronomique". Chaque menu (StorageClass) définit un type de service avec ses propres caractéristiques (ingrédients, temps de préparation, prix). Quand un client commande un "Menu Express" (PVC), le restaurant sait exactement comment le préparer (provisionner un PV).

### Le problème qu'elle résout

**Sans StorageClass (provisionnement statique)** :
```
1. Admin crée manuellement PV-1 (10Gi)
2. Admin crée manuellement PV-2 (20Gi)
3. Admin crée manuellement PV-3 (5Gi)
   ...
4. Utilisateur crée PVC demandant 15Gi
5. Problème : aucun PV ne correspond exactement !
6. Admin doit créer un nouveau PV manuellement
```

**Avec StorageClass (provisionnement dynamique)** :
```
1. Admin configure une StorageClass une seule fois
2. Utilisateur crée PVC demandant 15Gi avec cette StorageClass
3. Kubernetes crée automatiquement un PV de 15Gi
4. Le PVC est automatiquement lié au nouveau PV
5. Terminé ! Aucune intervention manuelle requise
```

## Architecture : Comment ça fonctionne ?

### Vue d'ensemble

```
┌─────────────────────────────────────────────────────────┐
│                  ADMINISTRATEUR                         │
│                                                         │
│  1. Configure la StorageClass une seule fois            │
│                                                         │
│     ┌─────────────────────────────────┐                 │
│     │      StorageClass               │                 │
│     │  ─────────────────              │                 │
│     │  • Provisioner: XXX             │                 │
│     │  • Parameters: {...}            │                 │
│     │  • ReclaimPolicy: Delete        │                 │
│     └────────────┬────────────────────┘                 │
│                  │                                      │
└──────────────────┼──────────────────────────────────────┘
                   │
                   │ Référence
                   │
┌──────────────────▼──────────────────────────────────────┐
│                  UTILISATEUR                            │
│                                                         │
│  2. Crée un PVC qui référence la StorageClass           │
│                                                         │
│     ┌─────────────────────────────────┐                 │
│     │  PersistentVolumeClaim          │                 │
│     │  ─────────────────────          │                 │
│     │  • storageClassName: XXX        │                 │
│     │  • size: 10Gi                   │                 │
│     └────────────┬────────────────────┘                 │
│                  │                                      │
└──────────────────┼──────────────────────────────────────┘
                   │
                   │ Déclenchement
                   │
┌──────────────────▼──────────────────────────────────────┐
│              KUBERNETES (automatique)                   │
│                                                         │
│  3. Détecte le nouveau PVC                              │
│  4. Lit la StorageClass référencée                      │
│  5. Appelle le Provisioner                              │
│  6. Le Provisioner crée le stockage physique            │
│  7. Kubernetes crée automatiquement un PV               │
│  8. Lie le PVC au nouveau PV                            │
│                                                         │
│     ┌─────────────────────────────────┐                 │
│     │    PersistentVolume             │                 │
│     │    (créé automatiquement)       │                 │
│     └─────────────────────────────────┘                 │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

### Les acteurs

1. **StorageClass** : Le modèle/recette
2. **Provisioner** : Le "chef" qui sait comment créer le stockage
3. **PVC** : La commande du client
4. **PV** : Le plat préparé (créé automatiquement)

## Anatomie d'une StorageClass

Examinons la structure complète d'une StorageClass :

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: fast-ssd
  annotations:
    storageclass.kubernetes.io/is-default-class: "false"
provisioner: kubernetes.io/host-path
parameters:
  type: pd-ssd
  fsType: ext4
reclaimPolicy: Delete
allowVolumeExpansion: true
volumeBindingMode: WaitForFirstConsumer
mountOptions:
  - debug
```

Décortiquons chaque partie :

### metadata : Identification de la StorageClass

```yaml
metadata:
  name: fast-ssd    # Nom unique de la classe
  annotations:
    storageclass.kubernetes.io/is-default-class: "true"    # Classe par défaut ?
```

**Points importants** :
- Le nom doit être unique dans le cluster
- Les StorageClasses sont des ressources au niveau cluster (pas namespacées)
- L'annotation `is-default-class` définit si c'est la classe par défaut

### provisioner : Le gestionnaire de provisionnement

```yaml
provisioner: kubernetes.io/host-path
```

Le provisioner est le composant qui sait comment créer physiquement le stockage. C'est le "moteur" du provisionnement dynamique.

**Provisioners intégrés (in-tree)** :

| Provisioner | Description | Usage |
|-------------|-------------|-------|
| `kubernetes.io/host-path` | Stockage local sur le nœud | Dev/Test, MicroK8s |
| `kubernetes.io/gce-pd` | Google Compute Engine Persistent Disk | GCP |
| `kubernetes.io/aws-ebs` | AWS Elastic Block Store | AWS |
| `kubernetes.io/azure-disk` | Azure Disk | Azure |
| `kubernetes.io/azure-file` | Azure File | Azure |
| `kubernetes.io/cinder` | OpenStack Cinder | OpenStack |
| `kubernetes.io/vsphere-volume` | vSphere VMDK | VMware |

**Provisioners externes (out-of-tree, via CSI)** :

Les provisioners modernes utilisent l'interface CSI (Container Storage Interface) :

```yaml
# Format CSI
provisioner: pd.csi.storage.gke.io          # GCP
provisioner: ebs.csi.aws.com                # AWS
provisioner: disk.csi.azure.com             # Azure
provisioner: nfs.csi.k8s.io                 # NFS
provisioner: rook-ceph.rbd.csi.ceph.com     # Ceph
```

**Pour MicroK8s** :
```yaml
provisioner: microk8s.io/hostpath    # Provisioner spécifique à MicroK8s
```

### parameters : Configuration spécifique au provisioner

```yaml
parameters:
  type: pd-ssd
  fsType: ext4
  replication-type: regional-pd
```

Les paramètres varient selon le provisioner. Ce sont les "options" pour la création du volume.

**Exemples selon le provisioner** :

#### Pour AWS EBS
```yaml
parameters:
  type: gp3              # Type de volume : gp2, gp3, io1, io2, st1, sc1
  iops: "3000"           # IOPS pour io1/io2/gp3
  throughput: "125"      # Throughput pour gp3
  encrypted: "true"      # Chiffrement
  kmsKeyId: "arn:..."    # Clé KMS pour le chiffrement
```

#### Pour GCP Persistent Disk
```yaml
parameters:
  type: pd-ssd           # pd-standard ou pd-ssd
  replication-type: none # none ou regional-pd
  fsType: ext4           # Système de fichiers
```

#### Pour Azure Disk
```yaml
parameters:
  storageaccounttype: Premium_LRS    # Type de disque
  kind: Managed                       # Managed ou Shared
  cachingmode: ReadOnly               # Cache
```

#### Pour NFS
```yaml
parameters:
  server: nfs-server.example.com     # Serveur NFS
  path: /exports/storage             # Chemin exporté
  mountOptions: "nfsvers=4.1"        # Options de montage
```

#### Pour MicroK8s hostpath
```yaml
parameters:
  # Généralement aucun paramètre nécessaire
  pvDir: /var/snap/microk8s/common/default-storage    # Optionnel
```

### reclaimPolicy : Politique de récupération

```yaml
reclaimPolicy: Delete    # ou Retain
```

Définit ce qui arrive au volume physique quand le PVC est supprimé :

#### Delete (par défaut)
- Le PV et le stockage sous-jacent sont supprimés automatiquement
- Les données sont perdues définitivement
- Idéal pour : environnements de développement, données temporaires

```yaml
reclaimPolicy: Delete
```

**Conséquence** :
```
PVC supprimé → PV supprimé → Stockage physique supprimé → Données perdues
```

#### Retain
- Le PV est marqué "Released" mais conservé
- Les données restent sur le stockage physique
- Un administrateur doit manuellement nettoyer
- Idéal pour : données critiques, production

```yaml
reclaimPolicy: Retain
```

**Conséquence** :
```
PVC supprimé → PV "Released" → Stockage physique conservé → Données préservées
```

**⚠️ Attention** : Avec le provisionnement dynamique, le défaut est `Delete` !

### allowVolumeExpansion : Autoriser l'expansion

```yaml
allowVolumeExpansion: true    # ou false
```

Permet aux utilisateurs d'augmenter la taille d'un PVC après sa création.

**Conditions** :
- Le provisioner doit supporter l'expansion
- On peut seulement **augmenter**, jamais réduire
- Certains types de stockage nécessitent un redémarrage du pod

**Exemple d'utilisation** :
```yaml
# StorageClass avec expansion activée
allowVolumeExpansion: true

# PVC initial
spec:
  resources:
    requests:
      storage: 10Gi

# Après modification du PVC
spec:
  resources:
    requests:
      storage: 20Gi    # Augmentation possible
```

**Support par provisioner** :
- ✅ AWS EBS : Oui
- ✅ GCP Persistent Disk : Oui
- ✅ Azure Disk : Oui
- ✅ Hostpath : Oui (mais limité)
- ❌ NFS : Non (généralement)

### volumeBindingMode : Mode de liaison

```yaml
volumeBindingMode: WaitForFirstConsumer    # ou Immediate
```

Contrôle **quand** le PV est créé et lié au PVC.

#### Immediate (par défaut)
- Le PV est créé et lié **immédiatement** quand le PVC est créé
- Même si aucun pod n'utilise encore le PVC
- Plus simple mais peut causer des problèmes de placement

```yaml
volumeBindingMode: Immediate
```

**Timeline** :
```
1. PVC créé
2. PV créé immédiatement
3. PVC lié au PV
4. (Plus tard) Pod créé et utilise le PVC
```

**Problème potentiel** : Si le PV est créé sur un nœud, et que le pod est schedulé sur un autre nœud incompatible.

#### WaitForFirstConsumer (recommandé)
- Le PV est créé seulement quand un pod utilisant le PVC est créé
- Kubernetes attend de savoir sur quel nœud le pod sera placé
- Garantit que le PV sera créé au bon endroit
- **Recommandé pour les clusters multi-nœuds**

```yaml
volumeBindingMode: WaitForFirstConsumer
```

**Timeline** :
```
1. PVC créé (reste Pending)
2. Pod créé qui utilise le PVC
3. Kubernetes choisit un nœud pour le pod
4. PV créé sur ce nœud (ou compatible avec)
5. PVC lié au PV
6. Pod démarre
```

**Comparaison visuelle** :

```
Immediate:
├─ PVC créé → PV créé immédiatement → Nœud A
├─ Plus tard : Pod schedulé → Nœud B
└─ Problème si les nœuds incompatibles !

WaitForFirstConsumer:
├─ PVC créé → Attend...
├─ Pod créé → Scheduler choisit Nœud B
├─ PV créé compatible avec Nœud B
└─ Tout fonctionne !
```

### mountOptions : Options de montage

```yaml
mountOptions:
  - hard
  - nfsvers=4.1
  - nolock
```

Options passées au système lors du montage du volume. Dépendent du type de système de fichiers.

**Exemples pour NFS** :
```yaml
mountOptions:
  - hard              # Retry indéfiniment en cas d'erreur
  - nfsvers=4.1       # Version NFS
  - timeo=600         # Timeout
  - retrans=2         # Nombre de retransmissions
  - nolock            # Pas de verrouillage
```

**Exemples pour SMB/CIFS** :
```yaml
mountOptions:
  - dir_mode=0777
  - file_mode=0777
  - uid=1000
  - gid=1000
```

**⚠️ Important** :
- Tous les provisioners ne supportent pas les mountOptions
- Des options incorrectes peuvent empêcher le montage du volume

## Exemples complets de StorageClasses

### Exemple 1 : StorageClass MicroK8s simple

La plus simple pour débuter avec MicroK8s :

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: microk8s-hostpath
  annotations:
    storageclass.kubernetes.io/is-default-class: "true"
provisioner: microk8s.io/hostpath
reclaimPolicy: Delete
allowVolumeExpansion: true
volumeBindingMode: Immediate
```

**Caractéristiques** :
- Classe par défaut de MicroK8s
- Stockage local sur le nœud
- Suppression automatique des données
- Expansion activée
- Binding immédiat

### Exemple 2 : StorageClass avec Retain pour la production

Pour des données critiques :

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: critical-data
  annotations:
    description: "Stockage pour données critiques avec conservation"
provisioner: microk8s.io/hostpath
reclaimPolicy: Retain    # Données conservées !
allowVolumeExpansion: true
volumeBindingMode: WaitForFirstConsumer
```

**Usage** :
```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: database-pvc
spec:
  storageClassName: critical-data    # Utilise cette classe
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 20Gi
```

### Exemple 3 : StorageClasses multiples pour différents besoins

Proposer plusieurs types de stockage :

```yaml
# Classe "fast" - SSD rapide, cher
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: fast
  annotations:
    description: "Stockage SSD haute performance"
provisioner: microk8s.io/hostpath
parameters:
  type: ssd
reclaimPolicy: Delete
allowVolumeExpansion: true
volumeBindingMode: WaitForFirstConsumer
---
# Classe "standard" - HDD standard, économique
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: standard
  annotations:
    description: "Stockage standard économique"
    storageclass.kubernetes.io/is-default-class: "true"
provisioner: microk8s.io/hostpath
parameters:
  type: hdd
reclaimPolicy: Delete
allowVolumeExpansion: true
volumeBindingMode: Immediate
---
# Classe "backup" - Avec rétention des données
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: backup
  annotations:
    description: "Stockage pour backups avec rétention"
provisioner: microk8s.io/hostpath
reclaimPolicy: Retain    # Conserve les données
allowVolumeExpansion: false
volumeBindingMode: Immediate
```

**Utilisation par les développeurs** :
```yaml
# Application critique : stockage rapide
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: app-cache
spec:
  storageClassName: fast
  accessModes: [ReadWriteOnce]
  resources:
    requests:
      storage: 10Gi
---
# Application standard : stockage par défaut
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: app-data
spec:
  # storageClassName omis = utilise la classe par défaut (standard)
  accessModes: [ReadWriteOnce]
  resources:
    requests:
      storage: 50Gi
---
# Backups : avec rétention
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: app-backup
spec:
  storageClassName: backup
  accessModes: [ReadWriteOnce]
  resources:
    requests:
      storage: 100Gi
```

### Exemple 4 : StorageClass NFS pour stockage partagé

Pour un stockage accessible par plusieurs nœuds :

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: nfs-storage
  annotations:
    description: "Stockage NFS partagé"
provisioner: nfs.csi.k8s.io
parameters:
  server: nfs-server.example.com
  share: /exports/kubernetes
  mountOptions: "nfsvers=4.1,hard"
reclaimPolicy: Retain
allowVolumeExpansion: false
volumeBindingMode: Immediate
mountOptions:
  - hard
  - nfsvers=4.1
  - nolock
```

### Exemple 5 : StorageClass AWS EBS (pour référence cloud)

Pour ceux qui travaillent sur AWS :

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: aws-ebs-gp3
provisioner: ebs.csi.aws.com
parameters:
  type: gp3
  iops: "3000"
  throughput: "125"
  encrypted: "true"
  fsType: ext4
reclaimPolicy: Delete
allowVolumeExpansion: true
volumeBindingMode: WaitForFirstConsumer
```

## StorageClass par défaut

### Qu'est-ce qu'une StorageClass par défaut ?

Une StorageClass peut être marquée comme **par défaut** (default). Quand un PVC ne spécifie pas de `storageClassName`, la classe par défaut est utilisée automatiquement.

### Définir une classe par défaut

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: standard
  annotations:
    storageclass.kubernetes.io/is-default-class: "true"    # Marque comme défaut
provisioner: microk8s.io/hostpath
reclaimPolicy: Delete
```

**Vérification** :
```bash
microk8s kubectl get storageclass
```

**Résultat** :
```
NAME                          PROVISIONER            RECLAIMPOLICY   AGE
microk8s-hostpath (default)   microk8s.io/hostpath   Delete          5d
fast                          microk8s.io/hostpath   Delete          2d
```

Le `(default)` indique la classe par défaut.

### Utilisation automatique

```yaml
# PVC sans storageClassName spécifié
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: auto-pvc
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 5Gi
  # Pas de storageClassName → utilise la classe par défaut
```

### Changer la classe par défaut

```bash
# Retirer le statut par défaut de l'ancienne classe
microk8s kubectl patch storageclass old-default \
  -p '{"metadata": {"annotations":{"storageclass.kubernetes.io/is-default-class":"false"}}}'

# Marquer la nouvelle classe comme par défaut
microk8s kubectl patch storageclass new-default \
  -p '{"metadata": {"annotations":{"storageclass.kubernetes.io/is-default-class":"true"}}}'
```

### Désactiver l'utilisation de la classe par défaut

Pour forcer l'utilisation explicite d'une classe :

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: explicit-pvc
spec:
  storageClassName: ""    # Chaîne vide = pas de classe (provisionnement statique)
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 5Gi
```

## Gestion des StorageClasses

### Créer une StorageClass

```bash
# Depuis un fichier YAML
microk8s kubectl apply -f ma-storageclass.yaml

# Vérification
microk8s kubectl get storageclass
# ou
microk8s kubectl get sc    # Abréviation
```

### Lister les StorageClasses

```bash
# Liste simple
microk8s kubectl get storageclass

# Avec plus de détails
microk8s kubectl get storageclass -o wide

# Format YAML
microk8s kubectl get storageclass <nom> -o yaml
```

### Voir les détails d'une StorageClass

```bash
microk8s kubectl describe storageclass <nom>
```

**Exemple de sortie** :
```
Name:                  fast
IsDefaultClass:        No
Annotations:           description=Stockage SSD haute performance
Provisioner:           microk8s.io/hostpath
Parameters:            type=ssd
AllowVolumeExpansion:  True
MountOptions:          <none>
ReclaimPolicy:         Delete
VolumeBindingMode:     WaitForFirstConsumer
Events:                <none>
```

### Modifier une StorageClass

⚠️ **Limitations importantes** :
- Une fois créée, la plupart des champs sont **immuables**
- Vous ne pouvez modifier que : annotations, labels
- Vous **ne pouvez pas** modifier : provisioner, parameters, reclaimPolicy, etc.

**Modifications possibles** :
```bash
# Modifier les annotations
microk8s kubectl annotate storageclass fast description="Nouveau texte"

# Ajouter un label
microk8s kubectl label storageclass fast tier=premium
```

**Si vous devez changer des paramètres immuables** :
```bash
# 1. Créer une nouvelle StorageClass avec les bons paramètres
microk8s kubectl apply -f nouvelle-storageclass.yaml

# 2. Migrer les PVC vers la nouvelle classe (recréation nécessaire)

# 3. Supprimer l'ancienne classe (après migration complète)
microk8s kubectl delete storageclass ancienne-classe
```

### Supprimer une StorageClass

```bash
microk8s kubectl delete storageclass <nom>
```

**⚠️ Important** :
- Les PVC existants utilisant cette classe continuent de fonctionner
- Les nouveaux PVC ne pourront plus utiliser cette classe
- Les PV existants ne sont pas affectés
- Vérifiez qu'aucun PVC ne référence cette classe avant suppression

**Vérifier les PVC utilisant une classe** :
```bash
microk8s kubectl get pvc --all-namespaces -o json | \
  jq -r '.items[] | select(.spec.storageClassName=="<nom-classe>") | .metadata.name'
```

## Provisionnement : Statique vs Dynamique

### Comparaison détaillée

```
┌──────────────────────────────────────────────────────────┐
│          PROVISIONNEMENT STATIQUE                        │
├──────────────────────────────────────────────────────────┤
│                                                          │
│  1. Admin crée manuellement des PV                       │
│     ├─ PV-1 : 10Gi                                       │
│     ├─ PV-2 : 20Gi                                       │
│     └─ PV-3 : 50Gi                                       │
│                                                          │
│  2. Utilisateur crée PVC (demande 15Gi)                  │
│                                                          │
│  3. Kubernetes cherche un PV compatible                  │
│     └─ Trouve PV-2 (20Gi >= 15Gi demandés)               │
│                                                          │
│  4. Lie le PVC au PV-2                                   │
│                                                          │
│  Avantages :                                             │
│  ✅ Contrôle total                                       │
│  ✅ Prévisibilité                                        │
│                                                          │
│  Inconvénients :                                         │
│  ❌ Travail manuel                                       │
│  ❌ Pas scalable                                         │
│  ❌ Risque de pénurie                                    │
│  ❌ Gaspillage (PV-2 a 5Gi inutilisés)                   │
│                                                          │
└──────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────┐
│          PROVISIONNEMENT DYNAMIQUE                       │
├──────────────────────────────────────────────────────────┤
│                                                          │
│  1. Admin configure une StorageClass                     │
│     └─ Définit comment créer des volumes                 │
│                                                          │
│  2. Utilisateur crée PVC (demande 15Gi)                  │
│     └─ Spécifie la StorageClass                          │
│                                                          │
│  3. Kubernetes détecte le PVC                            │
│                                                          │
│  4. Provisioner crée automatiquement un PV de 15Gi       │
│                                                          │
│  5. Kubernetes lie automatiquement le PVC au nouveau PV  │
│                                                          │
│  Avantages :                                             │
│  ✅ Automatique                                          │
│  ✅ Scalable                                             │
│  ✅ Taille exacte (pas de gaspillage)                    │
│  ✅ Pas de pénurie                                       │
│                                                          │
│  Inconvénients :                                         │
│  ❌ Moins de contrôle fin                                │
│  ❌ Dépend du provisioner                                │
│  ❌ Coûts moins prévisibles (cloud)                      │
│                                                          │
└──────────────────────────────────────────────────────────┘
```

### Quand utiliser quel mode ?

**Provisionnement Statique** :
- ✅ Environnements avec contraintes strictes
- ✅ Volumes avec des caractéristiques très spécifiques
- ✅ Stockage sur du matériel physique limité
- ✅ Contrôle total des coûts requis

**Provisionnement Dynamique** :
- ✅ Environnements cloud
- ✅ Développement et test
- ✅ Nombreux utilisateurs/applications
- ✅ Besoin de rapidité et d'automatisation
- ✅ **Recommandé dans la majorité des cas**

## Cas d'usage pratiques

### Cas 1 : Environnement de développement

```yaml
# StorageClass pour dev : simple et rapide
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: dev
  annotations:
    storageclass.kubernetes.io/is-default-class: "true"
    description: "Stockage pour développement"
provisioner: microk8s.io/hostpath
reclaimPolicy: Delete    # Nettoyage automatique
allowVolumeExpansion: true
volumeBindingMode: Immediate
```

### Cas 2 : Environnement de production

```yaml
# StorageClass standard pour production
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: production-standard
  annotations:
    description: "Stockage standard production"
provisioner: microk8s.io/hostpath
reclaimPolicy: Retain    # Conservation des données
allowVolumeExpansion: true
volumeBindingMode: WaitForFirstConsumer
---
# StorageClass haute performance pour bases de données
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: production-premium
  annotations:
    description: "Stockage haute performance pour DB"
provisioner: microk8s.io/hostpath
parameters:
  type: ssd
reclaimPolicy: Retain
allowVolumeExpansion: true
volumeBindingMode: WaitForFirstConsumer
```

### Cas 3 : Stockage partagé multi-application

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: shared-storage
  annotations:
    description: "Stockage partagé NFS"
provisioner: nfs.csi.k8s.io
parameters:
  server: nfs-server.local
  share: /exports/shared
reclaimPolicy: Retain
allowVolumeExpansion: false
volumeBindingMode: Immediate
mountOptions:
  - hard
  - nfsvers=4.1
```

**Utilisation** :
```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: webapp-shared-files
spec:
  storageClassName: shared-storage
  accessModes:
    - ReadWriteMany    # Plusieurs pods peuvent écrire
  resources:
    requests:
      storage: 50Gi
```

## Bonnes pratiques

### 1. Nommer clairement les StorageClasses

```yaml
# Bon : noms descriptifs
metadata:
  name: fast-ssd-delete
  name: standard-hdd-retain
  name: backup-slow-retain

# À éviter : noms vagues
metadata:
  name: storage1
  name: class-a
  name: default
```

**Pattern recommandé** : `<performance>-<type>-<policy>`

### 2. Documenter avec des annotations

```yaml
metadata:
  name: premium-ssd
  annotations:
    description: "Stockage SSD haute performance pour applications critiques"
    cost: "élevé"
    use-case: "bases de données, cache"
    backup: "journalier"
    retention: "90 jours"
```

### 3. Définir une classe par défaut

Toujours avoir une StorageClass par défaut pour simplifier l'utilisation :

```yaml
metadata:
  annotations:
    storageclass.kubernetes.io/is-default-class: "true"
```

### 4. Utiliser Retain en production

Pour les données critiques :

```yaml
reclaimPolicy: Retain    # Toujours pour la production
```

### 5. Activer l'expansion de volume

Sauf raison spécifique, activez toujours :

```yaml
allowVolumeExpansion: true
```

### 6. Utiliser WaitForFirstConsumer en multi-nœud

Pour les clusters avec plusieurs nœuds :

```yaml
volumeBindingMode: WaitForFirstConsumer
```

### 7. Créer plusieurs classes pour différents besoins

Offrez différents niveaux de service :

```yaml
# Bronze : économique
name: bronze
reclaimPolicy: Delete
parameters:
  type: hdd

# Silver : standard
name: silver
reclaimPolicy: Retain
parameters:
  type: standard

# Gold : premium
name: gold
reclaimPolicy: Retain
parameters:
  type: ssd
```

### 8. Tester avant de déployer en production

```bash
# Créer une StorageClass de test
microk8s kubectl apply -f test-storageclass.yaml

# Créer un PVC de test
cat <<EOF | microk8s kubectl apply -f -
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: test-pvc
spec:
  storageClassName: test
  accessModes: [ReadWriteOnce]
  resources:
    requests:
      storage: 1Gi
EOF

# Vérifier que ça fonctionne
microk8s kubectl get pvc test-pvc
microk8s kubectl describe pvc test-pvc

# Nettoyer
microk8s kubectl delete pvc test-pvc
```

## Erreurs courantes et solutions

### Erreur 1 : PVC reste en Pending avec StorageClass

**Symptôme** :
```bash
$ microk8s kubectl get pvc
NAME      STATUS    VOLUME   CAPACITY   STORAGECLASS
mon-pvc   Pending                       fast-ssd
```

**Diagnostic** :
```bash
microk8s kubectl describe pvc mon-pvc
```

**Causes possibles** :

#### a) StorageClass n'existe pas
```
Events:
  Warning  ProvisioningFailed  storageclass.storage.k8s.io "fast-ssd" not found
```

**Solution** :
```bash
# Vérifier les classes disponibles
microk8s kubectl get storageclass

# Corriger le nom dans le PVC
microk8s kubectl edit pvc mon-pvc
```

#### b) Provisioner non installé
```
Events:
  Warning  ProvisioningFailed  no volume plugin matched
```

**Solution pour MicroK8s** :
```bash
microk8s enable hostpath-storage
```

#### c) WaitForFirstConsumer sans pod
```
Events:
  Normal   WaitForFirstConsumer  waiting for first consumer
```

**Solution** : C'est normal ! Créez un pod qui utilise le PVC :
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: test-pod
spec:
  containers:
  - name: test
    image: nginx
    volumeMounts:
    - name: storage
      mountPath: /data
  volumes:
  - name: storage
    persistentVolumeClaim:
      claimName: mon-pvc
```

### Erreur 2 : Impossible de modifier la StorageClass

**Symptôme** : Erreur lors de la modification.

**Cause** : Les StorageClasses sont largement immuables.

**Solution** :
```bash
# Créer une nouvelle StorageClass
microk8s kubectl apply -f nouvelle-storageclass.yaml

# Mettre à jour les applications pour utiliser la nouvelle classe
# Supprimer l'ancienne classe (après migration)
microk8s kubectl delete storageclass ancienne-classe
```

### Erreur 3 : Expansion de volume échoue

**Symptôme** :
```
Events:
  Warning  VolumeResizeFailed  volume expansion not supported
```

**Causes** :

1. **allowVolumeExpansion n'est pas activé**
```bash
microk8s kubectl get storageclass <nom> -o yaml | grep allowVolumeExpansion
```

**Solution** : Créer une nouvelle StorageClass avec expansion activée.

2. **Le provisioner ne supporte pas l'expansion**

**Solution** : Vérifier la documentation du provisioner.

3. **Pod doit être redémarré**

**Solution** :
```bash
# Supprimer le pod (sera recréé par le Deployment)
microk8s kubectl delete pod <nom-pod>
```

### Erreur 4 : Conflits de classes par défaut

**Symptôme** : Plusieurs classes marquées comme par défaut.

**Solution** :
```bash
# Lister toutes les classes
microk8s kubectl get storageclass

# Retirer le statut par défaut des classes indésirables
microk8s kubectl patch storageclass <nom> \
  -p '{"metadata": {"annotations":{"storageclass.kubernetes.io/is-default-class":"false"}}}'
```

### Erreur 5 : Permissions insuffisantes sur hostpath

**Symptôme** : Les pods ne peuvent pas écrire dans le volume.

**Solution** :
```bash
# Sur le nœud MicroK8s
sudo chmod -R 777 /var/snap/microk8s/common/default-storage

# Ou pour un chemin spécifique
sudo chown -R 1000:1000 /mnt/k8s-data
```

## MicroK8s : Configuration de l'addon hostpath-storage

### Activation de l'addon

```bash
microk8s enable hostpath-storage
```

**Ce qui est créé** :
```bash
# Vérifier
microk8s kubectl get storageclass
```

**Résultat** :
```
NAME                          PROVISIONER            RECLAIMPOLICY   VOLUMEBINDINGMODE   ALLOWVOLUMEEXPANSION
microk8s-hostpath (default)   microk8s.io/hostpath   Delete          Immediate           true
```

### Configuration personnalisée

Le stockage par défaut est dans :
```
/var/snap/microk8s/common/default-storage
```

**Pour changer le chemin** :

1. Créer votre propre StorageClass :
```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: custom-hostpath
  annotations:
    storageclass.kubernetes.io/is-default-class: "false"
provisioner: microk8s.io/hostpath
parameters:
  pvDir: /mnt/custom-storage    # Votre chemin personnalisé
reclaimPolicy: Delete
allowVolumeExpansion: true
volumeBindingMode: Immediate
```

2. Créer le répertoire :
```bash
sudo mkdir -p /mnt/custom-storage
sudo chmod 777 /mnt/custom-storage
```

3. Utiliser cette classe dans vos PVC :
```yaml
spec:
  storageClassName: custom-hostpath
```

## Commandes utiles

### Gestion des StorageClasses

```bash
# Lister toutes les StorageClasses
microk8s kubectl get storageclass
microk8s kubectl get sc    # Abréviation

# Détails d'une StorageClass
microk8s kubectl describe storageclass <nom>

# Format YAML
microk8s kubectl get storageclass <nom> -o yaml

# Trouver la classe par défaut
microk8s kubectl get storageclass -o jsonpath='{.items[?(@.metadata.annotations.storageclass\.kubernetes\.io/is-default-class=="true")].metadata.name}'
```

### Inspection des PVC utilisant une classe

```bash
# PVC utilisant une classe spécifique
microk8s kubectl get pvc --all-namespaces -o json | \
  jq -r '.items[] | select(.spec.storageClassName=="<nom>") | "\(.metadata.namespace)/\(.metadata.name)"'

# Compter les PVC par StorageClass
microk8s kubectl get pvc --all-namespaces -o json | \
  jq -r '.items | group_by(.spec.storageClassName) | .[] | "\(.[0].spec.storageClassName): \(length)"'
```

### Tests rapides

```bash
# Test de provisionnement
cat <<EOF | microk8s kubectl apply -f -
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: test-provision
spec:
  storageClassName: microk8s-hostpath
  accessModes: [ReadWriteOnce]
  resources:
    requests:
      storage: 1Gi
EOF

# Vérifier
microk8s kubectl get pvc test-provision

# Nettoyer
microk8s kubectl delete pvc test-provision
```

## Résumé visuel : Flux complet

```
┌─────────────────────────────────────────────────────────┐
│              CONFIGURATION (Une fois)                   │
│                                                         │
│  Admin crée StorageClass                                │
│  ┌───────────────────────────────┐                      │
│  │ apiVersion: storage.k8s.io/v1 │                      │
│  │ kind: StorageClass            │                      │
│  │ metadata:                     │                      │
│  │   name: fast                  │                      │
│  │ provisioner: xxx              │                      │
│  │ reclaimPolicy: Delete         │                      │
│  └──────────────┬────────────────┘                      │
│                 │                                       │
└─────────────────┼───────────────────────────────────────┘
                  │
                  │ Disponible pour utilisation
                  │
┌─────────────────▼────────────────────────────────────────┐
│           UTILISATION (À chaque besoin)                  │
│                                                          │
│  Utilisateur crée PVC                                    │
│  ┌──────────────────────────────┐                        │
│  │ storageClassName: fast       │                        │
│  │ size: 10Gi                   │                        │
│  └──────────────┬───────────────┘                        │
│                 │                                        │
│                 ▼                                        │
│  Kubernetes détecte → Appelle Provisioner                │
│                 │                                        │
│                 ▼                                        │
│  Provisioner crée PV automatiquement                     │
│  ┌──────────────────────────────┐                        │
│  │ PersistentVolume (10Gi)      │                        │
│  └──────────────┬───────────────┘                        │
│                 │                                        │
│                 ▼                                        │
│  Kubernetes lie PVC ← → PV                               │
│                 │                                        │
│                 ▼                                        │
│  Pod peut utiliser le PVC                                │
│                                                          │
└──────────────────────────────────────────────────────────┘
```

## Prochaines étapes

Maintenant que vous maîtrisez les StorageClasses, vous êtes prêt pour :

- **Section 6.6** : Gestion des volumes persistants - Techniques avancées de gestion
- **Section 6.7** : StatefulSets - Applications stateful avec stockage persistant
- **Section 6.8** : Exemples pratiques - Déploiement complet d'applications avec stockage

## Conclusion

Les StorageClasses sont le pilier du provisionnement dynamique dans Kubernetes. Elles permettent de :
- Automatiser la création de volumes
- Offrir différents types de stockage aux utilisateurs
- Simplifier la gestion du stockage
- Rendre les applications plus portables

Avec MicroK8s, tout est déjà configuré grâce à l'addon `hostpath-storage`, vous permettant de vous concentrer sur vos applications plutôt que sur la gestion du stockage.

**Les points essentiels** :

- Les StorageClasses définissent des "types" de stockage
- Elles permettent le provisionnement dynamique automatique
- Le provisioner est le composant qui crée physiquement le stockage
- Les paramètres sont spécifiques à chaque provisioner
- Il est recommandé d'avoir une classe par défaut
- `Retain` est conseillé en production pour les données critiques
- `allowVolumeExpansion` devrait généralement être activé
- `WaitForFirstConsumer` est recommandé pour les clusters multi-nœuds

---

**Points clés à retenir** :

✅ **StorageClass** = Définit un type de stockage avec provisionnement automatique
✅ **Provisioner** = Composant qui crée physiquement le stockage
✅ **Provisionnement dynamique** = PV créés automatiquement à la demande
✅ **Classe par défaut** = Utilisée quand aucune classe n'est spécifiée
✅ **reclaimPolicy** = Delete (dev) ou Retain (production)
✅ **allowVolumeExpansion** = Permet d'augmenter la taille des volumes
✅ **volumeBindingMode** = Immediate ou WaitForFirstConsumer
✅ **MicroK8s** = Addon hostpath-storage configure tout automatiquement

⏭️ [Gestion des volumes persistants](/06-stockage-persistant/06-gestion-des-volumes-persistants.md)
