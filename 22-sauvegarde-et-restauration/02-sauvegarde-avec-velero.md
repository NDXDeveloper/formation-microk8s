🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 22.2 Sauvegarde avec Velero

## Introduction à Velero

**Velero** est un outil open-source spécialement conçu pour sauvegarder et restaurer des clusters Kubernetes. Développé initialement par VMware sous le nom "Ark", puis renommé Velero, c'est aujourd'hui l'outil de référence pour la protection des données Kubernetes.

### Pourquoi utiliser Velero ?

Vous pourriez vous demander : "Pourquoi ne pas simplement sauvegarder les fichiers YAML et les volumes avec des outils classiques ?". Velero apporte plusieurs avantages majeurs :

1. **Compréhension native de Kubernetes** : Velero comprend les relations entre les ressources (un Deployment utilise un ConfigMap, qui nécessite un Secret, etc.)

2. **Sauvegarde complète et cohérente** : Il capture l'état du cluster à un instant T, y compris les ressources créées dynamiquement

3. **Sauvegarde des volumes persistants** : Intégration avec les snapshots de volumes cloud ou copies de fichiers

4. **Restauration sélective** : Possibilité de restaurer uniquement certaines applications ou namespaces

5. **Migration facilitée** : Permet de migrer des applications entre clusters différents

6. **Planification automatique** : Sauvegardes programmées sans intervention manuelle

### Analogie pour comprendre Velero

Imaginez que vous déménagez :

- **Approche manuelle** : Vous notez sur papier où est chaque meuble, puis vous les déplacez un par un. Vous risquez d'oublier des choses, de perdre la liste, ou de ne pas savoir comment tout était connecté.

- **Approche Velero** : Vous prenez une photo complète de chaque pièce (avec toutes les connexions électriques, les positions exactes), puis vous emballez tout intelligemment. Quand vous arrivez, vous pouvez recréer exactement la même disposition, ou choisir de ne recréer que le salon.

## Architecture et fonctionnement de Velero

### Les composants principaux

Velero fonctionne avec deux composants clés :

1. **Le serveur Velero** : Un Pod qui tourne dans votre cluster Kubernetes
   - Surveille les demandes de sauvegarde
   - Communique avec l'API Kubernetes pour récupérer les ressources
   - Orchestre les sauvegardes et restaurations

2. **Le CLI Velero** : Un outil en ligne de commande sur votre machine
   - Permet de déclencher des sauvegardes
   - Visualiser l'état des backups
   - Lancer des restaurations

### Comment Velero sauvegarde-t-il les données ?

Velero effectue deux types de sauvegardes :

#### 1. Sauvegarde des ressources Kubernetes

Velero interroge l'API Kubernetes et récupère :
- Les définitions de tous les objets (Pods, Services, Deployments, ConfigMaps, etc.)
- Les métadonnées et labels
- Les relations entre objets

Ces données sont ensuite **compressées et archivées** dans un fichier au format JSON.

#### 2. Sauvegarde des volumes persistants

Velero propose deux méthodes :

**a) Snapshots natifs** (si votre infrastructure le supporte) :
- Utilise les fonctionnalités de snapshot du cloud provider (AWS EBS, Google Persistent Disk, Azure Disk)
- Rapide et efficace
- Nécessite un provider cloud compatible

**b) Restic/Kopia (sauvegarde par fichier)** :
- Copie les fichiers depuis les volumes
- Fonctionne avec n'importe quel stockage (idéal pour MicroK8s)
- Plus lent mais universel
- Ne nécessite pas de support de snapshot

### Où Velero stocke-t-il les sauvegardes ?

Velero utilise un **stockage objet** (Object Storage) comme destination. Les options principales sont :

- **AWS S3** : Le plus courant
- **Google Cloud Storage** : Pour GCP
- **Azure Blob Storage** : Pour Azure
- **MinIO** : Solution self-hosted compatible S3 (idéal pour un lab)
- **Autres** : Tout stockage compatible S3 (Backblaze B2, Wasabi, etc.)

