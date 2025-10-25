🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 18.7 Best Practices

## Introduction

Nous voici à la dernière section du chapitre 18 sur la gestion des ressources. Nous avons couvert énormément de terrain :
- **18.1** - Requests et Limits (fondamentaux)
- **18.2** - Resource Quotas (budget namespace)
- **18.3** - LimitRanges (contraintes individuelles)
- **18.4** - QoS Classes (priorité automatique)
- **18.5** - PriorityClasses (priorité explicite)
- **18.6** - Optimisation des ressources (efficacité)

Cette section **consolide** tout ce que nous avons appris en un ensemble de **best practices** actionnables, organisées et faciles à suivre. C'est votre guide de référence pour une gestion professionnelle des ressources Kubernetes.

## Récapitulatif : Les 6 piliers de la gestion des ressources

```
┌─────────────────────────────────────────────────────────┐
│         Les 6 piliers de la gestion des ressources      │
├─────────────────────────────────────────────────────────┤
│                                                         │
│  1️⃣  DÉFINIR                                           │
│      Requests et Limits sur chaque conteneur            │
│                                                         │
│  2️⃣  LIMITER                                           │
│      Resource Quotas par namespace                      │
│                                                         │
│  3️⃣  CONTRAINDRE                                       │
│      LimitRanges pour valeurs min/max/default           │
│                                                         │
│  4️⃣  PROTÉGER                                          │
│      QoS Classes (automatique) + PriorityClasses        │
│                                                         │
│  5️⃣  OPTIMISER                                         │
│      Right-sizing basé sur métriques réelles            │
│                                                         │
│  6️⃣  MONITORER                                         │
│      Surveillance continue et ajustements               │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

## Best Practices par thème

### 🎯 Configuration des Requests et Limits

#### BP 1.1 : Toujours définir des Requests

✅ **FAIRE** :
```yaml
# Toujours définir requests, même approximatifs
resources:
  requests:
    memory: "256Mi"
    cpu: "200m"
  limits:
    memory: "512Mi"
    cpu: "400m"
```

❌ **NE PAS FAIRE** :
```yaml
# Jamais de Pod sans resources en production
resources: {}
# ou pire : pas de section resources du tout
```

**Raison** : Sans requests, le scheduler ne peut pas prendre de décisions éclairées.

#### BP 1.2 : Toujours définir des Limits mémoire

✅ **FAIRE** :
```yaml
resources:
  requests:
    memory: "512Mi"
  limits:
    memory: "1Gi"    # TOUJOURS présent
```

❌ **NE PAS FAIRE** :
```yaml
resources:
  requests:
    memory: "512Mi"
  # Pas de limit mémoire = risque de fuite mémoire
```

**Raison** : Sans limit mémoire, une fuite peut consommer toute la RAM du nœud.

#### BP 1.3 : Limits CPU optionnels mais réfléchis

✅ **OPTION A - Avec limits** (prévisible) :
```yaml
resources:
  requests:
    cpu: "500m"
  limits:
    cpu: "1"    # Prévisible mais peut throttle
```

✅ **OPTION B - Sans limits** (flexible) :
```yaml
resources:
  requests:
    cpu: "500m"
  # Pas de limit CPU = peut burst si disponible
```

**Choix** :
- **Avec limits** : Applications critiques nécessitant performance prévisible
- **Sans limits** : Applications pouvant bénéficier de bursts occasionnels

#### BP 1.4 : Ratio Requests/Limits raisonnable

✅ **BON** :
```yaml
resources:
  requests:
    memory: "512Mi"
    cpu: "500m"
  limits:
    memory: "1Gi"     # Ratio 2:1 ✅
    cpu: "1"          # Ratio 2:1 ✅
```

❌ **MAUVAIS** :
```yaml
resources:
  requests:
    memory: "128Mi"
    cpu: "100m"
  limits:
    memory: "4Gi"     # Ratio 32:1 ❌
    cpu: "4"          # Ratio 40:1 ❌
```

**Règle** : Ratio maximum recommandé = 3:1 (jusqu'à 5:1 pour certains cas)

#### BP 1.5 : Requests = Limits pour applications critiques

✅ **FAIRE** (Guaranteed QoS) :
```yaml
# Base de données, cache, infrastructure critique
resources:
  limits:
    memory: "2Gi"
    cpu: "1"
  # requests = limits automatiquement
