🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 18.4 QoS Classes

## Introduction

Nous avons maintenant couvert tous les mécanismes de configuration des ressources :
- **18.1 - Requests et Limits** : définir les ressources des conteneurs
- **18.2 - Resource Quotas** : limiter le total d'un namespace
- **18.3 - LimitRanges** : contraindre les valeurs individuelles

Mais que se passe-t-il quand un nœud Kubernetes manque de ressources ? Quels Pods sont sacrifiés en premier ? C'est ici qu'interviennent les **QoS Classes** (Quality of Service Classes).

Les QoS Classes sont un système de **priorité automatique** que Kubernetes attribue à chaque Pod en fonction de la façon dont vous définissez ses requests et limits. Ce n'est pas quelque chose que vous configurez directement - c'est **calculé automatiquement** par Kubernetes.

## Qu'est-ce qu'une QoS Class ?

Une QoS Class (Classe de Qualité de Service) est une **étiquette de priorité** que Kubernetes assigne automatiquement à chaque Pod. Cette étiquette détermine **l'ordre dans lequel les Pods seront évincés** (tués) si le nœud manque de ressources.

**Analogie** : Pensez à un bateau qui prend l'eau et coule lentement.

Le capitaine doit décider qui évacuer en premier :
- **Passagers VIP** (billets première classe) : évacués en dernier - **Guaranteed**
- **Passagers standard** (billets économiques) : évacués en milieu - **Burstable**
- **Passagers clandestins** (sans billet) : évacués en premier - **BestEffort**

Dans Kubernetes :
- **Bateau** = Nœud Kubernetes
- **Eau qui monte** = Pression sur les ressources (RAM, CPU)
- **Passagers** = Pods
- **Ordre d'évacuation** = Ordre d'éviction des Pods

## Pourquoi les QoS Classes sont-elles importantes ?

### Le problème : Pression sur les ressources

Imaginons un nœud avec 16 Gi de RAM qui héberge plusieurs Pods. Si la consommation totale de RAM approche 16 Gi, le nœud entre en **pression mémoire** (memory pressure).

**Que fait Kubernetes ?**

Il doit **tuer des Pods** pour libérer de la RAM et éviter que le nœud entier ne plante. Mais lesquels ?

❌ **Sans système de priorité** :
- Kubernetes tue des Pods au hasard
- Risque de tuer des Pods critiques
- Instabilité imprévisible

✅ **Avec QoS Classes** :
- Kubernetes tue d'abord les Pods moins critiques (BestEffort)
- Puis les Pods moyennement critiques (Burstable)
- En dernier les Pods critiques (Guaranteed)
- Prédictibilité et stabilité

## Les trois QoS Classes

Kubernetes définit exactement **trois classes** de QoS, de la moins prioritaire à la plus prioritaire :

```
┌─────────────────────────────────────────────────────────┐
│            Ordre d'éviction (Memory Pressure)           │
├─────────────────────────────────────────────────────────┤
│                                                         │
│  1️⃣  BestEffort    ←  Tués en PREMIER                  │
│      (Priorité la plus basse)                           │
│                                                         │
│  2️⃣  Burstable     ←  Tués en DEUXIÈME                 │
│      (Priorité moyenne)                                 │
│                                                         │
│  3️⃣  Guaranteed    ←  Tués en DERNIER                  │
│      (Priorité la plus haute)                           │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

### Vue d'ensemble des classes

| QoS Class | Priorité | Conditions | Usage recommandé |
|-----------|----------|------------|------------------|
| **BestEffort** | 🔴 Basse | Aucune requests/limits | Tests, jamais en prod |
| **Burstable** | 🟡 Moyenne | Requests ≠ Limits | Cas général |
| **Guaranteed** | 🟢 Haute | Requests = Limits | Applications critiques |

## 1. BestEffort - La classe la plus basse

### Conditions pour être BestEffort

Un Pod est classé **BestEffort** quand **aucun** de ses conteneurs ne définit de requests ni de limits pour CPU et mémoire.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: besteffort-pod
spec:
  containers:
  - name: nginx
    image: nginx
    # Aucune section resources - BestEffort !
```

### Caractéristiques

**Avantages** :
- 🟢 Simple (pas besoin de définir les resources)
- 🟢 Peut utiliser toutes les ressources disponibles sur le nœud
- 🟢 Flexible

