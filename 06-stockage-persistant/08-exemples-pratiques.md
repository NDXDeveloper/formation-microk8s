🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 6.8 Exemples pratiques (bases de données, etc.)

## Introduction

Maintenant que nous maîtrisons les concepts du stockage persistant et des StatefulSets, il est temps de passer à la pratique avec des exemples concrets et fonctionnels. Cette section présente des déploiements complets d'applications stateful couramment utilisées.

Chaque exemple inclut :
- Une configuration complète prête à l'emploi
- Les ressources Kubernetes nécessaires (Services, StatefulSets, ConfigMaps, Secrets)
- Des commandes pour tester et vérifier le déploiement
- Des conseils de gestion et de maintenance
- Des solutions aux problèmes courants

Ces exemples sont optimisés pour MicroK8s et un environnement de lab, mais les principes s'appliquent à tout cluster Kubernetes.

## Prérequis

Avant de commencer, assurez-vous d'avoir :

```bash
# MicroK8s installé et démarré
microk8s status

# Addon de stockage activé
microk8s enable hostpath-storage

# Vérifier la StorageClass
microk8s kubectl get storageclass
```

## 1. MySQL - Base de données relationnelle

MySQL est l'une des bases de données les plus populaires au monde. Voyons comment la déployer avec un stockage persistant.

### Configuration complète

```yaml
# mysql-deployment.yaml

# Secret pour le mot de passe root
apiVersion: v1
kind: Secret
metadata:
  name: mysql-secret
  namespace: default
type: Opaque
stringData:
  root-password: "MySecureRootPassword123!"
  user-password: "MyUserPassword456!"
---
# ConfigMap pour la configuration MySQL
apiVersion: v1
kind: ConfigMap
metadata:
  name: mysql-config
  namespace: default
data:
  my.cnf: |
    [mysqld]
    # Configuration de base
    default_authentication_plugin=mysql_native_password
    character-set-server=utf8mb4
    collation-server=utf8mb4_unicode_ci

    # Performance
    max_connections=200
    innodb_buffer_pool_size=256M
    innodb_log_file_size=64M

    # Logs
    log_error=/var/log/mysql/error.log
    slow_query_log=1
    slow_query_log_file=/var/log/mysql/slow.log
    long_query_time=2
---
# Headless Service pour MySQL
apiVersion: v1
kind: Service
metadata:
  name: mysql
  namespace: default
  labels:
    app: mysql
spec:
  ports:
  - port: 3306
    name: mysql
  clusterIP: None
  selector:
    app: mysql
---
# Service pour accès externe (optionnel)
apiVersion: v1
kind: Service
metadata:
  name: mysql-external
  namespace: default
spec:
  type: NodePort
  ports:
  - port: 3306
    targetPort: 3306
    nodePort: 30306
  selector:
    app: mysql
---
# StatefulSet MySQL
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: mysql
  namespace: default
spec:
  serviceName: mysql
  replicas: 1
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
        ports:
        - containerPort: 3306
          name: mysql
        env:
        # Mot de passe root
        - name: MYSQL_ROOT_PASSWORD
          valueFrom:
            secretKeyRef:
              name: mysql-secret
              key: root-password
        # Créer une base de données
        - name: MYSQL_DATABASE
          value: "myapp"
        # Créer un utilisateur
        - name: MYSQL_USER
          value: "appuser"
        - name: MYSQL_PASSWORD
          valueFrom:
            secretKeyRef:
              name: mysql-secret
              key: user-password
        volumeMounts:
        # Données MySQL
        - name: data
          mountPath: /var/lib/mysql
        # Configuration personnalisée
        - name: config
          mountPath: /etc/mysql/conf.d
        # Logs
        - name: logs
          mountPath: /var/log/mysql
        resources:
          requests:
            memory: "512Mi"
            cpu: "500m"
          limits:
            memory: "1Gi"
            cpu: "1000m"
        livenessProbe:
          exec:
            command:
            - mysqladmin
            - ping
            - -h
            - localhost
          initialDelaySeconds: 30
          periodSeconds: 10
          timeoutSeconds: 5
        readinessProbe:
          exec:
            command:
            - mysql
            - -h
            - 127.0.0.1
            - -u
            - root
            - -p${MYSQL_ROOT_PASSWORD}
            - -e
            - "SELECT 1"
          initialDelaySeconds: 10
          periodSeconds: 5
          timeoutSeconds: 3
      volumes:
      # Configuration depuis ConfigMap
      - name: config
        configMap:
          name: mysql-config
      # EmptyDir pour les logs
      - name: logs
        emptyDir: {}
  volumeClaimTemplates:
  - metadata:
      name: data
    spec:
      accessModes: ["ReadWriteOnce"]
      storageClassName: microk8s-hostpath
      resources:
        requests:
          storage: 10Gi
```

### Déploiement

```bash
# Appliquer la configuration
microk8s kubectl apply -f mysql-deployment.yaml

# Vérifier le statut
microk8s kubectl get statefulset mysql
microk8s kubectl get pods -l app=mysql
microk8s kubectl get pvc

# Attendre que le pod soit prêt
microk8s kubectl wait --for=condition=ready pod -l app=mysql --timeout=120s
```

### Test de connexion

