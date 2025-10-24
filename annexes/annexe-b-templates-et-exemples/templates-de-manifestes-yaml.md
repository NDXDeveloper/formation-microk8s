🔝 Retour au [Sommaire](/SOMMAIRE.md)

# Annexe B : Templates de Manifestes YAML

## Introduction aux Manifestes YAML

Dans Kubernetes et MicroK8s, toutes les ressources (Pods, Deployments, Services, etc.) sont définies à l'aide de fichiers YAML. Ces fichiers décrivent l'état souhaité de vos applications et de votre infrastructure.

### Structure de Base d'un Manifeste YAML

Tous les manifestes Kubernetes partagent une structure commune :

```yaml
apiVersion: # Version de l'API Kubernetes à utiliser
kind:       # Type de ressource (Pod, Deployment, Service, etc.)
metadata:   # Métadonnées (nom, labels, annotations)
  name:
  namespace:
  labels:
spec:       # Spécification détaillée de la ressource
```

### Règles Importantes pour YAML

- **L'indentation est cruciale** : utilisez toujours 2 espaces (jamais de tabulations)
- **Sensible à la casse** : `Name` est différent de `name`
- **Les listes utilisent des tirets** : `- item`
- **Les commentaires commencent par #**

---

## 1. Template Pod

Le Pod est l'unité de déploiement la plus basique dans Kubernetes.

```yaml
apiVersion: v1
kind: Pod
metadata:
  # Nom unique du Pod dans le namespace
  name: mon-pod
  # Namespace où déployer (optionnel, défaut: default)
  namespace: default
  # Labels pour identifier et sélectionner le Pod
  labels:
    app: mon-application
    environment: production
    version: "1.0"
  # Annotations pour métadonnées supplémentaires
  annotations:
    description: "Pod exemple pour documentation"
spec:
  # Liste des conteneurs dans le Pod
  containers:
  - name: mon-conteneur
    # Image Docker à utiliser
    image: nginx:1.21
    # Politique de téléchargement (Always, IfNotPresent, Never)
    imagePullPolicy: IfNotPresent
    # Ports exposés par le conteneur
    ports:
    - containerPort: 80
      name: http
      protocol: TCP
    # Variables d'environnement
    env:
    - name: MA_VARIABLE
      value: "ma_valeur"
    - name: AUTRE_VARIABLE
      value: "autre_valeur"
    # Ressources CPU et mémoire
    resources:
      # Ressources demandées (minimum)
      requests:
        memory: "64Mi"
        cpu: "250m"
      # Limites maximales
      limits:
        memory: "128Mi"
        cpu: "500m"
    # Sondes de santé
    livenessProbe:
      httpGet:
        path: /healthz
        port: 80
      initialDelaySeconds: 30
      periodSeconds: 10
    readinessProbe:
      httpGet:
        path: /ready
        port: 80
      initialDelaySeconds: 5
      periodSeconds: 5
  # Politique de redémarrage (Always, OnFailure, Never)
  restartPolicy: Always
```

---

## 2. Template Deployment

Le Deployment gère les ReplicaSets et permet des mises à jour progressives.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mon-deployment
  namespace: default
  labels:
    app: mon-application
spec:
  # Nombre de réplicas souhaités
  replicas: 3
  # Sélecteur pour identifier les Pods gérés
  selector:
    matchLabels:
      app: mon-application
  # Stratégie de mise à jour
  strategy:
    type: RollingUpdate
    rollingUpdate:
      # Maximum de Pods indisponibles pendant la mise à jour
      maxUnavailable: 1
      # Maximum de Pods supplémentaires pendant la mise à jour
      maxSurge: 1
  # Template du Pod (similaire au Pod ci-dessus)
  template:
    metadata:
      labels:
        app: mon-application
        version: "1.0"
    spec:
      containers:
      - name: mon-conteneur
        image: nginx:1.21
        ports:
        - containerPort: 80
          name: http
        env:
        - name: ENVIRONMENT
          value: "production"
        resources:
          requests:
            memory: "64Mi"
            cpu: "250m"
          limits:
            memory: "128Mi"
            cpu: "500m"
        livenessProbe:
          httpGet:
            path: /
            port: 80
          initialDelaySeconds: 30
          periodSeconds: 10
        readinessProbe:
          httpGet:
            path: /
            port: 80
          initialDelaySeconds: 5
          periodSeconds: 5
