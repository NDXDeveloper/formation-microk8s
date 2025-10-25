🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 18.6 Optimisation des ressources

## Introduction

Nous avons maintenant couvert tous les **mécanismes** de gestion des ressources (18.1 à 18.5). Mais connaître les outils ne suffit pas - il faut savoir les **utiliser efficacement**. C'est le but de cette section : apprendre à **optimiser** les ressources de votre cluster.

L'optimisation des ressources consiste à trouver le **juste équilibre** entre :
- 🟢 Performance : Assez de ressources pour que les applications fonctionnent bien
- 🟢 Stabilité : Protection contre les pannes et surcharges
- 🟢 Coût : Ne pas gaspiller de ressources inutilisées
- 🟢 Densité : Maximiser le nombre de Pods par nœud

## Pourquoi optimiser ?

### Le problème du gaspillage

Sans optimisation, voici ce qui se passe souvent :

```
┌────────────────────────────────────────────────┐
│         Ressources configurées vs réelles      │
├────────────────────────────────────────────────┤
│                                                │
│  Application A                                 │
│  ├─ Requests configurés : 2 CPU, 4Gi RAM       │
│  └─ Usage réel moyen   : 0.3 CPU, 800Mi RAM    │
│     Gaspillage : 85% CPU, 80% RAM ❌           │
│                                                │
│  Application B                                 │
│  ├─ Requests configurés : 1 CPU, 2Gi RAM       │
│  └─ Usage réel moyen   : 0.1 CPU, 300Mi RAM    │
│     Gaspillage : 90% CPU, 85% RAM ❌           │
│                                                │
│  Résultat :                                    │
│  • Cluster sous-utilisé                        │
│  • Coûts élevés                                │
│  • Faible densité de Pods                      │
│                                                │
└────────────────────────────────────────────────┘
```

### Les bénéfices de l'optimisation

✅ **Réduction des coûts** :
- Moins de serveurs nécessaires
- Meilleure utilisation de l'infrastructure existante
- ROI amélioré

✅ **Meilleure densité** :
- Plus de Pods par nœud
- Cluster plus efficient
- Moins de ressources gaspillées

✅ **Stabilité accrue** :
- Requests réalistes = meilleur scheduling
- Limits appropriées = moins d'OOM kills
- Prévisibilité améliorée

✅ **Performance optimale** :
- Ressources suffisantes quand nécessaire
- Pas de sur-allocation qui ralentit
- Équilibre performance/coût

## Analogie : La gestion d'un parking

Imaginez un parking d'immeuble :

**Sans optimisation** (requests trop élevés) :
```
Réservations :
├─ Appartement 1 : 3 places (possède 1 voiture)
├─ Appartement 2 : 4 places (possède 1 voiture)
└─ Appartement 3 : 2 places (possède 2 voitures)

Résultat :
• 9 places réservées
• 4 voitures réelles
• 5 places gaspillées (55%) ❌
• Nouveaux locataires refusés alors qu'il y a de la place
```

**Avec optimisation** (right-sizing) :
```
Réservations :
├─ Appartement 1 : 1 place + 1 flex (possède 1 voiture)
├─ Appartement 2 : 1 place + 1 flex (possède 1 voiture)
└─ Appartement 3 : 2 places (possède 2 voitures)

Résultat :
• 6 places réservées (4 fixes + 2 flex)
• 4 voitures réelles
• 3 places libres pour nouveaux locataires ✅
• Flexibilité pour visites occasionnelles
```

Dans Kubernetes :
- **Places réservées** = Requests
- **Places flexibles** = Différence entre Requests et Limits
- **Voitures réelles** = Usage réel des ressources

## Méthodologie d'optimisation

### Le cycle d'optimisation continue

```
┌──────────────────────────────────────────────────┐
│       Cycle d'optimisation (itératif)            │
├──────────────────────────────────────────────────┤
│                                                  │
│  1️⃣  MESURER                                    │
│      └─ Collecter les métriques d'usage réel     │
│                                                  │
│  2️⃣  ANALYSER                                   │
│      └─ Comparer usage vs requests/limits        │
│                                                  │
│  3️⃣  IDENTIFIER                                 │
│      └─ Trouver les sur/sous-allocations         │
│                                                  │
│  4️⃣  AJUSTER                                    │
│      └─ Modifier les requests/limits             │
│                                                  │
│  5️⃣  DÉPLOYER                                   │
│      └─ Appliquer les changements                │
│                                                  │
│  6️⃣  VALIDER                                    │
│      └─ Vérifier que ça fonctionne bien          │
│                                                  │
│  7️⃣  RÉPÉTER ↺                                  │
│      └─ Recommencer après 2-4 semaines           │
│                                                  │
└──────────────────────────────────────────────────┘
```