```

**Raison** : QoS Guaranteed = derniers à être évincés

### 🏛️ Gouvernance avec Quotas et LimitRanges

#### BP 2.1 : Un Resource Quota par namespace

✅ **FAIRE** :
```yaml
# Chaque namespace a son quota
apiVersion: v1
kind: ResourceQuota
metadata:
  name: dev-quota
  namespace: development
spec:
  hard:
    requests.cpu: "10"
    requests.memory: "20Gi"
    pods: "50"
```

**Raison** : Protection contre la sur-consommation et équité entre équipes

#### BP 2.2 : Un LimitRange par namespace

✅ **FAIRE** :
```yaml
# Chaque namespace a ses contraintes
apiVersion: v1
kind: LimitRange
metadata:
  name: dev-limits
  namespace: development
spec:
  limits:
  - type: Container
    max:
      cpu: "2"
      memory: "4Gi"
    defaultRequest:
      cpu: "100m"
      memory: "128Mi"
```

**Raison** : Empêche qu'un Pod monopolise le quota + fournit des defaults

#### BP 2.3 : Defaults en dev, pas en prod

✅ **Development** :
```yaml
# Defaults = facilite le travail
spec:
  limits:
  - type: Container
    default:
      cpu: "500m"
      memory: "512Mi"
```

✅ **Production** :
```yaml
# Pas de defaults = configuration consciente
spec:
  limits:
  - type: Container
    max:
      cpu: "4"
      memory: "8Gi"
    # Pas de section default
```

**Raison** : En prod, les ressources doivent être explicitement définies

#### BP 2.4 : Documenter les quotas

✅ **FAIRE** :
```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: team-a-quota
  namespace: team-a
  annotations:
    contact: "team-a-lead@company.com"
    last-review: "2025-10-01"
    justification: "Basé sur usage moyen + 30% croissance"
spec:
  hard:
    requests.cpu: "20"
    requests.memory: "40Gi"
```

#### BP 2.5 : Révision trimestrielle

✅ **Processus** :
```
Tous les 3 mois :
1. Analyser l'utilisation réelle vs quotas
2. Identifier les namespaces proche de la limite
3. Ajuster si justifié
4. Documenter les changements
```

### 🎯 Priorités et Protection

#### BP 3.1 : Hiérarchie claire de PriorityClasses

✅ **FAIRE** (5-7 niveaux max) :
```yaml
critical:  10000  # Infrastructure, paiements
high:      5000   # APIs client
medium:    2000   # Services internes
default:   1000   # Par défaut (globalDefault)
low:       500    # Batch non urgent
very-low:  100    # Maintenance
```

❌ **NE PAS FAIRE** (trop de niveaux) :
```yaml
# 15+ niveaux = confusion inutile
ultra-mega-critical: 25000
super-critical: 22000
very-critical: 20000
...
```

#### BP 3.2 : GlobalDefault défini

✅ **FAIRE** :
```yaml
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: default-priority
value: 1000
globalDefault: true  # Un seul dans le cluster
```

**Raison** : Évite d'avoir des Pods à priorité 0 par défaut

#### BP 3.3 : Préemption avec précaution

✅ **Pattern recommandé** :
```yaml
# Haute priorité : préemption activée
value: 10000
preemptionPolicy: PreemptLowerPriority

# Priorité moyenne : pas de préemption
value: 2000
preemptionPolicy: Never
```

**Règle** : Plus la priorité est haute, plus on autorise la préemption

#### BP 3.4 : Combiner QoS Guaranteed + Haute PriorityClass

✅ **Protection maximale** :
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: critical-database
spec:
  priorityClassName: critical-priority  # 10000
  containers:
  - name: postgres
    resources:
      limits:
        memory: "4Gi"
        cpu: "2"
      # QoS: Guaranteed
```

**Résultat** : Dernier Pod à être évincé, toutes conditions confondues

### 📊 Monitoring et Observabilité

#### BP 4.1 : Dashboards essentiels

✅ **Dashboards Grafana minimum** :
1. **Utilisation globale** : CPU/RAM cluster
2. **Utilisation par namespace** : Breakdown détaillé
3. **Ratio usage/requests** : Identifier sur-allocation
4. **Top 10 consommateurs** : Pods gourmands
5. **QoS et Priority distribution** : Vue d'ensemble
6. **Évictions et OOMKills** : Indicateurs de problèmes