```

---

## 3. Template Service

Le Service expose les Pods et permet la communication réseau.

### 3.1 Service ClusterIP (Par défaut - interne uniquement)

```yaml
apiVersion: v1
kind: Service
metadata:
  name: mon-service-clusterip
  namespace: default
  labels:
    app: mon-application
spec:
  # Type de Service (ClusterIP par défaut)
  type: ClusterIP
  # Sélecteur pour cibler les Pods
  selector:
    app: mon-application
  # Ports exposés
  ports:
  - name: http
    # Port du Service
    port: 80
    # Port du conteneur cible
    targetPort: 80
    protocol: TCP
  # Affinité de session (None ou ClientIP)
  sessionAffinity: None
```

### 3.2 Service NodePort (Accessible depuis l'extérieur)

```yaml
apiVersion: v1
kind: Service
metadata:
  name: mon-service-nodeport
  namespace: default
spec:
  type: NodePort
  selector:
    app: mon-application
  ports:
  - name: http
    port: 80
    targetPort: 80
    # Port sur chaque nœud (30000-32767, optionnel)
    nodePort: 30080
    protocol: TCP
```

### 3.3 Service LoadBalancer (Avec MetalLB)

```yaml
apiVersion: v1
kind: Service
metadata:
  name: mon-service-loadbalancer
  namespace: default
spec:
  type: LoadBalancer
  selector:
    app: mon-application
  ports:
  - name: http
    port: 80
    targetPort: 80
    protocol: TCP
  # IP externe spécifique (optionnel)
  loadBalancerIP: 192.168.1.100
```

---

## 4. Template ConfigMap

Les ConfigMaps stockent des données de configuration non sensibles.

### 4.1 ConfigMap avec données littérales

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: mon-configmap
  namespace: default
data:
  # Clés et valeurs simples
  database_url: "postgresql://localhost:5432/mabase"
  max_connections: "100"
  log_level: "info"
  # Configuration multi-lignes
  application.properties: |
    server.port=8080
    server.context-path=/api
    spring.datasource.url=jdbc:postgresql://localhost:5432/mabase
    spring.datasource.username=user
```

### 4.2 Utilisation d'un ConfigMap dans un Pod

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-avec-configmap
spec:
  containers:
  - name: mon-conteneur
    image: nginx:1.21
    # Utiliser toutes les clés comme variables d'environnement
    envFrom:
    - configMapRef:
        name: mon-configmap
    # Ou utiliser des clés spécifiques
    env:
    - name: DATABASE_URL
      valueFrom:
        configMapKeyRef:
          name: mon-configmap
          key: database_url
    # Monter le ConfigMap comme volume
    volumeMounts:
    - name: config-volume
      mountPath: /etc/config
  volumes:
  - name: config-volume
    configMap:
      name: mon-configmap
```

---

## 5. Template Secret

Les Secrets stockent des données sensibles (mots de passe, tokens, clés).

### 5.1 Secret Opaque (type générique)

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: mon-secret
  namespace: default
type: Opaque
# Données encodées en base64
data:
  username: dXNlcm5hbWU=  # "username" encodé
  password: cGFzc3dvcmQ=  # "password" encodé
# OU données en clair (Kubernetes les encodera)
stringData:
  username: username
  password: password
  api-key: ma-cle-api-secrete
```

**Note** : Pour encoder en base64 manuellement :
```bash
echo -n "mon-texte" | base64
```

### 5.2 Secret pour Authentification Docker

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: registrypullsecret
  namespace: default
type: kubernetes.io/dockerconfigjson
data:
  .dockerconfigjson: <base64-encoded-docker-config>
```

### 5.3 Utilisation d'un Secret dans un Pod

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-avec-secret
spec:
  containers:
  - name: mon-conteneur
    image: nginx:1.21
    # Variables d'environnement depuis Secret
    env:
    - name: DB_USERNAME
      valueFrom:
        secretKeyRef:
          name: mon-secret
          key: username
    - name: DB_PASSWORD
      valueFrom:
        secretKeyRef:
          name: mon-secret
          key: password
    # Monter le Secret comme volume
    volumeMounts:
    - name: secret-volume
      mountPath: /etc/secrets
      readOnly: true
  volumes:
  - name: secret-volume
    secret:
      secretName: mon-secret
```

