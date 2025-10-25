🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 15.5 Tracing Distribué avec Jaeger

## Introduction au Tracing Distribué

Nous avons vu les **métriques** (section 15.1-15.2) et les **logs** (sections 15.3-15.4). Il nous manque le troisième pilier de l'observabilité : les **traces distribuées**.

### Le Problème des Architectures Distribuées

Dans une architecture microservices, une simple requête utilisateur peut traverser **plusieurs services** :

```
Utilisateur clique sur "Acheter" →

Frontend → API Gateway → Auth Service → Product Service →
Inventory Service → Payment Service → Order Service →
Email Service → ✅ Confirmation
```

**Questions critiques** :
- Pourquoi cette requête a pris 5 secondes ?
- Quel service est le goulot d'étranglement ?
- Où exactement se produit l'erreur ?
- Combien de temps chaque étape prend-elle ?

**Avec les métriques seules** :
```
✅ Je sais que l'API Gateway a une latence de 2s
✅ Je sais que Payment Service a une latence de 3s
❌ Mais je ne vois pas le chemin complet d'une requête spécifique
❌ Je ne peux pas relier les événements entre services
```

**Avec les logs seuls** :
```
✅ Je vois les logs de chaque service
❌ Mais comment savoir quels logs appartiennent à la même requête ?
❌ Comment reconstruire le parcours complet ?
```

### La Solution : Le Tracing Distribué

Le **tracing distribué** suit le parcours **complet** d'une requête à travers tous les services, avec :
- Le **temps passé** à chaque étape
- Les **appels** entre services
- Les **erreurs** à chaque niveau
- Le **contexte** partagé entre services

**Analogie** :
- **Métriques** = Statistiques de trafic routier (vitesse moyenne, nombre de véhicules)
- **Logs** = Caméras de surveillance à chaque carrefour
- **Traces** = GPS personnel qui enregistre votre trajet complet de A à Z

## Concepts Fondamentaux

### Trace (Trace)

Une **trace** représente le **parcours complet** d'une requête à travers le système distribué.

```
Trace ID: abc-123-xyz-789

[Frontend] 0ms ────────────────────────────────────────► 450ms
              [API Gateway] 50ms ──────────────────► 420ms
                              [Auth] 60ms ──► 110ms
                              [Product] 120ms ──────► 200ms
                              [Payment] 210ms ──────────► 410ms
                              [Email] 350ms ──► 400ms
```

**Caractéristiques** :
- ID unique (Trace ID)
- Timestamp de début et de fin
- Durée totale
- Collection de **spans**

### Span

Un **span** représente une **unité de travail** dans la trace (un service, une fonction, une requête DB).

```
Span: "Process Payment"
├─ Trace ID: abc-123-xyz-789
├─ Span ID: span-456
├─ Parent Span ID: span-123
├─ Service: payment-service
├─ Operation: process_payment
├─ Start: 210ms
├─ Duration: 200ms
├─ Tags:
│  ├─ payment_method: "credit_card"
│  ├─ amount: 99.99
│  └─ currency: "EUR"
└─ Logs:
   ├─ 210ms: "Starting payment processing"
   ├─ 250ms: "Validating card"
   └─ 410ms: "Payment successful"
```

**Composants d'un span** :
- **Span ID** : Identifiant unique du span
- **Parent Span ID** : Lien vers le span parent (hiérarchie)
- **Operation Name** : Nom de l'opération (ex: "HTTP GET /api/order")
- **Start time** et **Duration** : Quand et combien de temps
- **Tags** : Métadonnées clé-valeur (status, http.method, etc.)
- **Logs** : Événements ponctuels pendant le span

### Relations Parent-Enfant

Les spans s'organisent en **arbre hiérarchique** :

```
Span: HTTP Request (parent)
  ├─ Span: Authenticate User (enfant 1)
  ├─ Span: Query Database (enfant 2)
  │   ├─ Span: Parse Query (petit-enfant)
  │   └─ Span: Execute Query (petit-enfant)
  └─ Span: Send Response (enfant 3)
```

Chaque span enfant a un **Parent Span ID** qui pointe vers son parent.

### Contexte de Trace (Trace Context)

Le **contexte de trace** est propagé entre les services pour maintenir la continuité.

**Sans contexte** :
```
Service A (Trace ID: 123) → Service B (Trace ID: 456)
❌ Impossible de relier les deux !
```

**Avec contexte** :
```
Service A (Trace ID: 123, Span ID: aaa)
   → HTTP Header: traceparent: 00-123-aaa-01
      → Service B (Trace ID: 123, Parent Span: aaa, Span ID: bbb)
         ✅ Même trace ! Lien parent-enfant établi !
```

**Propagation** : Via des headers HTTP (ou autres protocoles) :
```
GET /api/order HTTP/1.1
traceparent: 00-abc123xyz789-span456-01
tracestate: vendor=value
```

Standards : **W3C Trace Context**, OpenTracing, OpenTelemetry

## Qu'est-ce que Jaeger ?

