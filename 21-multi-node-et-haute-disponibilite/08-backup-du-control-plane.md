🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 21.8 Backup du Control Plane

## Introduction

Vous avez construit un cluster multi-node résilient avec haute disponibilité, load balancing et stockage distribué. Mais il reste une question cruciale : **que se passe-t-il si tout le cluster est perdu ?** Incendie, catastrophe naturelle, erreur humaine massive, corruption de données...

C'est là qu'intervient le **backup du control plane**. Dans ce chapitre, nous allons apprendre à :
- Sauvegarder l'état complet de votre cluster
- Comprendre ce qui doit être sauvegardé et pourquoi
- Mettre en place des backups automatiques
- Restaurer un cluster depuis un backup
- Tester régulièrement vos sauvegardes
- Élaborer un plan de reprise d'activité (DRP)

Le backup du control plane est votre **dernière ligne de défense**. Même avec la haute disponibilité, vous devez pouvoir reconstruire votre cluster depuis zéro si nécessaire.

**Règle d'or :** Les backups ne valent rien sans tests de restauration réguliers.

## Pourquoi Sauvegarder le Control Plane ?

### Les Risques Sans Backup

**Scénario 1 : Erreur Humaine**
```bash
# Oups... mauvaise commande
microk8s kubectl delete namespace production --force
```
Sans backup : Tous vos déploiements, services, ConfigMaps, Secrets de production sont **définitivement perdus**.

**Scénario 2 : Corruption de Dqlite**
Un bug, une panne électrique au mauvais moment, un disque défaillant → la base de données Dqlite est corrompue. Le cluster ne démarre plus.

**Scénario 3 : Catastrophe Physique**
Incendie, inondation, vol du matériel → tous les nœuds sont perdus.

**Scénario 4 : Ransomware**
Attaque qui chiffre les données du cluster. Sans backup externe, tout est perdu.

**Scénario 5 : Mise à Jour Ratée**
Une mise à jour de Kubernetes casse le cluster. Impossible de revenir en arrière sans backup.

### Ce Que Vous Perdez Sans Backup

**État du cluster :**
- Tous les Deployments, StatefulSets, DaemonSets
- Tous les Services (ClusterIP, NodePort, LoadBalancer)
- Tous les ConfigMaps et Secrets
- Tous les Ingress et règles réseau
- Tous les RBAC (rôles, permissions)
- Tous les CRDs (Custom Resource Definitions)
- Configuration des addons

**Données applicatives :**
- Volumes persistants (si pas de backup séparé)
- Bases de données
- Fichiers uploadés par les utilisateurs

**Configuration cluster :**
- Settings Kubernetes
- Configuration des nœuds
- Historique des déploiements

### La Haute Disponibilité N'est PAS un Backup

**HA protège contre :** Panne d'un nœud, crash d'un pod
**Backup protège contre :** Perte totale du cluster, erreurs humaines, corruption de données

**Analogie :**
- HA = avoir plusieurs voitures (si une tombe en panne, vous en avez une autre)
- Backup = avoir une assurance tous risques (si toutes vos voitures brûlent, vous pouvez en racheter)

**Les deux sont complémentaires, pas alternatifs !**

## Qu'est-ce Qui Doit Être Sauvegardé ?

### 1. État du Control Plane (Dqlite)

**Dqlite contient TOUT l'état du cluster :**
- Définitions de tous les objets Kubernetes
- ConfigMaps et Secrets
- État actuel vs état désiré
- Historique des changements

**Emplacement :** `/var/snap/microk8s/current/var/kubernetes/backend/`

**C'est la sauvegarde la plus critique !**

### 2. Configurations et Certificats

**Fichiers importants :**
- Certificats TLS du cluster
- Configuration kubeconfig
- Tokens d'authentification
- Configuration des addons

**Emplacements :**
- `/var/snap/microk8s/current/credentials/`
- `/var/snap/microk8s/current/certs/`
- `/var/snap/microk8s/current/args/`

### 3. Définitions YAML

**Best practice :** Stocker tous vos manifestes YAML dans Git (GitOps).

Si vous déployez toujours avec `kubectl apply -f` sans conserver les YAML, vous ne pourrez pas recréer vos applications.

**Solution recommandée :**
```
📁 git-repo/
  📁 kubernetes/
    📁 apps/
      📄 deployment-app1.yaml
      📄 service-app1.yaml
    📁 configs/
      📄 configmap-app1.yaml
      📄 secret-app1.yaml
    📁 ingress/
      📄 ingress-rules.yaml
```