**Inconvénients** :
- 🔴 **Premier à être tué** en cas de pression mémoire
- 🔴 Aucune garantie de ressources
- 🔴 Performances imprévisibles
- 🔴 Peut être éjecté à tout moment

### Comportement en cas de pression mémoire

Quand un nœud manque de RAM :
1. Kubernetes identifie tous les Pods BestEffort
2. Calcule quel Pod BestEffort consomme le plus
3. **Tue ce Pod en premier**
4. Continue jusqu'à ce que la pression disparaisse

**Schéma** :
```
Nœud en pression mémoire (95% RAM utilisée)
│
├─ Pod A (Guaranteed) → Protégé ✅
├─ Pod B (Burstable)  → Protégé ✅
├─ Pod C (BestEffort) → TUÉ 💀
└─ Pod D (BestEffort) → TUÉ 💀
```

### Quand utiliser BestEffort ?

✅ **Acceptable** :
- Tests rapides en développement
- Expérimentations locales
- Scripts one-shot non critiques

❌ **À éviter** :
- **JAMAIS en production**
- Applications importantes
- Services avec SLA
- Tout ce qui doit être stable

### Exemple réel

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: test-rapide
  labels:
    qos-class: besteffort
spec:
  containers:
  - name: busybox
    image: busybox
    command: ["sleep", "3600"]
    # Pas de resources = BestEffort
```

**Vérification** :
```bash
kubectl get pod test-rapide -o jsonpath='{.status.qosClass}'
# Résultat : BestEffort
```

## 2. Burstable - La classe intermédiaire

### Conditions pour être Burstable

Un Pod est classé **Burstable** dans les cas suivants :
1. Au moins un conteneur a des requests **ou** des limits
2. Requests ≠ Limits (au moins pour une ressource)
3. Pas tous les conteneurs n'ont requests = limits pour tout

C'est le **cas le plus courant** et le plus flexible.

### Exemples de configurations Burstable

**Cas 1 : Requests sans limits**
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: burstable-pod-1
spec:
  containers:
  - name: app
    image: my-app
    resources:
      requests:
        memory: "256Mi"
        cpu: "250m"
      # Pas de limits - Burstable !
```

**Cas 2 : Limits sans requests**
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: burstable-pod-2
spec:
  containers:
  - name: app
    image: my-app
    resources:
      limits:
        memory: "1Gi"
        cpu: "1"
      # Pas de requests - Burstable !
```

**Cas 3 : Requests ≠ Limits**
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: burstable-pod-3
spec:
  containers:
  - name: app
    image: my-app
    resources:
      requests:
        memory: "256Mi"
        cpu: "250m"
      limits:
        memory: "512Mi"   # Différent de requests
        cpu: "500m"       # Différent de requests
      # Burstable !
```

**Cas 4 : Pod multi-conteneurs mixte**
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: burstable-pod-4
spec:
  containers:
  - name: app
    image: my-app
    resources:
      requests:
        memory: "512Mi"
        cpu: "500m"
      limits:
        memory: "512Mi"   # Égal pour ce conteneur
        cpu: "500m"
  - name: sidecar
    image: nginx
    resources:
      requests:
        memory: "128Mi"
        cpu: "100m"
      limits:
        memory: "256Mi"   # Différent pour ce conteneur
        cpu: "200m"
      # Pod global = Burstable car au moins un conteneur a requests ≠ limits
```

### Caractéristiques

**Avantages** :
- 🟢 Bon compromis flexibilité/stabilité
- 🟢 Peut "burst" (utiliser plus que les requests) si ressources disponibles
- 🟢 Garanties minimales avec requests
- 🟢 Protection contre sur-consommation avec limits
- 🟢 Meilleure priorité que BestEffort

**Inconvénients** :
- 🟡 Peut être tué en cas de pression mémoire (après BestEffort)
- 🟡 Moins de garanties que Guaranteed

### Comportement en cas de pression mémoire

Quand un nœud manque de RAM **après avoir tué tous les BestEffort** :
1. Kubernetes identifie tous les Pods Burstable
2. Calcule pour chaque Pod : `(usage actuel - requests) / requests`
3. **Tue d'abord les Pods qui dépassent le plus leurs requests**
4. Continue jusqu'à ce que la pression disparaisse

**Exemple** :
```
Pods Burstable sur le nœud :
├─ Pod A : requests 512Mi, usage 800Mi → dépassement 56%
├─ Pod B : requests 1Gi, usage 1.2Gi  → dépassement 20%
└─ Pod C : requests 256Mi, usage 400Mi → dépassement 56%

