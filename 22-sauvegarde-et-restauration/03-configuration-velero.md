🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 22.3 Configuration Velero

## Introduction

Maintenant que Velero est installé et que vous savez créer des sauvegardes de base, il est temps d'explorer les options de configuration pour adapter Velero à vos besoins spécifiques. Cette section vous guidera à travers les différents paramètres et options pour optimiser, personnaliser et affiner votre solution de sauvegarde.

La configuration de Velero peut sembler intimidante avec toutes ses options, mais ne vous inquiétez pas : nous allons explorer chaque aspect progressivement, en commençant par les configurations les plus courantes.

## Structure de configuration de Velero

Velero utilise plusieurs mécanismes de configuration :

### 1. Configuration à l'installation

Certains paramètres sont définis lors de l'installation avec la commande `velero install`. Ils configurent le comportement de base de Velero.

### 2. Ressources Kubernetes

Velero crée des Custom Resources (CRD) dans Kubernetes que vous pouvez modifier :
- **BackupStorageLocation** : Où stocker les sauvegardes
- **VolumeSnapshotLocation** : Où stocker les snapshots de volumes
- **Schedule** : Planifications automatiques
- **Backup** : Objets représentant chaque sauvegarde

### 3. ConfigMap et annotations

- **ConfigMap** : Pour certains paramètres globaux
- **Annotations sur les Pods** : Pour contrôler finement les sauvegardes

### 4. Ligne de commande

Paramètres passés lors de la création de sauvegardes ou restaurations individuelles.

## Configuration du Backup Storage Location

Le **BackupStorageLocation** (BSL) définit où Velero stocke les archives de sauvegarde.

### Voir la configuration actuelle

```bash
velero backup-location get
```

Affichage détaillé :

```bash
kubectl get backupstoragelocations.velero.io -n velero -o yaml
```

### Configuration de base avec MinIO

Voici un exemple de BackupStorageLocation pour MinIO :

```yaml
apiVersion: velero.io/v1
kind: BackupStorageLocation
metadata:
  name: default
  namespace: velero
spec:
  provider: aws
  objectStorage:
    bucket: velero-backups
    prefix: production  # Préfixe optionnel pour organiser les backups
  config:
    region: minio
    s3ForcePathStyle: "true"
    s3Url: http://minio.velero.svc:9000
  credential:
    name: cloud-credentials
    key: cloud
```

**Paramètres expliqués** :