Commitez chaque changement dans Git → backup automatique.

### 4. Volumes Persistants

**Les PV contiennent les données applicatives :**
- Bases de données
- Uploads utilisateurs
- Logs applicatifs

**Avec Longhorn :** Utiliser les snapshots et backups intégrés (vu au chapitre 21.7)

**Sans stockage distribué :** Backups manuels ou avec des outils comme Velero.

### 5. Configuration des Nœuds

**Moins critique mais utile :**
- Configuration réseau (`/etc/netplan/`)
- Configuration firewall
- Packages installés
- Configuration MicroK8s

**Architecture de Backup Complète :**
```
┌────────────────────────────────────────────────────┐
│            STRATÉGIE DE BACKUP COMPLÈTE            │
├────────────────────────────────────────────────────┤
│                                                    │
│  1. Control Plane (Dqlite)                         │
│     → Backup quotidien                             │
│     → Rétention: 7 jours                           │
│     → Stockage externe                             │
│                                                    │
│  2. Manifestes YAML                                │
│     → Git (push à chaque changement)               │
│     → GitHub/GitLab                                │
│                                                    │
│  3. Volumes Persistants                            │
│     → Snapshots Longhorn (quotidien)               │
│     → Backup vers S3/NFS (hebdomadaire)            │
│     → Rétention: 30 jours                          │
│                                                    │
│  4. Secrets et Certificats                         │
│     → Backup chiffré mensuel                       │
│     → Stockage offline (USB, coffre)               │
│                                                    │
│  5. Configuration des Nœuds                        │
│     → Documentation + scripts                      │
│     → Git repository                               │
│                                                    │
└────────────────────────────────────────────────────┘
```

## Méthodes de Backup du Control Plane

### Méthode 1 : Backup Manuel Complet

**Cette méthode sauvegarde tout l'état du cluster en quelques commandes.**

**Sur un nœud du control plane (node1, node2 ou node3) :**

```bash
#!/bin/bash
# backup-cluster-manual.sh

# Variables
BACKUP_DIR="/backups/microk8s"
TIMESTAMP=$(date +%Y%m%d-%H%M%S)
BACKUP_PATH="$BACKUP_DIR/backup-$TIMESTAMP"

# Créer le répertoire de backup
mkdir -p "$BACKUP_PATH"

echo "=== Début du backup MicroK8s ==="
echo "Date: $(date)"
echo "Backup dans: $BACKUP_PATH"

# 1. Arrêter MicroK8s (optionnel, mais garantit la cohérence)
echo "Arrêt de MicroK8s..."
microk8s stop

# 2. Backup de Dqlite (base de données du cluster)
echo "Backup de Dqlite..."
sudo cp -r /var/snap/microk8s/current/var/kubernetes/backend "$BACKUP_PATH/dqlite"

# 3. Backup des certificats et credentials
echo "Backup des certificats..."
sudo cp -r /var/snap/microk8s/current/credentials "$BACKUP_PATH/credentials"
sudo cp -r /var/snap/microk8s/current/certs "$BACKUP_PATH/certs"

# 4. Backup de la configuration
echo "Backup de la configuration..."
sudo cp -r /var/snap/microk8s/current/args "$BACKUP_PATH/args"

# 5. Backup de la version MicroK8s
echo "Backup de la version..."
snap list microk8s > "$BACKUP_PATH/microk8s-version.txt"

# 6. Redémarrer MicroK8s
echo "Redémarrage de MicroK8s..."
microk8s start
microk8s status --wait-ready

# 7. Export de tous les objets Kubernetes (backup logique)
echo "Export des objets Kubernetes..."
microk8s kubectl get all --all-namespaces -o yaml > "$BACKUP_PATH/all-resources.yaml"
microk8s kubectl get configmap --all-namespaces -o yaml > "$BACKUP_PATH/configmaps.yaml"
microk8s kubectl get secret --all-namespaces -o yaml > "$BACKUP_PATH/secrets.yaml"
microk8s kubectl get pvc --all-namespaces -o yaml > "$BACKUP_PATH/pvcs.yaml"
microk8s kubectl get ingress --all-namespaces -o yaml > "$BACKUP_PATH/ingress.yaml"

# 8. Compresser le backup
echo "Compression du backup..."
cd "$BACKUP_DIR"
tar -czf "backup-$TIMESTAMP.tar.gz" "backup-$TIMESTAMP"

# 9. Nettoyer le répertoire non compressé
rm -rf "backup-$TIMESTAMP"

# 10. Copier vers un stockage externe (optionnel)
# scp "backup-$TIMESTAMP.tar.gz" user@backup-server:/backups/
# ou
# rclone copy "backup-$TIMESTAMP.tar.gz" s3:mybucket/microk8s-backups/

echo "=== Backup terminé ==="
echo "Fichier: $BACKUP_DIR/backup-$TIMESTAMP.tar.gz"
echo "Taille: $(du -h "$BACKUP_DIR/backup-$TIMESTAMP.tar.gz" | cut -f1)"
```

