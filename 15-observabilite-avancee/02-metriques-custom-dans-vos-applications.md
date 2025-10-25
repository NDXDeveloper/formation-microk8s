🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 15.2 Métriques Custom dans Vos Applications

## Introduction

Dans la section précédente, nous avons découvert que les métriques sont l'un des trois piliers de l'observabilité. Prometheus collecte automatiquement de nombreuses métriques système (CPU, mémoire, réseau), mais qu'en est-il de **vos propres applications** ?

C'est là qu'interviennent les **métriques custom** : des métriques que vous définissez vous-même pour mesurer ce qui compte vraiment dans votre logique métier.

## Pourquoi Créer des Métriques Custom ?

### Les Métriques Système ne Suffisent Pas

Imaginons que vous développez une application de e-commerce. Prometheus vous dit que :
- ✅ Le CPU est à 30%
- ✅ La mémoire est normale
- ✅ Le pod est en bonne santé

**Mais** vous ne savez pas :
- ❌ Combien de commandes ont été passées dans la dernière heure
- ❌ Combien de paiements ont échoué
- ❌ Quel est le temps moyen de traitement d'une commande
- ❌ Combien d'utilisateurs sont connectés actuellement

### Ce Qui Compte pour Votre Business

Les métriques custom vous permettent de mesurer ce qui est **spécifique à votre application** :

**Exemples de métriques métier** :
- Nombre d'utilisateurs actifs
- Montant total des ventes
- Nombre d'erreurs métier (paiement refusé, stock insuffisant)
- Temps de traitement d'une opération spécifique
- Taille de la file d'attente de jobs
- Nombre d'emails envoyés
- Nombre de photos uploadées

Ces métriques ont un **sens métier** que les métriques système ne peuvent pas capturer.

## Les Types de Métriques

Prometheus définit quatre types de métriques. Choisir le bon type est essentiel pour exploiter correctement vos données.

### 1. Counter (Compteur)

**Définition** : Une valeur qui ne fait que **monter** (ou se réinitialiser à zéro).

**Quand l'utiliser** : Pour compter des événements qui se produisent dans le temps.

**Exemples** :
```
http_requests_total          → Nombre total de requêtes HTTP
orders_created_total         → Nombre total de commandes créées
emails_sent_total            → Nombre total d'emails envoyés
errors_total                 → Nombre total d'erreurs
payment_failures_total       → Nombre total d'échecs de paiement
```

