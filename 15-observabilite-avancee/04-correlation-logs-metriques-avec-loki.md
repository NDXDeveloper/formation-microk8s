🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 15.4 Corrélation Logs-Métriques avec Loki

## Introduction à Loki

Dans la section précédente, nous avons vu la stack EFK pour centraliser les logs. Mais il existe une alternative **plus légère** et **parfaitement intégrée à l'écosystème Prometheus/Grafana** : **Loki**.

### Qu'est-ce que Loki ?

**Loki** est un système d'agrégation de logs créé par Grafana Labs, conçu pour être :
- **Léger** : Consomme beaucoup moins de ressources qu'Elasticsearch
- **Simple** : Plus facile à déployer et maintenir
- **Intégré** : Natif dans Grafana, aux côtés de Prometheus
- **Efficace** : Optimisé pour les logs applicatifs de Kubernetes

**Slogan officiel** : "Like Prometheus, but for logs"

### Pourquoi Loki pour un Lab MicroK8s ?

| Aspect | Elasticsearch (EFK) | Loki |
|--------|---------------------|------|
| **Mémoire** | 2-4 GB minimum | 200-500 MB |
| **CPU** | Intensif | Léger |
| **Stockage** | Index complets | Labels seulement |
| **Recherche** | Full-text rapide | Par labels + grep |
| **Setup** | Complexe (3 composants) | Simple (2 composants) |
| **Coût** | Élevé pour petit lab | Faible |
| **Grafana** | Intégration possible | Natif |
| **Courbe d'apprentissage** | Moyenne | Facile si vous connaissez Prometheus |

**Recommandation** :
- **Lab personnel** ou ressources limitées → **Loki** ✅
- Production avec recherche complexe → EFK
- Besoin de corrélation logs-métriques dans Grafana → **Loki** ✅

## Philosophie de Loki vs Elasticsearch

### Elasticsearch : Index Everything

Elasticsearch **indexe tout le contenu** des logs :

```
Log: "User john.doe@example.com logged in from IP 192.168.1.100"

Index Elasticsearch:
- "User" → Doc 1
- "john" → Doc 1
- "doe" → Doc 1
- "example" → Doc 1
- "com" → Doc 1
- "logged" → Doc 1
- "in" → Doc 1
- "from" → Doc 1
- "IP" → Doc 1
- "192" → Doc 1
- "168" → Doc 1
... (chaque mot est indexé)
```

**Résultat** :
- ✅ Recherche full-text ultra-rapide
- ❌ Énorme consommation de stockage
- ❌ Beaucoup de CPU/mémoire pour indexer

### Loki : Index Labels Only

Loki **n'indexe que les labels** (métadonnées), pas le contenu :

```
Log: "User john.doe@example.com logged in from IP 192.168.1.100"

Index Loki:
Labels:
- namespace: "production"
- pod: "auth-service-abc123"
- app: "auth"
- level: "INFO"

Le contenu du log est stocké brut, non indexé.
```

**Résultat** :
- ✅ Stockage minimal
- ✅ Faible consommation CPU/mémoire
- ⚠️ Recherche full-text plus lente (grep sur les logs)

### Analogie

**Elasticsearch** = Bibliothèque où chaque mot de chaque livre est indexé
- Trouvez "dragon" instantanément dans tous les livres
- Nécessite un système d'indexation massif

**Loki** = Bibliothèque organisée par catégories (labels)
- Trouvez tous les livres fantasy (label)
- Puis feuilletez pour trouver "dragon"
- Plus simple, moins d'espace

## Architecture de Loki

### Composants

```
┌────────────────────────────────────────────────────┐
│          Cluster MicroK8s                          │
│                                                    │
│  ┌────────┐  ┌────────┐  ┌────────┐                │
│  │ Pod 1  │  │ Pod 2  │  │ Pod 3  │                │
│  │ logs   │  │ logs   │  │ logs   │                │
│  └───┬────┘  └───┬────┘  └───┬────┘                │
│      │           │           │                     │
│      └───────────┴───────────┘                     │
│                  ↓                                 │
│        ┌───────────────────┐                       │
│        │  Promtail         │ ← DaemonSet           │
│        │  (Agent)          │   (collecte logs)     │
│        └─────────┬─────────┘                       │
└──────────────────┼─────────────────────────────────┘
                   ↓
         ┌─────────────────┐
         │      Loki       │ ← StatefulSet
         │   (Stockage)    │   (agrège et stocke)
         └─────────┬───────┘
                   ↓
         ┌─────────────────┐
         │    Grafana      │ ← Interface de requête
         │  (Visualisation)│   et visualisation
         └─────────────────┘
              ↑
              │ Aussi connecté à
              ↓
         ┌─────────────────┐
         │   Prometheus    │ ← Métriques
         └─────────────────┘
```

