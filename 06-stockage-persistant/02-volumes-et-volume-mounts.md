🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 6.2 Volumes et Volume Mounts

## Introduction

Dans la section précédente, nous avons découvert les concepts fondamentaux du stockage dans Kubernetes. Maintenant, il est temps de rentrer dans le concret et de comprendre comment utiliser les **Volumes** et les **Volume Mounts** dans vos pods.

Ces deux concepts sont les briques de base du stockage dans Kubernetes. Maîtriser leur utilisation est essentiel pour toute application qui a besoin de conserver des données, partager des fichiers entre conteneurs, ou simplement stocker des logs et des fichiers de configuration.

## Rappel : Qu'est-ce qu'un Volume ?

Un **Volume** dans Kubernetes est un répertoire accessible aux conteneurs d'un pod. Contrairement au système de fichiers d'un conteneur qui est éphémère, un volume a une durée de vie liée au pod lui-même (et non à un conteneur individuel).

**Points importants à retenir** :
- Un volume existe aussi longtemps que le pod existe
- Si un conteneur dans le pod redémarre, le volume persiste
- Quand le pod est supprimé, le volume l'est aussi (sauf pour certains types de volumes)
- Plusieurs conteneurs dans le même pod peuvent partager le même volume

## Qu'est-ce qu'un Volume Mount ?

Un **Volume Mount** est le lien qui connecte un volume à un conteneur. Il définit :
- **Quel volume** utiliser (référence au nom du volume)
- **Où** monter ce volume dans le conteneur (le chemin, exemple : `/app/data`)
- **Comment** le monter (lecture seule ou lecture-écriture)

**Analogie** : Si le volume est une clé USB, le volume mount est l'action de brancher cette clé sur votre ordinateur et de décider dans quel dossier elle apparaîtra.

## Anatomie d'un pod avec volumes

Voyons la structure générale d'un pod utilisant des volumes :

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: mon-pod
spec:
  containers:
  - name: mon-conteneur
    image: nginx:latest
    volumeMounts:          # Section Volume Mounts
    - name: mon-volume     # Référence au volume
      mountPath: /app/data # Où dans le conteneur

  volumes:                 # Section Volumes
  - name: mon-volume       # Nom du volume
    emptyDir: {}           # Type de volume
```

**Observation importante** : Notez qu'il y a deux sections distinctes :
1. `volumeMounts` dans la définition du conteneur
2. `volumes` dans la spécification du pod

Cette séparation permet de définir un volume une fois et de le monter dans plusieurs conteneurs si nécessaire.

## Les différents types de volumes

Kubernetes propose de nombreux types de volumes. Nous allons explorer les plus courants, du plus simple au plus avancé.

### 1. emptyDir : Le volume temporaire

Le type `emptyDir` est le plus simple. Il crée un répertoire vide quand le pod démarre.

#### Caractéristiques
- **Créé** : Quand le pod est assigné à un nœud
- **Supprimé** : Quand le pod est retiré du nœud
- **Partagé** : Entre tous les conteneurs du pod
- **Stockage** : Sur le disque du nœud (ou en RAM avec `medium: Memory`)

#### Cas d'usage typiques
- Espace de travail temporaire pour des calculs
- Cache pour accélérer des opérations
- Partage de données entre conteneurs dans le même pod
- Espace pour des fichiers temporaires

#### Exemple basique

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-emptydir-simple
spec:
  containers:
  - name: nginx
    image: nginx:1.21
    volumeMounts:
    - name: cache-volume
      mountPath: /cache

  volumes:
  - name: cache-volume
    emptyDir: {}
```

**Explication** :
- Un volume nommé `cache-volume` est créé (type `emptyDir`)
- Il est monté dans le conteneur nginx au chemin `/cache`
- Tout fichier écrit dans `/cache` sera stocké dans ce volume
- Si le conteneur nginx redémarre, les fichiers seront toujours là
- Si le pod est supprimé, le volume et son contenu disparaissent

#### Exemple avec emptyDir en mémoire

Pour des performances maximales, vous pouvez stocker l'emptyDir en RAM :

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-emptydir-memory
spec:
  containers:
  - name: app
    image: busybox:1.35
    command: ['sh', '-c', 'while true; do date >> /memory/date.txt; sleep 5; done']
    volumeMounts:
    - name: memory-volume
      mountPath: /memory

  volumes:
  - name: memory-volume
    emptyDir:
      medium: Memory      # Stockage en RAM
      sizeLimit: 100Mi    # Limite de taille
