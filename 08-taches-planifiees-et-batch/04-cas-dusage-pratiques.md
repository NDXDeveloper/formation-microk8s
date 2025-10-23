🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 8.4 Cas d'Usage Pratiques (Backups, ETL, Cleanup)

## Introduction

Dans les sections précédentes, nous avons appris à utiliser les **Jobs** (8.1), les **CronJobs** (8.2), et à gérer les **échecs et reprises** (8.3). Maintenant, mettons en pratique ces connaissances avec des exemples concrets et réalistes que vous rencontrerez dans votre lab personnel ou en production.

Cette section présente trois catégories principales de tâches automatisées :

1. **Backups (Sauvegardes)** : Protéger vos données
2. **ETL (Extract, Transform, Load)** : Traiter et transformer des données
3. **Cleanup (Nettoyage)** : Maintenir un système propre et performant

Chaque exemple est complet, commenté, et prêt à être adapté à vos besoins.

## 1. Backups (Sauvegardes)

Les sauvegardes sont essentielles pour protéger vos données contre les pertes accidentelles, les pannes matérielles, ou les erreurs humaines.

### Principe des Sauvegardes

Une bonne stratégie de sauvegarde suit généralement la règle **3-2-1** :
- **3** copies de vos données
- Sur **2** supports différents
- Dont **1** copie hors site (offsite)

### 1.1 Sauvegarde de Base de Données PostgreSQL

C'est l'un des cas d'usage les plus courants. Voici une solution complète pour sauvegarder automatiquement une base PostgreSQL tous les jours.

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: backup-postgresql
  labels:
    app: postgresql
    component: backup
spec:
  # Exécution tous les jours à 2h du matin
  schedule: "0 2 * * *"

  # Ne jamais exécuter deux sauvegardes en même temps
  concurrencyPolicy: Forbid

  # Garder l'historique des 7 dernières sauvegardes réussies
  successfulJobsHistoryLimit: 7

  # Garder l'historique des 2 derniers échecs pour le débogage
  failedJobsHistoryLimit: 2

  # Si le système était en panne, rattraper dans les 10 minutes
  startingDeadlineSeconds: 600

  jobTemplate:
    spec:
      # Réessayer jusqu'à 2 fois en cas d'échec
      backoffLimit: 2

      # Timeout de 30 minutes pour la sauvegarde
      activeDeadlineSeconds: 1800

      template:
        metadata:
          labels:
            app: postgresql
            job: backup
        spec:
          restartPolicy: OnFailure

          containers:
          - name: postgres-backup
            image: postgres:15

            env:
            # Configuration de connexion depuis un Secret
            - name: PGHOST
              value: "postgresql.default.svc.cluster.local"
            - name: PGPORT
              value: "5432"
            - name: PGDATABASE
              value: "production"
            - name: PGUSER
              valueFrom:
                secretKeyRef:
                  name: postgres-credentials
                  key: username
            - name: PGPASSWORD
              valueFrom:
                secretKeyRef:
                  name: postgres-credentials
                  key: password

            command:
            - /bin/bash
            - -c
            - |
              set -e  # Arrêter en cas d'erreur

              # Variables
              BACKUP_DIR="/backup"
              DATE=$(date +%Y%m%d-%H%M%S)
              BACKUP_FILE="postgres-backup-${DATE}.sql.gz"
              BACKUP_PATH="${BACKUP_DIR}/${BACKUP_FILE}"

              echo "=========================================="
              echo "Sauvegarde PostgreSQL"
              echo "Date: $(date)"
              echo "Base de données: ${PGDATABASE}"
              echo "Fichier: ${BACKUP_FILE}"
              echo "=========================================="

              # Créer le répertoire de sauvegarde s'il n'existe pas
              mkdir -p ${BACKUP_DIR}

              # Vérifier la connexion à la base de données
              echo "Vérification de la connexion..."
              if ! pg_isready -h ${PGHOST} -p ${PGPORT}; then
                echo "❌ Impossible de se connecter à PostgreSQL"
                exit 1
              fi
              echo "✅ Connexion réussie"

              # Effectuer la sauvegarde
              echo "Création de la sauvegarde..."
              pg_dump --clean --if-exists --verbose \
                | gzip > ${BACKUP_PATH}

              # Vérifier que le fichier a été créé
              if [ ! -f ${BACKUP_PATH} ]; then
                echo "❌ Échec : fichier de sauvegarde non créé"
                exit 1
              fi

              # Afficher la taille de la sauvegarde
              BACKUP_SIZE=$(du -h ${BACKUP_PATH} | cut -f1)
              echo "✅ Sauvegarde créée : ${BACKUP_FILE} (${BACKUP_SIZE})"

              # Tester l'intégrité de l'archive
              echo "Vérification de l'intégrité..."
              if gzip -t ${BACKUP_PATH}; then
                echo "✅ Archive valide"
              else
                echo "❌ Archive corrompue"
                exit 1
              fi

              # Nettoyage des anciennes sauvegardes (garder 30 jours)
              echo "Nettoyage des anciennes sauvegardes..."
              find ${BACKUP_DIR} -name "postgres-backup-*.sql.gz" -mtime +30 -delete

              # Liste des sauvegardes existantes
              echo "Sauvegardes disponibles :"
              ls -lh ${BACKUP_DIR}/postgres-backup-*.sql.gz

              echo "=========================================="
              echo "✅ Sauvegarde terminée avec succès"
              echo "Date: $(date)"
              echo "=========================================="

            volumeMounts:
            - name: backup-storage
              mountPath: /backup

            resources:
              requests:
                memory: "256Mi"
                cpu: "100m"
              limits:
                memory: "512Mi"
                cpu: "500m"

          volumes:
          - name: backup-storage
            persistentVolumeClaim:
              claimName: postgres-backup-pvc
---
# PersistentVolumeClaim pour stocker les sauvegardes
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: postgres-backup-pvc
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 50Gi  # Adapter selon vos besoins
  storageClassName: hostpath  # Ou votre StorageClass
---
# Secret pour les credentials (à créer avec vos vraies valeurs)
apiVersion: v1
kind: Secret
metadata:
  name: postgres-credentials
type: Opaque
stringData:
  username: postgres
  password: votre-mot-de-passe-securise