### Phase 1 : Mesurer l'usage réel

**Outils nécessaires** :
- Prometheus (métriques)
- Grafana (visualisation)
- kubectl top (quick check)

**Métriques à collecter** :

```yaml
# CPU
- container_cpu_usage_seconds_total
- Percentile 95 sur 7-14 jours
- Pics d'utilisation

# Mémoire
- container_memory_working_set_bytes
- Maximum observé sur 7-14 jours
- Tendances de croissance
```

**Durée de mesure recommandée** :
- Minimum : **7 jours** (couvre un cycle hebdomadaire)
- Recommandé : **14-30 jours** (couvre variations mensuelles)
- Idéal : **60-90 jours** (patterns saisonniers)

### Phase 2 : Analyser les données

**Calculer les ratios** :

```
Ratio d'utilisation CPU = Usage P95 / Requests CPU
Ratio d'utilisation RAM = Usage Max / Requests Memory

Exemples :
• Usage P95: 300m, Requests: 500m → Ratio: 60% ✅ (OK)
• Usage P95: 400m, Requests: 500m → Ratio: 80% ✅ (Bon)
• Usage P95: 100m, Requests: 1000m → Ratio: 10% ❌ (Sur-alloué)
• Usage P95: 900m, Requests: 500m → Ratio: 180% ⚠️ (Sous-alloué)
```

**Catégorisation** :

| Ratio d'utilisation | Statut | Action |
|---------------------|--------|--------|
| < 20% | 🔴 Fortement sur-alloué | Réduire requests immédiatement |
| 20-50% | 🟡 Sur-alloué | Réduire requests prudemment |
| 50-70% | 🟢 Bien dimensionné | Surveiller |
| 70-85% | 🟢 Optimal | Parfait, maintenir |
| 85-95% | 🟡 Sous-alloué | Augmenter requests |
| > 95% | 🔴 Critique | Augmenter requests urgence |

### Phase 3 : Identifier les opportunités

**Questions à se poser** :

1. **Quels Pods sont sur-alloués ?**
   - Usage réel << Requests
   - Candidates à la réduction

2. **Quels Pods sont sous-alloués ?**
   - Usage réel >> Requests
   - Risque d'éviction ou throttling

3. **Quels Pods sont instables ?**
   - OOMKilled fréquents
   - CPU throttling élevé

4. **Quelle est la marge de sécurité ?**
   - Différence entre usage P95 et P99
   - Variabilité de l'usage

### Phase 4 : Ajuster les ressources

**Formules de calcul** :

**Pour le CPU (requests)** :
```
Nouveau Request CPU = Usage P95 × 1.2
                    = Usage P95 + 20% marge
```

**Pour le CPU (limits)** :
```
Option 1 (conservative) : Limit CPU = Request × 2
Option 2 (permissive)   : Pas de limit CPU (burst libre)
```

**Pour la mémoire (requests)** :
```
Nouveau Request Memory = Usage Max × 1.15
                       = Usage Max + 15% marge
```

**Pour la mémoire (limits)** :
```
Limit Memory = Request × 1.5
             = Request + 50% marge
```

**Exemple de calcul** :

```yaml
# Données observées sur 14 jours
CPU Usage :
  - P50: 200m
  - P95: 350m
  - P99: 450m
  - Max: 500m

Memory Usage :
  - P50: 400Mi
  - P95: 600Mi
  - P99: 700Mi
  - Max: 750Mi

# Configuration actuelle (avant optimisation)
resources:
  requests:
    cpu: "1"        # 1000m
    memory: "2Gi"   # 2048Mi
  limits:
    cpu: "2"
    memory: "4Gi"

# Analyse
CPU : Usage P95 350m vs Request 1000m → Ratio 35% (sur-alloué ❌)
RAM : Usage Max 750Mi vs Request 2Gi → Ratio 36% (sur-alloué ❌)

# Nouvelle configuration optimisée
resources:
  requests:
    cpu: "420m"     # 350m × 1.2 = 420m
    memory: "862Mi" # 750Mi × 1.15 = 862Mi
  limits:
    cpu: "840m"     # 420m × 2
    memory: "1293Mi"# 862Mi × 1.5

# Résultat
CPU économisé : 580m (58% réduction)
RAM économisée : 1186Mi (58% réduction)
```

## Right-Sizing : Les patterns d'optimisation

### Pattern 1 : Application web frontend

**Caractéristiques** :
- Usage CPU variable (pics pendant la journée)
- Mémoire stable
- Peut gérer du throttling CPU