```

**Points importants** :
- `medium: Memory` : Stocke les données en RAM (très rapide)
- `sizeLimit` : Limite la quantité de RAM utilisable
- Attention : La RAM est limitée, à utiliser avec parcimonie
- Utile pour : caches temporaires, fichiers de travail nécessitant de hautes performances

#### Exemple de partage entre conteneurs

Un des usages les plus utiles d'emptyDir est le partage de données entre conteneurs :

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-partage-fichiers
spec:
  containers:
  # Premier conteneur : génère des logs
  - name: generateur-logs
    image: busybox:1.35
    command: ['sh', '-c', 'while true; do echo "Log: $(date)" >> /logs/app.log; sleep 10; done']
    volumeMounts:
    - name: logs-partages
      mountPath: /logs

  # Deuxième conteneur : lit les logs
  - name: lecteur-logs
    image: busybox:1.35
    command: ['sh', '-c', 'while true; do tail -f /logs/app.log; sleep 30; done']
    volumeMounts:
    - name: logs-partages
      mountPath: /logs
      readOnly: true      # Lecture seule

  volumes:
  - name: logs-partages
    emptyDir: {}
```

**Ce qui se passe ici** :
- Le volume `logs-partages` est créé
- Le conteneur `generateur-logs` écrit dans `/logs/app.log`
- Le conteneur `lecteur-logs` lit depuis `/logs/app.log`
- Les deux conteneurs voient le même fichier
- Le lecteur est en mode `readOnly` pour éviter les modifications accidentelles

### 2. hostPath : Accéder au système de fichiers du nœud

Le type `hostPath` monte un fichier ou un répertoire du système de fichiers du nœud directement dans le pod.

#### Caractéristiques
- **Accès direct** : Au système de fichiers du nœud Kubernetes
- **Persistance** : Les données survivent au pod
- **Limitation** : Lié à un nœud spécifique (pas portable)
- **Risque** : Peut poser des problèmes de sécurité

#### Avertissement important

⚠️ **hostPath est à utiliser avec précaution** :
- Les données sont liées à un nœud particulier
- Si le pod est replanifié sur un autre nœud, il ne retrouvera pas ses données
- Peut créer des failles de sécurité
- Déconseillé en production (sauf cas très spécifiques)

#### Cas d'usage appropriés
- Lab personnel avec un seul nœud (comme avec MicroK8s)
- Accès aux logs du nœud
- Accès au socket Docker ou containerd
- Outils de monitoring qui ont besoin d'accéder au système hôte

#### Exemple simple

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-hostpath-simple
spec:
  containers:
  - name: app
    image: nginx:1.21
    volumeMounts:
    - name: data-hote
      mountPath: /data

  volumes:
  - name: data-hote
    hostPath:
      path: /mnt/data           # Chemin sur le nœud
      type: DirectoryOrCreate   # Crée le répertoire s'il n'existe pas
```

**Explication** :
- Le répertoire `/mnt/data` du nœud est monté dans le pod
- Il apparaît comme `/data` dans le conteneur
- `type: DirectoryOrCreate` : Kubernetes créera le répertoire s'il n'existe pas
- Les fichiers écrits dans `/data` seront visibles sur le nœud dans `/mnt/data`

#### Types de hostPath

Le paramètre `type` permet de spécifier ce que vous attendez :

| Type | Description |
|------|-------------|
| `DirectoryOrCreate` | Répertoire ; créé s'il n'existe pas |
| `Directory` | Le répertoire doit déjà exister |
| `FileOrCreate` | Fichier ; créé s'il n'existe pas |
| `File` | Le fichier doit déjà exister |
| `Socket` | Socket UNIX qui doit exister |
| `CharDevice` | Périphérique caractère qui doit exister |
| `BlockDevice` | Périphérique bloc qui doit exister |

#### Exemple avec vérification de type

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-hostpath-fichier
spec:
  containers:
  - name: configurateur
    image: busybox:1.35
    command: ['sh', '-c', 'cat /config/app.conf && sleep 3600']
    volumeMounts:
    - name: fichier-config
      mountPath: /config/app.conf

  volumes:
  - name: fichier-config
    hostPath:
      path: /etc/mon-app/config.conf
      type: File    # Le fichier doit exister
```