```

**Points importants :**

1. **Sécurité** : Les credentials sont dans un Secret, pas en clair
2. **Compression** : Utilisation de `gzip` pour réduire l'espace disque
3. **Validation** : Vérification de l'intégrité de l'archive
4. **Nettoyage** : Suppression automatique des sauvegardes de plus de 30 jours
5. **Logs détaillés** : Permet de suivre facilement l'exécution
6. **Ressources limitées** : Évite de surcharger le cluster

### 1.2 Sauvegarde de Base de Données MySQL

Similaire à PostgreSQL, mais avec les outils MySQL.

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: backup-mysql
spec:
  schedule: "0 3 * * *"  # 3h du matin
  concurrencyPolicy: Forbid
  successfulJobsHistoryLimit: 7
  failedJobsHistoryLimit: 2

  jobTemplate:
    spec:
      backoffLimit: 2
      activeDeadlineSeconds: 1800

      template:
        spec:
          restartPolicy: OnFailure

          containers:
          - name: mysql-backup
            image: mysql:8.0

            env:
            - name: MYSQL_HOST
              value: "mysql.default.svc.cluster.local"
            - name: MYSQL_DATABASE
              value: "production"
            - name: MYSQL_USER
              valueFrom:
                secretKeyRef:
                  name: mysql-credentials
                  key: username
            - name: MYSQL_PWD  # Variable spéciale MySQL
              valueFrom:
                secretKeyRef:
                  name: mysql-credentials
                  key: password

            command:
            - /bin/bash
            - -c
            - |
              set -e

              BACKUP_DIR="/backup"
              DATE=$(date +%Y%m%d-%H%M%S)
              BACKUP_FILE="mysql-backup-${DATE}.sql.gz"

              echo "Début de la sauvegarde MySQL..."

              mkdir -p ${BACKUP_DIR}

              # Sauvegarde avec mysqldump
              mysqldump \
                --host=${MYSQL_HOST} \
                --user=${MYSQL_USER} \
                --single-transaction \
                --routines \
                --triggers \
                --events \
                ${MYSQL_DATABASE} \
                | gzip > ${BACKUP_DIR}/${BACKUP_FILE}

              echo "✅ Sauvegarde créée : ${BACKUP_FILE}"
              echo "Taille : $(du -h ${BACKUP_DIR}/${BACKUP_FILE} | cut -f1)"

              # Nettoyage
              find ${BACKUP_DIR} -name "mysql-backup-*.sql.gz" -mtime +30 -delete

              echo "✅ Sauvegarde terminée"

            volumeMounts:
            - name: backup-storage
              mountPath: /backup

          volumes:
          - name: backup-storage
            persistentVolumeClaim:
              claimName: mysql-backup-pvc
```

### 1.3 Sauvegarde de Volumes Kubernetes

Pour sauvegarder le contenu d'un PersistentVolume (fichiers, données applicatives).

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: backup-volumes
spec:
  schedule: "0 1 * * *"  # 1h du matin
  concurrencyPolicy: Forbid

  jobTemplate:
    spec:
      template:
        spec:
          restartPolicy: OnFailure

          containers:
          - name: volume-backup
            image: alpine:latest

            command:
            - /bin/sh
            - -c
            - |
              set -e

              apk add --no-cache tar gzip

              BACKUP_DIR="/backup"
              SOURCE_DIR="/data"
              DATE=$(date +%Y%m%d-%H%M%S)
              BACKUP_FILE="volume-backup-${DATE}.tar.gz"

              echo "Sauvegarde du volume..."
              echo "Source : ${SOURCE_DIR}"
              echo "Destination : ${BACKUP_DIR}/${BACKUP_FILE}"

              # Créer l'archive tar compressée
              tar czf ${BACKUP_DIR}/${BACKUP_FILE} -C ${SOURCE_DIR} .

              BACKUP_SIZE=$(du -h ${BACKUP_DIR}/${BACKUP_FILE} | cut -f1)
              echo "✅ Archive créée : ${BACKUP_FILE} (${BACKUP_SIZE})"

              # Vérifier l'archive
              if tar tzf ${BACKUP_DIR}/${BACKUP_FILE} > /dev/null; then
                echo "✅ Archive valide"
              else
                echo "❌ Archive corrompue"
                exit 1
              fi

              # Nettoyage
              find ${BACKUP_DIR} -name "volume-backup-*.tar.gz" -mtime +14 -delete

              echo "✅ Sauvegarde terminée"

            volumeMounts:
            - name: data-to-backup
              mountPath: /data
              readOnly: true  # Lecture seule pour plus de sécurité
            - name: backup-storage
              mountPath: /backup

          volumes:
          - name: data-to-backup
            persistentVolumeClaim:
              claimName: app-data-pvc  # Le volume à sauvegarder
          - name: backup-storage
            persistentVolumeClaim:
              claimName: backup-pvc    # Où stocker les sauvegardes
