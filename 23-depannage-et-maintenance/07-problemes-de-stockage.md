🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 23.7 Problèmes de Stockage

## Introduction

Le stockage persistant est l'un des aspects les plus importants et parfois les plus complexes de Kubernetes. Contrairement aux conteneurs qui sont éphémères (temporaires), le stockage persistant permet de conserver les données même après le redémarrage ou la suppression d'un pod. Les problèmes de stockage peuvent entraîner la perte de données, des pods qui ne démarrent pas, ou des applications qui ne fonctionnent pas correctement.

> **Pour les débutants** : Imaginez un conteneur comme un ordinateur qui perd toute sa mémoire quand on l'éteint. Le stockage persistant, c'est comme un disque dur externe qui conserve vos fichiers même quand l'ordinateur est éteint. Dans Kubernetes, gérer ce "disque dur externe" peut parfois poser des problèmes, et cette section vous aidera à les résoudre.

## Comprendre le Stockage Kubernetes (Simplifié)

Avant de résoudre les problèmes, comprenons les concepts de base.

### Les Différents Types de Stockage

#### 1. Volumes Éphémères (EmptyDir)

**Définition** : Un volume temporaire qui existe tant que le pod existe.

**Caractéristiques** :
- Créé quand le pod démarre
- Supprimé quand le pod est supprimé
- Partagé entre tous les conteneurs du pod
- Stocké sur le nœud local

**Utilisation** : Cache temporaire, fichiers de travail

**Exemple** :
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: mon-pod
spec:
  containers:
  - name: mon-app
    image: nginx
    volumeMounts:
    - name: cache
      mountPath: /tmp/cache
  volumes:
  - name: cache
    emptyDir: {}
```

**Quand l'utiliser** : Données temporaires qui n'ont pas besoin de survivre au pod.

#### 2. Volumes Hôte (HostPath)

**Définition** : Monte un répertoire du nœud directement dans le pod.

**Caractéristiques** :
- Accès direct au système de fichiers du nœud
- Persiste après la suppression du pod
- Lié au nœud spécifique
- Risque de sécurité si mal utilisé

**Exemple** :
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: mon-pod
spec:
  containers:
  - name: mon-app
    image: nginx
    volumeMounts:
    - name: data
      mountPath: /data
  volumes:
  - name: data
    hostPath:
      path: /mnt/data  # Chemin sur le nœud
      type: DirectoryOrCreate
```

**Quand l'utiliser** : Développement local, accès à des fichiers système spécifiques.

**Attention** : Si le pod est reschedulé sur un autre nœud, il ne verra pas les mêmes données !

#### 3. PersistentVolumes (PV) et PersistentVolumeClaims (PVC)

**Définition** : Le mécanisme recommandé pour le stockage persistant.

**Comment ça fonctionne** :

1. **PersistentVolume (PV)** : Représente un morceau de stockage dans le cluster
2. **PersistentVolumeClaim (PVC)** : Une demande de stockage par un utilisateur
3. **StorageClass** : Définit différents types de stockage disponibles

**Analogie** :
- **PV** = Un appartement disponible
- **PVC** = Une demande de location d'appartement
- **StorageClass** = Type d'appartement (studio, T2, T3, etc.)

**Workflow** :
```
1. Admin crée un PV (ou StorageClass pour provisionnement dynamique)
   ↓
2. Utilisateur crée un PVC demandant du stockage
   ↓
3. Kubernetes lie (bind) le PVC à un PV approprié
   ↓
4. Pod utilise le PVC
```

### MicroK8s et le Stockage

MicroK8s fournit un addon de stockage simple :

```bash
# Activer le stockage hostpath
microk8s enable hostpath-storage
```

Cela installe :
- Un **StorageClass** appelé `microk8s-hostpath`
- Un provisioner qui crée automatiquement des PV sur demande
- Stockage basé sur le système de fichiers local du nœud

## Problème 1 : PVC Bloqué en Pending

### Symptômes

Le PersistentVolumeClaim reste indéfiniment en état `Pending`.

```bash
microk8s kubectl get pvc
# NAME        STATUS    VOLUME   CAPACITY   ACCESS MODES   STORAGECLASS
# mon-pvc     Pending                                      microk8s-hostpath
```

Le pod qui utilise ce PVC ne peut pas démarrer.

### Diagnostic

#### Étape 1 : Examiner le PVC

```bash
# Voir les détails du PVC
microk8s kubectl describe pvc mon-pvc
```

**Regarder la section Events** :
```
Events:
  Type     Reason              Message
  ----     ------              -------
  Warning  ProvisioningFailed  Failed to provision volume: timeout...
```