**Jaeger** est une plateforme de tracing distribué open-source créée par Uber.

### Architecture Jaeger

```
┌────────────────────────────────────────────────────┐
│         Vos Applications (Instrumentées)           │
│                                                    │
│  ┌─────────┐  ┌─────────┐  ┌─────────┐             │
│  │Service A│  │Service B│  │Service C│             │
│  └────┬────┘  └────┬────┘  └────┬────┘             │
│       │            │            │                  │
│       │ spans      │ spans      │ spans            │
│       └────────────┴────────────┘                  │
└────────────────────┼───────────────────────────────┘
                     ↓
            ┌─────────────────┐
            │  Jaeger Agent   │ ← Sidecar local
            │  (UDP/compact)  │   (agrège spans)
            └────────┬────────┘
                     ↓
            ┌─────────────────┐
            │ Jaeger Collector│ ← Service central
            │  (validation)   │   (reçoit et valide)
            └────────┬────────┘
                     ↓
            ┌─────────────────┐
            │     Storage     │ ← Backend de stockage
            │  (Cassandra/    │   (Elasticsearch/
            │   Badger)       │    In-memory)
            └────────┬────────┘
                     ↓
            ┌─────────────────┐
            │   Jaeger Query  │ ← API de requête
            │      (API)      │
            └────────┬────────┘
                     ↓
            ┌─────────────────┐
            │   Jaeger UI     │ ← Interface web
            │  (Visualisation)│
            └─────────────────┘
```

### Composants Jaeger

#### 1. Jaeger Agent (Optionnel)

**Rôle** : Écouter les spans envoyés par les applications et les transférer au Collector.

**Caractéristiques** :
- Déployé comme **sidecar** ou **DaemonSet**
- Écoute sur UDP (protocoles compact Thrift)
- Buffer local
- Léger : ~20-40 MB RAM

**Note** : Dans les déploiements modernes (Jaeger v1.35+), on peut **envoyer directement au Collector** sans agent.

#### 2. Jaeger Collector

**Rôle** : Recevoir les spans, valider, et les stocker.

**Caractéristiques** :
- Reçoit spans via plusieurs protocoles (Thrift, gRPC, HTTP)
- Validation et transformation
- Écriture dans le backend de stockage
- Peut scaler horizontalement

#### 3. Storage Backend

**Rôle** : Stocker les traces de manière persistante.

**Options** :
- **In-Memory** : Pour tests/démos (non persistant)
- **Badger** : Base de données embarquée (pour labs)
- **Cassandra** : Production, haute volumétrie
- **Elasticsearch** : Production, recherche avancée

**Pour MicroK8s** : On utilisera **Badger** (simple, léger, persistant).

#### 4. Jaeger Query

**Rôle** : API pour récupérer les traces.

**Caractéristiques** :
- Endpoint REST pour l'UI
- Recherche et filtrage
- Agrégations

#### 5. Jaeger UI

**Rôle** : Interface web pour visualiser et analyser les traces.

**Fonctionnalités** :
- Recherche de traces
- Visualisation timeline
- Graphe de dépendances
- Comparaison de traces
- Statistiques de latence

### Jaeger All-in-One

Pour simplifier le déploiement (idéal pour labs), Jaeger offre **jaeger-all-in-one** :
- Tous les composants dans **un seul binaire**
- In-memory ou Badger storage
- Parfait pour MicroK8s !

## Instrumentation des Applications

Pour que Jaeger puisse tracer vos applications, vous devez les **instrumenter** : ajouter du code qui crée et envoie des spans.

### Standards : OpenTracing et OpenTelemetry

**OpenTracing** : Ancien standard, en maintenance
**OpenTelemetry** : Nouveau standard unifié (métriques + traces + logs)

**Recommandation** : Utilisez **OpenTelemetry** (souvent appelé **OTel**).

### Concepts d'Instrumentation

1. **Initialiser un tracer** : Configurer la connexion à Jaeger
2. **Créer des spans** : Marquer les opérations importantes
3. **Ajouter du contexte** : Tags, logs, baggage
4. **Propager le contexte** : Entre services

### Exemple Python avec OpenTelemetry

