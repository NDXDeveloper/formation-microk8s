🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 6.4 PersistentVolumeClaims (PVC)

## Introduction

Dans la section précédente, nous avons exploré les PersistentVolumes (PV), qui représentent le stockage physique disponible dans un cluster Kubernetes. Maintenant, découvrons les **PersistentVolumeClaims (PVC)**, qui sont la manière dont les applications demandent et consomment ce stockage.

Un PVC est essentiellement une "demande" ou une "réclamation" de stockage. C'est le lien entre ce que les utilisateurs veulent (une certaine quantité de stockage avec certaines caractéristiques) et ce qui est disponible (les PV).

## Qu'est-ce qu'un PersistentVolumeClaim ?

### Définition

Un **PersistentVolumeClaim (PVC)** est :
- Une demande de stockage faite par un utilisateur ou une application
- Une ressource Kubernetes qui réserve un PersistentVolume
- Le moyen d'abstraction qui permet aux développeurs de ne pas se soucier des détails du stockage
- Une ressource namespaced (contrairement aux PV qui sont au niveau cluster)

**Analogie** : Si un PersistentVolume est un appartement disponible à la location, un PersistentVolumeClaim est la demande de location que vous faites. Vous spécifiez ce dont vous avez besoin (taille, caractéristiques), et le système trouve un appartement correspondant.

### La séparation des responsabilités

C'est l'un des concepts clés de Kubernetes :

```
┌────────────────────────────────────────────────────────┐
│                  ADMINISTRATEUR                        │
│                                                        │
│  • Configure le stockage physique                      │
│  • Crée les PersistentVolumes (ou configure            │
│    le provisionnement dynamique)                       │
│  • Définit les StorageClasses                          │
│  • Gère l'infrastructure de stockage                   │
│                                                        │
│           ┌──────────────────┐                         │
│           │ PersistentVolume │                         │
│           │   (Offre)        │                         │
│           └────────┬─────────┘                         │
│                    │                                   │
│                    │  Binding                          │
│                    │  (Kubernetes fait le lien)        │
│                    │                                   │
│           ┌────────▼─────────┐                         │
│           │ PersistentVolume │                         │
│           │      Claim       │                         │
│           │   (Demande)      │                         │
│           └────────┬─────────┘                         │
│                    │                                   │
│                  UTILISATEUR / DÉVELOPPEUR             │
│                                                        │
│  • Crée des PVC avec les besoins de l'application      │
│  • Utilise les PVC dans les pods                       │
│  • Ne se soucie pas de l'implémentation du stockage    │
│                                                        │
└────────────────────────────────────────────────────────┘
```

**Avantages de cette séparation** :
- Les développeurs n'ont pas besoin de connaître les détails du stockage
- Le code des applications reste portable
- L'administrateur garde le contrôle du stockage
- Changement d'infrastructure plus facile

## Comment fonctionne le Binding ?

Le **binding** (liaison) est le processus par lequel Kubernetes associe un PVC à un PV.

### Processus de binding

```
1. Création du PVC
   │
   ▼
2. Kubernetes recherche un PV disponible qui correspond
   │
   ├─ Critères de correspondance :
   │  • StorageClass identique (ou défaut)
   │  • Capacité suffisante (PV >= PVC)
   │  • Modes d'accès compatibles
   │  • Sélecteurs respectés
   │
   ▼
3. Si PV trouvé → BINDING
   │
   ├─ PV : Status = Bound
   └─ PVC : Status = Bound
   │
   ▼
4. Si aucun PV trouvé
   │
   ├─ Provisionnement dynamique ?
   │  ├─ OUI → Création automatique d'un PV
   │  └─ NON → PVC reste en Pending
   │
   ▼
5. Le PVC peut maintenant être utilisé dans un Pod
```

**Points importants** :
- Le binding est **exclusif** : un PV ne peut être lié qu'à un seul PVC
- Le binding est **bidirectionnel** : PVC ↔ PV
- Une fois lié, le PVC "possède" le PV jusqu'à sa suppression

## Anatomie d'un PersistentVolumeClaim

