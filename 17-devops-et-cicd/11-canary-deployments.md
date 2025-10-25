🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 17.11 Canary deployments

## Introduction

Dans la section précédente, nous avons exploré les déploiements Blue-Green qui permettent un basculement instantané entre deux versions. Maintenant, nous allons découvrir les **déploiements Canary**, une stratégie qui privilégie la **progressivité** et la **validation en conditions réelles** plutôt que le basculement brutal.

Le nom "Canary" vient d'une pratique historique dans les mines de charbon : les mineurs emmenaient des canaris avec eux car ces oiseaux sont très sensibles aux gaz toxiques. Si le canari montrait des signes de détresse, c'était le signal d'évacuer avant que les mineurs ne soient affectés.

En informatique, c'est le même principe : on expose la nouvelle version à un **petit groupe d'utilisateurs** (le canari) pour détecter les problèmes avant qu'ils n'impactent tout le monde. Si le "canari" survit et se comporte bien, on augmente progressivement l'exposition jusqu'à remplacer complètement l'ancienne version.

---

## Principe et philosophie

### Le concept Canary

Le déploiement Canary consiste à déployer une nouvelle version **en parallèle** de l'ancienne, puis à router **progressivement** le trafic vers la nouvelle version en surveillant attentivement les métriques.

**Workflow typique** :

```
Phase 0 : 100% v1
┌────────────────────────────────────────┐
│  v1 (100%)                             │
│  ████████████████████████████████████  │
└────────────────────────────────────────┘

Phase 1 : 95% v1, 5% v2 (Canary)
┌────────────────────────────────────────┐
│  v1 (95%)                    v2 (5%)   │
│  ██████████████████████████████ ██     │
└────────────────────────────────────────┘
        ↓ Surveiller 5-10 minutes

Phase 2 : 80% v1, 20% v2
┌────────────────────────────────────────┐
│  v1 (80%)               v2 (20%)       │
│  ██████████████████████ ████████       │
└────────────────────────────────────────┘
        ↓ Surveiller 10-15 minutes

Phase 3 : 50% v1, 50% v2
┌────────────────────────────────────────┐
│  v1 (50%)          v2 (50%)            │
│  ████████████████  ████████████████    │
└────────────────────────────────────────┘
        ↓ Surveiller 15-30 minutes

Phase 4 : 0% v1, 100% v2 (Promotion)
┌────────────────────────────────────────┐
│                            v2 (100%)   │
│  ████████████████████████████████████  │
└────────────────────────────────────────┘
```

### Différences clés avec Blue-Green

| Aspect | Blue-Green | Canary |
|--------|-----------|--------|
| **Transition** | Instantanée (switch) | Progressive (graduée) |
| **Risque** | Tous impactés en cas de bug | Seul un % est impacté |
| **Durée** | Rapide (minutes) | Lente (heures) |
| **Validation** | Tests pré-bascule | Validation continue |
| **Ressources** | 2x (temporaire) | 1.2-1.5x (progressif) |
| **Rollback** | Instantané | Automatique basé sur métriques |
| **Complexité** | Moyenne | Élevée |

**Blue-Green** : "J'ai testé, je suis confiant, on bascule tout d'un coup"

**Canary** : "Je ne suis pas 100% sûr, testons d'abord avec quelques utilisateurs réels"

### Cas d'usage idéaux

**✅ Quand utiliser Canary** :

- Nouvelle fonctionnalité jamais testée en production réelle
- Refonte majeure d'un algorithme
- Changement de comportement incertain
- Application à très fort trafic
- Besoin de valider les performances en condition réelle
- Incertitude sur la stabilité

**❌ Quand ne pas utiliser Canary** :

- Simple correction de bug
- Application à faible trafic
- Changements mineurs
- Urgence (hotfix critique)
- Migrations de schéma incompatibles

---

## Implémentation manuelle avec Kubernetes

### Architecture de base

Pour un Canary manuel, on utilise deux Deployments avec un Service qui les sélectionne tous les deux :

```
                     ┌────────────┐
                     │   Service  │
                     │ (app=mon)  │
                     └──────┬─────┘
                            │
              ┌─────────────┴─────────────┐
              │                           │
    ┌─────────▼─────────┐       ┌─────────▼───────┐
    │  Deployment       │       │  Deployment     │
    │  Stable           │       │  Canary         │
    │  (app=mon,        │       │  (app=mon,      │
    │   track=stable)   │       │   track=canary) │
    │                   │       │                 │
    │  Replicas: 9      │       │  Replicas: 1    │
    │  (90% traffic)    │       │  (10% traffic)  │
    └───────────────────┘       └─────────────────┘
```

### Manifestes Kubernetes

#### Déploiement stable

```yaml
---
# deployment-stable.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mon-app-stable
  namespace: production
  labels:
    app: mon-app
    track: stable
spec:
  replicas: 9  # 90% du trafic
  selector:
    matchLabels:
      app: mon-app
      track: stable
  template:
    metadata:
      labels:
        app: mon-app
        track: stable
        version: v1.0.0
      annotations:
        prometheus.io/scrape: "true"
        prometheus.io/port: "8080"
        prometheus.io/path: "/metrics"
    spec:
      containers:
      - name: app
        image: localhost:32000/mon-app:v1.0.0
        ports:
        - name: http
          containerPort: 8080
        env:
        - name: VERSION
          value: "v1.0.0"
        - name: TRACK
          value: "stable"
        resources:
          requests:
            memory: "256Mi"
            cpu: "250m"
          limits:
            memory: "512Mi"
            cpu: "500m"
        readinessProbe:
          httpGet:
            path: /health
            port: 8080
          initialDelaySeconds: 5
          periodSeconds: 5
          timeoutSeconds: 3
          successThreshold: 1
          failureThreshold: 3
        livenessProbe:
          httpGet:
            path: /health
            port: 8080
          initialDelaySeconds: 30
          periodSeconds: 10
          timeoutSeconds: 5
          failureThreshold: 3
```

