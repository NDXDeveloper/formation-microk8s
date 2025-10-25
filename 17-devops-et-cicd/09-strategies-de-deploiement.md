🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 17.9 Stratégies de déploiement

## Introduction

Vous avez construit un pipeline CI/CD complet, vos applications sont testées automatiquement, et vous êtes prêt à déployer en production. Mais comment minimiser les risques lors d'une mise à jour ? Comment déployer une nouvelle version sans interruption de service ? Comment tester une nouvelle fonctionnalité sur un petit pourcentage d'utilisateurs avant de la généraliser ?

C'est là qu'interviennent les **stratégies de déploiement**. Elles définissent **comment** une nouvelle version d'une application remplace l'ancienne, avec différents niveaux de risque, de complexité et d'interruption de service.

Dans ce chapitre, nous allons explorer les principales stratégies de déploiement, de la plus simple (Recreate) à la plus sophistiquée (Canary avec analyse automatique), en comprenant quand et comment utiliser chacune d'elles sur votre cluster MicroK8s.

---

## Pourquoi les stratégies de déploiement sont importantes

### Le problème du déploiement traditionnel

**Avant Kubernetes**, les déploiements ressemblaient à ceci :

1. Arrêter l'application (downtime)
2. Mettre à jour le code
3. Redémarrer l'application
4. Croiser les doigts 🤞

**Problèmes** :
- ❌ Interruption de service (downtime)
- ❌ Impossible de revenir en arrière facilement
- ❌ Tout ou rien : tous les utilisateurs impactés immédiatement
- ❌ Stress maximal lors des déploiements

### Les objectifs des stratégies modernes

✅ **Zero-downtime** : Pas d'interruption de service

✅ **Rollback rapide** : Retour en arrière en quelques secondes

✅ **Déploiement progressif** : Exposition graduelle de la nouvelle version

✅ **Validation automatique** : Détection automatique des problèmes

✅ **Réduction des risques** : Limiter l'impact d'un bug

✅ **Confiance** : Déployer sereinement plusieurs fois par jour

---

## Vue d'ensemble des stratégies

### Tableau comparatif

| Stratégie | Downtime | Rollback | Complexité | Coût ressources | Use case |
|-----------|----------|----------|------------|-----------------|----------|
| **Recreate** | ⚠️ Oui | Moyen | Très simple | Faible | Dev, apps non-critiques |
| **Rolling Update** | ✅ Non | Rapide | Simple | Faible | Production standard |
| **Blue-Green** | ✅ Non | Instantané | Moyen | Élevé (2x) | Production critique |
| **Canary** | ✅ Non | Rapide | Élevé | Moyen | Fonctionnalités risquées |
| **A/B Testing** | ✅ Non | Rapide | Élevé | Moyen | Test de features |

### Le spectre risque/complexité

```
Risque élevé                                         Risque faible
│                                                                │
Recreate ──────> Rolling Update ──────> Canary ──────> Blue-Green
│                                                                │
Simple                                                    Complexe
```

---

## Stratégie 1 : Recreate (Recréation)

### Principe

La stratégie **Recreate** est la plus simple : tous les pods existants sont **arrêtés** avant de créer les nouveaux pods avec la nouvelle version.

**Workflow** :

```
État initial:     v1  v1  v1  (3 pods version 1)
                   ↓   ↓   ↓
Arrêt:            🛑  🛑  🛑  (tous arrêtés)
                   ↓   ↓   ↓
Démarrage:        v2  v2  v2  (3 pods version 2)
```

### Configuration Kubernetes

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mon-app
spec:
  replicas: 3
  strategy:
    type: Recreate  # Stratégie Recreate
  selector:
    matchLabels:
      app: mon-app
  template:
    metadata:
      labels:
        app: mon-app
    spec:
      containers:
      - name: app
        image: localhost:32000/mon-app:v2.0.0
        ports:
        - containerPort: 8080
```

### Déploiement

```bash
# Mettre à jour l'image
kubectl set image deployment/mon-app app=localhost:32000/mon-app:v2.0.0

# Observer le déploiement
kubectl get pods -w