**Utilité** : Le pod ne démarrera pas si le fichier n'existe pas, ce qui évite des erreurs silencieuses.

### 3. configMap : Configuration sous forme de volume

Un ConfigMap peut être monté comme un volume, ce qui est très pratique pour injecter des fichiers de configuration.

#### Caractéristiques
- Chaque clé du ConfigMap devient un fichier
- Le nom de la clé = nom du fichier
- La valeur de la clé = contenu du fichier
- Lecture seule par défaut
- Peut être mis à jour sans recréer le pod

#### Exemple complet

Créons d'abord un ConfigMap :

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: config-nginx
data:
  nginx.conf: |
    server {
      listen 80;
      server_name localhost;
      location / {
        root /usr/share/nginx/html;
        index index.html;
      }
    }
  index.html: |
    <!DOCTYPE html>
    <html>
    <head><title>Mon App</title></head>
    <body><h1>Configuration via ConfigMap</h1></body>
    </html>
```

Maintenant, utilisons-le dans un pod :

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-avec-configmap
spec:
  containers:
  - name: nginx
    image: nginx:1.21
    volumeMounts:
    - name: config-volume
      mountPath: /etc/nginx/conf.d

  volumes:
  - name: config-volume
    configMap:
      name: config-nginx
```

**Ce qui se passe** :
- Le ConfigMap `config-nginx` contient 2 clés : `nginx.conf` et `index.html`
- Ces clés deviennent 2 fichiers dans `/etc/nginx/conf.d/`
- Nginx peut maintenant lire sa configuration depuis ces fichiers

#### Monter uniquement certaines clés

Vous n'êtes pas obligé de monter tout le ConfigMap :

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-configmap-partiel
spec:
  containers:
  - name: app
    image: busybox:1.35
    command: ['sh', '-c', 'cat /config/nginx.conf && sleep 3600']
    volumeMounts:
    - name: config-volume
      mountPath: /config

  volumes:
  - name: config-volume
    configMap:
      name: config-nginx
      items:                    # Sélectionner des clés spécifiques
      - key: nginx.conf         # Clé du ConfigMap
        path: nginx.conf        # Nom du fichier dans le volume
```

#### Spécifier des permissions

```yaml
volumes:
- name: config-volume
  configMap:
    name: config-nginx
    defaultMode: 0644    # Permissions des fichiers (en octal)
```

### 4. secret : Données sensibles comme volume

Les Secrets fonctionnent de manière similaire aux ConfigMaps, mais pour des données sensibles.

#### Exemple

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: credentials-db
type: Opaque
stringData:
  username: admin
  password: motdepasse-secret
---
apiVersion: v1
kind: Pod
metadata:
  name: pod-avec-secret
spec:
  containers:
  - name: app
    image: busybox:1.35
    command: ['sh', '-c', 'cat /secrets/username && cat /secrets/password && sleep 3600']
    volumeMounts:
    - name: secret-volume
      mountPath: /secrets
      readOnly: true    # Important : toujours en lecture seule

  volumes:
  - name: secret-volume
    secret:
      secretName: credentials-db
      defaultMode: 0400    # Permissions restrictives
```

**Bonnes pratiques** :
- Toujours monter les secrets en `readOnly`
- Utiliser des permissions restrictives (`0400` ou `0440`)
- Ne jamais logger ou afficher le contenu des secrets
- Les secrets sont en base64, pas chiffrés (attention !)

### 5. persistentVolumeClaim : Stockage persistant

C'est le type de volume le plus important pour du stockage qui doit survivre au pod.

#### Exemple

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
      storage: 5Gi
---
apiVersion: v1
kind: Pod
metadata:
  name: pod-avec-pvc
spec:
  containers:
  - name: app
    image: nginx:1.21
    volumeMounts:
    - name: persistent-storage
      mountPath: /data

  volumes:
  - name: persistent-storage
    persistentVolumeClaim:
      claimName: mon-pvc    # Référence au PVC