#### Déploiement Canary

```yaml
---
# deployment-canary.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mon-app-canary
  namespace: production
  labels:
    app: mon-app
    track: canary
spec:
  replicas: 1  # 10% du trafic initial
  selector:
    matchLabels:
      app: mon-app
      track: canary
  template:
    metadata:
      labels:
        app: mon-app
        track: canary
        version: v2.0.0
      annotations:
        prometheus.io/scrape: "true"
        prometheus.io/port: "8080"
        prometheus.io/path: "/metrics"
    spec:
      containers:
      - name: app
        image: localhost:32000/mon-app:v2.0.0
        ports:
        - name: http
          containerPort: 8080
        env:
        - name: VERSION
          value: "v2.0.0"
        - name: TRACK
          value: "canary"
        resources:
          requests:
            memory: "256Mi"
            cpu: "250m"
          limits:
            memory: "512Mi"
            cpu: "500m"
        readinessProbe:
          httpGet:
            path: /health
            port: 8080
          initialDelaySeconds: 5
          periodSeconds: 5
          timeoutSeconds: 3
          successThreshold: 1
          failureThreshold: 3
        livenessProbe:
          httpGet:
            path: /health
            port: 8080
          initialDelaySeconds: 30
          periodSeconds: 10
          timeoutSeconds: 5
          failureThreshold: 3
```

#### Service

```yaml
---
# service.yaml
apiVersion: v1
kind: Service
metadata:
  name: mon-app
  namespace: production
  labels:
    app: mon-app
spec:
  type: ClusterIP
  selector:
    app: mon-app
    # Note : Pas de selector sur 'track'
    # Le service sélectionne stable ET canary
  ports:
  - name: http
    port: 80
    targetPort: 8080
    protocol: TCP
```

### Script de progression manuelle

```bash
#!/bin/bash
# canary-manual.sh - Gestion manuelle d'un déploiement Canary

set -euo pipefail

APP_NAME="mon-app"
NAMESPACE="production"
NEW_VERSION="${1:-}"

# Couleurs
GREEN='\033[0;32m'
YELLOW='\033[1;33m'
RED='\033[0;31m'
NC='\033[0m'

log_info() { echo -e "${GREEN}[INFO]${NC} $1"; }
log_warn() { echo -e "${YELLOW}[WARN]${NC} $1"; }
log_error() { echo -e "${RED}[ERROR]${NC} $1"; }

if [ -z "$NEW_VERSION" ]; then
    log_error "Usage: $0 <version>"
    exit 1
fi

# Phase 1 : Déployer le Canary (10%)
log_info "Phase 1/5: Déploiement du Canary (10%)"
kubectl set image deployment/${APP_NAME}-canary \
    app=localhost:32000/${APP_NAME}:${NEW_VERSION} \
    -n ${NAMESPACE}

kubectl scale deployment/${APP_NAME}-stable --replicas=9 -n ${NAMESPACE}
kubectl scale deployment/${APP_NAME}-canary --replicas=1 -n ${NAMESPACE}

kubectl rollout status deployment/${APP_NAME}-canary -n ${NAMESPACE}
log_info "Canary déployé. Surveillance 5 minutes..."
sleep 300

# Vérification des métriques (simulée ici)
read -p "Les métriques sont-elles bonnes? (yes/no): " -r
if [[ ! $REPLY =~ ^[Yy][Ee][Ss]$ ]]; then
    log_error "Rollback du Canary"
    kubectl scale deployment/${APP_NAME}-canary --replicas=0 -n ${NAMESPACE}
    exit 1
fi

# Phase 2 : 20%
log_info "Phase 2/5: Augmentation à 20%"
kubectl scale deployment/${APP_NAME}-stable --replicas=8 -n ${NAMESPACE}
kubectl scale deployment/${APP_NAME}-canary --replicas=2 -n ${NAMESPACE}
log_info "20% sur Canary. Surveillance 10 minutes..."
sleep 600

read -p "Les métriques sont-elles bonnes? (yes/no): " -r
if [[ ! $REPLY =~ ^[Yy][Ee][Ss]$ ]]; then
    log_error "Rollback du Canary"
    kubectl scale deployment/${APP_NAME}-stable --replicas=9 -n ${NAMESPACE}
    kubectl scale deployment/${APP_NAME}-canary --replicas=1 -n ${NAMESPACE}
    exit 1
fi

# Phase 3 : 50%
log_info "Phase 3/5: Augmentation à 50%"
kubectl scale deployment/${APP_NAME}-stable --replicas=5 -n ${NAMESPACE}
kubectl scale deployment/${APP_NAME}-canary --replicas=5 -n ${NAMESPACE}
log_info "50% sur Canary. Surveillance 15 minutes..."
sleep 900

read -p "Les métriques sont-elles bonnes? (yes/no): " -r
if [[ ! $REPLY =~ ^[Yy][Ee][Ss]$ ]]; then
    log_error "Rollback du Canary"
    kubectl scale deployment/${APP_NAME}-stable --replicas=9 -n ${NAMESPACE}
    kubectl scale deployment/${APP_NAME}-canary --replicas=1 -n ${NAMESPACE}
    exit 1
fi

# Phase 4 : 100% (Promotion)
log_info "Phase 4/5: Promotion à 100%"
kubectl scale deployment/${APP_NAME}-stable --replicas=0 -n ${NAMESPACE}
kubectl scale deployment/${APP_NAME}-canary --replicas=10 -n ${NAMESPACE}
log_info "100% sur Canary. Surveillance 30 minutes..."
sleep 1800

# Phase 5 : Finalisation
log_info "Phase 5/5: Finalisation"
read -p "Promouvoir définitivement le Canary? (yes/no): " -r
if [[ $REPLY =~ ^[Yy][Ee][Ss]$ ]]; then
    # Mettre à jour stable avec la nouvelle version
    kubectl set image deployment/${APP_NAME}-stable \
        app=localhost:32000/${APP_NAME}:${NEW_VERSION} \
        -n ${NAMESPACE}
    kubectl scale deployment/${APP_NAME}-stable --replicas=10 -n ${NAMESPACE}
    kubectl scale deployment/${APP_NAME}-canary --replicas=0 -n ${NAMESPACE}

    log_info "✅ Déploiement Canary réussi!"
else
    log_warn "Canary maintenu à 100% sans promotion"
fi
```