**Configuration optimale** :

```yaml
# Avant optimisation (trop généreux)
resources:
  requests:
    memory: "512Mi"
    cpu: "500m"
  limits:
    memory: "1Gi"
    cpu: "1"

# Après observation : P95 = 150m CPU, Max = 320Mi RAM

# Après optimisation
resources:
  requests:
    memory: "368Mi"  # 320Mi × 1.15
    cpu: "180m"      # 150m × 1.2
  limits:
    memory: "552Mi"  # 368Mi × 1.5
    # Pas de limit CPU - peut burst si besoin
```

**Économie** : 64% CPU, 28% RAM

### Pattern 2 : API backend

**Caractéristiques** :
- Usage CPU prévisible
- Mémoire peut croître (caches)
- Performance critique

**Configuration optimale** :

```yaml
# Après observation : P95 = 400m CPU, Max = 800Mi RAM

resources:
  requests:
    memory: "920Mi"  # 800Mi × 1.15
    cpu: "480m"      # 400m × 1.2
  limits:
    memory: "1380Mi" # 920Mi × 1.5
    cpu: "960m"      # 480m × 2 (permet bursts)
```

### Pattern 3 : Base de données

**Caractéristiques** :
- Usage mémoire élevé et stable
- CPU peut pic lors des requêtes
- Critique - nécessite Guaranteed QoS

**Configuration optimale** :

```yaml
# Après observation : P95 = 1.5 CPU, Max = 3.8Gi RAM

resources:
  limits:  # Forme courte - Guaranteed
    memory: "4.37Gi"  # 3.8Gi × 1.15
    cpu: "1800m"      # 1.5 × 1.2
  # requests = limits automatiquement
```

### Pattern 4 : Worker batch

**Caractéristiques** :
- Usage très variable
- Bursts importants pendant le traitement
- Peut être interrompu

**Configuration optimale** :

```yaml
# Après observation : P95 = 600m CPU, Max = 1.2Gi RAM
# Mais bursts jusqu'à 3 CPU possibles

resources:
  requests:
    memory: "1.38Gi"  # 1.2Gi × 1.15
    cpu: "720m"       # 600m × 1.2
  limits:
    memory: "2.76Gi"  # 2× pour les pics
    cpu: "3"          # Permet bursts complets
```

### Pattern 5 : Service stateless léger

**Caractéristiques** :
- Usage minimal et constant
- Nombreux replicas
- Optimisation = forte densité

**Configuration optimale** :

```yaml
# Après observation : P95 = 50m CPU, Max = 80Mi RAM

resources:
  requests:
    memory: "92Mi"   # 80Mi × 1.15
    cpu: "60m"       # 50m × 1.2
  limits:
    memory: "138Mi"  # 92Mi × 1.5
    cpu: "120m"      # 60m × 2
```

**Bénéfice** : 100+ Pods par nœud possible

## Stratégies d'optimisation

### Stratégie 1 : Optimisation progressive

✅ **Approche sûre** pour la production

```
Phase 1 : Non-production (Dev/Staging)
├─ Appliquer l'optimisation
├─ Tester pendant 1 semaine
└─ Valider stabilité

Phase 2 : Production - Canary
├─ 10% des Pods optimisés
├─ Observer pendant 48h
└─ Valider métriques

Phase 3 : Production - Progressive
├─ 25% des Pods
├─ 50% des Pods
├─ 75% des Pods
└─ 100% des Pods (si tout va bien)
```

### Stratégie 2 : Par criticité

**Ordre d'optimisation** :

```
1️⃣  Services non-critiques
    └─ Impact faible si problème

2️⃣  Services internes
    └─ Utilisateurs internes tolérants

3️⃣  Services clients (non-transactionnels)
    └─ Tableau de bord, analytics

4️⃣  Services clients critiques
    └─ APIs principales

5️⃣  Services transactionnels
    └─ Paiements, commandes
    └─ Attention maximale
```

### Stratégie 3 : Quick wins d'abord

**Prioriser par impact** :

```yaml
# Score d'optimisation = Économie × Facilité

Pod A :
  - Requests actuels : 4 CPU, 8Gi RAM
  - Usage réel : 0.5 CPU, 1Gi RAM
  - Économie potentielle : 3.5 CPU, 7Gi RAM
  - Facilité : Facile (service simple)
  - Score : ⭐⭐⭐⭐⭐ (PRIORITAIRE)

Pod B :
  - Requests actuels : 0.5 CPU, 512Mi RAM
  - Usage réel : 0.4 CPU, 400Mi RAM
  - Économie potentielle : 0.1 CPU, 112Mi RAM
  - Facilité : Facile
  - Score : ⭐ (Faible priorité)

Pod C :
  - Requests actuels : 2 CPU, 4Gi RAM
  - Usage réel : 0.3 CPU, 800Mi RAM
  - Économie potentielle : 1.7 CPU, 3.2Gi RAM
  - Facilité : Difficile (base de données stateful)
  - Score : ⭐⭐ (Attention requise)
```