**Pour votre lab MicroK8s**, nous recommandons **MinIO** car :
- Gratuit et open-source
- S'installe facilement dans votre cluster ou sur une autre machine
- Compatible S3 (fonctionne exactement comme AWS S3)
- Permet de tester sans frais cloud

## Prérequis pour utiliser Velero

Avant d'installer Velero, assurez-vous d'avoir :

### 1. Un cluster MicroK8s fonctionnel

```bash
microk8s status
```

Le cluster doit être en état "running".

### 2. Kubectl configuré

```bash
microk8s kubectl version
```

Ou si vous avez configuré l'alias :

```bash
kubectl version
```

### 3. Un stockage objet disponible

Vous avez plusieurs choix :

**Option A : MinIO local (recommandé pour débuter)**
- Installation dans votre cluster MicroK8s ou sur une autre machine
- Gratuit, aucun compte cloud nécessaire
- Idéal pour l'apprentissage

**Option B : Compte cloud S3**
- AWS S3 (payant, quelques centimes/Go/mois)
- Backblaze B2 (plus économique)
- Nécessite création de compte et configuration de clés d'accès

**Option C : NAS avec support S3**
- Si vous avez un NAS moderne (Synology, QNAP)
- Souvent disponible via une application

### 4. Espace disque suffisant

- Pour le stockage MinIO : Au moins 20-50 Go libres
- Dépend de la taille de vos applications et volumes

### 5. CLI Velero installé

Nous verrons l'installation dans la section suivante.

## Installation de Velero

### Étape 1 : Installation du CLI Velero

Le CLI Velero est l'outil en ligne de commande qui permet d'interagir avec Velero.

#### Sur Linux

```bash
# Télécharger la dernière version
wget https://github.com/vmware-tanzu/velero/releases/download/v1.12.0/velero-v1.12.0-linux-amd64.tar.gz

# Extraire l'archive
tar -xvzf velero-v1.12.0-linux-amd64.tar.gz

# Déplacer le binaire dans le PATH
sudo mv velero-v1.12.0-linux-amd64/velero /usr/local/bin/

# Vérifier l'installation
velero version
```

**Note** : Vérifiez la dernière version sur https://github.com/vmware-tanzu/velero/releases

#### Sur macOS

```bash
# Avec Homebrew
brew install velero

# Vérifier l'installation
velero version
```

#### Sur Windows (WSL2)

Suivez les mêmes étapes que pour Linux dans votre terminal WSL2.

### Étape 2 : Déployer MinIO (stockage objet local)

Nous allons installer MinIO dans votre cluster MicroK8s pour avoir un stockage objet local.

#### Option A : Installation rapide avec les manifestes

Créez un fichier `minio-deployment.yaml` :

```yaml
---
apiVersion: v1
kind: Namespace
metadata:
  name: velero
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: minio-pvc
  namespace: velero
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 50Gi
  storageClassName: microk8s-hostpath
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: minio
  namespace: velero
spec:
  selector:
    matchLabels:
      app: minio
  template:
    metadata:
      labels:
        app: minio
    spec:
      containers:
      - name: minio
        image: minio/minio:latest
        args:
        - server
        - /data
        - --console-address
        - ":9001"
        env:
        - name: MINIO_ROOT_USER
          value: "minio"
        - name: MINIO_ROOT_PASSWORD
          value: "minio123"
        ports:
        - containerPort: 9000
          name: api
        - containerPort: 9001
          name: console
        volumeMounts:
        - name: data
          mountPath: /data
      volumes:
      - name: data
        persistentVolumeClaim:
          claimName: minio-pvc
---
apiVersion: v1
kind: Service
metadata:
  name: minio
  namespace: velero
spec:
  selector:
    app: minio
  ports:
  - name: api
    port: 9000
    targetPort: 9000
  - name: console
    port: 9001
    targetPort: 9001
  type: NodePort
```

Appliquez la configuration :