**Utilisation** :

```bash
chmod +x canary-manual.sh
./canary-manual.sh v2.0.0
```

---

## Implémentation automatisée avec Flagger

### Présentation de Flagger

**Flagger** est un outil de Weaveworks qui automatise complètement les déploiements Canary, incluant :

- Progression automatique du trafic
- Analyse des métriques Prometheus
- Rollback automatique en cas de problème
- Tests de charge automatiques
- Notifications

**Architecture** :

```
┌────────────────────────────────────────────────────────┐
│                    Flagger Controller                  │
│  ┌────────────┐  ┌──────────┐  ┌────────────────────┐  │
│  │  Canary    │→ │ Analysis │→ │  Promotion/        │  │
│  │  Detection │  │  Engine  │  │  Rollback          │  │
│  └────────────┘  └──────────┘  └────────────────────┘  │
└──────┬──────────────────┬────────────────────┬─────────┘
       │                  │                    │
       ▼                  ▼                    ▼
  Deployment         Prometheus           Service/Ingress
```

### Installation de Flagger

```bash
# Ajouter le repo Helm
helm repo add flagger https://flagger.app

# Installer Flagger avec support Prometheus
kubectl create namespace flagger-system

helm upgrade -i flagger flagger/flagger \
    --namespace flagger-system \
    --set meshProvider=kubernetes \
    --set metricsServer=http://prometheus.monitoring:9090

# Installer Grafana (optionnel mais recommandé)
helm upgrade -i flagger-grafana flagger/grafana \
    --namespace flagger-system \
    --set url=http://prometheus.monitoring:9090

# Installer le load tester (pour tests automatiques)
kubectl apply -k https://github.com/fluxcd/flagger//kustomize/tester?ref=main
```

### Configuration d'un Canary avec Flagger

#### Ressource Canary

```yaml
---
# canary.yaml
apiVersion: flagger.app/v1beta1
kind: Canary
metadata:
  name: mon-app
  namespace: production
spec:
  # Deployment cible à gérer
  targetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: mon-app

  # Service Kubernetes
  service:
    port: 80
    targetPort: 8080
    name: mon-app
    portDiscovery: true

  # Configuration de l'analyse
  analysis:
    # Intervalle entre chaque étape
    interval: 1m

    # Seuil : nombre d'échecs avant rollback
    threshold: 5

    # Nombre maximum d'itérations
    maxWeight: 50

    # Poids à ajouter à chaque itération
    stepWeight: 10

    # Métriques Prometheus à surveiller
    metrics:

    # Métrique 1 : Taux de succès des requêtes
    - name: request-success-rate
      # Template prédéfini par Flagger
      templateRef:
        name: request-success-rate
        namespace: flagger-system
      # Seuil minimum acceptable
      thresholdRange:
        min: 99
      interval: 1m

    # Métrique 2 : Durée des requêtes
    - name: request-duration
      templateRef:
        name: request-duration
        namespace: flagger-system
      # Seuil maximum acceptable (en ms)
      thresholdRange:
        max: 500
      interval: 1m

    # Métrique personnalisée 3 : Taux d'erreur
    - name: error-rate
      query: |
        100 - (
          sum(
            rate(
              http_requests_total{
                namespace="production",
                deployment="mon-app-canary",
                status!~"5.."
              }[1m]
            )
          )
          /
          sum(
            rate(
              http_requests_total{
                namespace="production",
                deployment="mon-app-canary"
              }[1m]
            )
          )
          * 100
        )
      thresholdRange:
        max: 1
      interval: 1m

    # Webhooks pour tests automatiques
    webhooks:

    # Pre-rollout : Tests avant de commencer
    - name: pre-rollout-tests
      type: pre-rollout
      url: http://flagger-loadtester.flagger-system/
      timeout: 15s
      metadata:
        type: bash
        cmd: |
          curl -sd 'test' http://mon-app-canary.production:80/ | grep -q OK

    # Rollout : Tests de charge pendant le déploiement
    - name: load-test
      type: rollout
      url: http://flagger-loadtester.flagger-system/
      timeout: 5s
      metadata:
        type: cmd
        cmd: "hey -z 1m -q 10 -c 2 http://mon-app-canary.production:80/"

    # Post-rollout : Tests après succès
    - name: post-rollout-tests
      type: post-rollout
      url: http://flagger-loadtester.flagger-system/
      timeout: 15s
      metadata:
        type: bash
        cmd: |
          curl -sd 'test' http://mon-app.production:80/version | grep -q v2.0.0

    # Alertes en cas de problème
    - name: send-alert
      type: event
      url: http://event-receiver.monitoring/
      metadata:
        alerts:
          - name: canary-failed
            severity: error
            message: "Canary deployment failed"
```