```python
from flask import Flask, request
from opentelemetry import trace
from opentelemetry.sdk.trace import TracerProvider
from opentelemetry.sdk.trace.export import BatchSpanProcessor
from opentelemetry.exporter.jaeger.thrift import JaegerExporter
from opentelemetry.instrumentation.flask import FlaskInstrumentor
from opentelemetry.instrumentation.requests import RequestsInstrumentor

app = Flask(__name__)

# Configuration du tracer
trace.set_tracer_provider(TracerProvider())
tracer = trace.get_tracer(__name__)

# Exporter vers Jaeger
jaeger_exporter = JaegerExporter(
    agent_host_name="jaeger-agent.monitoring.svc.cluster.local",
    agent_port=6831,
)
trace.get_tracer_provider().add_span_processor(
    BatchSpanProcessor(jaeger_exporter)
)

# Instrumentation automatique de Flask
FlaskInstrumentor().instrument_app(app)
# Instrumentation automatique des requêtes HTTP sortantes
RequestsInstrumentor().instrument()

@app.route('/api/order', methods=['POST'])
def create_order():
    # Span automatiquement créé par FlaskInstrumentor

    # Créer un span personnalisé pour une opération spécifique
    with tracer.start_as_current_span("validate_order") as span:
        span.set_attribute("order.items_count", 3)
        span.set_attribute("order.total", 99.99)

        # Logique de validation
        is_valid = validate_order_data(request.json)
        span.set_attribute("order.valid", is_valid)

        if not is_valid:
            span.set_status(trace.Status(trace.StatusCode.ERROR))
            span.record_exception(Exception("Invalid order data"))
            return {"error": "Invalid order"}, 400

    # Appeler un autre service (automatiquement tracé)
    with tracer.start_as_current_span("call_payment_service"):
        import requests
        response = requests.post(
            "http://payment-service/api/payment",
            json={"amount": 99.99}
        )

    # Enregistrer un événement dans le span courant
    span = trace.get_current_span()
    span.add_event("Order created successfully")

    return {"status": "success", "order_id": "12345"}

def validate_order_data(data):
    # Logique de validation
    return True

if __name__ == '__main__':
    app.run(host='0.0.0.0', port=8080)
```

**Points clés** :
- `FlaskInstrumentor()` : Instrumente automatiquement toutes les routes Flask
- `RequestsInstrumentor()` : Instrumente automatiquement les appels HTTP sortants
- `start_as_current_span()` : Crée un span personnalisé
- `set_attribute()` : Ajoute des tags (métadonnées)
- `add_event()` : Enregistre un événement ponctuel
- `record_exception()` : Capture les erreurs

### Exemple Node.js avec OpenTelemetry

```javascript
const { NodeTracerProvider } = require('@opentelemetry/sdk-trace-node');
const { JaegerExporter } = require('@opentelemetry/exporter-jaeger');
const { BatchSpanProcessor } = require('@opentelemetry/sdk-trace-base');
const { registerInstrumentations } = require('@opentelemetry/instrumentation');
const { HttpInstrumentation } = require('@opentelemetry/instrumentation-http');
const { ExpressInstrumentation } = require('@opentelemetry/instrumentation-express');

const express = require('express');
const axios = require('axios');

// Configuration du tracer
const provider = new NodeTracerProvider();
const exporter = new JaegerExporter({
    endpoint: 'http://jaeger-collector.monitoring.svc.cluster.local:14268/api/traces',
});
provider.addSpanProcessor(new BatchSpanProcessor(exporter));
provider.register();

// Instrumentation automatique
registerInstrumentations({
    instrumentations: [
        new HttpInstrumentation(),
        new ExpressInstrumentation(),
    ],
});

const tracer = provider.getTracer('payment-service');
const app = express();
app.use(express.json());

app.post('/api/payment', async (req, res) => {
    // Span automatiquement créé par ExpressInstrumentation

    const span = tracer.startSpan('process_payment');
    span.setAttributes({
        'payment.amount': req.body.amount,
        'payment.method': 'credit_card',
    });

    try {
        // Simuler le traitement du paiement
        await processPayment(req.body.amount);

        span.addEvent('Payment processed successfully');
        span.setStatus({ code: 0 }); // OK

        res.json({ status: 'success' });
    } catch (error) {
        span.recordException(error);
        span.setStatus({ code: 2, message: error.message }); // ERROR
        res.status(500).json({ error: 'Payment failed' });
    } finally {
        span.end();
    }
});

async function processPayment(amount) {
    const span = tracer.startSpan('validate_card');

    // Simuler validation
    await new Promise(resolve => setTimeout(resolve, 100));

    span.setAttributes({
        'card.valid': true,
    });
    span.end();

    // Simuler appel API bancaire
    const bankSpan = tracer.startSpan('call_bank_api');
    await new Promise(resolve => setTimeout(resolve, 200));
    bankSpan.end();
}

app.listen(8080, () => {
    console.log('Payment service listening on port 8080');
});
```

### Exemple Go avec OpenTelemetry

