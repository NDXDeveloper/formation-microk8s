🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 19.6 Custom Metrics

## Introduction aux Custom Metrics

Les **custom metrics (métriques personnalisées)** permettent au HPA de scaler vos applications basé sur des métriques **métier** ou **applicatives** plutôt que simplement sur le CPU et la mémoire. C'est un niveau avancé d'autoscaling qui permet une réactivité beaucoup plus pertinente.

### Le problème avec les métriques CPU/mémoire seules

Jusqu'à présent, nous avons utilisé le HPA avec des métriques CPU et mémoire :

```yaml
metrics:
- type: Resource
  resource:
    name: cpu
    target:
      type: Utilization
      averageUtilization: 50
```

**Limitation de cette approche :**

Imaginons une API web :
- **Scénario 1** : 1000 requêtes/sec, pods à 30% CPU → HPA ne fait rien (sous le seuil de 50%)
- **Scénario 2** : 100 requêtes/sec mais complexes, pods à 70% CPU → HPA scale up

Dans le scénario 1, votre application peut être surchargée (latence élevée) mais le HPA ne réagit pas car le CPU est bas. **C'est un problème !**

**Ce qui serait mieux :**
Scaler basé sur le **nombre de requêtes par seconde** ou la **latence moyenne**, pas le CPU.

### La solution : Custom Metrics

Avec les custom metrics, vous pouvez scaler basé sur :

✅ **Requêtes HTTP par seconde**
✅ **Latence moyenne des réponses**
✅ **Longueur d'une file d'attente de messages**
✅ **Nombre de connexions actives**
✅ **Taux d'erreurs**
✅ **Métriques métier** (commandes en attente, utilisateurs connectés, etc.)

### Analogie simple

**Autoscaling basé sur CPU (ancien) :**
- Restaurant qui embauche des cuisiniers basé sur la température dans la cuisine
- Si la cuisine chauffe → plus de cuisiniers
- **Problème :** La chaleur ne reflète pas directement le nombre de clients

**Autoscaling basé sur custom metrics (moderne) :**
- Restaurant qui embauche des cuisiniers basé sur le nombre de commandes en attente
- 50 commandes en attente → plus de cuisiniers
- **Avantage :** Indicateur direct du besoin réel

## Architecture des Custom Metrics

Pour utiliser les custom metrics avec le HPA, vous avez besoin de trois composants :

```
┌─────────────────────────────────────────────────────────┐
│                  HPA (Horizontal Pod Autoscaler)        │
│  "Je veux scaler basé sur http_requests_per_second"     │
└─────────────────────────────────────────────────────────┘
                           ↓ Interroge
┌─────────────────────────────────────────────────────────┐
│          Custom Metrics API (Prometheus Adapter)        │
│  "Traduction : Prometheus, donne-moi cette métrique"    │
└─────────────────────────────────────────────────────────┘
                           ↓ Interroge
┌─────────────────────────────────────────────────────────┐
│                    Prometheus                           │
│  "J'ai collecté cette métrique depuis vos pods"         │
└─────────────────────────────────────────────────────────┘
                           ↑ Collecte
┌─────────────────────────────────────────────────────────┐
│                  Votre Application                      │
│  Expose /metrics avec http_requests_total, etc.         │
└─────────────────────────────────────────────────────────┘
```

### Les trois composants expliqués

**1. Votre Application**
- Doit **exposer des métriques** au format Prometheus
- Généralement via un endpoint `/metrics`
- Utilise des bibliothèques client Prometheus (disponibles pour tous les langages)

**2. Prometheus**
- **Collecte** les métriques de vos applications (scraping)
- **Stocke** l'historique des métriques
- **Permet de les requêter** via PromQL
- Nous avons vu Prometheus dans la section 12

**3. Prometheus Adapter (Custom Metrics API)**
- **Pont** entre Prometheus et Kubernetes
- Traduit les requêtes du HPA en requêtes PromQL
- Expose la Custom Metrics API (`custom.metrics.k8s.io`)
- Le HPA interroge cette API au lieu de Metrics Server

## Prérequis

Avant d'utiliser les custom metrics, vous devez avoir :

### 1. Prometheus installé

```bash
microk8s enable prometheus
```

Vérification :
```bash
kubectl get pods -n monitoring | grep prometheus
```

### 2. Votre application expose des métriques

Votre application doit exposer un endpoint `/metrics` au format Prometheus.