**Rendre le script exécutable et le lancer :**
```bash
chmod +x backup-cluster-manual.sh
sudo ./backup-cluster-manual.sh
```

**Avantages :**
- Simple et rapide
- Backup complet
- Pas de dépendances externes

**Inconvénients :**
- Nécessite l'arrêt de MicroK8s (quelques secondes)
- Manuel (sauf si automatisé avec cron)
- Backup "froid" (point dans le temps)

### Méthode 2 : Backup à Chaud (Sans Arrêt)

**Pour éviter l'arrêt du cluster :**

```bash
#!/bin/bash
# backup-cluster-hot.sh

BACKUP_DIR="/backups/microk8s"
TIMESTAMP=$(date +%Y%m%d-%H%M%S)
BACKUP_PATH="$BACKUP_DIR/backup-$TIMESTAMP"

mkdir -p "$BACKUP_PATH"

echo "=== Backup à chaud MicroK8s ==="

# 1. Backup logique via kubectl (sans arrêt)
microk8s kubectl get all --all-namespaces -o yaml > "$BACKUP_PATH/all-resources.yaml"
microk8s kubectl get configmap --all-namespaces -o yaml > "$BACKUP_PATH/configmaps.yaml"
microk8s kubectl get secret --all-namespaces -o yaml > "$BACKUP_PATH/secrets.yaml"
microk8s kubectl get pvc --all-namespaces -o yaml > "$BACKUP_PATH/pvcs.yaml"
microk8s kubectl get pv -o yaml > "$BACKUP_PATH/pvs.yaml"
microk8s kubectl get storageclass -o yaml > "$BACKUP_PATH/storageclasses.yaml"
microk8s kubectl get ingress --all-namespaces -o yaml > "$BACKUP_PATH/ingress.yaml"
microk8s kubectl get networkpolicy --all-namespaces -o yaml > "$BACKUP_PATH/networkpolicies.yaml"

# 2. Backup des CRDs et namespaces
microk8s kubectl get crd -o yaml > "$BACKUP_PATH/crds.yaml"
microk8s kubectl get namespace -o yaml > "$BACKUP_PATH/namespaces.yaml"

# 3. Backup de la version
snap list microk8s > "$BACKUP_PATH/microk8s-version.txt"

# 4. Compression
cd "$BACKUP_DIR"
tar -czf "backup-hot-$TIMESTAMP.tar.gz" "backup-$TIMESTAMP"
rm -rf "backup-$TIMESTAMP"

echo "=== Backup à chaud terminé ==="
echo "Fichier: $BACKUP_DIR/backup-hot-$TIMESTAMP.tar.gz"
```

**Avantages :**
- Pas d'arrêt du cluster
- Peut tourner en production
- Automatisable facilement

**Inconvénients :**
- Ne backup pas Dqlite directement (backup logique uniquement)
- Légèrement moins fiable qu'un backup à froid

### Méthode 3 : Velero (Recommandé pour Production)

**Velero est l'outil de backup standard pour Kubernetes.**

**Fonctionnalités :**
- Backups automatiques programmés
- Restauration sélective
- Migration de clusters
- Backup des volumes persistants
- Interface CLI conviviale

#### Installation de Velero

**1. Installer le CLI Velero sur votre machine :**
```bash
# Linux
wget https://github.com/vmware-tanzu/velero/releases/download/v1.12.0/velero-v1.12.0-linux-amd64.tar.gz
tar -xvf velero-v1.12.0-linux-amd64.tar.gz
sudo mv velero-v1.12.0-linux-amd64/velero /usr/local/bin/
velero version --client-only

# macOS
brew install velero

# Vérifier
velero version --client-only
```