```bash
# Test depuis l'intérieur du cluster
microk8s kubectl run mysql-client --rm -it --image=mysql:8.0 -- \
  mysql -h mysql-0.mysql -u appuser -pMyUserPassword456! myapp

# Une fois connecté, tester :
mysql> SHOW DATABASES;
mysql> USE myapp;
mysql> CREATE TABLE test (id INT, name VARCHAR(50));
mysql> INSERT INTO test VALUES (1, 'Hello Kubernetes');
mysql> SELECT * FROM test;
mysql> EXIT;

# Test depuis l'extérieur (via NodePort)
mysql -h <IP-DU-NOEUD> -P 30306 -u appuser -pMyUserPassword456! myapp
```

### Sauvegarde MySQL

```bash
# Script de sauvegarde
cat > backup-mysql.sh << 'EOF'
#!/bin/bash
NAMESPACE="default"
POD_NAME="mysql-0"
BACKUP_DIR="/backup/mysql"
DATE=$(date +%Y%m%d-%H%M%S)

mkdir -p $BACKUP_DIR

echo "🔵 Démarrage de la sauvegarde MySQL..."

# Dump de toutes les bases
microk8s kubectl exec $POD_NAME -n $NAMESPACE -- \
  mysqldump -u root -p${MYSQL_ROOT_PASSWORD} --all-databases \
  --single-transaction --quick --lock-tables=false \
  > $BACKUP_DIR/mysql-backup-$DATE.sql

if [ $? -eq 0 ]; then
  # Compresser
  gzip $BACKUP_DIR/mysql-backup-$DATE.sql
  echo "✅ Sauvegarde terminée: mysql-backup-$DATE.sql.gz"

  # Nettoyer les anciennes sauvegardes (> 7 jours)
  find $BACKUP_DIR -name "mysql-backup-*.sql.gz" -mtime +7 -delete
else
  echo "❌ Erreur lors de la sauvegarde"
  exit 1
fi
EOF

chmod +x backup-mysql.sh
./backup-mysql.sh
```

### Restauration MySQL

```bash
# Restaurer depuis une sauvegarde
gunzip < /backup/mysql/mysql-backup-20240101.sql.gz | \
  microk8s kubectl exec -i mysql-0 -- \
  mysql -u root -pMySecureRootPassword123!
```

## 2. PostgreSQL - Base de données relationnelle avancée

PostgreSQL est une base de données relationnelle puissante, souvent utilisée pour des applications complexes.

### Configuration complète

```yaml
# postgres-deployment.yaml

# Secret pour les credentials
apiVersion: v1
kind: Secret
metadata:
  name: postgres-secret
  namespace: default
type: Opaque
stringData:
  postgres-password: "PostgresSecurePassword123!"
---
# ConfigMap pour la configuration PostgreSQL
apiVersion: v1
kind: ConfigMap
metadata:
  name: postgres-config
  namespace: default
data:
  postgresql.conf: |
    # Configuration PostgreSQL
    listen_addresses = '*'
    max_connections = 100
    shared_buffers = 256MB
    effective_cache_size = 1GB
    maintenance_work_mem = 64MB
    checkpoint_completion_target = 0.9
    wal_buffers = 16MB
    default_statistics_target = 100
    random_page_cost = 1.1
    effective_io_concurrency = 200
    work_mem = 2621kB
    min_wal_size = 1GB
    max_wal_size = 4GB

  pg_hba.conf: |
    # TYPE  DATABASE        USER            ADDRESS                 METHOD
    local   all             all                                     trust
    host    all             all             127.0.0.1/32            trust
    host    all             all             ::1/128                 trust
    host    all             all             0.0.0.0/0               md5
---
# Headless Service
apiVersion: v1
kind: Service
metadata:
  name: postgres
  namespace: default
  labels:
    app: postgres
spec:
  ports:
  - port: 5432
    name: postgres
  clusterIP: None
  selector:
    app: postgres
---
# Service externe (optionnel)
apiVersion: v1
kind: Service
metadata:
  name: postgres-external
  namespace: default
spec:
  type: NodePort
  ports:
  - port: 5432
    targetPort: 5432
    nodePort: 30432
  selector:
    app: postgres
---
# StatefulSet PostgreSQL
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: postgres
  namespace: default
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
      initContainers:
      # Init container pour les permissions
      - name: init-chmod
        image: busybox:1.35
        command:
        - sh
        - -c
        - |
          chown -R 999:999 /var/lib/postgresql/data
          chmod 700 /var/lib/postgresql/data
        volumeMounts:
        - name: data
          mountPath: /var/lib/postgresql/data
      containers:
      - name: postgres
        image: postgres:15-alpine
        ports:
        - containerPort: 5432
          name: postgres
        env:
        - name: POSTGRES_DB
          value: "myapp"
        - name: POSTGRES_USER
          value: "postgres"
        - name: POSTGRES_PASSWORD
          valueFrom:
            secretKeyRef:
              name: postgres-secret
              key: postgres-password
        - name: PGDATA
          value: /var/lib/postgresql/data/pgdata
        volumeMounts:
        - name: data
          mountPath: /var/lib/postgresql/data
        - name: config
          mountPath: /etc/postgresql
        resources:
          requests:
            memory: "512Mi"
            cpu: "500m"
          limits:
            memory: "2Gi"
            cpu: "2000m"
        livenessProbe:
          exec:
            command:
            - pg_isready
            - -U
            - postgres
            - -d
            - myapp
          initialDelaySeconds: 30
          periodSeconds: 10
          timeoutSeconds: 5
          failureThreshold: 3
        readinessProbe:
          exec:
            command:
            - pg_isready
            - -U
            - postgres
            - -d
            - myapp
          initialDelaySeconds: 5
          periodSeconds: 5
          timeoutSeconds: 3
          failureThreshold: 3
      volumes:
      - name: config
        configMap:
          name: postgres-config
  volumeClaimTemplates:
  - metadata:
      name: data
    spec:
      accessModes: ["ReadWriteOnce"]
      storageClassName: microk8s-hostpath
      resources:
        requests:
          storage: 20Gi
```