**Commencer par les Pods avec score ⭐⭐⭐⭐⭐**

## Over-provisioning vs Under-provisioning

### Le dilemme

```
┌────────────────────────────────────────────────┐
│         Over vs Under Provisioning             │
├────────────────────────────────────────────────┤
│                                                │
│  Over-provisioning (requests trop élevés)      │
│  ❌ Gaspillage de ressources                   │
│  ❌ Coûts élevés                               │
│  ❌ Faible densité                             │
│  ✅ Haute stabilité                            │
│  ✅ Bonnes performances                        │
│                                                │
│  Under-provisioning (requests trop bas)        │
│  ✅ Utilisation maximale                       │
│  ✅ Coûts réduits                              │
│  ✅ Haute densité                              │
│  ❌ Risque d'instabilité                       │
│  ❌ Évictions possibles                        │
│  ❌ Performance dégradée                       │
│                                                │
│  Sweet Spot (équilibre optimal)                │
│  ✅ Bonne utilisation                          │
│  ✅ Coûts contrôlés                            │
│  ✅ Bonne densité                              │
│  ✅ Stabilité maintenue                        │
│  ✅ Performances acceptables                   │
│                                                │
└────────────────────────────────────────────────┘
```

### Trouver le sweet spot

**Règle générale** :

```
Requests = Usage P95 + marge de sécurité

Marge de sécurité recommandée :
• CPU : +20% (moins critique, compressible)
• RAM : +15% (plus critique, non-compressible)
```

**Adapter selon le contexte** :

```yaml
# Service critique (priorité stabilité)
Marge CPU : +30-40%
Marge RAM : +25-30%

# Service standard (équilibre)
Marge CPU : +20%
Marge RAM : +15%

# Service non-critique (priorité densité)
Marge CPU : +10-15%
Marge RAM : +10%
```

### Signes d'under-provisioning

⚠️ **Indicateurs à surveiller** :

```yaml
CPU Under-provisioning :
  - Throttling élevé (> 25% du temps)
  - Latence accrue
  - Timeouts fréquents
  - Usage constant proche de 100%

Mémoire Under-provisioning :
  - OOMKilled répétés
  - Pods redémarrés fréquemment
  - Évictions par memory pressure
  - Usage constamment > 90%
```

**Action** : Augmenter les requests immédiatement

### Signes d'over-provisioning

✅ **Indicateurs** :

```yaml
CPU Over-provisioning :
  - Usage moyen < 30% des requests
  - Usage P95 < 50% des requests
  - Pas de pics significatifs

Mémoire Over-provisioning :
  - Usage max < 50% des requests
  - Mémoire stable et basse
  - Pas de croissance tendancielle
```

**Action** : Réduire les requests progressivement

## Outils d'optimisation

### 1. kubectl top - Quick check

**Usage basique** :

```bash
# Usage actuel des Pods
kubectl top pods -n production

# Tri par mémoire
kubectl top pods -n production --sort-by=memory

# Tri par CPU
kubectl top pods -n production --sort-by=cpu

# Tous les namespaces
kubectl top pods --all-namespaces
```