Examinons la structure complète d'un manifeste PVC :

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: mon-pvc
  namespace: default
  labels:
    app: mon-app
  annotations:
    description: "Stockage pour ma base de données"
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 10Gi
  storageClassName: fast-storage
  selector:
    matchLabels:
      type: ssd
  volumeMode: Filesystem
```

Décortiquons chaque section :

### metadata : Identification du PVC

```yaml
metadata:
  name: mon-pvc              # Nom unique dans le namespace
  namespace: default         # Namespace (contrairement aux PV)
  labels:                    # Labels pour organisation
    app: mon-app
    environment: production
```

**Points clés** :
- Le nom doit être unique **dans le namespace**
- Les PVC sont des ressources namespacées (contrairement aux PV)
- Les labels facilitent la gestion et le filtrage

### accessModes : Modes d'accès demandés

```yaml
spec:
  accessModes:
    - ReadWriteOnce
```

Le PVC demande des modes d'accès spécifiques. Les mêmes que pour les PV :

| Mode | Abréviation | Description |
|------|-------------|-------------|
| ReadWriteOnce | RWO | Lecture-écriture par un seul nœud |
| ReadOnlyMany | ROX | Lecture seule par plusieurs nœuds |
| ReadWriteMany | RWX | Lecture-écriture par plusieurs nœuds |
| ReadWriteOncePod | RWOP | Lecture-écriture par un seul pod |

**Important** : Le PV lié doit supporter au moins un des modes demandés.

#### Demander plusieurs modes

Vous pouvez spécifier plusieurs modes, et Kubernetes choisira un PV qui en supporte au moins un :

```yaml
accessModes:
  - ReadWriteOnce
  - ReadOnlyMany
```

### resources.requests : Quantité de stockage demandée

```yaml
spec:
  resources:
    requests:
      storage: 10Gi
```

**Points importants** :
- C'est la taille minimale dont vous avez besoin
- Le PV lié doit avoir **au moins** cette capacité
- Si le PV a plus, tant mieux (vous pourrez utiliser la capacité totale)
- Vous ne pouvez pas demander moins après création

**Unités de mesure** :
```yaml
storage: 1Ki    # 1024 bytes
storage: 1Mi    # 1024 Ki = 1 048 576 bytes
storage: 1Gi    # 1024 Mi = 1 073 741 824 bytes
storage: 1Ti    # 1024 Gi
```

#### Exemple de correspondance

```yaml
# PVC demande
resources:
  requests:
    storage: 5Gi

# PV disponible (10Gi) → ✅ Match
# Le PVC obtiendra les 10Gi complets
```

### storageClassName : Classe de stockage

```yaml
spec:
  storageClassName: fast-storage
```

La StorageClass détermine :
- Quel type de stockage sera utilisé
- Comment le PV sera provisionné (dynamiquement ou non)
- Les caractéristiques du stockage (rapide, lent, répliqué, etc.)

**Cas spéciaux** :

```yaml
# Utiliser la StorageClass par défaut du cluster
storageClassName: ""    # Chaîne vide (pas omis)

# Laisser Kubernetes choisir automatiquement
# (Omis complètement)
spec:
  accessModes: [...]
  # pas de storageClassName
```

**Comportements** :

| Configuration | Comportement |
|---------------|--------------|
| `storageClassName: "ma-classe"` | Utilise cette classe spécifique |
| `storageClassName: ""` | PV sans classe uniquement |
| Champ omis | Utilise la classe par défaut |

### selector : Sélectionner un PV spécifique

Le `selector` permet de cibler des PV spécifiques via leurs labels :

```yaml
spec:
  selector:
    matchLabels:
      type: ssd
      environment: production
```

**Utilité** :
- Contrôle plus fin sur quel PV sera utilisé
- Peut cibler un PV spécifique préparé à l'avance
- Utile pour des besoins particuliers

#### Exemple complet de sélection

```yaml
# PV avec labels
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv-ssd-fast
  labels:
    type: ssd
    speed: fast
    tier: premium
spec:
  capacity:
    storage: 100Gi
  accessModes:
    - ReadWriteOnce
  storageClassName: premium

---
# PVC qui cible ce PV
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: pvc-fast-storage
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 50Gi
  storageClassName: premium
  selector:
    matchLabels:
      type: ssd
      speed: fast