```

### 1.4 Sauvegarde des Configurations Kubernetes

Sauvegarder les manifestes de tous vos objets Kubernetes.

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: backup-kubernetes-configs
spec:
  schedule: "0 4 * * *"  # 4h du matin

  jobTemplate:
    spec:
      template:
        spec:
          serviceAccountName: backup-sa  # ServiceAccount avec permissions lecture
          restartPolicy: OnFailure

          containers:
          - name: config-backup
            image: bitnami/kubectl:latest

            command:
            - /bin/bash
            - -c
            - |
              set -e

              BACKUP_DIR="/backup"
              DATE=$(date +%Y%m%d-%H%M%S)
              NAMESPACE="default"  # Ou votre namespace

              echo "Sauvegarde des configurations Kubernetes..."

              mkdir -p ${BACKUP_DIR}/${DATE}
              cd ${BACKUP_DIR}/${DATE}

              # Sauvegarder différents types de ressources
              echo "Sauvegarde des Deployments..."
              kubectl get deployments -n ${NAMESPACE} -o yaml > deployments.yaml

              echo "Sauvegarde des Services..."
              kubectl get services -n ${NAMESPACE} -o yaml > services.yaml

              echo "Sauvegarde des ConfigMaps..."
              kubectl get configmaps -n ${NAMESPACE} -o yaml > configmaps.yaml

              echo "Sauvegarde des Secrets..."
              kubectl get secrets -n ${NAMESPACE} -o yaml > secrets.yaml

              echo "Sauvegarde des PersistentVolumeClaims..."
              kubectl get pvc -n ${NAMESPACE} -o yaml > pvcs.yaml

              echo "Sauvegarde des Ingress..."
              kubectl get ingress -n ${NAMESPACE} -o yaml > ingress.yaml

              # Créer une archive
              cd ${BACKUP_DIR}
              tar czf kubernetes-config-${DATE}.tar.gz ${DATE}/
              rm -rf ${DATE}

              echo "✅ Configuration sauvegardée : kubernetes-config-${DATE}.tar.gz"

              # Nettoyage (garder 60 jours)
              find ${BACKUP_DIR} -name "kubernetes-config-*.tar.gz" -mtime +60 -delete

              echo "✅ Sauvegarde terminée"

            volumeMounts:
            - name: backup-storage
              mountPath: /backup

          volumes:
          - name: backup-storage
            persistentVolumeClaim:
              claimName: config-backup-pvc
---
# ServiceAccount avec permissions de lecture
apiVersion: v1
kind: ServiceAccount
metadata:
  name: backup-sa
---
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: backup-role
rules:
- apiGroups: ["", "apps", "networking.k8s.io"]
  resources: ["*"]
  verbs: ["get", "list"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: backup-rolebinding
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: backup-role
subjects:
- kind: ServiceAccount
  name: backup-sa
```

### 1.5 Sauvegarde vers un Service Cloud (S3)

Pour une vraie stratégie de backup, il faut envoyer les sauvegardes hors site.

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: backup-to-s3
spec:
  schedule: "0 5 * * *"  # 5h du matin

  jobTemplate:
    spec:
      template:
        spec:
          restartPolicy: OnFailure

          containers:
          - name: s3-sync
            image: amazon/aws-cli:latest

            env:
            - name: AWS_ACCESS_KEY_ID
              valueFrom:
                secretKeyRef:
                  name: aws-credentials
                  key: access-key-id
            - name: AWS_SECRET_ACCESS_KEY
              valueFrom:
                secretKeyRef:
                  name: aws-credentials
                  key: secret-access-key
            - name: AWS_DEFAULT_REGION
              value: "eu-west-3"  # Paris
            - name: S3_BUCKET
              value: "s3://mon-bucket-backup"

            command:
            - /bin/bash
            - -c
            - |
              set -e

              echo "Synchronisation vers S3..."

              # Synchroniser les sauvegardes vers S3
              aws s3 sync /backup ${S3_BUCKET}/kubernetes-backups/ \
                --storage-class GLACIER_INSTANT_RETRIEVAL \
                --exclude "*" \
                --include "*.sql.gz" \
                --include "*.tar.gz"

              echo "✅ Synchronisation terminée"

              # Lister les fichiers sur S3
              echo "Fichiers sur S3 :"
              aws s3 ls ${S3_BUCKET}/kubernetes-backups/ --recursive --human-readable

            volumeMounts:
            - name: backup-storage
              mountPath: /backup
              readOnly: true

          volumes:
          - name: backup-storage
            persistentVolumeClaim:
              claimName: backup-pvc
```

## 2. ETL (Extract, Transform, Load)

**ETL** signifie **Extract, Transform, Load** (Extraire, Transformer, Charger). C'est un processus classique pour :
1. **Extraire** des données depuis une source
2. **Transformer** ces données (nettoyer, agréger, calculer)
3. **Charger** les données transformées dans une destination

### Exemple d'ETL Complet

Imaginons que vous avez un site e-commerce et que vous voulez générer un rapport quotidien des ventes.

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: etl-rapport-ventes
spec:
  schedule: "0 6 * * *"  # 6h du matin
  concurrencyPolicy: Forbid

  jobTemplate:
    spec:
      backoffLimit: 2
      activeDeadlineSeconds: 3600  # 1 heure max

      template:
        spec:
          restartPolicy: OnFailure

          # InitContainer : Vérifier que la base de données est prête
          initContainers:
          - name: wait-for-database
            image: postgres:15
            command:
            - sh
            - -c
            - |
              until pg_isready -h postgresql.default.svc.cluster.local -p 5432; do
                echo "En attente de PostgreSQL..."
                sleep 2
              done
              echo "PostgreSQL est prêt"

          containers:
          - name: etl-processor
            image: python:3.11

            env:
            - name: DB_HOST
              value: "postgresql.default.svc.cluster.local"
            - name: DB_NAME
              value: "ecommerce"
            - name: DB_USER
              valueFrom:
                secretKeyRef:
                  name: db-credentials
                  key: username
            - name: DB_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: db-credentials
                  key: password

            command:
            - python
            - -c
            - |
              import psycopg2
              import json
              import os
              from datetime import datetime, timedelta

              print("="*50)
              print("ETL - Rapport Quotidien des Ventes")
              print(f"Date : {datetime.now()}")
              print("="*50)

              # ===== EXTRACT (Extraction) =====
              print("\n1. EXTRACTION des données...")

              conn = psycopg2.connect(
                  host=os.environ['DB_HOST'],
                  database=os.environ['DB_NAME'],
                  user=os.environ['DB_USER'],
                  password=os.environ['DB_PASSWORD']
              )
              cursor = conn.cursor()

              # Extraire les ventes de la veille
              hier = datetime.now() - timedelta(days=1)
              date_debut = hier.replace(hour=0, minute=0, second=0)
              date_fin = hier.replace(hour=23, minute=59, second=59)

              query = """
                  SELECT
                      order_id,
                      customer_id,
                      product_id,
                      quantity,
                      price,
                      timestamp
                  FROM orders
                  WHERE timestamp BETWEEN %s AND %s
              """

              cursor.execute(query, (date_debut, date_fin))
              ventes = cursor.fetchall()

              print(f"✅ {len(ventes)} commandes extraites")

              # ===== TRANSFORM (Transformation) =====
              print("\n2. TRANSFORMATION des données...")

              # Calculer les statistiques
              total_ventes = 0
              total_articles = 0
              ventes_par_produit = {}

              for vente in ventes:
                  order_id, customer_id, product_id, quantity, price, timestamp = vente

                  montant = quantity * price
                  total_ventes += montant
                  total_articles += quantity

                  if product_id not in ventes_par_produit:
                      ventes_par_produit[product_id] = {
                          'quantite': 0,
                          'montant': 0
                      }

                  ventes_par_produit[product_id]['quantite'] += quantity
                  ventes_par_produit[product_id]['montant'] += montant

              # Trouver le produit le plus vendu
              produit_top = max(
                  ventes_par_produit.items(),
                  key=lambda x: x[1]['quantite']
              ) if ventes_par_produit else (None, {'quantite': 0, 'montant': 0})

              print(f"✅ Données transformées")
              print(f"   Total des ventes : {total_ventes:.2f}€")
              print(f"   Total des articles : {total_articles}")
              print(f"   Produit le plus vendu : {produit_top[0]}")

              # ===== LOAD (Chargement) =====
              print("\n3. CHARGEMENT des résultats...")

              # Créer la table de rapports si elle n'existe pas
              cursor.execute("""
                  CREATE TABLE IF NOT EXISTS rapports_quotidiens (
                      id SERIAL PRIMARY KEY,
                      date DATE NOT NULL,
                      total_ventes DECIMAL(10, 2),
                      total_articles INTEGER,
                      nombre_commandes INTEGER,
                      produit_top INTEGER,
                      created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
                  )
              """)

              # Insérer le rapport
              cursor.execute("""
                  INSERT INTO rapports_quotidiens
                      (date, total_ventes, total_articles, nombre_commandes, produit_top)
                  VALUES (%s, %s, %s, %s, %s)
              """, (
                  hier.date(),
                  total_ventes,
                  total_articles,
                  len(ventes),
                  produit_top[0]
              ))

              conn.commit()

              print("✅ Rapport inséré dans la base de données")

              # Générer un fichier JSON pour archivage
              rapport = {
                  'date': hier.strftime('%Y-%m-%d'),
                  'total_ventes': float(total_ventes),
                  'total_articles': total_articles,
                  'nombre_commandes': len(ventes),
                  'produit_top': produit_top[0],
                  'ventes_par_produit': {
                      str(k): v for k, v in ventes_par_produit.items()
                  }
              }

              filename = f"/rapports/rapport-{hier.strftime('%Y%m%d')}.json"
              with open(filename, 'w') as f:
                  json.dump(rapport, f, indent=2)

              print(f"✅ Rapport sauvegardé : {filename}")

              cursor.close()
              conn.close()

              print("\n" + "="*50)
              print("✅ ETL terminé avec succès")
              print("="*50)

            volumeMounts:
            - name: rapports
              mountPath: /rapports

            resources:
              requests:
                memory: "256Mi"
                cpu: "200m"
              limits:
                memory: "512Mi"
                cpu: "500m"

          volumes:
          - name: rapports
            persistentVolumeClaim:
              claimName: rapports-pvc
```