**Limitations** :
- Snapshot instantané (pas d'historique)
- Données récentes uniquement
- Pas de percentiles

### 2. Prometheus queries

**Requêtes utiles** :

```promql
# CPU P95 sur 7 jours
quantile(0.95,
  rate(container_cpu_usage_seconds_total[7d])
) by (pod, container)

# Mémoire maximum sur 7 jours
max_over_time(
  container_memory_working_set_bytes[7d]
) by (pod, container)

# Ratio d'utilisation CPU
rate(container_cpu_usage_seconds_total[5m])
/ on(pod, container) group_left
kube_pod_container_resource_requests{resource="cpu"}

# Throttling CPU
rate(container_cpu_cfs_throttled_seconds_total[5m])
```

### 3. Goldilocks - Recommandations automatiques

**Goldilocks** est un outil qui génère des recommandations de requests/limits automatiquement.

**Installation** :

```bash
# Avec Helm
helm repo add fairwinds-stable https://charts.fairwinds.com/stable
helm install goldilocks fairwinds-stable/goldilocks \
  --namespace goldilocks --create-namespace
```

**Utilisation** :

```bash
# Activer pour un namespace
kubectl label namespace production goldilocks.fairwinds.com/enabled=true

# Accéder au dashboard
kubectl -n goldilocks port-forward svc/goldilocks-dashboard 8080:80
# Ouvrir http://localhost:8080
```

**Résultat** : Recommandations basées sur VPA (Vertical Pod Autoscaler)

### 4. Kubecost - Analyse financière

**Kubecost** fournit des insights sur les coûts et l'utilisation.

**Installation** :

```bash
# Avec Helm
helm repo add kubecost https://kubecost.github.io/cost-analyzer/
helm install kubecost kubecost/cost-analyzer \
  --namespace kubecost --create-namespace
```

**Métriques fournies** :
- Coût par namespace
- Coût par Pod
- Recommandations d'optimisation
- Idle resources (ressources gaspillées)

### 5. kube-resource-report

**Génère des rapports HTML** sur l'utilisation des ressources.

```bash
# Installation
pip install kube-resource-report

# Génération du rapport
kube-resource-report --output-dir ./reports
```

**Contenu** :
- Ratio utilisation/requests
- Pods sur-alloués
- Pods sous-alloués
- Recommandations

### 6. Scripts personnalisés

**Script d'analyse simple** :

```bash
#!/bin/bash
# analyze-resources.sh

echo "=== Analyse des ressources ==="

for namespace in $(kubectl get ns -o jsonpath='{.items[*].metadata.name}'); do
  echo ""
  echo "Namespace: $namespace"

  # Requests totaux
  requests_cpu=$(kubectl get pods -n $namespace -o json | \
    jq '[.items[].spec.containers[].resources.requests.cpu // "0"] |
        map(gsub("m";"") | tonumber) | add')

  requests_mem=$(kubectl get pods -n $namespace -o json | \
    jq '[.items[].spec.containers[].resources.requests.memory // "0"] |
        map(gsub("Mi";"") | gsub("Gi";"000") | tonumber) | add')

  echo "  Total CPU requests: ${requests_cpu}m"
  echo "  Total Memory requests: ${requests_mem}Mi"
done
```

## Dashboard Grafana pour l'optimisation

### Panels recommandés

**Panel 1 : Ratio d'utilisation par Pod** :

```promql
# CPU
sum(rate(container_cpu_usage_seconds_total[5m])) by (pod)
/ on(pod) group_left
sum(kube_pod_container_resource_requests{resource="cpu"}) by (pod)
× 100
```

**Panel 2 : Top 10 Pods sur-alloués** :

```promql
# Différence entre requests et usage réel
(
  sum(kube_pod_container_resource_requests{resource="cpu"}) by (pod)
  -
  sum(rate(container_cpu_usage_seconds_total[5m])) by (pod)
)
> 0.5  # Plus de 500m gaspillés
```

**Panel 3 : Économies potentielles** :

```promql
# CPU gaspillé total
sum(
  kube_pod_container_resource_requests{resource="cpu"}
  -
  rate(container_cpu_usage_seconds_total[5m])
)
```

**Panel 4 : Pods à risque (sous-alloués)** :

```promql
# Usage > 90% des requests
sum(rate(container_cpu_usage_seconds_total[5m])) by (pod)
/ on(pod) group_left
sum(kube_pod_container_resource_requests{resource="cpu"}) by (pod)
> 0.9
```

### Dashboard complet exemple

```json
{
  "dashboard": {
    "title": "Resource Optimization",
    "panels": [
      {
        "title": "Cluster Efficiency",
        "targets": [
          {
            "expr": "sum(rate(container_cpu_usage_seconds_total[5m])) / sum(kube_node_status_capacity{resource='cpu'}) * 100"
          }
        ]
      },
      {
        "title": "Over-provisioned Pods (CPU)",
        "targets": [
          {
            "expr": "topk(10, sum(kube_pod_container_resource_requests{resource='cpu'} - rate(container_cpu_usage_seconds_total[5m])) by (pod))"
          }
        ]
      },
      {
        "title": "Under-provisioned Pods (Memory)",
        "targets": [
          {
            "expr": "sum(container_memory_working_set_bytes / kube_pod_container_resource_requests{resource='memory'}) by (pod) > 0.9"
          }
        ]
      }
    ]
  }
}
```

## Optimisation par type de charge

### Workload prévisible

**Exemples** : API stable, service web constant

**Stratégie** :
```yaml
# Requests serrés (usage P95 + 15%)
# Limits modérés (2× requests)
resources:
  requests:
    cpu: "500m"      # Basé sur P95
    memory: "512Mi"
  limits:
    cpu: "1"         # 2× pour pics occasionnels
    memory: "768Mi"  # 1.5× pour sécurité
```

### Workload variable

**Exemples** : E-commerce (pics journée), streaming

**Stratégie** :
```yaml
# Requests basés sur charge moyenne haute
# Limits généreux pour absorber les pics
resources:
  requests:
    cpu: "300m"      # Charge moyenne-haute
    memory: "512Mi"
  limits:
    cpu: "2"         # Absorbe pics (6× requests)
    memory: "1Gi"    # 2× pour pics
```

### Workload par batch

**Exemples** : Jobs ETL, traitement de données

**Stratégie** :
```yaml
# Requests modérés
# Limits élevés (bursts importants OK)
resources:
  requests:
    cpu: "500m"      # Garantie minimale
    memory: "1Gi"
  limits:
    cpu: "4"         # Peut utiliser beaucoup pendant traitement
    memory: "4Gi"    # Données en mémoire
```

### Workload critique

**Exemples** : Bases de données, caches

**Stratégie** :
```yaml
# Guaranteed QoS
# Requests = Limits (aucun burst)
resources:
  limits:
    cpu: "2"         # Stable et prévisible
    memory: "4Gi"    # Pas de surprise
  # requests = limits automatiquement
```

## Bonnes pratiques d'optimisation

### 1. Ne jamais optimiser aveuglément

❌ **Mauvais** :
```yaml
# Réduire de 50% partout sans analyse
resources:
  requests:
    cpu: "250m"    # Avant: 500m
    memory: "256Mi" # Avant: 512Mi
```

✅ **Bon** :
```yaml
# Basé sur 14 jours d'observations
# P95 CPU = 180m, Max RAM = 220Mi
resources:
  requests:
    cpu: "216m"    # 180m × 1.2
    memory: "253Mi" # 220Mi × 1.15
```

### 2. Optimiser namespace par namespace

✅ **Approche structurée** :

```
Semaine 1 : Namespace "dev" (non-critique)
Semaine 2 : Namespace "staging"
Semaine 3 : Namespace "production-internal"
Semaine 4 : Namespace "production-customer"
```

### 3. Conserver un historique

✅ **Documenter les changements** :

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api-backend
  annotations:
    optimization-history: |
      2025-10-01: Initial deployment - 1 CPU, 2Gi RAM
      2025-10-15: Reduced to 500m CPU, 1Gi RAM (usage analysis)
      2025-11-01: Increased to 600m CPU, 1Gi RAM (latency issues)
spec:
  template:
    spec:
      containers:
      - name: api
        resources:
          requests:
            cpu: "600m"
            memory: "1Gi"
```

### 4. Alertes sur les changements

✅ **Monitoring post-optimisation** :

```yaml
# Alerte Prometheus
- alert: HighCPUAfterOptimization
  expr: |
    rate(container_cpu_usage_seconds_total[5m])
    / on(pod) group_left
    kube_pod_container_resource_requests{resource="cpu"}
    > 0.95
  for: 15m
  annotations:
    summary: "Pod {{ $labels.pod }} using >95% CPU after optimization"
```

### 5. Rollback plan

✅ **Toujours prévoir un rollback** :

```bash
# Avant optimisation - sauvegarder la config actuelle
kubectl get deployment api-backend -o yaml > backup-api-backend-pre-optim.yaml

# Déployer l'optimisation
kubectl apply -f api-backend-optimized.yaml

# Si problème - rollback immédiat
kubectl apply -f backup-api-backend-pre-optim.yaml
```

### 6. Tests de charge post-optimisation

✅ **Valider avec des tests** :

```bash
# Test de charge avec k6
k6 run --vus 100 --duration 10m load-test.js

# Observer pendant le test
kubectl top pods -n production --sort-by=cpu
watch -n 2 'kubectl get hpa -n production'

# Vérifier les métriques
# - Latence : doit rester acceptable
# - Error rate : pas d'augmentation
# - CPU/RAM : dans les limites
```

### 7. Optimisation différenciée par environnement

✅ **Adapter selon l'environnement** :

```yaml
# Development - agressif
resources:
  requests:
    cpu: "100m"    # Minimal
    memory: "256Mi"

# Staging - modéré
resources:
  requests:
    cpu: "300m"    # Basé sur usage
    memory: "512Mi"

# Production - conservateur
resources:
  requests:
    cpu: "600m"    # Usage P95 + 30%
    memory: "1Gi"  # Usage Max + 25%
```

## Automatisation de l'optimisation

### Vertical Pod Autoscaler (VPA)

**VPA** ajuste automatiquement les requests/limits.

**Installation** :

```bash
git clone https://github.com/kubernetes/autoscaler.git
cd autoscaler/vertical-pod-autoscaler
./hack/vpa-up.sh
```

**Utilisation** :

```yaml
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
metadata:
  name: api-backend-vpa
spec:
  targetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: api-backend
  updatePolicy:
    updateMode: "Auto"  # Auto, Recreate, Initial, ou Off
  resourcePolicy:
    containerPolicies:
    - containerName: api
      minAllowed:
        cpu: "100m"
        memory: "128Mi"
      maxAllowed:
        cpu: "2"
        memory: "4Gi"
```

**Modes** :
- `Off` : Recommandations seulement (pas d'action)
- `Initial` : Applique à la création du Pod
- `Recreate` : Recrée les Pods pour appliquer
- `Auto` : Applique en direct (éviction + recréation)

⚠️ **Attention** : VPA et HPA sont incompatibles sur les mêmes métriques.

### Script de recommandations périodique

```bash
#!/bin/bash
# weekly-optimization-check.sh

echo "=== Weekly Optimization Report ==="
date

NAMESPACES="production staging"

for ns in $NAMESPACES; do
  echo ""
  echo "Namespace: $ns"

  # Pods with low utilization (<30%)
  echo "  Over-provisioned Pods (CPU <30%):"
  kubectl top pods -n $ns --no-headers | awk '{
    if ($2 != "0m") {
      # Parse CPU usage (remove 'm')
      cpu = substr($2, 1, length($2)-1)
      # Get requests
      cmd = "kubectl get pod "$1" -n '"$ns"' -o jsonpath='\''{.spec.containers[0].resources.requests.cpu}'\''";
      cmd | getline requests
      close(cmd)
      # Parse requests
      if (requests ~ /m$/) {
        req = substr(requests, 1, length(requests)-1)
      } else {
        req = requests * 1000
      }
      # Calculate ratio
      if (req > 0) {
        ratio = (cpu / req) * 100
        if (ratio < 30) {
          printf "    - %s: %.0f%% utilization\n", $1, ratio
        }
      }
    }
  }'

  # Pods with high utilization (>85%)
  echo "  Under-provisioned Pods (CPU >85%):"
  kubectl top pods -n $ns --no-headers | awk '{
    if ($2 != "0m") {
      cpu = substr($2, 1, length($2)-1)
      cmd = "kubectl get pod "$1" -n '"$ns"' -o jsonpath='\''{.spec.containers[0].resources.requests.cpu}'\''";
      cmd | getline requests
      close(cmd)
      if (requests ~ /m$/) {
        req = substr(requests, 1, length(requests)-1)
      } else {
        req = requests * 1000
      }
      if (req > 0) {
        ratio = (cpu / req) * 100
        if (ratio > 85) {
          printf "    - %s: %.0f%% utilization ⚠️\n", $1, ratio
        }
      }
    }
  }'