```bash
kubectl apply -f minio-deployment.yaml
```

Vérifiez que MinIO est démarré :

```bash
kubectl get pods -n velero
```

Vous devriez voir un pod `minio-xxxxx` en état "Running".

#### Option B : Installation avec Helm (alternative)

Si vous préférez utiliser Helm :

```bash
# Ajouter le repo Helm de MinIO
helm repo add minio https://charts.min.io/

# Installer MinIO
helm install minio minio/minio \
  --namespace velero \
  --create-namespace \
  --set rootUser=minio \
  --set rootPassword=minio123 \
  --set persistence.size=50Gi
```

### Étape 3 : Configurer MinIO

#### Créer le bucket pour Velero

MinIO nécessite un "bucket" (conteneur) pour stocker les sauvegardes.

**Accéder à l'interface web MinIO** :

```bash
# Obtenir le port NodePort
kubectl get svc -n velero minio

# Si le port console est 30001, accédez à :
# http://VOTRE_IP_SERVEUR:30001
```

Dans l'interface MinIO :
1. Connectez-vous avec `minio` / `minio123`
2. Créez un nouveau bucket nommé `velero-backups`
3. Le bucket doit être de type "private"

#### Alternative : Créer le bucket via CLI

Si vous avez installé le client MinIO (`mc`) :

```bash
# Télécharger mc
wget https://dl.min.io/client/mc/release/linux-amd64/mc
chmod +x mc
sudo mv mc /usr/local/bin/

# Configurer l'alias MinIO
mc alias set myminio http://localhost:9000 minio minio123

# Créer le bucket
mc mb myminio/velero-backups
```

### Étape 4 : Créer le fichier de credentials

Velero a besoin des identifiants pour accéder à MinIO.

Créez un fichier `credentials-velero` :

```ini
[default]
aws_access_key_id = minio
aws_secret_access_key = minio123
```

**Important** : Protégez ce fichier car il contient des informations sensibles :

```bash
chmod 600 credentials-velero
```

### Étape 5 : Installer Velero dans le cluster

Maintenant que MinIO est prêt, installons Velero :

```bash
velero install \
  --provider aws \
  --plugins velero/velero-plugin-for-aws:v1.8.0 \
  --bucket velero-backups \
  --secret-file ./credentials-velero \
  --use-volume-snapshots=false \
  --backup-location-config region=minio,s3ForcePathStyle="true",s3Url=http://minio.velero.svc:9000
```

**Explication des paramètres** :

- `--provider aws` : Utilise le plugin AWS (compatible S3/MinIO)
- `--plugins` : Version du plugin à utiliser
- `--bucket velero-backups` : Nom du bucket créé dans MinIO
- `--secret-file` : Fichier contenant les credentials
- `--use-volume-snapshots=false` : Désactive les snapshots natifs (non supportés avec MinIO)
- `--backup-location-config` : Configuration spécifique MinIO
  - `s3ForcePathStyle="true"` : Nécessaire pour MinIO
  - `s3Url` : URL interne du service MinIO dans le cluster

### Étape 6 : Vérifier l'installation

Vérifiez que Velero est correctement installé :

```bash
# Vérifier les pods Velero
kubectl get pods -n velero

# Vérifier la version
velero version

# Vérifier la configuration du backup location
velero backup-location get
```

Vous devriez voir :
- Un pod `velero-xxxxx` en état "Running"
- La version du client et du serveur affichée
- Un backup-location nommé "default" avec status "Available"

## Installation de Restic pour la sauvegarde des volumes

Par défaut, Velero ne sauvegarde que les ressources Kubernetes, pas les données dans les volumes. Pour sauvegarder les volumes, nous utilisons **Restic** (ou son successeur **Kopia**).

### Activer Restic dans Velero

```bash
# Installer le plugin Restic
velero install \
  --provider aws \
  --plugins velero/velero-plugin-for-aws:v1.8.0 \
  --bucket velero-backups \
  --secret-file ./credentials-velero \
  --use-volume-snapshots=false \
  --use-node-agent \
  --backup-location-config region=minio,s3ForcePathStyle="true",s3Url=http://minio.velero.svc:9000
```