**2. Configurer un backend de stockage (exemple avec MinIO local) :**

**Installer MinIO dans le cluster :**
```bash
# Créer un namespace
microk8s kubectl create namespace velero

# Installer MinIO
microk8s kubectl apply -f - <<EOF
apiVersion: apps/v1
kind: Deployment
metadata:
  name: minio
  namespace: velero
spec:
  replicas: 1
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
        env:
        - name: MINIO_ROOT_USER
          value: "minio"
        - name: MINIO_ROOT_PASSWORD
          value: "minio123"
        ports:
        - containerPort: 9000
        volumeMounts:
        - name: data
          mountPath: /data
      volumes:
      - name: data
        emptyDir: {}
---
apiVersion: v1
kind: Service
metadata:
  name: minio
  namespace: velero
spec:
  type: ClusterIP
  ports:
  - port: 9000
    targetPort: 9000
  selector:
    app: minio
EOF
```

**3. Créer un bucket dans MinIO :**
```bash
# Port-forward vers MinIO
microk8s kubectl port-forward -n velero svc/minio 9000:9000 &

# Installer le client MinIO
wget https://dl.min.io/client/mc/release/linux-amd64/mc
chmod +x mc
sudo mv mc /usr/local/bin/

# Configurer
mc alias set minio http://localhost:9000 minio minio123

# Créer le bucket
mc mb minio/velero
```

**4. Créer un fichier de credentials pour Velero :**
```bash
cat > credentials-velero <<EOF
[default]
aws_access_key_id = minio
aws_secret_access_key = minio123
EOF
```

**5. Installer Velero dans le cluster :**
```bash
velero install \
  --provider aws \
  --plugins velero/velero-plugin-for-aws:v1.8.0 \
  --bucket velero \
  --secret-file ./credentials-velero \
  --use-volume-snapshots=false \
  --backup-location-config region=minio,s3ForcePathStyle="true",s3Url=http://minio.velero.svc:9000
```

**Vérifier l'installation :**
```bash
microk8s kubectl get pods -n velero
```

```
NAME                     READY   STATUS
velero-xxxxxxxxxx-xxxxx  1/1     Running
```

#### Utiliser Velero

**Créer un backup complet :**
```bash
velero backup create backup-complet-$(date +%Y%m%d)
```

**Voir les backups :**
```bash
velero backup get
```

**Créer un backup d'un namespace spécifique :**
```bash
velero backup create backup-production --include-namespaces production
```

**Programmer un backup quotidien :**
```bash
velero schedule create daily-backup --schedule="0 2 * * *"
```

Backup quotidien à 2h du matin.

**Restaurer depuis un backup :**
```bash
# Lister les backups
velero backup get

# Restaurer
velero restore create --from-backup backup-complet-20250101
```

**Voir le statut de la restauration :**
```bash
velero restore get
velero restore describe <nom-restore>
```

**Avantages de Velero :**
- Standard de l'industrie
- Backups automatiques programmés
- Restauration granulaire (namespace, labels, etc.)
- Support des volumes persistants
- Migration de clusters facilitée

**Inconvénients :**
- Plus complexe à configurer
- Nécessite un backend S3-compatible
- Consomme des ressources

## Automatisation des Backups

### CronJob Kubernetes pour Backup Quotidien

**Créer un CronJob qui exécute un backup tous les jours :**