**Exemple de métriques exposées :**
```
# HELP http_requests_total Total HTTP requests
# TYPE http_requests_total counter
http_requests_total{method="GET",status="200"} 1547

# HELP http_request_duration_seconds HTTP request latency
# TYPE http_request_duration_seconds histogram
http_request_duration_seconds_bucket{le="0.1"} 824
http_request_duration_seconds_bucket{le="0.5"} 1420
http_request_duration_seconds_sum 345.2
http_request_duration_seconds_count 1547
```

Ne vous inquiétez pas si cela semble complexe ! Les bibliothèques client Prometheus génèrent cela automatiquement.

### 3. Prometheus Adapter installé

C'est le composant qui fait le pont entre Prometheus et l'API Kubernetes.

## Installation du Prometheus Adapter

### Méthode 1 : Via Helm (Recommandé)

**Étape 1 : S'assurer que Helm est disponible**

```bash
microk8s enable helm3
alias helm='microk8s helm3'
```

**Étape 2 : Ajouter le repository Helm**

```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update
```

**Étape 3 : Créer un fichier de configuration**

Créez `prometheus-adapter-values.yaml` :

```yaml
prometheus:
  url: http://prometheus-k8s.monitoring.svc
  port: 9090

rules:
  default: true
  custom:
  - seriesQuery: 'http_requests_total{namespace!="",pod!=""}'
    resources:
      overrides:
        namespace: {resource: "namespace"}
        pod: {resource: "pod"}
    name:
      matches: "^(.*)_total"
      as: "${1}_per_second"
    metricsQuery: 'sum(rate(<<.Series>>{<<.LabelMatchers>>}[2m])) by (<<.GroupBy>>)'

  - seriesQuery: 'http_request_duration_seconds_sum{namespace!="",pod!=""}'
    resources:
      overrides:
        namespace: {resource: "namespace"}
        pod: {resource: "pod"}
    name:
      as: "http_request_latency_seconds"
    metricsQuery: 'sum(rate(<<.Series>>{<<.LabelMatchers>>}[2m])) by (<<.GroupBy>>) / sum(rate(http_request_duration_seconds_count{<<.LabelMatchers>>}[2m])) by (<<.GroupBy>>)'

resources:
  requests:
    cpu: 100m
    memory: 128Mi
  limits:
    cpu: 100m
    memory: 128Mi
```

Cette configuration définit des **règles** pour transformer les métriques Prometheus en métriques Kubernetes.

**Étape 4 : Installer le Prometheus Adapter**

```bash
helm install prometheus-adapter prometheus-community/prometheus-adapter \
  --namespace monitoring \
  --values prometheus-adapter-values.yaml
```

**Étape 5 : Vérifier l'installation**

```bash
kubectl get pods -n monitoring | grep prometheus-adapter
```

Sortie attendue :
```
prometheus-adapter-xxxxxxxxx-xxxxx   1/1     Running   0          2m
```

**Étape 6 : Vérifier que l'API Custom Metrics est disponible**

```bash
kubectl get apiservices | grep custom.metrics
```

Sortie :
```
v1beta1.custom.metrics.k8s.io   monitoring/prometheus-adapter   True   5m
```

Le statut doit être `True`.

## Exposer des métriques depuis votre application

Pour que tout cela fonctionne, votre application doit exposer des métriques. Voici comment faire selon le langage.

### Exemple en Python (Flask)

**Installer la bibliothèque Prometheus :**

```bash
pip install prometheus-client
```

**Code de l'application :**

```python
from flask import Flask, request
from prometheus_client import Counter, Histogram, generate_latest, REGISTRY
import time

app = Flask(__name__)

# Définir les métriques
request_count = Counter(
    'http_requests_total',
    'Total HTTP Requests',
    ['method', 'endpoint', 'status']
)

request_latency = Histogram(
    'http_request_duration_seconds',
    'HTTP Request Latency',
    ['method', 'endpoint']
)

@app.before_request
def before_request():
    request.start_time = time.time()

@app.after_request
def after_request(response):
    request_latency.labels(
        method=request.method,
        endpoint=request.path
    ).observe(time.time() - request.start_time)

    request_count.labels(
        method=request.method,
        endpoint=request.path,
        status=response.status_code
    ).inc()

    return response

@app.route('/')
def hello():
    return "Hello, World!"

@app.route('/api/data')
def get_data():
    time.sleep(0.1)  # Simuler un traitement
    return {"data": "example"}

@app.route('/metrics')
def metrics():
    return generate_latest(REGISTRY)

if __name__ == '__main__':
    app.run(host='0.0.0.0', port=8080)
```