```

#### matchExpressions : Sélection avancée

Pour des critères plus complexes :

```yaml
spec:
  selector:
    matchExpressions:
    - key: environment
      operator: In
      values:
      - production
      - staging
    - key: type
      operator: NotIn
      values:
      - test
```

**Opérateurs disponibles** :
- `In` : La valeur doit être dans la liste
- `NotIn` : La valeur ne doit pas être dans la liste
- `Exists` : La clé doit exister (peu importe la valeur)
- `DoesNotExist` : La clé ne doit pas exister

### volumeMode : Mode de volume

```yaml
spec:
  volumeMode: Filesystem    # ou Block
```

Doit correspondre au `volumeMode` du PV :
- `Filesystem` (défaut) : Système de fichiers monté
- `Block` : Périphérique bloc brut

### dataSource : Cloner ou restaurer

Fonctionnalité avancée pour créer un PVC depuis une source existante :

```yaml
spec:
  dataSource:
    name: mon-pvc-source
    kind: PersistentVolumeClaim
```

**Cas d'usage** :
- Cloner un PVC existant
- Restaurer depuis un snapshot
- Créer des copies pour tests

## Exemples complets de PVC

### Exemple 1 : PVC simple et basique

Le plus simple possible, pour débuter :

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: pvc-simple
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 5Gi
```

**Ce qui se passe** :
- Demande 5 Gi de stockage
- Mode RWO (un seul nœud)
- Utilise la StorageClass par défaut
- Kubernetes trouvera automatiquement un PV compatible

### Exemple 2 : PVC avec classe de stockage spécifique

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: pvc-database
  namespace: production
  labels:
    app: postgresql
    tier: database
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 20Gi
  storageClassName: fast-ssd
```

**Explications** :
- Demande 20 Gi
- Classe "fast-ssd" pour des performances élevées
- Labels pour identifier facilement
- Namespace production

### Exemple 3 : PVC avec sélecteur

Pour cibler un PV spécifique :

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: pvc-selective
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 10Gi
  storageClassName: manual
  selector:
    matchLabels:
      disk-type: ssd
      availability-zone: us-east-1a
```

**Utilité** :
- Garantit que le stockage sera sur un SSD
- Garantit la zone de disponibilité
- Utile pour optimisation des performances ou contraintes géographiques

### Exemple 4 : PVC pour stockage partagé

Pour plusieurs pods qui accèdent au même stockage :

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: pvc-shared-files
spec:
  accessModes:
    - ReadWriteMany      # Support multi-nœuds
  resources:
    requests:
      storage: 50Gi
  storageClassName: nfs-storage
```

**Note** : ReadWriteMany nécessite un type de stockage qui le supporte (NFS, CephFS, etc.)

### Exemple 5 : PVC avec annotations

Pour documenter et ajouter des métadonnées :

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: pvc-documented
  annotations:
    description: "Stockage pour les uploads utilisateurs"
    owner: "team-backend@example.com"
    backup-policy: "daily"
    retention-days: "30"
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 100Gi
  storageClassName: standard
```

## Utiliser un PVC dans un Pod

### Méthode simple

Une fois le PVC créé, vous pouvez l'utiliser dans un pod :

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-with-pvc
spec:
  containers:
  - name: app
    image: nginx:1.21
    volumeMounts:
    - name: persistent-storage    # Nom du volume dans le pod
      mountPath: /usr/share/nginx/html

  volumes:
  - name: persistent-storage      # Définition du volume
    persistentVolumeClaim:
      claimName: pvc-simple       # Référence au PVC
```

**Processus** :
1. Le PVC doit exister et être `Bound`
2. Le pod référence le PVC par son nom
3. Le volume est monté au chemin spécifié
4. Les données écrites dans `/usr/share/nginx/html` sont persistantes

### Avec un Deployment

Plus courant en production :

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: app-data
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
  name: app-deployment
spec:
  replicas: 1    # Important : RWO = 1 seul replica
  selector:
    matchLabels:
      app: myapp
  template:
    metadata:
      labels:
        app: myapp
    spec:
      containers:
      - name: app
        image: myapp:1.0
        volumeMounts:
        - name: data
          mountPath: /data
      volumes:
      - name: data
        persistentVolumeClaim:
          claimName: app-data
```