### 2.2 ETL : Agrégation de Logs

Extraire les logs, les analyser, et générer des statistiques.

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: etl-analyse-logs
spec:
  schedule: "0 */6 * * *"  # Toutes les 6 heures

  jobTemplate:
    spec:
      template:
        spec:
          restartPolicy: OnFailure

          containers:
          - name: log-analyzer
            image: python:3.11-slim

            command:
            - python
            - -c
            - |
              import re
              import json
              from collections import Counter
              from datetime import datetime

              print("Analyse des logs d'accès...")

              # ===== EXTRACT =====
              print("1. Lecture des logs...")

              with open('/logs/access.log', 'r') as f:
                  lignes = f.readlines()

              print(f"✅ {len(lignes)} lignes de logs lues")

              # ===== TRANSFORM =====
              print("2. Analyse des logs...")

              # Pattern pour parser les logs Apache/Nginx
              pattern = r'(\S+) \S+ \S+ \[(.*?)\] "(\S+) (\S+) \S+" (\d+) (\d+)'

              ips = []
              status_codes = []
              urls = []
              user_agents = []

              for ligne in lignes:
                  match = re.match(pattern, ligne)
                  if match:
                      ip, timestamp, method, url, status, size = match.groups()
                      ips.append(ip)
                      status_codes.append(status)
                      urls.append(url)

              # Statistiques
              ip_counter = Counter(ips)
              status_counter = Counter(status_codes)
              url_counter = Counter(urls)

              stats = {
                  'total_requetes': len(lignes),
                  'ips_uniques': len(set(ips)),
                  'top_10_ips': dict(ip_counter.most_common(10)),
                  'codes_status': dict(status_counter),
                  'top_10_urls': dict(url_counter.most_common(10)),
                  'timestamp': datetime.now().isoformat()
              }

              print(f"✅ Statistiques calculées")
              print(f"   Total requêtes : {stats['total_requetes']}")
              print(f"   IPs uniques : {stats['ips_uniques']}")

              # ===== LOAD =====
              print("3. Sauvegarde des statistiques...")

              output_file = f"/output/log-stats-{datetime.now().strftime('%Y%m%d-%H%M')}.json"
              with open(output_file, 'w') as f:
                  json.dump(stats, f, indent=2)

              print(f"✅ Statistiques sauvegardées : {output_file}")

            volumeMounts:
            - name: logs
              mountPath: /logs
              readOnly: true
            - name: output
              mountPath: /output

          volumes:
          - name: logs
            persistentVolumeClaim:
              claimName: app-logs-pvc
          - name: output
            persistentVolumeClaim:
              claimName: stats-pvc
