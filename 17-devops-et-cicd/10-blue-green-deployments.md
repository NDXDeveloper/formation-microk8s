🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 17.10 Blue-Green deployments

## Introduction

Dans la section précédente, nous avons découvert les différentes stratégies de déploiement et leurs principes généraux. Maintenant, nous allons approfondir la stratégie **Blue-Green**, une des plus puissantes pour les déploiements en production nécessitant un rollback instantané.

Le déploiement Blue-Green est particulièrement adapté aux **applications critiques** où quelques secondes de downtime sont inacceptables et où la capacité de revenir en arrière instantanément est primordiale. C'est une stratégie qui privilégie la **sécurité** au détriment du coût (doublement temporaire des ressources).

Dans ce chapitre, nous allons explorer en profondeur comment implémenter des déploiements Blue-Green sur MicroK8s, avec tous les détails pratiques, les pièges à éviter, et les techniques avancées pour gérer même les applications les plus complexes.

---

## Rappel du principe

### Concept fondamental

Le déploiement Blue-Green repose sur la maintenance de **deux environnements de production identiques** :

- **Blue (Bleu)** : L'environnement actuellement en production
- **Green (Vert)** : L'environnement de préparation pour la nouvelle version

**Le processus** :

```
┌─────────────────────────────────────────────────────────────┐
│  ÉTAPE 1 : État initial                                     │
│                                                             │
│  ┌──────────┐                                               │
│  │  Blue    │  ◄────── 100% du trafic                       │
│  │  (v1.0)  │                                               │
│  └──────────┘                                               │
│                                                             │
│  ┌──────────┐                                               │
│  │  Green   │  (Idle ou ancienne version)                   │
│  │          │                                               │
│  └──────────┘                                               │
└─────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────┐
│  ÉTAPE 2 : Déploiement sur Green                            │
│                                                             │
│  ┌──────────┐                                               │
│  │  Blue    │  ◄────── 100% du trafic                       │
│  │  (v1.0)  │                                               │
│  └──────────┘                                               │
│                                                             │
│  ┌──────────┐                                               │
│  │  Green   │  ◄────── Déploiement v2.0                     │
│  │  (v2.0)  │          Tests, warm-up                       │
│  └──────────┘                                               │
└─────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────┐
│  ÉTAPE 3 : Bascule instantanée                              │
│                                                             │
│  ┌──────────┐                                               │
│  │  Blue    │  (Idle, disponible pour rollback)             │
│  │  (v1.0)  │                                               │
│  └──────────┘                                               │
│                                                             │
│  ┌──────────┐                                               │
│  │  Green   │  ◄────── 100% du trafic                       │
│  │  (v2.0)  │                                               │
│  └──────────┘                                               │
└─────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────┐
│  ÉTAPE 4 : Cleanup (après validation)                       │
│                                                             │
│  ┌──────────┐                                               │
│  │  Blue    │  ◄────── Devient le nouveau Green             │
│  │  (v2.0)  │          Prêt pour v3.0                       │
│  └──────────┘                                               │
│                                                             │
│  ┌──────────┐                                               │
│  │  Green   │  ◄────── 100% du trafic                       │
│  │  (v2.0)  │                                               │
│  └──────────┘                                               │
└─────────────────────────────────────────────────────────────┘
```

### Avantages spécifiques

**Rollback instantané** : En cas de problème, basculer de Green vers Blue prend **quelques secondes** (le temps de modifier le routage).

**Tests en conditions réelles** : Green peut être testé exhaustivement avant bascule, avec la même configuration que la production.

**Zero-downtime garanti** : Le trafic est simplement redirigé, pas de pods qui démarrent ou s'arrêtent pendant la transition.

**Isolation complète** : Blue et Green sont totalement séparés, aucun risque d'impact mutuel.

---

## Implémentation détaillée sur Kubernetes

### Architecture des composants

Voici l'architecture complète d'un déploiement Blue-Green :

```
┌─────────────────────────────────────────────────────────────┐
│                        Ingress                              │
│                    (Point d'entrée)                         │
└────────────┬────────────────────────────────────────────────┘
             │
             │ Routage basé sur selector
             │
    ┌────────▼────────┐
    │     Service     │
    │   (Production)  │
    │  selector: blue │  ◄── Pointe vers Blue ou Green
    └────────┬────────┘
             │
       ┌─────┴─────┐
       │           │
┌──────▼──────┐  ┌─▼──────────┐
│ Deployment  │  │ Deployment │
│   Blue      │  │   Green    │
│   (v1.0)    │  │   (v2.0)   │
│             │  │            │
│ ┌─────────┐ │  │ ┌─────────┐│
│ │  Pod    │ │  │ │  Pod    ││
│ │  Pod    │ │  │ │  Pod    ││
│ │  Pod    │ │  │ │  Pod    ││
│ └─────────┘ │  │ └─────────┘│
└─────────────┘  └────────────┘
```

