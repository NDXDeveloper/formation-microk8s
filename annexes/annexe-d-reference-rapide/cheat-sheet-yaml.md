🔝 Retour au [Sommaire](/SOMMAIRE.md)

# Cheat Sheet YAML pour Kubernetes

## Introduction à YAML

### Qu'est-ce que YAML ?

**YAML** (YAML Ain't Markup Language) est un format de sérialisation de données lisible par l'homme, utilisé pour écrire les fichiers de configuration Kubernetes.

**Pourquoi YAML dans Kubernetes ?**
- **Lisible** : Facile à comprendre pour les humains
- **Déclaratif** : Vous décrivez l'état désiré, pas les étapes
- **Versionnable** : Parfait pour Git et le contrôle de version
- **Portable** : Fonctionne sur tous les clusters Kubernetes

### Règles de Base YAML

#### 1. Indentation

**IMPORTANT** : YAML utilise des **espaces** (jamais de tabulations) pour l'indentation.

```yaml
# ✓ Correct - 2 espaces d'indentation
apiVersion: v1
kind: Pod
metadata:
  name: nginx
  labels:
    app: web

# ✗ Incorrect - tabulations (invisible ici mais causent des erreurs)
apiVersion: v1
kind: Pod
metadata:
	name: nginx  # ← Tabulation ici = ERREUR
```

**Convention Kubernetes** : Utiliser **2 espaces** par niveau d'indentation.

#### 2. Structures de Données

**Scalaires (valeurs simples) :**
```yaml
name: nginx
replicas: 3
enabled: true
version: 1.21
```

**Listes (arrays) :**
```yaml
# Style avec tirets
ports:
- 80
- 443
- 8080

# Ou style inline
ports: [80, 443, 8080]
```

**Dictionnaires (maps/objets) :**
```yaml
# Style multiligne
metadata:
  name: nginx
  namespace: production
  labels:
    app: web

# Ou style inline
metadata: {name: nginx, namespace: production}
```

#### 3. Types de Valeurs

```yaml
# Chaînes de caractères (quotes optionnelles)
name: nginx
message: "Hello World"
path: '/var/log'

# Nombres
replicas: 3
port: 8080
cpu: 0.5

# Booléens
enabled: true
debug: false
# Alternatives acceptées: yes/no, on/off

# Null
database: null
# Ou simplement omettre la clé

# Multilignes
description: |
  Ceci est une description
  sur plusieurs lignes
  avec retours à la ligne préservés

summary: >
  Ceci est un texte
  qui sera replié
  sur une seule ligne
```

#### 4. Commentaires

```yaml
# Ceci est un commentaire
apiVersion: v1  # Commentaire en fin de ligne
kind: Pod
# Les commentaires peuvent apparaître n'importe où
metadata:
  name: nginx  # Nom du pod
```

#### 5. Documents Multiples

```yaml
---
# Premier document
apiVersion: v1
kind: Pod
metadata:
  name: pod1
---
# Deuxième document
apiVersion: v1
kind: Service
metadata:
  name: service1
---
# Troisième document
apiVersion: apps/v1
kind: Deployment
metadata:
  name: deployment1
```

**Note** : `---` sépare plusieurs ressources dans un même fichier.

---

## Structure d'un Manifeste Kubernetes

Tous les manifestes Kubernetes suivent cette structure de base :

```yaml
apiVersion: [group/]version    # Version de l'API
kind: ResourceType             # Type de ressource
metadata:                      # Métadonnées
  name: resource-name          # Nom (obligatoire)
  namespace: namespace-name    # Namespace (optionnel)
  labels:                      # Labels (optionnel)
    key: value
  annotations:                 # Annotations (optionnel)
    key: value
spec:                          # Spécification (varie selon le kind)
  # ... configuration spécifique à la ressource
```

### Les 4 Champs Obligatoires

| Champ | Description | Exemple |
|-------|-------------|---------|
| `apiVersion` | Version de l'API Kubernetes | `v1`, `apps/v1`, `networking.k8s.io/v1` |
| `kind` | Type de ressource | `Pod`, `Deployment`, `Service` |
| `metadata.name` | Nom unique de la ressource | `nginx`, `my-app` |
| `spec` | Spécification désirée | Varie selon le `kind` |

---

## Pods

### Pod Simple

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx
  labels:
    app: web
    environment: production
spec:
  containers:
  - name: nginx
    image: nginx:1.21
    ports:
    - containerPort: 80
```

### Pod avec Plusieurs Conteneurs

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: multi-container-pod
spec:
  containers:
  # Conteneur principal
  - name: app
    image: myapp:latest
    ports:
    - containerPort: 8080

  # Conteneur sidecar (logs)
  - name: log-collector
    image: fluent/fluentd:latest
    volumeMounts:
    - name: logs
      mountPath: /logs

  volumes:
  - name: logs
    emptyDir: {}
```

### Pod avec Variables d'Environnement

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: env-pod
spec:
  containers:
  - name: app
    image: myapp:latest
    env:
    # Variable simple
    - name: DATABASE_URL
      value: "postgres://db:5432/mydb"

    # Depuis un ConfigMap
    - name: API_KEY
      valueFrom:
        configMapKeyRef:
          name: app-config
          key: api_key

    # Depuis un Secret
    - name: DB_PASSWORD
      valueFrom:
        secretKeyRef:
          name: db-secret
          key: password

    # Toutes les clés d'un ConfigMap
    envFrom:
    - configMapRef:
        name: app-config

    # Toutes les clés d'un Secret
    - secretRef:
        name: app-secrets
```

### Pod avec Volumes

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: volume-pod
spec:
  containers:
  - name: app
    image: myapp:latest
    volumeMounts:
    # Volume depuis ConfigMap
    - name: config
      mountPath: /etc/config
      readOnly: true

    # Volume depuis Secret
    - name: secrets
      mountPath: /etc/secrets
      readOnly: true

    # Volume persistant
    - name: data
      mountPath: /data

    # Volume temporaire
    - name: cache
      mountPath: /cache

  volumes:
  # ConfigMap
  - name: config
    configMap:
      name: app-config

  # Secret
  - name: secrets
    secret:
      secretName: app-secrets

  # PersistentVolumeClaim
  - name: data
    persistentVolumeClaim:
      claimName: data-pvc

  # EmptyDir (temporaire)
  - name: cache
    emptyDir: {}
```

### Pod avec Ressources

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: resource-pod
spec:
  containers:
  - name: app
    image: myapp:latest
    resources:
      # Requêtes (garanties)
      requests:
        memory: "256Mi"
        cpu: "250m"

      # Limites (maximum)
      limits:
        memory: "512Mi"
        cpu: "500m"
```

**Unités :**
- **CPU** : `1` = 1 cœur, `0.5` ou `500m` = 0.5 cœur
- **Mémoire** : `128Mi`, `1Gi`, `512Mi`

### Pod avec Probes (Santé)

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: probe-pod
spec:
  containers:
  - name: app
    image: myapp:latest
    ports:
    - containerPort: 8080

    # Liveness Probe (redémarre si échoue)
    livenessProbe:
      httpGet:
        path: /health
        port: 8080
      initialDelaySeconds: 30
      periodSeconds: 10
      timeoutSeconds: 5
      failureThreshold: 3

    # Readiness Probe (retire du service si échoue)
    readinessProbe:
      httpGet:
        path: /ready
        port: 8080
      initialDelaySeconds: 5
      periodSeconds: 5
      timeoutSeconds: 3
      successThreshold: 1
      failureThreshold: 3

    # Startup Probe (pour démarrage lent)
    startupProbe:
      httpGet:
        path: /startup
        port: 8080
      initialDelaySeconds: 0
      periodSeconds: 10
      timeoutSeconds: 3
      failureThreshold: 30
```

**Types de probes :**
```yaml
# HTTP GET
httpGet:
  path: /health
  port: 8080
  httpHeaders:
  - name: Custom-Header
    value: value

# TCP Socket
tcpSocket:
  port: 8080

# Exec Command
exec:
  command:
  - cat
  - /tmp/healthy
```

---

## Deployments

### Deployment de Base

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  labels:
    app: nginx
spec:
  # Nombre de replicas
  replicas: 3

  # Sélecteur de pods
  selector:
    matchLabels:
      app: nginx

  # Template de pods
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.21
        ports:
        - containerPort: 80
```

### Deployment Complet

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app-deployment
  namespace: production
  labels:
    app: myapp
    version: v2.0
  annotations:
    description: "Application principale"
spec:
  # Nombre de replicas
  replicas: 5

  # Sélecteur (doit correspondre aux labels du template)
  selector:
    matchLabels:
      app: myapp
      tier: backend

  # Stratégie de mise à jour
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1        # Pods supplémentaires pendant update
      maxUnavailable: 0  # Pods indisponibles pendant update

  # Historique des versions
  revisionHistoryLimit: 10

  # Template de pods
  template:
    metadata:
      labels:
        app: myapp
        tier: backend
        version: v2.0
      annotations:
        prometheus.io/scrape: "true"
        prometheus.io/port: "9090"
    spec:
      # Service Account
      serviceAccountName: app-sa

      # Conteneurs
      containers:
      - name: app
        image: myapp:v2.0
        imagePullPolicy: IfNotPresent

        ports:
        - name: http
          containerPort: 8080
          protocol: TCP
        - name: metrics
          containerPort: 9090

        # Variables d'environnement
        env:
        - name: ENV
          value: "production"
        envFrom:
        - configMapRef:
            name: app-config
        - secretRef:
            name: app-secrets

        # Ressources
        resources:
          requests:
            memory: "256Mi"
            cpu: "250m"
          limits:
            memory: "512Mi"
            cpu: "500m"

        # Probes
        livenessProbe:
          httpGet:
            path: /health
            port: 8080
          initialDelaySeconds: 30
          periodSeconds: 10

        readinessProbe:
          httpGet:
            path: /ready
            port: 8080
          initialDelaySeconds: 5
          periodSeconds: 5

        # Volumes
        volumeMounts:
        - name: config
          mountPath: /etc/config
        - name: data
          mountPath: /data

      # Volumes
      volumes:
      - name: config
        configMap:
          name: app-config
      - name: data
        persistentVolumeClaim:
          claimName: app-data-pvc

      # Security Context
      securityContext:
        runAsNonRoot: true
        runAsUser: 1000
        fsGroup: 2000
```

### Stratégies de Déploiement

```yaml
# RollingUpdate (par défaut)
strategy:
  type: RollingUpdate
  rollingUpdate:
    maxSurge: 1        # 25% par défaut
    maxUnavailable: 0  # 25% par défaut

---
# Recreate (arrête tous puis redémarre)
strategy:
  type: Recreate
```

---

## Services

### Service ClusterIP (Interne)

```yaml
apiVersion: v1
kind: Service
metadata:
  name: backend-service
  labels:
    app: backend
spec:
  type: ClusterIP  # Par défaut
  selector:
    app: backend
    tier: api
  ports:
  - name: http
    protocol: TCP
    port: 80        # Port du service
    targetPort: 8080  # Port du conteneur
```

### Service NodePort (Exposition externe)

```yaml
apiVersion: v1
kind: Service
metadata:
  name: frontend-service
spec:
  type: NodePort
  selector:
    app: frontend
  ports:
  - name: http
    protocol: TCP
    port: 80
    targetPort: 8080
    nodePort: 30080  # Port sur les nœuds (30000-32767)
```

### Service LoadBalancer

```yaml
apiVersion: v1
kind: Service
metadata:
  name: app-loadbalancer
spec:
  type: LoadBalancer
  selector:
    app: myapp
  ports:
  - name: http
    protocol: TCP
    port: 80
    targetPort: 8080
  - name: https
    protocol: TCP
    port: 443
    targetPort: 8443

  # IP externe spécifique (optionnel)
  loadBalancerIP: 192.168.1.100
```

### Service avec Plusieurs Ports

```yaml
apiVersion: v1
kind: Service
metadata:
  name: multi-port-service
spec:
  selector:
    app: myapp
  ports:
  - name: http
    protocol: TCP
    port: 80
    targetPort: 8080
  - name: https
    protocol: TCP
    port: 443
    targetPort: 8443
  - name: metrics
    protocol: TCP
    port: 9090
    targetPort: 9090
```

### Service Headless (Sans ClusterIP)

```yaml
apiVersion: v1
kind: Service
metadata:
  name: database-headless
spec:
  clusterIP: None  # Service headless
  selector:
    app: database
  ports:
  - port: 5432
    targetPort: 5432
```

---

## ConfigMaps

### ConfigMap avec Données Littérales

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
  namespace: production
data:
  # Clés-valeurs simples
  database_url: "postgres://db:5432/mydb"
  log_level: "info"
  api_endpoint: "https://api.example.com"
  max_connections: "100"

  # Configuration multilignes
  app.conf: |
    server {
      listen 80;
      server_name example.com;

      location / {
        proxy_pass http://backend:8080;
      }
    }

  config.json: |
    {
      "api": {
        "url": "https://api.example.com",
        "timeout": 30
      },
      "features": {
        "cache": true,
        "debug": false
      }
    }
```

### ConfigMap depuis Fichiers

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
data:
  # Contenu de config.properties
  config.properties: |
    app.name=MyApp
    app.version=1.0
    database.host=localhost
    database.port=5432

  # Contenu de logging.conf
  logging.conf: |
    [loggers]
    keys=root

    [handlers]
    keys=console

    [logger_root]
    level=INFO
    handlers=console
```

---

## Secrets

### Secret Opaque (Générique)

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: db-secret
type: Opaque
data:
  # Valeurs encodées en base64
  username: YWRtaW4=        # "admin" en base64
  password: cGFzc3dvcmQxMjM=  # "password123" en base64
```

**Encoder/Décoder base64 :**
```bash
# Encoder
echo -n "admin" | base64
# Résultat : YWRtaW4=

# Décoder
echo "YWRtaW4=" | base64 -d
# Résultat : admin
```

### Secret avec stringData (Non encodé)

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: db-secret
type: Opaque
stringData:
  # Valeurs en clair (Kubernetes les encode automatiquement)
  username: admin
  password: password123
  connection_string: "postgres://admin:password123@db:5432/mydb"
```

### Secret TLS

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: tls-secret
type: kubernetes.io/tls
data:
  tls.crt: LS0tLS1CRUdJTi...  # Certificat encodé en base64
  tls.key: LS0tLS1CRUdJTi...  # Clé privée encodée en base64
```

### Secret Docker Registry

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: registry-secret
type: kubernetes.io/dockerconfigjson
data:
  .dockerconfigjson: eyJhdXRocyI6eyJyZWdpc3RyeS5leGFtcGxlLmNvbSI6...
```

---

## PersistentVolumes et PersistentVolumeClaims

### PersistentVolume (PV)

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv-data
  labels:
    type: local
spec:
  # Capacité
  capacity:
    storage: 10Gi

  # Mode d'accès
  accessModes:
  - ReadWriteOnce  # RWO : un seul nœud en lecture/écriture
  # - ReadOnlyMany   # ROX : plusieurs nœuds en lecture seule
  # - ReadWriteMany  # RWX : plusieurs nœuds en lecture/écriture

  # Politique de récupération
  persistentVolumeReclaimPolicy: Retain  # Retain, Recycle, Delete

  # Classe de stockage
  storageClassName: fast

  # Type de volume (exemple : hostPath)
  hostPath:
    path: /mnt/data
    type: DirectoryOrCreate
```

### PersistentVolumeClaim (PVC)

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: app-data-pvc
spec:
  # Mode d'accès
  accessModes:
  - ReadWriteOnce

  # Ressources demandées
  resources:
    requests:
      storage: 5Gi

  # Classe de stockage (doit correspondre au PV)
  storageClassName: fast

  # Sélecteur (optionnel)
  selector:
    matchLabels:
      type: local
```

### Pod utilisant un PVC

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-with-pvc
spec:
  containers:
  - name: app
    image: myapp:latest
    volumeMounts:
    - name: data
      mountPath: /data

  volumes:
  - name: data
    persistentVolumeClaim:
      claimName: app-data-pvc
```

---

## Ingress

### Ingress Simple

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: simple-ingress
spec:
  ingressClassName: nginx  # ou public
  rules:
  - host: example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: frontend-service
            port:
              number: 80
```

### Ingress avec Plusieurs Hôtes

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: multi-host-ingress
spec:
  ingressClassName: nginx
  rules:
  # Site web
  - host: www.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: frontend-service
            port:
              number: 80

  # API
  - host: api.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: api-service
            port:
              number: 8080

  # Dashboard
  - host: dashboard.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: dashboard-service
            port:
              number: 3000
```

### Ingress avec Routage par Chemin

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: path-based-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /$2
spec:
  ingressClassName: nginx
  rules:
  - host: example.com
    http:
      paths:
      # Frontend
      - path: /
        pathType: Prefix
        backend:
          service:
            name: frontend-service
            port:
              number: 80

      # API
      - path: /api(/|$)(.*)
        pathType: ImplementationSpecific
        backend:
          service:
            name: api-service
            port:
              number: 8080

      # Admin
      - path: /admin
        pathType: Prefix
        backend:
          service:
            name: admin-service
            port:
              number: 3000
```

### Ingress avec TLS

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: tls-ingress
  annotations:
    cert-manager.io/cluster-issuer: "letsencrypt-prod"
spec:
  ingressClassName: nginx
  tls:
  - hosts:
    - example.com
    - www.example.com
    secretName: example-tls  # Secret contenant le certificat
  rules:
  - host: example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: frontend-service
            port:
              number: 80
```

### Ingress avec Annotations Courantes

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: annotated-ingress
  annotations:
    # Redirection HTTP vers HTTPS
    nginx.ingress.kubernetes.io/ssl-redirect: "true"

    # Rewrite URL
    nginx.ingress.kubernetes.io/rewrite-target: /

    # Timeout
    nginx.ingress.kubernetes.io/proxy-connect-timeout: "60"
    nginx.ingress.kubernetes.io/proxy-send-timeout: "60"
    nginx.ingress.kubernetes.io/proxy-read-timeout: "60"

    # Taille max upload
    nginx.ingress.kubernetes.io/proxy-body-size: "50m"

    # Rate limiting
    nginx.ingress.kubernetes.io/limit-rps: "10"

    # Whitelist IP
    nginx.ingress.kubernetes.io/whitelist-source-range: "10.0.0.0/8,192.168.0.0/16"

    # CORS
    nginx.ingress.kubernetes.io/enable-cors: "true"
    nginx.ingress.kubernetes.io/cors-allow-origin: "*"

    # Basic Auth
    nginx.ingress.kubernetes.io/auth-type: basic
    nginx.ingress.kubernetes.io/auth-secret: basic-auth
    nginx.ingress.kubernetes.io/auth-realm: "Authentication Required"
spec:
  ingressClassName: nginx
  rules:
  - host: example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: app-service
            port:
              number: 80
```

---

## Namespaces

### Namespace Simple

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: production
```

### Namespace avec Labels et Quotas

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: production
  labels:
    environment: production
    team: platform
  annotations:
    description: "Production environment for all apps"
```

---

## Jobs et CronJobs

### Job

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: data-migration
spec:
  # Nombre d'exécutions réussies requises
  completions: 1

  # Nombre d'exécutions parallèles
  parallelism: 1

  # Nombre de tentatives avant échec
  backoffLimit: 3

  # Durée de vie après succès (secondes)
  ttlSecondsAfterFinished: 100

  template:
    metadata:
      labels:
        job: data-migration
    spec:
      restartPolicy: OnFailure  # ou Never
      containers:
      - name: migration
        image: migration-tool:latest
        command: ["python", "migrate.py"]
        env:
        - name: DATABASE_URL
          valueFrom:
            secretKeyRef:
              name: db-secret
              key: url
```

### CronJob

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: backup-job
spec:
  # Planification Cron
  schedule: "0 2 * * *"  # Tous les jours à 2h du matin

  # Fuseau horaire (optionnel)
  timeZone: "Europe/Paris"

  # Politique de concurrence
  concurrencyPolicy: Forbid  # Forbid, Allow, Replace

  # Nombre de jobs réussis/échoués à garder
  successfulJobsHistoryLimit: 3
  failedJobsHistoryLimit: 1

  # Deadline de démarrage (secondes)
  startingDeadlineSeconds: 300

  # Template de Job
  jobTemplate:
    spec:
      template:
        spec:
          restartPolicy: OnFailure
          containers:
          - name: backup
            image: backup-tool:latest
            command: ["sh", "-c", "backup.sh"]
            env:
            - name: BACKUP_DIR
              value: "/backups"
            volumeMounts:
            - name: backup-storage
              mountPath: /backups
          volumes:
          - name: backup-storage
            persistentVolumeClaim:
              claimName: backup-pvc
```

**Syntaxe Cron :**
```
# ┌───────────── minute (0 - 59)
# │ ┌───────────── heure (0 - 23)
# │ │ ┌───────────── jour du mois (1 - 31)
# │ │ │ ┌───────────── mois (1 - 12)
# │ │ │ │ ┌───────────── jour de la semaine (0 - 6) (0 = Dimanche)
# │ │ │ │ │
# * * * * *

# Exemples
"0 0 * * *"      # Tous les jours à minuit
"0 2 * * *"      # Tous les jours à 2h
"*/15 * * * *"   # Toutes les 15 minutes
"0 */2 * * *"    # Toutes les 2 heures
"0 0 * * 0"      # Tous les dimanches à minuit
"0 0 1 * *"      # Le premier de chaque mois à minuit
"0 9 * * 1-5"    # En semaine à 9h
```

---

## StatefulSets

### StatefulSet de Base

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: database
spec:
  # Nom du service headless
  serviceName: "database"

  replicas: 3

  selector:
    matchLabels:
      app: database

  template:
    metadata:
      labels:
        app: database
    spec:
      containers:
      - name: postgres
        image: postgres:14
        ports:
        - containerPort: 5432
          name: postgres
        env:
        - name: POSTGRES_PASSWORD
          valueFrom:
            secretKeyRef:
              name: db-secret
              key: password
        volumeMounts:
        - name: data
          mountPath: /var/lib/postgresql/data

  # Template de PVC (créé automatiquement pour chaque replica)
  volumeClaimTemplates:
  - metadata:
      name: data
    spec:
      accessModes: ["ReadWriteOnce"]
      storageClassName: "fast"
      resources:
        requests:
          storage: 10Gi
```

### Service Headless pour StatefulSet

```yaml
apiVersion: v1
kind: Service
metadata:
  name: database
spec:
  clusterIP: None  # Service headless
  selector:
    app: database
  ports:
  - port: 5432
    name: postgres
```

---

## DaemonSets

### DaemonSet

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: log-collector
  labels:
    app: logging
spec:
  selector:
    matchLabels:
      app: log-collector

  template:
    metadata:
      labels:
        app: log-collector
    spec:
      # Tolérations pour s'exécuter sur tous les nœuds
      tolerations:
      - key: node-role.kubernetes.io/control-plane
        effect: NoSchedule

      containers:
      - name: fluentd
        image: fluent/fluentd:latest
        resources:
          limits:
            memory: 200Mi
          requests:
            cpu: 100m
            memory: 200Mi
        volumeMounts:
        - name: varlog
          mountPath: /var/log
        - name: varlibdockercontainers
          mountPath: /var/lib/docker/containers
          readOnly: true

      volumes:
      - name: varlog
        hostPath:
          path: /var/log
      - name: varlibdockercontainers
        hostPath:
          path: /var/lib/docker/containers
```

---

## Network Policies

### Network Policy Deny All

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: deny-all
  namespace: production
spec:
  podSelector: {}  # S'applique à tous les pods
  policyTypes:
  - Ingress
  - Egress
```

### Network Policy Allow Specific

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-frontend-to-backend
  namespace: production
spec:
  podSelector:
    matchLabels:
      app: backend

  policyTypes:
  - Ingress

  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: frontend
    ports:
    - protocol: TCP
      port: 8080
```

### Network Policy avec Egress

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: backend-policy
  namespace: production
spec:
  podSelector:
    matchLabels:
      app: backend

  policyTypes:
  - Ingress
  - Egress

  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: frontend
    ports:
    - protocol: TCP
      port: 8080

  egress:
  # Autoriser DNS
  - to:
    - namespaceSelector:
        matchLabels:
          name: kube-system
      podSelector:
        matchLabels:
          k8s-app: kube-dns
    ports:
    - protocol: UDP
      port: 53

  # Autoriser la base de données
  - to:
    - podSelector:
        matchLabels:
          app: database
    ports:
    - protocol: TCP
      port: 5432
```

---

## ServiceAccounts, Roles et RoleBindings

### ServiceAccount

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: app-service-account
  namespace: production
```

### Role (Namespace)

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: pod-reader
  namespace: production
rules:
- apiGroups: [""]  # "" pour core API group
  resources: ["pods"]
  verbs: ["get", "list", "watch"]
- apiGroups: [""]
  resources: ["pods/log"]
  verbs: ["get"]
```

### ClusterRole (Global)

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: node-reader
rules:
- apiGroups: [""]
  resources: ["nodes"]
  verbs: ["get", "list", "watch"]
```

### RoleBinding

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: read-pods
  namespace: production
subjects:
- kind: ServiceAccount
  name: app-service-account
  namespace: production
roleRef:
  kind: Role
  name: pod-reader
  apiGroup: rbac.authorization.k8s.io
```

### ClusterRoleBinding

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: read-nodes
subjects:
- kind: ServiceAccount
  name: app-service-account
  namespace: production
roleRef:
  kind: ClusterRole
  name: node-reader
  apiGroup: rbac.authorization.k8s.io
```

---

## HorizontalPodAutoscaler

### HPA basé sur CPU

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: app-hpa
  namespace: production
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: app-deployment

  minReplicas: 2
  maxReplicas: 10

  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
```

### HPA basé sur CPU et Mémoire

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: app-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: app-deployment

  minReplicas: 2
  maxReplicas: 10

  metrics:
  # CPU
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70

  # Mémoire
  - type: Resource
    resource:
      name: memory
      target:
        type: Utilization
        averageUtilization: 80

  behavior:
    scaleDown:
      stabilizationWindowSeconds: 300
      policies:
      - type: Percent
        value: 50
        periodSeconds: 60
    scaleUp:
      stabilizationWindowSeconds: 0
      policies:
      - type: Percent
        value: 100
        periodSeconds: 30
      - type: Pods
        value: 2
        periodSeconds: 30
      selectPolicy: Max
```

---

## ResourceQuotas et LimitRanges

### ResourceQuota

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: compute-quota
  namespace: production
spec:
  hard:
    # Limites pour le namespace
    requests.cpu: "10"
    requests.memory: 20Gi
    limits.cpu: "20"
    limits.memory: 40Gi

    # Nombre de ressources
    pods: "100"
    services: "50"
    persistentvolumeclaims: "20"

    # Stockage
    requests.storage: 100Gi
```

### LimitRange

```yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: resource-limits
  namespace: production
spec:
  limits:
  # Limites pour les conteneurs
  - type: Container
    default:        # Limites par défaut
      cpu: 500m
      memory: 512Mi
    defaultRequest: # Requêtes par défaut
      cpu: 250m
      memory: 256Mi
    max:            # Maximum autorisé
      cpu: 2
      memory: 2Gi
    min:            # Minimum autorisé
      cpu: 100m
      memory: 128Mi

  # Limites pour les pods
  - type: Pod
    max:
      cpu: 4
      memory: 4Gi

  # Limites pour les PVC
  - type: PersistentVolumeClaim
    max:
      storage: 50Gi
    min:
      storage: 1Gi
```

---

## Templates Réutilisables

### Template Application Complète

```yaml
---
# Namespace
apiVersion: v1
kind: Namespace
metadata:
  name: myapp

---
# ConfigMap
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
  namespace: myapp
data:
  API_URL: "https://api.example.com"
  LOG_LEVEL: "info"

---
# Secret
apiVersion: v1
kind: Secret
metadata:
  name: app-secrets
  namespace: myapp
type: Opaque
stringData:
  API_KEY: "secret-key-123"
  DB_PASSWORD: "password123"

---
# Deployment
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app
  namespace: myapp
spec:
  replicas: 3
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
        image: myapp:latest
        ports:
        - containerPort: 8080
        envFrom:
        - configMapRef:
            name: app-config
        - secretRef:
            name: app-secrets
        resources:
          requests:
            memory: "256Mi"
            cpu: "250m"
          limits:
            memory: "512Mi"
            cpu: "500m"
        livenessProbe:
          httpGet:
            path: /health
            port: 8080
          initialDelaySeconds: 30
          periodSeconds: 10
        readinessProbe:
          httpGet:
            path: /ready
            port: 8080
          initialDelaySeconds: 5
          periodSeconds: 5

---
# Service
apiVersion: v1
kind: Service
metadata:
  name: app-service
  namespace: myapp
spec:
  selector:
    app: myapp
  ports:
  - port: 80
    targetPort: 8080

---
# Ingress
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: app-ingress
  namespace: myapp
  annotations:
    cert-manager.io/cluster-issuer: "letsencrypt-prod"
spec:
  ingressClassName: nginx
  tls:
  - hosts:
    - app.example.com
    secretName: app-tls
  rules:
  - host: app.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: app-service
            port:
              number: 80
```

---

## Bonnes Pratiques YAML

### 1. Utiliser des Labels Cohérents

```yaml
metadata:
  labels:
    app: myapp              # Nom de l'application
    component: backend      # Composant (frontend, backend, database)
    tier: api               # Tier (presentation, logic, data)
    environment: production # Environnement
    version: v2.1          # Version
    managed-by: helm        # Gestionnaire
```

### 2. Toujours Définir des Ressources

```yaml
resources:
  requests:  # Ce dont l'app a besoin
    memory: "256Mi"
    cpu: "250m"
  limits:    # Maximum autorisé
    memory: "512Mi"
    cpu: "500m"
```

### 3. Utiliser des Health Checks

```yaml
livenessProbe:   # Redémarre si échoue
  httpGet:
    path: /health
    port: 8080
  initialDelaySeconds: 30

readinessProbe:  # Retire du service si échoue
  httpGet:
    path: /ready
    port: 8080
  initialDelaySeconds: 5
```

### 4. Ne Jamais Stocker de Secrets en Clair

```yaml
# ✗ Mauvais
env:
- name: PASSWORD
  value: "password123"

# ✓ Bon
env:
- name: PASSWORD
  valueFrom:
    secretKeyRef:
      name: db-secret
      key: password
```

### 5. Utiliser des Namespaces

```yaml
# Toujours spécifier le namespace
metadata:
  name: app
  namespace: production
```

### 6. Documenter avec Annotations

```yaml
metadata:
  annotations:
    description: "Application principale de production"
    maintainer: "team@example.com"
    documentation: "https://docs.example.com/app"
```

### 7. Versionner les Images

```yaml
# ✗ Éviter
image: nginx:latest

# ✓ Préférer
image: nginx:1.21.3
```

---

## Erreurs Courantes à Éviter

### 1. Indentation Incorrecte

```yaml
# ✗ Mauvais
apiVersion: v1
kind: Pod
metadata:
name: nginx  # ← Indentation incorrecte
  labels:
    app: web

# ✓ Correct
apiVersion: v1
kind: Pod
metadata:
  name: nginx
  labels:
    app: web
```

### 2. Sélecteurs qui ne Correspondent Pas

```yaml
# ✗ Mauvais - Les labels ne correspondent pas
apiVersion: apps/v1
kind: Deployment
spec:
  selector:
    matchLabels:
      app: nginx  # ← Label dans le selector
  template:
    metadata:
      labels:
        app: web  # ← Label différent dans le template

# ✓ Correct
apiVersion: apps/v1
kind: Deployment
spec:
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx  # ← Doit correspondre
```

### 3. Oublier le Type de Path dans Ingress

```yaml
# ✗ Mauvais
paths:
- path: /
  backend:
    service:
      name: app-service

# ✓ Correct
paths:
- path: /
  pathType: Prefix  # ← Obligatoire
  backend:
    service:
      name: app-service
      port:
        number: 80
```

### 4. Mélanger Tabulations et Espaces

```yaml
# ✗ Mauvais - Mélange invisible de tabs et espaces
metadata:
  name: nginx
	labels:  # ← Tabulation ici
    app: web

# ✓ Correct - Uniquement des espaces
metadata:
  name: nginx
  labels:
    app: web
```

### 5. Oublier les Requests/Limits

```yaml
# ✗ Mauvais - Pas de ressources définies
containers:
- name: app
  image: myapp:latest

# ✓ Correct
containers:
- name: app
  image: myapp:latest
  resources:
    requests:
      memory: "256Mi"
      cpu: "250m"
    limits:
      memory: "512Mi"
      cpu: "500m"
```

---

## Outils de Validation

### kubectl dry-run

```bash
# Vérifier un manifeste sans l'appliquer
kubectl apply -f deployment.yaml --dry-run=client

# Vérifier sur le serveur
kubectl apply -f deployment.yaml --dry-run=server
```

### kubectl diff

```bash
# Voir les différences avant application
kubectl diff -f deployment.yaml
```

### yamllint

```bash
# Installer yamllint
pip install yamllint

# Valider un fichier
yamllint deployment.yaml

# Avec configuration personnalisée
yamllint -d "{extends: default, rules: {line-length: {max: 120}}}" deployment.yaml
```

### kubeval

```bash
# Installer kubeval
wget https://github.com/instrumenta/kubeval/releases/latest/download/kubeval-linux-amd64.tar.gz
tar xf kubeval-linux-amd64.tar.gz
sudo cp kubeval /usr/local/bin

# Valider un fichier
kubeval deployment.yaml

# Valider tous les fichiers d'un dossier
kubeval manifests/*.yaml
```

---

## Raccourcis et Astuces

### Générer des Manifestes YAML

```bash
# Pod
kubectl run nginx --image=nginx --dry-run=client -o yaml > pod.yaml

# Deployment
kubectl create deployment nginx --image=nginx --dry-run=client -o yaml > deployment.yaml

# Service
kubectl expose deployment nginx --port=80 --dry-run=client -o yaml > service.yaml

# ConfigMap
kubectl create configmap app-config \
  --from-literal=key=value \
  --dry-run=client -o yaml > configmap.yaml

# Secret
kubectl create secret generic app-secret \
  --from-literal=password=secret \
  --dry-run=client -o yaml > secret.yaml

# Job
kubectl create job test --image=busybox --dry-run=client -o yaml > job.yaml

# CronJob
kubectl create cronjob backup --image=backup:latest \
  --schedule="0 2 * * *" \
  --dry-run=client -o yaml > cronjob.yaml
```

### Combiner Plusieurs Ressources

```yaml
# Utiliser --- pour séparer
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: config
data:
  key: value

---
apiVersion: v1
kind: Secret
metadata:
  name: secret
stringData:
  password: secret123

---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app
spec:
  replicas: 3
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
        image: myapp:latest
```

### Utiliser Kustomize

```bash
# Structure
base/
  deployment.yaml
  service.yaml
  kustomization.yaml
overlays/
  production/
    kustomization.yaml
  staging/
    kustomization.yaml

# Appliquer
kubectl apply -k overlays/production/
```

---

## Références API Versions

| Ressource | apiVersion |
|-----------|------------|
| Pod, Service, ConfigMap, Secret, PV, PVC | `v1` |
| Deployment, ReplicaSet, StatefulSet, DaemonSet | `apps/v1` |
| Job, CronJob | `batch/v1` |
| Ingress | `networking.k8s.io/v1` |
| NetworkPolicy | `networking.k8s.io/v1` |
| HPA | `autoscaling/v2` |
| Role, RoleBinding, ClusterRole, ClusterRoleBinding | `rbac.authorization.k8s.io/v1` |
| ServiceAccount | `v1` |
| PodDisruptionBudget | `policy/v1` |
| StorageClass | `storage.k8s.io/v1` |

---

## Ressources et Documentation

### Documentation Officielle

- **Kubernetes API Reference** : https://kubernetes.io/docs/reference/kubernetes-api/
- **YAML Syntax** : https://yaml.org/spec/1.2/spec.html
- **Kubernetes Objects** : https://kubernetes.io/docs/concepts/overview/working-with-objects/

### Outils en Ligne

- **YAML Validator** : http://www.yamllint.com/
- **Kubernetes YAML Generator** : https://k8syaml.com/
- **Base64 Encoder/Decoder** : https://www.base64decode.org/

### Éditeurs Recommandés

- **VSCode** avec extension "Kubernetes"
- **IntelliJ IDEA** avec plugin "Kubernetes"
- **Vim** avec plugin "vim-kubernetes"

---

## Conclusion

Ce cheat sheet couvre les structures YAML les plus courantes dans Kubernetes. Points clés à retenir :

✅ **Indentation** : 2 espaces, jamais de tabulations
✅ **Structure** : apiVersion, kind, metadata, spec
✅ **Labels** : Cohérents et descriptifs
✅ **Ressources** : Toujours définir requests et limits
✅ **Health Checks** : livenessProbe et readinessProbe
✅ **Secrets** : Jamais en clair dans les manifestes
✅ **Validation** : Utiliser dry-run et diff

**Pro Tip** : Utilisez `kubectl explain` pour obtenir la documentation d'un champ :
```bash
kubectl explain pod.spec.containers
kubectl explain deployment.spec.strategy
```

Avec ce cheat sheet, vous avez toutes les bases pour écrire des manifestes Kubernetes propres et fonctionnels ! 🚀

⏭️ [Troubleshooting rapide](/annexes/annexe-d-reference-rapide/troubleshooting-rapide.md)