```

### 2.3 ETL : Synchronisation de Données entre Systèmes

Extraire des données d'une API externe, les transformer, et les charger dans votre base de données.

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: etl-sync-api-externe
spec:
  schedule: "*/30 * * * *"  # Toutes les 30 minutes
  concurrencyPolicy: Replace  # Remplacer l'ancienne exécution si elle tourne encore

  jobTemplate:
    spec:
      backoffLimit: 3

      template:
        spec:
          restartPolicy: OnFailure

          containers:
          - name: api-sync
            image: python:3.11-slim

            env:
            - name: API_URL
              value: "https://api.example.com/data"
            - name: API_KEY
              valueFrom:
                secretKeyRef:
                  name: api-credentials
                  key: api-key
            - name: DB_URL
              valueFrom:
                secretKeyRef:
                  name: db-credentials
                  key: connection-string

            command:
            - python
            - -c
            - |
              import requests
              import psycopg2
              import json
              import os
              from datetime import datetime

              print("="*50)
              print("ETL - Synchronisation API Externe")
              print("="*50)

              # ===== EXTRACT =====
              print("\n1. EXTRACTION depuis l'API...")

              headers = {'Authorization': f'Bearer {os.environ["API_KEY"]}'}

              try:
                  response = requests.get(
                      os.environ['API_URL'],
                      headers=headers,
                      timeout=30
                  )
                  response.raise_for_status()

                  data = response.json()
                  print(f"✅ {len(data)} enregistrements extraits")

              except requests.exceptions.RequestException as e:
                  print(f"❌ Erreur API : {e}")
                  exit(1)

              # ===== TRANSFORM =====
              print("\n2. TRANSFORMATION des données...")

              transformed_data = []

              for item in data:
                  # Nettoyer et normaliser les données
                  transformed = {
                      'external_id': item.get('id'),
                      'name': item.get('name', '').strip().upper(),
                      'value': float(item.get('value', 0)),
                      'category': item.get('category', 'UNKNOWN'),
                      'synced_at': datetime.now()
                  }
                  transformed_data.append(transformed)

              print(f"✅ {len(transformed_data)} enregistrements transformés")

              # ===== LOAD =====
              print("\n3. CHARGEMENT dans la base de données...")

              conn = psycopg2.connect(os.environ['DB_URL'])
              cursor = conn.cursor()

              # Créer la table si nécessaire
              cursor.execute("""
                  CREATE TABLE IF NOT EXISTS external_data (
                      id SERIAL PRIMARY KEY,
                      external_id INTEGER UNIQUE,
                      name VARCHAR(255),
                      value DECIMAL(10, 2),
                      category VARCHAR(100),
                      synced_at TIMESTAMP,
                      created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
                      updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
                  )
              """)

              # Insérer ou mettre à jour les données (UPSERT)
              inserted = 0
              updated = 0

              for item in transformed_data:
                  cursor.execute("""
                      INSERT INTO external_data
                          (external_id, name, value, category, synced_at)
                      VALUES (%s, %s, %s, %s, %s)
                      ON CONFLICT (external_id)
                      DO UPDATE SET
                          name = EXCLUDED.name,
                          value = EXCLUDED.value,
                          category = EXCLUDED.category,
                          synced_at = EXCLUDED.synced_at,
                          updated_at = CURRENT_TIMESTAMP
                      RETURNING (xmax = 0) AS inserted
                  """, (
                      item['external_id'],
                      item['name'],
                      item['value'],
                      item['category'],
                      item['synced_at']
                  ))

                  was_insert = cursor.fetchone()[0]
                  if was_insert:
                      inserted += 1
                  else:
                      updated += 1

              conn.commit()

              print(f"✅ Chargement terminé")
              print(f"   Nouveaux : {inserted}")
              print(f"   Mis à jour : {updated}")

              cursor.close()
              conn.close()

              print("\n" + "="*50)
              print("✅ Synchronisation terminée avec succès")
              print("="*50)

            resources:
              requests:
                memory: "128Mi"
                cpu: "100m"
              limits:
                memory: "256Mi"
                cpu: "200m"
```

## 3. Cleanup (Nettoyage)

Le nettoyage régulier est essentiel pour maintenir un système performant et éviter de manquer d'espace disque.

### 3.1 Nettoyage des Fichiers Temporaires

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: cleanup-temp-files
spec:
  schedule: "0 */12 * * *"  # Toutes les 12 heures

  jobTemplate:
    spec:
      template:
        spec:
          restartPolicy: OnFailure

          containers:
          - name: cleaner
            image: busybox

            command:
            - sh
            - -c
            - |
              set -e

              echo "=========================================="
              echo "Nettoyage des fichiers temporaires"
              echo "Date: $(date)"
              echo "=========================================="

              TEMP_DIR="/tmp"

              # Afficher l'espace disque avant nettoyage
              echo "Espace disque AVANT nettoyage :"
              df -h ${TEMP_DIR}

              # Compter les fichiers avant
              BEFORE=$(find ${TEMP_DIR} -type f | wc -l)
              echo "Fichiers temporaires : ${BEFORE}"

              # Supprimer les fichiers de plus de 7 jours
              echo "Suppression des fichiers de plus de 7 jours..."
              find ${TEMP_DIR} -type f -mtime +7 -delete

              # Supprimer les répertoires vides
              echo "Suppression des répertoires vides..."
              find ${TEMP_DIR} -type d -empty -delete

              # Supprimer les fichiers commençant par .tmp
              echo "Suppression des fichiers .tmp*..."
              find ${TEMP_DIR} -type f -name ".tmp*" -delete

              # Compter les fichiers après
              AFTER=$(find ${TEMP_DIR} -type f | wc -l)
              CLEANED=$((BEFORE - AFTER))

              echo "Fichiers supprimés : ${CLEANED}"
              echo "Fichiers restants : ${AFTER}"

              # Afficher l'espace disque après nettoyage
              echo "Espace disque APRÈS nettoyage :"
              df -h ${TEMP_DIR}

              echo "=========================================="
              echo "✅ Nettoyage terminé"
              echo "=========================================="

            volumeMounts:
            - name: temp
              mountPath: /tmp

          volumes:
          - name: temp
            persistentVolumeClaim:
              claimName: temp-pvc