#### BP 4.2 : Alertes critiques

✅ **Alertes Prometheus recommandées** :

```yaml
# 1. Utilisation élevée vs requests
- alert: HighResourceUsageVsRequests
  expr: |
    container_memory_working_set_bytes
    / on(pod) kube_pod_container_resource_requests{resource="memory"}
    > 0.9
  for: 15m

# 2. OOMKills répétés
- alert: FrequentOOMKills
  expr: |
    increase(kube_pod_container_status_restarts_total{reason="OOMKilled"}[1h]) > 3

# 3. Quota proche de la limite
- alert: QuotaNearLimit
  expr: |
    kube_resourcequota{type="used"}
    / on(resource, namespace) kube_resourcequota{type="hard"}
    > 0.85

# 4. Pods sans resources en production
- alert: PodsWithoutResourcesInProd
  expr: |
    count(kube_pod_container_resource_requests{namespace="production"} == 0) > 0
```

#### BP 4.3 : Révision hebdomadaire

✅ **Routine recommandée** :

```
Tous les lundis matin :
1. Vérifier les alertes de la semaine passée
2. Analyser les OOMKills et évictions
3. Identifier les Pods problématiques
4. Planifier les actions correctives
5. Vérifier la tendance d'utilisation
```

#### BP 4.4 : Métriques clés à suivre

✅ **KPIs essentiels** :
```yaml
Cluster :
  - Taux d'utilisation CPU global (cible: 60-75%)
  - Taux d'utilisation RAM global (cible: 70-80%)
  - Nombre de nœuds
  - Coût par Pod

Namespace :
  - Utilisation vs quota (ne pas dépasser 85%)
  - Nombre de Pods
  - Distribution QoS Classes

Pod :
  - Ratio usage/requests (cible: 60-80%)
  - Nombre de restarts (OOMKills)
  - CPU throttling (< 25%)
```

### 🚀 Optimisation Continue

#### BP 5.1 : Cycle d'optimisation régulier

✅ **Planning recommandé** :

```
Semaine 1-2 : Collecte de données (14 jours minimum)
Semaine 3   : Analyse et identification
Semaine 4   : Calculs et préparation
Semaine 5   : Tests en staging
Semaine 6   : Rollout production (10% → 50% → 100%)
Semaine 7-8 : Validation et monitoring
```

Répéter tous les **3 mois**

#### BP 5.2 : Analyse basée sur percentiles

✅ **FAIRE** :
```yaml
# Utiliser P95 pour CPU (pas la moyenne)
CPU Requests = P95 usage × 1.2

# Utiliser Max pour mémoire (pas la moyenne)
Memory Requests = Max usage × 1.15
```

❌ **NE PAS FAIRE** :
```yaml
# Moyenne = sous-estime les pics
CPU Requests = Avg usage × 1.2  # ❌
Memory Requests = Avg usage × 1.15  # ❌
```

#### BP 5.3 : Optimiser par vagues

✅ **Ordre recommandé** :
```
1. Non-production (dev, staging)
2. Services non-critiques
3. Services internes
4. Services clients (non-transactionnels)
5. Services critiques (avec extrême précaution)
```

#### BP 5.4 : Toujours garder un rollback

✅ **FAIRE** :
```bash
# Backup avant changement
kubectl get deployment api-backend -o yaml > backup-pre-optim.yaml

# Appliquer optimisation
kubectl apply -f api-backend-optimized.yaml

# Si problème
kubectl apply -f backup-pre-optim.yaml
```

#### BP 5.5 : Tests de charge post-optimisation

✅ **Valider avec tests** :
```bash
# Test de charge
k6 run --vus 100 --duration 10m load-test.js

# Pendant le test
kubectl top pods -w
kubectl get hpa -w

# Vérifier
# - Latence acceptable
# - Pas d'errors
# - Ressources dans les limites
# - Pas d'OOMKills
```

### 🏗️ Organisation et Gouvernance

#### BP 6.1 : Ownership clair

✅ **Définir les responsabilités** :
```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: team-frontend
  annotations:
    owner: "frontend-team@company.com"
    budget: "$2000/month"
    review-frequency: "monthly"
```

#### BP 6.2 : Process d'approbation