### Déploiement

```bash
# Déployer PostgreSQL
microk8s kubectl apply -f postgres-deployment.yaml

# Vérifier
microk8s kubectl get statefulset postgres
microk8s kubectl get pods -l app=postgres
microk8s kubectl get pvc

# Attendre le démarrage
microk8s kubectl wait --for=condition=ready pod/postgres-0 --timeout=120s
```

### Test de connexion

```bash
# Connexion depuis l'intérieur du cluster
microk8s kubectl run psql-client --rm -it --image=postgres:15-alpine -- \
  psql -h postgres-0.postgres -U postgres -d myapp

# Tests SQL
postgres=# \l                    -- Lister les bases
postgres=# \c myapp              -- Se connecter à myapp
myapp=# CREATE TABLE users (id SERIAL PRIMARY KEY, name VARCHAR(100));
myapp=# INSERT INTO users (name) VALUES ('Alice'), ('Bob');
myapp=# SELECT * FROM users;
myapp=# \dt                      -- Lister les tables
myapp=# \q                       -- Quitter

# Port-forward pour accès local
microk8s kubectl port-forward postgres-0 5432:5432 &
psql -h localhost -U postgres -d myapp
```

### Sauvegarde PostgreSQL

```bash
# Sauvegarde complète
microk8s kubectl exec postgres-0 -- \
  pg_dumpall -U postgres > /backup/postgres/postgres-backup-$(date +%Y%m%d).sql

# Sauvegarde d'une base spécifique
microk8s kubectl exec postgres-0 -- \
  pg_dump -U postgres myapp > /backup/postgres/myapp-backup-$(date +%Y%m%d).sql

# Avec compression
microk8s kubectl exec postgres-0 -- \
  pg_dump -U postgres -Fc myapp > /backup/postgres/myapp-backup-$(date +%Y%m%d).dump
```

### Restauration PostgreSQL

```bash
# Restaurer une base
cat /backup/postgres/myapp-backup-20240101.sql | \
  microk8s kubectl exec -i postgres-0 -- \
  psql -U postgres -d myapp

# Restaurer depuis un dump compressé
microk8s kubectl exec -i postgres-0 -- \
  pg_restore -U postgres -d myapp < /backup/postgres/myapp-backup-20240101.dump
```

## 3. MongoDB - Base de données NoSQL

MongoDB est une base de données NoSQL orientée documents, idéale pour des données flexibles et évolutives.

### Configuration complète

```yaml
# mongodb-deployment.yaml

# Secret pour le mot de passe
apiVersion: v1
kind: Secret
metadata:
  name: mongodb-secret
  namespace: default
type: Opaque
stringData:
  mongodb-root-password: "MongoSecurePassword123!"
  mongodb-password: "UserPassword456!"
---
# Headless Service
apiVersion: v1
kind: Service
metadata:
  name: mongodb
  namespace: default
  labels:
    app: mongodb
spec:
  ports:
  - port: 27017
    name: mongodb
  clusterIP: None
  selector:
    app: mongodb
---
# Service externe (optionnel)
apiVersion: v1
kind: Service
metadata:
  name: mongodb-external
  namespace: default
spec:
  type: NodePort
  ports:
  - port: 27017
    targetPort: 27017
    nodePort: 30017
  selector:
    app: mongodb
---
# StatefulSet MongoDB
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: mongodb
  namespace: default
spec:
  serviceName: mongodb
  replicas: 1
  selector:
    matchLabels:
      app: mongodb
  template:
    metadata:
      labels:
        app: mongodb
    spec:
      containers:
      - name: mongodb
        image: mongo:7
        ports:
        - containerPort: 27017
          name: mongodb
        env:
        - name: MONGO_INITDB_ROOT_USERNAME
          value: "admin"
        - name: MONGO_INITDB_ROOT_PASSWORD
          valueFrom:
            secretKeyRef:
              name: mongodb-secret
              key: mongodb-root-password
        - name: MONGO_INITDB_DATABASE
          value: "myapp"
        volumeMounts:
        - name: data
          mountPath: /data/db
        - name: config
          mountPath: /data/configdb
        resources:
          requests:
            memory: "512Mi"
            cpu: "500m"
          limits:
            memory: "1Gi"
            cpu: "1000m"
        livenessProbe:
          exec:
            command:
            - mongosh
            - --eval
            - "db.adminCommand('ping')"
          initialDelaySeconds: 30
          periodSeconds: 10
          timeoutSeconds: 5
          failureThreshold: 3
        readinessProbe:
          exec:
            command:
            - mongosh
            - --eval
            - "db.adminCommand('ping')"
          initialDelaySeconds: 5
          periodSeconds: 5
          timeoutSeconds: 3
          failureThreshold: 3
      volumes:
      - name: config
        emptyDir: {}
  volumeClaimTemplates:
  - metadata:
      name: data
    spec:
      accessModes: ["ReadWriteOnce"]
      storageClassName: microk8s-hostpath
      resources:
        requests:
          storage: 15Gi
```