**⚠️ Attention avec les Deployments et RWO** :
- Si `accessMode: ReadWriteOnce` et `replicas > 1`, les pods peuvent échouer
- Solution : Utiliser `ReadWriteMany` ou un StatefulSet

### Avec un StatefulSet

Pour des applications avec état :

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: mysql
spec:
  serviceName: mysql
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
        image: mysql:8.0
        volumeMounts:
        - name: data
          mountPath: /var/lib/mysql
  volumeClaimTemplates:    # Crée automatiquement un PVC par replica
  - metadata:
      name: data
    spec:
      accessModes:
        - ReadWriteOnce
      resources:
        requests:
          storage: 20Gi
      storageClassName: fast
```

**Avantage** : Chaque replica obtient son propre PVC automatiquement !

## Gestion des PVC

### Créer un PVC

```bash
# Depuis un fichier YAML
microk8s kubectl apply -f mon-pvc.yaml

# Vérifier la création
microk8s kubectl get pvc
```

**Résultat attendu** :
```
NAME        STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS
mon-pvc     Bound    pvc-12345678-1234-1234-1234-123456789012  5Gi        RWO            microk8s-hostpath
```

**Statuts possibles** :
- `Pending` : En attente de binding avec un PV
- `Bound` : Lié à un PV, prêt à être utilisé
- `Lost` : Le PV lié a été supprimé (rare)

### Lister les PVC

```bash
# Liste simple
microk8s kubectl get pvc

# Dans tous les namespaces
microk8s kubectl get pvc --all-namespaces

# Avec plus de détails
microk8s kubectl get pvc -o wide

# Format personnalisé
microk8s kubectl get pvc -o custom-columns=NAME:.metadata.name,STATUS:.status.phase,VOLUME:.spec.volumeName,CAPACITY:.status.capacity.storage
```

### Voir les détails d'un PVC

```bash
microk8s kubectl describe pvc mon-pvc
```

**Informations importantes** :
- **Status** : État actuel
- **Volume** : PV lié (si bound)
- **Capacity** : Capacité réellement obtenue
- **Access Modes** : Modes d'accès
- **StorageClass** : Classe utilisée
- **Events** : Événements récents

**Exemple de sortie** :
```
Name:          mon-pvc
Namespace:     default
StorageClass:  microk8s-hostpath
Status:        Bound
Volume:        pvc-abc123...
Labels:        <none>
Capacity:      5Gi
Access Modes:  RWO
Events:
  Type    Reason                 Age   Message
  ----    ------                 ----  -------
  Normal  ProvisioningSucceeded  2m    Successfully provisioned volume
```

### Modifier un PVC

⚠️ **Limitations** : La plupart des champs ne peuvent pas être modifiés après création :
- `accessModes` : Immuable
- `storageClassName` : Immuable
- `selector` : Immuable
- `resources.requests.storage` : Ne peut qu'augmenter (avec expansion activée)

**Modifications possibles** :
- Labels et annotations
- Augmentation de la capacité (si supporté)

### Expansion de volume

Si la StorageClass le permet, vous pouvez augmenter la taille :

```yaml
# PVC original
spec:
  resources:
    requests:
      storage: 10Gi

# Modification
spec:
  resources:
    requests:
      storage: 20Gi    # Augmentation