done

echo ""
echo "=== Run this script weekly for continuous optimization ==="
```

**Automatiser avec CronJob** :

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: weekly-optimization-report
  namespace: monitoring
spec:
  schedule: "0 9 * * 1"  # Tous les lundis à 9h
  jobTemplate:
    spec:
      template:
        spec:
          serviceAccountName: resource-analyzer
          containers:
          - name: analyzer
            image: bitnami/kubectl:latest
            command:
            - /bin/bash
            - -c
            - |
              # Script d'analyse ici
              echo "Optimization report generated"
          restartPolicy: OnFailure
```

## Cas pratiques d'optimisation

### Cas 1 : Startup en croissance

**Contexte** :
- Cluster de 10 nœuds
- Budget serré
- Croissance rapide

**Objectif** : Maximiser la densité sans sacrifier la stabilité

**Actions** :
1. Analyse complète sur 30 jours
2. Optimisation agressive (marge 10-15%)
3. Monitoring intensif post-optimisation
4. Gain : 30% de capacité supplémentaire

**Résultat** :
```
Avant :
- 200 Pods sur 10 nœuds
- Utilisation moyenne : 45%
- Coût mensuel : $5000

Après :
- 280 Pods sur 10 nœuds
- Utilisation moyenne : 65%
- Coût mensuel : $5000
- Économie effective : $1600 (capacité supplémentaire)
```