### Déploiement

```bash
# Déployer MongoDB
microk8s kubectl apply -f mongodb-deployment.yaml

# Vérifier
microk8s kubectl get statefulset mongodb
microk8s kubectl get pods -l app=mongodb
microk8s kubectl wait --for=condition=ready pod/mongodb-0 --timeout=120s
```

### Test de connexion

```bash
# Connexion depuis l'intérieur du cluster
microk8s kubectl run mongo-client --rm -it --image=mongo:7 -- \
  mongosh mongodb://admin:MongoSecurePassword123!@mongodb-0.mongodb:27017

# Tests MongoDB
test> show dbs
test> use myapp
myapp> db.users.insertOne({name: "Alice", email: "alice@example.com"})
myapp> db.users.find()
myapp> db.users.countDocuments()
myapp> exit

# Port-forward pour accès local
microk8s kubectl port-forward mongodb-0 27017:27017 &
mongosh mongodb://admin:MongoSecurePassword123!@localhost:27017/myapp
```

### Sauvegarde MongoDB

```bash
# Sauvegarde avec mongodump
microk8s kubectl exec mongodb-0 -- \
  mongodump --username admin --password MongoSecurePassword123! \
  --authenticationDatabase admin --out /tmp/backup

# Copier la sauvegarde localement
microk8s kubectl cp mongodb-0:/tmp/backup ./mongodb-backup-$(date +%Y%m%d)

# Compresser
tar czf mongodb-backup-$(date +%Y%m%d).tar.gz mongodb-backup-$(date +%Y%m%d)
```

### Restauration MongoDB

```bash
# Copier la sauvegarde dans le pod
microk8s kubectl cp ./mongodb-backup-20240101 mongodb-0:/tmp/restore

# Restaurer
microk8s kubectl exec mongodb-0 -- \
  mongorestore --username admin --password MongoSecurePassword123! \
  --authenticationDatabase admin /tmp/restore
```

## 4. Redis - Cache et stockage clé-valeur

Redis est un système de stockage clé-valeur en mémoire, souvent utilisé comme cache ou broker de messages.

### Configuration complète

```yaml
# redis-deployment.yaml

# ConfigMap pour la configuration Redis
apiVersion: v1
kind: ConfigMap
metadata:
  name: redis-config
  namespace: default
data:
  redis.conf: |
    # Configuration Redis
    bind 0.0.0.0
    protected-mode yes
    port 6379

    # Persistence
    save 900 1
    save 300 10
    save 60 10000

    # AOF (Append Only File)
    appendonly yes
    appendfsync everysec

    # Memory
    maxmemory 512mb
    maxmemory-policy allkeys-lru

    # Performance
    tcp-backlog 511
    timeout 0
    tcp-keepalive 300
---
# Secret pour le mot de passe Redis (optionnel)
apiVersion: v1
kind: Secret
metadata:
  name: redis-secret
  namespace: default
type: Opaque
stringData:
  redis-password: "RedisSecurePassword123!"
---
# Headless Service
apiVersion: v1
kind: Service
metadata:
  name: redis
  namespace: default
  labels:
    app: redis
spec:
  ports:
  - port: 6379
    name: redis
  clusterIP: None
  selector:
    app: redis
---
# Service externe (optionnel)
apiVersion: v1
kind: Service
metadata:
  name: redis-external
  namespace: default
spec:
  type: NodePort
  ports:
  - port: 6379
    targetPort: 6379
    nodePort: 30379
  selector:
    app: redis
---
# StatefulSet Redis
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: redis
  namespace: default
spec:
  serviceName: redis
  replicas: 1
  selector:
    matchLabels:
      app: redis
  template:
    metadata:
      labels:
        app: redis
    spec:
      containers:
      - name: redis
        image: redis:7-alpine
        ports:
        - containerPort: 6379
          name: redis
        command:
        - redis-server
        - /etc/redis/redis.conf
        # - --requirepass
        # - $(REDIS_PASSWORD)   # Décommenter pour activer le mot de passe
        # env:
        # - name: REDIS_PASSWORD
        #   valueFrom:
        #     secretKeyRef:
        #       name: redis-secret
        #       key: redis-password
        volumeMounts:
        - name: data
          mountPath: /data
        - name: config
          mountPath: /etc/redis
        resources:
          requests:
            memory: "256Mi"
            cpu: "250m"
          limits:
            memory: "512Mi"
            cpu: "500m"
        livenessProbe:
          exec:
            command:
            - redis-cli
            - ping
          initialDelaySeconds: 30
          periodSeconds: 10
          timeoutSeconds: 5
        readinessProbe:
          exec:
            command:
            - redis-cli
            - ping
          initialDelaySeconds: 5
          periodSeconds: 5
          timeoutSeconds: 3
      volumes:
      - name: config
        configMap:
          name: redis-config
  volumeClaimTemplates:
  - metadata:
      name: data
    spec:
      accessModes: ["ReadWriteOnce"]
      storageClassName: microk8s-hostpath
      resources:
        requests:
          storage: 5Gi
```