```

**Commande** :
```bash
microk8s kubectl edit pvc mon-pvc
# Modifier la valeur storage
```

**Conditions requises** :
- La StorageClass doit avoir `allowVolumeExpansion: true`
- Le type de stockage doit supporter l'expansion
- On peut seulement **augmenter**, jamais réduire

**Vérifier si l'expansion est supportée** :
```bash
microk8s kubectl get storageclass
```

Cherchez la colonne `ALLOWVOLUMEEXPANSION`.

### Supprimer un PVC

```bash
microk8s kubectl delete pvc mon-pvc
```

**Ce qui se passe** :
1. Le PVC est marqué pour suppression
2. Si un pod utilise le PVC, la suppression attend
3. Une fois libéré, le PVC est supprimé
4. Le PV associé suit sa `reclaimPolicy` :
   - `Retain` : Le PV passe en "Released", données conservées
   - `Delete` : Le PV et le stockage sont supprimés

**Forcer la suppression** (avec précaution) :
```bash
microk8s kubectl delete pvc mon-pvc --force --grace-period=0
```

⚠️ **Attention** : Peut causer des problèmes si un pod utilise encore le PVC !

## États et cycle de vie d'un PVC

### Diagramme du cycle de vie

```
┌─────────────────────────────────────────────────────┐
│                   PVC créé                          │
│                      │                              │
│                      ▼                              │
│              ┌──────────────┐                       │
│              │   PENDING    │                       │
│              │              │                       │
│              │ Recherche un │                       │
│              │ PV compatible│                       │
│              └──────┬───────┘                       │
│                     │                               │
│        ┌────────────┴─────────────┐                 │
│        │                          │                 │
│        ▼                          ▼                 │
│  PV trouvé                   Pas de PV              │
│  ou créé                     disponible             │
│        │                          │                 │
│        │                          │                 │
│        ▼                          ▼                 │
│  ┌──────────┐            Reste en PENDING           │
│  │  BOUND   │            (Provisionnement           │
│  │          │             dynamique tente           │
│  │ PVC prêt │             de créer un PV)           │
│  └────┬─────┘                                       │
│       │                                             │
│       │ PVC utilisé par des pods                    │
│       │                                             │
│       ▼                                             │
│  Suppression du PVC                                 │
│       │                                             │
│       ▼                                             │
│  PV suit sa reclaimPolicy                           │
│  • Retain → PV Released                             │
│  • Delete → PV supprimé                             │
│                                                     │
└─────────────────────────────────────────────────────┘
```

### État PENDING : Pourquoi et que faire ?

**Causes courantes** :

1. **Aucun PV disponible**
   ```bash
   microk8s kubectl get pv
   # Vérifier s'il y a des PV "Available"
   ```

2. **Aucun PV ne correspond aux critères**
   - Capacité insuffisante
   - Modes d'accès incompatibles
   - StorageClass différente
   - Sélecteurs non satisfaits

3. **Provisionnement dynamique en cours**
   - Normal, attendez quelques secondes

4. **Provisionnement dynamique échoue**
   ```bash
   microk8s kubectl describe pvc mon-pvc
   # Regarder la section Events
   ```

**Solutions** :
```bash
# Vérifier les événements
microk8s kubectl describe pvc mon-pvc

# Vérifier la StorageClass
microk8s kubectl get storageclass

# Vérifier les PV disponibles
microk8s kubectl get pv

# Pour MicroK8s, vérifier l'addon
microk8s status
# S'assurer que hostpath-storage est activé
```

## Protection et finalizers

### Protection contre la suppression

Kubernetes protège les PVC utilisés contre la suppression :

```bash
# Tenter de supprimer un PVC utilisé
microk8s kubectl delete pvc mon-pvc
# Le PVC reste en état "Terminating" jusqu'à ce qu'aucun pod ne l'utilise
```

**Vérifier quels pods utilisent un PVC** :
```bash
microk8s kubectl get pods -o json | jq -r '.items[] | select(.spec.volumes[]?.persistentVolumeClaim.claimName=="mon-pvc") | .metadata.name'
```

### Finalizers

Les finalizers sont des mécanismes de protection :

```yaml
metadata:
  finalizers:
  - kubernetes.io/pvc-protection
```

**Rôle** : Empêche la suppression du PVC tant qu'il est utilisé par un pod.

**Supprimer les finalizers** (en dernier recours) :
```bash
microk8s kubectl patch pvc mon-pvc -p '{"metadata":{"finalizers":null}}'
```

⚠️ **Danger** : À utiliser uniquement si vous êtes certain que le PVC n'est plus utilisé !

## Bonnes pratiques

### 1. Nommage cohérent et descriptif

```yaml
# Bon
metadata:
  name: postgres-data-prod
  name: webapp-uploads-staging
  name: redis-cache-dev

# À éviter
metadata:
  name: pvc1
  name: data
  name: storage
```

**Pattern recommandé** : `<app>-<usage>-<environnement>`

### 2. Utiliser des labels

```yaml
metadata:
  name: mysql-data
  labels:
    app: mysql
    component: database
    environment: production
    managed-by: helm