- **bucket** : Nom du bucket MinIO (créé précédemment)
- **prefix** : Sous-dossier dans le bucket (optionnel, utile pour organiser)
- **region** : "minio" par convention (peut être n'importe quoi)
- **s3ForcePathStyle** : Obligatoire pour MinIO (style d'URL différent d'AWS)
- **s3Url** : URL du service MinIO dans le cluster
- **credential** : Référence au Secret contenant les credentials

### Ajouter un préfixe pour organiser les sauvegardes

Si vous voulez séparer vos sauvegardes par environnement :

```yaml
objectStorage:
  bucket: velero-backups
  prefix: cluster-production  # Toutes les sauvegardes iront dans ce dossier
```

Dans MinIO, vous verrez :
```
velero-backups/
  └── cluster-production/
      └── backups/
          └── my-backup/
```

### Utiliser plusieurs destinations de sauvegarde

Vous pouvez configurer plusieurs BackupStorageLocations :

```bash
# Créer un second BSL (exemple : stockage cloud en plus de local)
velero backup-location create cloud-backup \
  --provider aws \
  --bucket my-s3-bucket \
  --config region=eu-west-1 \
  --credential cloud-credentials-aws
```

Puis spécifier le BSL lors de la création d'un backup :

```bash
velero backup create my-backup \
  --storage-location cloud-backup
```

### Configuration pour AWS S3 réel

Si vous utilisez AWS S3 au lieu de MinIO :

```yaml
apiVersion: velero.io/v1
kind: BackupStorageLocation
metadata:
  name: aws-backup
  namespace: velero
spec:
  provider: aws
  objectStorage:
    bucket: my-velero-backups-bucket
    prefix: microk8s-cluster
  config:
    region: eu-west-1
    # s3ForcePathStyle pas nécessaire pour AWS S3
```

### Configuration pour Google Cloud Storage

```yaml
apiVersion: velero.io/v1
kind: BackupStorageLocation
metadata:
  name: gcs-backup
  namespace: velero
spec:
  provider: gcp
  objectStorage:
    bucket: my-gcs-bucket
    prefix: velero
  config:
    # Pas de config supplémentaire nécessaire
```

### Changer la configuration d'un BSL existant

```bash
kubectl edit backupstoragelocation default -n velero
```

Ou appliquer un fichier YAML modifié :

```bash
kubectl apply -f backup-storage-location.yaml
```

### Désactiver temporairement un BSL

```bash
# Marquer comme read-only
kubectl patch backupstoragelocation default -n velero \
  --type merge \
  --patch '{"spec":{"accessMode":"ReadOnly"}}'
```

Modes disponibles :
- **ReadWrite** : Normal (défaut)
- **ReadOnly** : Velero peut lire mais pas écrire (utile pour la maintenance)

## Configuration des credentials

Les credentials permettent à Velero d'accéder au stockage objet.

### Format du fichier de credentials

Pour MinIO ou AWS S3 :

```ini
[default]
aws_access_key_id = votre-access-key
aws_secret_access_key = votre-secret-key
```

Pour Google Cloud :

```json
{
  "type": "service_account",
  "project_id": "votre-projet",
  "private_key_id": "...",
  "private_key": "...",
  "client_email": "...",
  "client_id": "...",
  "auth_uri": "https://accounts.google.com/o/oauth2/auth",
  "token_uri": "https://oauth2.googleapis.com/token"
}
```

### Créer ou mettre à jour le Secret

```bash
# Créer le secret
kubectl create secret generic cloud-credentials \
  --namespace velero \
  --from-file=cloud=credentials-file

# Mettre à jour un secret existant
kubectl delete secret cloud-credentials -n velero
kubectl create secret generic cloud-credentials \
  --namespace velero \
  --from-file=cloud=credentials-file
```

### Utiliser plusieurs jeux de credentials

Pour différents providers :

```bash
# Credentials MinIO
kubectl create secret generic minio-credentials \
  --namespace velero \
  --from-file=cloud=minio-creds

# Credentials AWS
kubectl create secret generic aws-credentials \
  --namespace velero \
  --from-file=cloud=aws-creds
```

Référencez-les dans vos BSL respectifs.

## Configuration de Restic/Kopia pour les volumes

Restic (ou Kopia, son successeur) est responsable de la sauvegarde des volumes persistants.

### Vérifier la configuration actuelle

```bash
# Vérifier que node-agent tourne
kubectl get pods -n velero -l name=node-agent

# Vérifier les logs
kubectl logs -n velero -l name=node-agent
```

### Configuration globale pour tous les volumes

Par défaut, les volumes ne sont pas sauvegardés. Vous devez l'activer :

**Option 1 : Au niveau du backup**

```bash
velero backup create my-backup \
  --default-volumes-to-fs-backup
```

**Option 2 : Au niveau du pod (annotation)**

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-app
  annotations:
    backup.velero.io/backup-volumes: volume1,volume2
spec:
  volumes:
  - name: volume1
    persistentVolumeClaim:
      claimName: data-pvc
  - name: volume2
    persistentVolumeClaim:
      claimName: logs-pvc
```

**Option 3 : Configuration globale Velero**

Modifier le deployment Velero pour activer par défaut :

```bash
kubectl edit deployment velero -n velero
```

Ajoutez dans les args :

```yaml
spec:
  template:
    spec:
      containers:
      - name: velero
        args:
        - server
        - --default-volumes-to-fs-backup
```

### Exclure des volumes spécifiques

Si vous avez activé la sauvegarde par défaut mais voulez exclure certains volumes :

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-app
  annotations:
    backup.velero.io/backup-volumes-excludes: cache-volume,temp-volume
```

### Configuration de la performance de Restic

Le DaemonSet node-agent peut être configuré pour optimiser les performances :

```bash
kubectl edit daemonset node-agent -n velero
```

**Limites de ressources** :

```yaml
spec:
  template:
    spec:
      containers:
      - name: node-agent
        resources:
          limits:
            cpu: "1"
            memory: 1Gi
          requests:
            cpu: 500m
            memory: 512Mi
```

**Variables d'environnement utiles** :

```yaml
env:
# Timeout pour les opérations de backup (défaut: 1h)
- name: FS_BACKUP_TIMEOUT
  value: "2h"

# Nombre de transferts parallèles (défaut: 1)
- name: PARALLEL_FILES_UPLOAD
  value: "4"
```

### Choisir Kopia au lieu de Restic

Kopia est le successeur de Restic, plus performant :

```bash
# Lors de l'installation
velero install \
  --use-node-agent \
  --uploader-type=kopia \
  ...autres paramètres...
```

Pour un cluster existant :

```bash
kubectl set env daemonset/node-agent -n velero VELERO_UPLOADER_TYPE=kopia
```

## Configuration des exclusions et inclusions

Contrôler finement ce qui est sauvegardé est essentiel pour optimiser les performances et la taille des backups.

### Exclure des ressources par type

Exclure tous les Secrets (par exemple, pour des raisons de sécurité) :

```bash
velero backup create my-backup \
  --exclude-resources secrets
```

Exclure plusieurs types :

```bash
velero backup create my-backup \
  --exclude-resources events,endpoints,secrets
```

### Exclure des namespaces

```bash
velero backup create my-backup \
  --exclude-namespaces kube-system,kube-public,velero
```

**Bonne pratique** : Toujours exclure au minimum :
- `kube-system` (géré par Kubernetes)
- `kube-public` (ressources publiques système)
- `velero` (ne pas sauvegarder Velero lui-même)

### Inclure seulement certains namespaces

```bash
velero backup create production-backup \
  --include-namespaces production,database
```

### Utiliser des labels pour filtrer

Sauvegarder seulement les ressources avec un label spécifique :

```bash
velero backup create app-backup \
  --selector app=myapp,tier=frontend
```

Exclure les ressources avec un label :

```bash
velero backup create backup \
  --selector "backup=enabled" \
  --exclude-namespaces "environment=test"
```

### Configuration globale des exclusions

Créer un ConfigMap pour définir des exclusions permanentes :

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: velero-config
  namespace: velero
data:
  excludedResourceTypes: events,endpoints,events.events.k8s.io
```

### Exclusions au niveau des pods

Marquer un pod pour qu'il soit exclu :

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: temporary-pod
  labels:
    velero.io/exclude-from-backup: "true"
```

## Configuration des Schedules (planifications)

Les schedules permettent d'automatiser les sauvegardes. Voici comment les configurer finement.

### Schedule de base avec rétention

```bash
velero schedule create daily-full-backup \
  --schedule="0 2 * * *" \
  --ttl 168h
```

### Schedule avec toutes les options

```bash
velero schedule create advanced-backup \
  --schedule="0 3 * * *" \
  --ttl 720h \
  --include-namespaces production,staging \
  --exclude-resources events,endpoints \
  --default-volumes-to-fs-backup \
  --snapshot-volumes=false \
  --labels environment=production,schedule=nightly
```

**Options expliquées** :

- `--schedule` : Expression cron
- `--ttl` : Durée de rétention (720h = 30 jours)
- `--include-namespaces` : Namespaces à inclure
- `--exclude-resources` : Types de ressources à exclure
- `--default-volumes-to-fs-backup` : Sauvegarder les volumes avec Restic
- `--snapshot-volumes` : Désactiver les snapshots natifs
- `--labels` : Labels pour organiser les backups

### Schedule avec template de nom

Personnaliser le nom des backups générés :

```bash
velero schedule create named-backup \
  --schedule="0 2 * * *" \
  --use-owner-references-in-backup=true \
  --labels schedule=daily
```

Les backups seront nommés : `named-backup-20250115020000`

### Schedule conditionnel (selon les ressources)

Créer des schedules différents pour différents types d'applications :

```bash
# Backup fréquent des bases de données
velero schedule create db-backup \
  --schedule="0 */4 * * *" \
  --selector app=database \
  --default-volumes-to-fs-backup \
  --ttl 168h

# Backup quotidien des applications
velero schedule create app-backup \
  --schedule="0 2 * * *" \
  --selector tier=frontend \
  --ttl 720h

# Backup hebdomadaire complet
velero schedule create weekly-full \
  --schedule="0 1 * * 0" \
  --ttl 2160h
```

### Définir un schedule via YAML

Pour une configuration plus complexe :

```yaml
apiVersion: velero.io/v1
kind: Schedule
metadata:
  name: comprehensive-backup
  namespace: velero
spec:
  schedule: "0 2 * * *"
  template:
    ttl: 720h0m0s
    includedNamespaces:
    - production
    - staging
    excludedResources:
    - events
    - endpoints
    defaultVolumesToFsBackup: true
    storageLocation: default
    hooks:
      resources:
      - name: postgres-backup
        includedNamespaces:
        - production
        labelSelector:
          matchLabels:
            app: postgres
        pre:
        - exec:
            container: postgres
            command:
            - /bin/bash
            - -c
            - pg_dumpall -U postgres > /tmp/backup.sql
            onError: Fail
            timeout: 5m
        post:
        - exec:
            container: postgres
            command:
            - /bin/bash
            - -c
            - rm -f /tmp/backup.sql
```

Appliquer :

```bash
kubectl apply -f comprehensive-schedule.yaml
```

## Configuration des Backup Hooks

Les hooks permettent d'exécuter des commandes avant ou après une sauvegarde. C'est crucial pour les bases de données.

### Hook simple au niveau du pod

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: postgres-pod
  annotations:
    # Commande avant le backup
    pre.hook.backup.velero.io/command: '["/bin/bash", "-c", "pg_dump mydb > /backup/dump.sql"]'

    # Container où exécuter (si plusieurs containers)
    pre.hook.backup.velero.io/container: postgres

    # Timeout pour la commande
    pre.hook.backup.velero.io/timeout: 5m

    # Que faire en cas d'erreur (Fail ou Continue)
    pre.hook.backup.velero.io/on-error: Fail

    # Commande après le backup
    post.hook.backup.velero.io/command: '["/bin/bash", "-c", "rm -f /backup/dump.sql"]'
```

### Hook défini au niveau du backup

```bash
velero backup create postgres-backup \
  --include-namespaces database \
  --hooks='{"resources":[
    {
      "name":"postgres-hook",
      "includedNamespaces":["database"],
      "labelSelector":{
        "matchLabels":{"app":"postgres"}
      },
      "pre":[
        {
          "exec":{
            "container":"postgres",
            "command":["/bin/bash","-c","pg_dumpall > /tmp/backup.sql"],
            "onError":"Fail",
            "timeout":"5m"
          }
        }
      ],
      "post":[
        {
          "exec":{
            "container":"postgres",
            "command":["/bin/bash","-c","rm -f /tmp/backup.sql"]
          }
        }
      ]
    }
  ]}'