**Caractéristiques** :
- ✅ Ne décroît jamais (sauf au redémarrage de l'application)
- ✅ Idéal pour calculer des taux (requêtes/seconde)
- ✅ Léger et performant

**Exemple de valeur** :
```
Temps       Valeur
10:00       1523
10:01       1687
10:02       1842
```

**Requête PromQL typique** :
```promql
# Taux de requêtes par seconde sur 5 minutes
rate(http_requests_total[5m])
```

### 2. Gauge (Jauge)

**Définition** : Une valeur qui peut **monter et descendre**.

**Quand l'utiliser** : Pour mesurer une valeur instantanée qui fluctue.

**Exemples** :
```
active_users                 → Nombre d'utilisateurs connectés actuellement
queue_size                   → Taille actuelle de la file d'attente
memory_usage_bytes           → Mémoire utilisée actuellement
temperature_celsius          → Température actuelle
active_connections           → Nombre de connexions actives
items_in_cart                → Articles dans le panier (moyenne)
```

**Caractéristiques** :
- ✅ Peut augmenter ou diminuer
- ✅ Représente un état actuel
- ✅ Idéal pour les dashboards "en temps réel"

**Exemple de valeur** :
```
Temps       Valeur
10:00       42 (utilisateurs connectés)
10:01       38
10:02       45
```

**Requête PromQL typique** :
```promql
# Valeur actuelle
active_users

# Moyenne sur 5 minutes
avg_over_time(active_users[5m])
```

### 3. Histogram (Histogramme)

**Définition** : Mesure la **distribution** d'une valeur et organise les observations en buckets.

**Quand l'utiliser** : Pour mesurer des durées, des tailles, et calculer des percentiles.

**Exemples** :
```
http_request_duration_seconds    → Durée des requêtes HTTP
order_processing_time_seconds    → Temps de traitement des commandes
file_upload_size_bytes           → Taille des fichiers uploadés
database_query_duration_seconds  → Durée des requêtes DB
```

**Caractéristiques** :
- ✅ Calcule automatiquement des percentiles (p50, p95, p99)
- ✅ Compte le nombre d'observations
- ✅ Calcule la somme totale
- ⚠️ Plus coûteux en mémoire (multiple buckets)

**Ce qui est collecté** :
```
http_request_duration_seconds_bucket{le="0.1"}    34
http_request_duration_seconds_bucket{le="0.5"}    89
http_request_duration_seconds_bucket{le="1.0"}    142
http_request_duration_seconds_bucket{le="+Inf"}   150
http_request_duration_seconds_sum                 87.3
http_request_duration_seconds_count               150
```

**Requête PromQL typique** :
```promql
# 95e percentile du temps de réponse
histogram_quantile(0.95, rate(http_request_duration_seconds_bucket[5m]))
```

### 4. Summary (Résumé)

**Définition** : Similaire à l'histogram, mais calcule les percentiles **côté client** (dans votre application).

**Quand l'utiliser** : Quand vous voulez des percentiles précis sur une fenêtre glissante.

**Exemples** : Mêmes cas d'usage que les histograms.

**Différences avec Histogram** :
- ✅ Percentiles plus précis
- ❌ Ne peut pas être agrégé entre plusieurs instances
- ❌ Plus coûteux en CPU côté application
- ❌ Moins flexible dans Prometheus

**Recommandation** : Pour débuter, **préférez les histograms** aux summaries.

### Tableau Récapitulatif

| Type | Peut monter | Peut descendre | Cas d'usage principal | Exemple |
|------|-------------|----------------|----------------------|---------|
| **Counter** | ✅ | ❌ | Compter des événements | Nombre de requêtes |
| **Gauge** | ✅ | ✅ | Mesurer un état actuel | Utilisateurs connectés |
| **Histogram** | ✅ | ❌ | Distribution et percentiles | Temps de réponse |
| **Summary** | ✅ | ❌ | Percentiles précis | Latence de l'API |

## Instrumenter Votre Application

### Concept d'Instrumentation

**Instrumenter** une application signifie ajouter du code qui expose des métriques pour Prometheus. Voici le processus général :

1. **Importer** une bibliothèque client Prometheus
2. **Créer** des objets métriques (counter, gauge, etc.)
3. **Incrémenter/Modifier** ces métriques dans votre code
4. **Exposer** un endpoint HTTP pour que Prometheus puisse récupérer les métriques

### Architecture de Collecte

```
┌────────────────────────────────────┐
│   Votre Application (Pod)          │
│                                    │
│  ┌──────────────────────────────┐  │
│  │  Code métier                 │  │
│  │  app.order() ────► counter++ │  │
│  │  app.process() ──► gauge.set │  │
│  └──────────────────────────────┘  │
│                                    │
│  ┌──────────────────────────────┐  │
│  │  Bibliothèque Prometheus     │  │
│  │  Stocke les métriques        │  │
│  └──────────────────────────────┘  │
│                                    │
│  ┌──────────────────────────────┐  │
│  │  Endpoint /metrics           │  │
│  │  Port 8080                   │  │
│  └──────────────────────────────┘  │
└────────────────────────────────────┘
              ↑
              │ HTTP GET /metrics
              │ (scrape toutes les 15s)
              │
      ┌───────────────┐
      │  Prometheus   │
      └───────────────┘
```

## Exemples d'Implémentation

### Python avec Flask

Voici un exemple d'application Flask instrumentée avec Prometheus :

```python
from flask import Flask
from prometheus_client import Counter, Gauge, Histogram, generate_latest

app = Flask(__name__)

# Définition des métriques
http_requests_total = Counter(
    'http_requests_total',
    'Total HTTP requests',
    ['method', 'endpoint', 'status']
)

active_users = Gauge(
    'active_users',
    'Number of currently active users'
)

request_duration = Histogram(
    'http_request_duration_seconds',
    'HTTP request duration in seconds',
    ['endpoint']
)

orders_created = Counter(
    'orders_created_total',
    'Total number of orders created'
)

# Routes de votre application
@app.route('/api/order', methods=['POST'])
def create_order():
    # Mesurer le temps de traitement
    with request_duration.labels(endpoint='/api/order').time():
        # Votre logique métier
        process_order()

        # Incrémenter le compteur
        orders_created.inc()

    # Incrémenter le compteur de requêtes
    http_requests_total.labels(
        method='POST',
        endpoint='/api/order',
        status='200'
    ).inc()

    return {'status': 'success'}

@app.route('/api/users/login', methods=['POST'])
def login():
    # Logique de connexion
    user_logged_in()

    # Augmenter le nombre d'utilisateurs actifs
    active_users.inc()

    return {'status': 'logged_in'}

@app.route('/api/users/logout', methods=['POST'])
def logout():
    # Logique de déconnexion
    user_logged_out()

    # Diminuer le nombre d'utilisateurs actifs
    active_users.dec()

    return {'status': 'logged_out'}

# Endpoint pour Prometheus
@app.route('/metrics')
def metrics():
    return generate_latest()

if __name__ == '__main__':
    app.run(host='0.0.0.0', port=8080)
```

**Explication** :
- `Counter.inc()` : Incrémente le compteur de 1
- `Gauge.inc()` / `Gauge.dec()` : Augmente/diminue la jauge
- `Gauge.set(42)` : Définit la jauge à une valeur spécifique
- `Histogram.time()` : Mesure automatiquement la durée d'un bloc de code
- `generate_latest()` : Génère le format texte pour Prometheus

### Node.js avec Express

```javascript
const express = require('express');
const client = require('prom-client');

const app = express();

// Créer un registre pour stocker les métriques
const register = new client.Registry();

// Métriques custom
const httpRequestsTotal = new client.Counter({
    name: 'http_requests_total',
    help: 'Total HTTP requests',
    labelNames: ['method', 'endpoint', 'status'],
    registers: [register]
});

const activeUsers = new client.Gauge({
    name: 'active_users',
    help: 'Number of currently active users',
    registers: [register]
});

const requestDuration = new client.Histogram({
    name: 'http_request_duration_seconds',
    help: 'HTTP request duration in seconds',
    labelNames: ['endpoint'],
    buckets: [0.1, 0.5, 1, 2, 5],
    registers: [register]
});

// Route API
app.post('/api/order', async (req, res) => {
    const end = requestDuration.startTimer({ endpoint: '/api/order' });

    try {
        // Votre logique métier
        await processOrder();

        httpRequestsTotal.labels('POST', '/api/order', '200').inc();
        res.json({ status: 'success' });
    } catch (error) {
        httpRequestsTotal.labels('POST', '/api/order', '500').inc();
        res.status(500).json({ error: 'Failed' });
    } finally {
        end();
    }
});

app.post('/api/users/login', (req, res) => {
    // Logique de connexion
    activeUsers.inc();
    res.json({ status: 'logged_in' });
});

app.post('/api/users/logout', (req, res) => {
    // Logique de déconnexion
    activeUsers.dec();
    res.json({ status: 'logged_out' });
});

// Endpoint /metrics pour Prometheus
app.get('/metrics', async (req, res) => {
    res.set('Content-Type', register.contentType);
    res.end(await register.metrics());
});

app.listen(8080, () => {
    console.log('Server running on port 8080');
});
```

### Go

```go
package main

import (
    "net/http"
    "github.com/prometheus/client_golang/prometheus"
    "github.com/prometheus/client_golang/prometheus/promhttp"
)

var (
    httpRequestsTotal = prometheus.NewCounterVec(
        prometheus.CounterOpts{
            Name: "http_requests_total",
            Help: "Total HTTP requests",
        },
        []string{"method", "endpoint", "status"},
    )

    activeUsers = prometheus.NewGauge(
        prometheus.GaugeOpts{
            Name: "active_users",
            Help: "Number of currently active users",
        },
    )

    requestDuration = prometheus.NewHistogramVec(
        prometheus.HistogramOpts{
            Name:    "http_request_duration_seconds",
            Help:    "HTTP request duration in seconds",
            Buckets: prometheus.DefBuckets,
        },
        []string{"endpoint"},
    )
)

func init() {
    // Enregistrer les métriques
    prometheus.MustRegister(httpRequestsTotal)
    prometheus.MustRegister(activeUsers)
    prometheus.MustRegister(requestDuration)
}

func createOrderHandler(w http.ResponseWriter, r *http.Request) {
    timer := prometheus.NewTimer(requestDuration.WithLabelValues("/api/order"))
    defer timer.ObserveDuration()

    // Votre logique métier
    processOrder()

    httpRequestsTotal.WithLabelValues("POST", "/api/order", "200").Inc()
    w.Write([]byte("Order created"))
}

func loginHandler(w http.ResponseWriter, r *http.Request) {
    // Logique de connexion
    activeUsers.Inc()
    w.Write([]byte("Logged in"))
}

func logoutHandler(w http.ResponseWriter, r *http.Request) {
    // Logique de déconnexion
    activeUsers.Dec()
    w.Write([]byte("Logged out"))
}

func main() {
    http.HandleFunc("/api/order", createOrderHandler)
    http.HandleFunc("/api/users/login", loginHandler)
    http.HandleFunc("/api/users/logout", logoutHandler)

    // Endpoint /metrics pour Prometheus
    http.Handle("/metrics", promhttp.Handler())

    http.ListenAndServe(":8080", nil)
}
```

## Format des Métriques Exposées

Quand Prometheus fait un `GET /metrics`, votre application retourne un format texte spécifique :

```
# HELP http_requests_total Total HTTP requests
# TYPE http_requests_total counter
http_requests_total{method="GET",endpoint="/api/products",status="200"} 1523
http_requests_total{method="POST",endpoint="/api/order",status="200"} 87
http_requests_total{method="POST",endpoint="/api/order",status="500"} 3

# HELP active_users Number of currently active users
# TYPE active_users gauge
active_users 42

# HELP http_request_duration_seconds HTTP request duration in seconds
# TYPE http_request_duration_seconds histogram
http_request_duration_seconds_bucket{endpoint="/api/order",le="0.1"} 34
http_request_duration_seconds_bucket{endpoint="/api/order",le="0.5"} 89
http_request_duration_seconds_bucket{endpoint="/api/order",le="1.0"} 142
http_request_duration_seconds_bucket{endpoint="/api/order",le="+Inf"} 150
http_request_duration_seconds_sum{endpoint="/api/order"} 87.3
http_request_duration_seconds_count{endpoint="/api/order"} 150
```

**Composants** :
- `# HELP` : Description de la métrique
- `# TYPE` : Type de métrique (counter, gauge, histogram, summary)
- Nom de la métrique + labels entre `{}` + valeur

## Configurer Prometheus pour Scraper Votre Application

### 1. Déployer Votre Application dans MicroK8s

Fichier `app-deployment.yaml` :

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
  namespace: default
spec:
  replicas: 2
  selector:
    matchLabels:
      app: my-app
  template:
    metadata:
      labels:
        app: my-app
      annotations:
        # Annotations pour la découverte automatique par Prometheus
        prometheus.io/scrape: "true"
        prometheus.io/port: "8080"
        prometheus.io/path: "/metrics"
    spec:
      containers:
      - name: my-app
        image: my-registry/my-app:1.0
        ports:
        - containerPort: 8080
          name: metrics
---
apiVersion: v1
kind: Service
metadata:
  name: my-app
  namespace: default
spec:
  selector:
    app: my-app
  ports:
  - port: 8080
    targetPort: 8080
    name: metrics
```

**Points clés** :
- **Annotations** : Indiquent à Prometheus comment scraper l'application
- `prometheus.io/scrape: "true"` : Active le scraping
- `prometheus.io/port: "8080"` : Port où se trouve l'endpoint /metrics
- `prometheus.io/path: "/metrics"` : Chemin de l'endpoint

### 2. Vérifier que Prometheus Scrape Votre Application

Une fois déployé, vérifiez dans l'interface Prometheus :

1. Accédez à Prometheus : `http://<votre-cluster>:9090`
2. Allez dans **Status → Targets**
3. Vous devriez voir votre application listée
4. Status devrait être **UP**

Si le status est **DOWN** :
- Vérifiez que l'endpoint `/metrics` répond
- Vérifiez les annotations dans le déploiement
- Consultez les logs de Prometheus

### 3. Interroger Vos Métriques

Dans Prometheus, allez dans **Graph** et essayez :

```promql
# Voir toutes vos métriques custom
{__name__=~"http_requests_total|active_users|orders_created_total"}

# Taux de requêtes par seconde
rate(http_requests_total[5m])

# Utilisateurs actifs actuellement
active_users

# 95e percentile du temps de réponse
histogram_quantile(0.95, rate(http_request_duration_seconds_bucket[5m]))
```

## Bonnes Pratiques

### 1. Nommage des Métriques

**Convention de nommage** :
- Format : `<namespace>_<subsystem>_<metric_name>_<unit>`
- Utiliser des underscores `_`, pas de tirets `-`
- Terminer les counters par `_total`
- Inclure l'unité si applicable (`_seconds`, `_bytes`, `_celsius`)

**Exemples** :
```
✅ http_requests_total
✅ order_processing_duration_seconds
✅ database_connection_errors_total
✅ memory_usage_bytes

❌ httpRequests (camelCase)
❌ requests-total (tirets)
❌ order_time (pas d'unité)
```

### 2. Utilisation des Labels

Les **labels** permettent de filtrer et d'agréger les métriques :

```python
http_requests_total.labels(
    method='POST',
    endpoint='/api/order',
    status='200'
).inc()
```

**Bonnes pratiques** :
- ✅ Utiliser des labels pour les dimensions importantes (méthode, endpoint, status)
- ✅ Garder la cardinalité faible (nombre de combinaisons de labels)
- ❌ Ne JAMAIS mettre d'IDs uniques dans les labels (user_id, request_id)
- ❌ Éviter trop de labels (max 5-10 par métrique)

**Pourquoi éviter les hautes cardinalités ?**

```python
# ❌ MAUVAIS : Créera des millions de séries temporelles
requests.labels(user_id='123456').inc()
requests.labels(user_id='123457').inc()
# ... des millions d'utilisateurs = des millions de métriques !

# ✅ BON : Cardinalité contrôlée
requests.labels(user_type='premium').inc()
requests.labels(user_type='free').inc()
# Seulement 2 séries temporelles
```

### 3. Métriques Métier vs Techniques

**Métriques techniques** (déjà fournies par Prometheus/système) :
- CPU, mémoire, réseau
- Latence HTTP générique
- Nombre de pods

**Métriques métier** (à ajouter vous-même) :
- Nombre de commandes passées
- Montant des transactions
- Nombre d'utilisateurs actifs
- Taux de conversion
- Inventaire produits

**Focus** : Concentrez-vous sur les **métriques métier** qui ont du sens pour votre application.

### 4. Éviter la Sur-Instrumentation

**Ne mesurez pas tout** :
- ✅ Mesurez ce qui est important pour le business
- ✅ Mesurez ce qui peut casser ou ralentir
- ❌ Ne mesurez pas chaque ligne de code
- ❌ Ne créez pas 100 métriques "au cas où"

**Règle** : Si vous ne savez pas pourquoi vous créez une métrique, ne la créez probablement pas.

### 5. Performance

Les métriques ont un **coût** :
- Mémoire pour stocker les valeurs
- CPU pour incrémenter/calculer
- Bande passante réseau (scraping)

**Optimisations** :
- Utilisez des counters plutôt que des gauges quand possible (plus légers)
- Limitez le nombre de buckets dans les histograms
- Évitez la haute cardinalité des labels
- Ne mesurez pas dans des boucles critiques

### 6. Documentation

**Documentez vos métriques** :
```python
orders_created = Counter(
    'orders_created_total',
    'Total number of orders created. Incremented when an order is successfully validated and persisted to the database.',
    ['payment_method', 'country']
)
```

La description apparaîtra dans Prometheus avec `# HELP`.

## Exemples de Métriques Métier par Type d'Application

### E-commerce

```python
# Counters
orders_total = Counter('orders_total', 'Total orders')
revenue_total = Counter('revenue_total', 'Total revenue in euros')
cart_abandoned_total = Counter('cart_abandoned_total', 'Abandoned carts')
payment_failures_total = Counter('payment_failures_total', 'Payment failures')

# Gauges
active_carts = Gauge('active_carts', 'Number of active shopping carts')
inventory_level = Gauge('inventory_level', 'Product inventory', ['product_id'])
wishlist_items = Gauge('wishlist_items', 'Items in wishlists')

# Histograms
order_value_euros = Histogram('order_value_euros', 'Order value distribution')
checkout_duration_seconds = Histogram('checkout_duration_seconds', 'Time to complete checkout')
```

### API / Microservice

```python
# Counters
api_requests_total = Counter('api_requests_total', 'API requests', ['method', 'endpoint', 'status'])
api_errors_total = Counter('api_errors_total', 'API errors', ['type'])
cache_hits_total = Counter('cache_hits_total', 'Cache hits')
cache_misses_total = Counter('cache_misses_total', 'Cache misses')

# Gauges
active_connections = Gauge('active_connections', 'Active DB connections')
queue_depth = Gauge('queue_depth', 'Message queue depth')

# Histograms
api_latency_seconds = Histogram('api_latency_seconds', 'API latency', ['endpoint'])
db_query_duration_seconds = Histogram('db_query_duration_seconds', 'DB query time')
```

### Application de Traitement de Données

```python
# Counters
jobs_processed_total = Counter('jobs_processed_total', 'Jobs processed')
jobs_failed_total = Counter('jobs_failed_total', 'Jobs failed')
records_processed_total = Counter('records_processed_total', 'Records processed')
data_errors_total = Counter('data_errors_total', 'Data validation errors')

# Gauges
pending_jobs = Gauge('pending_jobs', 'Jobs waiting in queue')
processing_jobs = Gauge('processing_jobs', 'Jobs currently processing')
worker_threads = Gauge('worker_threads', 'Active worker threads')

# Histograms
job_duration_seconds = Histogram('job_duration_seconds', 'Job processing time')
batch_size = Histogram('batch_size', 'Number of records per batch')
```

## Visualiser Vos Métriques dans Grafana

Une fois vos métriques exposées et collectées par Prometheus, créez des dashboards dans Grafana :

### Exemples de Panneaux

**1. Taux de requêtes**
```promql
sum(rate(http_requests_total[5m])) by (endpoint)
```

**2. Utilisateurs actifs en temps réel**
```promql
active_users
```

**3. Percentile 95 du temps de réponse**
```promql
histogram_quantile(0.95,
  sum(rate(http_request_duration_seconds_bucket[5m])) by (le, endpoint)
)
```

**4. Taux d'erreur**
```promql
sum(rate(http_requests_total{status=~"5.."}[5m]))
/
sum(rate(http_requests_total[5m])) * 100
```

**5. Commandes par heure**
```promql
increase(orders_created_total[1h])
```

Nous verrons la création de dashboards Grafana en détail au chapitre 13.

## Déboguer Vos Métriques

### Problème : Métriques non visibles dans Prometheus

**Vérifications** :
1. L'endpoint `/metrics` répond-il ?
   ```bash
   kubectl exec -it <pod-name> -- curl localhost:8080/metrics
   ```

2. Les annotations sont-elles correctes dans le déploiement ?
   ```bash
   kubectl get pod <pod-name> -o yaml | grep prometheus.io
   ```

3. Prometheus scrape-t-il le pod ?
   - Interface Prometheus → Status → Targets
   - Cherchez votre application

4. Y a-t-il des erreurs dans les logs Prometheus ?
   ```bash
   kubectl logs -n monitoring <prometheus-pod>
   ```

### Problème : Métrique à zéro alors qu'elle devrait avoir une valeur

**Causes possibles** :
- Le code qui incrémente la métrique n'est jamais exécuté
- Erreur dans le code (exception levée avant l'incrémentation)
- Label incorrect (crée une nouvelle série au lieu de modifier l'existante)

**Débogage** :
```python
# Ajoutez des logs temporaires
@app.route('/api/order')
def create_order():
    print("DEBUG: Creating order")  # Log
    orders_created.inc()
    print(f"DEBUG: Orders count: {orders_created._value.get()}")  # Valeur actuelle
    return {'status': 'success'}
```

## Ce Qu'il Faut Retenir

📊 **Les métriques custom sont essentielles** pour comprendre votre application :
- Elles capturent ce qui compte pour votre business
- Les métriques système ne suffisent pas

🎯 **Choisissez le bon type de métrique** :
- **Counter** : Pour compter des événements (requêtes, commandes, erreurs)
- **Gauge** : Pour mesurer un état actuel (utilisateurs connectés, queue)
- **Histogram** : Pour mesurer des distributions (temps de réponse, tailles)

🏷️ **Utilisez les labels intelligemment** :
- Ajoutent des dimensions à vos métriques
- Mais attention à la cardinalité !
- Jamais d'IDs uniques

🔧 **L'instrumentation est simple** :
- Installer une bibliothèque client Prometheus
- Créer des objets métriques
- Les incrémenter/modifier dans votre code
- Exposer l'endpoint `/metrics`

📈 **Visualisez dans Grafana** :
- Prometheus collecte, Grafana visualise
- Créez des dashboards métier parlants
- Partagez avec votre équipe

✅ **Bonnes pratiques** :
- Nommage cohérent
- Documentation des métriques
- Éviter la sur-instrumentation
- Tester les métriques en local avant déploiement

---

## Prochaines Étapes

Maintenant que vous savez créer des métriques custom, les prochaines sections vous apprendront à :
- **15.3** : Centraliser les logs avec ELK/EFK
- **15.4** : Corréler logs et métriques avec Loki
- **15.5** : Implémenter le tracing distribué avec Jaeger

Vos applications deviennent de plus en plus observables ! 🔍✨

⏭️ [Logs centralisés (ELK/EFK stack)](/15-observabilite-avancee/03-logs-centralises-elk-efk-stack.md)