### Manifestes Kubernetes complets

#### 1. Deployments Blue et Green

```yaml
---
# deployment-blue.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mon-app-blue
  namespace: production
  labels:
    app: mon-app
    environment: blue
  annotations:
    deployment.kubernetes.io/revision: "1"
    description: "Blue environment - Stable version"
spec:
  replicas: 3
  selector:
    matchLabels:
      app: mon-app
      environment: blue
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
  template:
    metadata:
      labels:
        app: mon-app
        environment: blue
        version: v1.0.0
      annotations:
        prometheus.io/scrape: "true"
        prometheus.io/port: "8080"
    spec:
      containers:
      - name: app
        image: localhost:32000/mon-app:v1.0.0
        imagePullPolicy: IfNotPresent
        ports:
        - name: http
          containerPort: 8080
          protocol: TCP
        env:
        - name: ENVIRONMENT
          value: "blue"
        - name: VERSION
          value: "v1.0.0"
        - name: APP_COLOR
          value: "blue"
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
          timeoutSeconds: 5
          failureThreshold: 3
        readinessProbe:
          httpGet:
            path: /ready
            port: 8080
          initialDelaySeconds: 5
          periodSeconds: 5
          timeoutSeconds: 3
          successThreshold: 1
          failureThreshold: 3
        lifecycle:
          preStop:
            exec:
              command: ["/bin/sh", "-c", "sleep 15"]
---
# deployment-green.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mon-app-green
  namespace: production
  labels:
    app: mon-app
    environment: green
  annotations:
    deployment.kubernetes.io/revision: "1"
    description: "Green environment - New version staging"
spec:
  replicas: 3
  selector:
    matchLabels:
      app: mon-app
      environment: green
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
  template:
    metadata:
      labels:
        app: mon-app
        environment: green
        version: v2.0.0
      annotations:
        prometheus.io/scrape: "true"
        prometheus.io/port: "8080"
    spec:
      containers:
      - name: app
        image: localhost:32000/mon-app:v2.0.0
        imagePullPolicy: IfNotPresent
        ports:
        - name: http
          containerPort: 8080
          protocol: TCP
        env:
        - name: ENVIRONMENT
          value: "green"
        - name: VERSION
          value: "v2.0.0"
        - name: APP_COLOR
          value: "green"
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
          timeoutSeconds: 5
          failureThreshold: 3
        readinessProbe:
          httpGet:
            path: /ready
            port: 8080
          initialDelaySeconds: 5
          periodSeconds: 5
          timeoutSeconds: 3
          successThreshold: 1
          failureThreshold: 3
        lifecycle:
          preStop:
            exec:
              command: ["/bin/sh", "-c", "sleep 15"]
```

#### 2. Services

```yaml
---
# service-production.yaml
apiVersion: v1
kind: Service
metadata:
  name: mon-app
  namespace: production
  labels:
    app: mon-app
  annotations:
    description: "Production service - Routes to active environment"
spec:
  type: ClusterIP
  selector:
    app: mon-app
    environment: blue  # Initialement pointe vers blue
  ports:
  - name: http
    port: 80
    targetPort: 8080
    protocol: TCP
  sessionAffinity: ClientIP  # Important pour les applications stateful
  sessionAffinityConfig:
    clientIP:
      timeoutSeconds: 3600
---
# service-blue.yaml
apiVersion: v1
kind: Service
metadata:
  name: mon-app-blue
  namespace: production
  labels:
    app: mon-app
    environment: blue
spec:
  type: ClusterIP
  selector:
    app: mon-app
    environment: blue
  ports:
  - name: http
    port: 80
    targetPort: 8080
    protocol: TCP
---
# service-green.yaml
apiVersion: v1
kind: Service
metadata:
  name: mon-app-green
  namespace: production
  labels:
    app: mon-app
    environment: green
spec:
  type: ClusterIP
  selector:
    app: mon-app
    environment: green
  ports:
  - name: http
    port: 80
    targetPort: 8080
    protocol: TCP
```

#### 3. Ingress

```yaml
---
# ingress-production.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: mon-app
  namespace: production
  annotations:
    cert-manager.io/cluster-issuer: letsencrypt-prod
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
    nginx.ingress.kubernetes.io/force-ssl-redirect: "true"
spec:
  ingressClassName: nginx
  tls:
  - hosts:
    - app.example.com
    secretName: mon-app-tls
  rules:
  - host: app.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: mon-app  # Service principal
            port:
              number: 80
---
# ingress-blue-testing.yaml (pour tests directs)
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: mon-app-blue-test
  namespace: production
  annotations:
    cert-manager.io/cluster-issuer: letsencrypt-prod
spec:
  ingressClassName: nginx
  tls:
  - hosts:
    - app-blue.example.com
    secretName: mon-app-blue-tls
  rules:
  - host: app-blue.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: mon-app-blue
            port:
              number: 80
---
# ingress-green-testing.yaml (pour tests directs)
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: mon-app-green-test
  namespace: production
  annotations:
    cert-manager.io/cluster-issuer: letsencrypt-prod
spec:
  ingressClassName: nginx
  tls:
  - hosts:
    - app-green.example.com
    secretName: mon-app-green-tls
  rules:
  - host: app-green.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: mon-app-green
            port:
              number: 80
```