✅ **Pour ressources importantes** :
```yaml
# Haute priorité nécessite approbation
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: ultra-high
  annotations:
    approval-required: "yes"
    approved-by: "CTO"
    approval-date: "2025-10-15"
value: 10000
```

#### BP 6.3 : Documentation centralisée

✅ **Wiki/Confluence/Notion** :
```markdown
# Guide des ressources Kubernetes

## Requests et Limits recommandés
- Frontend: 100m CPU, 128Mi RAM
- API: 500m CPU, 512Mi RAM
- Base de données: 2 CPU, 4Gi RAM

## PriorityClasses disponibles
- critical (10000): Infrastructure core
- high (5000): Services clients
- default (1000): Services standard

## Process de demande d'augmentation de quota
1. Remplir le formulaire [lien]
2. Justifier avec métriques
3. Approbation platform team
4. Mise à jour sous 48h
```

#### BP 6.4 : Labels et annotations standards

✅ **Convention de nommage** :
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api-backend
  labels:
    app: api-backend
    team: backend-team
    env: production
    cost-center: engineering
  annotations:
    resource-profile: "medium"
    last-optimized: "2025-10-01"
    optimization-target: "cost"
```

### 🔒 Sécurité et Conformité

#### BP 7.1 : Pas de BestEffort en production

✅ **Forcer avec LimitRange** :
```yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: no-besteffort
  namespace: production
spec:
  limits:
  - type: Container
    defaultRequest:
      memory: "128Mi"
      cpu: "100m"
```

**Résultat** : Tout Pod sans resources aura au minimum ces requests

#### BP 7.2 : Network Policies + Resource Quotas

✅ **Sécurité en profondeur** :
```yaml
# Limiter les ressources
kind: ResourceQuota
# +
# Limiter le réseau
kind: NetworkPolicy
```

**Principe** : Defense in depth

#### BP 7.3 : PodDisruptionBudgets pour services critiques

✅ **FAIRE** :
```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: api-pdb
spec:
  minAvailable: 2  # Toujours 2 Pods minimum
  selector:
    matchLabels:
      app: api-backend
```

**+ Haute PriorityClass + Guaranteed QoS** = Triple protection

#### BP 7.4 : Audit régulier

✅ **Checklist trimestrielle** :
```
□ Pods sans resources en production ? (doit être 0)
□ Quotas respectés dans tous les namespaces ?
□ LimitRanges définis partout ?
□ PriorityClasses utilisées correctement ?
□ Pas de valeurs aberrantes (>10 CPU, >64Gi RAM) ?
□ Documentation à jour ?
```

## Anti-patterns à éviter

### ❌ Anti-pattern 1 : "Ça marche en dev, ship en prod"

```yaml
# ❌ MAUVAIS
# Configuration dev poussée en prod sans ajustement
resources:
  requests:
    cpu: "50m"     # Trop faible pour prod
    memory: "64Mi"
```

✅ **CORRECT** :
```yaml
# Adapter les resources par environnement
# dev/config.yaml
resources:
  requests:
    cpu: "50m"
    memory: "64Mi"

# prod/config.yaml
resources:
  requests:
    cpu: "500m"
    memory: "512Mi"
```

### ❌ Anti-pattern 2 : "Pas sûr, je mets 10 CPU"

```yaml
# ❌ MAUVAIS
# Sur-allocation par précaution
resources:
  requests:
    cpu: "10"      # Gaspillage énorme
    memory: "32Gi"
```

✅ **CORRECT** :
```yaml
# Basé sur des métriques réelles
# Usage observé P95 = 600m, Max = 1.2Gi
resources:
  requests:
    cpu: "720m"    # 600m × 1.2
    memory: "1.38Gi"  # 1.2Gi × 1.15
```

### ❌ Anti-pattern 3 : "Je copie-colle la config du voisin"

```yaml
# ❌ MAUVAIS
# Copier sans comprendre
resources:
  requests:
    cpu: "2"       # Pour une base de données
    memory: "8Gi"
# Alors que c'est un frontend léger !
```

✅ **CORRECT** :
```yaml
# Adapter au workload réel
resources:
  requests:
    cpu: "200m"    # Frontend léger
    memory: "256Mi"