```

### 3.2 Nettoyage des Logs Applicatifs

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: cleanup-application-logs
spec:
  schedule: "0 3 * * 0"  # Tous les dimanches à 3h

  jobTemplate:
    spec:
      template:
        spec:
          restartPolicy: OnFailure

          containers:
          - name: log-cleaner
            image: busybox

            command:
            - sh
            - -c
            - |
              set -e

              echo "Nettoyage des logs applicatifs..."

              LOG_DIR="/var/log/app"

              # Espace avant
              SPACE_BEFORE=$(du -sh ${LOG_DIR} | cut -f1)
              echo "Espace utilisé AVANT : ${SPACE_BEFORE}"

              # Compresser les logs de plus de 7 jours
              echo "Compression des logs de plus de 7 jours..."
              find ${LOG_DIR} -name "*.log" -type f -mtime +7 -exec gzip {} \;

              # Supprimer les logs compressés de plus de 30 jours
              echo "Suppression des logs de plus de 30 jours..."
              find ${LOG_DIR} -name "*.log.gz" -type f -mtime +30 -delete

              # Tronquer les logs actifs trop volumineux (> 1GB)
              echo "Troncature des logs volumineux..."
              find ${LOG_DIR} -name "*.log" -type f -size +1G -exec truncate -s 100M {} \;

              # Espace après
              SPACE_AFTER=$(du -sh ${LOG_DIR} | cut -f1)
              echo "Espace utilisé APRÈS : ${SPACE_AFTER}"

              # Lister les 10 plus gros fichiers restants
              echo "Les 10 plus gros fichiers :"
              find ${LOG_DIR} -type f -exec du -h {} \; | sort -rh | head -10

              echo "✅ Nettoyage des logs terminé"

            volumeMounts:
            - name: app-logs
              mountPath: /var/log/app

          volumes:
          - name: app-logs
            persistentVolumeClaim:
              claimName: app-logs-pvc
```

### 3.3 Nettoyage des Images Docker Inutilisées

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: cleanup-docker-images
spec:
  schedule: "0 2 * * 6"  # Tous les samedis à 2h

  jobTemplate:
    spec:
      template:
        spec:
          restartPolicy: OnFailure

          # Accès au socket Docker du node
          hostNetwork: true

          containers:
          - name: docker-cleaner
            image: docker:latest

            command:
            - sh
            - -c
            - |
              set -e

              echo "Nettoyage des images Docker..."

              # Afficher l'espace disque avant
              echo "Espace disque AVANT :"
              docker system df

              # Supprimer les images non utilisées
              echo "Suppression des images non utilisées..."
              docker image prune -a -f --filter "until=168h"  # 7 jours

              # Supprimer les conteneurs arrêtés
              echo "Suppression des conteneurs arrêtés..."
              docker container prune -f --filter "until=24h"

              # Supprimer les volumes non utilisés
              echo "Suppression des volumes non utilisés..."
              docker volume prune -f

              # Supprimer les réseaux non utilisés
              echo "Suppression des réseaux non utilisés..."
              docker network prune -f

              # Afficher l'espace disque après
              echo "Espace disque APRÈS :"
              docker system df

              echo "✅ Nettoyage Docker terminé"

            volumeMounts:
            - name: docker-sock
              mountPath: /var/run/docker.sock

            securityContext:
              privileged: true

          volumes:
          - name: docker-sock
            hostPath:
              path: /var/run/docker.sock
              type: Socket
```

**Note de sécurité** : Ce Job nécessite des privilèges élevés. Utilisez-le avec précaution et uniquement si vous comprenez les implications de sécurité.

### 3.4 Nettoyage de Base de Données (Purge des Anciennes Données)

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: cleanup-old-database-records
spec:
  schedule: "0 4 * * 0"  # Tous les dimanches à 4h

  jobTemplate:
    spec:
      backoffLimit: 1
      activeDeadlineSeconds: 7200  # 2 heures max

      template:
        spec:
          restartPolicy: OnFailure

          containers:
          - name: db-cleaner
            image: postgres:15

            env:
            - name: PGHOST
              value: "postgresql.default.svc.cluster.local"
            - name: PGDATABASE
              value: "production"
            - name: PGUSER
              valueFrom:
                secretKeyRef:
                  name: db-credentials
                  key: username
            - name: PGPASSWORD
              valueFrom:
                secretKeyRef:
                  name: db-credentials
                  key: password

            command:
            - /bin/bash
            - -c
            - |
              set -e

              echo "=========================================="
              echo "Nettoyage de la base de données"
              echo "Date: $(date)"
              echo "=========================================="

              # Fonction pour compter les lignes
              count_rows() {
                  psql -t -c "SELECT COUNT(*) FROM $1" | xargs
              }

              # Supprimer les logs de plus de 90 jours
              echo "Suppression des logs anciens..."
              BEFORE=$(count_rows "application_logs")

              psql -c "DELETE FROM application_logs WHERE created_at < NOW() - INTERVAL '90 days'"

              AFTER=$(count_rows "application_logs")
              DELETED=$((BEFORE - AFTER))
              echo "✅ Logs supprimés : ${DELETED}"

              # Supprimer les sessions expirées
              echo "Suppression des sessions expirées..."
              BEFORE=$(count_rows "user_sessions")

              psql -c "DELETE FROM user_sessions WHERE expires_at < NOW()"

              AFTER=$(count_rows "user_sessions")
              DELETED=$((BEFORE - AFTER))
              echo "✅ Sessions supprimées : ${DELETED}"

              # Archiver les anciennes commandes (> 2 ans)
              echo "Archivage des anciennes commandes..."
              BEFORE=$(count_rows "orders")

              psql -c "
                  INSERT INTO orders_archive
                  SELECT * FROM orders
                  WHERE created_at < NOW() - INTERVAL '2 years'
              "

              psql -c "DELETE FROM orders WHERE created_at < NOW() - INTERVAL '2 years'"

              AFTER=$(count_rows "orders")
              ARCHIVED=$((BEFORE - AFTER))
              echo "✅ Commandes archivées : ${ARCHIVED}"

              # Vacuum pour récupérer l'espace
              echo "Optimisation de la base de données..."
              psql -c "VACUUM FULL ANALYZE"

              # Afficher les statistiques
              echo "Statistiques des tables :"
              psql -c "
                  SELECT
                      schemaname,
                      tablename,
                      pg_size_pretty(pg_total_relation_size(schemaname||'.'||tablename)) AS size
                  FROM pg_tables
                  WHERE schemaname = 'public'
                  ORDER BY pg_total_relation_size(schemaname||'.'||tablename) DESC
                  LIMIT 10
              "

              echo "=========================================="
              echo "✅ Nettoyage terminé"
              echo "=========================================="

            resources:
              requests:
                memory: "256Mi"
                cpu: "200m"
              limits:
                memory: "512Mi"
                cpu: "500m"
```