#### Templates de métriques personnalisés

```yaml
---
# metric-templates.yaml
apiVersion: flagger.app/v1beta1
kind: MetricTemplate
metadata:
  name: request-success-rate
  namespace: flagger-system
spec:
  provider:
    type: prometheus
    address: http://prometheus.monitoring:9090
  query: |
    sum(
      rate(
        http_requests_total{
          namespace="{{ namespace }}",
          deployment="{{ target }}",
          status!~"5.."
        }[{{ interval }}]
      )
    )
    /
    sum(
      rate(
        http_requests_total{
          namespace="{{ namespace }}",
          deployment="{{ target }}"
        }[{{ interval }}]
      )
    )
    * 100
---
apiVersion: flagger.app/v1beta1
kind: MetricTemplate
metadata:
  name: request-duration
  namespace: flagger-system
spec:
  provider:
    type: prometheus
    address: http://prometheus.monitoring:9090
  query: |
    histogram_quantile(
      0.99,
      sum(
        rate(
          http_request_duration_seconds_bucket{
            namespace="{{ namespace }}",
            deployment="{{ target }}"
          }[{{ interval }}]
        )
      ) by (le)
    )
    * 1000
---
apiVersion: flagger.app/v1beta1
kind: MetricTemplate
metadata:
  name: cpu-usage
  namespace: flagger-system
spec:
  provider:
    type: prometheus
    address: http://prometheus.monitoring:9090
  query: |
    sum(
      rate(
        container_cpu_usage_seconds_total{
          namespace="{{ namespace }}",
          pod=~"{{ target }}-.*"
        }[{{ interval }}]
      )
    )
```

### Déclenchement d'un déploiement Canary

Avec Flagger, déclencher un Canary est **extrêmement simple** :

```bash
# Il suffit de mettre à jour l'image du Deployment
kubectl set image deployment/mon-app \
    app=localhost:32000/mon-app:v2.0.0 \
    -n production

# Flagger détecte automatiquement le changement et :
# 1. Crée un déploiement canary
# 2. Commence la progression automatique
# 3. Analyse les métriques à chaque étape
# 4. Promeut ou rollback automatiquement
```

**Observer le processus** :

```bash
# Voir le statut du Canary
kubectl get canary -n production -w

# Exemple de sortie :
# NAME      STATUS        WEIGHT   LASTTRANSITIONTIME
# mon-app   Progressing   0        2024-01-15T10:00:00Z
# mon-app   Progressing   10       2024-01-15T10:01:00Z
# mon-app   Progressing   20       2024-01-15T10:02:00Z
# mon-app   Progressing   30       2024-01-15T10:03:00Z
# mon-app   Progressing   40       2024-01-15T10:04:00Z
# mon-app   Progressing   50       2024-01-15T10:05:00Z
# mon-app   Promoting     50       2024-01-15T10:06:00Z
# mon-app   Succeeded     0        2024-01-15T10:07:00Z

# Voir les events détaillés
kubectl describe canary mon-app -n production

# Logs du controller Flagger
kubectl logs -n flagger-system deployment/flagger -f
```

### Timeline d'un déploiement Canary avec Flagger

```
┌─────────────────────────────────────────────────────────────┐
│ T+0min : Détection de la nouvelle version                   │
│   - Flagger détecte le changement d'image                   │
│   - Crée un nouveau ReplicaSet canary                       │
│   - Status: Initializing                                    │
└─────────────────────────────────────────────────────────────┘
                           ↓
┌─────────────────────────────────────────────────────────────┐
│ T+1min : Pre-rollout tests                                  │
│   - Exécute les webhooks pre-rollout                        │
│   - Si échec → ABORT                                        │
│   - Status: Initialized                                     │
└─────────────────────────────────────────────────────────────┘
                           ↓
┌─────────────────────────────────────────────────────────────┐
│ T+2min : Canary 10%                                         │
│   - Route 10% du trafic vers canary                         │
│   - Lance les load tests                                    │
│   - Analyse les métriques pendant 1min                      │
│   - Si échec > threshold → ROLLBACK                         │
│   - Status: Progressing (weight: 10)                        │
└─────────────────────────────────────────────────────────────┘
                           ↓
┌─────────────────────────────────────────────────────────────┐
│ T+3min : Canary 20%                                         │
│   - Route 20% du trafic vers canary                         │
│   - Analyse continue                                        │
│   - Status: Progressing (weight: 20)                        │
└─────────────────────────────────────────────────────────────┘
                           ↓
┌─────────────────────────────────────────────────────────────┐
│ ... Continue jusqu'à maxWeight (50%) ...                    │
└─────────────────────────────────────────────────────────────┘
                           ↓
┌─────────────────────────────────────────────────────────────┐
│ T+6min : Canary 50%                                         │
│   - Route 50% du trafic vers canary                         │
│   - Dernière validation des métriques                       │
│   - Status: Progressing (weight: 50)                        │
└─────────────────────────────────────────────────────────────┘
                           ↓
┌─────────────────────────────────────────────────────────────┐
│ T+7min : Promotion                                          │
│   - Remplace la version stable par canary                   │
│   - Met à jour le primary deployment                        │
│   - Status: Promoting                                       │
└─────────────────────────────────────────────────────────────┘
                           ↓
┌─────────────────────────────────────────────────────────────┐
│ T+8min : Post-rollout tests                                 │
│   - Exécute les tests post-rollout                          │
│   - Finalisation                                            │
│   - Status: Succeeded                                       │
└─────────────────────────────────────────────────────────────┘
```