---

## 6. Template Namespace

Les Namespaces permettent de séparer les ressources logiquement.

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: mon-namespace
  labels:
    name: mon-namespace
    environment: production
  annotations:
    description: "Namespace pour l'environnement de production"
```

---

## 7. Template PersistentVolumeClaim (PVC)

Les PVC demandent du stockage persistant.

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: mon-pvc
  namespace: default
  labels:
    app: mon-application
spec:
  # Modes d'accès possibles:
  # - ReadWriteOnce (RWO): lecture/écriture par un seul nœud
  # - ReadOnlyMany (ROX): lecture par plusieurs nœuds
  # - ReadWriteMany (RWX): lecture/écriture par plusieurs nœuds
  accessModes:
    - ReadWriteOnce
  # Ressources demandées
  resources:
    requests:
      storage: 10Gi
  # StorageClass à utiliser (optionnel)
  storageClassName: microk8s-hostpath
  # Sélecteur de volume (optionnel)
  selector:
    matchLabels:
      type: fast-storage
```

### 7.1 Utilisation d'un PVC dans un Pod

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-avec-pvc
spec:
  containers:
  - name: mon-conteneur
    image: nginx:1.21
    volumeMounts:
    - name: mon-stockage
      mountPath: /usr/share/nginx/html
  volumes:
  - name: mon-stockage
    persistentVolumeClaim:
      claimName: mon-pvc
```

---

## 8. Template StatefulSet

Les StatefulSets gèrent des applications avec état (bases de données, etc.).

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: mon-statefulset
  namespace: default
spec:
  # Nom du Service headless associé
  serviceName: mon-service-headless
  # Nombre de réplicas
  replicas: 3
  selector:
    matchLabels:
      app: mon-application
  # Template des Pods
  template:
    metadata:
      labels:
        app: mon-application
    spec:
      containers:
      - name: mon-conteneur
        image: mysql:8.0
        ports:
        - containerPort: 3306
          name: mysql
        env:
        - name: MYSQL_ROOT_PASSWORD
          valueFrom:
            secretKeyRef:
              name: mysql-secret
              key: root-password
        volumeMounts:
        - name: data
          mountPath: /var/lib/mysql
  # Template des volumes persistants
  volumeClaimTemplates:
  - metadata:
      name: data
    spec:
      accessModes: [ "ReadWriteOnce" ]
      storageClassName: microk8s-hostpath
      resources:
        requests:
          storage: 10Gi
```

### 8.1 Service Headless pour StatefulSet

```yaml
apiVersion: v1
kind: Service
metadata:
  name: mon-service-headless
  namespace: default
spec:
  # Service headless (pas d'IP de cluster)
  clusterIP: None
  selector:
    app: mon-application
  ports:
  - port: 3306
    name: mysql
```

---

## 9. Template Job

Les Jobs exécutent des tâches ponctuelles.

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: mon-job
  namespace: default
spec:
  # Nombre de complétions réussies requises
  completions: 1
  # Nombre de Pods parallèles
  parallelism: 1
  # Limite de temps pour compléter le Job (en secondes)
  activeDeadlineSeconds: 600
  # Nombre de tentatives avant échec
  backoffLimit: 4
  # Template du Pod
  template:
    metadata:
      labels:
        app: mon-job
    spec:
      # Ne pas redémarrer automatiquement
      restartPolicy: OnFailure
      containers:
      - name: job-conteneur
        image: busybox:1.35
        command:
        - /bin/sh
        - -c
        - |
          echo "Début du Job"
          sleep 10
          echo "Job terminé avec succès"
        resources:
          requests:
            memory: "64Mi"
            cpu: "100m"
          limits:
            memory: "128Mi"
            cpu: "200m"