---

## Script de déploiement complet

### Script bash avancé

```bash
#!/bin/bash
# blue-green-deploy.sh - Script complet de déploiement Blue-Green

set -euo pipefail

# Configuration
APP_NAME="${APP_NAME:-mon-app}"
NAMESPACE="${NAMESPACE:-production}"
NEW_VERSION="${1:-}"
DRY_RUN="${DRY_RUN:-false}"
SMOKE_TEST_TIMEOUT="${SMOKE_TEST_TIMEOUT:-300}"
WARM_UP_TIME="${WARM_UP_TIME:-60}"

# Couleurs pour l'affichage
RED='\033[0;31m'
GREEN='\033[0;32m'
YELLOW='\033[1;33m'
BLUE='\033[0;34m'
NC='\033[0m' # No Color

# Fonctions utilitaires
log_info() {
    echo -e "${BLUE}[INFO]${NC} $1"
}

log_success() {
    echo -e "${GREEN}[SUCCESS]${NC} $1"
}

log_warning() {
    echo -e "${YELLOW}[WARNING]${NC} $1"
}

log_error() {
    echo -e "${RED}[ERROR]${NC} $1"
}

# Validation des arguments
if [ -z "$NEW_VERSION" ]; then
    log_error "Usage: $0 <version>"
    log_error "Exemple: $0 v2.0.0"
    exit 1
fi

# Détection de l'environnement actif
get_active_environment() {
    kubectl get service ${APP_NAME} -n ${NAMESPACE} \
        -o jsonpath='{.spec.selector.environment}' 2>/dev/null || echo "unknown"
}

# Déterminer l'environnement cible
ACTIVE_ENV=$(get_active_environment)

if [ "$ACTIVE_ENV" = "blue" ]; then
    TARGET_ENV="green"
    CURRENT_ENV="blue"
elif [ "$ACTIVE_ENV" = "green" ]; then
    TARGET_ENV="blue"
    CURRENT_ENV="green"
else
    log_error "Unable to determine active environment"
    log_info "Current service selector: $(kubectl get service ${APP_NAME} -n ${NAMESPACE} -o jsonpath='{.spec.selector}')"
    exit 1
fi

log_info "==================================="
log_info "Blue-Green Deployment"
log_info "==================================="
log_info "Application: ${APP_NAME}"
log_info "Namespace: ${NAMESPACE}"
log_info "New Version: ${NEW_VERSION}"
log_info "Active Environment: ${CURRENT_ENV}"
log_info "Target Environment: ${TARGET_ENV}"
log_info "Dry Run: ${DRY_RUN}"
log_info "==================================="

# Confirmation
if [ "$DRY_RUN" != "true" ]; then
    echo
    read -p "Continuer avec le déploiement? (yes/no): " -r
    echo
    if [[ ! $REPLY =~ ^[Yy][Ee][Ss]$ ]]; then
        log_warning "Déploiement annulé"
        exit 0
    fi
fi

# ÉTAPE 1 : Déploiement sur l'environnement cible
log_info "ÉTAPE 1/7: Déploiement de la version ${NEW_VERSION} sur ${TARGET_ENV}"

if [ "$DRY_RUN" != "true" ]; then
    kubectl set image deployment/${APP_NAME}-${TARGET_ENV} \
        app=localhost:32000/${APP_NAME}:${NEW_VERSION} \
        -n ${NAMESPACE}

    log_info "Attente du rollout..."
    kubectl rollout status deployment/${APP_NAME}-${TARGET_ENV} -n ${NAMESPACE} --timeout=5m
else
    log_info "[DRY RUN] Would deploy ${NEW_VERSION} to ${TARGET_ENV}"
fi

log_success "Déploiement sur ${TARGET_ENV} terminé"

# ÉTAPE 2 : Vérification de la santé
log_info "ÉTAPE 2/7: Vérification de la santé des pods ${TARGET_ENV}"

if [ "$DRY_RUN" != "true" ]; then
    READY_PODS=$(kubectl get deployment ${APP_NAME}-${TARGET_ENV} -n ${NAMESPACE} \
        -o jsonpath='{.status.readyReplicas}')
    DESIRED_PODS=$(kubectl get deployment ${APP_NAME}-${TARGET_ENV} -n ${NAMESPACE} \
        -o jsonpath='{.spec.replicas}')

    if [ "$READY_PODS" != "$DESIRED_PODS" ]; then
        log_error "Tous les pods ne sont pas prêts: ${READY_PODS}/${DESIRED_PODS}"
        exit 1
    fi

    log_success "Tous les pods sont prêts: ${READY_PODS}/${DESIRED_PODS}"
else
    log_info "[DRY RUN] Would check pod health"
fi

# ÉTAPE 3 : Warm-up
log_info "ÉTAPE 3/7: Warm-up de l'environnement ${TARGET_ENV} (${WARM_UP_TIME}s)"

if [ "$DRY_RUN" != "true" ]; then
    sleep ${WARM_UP_TIME}
    log_success "Warm-up terminé"
else
    log_info "[DRY RUN] Would wait ${WARM_UP_TIME} seconds for warm-up"
fi

# ÉTAPE 4 : Smoke tests
log_info "ÉTAPE 4/7: Exécution des smoke tests sur ${TARGET_ENV}"

if [ "$DRY_RUN" != "true" ]; then
    # Test via le service dédié
    POD_NAME=$(kubectl get pod -l environment=${TARGET_ENV} -n ${NAMESPACE} \
        -o jsonpath='{.items[0].metadata.name}')

    # Health check
    log_info "Test 1/3: Health check..."
    kubectl exec ${POD_NAME} -n ${NAMESPACE} -- wget -qO- http://localhost:8080/health > /dev/null
    log_success "Health check OK"

    # Ready check
    log_info "Test 2/3: Readiness check..."
    kubectl exec ${POD_NAME} -n ${NAMESPACE} -- wget -qO- http://localhost:8080/ready > /dev/null
    log_success "Readiness check OK"

    # Version check
    log_info "Test 3/3: Version check..."
    VERSION_RESPONSE=$(kubectl exec ${POD_NAME} -n ${NAMESPACE} -- wget -qO- http://localhost:8080/version)
    if [[ "$VERSION_RESPONSE" == *"${NEW_VERSION}"* ]]; then
        log_success "Version correcte: ${NEW_VERSION}"
    else
        log_error "Version incorrecte: ${VERSION_RESPONSE}"
        exit 1
    fi
else
    log_info "[DRY RUN] Would run smoke tests"
fi

log_success "Tous les smoke tests réussis"

# ÉTAPE 5 : Bascule du trafic
log_info "ÉTAPE 5/7: Bascule du trafic vers ${TARGET_ENV}"

if [ "$DRY_RUN" != "true" ]; then
    # Sauvegarder la configuration actuelle pour rollback
    kubectl get service ${APP_NAME} -n ${NAMESPACE} -o yaml > /tmp/${APP_NAME}-service-backup.yaml

    # Basculer
    kubectl patch service ${APP_NAME} -n ${NAMESPACE} \
        -p "{\"spec\":{\"selector\":{\"environment\":\"${TARGET_ENV}\"}}}"

    log_success "Trafic basculé vers ${TARGET_ENV}"
else
    log_info "[DRY RUN] Would switch traffic to ${TARGET_ENV}"
fi

# ÉTAPE 6 : Validation post-bascule
log_info "ÉTAPE 6/7: Validation post-bascule"

if [ "$DRY_RUN" != "true" ]; then
    sleep 10

    # Test via l'Ingress public
    log_info "Test de l'endpoint public..."
    RESPONSE=$(curl -s -o /dev/null -w "%{http_code}" https://app.example.com/health)

    if [ "$RESPONSE" = "200" ]; then
        log_success "Endpoint public OK (HTTP ${RESPONSE})"
    else
        log_error "Endpoint public KO (HTTP ${RESPONSE})"
        log_warning "Rollback automatique..."

        # Rollback
        kubectl patch service ${APP_NAME} -n ${NAMESPACE} \
            -p "{\"spec\":{\"selector\":{\"environment\":\"${CURRENT_ENV}\"}}}"

        log_error "Déploiement échoué et rollback effectué"
        exit 1
    fi
else
    log_info "[DRY RUN] Would validate public endpoint"
fi

# ÉTAPE 7 : Nettoyage (optionnel)
log_info "ÉTAPE 7/7: Nettoyage de l'ancien environnement ${CURRENT_ENV}"

if [ "$DRY_RUN" != "true" ]; then
    read -p "Voulez-vous réduire les ressources de ${CURRENT_ENV}? (yes/no): " -r
    echo
    if [[ $REPLY =~ ^[Yy][Ee][Ss]$ ]]; then
        # Réduire à 1 replica au lieu de supprimer complètement
        kubectl scale deployment/${APP_NAME}-${CURRENT_ENV} --replicas=1 -n ${NAMESPACE}
        log_success "Environnement ${CURRENT_ENV} réduit à 1 replica (disponible pour rollback)"
    else
        log_info "Environnement ${CURRENT_ENV} conservé tel quel"
    fi
else
    log_info "[DRY RUN] Would offer to scale down ${CURRENT_ENV}"
fi

# Résumé final
echo
log_success "========================================="
log_success "DÉPLOIEMENT BLUE-GREEN RÉUSSI!"
log_success "========================================="
log_info "Application: ${APP_NAME}"
log_info "Version déployée: ${NEW_VERSION}"
log_info "Environnement actif: ${TARGET_ENV}"
log_info "Environnement standby: ${CURRENT_ENV}"
log_info ""
log_info "Commandes utiles:"
log_info "  - Voir les pods: kubectl get pods -n ${NAMESPACE} -l app=${APP_NAME}"
log_info "  - Voir les services: kubectl get svc -n ${NAMESPACE}"
log_info "  - Rollback: kubectl patch service ${APP_NAME} -n ${NAMESPACE} -p '{\"spec\":{\"selector\":{\"environment\":\"${CURRENT_ENV}\"}}}'"
log_success "========================================="
```