```go
package main

import (
    "context"
    "log"
    "net/http"
    "time"

    "go.opentelemetry.io/otel"
    "go.opentelemetry.io/otel/attribute"
    "go.opentelemetry.io/otel/exporters/jaeger"
    "go.opentelemetry.io/otel/sdk/resource"
    sdktrace "go.opentelemetry.io/otel/sdk/trace"
    semconv "go.opentelemetry.io/otel/semconv/v1.4.0"
    "go.opentelemetry.io/contrib/instrumentation/net/http/otelhttp"
)

func initTracer() (*sdktrace.TracerProvider, error) {
    // Créer l'exporter Jaeger
    exp, err := jaeger.New(jaeger.WithCollectorEndpoint(
        jaeger.WithEndpoint("http://jaeger-collector.monitoring.svc.cluster.local:14268/api/traces"),
    ))
    if err != nil {
        return nil, err
    }

    tp := sdktrace.NewTracerProvider(
        sdktrace.WithBatcher(exp),
        sdktrace.WithResource(resource.NewWithAttributes(
            semconv.SchemaURL,
            semconv.ServiceNameKey.String("order-service"),
        )),
    )
    otel.SetTracerProvider(tp)
    return tp, nil
}

func main() {
    tp, err := initTracer()
    if err != nil {
        log.Fatal(err)
    }
    defer func() {
        if err := tp.Shutdown(context.Background()); err != nil {
            log.Printf("Error shutting down tracer provider: %v", err)
        }
    }()

    tracer := otel.Tracer("order-service")

    // Instrumentation automatique des handlers HTTP
    handler := http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        ctx := r.Context()

        // Créer un span personnalisé
        ctx, span := tracer.Start(ctx, "create_order")
        defer span.End()

        span.SetAttributes(
            attribute.String("order.id", "12345"),
            attribute.Float64("order.total", 99.99),
        )

        // Simuler traitement
        time.Sleep(100 * time.Millisecond)

        // Appeler une fonction avec propagation du contexte
        processOrder(ctx)

        span.AddEvent("Order created")
        w.Write([]byte("Order created"))
    })

    // Wrapper avec instrumentation automatique
    wrappedHandler := otelhttp.NewHandler(handler, "create_order")
    http.Handle("/api/order", wrappedHandler)

    log.Println("Server starting on :8080")
    log.Fatal(http.ListenAndServe(":8080", nil))
}

func processOrder(ctx context.Context) {
    tracer := otel.Tracer("order-service")
    _, span := tracer.Start(ctx, "process_order")
    defer span.End()

    span.SetAttributes(attribute.String("processing.step", "validation"))
    time.Sleep(50 * time.Millisecond)
}
```

### Instrumentation Automatique vs Manuelle

**Automatique** :
- ✅ Facile, zéro code (ou presque)
- ✅ Trace HTTP, DB, cache automatiquement
- ❌ Moins de contrôle
- ❌ Pas de contexte métier

**Manuelle** :
- ✅ Contrôle total
- ✅ Contexte métier riche
- ❌ Plus de code
- ❌ Maintenance

**Recommandation** : **Hybride** - Automatique pour l'infrastructure, manuelle pour la logique métier.

## Déploiement de Jaeger dans MicroK8s

### Option 1 : Jaeger All-in-One (Recommandé pour Labs)

**Deployment Jaeger**

Fichier `jaeger-all-in-one.yaml` :

```yaml
apiVersion: v1
kind: Service
metadata:
  name: jaeger
  namespace: monitoring
  labels:
    app: jaeger
spec:
  type: ClusterIP
  ports:
  # Agent ports
  - name: agent-zipkin-thrift
    port: 5775
    protocol: UDP
    targetPort: 5775
  - name: agent-compact
    port: 6831
    protocol: UDP
    targetPort: 6831
  - name: agent-binary
    port: 6832
    protocol: UDP
    targetPort: 6832
  # Collector ports
  - name: collector-http
    port: 14268
    protocol: TCP
    targetPort: 14268
  - name: collector-grpc
    port: 14250
    protocol: TCP
    targetPort: 14250
  # Query/UI ports
  - name: query-http
    port: 16686
    protocol: TCP
    targetPort: 16686
  # Health check
  - name: admin
    port: 14269
    protocol: TCP
    targetPort: 14269
  selector:
    app: jaeger
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: jaeger
  namespace: monitoring
  labels:
    app: jaeger
spec:
  replicas: 1
  selector:
    matchLabels:
      app: jaeger
  template:
    metadata:
      labels:
        app: jaeger
    spec:
      containers:
      - name: jaeger
        image: jaegertracing/all-in-one:1.51
        env:
        - name: COLLECTOR_ZIPKIN_HOST_PORT
          value: ":9411"
        - name: COLLECTOR_OTLP_ENABLED
          value: "true"
        - name: SPAN_STORAGE_TYPE
          value: "badger"
        - name: BADGER_EPHEMERAL
          value: "false"
        - name: BADGER_DIRECTORY_VALUE
          value: "/badger/data"
        - name: BADGER_DIRECTORY_KEY
          value: "/badger/key"
        ports:
        - containerPort: 5775
          protocol: UDP
        - containerPort: 6831
          protocol: UDP
        - containerPort: 6832
          protocol: UDP
        - containerPort: 14268
          protocol: TCP
        - containerPort: 14250
          protocol: TCP
        - containerPort: 16686
          protocol: TCP
        - containerPort: 14269
          protocol: TCP
        resources:
          requests:
            memory: "256Mi"
            cpu: "200m"
          limits:
            memory: "512Mi"
            cpu: "500m"
        volumeMounts:
        - name: badger-data
          mountPath: /badger
        readinessProbe:
          httpGet:
            path: /
            port: 14269
          initialDelaySeconds: 5
        livenessProbe:
          httpGet:
            path: /
            port: 14269
          initialDelaySeconds: 10
      volumes:
      - name: badger-data
        persistentVolumeClaim:
          claimName: jaeger-badger-pvc
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: jaeger-badger-pvc
  namespace: monitoring
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: microk8s-hostpath
  resources:
    requests:
      storage: 5Gi
```