## 4. Autres Cas d'Usage Pratiques

### 4.1 Génération de Rapports Automatiques

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: generate-weekly-report
spec:
  schedule: "0 9 * * 1"  # Tous les lundis à 9h

  jobTemplate:
    spec:
      template:
        spec:
          restartPolicy: OnFailure

          containers:
          - name: report-generator
            image: python:3.11

            command:
            - python
            - -c
            - |
              import smtplib
              from email.mime.text import MIMEText
              from email.mime.multipart import MIMEMultipart
              from datetime import datetime, timedelta
              import os

              print("Génération du rapport hebdomadaire...")

              # Récupérer les données de la semaine passée
              # (simplifié pour l'exemple)

              # Créer le rapport HTML
              html_content = f"""
              <html>
                <body>
                  <h1>Rapport Hebdomadaire - Semaine {datetime.now().isocalendar()[1]}</h1>
                  <h2>Résumé</h2>
                  <ul>
                    <li>Nouvelles inscriptions : 150</li>
                    <li>Ventes totales : 45,230€</li>
                    <li>Taux de conversion : 3.2%</li>
                  </ul>
                  <h2>Top Produits</h2>
                  <ol>
                    <li>Produit A - 85 ventes</li>
                    <li>Produit B - 62 ventes</li>
                    <li>Produit C - 41 ventes</li>
                  </ol>
                </body>
              </html>
              """

              # Envoyer par email
              msg = MIMEMultipart('alternative')
              msg['Subject'] = f"Rapport Hebdomadaire - Semaine {datetime.now().isocalendar()[1]}"
              msg['From'] = "reports@monentreprise.com"
              msg['To'] = "direction@monentreprise.com"

              msg.attach(MIMEText(html_content, 'html'))

              # Connexion SMTP (exemple)
              # smtp = smtplib.SMTP(os.environ['SMTP_HOST'], 587)
              # smtp.starttls()
              # smtp.login(os.environ['SMTP_USER'], os.environ['SMTP_PASS'])
              # smtp.send_message(msg)
              # smtp.quit()

              print("✅ Rapport généré et envoyé")

            env:
            - name: SMTP_HOST
              value: "smtp.example.com"
            - name: SMTP_USER
              valueFrom:
                secretKeyRef:
                  name: smtp-credentials
                  key: username
            - name: SMTP_PASS
              valueFrom:
                secretKeyRef:
                  name: smtp-credentials
                  key: password
```

### 4.2 Vérification de Santé Périodique

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: health-check-services
spec:
  schedule: "*/15 * * * *"  # Toutes les 15 minutes

  jobTemplate:
    spec:
      template:
        spec:
          restartPolicy: OnFailure

          containers:
          - name: health-checker
            image: curlimages/curl:latest

            command:
            - sh
            - -c
            - |
              set -e

              echo "Vérification de santé des services..."

              ERRORS=0

              # Vérifier l'application web
              if curl -f -s -o /dev/null http://web-app.default.svc.cluster.local/health; then
                echo "✅ Application web : OK"
              else
                echo "❌ Application web : ERREUR"
                ERRORS=$((ERRORS + 1))
              fi

              # Vérifier l'API
              if curl -f -s -o /dev/null http://api.default.svc.cluster.local/healthz; then
                echo "✅ API : OK"
              else
                echo "❌ API : ERREUR"
                ERRORS=$((ERRORS + 1))
              fi

              # Vérifier Redis
              if nc -zv redis.default.svc.cluster.local 6379 2>&1 | grep -q succeeded; then
                echo "✅ Redis : OK"
              else
                echo "❌ Redis : ERREUR"
                ERRORS=$((ERRORS + 1))
              fi

              # Vérifier PostgreSQL
              if nc -zv postgresql.default.svc.cluster.local 5432 2>&1 | grep -q succeeded; then
                echo "✅ PostgreSQL : OK"
              else
                echo "❌ PostgreSQL : ERREUR"
                ERRORS=$((ERRORS + 1))
              fi

              if [ $ERRORS -gt 0 ]; then
                echo "⚠️  ${ERRORS} service(s) en erreur"
                # Ici, vous pourriez envoyer une alerte
                exit 1
              else
                echo "✅ Tous les services sont opérationnels"
                exit 0
              fi
```

### 4.3 Rotation des Secrets et Certificats

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: rotate-api-keys
spec:
  schedule: "0 0 1 * *"  # Le 1er de chaque mois à minuit

  jobTemplate:
    spec:
      template:
        spec:
          serviceAccountName: secret-rotator-sa
          restartPolicy: OnFailure

          containers:
          - name: key-rotator
            image: bitnami/kubectl:latest

            command:
            - /bin/bash
            - -c
            - |
              set -e

              echo "Rotation des clés API..."

              # Générer une nouvelle clé API
              NEW_API_KEY=$(openssl rand -hex 32)

              echo "Nouvelle clé générée"

              # Mettre à jour le Secret
              kubectl create secret generic api-key \
                --from-literal=key=${NEW_API_KEY} \
                --dry-run=client -o yaml | kubectl apply -f -

              echo "✅ Secret mis à jour"

              # Redémarrer les Pods qui utilisent cette clé
              kubectl rollout restart deployment api-deployment

              echo "✅ Déploiement redémarré"
              echo "Rotation terminée avec succès"
```

### 4.4 Mise à Jour Automatique de DNS

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: update-dynamic-dns
spec:
  schedule: "*/10 * * * *"  # Toutes les 10 minutes

  jobTemplate:
    spec:
      template:
        spec:
          restartPolicy: OnFailure

          containers:
          - name: dns-updater
            image: curlimages/curl:latest

            env:
            - name: DDNS_DOMAIN
              value: "monlab.exemple.com"
            - name: DDNS_TOKEN
              valueFrom:
                secretKeyRef:
                  name: ddns-credentials
                  key: token

            command:
            - sh
            - -c
            - |
              set -e

              echo "Mise à jour du DNS dynamique..."

              # Récupérer l'IP publique
              PUBLIC_IP=$(curl -s https://api.ipify.org)

              echo "IP publique : ${PUBLIC_IP}"

              # Mettre à jour le DNS (exemple avec API générique)
              curl -X POST "https://api.dns-provider.com/update" \
                -H "Authorization: Bearer ${DDNS_TOKEN}" \
                -d "domain=${DDNS_DOMAIN}" \
                -d "ip=${PUBLIC_IP}"

              echo "✅ DNS mis à jour"
```