```yaml
# cronjob-backup.yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: cluster-backup
  namespace: kube-system
spec:
  schedule: "0 2 * * *"  # Tous les jours à 2h du matin
  successfulJobsHistoryLimit: 7
  failedJobsHistoryLimit: 3
  jobTemplate:
    spec:
      template:
        spec:
          serviceAccountName: backup-sa
          containers:
          - name: backup
            image: bitnami/kubectl:latest
            command:
            - /bin/bash
            - -c
            - |
              set -e
              TIMESTAMP=$(date +%Y%m%d-%H%M%S)
              BACKUP_DIR="/backups/backup-$TIMESTAMP"

              mkdir -p $BACKUP_DIR

              # Export de tous les objets
              kubectl get all --all-namespaces -o yaml > $BACKUP_DIR/all-resources.yaml
              kubectl get configmap --all-namespaces -o yaml > $BACKUP_DIR/configmaps.yaml
              kubectl get secret --all-namespaces -o yaml > $BACKUP_DIR/secrets.yaml
              kubectl get pvc --all-namespaces -o yaml > $BACKUP_DIR/pvcs.yaml
              kubectl get ingress --all-namespaces -o yaml > $BACKUP_DIR/ingress.yaml
              kubectl get crd -o yaml > $BACKUP_DIR/crds.yaml

              # Compresser
              cd /backups
              tar -czf backup-$TIMESTAMP.tar.gz backup-$TIMESTAMP
              rm -rf backup-$TIMESTAMP

              echo "Backup créé: backup-$TIMESTAMP.tar.gz"

              # Copier vers stockage externe (exemple NFS)
              # cp backup-$TIMESTAMP.tar.gz /mnt/nfs-backup/

              # Nettoyer les anciens backups (garder 7 jours)
              find /backups -name "backup-*.tar.gz" -mtime +7 -delete

            volumeMounts:
            - name: backup-storage
              mountPath: /backups
          restartPolicy: OnFailure
          volumes:
          - name: backup-storage
            persistentVolumeClaim:
              claimName: backup-pvc
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: backup-pvc
  namespace: kube-system
spec:
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 50Gi
  storageClassName: longhorn
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: backup-sa
  namespace: kube-system
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: backup-role
rules:
- apiGroups: ["*"]
  resources: ["*"]
  verbs: ["get", "list"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: backup-binding
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: backup-role
subjects:
- kind: ServiceAccount
  name: backup-sa
  namespace: kube-system
```

**Appliquer :**
```bash
microk8s kubectl apply -f cronjob-backup.yaml
```

**Vérifier les CronJobs :**
```bash
microk8s kubectl get cronjob -n kube-system
```

**Déclencher manuellement (pour tester) :**
```bash
microk8s kubectl create job --from=cronjob/cluster-backup manual-backup-1 -n kube-system
```

**Voir les logs :**
```bash
microk8s kubectl logs -n kube-system job/manual-backup-1
```

### Script Cron sur le Serveur

**Alternative : Cron classique sur un nœud du cluster :**

```bash
# Éditer la crontab
sudo crontab -e

# Ajouter la ligne
0 2 * * * /usr/local/bin/backup-cluster-manual.sh >> /var/log/microk8s-backup.log 2>&1
```

Backup tous les jours à 2h du matin, logs dans `/var/log/microk8s-backup.log`.

## Restauration depuis un Backup

### Scénario 1 : Restauration Complète (Cluster Perdu)

**Situation :** Tous les nœuds sont perdus, vous devez reconstruire le cluster depuis zéro.

**Étapes :**

**1. Reconstruire l'infrastructure :**
```bash
# Sur 3 nouvelles machines (node1, node2, node3)
# Installer MicroK8s (même version que le backup)

# Sur chaque nœud
sudo snap install microk8s --classic --channel=1.28/stable
```

**2. Former le cluster HA :**
```bash
# Sur node1
microk8s add-node

# Sur node2
microk8s join <token>

# Sur node1
microk8s add-node

# Sur node3
microk8s join <token>
```

**3. Restaurer les addons de base :**
```bash
microk8s enable dns
microk8s enable storage
```

**4. Restaurer depuis le backup YAML :**
```bash
# Copier le backup sur node1
scp backup-20250101.tar.gz node1:/tmp/

# Sur node1, extraire
cd /tmp
tar -xzf backup-20250101.tar.gz
cd backup-20250101

# Restaurer les namespaces d'abord
microk8s kubectl apply -f namespaces.yaml

# Restaurer les CRDs
microk8s kubectl apply -f crds.yaml

# Attendre que les CRDs soient prêts
sleep 10

# Restaurer les ConfigMaps et Secrets
microk8s kubectl apply -f configmaps.yaml
microk8s kubectl apply -f secrets.yaml

# Restaurer les StorageClasses
microk8s kubectl apply -f storageclasses.yaml

# Restaurer les PVCs
microk8s kubectl apply -f pvcs.yaml

# Restaurer les ressources applicatives
microk8s kubectl apply -f all-resources.yaml

# Restaurer les Ingress
microk8s kubectl apply -f ingress.yaml
```

**5. Vérifier la restauration :**
```bash
microk8s kubectl get all --all-namespaces
microk8s kubectl get pvc --all-namespaces
```

**6. Restaurer les données des volumes (si sauvegardés séparément) :**
Avec Longhorn : restaurer depuis les snapshots/backups Longhorn.