```

### ❌ Anti-pattern 4 : "Set and forget"

```yaml
# ❌ MAUVAIS
# Configuration jamais révisée depuis 2 ans
resources:
  requests:
    cpu: "4"       # Surdimensionné
    memory: "16Gi"
# Usage réel = 10%
```

✅ **CORRECT** :
```
Routine d'optimisation tous les 3 mois
→ Analyse
→ Ajustement
→ Validation
```

### ❌ Anti-pattern 5 : "Requests = Limits partout"

```yaml
# ❌ MAUVAIS (sauf si vraiment critique)
# Guaranteed pour tout = gaspillage
resources:
  limits:
    cpu: "1"
    memory: "2Gi"
# Pour un service non-critique
```

✅ **CORRECT** :
```yaml
# Guaranteed uniquement pour critique
# Burstable pour le reste
resources:
  requests:
    cpu: "500m"
    memory: "1Gi"
  limits:
    cpu: "2"       # Peut burst
    memory: "2Gi"
```

### ❌ Anti-pattern 6 : "Pas de monitoring, on verra bien"

```yaml
# ❌ MAUVAIS
# Déployer sans monitoring
kubectl apply -f deployment.yaml
# Puis oublier
```

✅ **CORRECT** :
```yaml
# Monitoring + alertes + dashboards
# Surveillance continue
# Ajustements réguliers
```

### ❌ Anti-pattern 7 : "Pas besoin de quotas, on se fait confiance"

```yaml
# ❌ MAUVAIS
# Pas de Resource Quotas
# Résultat : Un namespace monopolise tout
```

✅ **CORRECT** :
```yaml
# Quotas systématiques
# Protection collective
# Équité garantie
```

## Configurations recommandées par type d'application

### 📱 Application Frontend (React, Vue, Angular)

```yaml
# Caractéristiques : Léger, stateless, nombreux replicas
apiVersion: apps/v1
kind: Deployment
metadata:
  name: frontend
spec:
  replicas: 5
  template:
    spec:
      priorityClassName: medium-priority
      containers:
      - name: nginx
        image: nginx:alpine
        resources:
          requests:
            memory: "64Mi"
            cpu: "50m"
          limits:
            memory: "128Mi"
            # Pas de limit CPU (peut burst)
        # QoS: Burstable
```

### 🔌 API Backend (REST/GraphQL)

```yaml
# Caractéristiques : Performance importante, usage variable
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api-backend
spec:
  replicas: 3
  template:
    spec:
      priorityClassName: high-priority
      containers:
      - name: api
        image: my-api:v1.0
        resources:
          requests:
            memory: "512Mi"
            cpu: "500m"
          limits:
            memory: "1Gi"
            cpu: "1"
        # QoS: Burstable
---
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: api-hpa
spec:
  minReplicas: 3
  maxReplicas: 20
  targetCPUUtilizationPercentage: 70
```

### 🗄️ Base de données (PostgreSQL, MySQL, MongoDB)

```yaml
# Caractéristiques : Critique, usage stable, stateful
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: postgresql
spec:
  replicas: 1
  template:
    spec:
      priorityClassName: critical-priority
      containers:
      - name: postgres
        image: postgres:15
        resources:
          limits:
            memory: "4Gi"
            cpu: "2"
          # requests = limits automatiquement
          # QoS: Guaranteed
---
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: postgres-pdb
spec:
  maxUnavailable: 0  # Aucune disruption acceptée
  selector:
    matchLabels:
      app: postgresql
```

### ⚡ Cache (Redis, Memcached)

```yaml
# Caractéristiques : Critique, mémoire importante, lecture intensive
apiVersion: apps/v1
kind: Deployment
metadata:
  name: redis
spec:
  replicas: 1
  template:
    spec:
      priorityClassName: critical-priority
      containers:
      - name: redis
        image: redis:7-alpine
        resources:
          limits:
            memory: "2Gi"
            cpu: "1"
          # QoS: Guaranteed
        # Pas de HPA (stateful cache)
```

### 🔄 Worker de traitement

```yaml
# Caractéristiques : Bursts importants, peut être interrompu
apiVersion: apps/v1
kind: Deployment
metadata:
  name: worker
spec:
  replicas: 10
  template:
    spec:
      priorityClassName: low-priority
      containers:
      - name: worker
        image: my-worker:v1.0
        resources:
          requests:
            memory: "256Mi"
            cpu: "250m"
          limits:
            memory: "1Gi"
            cpu: "2"         # Bursts importants OK
        # QoS: Burstable