**Endpoint /metrics :**

Quand vous appelez `http://votre-app:8080/metrics`, vous obtenez :

```
# HELP http_requests_total Total HTTP Requests
# TYPE http_requests_total counter
http_requests_total{method="GET",endpoint="/",status="200"} 1547
http_requests_total{method="GET",endpoint="/api/data",status="200"} 823

# HELP http_request_duration_seconds HTTP Request Latency
# TYPE http_request_duration_seconds histogram
http_request_duration_seconds_bucket{method="GET",endpoint="/api/data",le="0.005"} 0
http_request_duration_seconds_bucket{method="GET",endpoint="/api/data",le="0.01"} 0
http_request_duration_seconds_bucket{method="GET",endpoint="/api/data",le="0.1"} 620
http_request_duration_seconds_sum{method="GET",endpoint="/api/data"} 82.3
http_request_duration_seconds_count{method="GET",endpoint="/api/data"} 823
```

### Exemple en Go

```go
package main

import (
    "net/http"
    "github.com/prometheus/client_golang/prometheus"
    "github.com/prometheus/client_golang/prometheus/promhttp"
    "github.com/prometheus/client_golang/prometheus/promauto"
)

var (
    httpRequestsTotal = promauto.NewCounterVec(
        prometheus.CounterOpts{
            Name: "http_requests_total",
            Help: "Total HTTP requests",
        },
        []string{"method", "endpoint", "status"},
    )

    httpRequestDuration = promauto.NewHistogramVec(
        prometheus.HistogramOpts{
            Name: "http_request_duration_seconds",
            Help: "HTTP request latency",
        },
        []string{"method", "endpoint"},
    )
)

func helloHandler(w http.ResponseWriter, r *http.Request) {
    timer := prometheus.NewTimer(httpRequestDuration.WithLabelValues(r.Method, r.URL.Path))
    defer timer.ObserveDuration()

    w.Write([]byte("Hello, World!"))
    httpRequestsTotal.WithLabelValues(r.Method, r.URL.Path, "200").Inc()
}

func main() {
    http.HandleFunc("/", helloHandler)
    http.Handle("/metrics", promhttp.Handler())
    http.ListenAndServe(":8080", nil)
}
```

### Exemple en Node.js

```javascript
const express = require('express');
const promClient = require('prom-client');

const app = express();
const register = new promClient.Registry();

// Définir les métriques
const httpRequestsTotal = new promClient.Counter({
    name: 'http_requests_total',
    help: 'Total HTTP requests',
    labelNames: ['method', 'endpoint', 'status'],
    registers: [register]
});

const httpRequestDuration = new promClient.Histogram({
    name: 'http_request_duration_seconds',
    help: 'HTTP request latency',
    labelNames: ['method', 'endpoint'],
    registers: [register]
});

// Middleware pour capturer les métriques
app.use((req, res, next) => {
    const start = Date.now();

    res.on('finish', () => {
        const duration = (Date.now() - start) / 1000;
        httpRequestDuration.labels(req.method, req.path).observe(duration);
        httpRequestsTotal.labels(req.method, req.path, res.statusCode).inc();
    });

    next();
});

app.get('/', (req, res) => {
    res.send('Hello, World!');
});

app.get('/metrics', async (req, res) => {
    res.set('Content-Type', register.contentType);
    res.end(await register.metrics());
});

app.listen(8080, () => {
    console.log('Server running on port 8080');
});
```

## Configurer Prometheus pour scraper vos métriques

Une fois que votre application expose des métriques, vous devez dire à Prometheus de les collecter.

### Méthode 1 : Annotations sur le pod

La façon la plus simple avec le Prometheus installé par MicroK8s :

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mon-api
spec:
  replicas: 3
  selector:
    matchLabels:
      app: mon-api
  template:
    metadata:
      labels:
        app: mon-api
      annotations:
        prometheus.io/scrape: "true"    # Activer le scraping
        prometheus.io/port: "8080"       # Port où /metrics est exposé
        prometheus.io/path: "/metrics"   # Chemin de l'endpoint
    spec:
      containers:
      - name: api
        image: mon-api:latest
        ports:
        - containerPort: 8080