```

### Exemples de hooks pour différentes bases de données

#### PostgreSQL

```yaml
annotations:
  pre.hook.backup.velero.io/command: >-
    ["/bin/bash", "-c",
     "PGPASSWORD=$POSTGRES_PASSWORD pg_dumpall -U postgres > /backup/dump.sql"]
  post.hook.backup.velero.io/command: >-
    ["/bin/bash", "-c", "rm -f /backup/dump.sql"]
```

#### MySQL/MariaDB

```yaml
annotations:
  pre.hook.backup.velero.io/command: >-
    ["/bin/bash", "-c",
     "mysqldump --all-databases -u root -p$MYSQL_ROOT_PASSWORD > /backup/dump.sql"]
  post.hook.backup.velero.io/command: >-
    ["/bin/bash", "-c", "rm -f /backup/dump.sql"]
```

#### MongoDB

```yaml
annotations:
  pre.hook.backup.velero.io/command: >-
    ["/bin/bash", "-c",
     "mongodump --out /backup/dump --username $MONGO_INITDB_ROOT_USERNAME --password $MONGO_INITDB_ROOT_PASSWORD"]
  post.hook.backup.velero.io/command: >-
    ["/bin/bash", "-c", "rm -rf /backup/dump"]
```

#### Redis

```yaml
annotations:
  pre.hook.backup.velero.io/command: >-
    ["/bin/bash", "-c", "redis-cli BGSAVE"]
  pre.hook.backup.velero.io/timeout: "10m"