---
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: worker-hpa
spec:
  minReplicas: 5
  maxReplicas: 50
  targetCPUUtilizationPercentage: 75
```

### ⏰ CronJob de maintenance

```yaml
# Caractéristiques : Périodique, non-critique, peut prendre du temps
apiVersion: batch/v1
kind: CronJob
metadata:
  name: nightly-cleanup
spec:
  schedule: "0 2 * * *"  # 2h du matin
  jobTemplate:
    spec:
      template:
        spec:
          priorityClassName: very-low-priority
          containers:
          - name: cleanup
            image: cleanup-tool:v1
            resources:
              requests:
                memory: "512Mi"
                cpu: "500m"
              limits:
                memory: "2Gi"
                cpu: "2"
          restartPolicy: OnFailure
```

## Configuration par environnement

### 🔧 Environnement Development

```yaml
# Philosophie : Ressources minimales, haute densité, économie

# Namespace
apiVersion: v1
kind: Namespace
metadata:
  name: development
---
# Resource Quota
apiVersion: v1
kind: ResourceQuota
metadata:
  name: dev-quota
  namespace: development
spec:
  hard:
    requests.cpu: "10"
    requests.memory: "20Gi"
    limits.cpu: "20"
    limits.memory: "40Gi"
    pods: "50"
---
# LimitRange
apiVersion: v1
kind: LimitRange
metadata:
  name: dev-limits
  namespace: development
spec:
  limits:
  - type: Container
    max:
      cpu: "1"
      memory: "2Gi"
    default:
      cpu: "200m"       # Defaults généreux
      memory: "256Mi"
    defaultRequest:
      cpu: "100m"
      memory: "128Mi"
---
# PriorityClass
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: dev-priority
value: 500
globalDefault: true  # Pour ce namespace
```

### 🧪 Environnement Staging

```yaml
# Philosophie : Similaire à prod mais ressources réduites

# Namespace
apiVersion: v1
kind: Namespace
metadata:
  name: staging
---
# Resource Quota
apiVersion: v1
kind: ResourceQuota
metadata:
  name: staging-quota
  namespace: staging
spec:
  hard:
    requests.cpu: "30"
    requests.memory: "60Gi"
    limits.cpu: "60"
    limits.memory: "120Gi"
    pods: "100"
---
# LimitRange
apiVersion: v1
kind: LimitRange
metadata:
  name: staging-limits
  namespace: staging
spec:
  limits:
  - type: Container
    max:
      cpu: "2"
      memory: "4Gi"
    default:
      cpu: "500m"
      memory: "512Mi"
    defaultRequest:
      cpu: "250m"
      memory: "256Mi"
---
# PriorityClass
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: staging-priority
value: 2000
```

### 🚀 Environnement Production

```yaml
# Philosophie : Stabilité maximale, ressources généreuses, monitoring intense

# Namespace
apiVersion: v1
kind: Namespace
metadata:
  name: production
---
# Resource Quota
apiVersion: v1
kind: ResourceQuota
metadata:
  name: prod-quota
  namespace: production
spec:
  hard:
    requests.cpu: "100"
    requests.memory: "200Gi"
    limits.cpu: "200"
    limits.memory: "400Gi"
    pods: "500"
---
# LimitRange
apiVersion: v1
kind: LimitRange
metadata:
  name: prod-limits
  namespace: production
spec:
  limits:
  - type: Container
    max:
      cpu: "8"
      memory: "16Gi"
    min:
      cpu: "100m"
      memory: "128Mi"
    # PAS de defaults en prod
  - type: Pod
    max:
      cpu: "16"
      memory: "32Gi"