**Messages courants** :
- `waiting for a volume to be created`
- `no persistent volumes available`
- `storageclass not found`

#### Étape 2 : Vérifier la StorageClass

```bash
# Lister les StorageClasses
microk8s kubectl get storageclass

# Détails
microk8s kubectl describe storageclass microk8s-hostpath
```

**Vérifications** :
- La StorageClass existe-t-elle ?
- Est-elle marquée comme `(default)` si aucune n'est spécifiée dans le PVC ?

#### Étape 3 : Vérifier les PersistentVolumes

```bash
# Lister les PV
microk8s kubectl get pv

# Voir si des PV sont Available (non utilisés)
microk8s kubectl get pv | grep Available
```

#### Étape 4 : Vérifier le Provisioner

```bash
# Le provisioner hostpath-storage est-il Running ?
microk8s kubectl get pods -n kube-system | grep hostpath

# Logs du provisioner
microk8s kubectl logs -n kube-system -l k8s-app=hostpath-provisioner
```

### Solutions Courantes

#### Solution 1 : Activer l'Addon Storage

Si l'addon n'est pas activé :

```bash
# Activer hostpath-storage
microk8s enable hostpath-storage

# Vérifier qu'il est actif
microk8s status | grep hostpath-storage

# Attendre que le provisioner soit prêt
microk8s kubectl wait --for=condition=ready pod -n kube-system -l k8s-app=hostpath-provisioner --timeout=60s
```

#### Solution 2 : Corriger la StorageClass dans le PVC

Si le PVC référence une StorageClass inexistante :

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: mon-pvc
spec:
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
  storageClassName: microk8s-hostpath  # Assurez-vous que c'est correct
```

**Vérifier le nom exact** :
```bash
microk8s kubectl get storageclass
```

Appliquer :
```bash
microk8s kubectl apply -f pvc.yaml
```

#### Solution 3 : Définir une StorageClass par Défaut

Si votre PVC n'a pas de `storageClassName` spécifié :

```bash
# Vérifier quelle StorageClass est marquée par défaut
microk8s kubectl get storageclass

# Si aucune n'est par défaut, en définir une
microk8s kubectl patch storageclass microk8s-hostpath -p '{"metadata": {"annotations":{"storageclass.kubernetes.io/is-default-class":"true"}}}'
```

#### Solution 4 : Vérifier les Permissions du Répertoire

Le provisioner doit avoir accès au répertoire où il crée les volumes :

```bash
# Par défaut : /var/snap/microk8s/common/default-storage

# Vérifier les permissions
ls -ld /var/snap/microk8s/common/default-storage

# Si nécessaire, corriger
sudo chmod 755 /var/snap/microk8s/common/default-storage
```

#### Solution 5 : Redémarrer le Provisioner

```bash
# Supprimer le pod (il redémarrera automatiquement)
microk8s kubectl delete pod -n kube-system -l k8s-app=hostpath-provisioner

# Attendre qu'il redémarre
microk8s kubectl wait --for=condition=ready pod -n kube-system -l k8s-app=hostpath-provisioner --timeout=60s
```

#### Solution 6 : Créer Manuellement un PV (Provisionnement Statique)

Si le provisionnement dynamique ne fonctionne pas, créez un PV manuellement :

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: mon-pv-manuel
spec:
  capacity:
    storage: 1Gi
  accessModes:
  - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  storageClassName: ""  # Ou le nom de votre StorageClass
  hostPath:
    path: /mnt/data/mon-volume
    type: DirectoryOrCreate
```

Appliquer :
```bash
microk8s kubectl apply -f pv.yaml
```

Puis le PVC correspondant :
```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: mon-pvc
spec:
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
  storageClassName: ""  # Doit correspondre au PV
```

## Problème 2 : Pod Ne Peut Pas Monter le Volume

### Symptômes

Le pod est bloqué en état `ContainerCreating` ou `Pending`.

```bash
microk8s kubectl get pods
# NAME      READY   STATUS              RESTARTS   AGE
# mon-pod   0/1     ContainerCreating   0          5m
```

**Dans describe** :
```
Events:
  Warning  FailedMount  Unable to mount volumes...
  Warning  FailedMount  MountVolume.SetUp failed...
```

### Diagnostic

#### Étape 1 : Examiner le Pod

```bash
# Détails du pod
microk8s kubectl describe pod mon-pod
```

**Chercher dans Events** :
- `Unable to mount volumes`
- `MountVolume.SetUp failed`
- `Volume is already exclusively attached`
- `Permission denied`