### Rollback automatique

Si les métriques dépassent les seuils, Flagger effectue un **rollback automatique** :

```
T+0min : Canary 10%
         Métriques OK
         ↓
T+1min : Canary 20%
         Métriques OK
         ↓
T+2min : Canary 30%
         ❌ Error rate: 5% (seuil: 1%)
         ❌ Échec 1/5
         ↓
T+3min : Canary 30% (retry)
         ❌ Error rate: 6%
         ❌ Échec 2/5
         ↓
T+4min : Canary 30% (retry)
         ❌ Error rate: 7%
         ❌ Échec 3/5
         ↓
         ... continues jusqu'à 5 échecs ...
         ↓
T+7min : 🚨 ROLLBACK AUTOMATIQUE
         - Trafic retourné à 100% vers stable
         - Canary supprimé
         - Status: Failed
         - Alertes envoyées
```

---

## Métriques et observabilité

### Métriques essentielles

Pour un Canary réussi, vous devez surveiller ces métriques clés :

#### 1. Taux de succès des requêtes

```promql
# Pourcentage de requêtes réussies (non 5xx)
sum(rate(http_requests_total{status!~"5..", deployment="mon-app-canary"}[1m]))
/
sum(rate(http_requests_total{deployment="mon-app-canary"}[1m]))
* 100
```

**Seuil recommandé** : > 99%

#### 2. Latence P95/P99

```promql
# 95ème percentile de latence
histogram_quantile(0.95,
  sum(rate(http_request_duration_seconds_bucket{deployment="mon-app-canary"}[1m]))
  by (le)
)
```

**Seuil recommandé** : < 500ms

#### 3. Taux d'erreur

```promql
# Pourcentage d'erreurs
sum(rate(http_requests_total{status=~"5..", deployment="mon-app-canary"}[1m]))
/
sum(rate(http_requests_total{deployment="mon-app-canary"}[1m]))
* 100
```

**Seuil recommandé** : < 1%

#### 4. Utilisation des ressources

```promql
# CPU usage
sum(rate(container_cpu_usage_seconds_total{pod=~"mon-app-canary-.*"}[1m]))

# Memory usage
sum(container_memory_working_set_bytes{pod=~"mon-app-canary-.*"})
```

**Seuil recommandé** : Pas plus de 20% d'augmentation par rapport à stable

### Dashboard Grafana pour Canary

```json
{
  "dashboard": {
    "title": "Canary Deployment Monitoring",
    "rows": [
      {
        "panels": [
          {
            "title": "Canary Status",
            "targets": [
              {
                "expr": "flagger_canary_status{name='mon-app'}",
                "legendFormat": "Status: {{ status }}"
              }
            ]
          },
          {
            "title": "Traffic Weight",
            "targets": [
              {
                "expr": "flagger_canary_weight{name='mon-app'}",
                "legendFormat": "Canary Weight"
              }
            ]
          }
        ]
      },
      {
        "panels": [
          {
            "title": "Request Success Rate",
            "targets": [
              {
                "expr": "sum(rate(http_requests_total{status!~'5..',deployment='mon-app-canary'}[1m])) / sum(rate(http_requests_total{deployment='mon-app-canary'}[1m])) * 100",
                "legendFormat": "Canary"
              },
              {
                "expr": "sum(rate(http_requests_total{status!~'5..',deployment='mon-app-stable'}[1m])) / sum(rate(http_requests_total{deployment='mon-app-stable'}[1m])) * 100",
                "legendFormat": "Stable"
              }
            ],
            "alert": {
              "conditions": [
                {
                  "evaluator": {
                    "params": [99],
                    "type": "lt"
                  }
                }
              ]
            }
          },
          {
            "title": "Request Duration P95",
            "targets": [
              {
                "expr": "histogram_quantile(0.95, sum(rate(http_request_duration_seconds_bucket{deployment='mon-app-canary'}[1m])) by (le))",
                "legendFormat": "Canary P95"
              },
              {
                "expr": "histogram_quantile(0.95, sum(rate(http_request_duration_seconds_bucket{deployment='mon-app-stable'}[1m])) by (le))",
                "legendFormat": "Stable P95"
              }
            ]
          }
        ]
      },
      {
        "panels": [
          {
            "title": "Error Rate",
            "targets": [
              {
                "expr": "sum(rate(http_requests_total{status=~'5..',deployment='mon-app-canary'}[1m])) / sum(rate(http_requests_total{deployment='mon-app-canary'}[1m])) * 100",
                "legendFormat": "Canary Errors"
              },
              {
                "expr": "sum(rate(http_requests_total{status=~'5..',deployment='mon-app-stable'}[1m])) / sum(rate(http_requests_total{deployment='mon-app-stable'}[1m])) * 100",
                "legendFormat": "Stable Errors"
              }
            ]
          },
          {
            "title": "Request Rate",
            "targets": [
              {
                "expr": "sum(rate(http_requests_total{deployment='mon-app-canary'}[1m]))",
                "legendFormat": "Canary RPS"
              },
              {
                "expr": "sum(rate(http_requests_total{deployment='mon-app-stable'}[1m]))",
                "legendFormat": "Stable RPS"
              }
            ]
          }
        ]
      }
    ]
  }
}
```