**Note** : L'option `--use-node-agent` active le support de sauvegarde des volumes avec Restic/Kopia.

Vérifiez que les node-agents sont démarrés :

```bash
kubectl get pods -n velero -l name=node-agent
```

Vous devriez voir un pod `node-agent` par nœud de votre cluster.

## Concepts clés de Velero

Avant de commencer à utiliser Velero, comprenons quelques concepts fondamentaux.

### 1. Backup (Sauvegarde)

Un **backup** est une copie de l'état de votre cluster à un instant précis. Il contient :
- Les définitions de toutes les ressources Kubernetes sélectionnées
- Optionnellement, les données des volumes persistants

**Caractéristiques** :
- Immuable (ne peut pas être modifié après création)
- Horodaté avec la date/heure de création
- Peut être planifié ou déclenché manuellement

### 2. Schedule (Planification)

Un **schedule** est une sauvegarde automatique récurrente, comme un CronJob.

**Exemple** : "Sauvegarder tous les jours à 2h00 du matin"

### 3. Restore (Restauration)

Une **restauration** recréé les ressources Kubernetes à partir d'une sauvegarde existante.

**Options de restauration** :
- Restauration complète (tout le backup)
- Restauration sélective (seulement certains namespaces ou ressources)
- Restauration dans un cluster différent

### 4. Backup Location

Le **backup location** définit où Velero stocke les sauvegardes (dans notre cas, le bucket MinIO).

### 5. Volume Snapshot Location

Définit où stocker les snapshots de volumes (non utilisé avec MinIO/Restic).

### 6. Backup Hooks

Les **hooks** sont des commandes à exécuter avant ou après une sauvegarde, par exemple :
- Flush des caches d'une base de données
- Arrêt temporaire d'une application
- Scripts de pre-backup ou post-backup

## Utilisation de base de Velero

Maintenant que Velero est installé, voyons comment l'utiliser.

### Créer votre première sauvegarde

#### Sauvegarde complète du cluster

Pour sauvegarder tout le cluster :

```bash
velero backup create my-first-backup
```

**Explication** : Cette commande crée une sauvegarde nommée "my-first-backup" de toutes les ressources dans tous les namespaces.

Suivre la progression :

```bash
velero backup describe my-first-backup
```

Ou en temps réel :

```bash
velero backup logs my-first-backup
```

#### Sauvegarde d'un namespace spécifique

Pour sauvegarder seulement un namespace (par exemple, votre application web) :

```bash
velero backup create backup-webapp \
  --include-namespaces webapp
```

#### Sauvegarde avec les volumes

Pour inclure les données des volumes persistants :

```bash
velero backup create backup-with-volumes \
  --include-namespaces webapp \
  --default-volumes-to-fs-backup
```

**Important** : L'option `--default-volumes-to-fs-backup` active la sauvegarde des volumes avec Restic.

#### Sauvegarde sélective par label

Pour sauvegarder seulement les ressources avec un label spécifique :

```bash
velero backup create backup-production \
  --selector environment=production
```

### Lister les sauvegardes

Voir toutes les sauvegardes disponibles :

```bash
velero backup get
```

Affichage typique :

```
NAME                STATUS      CREATED                         EXPIRES   STORAGE LOCATION
my-first-backup     Completed   2025-01-15 10:30:00 +0100 CET   29d       default
backup-webapp       Completed   2025-01-15 11:00:00 +0100 CET   29d       default
```

### Examiner une sauvegarde en détail

```bash
velero backup describe my-first-backup
```

Cette commande affiche :
- Le statut de la sauvegarde
- Le nombre de ressources sauvegardées
- Les warnings ou erreurs éventuelles
- Les volumes inclus
- La taille de la sauvegarde

### Restaurer depuis une sauvegarde

#### Restauration complète