### Déploiement

```bash
# Déployer Redis
microk8s kubectl apply -f redis-deployment.yaml

# Vérifier
microk8s kubectl get statefulset redis
microk8s kubectl get pods -l app=redis
microk8s kubectl wait --for=condition=ready pod/redis-0 --timeout=120s
```

### Test de connexion

```bash
# Connexion depuis l'intérieur du cluster
microk8s kubectl run redis-client --rm -it --image=redis:7-alpine -- \
  redis-cli -h redis-0.redis

# Tests Redis
127.0.0.1:6379> PING
127.0.0.1:6379> SET mykey "Hello Kubernetes"
127.0.0.1:6379> GET mykey
127.0.0.1:6379> INCR counter
127.0.0.1:6379> GET counter
127.0.0.1:6379> KEYS *
127.0.0.1:6379> INFO stats
127.0.0.1:6379> EXIT

# Port-forward pour accès local
microk8s kubectl port-forward redis-0 6379:6379 &
redis-cli -h localhost
```

### Sauvegarde Redis

```bash
# Redis sauvegarde automatiquement avec RDB et AOF
# Copier les fichiers de sauvegarde
microk8s kubectl exec redis-0 -- cat /data/dump.rdb > redis-backup-$(date +%Y%m%d).rdb
microk8s kubectl exec redis-0 -- cat /data/appendonly.aof > redis-backup-$(date +%Y%m%d).aof

# Forcer une sauvegarde
microk8s kubectl exec redis-0 -- redis-cli BGSAVE
```

## 5. MariaDB - Alternative à MySQL

MariaDB est un fork de MySQL, offrant des fonctionnalités supplémentaires.

### Configuration complète

```yaml
# mariadb-deployment.yaml

# Secret pour les credentials
apiVersion: v1
kind: Secret
metadata:
  name: mariadb-secret
  namespace: default
type: Opaque
stringData:
  mariadb-root-password: "MariaDBRootPassword123!"
  mariadb-password: "MariaDBUserPassword456!"
---
# Headless Service
apiVersion: v1
kind: Service
metadata:
  name: mariadb
  namespace: default
  labels:
    app: mariadb
spec:
  ports:
  - port: 3306
    name: mariadb
  clusterIP: None
  selector:
    app: mariadb
---
# StatefulSet MariaDB
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: mariadb
  namespace: default
spec:
  serviceName: mariadb
  replicas: 1
  selector:
    matchLabels:
      app: mariadb
  template:
    metadata:
      labels:
        app: mariadb
    spec:
      containers:
      - name: mariadb
        image: mariadb:11
        ports:
        - containerPort: 3306
          name: mariadb
        env:
        - name: MARIADB_ROOT_PASSWORD
          valueFrom:
            secretKeyRef:
              name: mariadb-secret
              key: mariadb-root-password
        - name: MARIADB_DATABASE
          value: "myapp"
        - name: MARIADB_USER
          value: "appuser"
        - name: MARIADB_PASSWORD
          valueFrom:
            secretKeyRef:
              name: mariadb-secret
              key: mariadb-password
        volumeMounts:
        - name: data
          mountPath: /var/lib/mysql
        resources:
          requests:
            memory: "512Mi"
            cpu: "500m"
          limits:
            memory: "1Gi"
            cpu: "1000m"
        livenessProbe:
          exec:
            command:
            - mariadb-admin
            - ping
            - -h
            - localhost
          initialDelaySeconds: 30
          periodSeconds: 10
        readinessProbe:
          exec:
            command:
            - mariadb
            - -h
            - 127.0.0.1
            - -u
            - root
            - -p${MARIADB_ROOT_PASSWORD}
            - -e
            - "SELECT 1"
          initialDelaySeconds: 10
          periodSeconds: 5
  volumeClaimTemplates:
  - metadata:
      name: data
    spec:
      accessModes: ["ReadWriteOnce"]
      storageClassName: microk8s-hostpath
      resources:
        requests:
          storage: 10Gi
```

### Déploiement et test

```bash
# Déployer
microk8s kubectl apply -f mariadb-deployment.yaml

# Tester
microk8s kubectl run mariadb-client --rm -it --image=mariadb:11 -- \
  mariadb -h mariadb-0.mariadb -u appuser -pMariaDBUserPassword456! myapp
```

## 6. Elasticsearch - Recherche et analyse

Elasticsearch est un moteur de recherche et d'analyse distribué.

### Configuration simplifiée (single node)