### Alertes Prometheus

```yaml
# prometheus-alerts.yaml
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: canary-alerts
  namespace: monitoring
spec:
  groups:
  - name: canary-deployment
    interval: 30s
    rules:

    # Alerte si le Canary échoue
    - alert: CanaryDeploymentFailed
      expr: flagger_canary_status{status="failed"} == 1
      for: 1m
      labels:
        severity: critical
      annotations:
        summary: "Canary deployment failed for {{ $labels.name }}"
        description: "Canary deployment has failed and rolled back"

    # Alerte si le taux d'erreur est élevé sur le Canary
    - alert: CanaryHighErrorRate
      expr: |
        (
          sum(rate(http_requests_total{status=~"5..", track="canary"}[1m]))
          /
          sum(rate(http_requests_total{track="canary"}[1m]))
        ) > 0.01
      for: 2m
      labels:
        severity: warning
      annotations:
        summary: "High error rate on canary deployment"
        description: "Canary has {{ $value | humanizePercentage }} error rate"

    # Alerte si la latence est trop élevée
    - alert: CanaryHighLatency
      expr: |
        histogram_quantile(0.95,
          sum(rate(http_request_duration_seconds_bucket{track="canary"}[1m]))
          by (le)
        ) > 0.5
      for: 3m
      labels:
        severity: warning
      annotations:
        summary: "High latency on canary deployment"
        description: "Canary P95 latency is {{ $value }}s"

    # Alerte si le Canary progresse (informatif)
    - alert: CanaryProgressing
      expr: flagger_canary_status{status="progressing"} == 1
      labels:
        severity: info
      annotations:
        summary: "Canary deployment in progress for {{ $labels.name }}"
        description: "Current weight: {{ $labels.weight }}%"
```

---

## Patterns avancés

### Pattern 1 : Canary avec header-based routing

Router vers le Canary basé sur un header HTTP (pour tests ciblés).

```yaml
# Avec Istio
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: mon-app
spec:
  hosts:
  - mon-app.example.com
  http:
  # Route 1 : Header spécifique → Canary
  - match:
    - headers:
        x-canary:
          exact: "true"
    route:
    - destination:
        host: mon-app
        subset: canary

  # Route 2 : Utilisateurs premium → Canary
  - match:
    - headers:
        x-user-type:
          exact: "premium"
    route:
    - destination:
        host: mon-app
        subset: canary
      weight: 100

  # Route 3 : Traffic normal → Split
  - route:
    - destination:
        host: mon-app
        subset: stable
      weight: 90
    - destination:
        host: mon-app
        subset: canary
      weight: 10
```

**Usage** :

```bash
# Tester le Canary directement
curl -H "x-canary: true" https://app.example.com/

# Utilisateur normal (10% chance de canary)
curl https://app.example.com/
```

### Pattern 2 : Canary par région géographique

Déployer le Canary dans une région avant les autres.

```yaml
# Canary pour la région EU seulement
apiVersion: flagger.app/v1beta1
kind: Canary
metadata:
  name: mon-app-eu
  namespace: production
spec:
  targetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: mon-app-eu

  analysis:
    interval: 5m
    threshold: 3
    maxWeight: 50
    stepWeight: 25

    # Métriques filtrées par région
    metrics:
    - name: request-success-rate
      query: |
        sum(rate(http_requests_total{region="eu", status!~"5.."}[1m]))
        /
        sum(rate(http_requests_total{region="eu"}[1m]))
        * 100
      thresholdRange:
        min: 99
```

**Workflow** :

1. Déployer Canary en EU
2. Si succès → Déployer en US
3. Si succès → Déployer en ASIA

### Pattern 3 : Canary avec mirroring (Shadow traffic)

Combiner Canary avec traffic mirroring pour valider sans risque.

```yaml
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: mon-app
spec:
  hosts:
  - mon-app
  http:
  - route:
    - destination:
        host: mon-app
        subset: stable
      weight: 100
    mirror:
      host: mon-app
      subset: canary
    mirrorPercentage:
      value: 100.0
```

**Bénéfice** : Le Canary reçoit 100% du trafic en copie, mais les réponses sont ignorées. Validation des performances sans risque.

### Pattern 4 : Multi-stage Canary

Plusieurs phases avec des seuils différents.

```yaml
apiVersion: flagger.app/v1beta1
kind: Canary
metadata:
  name: mon-app
spec:
  analysis:
    interval: 1m
    threshold: 5

    # Phase 1 : 0-10% (seuils stricts)
    metrics:
    - name: early-stage-success-rate
      query: |
        sum(rate(http_requests_total{status!~"5.."}[1m]))
        / sum(rate(http_requests_total[1m])) * 100
      thresholdRange:
        min: 99.5  # Très strict au début
      interval: 1m

    # Phase 2 : 10-50% (seuils normaux)
    - name: late-stage-success-rate
      query: |
        sum(rate(http_requests_total{status!~"5.."}[5m]))
        / sum(rate(http_requests_total[5m])) * 100
      thresholdRange:
        min: 99  # Un peu plus tolérant
      interval: 1m

    stepWeightPromotion: 10  # Commencer avec 10%
    maxWeight: 50
    stepWeight: 10
```

