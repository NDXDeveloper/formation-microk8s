🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 18.5 PriorityClasses

## Introduction

Dans la section précédente (18.4), nous avons découvert les **QoS Classes** - un système de priorité **automatique** basé sur vos requests et limits. Maintenant, nous allons explorer les **PriorityClasses** - un système de priorité **explicite** que vous contrôlez directement.

Les PriorityClasses vous permettent de dire à Kubernetes : "Ce Pod est plus important que cet autre Pod", indépendamment de leur configuration de ressources. C'est un outil puissant pour gérer les priorités dans votre cluster.

## Différence QoS Classes vs PriorityClasses

Ces deux mécanismes sont complémentaires mais différents :

| Aspect | QoS Classes | PriorityClasses |
|--------|-------------|-----------------|
| **Type** | Automatique | Explicite |
| **Basé sur** | Requests/Limits | Valeur numérique que vous définissez |
| **Scope** | Éviction (memory pressure) | Scheduling + Éviction |
| **Configuration** | Implicite (calculé) | Explicite (vous le définissez) |
| **Nombre de niveaux** | 3 (fixe) | Illimité (vous choisissez) |
| **But principal** | Protection ressources | Priorité métier |

**Analogie** :

**QoS Classes** = Catégories de passagers dans un train
- Automatiquement assignées selon le type de billet
- 1ère classe, 2ème classe, sans billet

**PriorityClasses** = Ordre d'embarquement
- Vous décidez explicitement qui embarque en premier
- Familles avec enfants, personnes âgées, VIP, etc.

## Qu'est-ce qu'une PriorityClass ?