### Les Composants Détaillés

#### 1. Promtail (Agent de Collecte)

**Rôle** : Collecter les logs et les envoyer à Loki

**Caractéristiques** :
- Déployé en **DaemonSet** (un pod par nœud)
- Lit les logs des containers
- Extrait et ajoute des **labels** (namespace, pod, app, etc.)
- Parsing des logs pour enrichir les labels
- Léger : ~40-80 MB de RAM

**Similaire à** : Fluentd dans la stack EFK

#### 2. Loki (Serveur Central)

**Rôle** : Recevoir, indexer (labels uniquement) et stocker les logs

**Caractéristiques** :
- Déployé en **StatefulSet** (besoin de stockage persistant)
- Index seulement les labels (pas le contenu)
- Stockage en chunks compressés
- API pour requêter les logs
- ~200-500 MB de RAM

**Similaire à** : Elasticsearch dans la stack EFK (mais beaucoup plus léger)

#### 3. Grafana (Interface)

**Rôle** : Visualiser les logs ET les métriques dans la même interface

**Caractéristiques** :
- Datasource Loki intégrée nativement
- Explore UI pour naviguer dans les logs
- Dashboards unifiés logs + métriques
- Corrélation automatique

**Vous l'avez déjà** : Si vous avez suivi le chapitre 13 !

## Déploiement de Loki dans MicroK8s

### Option 1 : Via l'Addon MicroK8s (Recommandé)

MicroK8s propose un addon pour Loki qui simplifie grandement l'installation :

```bash
# Activer l'addon observability (inclut Loki)
microk8s enable observability

# Cet addon installe automatiquement :
# - Prometheus
# - Grafana
# - Loki
# - Promtail
```

**Avantages** :
- ✅ Installation en une commande
- ✅ Configuration pré-optimisée pour MicroK8s
- ✅ Intégration automatique dans Grafana

**Inconvénient** :
- ⚠️ Moins de contrôle sur la configuration

### Option 2 : Déploiement Manuel (Plus de Contrôle)

Si vous voulez plus de contrôle ou si vous avez déjà Prometheus/Grafana :

#### Créer le Namespace

```bash
microk8s kubectl create namespace monitoring
# Ou réutilisez votre namespace existant
```

#### 1. Déployer Loki

**ConfigMap Loki**

Fichier `loki-config.yaml` :

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: loki-config
  namespace: monitoring
data:
  loki.yaml: |
    auth_enabled: false

    server:
      http_listen_port: 3100
      grpc_listen_port: 9096

    common:
      path_prefix: /loki
      storage:
        filesystem:
          chunks_directory: /loki/chunks
          rules_directory: /loki/rules
      replication_factor: 1
      ring:
        instance_addr: 127.0.0.1
        kvstore:
          store: inmemory

    schema_config:
      configs:
        - from: 2020-10-24
          store: boltdb-shipper
          object_store: filesystem
          schema: v11
          index:
            prefix: index_
            period: 24h

    limits_config:
      retention_period: 168h  # 7 jours
      reject_old_samples: true
      reject_old_samples_max_age: 168h

    chunk_store_config:
      max_look_back_period: 168h

    table_manager:
      retention_deletes_enabled: true
      retention_period: 168h
```

**Points importants** :
- `retention_period: 168h` : Garde les logs 7 jours
- `filesystem` : Stockage local (pour un lab)
- `replication_factor: 1` : Une seule instance

**Service Loki**

Fichier `loki-service.yaml` :

```yaml
apiVersion: v1
kind: Service
metadata:
  name: loki
  namespace: monitoring
  labels:
    app: loki
spec:
  selector:
    app: loki
  ports:
  - name: http
    port: 3100
    targetPort: 3100
  type: ClusterIP