Ordre d'éviction : Pod A ou Pod C d'abord (dépassement le plus élevé)
```

### Quand utiliser Burstable ?

✅ **Recommandé pour** :
- **La majorité des applications** en production
- Services web classiques
- APIs backend
- Workers de traitement
- Applications dont la charge varie

C'est le **choix par défaut** pour la plupart des workloads.

### Exemple réel

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api-backend
spec:
  replicas: 3
  selector:
    matchLabels:
      app: api-backend
  template:
    metadata:
      labels:
        app: api-backend
    spec:
      containers:
      - name: api
        image: my-api:v1.0
        resources:
          requests:
            memory: "512Mi"
            cpu: "500m"
          limits:
            memory: "1Gi"    # Peut burst jusqu'à 1Gi
            cpu: "1"         # Peut burst jusqu'à 1 CPU
        # QoS = Burstable
```

**Vérification** :
```bash
kubectl get pods -l app=api-backend -o jsonpath='{.items[0].status.qosClass}'
# Résultat : Burstable
```

## 3. Guaranteed - La classe la plus haute

### Conditions pour être Guaranteed

Un Pod est classé **Guaranteed** quand **tous** ses conteneurs respectent ces conditions **strictes** :

1. ✅ Chaque conteneur doit avoir requests **ET** limits pour CPU et mémoire
2. ✅ Requests CPU = Limits CPU (pour chaque conteneur)
3. ✅ Requests Memory = Limits Memory (pour chaque conteneur)

C'est la configuration la **plus stricte** mais aussi la **plus protégée**.

### Configuration correcte Guaranteed

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: guaranteed-pod
spec:
  containers:
  - name: app
    image: my-app
    resources:
      requests:
        memory: "1Gi"
        cpu: "1"
      limits:
        memory: "1Gi"    # Exactement égal à requests
        cpu: "1"         # Exactement égal à requests
      # Guaranteed ✅
```

### Configurations incorrectes (non-Guaranteed)

**Erreur 1 : Manque CPU limits**
```yaml
resources:
  requests:
    memory: "1Gi"
    cpu: "1"
  limits:
    memory: "1Gi"
    # Manque cpu limits - Burstable ❌
```

**Erreur 2 : Requests ≠ Limits**
```yaml
resources:
  requests:
    memory: "1Gi"
    cpu: "1"
  limits:
    memory: "2Gi"    # Différent - Burstable ❌
    cpu: "1"
```

**Erreur 3 : Manque requests**
```yaml
resources:
  limits:
    memory: "1Gi"
    cpu: "1"
    # Manque requests - Burstable ❌
```

### Forme courte (recommandée)

Kubernetes permet une forme plus courte. Si vous définissez **seulement** les limits, les requests sont automatiquement égales :

```yaml
resources:
  limits:
    memory: "1Gi"
    cpu: "1"
  # requests seront automatiquement = limits
  # Guaranteed ✅
```

C'est **équivalent** à :
```yaml
resources:
  requests:
    memory: "1Gi"
    cpu: "1"
  limits:
    memory: "1Gi"
    cpu: "1"
```

### Pod multi-conteneurs Guaranteed

**Tous les conteneurs** doivent être Guaranteed :

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: guaranteed-multi
spec:
  containers:
  - name: app
    image: my-app
    resources:
      limits:
        memory: "2Gi"
        cpu: "1"
  - name: sidecar
    image: nginx
    resources:
      limits:
        memory: "512Mi"
        cpu: "500m"
  # Les deux conteneurs sont Guaranteed
  # Donc le Pod est Guaranteed ✅
```

### Caractéristiques

**Avantages** :
- 🟢 **Priorité maximale** - tué en dernier
- 🟢 Performances prévisibles et constantes
- 🟢 Ressources garanties et fixes
- 🟢 Idéal pour applications critiques
- 🟢 Pas de throttling CPU surprise
- 🟢 Meilleure stabilité

**Inconvénients** :
- 🔴 Moins de flexibilité (pas de burst)
- 🔴 Peut "gaspiller" des ressources si sous-utilisé
- 🔴 Coût d'infrastructure potentiellement plus élevé
- 🔴 Nécessite de bien dimensionner les ressources