Une **PriorityClass** est un objet Kubernetes (cluster-scoped) qui définit un **niveau de priorité** avec :
- Un **nom** (exemple : "high-priority")
- Une **valeur numérique** (exemple : 1000)
- Une **description** optionnelle
- Un comportement de **préemption** (peut-il évincer d'autres Pods ?)

Plus la valeur numérique est **élevée**, plus la priorité est **haute**.

```
┌─────────────────────────────────────────────────────┐
│          Échelle des priorités                      │
├─────────────────────────────────────────────────────┤
│                                                     │
│  2000000000  ← system-node-critical (système)       │
│  1000000000  ← system-cluster-critical (système)    │
│                                                     │
│  ─────────── Zone personnalisée ────────────        │
│                                                     │
│  10000       ← critical-apps                        │
│  5000        ← high-priority                        │
│  1000        ← medium-priority                      │
│  100         ← low-priority                         │
│  0           ← default (pas de PriorityClass)       │
│                                                     │
│  -1000       ← very-low-priority                    │
│                                                     │
└─────────────────────────────────────────────────────┘
```

## Pourquoi utiliser les PriorityClasses ?

### Cas d'usage principaux

#### 1. Garantir le scheduling des Pods critiques

Sans PriorityClass :
```
Cluster plein (tous les nœuds saturés)
│
Nouvelle tentative de déploiement d'un Pod critique
│
❌ Pod reste en Pending - pas de place
```

Avec PriorityClass élevée :
```
Cluster plein
│
Nouveau Pod critique avec haute priorité
│
✅ Kubernetes évince un Pod basse priorité
✅ Pod critique peut démarrer
```

#### 2. Gérer les priorités métier

```yaml
# Pipeline de priorités métier
Paiement en ligne       → PriorityClass: 10000 (critique)
API client              → PriorityClass: 5000 (haute)
Tableau de bord admin   → PriorityClass: 1000 (moyenne)
Job de nettoyage nuit   → PriorityClass: 100 (basse)
```

#### 3. Optimiser l'utilisation du cluster

```
Cluster à 90% d'utilisation
│
Job batch basse priorité en cours
│
Nouveau Pod API haute priorité arrive
│
✅ Job batch évincé temporairement
✅ API démarre immédiatement
✅ Job batch reprendra plus tard
```

## Création d'une PriorityClass

### Structure de base

```yaml
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: high-priority
value: 5000
globalDefault: false
description: "Priorité élevée pour applications critiques"
```

**Champs** :
- `name` : Nom de la PriorityClass (utilisé dans les Pods)
- `value` : Valeur numérique (0 à 1000000000+)
- `globalDefault` : Si `true`, appliqué par défaut aux Pods sans PriorityClass
- `description` : Documentation (optionnel mais recommandé)

### Créer une PriorityClass

```bash
kubectl apply -f priority-class.yaml
```

### Lister les PriorityClasses

```bash
kubectl get priorityclasses
```

Résultat :
```
NAME                      VALUE        GLOBAL-DEFAULT   AGE
high-priority             5000         false            10m
medium-priority           1000         false            10m
low-priority              100          false            10m
system-cluster-critical   1000000000   false            50d
system-node-critical      2000000000   false            50d
```

## Utiliser une PriorityClass dans un Pod

### Syntaxe de base

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: mon-pod-prioritaire
spec:
  priorityClassName: high-priority  # Référence la PriorityClass
  containers:
  - name: nginx
    image: nginx
```

C'est tout ! Le Pod hérite de la priorité 5000.

### Avec un Deployment

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api-critique
spec:
  replicas: 3
  selector:
    matchLabels:
      app: api-critique
  template:
    metadata:
      labels:
        app: api-critique
    spec:
      priorityClassName: high-priority  # Tous les Pods du Deployment
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
```

### Vérifier la priorité d'un Pod

```bash
kubectl get pod mon-pod-prioritaire -o jsonpath='{.spec.priority}'
```

Résultat : `5000`

Ou avec describe :
```bash
kubectl describe pod mon-pod-prioritaire | grep -i priority
```

## Préemption (Preemption)

La **préemption** est le mécanisme par lequel Kubernetes peut **évincer** (tuer) des Pods de priorité inférieure pour faire de la place à des Pods de priorité supérieure.

### Configuration de la préemption

```yaml
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: high-priority-preempting
value: 5000
preemptionPolicy: PreemptLowerPriority  # Peut évincer (défaut)
description: "Haute priorité avec préemption activée"
```

**Valeurs possibles** :
- `PreemptLowerPriority` (défaut) : Peut évincer des Pods de priorité inférieure
- `Never` : Ne peut pas évincer d'autres Pods

### Comment fonctionne la préemption ?

**Scénario** :

```
1. Cluster plein, tous les nœuds à capacité

2. Pods existants :
   ├─ Pod A (priority: 100)  - 2 CPU
   ├─ Pod B (priority: 500)  - 2 CPU
   └─ Pod C (priority: 1000) - 2 CPU

3. Nouveau Pod X arrive (priority: 5000) - demande 2 CPU

4. Scheduler analyse :
   - Besoin : 2 CPU
   - Aucun nœud n'a 2 CPU libres
   - Pod X a priorité 5000
   - Pods de priorité inférieure disponibles

5. Décision : Évincer Pod A (priorité la plus basse)
   ├─ Pod A est gracefully terminated (30s par défaut)
   └─ Libère 2 CPU

6. Pod X est schedulé sur le nœud libéré ✅
```

### Processus de préemption en détail

```
┌─────────────────────────────────────────────────┐
│      Processus de préemption                    │
├─────────────────────────────────────────────────┤
│                                                 │
│  1. Pod haute priorité créé → Pending           │
│                                                 │
│  2. Scheduler : pas de nœud disponible ?        │
│                                                 │
│  3. Activation préemption :                     │
│     • Identifier les nœuds avec Pods            │
│       de priorité inférieure                    │
│     • Calculer quelle combinaison évincer       │
│     • Choisir la solution minimale              │
│                                                 │
│  4. Victims (victimes) sélectionnées :          │
│     • Pods de priorité inférieure               │
│     • graceful termination (terminationGrace)   │
│     • PodDisruptionBudget respecté si possible  │
│                                                 │
│  5. Attente libération des ressources           │
│                                                 │
│  6. Pod haute priorité schedulé ✅              │
│                                                 │
└─────────────────────────────────────────────────┘
```

### Désactiver la préemption

Pour certains cas, vous voulez une haute priorité **sans** évincer d'autres Pods :

```yaml
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: high-priority-no-preempt
value: 5000
preemptionPolicy: Never  # Ne peut PAS évincer
description: "Haute priorité mais sans préemption"
```

**Cas d'usage** :
- Pods importants mais qui peuvent attendre
- Éviter de perturber des workloads sensibles
- Environnements où la stabilité prime

## Exemples de PriorityClasses

### Exemple 1 : Hiérarchie standard

```yaml
# Basse priorité - Jobs batch, nettoyage
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: low-priority
value: 100
globalDefault: false
description: "Jobs batch et tâches non critiques"
---
# Priorité par défaut - Applications standard
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: default-priority
value: 1000
globalDefault: true  # Appliqué par défaut
description: "Priorité par défaut pour la plupart des applications"
---
# Haute priorité - Services critiques
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: high-priority
value: 5000
globalDefault: false
description: "Services critiques - APIs, bases de données"
---
# Très haute priorité - Infrastructure core
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: critical-priority
value: 10000
globalDefault: false
description: "Infrastructure critique - monitoring, logging"
```

### Exemple 2 : Par type d'application

```yaml
# Paiements et transactions
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: payment-priority
value: 10000
preemptionPolicy: PreemptLowerPriority
description: "Services de paiement - priorité maximale"
---
# APIs client
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: api-priority
value: 5000
description: "APIs exposées aux clients"
---
# Services internes
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: internal-priority
value: 2000
description: "Services internes et back-office"
---
# Jobs de reporting
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: reporting-priority
value: 500
preemptionPolicy: Never
description: "Jobs de reporting - peuvent attendre"
---
# Jobs de nettoyage nocturne
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: cleanup-priority
value: 100
preemptionPolicy: Never
description: "Nettoyage et maintenance - basse priorité"
```

### Exemple 3 : Par environnement

```yaml
# Production - haute priorité
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: production-priority
value: 8000
description: "Workloads de production"
---
# Staging - priorité moyenne
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: staging-priority
value: 2000
description: "Environnement de staging"
---
# Development - basse priorité
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: development-priority
value: 500
preemptionPolicy: Never
description: "Environnement de développement"
```

## GlobalDefault : La PriorityClass par défaut

### Définir une priorité par défaut

```yaml
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: default-app-priority
value: 1000
globalDefault: true  # Priorité par défaut
description: "Priorité appliquée automatiquement si non spécifiée"
```

**Comportement** :
- Tous les Pods **sans** `priorityClassName` utilisent cette priorité
- Un seul `globalDefault: true` autorisé dans le cluster
- Évite d'avoir des Pods à priorité 0 par défaut

### Sans globalDefault

```yaml
# Pod sans priorityClassName
apiVersion: v1
kind: Pod
metadata:
  name: pod-sans-priorite
spec:
  containers:
  - name: nginx
    image: nginx
  # Pas de priorityClassName
  # Priority = 0 (plus basse possible)
```

### Avec globalDefault

```yaml
# Même Pod, mais globalDefault existe
apiVersion: v1
kind: Pod
metadata:
  name: pod-sans-priorite
spec:
  containers:
  - name: nginx
    image: nginx
  # Pas de priorityClassName
  # Priority = 1000 (hérite de globalDefault)
```

## PriorityClasses système

Kubernetes préinstalle deux PriorityClasses système :

### system-node-critical

```yaml
name: system-node-critical
value: 2000000000  # 2 milliards
```

**Usage** :
- Composants critiques du nœud
- kube-proxy
- CNI plugins
- Ne jamais utiliser pour vos applications !

### system-cluster-critical

```yaml
name: system-cluster-critical
value: 1000000000  # 1 milliard
```

**Usage** :
- Composants critiques du cluster
- CoreDNS
- kube-apiserver
- etcd
- Ne jamais utiliser pour vos applications !

### Réservation système

⚠️ **Important** : N'utilisez jamais de valeurs > 1000000000 pour vos applications.

```yaml
# ❌ MAUVAIS - Ne jamais faire
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: my-app
value: 2000000000  # Conflit avec système !
```

```yaml
# ✅ BON - Rester sous 1 million
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: my-app
value: 10000
```

## Interaction PriorityClass + QoS Class

Les deux systèmes travaillent ensemble pour déterminer l'ordre d'éviction :

### Matrice de priorité combinée

```
┌────────────────────────────────────────────────────┐
│  Ordre d'éviction (du premier au dernier)          │
├────────────────────────────────────────────────────┤
│                                                    │
│  1️⃣  BestEffort + Low Priority                    │
│  2️⃣  BestEffort + Default Priority                │
│  3️⃣  BestEffort + High Priority                   │
│                                                    │
│  4️⃣  Burstable + Low Priority                     │
│  5️⃣  Burstable + Default Priority                 │
│  6️⃣  Burstable + High Priority                    │
│                                                    │
│  7️⃣  Guaranteed + Low Priority                    │
│  8️⃣  Guaranteed + Default Priority                │
│  9️⃣  Guaranteed + High Priority                   │
│                                                    │
└────────────────────────────────────────────────────┘
```

### Exemple concret

```yaml
# Pod 1 : Protection maximale
apiVersion: v1
kind: Pod
metadata:
  name: critical-db
spec:
  priorityClassName: critical-priority  # 10000
  containers:
  - name: postgres
    resources:
      limits:
        memory: "4Gi"
        cpu: "2"
      # QoS: Guaranteed
# Priorité combinée : Guaranteed + 10000 → Éviction en dernier

---
# Pod 2 : Protection moyenne
apiVersion: v1
kind: Pod
metadata:
  name: api-backend
spec:
  priorityClassName: high-priority  # 5000
  containers:
  - name: api
    resources:
      requests:
        memory: "512Mi"
        cpu: "500m"
      limits:
        memory: "1Gi"
        cpu: "1"
      # QoS: Burstable
# Priorité combinée : Burstable + 5000 → Éviction en milieu

---
# Pod 3 : Éviction rapide
apiVersion: v1
kind: Pod
metadata:
  name: batch-job
spec:
  priorityClassName: low-priority  # 100
  containers:
  - name: worker
    resources:
      requests:
        memory: "256Mi"
        cpu: "250m"
      limits:
        memory: "512Mi"
        cpu: "500m"
      # QoS: Burstable
# Priorité combinée : Burstable + 100 → Éviction rapide
```

## Cas d'usage pratiques

### Cas 1 : Environnement multi-tenant

```yaml
# Équipe A - Client Premium
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: team-a-priority
value: 8000
description: "Équipe A - Client Premium - Haute priorité"
---
# Équipe B - Client Standard
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: team-b-priority
value: 3000
description: "Équipe B - Client Standard - Priorité normale"
---
# Équipe C - Client Freemium
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: team-c-priority
value: 1000
preemptionPolicy: Never
description: "Équipe C - Freemium - Basse priorité sans préemption"
```

### Cas 2 : Pipeline CI/CD

```yaml
# Déploiements production
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: production-deployment
value: 9000
preemptionPolicy: PreemptLowerPriority
description: "Déploiements production - priorité maximale"
---
# Tests d'intégration
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: integration-tests
value: 2000
description: "Tests d'intégration - priorité moyenne"
---
# Builds de développement
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: dev-builds
value: 500
preemptionPolicy: Never
description: "Builds développement - basse priorité"
```

### Cas 3 : E-commerce

```yaml
# Checkout et paiement
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: checkout-priority
value: 10000
description: "Checkout et paiement - critique business"
---
# Catalogue produits
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: catalog-priority
value: 5000
description: "Catalogue produits - haute priorité"
---
# Recommandations
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: recommendations-priority
value: 2000
description: "Moteur de recommandations - priorité normale"
---
# Analytics
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: analytics-priority
value: 500
preemptionPolicy: Never
description: "Analytics et rapports - basse priorité"
```

### Cas 4 : Services de données

```yaml
# Bases de données de production
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: database-priority
value: 9000
description: "Bases de données production - très haute priorité"
---
# Caches
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: cache-priority
value: 7000
description: "Redis/Memcached - haute priorité"
---
# ETL temps réel
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: realtime-etl-priority
value: 4000
description: "ETL temps réel - priorité moyenne-haute"
---
# ETL batch
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: batch-etl-priority
value: 1000
preemptionPolicy: Never
description: "ETL batch - peut attendre"
```

## PodDisruptionBudget et PriorityClass

Les **PodDisruptionBudgets** (PDB) interagissent avec la préemption :

```yaml
# PDB pour protéger une application critique
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: api-pdb
spec:
  minAvailable: 2  # Toujours 2 Pods minimum
  selector:
    matchLabels:
      app: api-critique
---
# Deployment avec haute priorité ET PDB
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api-critique
spec:
  replicas: 3
  selector:
    matchLabels:
      app: api-critique
  template:
    metadata:
      labels:
        app: api-critique
    spec:
      priorityClassName: high-priority
      containers:
      - name: api
        image: my-api:v1.0
```

**Comportement** :
- Préemption respecte le PDB si possible
- Si impossible de respecter le PDB, le Pod haute priorité reste Pending
- Protection supplémentaire contre les évictions

## Bonnes pratiques

### 1. Définir une hiérarchie claire

✅ **Recommandation** : Maximum 5-7 niveaux de priorité

```yaml
# Hiérarchie simple et claire
critical:  10000  # Infrastructure core
high:      5000   # Services critiques métier
medium:    2000   # Services standard
default:   1000   # Défaut (globalDefault)
low:       500    # Jobs non critiques
very-low:  100    # Maintenance, nettoyage
```

❌ **À éviter** : Trop de niveaux (confusion)
```yaml
# 15 niveaux différents = complexité inutile
ultra-critical: 20000
super-critical: 18000
very-critical:  16000
...
```

### 2. Documenter chaque PriorityClass

✅ **Bonne pratique** :

```yaml
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: high-priority
  annotations:
    documentation: "https://wiki.company.com/k8s-priorities"
    owner: "platform-team@company.com"
    use-cases: "APIs critiques, bases de données, paiements"
    approval-required: "yes"
value: 5000
description: |
  Haute priorité pour les services critiques métier.
  Nécessite l'approbation de l'équipe plateforme.
  Exemples : API paiement, API client, base de données principale.
```

### 3. Utiliser globalDefault

✅ **Recommandation** : Définir une priorité par défaut raisonnable

```yaml
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: default-app-priority
value: 1000
globalDefault: true
description: "Priorité par défaut - applications standard"
```

Évite d'avoir des applications à priorité 0.

### 4. Préemption avec précaution

✅ **Pattern recommandé** :

```yaml
# Haute priorité : préemption activée (critique)
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: critical-priority
value: 10000
preemptionPolicy: PreemptLowerPriority  # Peut évincer
---
# Priorité moyenne : pas de préemption (gentil)
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: medium-priority
value: 2000
preemptionPolicy: Never  # Ne peut pas évincer
```

**Principe** : Plus la priorité est haute, plus on peut évincer.

### 5. Combiner avec QoS Guaranteed

✅ **Protection maximale** :

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: ultra-protected
spec:
  priorityClassName: critical-priority  # Haute priorité
  containers:
  - name: app
    resources:
      limits:
        memory: "2Gi"
        cpu: "2"
      # requests = limits automatiquement
      # QoS : Guaranteed
```

Combinaison idéale pour les applications les plus critiques.

### 6. Monitoring des priorités

✅ **Métriques à surveiller** :

```yaml
# Prometheus queries utiles
- Nombre de Pods par PriorityClass
- Nombre de préemptions par heure
- Pods en Pending à cause de la préemption
- Distribution des priorités par namespace
```

### 7. Processus d'approbation

✅ **Gouvernance** :

```yaml
# Règle : Les PriorityClasses > 5000 nécessitent approbation
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: ultra-high-priority
  annotations:
    approval-required: "yes"
    approved-by: "CTO"
    approval-date: "2025-10-01"
    justification: "Service de paiement - critique business"
value: 10000
```

### 8. Testing de la préemption

✅ **Tester avant la production** :

```bash
# 1. Créer un Pod basse priorité
kubectl run low-priority-pod \
  --image=nginx \
  --overrides='{"spec":{"priorityClassName":"low-priority"}}'

# 2. Vérifier qu'il tourne
kubectl get pods

# 3. Créer un Pod haute priorité qui nécessite préemption
kubectl run high-priority-pod \
  --image=nginx \
  --overrides='{"spec":{"priorityClassName":"high-priority"}}'

# 4. Observer la préemption
kubectl get events --sort-by='.lastTimestamp' | grep -i preempt
```

## Commandes utiles

### Lister les PriorityClasses

```bash
# Toutes les PriorityClasses
kubectl get priorityclasses

# Format détaillé
kubectl get priorityclasses -o wide

# Avec valeurs
kubectl get priorityclasses -o custom-columns=NAME:.metadata.name,VALUE:.value,GLOBAL:.globalDefault
```

### Voir les détails d'une PriorityClass

```bash
kubectl describe priorityclass high-priority
```

### Trouver les Pods par PriorityClass

```bash
# Pods utilisant une PriorityClass spécifique
kubectl get pods --all-namespaces -o json | \
  jq -r '.items[] | select(.spec.priorityClassName=="high-priority") | .metadata.name'

# Pods avec leur priorité
kubectl get pods -o custom-columns=NAME:.metadata.name,PRIORITY:.spec.priority,PRIORITYCLASS:.spec.priorityClassName
```

### Voir les événements de préemption

```bash
# Événements de préemption récents
kubectl get events --all-namespaces --field-selector reason=Preempted

# Détails des préemptions
kubectl get events --all-namespaces --sort-by='.lastTimestamp' | grep -i preempt
```

### Modifier une PriorityClass

```bash
# Éditer
kubectl edit priorityclass high-priority
```

⚠️ **Attention** : Modifier une PriorityClass n'affecte pas les Pods existants, seulement les nouveaux.

### Supprimer une PriorityClass

```bash
kubectl delete priorityclass low-priority
```

⚠️ **Attention** : Ne pas supprimer une PriorityClass utilisée par des Pods existants.

## Debugging et diagnostic

### Problème 1 : Pod en Pending malgré haute priorité

**Symptôme** :
```bash
kubectl get pods
# NAME                READY   STATUS    RESTARTS   AGE
# high-priority-pod   0/1     Pending   0          5m
```

**Diagnostic** :
```bash
kubectl describe pod high-priority-pod
```

**Causes possibles** :
1. Préemption désactivée (`preemptionPolicy: Never`)
2. PodDisruptionBudget empêche la préemption
3. Pas de victimes disponibles (tous les Pods ont priorité égale ou supérieure)
4. Contraintes d'affinité impossibles à satisfaire

**Solutions** :
- Vérifier la politique de préemption
- Vérifier les PDB
- Augmenter la capacité du cluster
- Réduire les contraintes d'affinité

### Problème 2 : PriorityClass non reconnue

**Symptôme** :
```
Error: priorityclass.scheduling.k8s.io "my-priority" not found
```

**Diagnostic** :
```bash
kubectl get priorityclasses | grep my-priority
```

**Solutions** :
1. Créer la PriorityClass d'abord
2. Vérifier l'orthographe du nom
3. Vérifier que c'est un objet cluster-scoped (pas namespace-scoped)

### Problème 3 : Préemptions excessives

**Symptôme** :
```bash
kubectl get events | grep -i preempt
# Nombreuses préemptions chaque minute
```

**Diagnostic** :
```bash
# Voir quels Pods sont préemptés
kubectl get events --sort-by='.lastTimestamp' | grep -i preempt

# Identifier les Pods "agressifs"
kubectl get pods -o json | \
  jq -r '.items[] | select(.spec.priority > 5000) | .metadata.name'
```

**Solutions** :
1. Revoir la distribution des priorités
2. Désactiver la préemption pour certaines PriorityClasses
3. Augmenter les ressources du cluster
4. Utiliser des PodDisruptionBudgets

### Problème 4 : GlobalDefault non appliqué

**Symptôme** :
```bash
# Pod créé sans priorityClassName
kubectl get pod my-pod -o jsonpath='{.spec.priority}'
# Résultat : 0 (au lieu de la valeur globalDefault)
```

**Diagnostic** :
```bash
# Vérifier qu'une PriorityClass a globalDefault: true
kubectl get priorityclasses -o jsonpath='{.items[?(@.globalDefault==true)].metadata.name}'
```

**Causes possibles** :
1. Aucune PriorityClass avec `globalDefault: true`
2. Plusieurs PriorityClasses avec `globalDefault: true` (conflit)

**Solutions** :
1. Créer/Modifier une PriorityClass avec `globalDefault: true`
2. Vérifier qu'une seule a ce flag

## Exemples complets

### Exemple 1 : Plateforme e-commerce complète

```yaml
# Infrastructure
---
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: infra-priority
value: 9500
description: "Infrastructure - Monitoring, Logging, Ingress"
---
# Paiement
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: payment-priority
value: 10000
preemptionPolicy: PreemptLowerPriority
description: "Services de paiement - priorité absolue"
---
# Checkout
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: checkout-priority
value: 8000
description: "Processus de checkout"
---
# APIs client
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: customer-api-priority
value: 6000
description: "APIs exposées aux clients"
---
# Services internes
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: internal-priority
value: 3000
globalDefault: true
description: "Services internes - priorité par défaut"
---
# Batch nocturne
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: batch-priority
value: 500
preemptionPolicy: Never
description: "Jobs batch - s'exécutent la nuit"
---
# Déploiements du Deployment
apiVersion: apps/v1
kind: Deployment
metadata:
  name: payment-service
spec:
  replicas: 5
  template:
    spec:
      priorityClassName: payment-priority
      containers:
      - name: payment
        image: payment:v1.0
        resources:
          limits:
            memory: "2Gi"
            cpu: "1"
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: checkout-service
spec:
  replicas: 3
  template:
    spec:
      priorityClassName: checkout-priority
      containers:
      - name: checkout
        image: checkout:v1.0
        resources:
          requests:
            memory: "512Mi"
            cpu: "500m"
          limits:
            memory: "1Gi"
            cpu: "1"
```

### Exemple 2 : Lab de formation

```yaml
# Formateurs (priorité élevée)
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: instructor-priority
value: 5000
description: "Demos et environnements formateurs"
---
# Étudiants (priorité normale)
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: student-priority
value: 1000
globalDefault: true
preemptionPolicy: Never
description: "Environnements étudiants"
---
# Exercices temporaires
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: exercise-priority
value: 500
preemptionPolicy: Never
description: "Exercices pratiques temporaires"
```

## Points clés à retenir

🔑 **Les essentiels** :

1. **PriorityClass** = priorité **explicite** que vous définissez
2. **Valeur numérique** : plus haute = plus prioritaire
3. **Scheduling** : Pods haute priorité peuvent évincer les basses priorités
4. **Préemption** : `PreemptLowerPriority` ou `Never`
5. **GlobalDefault** : définir une priorité par défaut
6. **< 1 million** : ne jamais dépasser (réservé au système)
7. **5-7 niveaux** maximum pour la clarté
8. **Combinaison** QoS + PriorityClass = protection maximale

## Récapitulatif du chapitre 18

Nous avons maintenant couvert tous les aspects de la gestion des ressources :

| Section | Concept | Rôle |
|---------|---------|------|
| **18.1** | Requests/Limits | Définir les ressources des conteneurs |
| **18.2** | Resource Quotas | Limiter le total d'un namespace |
| **18.3** | LimitRanges | Contraindre les valeurs individuelles |
| **18.4** | QoS Classes | Priorité automatique (ressources) |
| **18.5** | PriorityClasses | Priorité explicite (métier) |

**Ces 5 mécanismes ensemble assurent** :
- ✅ Allocation efficace des ressources
- ✅ Protection des applications critiques
- ✅ Stabilité du cluster
- ✅ Prévisibilité et gouvernance
- ✅ Optimisation des coûts

## Conclusion

Les PriorityClasses sont un outil puissant pour gérer les priorités métier dans votre cluster. Contrairement aux QoS Classes qui sont automatiques, les PriorityClasses vous donnent un contrôle explicite sur ce qui est critique et ce qui peut attendre.

**En pratique** :
- Définissez une hiérarchie simple (5-7 niveaux)
- Documentez chaque PriorityClass
- Utilisez la préemption avec précaution
- Combinez avec QoS Guaranteed pour protection maximale
- Monitorez les préemptions
- Révisez régulièrement vos choix de priorités

**Prochaine étape** : Dans la section suivante (18.6), nous verrons l'**Optimisation des ressources** pour tirer le meilleur parti de votre configuration.

---


⏭️ [Optimisation des ressources](/18-gestion-des-ressources/06-optimisation-des-ressources.md)