---

## Intégration avec GitOps (ArgoCD + Flagger)

### Architecture GitOps Canary

```
┌──────────────┐
│     Git      │  Version source de vérité
│  Repository  │
└──────┬───────┘
       │
       │ git push
       │
┌──────▼───────┐
│   ArgoCD     │  Synchronise les manifestes
│              │
└──────┬───────┘
       │
       │ kubectl apply
       │
┌──────▼────────────┐
│   Kubernetes      │
│  ┌─────────────┐  │
│  │ Deployment  │  │  Détecte changement
│  │   (v2.0)    │  │         ↓
│  └─────────────┘  │  ┌──────────────┐
│                   │  │   Flagger    │
│                   │  │  Controller  │
│                   │  └──────────────┘
│                   │         ↓
│  ┌─────────────┐  │  Gère le Canary
│  │   Canary    │  │  automatiquement
│  │ Deployment  │  │
│  └─────────────┘  │
└───────────────────┘
```

### Configuration ArgoCD

```yaml
# argocd-application.yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: mon-app
  namespace: argocd
spec:
  project: default

  source:
    repoURL: https://github.com/mon-org/mon-app.git
    targetRevision: main
    path: k8s/

  destination:
    server: https://kubernetes.default.svc
    namespace: production

  syncPolicy:
    automated:
      prune: false  # Important : Ne pas supprimer le canary
      selfHeal: true
    syncOptions:
    - CreateNamespace=true

    # Ignorer les différences gérées par Flagger
    ignoreDifferences:
    - group: apps
      kind: Deployment
      jsonPointers:
      - /spec/replicas
      - /spec/template/spec/containers/0/image
```

### Workflow complet

```bash
# 1. Developer push une nouvelle version
git add deployment.yaml
git commit -m "Update to v2.0.0"
git push origin main

# 2. ArgoCD détecte le changement et synchronise
# (peut être automatique ou manuel selon config)

# 3. ArgoCD met à jour le Deployment avec la nouvelle image
kubectl set image deployment/mon-app app=mon-app:v2.0.0

# 4. Flagger détecte le changement
#    → Crée automatiquement le Canary
#    → Lance la progression automatique
#    → Analyse les métriques
#    → Promeut ou rollback

# 5. Monitoring dans ArgoCD
argocd app get mon-app

# 6. Monitoring dans Flagger
kubectl get canary mon-app -w
```

---

## Bonnes pratiques

### 1. Définir des SLIs/SLOs clairs

**SLI** (Service Level Indicator) : Métrique mesurable
**SLO** (Service Level Objective) : Objectif pour cette métrique

```yaml
# Exemple de SLOs pour un Canary
SLOs:
  - metric: availability
    threshold: 99.9%

  - metric: latency_p95
    threshold: 500ms

  - metric: error_rate
    threshold: 0.1%

  - metric: saturation_cpu
    threshold: 80%
```

### 2. Commencer conservateur

```yaml
# Première fois : Très conservateur
analysis:
  interval: 5m      # Long intervalle
  threshold: 3      # Peu tolérant
  maxWeight: 20     # Seulement 20%
  stepWeight: 5     # Petites étapes

# Après confiance : Plus agressif
analysis:
  interval: 1m
  threshold: 5
  maxWeight: 50
  stepWeight: 10
```

### 3. Tester les webhooks

```bash
# Tester manuellement les webhooks avant utilisation
kubectl run test-webhook --rm -it --image=curlimages/curl -- \
  curl -d '{"name":"test"}' \
  http://flagger-loadtester.flagger-system/
```

### 4. Documenter les décisions

```yaml
metadata:
  annotations:
    canary-strategy: "progressive-10-50"
    canary-reason: "Major algorithm refactoring"
    canary-started-by: "john.doe@example.com"
    canary-started-at: "2024-01-15T10:00:00Z"
```

### 5. Alerter l'équipe

```yaml
webhooks:
- name: notify-slack
  type: event
  url: https://hooks.slack.com/services/YOUR/WEBHOOK/URL
  metadata:
    type: slack
    channel: "#deployments"
    username: "Flagger"
```

### 6. Avoir un plan de rollback manuel

```bash
# Script de rollback d'urgence
#!/bin/bash
# rollback-canary.sh

kubectl patch canary mon-app -n production \
  --type=json \
  -p='[{"op": "replace", "path": "/spec/analysis/threshold", "value": 0}]'

# Force le rollback en mettant threshold à 0
```

### 7. Monitorer les coûts

Le Canary double temporairement les ressources :

```yaml
# Calculer le coût
# Stable: 10 pods × 0.5 CPU = 5 CPU
# Canary:  5 pods × 0.5 CPU = 2.5 CPU (à 50%)
# Total: 7.5 CPU pendant 30 minutes
# Surcoût: 50% pendant la durée du Canary
```

---

## Troubleshooting

### Problème 1 : Canary bloqué en "Initializing"

**Symptômes** :
```bash
kubectl get canary
# NAME      STATUS         WEIGHT
# mon-app   Initializing   0
```

**Causes possibles** :
- Service ou Deployment introuvable
- Métriques Prometheus non disponibles
- Webhooks qui échouent