### Comportement en cas de pression mémoire

Quand un nœud manque de RAM **après avoir tué tous les BestEffort et Burstable** :
1. Kubernetes identifie tous les Pods Guaranteed
2. Calcule pour chaque Pod : `usage actuel`
3. **Tue en dernier recours le Pod qui consomme le plus**
4. C'est la situation d'urgence absolue

**Schéma** :
```
Pression mémoire extrême (98% RAM)
│
Phase 1 : Tue tous les BestEffort    💀
Phase 2 : Tue tous les Burstable     💀
Phase 3 : Tue Guaranteed si nécessaire 💀 (dernier recours)
```

### Quand utiliser Guaranteed ?

✅ **Recommandé pour** :
- Bases de données (PostgreSQL, MySQL, MongoDB)
- Caches critiques (Redis, Memcached)
- Files de messages (RabbitMQ, Kafka)
- Applications avec SLA strict
- Services core infrastructure
- Systèmes de monitoring critiques

❌ **Éviter pour** :
- Applications dont la charge fluctue beaucoup
- Workers batch occasionnels
- Services non critiques
- Environnements de développement

### Exemple réel

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: postgresql
spec:
  serviceName: postgresql
  replicas: 1
  selector:
    matchLabels:
      app: postgresql
  template:
    metadata:
      labels:
        app: postgresql
    spec:
      containers:
      - name: postgresql
        image: postgres:15
        resources:
          limits:
            memory: "2Gi"
            cpu: "2"
          # requests = limits automatiquement
          # QoS = Guaranteed ✅
        env:
        - name: POSTGRES_PASSWORD
          value: "secret"
```

**Vérification** :
```bash
kubectl get pods -l app=postgresql -o jsonpath='{.items[0].status.qosClass}'
# Résultat : Guaranteed
```

## Comparaison des trois classes

### Tableau récapitulatif

| Aspect | BestEffort | Burstable | Guaranteed |
|--------|------------|-----------|------------|
| **Priorité d'éviction** | 1️⃣ Premier | 2️⃣ Deuxième | 3️⃣ Dernier |
| **Configuration** | Aucune resources | Requests ≠ Limits | Requests = Limits |
| **Flexibilité** | 🟢 Maximale | 🟡 Moyenne | 🔴 Minimale |
| **Prévisibilité** | 🔴 Faible | 🟡 Moyenne | 🟢 Haute |
| **Stabilité** | 🔴 Faible | 🟡 Moyenne | 🟢 Haute |
| **Coût ressources** | 🟢 Minimal | 🟡 Moyen | 🔴 Maximal |
| **Usage en prod** | ❌ Jamais | ✅ Recommandé | ✅ Pour critiques |

### Schéma de décision

```
                    Votre application
                          |
                          |
              ┌───────────┴───────────┐
              |                       |
         Critique ?              Non critique ?
              |                       |
              |                       |
         ┌────┴────┐             ┌────┴────┐
         |         |             |         |
    Stable    Variable     Stable    Variable
    charge    charge       charge    charge
         |         |             |         |
         |         |             |         |
    Guaranteed  Burstable   Burstable  Burstable
```

**Règle simple** :
- Application critique + charge stable = **Guaranteed**
- Tout le reste en production = **Burstable**
- Tests uniquement = **BestEffort** (acceptable)

## Vérifier la QoS Class d'un Pod

### Méthode 1 : kubectl get

```bash
kubectl get pod mon-pod -o jsonpath='{.status.qosClass}'
```

Résultat : `Guaranteed`, `Burstable`, ou `BestEffort`

### Méthode 2 : kubectl describe

```bash
kubectl describe pod mon-pod
```

Cherchez la ligne :
```
QoS Class:       Burstable
```

### Méthode 3 : Lister tous les Pods par QoS

```bash
# Pods Guaranteed
kubectl get pods -o jsonpath='{range .items[?(@.status.qosClass=="Guaranteed")]}{.metadata.name}{"\n"}{end}'

# Pods Burstable
kubectl get pods -o jsonpath='{range .items[?(@.status.qosClass=="Burstable")]}{.metadata.name}{"\n"}{end}'