```

---

## 10. Template CronJob

Les CronJobs planifient des tâches récurrentes.

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: mon-cronjob
  namespace: default
spec:
  # Expression cron (minute heure jour mois jour-semaine)
  # Exemples:
  # "*/5 * * * *" - toutes les 5 minutes
  # "0 * * * *" - toutes les heures
  # "0 0 * * *" - tous les jours à minuit
  # "0 0 * * 0" - tous les dimanches à minuit
  schedule: "0 2 * * *"  # Tous les jours à 2h du matin
  # Fuseau horaire (nécessite Kubernetes 1.25+)
  timeZone: "Europe/Paris"
  # Politique de concurrence (Allow, Forbid, Replace)
  concurrencyPolicy: Forbid
  # Nombre de Jobs réussis à conserver
  successfulJobsHistoryLimit: 3
  # Nombre de Jobs échoués à conserver
  failedJobsHistoryLimit: 1
  # Délai de démarrage en secondes (optionnel)
  startingDeadlineSeconds: 200
  # Template du Job
  jobTemplate:
    spec:
      template:
        metadata:
          labels:
            app: mon-cronjob
        spec:
          restartPolicy: OnFailure
          containers:
          - name: cronjob-conteneur
            image: busybox:1.35
            command:
            - /bin/sh
            - -c
            - |
              echo "Exécution du CronJob: $(date)"
              # Votre script ici
              echo "Backup terminé"
            resources:
              requests:
                memory: "64Mi"
                cpu: "100m"
              limits:
                memory: "128Mi"
                cpu: "200m"
```

---

## 11. Template Ingress

Les Ingress gèrent le routage HTTP/HTTPS externe.

### 11.1 Ingress Simple (routage par nom d'hôte)

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: mon-ingress
  namespace: default
  annotations:
    # Annotations spécifiques à NGINX Ingress
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  # Classe d'Ingress (nginx, traefik, etc.)
  ingressClassName: nginx
  # Règles de routage
  rules:
  - host: mon-application.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: mon-service
            port:
              number: 80
```

### 11.2 Ingress avec TLS/SSL

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: mon-ingress-tls
  namespace: default
  annotations:
    # Redirection HTTPS
    nginx.ingress.kubernetes.io/force-ssl-redirect: "true"
    # Émetteur de certificat (Cert-Manager)
    cert-manager.io/cluster-issuer: "letsencrypt-prod"
spec:
  ingressClassName: nginx
  # Configuration TLS
  tls:
  - hosts:
    - mon-application.example.com
    # Nom du Secret contenant le certificat
    secretName: mon-application-tls
  rules:
  - host: mon-application.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: mon-service
            port:
              number: 80
```

### 11.3 Ingress Multi-Domaines et Multi-Chemins

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: mon-ingress-multi
  namespace: default
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /$2
spec:
  ingressClassName: nginx
  tls:
  - hosts:
    - app1.example.com
    - app2.example.com
    secretName: multi-domain-tls
  rules:
  # Premier domaine
  - host: app1.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: service-app1
            port:
              number: 80
  # Deuxième domaine
  - host: app2.example.com
    http:
      paths:
      - path: /api(/|$)(.*)
        pathType: Prefix
        backend:
          service:
            name: service-app2-api
            port:
              number: 8080
      - path: /
        pathType: Prefix
        backend:
          service:
            name: service-app2-frontend
            port:
              number: 80
```

---

## 12. Template NetworkPolicy

Les NetworkPolicies contrôlent le trafic réseau entre les Pods.

### 12.1 NetworkPolicy - Blocage par Défaut

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: deny-all
  namespace: default
spec:
  # Sélectionner tous les Pods du namespace
  podSelector: {}
  # Types de trafic à appliquer
  policyTypes:
  - Ingress
  - Egress
  # Pas de règles = tout est bloqué
```

### 12.2 NetworkPolicy - Autorisation Sélective

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-specific
  namespace: default
spec:
  # Pods cibles
  podSelector:
    matchLabels:
      app: mon-application
  policyTypes:
  - Ingress
  - Egress
  # Règles d'entrée
  ingress:
  # Autoriser depuis des Pods spécifiques
  - from:
    - podSelector:
        matchLabels:
          app: frontend
    # Sur des ports spécifiques
    ports:
    - protocol: TCP
      port: 80
  # Règles de sortie
  egress:
  # Autoriser vers des Pods spécifiques
  - to:
    - podSelector:
        matchLabels:
          app: database
    ports:
    - protocol: TCP
      port: 5432
  # Autoriser DNS
  - to:
    - namespaceSelector:
        matchLabels:
          name: kube-system
    ports:
    - protocol: UDP
      port: 53