**Diagnostic** :
```bash
kubectl describe canary mon-app
kubectl logs -n flagger-system deployment/flagger
```

**Solution** :
```bash
# Vérifier le service
kubectl get svc mon-app

# Vérifier Prometheus
kubectl run test --rm -it --image=curlimages/curl -- \
  curl http://prometheus.monitoring:9090/-/healthy

# Tester les webhooks
kubectl run test --rm -it --image=curlimages/curl -- \
  curl http://flagger-loadtester.flagger-system/
```

### Problème 2 : Rollback intempestif

**Symptômes** :
Le Canary rollback alors que tout semble OK.

**Causes** :
- Seuils trop stricts
- Métriques incorrectes
- Pas assez de trafic pour des métriques fiables

**Solution** :
```yaml
# Ajuster les seuils
analysis:
  threshold: 10  # Plus tolérant

  metrics:
  - name: request-success-rate
    thresholdRange:
      min: 95  # Moins strict (au lieu de 99)
```

### Problème 3 : Métriques non disponibles

**Symptômes** :
```
Halt advancement: no values found for metric request-success-rate
```

**Solution** :
```bash
# Vérifier que l'app expose des métriques
kubectl port-forward deployment/mon-app-canary 8080:8080
curl http://localhost:8080/metrics

# Vérifier que Prometheus scrape
kubectl port-forward -n monitoring svc/prometheus 9090:9090
# Ouvrir http://localhost:9090 et chercher les métriques
```

### Problème 4 : Progression trop lente

**Symptômes** :
Le Canary prend des heures.

**Solution** :
```yaml
# Réduire l'intervalle et augmenter stepWeight
analysis:
  interval: 30s      # Au lieu de 1m
  stepWeight: 20     # Au lieu de 10
  maxWeight: 50
```

---

## Comparaison finale : Manuel vs Flagger

| Aspect | Manuel | Flagger |
|--------|--------|---------|
| **Setup initial** | Simple | Complexe |
| **Opération** | Manuelle | Automatique |
| **Analyse métriques** | Manuelle | Automatique |
| **Rollback** | Manuel | Automatique |
| **Tests** | Manuels | Automatiques |
| **Notifications** | Custom | Intégrées |
| **Courbe d'apprentissage** | Faible | Élevée |
| **Maintenance** | Élevée | Faible |
| **Fiabilité** | Dépend de l'opérateur | Très élevée |

**Recommandation** :
- **Manuel** : Apprentissage, POC, petites équipes
- **Flagger** : Production, équipes matures, multiples apps

---

## Récapitulatif

### Points clés

**Les déploiements Canary, c'est** :
- Exposition **progressive** de la nouvelle version
- Validation en **conditions réelles**
- **Rollback automatique** si problème détecté
- Réduction du **risque** (seul un % est impacté)

**Implémentation** :
1. **Manuelle** : Deux Deployments + scaling progressif
2. **Automatisée** : Flagger + Prometheus + métriques

**Métriques essentielles** :
- Taux de succès (> 99%)
- Latence P95/P99 (< 500ms)
- Taux d'erreur (< 1%)
- Utilisation ressources

**Progression typique** :
10% → 20% → 30% → 40% → 50% → 100%

**Durée** : 30 minutes à 2 heures selon configuration

### Avantages et limites

**✅ Avantages** :
- Risque minimal (impact limité)
- Validation en production réelle
- Rollback automatique possible
- Détection précoce des problèmes
- Confiance accrue

**❌ Limites** :
- Durée longue
- Complexité (surtout avec Flagger)
- Nécessite monitoring robuste
- Coût temporaire (ressources supplémentaires)
- Versions mixtes en production

### Quand utiliser Canary

**✅ Recommandé pour** :
- Nouvelle fonctionnalité majeure
- Refonte d'algorithme
- Changement à risque élevé
- Production avec fort trafic
- Besoin de validation réelle

**❌ Non recommandé pour** :
- Hotfix urgent
- Correction de bug simple
- Migration incompatible
- Application à faible trafic
- Dev/Test

---

## Conclusion

Les déploiements Canary représentent l'une des stratégies les plus **sûres et sophistiquées** pour mettre à jour vos applications en production. En exposant progressivement la nouvelle version et en surveillant les métriques en temps réel, vous minimisez drastiquement le risque d'un déploiement catastrophique.

**Avec la méthode manuelle**, vous comprenez les fondamentaux et pouvez implémenter un Canary basique avec les outils natifs de Kubernetes.

**Avec Flagger**, vous automatisez complètement le processus, incluant l'analyse des métriques, les tests de charge, et le rollback automatique. C'est l'outil de choix pour les équipes qui déploient fréquemment en production.

**Votre progression DevOps** :

Vous maîtrisez maintenant toutes les stratégies de déploiement :

1. **Recreate** : Simple mais downtime
2. **Rolling Update** : Standard Kubernetes
3. **Blue-Green** : Rollback instantané
4. **Canary** : Validation progressive ✅

**Prochaine étape** : Vous avez terminé le chapitre 17 sur DevOps et CI/CD ! Vous disposez maintenant d'un workflow complet et professionnel pour déployer vos applications sur MicroK8s avec confiance.

Les chapitres suivants vous attendent pour découvrir d'autres aspects avancés de Kubernetes : gestion des ressources, sécurité, networking avancé, et bien plus !

**Félicitations pour ce parcours ! 🎉🚀**

⏭️ [Gestion des Ressources](/18-gestion-des-ressources/README.md)