# Pods BestEffort
kubectl get pods -o jsonpath='{range .items[?(@.status.qosClass=="BestEffort")]}{.metadata.name}{"\n"}{end}'
```

### Méthode 4 : Statistiques par namespace

Script pour voir la distribution des QoS dans un namespace :

```bash
echo "QoS Class distribution:"
echo "Guaranteed: $(kubectl get pods -n production -o json | jq '[.items[].status.qosClass] | map(select(. == "Guaranteed")) | length')"
echo "Burstable:  $(kubectl get pods -n production -o json | jq '[.items[].status.qosClass] | map(select(. == "Burstable")) | length')"
echo "BestEffort: $(kubectl get pods -n production -o json | jq '[.items[].status.qosClass] | map(select(. == "BestEffort")) | length')"
```

## Ordre d'éviction détaillé

### En cas de pression mémoire

Voici le processus complet d'éviction :

```
┌─────────────────────────────────────────────────────────┐
│         Processus d'éviction (Memory Pressure)          │
├─────────────────────────────────────────────────────────┤
│                                                         │
│  Étape 1 : BestEffort Pods                              │
│  ─────────────────────────────                          │
│  • Tous les Pods BestEffort                             │
│  • Ordre : celui qui consomme le plus en premier        │
│                                                         │
│  Étape 2 : Burstable Pods                               │
│  ───────────────────────                                │
│  • Tous les Pods Burstable                              │
│  • Ordre : celui qui dépasse le plus ses requests       │
│  • Formule : (usage - requests) / requests              │
│                                                         │
│  Étape 3 : Guaranteed Pods (dernier recours)            │
│  ──────────────────────────────────────────             │
│  • Tous les Pods Guaranteed                             │
│  • Ordre : celui qui consomme le plus                   │
│  • Situation critique !                                 │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

### Exemple concret

**Nœud** : 16 Gi RAM total, 15.8 Gi utilisé (98.75%) → Pression mémoire !

**Pods sur le nœud** :
```yaml
# Pod A - BestEffort
Usage : 2 Gi
Priorité d'éviction : 1 (tué en premier)

# Pod B - BestEffort
Usage : 1 Gi
Priorité d'éviction : 2

# Pod C - Burstable
Requests : 4 Gi, Usage : 6 Gi (dépassement 50%)
Priorité d'éviction : 3

# Pod D - Burstable
Requests : 2 Gi, Usage : 3 Gi (dépassement 50%)
Priorité d'éviction : 4

# Pod E - Burstable
Requests : 1 Gi, Usage : 1.2 Gi (dépassement 20%)
Priorité d'éviction : 5

# Pod F - Guaranteed
Requests = Limits = 2 Gi, Usage : 1.8 Gi
Priorité d'éviction : 6 (tué en dernier)
```

**Ordre réel d'éviction** :
1. **Pod A** (BestEffort, 2 Gi) → Libère 2 Gi → Pression résolue ✅

Si pas suffisant :
2. **Pod B** (BestEffort, 1 Gi)
3. **Pod C ou D** (Burstable, même dépassement 50%)
4. **Pod E** (Burstable, dépassement 20%)
5. **Pod F** (Guaranteed) - dernier recours

## Bonnes pratiques

### 1. Production = Pas de BestEffort

✅ **Règle absolue** : Zéro Pod BestEffort en production.

**Mise en place** : Utilisez un LimitRange pour forcer les resources :

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
      memory: "256Mi"
      cpu: "250m"
    default:
      memory: "512Mi"
      cpu: "500m"
```

Cela garantit que tous les Pods auront au minimum Burstable.

### 2. Identifier les applications critiques

✅ **Processus** :

1. Lister vos applications
2. Identifier celles qui sont critiques :
   - Bases de données
   - Caches
   - APIs core
   - Services de paiement
3. Passer ces applications en Guaranteed
4. Garder le reste en Burstable

### 3. Monitoring des QoS Classes

✅ **Métriques à suivre** :

```yaml
# Exemple de dashboard Grafana
- Nombre de Pods par QoS Class
- Pourcentage de Pods Guaranteed vs Burstable
- Alertes si Pods BestEffort en production
- Historique des évictions par QoS Class
```

### 4. Documented QoS choices

✅ **Documenter dans les annotations** :

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
  annotations:
    qos-strategy: "Burstable"
    qos-rationale: "Variable load, non-critical service"
spec:
  template:
    spec:
      containers:
      - name: app
        resources:
          requests:
            memory: "512Mi"
            cpu: "500m"
          limits:
            memory: "1Gi"
            cpu: "1"
```