**Points importants** :
- **All-in-one** : Tous les composants dans un pod
- **Badger storage** : Base de données embarquée, persistante
- **PVC** : Volume persistant de 5GB pour les traces
- **Multi-ports** : Agent (UDP), Collector (HTTP/gRPC), UI (HTTP)

**Déployer Jaeger** :

```bash
# Créer le namespace si nécessaire
microk8s kubectl create namespace monitoring

# Déployer Jaeger
microk8s kubectl apply -f jaeger-all-in-one.yaml

# Vérifier
microk8s kubectl get pods -n monitoring -l app=jaeger
microk8s kubectl logs -n monitoring -l app=jaeger

# Attendre que le pod soit prêt
microk8s kubectl wait --for=condition=ready pod -l app=jaeger -n monitoring --timeout=120s
```

### Accéder à l'Interface Jaeger

**Option 1 : Port-forward (temporaire)**

```bash
microk8s kubectl port-forward -n monitoring svc/jaeger 16686:16686

# Accédez à http://localhost:16686
```

**Option 2 : Ingress (permanent)**

Fichier `jaeger-ingress.yaml` :

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: jaeger
  namespace: monitoring
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  ingressClassName: nginx
  rules:
  - host: jaeger.votre-domaine.local
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: jaeger
            port:
              number: 16686
```

```bash
microk8s kubectl apply -f jaeger-ingress.yaml

# Ajoutez dans /etc/hosts :
# 192.168.1.100  jaeger.votre-domaine.local

# Accédez à http://jaeger.votre-domaine.local
```

### Option 2 : Déploiement Production (Séparé)

Pour un environnement production, déployez les composants séparément :
- **Jaeger Collector** : StatefulSet avec réplication
- **Jaeger Query** : Deployment scalable
- **Elasticsearch** : Pour le storage (au lieu de Badger)
- **Jaeger Agent** : DaemonSet sur tous les nœuds

(Configuration complète disponible dans la documentation officielle)

### Vérification

```bash
# Vérifier que tous les services sont UP
microk8s kubectl get svc -n monitoring -l app=jaeger

# Vérifier les logs
microk8s kubectl logs -n monitoring -l app=jaeger --tail=50

# Tester l'API
kubectl port-forward -n monitoring svc/jaeger 14269:14269
curl http://localhost:14269/ # Devrait retourner OK
```

## Utiliser l'Interface Jaeger

### Page d'Accueil

Ouvrez l'interface Jaeger : `http://localhost:16686`

**Sections principales** :
1. **Search** : Rechercher des traces
2. **Compare** : Comparer plusieurs traces
3. **System Architecture** : Graphe de dépendances
4. **Monitor** : Métriques et statistiques

### Rechercher des Traces

**Interface de recherche** :

```
┌────────────────────────────────────────────────────┐
│  Service: [payment-service ▼]                      │
│  Operation: [All ▼]                                │
│  Tags: [http.status_code=500]                      │
│  Lookback: [1h ▼]                                  │
│  Max Duration:                Min Duration:        │
│  Limit Results: 20                                 │
│                                   [Find Traces]    │
└────────────────────────────────────────────────────┘
```

**Filtres disponibles** :
- **Service** : Sélectionnez un service spécifique
- **Operation** : Type d'opération (ex: "HTTP GET /api/order")
- **Tags** : Filtrer par attributs (ex: http.status_code, error=true)
- **Lookback** : Période de recherche (dernière heure, 2h, etc.)
- **Min/Max Duration** : Filtrer par durée (ex: traces > 1s)
- **Limit** : Nombre de résultats

**Exemples de recherches** :

```
# Toutes les erreurs du service payment
Service: payment-service
Tags: error=true

# Requêtes lentes (> 2 secondes)
Service: api-gateway
Min Duration: 2s

# Toutes les requêtes POST
Operation: HTTP POST

# Statut 500
Tags: http.status_code=500
```

### Visualiser une Trace

Cliquez sur une trace dans les résultats :

```
┌──────────────────────────────────────────────────────────┐
│ Trace: abc-123-xyz-789                                   │
│ Duration: 450ms    Spans: 8    Services: 4    Errors: 0  │
├──────────────────────────────────────────────────────────┤
│                                                          │
│ Frontend ──────────────────────────────────► 450ms       │
│   └─ API Gateway ────────────────────► 420ms             │
│        ├─ Auth Service ─► 50ms                           │
│        ├─ Product Service ──────► 80ms                   │
│        └─ Payment Service ────────────► 200ms            │
│             ├─ Validate Card ─► 50ms                     │
│             └─ Bank API ────────► 150ms                  │
│                                                          │
└──────────────────────────────────────────────────────────┘
```

**Timeline View** : Visualisation chronologique
- Chaque barre représente un span
- Longueur = durée
- Indentation = hiérarchie parent-enfant
- Couleurs = services différents