```

Avec ces annotations, Prometheus découvre automatiquement vos pods et collecte les métriques.

### Méthode 2 : ServiceMonitor (si vous utilisez Prometheus Operator)

```yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: mon-api-monitor
  labels:
    app: mon-api
spec:
  selector:
    matchLabels:
      app: mon-api
  endpoints:
  - port: http
    path: /metrics
    interval: 30s
```

### Vérifier que Prometheus collecte les métriques

**1. Accéder à l'interface Prometheus :**

```bash
kubectl port-forward -n monitoring svc/prometheus-k8s 9090:9090
```

Puis ouvrez : http://localhost:9090

**2. Tester une requête :**

Dans l'interface Prometheus, testez la requête :

```
http_requests_total
```

Si vous voyez des résultats, c'est que Prometheus collecte bien vos métriques !

## Configurer le HPA avec Custom Metrics

Maintenant que tout est en place, créons un HPA qui utilise des custom metrics.

### Exemple 1 : Scaler basé sur les requêtes HTTP par seconde

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: mon-api-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: mon-api
  minReplicas: 2
  maxReplicas: 20
  metrics:
  - type: Pods
    pods:
      metric:
        name: http_requests_per_second
      target:
        type: AverageValue
        averageValue: "1000"    # 1000 requêtes/sec par pod
```

**Explication :**

- **type: Pods** : Métrique par pod (pas par ressource comme CPU)
- **metric.name** : Nom de la métrique custom (défini dans les règles du Prometheus Adapter)
- **averageValue: "1000"** : Chaque pod doit gérer en moyenne 1000 requêtes/sec
- Si un pod reçoit 1500 req/sec → scale up
- Si un pod reçoit 500 req/sec → scale down

**Comment ça marche :**

```
État actuel :
- 2 pods
- Total : 4000 requêtes/sec
- Par pod : 2000 requêtes/sec

Calcul du HPA :
- Cible : 1000 requêtes/sec par pod
- Actuel : 2000 requêtes/sec par pod
- Pods nécessaires : (2000 / 1000) × 2 = 4 pods

Résultat : HPA scale à 4 pods
```

### Exemple 2 : Scaler basé sur la latence

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: mon-api-hpa-latency
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: mon-api
  minReplicas: 2
  maxReplicas: 20
  metrics:
  - type: Pods
    pods:
      metric:
        name: http_request_latency_seconds
      target:
        type: AverageValue
        averageValue: "0.5"    # Latence max : 500ms
```

Si la latence moyenne dépasse 500ms, le HPA ajoute des pods pour améliorer les performances.

### Exemple 3 : Combiner CPU et custom metrics

Vous pouvez utiliser plusieurs métriques. Le HPA prendra le plus grand nombre de répliques calculé :

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: mon-api-hpa-combined
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: mon-api
  minReplicas: 2
  maxReplicas: 20
  metrics:
  # Métrique CPU classique
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70

  # Custom metric : requêtes/sec
  - type: Pods
    pods:
      metric:
        name: http_requests_per_second
      target:
        type: AverageValue
        averageValue: "1000"

  # Custom metric : latence
  - type: Pods
    pods:
      metric:
        name: http_request_latency_seconds
      target:
        type: AverageValue
        averageValue: "0.5"
```

**Comportement :**

Le HPA calcule le nombre de répliques nécessaires pour chaque métrique :
- CPU recommande : 5 répliques
- Requêtes/sec recommande : 8 répliques
- Latence recommande : 6 répliques

**Résultat :** Le HPA scale à **8 répliques** (le maximum).

## Vérifier le fonctionnement

### Voir les custom metrics disponibles

```bash
kubectl get --raw /apis/custom.metrics.k8s.io/v1beta1 | jq .
```

Vous devriez voir vos métriques custom listées.

### Consulter une métrique spécifique

```bash
kubectl get --raw "/apis/custom.metrics.k8s.io/v1beta1/namespaces/default/pods/*/http_requests_per_second" | jq .
```

Sortie exemple :
```json
{
  "kind": "MetricValueList",
  "apiVersion": "custom.metrics.k8s.io/v1beta1",
  "metadata": {},
  "items": [
    {
      "describedObject": {
        "kind": "Pod",
        "namespace": "default",
        "name": "mon-api-xxxxx-abcd",
        "apiVersion": "/v1"
      },
      "metricName": "http_requests_per_second",
      "timestamp": "2025-10-25T10:30:00Z",
      "value": "1250"
    }
  ]
}
```