```

### Hook pour arrêter temporairement une application

Parfois il faut arrêter l'application pour garantir la cohérence :

```yaml
annotations:
  pre.hook.backup.velero.io/command: >-
    ["/bin/bash", "-c", "kill -STOP 1"]
  post.hook.backup.velero.io/command: >-
    ["/bin/bash", "-c", "kill -CONT 1"]
```

**Attention** : Utilisez avec précaution, cela rend l'application indisponible pendant le backup.

## Configuration des options de performance

### Paramètres du serveur Velero

Modifier le deployment Velero pour ajuster les performances :

```bash
kubectl edit deployment velero -n velero
```

**Options de performance** :

```yaml
spec:
  template:
    spec:
      containers:
      - name: velero
        args:
        - server

        # Nombre de workers pour les backups (défaut: 1)
        - --default-backup-workers=3

        # Nombre de workers pour les restaurations (défaut: 1)
        - --default-restore-workers=3

        # Timeout par défaut pour les backups (défaut: 1h)
        - --default-backup-ttl=24h

        # Niveau de log (défaut: info)
        - --log-level=info

        # Format de log (text ou json)
        - --log-format=text

        # Activer les profiling pour debug
        - --profiler-address=:6060
```

### Ajuster les ressources du pod Velero

```yaml
resources:
  limits:
    cpu: "2"
    memory: 2Gi
  requests:
    cpu: 500m
    memory: 512Mi