```

**Avantages** :
- Facilite le filtrage et la recherche
- Permet l'automatisation
- Améliore la traçabilité

### 3. Documenter avec des annotations

```yaml
metadata:
  annotations:
    description: "Base de données principale de production"
    backup-schedule: "0 2 * * *"
    owner: "team-data@example.com"
    cost-center: "engineering"
```

### 4. Demander la bonne quantité

```yaml
# Prévoyez de la marge pour la croissance
resources:
  requests:
    storage: 50Gi    # Si vous estimez 30Gi, demandez plus
```

**Conseils** :
- Estimez vos besoins à 6-12 mois
- Ajoutez 30-50% de marge
- Surveillez l'utilisation réelle

### 5. Choisir le bon mode d'accès

```yaml
# Pour une base de données (1 pod)
accessModes:
  - ReadWriteOnce

# Pour des fichiers statiques (plusieurs pods en lecture)
accessModes:
  - ReadOnlyMany

# Pour du partage actif (plusieurs pods en écriture)
accessModes:
  - ReadWriteMany    # Nécessite un stockage approprié
```

### 6. Utiliser le provisionnement dynamique

```yaml
# Préférez ceci (provisionnement dynamique)
spec:
  storageClassName: fast

# Plutôt que ceci (provisionnement statique)
spec:
  storageClassName: ""
  selector:
    matchLabels:
      specific-pv: "true"
```

**Sauf si** : Vous avez vraiment besoin d'un contrôle précis sur le PV utilisé.

### 7. Gérer par namespace

Organisez vos PVC par namespace pour une meilleure isolation :

```yaml
# Production
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: app-data
  namespace: production

---
# Staging
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: app-data
  namespace: staging
```

## Erreurs courantes et solutions

### Erreur 1 : PVC reste en PENDING indéfiniment

**Symptôme** :
```bash
$ microk8s kubectl get pvc
NAME      STATUS    VOLUME   CAPACITY   ACCESS MODES   STORAGECLASS
mon-pvc   Pending                                      fast-ssd
```

**Diagnostic** :
```bash
microk8s kubectl describe pvc mon-pvc
```

**Causes et solutions** :

#### a) Aucun PV disponible
```
Events:
  Type     Reason              Message
  ----     ------              -------
  Warning  ProvisioningFailed  no persistent volumes available
```

**Solution** : Créer un PV manuellement ou activer le provisionnement dynamique.

#### b) StorageClass inexistante
```
Events:
  Warning  ProvisioningFailed  storageclass "fast-ssd" not found
```

**Solution** :
```bash
# Vérifier les StorageClasses disponibles
microk8s kubectl get storageclass

# Corriger le PVC
microk8s kubectl edit pvc mon-pvc
# Changer storageClassName vers une classe existante
```

#### c) Provisionneur manquant
```
Events:
  Warning  ProvisioningFailed  Failed to provision volume: no provisioner found
```

**Solution pour MicroK8s** :
```bash
microk8s enable hostpath-storage
```

### Erreur 2 : PVC demande trop de stockage

**Symptôme** :
```bash
$ microk8s kubectl get pvc
NAME      STATUS    VOLUME   CAPACITY   ACCESS MODES   STORAGECLASS
mon-pvc   Pending                                      standard
```

**Events** :
```
Warning  ProvisioningFailed  Failed to find PV with sufficient capacity
```

**Solution** :
```yaml
# Réduire la demande ou créer un PV plus grand
spec:
  resources:
    requests:
      storage: 5Gi    # Au lieu de 100Gi
```

### Erreur 3 : Modes d'accès incompatibles

**Symptôme** : Le PVC reste Pending malgré des PV disponibles.

**Cause** :
```yaml
# PVC demande
accessModes:
  - ReadWriteMany

# Mais les PV disponibles n'offrent que
accessModes:
  - ReadWriteOnce