### Détails d'un Span

Cliquez sur un span pour voir les détails :

```
┌──────────────────────────────────────────────────────┐
│ Span: process_payment                                │
│ Service: payment-service                             │
│ Duration: 200ms                                      │
├──────────────────────────────────────────────────────┤
│ Tags:                                                │
│   payment.amount: 99.99                              │
│   payment.method: credit_card                        │
│   payment.currency: EUR                              │
│   http.method: POST                                  │
│   http.status_code: 200                              │
├──────────────────────────────────────────────────────┤
│ Process:                                             │
│   service.name: payment-service                      │
│   service.version: v1.2.3                            │
│   hostname: payment-abc123                           │
├──────────────────────────────────────────────────────┤
│ Logs:                                                │
│   210ms: Starting payment processing                 │
│   250ms: Validating card                             │
│   300ms: Calling bank API                            │
│   410ms: Payment successful                          │
└──────────────────────────────────────────────────────┘
```

### Graphe de Dépendances

**System Architecture** affiche le graphe de vos services :

```
       Frontend
          │
          ▼
     API Gateway
        /  |  \
       /   |   \
      ▼    ▼    ▼
   Auth Product Payment
              │
              ▼
          Bank API
```

**Informations affichées** :
- Nœuds : Services
- Flèches : Appels entre services
- Épaisseur : Volume de trafic
- Couleur : Taux d'erreur

**Utilité** :
- Comprendre l'architecture réelle
- Identifier les dépendances cachées
- Repérer les services critiques (hub)

### Monitor : Statistiques

**Monitor** affiche des métriques sur les traces :

- **Latency** : Graphe de latence au fil du temps
- **Error Rate** : Taux d'erreur
- **Request Rate** : Nombre de requêtes
- **Service Performance** : Comparaison entre services

## Patterns d'Analyse

### Pattern 1 : Identifier le Service Lent

**Problème** : Requêtes globalement lentes

**Démarche** :
1. Recherchez les traces lentes (Min Duration > 2s)
2. Regardez la timeline
3. Identifiez le span le plus long

**Exemple** :
```
Total: 5.2s
├─ Frontend: 5.2s
   └─ API Gateway: 5.1s
      ├─ Auth: 0.05s
      └─ Payment: 5.0s ← COUPABLE !
         └─ Bank API: 4.9s ← VRAIE SOURCE
```

**Conclusion** : Le problème vient de l'API bancaire externe.

### Pattern 2 : Comparer Traces Rapides vs Lentes

**Problème** : Certaines requêtes sont rapides, d'autres lentes

**Démarche** :
1. Trouvez une trace rapide (ex: 200ms)
2. Trouvez une trace lente (ex: 5s)
3. Cliquez sur **Compare** → Sélectionnez les deux traces
4. Comparez les timings

**Ce que vous cherchez** :
- Quel span diffère ?
- Y a-t-il un span absent dans la rapide ?
- Les attributs sont-ils différents ?

### Pattern 3 : Suivre une Requête Utilisateur

**Problème** : Un utilisateur signale un problème

**Si vous avez un Request ID** :
1. Recherchez par tag : `request.id=abc-123`
2. Vous voyez toute la trace de cette requête spécifique