```

### Optimiser la taille des backups

**Compression** :

Velero compresse automatiquement les archives, mais vous pouvez ajuster :

```bash
# Désactiver la compression (plus rapide, mais plus gros)
velero backup create fast-backup \
  --snapshot-volumes=false \
  --default-volumes-to-fs-backup=false
```

**Deduplication avec Restic** :

Restic fait de la déduplication automatique. Pour l'optimiser :

```bash
# Augmenter le cache Restic
kubectl set env daemonset/node-agent -n velero \
  RESTIC_CACHE_DIR=/tmp/restic-cache
```

### Limiter la bande passante

Pour ne pas saturer le réseau pendant les backups :

```bash
# Limiter Restic à 10 MB/s
kubectl set env daemonset/node-agent -n velero \
  RESTIC_LIMIT_UPLOAD=10000
```

## Configuration du garbage collection

Velero nettoie automatiquement les sauvegardes expirées.

### Paramètres de garbage collection

```bash
kubectl edit deployment velero -n velero
```

Ajouter :

```yaml
args:
- server
# Fréquence du garbage collection (défaut: 1h)
- --garbage-collection-frequency=2h
```

### Désactiver le garbage collection automatique

Si vous préférez gérer manuellement :

```yaml
args:
- server
- --garbage-collection-frequency=0
```

Puis supprimer manuellement les backups expirés :

```bash
velero backup delete old-backup
```

### Politique de rétention personnalisée

Créer un script qui implémente votre propre logique :

```bash
#!/bin/bash
# Garder les 7 derniers backups quotidiens
# Garder les 4 derniers backups hebdomadaires
# Garder les 12 derniers backups mensuels

# Lister tous les backups
BACKUPS=$(velero backup get -o json)

# Logique personnalisée de nettoyage
# ...
```

Planifier avec un CronJob Kubernetes.

## Configuration de la sécurité

### Chiffrement des sauvegardes

#### Option 1 : Chiffrement côté serveur MinIO

```bash
# Dans MinIO
mc encrypt set sse-s3 myminio/velero-backups
```

#### Option 2 : Chiffrement avec Restic

Restic chiffre automatiquement. Définir un mot de passe :

```bash
# Créer un secret avec le mot de passe Restic
kubectl create secret generic restic-encryption \
  --namespace velero \
  --from-literal=password=VotreSuperMotDePasse123!
```

Configurer Velero pour l'utiliser :

```bash
kubectl set env daemonset/node-agent -n velero \
  RESTIC_PASSWORD_FILE=/credentials/restic-password