#### Étape 2 : Vérifier le PVC

```bash
# Le PVC est-il Bound ?
microk8s kubectl get pvc

# Détails du PVC
microk8s kubectl describe pvc mon-pvc
```

**Le status doit être** : `Bound`

#### Étape 3 : Vérifier le PV

```bash
# Quel PV est lié au PVC ?
PV_NAME=$(microk8s kubectl get pvc mon-pvc -o jsonpath='{.spec.volumeName}')

# Détails du PV
microk8s kubectl describe pv $PV_NAME
```

#### Étape 4 : Vérifier le Répertoire sur le Nœud

Si vous utilisez hostPath ou hostpath-storage :

```bash
# Trouver le chemin du volume
PATH_VOLUME=$(microk8s kubectl get pv $PV_NAME -o jsonpath='{.spec.hostPath.path}')

# Vérifier que le répertoire existe
ls -ld $PATH_VOLUME

# Vérifier les permissions
stat $PATH_VOLUME
```

### Solutions Courantes

#### Solution 1 : Attendre que le PVC Soit Bound

Si le PVC est encore en Pending, le pod ne peut pas démarrer. Résolvez d'abord le problème du PVC (voir Problème 1).

#### Solution 2 : Corriger les AccessModes

**Problème** : Le PVC demande un AccessMode incompatible.

**AccessModes disponibles** :
- `ReadWriteOnce` (RWO) : Lecture-écriture par un seul nœud
- `ReadOnlyMany` (ROX) : Lecture seule par plusieurs nœuds
- `ReadWriteMany` (RWX) : Lecture-écriture par plusieurs nœuds

**hostpath-storage supporte seulement** : `ReadWriteOnce`

**Vérifier** :
```bash
microk8s kubectl get pvc mon-pvc -o jsonpath='{.spec.accessModes}'
```

**Corriger le PVC** :
```yaml
spec:
  accessModes:
  - ReadWriteOnce  # Pour hostpath-storage
```

#### Solution 3 : Volume Déjà Attaché à un Autre Pod

**Cause** : Avec `ReadWriteOnce`, un volume ne peut être attaché qu'à un seul nœud.

**Diagnostic** :
```bash
# Trouver tous les pods utilisant ce PVC
microk8s kubectl get pods --all-namespaces -o json | \
  jq -r '.items[] | select(.spec.volumes[]?.persistentVolumeClaim.claimName=="mon-pvc") | .metadata.name'
```

**Solution** :
- Supprimer l'ancien pod utilisant le volume
- Ou attendre qu'il se termine
- Ou utiliser un stockage ReadWriteMany si nécessaire

#### Solution 4 : Permissions Incorrectes

**Problème** : Le conteneur n'a pas les permissions pour accéder au volume.

**Solution temporaire (test)** :
```yaml
spec:
  containers:
  - name: mon-app
    image: nginx
    securityContext:
      runAsUser: 0  # Root (seulement pour tester !)
    volumeMounts:
    - name: data
      mountPath: /data
```

**Solution appropriée** :
Corriger les permissions sur le nœud :

```bash
# Trouver le chemin du volume
PATH_VOLUME=$(microk8s kubectl get pv $PV_NAME -o jsonpath='{.spec.hostPath.path}')

# Changer les permissions
sudo chown -R 1000:1000 $PATH_VOLUME  # Remplacer 1000 par l'UID/GID de votre app
sudo chmod -R 755 $PATH_VOLUME
```

Ou définir `fsGroup` dans le pod :

```yaml
spec:
  securityContext:
    fsGroup: 1000  # Le groupe qui aura accès au volume
  containers:
  - name: mon-app
    image: nginx
    volumeMounts:
    - name: data
      mountPath: /data
```

#### Solution 5 : Recréer le Pod

Si tout semble correct mais le pod ne démarre toujours pas :

```bash
# Supprimer le pod
microk8s kubectl delete pod mon-pod

# Le deployment le recréera automatiquement
# Ou recréez-le manuellement
microk8s kubectl apply -f pod.yaml
```

## Problème 3 : Données Perdues Après Redémarrage

### Symptômes

- Les données disparaissent quand un pod redémarre
- Les modifications ne sont pas sauvegardées
- L'application perd son état

### Diagnostic

#### Étape 1 : Vérifier le Type de Volume

```bash
# Voir la configuration du pod
microk8s kubectl get pod mon-pod -o yaml | grep -A 10 "volumes:"
```

**Type de volume** :
- `emptyDir` → Données perdues à chaque redémarrage du pod
- `persistentVolumeClaim` → Données conservées