### Scénario 2 : Restauration Partielle (Namespace Supprimé)

**Situation :** Vous avez accidentellement supprimé le namespace `production`.

**Avec un backup YAML :**
```bash
# Extraire le backup
tar -xzf backup-20250101.tar.gz
cd backup-20250101

# Filtrer uniquement le namespace production
grep -A 10000 "namespace: production" all-resources.yaml > production-only.yaml

# Restaurer
microk8s kubectl apply -f production-only.yaml
```

**Avec Velero :**
```bash
# Restaurer uniquement le namespace production
velero restore create restore-production \
  --from-backup backup-complet-20250101 \
  --include-namespaces production
```

### Scénario 3 : Restauration de Dqlite (Backup Physique)

**Si vous avez un backup de Dqlite (méthode 1 - backup à froid) :**

```bash
# 1. Arrêter MicroK8s sur TOUS les nœuds
# Sur node1, node2, node3
microk8s stop

# 2. Sur un nœud, restaurer Dqlite
sudo rm -rf /var/snap/microk8s/current/var/kubernetes/backend
sudo cp -r /path/to/backup/dqlite /var/snap/microk8s/current/var/kubernetes/backend

# 3. Restaurer les certificats (si nécessaire)
sudo cp -r /path/to/backup/credentials/* /var/snap/microk8s/current/credentials/
sudo cp -r /path/to/backup/certs/* /var/snap/microk8s/current/certs/

# 4. Redémarrer MicroK8s sur tous les nœuds
microk8s start
microk8s status --wait-ready

# 5. Vérifier
microk8s kubectl get nodes
microk8s kubectl get all --all-namespaces
```

**⚠️ Cette méthode est risquée et doit être testée en environnement de dev d'abord !**

## Tests de Restauration

### Pourquoi Tester ?

**Statistique effrayante :** 34% des entreprises découvrent que leurs backups sont corrompus ou incomplets... **lors d'une vraie catastrophe**.

**Un backup non testé n'est pas un backup.**

### Processus de Test Recommandé

**Fréquence :** Tous les trimestres (4 fois par an minimum).

**Processus :**

**1. Environnement de test isolé**
Utilisez un cluster séparé ou des VMs temporaires.

**2. Simulation de panne**
```bash
# Sur le cluster de test
microk8s kubectl delete namespace production
# Ou pire : supprimer tous les nœuds
```

**3. Restauration depuis le backup**
Suivez la procédure de restauration complète.

**4. Validation**
```bash
# Vérifier que tout fonctionne
microk8s kubectl get all --all-namespaces
microk8s kubectl get pvc --all-namespaces

# Tester l'accès aux applications
curl http://<app-url>

# Vérifier les données
# Se connecter à la DB et vérifier les tables
```

**5. Mesurer le RTO et RPO**
- **RTO** (Recovery Time Objective) : Temps pour restaurer
- **RPO** (Recovery Point Objective) : Perte de données acceptable

Exemple :
- RTO = 2 heures (restauration complète en 2h)
- RPO = 24 heures (backup quotidien, perte max = 1 jour de données)

**6. Documenter**
Notez :
- Temps de restauration réel
- Problèmes rencontrés
- Données perdues/non restaurées
- Améliorations nécessaires

### Checklist de Test de Restauration

```markdown
# Test de Restauration - [DATE]

## Préparation
- [ ] Backup récent disponible
- [ ] Environnement de test prêt
- [ ] Documentation à jour

## Restauration
- [ ] Extraction du backup : OK
- [ ] Reconstruction du cluster : OK (temps: ___ min)
- [ ] Restauration des namespaces : OK
- [ ] Restauration des CRDs : OK
- [ ] Restauration des ConfigMaps/Secrets : OK
- [ ] Restauration des PVCs : OK
- [ ] Restauration des applications : OK

## Validation
- [ ] Tous les pods en Running
- [ ] Services accessibles
- [ ] Données intègres
- [ ] Ingress fonctionnel
- [ ] Monitoring opérationnel

## Métriques
- Temps total : ___ heures
- Données perdues : ___ (aucune/quelques/beaucoup)
- Problèmes majeurs : ___

## Actions
- [ ] Documenter les problèmes
- [ ] Mettre à jour la procédure
- [ ] Améliorer les backups si nécessaire

## Résultat
- [ ] ✅ Succès complet
- [ ] ⚠️ Succès avec réserves
- [ ] ❌ Échec (raison: ___)
```