```

### RBAC pour Velero

Velero nécessite des permissions larges. Vous pouvez les restreindre :

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: velero-restricted
rules:
# Permettre seulement lecture pour certaines ressources sensibles
- apiGroups: [""]
  resources: ["secrets"]
  verbs: ["get", "list"]
# Interdire la modification
- apiGroups: [""]
  resources: ["secrets"]
  verbs: ["create", "update", "delete", "patch"]
```

### Limiter l'accès aux sauvegardes

Utiliser des ServiceAccounts différents pour différents usages :

```bash
# ServiceAccount pour les backups production
kubectl create serviceaccount velero-prod -n velero

# ServiceAccount pour les backups dev
kubectl create serviceaccount velero-dev -n velero
```

## Configuration multi-cluster

Si vous gérez plusieurs clusters MicroK8s, vous pouvez centraliser les sauvegardes.

### Approche : Bucket partagé avec préfixes

**Cluster 1 (production)** :

```yaml
spec:
  objectStorage:
    bucket: velero-backups-shared
    prefix: cluster-production
```

**Cluster 2 (staging)** :

```yaml
spec:
  objectStorage:
    bucket: velero-backups-shared
    prefix: cluster-staging
```

**Cluster 3 (dev)** :

```yaml
spec:
  objectStorage:
    bucket: velero-backups-shared
    prefix: cluster-dev
```

### Migration entre clusters

Pour migrer des applications d'un cluster à l'autre :

1. **Cluster source** : Créer un backup
2. **Cluster destination** : Configurer le même BackupStorageLocation
3. **Cluster destination** : Restaurer

```bash
# Sur le cluster destination
velero backup get
# Vous verrez les backups du cluster source

velero restore create migration \
  --from-backup production-backup \
  --namespace-mappings oldns:newns
```

## Configuration des notifications

### Notifications par webhook

Créer un hook pour envoyer une notification après chaque backup :

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: velero-notifications
  namespace: velero
data:
  webhook-url: "https://hooks.slack.com/services/YOUR/WEBHOOK/URL"
```

Script à exécuter après le backup (dans un Job) :

```bash
#!/bin/bash
BACKUP_NAME=$1
STATUS=$(velero backup describe $BACKUP_NAME -o json | jq -r '.status.phase')
WEBHOOK_URL=$(kubectl get cm velero-notifications -n velero -o jsonpath='{.data.webhook-url}')

curl -X POST $WEBHOOK_URL \
  -H 'Content-Type: application/json' \
  -d "{\"text\":\"Backup $BACKUP_NAME: $STATUS\"}"
```

### Intégration avec Prometheus Alertmanager

Si vous utilisez Prometheus :

```yaml
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: velero-alerts
  namespace: velero
spec:
  groups:
  - name: velero
    interval: 30s
    rules:
    - alert: VeleroBackupFailed
      expr: velero_backup_failure_total > 0
      for: 5m
      labels:
        severity: critical
      annotations:
        summary: "Velero backup failed"
        description: "A Velero backup has failed"

    - alert: VeleroBackupTooOld
      expr: time() - velero_backup_last_successful_timestamp > 86400
      for: 1h
      labels:
        severity: warning
      annotations:
        summary: "No recent Velero backup"
        description: "Last successful backup was more than 24h ago"
```

## Configuration du monitoring

### Activer les métriques Prometheus

Velero expose des métriques par défaut sur le port 8085.

Créer un Service pour les exposer :

```yaml
apiVersion: v1
kind: Service
metadata:
  name: velero-metrics
  namespace: velero
  labels:
    app.kubernetes.io/name: velero
spec:
  selector:
    name: velero
  ports:
  - name: metrics
    port: 8085
    targetPort: 8085
```

### ServiceMonitor pour Prometheus Operator

```yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: velero
  namespace: velero
spec:
  selector:
    matchLabels:
      app.kubernetes.io/name: velero
  endpoints:
  - port: metrics
    interval: 30s
    path: /metrics
```

### Métriques importantes

Velero expose ces métriques clés :

- `velero_backup_total` : Nombre total de backups
- `velero_backup_failure_total` : Nombre de backups échoués
- `velero_backup_success_total` : Nombre de backups réussis
- `velero_backup_duration_seconds` : Durée des backups
- `velero_backup_last_successful_timestamp` : Timestamp du dernier backup réussi
- `velero_restore_total` : Nombre total de restaurations
- `velero_volume_snapshot_success_total` : Snapshots de volumes réussis

## Configuration avancée des restaurations

### Options de restauration personnalisées

```bash
velero restore create advanced-restore \
  --from-backup my-backup \
  --include-namespaces app \
  --exclude-resources services,endpoints \
  --namespace-mappings source-ns:target-ns \
  --restore-volumes \
  --preserve-nodeports=false \
  --include-cluster-resources=false