### Voir l'état du HPA

```bash
kubectl get hpa mon-api-hpa
```

Sortie :
```
NAME           REFERENCE          TARGETS      MINPODS   MAXPODS   REPLICAS   AGE
mon-api-hpa    Deployment/mon-api 1250/1000    2         20        3          5m
```

**TARGETS** montre `1250/1000` :
- **1250** : Valeur actuelle (requêtes/sec par pod)
- **1000** : Cible configurée

Le HPA a scalé à 3 répliques pour ramener la charge par pod vers 1000.

### Détails du HPA

```bash
kubectl describe hpa mon-api-hpa
```

Sortie :
```
Name:                     mon-api-hpa
Namespace:                default
Reference:                Deployment/mon-api
Metrics:                  ( current / target )
  "http_requests_per_second" on pods:  1250 / 1k
Min replicas:             2
Max replicas:             20
Deployment pods:          3 current / 3 desired
Conditions:
  Type            Status  Reason              Message
  ----            ------  ------              -------
  AbleToScale     True    ReadyForNewScale    recommended size matches current size
  ScalingActive   True    ValidMetricFound    the HPA was able to successfully calculate
  ScalingLimited  False   DesiredWithinRange  the desired count is within the acceptable range
Events:
  Type    Reason             Age   From                       Message
  ----    ------             ----  ----                       -------
  Normal  SuccessfulRescale  2m    horizontal-pod-autoscaler  New size: 3; reason: pods metric http_requests_per_second above target
```

## Cas d'usage pratiques

### 1. API web avec charge variable

**Problème :** Scaler basé uniquement sur le CPU ne reflète pas la vraie charge.

**Solution :** Scaler basé sur les requêtes/sec et la latence.

```yaml
metrics:
- type: Pods
  pods:
    metric:
      name: http_requests_per_second
    target:
      type: AverageValue
      averageValue: "1000"
```

### 2. Worker de file d'attente (RabbitMQ, Kafka)

**Problème :** Besoin de scaler basé sur le nombre de messages en attente, pas le CPU.

**Métrique à exposer :**
```
queue_messages_pending{queue="orders"} 150
```

**HPA :**
```yaml
metrics:
- type: Pods
  pods:
    metric:
      name: queue_messages_pending
    target:
      type: AverageValue
      averageValue: "10"    # Max 10 messages par worker
```

Si la file a 150 messages et 5 workers :
- Par worker : 30 messages
- Cible : 10 messages par worker
- HPA scale à : 150 / 10 = 15 workers

### 3. Connexions WebSocket

**Problème :** Nombre de connexions simultanées limitées par pod.

**Métrique à exposer :**
```
websocket_connections_active 450
```

**HPA :**
```yaml
metrics:
- type: Pods
  pods:
    metric:
      name: websocket_connections_active
    target:
      type: AverageValue
      averageValue: "100"    # Max 100 connexions par pod
```

### 4. Traitement batch basé sur le backlog

**Problème :** Jobs en attente qui s'accumulent.

**Métrique à exposer :**
```
batch_jobs_pending 5000
```

**HPA :**
```yaml
metrics:
- type: Pods
  pods:
    metric:
      name: batch_jobs_pending
    target:
      type: AverageValue
      averageValue: "50"    # Chaque worker traite 50 jobs
```

### 5. Gaming : Joueurs connectés

**Problème :** Serveurs de jeu qui doivent scaler selon le nombre de joueurs.

**Métrique à exposer :**
```
game_players_connected{server="lobby"} 250
```

**HPA :**
```yaml
metrics:
- type: Pods
  pods:
    metric:
      name: game_players_connected
    target:
      type: AverageValue
      averageValue: "50"    # Max 50 joueurs par instance
```

## Règles du Prometheus Adapter expliquées

Les règles dans la configuration du Prometheus Adapter transforment les métriques Prometheus en métriques Kubernetes.

### Anatomie d'une règle

```yaml
- seriesQuery: 'http_requests_total{namespace!="",pod!=""}'
  resources:
    overrides:
      namespace: {resource: "namespace"}
      pod: {resource: "pod"}
  name:
    matches: "^(.*)_total"
    as: "${1}_per_second"
  metricsQuery: 'sum(rate(<<.Series>>{<<.LabelMatchers>>}[2m])) by (<<.GroupBy>>)'
```