#### Étape 2 : Vérifier le MountPath

```bash
# Où est monté le volume dans le conteneur ?
microk8s kubectl get pod mon-pod -o jsonpath='{.spec.containers[0].volumeMounts[*].mountPath}'
```

**Vérifier** : Est-ce le bon répertoire où l'application stocke ses données ?

**Exemple** : Pour PostgreSQL, les données sont dans `/var/lib/postgresql/data`

#### Étape 3 : Tester l'Écriture

```bash
# Entrer dans le pod
microk8s kubectl exec -it mon-pod -- bash

# Créer un fichier de test
echo "test" > /data/test.txt

# Sortir et redémarrer le pod
exit
microk8s kubectl delete pod mon-pod

# Attendre que le pod redémarre
microk8s kubectl wait --for=condition=ready pod mon-pod --timeout=60s

# Vérifier si le fichier existe toujours
microk8s kubectl exec -it mon-pod -- cat /data/test.txt
```

### Solutions Courantes

#### Solution 1 : Utiliser un PVC au Lieu d'emptyDir

**Problème** : Vous utilisez `emptyDir` pour des données importantes.

**Solution** : Remplacer par un PVC.

**Avant (emptyDir)** :
```yaml
spec:
  containers:
  - name: mon-app
    volumeMounts:
    - name: data
      mountPath: /data
  volumes:
  - name: data
    emptyDir: {}  # ❌ Données perdues !
```

**Après (PVC)** :
```yaml
spec:
  containers:
  - name: mon-app
    volumeMounts:
    - name: data
      mountPath: /data
  volumes:
  - name: data
    persistentVolumeClaim:
      claimName: mon-pvc  # ✅ Données persistantes
```

Créer le PVC :
```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: mon-pvc
spec:
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
  storageClassName: microk8s-hostpath
```

#### Solution 2 : Corriger le MountPath

**Problème** : Le volume est monté au mauvais endroit.

**Exemple - PostgreSQL** :

```yaml
spec:
  containers:
  - name: postgres
    image: postgres:15
    volumeMounts:
    - name: data
      mountPath: /var/lib/postgresql/data  # Bon répertoire pour PostgreSQL
  volumes:
  - name: data
    persistentVolumeClaim:
      claimName: postgres-pvc
```

**Vérifier la documentation** de votre application pour connaître le bon répertoire.

#### Solution 3 : Utiliser un StatefulSet

Pour les applications avec état (bases de données, etc.), utilisez un StatefulSet au lieu d'un Deployment :

```yaml
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
        image: postgres:15
        volumeMounts:
        - name: data
          mountPath: /var/lib/postgresql/data
  volumeClaimTemplates:  # Crée automatiquement un PVC par réplica
  - metadata:
      name: data
    spec:
      accessModes: [ "ReadWriteOnce" ]
      resources:
        requests:
          storage: 1Gi
      storageClassName: microk8s-hostpath
```

**Avantages** :
- PVC créés automatiquement
- Identité stable pour chaque pod
- Ordre de démarrage/arrêt contrôlé

## Problème 4 : Espace Disque Insuffisant

### Symptômes

- Pods évictés (expulsés)
- Erreurs d'écriture dans les logs
- `no space left on device`
- Pods en état `Evicted`

```bash
microk8s kubectl get pods
# NAME      READY   STATUS    RESTARTS   AGE
# mon-pod   0/1     Evicted   0          5m
```

### Diagnostic

#### Étape 1 : Vérifier l'Espace Disque du Nœud

```bash
# Utilisation du disque sur le nœud
df -h

# Spécifiquement le répertoire des volumes
df -h /var/snap/microk8s/common/default-storage
```

**Seuils d'alerte** :
- > 85% : Kubernetes commence à évincer des pods
- > 90% : Problèmes imminents

#### Étape 2 : Voir les Pods Évictés

```bash
# Lister tous les pods évictés
microk8s kubectl get pods -A | grep Evicted

# Voir pourquoi
microk8s kubectl describe pod mon-pod
```

**Dans describe** :
```
Reason: Evicted
Message: The node was low on resource: ephemeral-storage.
```

#### Étape 3 : Identifier ce qui Prend de l'Espace

```bash
# Espace utilisé par les conteneurs et images
microk8s ctr images ls -q | xargs microk8s ctr images size

# Espace utilisé par les volumes
sudo du -sh /var/snap/microk8s/common/default-storage/*

# Plus gros répertoires
sudo du -h /var/snap/microk8s/common/default-storage | sort -rh | head -20
```

### Solutions Courantes

#### Solution 1 : Nettoyer les Pods Évictés