```

**Points clés** :
- Le PVC doit exister avant le pod
- Le stockage survit à la suppression du pod
- Les données restent accessibles si vous recréez le pod
- Nous verrons les PVC en détail dans la section 6.4

## Options des Volume Mounts

Les Volume Mounts ont plusieurs options importantes à connaître.

### mountPath : Le chemin de destination

C'est le chemin dans le conteneur où le volume sera accessible :

```yaml
volumeMounts:
- name: mon-volume
  mountPath: /app/data    # Accessible dans le conteneur à ce chemin
```

### subPath : Monter un sous-répertoire

Parfois, vous ne voulez monter qu'une partie d'un volume :

```yaml
volumeMounts:
- name: mon-volume
  mountPath: /app/config.yaml
  subPath: config.yaml    # Monte uniquement ce fichier
```

**Cas d'usage** :
- Monter un seul fichier d'un ConfigMap sans écraser un répertoire entier
- Utiliser différentes parties d'un PVC pour différents conteneurs

#### Exemple pratique avec subPath

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-subpath-exemple
spec:
  containers:
  - name: nginx
    image: nginx:1.21
    volumeMounts:
    # Monte uniquement nginx.conf
    - name: config-volume
      mountPath: /etc/nginx/nginx.conf
      subPath: nginx.conf
    # Monte uniquement default.conf
    - name: config-volume
      mountPath: /etc/nginx/conf.d/default.conf
      subPath: default.conf

  volumes:
  - name: config-volume
    configMap:
      name: nginx-configs
```

### readOnly : Protection contre les modifications

Empêche les écritures dans le volume :

```yaml
volumeMounts:
- name: mon-volume
  mountPath: /data
  readOnly: true    # Aucune écriture possible
```

**Quand l'utiliser** :
- Pour des configurations qui ne doivent jamais être modifiées
- Pour des secrets
- Pour partager des données en lecture seule entre conteneurs

### mountPropagation : Propagation des montages

Option avancée qui contrôle comment les montages sont propagés :

```yaml
volumeMounts:
- name: mon-volume
  mountPath: /data
  mountPropagation: HostToContainer
```

**Valeurs possibles** :
- `None` (défaut) : Pas de propagation
- `HostToContainer` : Les montages du host apparaissent dans le conteneur
- `Bidirectional` : Propagation bidirectionnelle

⚠️ **Usage avancé** : Nécessaire seulement pour des cas très spécifiques (comme des gestionnaires de volumes)

## Patterns courants d'utilisation

### Pattern 1 : Séparation des conteneurs

Un conteneur génère des données, un autre les traite :

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: producer-consumer
spec:
  containers:
  # Producteur de données
  - name: producer
    image: busybox:1.35
    command: ['sh', '-c', 'while true; do echo "Data: $(date)" >> /shared/data.txt; sleep 5; done']
    volumeMounts:
    - name: shared-data
      mountPath: /shared

  # Consommateur de données
  - name: consumer
    image: busybox:1.35
    command: ['sh', '-c', 'tail -f /shared/data.txt']
    volumeMounts:
    - name: shared-data
      mountPath: /shared
      readOnly: true

  volumes:
  - name: shared-data
    emptyDir: {}
```

### Pattern 2 : Init Container avec volume

Un Init Container prépare des données pour le conteneur principal :

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-avec-init
spec:
  initContainers:
  - name: initializer
    image: busybox:1.35
    command: ['sh', '-c', 'echo "Données initiales" > /data/init.txt']
    volumeMounts:
    - name: data-volume
      mountPath: /data

  containers:
  - name: app
    image: busybox:1.35
    command: ['sh', '-c', 'cat /data/init.txt && sleep 3600']
    volumeMounts:
    - name: data-volume
      mountPath: /data

  volumes:
  - name: data-volume
    emptyDir: {}
```

**Utilité** :
- Télécharger des dépendances
- Préparer une configuration
- Initialiser une base de données

### Pattern 3 : Sidecar pour les logs

Un conteneur sidecar exporte les logs d'un conteneur principal :

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-avec-sidecar-logs
spec:
  containers:
  # Application principale
  - name: app
    image: busybox:1.35
    command: ['sh', '-c', 'while true; do echo "Application log" >> /var/log/app.log; sleep 5; done']
    volumeMounts:
    - name: logs
      mountPath: /var/log

  # Sidecar pour exporter les logs
  - name: log-exporter
    image: busybox:1.35
    command: ['sh', '-c', 'tail -f /var/log/app.log']
    volumeMounts:
    - name: logs
      mountPath: /var/log
      readOnly: true

  volumes:
  - name: logs
    emptyDir: {}