```

**Options expliquées** :

- `--namespace-mappings` : Renommer les namespaces lors de la restauration
- `--restore-volumes` : Restaurer aussi les volumes (défaut: true)
- `--preserve-nodeports` : Garder les NodePort existants (peut causer des conflits)
- `--include-cluster-resources` : Inclure les ressources au niveau cluster (PV, etc.)

### Restauration avec transformation

Modifier les ressources pendant la restauration avec des patches JSON :

```yaml
apiVersion: velero.io/v1
kind: Restore
metadata:
  name: transformed-restore
  namespace: velero
spec:
  backupName: my-backup
  includedNamespaces:
  - production
  labelSelector:
    matchLabels:
      app: myapp
  restorePVs: true
  # Patches JSON pour transformer les ressources
  resourceModifier:
    filters:
    - resourceNamespaceModifier:
        mapping:
          production: staging
    - resourceModifier:
        conditions:
          groupResource: deployments.apps
        patches:
        # Réduire les replicas
        - operation: replace
          path: "/spec/replicas"
          value: "1"
```

### Configuration des restore hooks

Exécuter des commandes après une restauration :

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: postgres-pod
  annotations:
    # Après la restauration, réindexer la base
    post.hook.restore.velero.io/command: >-
      ["/bin/bash", "-c", "psql -U postgres -c 'REINDEX DATABASE mydb;'"]
    post.hook.restore.velero.io/container: postgres
    post.hook.restore.velero.io/timeout: 10m
```

## Fichiers de configuration pour référence

### Fichier de configuration complet pour un lab

Voici un exemple de configuration complète et commentée :

```yaml
---
# Namespace pour Velero
apiVersion: v1
kind: Namespace
metadata:
  name: velero

---
# Backup Storage Location
apiVersion: velero.io/v1
kind: BackupStorageLocation
metadata:
  name: default
  namespace: velero
spec:
  provider: aws
  objectStorage:
    bucket: velero-backups
    prefix: microk8s-lab
  config:
    region: minio
    s3ForcePathStyle: "true"
    s3Url: http://minio.velero.svc:9000
  credential:
    name: cloud-credentials
    key: cloud

---
# Schedule - Backup quotidien complet
apiVersion: velero.io/v1
kind: Schedule
metadata:
  name: daily-full-backup
  namespace: velero
spec:
  schedule: "0 2 * * *"
  template:
    ttl: 168h0m0s  # 7 jours
    includedNamespaces:
    - "*"
    excludedNamespaces:
    - kube-system
    - kube-public
    - velero
    excludedResources:
    - events
    - endpoints
    defaultVolumesToFsBackup: true
    storageLocation: default

---
# Schedule - Backup hebdomadaire avec longue rétention
apiVersion: velero.io/v1
kind: Schedule
metadata:
  name: weekly-archive
  namespace: velero
spec:
  schedule: "0 1 * * 0"  # Dimanche à 1h
  template:
    ttl: 2160h0m0s  # 90 jours
    includedNamespaces:
    - "*"
    excludedNamespaces:
    - kube-system
    - kube-public
    - velero
    defaultVolumesToFsBackup: true
    storageLocation: default
    labels:
      backup-type: archive
      retention: long

---
# Schedule - Backup fréquent des apps critiques
apiVersion: velero.io/v1
kind: Schedule
metadata:
  name: critical-apps-backup
  namespace: velero
spec:
  schedule: "0 */4 * * *"  # Toutes les 4 heures
  template:
    ttl: 72h0m0s  # 3 jours
    labelSelector:
      matchLabels:
        criticality: high
    defaultVolumesToFsBackup: true
    storageLocation: default
```

### Script de configuration post-installation

Créez un script `configure-velero.sh` :