### Utilisation du script

```bash
# Rendre le script exécutable
chmod +x blue-green-deploy.sh

# Dry run (test sans déploiement réel)
DRY_RUN=true ./blue-green-deploy.sh v2.0.0

# Déploiement réel
./blue-green-deploy.sh v2.0.0

# Avec paramètres personnalisés
APP_NAME=mon-app NAMESPACE=production WARM_UP_TIME=120 ./blue-green-deploy.sh v2.1.0
```

---

## Gestion des états et données

### Problème des applications stateful

Les applications avec état (bases de données, sessions utilisateur, caches) posent des défis spécifiques pour Blue-Green.

#### Challenge 1 : Synchronisation des bases de données

**Problème** : Blue et Green peuvent avoir des versions de schéma différentes.

**Solutions** :

**Option A : Migrations backward-compatible**

```sql
-- Migration v2.0: Ajouter une colonne (compatible v1.0)
ALTER TABLE users ADD COLUMN phone VARCHAR(20) NULL;

-- v1.0 peut toujours fonctionner (colonne NULL)
-- v2.0 utilise la nouvelle colonne
```

**Option B : Base de données partagée avec versions**

```yaml
# Schéma versionné
apiVersion: v1
kind: ConfigMap
metadata:
  name: db-schema-version
data:
  min-version: "1.0"
  max-version: "2.0"
```