Les pods évictés restent dans le système :

```bash
# Supprimer tous les pods évictés
microk8s kubectl get pods -A | grep Evicted | awk '{print $2 " -n " $1}' | xargs -L1 microk8s kubectl delete pod
```

#### Solution 2 : Nettoyer les Images Non Utilisées

```bash
# Supprimer les images non utilisées
microk8s ctr images ls -q | xargs microk8s ctr images rm

# Ou avec crictl
microk8s crictl rmi --prune
```

#### Solution 3 : Augmenter la Taille du PVC

Si votre application a besoin de plus d'espace :

**Vérifier si la StorageClass supporte l'expansion** :
```bash
microk8s kubectl get storageclass microk8s-hostpath -o jsonpath='{.allowVolumeExpansion}'
```

**Si true, augmenter la taille** :
```bash
# Éditer le PVC
microk8s kubectl edit pvc mon-pvc

# Modifier la valeur de storage
spec:
  resources:
    requests:
      storage: 5Gi  # Augmenter de 1Gi à 5Gi
```

**Note** : Avec hostpath-storage, vous devez aussi augmenter manuellement l'espace disque disponible sur le nœud.

#### Solution 4 : Ajouter de l'Espace Disque au Nœud

**Options** :
- Ajouter un disque physique
- Étendre la partition existante
- Monter un nouveau volume

**Exemple - Ajouter un nouveau disque** :
```bash
# Supposons un nouveau disque /dev/sdb

# Formater
sudo mkfs.ext4 /dev/sdb1

# Monter
sudo mount /dev/sdb1 /mnt/extra-storage

# Créer un nouveau StorageClass pointant vers ce disque
# (Configuration avancée)
```

#### Solution 5 : Limiter l'Utilisation du Stockage

Définir des limites d'espace disque pour les pods :

```yaml
spec:
  containers:
  - name: mon-app
    resources:
      limits:
        ephemeral-storage: "2Gi"  # Limite d'espace disque temporaire
      requests:
        ephemeral-storage: "1Gi"
```

#### Solution 6 : Nettoyer les Logs

Les logs de conteneurs peuvent occuper beaucoup d'espace :

```bash
# Espace utilisé par les logs
sudo du -sh /var/snap/microk8s/common/var/log

# Nettoyer les vieux logs
sudo find /var/snap/microk8s/common/var/log -type f -name "*.log" -mtime +7 -delete
```

**Configuration de la rotation des logs** : Généralement gérée automatiquement par Kubernetes, mais peut être ajustée.

## Problème 5 : Performance Lente du Stockage

### Symptômes

- Opérations d'I/O très lentes
- Base de données lente
- Timeouts d'applications
- Forte utilisation du CPU en attente d'I/O

### Diagnostic

#### Étape 1 : Tester les Performances I/O

Créer un pod de test :

```bash
# Pod pour tester I/O
microk8s kubectl run io-test --image=ubuntu --restart=Never --rm -it -- bash
```

Dans le pod :
```bash
# Installer les outils
apt update && apt install -y fio

# Tester l'écriture séquentielle
fio --name=write-test --ioengine=libaio --rw=write --bs=1M --size=1G --numjobs=1 --time_based --runtime=30s --group_reporting

# Tester la lecture séquentielle
fio --name=read-test --ioengine=libaio --rw=read --bs=1M --size=1G --numjobs=1 --time_based --runtime=30s --group_reporting

# Tester I/O aléatoire (plus représentatif pour bases de données)
fio --name=random-rw --ioengine=libaio --rw=randrw --bs=4k --size=1G --numjobs=4 --time_based --runtime=60s --group_reporting
```

#### Étape 2 : Surveiller l'Utilisation Disque

```bash
# Sur le nœud
iostat -x 5

# Ou
iotop
```

#### Étape 3 : Vérifier si le Stockage est le Goulot

```bash
# Utilisation CPU en attente I/O
top
# Regarder %wa (wait) dans les statistiques CPU

# Ou
vmstat 1
# Regarder la colonne 'wa'
```

**Si %wa est élevé (>20%)** : Le système attend beaucoup sur les I/O.

### Solutions Courantes

#### Solution 1 : Utiliser un Stockage Plus Rapide

**hostpath-storage utilise le disque local du nœud**. Pour de meilleures performances :

**Options** :
- Utiliser un SSD au lieu d'un HDD
- Utiliser un stockage NVMe
- Configurer un RAID pour de meilleures performances

**Sur MicroK8s** : Le hostpath-storage utilise le système de fichiers local. Assurez-vous que `/var/snap/microk8s/common/default-storage` est sur un disque rapide.