```

**Solution** : Adapter les modes d'accès ou utiliser un type de stockage supportant RWX.

### Erreur 4 : Impossible de supprimer un PVC

**Symptôme** :
```bash
$ microk8s kubectl delete pvc mon-pvc
persistentvolumeclaim "mon-pvc" deleted
# Mais la commande ne se termine pas et le PVC reste
```

**Cause** : Un pod utilise encore le PVC.

**Solution** :
```bash
# 1. Trouver les pods utilisant le PVC
microk8s kubectl get pods -o json | \
  jq -r '.items[] | select(.spec.volumes[]?.persistentVolumeClaim.claimName=="mon-pvc") | .metadata.name'

# 2. Supprimer ces pods d'abord
microk8s kubectl delete pod <nom-pod>

# 3. Le PVC sera alors supprimé automatiquement
```

### Erreur 5 : L'expansion de volume ne fonctionne pas

**Symptôme** : Vous augmentez la taille dans le PVC mais rien ne change.

**Causes possibles** :

1. **StorageClass ne supporte pas l'expansion**
```bash
microk8s kubectl get storageclass
# Vérifier ALLOWVOLUMEEXPANSION = true
```

2. **Le pod doit être redémarré**
Pour certains types de stockage, le pod doit être supprimé et recréé pour que l'expansion prenne effet.

```bash
# Supprimer le pod
microk8s kubectl delete pod <nom-pod>
# Le pod se recrée automatiquement (si géré par un Deployment)
```

### Erreur 6 : PVC lié au mauvais PV

**Symptôme** : Le PVC se lie à un PV non désiré.

**Cause** : Pas de sélecteur, Kubernetes choisit le premier PV compatible.

**Prévention** :
```yaml
spec:
  selector:
    matchLabels:
      environment: production
      type: fast
```

**Solution si déjà lié** :
```bash
# 1. Supprimer le PVC
microk8s kubectl delete pvc mon-pvc

# 2. Ajouter un sélecteur
# 3. Recréer le PVC
microk8s kubectl apply -f mon-pvc-avec-selector.yaml
```

## Monitoring et observation

### Vérifier l'utilisation du stockage

```bash
# Voir la capacité allouée
microk8s kubectl get pvc

# Pour voir l'utilisation réelle (nécessite metrics-server)
microk8s kubectl top pvc
```

### Voir les PVC par StorageClass

```bash
microk8s kubectl get pvc --all-namespaces -o custom-columns=\
NAME:.metadata.name,\
NAMESPACE:.metadata.namespace,\
STORAGECLASS:.spec.storageClassName,\
SIZE:.spec.resources.requests.storage
```

### Auditer tous les PVC

Script bash pratique :
```bash
#!/bin/bash
echo "PVC Status Report"
echo "================="
for ns in $(microk8s kubectl get ns -o jsonpath='{.items[*].metadata.name}'); do
  echo "Namespace: $ns"
  microk8s kubectl get pvc -n $ns -o custom-columns=\
NAME:.metadata.name,\
STATUS:.status.phase,\
VOLUME:.spec.volumeName,\
CAPACITY:.status.capacity.storage,\
STORAGECLASS:.spec.storageClassName
  echo ""
done
```

### Alerting

Points à surveiller :
- PVC en PENDING trop longtemps
- Utilisation proche de la capacité
- Échecs de provisionnement répétés

## Scénarios d'utilisation courants

### Scénario 1 : Base de données stateful

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: postgres-data
  labels:
    app: postgres
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 20Gi
  storageClassName: fast-ssd
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: postgres
spec:
  serviceName: postgres
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
        image: postgres:14
        env:
        - name: POSTGRES_DB
          value: mydb
        - name: POSTGRES_PASSWORD
          valueFrom:
            secretKeyRef:
              name: postgres-secret
              key: password
        volumeMounts:
        - name: data
          mountPath: /var/lib/postgresql/data
      volumes:
      - name: data
        persistentVolumeClaim:
          claimName: postgres-data
```

### Scénario 2 : Application web avec uploads

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: webapp-uploads
spec:
  accessModes:
    - ReadWriteMany    # Plusieurs pods peuvent écrire
  resources:
    requests:
      storage: 50Gi
  storageClassName: nfs-storage
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: webapp
spec:
  replicas: 3    # Plusieurs replicas possibles avec RWX
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
        image: mywebapp:1.0
        volumeMounts:
        - name: uploads
          mountPath: /app/uploads
      volumes:
      - name: uploads
        persistentVolumeClaim:
          claimName: webapp-uploads