**seriesQuery :**
- Sélectionne quelles métriques Prometheus utiliser
- Ici : `http_requests_total` avec les labels `namespace` et `pod`

**resources.overrides :**
- Mappe les labels Prometheus aux ressources Kubernetes
- `namespace` → ressource Kubernetes `namespace`
- `pod` → ressource Kubernetes `pod`

**name :**
- Transforme le nom de la métrique
- `matches: "^(.*)_total"` : Capture tout avant `_total`
- `as: "${1}_per_second"` : Renomme en `_per_second`
- Exemple : `http_requests_total` → `http_requests_per_second`

**metricsQuery :**
- Requête PromQL pour calculer la valeur finale
- `rate(...[2m])` : Calcule le taux par seconde sur 2 minutes
- `sum(...) by (...)` : Agrège par pod/namespace

### Exemple de transformation

**Métrique Prometheus brute :**
```
http_requests_total{namespace="default",pod="mon-api-xxxxx",method="GET",status="200"} 15470
```

**Après transformation (rate sur 2 minutes) :**
```
http_requests_per_second{namespace="default",pod="mon-api-xxxxx"} 128.5
```

Cette métrique transformée est maintenant disponible pour le HPA !

### Règle pour la latence

```yaml
- seriesQuery: 'http_request_duration_seconds_sum{namespace!="",pod!=""}'
  resources:
    overrides:
      namespace: {resource: "namespace"}
      pod: {resource: "pod"}
  name:
    as: "http_request_latency_seconds"
  metricsQuery: 'sum(rate(<<.Series>>{<<.LabelMatchers>>}[2m])) by (<<.GroupBy>>) / sum(rate(http_request_duration_seconds_count{<<.LabelMatchers>>}[2m])) by (<<.GroupBy>>)'
```

**Calcul de la latence moyenne :**
```
Latence = Somme des durées / Nombre de requêtes
        = rate(http_request_duration_seconds_sum) / rate(http_request_duration_seconds_count)
```

C'est un calcul PromQL standard pour obtenir la latence moyenne.

## Types de métriques custom

Il existe trois types de métriques que vous pouvez utiliser avec le HPA :

### 1. Pods metrics

Métriques **par pod** (ce que nous avons vu principalement) :

```yaml
- type: Pods
  pods:
    metric:
      name: http_requests_per_second
    target:
      type: AverageValue
      averageValue: "1000"
```

**Usage :** Quand la métrique est directement liée à chaque pod.

### 2. Object metrics

Métriques liées à un **objet Kubernetes spécifique** (Ingress, Service, etc.) :

```yaml
- type: Object
  object:
    metric:
      name: requests_per_second
    describedObject:
      apiVersion: networking.k8s.io/v1
      kind: Ingress
      name: mon-ingress
    target:
      type: Value
      value: "10000"    # 10k requêtes/sec au total (pas par pod)
```

**Usage :** Quand la métrique concerne l'entrée/sortie globale, pas individuellement par pod.

### 3. External metrics

Métriques provenant de sources **externes** à Kubernetes (AWS CloudWatch, Datadog, etc.) :

```yaml
- type: External
  external:
    metric:
      name: sqs_queue_length
      selector:
        matchLabels:
          queue: "orders"
    target:
      type: AverageValue
      averageValue: "30"
```

**Usage :** Scaler basé sur des métriques cloud (longueur file SQS, métriques RDS, etc.).

## Debugging et dépannage

### Problème 1 : HPA affiche "unknown" pour custom metric

**Symptôme :**

```bash
kubectl get hpa
NAME      REFERENCE        TARGETS         MINPODS   MAXPODS   REPLICAS
my-hpa    Deployment/app   <unknown>/1000  2         10        2
```

**Causes possibles :**

**1. Prometheus Adapter pas installé ou pas fonctionnel**

Vérifier :
```bash
kubectl get pods -n monitoring | grep prometheus-adapter
```

**2. Métrique pas exposée par l'application**

Vérifier que `/metrics` fonctionne :
```bash
kubectl port-forward pod/mon-api-xxxxx 8080:8080
curl http://localhost:8080/metrics | grep http_requests
```

**3. Prometheus ne collecte pas la métrique**