## Plan de Reprise d'Activité (DRP)

### Qu'est-ce qu'un DRP ?

Un **Disaster Recovery Plan** (Plan de Reprise d'Activité) est un document qui décrit :
- Quoi faire en cas de catastrophe
- Qui fait quoi
- Dans quel ordre
- En combien de temps

### Structure d'un DRP Simplifié

```markdown
# Plan de Reprise d'Activité - Cluster Kubernetes

## 1. Contact d'Urgence
- Admin principal : Jean Dupont - 06 XX XX XX XX
- Admin secondaire : Marie Martin - 06 YY YY YY YY
- Support : support@example.com

## 2. Localisation des Backups
- Backups quotidiens : /backups/microk8s/ sur node1
- Backups hebdomadaires : NFS 192.168.1.100:/exports/backups
- Backups mensuels : S3 bucket s3://company-backups/kubernetes/
- Manifestes YAML : GitHub https://github.com/company/k8s-configs

## 3. Inventaire du Cluster
- 3 nœuds : node1 (192.168.1.10), node2 (.11), node3 (.12)
- Version MicroK8s : 1.28/stable
- Addons critiques : dns, storage, ingress, cert-manager, metallb, longhorn
- Applications critiques : production (namespace), database (namespace)

## 4. Procédure de Restauration Complète

### Étape 1 : Évaluation (30 min)
- Identifier l'étendue du sinistre
- Vérifier la disponibilité des backups
- Mobiliser l'équipe

### Étape 2 : Reconstruction Infrastructure (1h)
- Provisionner 3 nouvelles machines
- Installer Ubuntu 22.04
- Configurer réseau (IPs statiques)
- Installer MicroK8s (version : voir backup)

### Étape 3 : Formation du Cluster (30 min)
- Former le cluster HA (3 nœuds)
- Activer les addons de base

### Étape 4 : Restauration des Données (2h)
- Restaurer depuis le dernier backup
- Vérifier l'intégrité
- Restaurer les volumes persistants

### Étape 5 : Validation (1h)
- Tests de connectivité
- Validation des applications
- Monitoring

### Étape 6 : Communication (15 min)
- Informer les utilisateurs
- Documenter l'incident
- Post-mortem

**Temps Total Estimé : 5 heures**

## 5. Contacts Fournisseurs
- Hébergeur : XXX
- Support Canonical : XXX
- DNS : XXX

## 6. Dernière Mise à Jour
- Date : 2025-01-01
- Par : Jean Dupont
- Prochain test : 2025-04-01
```

### RTO et RPO

**Définir vos objectifs :**

**Exemple pour une PME :**
- RTO : 4 heures (le cluster doit être opérationnel en 4h max)
- RPO : 24 heures (perte maximale = 1 jour de données)

**Exemple pour une startup non-critique :**
- RTO : 24 heures
- RPO : 1 semaine

**Exemple pour une entreprise critique :**
- RTO : 1 heure
- RPO : 1 heure

**Adapter votre stratégie de backup en fonction :**
- RPO 1h → Backups toutes les heures (Velero schedulé)
- RPO 24h → Backups quotidiens (CronJob)

## Bonnes Pratiques

### 1. Règle du 3-2-1

**3 copies de vos données :**
- 1 copie en production (le cluster lui-même)
- 1 copie sur le cluster (snapshots Longhorn)
- 1 copie externe (NFS, S3, etc.)

**2 supports différents :**
- Disque dur local
- Stockage cloud/NFS

**1 copie hors-site :**
- Cloud (S3, Backblaze B2)
- Datacentre distant
- USB stocké dans un coffre

### 2. Chiffrer les Backups

**Les backups contiennent des Secrets !**

```bash
# Chiffrer un backup
gpg --symmetric --cipher-algo AES256 backup-20250101.tar.gz

# Déchiffrer
gpg --decrypt backup-20250101.tar.gz.gpg > backup-20250101.tar.gz
```

**Stocker la clé de chiffrement de manière sécurisée** (gestionnaire de mots de passe, coffre).

### 3. Rotation des Backups

**Politique recommandée :**
- **Quotidien** : Garder 7 jours
- **Hebdomadaire** : Garder 4 semaines
- **Mensuel** : Garder 12 mois
- **Annuel** : Garder 3 ans (conformité légale)

**Script de rotation automatique :**
```bash
#!/bin/bash
# rotate-backups.sh

BACKUP_DIR="/backups/microk8s"

# Supprimer les backups quotidiens > 7 jours
find $BACKUP_DIR/daily -name "*.tar.gz" -mtime +7 -delete

# Supprimer les backups hebdomadaires > 28 jours
find $BACKUP_DIR/weekly -name "*.tar.gz" -mtime +28 -delete

# Supprimer les backups mensuels > 365 jours
find $BACKUP_DIR/monthly -name "*.tar.gz" -mtime +365 -delete
```

### 4. Monitoring des Backups

**Alertes essentielles :**
- Backup quotidien échoué
- Espace disque < 20% sur le stockage de backup
- Backup pas testé depuis > 90 jours

**Avec Prometheus :**
```yaml
# Exemple de règle d'alerte
groups:
- name: backup-alerts
  rules:
  - alert: BackupFailed
    expr: kube_job_status_failed{job_name=~"cluster-backup.*"} > 0
    for: 10m
    annotations:
      summary: "Backup du cluster a échoué"
```

### 5. Documenter Tout

**Maintenir à jour :**
- Procédure de backup
- Procédure de restauration
- Inventaire du cluster
- Derniers tests de restauration
- DRP complet

**Stocker la documentation :**
- Dans Git (avec les manifestes)
- Dans un wiki interne
- Copie papier dans un coffre

### 6. GitOps : Version Control pour Tout

**Stocker dans Git :**
```
📁 k8s-infrastructure/
  📁 manifests/
    📁 apps/
    📁 configs/
    📁 monitoring/
  📁 docs/
    📄 DRP.md
    📄 backup-procedure.md
    📄 restore-procedure.md
  📁 scripts/
    📄 backup-cluster.sh
    📄 restore-cluster.sh
  📄 README.md
```

**Avantages :**
- Historique des changements (git log)
- Backup automatique (GitHub, GitLab)
- Collaboration facilitée
- Possibilité de rollback (git revert)

### 7. Automatiser et Oublier

**Principe :** Les backups manuels finissent par être oubliés.

**Solution :**
- CronJob Kubernetes
- Velero avec schedule
- CI/CD qui backup après chaque déploiement

**Une fois configuré, ça doit tourner sans intervention.**

## Outils Complémentaires

### Restic

**Backup incrémental efficace :**
```bash
# Installer Restic
sudo apt install restic

# Initialiser un repo
restic init --repo /mnt/backup-repo

# Backup
restic backup /var/snap/microk8s/current --repo /mnt/backup-repo

# Restaurer
restic restore latest --repo /mnt/backup-repo --target /restore
```

### Kopia

**Alternative moderne à Restic :**
- Chiffrement intégré
- Déduplication
- Snapshots incrémentiels

### Snapshot du Serveur Complet

**Si sur VMs (Proxmox, VMware) :**
Snapshot de la VM entière = backup ultra rapide.

**Attention :** Snapshot ≠ backup (ne protège pas contre corruption du stockage).

## Points Clés à Retenir

1. **HA ≠ Backup** : Les deux sont complémentaires
2. **3-2-1** : 3 copies, 2 supports, 1 hors-site
3. **Tester régulièrement** : Un backup non testé est inutile
4. **Automatiser** : CronJob ou Velero
5. **GitOps** : Versionner tous les manifestes
6. **Chiffrer** : Les backups contiennent des secrets
7. **DRP** : Avoir un plan écrit et à jour
8. **RTO/RPO** : Définir vos objectifs
9. **Rotation** : Garder plusieurs générations de backups
10. **Documenter** : Procédures, tests, résultats

## Prochaines Étapes

Vous maîtrisez maintenant les backups du control plane ! Les prochains chapitres couvriront :

- **21.9** : Stratégies de redondance avancées
- **21.10** : Tests de failover et chaos engineering

Avec des backups réguliers, testés et automatisés, votre cluster est maintenant protégé contre les catastrophes majeures. Vous dormez sur vos deux oreilles ! 🚀

---

**Félicitations !** Vous savez maintenant sauvegarder et restaurer votre cluster Kubernetes. C'est votre police d'assurance contre les catastrophes ! Ne négligez jamais les backups !

⏭️ [Stratégies de redondance](/21-multi-node-et-haute-disponibilite/09-strategies-de-redondance.md)