```

**StatefulSet Loki**

Fichier `loki-statefulset.yaml` :

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: loki
  namespace: monitoring
spec:
  serviceName: loki
  replicas: 1
  selector:
    matchLabels:
      app: loki
  template:
    metadata:
      labels:
        app: loki
    spec:
      containers:
      - name: loki
        image: grafana/loki:2.9.3
        args:
        - -config.file=/etc/loki/loki.yaml
        ports:
        - name: http
          containerPort: 3100
        - name: grpc
          containerPort: 9096
        resources:
          requests:
            memory: "256Mi"
            cpu: "200m"
          limits:
            memory: "512Mi"
            cpu: "500m"
        volumeMounts:
        - name: config
          mountPath: /etc/loki
        - name: storage
          mountPath: /loki
      volumes:
      - name: config
        configMap:
          name: loki-config
  volumeClaimTemplates:
  - metadata:
      name: storage
    spec:
      accessModes: ["ReadWriteOnce"]
      storageClassName: "microk8s-hostpath"
      resources:
        requests:
          storage: 10Gi
```

**Déployer Loki** :

```bash
microk8s kubectl apply -f loki-config.yaml
microk8s kubectl apply -f loki-service.yaml
microk8s kubectl apply -f loki-statefulset.yaml

# Vérifier
microk8s kubectl get pods -n monitoring -l app=loki
microk8s kubectl logs -n monitoring loki-0
```

#### 2. Déployer Promtail

**ConfigMap Promtail**

Fichier `promtail-config.yaml` :

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: promtail-config
  namespace: monitoring