---
# PriorityClasses multiples
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: prod-critical
value: 10000
---
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: prod-high
value: 5000
globalDefault: true  # Pour ce namespace
---
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: prod-medium
value: 2000
```

## Checklist de mise en production

### ✅ Avant le premier déploiement

**Infrastructure** :
- [ ] Prometheus + Grafana installés
- [ ] Dashboards de monitoring configurés
- [ ] Alertes critiques configurées
- [ ] Metrics Server actif (`kubectl top` fonctionne)

**Gouvernance** :
- [ ] Resource Quotas définis par namespace
- [ ] LimitRanges définis par namespace
- [ ] PriorityClasses créées
- [ ] GlobalDefault PriorityClass définie
- [ ] Documentation publiée et accessible

**Configuration** :
- [ ] Tous les Pods ont requests/limits
- [ ] Pas de Pod BestEffort en production
- [ ] PriorityClasses assignées correctement
- [ ] PodDisruptionBudgets pour services critiques
- [ ] Network Policies définies

### ✅ Revue hebdomadaire

- [ ] Analyser les alertes de la semaine
- [ ] Vérifier les OOMKills et évictions
- [ ] Examiner l'utilisation vs quotas
- [ ] Identifier les Pods problématiques
- [ ] Vérifier les tendances d'utilisation

### ✅ Revue mensuelle

- [ ] Analyser les coûts par namespace
- [ ] Vérifier les ratios utilisation/requests
- [ ] Identifier les opportunités d'optimisation
- [ ] Réviser les quotas (augmentation si nécessaire)
- [ ] Mettre à jour la documentation

### ✅ Revue trimestrielle

- [ ] Cycle d'optimisation complet (18.6)
- [ ] Révision des PriorityClasses
- [ ] Audit de sécurité (resources sans limits)
- [ ] Tests de charge sur services critiques
- [ ] Planification capacité pour 6 mois

### ✅ Revue annuelle

- [ ] Réévaluation de la stratégie globale
- [ ] Benchmark vs best practices industrie
- [ ] Formation équipes sur nouveautés
- [ ] Révision de la documentation complète
- [ ] Planning d'amélioration pour l'année

## Métriques de succès

### 📊 KPIs à suivre

**Efficacité** :
```yaml
Taux d'utilisation cluster :
  - Objectif : 60-75% CPU, 70-80% RAM
  - Calcul : (Usage total) / (Capacité totale)

Ratio usage/requests :
  - Objectif : 60-80% par Pod
  - Calcul : (Usage réel) / (Requests configurés)

Densité de Pods :
  - Objectif : Maximiser sans dégrader stabilité
  - Calcul : Nombre de Pods / Nombre de nœuds
```

**Stabilité** :
```yaml
Taux d'OOMKills :
  - Objectif : < 0.1% par jour
  - Calcul : (OOMKills) / (Total Pods) × 100

Taux d'évictions :
  - Objectif : < 0.5% par jour
  - Calcul : (Évictions) / (Total Pods) × 100

Taux de restarts :
  - Objectif : < 1% par jour
  - Calcul : (Restarts) / (Total Pods) × 100
```

**Coût** :
```yaml
Coût par Pod :
  - Objectif : Réduction de 20-30% après optimisation
  - Calcul : (Coût infrastructure) / (Nombre de Pods)

Ressources gaspillées :
  - Objectif : < 20%
  - Calcul : (Requests - Usage réel) / Requests × 100

ROI optimisation :
  - Objectif : Positif dans les 3 mois
  - Calcul : Économies - Coût temps ingénieur