**Option C : Bases de données séparées**

```
Blue → Database Blue (v1 schema)
Green → Database Green (v2 schema)

Après bascule:
- Migrer les données si nécessaire
- Ou garder les deux en sync
```

#### Challenge 2 : Sessions utilisateur

**Problème** : Les sessions peuvent être perdues lors de la bascule.

**Solutions** :

**Option A : Session affinity (sticky sessions)**

```yaml
# Dans le Service
spec:
  sessionAffinity: ClientIP
  sessionAffinityConfig:
    clientIP:
      timeoutSeconds: 10800  # 3 heures
```

**Option B : Sessions externes (Redis)**

```yaml
env:
- name: SESSION_STORE
  value: "redis"
- name: REDIS_HOST
  value: "redis.production.svc.cluster.local"
```

Les sessions sont partagées entre Blue et Green.

**Option C : JWT/Tokens stateless**

Pas de sessions serveur, tout est dans le token côté client.

#### Challenge 3 : Caches

**Problème** : Les caches peuvent être désynchronisés.

**Solutions** :

**Option A : Cache partagé**

```yaml
# Blue et Green utilisent le même Redis
env:
- name: CACHE_BACKEND
  value: "redis://redis.production.svc.cluster.local:6379"
```

**Option B : Warm-up du cache**

```bash
# Script de warm-up avant bascule
#!/bin/bash
GREEN_POD=$(kubectl get pod -l environment=green -o jsonpath='{.items[0].metadata.name}')

# Appeler les endpoints pour peupler le cache
kubectl exec $GREEN_POD -- curl -X POST http://localhost:8080/admin/cache/warm-up
```

**Option C : Invalidation à la bascule**

```bash
# Vider les caches lors de la bascule
kubectl exec -it $GREEN_POD -- redis-cli FLUSHDB
```

---

## Tests et validation

### Suite de tests complète