Pour restaurer tout le contenu d'une sauvegarde :

```bash
velero restore create --from-backup my-first-backup
```

Velero génère automatiquement un nom pour la restauration (ex: `my-first-backup-20250115110000`).

#### Restauration avec un nom spécifique

```bash
velero restore create my-restore --from-backup my-first-backup
```

#### Restauration sélective

Restaurer seulement un namespace :

```bash
velero restore create my-restore \
  --from-backup my-first-backup \
  --include-namespaces webapp
```

Exclure certains namespaces :

```bash
velero restore create my-restore \
  --from-backup my-first-backup \
  --exclude-namespaces kube-system,velero
```

#### Suivre la restauration

```bash
velero restore describe my-restore
velero restore logs my-restore
```

### Supprimer une sauvegarde

Pour supprimer une sauvegarde (et libérer de l'espace) :

```bash
velero backup delete my-first-backup
```

**Attention** : Cette action est irréversible !

Pour supprimer toutes les sauvegardes (avec confirmation) :

```bash
velero backup delete --all --confirm
```

## Planification automatique des sauvegardes

La vraie puissance de Velero réside dans la capacité à planifier des sauvegardes automatiques.

### Créer une sauvegarde quotidienne

Pour sauvegarder tout le cluster tous les jours à 2h00 du matin :

```bash
velero schedule create daily-backup \
  --schedule="0 2 * * *"
```

**Format** : Le format est celui de cron : `minute heure jour mois jour_semaine`

Exemples de planifications :

- `0 2 * * *` : Tous les jours à 2h00
- `0 */6 * * *` : Toutes les 6 heures
- `0 0 * * 0` : Tous les dimanches à minuit
- `0 3 * * 1-5` : Du lundi au vendredi à 3h00

### Planification avec rétention

Pour ne garder que les 7 dernières sauvegardes :

```bash
velero schedule create daily-backup \
  --schedule="0 2 * * *" \
  --ttl 168h
```

**Note** : `168h` = 7 jours. Les sauvegardes plus anciennes seront automatiquement supprimées.

### Planification d'un namespace spécifique

```bash
velero schedule create webapp-backup \
  --schedule="0 */6 * * *" \
  --include-namespaces webapp \
  --default-volumes-to-fs-backup \
  --ttl 72h
```

Cette commande :
- Sauvegarde le namespace "webapp" toutes les 6 heures
- Inclut les volumes persistants
- Garde les sauvegardes pendant 72 heures (3 jours)

### Lister les planifications

```bash
velero schedule get
```

### Modifier une planification

Pour modifier une planification existante, il faut la supprimer et la recréer :

```bash
velero schedule delete daily-backup
velero schedule create daily-backup --schedule="0 3 * * *"
```

### Suspendre une planification

```bash
velero schedule pause daily-backup
```

Reprendre une planification :

```bash
velero schedule unpause daily-backup
```

## Scénarios de restauration courants

Voyons quelques scénarios pratiques de restauration.

### Scénario 1 : Récupération après suppression accidentelle

Vous avez accidentellement supprimé un deployment important :

```bash
# Oops !
kubectl delete deployment webapp -n production

# Restaurer uniquement ce deployment
velero restore create restore-webapp \
  --from-backup latest-backup \
  --include-namespaces production \
  --include-resources deployments \
  --selector app=webapp
```

### Scénario 2 : Retour arrière après une mise à jour problématique

Une mise à jour a cassé votre application, vous voulez revenir en arrière :

```bash
# Identifier la dernière bonne sauvegarde
velero backup get

# Supprimer les ressources actuelles
kubectl delete namespace production

# Restaurer depuis la bonne sauvegarde
velero restore create rollback-production \
  --from-backup backup-before-update \
  --include-namespaces production
```

### Scénario 3 : Migration vers un nouveau cluster

Vous voulez migrer vos applications vers un nouveau cluster MicroK8s :

**Sur l'ancien cluster** :

```bash
# Créer une sauvegarde complète
velero backup create migration-backup \
  --default-volumes-to-fs-backup
```

**Sur le nouveau cluster** :

1. Installer Velero avec le même backup-location (même MinIO)
2. Restaurer :

```bash
velero restore create migration-restore \
  --from-backup migration-backup
```

### Scénario 4 : Clonage d'un namespace

Vous voulez créer un environnement de staging identique à production :

```bash
velero restore create staging-clone \
  --from-backup production-backup \
  --namespace-mappings production:staging
```

Cette commande restaure le namespace "production" en le renommant "staging".

## Surveillance et maintenance de Velero

### Vérifier le statut global

```bash
# Status général
velero version

# Vérifier les backup locations
velero backup-location get

# Vérifier les schedules actives
velero schedule get

# Statistiques des backups
velero backup get
```

### Logs de Velero

Pour diagnostiquer des problèmes :

```bash
# Logs du pod Velero
kubectl logs -n velero deployment/velero

# Logs d'une sauvegarde spécifique
velero backup logs my-backup

# Logs d'une restauration
velero restore logs my-restore
```

### Vérifier l'espace disque

Les sauvegardes peuvent consommer beaucoup d'espace. Vérifiez régulièrement :

```bash
# Dans MinIO, connectez-vous à l'interface web
# Ou via kubectl :
kubectl exec -n velero deployment/minio -- df -h /data
```

### Nettoyer les vieilles sauvegardes

Supprimer manuellement les sauvegardes obsolètes :

```bash
# Lister avec les dates
velero backup get

# Supprimer une sauvegarde spécifique
velero backup delete old-backup-name
```

## Bonnes pratiques avec Velero

### 1. Testez vos restaurations régulièrement

**Créez une procédure de test mensuelle** :

```bash
# 1. Créer un backup de test
velero backup create test-backup --include-namespaces test-app

# 2. Supprimer le namespace
kubectl delete namespace test-app

# 3. Restaurer
velero restore create test-restore --from-backup test-backup

# 4. Vérifier que tout fonctionne
kubectl get all -n test-app
```

### 2. Utilisez des labels et annotations

Ajoutez des métadonnées à vos sauvegardes :

```bash
velero backup create my-backup \
  --labels environment=production,app=webapp \
  --default-volumes-to-fs-backup
```

Cela facilite la recherche et le filtrage des sauvegardes.

### 3. Excluez les ressources non nécessaires

Certaines ressources n'ont pas besoin d'être sauvegardées :

```bash
velero backup create optimized-backup \
  --exclude-namespaces kube-system,kube-public \
  --exclude-resources events,endpoints
```

### 4. Documentez votre stratégie

Créez un fichier `BACKUP_STRATEGY.md` qui documente :
- Quels schedules sont configurés
- Politique de rétention
- Procédure de restauration
- Contacts en cas de problème

### 5. Surveillez les échecs

Configurez des alertes pour les sauvegardes échouées. Vous pouvez :
- Vérifier les logs régulièrement
- Utiliser Prometheus pour monitorer les métriques Velero
- Mettre en place des webhooks de notification

### 6. Chiffrez les sauvegardes

Si vos sauvegardes contiennent des données sensibles, chiffrez le bucket MinIO :

```bash
# Dans MinIO, activez le chiffrement côté serveur
mc encrypt set sse-s3 myminio/velero-backups
```

### 7. Sauvegardez aussi la configuration Velero

Exportez la configuration de Velero elle-même :

```bash
# Exporter les schedules
kubectl get schedules.velero.io -n velero -o yaml > velero-schedules-backup.yaml

# Exporter les backup locations
kubectl get backupstoragelocations.velero.io -n velero -o yaml > velero-locations-backup.yaml
```

### 8. Utilisez des backup hooks pour les bases de données

Pour les bases de données, utilisez des hooks pour garantir la cohérence :

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: postgres
  annotations:
    pre.hook.backup.velero.io/command: '["/bin/bash", "-c", "PGPASSWORD=$POSTGRES_PASSWORD pg_dump -U postgres -d mydb > /tmp/backup.sql"]'
    post.hook.backup.velero.io/command: '["/bin/bash", "-c", "rm -f /tmp/backup.sql"]'
```

## Limitations et considérations

### Limitations de Velero

1. **Performances** :
   - Les sauvegardes avec Restic sont plus lentes que les snapshots natifs
   - La sauvegarde de gros volumes peut prendre du temps

2. **Ressources système** :
   - Restic utilise de la CPU et de la RAM pendant les backups
   - Impact sur les performances du cluster pendant la sauvegarde

3. **Cohérence des données** :
   - Velero ne garantit pas la cohérence applicative sans hooks
   - Les bases de données nécessitent une attention particulière

4. **Limitations de stockage** :
   - Dépend de l'espace disponible dans MinIO
   - Les gros clusters génèrent de grosses sauvegardes

### Considérations spécifiques MicroK8s

1. **Dqlite vs etcd** :
   - MicroK8s utilise Dqlite au lieu d'etcd
   - Velero fonctionne normalement, mais ne sauvegarde pas directement Dqlite
   - Utilisez aussi les snapshots MicroK8s natifs pour une protection complète

2. **Stockage hostpath** :
   - Par défaut, MicroK8s utilise hostpath-storage
   - Les sauvegardes Restic fonctionnent bien avec hostpath
   - Attention lors de la restauration sur un autre nœud

3. **Single-node vs Multi-node** :
   - Sur un cluster single-node, Velero fonctionne parfaitement
   - Sur multi-node, assurez-vous que node-agent tourne sur tous les nœuds

### Quand ne pas utiliser Velero

Velero n'est pas toujours la meilleure solution :

- **Très petit cluster simple** : Git + scripts bash peuvent suffire
- **Cluster très dynamique** : Trop de changements rapides
- **Volumes énormes** : Les sauvegardes prendront trop de temps
- **Ressources limitées** : Velero et Restic consomment des ressources

Dans ces cas, considérez des alternatives :
- GitOps pur (ArgoCD, Flux)
- Snapshots Dqlite natifs MicroK8s
- Scripts de sauvegarde personnalisés
- Outils spécifiques aux applications (pg_dump, mongodump, etc.)

## Dépannage commun

### Backup-location indisponible

**Symptôme** : `velero backup-location get` affiche "Unavailable"

**Solutions** :
```bash
# Vérifier que MinIO est accessible
kubectl get svc -n velero minio

# Tester la connectivité
kubectl run -it --rm debug --image=curlimages/curl --restart=Never -- \
  curl http://minio.velero.svc:9000

# Vérifier les credentials
kubectl get secret -n velero cloud-credentials -o yaml
```

### Sauvegarde bloquée en "InProgress"

**Symptôme** : Une sauvegarde reste indéfiniment en cours

**Solutions** :
```bash
# Vérifier les logs
velero backup logs problematic-backup

# Vérifier les logs du pod Velero
kubectl logs -n velero deployment/velero

# En dernier recours, supprimer et recréer
velero backup delete problematic-backup
```

### Volumes non sauvegardés

**Symptôme** : La sauvegarde réussit mais les volumes ne sont pas inclus

**Solutions** :
```bash
# Vérifier que node-agent fonctionne
kubectl get pods -n velero -l name=node-agent

# Vérifier les annotations du pod
kubectl describe pod -n votre-namespace votre-pod

# Ajouter l'annotation pour forcer la sauvegarde
kubectl annotate pod/votre-pod \
  backup.velero.io/backup-volumes=volume-name
```

### Erreurs de restauration

**Symptôme** : La restauration échoue ou certaines ressources ne sont pas recréées

**Solutions** :
```bash
# Vérifier les logs de restauration
velero restore logs votre-restore

# Restaurer en mode debug
velero restore create debug-restore \
  --from-backup votre-backup \
  --restore-volumes=false \
  --log-level=debug
```

## Alternatives et compléments à Velero

### Alternatives

1. **Kasten K10** : Solution commerciale (version gratuite limitée)
2. **Stash** : Alternative open-source avec moins de fonctionnalités
3. **Scripts personnalisés** : Pour des besoins très spécifiques

### Compléments

1. **Snapshots MicroK8s natifs** : Sauvegarde de Dqlite
2. **GitOps (ArgoCD)** : Pour les configurations
3. **Outils de backup applicatifs** : pg_dump, mysqldump, etc.
4. **Monitoring** : Prometheus + Grafana pour surveiller Velero

## Intégration avec l'écosystème

### Velero + Prometheus

Velero expose des métriques Prometheus. Pour les collecter :

```yaml
apiVersion: v1
kind: Service
metadata:
  name: velero-metrics
  namespace: velero
  labels:
    app: velero
spec:
  selector:
    name: velero
  ports:
  - name: metrics
    port: 8085
    targetPort: 8085
```

Ajoutez ensuite un ServiceMonitor si vous utilisez Prometheus Operator.

### Velero + Grafana

Importez le dashboard Velero pour Grafana (ID: 11055) pour visualiser :
- Nombre de sauvegardes
- Taux de succès
- Durée des opérations
- Taille des backups

### Velero + Notifications

Configurez des webhooks pour recevoir des notifications :

```bash
velero backup create my-backup \
  --status-request-timeout=5m \
  --hooks '{"resources":[{"name":"*","includedNamespaces":["*"],"postHooks":[{"exec":{"command":["/bin/sh","-c","curl -X POST https://hooks.slack.com/services/YOUR/WEBHOOK/URL"]}}]}]}'
```

## Ressources et documentation

### Documentation officielle

- **Site officiel Velero** : https://velero.io/
- **Documentation** : https://velero.io/docs/
- **GitHub** : https://github.com/vmware-tanzu/velero

### Tutoriels et guides

- **Velero Examples** : https://github.com/vmware-tanzu/velero/tree/main/examples
- **Community docs** : Nombreux articles de blog et tutoriels

### Support et communauté

- **Slack Kubernetes** : Canal #velero
- **GitHub Issues** : Pour rapporter des bugs
- **Stack Overflow** : Tag `velero`

## Conclusion

Velero est un outil puissant et mature pour la sauvegarde et la restauration de clusters Kubernetes. Bien qu'il puisse sembler complexe au premier abord, ses capacités en font un choix excellent pour protéger votre environnement MicroK8s.

### Points à retenir

✅ **Velero sauvegarde** : Ressources Kubernetes + volumes persistants
✅ **Installation simple** : CLI + serveur dans le cluster
✅ **Stockage flexible** : S3, MinIO, ou tout stockage compatible
✅ **Automatisation** : Schedules pour des backups réguliers
✅ **Restauration sélective** : Par namespace, par label, par ressource
✅ **Migration facilitée** : Entre clusters différents
✅ **Testez régulièrement** : Une sauvegarde non testée n'est pas fiable

### Prochaines étapes

Maintenant que vous comprenez Velero, dans la prochaine section nous verrons :
- La configuration avancée de Velero
- L'optimisation des performances
- Les stratégies de rétention sophistiquées
- L'intégration dans un plan de reprise d'activité complet

N'oubliez pas : **la meilleure sauvegarde est celle que vous testez régulièrement** !

---

**Checklist de mise en place Velero** :

- [ ] Installer le CLI Velero
- [ ] Déployer MinIO dans le cluster
- [ ] Créer le bucket velero-backups
- [ ] Créer le fichier de credentials
- [ ] Installer Velero dans le cluster
- [ ] Activer le support des volumes (node-agent)
- [ ] Créer une première sauvegarde de test
- [ ] Tester la restauration
- [ ] Configurer un schedule automatique
- [ ] Documenter la procédure de restauration
- [ ] Planifier des tests réguliers

⏭️ [Configuration Velero](/22-sauvegarde-et-restauration/03-configuration-velero.md)