#### Solution 2 : Optimiser la Configuration de l'Application

**Pour PostgreSQL** :
```yaml
env:
- name: POSTGRES_INITDB_ARGS
  value: "--data-checksums"
- name: PGDATA
  value: /var/lib/postgresql/data/pgdata
# Configurations de performance dans postgresql.conf
```

**Pour MySQL** :
```yaml
env:
- name: MYSQL_INITDB_SKIP_TZINFO
  value: "1"
# Désactiver certaines fonctionnalités pour de meilleures performances
```

#### Solution 3 : Ajuster les Paramètres du Système de Fichiers

Sur le nœud :

```bash
# Monter avec des options optimisées
# Dans /etc/fstab, ajouter des options comme:
# noatime,nodiratime pour réduire les écritures

# Ou remonter temporairement
sudo mount -o remount,noatime /var/snap/microk8s/common/default-storage
```

#### Solution 4 : Utiliser un Cache

Pour les lectures fréquentes :

```yaml
# Ajouter un volume emptyDir comme cache
volumes:
- name: cache
  emptyDir:
    medium: Memory  # Cache en RAM (très rapide)
    sizeLimit: 1Gi
- name: data
  persistentVolumeClaim:
    claimName: mon-pvc
```

#### Solution 5 : Distribuer la Charge

Si plusieurs applications utilisent le même disque :
- Utiliser différents disques pour différentes applications
- Planifier les tâches lourdes en I/O à des moments différents

## Problème 6 : PV Non Supprimé

### Symptômes

Après avoir supprimé un PVC, le PersistentVolume reste en état `Released` et ne peut pas être réutilisé.

```bash
microk8s kubectl get pv
# NAME      CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS     CLAIM
# pv-123    1Gi        RWO            Retain           Released   default/mon-pvc
```

### Diagnostic

#### Vérifier la Reclaim Policy

```bash
microk8s kubectl get pv pv-123 -o jsonpath='{.spec.persistentVolumeReclaimPolicy}'
```

**Policies possibles** :
- **Retain** : Le PV est conservé après suppression du PVC (nécessite nettoyage manuel)
- **Delete** : Le PV est automatiquement supprimé (et les données aussi !)
- **Recycle** : Déprécié (nettoyage basique et réutilisation)

### Solutions

#### Solution 1 : Nettoyer Manuellement (Retain)

Si la policy est `Retain` :

```bash
# 1. Nettoyer les données si nécessaire
PATH_VOLUME=$(microk8s kubectl get pv pv-123 -o jsonpath='{.spec.hostPath.path}')
sudo rm -rf $PATH_VOLUME/*

# 2. Supprimer la référence au PVC
microk8s kubectl patch pv pv-123 -p '{"spec":{"claimRef": null}}'

# 3. Le PV est maintenant Available et peut être réutilisé
microk8s kubectl get pv pv-123
```

#### Solution 2 : Supprimer et Recréer

Si vous n'avez plus besoin du PV :

```bash
# Supprimer le PV
microk8s kubectl delete pv pv-123

# Un nouveau PV sera créé automatiquement à la prochaine demande (si provisionnement dynamique)
```

#### Solution 3 : Changer la Reclaim Policy

Pour de futurs PV, définir la policy dans la StorageClass :

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: ma-storageclass
provisioner: microk8s.io/hostpath
reclaimPolicy: Delete  # ou Retain selon vos besoins
```

## Outils et Commandes Utiles

### Inspection Générale

```bash
# Vue d'ensemble du stockage
microk8s kubectl get pv,pvc,storageclass

# Avec plus de détails
microk8s kubectl get pv,pvc -o wide

# Toutes les ressources de stockage dans tous les namespaces
microk8s kubectl get pv,pvc --all-namespaces
```

### Détails Spécifiques

```bash
# Détails d'un PV
microk8s kubectl describe pv <pv-name>

# Détails d'un PVC
microk8s kubectl describe pvc <pvc-name>

# Détails d'une StorageClass
microk8s kubectl describe storageclass <name>
```

### Vérifier l'Utilisation

```bash
# Utilisation du disque par pod
microk8s kubectl exec <pod-name> -- df -h