```

### Pattern 4 : Configuration multi-sources

Combiner plusieurs sources de configuration :

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-multi-config
spec:
  containers:
  - name: app
    image: nginx:1.21
    volumeMounts:
    # Configuration générale
    - name: config-general
      mountPath: /etc/config
    # Secrets
    - name: secrets
      mountPath: /etc/secrets
      readOnly: true
    # Données persistantes
    - name: data
      mountPath: /data

  volumes:
  - name: config-general
    configMap:
      name: app-config
  - name: secrets
    secret:
      secretName: app-secrets
  - name: data
    persistentVolumeClaim:
      claimName: app-data-pvc
```

## Bonnes pratiques

### 1. Nommage cohérent

Utilisez des noms descriptifs pour vos volumes :

```yaml
# Bon
volumes:
- name: nginx-config
- name: app-data
- name: shared-logs

# À éviter
volumes:
- name: vol1
- name: temp
- name: data
```

### 2. Documenter l'usage

Ajoutez des commentaires dans vos manifestes :

```yaml
volumeMounts:
- name: app-data
  mountPath: /data
  # Stockage persistant pour les uploads utilisateurs
```

### 3. Permissions appropriées

Pour les secrets et données sensibles :

```yaml
volumes:
- name: secret-data
  secret:
    secretName: api-keys
    defaultMode: 0400    # Lecture uniquement pour le propriétaire
```

### 4. Utiliser readOnly quand possible

Protégez vos données contre les modifications accidentelles :

```yaml
volumeMounts:
- name: config
  mountPath: /etc/config
  readOnly: true    # Évite les modifications
```

### 5. Taille appropriée pour emptyDir

Si vous utilisez `medium: Memory`, limitez toujours la taille :

```yaml
volumes:
- name: cache
  emptyDir:
    medium: Memory
    sizeLimit: 100Mi    # Toujours spécifier une limite
```

### 6. Éviter hostPath en production

```yaml
# OK pour un lab/dev
volumes:
- name: data
  hostPath:
    path: /mnt/data

# Mieux pour la production
volumes:
- name: data
  persistentVolumeClaim:
    claimName: app-data
```

## Erreurs courantes et solutions

### Erreur 1 : Volume mount échoue

**Symptôme** : Le pod ne démarre pas, état `ContainerCreating`

```bash
microk8s kubectl describe pod mon-pod
```

Recherchez :
```
Warning  FailedMount  kubelet  MountVolume.SetUp failed for volume "XXX"
```

**Causes possibles** :
- Le nom du volume dans `volumeMounts` ne correspond pas au nom dans `volumes`
- Le PVC référencé n'existe pas
- Le ConfigMap ou Secret référencé n'existe pas
- Type hostPath avec un chemin qui n'existe pas (et type non-create)

**Solution** : Vérifiez la correspondance des noms et l'existence des ressources référencées.

### Erreur 2 : Permissions insuffisantes

**Symptôme** : Le conteneur ne peut pas écrire dans le volume

```
Permission denied when writing to /data
```

**Causes** :
- Le volume est monté en `readOnly`
- Les permissions du volume ne correspondent pas à l'utilisateur du conteneur
- Pour hostPath : permissions incorrectes sur le nœud

**Solution** :
```yaml
# Vérifier readOnly
volumeMounts:
- name: data
  mountPath: /data
  readOnly: false    # Ou supprimer cette ligne (false par défaut)

# Ajuster les permissions pour secrets/configMaps
volumes:
- name: config
  configMap:
    name: app-config
    defaultMode: 0644    # Lecture pour tous, écriture pour propriétaire
```

### Erreur 3 : Conflit de subPath

**Symptôme** : Le fichier ou répertoire n'est pas visible

**Cause** : Mauvaise utilisation de `subPath`

```yaml
# Incorrect : subPath pointe vers un fichier qui n'existe pas
volumeMounts:
- name: config
  mountPath: /app/config.yaml
  subPath: fichier-inexistant.yaml
```

**Solution** : Vérifier que le subPath existe dans la source.

### Erreur 4 : Données perdues avec emptyDir

**Symptôme** : Les données disparaissent après un redéploiement