# Résultat :
# mon-app-v1-xxx   1/1   Running       0   5m
# mon-app-v1-yyy   1/1   Running       0   5m
# mon-app-v1-zzz   1/1   Running       0   5m
# mon-app-v1-xxx   1/1   Terminating   0   5m
# mon-app-v1-yyy   1/1   Terminating   0   5m
# mon-app-v1-zzz   1/1   Terminating   0   5m
# ...quelques secondes plus tard...
# mon-app-v2-aaa   0/1   Pending       0   0s
# mon-app-v2-bbb   0/1   Pending       0   0s
# mon-app-v2-ccc   0/1   Pending       0   0s
# mon-app-v2-aaa   1/1   Running       0   10s
# mon-app-v2-bbb   1/1   Running       0   10s
# mon-app-v2-ccc   1/1   Running       0   10s
```

### Avantages et inconvénients

**✅ Avantages** :
- Très simple à implémenter
- Pas de versions mixtes (v1 et v2 simultanément)
- Consommation de ressources minimale
- Évite les problèmes de compatibilité entre versions

**❌ Inconvénients** :
- **Downtime** : service indisponible pendant plusieurs secondes/minutes
- Pas adapté à la production
- Tous les utilisateurs impactés simultanément
- Rollback lent (nécessite de recréer les pods)

**Cas d'usage** :
- Environnements de développement
- Applications non-critiques
- Maintenance planifiée avec fenêtre de downtime
- Migrations de base de données incompatibles

---

## Stratégie 2 : Rolling Update (Mise à jour progressive)

### Principe

La stratégie **Rolling Update** (par défaut dans Kubernetes) remplace progressivement les pods de l'ancienne version par la nouvelle, en maintenant toujours un certain nombre de pods disponibles.

**Workflow** :

```
État initial:     v1  v1  v1  v1  v1  (5 pods)
                   ↓
Étape 1:          v1  v1  v1  v1  v2  (1 nouveau pod v2 créé)
                   ↓
Étape 2:          v1  v1  v1  v2  v2  (1 ancien pod v1 supprimé)
                   ↓
Étape 3:          v1  v1  v2  v2  v2  (continue...)
                   ↓
Étape 4:          v1  v2  v2  v2  v2
                   ↓
État final:       v2  v2  v2  v2  v2  (tous en v2)
```

### Configuration Kubernetes

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mon-app
spec:
  replicas: 5
  strategy:
    type: RollingUpdate  # Par défaut
    rollingUpdate:
      maxSurge: 1        # Nombre max de pods supplémentaires
      maxUnavailable: 1  # Nombre max de pods indisponibles
  selector:
    matchLabels:
      app: mon-app
  template:
    metadata:
      labels:
        app: mon-app
        version: v2.0.0
    spec:
      containers:
      - name: app
        image: localhost:32000/mon-app:v2.0.0
        ports:
        - containerPort: 8080
        readinessProbe:
          httpGet:
            path: /health
            port: 8080
          initialDelaySeconds: 5
          periodSeconds: 5
```

### Paramètres importants

#### maxSurge

Nombre **maximum de pods** qui peuvent être créés au-dessus du nombre désiré pendant la mise à jour.

- **Valeur** : nombre absolu (ex: `2`) ou pourcentage (ex: `25%`)
- **Exemple** : Si `replicas: 5` et `maxSurge: 2`, il peut y avoir jusqu'à **7 pods** temporairement

```yaml
maxSurge: 1       # Déploiement conservateur (1 pod à la fois)
maxSurge: 2       # Équilibré
maxSurge: 50%     # Rapide (la moitié des pods en plus)
maxSurge: 100%    # Très rapide (double la capacité temporairement)
```

#### maxUnavailable

Nombre **maximum de pods** qui peuvent être indisponibles pendant la mise à jour.

- **Valeur** : nombre absolu (ex: `1`) ou pourcentage (ex: `25%`)
- **Exemple** : Si `replicas: 5` et `maxUnavailable: 1`, il y aura toujours au minimum **4 pods disponibles**

```yaml
maxUnavailable: 0    # Aucun pod indisponible (très sûr, mais lent)
maxUnavailable: 1    # Standard
maxUnavailable: 25%  # Un quart peut être indisponible
```

**⚠️ Important** : `maxSurge` et `maxUnavailable` ne peuvent pas être tous les deux à 0.

### Scénarios de configuration

#### Configuration conservatrice (production critique)

```yaml
strategy:
  type: RollingUpdate
  rollingUpdate:
    maxSurge: 1
    maxUnavailable: 0
```

**Comportement** :
- Toujours 5 pods disponibles
- 1 nouveau pod créé, attend qu'il soit ready
- 1 ancien pod supprimé
- Répète jusqu'à complétion
- **Lent mais très sûr**

#### Configuration équilibrée (standard)

```yaml
strategy:
  type: RollingUpdate
  rollingUpdate:
    maxSurge: 1
    maxUnavailable: 1
```

**Comportement** :
- 3-6 pods disponibles en permanence
- Équilibre entre vitesse et sécurité
- **Recommandé pour la plupart des cas**

#### Configuration rapide (non-critique)

```yaml
strategy:
  type: RollingUpdate
  rollingUpdate:
    maxSurge: 100%
    maxUnavailable: 50%
```