### 5. Testing des configurations

✅ **Tester la résilience** :

```bash
# Simuler la pression mémoire avec stress-ng
kubectl run stress-test --image=polinux/stress-ng --restart=Never -- \
  --vm 1 --vm-bytes 90% --timeout 60s

# Observer les évictions
kubectl get events --sort-by='.lastTimestamp' | grep -i evict
```

### 6. Équilibrer le cluster

✅ **Distribution recommandée** dans un cluster de production :

```
Guaranteed:  20-30% des Pods (applications critiques)
Burstable:   70-80% des Pods (applications standard)
BestEffort:  0% des Pods (interdit)
```

### 7. Révision régulière

✅ **Processus trimestriel** :

1. Analyser les métriques d'utilisation des ressources
2. Identifier les Pods qui mériteraient Guaranteed
3. Identifier les Pods Guaranteed sous-utilisés → passer en Burstable
4. Ajuster les requests/limits en fonction de l'usage réel

## Cas d'usage par type d'application

### Applications web frontend

```yaml
# Configuration : Burstable
apiVersion: apps/v1
kind: Deployment
metadata:
  name: frontend
spec:
  replicas: 3
  template:
    spec:
      containers:
      - name: nginx
        image: nginx:latest
        resources:
          requests:
            memory: "128Mi"
            cpu: "100m"
          limits:
            memory: "256Mi"
            cpu: "200m"
        # QoS : Burstable
        # Raison : Charge variable, non critique
```

### API backend

```yaml
# Configuration : Burstable (ou Guaranteed si critique)
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api-backend
spec:
  replicas: 5
  template:
    spec:
      containers:
      - name: api
        image: my-api:v2
        resources:
          requests:
            memory: "512Mi"
            cpu: "500m"
          limits:
            memory: "1Gi"
            cpu: "1"
        # QoS : Burstable
        # Peut burst pendant les pics de charge
```

### Base de données

```yaml
# Configuration : Guaranteed
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: postgres
spec:
  replicas: 1
  template:
    spec:
      containers:
      - name: postgres
        image: postgres:15
        resources:
          limits:
            memory: "4Gi"
            cpu: "2"
          # requests = limits automatiquement
        # QoS : Guaranteed
        # Critique - ne doit JAMAIS être évincé
```

### Cache Redis

```yaml
# Configuration : Guaranteed
apiVersion: apps/v1
kind: Deployment
metadata:
  name: redis
spec:
  replicas: 1
  template:
    spec:
      containers:
      - name: redis
        image: redis:7
        resources:
          limits:
            memory: "2Gi"
            cpu: "1"
        # QoS : Guaranteed
        # Critique - perte de cache = dégradation service
```

### Worker batch

```yaml
# Configuration : Burstable
apiVersion: apps/v1
kind: Deployment
metadata:
  name: worker
spec:
  replicas: 10
  template:
    spec:
      containers:
      - name: worker
        image: my-worker:latest
        resources:
          requests:
            memory: "256Mi"
            cpu: "250m"
          limits:
            memory: "1Gi"
            cpu: "2"
        # QoS : Burstable
        # Peut burst pendant le traitement
        # Peut être évincé sans impact critique
```

### CronJob de backup

```yaml
# Configuration : Burstable
apiVersion: batch/v1
kind: CronJob
metadata:
  name: backup
spec:
  schedule: "0 2 * * *"
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: backup
            image: backup-tool:latest
            resources:
              requests:
                memory: "512Mi"
                cpu: "500m"
              limits:
                memory: "2Gi"
                cpu: "2"
            # QoS : Burstable
            # S'exécute la nuit, peut utiliser plus si disponible
          restartPolicy: OnFailure
```

## Interaction avec d'autres mécanismes

### QoS + PriorityClass

Les QoS Classes et PriorityClasses travaillent ensemble mais sont différentes :

```yaml
# Pod avec PriorityClass ET QoS
apiVersion: v1
kind: Pod
metadata:
  name: high-priority-guaranteed
spec:
  priorityClassName: system-cluster-critical  # PriorityClass
  containers:
  - name: app
    image: critical-app
    resources:
      limits:
        memory: "2Gi"
        cpu: "2"
      # QoS : Guaranteed
```

**Ordre d'éviction combiné** :
1. BestEffort + Low Priority (premier)
2. BestEffort + High Priority
3. Burstable + Low Priority
4. Burstable + High Priority
5. Guaranteed + Low Priority
6. Guaranteed + High Priority (dernier)