Dans l'interface Prometheus (port-forward 9090), tester :
```
http_requests_total
```

Si aucun résultat → Prometheus ne collecte pas. Vérifier les annotations du pod.

**4. Règle du Prometheus Adapter incorrecte**

Consulter les logs :
```bash
kubectl logs -n monitoring deployment/prometheus-adapter
```

Cherchez des erreurs de parsing des règles.

**5. API Custom Metrics pas enregistrée**

Vérifier :
```bash
kubectl get apiservices | grep custom.metrics
```

Doit afficher `True`.

### Problème 2 : Métrique disponible mais HPA ne scale pas

**Symptôme :**

La métrique a une valeur, mais le HPA reste à 2 répliques alors qu'il devrait scaler.

**Causes possibles :**

**1. Valeur en dessous du seuil**

Vérifier les valeurs réelles :
```bash
kubectl get --raw "/apis/custom.metrics.k8s.io/v1beta1/namespaces/default/pods/*/http_requests_per_second" | jq .
```

Si la valeur est inférieure à la cible, pas de scale up normal.

**2. Cooldown period**

Le HPA attend un certain temps avant de scaler :
- **Scale up** : 3 minutes par défaut
- **Scale down** : 5 minutes par défaut

**3. maxReplicas atteint**

Vérifier :
```bash
kubectl get hpa my-hpa
```

Si `REPLICAS` = `MAXPODS`, vous avez atteint la limite.

### Problème 3 : Scaling instable (flapping)

**Symptôme :**

Le HPA scale up puis down puis up constamment (oscillations).

**Causes :**

- Seuil trop proche de la valeur actuelle
- Métriques trop volatiles

**Solutions :**

**1. Augmenter la fenêtre de calcul dans les règles :**

```yaml
metricsQuery: 'sum(rate(<<.Series>>{<<.LabelMatchers>>}[5m])) by (<<.GroupBy>>)'
```

Changez `[2m]` en `[5m]` pour lisser les variations.

**2. Configurer le comportement du HPA :**

```yaml
behavior:
  scaleDown:
    stabilizationWindowSeconds: 300    # 5 minutes
    policies:
    - type: Percent
      value: 50
      periodSeconds: 60
```

**3. Augmenter la marge du seuil :**

Au lieu de `averageValue: "1000"`, utilisez `averageValue: "800"` pour avoir plus de marge.

## Métriques custom avancées

### Métriques business

Vous pouvez exposer n'importe quelle métrique métier :

```python
orders_pending = Gauge('orders_pending', 'Pending orders waiting for processing')
users_active = Gauge('users_active', 'Currently active users')
revenue_per_minute = Counter('revenue_per_minute', 'Revenue per minute')

# Dans votre code
orders_pending.set(len(pending_orders))
users_active.set(count_active_users())
revenue_per_minute.inc(order.amount)
```

Puis scaler basé sur ces métriques !

### Métriques composées

Vous pouvez créer des métriques calculées dans Prometheus Adapter :

**Exemple : Ratio erreurs/succès**

```yaml
- seriesQuery: 'http_requests_total'
  resources:
    overrides:
      namespace: {resource: "namespace"}
      pod: {resource: "pod"}
  name:
    as: "http_error_rate"
  metricsQuery: 'sum(rate(http_requests_total{status=~"5.."}[2m])) by (<<.GroupBy>>) / sum(rate(http_requests_total[2m])) by (<<.GroupBy>>)'
```

Cela calcule le pourcentage de requêtes en erreur (5xx).

## Limitations des custom metrics

### 1. Complexité accrue

Configuration plus complexe que les métriques CPU/mémoire :
- Besoin de Prometheus
- Besoin de Prometheus Adapter
- Configuration des règles
- Instrumentation de l'application

### 2. Délai de réaction

Les custom metrics ont un délai :
- Application expose la métrique (temps réel)
- Prometheus collecte toutes les 30-60s
- Prometheus Adapter interroge Prometheus
- HPA interroge l'Adapter toutes les 15s
- **Délai total : 1-2 minutes**

Pour des réactions instantanées, ce n'est pas idéal.

### 3. Debugging plus difficile

Plus de composants = plus de points de défaillance :
- Application ne génère pas la métrique ?
- Prometheus ne la collecte pas ?
- Règle de l'Adapter incorrecte ?
- Nom de métrique incorrect dans le HPA ?