```bash
#!/bin/bash
# test-suite.sh - Suite de tests pour Blue-Green

ENVIRONMENT=$1  # blue ou green
NAMESPACE=${2:-production}
APP_NAME=${3:-mon-app}

echo "🧪 Suite de tests pour ${ENVIRONMENT}"

# Configuration
SERVICE_NAME="${APP_NAME}-${ENVIRONMENT}"
BASE_URL="http://${SERVICE_NAME}.${NAMESPACE}.svc.cluster.local"

# Compteurs
PASSED=0
FAILED=0

# Fonction de test
run_test() {
    local test_name=$1
    local test_command=$2

    echo -n "Test: ${test_name}... "

    if eval "$test_command" > /dev/null 2>&1; then
        echo "✅ PASS"
        ((PASSED++))
        return 0
    else
        echo "❌ FAIL"
        ((FAILED++))
        return 1
    fi
}

# Tests de santé
echo "━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━"
echo "Tests de santé"
echo "━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━"

run_test "Health endpoint" \
    "kubectl run test-pod --rm -i --restart=Never --image=curlimages/curl -n ${NAMESPACE} -- curl -f ${BASE_URL}/health"

run_test "Readiness endpoint" \
    "kubectl run test-pod --rm -i --restart=Never --image=curlimages/curl -n ${NAMESPACE} -- curl -f ${BASE_URL}/ready"

# Tests fonctionnels
echo "━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━"
echo "Tests fonctionnels"
echo "━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━"

run_test "Homepage loads" \
    "kubectl run test-pod --rm -i --restart=Never --image=curlimages/curl -n ${NAMESPACE} -- curl -f ${BASE_URL}/"

run_test "API responds" \
    "kubectl run test-pod --rm -i --restart=Never --image=curlimages/curl -n ${NAMESPACE} -- curl -f ${BASE_URL}/api/status"

run_test "Database connection" \
    "kubectl run test-pod --rm -i --restart=Never --image=curlimages/curl -n ${NAMESPACE} -- curl -f ${BASE_URL}/api/db-check"

# Tests de performance
echo "━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━"
echo "Tests de performance"
echo "━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━"

run_test "Response time < 500ms" \
    "kubectl run test-pod --rm -i --restart=Never --image=curlimages/curl -n ${NAMESPACE} -- curl -w '%{time_total}' -o /dev/null -s ${BASE_URL}/ | awk '{exit (\$1 > 0.5)}'"

# Tests de charge légers
echo "━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━"
echo "Tests de charge"
echo "━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━"

POD_NAME=$(kubectl get pod -l environment=${ENVIRONMENT} -n ${NAMESPACE} -o jsonpath='{.items[0].metadata.name}')

run_test "Handles 100 requests" \
    "kubectl exec ${POD_NAME} -n ${NAMESPACE} -- ab -n 100 -c 10 http://localhost:8080/ 2>&1 | grep -q 'Complete requests:      100'"

# Résumé
echo "━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━"
echo "Résumé"
echo "━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━"
echo "Tests réussis: ${PASSED}"
echo "Tests échoués: ${FAILED}"
echo "━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━"

if [ $FAILED -eq 0 ]; then
    echo "✅ Tous les tests sont passés!"
    exit 0
else
    echo "❌ Certains tests ont échoué"
    exit 1
fi
```

**Utilisation** :

```bash
# Tester l'environnement Green avant bascule
./test-suite.sh green production mon-app

# Si tous les tests passent, procéder à la bascule
```

---

## Intégration avec ArgoCD

### Application ArgoCD pour Blue-Green

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: mon-app-blue
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/mon-org/mon-app-manifests.git
    targetRevision: HEAD
    path: blue-green/blue
    helm:
      valueFiles:
        - values-blue.yaml
  destination:
    server: https://kubernetes.default.svc
    namespace: production
  syncPolicy:
    automated:
      prune: false  # Ne pas supprimer automatiquement
      selfHeal: true
---
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: mon-app-green
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/mon-org/mon-app-manifests.git
    targetRevision: HEAD
    path: blue-green/green
    helm:
      valueFiles:
        - values-green.yaml
  destination:
    server: https://kubernetes.default.svc
    namespace: production
  syncPolicy:
    automated:
      prune: false
      selfHeal: true
```

### Workflow avec ArgoCD

```bash
# 1. Mettre à jour la version Green dans Git
git checkout main
sed -i 's/image: mon-app:v1.0.0/image: mon-app:v2.0.0/' blue-green/green/values-green.yaml
git commit -am "Update green to v2.0.0"
git push

# 2. ArgoCD synchronise automatiquement
argocd app sync mon-app-green

# 3. Tester Green
./test-suite.sh green

# 4. Basculer manuellement le service
kubectl patch service mon-app -n production \
    -p '{"spec":{"selector":{"environment":"green"}}}'

# 5. Mettre à jour Git pour documenter la bascule
git tag -a blue-green-v2.0.0 -m "Switched to v2.0.0 on green"
git push --tags
```

---

## Helm Chart pour Blue-Green

### Structure du Chart

```
blue-green-chart/
├── Chart.yaml
├── values.yaml
├── values-blue.yaml
├── values-green.yaml
└── templates/
    ├── deployment-blue.yaml
    ├── deployment-green.yaml
    ├── service.yaml
    ├── service-blue.yaml
    ├── service-green.yaml
    └── ingress.yaml
```

### Valeurs configurables

```yaml
# values.yaml
app:
  name: mon-app
  namespace: production