```yaml
# elasticsearch-deployment.yaml

# Service Headless
apiVersion: v1
kind: Service
metadata:
  name: elasticsearch
  namespace: default
  labels:
    app: elasticsearch
spec:
  ports:
  - port: 9200
    name: http
  - port: 9300
    name: transport
  clusterIP: None
  selector:
    app: elasticsearch
---
# Service externe pour accès HTTP
apiVersion: v1
kind: Service
metadata:
  name: elasticsearch-http
  namespace: default
spec:
  type: NodePort
  ports:
  - port: 9200
    targetPort: 9200
    nodePort: 30920
  selector:
    app: elasticsearch
---
# StatefulSet Elasticsearch
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: elasticsearch
  namespace: default
spec:
  serviceName: elasticsearch
  replicas: 1
  selector:
    matchLabels:
      app: elasticsearch
  template:
    metadata:
      labels:
        app: elasticsearch
    spec:
      initContainers:
      # Augmenter vm.max_map_count (requis pour Elasticsearch)
      - name: increase-vm-max-map
        image: busybox:1.35
        command:
        - sysctl
        - -w
        - vm.max_map_count=262144
        securityContext:
          privileged: true
      # Fixer les permissions
      - name: fix-permissions
        image: busybox:1.35
        command:
        - sh
        - -c
        - chown -R 1000:1000 /usr/share/elasticsearch/data
        securityContext:
          privileged: true
        volumeMounts:
        - name: data
          mountPath: /usr/share/elasticsearch/data
      containers:
      - name: elasticsearch
        image: docker.elastic.co/elasticsearch/elasticsearch:8.11.0
        ports:
        - containerPort: 9200
          name: http
        - containerPort: 9300
          name: transport
        env:
        # Configuration single node
        - name: discovery.type
          value: "single-node"
        - name: cluster.name
          value: "k8s-logs"
        - name: node.name
          valueFrom:
            fieldRef:
              fieldPath: metadata.name
        - name: ES_JAVA_OPTS
          value: "-Xms512m -Xmx512m"
        # Désactiver la sécurité pour simplicité (développement uniquement)
        - name: xpack.security.enabled
          value: "false"
        - name: xpack.security.enrollment.enabled
          value: "false"
        volumeMounts:
        - name: data
          mountPath: /usr/share/elasticsearch/data
        resources:
          requests:
            memory: "1Gi"
            cpu: "500m"
          limits:
            memory: "2Gi"
            cpu: "2000m"
        readinessProbe:
          httpGet:
            path: /_cluster/health
            port: 9200
          initialDelaySeconds: 30
          periodSeconds: 10
        livenessProbe:
          httpGet:
            path: /_cluster/health
            port: 9200
          initialDelaySeconds: 60
          periodSeconds: 10
  volumeClaimTemplates:
  - metadata:
      name: data
    spec:
      accessModes: ["ReadWriteOnce"]
      storageClassName: microk8s-hostpath
      resources:
        requests:
          storage: 20Gi
```

### Déploiement et test

```bash
# Déployer Elasticsearch
microk8s kubectl apply -f elasticsearch-deployment.yaml

# Attendre le démarrage (peut prendre 1-2 minutes)
microk8s kubectl wait --for=condition=ready pod/elasticsearch-0 --timeout=300s

# Tester l'API REST
microk8s kubectl run curl --rm -it --image=curlimages/curl -- \
  curl http://elasticsearch-0.elasticsearch:9200

# Créer un index et insérer des données
microk8s kubectl run curl --rm -it --image=curlimages/curl -- \
  curl -X POST "http://elasticsearch-0.elasticsearch:9200/myindex/_doc/1" \
  -H 'Content-Type: application/json' \
  -d '{"title":"Hello Kubernetes","content":"Elasticsearch on K8s"}'

# Rechercher
microk8s kubectl run curl --rm -it --image=curlimages/curl -- \
  curl "http://elasticsearch-0.elasticsearch:9200/myindex/_search?q=Kubernetes"

# Port-forward pour accès local
microk8s kubectl port-forward elasticsearch-0 9200:9200 &
curl http://localhost:9200/_cluster/health
```

## 7. Application complète : WordPress avec MySQL

Exemple d'application complète avec base de données.

### Configuration WordPress + MySQL