# Top des fichiers/répertoires dans un volume
microk8s kubectl exec <pod-name> -- du -sh /data/* | sort -rh
```

### Logs du Provisioner

```bash
# Logs du provisioner hostpath
microk8s kubectl logs -n kube-system -l k8s-app=hostpath-provisioner

# Suivre en temps réel
microk8s kubectl logs -n kube-system -l k8s-app=hostpath-provisioner -f
```

### Nettoyage

```bash
# Supprimer tous les PVC non utilisés d'un namespace
microk8s kubectl get pvc | grep -v NAME | awk '{print $1}' | xargs microk8s kubectl delete pvc

# Supprimer tous les PV Released
microk8s kubectl get pv | grep Released | awk '{print $1}' | xargs microk8s kubectl delete pv

# Attention : Vérifiez avant de supprimer !
```

## Workflows de Dépannage

### Workflow 1 : PVC Pending

```bash
# 1. Vérifier le PVC
microk8s kubectl describe pvc mon-pvc
# Regarder Events

# 2. Vérifier la StorageClass
microk8s kubectl get storageclass
# Existe-t-elle ? Est-elle par défaut ?

# 3. Vérifier le provisioner
microk8s kubectl get pods -n kube-system | grep hostpath

# 4. Voir les logs
microk8s kubectl logs -n kube-system -l k8s-app=hostpath-provisioner

# 5. Si hostpath-storage pas activé
microk8s enable hostpath-storage

# 6. Re-tester
microk8s kubectl get pvc mon-pvc
```

### Workflow 2 : Pod Ne Monte Pas le Volume

```bash
# 1. Vérifier l'état du pod
microk8s kubectl describe pod mon-pod
# Regarder Events pour erreurs de montage

# 2. Le PVC est-il Bound ?
microk8s kubectl get pvc
# Status doit être Bound

# 3. Vérifier le PV
PV=$(microk8s kubectl get pvc mon-pvc -o jsonpath='{.spec.volumeName}')
microk8s kubectl describe pv $PV

# 4. Vérifier les permissions
PATH=$(microk8s kubectl get pv $PV -o jsonpath='{.spec.hostPath.path}')
ls -ld $PATH

# 5. Si permissions incorrectes
sudo chmod 755 $PATH

# 6. Redémarrer le pod
microk8s kubectl delete pod mon-pod
```

### Workflow 3 : Données Perdues

```bash
# 1. Vérifier le type de volume
microk8s kubectl get pod mon-pod -o yaml | grep -A5 volumes:

# 2. Si emptyDir, convertir en PVC
# Créer un PVC
cat > pvc.yaml <<EOF
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: mon-pvc
spec:
  accessModes: [ReadWriteOnce]
  resources:
    requests:
      storage: 1Gi
  storageClassName: microk8s-hostpath
EOF

microk8s kubectl apply -f pvc.yaml

# 3. Modifier le pod/deployment pour utiliser le PVC
# (éditer le YAML)

# 4. Redéployer
microk8s kubectl apply -f deployment.yaml
```

## Bonnes Pratiques

### 1. Toujours Utiliser des PVC pour Données Importantes

❌ **Ne faites PAS** :
```yaml
volumes:
- name: data
  emptyDir: {}  # Données perdues !
```

✅ **Faites** :
```yaml
volumes:
- name: data
  persistentVolumeClaim:
    claimName: mon-pvc  # Données persistantes
```

### 2. Définir des Tailles Appropriées

```yaml
resources:
  requests:
    storage: 10Gi  # Prévoir large, mais pas trop
```

**Conseils** :
- Estimer les besoins réels
- Ajouter 30-50% de marge
- Surveiller l'utilisation

### 3. Utiliser StorageClass Appropriée

```yaml
spec:
  storageClassName: microk8s-hostpath  # Toujours spécifier
```

### 4. Définir la Reclaim Policy

Pour le développement :
```yaml
reclaimPolicy: Delete  # Nettoyage automatique
```

Pour la production :
```yaml
reclaimPolicy: Retain  # Éviter la perte accidentelle
```

### 5. Sauvegardes Régulières

Même avec du stockage persistant, faites des backups :

```bash
# Exemple simple
microk8s kubectl exec mon-pod -- tar czf /tmp/backup.tar.gz /data

microk8s kubectl cp mon-pod:/tmp/backup.tar.gz ./backup-$(date +%Y%m%d).tar.gz
```

**Automatisez avec des CronJobs** :
```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: backup-job
spec:
  schedule: "0 2 * * *"  # Tous les jours à 2h
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: backup
            image: busybox
            command:
            - /bin/sh
            - -c
            - tar czf /backup/data-$(date +%Y%m%d).tar.gz /data
            volumeMounts:
            - name: data
              mountPath: /data
            - name: backup
              mountPath: /backup
          volumes:
          - name: data
            persistentVolumeClaim:
              claimName: app-pvc
          - name: backup
            persistentVolumeClaim:
              claimName: backup-pvc
          restartPolicy: OnFailure
```

### 6. Surveiller l'Utilisation

Créer des alertes pour :
- Espace disque > 80%
- PVC presque pleins
- PV en état Released

### 7. Documenter les Volumes

Gardez une trace de :
- Quel PVC contient quelles données
- Importance de chaque PVC (critique/non-critique)
- Politique de backup
- Propriétaire/responsable

**Exemple** :
```markdown
## Volumes Persistants

| PVC | Application | Taille | Données | Backup | Responsable |
|-----|-------------|--------|---------|--------|-------------|
| postgres-pvc | PostgreSQL | 20Gi | Base de données principale | Quotidien | DBA Team |
| uploads-pvc | Web App | 50Gi | Fichiers uploadés | Hebdomadaire | Dev Team |
| logs-pvc | Logging | 10Gi | Logs d'application | Aucun | Ops Team |
```

### 8. Utiliser des Labels

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: mon-pvc
  labels:
    app: mon-app
    environment: production
    backup: "true"
```

Facilite la gestion et l'automation.

## Checklist de Dépannage Stockage

### PVC en Pending

- [ ] L'addon hostpath-storage est-il activé ?
- [ ] Le provisioner est-il Running ?
- [ ] La StorageClass existe-t-elle ?
- [ ] Le nom de StorageClass est-il correct dans le PVC ?
- [ ] Y a-t-il de l'espace disque disponible sur le nœud ?
- [ ] Les logs du provisioner montrent-ils des erreurs ?

### Pod Ne Monte Pas le Volume

- [ ] Le PVC est-il Bound ?
- [ ] Le PV existe-t-il ?
- [ ] Les AccessModes sont-ils compatibles ?
- [ ] Le volume est-il déjà attaché à un autre pod ?
- [ ] Les permissions du répertoire sont-elles correctes ?
- [ ] Le pod et le PVC sont-ils dans le même namespace ?

### Données Perdues

- [ ] Utilisez-vous un PVC (pas emptyDir) ?
- [ ] Le volume est-il monté au bon endroit ?
- [ ] Le PVC est-il toujours Bound ?
- [ ] Les données sont-elles dans le bon répertoire ?

### Espace Disque Insuffisant

- [ ] Espace disque disponible sur le nœud ?
- [ ] Images non utilisées à nettoyer ?
- [ ] Pods évictés à supprimer ?
- [ ] Logs à nettoyer ?
- [ ] Taille du PVC suffisante ?

## Résumé

### Commandes Essentielles

```bash
# Vue d'ensemble
kubectl get pv,pvc,storageclass

# Détails
kubectl describe pvc <pvc-name>
kubectl describe pv <pv-name>

# Activer stockage
microk8s enable hostpath-storage

# Vérifier provisioner
kubectl get pods -n kube-system | grep hostpath
kubectl logs -n kube-system -l k8s-app=hostpath-provisioner

# Espace disque
df -h
du -sh /var/snap/microk8s/common/default-storage/*

# Nettoyer
kubectl delete pvc <pvc-name>
kubectl delete pv <pv-name>
```

### Types de Volumes

| Type | Persistance | Cas d'Usage |
|------|-------------|-------------|
| emptyDir | Temporaire (vie du pod) | Cache, fichiers temporaires |
| hostPath | Persiste sur le nœud | Dev local, accès système |
| PVC | Persiste (géré par K8s) | Production, données importantes |

### Points Clés

1. **Toujours utiliser PVC** pour les données importantes
2. **hostpath-storage** doit être activé pour le provisionnement dynamique
3. **emptyDir** = données perdues au redémarrage du pod
4. **AccessMode RWO** = un seul nœud à la fois
5. **Surveiller l'espace disque** régulièrement
6. **Sauvegarder** les données critiques
7. **Définir les tailles** appropriées dès le début

### Messages d'Erreur Courants

| Erreur | Cause | Solution |
|--------|-------|----------|
| PVC stays Pending | Provisioner absent ou erreur | Activer hostpath-storage, vérifier logs |
| Unable to mount volume | PVC non Bound ou permissions | Vérifier PVC status et permissions |
| No space left on device | Disque plein | Nettoyer images, logs, augmenter espace |
| Volume already attached | Volume RWO sur autre nœud | Attendre détachement ou utiliser RWX |
| Permission denied | Permissions incorrectes | Corriger permissions ou utiliser fsGroup |

**Prochaine étape** : Section 23.8 - Performance et ressources

---


⏭️ [Performance et ressources](/23-depannage-et-maintenance/08-performance-et-ressources.md)