```

---

## 13. Template HorizontalPodAutoscaler (HPA)

Le HPA ajuste automatiquement le nombre de réplicas selon la charge.

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: mon-hpa
  namespace: default
spec:
  # Référence à la ressource à scaler
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: mon-deployment
  # Nombre minimum de réplicas
  minReplicas: 2
  # Nombre maximum de réplicas
  maxReplicas: 10
  # Métriques à surveiller
  metrics:
  # Basé sur l'utilisation CPU
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        # Pourcentage cible d'utilisation CPU
        averageUtilization: 70
  # Basé sur l'utilisation mémoire
  - type: Resource
    resource:
      name: memory
      target:
        type: Utilization
        averageUtilization: 80
  # Comportement de scaling
  behavior:
    scaleDown:
      # Politique de réduction
      stabilizationWindowSeconds: 300
      policies:
      - type: Percent
        value: 50
        periodSeconds: 15
    scaleUp:
      # Politique d'augmentation
      stabilizationWindowSeconds: 0
      policies:
      - type: Percent
        value: 100
        periodSeconds: 15
      - type: Pods
        value: 4
        periodSeconds: 15
      selectPolicy: Max
```

---

## 14. Template ResourceQuota

Les ResourceQuotas limitent l'utilisation des ressources dans un namespace.

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: mon-quota
  namespace: default
spec:
  hard:
    # Limites de calcul
    requests.cpu: "4"
    requests.memory: 8Gi
    limits.cpu: "8"
    limits.memory: 16Gi
    # Limites de stockage
    requests.storage: 100Gi
    persistentvolumeclaims: "10"
    # Limites d'objets
    pods: "20"
    services: "10"
    services.loadbalancers: "2"
    services.nodeports: "5"
    configmaps: "20"
    secrets: "20"
```

---

## 15. Template LimitRange

Les LimitRanges définissent des contraintes par défaut pour les ressources.

```yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: mon-limitrange
  namespace: default
spec:
  limits:
  # Limites pour les conteneurs
  - type: Container
    # Valeurs par défaut si non spécifiées
    default:
      cpu: "500m"
      memory: "512Mi"
    # Requêtes par défaut si non spécifiées
    defaultRequest:
      cpu: "250m"
      memory: "256Mi"
    # Valeurs maximales
    max:
      cpu: "2"
      memory: "2Gi"
    # Valeurs minimales
    min:
      cpu: "100m"
      memory: "128Mi"
  # Limites pour les Pods
  - type: Pod
    max:
      cpu: "4"
      memory: "4Gi"
  # Limites pour les PVC
  - type: PersistentVolumeClaim
    max:
      storage: "50Gi"
    min:
      storage: "1Gi"
```

---

## 16. Template ServiceAccount

Les ServiceAccounts identifient les Pods pour le contrôle d'accès.

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: mon-serviceaccount
  namespace: default
  labels:
    app: mon-application
# Secrets associés (automatiquement créés)
automountServiceAccountToken: true
```

### 16.1 Utilisation d'un ServiceAccount dans un Pod

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-avec-sa
  namespace: default
spec:
  # Référence au ServiceAccount
  serviceAccountName: mon-serviceaccount
  containers:
  - name: mon-conteneur
    image: nginx:1.21
```

---

## 17. Template RBAC (Role et RoleBinding)

Le RBAC contrôle les autorisations d'accès aux ressources.

### 17.1 Role (autorisations dans un namespace)

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: mon-role
  namespace: default
rules:
# Autoriser la lecture des Pods
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["get", "list", "watch"]
# Autoriser la gestion des Deployments
- apiGroups: ["apps"]
  resources: ["deployments"]
  verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]
# Autoriser la lecture des logs
- apiGroups: [""]
  resources: ["pods/log"]
  verbs: ["get", "list"]
```

### 17.2 RoleBinding (lier un Role à un utilisateur/ServiceAccount)

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: mon-rolebinding
  namespace: default
subjects:
# Lier à un ServiceAccount
- kind: ServiceAccount
  name: mon-serviceaccount
  namespace: default
# OU lier à un utilisateur
- kind: User
  name: mon-utilisateur
  apiGroup: rbac.authorization.k8s.io