```

### 🎯 Objectifs par maturité

**Niveau 1 - Débutant** (0-6 mois) :
```
✓ Tous les Pods ont requests/limits
✓ Resource Quotas en place
✓ Monitoring basique (kubectl top)
✓ Pas de BestEffort en production
Objectif utilisation : 40-50%
```

**Niveau 2 - Intermédiaire** (6-12 mois) :
```
✓ Niveau 1 +
✓ LimitRanges configurés
✓ PriorityClasses utilisées
✓ Prometheus + Grafana opérationnels
✓ Alertes configurées
✓ Première optimisation réalisée
Objectif utilisation : 55-65%
```

**Niveau 3 - Avancé** (12-24 mois) :
```
✓ Niveau 2 +
✓ Optimisation continue (trimestrielle)
✓ Dashboards personnalisés
✓ Automation (VPA ou scripts)
✓ Tests de charge réguliers
✓ Documentation complète
Objectif utilisation : 65-75%
```

**Niveau 4 - Expert** (24+ mois) :
```
✓ Niveau 3 +
✓ FinOps intégré
✓ Prédiction de capacité
✓ Optimisation multi-cluster
✓ Coût par feature/service
✓ Continuous optimization pipeline
Objectif utilisation : 70-80%
```

## Outils et ressources

### 🛠️ Outils essentiels

**Monitoring** :
- ✅ Prometheus (métriques)
- ✅ Grafana (visualisation)
- ✅ kube-state-metrics (métriques Kubernetes)
- ✅ Metrics Server (kubectl top)

**Optimisation** :
- ✅ Goldilocks (recommandations VPA)
- ✅ Kubecost (analyse coûts)
- ✅ kube-resource-report (rapports HTML)
- ✅ Vertical Pod Autoscaler (automatisation)

**Gouvernance** :
- ✅ OPA/Gatekeeper (policies as code)
- ✅ Kyverno (policies Kubernetes-native)
- ✅ Polaris (best practices audit)

### 📚 Documentation de référence

**Officielle** :
- [Kubernetes Resource Management](https://kubernetes.io/docs/concepts/configuration/manage-resources-containers/)
- [Quality of Service for Pods](https://kubernetes.io/docs/tasks/configure-pod-container/quality-service-pod/)
- [Resource Quotas](https://kubernetes.io/docs/concepts/policy/resource-quotas/)
- [Limit Ranges](https://kubernetes.io/docs/concepts/policy/limit-range/)

**Communauté** :
- [Kubernetes Best Practices](https://www.oreilly.com/library/view/kubernetes-best-practices/9781492056461/)
- [CNCF Resource Management SIG](https://github.com/kubernetes/community/tree/master/sig-scheduling)

## Points clés à retenir

🔑 **Les 10 commandements de la gestion des ressources** :

1. **Tu définiras** requests et limits sur tous les conteneurs
2. **Tu protégeras** avec Resource Quotas et LimitRanges
3. **Tu prioriseras** avec QoS et PriorityClasses appropriées
4. **Tu monitoreras** en continu avec alertes
5. **Tu optimiseras** régulièrement (tous les 3 mois)
6. **Tu documenteras** toutes les décisions
7. **Tu testeras** après chaque optimisation
8. **Tu adapteras** selon l'environnement
9. **Tu auditeras** régulièrement la conformité
10. **Tu amélioreras** continuellement le processus

## Conclusion du chapitre 18

Félicitations ! 🎉 Vous avez terminé le chapitre complet sur la **Gestion des Ressources** dans Kubernetes. Vous maîtrisez maintenant :

### Ce que vous savez faire

✅ **Configurer correctement** :
- Requests et Limits adaptés à chaque workload
- Resource Quotas pour gouverner les namespaces
- LimitRanges pour contraindre et fournir des defaults
- PriorityClasses pour gérer les priorités métier

✅ **Comprendre en profondeur** :
- Comment Kubernetes schedule les Pods
- L'ordre d'éviction (QoS + Priority)
- L'impact des ressources sur la stabilité
- Le coût réel de l'infrastructure

✅ **Optimiser professionnellement** :
- Analyser les métriques d'utilisation
- Calculer les ressources optimales
- Déployer les changements en sécurité
- Mesurer l'impact et l'amélioration

✅ **Maintenir durablement** :
- Monitoring continu et alertes
- Cycles d'optimisation réguliers
- Documentation et gouvernance
- Formation et amélioration continue

### L'impact sur votre cluster

Avec ces pratiques, vous pouvez vous attendre à :

📈 **Amélioration de 20-50%** de l'utilisation des ressources
💰 **Réduction de 20-40%** des coûts d'infrastructure
🎯 **Augmentation de 30-60%** de la densité de Pods
🛡️ **Stabilité accrue** avec moins d'OOMKills et d'évictions
🚀 **Performance optimale** pour toutes les applications

### La suite de votre parcours

La gestion des ressources est un **voyage continu**, pas une destination. Continuez à :
- 📊 Monitorer vos métriques
- 🔄 Optimiser régulièrement
- 📖 Vous former sur les nouveautés
- 🤝 Partager vos connaissances avec votre équipe

**Vous êtes maintenant un expert en gestion des ressources Kubernetes !** 🎓

Dans le prochain chapitre (19), nous aborderons le **Scaling et Autoscaling** - comment faire évoluer automatiquement vos applications en fonction de la charge. Les fondations que vous avez acquises ici seront essentielles pour comprendre comment Kubernetes scale efficacement.

---


⏭️ [Scaling et Autoscaling](/19-scaling-et-autoscaling/README.md)