### Cas 2 : Entreprise établie

**Contexte** :
- Cluster de 100 nœuds
- Stabilité primordiale
- Budget confortable

**Objectif** : Optimiser sans risque

**Actions** :
1. Analyse sur 90 jours (patterns saisonniers)
2. Optimisation conservative (marge 25-30%)
3. Rollout ultra-progressif (1% par jour)
4. Tests de charge à chaque étape

**Résultat** :
```
Avant :
- 2000 Pods sur 100 nœuds
- Utilisation moyenne : 30%
- Coût mensuel : $80000

Après :
- 2000 Pods sur 75 nœuds
- Utilisation moyenne : 55%
- Coût mensuel : $60000
- Économie : $20000/mois ($240000/an)
```

### Cas 3 : Application variable (e-commerce)

**Contexte** :
- Pics Black Friday, Noël
- Charge normale × 10 pendant les pics
- HPA configuré

**Objectif** : Optimiser pour charge normale, burst pour pics

**Actions** :
1. Analyse hors-périodes de pic
2. Requests basés sur charge normale
3. Limits généreux pour les pics
4. HPA pour scaling horizontal

**Configuration finale** :
```yaml
resources:
  requests:
    cpu: "300m"      # Charge normale
    memory: "512Mi"
  limits:
    cpu: "2"         # Absorbe pics × 6
    memory: "2Gi"    # × 4
---
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: frontend-hpa
spec:
  minReplicas: 10    # Normal
  maxReplicas: 100   # Pics
  targetCPUUtilizationPercentage: 70
```