**Cause** : emptyDir est éphémère, lié au cycle de vie du pod

**Solution** : Utiliser un PersistentVolumeClaim pour les données importantes :

```yaml
volumes:
- name: data
  persistentVolumeClaim:
    claimName: important-data
```

## Commandes utiles pour diagnostiquer

### Vérifier les volumes d'un pod

```bash
microk8s kubectl describe pod <nom-pod>
```

Recherchez les sections :
- `Mounts:` dans la description du conteneur
- `Volumes:` dans la description du pod

### Voir le contenu d'un volume

```bash
# Accéder au conteneur
microk8s kubectl exec -it <nom-pod> -- /bin/sh

# Puis lister le contenu du volume
ls -la /chemin/vers/volume
```

### Vérifier les permissions

```bash
microk8s kubectl exec <nom-pod> -- ls -la /chemin/vers/volume
```

### Tester l'écriture dans un volume

```bash
microk8s kubectl exec <nom-pod> -- touch /chemin/vers/volume/test.txt
```

## Résumé visuel des types de volumes

```
┌─────────────────────────────────────────────────────────────┐
│                    Types de Volumes                         │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  emptyDir                                                   │
│  └─ Temporaire, lié au pod                                  │
│     └─ Usage : cache, partage entre conteneurs              │
│                                                             │
│  hostPath                                                   │
│  └─ Accès direct au nœud                                    │
│     └─ Usage : lab/dev, accès système (prudence)            │
│                                                             │
│  configMap                                                  │
│  └─ Configuration sous forme de fichiers                    │
│     └─ Usage : fichiers de config, templates                │
│                                                             │
│  secret                                                     │
│  └─ Données sensibles sous forme de fichiers                │
│     └─ Usage : credentials, certificats, clés API           │
│                                                             │
│  persistentVolumeClaim                                      │
│  └─ Stockage persistant indépendant du pod                  │
│     └─ Usage : bases de données, fichiers permanents        │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

## Récapitulatif : Volume vs Volume Mount

**Volume** (dans `spec.volumes`) :
- Définit **QUOI** : quel stockage utiliser
- Définit **D'OÙ** vient le stockage
- Définit le **TYPE** de stockage
- Défini une seule fois au niveau du pod

**Volume Mount** (dans `spec.containers[].volumeMounts`) :
- Définit **OÙ** dans le conteneur
- Définit **COMMENT** le monter (readOnly, subPath, etc.)
- Peut être défini différemment pour chaque conteneur
- Référence un volume défini dans `spec.volumes`

## Prochaines étapes

Maintenant que vous maîtrisez les volumes et volume mounts, vous êtes prêt à explorer :

- **Section 6.3** : PersistentVolumes (PV) - Le stockage physique dans Kubernetes
- **Section 6.4** : PersistentVolumeClaims (PVC) - Comment demander du stockage
- **Section 6.5** : StorageClasses - Le provisionnement dynamique
- **Section 6.6** : Gestion avancée des volumes persistants
- **Section 6.7** : StatefulSets - Applications avec état nécessitant du stockage

## Conclusion

Les volumes et volume mounts sont les fondations du stockage dans Kubernetes. Comprendre leur fonctionnement est essentiel pour :

- Partager des données entre conteneurs
- Injecter de la configuration dans vos applications
- Gérer des secrets de manière sécurisée
- Préparer le terrain pour le stockage persistant

Avec MicroK8s, ces concepts sont particulièrement simples à mettre en pratique grâce à l'environnement de lab unifié. Dans les prochaines sections, nous construirons sur ces bases pour implémenter du stockage persistant robuste et professionnel.

---

**Points clés à retenir** :

✅ **Volumes** : Définissent le stockage au niveau du pod
✅ **Volume Mounts** : Lient les volumes aux conteneurs
✅ **emptyDir** : Simple et temporaire, parfait pour le partage
✅ **hostPath** : Accès au nœud, à utiliser avec prudence
✅ **configMap/secret** : Configuration et données sensibles sous forme de volumes
✅ **persistentVolumeClaim** : Pour du stockage vraiment persistant
✅ **subPath** : Permet de monter seulement une partie d'un volume
✅ **readOnly** : Protection importante pour configurations et secrets

⏭️ [PersistentVolumes (PV)](/06-stockage-persistant/03-persistentvolumes-pv.md)