activeEnvironment: blue  # blue ou green

blue:
  enabled: true
  replicas: 3
  image:
    repository: localhost:32000/mon-app
    tag: v1.0.0
  resources:
    requests:
      memory: 256Mi
      cpu: 250m
    limits:
      memory: 512Mi
      cpu: 500m

green:
  enabled: true
  replicas: 3
  image:
    repository: localhost:32000/mon-app
    tag: v2.0.0
  resources:
    requests:
      memory: 256Mi
      cpu: 250m
    limits:
      memory: 512Mi
      cpu: 500m

service:
  type: ClusterIP
  port: 80

ingress:
  enabled: true
  className: nginx
  hosts:
    - host: app.example.com
      paths:
        - path: /
          pathType: Prefix
  tls:
    - secretName: mon-app-tls
      hosts:
        - app.example.com
```

### Déploiement avec Helm

```bash
# Installation initiale (Blue actif)
helm install mon-app ./blue-green-chart

# Mise à jour de Green
helm upgrade mon-app ./blue-green-chart \
    --set green.image.tag=v2.0.0 \
    --reuse-values

# Bascule vers Green
helm upgrade mon-app ./blue-green-chart \
    --set activeEnvironment=green \
    --reuse-values

# Retour vers Blue (rollback)
helm upgrade mon-app ./blue-green-chart \
    --set activeEnvironment=blue \
    --reuse-values
```

---

## Monitoring et observabilité

### Métriques Prometheus

```yaml
# ServiceMonitor pour Prometheus
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: mon-app-blue-green
  namespace: production
spec:
  selector:
    matchLabels:
      app: mon-app
  endpoints:
  - port: http
    path: /metrics
    interval: 30s
    relabelings:
    - sourceLabels: [__meta_kubernetes_pod_label_environment]
      targetLabel: environment
    - sourceLabels: [__meta_kubernetes_pod_label_version]
      targetLabel: version
```

### Alertes spécifiques Blue-Green

```yaml
# prometheus-rules.yaml
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: blue-green-alerts
  namespace: monitoring
spec:
  groups:
  - name: blue-green
    interval: 30s
    rules:
    - alert: BlueGreenEnvironmentDown
      expr: up{job="mon-app"} == 0
      for: 2m
      labels:
        severity: critical
      annotations:
        summary: "Blue-Green environment {{ $labels.environment }} is down"
        description: "Environment {{ $labels.environment }} has been down for more than 2 minutes"

    - alert: BlueGreenHighErrorRate
      expr: |
        rate(http_requests_total{status=~"5.."}[5m])
        / rate(http_requests_total[5m]) > 0.05
      for: 3m
      labels:
        severity: warning
      annotations:
        summary: "High error rate on {{ $labels.environment }}"
        description: "Error rate is {{ $value | humanizePercentage }} on {{ $labels.environment }}"

    - alert: BlueGreenVersionMismatch
      expr: count(up{job="mon-app"}) by (version) > 1
      for: 15m
      labels:
        severity: info
      annotations:
        summary: "Multiple versions running"
        description: "Both Blue and Green environments are active (expected during deployment)"
```

### Dashboard Grafana

```json
{
  "dashboard": {
    "title": "Blue-Green Deployment",
    "panels": [
      {
        "title": "Active Environment",
        "targets": [
          {
            "expr": "up{job='mon-app'}",
            "legendFormat": "{{ environment }}"
          }
        ]
      },
      {
        "title": "Request Rate by Environment",
        "targets": [
          {
            "expr": "rate(http_requests_total[5m])",
            "legendFormat": "{{ environment }}"
          }
        ]
      },
      {
        "title": "Error Rate by Environment",
        "targets": [
          {
            "expr": "rate(http_requests_total{status=~'5..'}[5m]) / rate(http_requests_total[5m])",
            "legendFormat": "{{ environment }}"
          }
        ]
      },
      {
        "title": "Response Time P95",
        "targets": [
          {
            "expr": "histogram_quantile(0.95, rate(http_request_duration_seconds_bucket[5m]))",
            "legendFormat": "{{ environment }}"
          }
        ]
      }
    ]
  }
}
```

---

## Troubleshooting

### Problèmes courants

#### Problème 1 : Pods Green ne démarrent pas

**Symptômes** :
```bash
kubectl get pods -l environment=green
# NAME                            READY   STATUS             RESTARTS   AGE
# mon-app-green-xxx               0/1     CrashLoopBackOff   5          5m
```

**Diagnostic** :
```bash
# Voir les logs
kubectl logs mon-app-green-xxx

# Décrire le pod
kubectl describe pod mon-app-green-xxx