```yaml
# wordpress-complete.yaml

# Namespace dédié
apiVersion: v1
kind: Namespace
metadata:
  name: wordpress
---
# Secret MySQL
apiVersion: v1
kind: Secret
metadata:
  name: mysql-secret
  namespace: wordpress
type: Opaque
stringData:
  mysql-root-password: "MySQLRootPassword123!"
  mysql-password: "WordPressDBPassword456!"
---
# Service MySQL
apiVersion: v1
kind: Service
metadata:
  name: mysql
  namespace: wordpress
spec:
  ports:
  - port: 3306
  clusterIP: None
  selector:
    app: mysql
---
# StatefulSet MySQL
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: mysql
  namespace: wordpress
spec:
  serviceName: mysql
  replicas: 1
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
        ports:
        - containerPort: 3306
        env:
        - name: MYSQL_ROOT_PASSWORD
          valueFrom:
            secretKeyRef:
              name: mysql-secret
              key: mysql-root-password
        - name: MYSQL_DATABASE
          value: wordpress
        - name: MYSQL_USER
          value: wordpress
        - name: MYSQL_PASSWORD
          valueFrom:
            secretKeyRef:
              name: mysql-secret
              key: mysql-password
        volumeMounts:
        - name: data
          mountPath: /var/lib/mysql
        resources:
          requests:
            memory: "512Mi"
            cpu: "500m"
          limits:
            memory: "1Gi"
            cpu: "1000m"
  volumeClaimTemplates:
  - metadata:
      name: data
    spec:
      accessModes: ["ReadWriteOnce"]
      resources:
        requests:
          storage: 10Gi
---
# Service WordPress
apiVersion: v1
kind: Service
metadata:
  name: wordpress
  namespace: wordpress
spec:
  type: NodePort
  ports:
  - port: 80
    targetPort: 80
    nodePort: 30080
  selector:
    app: wordpress
---
# Deployment WordPress
apiVersion: apps/v1
kind: Deployment
metadata:
  name: wordpress
  namespace: wordpress
spec:
  replicas: 2
  selector:
    matchLabels:
      app: wordpress
  template:
    metadata:
      labels:
        app: wordpress
    spec:
      containers:
      - name: wordpress
        image: wordpress:6.4-apache
        ports:
        - containerPort: 80
        env:
        - name: WORDPRESS_DB_HOST
          value: mysql-0.mysql
        - name: WORDPRESS_DB_NAME
          value: wordpress
        - name: WORDPRESS_DB_USER
          value: wordpress
        - name: WORDPRESS_DB_PASSWORD
          valueFrom:
            secretKeyRef:
              name: mysql-secret
              key: mysql-password
        volumeMounts:
        - name: wordpress-data
          mountPath: /var/www/html
        resources:
          requests:
            memory: "256Mi"
            cpu: "250m"
          limits:
            memory: "512Mi"
            cpu: "500m"
        readinessProbe:
          httpGet:
            path: /
            port: 80
          initialDelaySeconds: 10
          periodSeconds: 5
  volumeClaimTemplates:
  - metadata:
      name: wordpress-data
    spec:
      accessModes: ["ReadWriteOnce"]
      resources:
        requests:
          storage: 5Gi
```

**Note** : WordPress nécessite ReadWriteMany pour plusieurs replicas avec stockage partagé, ou utiliser un StatefulSet avec un replica.

### Déploiement

```bash
# Déployer l'application complète
microk8s kubectl apply -f wordpress-complete.yaml

# Vérifier
microk8s kubectl get all -n wordpress
microk8s kubectl get pvc -n wordpress

# Accéder à WordPress
# http://<IP-NOEUD>:30080
```

## Bonnes pratiques générales

### 1. Utiliser des namespaces

```bash
# Créer un namespace par application
microk8s kubectl create namespace production
microk8s kubectl create namespace staging
microk8s kubectl create namespace development
```

### 2. Définir des resource limits

Toujours définir requests et limits :

```yaml
resources:
  requests:
    memory: "512Mi"
    cpu: "500m"
  limits:
    memory: "1Gi"
    cpu: "1000m"
```

### 3. Utiliser des probes

Définir liveness et readiness probes pour chaque base de données :

```yaml
livenessProbe:
  exec:
    command: [...]
  initialDelaySeconds: 30
  periodSeconds: 10
readinessProbe:
  exec:
    command: [...]
  initialDelaySeconds: 5
  periodSeconds: 5
```

### 4. Sécuriser avec des Secrets

Ne jamais mettre de mots de passe en clair dans les manifestes :

```yaml
# ❌ Mauvais
env:
- name: DB_PASSWORD
  value: "password123"

# ✅ Bon
env:
- name: DB_PASSWORD
  valueFrom:
    secretKeyRef:
      name: db-secret
      key: password
```

### 5. Prévoir les sauvegardes

Automatiser les sauvegardes avec des CronJobs :

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: mysql-backup
spec:
  schedule: "0 2 * * *"  # Tous les jours à 2h
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: backup
            image: mysql:8.0
            command:
            - /bin/sh
            - -c
            - mysqldump -h mysql-0.mysql -u root -p${MYSQL_ROOT_PASSWORD} --all-databases > /backup/mysql-$(date +\%Y\%m\%d).sql
            env:
            - name: MYSQL_ROOT_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: mysql-secret
                  key: root-password
            volumeMounts:
            - name: backup
              mountPath: /backup
          volumes:
          - name: backup
            hostPath:
              path: /backup/mysql
          restartPolicy: OnFailure
```

### 6. Monitoring et alerting

Surveiller les métriques importantes :

```bash
# Vérifier l'utilisation du stockage
microk8s kubectl get pvc --all-namespaces

# Vérifier les ressources
microk8s kubectl top pods -n <namespace>

# Vérifier les logs
microk8s kubectl logs -f <pod-name> -n <namespace>
```

## Scripts utiles

### Script de déploiement rapide

```bash
#!/bin/bash
# deploy-database.sh

DB_TYPE=$1  # mysql, postgres, mongodb, redis

if [ -z "$DB_TYPE" ]; then
  echo "Usage: $0 <mysql|postgres|mongodb|redis>"
  exit 1
fi

echo "🚀 Déploiement de $DB_TYPE..."

# Créer le namespace si nécessaire
microk8s kubectl create namespace databases --dry-run=client -o yaml | microk8s kubectl apply -f -

# Déployer selon le type
case $DB_TYPE in
  mysql)
    microk8s kubectl apply -f mysql-deployment.yaml -n databases
    ;;
  postgres)
    microk8s kubectl apply -f postgres-deployment.yaml -n databases
    ;;
  mongodb)
    microk8s kubectl apply -f mongodb-deployment.yaml -n databases
    ;;
  redis)
    microk8s kubectl apply -f redis-deployment.yaml -n databases
    ;;
  *)
    echo "❌ Type de base de données non reconnu"
    exit 1
    ;;