## Bonnes Pratiques pour les Cas d'Usage

### 1. Utiliser des Variables d'Environnement

```yaml
env:
- name: RETENTION_DAYS
  value: "30"
- name: MAX_FILE_SIZE
  value: "1G"
```

Facilite la modification des paramètres sans changer le code.

### 2. Implémenter des Notifications

Envoyer des alertes en cas de succès ou d'échec.

```yaml
command:
- sh
- -c
- |
  if backup.sh; then
    # Succès - notification optionnelle
    curl -X POST https://api.slack.com/webhook \
      -d '{"text":"✅ Backup réussi"}'
  else
    # Échec - notification importante
    curl -X POST https://api.slack.com/webhook \
      -d '{"text":"❌ Backup échoué"}'
    exit 1
  fi
```

### 3. Documenter les Jobs

```yaml
metadata:
  name: backup-postgresql
  annotations:
    description: "Sauvegarde quotidienne de PostgreSQL avec rétention de 30 jours"
    contact: "ops@monentreprise.com"
    runbook: "https://wiki.monentreprise.com/runbooks/postgres-backup"
    schedule-human: "Tous les jours à 2h du matin (heure Paris)"
```

### 4. Monitorer les Métriques

```yaml
command:
- sh
- -c
- |
  START_TIME=$(date +%s)

  # Votre code ici
  backup.sh

  END_TIME=$(date +%s)
  DURATION=$((END_TIME - START_TIME))

  echo "Durée : ${DURATION} secondes"

  # Envoyer la métrique à Prometheus (si configuré)
  # echo "backup_duration_seconds ${DURATION}" | curl --data-binary @- \
  #   http://pushgateway:9091/metrics/job/backup
```

### 5. Tester en Mode "Dry Run"

Avant de déployer en production, testez sans effectuer de modifications réelles.

```yaml
env:
- name: DRY_RUN
  value: "true"

command:
- sh
- -c
- |
  if [ "${DRY_RUN}" = "true" ]; then
    echo "MODE DRY RUN - Aucune modification effectuée"
    # Simuler les actions
  else
    # Effectuer les vraies actions
  fi
```

### 6. Versionner les Scripts

```yaml
containers:
- name: backup
  image: mon-backup-script:v2.1.0  # Version spécifique
```

Plutôt que `:latest`, utilisez des versions spécifiques pour la reproductibilité.

### 7. Limiter les Ressources

```yaml
resources:
  requests:
    memory: "256Mi"
    cpu: "100m"
  limits:
    memory: "512Mi"
    cpu: "500m"
```

Évite qu'un Job ne consomme toutes les ressources du cluster.

### 8. Utiliser des Init Containers pour les Prérequis

```yaml
initContainers:
- name: check-dependencies
  image: busybox
  command:
  - sh
  - -c
  - |
    until nc -z database 5432; do
      echo "En attente de la base de données..."
      sleep 2
    done
```

## Patterns de Planification Recommandés

### Heures Creuses pour les Tâches Lourdes

```yaml
# Sauvegardes lourdes : la nuit
schedule: "0 2 * * *"  # 2h du matin

# ETL intensifs : tôt le matin
schedule: "0 5 * * *"  # 5h du matin

# Rapports : début de journée
schedule: "0 8 * * 1-5"  # 8h, du lundi au vendredi
```

### Éviter les Conflits de Ressources

```yaml
# Backup DB : 2h
schedule: "0 2 * * *"

# Cleanup : 3h (après le backup)
schedule: "0 3 * * *"

# ETL : 5h (après le cleanup)
schedule: "0 5 * * *"
```

### Planification par Type de Tâche

| Type de Tâche | Fréquence Recommandée | Exemple |
|---------------|----------------------|---------|
| **Backup critique** | Quotidien | `0 2 * * *` |
| **Backup incrémental** | Toutes les 6h | `0 */6 * * *` |
| **ETL** | Quotidien ou horaire | `0 * * * *` |
| **Cleanup logs** | Hebdomadaire | `0 3 * * 0` |
| **Cleanup fichiers temp** | Quotidien | `0 1 * * *` |
| **Rapports** | Hebdomadaire | `0 9 * * 1` |
| **Health checks** | 15 minutes | `*/15 * * * *` |
| **Sync données** | 30 minutes | `*/30 * * * *` |

## Résumé

Cette section a présenté des cas d'usage pratiques réels pour Jobs et CronJobs :

### Backups (Sauvegardes)
✅ Bases de données (PostgreSQL, MySQL)
✅ Volumes Kubernetes
✅ Configurations Kubernetes
✅ Synchronisation vers le cloud (S3)

### ETL (Extract, Transform, Load)
✅ Rapports quotidiens
✅ Analyse de logs
✅ Synchronisation avec APIs externes
✅ Agrégation et transformation de données

### Cleanup (Nettoyage)
✅ Fichiers temporaires
✅ Logs applicatifs
✅ Images Docker
✅ Anciennes données en base

### Autres Cas d'Usage
✅ Génération de rapports
✅ Vérifications de santé
✅ Rotation de secrets
✅ Mise à jour DNS dynamique

**Points clés à retenir :**

1. **Automatiser ce qui est répétitif** : Backups, cleanups, rapports
2. **Utiliser des Secrets** pour les informations sensibles
3. **Implémenter des vérifications** : Tester l'intégrité des backups
4. **Documenter** : Annotations, logs clairs, runbooks
5. **Monitorer** : Alertes en cas d'échec, métriques de durée
6. **Tester** : Mode dry-run avant production
7. **Planifier intelligemment** : Heures creuses, éviter les conflits

Ces patterns sont la base pour construire une infrastructure Kubernetes fiable et automatisée. Adaptez-les à vos besoins spécifiques !

Dans les prochaines sections, nous verrons comment exposer vos applications avec **MetalLB** (section 9) et configurer le **routage avec Ingress** (section 10).

⏭️ [Load Balancing avec MetalLB](/09-load-balancing-avec-metallb/README.md)