**Si vous n'avez pas de Request ID** :
1. Recherchez par période (quand l'utilisateur a signalé)
2. Filtrez par tags pertinents (user_id, order_id, etc.)

### Pattern 4 : Analyse d'Erreur

**Problème** : Erreurs sporadiques

**Démarche** :
1. Recherchez : `Tags: error=true`
2. Regardez plusieurs traces d'erreur
3. Cherchez les patterns communs :
   - Même service qui échoue ?
   - Même tag (ex: payment_method=paypal) ?
   - Même timing (ex: toujours après 30s = timeout) ?

**Exemple** :
```
5 traces d'erreur sur 100
Toutes ont le tag: payment_method=paypal
Toutes échouent sur le span "Bank API"
→ Problème spécifique à l'intégration PayPal !
```

### Pattern 5 : Détection de N+1 Queries

**Problème** : Performance dégradée avec plus de données

**Ce que vous voyez** :
```
Get Order (1ms)
├─ Get Items (1ms)
│  ├─ DB Query Item 1 (10ms)
│  ├─ DB Query Item 2 (10ms)
│  ├─ DB Query Item 3 (10ms)
│  ├─ DB Query Item 4 (10ms)
│  └─ DB Query Item 5 (10ms)
└─ Total: 51ms
```

**Problème identifié** : 5 requêtes DB séparées (N+1 problem)

**Solution** : Une seule requête avec JOIN ou IN clause.

## Corrélation avec Logs et Métriques

### Lier Traces et Logs

**Dans vos logs, incluez le Trace ID** :

```python
# Python avec OpenTelemetry
from opentelemetry import trace

span = trace.get_current_span()
trace_id = span.get_span_context().trace_id
span_id = span.get_span_context().span_id

logger.info(
    "Processing payment",
    extra={
        'trace_id': format(trace_id, '032x'),
        'span_id': format(span_id, '016x'),
        'payment_amount': 99.99
    }
)
```

**Dans Loki/Elasticsearch, vous pouvez alors** :
```logql
{app="payment"} | json | trace_id="abc123xyz789"
```

Et voir tous les logs de cette trace !

### Lier Traces et Métriques

**Utilisez les mêmes labels** :

```python
# Métrique Prometheus
payment_duration.labels(
    payment_method='credit_card',
    status='success'
).observe(0.2)

# Tag dans le span Jaeger
span.set_attribute('payment.method', 'credit_card')
span.set_attribute('payment.status', 'success')
```

**Dans Grafana** :
- Panneau gauche : Graphe Prometheus de `payment_duration`
- Panneau droit : Lien vers Jaeger filtré par les mêmes labels

### Dashboard Unifié dans Grafana

Grafana peut intégrer Jaeger comme datasource :

**Configuration** :
1. **Configuration** → **Data Sources** → **Add data source**
2. Sélectionnez **Jaeger**
3. URL : `http://jaeger.monitoring.svc.cluster.local:16686`
4. **Save & Test**

**Utilisation dans un dashboard** :

Panel de traces :
```
Data source: Jaeger
Query: {service="payment-service"}
```

**Trace to Metrics** : Grafana peut créer des liens automatiques entre traces et métriques si configuré.

### Workflow Complet d'Investigation

1. **Alerte Prometheus** : Latence élevée détectée
2. **Dashboard Grafana** : Voir le graphe de latence
3. **Cliquer sur le pic** : Naviguer vers les traces Jaeger de cette période
4. **Analyser la trace** : Identifier le service lent
5. **Voir les logs** : Consulter les logs de ce service avec le trace_id
6. **Conclusion** : Comprendre la cause racine

**Temps d'investigation** : Minutes au lieu d'heures ! 🚀

## Bonnes Pratiques

### 1. Sampling (Échantillonnage)

**Problème** : Tracer 100% du trafic est coûteux (CPU, bande passante, stockage).

**Solution** : Échantillonnage intelligent

**Stratégies** :

**A. Probabilistic Sampling** (aléatoire)
```python
# 10% des requêtes
sampler = TraceIdRatioBased(0.1)
```

**B. Rate Limiting Sampling**
```python
# Max 100 traces/seconde
```

**C. Adaptive Sampling** (selon la charge)
- Haute charge : 1% échantillonnage
- Charge normale : 10%
- Faible charge : 100%

**D. Always Sample Errors**
```python
# Toujours tracer les erreurs
if error:
    span.set_attribute('sampling.priority', 1)
```

**Recommandation pour lab** : 100% (volume faible)
**Recommandation production** : 1-10% avec priorité sur les erreurs

### 2. Nommage des Spans

**❌ Mauvais** :
```
span_name = "function"
span_name = "process"
span_name = f"order_{order_id}"  # Cardinalité élevée
```

**✅ Bon** :
```
span_name = "HTTP GET /api/orders"
span_name = "db_query_get_user"
span_name = "process_payment"
```

**Règles** :
- Noms descriptifs et génériques
- Pas de valeurs uniques (IDs, timestamps)
- Format cohérent

### 3. Tags Utiles

**Tags essentiels** :
```python
span.set_attribute('http.method', 'POST')
span.set_attribute('http.url', '/api/order')
span.set_attribute('http.status_code', 200)
span.set_attribute('db.system', 'postgresql')
span.set_attribute('db.statement', 'SELECT * FROM orders')
span.set_attribute('error', True)  # Si erreur
span.set_attribute('error.message', str(e))
```

**Tags métier** :
```python
span.set_attribute('user.id', user_id)
span.set_attribute('order.id', order_id)
span.set_attribute('payment.method', 'credit_card')
span.set_attribute('payment.amount', 99.99)
```

**Attention** : Pas de données sensibles (mots de passe, numéros de carte).

### 4. Logs dans les Spans

Enregistrez les événements importants :

```python
span.add_event('User authenticated')
span.add_event('Database connection acquired')
span.add_event('Payment processed', attributes={
    'transaction.id': 'txn-123',
    'amount': 99.99
})
```

**Pas besoin de tout logger** : Seulement les étapes importantes.

### 5. Gestion des Erreurs

**Toujours** enregistrer les exceptions :

```python
try:
    process_payment()
except Exception as e:
    span.record_exception(e)
    span.set_status(trace.Status(trace.StatusCode.ERROR))
    raise
```

Jaeger marquera le span en rouge et inclura la stack trace.

### 6. Propagation du Contexte

**HTTP** : Automatique avec les bibliothèques
**Message Queues** : Propagez manuellement

```python
# Extracting context from message
from opentelemetry.propagate import extract

context = extract(message.headers)
with tracer.start_as_current_span("process_message", context=context):
    # Votre logique
```

### 7. Span Granularité

**Trop fin** :
```python
span1 = tracer.start_span("variable_x = 1")
span2 = tracer.start_span("variable_y = 2")
span3 = tracer.start_span("sum = x + y")
```
❌ Trop de détails, surcharge

**Trop large** :
```python
span = tracer.start_span("process_everything")
# 200 lignes de code
span.end()
```
❌ Pas assez de détails

**Juste bien** :
```python
with tracer.start_span("validate_order"):
    # Validation logic

with tracer.start_span("process_payment"):
    # Payment logic

with tracer.start_span("send_confirmation"):
    # Email logic
```
✅ Niveau de détail approprié

## Performance et Coût

### Impact sur l'Application

**Overhead** :
- **CPU** : 1-5% supplémentaire
- **Mémoire** : Négligeable (~10-50 MB buffer)
- **Latence** : < 1ms par span (envoi asynchrone)

**Optimisations** :
- Batch processing (groupe plusieurs spans avant envoi)
- Async export (envoi en arrière-plan)
- Sampling (réduire le volume)

### Stockage

**Estimation** :
- 1 span ≈ 1-2 KB
- Trace moyenne ≈ 10-50 KB (10-25 spans)
- 1000 req/s avec 10% sampling = 100 traces/s = 5 MB/s
- Sur 7 jours : ~3 TB

**Pour MicroK8s (faible trafic)** :
- 10 req/s = 1 MB/minute = ~600 MB/jour
- Rétention 7 jours = ~4 GB

### Rétention

Configuration Badger dans Jaeger :
```yaml
env:
- name: SPAN_STORAGE_TYPE
  value: badger
- name: BADGER_TTL
  value: 168h  # 7 jours
```

Ajustez selon votre espace disque disponible.

## Alternatives à Jaeger

### Zipkin

**Similaire à Jaeger**, créé par Twitter.

**Différences** :
- Plus ancien, plus mature
- Interface plus simple
- Moins de fonctionnalités avancées

### Tempo (Grafana)

**Alternative Grafana** pour le tracing.

**Avantages** :
- Intégration native Grafana (comme Loki)
- Stockage S3/GCS
- Très scalable

**Inconvénients** :
- Plus récent, moins mature
- Configuration plus complexe

### OpenTelemetry Collector

**Hub universel** pour métriques, logs, ET traces.

**Architecture** :
```
Applications → OTel Collector → Jaeger/Tempo/Zipkin/etc.
```

Permet de changer de backend sans modifier vos applications.

## Ce Qu'il Faut Retenir

🔍 **Le tracing distribué suit les requêtes** :
- À travers tous les services
- Avec le timing de chaque étape
- En conservant le contexte

🎯 **Concepts clés** :
- **Trace** : Parcours complet d'une requête
- **Span** : Unité de travail (service, fonction, query)
- **Contexte** : Propagé entre services (Trace ID)

🛠️ **Jaeger** :
- Plateforme open-source de tracing
- All-in-one parfait pour MicroK8s
- Interface web intuitive

💻 **Instrumentation** :
- OpenTelemetry = standard moderne
- Automatique pour HTTP, DB, cache
- Manuelle pour logique métier

🔗 **Corrélation** :
- Trace ID dans les logs
- Mêmes labels que Prometheus
- Dashboard Grafana unifié

📊 **Analyses possibles** :
- Identifier les services lents
- Comprendre les dépendances
- Déboguer les erreurs
- Optimiser les performances

⚡ **Bonnes pratiques** :
- Sampling pour réduire le coût
- Noms de spans descriptifs
- Tags riches mais pas sensibles
- Toujours capturer les erreurs

🎨 **Les trois piliers ensemble** :
- **Métriques** : QUOI et COMBIEN
- **Logs** : POURQUOI et DÉTAILS
- **Traces** : OÙ et COMMENT

---

## Conclusion

Avec Jaeger déployé, vous complétez les **trois piliers de l'observabilité** :

1. ✅ **Métriques** (Prometheus) : Statistiques numériques
2. ✅ **Logs** (Loki) : Événements textuels
3. ✅ **Traces** (Jaeger) : Parcours des requêtes

Ensemble, ils vous donnent une **visibilité complète** sur votre système distribué dans MicroK8s.

**Workflow d'investigation idéal** :
1. **Alerte** déclenché (métrique)
2. **Dashboard** consulté (métriques + logs)
3. **Trace** analysée (parcours complet)
4. **Logs** examinés (détails de l'erreur)
5. **Cause racine** identifiée
6. **Fix** déployé
7. **Vérification** avec métriques/traces

Votre cluster MicroK8s est maintenant **observé de bout en bout**. Félicitations ! 🎉

**Prochaines sections** :
- 15.6 : Synthetic monitoring avec Blackbox exporter
- 15.7 : SLI/SLO et dashboards de fiabilité

Vous avez maintenant toutes les bases pour une observabilité professionnelle, même sur un simple lab personnel ! 🚀

⏭️ [Synthetic monitoring avec Blackbox exporter](/15-observabilite-avancee/06-synthetic-monitoring-avec-blackbox-exporter.md)