roleRef:
  # Référence au Role
  kind: Role
  name: mon-role
  apiGroup: rbac.authorization.k8s.io
```

### 17.3 ClusterRole et ClusterRoleBinding (pour tout le cluster)

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: mon-clusterrole
rules:
- apiGroups: [""]
  resources: ["nodes"]
  verbs: ["get", "list", "watch"]
- apiGroups: [""]
  resources: ["namespaces"]
  verbs: ["get", "list"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: mon-clusterrolebinding
subjects:
- kind: ServiceAccount
  name: mon-serviceaccount
  namespace: default
roleRef:
  kind: ClusterRole
  name: mon-clusterrole
  apiGroup: rbac.authorization.k8s.io
```

---

## 18. Template DaemonSet

Les DaemonSets déploient un Pod sur chaque nœud du cluster.

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: mon-daemonset
  namespace: default
  labels:
    app: monitoring-agent
spec:
  selector:
    matchLabels:
      app: monitoring-agent
  # Stratégie de mise à jour
  updateStrategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 1
  template:
    metadata:
      labels:
        app: monitoring-agent
    spec:
      # Tolérations pour s'exécuter sur les nœuds master
      tolerations:
      - key: node-role.kubernetes.io/master
        effect: NoSchedule
      - key: node-role.kubernetes.io/control-plane
        effect: NoSchedule
      containers:
      - name: agent
        image: monitoring-agent:1.0
        resources:
          requests:
            memory: "64Mi"
            cpu: "100m"
          limits:
            memory: "128Mi"
            cpu: "200m"
        volumeMounts:
        # Monter le système de fichiers hôte en lecture seule
        - name: host-root
          mountPath: /host
          readOnly: true
      volumes:
      - name: host-root
        hostPath:
          path: /
          type: Directory
```

---

## 19. Templates Avancés - Application Complète

Voici un exemple d'application complète avec tous les composants.

### 19.1 Namespace

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: mon-app
  labels:
    name: mon-app
    environment: production
```

### 19.2 ConfigMap et Secret

```yaml
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
  namespace: mon-app
data:
  app.conf: |
    server {
      listen 80;
      server_name _;
      location / {
        proxy_pass http://backend:8080;
      }
    }
---
apiVersion: v1
kind: Secret
metadata:
  name: app-secrets
  namespace: mon-app
type: Opaque
stringData:
  db-password: super-secret-password
  api-key: ma-cle-api-secrete
```

### 19.3 PersistentVolumeClaim

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: app-data
  namespace: mon-app
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 5Gi
  storageClassName: microk8s-hostpath
```

### 19.4 Deployment Backend

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: backend
  namespace: mon-app
spec:
  replicas: 2
  selector:
    matchLabels:
      app: backend
  template:
    metadata:
      labels:
        app: backend
    spec:
      containers:
      - name: backend
        image: mon-backend:1.0
        ports:
        - containerPort: 8080
        env:
        - name: DB_PASSWORD
          valueFrom:
            secretKeyRef:
              name: app-secrets
              key: db-password
        - name: API_KEY
          valueFrom:
            secretKeyRef:
              name: app-secrets
              key: api-key
        resources:
          requests:
            memory: "256Mi"
            cpu: "250m"
          limits:
            memory: "512Mi"
            cpu: "500m"
        volumeMounts:
        - name: data
          mountPath: /app/data
      volumes:
      - name: data
        persistentVolumeClaim:
          claimName: app-data
```

### 19.5 Service Backend

```yaml
apiVersion: v1
kind: Service
metadata:
  name: backend
  namespace: mon-app
spec:
  selector:
    app: backend
  ports:
  - port: 8080
    targetPort: 8080
```

### 19.6 Deployment Frontend

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: frontend
  namespace: mon-app
spec:
  replicas: 3
  selector:
    matchLabels:
      app: frontend
  template:
    metadata:
      labels:
        app: frontend
    spec:
      containers:
      - name: nginx
        image: nginx:1.21
        ports:
        - containerPort: 80
        volumeMounts:
        - name: nginx-config
          mountPath: /etc/nginx/conf.d
        resources:
          requests:
            memory: "64Mi"
            cpu: "100m"
          limits:
            memory: "128Mi"
            cpu: "200m"
      volumes:
      - name: nginx-config
        configMap:
          name: app-config