```bash
#!/bin/bash

# Configuration post-installation de Velero

echo "=== Configuration de Velero ==="

# 1. Créer les schedules
echo "Création des schedules..."
velero schedule create daily-full \
  --schedule="0 2 * * *" \
  --ttl 168h \
  --exclude-namespaces kube-system,kube-public,velero \
  --default-volumes-to-fs-backup

velero schedule create weekly-archive \
  --schedule="0 1 * * 0" \
  --ttl 2160h \
  --default-volumes-to-fs-backup

# 2. Ajuster les ressources Velero
echo "Ajustement des ressources..."
kubectl patch deployment velero -n velero --type='json' \
  -p='[{"op": "replace", "path": "/spec/template/spec/containers/0/resources/limits/cpu", "value": "1"}]'

kubectl patch deployment velero -n velero --type='json' \
  -p='[{"op": "replace", "path": "/spec/template/spec/containers/0/resources/limits/memory", "value": "1Gi"}]'

# 3. Configurer les métriques
echo "Configuration du monitoring..."
kubectl apply -f - <<EOF
apiVersion: v1
kind: Service
metadata:
  name: velero-metrics
  namespace: velero
spec:
  selector:
    name: velero
  ports:
  - name: metrics
    port: 8085
    targetPort: 8085
EOF

# 4. Premier backup de test
echo "Création d'un backup de test..."
velero backup create initial-test-backup

echo "=== Configuration terminée ==="
echo "Vérifiez le statut avec: velero backup get"
```

## Troubleshooting de la configuration

### Problèmes de connexion au stockage

```bash
# Tester la connectivité
kubectl run -it --rm debug --image=amazon/aws-cli --restart=Never -- \
  aws s3 ls s3://velero-backups \
  --endpoint-url=http://minio.velero.svc:9000 \
  --region=minio
```

### Vérifier la configuration

```bash
# Configuration Velero
kubectl get deployment velero -n velero -o yaml

# Configuration BSL
kubectl get backupstoragelocations.velero.io -n velero -o yaml

# Credentials
kubectl get secret cloud-credentials -n velero -o yaml
```

### Valider les schedules

```bash
# Lister tous les schedules
velero schedule get

# Vérifier qu'ils créent bien des backups
velero backup get

# Voir les détails d'un schedule
kubectl get schedule daily-full-backup -n velero -o yaml
```

## Conclusion

La configuration de Velero offre une grande flexibilité pour adapter l'outil à vos besoins spécifiques. Les points clés à retenir :

### Points essentiels

✅ **BackupStorageLocation** : Configure où stocker les sauvegardes
✅ **Schedules** : Automatisent les sauvegardes régulières
✅ **Hooks** : Garantissent la cohérence des données (bases de données)
✅ **Exclusions** : Optimisent la taille et la vitesse des backups
✅ **Performance** : Ajustez workers et ressources selon vos besoins
✅ **Sécurité** : Chiffrez les sauvegardes et gérez les accès
✅ **Monitoring** : Surveillez vos sauvegardes avec Prometheus

### Recommandations pour un lab

Pour un lab personnel MicroK8s, une configuration simple et efficace :

1. Un schedule quotidien avec rétention de 7 jours
2. Un schedule hebdomadaire avec rétention de 90 jours
3. Exclusion des namespaces système
4. Hooks pour les bases de données si nécessaire
5. Monitoring basique avec métriques

### Prochaine étape

Dans la section suivante (22.4), nous verrons comment gérer les snapshots de volumes de manière plus avancée et comment optimiser encore plus vos sauvegardes.

---

**Checklist de configuration Velero** :

- [ ] BackupStorageLocation configuré et accessible
- [ ] Credentials correctement définis
- [ ] Node-agent (Restic/Kopia) déployé et fonctionnel
- [ ] Schedules créés avec bonnes fréquences et rétentions
- [ ] Exclusions configurées (namespaces système)
- [ ] Hooks définis pour les bases de données
- [ ] Ressources ajustées (CPU/RAM)
- [ ] Métriques exposées pour le monitoring
- [ ] Tests de restauration effectués
- [ ] Documentation de votre configuration créée

⏭️ [Snapshots de volumes](/22-sauvegarde-et-restauration/04-snapshots-de-volumes.md)