**Comportement** :
- Peut aller jusqu'à 10 pods temporairement
- Au moins 3 pods disponibles
- **Très rapide mais consomme plus de ressources**

### Déploiement et monitoring

```bash
# Mettre à jour l'image
kubectl set image deployment/mon-app app=localhost:32000/mon-app:v2.0.0

# Surveiller le rollout
kubectl rollout status deployment/mon-app

# Observer en temps réel
kubectl get pods -w -l app=mon-app

# Voir l'historique
kubectl rollout history deployment/mon-app
```

### Rollback

```bash
# Annuler le déploiement en cours
kubectl rollout undo deployment/mon-app

# Revenir à une révision spécifique
kubectl rollout history deployment/mon-app
kubectl rollout undo deployment/mon-app --to-revision=3

# Pause/Resume
kubectl rollout pause deployment/mon-app
# ...vérifications...
kubectl rollout resume deployment/mon-app
```

### Avantages et inconvénients

**✅ Avantages** :
- Zero-downtime
- Natif dans Kubernetes (pas d'outil externe)
- Rollback rapide et facile
- Consommation de ressources contrôlée
- Adapté à la plupart des cas d'usage

**❌ Inconvénients** :
- Versions mixtes temporaires (v1 et v2 coexistent)
- Difficile de contrôler quel utilisateur voit quelle version
- Si v2 a un bug, une partie des utilisateurs est impactée avant détection
- Pas de validation automatique avant complétion

**Cas d'usage** :
- Déploiements en production standard
- Applications stateless
- Mises à jour compatibles backward

---

## Stratégie 3 : Blue-Green Deployment

### Principe

Le déploiement **Blue-Green** maintient deux environnements de production identiques :
- **Blue** : version actuelle en production
- **Green** : nouvelle version en préparation

Une fois la version Green validée, on bascule le trafic instantanément de Blue vers Green.

**Workflow** :

```
État initial:
  Blue (v1)  ←─────────── Trafic (100%)
  Green      ←─────────── Idle

Déploiement Green:
  Blue (v1)  ←─────────── Trafic (100%)
  Green (v2) ←─────────── Tests

Validation OK, bascule:
  Blue (v1)  ←─────────── Idle
  Green (v2) ←─────────── Trafic (100%)

Cleanup (optionnel):
  Green (v2) ←─────────── Trafic (100%)
  (Blue supprimé)
```

### Implémentation avec Kubernetes

#### Méthode 1 : Via les labels de Service

**Déploiements Blue et Green** :

```yaml
# deployment-blue.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mon-app-blue
spec:
  replicas: 3
  selector:
    matchLabels:
      app: mon-app
      version: blue
  template:
    metadata:
      labels:
        app: mon-app
        version: blue
    spec:
      containers:
      - name: app
        image: localhost:32000/mon-app:v1.0.0
        ports:
        - containerPort: 8080
---
# deployment-green.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mon-app-green
spec:
  replicas: 3
  selector:
    matchLabels:
      app: mon-app
      version: green
  template:
    metadata:
      labels:
        app: mon-app
        version: green
    spec:
      containers:
      - name: app
        image: localhost:32000/mon-app:v2.0.0
        ports:
        - containerPort: 8080
```

**Service pointant initialement vers Blue** :

```yaml
apiVersion: v1
kind: Service
metadata:
  name: mon-app
spec:
  selector:
    app: mon-app
    version: blue  # Pointe vers Blue
  ports:
  - port: 80
    targetPort: 8080
```

**Bascule vers Green** :

```bash
# Déployer Green
kubectl apply -f deployment-green.yaml

# Attendre que Green soit prêt
kubectl wait --for=condition=ready pod -l version=green --timeout=300s

# Tests sur Green (via port-forward ou service temporaire)
kubectl port-forward deployment/mon-app-green 8081:8080
curl http://localhost:8081/health

# Bascule instantanée du trafic
kubectl patch service mon-app -p '{"spec":{"selector":{"version":"green"}}}'

# Vérifier
kubectl describe service mon-app

# Après validation, supprimer Blue
kubectl delete deployment mon-app-blue
```

#### Méthode 2 : Via deux Services et Ingress

**Services dédiés** :

```yaml
---
apiVersion: v1
kind: Service
metadata:
  name: mon-app-blue
spec:
  selector:
    app: mon-app
    version: blue
  ports:
  - port: 80
    targetPort: 8080
---
apiVersion: v1
kind: Service
metadata:
  name: mon-app-green
spec:
  selector:
    app: mon-app
    version: green
  ports:
  - port: 80
    targetPort: 8080
```

**Ingress avec bascule** :

```yaml
# ingress-blue.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: mon-app
spec:
  ingressClassName: nginx
  rules:
  - host: mon-app.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: mon-app-blue  # Trafic vers Blue
            port:
              number: 80
```

**Bascule** :

```bash
# Modifier l'Ingress pour pointer vers Green
kubectl patch ingress mon-app --type='json' \
  -p='[{"op": "replace", "path": "/spec/rules/0/http/paths/0/backend/service/name", "value":"mon-app-green"}]'
```

### Script d'automatisation

```bash
#!/bin/bash
# blue-green-deploy.sh

set -e

APP_NAME="mon-app"
NEW_VERSION=$1
CURRENT_COLOR=$2  # blue ou green
NEW_COLOR=$3      # green ou blue

echo "🚀 Déploiement Blue-Green"
echo "Version actuelle: ${CURRENT_COLOR}"
echo "Nouvelle version: ${NEW_VERSION} sur ${NEW_COLOR}"

# 1. Déployer la nouvelle version
echo "📦 Déploiement de ${NEW_COLOR}..."
kubectl set image deployment/${APP_NAME}-${NEW_COLOR} \
  app=localhost:32000/${APP_NAME}:${NEW_VERSION}

# 2. Attendre que ce soit prêt
echo "⏳ Attente que ${NEW_COLOR} soit prêt..."
kubectl rollout status deployment/${APP_NAME}-${NEW_COLOR}
kubectl wait --for=condition=ready pod -l version=${NEW_COLOR} --timeout=300s

# 3. Tests de smoke
echo "🔥 Tests de smoke sur ${NEW_COLOR}..."
POD=$(kubectl get pod -l version=${NEW_COLOR} -o jsonpath='{.items[0].metadata.name}')
kubectl exec ${POD} -- wget -qO- http://localhost:8080/health || {
  echo "❌ Tests échoués!"
  exit 1
}

# 4. Bascule du trafic
echo "🔀 Bascule du trafic vers ${NEW_COLOR}..."
kubectl patch service ${APP_NAME} -p "{\"spec\":{\"selector\":{\"version\":\"${NEW_COLOR}\"}}}"

# 5. Vérification post-bascule
echo "✅ Vérification..."
sleep 10
curl -f http://mon-app.example.com/health || {
  echo "❌ Échec post-bascule! Rollback..."
  kubectl patch service ${APP_NAME} -p "{\"spec\":{\"selector\":{\"version\":\"${CURRENT_COLOR}\"}}}"
  exit 1
}

echo "🎉 Déploiement Blue-Green réussi!"
echo "💡 Pour nettoyer l'ancienne version: kubectl scale deployment/${APP_NAME}-${CURRENT_COLOR} --replicas=0"
```

**Utilisation** :

```bash
./blue-green-deploy.sh v2.0.0 blue green
```

### Avantages et inconvénients

**✅ Avantages** :
- Rollback **instantané** (basculer vers l'ancienne version)
- Tests complets sur Green avant bascule
- Pas de versions mixtes en production
- Downtime minimal (quelques secondes max pour la bascule)

**❌ Inconvénients** :
- **Coût** : double des ressources (2x pods)
- Complexité de mise en œuvre
- Nécessite synchronisation des données si stateful
- Gestion de deux environnements simultanés

**Cas d'usage** :
- Applications critiques nécessitant rollback instantané
- Déploiements à fort enjeu
- Validation complète nécessaire avant production
- Budget permettant le doublement des ressources

---

## Stratégie 4 : Canary Deployment

### Principe

Le déploiement **Canary** expose progressivement la nouvelle version à un pourcentage croissant d'utilisateurs, permettant de valider en conditions réelles avant généralisation.

**Origine** : Le nom vient des canaris dans les mines de charbon, utilisés pour détecter les gaz toxiques avant qu'ils n'affectent les mineurs.

**Workflow** :

```
Phase 1:  v1 (95%)  v2 (5%)   ← 5% du trafic sur v2
          ↓
Phase 2:  v1 (80%)  v2 (20%)  ← Si OK, 20%
          ↓
Phase 3:  v1 (50%)  v2 (50%)  ← Si OK, 50%
          ↓
Phase 4:  v1 (0%)   v2 (100%) ← Si OK, 100%
```

### Implémentation manuelle avec Kubernetes

**Déploiements stable et canary** :

```yaml
---
# deployment-stable.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mon-app-stable
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
    spec:
      containers:
      - name: app
        image: localhost:32000/mon-app:v1.0.0
        ports:
        - containerPort: 8080
---
# deployment-canary.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mon-app-canary
spec:
  replicas: 1  # 10% du trafic
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
    spec:
      containers:
      - name: app
        image: localhost:32000/mon-app:v2.0.0
        ports:
        - containerPort: 8080
---
# service.yaml
apiVersion: v1
kind: Service
metadata:
  name: mon-app
spec:
  selector:
    app: mon-app  # Sélectionne stable ET canary
  ports:
  - port: 80
    targetPort: 8080
```

**Progression manuelle** :

```bash
# Phase 1 : 10% canary
kubectl apply -f deployment-stable.yaml  # 9 replicas
kubectl apply -f deployment-canary.yaml  # 1 replica

# Attendre et surveiller les métriques
sleep 300

# Phase 2 : 20% canary
kubectl scale deployment/mon-app-stable --replicas=8
kubectl scale deployment/mon-app-canary --replicas=2

# Phase 3 : 50% canary
kubectl scale deployment/mon-app-stable --replicas=5
kubectl scale deployment/mon-app-canary --replicas=5

# Phase 4 : 100% canary
kubectl scale deployment/mon-app-stable --replicas=0
kubectl scale deployment/mon-app-canary --replicas=10
```

### Implémentation avancée avec Flagger

**Flagger** automatise les déploiements Canary avec analyse de métriques.

**Installation** :

```bash
# Ajouter le repo Helm
helm repo add flagger https://flagger.app

# Installer Flagger
helm upgrade -i flagger flagger/flagger \
  --namespace flagger-system \
  --create-namespace \
  --set meshProvider=nginx \
  --set metricsServer=http://prometheus:9090
```

**Définition d'un Canary** :

```yaml
apiVersion: flagger.app/v1beta1
kind: Canary
metadata:
  name: mon-app
  namespace: production
spec:
  # Deployment à gérer
  targetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: mon-app

  # Service
  service:
    port: 80
    targetPort: 8080

  # Analyse des métriques
  analysis:
    # Intervalle entre chaque étape
    interval: 1m
    # Seuil avant rollback automatique
    threshold: 5
    # Nombre d'itérations
    iterations: 10

    # Métriques Prometheus à surveiller
    metrics:
    - name: request-success-rate
      thresholdRange:
        min: 99
      interval: 1m
    - name: request-duration
      thresholdRange:
        max: 500
      interval: 1m

    # Webhooks pour tests (optionnel)
    webhooks:
    - name: smoke-test
      url: http://flagger-loadtester/
      timeout: 5s
      metadata:
        type: cmd
        cmd: "curl -sd 'test' http://mon-app-canary/health"
    - name: load-test
      url: http://flagger-loadtester/
      timeout: 5s
      metadata:
        cmd: "hey -z 1m -q 10 -c 2 http://mon-app-canary/"

  # Progression du Canary
  canaryAnalysis:
    stepWeight: 10  # Augmente de 10% à chaque étape
    maxWeight: 50   # Maximum 50%
```

**Déployer une nouvelle version** :

```bash
# Simplement mettre à jour le Deployment
kubectl set image deployment/mon-app app=localhost:32000/mon-app:v2.0.0

# Flagger détecte le changement et commence automatiquement :
# 1. Crée un déploiement canary
# 2. Envoie 10% du trafic
# 3. Analyse les métriques pendant 1 minute
# 4. Si OK, augmente à 20%
# 5. Répète jusqu'à 50%
# 6. Promeut la version si tout va bien
# 7. OU fait un rollback automatique si problème
```

**Surveiller** :

```bash
# Statut du Canary
kubectl get canary -w

# Événements détaillés
kubectl describe canary mon-app

# Exemple de sortie :
# Events:
#   Type     Reason  Age   From     Message
#   ----     ------  ----  ----     -------
#   Normal   Synced  3m    flagger  New revision detected! Scaling up mon-app.production
#   Normal   Synced  2m    flagger  Starting canary analysis for mon-app.production
#   Normal   Synced  1m    flagger  Advance mon-app.production canary weight 10
#   Warning  Synced  30s   flagger  Halt advancement no values found for metric request-success-rate
```

### Avantages et inconvénients

**✅ Avantages** :
- **Risque minimal** : seul un petit % d'utilisateurs est impacté
- Validation en conditions réelles
- Rollback automatique possible (avec Flagger)
- Permet de tester progressivement

**❌ Inconvénients** :
- Complexité de mise en œuvre
- Nécessite du monitoring et des métriques
- Déploiement plus long (progressif)
- Versions mixtes en production

**Cas d'usage** :
- Fonctionnalités à risque
- Refonte majeure d'une application
- Changement d'algorithme sensible
- Nouvelle version avec incertitude

---

## Stratégie 5 : A/B Testing

### Principe

L'**A/B Testing** est similaire au Canary, mais avec un objectif différent : tester **deux variantes** d'une fonctionnalité pour mesurer laquelle performe le mieux.

**Différence avec Canary** :
- **Canary** : Valider qu'une nouvelle version fonctionne (pas de bug)
- **A/B Test** : Comparer deux versions pour choisir la meilleure (business metrics)

**Exemple** : Tester deux designs de bouton d'achat pour voir lequel convertit le mieux.

### Implémentation avec Ingress NGINX

**Déploiements A et B** :

```yaml
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mon-app-variant-a
spec:
  replicas: 3
  selector:
    matchLabels:
      app: mon-app
      variant: a
  template:
    metadata:
      labels:
        app: mon-app
        variant: a
    spec:
      containers:
      - name: app
        image: localhost:32000/mon-app:variant-a
        env:
        - name: VARIANT
          value: "A"
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mon-app-variant-b
spec:
  replicas: 3
  selector:
    matchLabels:
      app: mon-app
      variant: b
  template:
    metadata:
      labels:
        app: mon-app
        variant: b
    spec:
      containers:
      - name: app
        image: localhost:32000/mon-app:variant-b
        env:
        - name: VARIANT
          value: "B"
```

**Services** :

```yaml
---
apiVersion: v1
kind: Service
metadata:
  name: mon-app-a
spec:
  selector:
    app: mon-app
    variant: a
  ports:
  - port: 80
    targetPort: 8080
---
apiVersion: v1
kind: Service
metadata:
  name: mon-app-b
spec:
  selector:
    app: mon-app
    variant: b
  ports:
  - port: 80
    targetPort: 8080
```

**Ingress avec split de trafic** :

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: mon-app-ab
  annotations:
    nginx.ingress.kubernetes.io/canary: "true"
    nginx.ingress.kubernetes.io/canary-weight: "50"
spec:
  ingressClassName: nginx
  rules:
  - host: mon-app.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: mon-app-a
            port:
              number: 80
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: mon-app-ab-canary
  annotations:
    nginx.ingress.kubernetes.io/canary: "true"
    nginx.ingress.kubernetes.io/canary-weight: "50"
spec:
  ingressClassName: nginx
  rules:
  - host: mon-app.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: mon-app-b
            port:
              number: 80
```

### Routage basé sur les headers

```yaml
# Routage selon un header (ex: user type)
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: mon-app-ab-header
  annotations:
    nginx.ingress.kubernetes.io/canary: "true"
    nginx.ingress.kubernetes.io/canary-by-header: "X-User-Type"
    nginx.ingress.kubernetes.io/canary-by-header-value: "premium"
spec:
  ingressClassName: nginx
  rules:
  - host: mon-app.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: mon-app-b  # Users premium voient variant B
            port:
              number: 80
```

**Test** :

```bash
# Utilisateur normal → variant A
curl http://mon-app.example.com/

# Utilisateur premium → variant B
curl -H "X-User-Type: premium" http://mon-app.example.com/
```

---

## Stratégie 6 : Shadow Deployment (Traffic Mirroring)

### Principe

Le **Shadow Deployment** (ou traffic mirroring) envoie le trafic de production à **deux versions simultanément** :
- Version **stable** : reçoit et répond aux requêtes (production)
- Version **shadow** : reçoit une copie des requêtes mais ses réponses sont ignorées

**Intérêt** : Tester la nouvelle version avec du trafic réel sans risque pour les utilisateurs.

```
                    ┌─────────────┐
                    │   Requête   │
                    └──────┬──────┘
                           │
                ┌──────────┴──────────┐
                │                     │
         ┌──────▼──────┐      ┌───────▼─────┐
         │  Version    │      │  Version    │
         │  Stable     │      │  Shadow     │
         │  (v1)       │      │  (v2)       │
         └──────┬──────┘      └───────┬─────┘
                │                     │
         Réponse retournée      Réponse ignorée
                │
         ┌──────▼──────┐
         │   Client    │
         └─────────────┘
```

### Implémentation avec Istio

Istio facilite le traffic mirroring.

**Installation d'Istio sur MicroK8s** :

```bash
# Télécharger Istio
curl -L https://istio.io/downloadIstio | sh -
cd istio-*
export PATH=$PWD/bin:$PATH

# Installer
istioctl install --set profile=demo -y

# Activer l'injection automatique
kubectl label namespace default istio-injection=enabled
```

**Déploiements** :

```yaml
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mon-app-v1
spec:
  replicas: 3
  selector:
    matchLabels:
      app: mon-app
      version: v1
  template:
    metadata:
      labels:
        app: mon-app
        version: v1
    spec:
      containers:
      - name: app
        image: localhost:32000/mon-app:v1.0.0
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mon-app-v2
spec:
  replicas: 3
  selector:
    matchLabels:
      app: mon-app
      version: v2
  template:
    metadata:
      labels:
        app: mon-app
        version: v2
    spec:
      containers:
      - name: app
        image: localhost:32000/mon-app:v2.0.0
```

**VirtualService avec mirroring** :

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
        subset: v1
      weight: 100
    mirror:
      host: mon-app
      subset: v2
    mirrorPercentage:
      value: 100.0  # Miroir 100% du trafic
---
apiVersion: networking.istio.io/v1beta1
kind: DestinationRule
metadata:
  name: mon-app
spec:
  host: mon-app
  subsets:
  - name: v1
    labels:
      version: v1
  - name: v2
    labels:
      version: v2
```

**Résultat** :
- Tous les utilisateurs reçoivent des réponses de v1
- v2 reçoit une copie de toutes les requêtes
- Vous pouvez analyser les logs/métriques de v2 pour détecter des problèmes

### Avantages et inconvénients

**✅ Avantages** :
- **Zero-risque** : la nouvelle version n'impacte pas les utilisateurs
- Test avec trafic réel de production
- Permet de comparer les performances
- Détection de bugs avant mise en production

**❌ Inconvénients** :
- Nécessite un service mesh (Istio, Linkerd)
- Double les ressources de compute
- Pas de test des effets de bord (écriture DB par exemple)
- Complexité de mise en œuvre

**Cas d'usage** :
- Refonte majeure d'un algorithme
- Migration de stack technique
- Validation de performances

---

## Comparaison et choix de stratégie

### Arbre de décision

```
┌─────────────────────────────────┐
│ Downtime acceptable ?           │
└────────┬────────────────────────┘
         │
    ┌────▼────┐
    │  OUI    │───────► RECREATE
    └─────────┘
    ┌────▼────┐
    │  NON    │
    └────┬────┘
         │
┌────────▼────────────────────────┐
│ Budget pour doubler ressources? │
└─┬───────┬───────────────────────┘
  │       │
  │  ┌────▼────┐
  │  │  OUI    │
  │  └────┬────┘
  │       │
  │  ┌────▼───────────────────────────┐
  │  │ Besoin de rollback instantané? │
  │  └─┬────┬─────────────────────────┘
  │    │    │
  │    │  ┌─▼───────┐                ┌────────────┐
  │    │  │  OUI    │───────────────►│ BLUE-GREEN │
  │    │  └─────────┘                └────────────┘
  │  ┌─▼───────┐
  │  │  NON    │───────────────► CANARY
  │  └─────────┘
  │
┌─▼───────┐
│  NON    │───────────────► ROLLING UPDATE
└─────────┘
```

### Tableau récapitulatif détaillé

| Critère | Recreate | Rolling | Blue-Green | Canary | A/B Test |
|---------|----------|---------|------------|--------|----------|
| **Downtime** | ⚠️ Oui | ✅ Non | ✅ Non | ✅ Non | ✅ Non |
| **Rollback** | Lent | Rapide | Instantané | Automatique | Rapide |
| **Ressources** | 1x | 1.2x | 2x | 1.3x | 1.5x |
| **Complexité** | ⭐ | ⭐⭐ | ⭐⭐⭐ | ⭐⭐⭐⭐ | ⭐⭐⭐⭐ |
| **Durée** | Rapide | Moyenne | Rapide | Lente | Variable |
| **Testing** | Aucun | Basique | Complet | Progressif | Métrique |
| **Outils nécessaires** | Aucun | Natif K8s | Natif K8s | Flagger | Istio/Nginx |

---

## Bonnes pratiques

### 1. Toujours définir des readinessProbe

```yaml
readinessProbe:
  httpGet:
    path: /health
    port: 8080
  initialDelaySeconds: 5
  periodSeconds: 5
  failureThreshold: 3
```

**Pourquoi** : Kubernetes n'envoie du trafic qu'aux pods "ready", évitant les erreurs pendant le déploiement.

### 2. Utiliser PodDisruptionBudget

```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: mon-app-pdb
spec:
  minAvailable: 2
  selector:
    matchLabels:
      app: mon-app
```

**Pourquoi** : Garantit qu'au moins 2 pods sont toujours disponibles, même pendant les mises à jour.

### 3. Tester en staging d'abord

```bash
# Toujours déployer en staging avant production
helm upgrade mon-app ./chart -f values-staging.yaml -n staging
# Tests...
helm upgrade mon-app ./chart -f values-production.yaml -n production
```

### 4. Automatiser les smoke tests

```bash
#!/bin/bash
# smoke-test.sh

# Attendre que les pods soient prêts
kubectl rollout status deployment/mon-app

# Tests de base
curl -f http://mon-app.example.com/health || exit 1
curl -f http://mon-app.example.com/api/version || exit 1

# Test de charge léger
ab -n 100 -c 10 http://mon-app.example.com/ || exit 1

echo "✅ Smoke tests réussis"
```

### 5. Monitorer activement pendant le déploiement

```yaml
# Créer des alertes Prometheus pour déploiements
- alert: DeploymentHighErrorRate
  expr: rate(http_requests_total{status=~"5.."}[5m]) > 0.05
  for: 2m
  annotations:
    summary: "Taux d'erreur élevé pendant le déploiement"
```

### 6. Documenter la stratégie

```yaml
# Dans le Chart.yaml ou README
annotations:
  deployment-strategy: blue-green
  rollback-procedure: "kubectl patch service mon-app -p '{\"spec\":{\"selector\":{\"version\":\"blue\"}}}'"
```

### 7. Versionner sémantiquement

```bash
# Tags Git clairs
git tag -a v2.1.0 -m "Feature: nouveau dashboard"
git push origin v2.1.0

# Images Docker avec version
docker build -t mon-app:v2.1.0 .
docker tag mon-app:v2.1.0 mon-app:2.1
docker tag mon-app:v2.1.0 mon-app:2
```

### 8. Planifier les rollbacks

```bash
# Avoir un runbook de rollback
# 1. Identifier la version stable
kubectl rollout history deployment/mon-app

# 2. Rollback
kubectl rollout undo deployment/mon-app --to-revision=3

# 3. Vérifier
kubectl rollout status deployment/mon-app

# 4. Alerter l'équipe
```

---

## Récapitulatif

### Points clés

**Les stratégies de déploiement, c'est** :
- Minimiser les risques lors des mises à jour
- Offrir différents niveaux de sécurité et rapidité
- Permettre le rollback rapide
- Adapter la stratégie au contexte

**Stratégies principales** :

1. **Recreate** : Simple, downtime, dev/test
2. **Rolling Update** : Standard, zero-downtime, production
3. **Blue-Green** : Rollback instantané, coûteux, critique
4. **Canary** : Progressif, validation réelle, risqué
5. **A/B Testing** : Comparaison métrique, business
6. **Shadow** : Zero-risque, test réel, complexe

**Choix de stratégie** :
- Production standard → **Rolling Update**
- Application critique → **Blue-Green**
- Fonctionnalité risquée → **Canary**
- Test de feature → **A/B Testing**
- Dev/Staging → **Recreate** ou **Rolling**

**Outils** :
- Kubernetes natif : Recreate, Rolling, Blue-Green
- Flagger : Canary automatisé
- Istio : A/B Testing, Shadow
- Argo Rollouts : Alternative à Flagger

### Bénéfices

Avec des stratégies de déploiement adaptées :
- ✅ Déploiements sans stress
- ✅ Zero ou minimal downtime
- ✅ Rollback rapide en cas de problème
- ✅ Validation progressive
- ✅ Réduction des risques
- ✅ Confiance pour déployer fréquemment

---

## Conclusion

Les stratégies de déploiement sont la **dernière pièce du puzzle DevOps**. Elles transforment les déploiements d'événements stressants en opérations routinières et sécurisées.

**Votre progression** :

Vous êtes parti de déploiements manuels avec `kubectl apply`, et vous maîtrisez maintenant :

1. **Git** : Versioning du code et des configs
2. **Registry** : Stockage des images
3. **CI/CD** : Automatisation des builds
4. **ArgoCD** : Synchronisation GitOps
5. **Helm** : Packaging des applications
6. **Tests** : Validation automatique
7. **Stratégies** : Déploiement sécurisé

**Le workflow complet** :

```
Code → Git → CI/CD → Registry → Helm Chart → ArgoCD
                                                  ↓
                                        Stratégie de déploiement
                                                  ↓
                                            Kubernetes
                                                  ↓
                                            Monitoring
```

Avec ce workflow, vous pouvez :
- Déployer plusieurs fois par jour
- Rollback en quelques secondes
- Tester de nouvelles features en toute sécurité
- Maintenir une qualité élevée
- Dormir tranquille

**Félicitations !** 🎉

Vous avez complété le chapitre DevOps et CI/CD. Vous disposez maintenant d'une infrastructure professionnelle complète sur MicroK8s, avec toutes les pratiques et outils utilisés par les équipes les plus avancées.

La suite de votre parcours vous attend dans les chapitres suivants sur la gestion des ressources, le scaling, la sécurité et bien plus encore !

**Continuez l'aventure Kubernetes ! 🚀**

⏭️ [Blue-Green deployments](/17-devops-et-cicd/10-blue-green-deployments.md)