```

### Scénario 3 : Cache Redis

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: redis-data
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 5Gi
  storageClassName: standard
---
apiVersion: v1
kind: Pod
metadata:
  name: redis
spec:
  containers:
  - name: redis
    image: redis:7
    volumeMounts:
    - name: data
      mountPath: /data
  volumes:
  - name: data
    persistentVolumeClaim:
      claimName: redis-data
```

## Résumé visuel : PVC vs PV

```
┌──────────────────────────────────────────────────────────┐
│                    DIFFÉRENCES                           │
├──────────────────────────────────────────────────────────┤
│                                                          │
│  PersistentVolume (PV)    │  PersistentVolumeClaim (PVC) │
│  ───────────────────────  │  ────────── ───────────────  │
│                           │                              │
│  • Cluster-wide           │  • Namespacé                 │
│  • Créé par admin         │  • Créé par utilisateur      │
│  • Représente le stockage │  • Demande du stockage       │
│    physique               │                              │
│  • Offre de stockage      │  • Demande de stockage       │
│  • Indépendant des apps   │  • Utilisé par les apps      │
│                           │                              │
└──────────────────────────────────────────────────────────┘
```

## Commandes de diagnostic essentielles

```bash
# Liste des PVC
microk8s kubectl get pvc
microk8s kubectl get pvc --all-namespaces

# Détails d'un PVC
microk8s kubectl describe pvc <nom-pvc>

# Voir le PV lié
microk8s kubectl get pvc <nom-pvc> -o jsonpath='{.spec.volumeName}'

# Voir les événements
microk8s kubectl get events --field-selector involvedObject.name=<nom-pvc>

# Format YAML complet
microk8s kubectl get pvc <nom-pvc> -o yaml

# Voir l'utilisation (si metrics-server installé)
microk8s kubectl top pvc

# Trouver les pods utilisant un PVC
microk8s kubectl get pods -o json | \
  jq -r '.items[] | select(.spec.volumes[]?.persistentVolumeClaim.claimName=="<nom-pvc>") | .metadata.name'
```

## Prochaines étapes

Maintenant que vous maîtrisez les PersistentVolumeClaims, vous êtes prêt pour :

- **Section 6.5** : StorageClasses - Configuration du provisionnement dynamique en détail
- **Section 6.6** : Gestion des volumes persistants - Techniques avancées
- **Section 6.7** : StatefulSets - Applications stateful avec stockage persistant
- **Section 6.8** : Exemples pratiques - Déploiement de bases de données et applications

## Conclusion

Les PersistentVolumeClaims sont le moyen par lequel vos applications demandent et utilisent du stockage persistant dans Kubernetes. Ils fournissent une abstraction simple qui cache la complexité de l'infrastructure de stockage sous-jacente.

**Les points essentiels** :

- Les PVC sont des **demandes** de stockage (les PV sont l'offre)
- Ils sont **namespacés** (contrairement aux PV)
- Le **binding** est automatique selon des critères de correspondance
- Le provisionnement peut être **statique** ou **dynamique**
- Les PVC sont **protégés** contre la suppression tant qu'ils sont utilisés
- Avec MicroK8s, tout est simplifié grâce au provisionnement dynamique

La combinaison PV/PVC est l'une des abstractions les plus puissantes de Kubernetes, permettant aux développeurs de se concentrer sur leurs applications sans se soucier des détails du stockage physique.

---

**Points clés à retenir** :

✅ **PVC** = Demande de stockage par l'utilisateur
✅ **Namespacé** = Isolé par namespace
✅ **Binding automatique** = Kubernetes trouve le PV correspondant
✅ **Provisionnement dynamique** = Création automatique de PV à la demande
✅ **Protection** = Impossible de supprimer un PVC utilisé
✅ **Expansion** = Possibilité d'augmenter la taille (si supporté)
✅ **Simple avec MicroK8s** = Addon hostpath-storage pour le provisionnement dynamique
✅ **Labels et annotations** = Essentiels pour une bonne gestion

⏭️ [StorageClasses](/06-stockage-persistant/05-storageclasses.md)