### 4. Overhead

Plus de ressources consommées :
- Prometheus stocke toutes les métriques
- Prometheus Adapter effectue des calculs
- Plus de trafic réseau

## Bonnes pratiques

### 1. Commencer simple

Démarrez avec des métriques CPU/mémoire, puis ajoutez progressivement des custom metrics :

```yaml
# Phase 1 : CPU uniquement
metrics:
- type: Resource
  resource:
    name: cpu
    target:
      type: Utilization
      averageUtilization: 70

# Phase 2 : Ajouter requêtes/sec
metrics:
- type: Resource
  resource:
    name: cpu
    target:
      type: Utilization
      averageUtilization: 70
- type: Pods
  pods:
    metric:
      name: http_requests_per_second
    target:
      type: AverageValue
      averageValue: "1000"
```

### 2. Choisir les bonnes métriques

Les meilleures métriques pour l'autoscaling sont :
- ✅ **Proportionnelles à la charge** (requêtes/sec, messages en file)
- ✅ **Faciles à interpréter** (valeurs claires)
- ✅ **Stables** (pas trop volatiles)

Évitez :
- ❌ Métriques binaires (0 ou 1)
- ❌ Métriques trop volatiles (fluctuations constantes)
- ❌ Métriques sans lien direct avec la charge

### 3. Tester en environnement de dev

Avant la production :

```bash
# Générer de la charge
hey -z 5m -c 100 http://mon-api.example.com

# Observer le HPA en temps réel
watch -n 5 kubectl get hpa
```

Ajustez les seuils selon les résultats observés.

### 4. Surveiller le HPA

Configurez des alertes pour :
- HPA qui n'arrive pas à obtenir les métriques
- HPA qui atteint constamment maxReplicas
- Oscillations (flapping)

### 5. Documenter vos métriques

```yaml
metadata:
  annotations:
    metrics-documentation: |
      http_requests_per_second: Nombre de requêtes HTTP par seconde par pod.
      Cible: 1000 req/sec par pod (latence < 100ms observée à cette charge).
      Dernière calibration: 2025-10-25
```

### 6. Utiliser des noms de métriques cohérents

Convention Prometheus :
- `_total` pour les counters (cumulatifs)
- `_seconds` pour les durées
- `_bytes` pour les tailles
- Pas d'unité pour les gauges

### 7. Attention au coût

Les custom metrics peuvent générer beaucoup de données dans Prometheus. Surveillez :
- La taille de stockage Prometheus
- Le nombre de time series
- L'utilisation CPU/RAM de Prometheus

## Résumé

Les **custom metrics** permettent un autoscaling intelligent basé sur des métriques métier plutôt que simplement CPU/mémoire.

**Points clés :**

1. Nécessite trois composants : Application instrumentée, Prometheus, Prometheus Adapter
2. Permet de scaler sur n'importe quelle métrique (requêtes/sec, latence, files d'attente, etc.)
3. Plus pertinent que CPU/mémoire pour beaucoup d'applications
4. Configuration plus complexe mais beaucoup plus puissant
5. Délai de réaction de 1-2 minutes (pas instantané)
6. Trois types : Pods metrics, Object metrics, External metrics

**Quand utiliser les custom metrics ?**

✅ API web où la charge n'est pas reflétée par le CPU
✅ Workers de files d'attente (RabbitMQ, Kafka, SQS)
✅ Applications WebSocket (connexions simultanées)
✅ Métriques business (commandes en attente, utilisateurs actifs)
✅ Latence comme critère de qualité de service

❌ Applications simples où CPU/mémoire suffisent
❌ Si vous n'avez pas les ressources pour maintenir Prometheus
❌ Besoin de réactivité instantanée (< 30 secondes)

**Progression recommandée :**

1. **Scaling manuel** → Comprendre les bases
2. **HPA avec CPU** → Autoscaling basique
3. **Metrics Server** → Fondation technique
4. **Custom metrics** → Autoscaling intelligent

Les custom metrics représentent le **niveau expert** de l'autoscaling Kubernetes. Elles offrent une précision et une pertinence inégalées, mais nécessitent plus d'investissement en configuration et maintenance.

---

**Prochaine section :** 19.7 Tests de charge - Valider que votre autoscaling fonctionne correctement !

⏭️ [Tests de charge](/19-scaling-et-autoscaling/07-tests-de-charge.md)