### QoS + Node Affinity

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: db-on-dedicated-node
spec:
  # QoS Guaranteed
  containers:
  - name: postgres
    resources:
      limits:
        memory: "8Gi"
        cpu: "4"
  # Sur un nœud dédié
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
        - matchExpressions:
          - key: workload-type
            operator: In
            values:
            - database
```

Combinaison idéale pour applications critiques.

### QoS + Resource Quotas

Les Resource Quotas peuvent indirectement favoriser certaines QoS :

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: guaranteed-only
  namespace: critical-services
spec:
  hard:
    requests.cpu: "20"
    limits.cpu: "20"      # Même valeur
    requests.memory: "40Gi"
    limits.memory: "40Gi"  # Même valeur
  # Force requests = limits → tous les Pods seront Guaranteed
```

## Debugging et observabilité

### Voir les évictions récentes

```bash
# Événements d'éviction
kubectl get events --all-namespaces --field-selector reason=Evicted

# Détails d'une éviction
kubectl describe pod <pod-name> | grep -A 10 "State:"
```

### Identifier les Pods à risque

```bash
# Pods Burstable qui dépassent beaucoup leurs requests
kubectl top pods --all-namespaces --sort-by=memory
```

### Monitoring avec Prometheus

```yaml
# Exemple de métriques utiles
- kube_pod_status_qos_class
- kube_pod_container_resource_requests
- kube_pod_container_resource_limits
- container_memory_working_set_bytes
```

**Alerte exemple** :
```yaml
alert: HighMemoryUsageVsRequests
expr: |
  (container_memory_working_set_bytes /
   kube_pod_container_resource_requests{resource="memory"}) > 1.5
annotations:
  summary: "Pod {{ $labels.pod }} uses 50% more than requests"
```

### Dashboard Grafana recommandé

Panels à créer :
1. **Distribution des QoS Classes** (pie chart)
2. **Pods par QoS et namespace** (bar chart)
3. **Utilisation vs Requests** par QoS (time series)
4. **Historique des évictions** par QoS (counter)
5. **Pods BestEffort en production** (table - devrait être vide)

## Points clés à retenir

🔑 **Les essentiels** :

1. **QoS Classes** = système de priorité automatique de Kubernetes
2. **Trois classes** : BestEffort < Burstable < Guaranteed
3. **Calculé automatiquement** selon requests/limits
4. **Ordre d'éviction** : BestEffort tués en premier, Guaranteed en dernier
5. **Production** : JAMAIS de BestEffort, majorité Burstable, critiques en Guaranteed
6. **Guaranteed** : requests = limits pour tout (CPU et mémoire)
7. **Burstable** : requests ≠ limits (cas général)
8. **BestEffort** : aucune resources définie (tests uniquement)

## Récapitulatif des 4 sections

Nous avons maintenant couvert l'ensemble de la gestion des ressources :

| Section | Concept | Niveau | Rôle |
|---------|---------|--------|------|
| **18.1** | Requests/Limits | Conteneur | Définir les ressources |
| **18.2** | Resource Quotas | Namespace | Limiter le total |
| **18.3** | LimitRanges | Namespace | Contraindre les valeurs |
| **18.4** | QoS Classes | Pod | Priorité d'éviction |

**Ensemble, ces mécanismes assurent** :
- ✅ Stabilité du cluster
- ✅ Protection des applications critiques
- ✅ Utilisation efficace des ressources
- ✅ Prévisibilité et observabilité

## Conclusion

Les QoS Classes sont un mécanisme élégant qui transforme votre configuration de resources (requests/limits) en un système de priorité intelligent. Vous n'avez rien à configurer directement - Kubernetes calcule automatiquement la QoS Class de chaque Pod.

**En pratique** :
- Commencez par définir des requests et limits raisonnables
- La majorité de vos Pods seront Burstable (c'est normal et bien)
- Identifiez vos applications critiques et passez-les en Guaranteed
- Bannissez BestEffort de la production
- Monitorer les évictions pour ajuster votre stratégie

**Prochaine étape** : Dans la section suivante (18.5), nous verrons les **PriorityClasses** qui permettent un contrôle encore plus fin des priorités.

---


⏭️ [PriorityClasses](/18-gestion-des-ressources/05-priorityclasses.md)