esac

# Attendre le démarrage
echo "⏳ Attente du démarrage..."
microk8s kubectl wait --for=condition=ready pod -l app=$DB_TYPE -n databases --timeout=300s

if [ $? -eq 0 ]; then
  echo "✅ $DB_TYPE déployé avec succès !"
  microk8s kubectl get pods -n databases -l app=$DB_TYPE
  microk8s kubectl get pvc -n databases
else
  echo "❌ Erreur lors du déploiement"
  exit 1
fi
```

### Script de sauvegarde universel

```bash
#!/bin/bash
# backup-database.sh

DB_TYPE=$1
NAMESPACE=${2:-default}
BACKUP_DIR="/backup/databases"
DATE=$(date +%Y%m%d-%H%M%S)

mkdir -p $BACKUP_DIR

case $DB_TYPE in
  mysql)
    POD=$(microk8s kubectl get pod -n $NAMESPACE -l app=mysql -o jsonpath='{.items[0].metadata.name}')
    microk8s kubectl exec $POD -n $NAMESPACE -- \
      mysqldump -u root -p${MYSQL_ROOT_PASSWORD} --all-databases \
      > $BACKUP_DIR/mysql-$DATE.sql
    ;;
  postgres)
    POD=$(microk8s kubectl get pod -n $NAMESPACE -l app=postgres -o jsonpath='{.items[0].metadata.name}')
    microk8s kubectl exec $POD -n $NAMESPACE -- \
      pg_dumpall -U postgres \
      > $BACKUP_DIR/postgres-$DATE.sql
    ;;
  mongodb)
    POD=$(microk8s kubectl get pod -n $NAMESPACE -l app=mongodb -o jsonpath='{.items[0].metadata.name}')
    microk8s kubectl exec $POD -n $NAMESPACE -- \
      mongodump --archive --gzip \
      > $BACKUP_DIR/mongodb-$DATE.gz
    ;;
  *)
    echo "Type non supporté"
    exit 1
    ;;
esac

echo "✅ Sauvegarde terminée: $BACKUP_DIR/$DB_TYPE-$DATE.*"
```

## Dépannage général

### Problème : Pod bloqué en "Pending"

```bash
# Vérifier les événements
microk8s kubectl describe pod <pod-name>

# Vérifier les PVC
microk8s kubectl get pvc

# Vérifier la StorageClass
microk8s kubectl get storageclass
```

### Problème : Échec de connexion à la base de données

```bash
# Vérifier que le pod est Ready
microk8s kubectl get pods

# Vérifier les logs
microk8s kubectl logs <pod-name>

# Tester la connectivité réseau
microk8s kubectl exec <pod-name> -- ping google.com

# Vérifier le Service DNS
microk8s kubectl exec <pod-name> -- nslookup <service-name>
```

### Problème : Données perdues après redémarrage

```bash
# Vérifier que le PVC existe
microk8s kubectl get pvc

# Vérifier que le PVC est bien monté
microk8s kubectl describe pod <pod-name> | grep -A5 Volumes

# Vérifier le contenu du volume
microk8s kubectl exec <pod-name> -- ls -la /data
```

## Checklist de déploiement

Avant de déployer une base de données en production :

- [ ] Secret créé avec des mots de passe forts
- [ ] Resource limits définis
- [ ] Probes configurées
- [ ] StorageClass appropriée sélectionnée
- [ ] Taille de PVC suffisante (avec marge de croissance)
- [ ] Namespace dédié créé
- [ ] Labels et annotations ajoutés
- [ ] Sauvegarde automatique configurée
- [ ] Monitoring activé
- [ ] Documentation à jour
- [ ] Plan de restauration testé

## Conclusion

Ces exemples pratiques vous donnent une base solide pour déployer des applications stateful dans Kubernetes avec MicroK8s. Les principes appliqués ici (StatefulSets, PVCs, Secrets, ConfigMaps, Services) sont transposables à toute application nécessitant du stockage persistant.

**Points essentiels à retenir** :

- Toujours utiliser des StatefulSets pour les bases de données
- Séparer les préoccupations : données (PVC), configuration (ConfigMap), secrets (Secret)
- Définir des probes pour la haute disponibilité
- Prévoir des sauvegardes automatiques
- Tester les restaurations régulièrement
- Surveiller l'utilisation du stockage
- Documenter les déploiements

Avec ces exemples et bonnes pratiques, vous êtes maintenant équipé pour gérer des applications stateful professionnelles dans Kubernetes !

---

**Points clés à retenir** :

✅ **StatefulSets** = Obligatoires pour les bases de données
✅ **Secrets** = Pour tous les mots de passe et données sensibles
✅ **ConfigMaps** = Pour la configuration applicative
✅ **Probes** = Liveness et readiness pour la haute disponibilité
✅ **Sauvegardes** = Automatisées et testées régulièrement
✅ **Monitoring** = Surveillance de l'utilisation et des performances
✅ **Namespaces** = Isolation des environnements
✅ **Documentation** = Labels, annotations et READMEs à jour

⏭️ [Réseau Kubernetes Interne](/07-reseau-kubernetes-interne/README.md)