**Résultat** :
- Charges normales : Optimisé (économie 40%)
- Pics : Scale horizontal + burst CPU/RAM
- Coût moyen réduit de 35%

## Points clés à retenir

🔑 **Les essentiels** :

1. **Mesurer avant d'optimiser** : 7-30 jours de données minimum
2. **Analyser les percentiles** : P95 pour CPU, Max pour RAM
3. **Ajouter des marges** : +20% CPU, +15% RAM (minimum)
4. **Optimiser progressivement** : Non-critique → Critique
5. **Monitoring intensif** : Post-optimisation pendant 2 semaines
6. **Automatiser** : Scripts, VPA, dashboards
7. **Documenter** : Historique des changements et justifications
8. **Adapter** : Par environnement et type de workload

## Checklist d'optimisation

✅ **Avant de commencer** :
- [ ] Prometheus + Grafana installés et fonctionnels
- [ ] Métriques collectées depuis > 7 jours
- [ ] Dashboard d'analyse créé
- [ ] Backup des configurations actuelles

✅ **Analyse** :
- [ ] Usage P95 CPU calculé par Pod
- [ ] Usage Max RAM calculé par Pod
- [ ] Ratios d'utilisation documentés
- [ ] Pods sur-alloués identifiés (>20% gaspillage)
- [ ] Pods sous-alloués identifiés (>85% usage)

✅ **Planification** :
- [ ] Ordre d'optimisation défini
- [ ] Nouvelles valeurs calculées (formules appliquées)
- [ ] Fichiers YAML préparés
- [ ] Rollback plan documenté

✅ **Déploiement** :
- [ ] Tests en non-production d'abord
- [ ] Rollout progressif (10% → 25% → 50% → 100%)
- [ ] Monitoring actif pendant le rollout
- [ ] Alertes configurées

✅ **Validation** :
- [ ] Aucune OOMKilled après 48h
- [ ] Pas de dégradation de performance
- [ ] Utilisation dans les ratios cibles (50-85%)
- [ ] Tests de charge passés

✅ **Documentation** :
- [ ] Changements documentés (annotations)
- [ ] Économies calculées et reportées
- [ ] Leçons apprises notées
- [ ] Prochaine revue planifiée (3 mois)

## Conclusion

L'optimisation des ressources est un **processus continu**, pas une action ponctuelle. C'est l'équilibre entre performance, stabilité et coût.

**Principes directeurs** :
- 📊 **Mesurer** : Données réelles > Suppositions
- 🎯 **Cibler** : Quick wins d'abord, critique en dernier
- 🐌 **Progresser** : Lent et sûr > Rapide et risqué
- 🔁 **Itérer** : Optimiser → Mesurer → Ajuster → Répéter

**Bénéfices** :
- 💰 Réduction des coûts (20-50% typique)
- 📈 Meilleure densité de Pods
- 🎯 Stabilité maintenue ou améliorée
- 🚀 Performances optimales

Avec les outils et méthodologies de ce chapitre, vous êtes maintenant équipé pour optimiser votre cluster Kubernetes de manière professionnelle et efficace.

**Prochaine étape** : Dans la section suivante (18.7), nous verrons les **Best Practices** qui consolident tout ce que nous avons appris dans ce chapitre.

---


⏭️ [Best practices](/18-gestion-des-ressources/07-best-practices.md)