```

### 19.7 Service Frontend

```yaml
apiVersion: v1
kind: Service
metadata:
  name: frontend
  namespace: mon-app
spec:
  selector:
    app: frontend
  ports:
  - port: 80
    targetPort: 80
```

### 19.8 Ingress

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: app-ingress
  namespace: mon-app
  annotations:
    cert-manager.io/cluster-issuer: "letsencrypt-prod"
    nginx.ingress.kubernetes.io/force-ssl-redirect: "true"
spec:
  ingressClassName: nginx
  tls:
  - hosts:
    - mon-app.example.com
    secretName: app-tls
  rules:
  - host: mon-app.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: frontend
            port:
              number: 80
```

### 19.9 HorizontalPodAutoscaler

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: backend-hpa
  namespace: mon-app
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: backend
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

---

## 20. Conseils et Bonnes Pratiques

### 20.1 Organisation des Fichiers

```
mon-projet/
├── namespaces/
│   └── namespace.yaml
├── configmaps/
│   ├── app-config.yaml
│   └── database-config.yaml
├── secrets/
│   └── app-secrets.yaml
├── deployments/
│   ├── backend-deployment.yaml
│   └── frontend-deployment.yaml
├── services/
│   ├── backend-service.yaml
│   └── frontend-service.yaml
├── ingress/
│   └── app-ingress.yaml
└── storage/
    └── pvc.yaml
```

### 20.2 Validation des Manifestes

```bash
# Valider la syntaxe YAML sans appliquer
kubectl apply --dry-run=client -f mon-fichier.yaml

# Valider avec le serveur (vérifie aussi les politiques)
kubectl apply --dry-run=server -f mon-fichier.yaml

# Linter YAML avec yamllint
yamllint mon-fichier.yaml
```

### 20.3 Application des Manifestes

```bash
# Appliquer un seul fichier
kubectl apply -f mon-fichier.yaml

# Appliquer tous les fichiers d'un répertoire
kubectl apply -f mon-repertoire/

# Appliquer récursivement
kubectl apply -R -f mon-repertoire/

# Supprimer les ressources d'un fichier
kubectl delete -f mon-fichier.yaml
```

### 20.4 Bonnes Pratiques YAML

1. **Toujours spécifier les versions d'images** : `nginx:1.21` au lieu de `nginx:latest`
2. **Définir les ressources** : requests et limits pour CPU et mémoire
3. **Utiliser des labels cohérents** : facilite la sélection et le tri
4. **Ajouter des sondes de santé** : livenessProbe et readinessProbe
5. **Utiliser des Secrets pour les données sensibles** : jamais en clair dans les ConfigMaps
6. **Documenter avec des annotations** : descriptions, URLs, contacts
7. **Versionner vos manifestes** : utilisez Git
8. **Séparer les environnements** : dev, staging, production dans des namespaces différents
9. **Utiliser kustomize ou Helm** : pour gérer les variations entre environnements
10. **Tester avant de déployer** : utilisez --dry-run

### 20.5 Commandes Utiles

```bash
# Obtenir le template d'une ressource
kubectl create deployment nginx --image=nginx --dry-run=client -o yaml

# Exporter une ressource existante
kubectl get deployment mon-deployment -o yaml > mon-deployment.yaml

# Afficher les différences avant application
kubectl diff -f mon-fichier.yaml

# Obtenir les événements liés à une ressource
kubectl describe pod mon-pod

# Afficher les logs
kubectl logs mon-pod

# Mode interactif dans un Pod
kubectl exec -it mon-pod -- /bin/sh
```

---

## Conclusion

Ces templates constituent une base solide pour déployer et gérer vos applications sur MicroK8s. N'hésitez pas à les adapter à vos besoins spécifiques et à les enrichir au fur et à mesure de votre apprentissage de Kubernetes.

Pour aller plus loin :
- Consultez la documentation officielle Kubernetes : https://kubernetes.io/docs/
- Explorez les exemples de la communauté
- Pratiquez avec différents scénarios de déploiement
- Automatisez avec des outils comme Helm ou Kustomize

Bon déploiement ! 🚀

⏭️ [Exemples de Helm Charts](/annexes/annexe-b-templates-et-exemples/exemples-de-helm-charts.md)