data:
  promtail.yaml: |
    server:
      http_listen_port: 9080
      grpc_listen_port: 0

    positions:
      filename: /tmp/positions.yaml

    clients:
      - url: http://loki.monitoring.svc.cluster.local:3100/loki/api/v1/push

    scrape_configs:
      # Logs des pods
      - job_name: kubernetes-pods
        kubernetes_sd_configs:
          - role: pod
        pipeline_stages:
          - cri: {}
        relabel_configs:
          # Copier les labels Kubernetes
          - source_labels:
              - __meta_kubernetes_pod_node_name
            target_label: __host__
          - action: labelmap
            regex: __meta_kubernetes_pod_label_(.+)
          - action: replace
            replacement: $1
            separator: /
            source_labels:
              - __meta_kubernetes_namespace
              - __meta_kubernetes_pod_name
            target_label: job
          - action: replace
            source_labels:
              - __meta_kubernetes_namespace
            target_label: namespace
          - action: replace
            source_labels:
              - __meta_kubernetes_pod_name
            target_label: pod
          - action: replace
            source_labels:
              - __meta_kubernetes_pod_container_name
            target_label: container
          - replacement: /var/log/pods/*$1/*.log
            separator: /
            source_labels:
              - __meta_kubernetes_pod_uid
              - __meta_kubernetes_pod_container_name
            target_label: __path__
```

**ServiceAccount et RBAC**

Fichier `promtail-rbac.yaml` :

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: promtail
  namespace: monitoring
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: promtail
rules:
- apiGroups:
  - ""
  resources:
  - nodes
  - nodes/proxy
  - services
  - endpoints
  - pods
  verbs:
  - get
  - watch
  - list
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: promtail
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: promtail
subjects:
- kind: ServiceAccount
  name: promtail
  namespace: monitoring
```

**DaemonSet Promtail**

Fichier `promtail-daemonset.yaml` :

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: promtail
  namespace: monitoring
spec:
  selector:
    matchLabels:
      app: promtail
  template:
    metadata:
      labels:
        app: promtail
    spec:
      serviceAccountName: promtail
      containers:
      - name: promtail
        image: grafana/promtail:2.9.3
        args:
        - -config.file=/etc/promtail/promtail.yaml
        env:
        - name: HOSTNAME
          valueFrom:
            fieldRef:
              fieldPath: spec.nodeName
        resources:
          requests:
            memory: "64Mi"
            cpu: "100m"
          limits:
            memory: "128Mi"
            cpu: "200m"
        volumeMounts:
        - name: config
          mountPath: /etc/promtail
        - name: varlog
          mountPath: /var/log
          readOnly: true
        - name: varlibdockercontainers
          mountPath: /var/lib/docker/containers
          readOnly: true
      volumes:
      - name: config
        configMap:
          name: promtail-config
      - name: varlog
        hostPath:
          path: /var/log
      - name: varlibdockercontainers
        hostPath:
          path: /var/lib/docker/containers
```

**Déployer Promtail** :

```bash
microk8s kubectl apply -f promtail-config.yaml
microk8s kubectl apply -f promtail-rbac.yaml
microk8s kubectl apply -f promtail-daemonset.yaml

# Vérifier (un pod par nœud)
microk8s kubectl get daemonset -n monitoring
microk8s kubectl get pods -n monitoring -l app=promtail
microk8s kubectl logs -n monitoring -l app=promtail --tail=20
```

#### 3. Configurer Grafana

**Ajouter Loki comme Datasource**

Si Grafana est déjà déployé (chapitre 13) :

1. Connectez-vous à Grafana
2. **Configuration** → **Data Sources**
3. **Add data source**
4. Sélectionnez **Loki**
5. Configurez :
   - **Name** : Loki
   - **URL** : `http://loki.monitoring.svc.cluster.local:3100`
   - **Access** : Server (default)
6. **Save & Test**

**Via ConfigMap (automatique)** :

Fichier `grafana-datasource-loki.yaml` :

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: grafana-datasource-loki
  namespace: monitoring
  labels:
    grafana_datasource: "1"
data:
  loki-datasource.yaml: |
    apiVersion: 1
    datasources:
    - name: Loki
      type: loki
      access: proxy
      url: http://loki.monitoring.svc.cluster.local:3100
      isDefault: false
      editable: true
```

```bash
microk8s kubectl apply -f grafana-datasource-loki.yaml

# Redémarrez Grafana pour charger la datasource
microk8s kubectl rollout restart deployment grafana -n monitoring
```

### Vérification Complète

```bash
# Tous les composants doivent être UP
microk8s kubectl get all -n monitoring

# Tester Loki directement
microk8s kubectl port-forward -n monitoring svc/loki 3100:3100

# Dans un autre terminal
curl http://localhost:3100/ready
# Devrait retourner "ready"

curl http://localhost:3100/metrics
# Devrait afficher les métriques Loki
```

## LogQL : Le Langage de Requête de Loki

LogQL est le langage pour requêter Loki, similaire à PromQL pour Prometheus.

### Structure d'une Requête

```
{label="value"} |= "search text" | filter | aggregation
```

**Composants** :
1. **Log Stream Selector** : `{label="value"}` - Sélectionne les flux de logs
2. **Filter Expression** : `|= "text"` - Filtre sur le contenu
3. **Parser** : `| json` - Parse le contenu
4. **Aggregation** : `| count_over_time()` - Agrège les résultats

### Sélecteurs de Flux (Log Stream Selectors)

**Syntaxe** : `{label=value}`

**Exemples** :

```logql
# Tous les logs du namespace production
{namespace="production"}

# Logs d'une application spécifique
{app="payment-service"}

# Combiner plusieurs labels
{namespace="production", app="payment-service"}

# Logs d'un pod spécifique
{pod="payment-service-7d8c9-x4b2k"}

# Utiliser les labels Kubernetes
{namespace="production", container="api"}
```

### Opérateurs de Filtrage

#### `|=` : Contient (case-sensitive)

```logql
# Logs contenant "error"
{app="payment"} |= "error"
```

#### `!=` : Ne contient pas

```logql
# Logs ne contenant pas "debug"
{app="payment"} != "debug"
```

#### `|~` : Regex correspond

```logql
# Logs correspondant au pattern ERROR|WARN
{app="payment"} |~ "ERROR|WARN"
```

#### `!~` : Regex ne correspond pas

```logql
# Logs ne correspondant pas au pattern DEBUG|TRACE
{app="payment"} !~ "DEBUG|TRACE"
```

#### Chaîner les filtres

```logql
# Logs contenant "error" ET "database"
{app="payment"} |= "error" |= "database"

# Logs contenant "error" mais pas "timeout"
{app="payment"} |= "error" != "timeout"
```

### Parsers

Si vos logs sont structurés (JSON, logfmt), vous pouvez les parser :

#### JSON Parser

```logql
{app="payment"} | json
```

Exemple de log JSON :
```json
{"level":"ERROR","message":"Payment failed","user_id":"12345"}
```

Après parsing, vous pouvez utiliser les champs :
```logql
{app="payment"} | json | level="ERROR"
```

#### Logfmt Parser

```logql
{app="payment"} | logfmt
```

Exemple de log logfmt :
```
level=error message="Payment failed" user_id=12345
```

#### Pattern Parser

Pour parser des logs non structurés :

```logql
{app="payment"}
| pattern `<timestamp> <level> <message>`
| level = "ERROR"
```

### Agrégations et Métriques

#### count_over_time

Compte les logs sur une période :

```logql
# Nombre de logs par minute
count_over_time({app="payment"}[1m])

# Logs d'erreur par minute
count_over_time({app="payment"} |= "error"[1m])
```

#### rate

Taux de logs par seconde :

```logql
# Logs par seconde
rate({app="payment"}[5m])

# Erreurs par seconde
rate({app="payment"} |= "error"[5m])
```

#### bytes_over_time

Volume de logs en bytes :

```logql
bytes_over_time({app="payment"}[1h])
```

#### sum by

Grouper et sommer :

```logql
# Nombre de logs par namespace
sum by (namespace) (count_over_time({job="kubernetes-pods"}[1m]))

# Erreurs par application
sum by (app) (count_over_time({namespace="production"} |= "error"[5m]))
```

## Corrélation Logs-Métriques : Le Cœur du Sujet

La **vraie magie** de Loki avec Grafana, c'est la capacité à **corréler logs et métriques** dans la même interface.

### Pourquoi la Corrélation est Importante

**Scénario typique** :

1. Vous voyez une **métrique anormale** dans Prometheus :
   ```promql
   # Latence API soudainement à 5 secondes
   http_request_duration_seconds_p95 > 5
   ```

2. Vous vous demandez **"Pourquoi ?"**

3. Avec la corrélation, vous pouvez **immédiatement voir les logs** au moment exact où la métrique a dévié

### Méthodes de Corrélation

#### 1. Split View dans Grafana Explore

**Grafana Explore** permet de visualiser logs ET métriques côte à côte.

**Comment faire** :

1. Ouvrez **Grafana** → **Explore**
2. En haut à droite, cliquez sur **Split** (icône de séparation)
3. Panneau gauche : Sélectionnez datasource **Prometheus**
4. Panneau droit : Sélectionnez datasource **Loki**

**Panneau Prometheus (gauche)** :
```promql
rate(http_requests_total{app="payment"}[5m])
```

**Panneau Loki (droit)** :
```logql
{app="payment"} |= "error"
```

**Synchronisation temporelle** :
- Les deux panneaux partagent le même time range
- Quand vous zoomez sur le graphique de métriques, les logs se mettent à jour automatiquement
- Vous voyez **exactement** quels logs correspondent à la spike de métriques

#### 2. Dashboards Unifiés

Créez des dashboards qui affichent logs ET métriques :

**Exemple de dashboard** :

```
┌─────────────────────────────────────────────┐
│  [Payment Service Overview]                 │
├─────────────────────────────────────────────┤
│                                             │
│  📊 Métriques                               │
│  ┌────────────────────────────────────────┐ │
│  │ Request Rate    [graph]                │ │
│  │ Error Rate      [graph]                │ │
│  │ Latency P95     [graph]                │ │
│  └────────────────────────────────────────┘ │
│                                             │
│  📋 Logs                                    │
│  ┌────────────────────────────────────────┐ │
│  │ Recent Errors   [log panel]            │ │
│  │ - 14:32:15 ERROR Payment timeout       │ │
│  │ - 14:31:42 ERROR DB connection lost    │ │
│  └────────────────────────────────────────┘ │
└─────────────────────────────────────────────┘
```

**Configuration d'un panneau de logs** :

1. **Add panel** → **Logs**
2. **Data source** : Loki
3. **Query** :
   ```logql
   {app="payment-service"} |~ "ERROR|WARN"
   ```
4. **Options** :
   - **Show time** : Yes
   - **Order** : Newest first
   - **Limit** : 100

#### 3. Liens de Navigation

Créez des **liens cliquables** entre panneaux de métriques et logs.

**Dans un panneau de métriques** :

1. **Panel settings** → **Links**
2. **Add link** :
   - **Title** : "View Logs"
   - **URL** : Template vers Explore avec requête Loki
   ```
   /explore?left={"datasource":"Loki","queries":[{"expr":"{app=\"payment-service\"}","refId":"A"}],"range":{"from":"now-5m","to":"now"}}
   ```

Maintenant, quand vous voyez une anomalie dans les métriques, un clic vous amène directement aux logs !

#### 4. Utiliser les Mêmes Labels

**Clé de la corrélation** : Utilisez les **mêmes labels** dans Prometheus et Loki.

**Dans votre application** :

```python
# Métrique Prometheus
http_requests_total.labels(
    app="payment-service",
    namespace="production",
    method="POST"
).inc()

# Log avec les mêmes infos
logger.error(
    "Payment failed",
    extra={
        'app': 'payment-service',
        'namespace': 'production',
        'method': 'POST'
    }
)
```

**Promtail** extrait automatiquement `app` et `namespace` des labels Kubernetes.

**Résultat** :

```promql
# Métrique Prometheus
http_requests_total{app="payment-service", namespace="production"}

# Logs Loki
{app="payment-service", namespace="production"}
```

Même structure de labels = Corrélation facile ! 🎯

### Exemple Pratique de Corrélation

#### Scénario : Debugging d'une Dégradation de Performance

**Étape 1 : Alerte Prometheus**

Une alerte se déclenche :
```
Alert: HighLatency
Expr: http_request_duration_seconds_p95 > 2
For: 5m
```

**Étape 2 : Ouvrir Grafana Explore en Split View**

Panneau gauche (Prometheus) :
```promql
http_request_duration_seconds_p95{app="payment-service"}
```

Vous voyez un pic à 14:32.

**Étape 3 : Requêter les Logs au Même Moment**

Panneau droit (Loki) :
```logql
{app="payment-service"} |~ "error|timeout|slow"
```

Time range : 14:25 - 14:35

**Étape 4 : Analyse des Logs**

Vous trouvez dans les logs :
```
14:32:12 WARN  Database query took 3.2s
14:32:15 ERROR Connection pool exhausted
14:32:18 ERROR Payment processing timeout after 5s
14:32:21 WARN  Database query took 4.1s
```

**Conclusion** : Problème de base de données !

**Étape 5 : Approfondir**

Requête Loki plus précise :
```logql
{app="payment-service"} |= "database" |~ "slow|timeout|error"
```

Agrégation pour voir l'ampleur :
```logql
sum by (level) (count_over_time({app="payment-service"} |= "database"[5m]))
```

**Résultat** :
- Corrélation claire entre le spike de latence et les erreurs DB
- Investigation en quelques minutes au lieu d'heures
- Vous savez maintenant où chercher (pool de connexions DB)

### Patterns de Corrélation Courants

#### Pattern 1 : Pic de Trafic → Logs d'Erreur

**Métrique** :
```promql
rate(http_requests_total[5m])
```

**Logs associés** :
```logql
{namespace="production"} |~ "ERROR|FAIL"
```

**Question** : Le pic de trafic cause-t-il plus d'erreurs ?

#### Pattern 2 : Erreur Rate → Logs Spécifiques

**Métrique** :
```promql
rate(http_requests_total{status=~"5.."}[5m]) / rate(http_requests_total[5m])
```

**Logs associés** :
```logql
{app="api"} | json | status >= 500
```

**Question** : Quels sont les messages d'erreur exacts ?

#### Pattern 3 : Resource Usage → Application Logs

**Métrique** :
```promql
container_memory_usage_bytes{pod=~"payment.*"}
```

**Logs associés** :
```logql
{app="payment-service"} |~ "OutOfMemory|allocation failed|GC"
```

**Question** : Y a-t-il des logs indiquant un problème de mémoire ?

#### Pattern 4 : Déploiement → Impact

**Métrique** :
```promql
# Avant/après déploiement
rate(http_request_duration_seconds_sum[5m]) / rate(http_request_duration_seconds_count[5m])
```

**Logs associés** :
```logql
{app="api"} |= "started" or "deployment"
```

**Annotation de déploiement** dans Grafana pour marquer le moment du déploiement.

**Question** : Le nouveau déploiement a-t-il dégradé les performances ?

## Labels Dérivés : Enrichir les Logs

Loki permet de créer des **labels dérivés** à partir du contenu des logs.

### Exemple : Extraire le Niveau de Log

Si vos logs ont un format :
```
2025-10-25 14:32:15 ERROR Payment failed
```

Configuration Promtail pour extraire `level` :

```yaml
pipeline_stages:
  - regex:
      expression: '^.* (?P<level>\w+) .*$'
  - labels:
      level:
```

Maintenant vous pouvez requêter :
```logql
{app="payment", level="ERROR"}
```

Au lieu de :
```logql
{app="payment"} |~ "ERROR"
```

**Avantage** : Le niveau devient un label indexé = requêtes plus rapides.

### Exemple : Extraire le Status Code HTTP

Log :
```
POST /api/order HTTP/1.1 200 OK 0.123s
```

Pipeline Promtail :
```yaml
pipeline_stages:
  - regex:
      expression: '.* (?P<status_code>\d{3}) .*'
  - labels:
      status_code:
```

Requête :
```logql
{app="api", status_code=~"5.."}  # Toutes les 5xx
```

**Attention** : Ne créez pas trop de labels dérivés (risque de haute cardinalité).

## Grafana : Visualisations Avancées

### Log Panel avec Déduplication

Parfois, les mêmes erreurs se répètent. Le panneau Logs de Grafana peut les dédupliquer :

**Options du panneau** :
- **Deduplication** : Signature
- Grafana regroupe les logs identiques et affiche le compte

**Affichage** :
```
[12x] 14:32:15 ERROR Payment timeout (repeated 12 times)
```

### Annotations Automatiques

Utilisez les logs pour créer des **annotations** automatiques sur les graphiques de métriques.

**Exemple** : Marquer chaque déploiement

**Requête Loki** :
```logql
{app="deployment-tracker"} |= "deployed version"
```

**Configuration** :
1. **Dashboard settings** → **Annotations**
2. **Add annotation query**
3. **Data source** : Loki
4. **Query** : `{app="deployment-tracker"} |= "deployed"`

Maintenant, chaque déploiement apparaît comme une ligne verticale sur vos graphiques !

### Alerting sur les Logs

Grafana peut créer des alertes basées sur les logs Loki :

**Exemple** : Alerte si plus de 10 erreurs par minute

**Requête** :
```logql
sum(rate({app="payment"} |= "ERROR"[1m])) > 10
```

**Alert rule** :
1. **Alerting** → **Alert rules** → **New alert rule**
2. **Query** : La requête LogQL ci-dessus
3. **Condition** : WHEN last() IS ABOVE 10
4. **Notification** : Slack, email, etc.

### Live Tail

**Grafana Explore** offre un mode "tail" en temps réel :

1. **Explore** → **Loki**
2. Entrez votre requête
3. Cliquez sur **Live** (en haut à droite)

Les logs arrivent en **streaming en temps réel** (comme `tail -f`).

**Utile pour** :
- Déboguer en live
- Suivre les déploiements
- Monitoring actif pendant un incident

## Cas d'Usage Pratiques

### 1. Dashboard de Monitoring Complet

**Panels** :

1. **Request Rate** (Prometheus)
   ```promql
   rate(http_requests_total[5m])
   ```

2. **Error Rate** (Prometheus)
   ```promql
   rate(http_requests_total{status=~"5.."}[5m])
   ```

3. **Recent Errors** (Loki)
   ```logql
   {app="api"} |~ "ERROR|FAIL" | json | line_format "{{.timestamp}} {{.level}} {{.message}}"
   ```

4. **Latency P95** (Prometheus)
   ```promql
   histogram_quantile(0.95, rate(http_request_duration_seconds_bucket[5m]))
   ```

5. **Slow Queries** (Loki)
   ```logql
   {app="api"} |= "slow query" | logfmt | duration > 1000
   ```

### 2. Dashboard d'Investigation d'Incident

**Variables de dashboard** :
- `$namespace` : Liste des namespaces
- `$app` : Liste des applications
- `$timerange` : Période d'investigation

**Panels** :

1. **Timeline des Événements**
   - Métriques + Logs + Annotations de déploiement

2. **Volume de Logs par Niveau**
   ```logql
   sum by (level) (count_over_time({namespace="$namespace", app="$app"}[1m]))
   ```

3. **Top 10 Messages d'Erreur**
   ```logql
   topk(10, sum by (message) (count_over_time({namespace="$namespace", app="$app"} |= "ERROR"[$timerange])))
   ```

4. **Logs Détaillés Filtrables**
   ```logql
   {namespace="$namespace", app="$app"} |~ "ERROR|WARN"
   ```

### 3. Dashboard de Santé Application

Pour chaque application critique :

**Section Métriques** :
- Throughput
- Latency
- Error rate
- Resource usage (CPU, memory)

**Section Logs** :
- Recent errors (last 15 min)
- Warnings count
- Critical events

**Lien vers Explore** pour investigation approfondie

## Performance et Optimisation

### Limiter le Volume de Logs

**Dans Promtail** : Filtrez les logs non essentiels avant de les envoyer à Loki.

```yaml
scrape_configs:
  - job_name: kubernetes-pods
    pipeline_stages:
      # Ne pas envoyer les logs DEBUG en production
      - match:
          selector: '{namespace="production"}'
          stages:
            - drop:
                expression: ".*DEBUG.*"

      # Limiter la taille des logs
      - limit:
          rate: 1000
          burst: 2000
```

### Cardinalité des Labels

**Règle d'or** : Gardez la cardinalité des labels **basse**.

**❌ Mauvais** :
```yaml
# Créera des millions de stream
- labels:
    user_id:  # Des millions d'utilisateurs
    request_id:  # Chaque requête unique
```

**✅ Bon** :
```yaml
# Cardinalité contrôlée
- labels:
    app:
    namespace:
    level:
    pod:
```

**Limite recommandée** : < 10 000 streams par pod.

### Rétention et Compaction

**Configuration Loki** :

```yaml
limits_config:
  retention_period: 168h  # 7 jours

compactor:
  working_directory: /loki/compactor
  shared_store: filesystem
  retention_enabled: true
```

**Ajustez selon vos besoins** :
- Lab/dev : 3-7 jours
- Production : 30-90 jours
- Logs critiques : 1 an+

### Queries Efficaces

**❌ Requête inefficace** :
```logql
{namespace="production"} |~ ".*"  # Trop large, lit tout
```

**✅ Requête efficace** :
```logql
{namespace="production", app="payment"} |= "error"  # Ciblé
```

**Tips** :
- Utilisez autant de labels que possible dans le sélecteur
- Filtrez tôt avec `|=` ou `|~`
- Limitez le time range quand possible

## Comparaison Finale : Loki vs EFK

| Critère | Loki | EFK |
|---------|------|-----|
| **Ressources** | ✅ Très léger | ❌ Lourd |
| **Setup** | ✅ Simple | ⚠️ Complexe |
| **Recherche full-text** | ⚠️ Plus lent | ✅ Très rapide |
| **Recherche par labels** | ✅ Très rapide | ⚠️ Moyen |
| **Intégration Grafana** | ✅ Native | ⚠️ Possible |
| **Corrélation métriques** | ✅ Excellente | ❌ Manuelle |
| **Coût stockage** | ✅ Faible | ❌ Élevé |
| **Courbe apprentissage** | ✅ Facile (si PromQL) | ⚠️ Moyenne |
| **Cas d'usage** | Lab, K8s observability | Production complexe |

## Bonnes Pratiques avec Loki

### 1. Logs Structurés

**Préférez JSON** :
```json
{"level":"ERROR","msg":"Payment failed","duration":3.2,"user_id":"12345"}
```

Puis parsez avec `| json` pour exploiter tous les champs.

### 2. Labels Appropriés

**Bons labels** :
- `namespace`, `app`, `pod`, `container` (automatiques via K8s)
- `level` (ERROR, WARN, INFO) si extrait
- `environment` (prod, staging, dev)

**Mauvais labels** :
- Toute valeur à haute cardinalité
- Données temporelles
- IDs uniques

### 3. Utilisez les Filtres

**Filtrez tôt** pour de meilleures performances :
```logql
{app="payment"} |= "error" != "ignore" |~ "timeout|failed"
```

### 4. Nommage Cohérent avec Prometheus

Utilisez les mêmes noms de labels :
```
Prometheus: http_requests_total{app="payment", namespace="prod"}
Loki: {app="payment", namespace="prod"}
```

### 5. Dashboards Template

Créez des **dashboard templates** réutilisables :
- Un template pour chaque type d'application (API, worker, frontend)
- Variables pour `$namespace`, `$app`
- Dupliquez et adaptez

## Ce Qu'il Faut Retenir

🚀 **Loki est idéal pour MicroK8s** :
- Léger : 10x moins de ressources qu'Elasticsearch
- Simple : 2 composants (Loki + Promtail)
- Intégré : Natif dans Grafana

📊 **Corrélation logs-métriques** :
- Même interface (Grafana)
- Split view pour voir logs + métriques côte à côte
- Dashboards unifiés
- Mêmes labels = corrélation facile

🔍 **LogQL** :
- Similaire à PromQL
- Sélecteurs de labels + filtres de contenu
- Parsers pour logs structurés
- Agrégations pour analytics

🎯 **Labels sont clés** :
- Indexés = recherche rapide
- Mais attention à la cardinalité
- Cohérence avec Prometheus

💡 **Utilisez les deux piliers ensemble** :
- Prometheus : Métriques numériques
- Loki : Logs textuels
- Correlation = Debugging ultra-rapide

⚙️ **Configuration optimale** :
- Promtail en DaemonSet
- Loki en StatefulSet avec stockage persistant
- Rétention adaptée (7-30 jours)
- Filtrage des logs non essentiels

📈 **Prochaine étape** :
- Section 15.5 : Tracing distribué avec Jaeger (le 3ème pilier)
- Vous aurez alors les trois piliers complets de l'observabilité !

Avec Loki, vous avez maintenant une solution légère et puissante pour centraliser vos logs et les corréler avec vos métriques Prometheus. L'observabilité de votre cluster MicroK8s est désormais de niveau professionnel ! 🎉

⏭️ [Tracing distribué avec Jaeger](/15-observabilite-avancee/05-tracing-distribue-avec-jaeger.md)