# Vérifier les events
kubectl get events --sort-by='.lastTimestamp'
```

**Solutions possibles** :
- Image incorrecte ou non disponible
- Configuration manquante (ConfigMap, Secret)
- Problèmes de ressources (CPU/Memory)
- Erreurs dans le code de l'application

#### Problème 2 : Service ne route pas vers Green

**Symptômes** :
```bash
# Service modifié mais trafic toujours vers Blue
kubectl patch service mon-app -p '{"spec":{"selector":{"environment":"green"}}}'
curl https://app.example.com/version
# Retourne toujours v1.0.0 (Blue)
```

**Diagnostic** :
```bash
# Vérifier le selector du service
kubectl get service mon-app -o yaml | grep -A2 selector

# Vérifier les endpoints
kubectl get endpoints mon-app

# Tester directement le service
kubectl run test --rm -it --image=curlimages/curl -- curl http://mon-app.production.svc.cluster.local/version
```

**Solutions** :
- Vérifier que les pods Green ont le bon label
- Attendre la propagation DNS/iptables (quelques secondes)
- Vérifier les readinessProbes

#### Problème 3 : Sessions utilisateur perdues

**Symptômes** :
Utilisateurs déconnectés après la bascule.

**Diagnostic** :
```bash
# Vérifier la configuration de session affinity
kubectl get service mon-app -o yaml | grep -A3 sessionAffinity
```

**Solutions** :
```yaml
# Activer session affinity
spec:
  sessionAffinity: ClientIP
  sessionAffinityConfig:
    clientIP:
      timeoutSeconds: 10800

# Ou utiliser un store externe (Redis)
```

#### Problème 4 : Performance dégradée sur Green

**Symptômes** :
Latence plus élevée sur Green.

**Diagnostic** :
```bash
# Comparer les métriques
kubectl top pods -l environment=green
kubectl top pods -l environment=blue

# Vérifier les resources limits
kubectl describe deployment mon-app-green | grep -A5 Limits
```

**Solutions** :
- Augmenter les resources
- Warm-up insuffisant (cache froid)
- Problème de configuration

---

## Récapitulatif

### Checklist de déploiement Blue-Green

**Avant le déploiement** :
- ☐ Version de l'application testée en CI/CD
- ☐ Migrations de base de données backward-compatible
- ☐ Environnement Green disponible
- ☐ Backup de la configuration actuelle
- ☐ Plan de rollback documenté
- ☐ Fenêtre de maintenance communiquée (si nécessaire)

**Pendant le déploiement** :
- ☐ Déployer sur Green
- ☐ Attendre que tous les pods soient Ready
- ☐ Warm-up (60-120 secondes)
- ☐ Smoke tests réussis
- ☐ Tests de charge légers OK
- ☐ Bascule du trafic
- ☐ Validation post-bascule immédiate

**Après le déploiement** :
- ☐ Monitoring actif pendant 1 heure
- ☐ Vérification des métriques business
- ☐ Pas d'augmentation d'erreurs
- ☐ Feedback utilisateurs positif
- ☐ Blue conservé pour rollback (24-48h)
- ☐ Documentation du déploiement

### Avantages et limites

**✅ Avantages** :
- Rollback instantané (< 30 secondes)
- Tests exhaustifs avant production
- Zero-downtime garanti
- Confiance maximale
- Isolation complète des environnements

**❌ Limites** :
- Coût : 2x ressources
- Complexité de gestion des états
- Synchronisation des données
- Nécessite infrastructure suffisante

### Quand utiliser Blue-Green

**Recommandé pour** :
- Applications critiques (e-commerce, finance)
- Services avec SLA strict
- Migrations majeures
- Refonte d'architecture
- Changements à fort impact

**Non recommandé pour** :
- Petites applications
- Environnements de dev/test
- Budget limité
- Applications hautement stateful sans solution de sync

---

## Conclusion

Le déploiement Blue-Green est une stratégie puissante mais exigeante qui offre le plus haut niveau de sécurité et de contrôle pour vos déploiements en production. En échange du doublement temporaire des ressources, vous obtenez :

- Une **garantie de rollback instantané**
- Une **tranquillité d'esprit** lors des déploiements
- Une **validation complète** avant exposition aux utilisateurs
- Un **downtime quasi-nul**

Sur MicroK8s, vous pouvez implémenter Blue-Green avec les outils natifs de Kubernetes (Services, Deployments, Ingress), sans dépendance à des outils complexes. C'est une excellente façon d'apprendre cette technique avant de l'appliquer en production à grande échelle.

**Vous maîtrisez maintenant** :
- L'architecture Blue-Green complète
- Les manifestes Kubernetes nécessaires
- Les scripts d'automatisation
- La gestion des états et données
- Le monitoring et les alertes
- Le troubleshooting

**Prochaine étape** : La section 17.11 vous présentera les déploiements Canary, une alternative qui permet une exposition progressive plutôt qu'un switch instantané !

**Continuez votre progression ! 🚀**

⏭️ [Canary deployments](/17-devops-et-cicd/11-canary-deployments.md)
