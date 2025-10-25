🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 19.2 Horizontal Pod Autoscaler (HPA)

## Introduction au HPA

Le **Horizontal Pod Autoscaler (HPA)** est un contrôleur Kubernetes qui ajuste automatiquement le nombre de répliques de vos pods en fonction de la charge observée. Contrairement au scaling manuel où vous devez intervenir, le HPA surveille vos applications et scale automatiquement.

### Analogie simple

Imaginez un restaurant :
- **Scaling manuel** : Le gérant décide à l'avance combien de serveurs seront présents (5 le midi, 2 le soir)
- **HPA (autoscaling)** : Le gérant observe en temps réel le nombre de clients et appelle des serveurs supplémentaires quand c'est nécessaire, puis les renvoie quand ça se calme

Le HPA fait exactement la même chose avec vos pods !

### Pourquoi utiliser le HPA ?

**Avantages principaux :**

1. **Réactivité 24/7** : Le HPA surveille votre application en permanence, même la nuit
2. **Adaptation automatique** : Plus besoin d'intervention humaine pour ajuster les ressources
3. **Optimisation des coûts** : Scale down automatiquement quand la charge diminue
4. **Amélioration des performances** : Scale up rapidement lors des pics de trafic
5. **Tranquillité d'esprit** : Vous pouvez dormir tranquille, le HPA gère les imprévus

## Concept "Horizontal" vs "Vertical"

Avant d'aller plus loin, comprenons bien la différence :

### Horizontal Scaling (HPA)
- **Ajoute ou retire des pods** (plus de copies de l'application)
- Exemple : 1 serveur web → 5 serveurs web
- C'est le sujet de cette section

### Vertical Scaling (VPA)
- **Augmente ou diminue les ressources** d'un pod existant (plus de CPU/RAM)
- Exemple : Un pod avec 1 CPU → le même pod avec 4 CPU
- Nous verrons cela dans la section 19.3

**Métaphore :**
- **Horizontal** : Embaucher plus de serveurs au restaurant
- **Vertical** : Rendre chaque serveur plus rapide et efficace

## Comment fonctionne le HPA ?

Le HPA suit un cycle de décision simple :

```
1. Observer les métriques (CPU, mémoire, requêtes/sec, etc.)
   ↓
2. Comparer avec les seuils définis
   ↓
3. Calculer le nombre de répliques idéal
   ↓
4. Scaler le Deployment (si nécessaire)
   ↓
5. Attendre un peu (cooldown period)
   ↓
6. Retour à l'étape 1
```

### Cycle de vérification

Par défaut, le HPA :
- Vérifie les métriques **toutes les 15 secondes**
- Prend une décision de scaling **toutes les 3 minutes** pour scale up
- Attend **5 minutes** avant de scale down (pour éviter les fluctuations)

Ces délais peuvent être ajustés selon vos besoins.

## Prérequis : Metrics Server

Le HPA a besoin de métriques pour fonctionner. Sur MicroK8s, il faut activer le **Metrics Server** :

```bash
microk8s enable metrics-server
```

Le Metrics Server collecte les métriques de consommation CPU et mémoire de tous les pods du cluster.

### Vérifier que Metrics Server fonctionne

Après quelques minutes, testez :

```bash
kubectl top nodes
```

Sortie attendue :
```
NAME          CPU(cores)   CPU%   MEMORY(bytes)   MEMORY%
microk8s-vm   450m         22%    2048Mi          51%
```

Testez aussi pour les pods :

```bash
kubectl top pods
```

Si vous obtenez des résultats, c'est que Metrics Server fonctionne correctement !

## Configuration de base du HPA

### Méthode 1 : Commande impérative (rapide)

La façon la plus simple de créer un HPA :

```bash
kubectl autoscale deployment nginx-web --cpu-percent=50 --min=1 --max=10
```

Cette commande signifie :
- **nginx-web** : Le deployment à scaler
- **--cpu-percent=50** : Maintenir l'utilisation CPU autour de 50%
- **--min=1** : Minimum 1 réplique (jamais en dessous)
- **--max=10** : Maximum 10 répliques (jamais au-dessus)

### Méthode 2 : Fichier YAML (recommandé)

Pour une configuration plus pérenne et versionnable, créez un fichier `hpa-nginx.yaml` :

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: nginx-web-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: nginx-web
  minReplicas: 1
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 50
```

Appliquez la configuration :

```bash
kubectl apply -f hpa-nginx.yaml
```

### Décomposition du fichier YAML

Analysons chaque partie :

**scaleTargetRef** : Quel Deployment scaler ?
```yaml
scaleTargetRef:
  apiVersion: apps/v1
  kind: Deployment
  name: nginx-web    # Le nom de votre deployment
```

**minReplicas** : Nombre minimum de pods
```yaml
minReplicas: 1    # Ne jamais descendre en dessous de 1
```

**maxReplicas** : Nombre maximum de pods
```yaml
maxReplicas: 10    # Ne jamais dépasser 10 pods
```

**metrics** : Sur quoi baser le scaling ?
```yaml
metrics:
- type: Resource
  resource:
    name: cpu    # Surveiller le CPU
    target:
      type: Utilization
      averageUtilization: 50    # Cible : 50% d'utilisation
```

## Vérifier l'état du HPA

### Lister les HPA actifs

```bash
kubectl get hpa
```

Sortie :
```
NAME             REFERENCE              TARGETS   MINPODS   MAXPODS   REPLICAS   AGE
nginx-web-hpa    Deployment/nginx-web   15%/50%   1         10        1          5m
```

Lecture de cette sortie :
- **TARGETS** : `15%/50%` = utilisation actuelle de 15% / cible de 50%
- **REPLICAS** : Actuellement 1 réplique (car charge faible)
- **MINPODS/MAXPODS** : Limites configurées

### Détails complets du HPA

Pour plus d'informations :

```bash
kubectl describe hpa nginx-web-hpa
```

Sortie détaillée :
```
Name:                     nginx-web-hpa
Namespace:                default
Reference:                Deployment/nginx-web
Metrics:                  ( current / target )
  resource cpu on pods:   15% / 50%
Min replicas:             1
Max replicas:             10
Deployment pods:          1 current / 1 desired
Conditions:
  Type            Status  Reason            Message
  ----            ------  ------            -------
  AbleToScale     True    ReadyForNewScale  recommended size matches current size
  ScalingActive   True    ValidMetricFound  the HPA was able to successfully calculate a replica count
Events:
  Type    Reason             Age   From                       Message
  ----    ------             ----  ----                       -------
  Normal  SuccessfulRescale  2m    horizontal-pod-autoscaler  New size: 3; reason: cpu resource utilization above target
```

## Comprendre le calcul du HPA

Le HPA utilise une formule simple pour déterminer le nombre de répliques :

```
Nombre de répliques désiré = ⌈Répliques actuelles × (Métrique actuelle / Métrique cible)⌉
```

**Exemple concret :**

État actuel :
- 2 répliques en cours d'exécution
- Utilisation CPU moyenne : 80%
- Cible CPU : 50%

Calcul :
```
Répliques désirées = 2 × (80 / 50) = 2 × 1.6 = 3.2 → 4 répliques
```

Le HPA va scaler à **4 répliques** (arrondi au supérieur).

### Pourquoi cette formule ?

Si l'utilisation actuelle (80%) est supérieure à la cible (50%), cela signifie que les pods sont surchargés. En ajoutant plus de répliques, la charge se répartit et l'utilisation moyenne diminue.

## Métriques disponibles

Le HPA peut se baser sur différents types de métriques :

### 1. Métriques de ressources (CPU et Mémoire)

**CPU - La plus courante :**

```yaml
metrics:
- type: Resource
  resource:
    name: cpu
    target:
      type: Utilization
      averageUtilization: 50    # 50% d'utilisation CPU
```

**Mémoire :**

```yaml
metrics:
- type: Resource
  resource:
    name: memory
    target:
      type: Utilization
      averageUtilization: 70    # 70% d'utilisation mémoire
```

**Important :** Pour que ces métriques fonctionnent, vos pods doivent avoir des `requests` définis :

```yaml
resources:
  requests:
    cpu: "100m"
    memory: "128Mi"
  limits:
    cpu: "500m"
    memory: "256Mi"
```

### 2. Métriques multiples

Vous pouvez combiner plusieurs métriques. Le HPA prendra le plus grand nombre de répliques calculé :

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: multi-metric-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: mon-app
  minReplicas: 2
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 50
  - type: Resource
    resource:
      name: memory
      target:
        type: Utilization
        averageUtilization: 80
```

**Comportement :**
- Si CPU recommande 5 répliques et mémoire recommande 3 répliques
- Le HPA choisira **5 répliques** (le maximum)

### 3. Métriques personnalisées (avancé)

Le HPA peut aussi utiliser des métriques custom comme :
- Nombre de requêtes HTTP par seconde
- Longueur de la file d'attente de messages
- Latence moyenne des requêtes
- Métriques métier spécifiques

Nous verrons cela dans la section 19.6 (Custom metrics).

## HPA en action : Simulation

Voyons comment le HPA réagit à une augmentation de charge.

### Étape 1 : Créer un deployment simple

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: php-apache
spec:
  replicas: 1
  selector:
    matchLabels:
      app: php-apache
  template:
    metadata:
      labels:
        app: php-apache
    spec:
      containers:
      - name: php-apache
        image: k8s.gcr.io/hpa-example
        ports:
        - containerPort: 80
        resources:
          requests:
            cpu: 200m
          limits:
            cpu: 500m
```

Cette image est conçue pour consommer du CPU lors des requêtes.

### Étape 2 : Créer le HPA

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: php-apache-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: php-apache
  minReplicas: 1
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 50
```

### Étape 3 : État initial

```bash
kubectl get hpa php-apache-hpa --watch
```

Sortie initiale (charge faible) :
```
NAME              REFERENCE                TARGETS   MINPODS   MAXPODS   REPLICAS
php-apache-hpa    Deployment/php-apache    0%/50%    1         10        1
```

### Étape 4 : Générer de la charge (optionnel pour test)

Dans un nouveau terminal :

```bash
kubectl run -i --tty load-generator --rm --image=busybox --restart=Never -- /bin/sh -c "while sleep 0.01; do wget -q -O- http://php-apache; done"
```

Cela envoie des requêtes en continu vers l'application.

### Étape 5 : Observer le scaling automatique

Avec `kubectl get hpa --watch`, vous verrez :

**Après 30 secondes :**
```
NAME              REFERENCE                TARGETS    MINPODS   MAXPODS   REPLICAS
php-apache-hpa    Deployment/php-apache    250%/50%   1         10        1
```

Le CPU monte à 250% (beaucoup plus que la cible de 50%).

**Après 1 minute :**
```
NAME              REFERENCE                TARGETS   MINPODS   MAXPODS   REPLICAS
php-apache-hpa    Deployment/php-apache    250%/50%  1         10        5
```

Le HPA a scalé à **5 répliques**.

**Après 2 minutes :**
```
NAME              REFERENCE                TARGETS   MINPODS   MAXPODS   REPLICAS
php-apache-hpa    Deployment/php-apache    45%/50%   1         10        5
```

Avec 5 répliques, l'utilisation CPU est maintenant autour de 45%, proche de la cible de 50%.

**Après avoir arrêté la charge (5-10 minutes) :**
```
NAME              REFERENCE                TARGETS   MINPODS   MAXPODS   REPLICAS
php-apache-hpa    Deployment/php-apache    0%/50%    1         10        1
```

Le HPA a automatiquement réduit à 1 réplique (le minimum).

## Configuration avancée du HPA

### Comportement de scaling (Behavior)

Depuis Kubernetes 1.18, vous pouvez contrôler finement le comportement de scaling :

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: advanced-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: mon-app
  minReplicas: 2
  maxReplicas: 20
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 50
  behavior:
    scaleUp:
      stabilizationWindowSeconds: 0
      policies:
      - type: Percent
        value: 100
        periodSeconds: 15
      - type: Pods
        value: 4
        periodSeconds: 15
      selectPolicy: Max
    scaleDown:
      stabilizationWindowSeconds: 300
      policies:
      - type: Percent
        value: 50
        periodSeconds: 15
      - type: Pods
        value: 2
        periodSeconds: 60
      selectPolicy: Min
```

**Explication des comportements :**

**scaleUp (montée en charge) :**
- **stabilizationWindowSeconds: 0** : Réagir immédiatement
- **100% toutes les 15 secondes** : Peut doubler le nombre de pods rapidement
- **4 pods toutes les 15 secondes** : Alternative pour scaling agressif
- **selectPolicy: Max** : Choisir le scaling le plus agressif

**scaleDown (réduction) :**
- **stabilizationWindowSeconds: 300** : Attendre 5 minutes avant de réduire
- **50% toutes les 15 secondes** : Réduire progressivement
- **2 pods par minute** : Réduction modérée
- **selectPolicy: Min** : Choisir le scaling le plus conservateur

### Pourquoi ces différences ?

**Scale up agressif :** Mieux vaut avoir trop de capacité temporairement que de perdre des requêtes
**Scale down prudent :** Éviter les oscillations (flapping) et donner du temps pour observer la tendance

## Bonnes pratiques du HPA

### 1. Toujours définir requests et limits

**Mauvais exemple (ne fonctionne pas) :**
```yaml
containers:
- name: app
  image: mon-app:latest
  # Pas de resources définis !
```

**Bon exemple :**
```yaml
containers:
- name: app
  image: mon-app:latest
  resources:
    requests:
      cpu: "250m"
      memory: "128Mi"
    limits:
      cpu: "1000m"
      memory: "512Mi"
```

Sans `requests`, le HPA ne peut pas calculer les pourcentages !

### 2. Choisir des seuils réalistes

**Trop bas (inefficace) :**
```yaml
averageUtilization: 20    # Scale trop tôt, gaspillage
```

**Trop haut (dangereux) :**
```yaml
averageUtilization: 95    # Pas de marge, risque de saturation
```

**Recommandé :**
```yaml
averageUtilization: 50-70    # Bon équilibre
```

### 3. Définir des limites min/max appropriées

```yaml
minReplicas: 2    # Au moins 2 pour la redondance
maxReplicas: 20   # Limite raisonnable pour éviter d'exploser les coûts
```

**Erreur courante :**
```yaml
minReplicas: 1    # Dangereux : un seul point de défaillance
maxReplicas: 1000 # Irréaliste : pourrait saturer le cluster
```

### 4. Tester avec des charges réalistes

Avant la production, testez avec des outils de charge :
- Apache Bench (ab)
- Hey
- Locust
- K6

Exemple avec hey :
```bash
hey -z 5m -c 50 http://mon-app.example.com
```

Observez comment le HPA réagit et ajustez les paramètres.

### 5. Surveiller le HPA

Utilisez les commandes suivantes régulièrement :

```bash
# État général
kubectl get hpa

# Détails et événements
kubectl describe hpa <nom>

# Métriques en temps réel
kubectl top pods

# Historique des scales
kubectl get events --field-selector involvedObject.name=<nom-hpa>
```

### 6. Commencer simple, complexifier progressivement

**Étape 1 :** HPA basé sur CPU uniquement
```yaml
metrics:
- type: Resource
  resource:
    name: cpu
    target:
      type: Utilization
      averageUtilization: 50
```

**Étape 2 :** Ajouter la mémoire
```yaml
metrics:
- type: Resource
  resource:
    name: cpu
    target:
      type: Utilization
      averageUtilization: 50
- type: Resource
  resource:
    name: memory
    target:
      type: Utilization
      averageUtilization: 70
```

**Étape 3 :** Ajouter des métriques custom (voir section 19.6)

### 7. Documenter vos choix

Utilisez des annotations pour expliquer vos décisions :

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: mon-hpa
  annotations:
    description: "HPA pour l'API principale. Scaling basé sur CPU à 50%. Max 10 pods pour respecter le budget."
    contact: "equipe-devops@example.com"
    last-tuned: "2025-10-15"
    tuning-notes: "Réduit maxReplicas de 20 à 10 après analyse des coûts"
spec:
  # ...
```

## Cas d'usage courants

### 1. API web avec trafic variable

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: api-web-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: api-web
  minReplicas: 3      # Minimum pour haute dispo
  maxReplicas: 50     # Peut gérer de gros pics
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 60
```

### 2. Worker de traitement de files

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: queue-worker-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: queue-worker
  minReplicas: 1      # Peut être à 0 avec KEDA
  maxReplicas: 20
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
```

### 3. Application batch nocturne

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: batch-processor-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: batch-processor
  minReplicas: 0      # Attention : nécessite une configuration spéciale
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 80
```

**Note :** Le scaling à 0 nécessite des solutions supplémentaires comme KEDA.

## Limitations et pièges à éviter

### 1. Le HPA ne peut pas scaler en dessous de minReplicas

Si vous définissez `minReplicas: 2`, le HPA ne descendra jamais à 0 ou 1, même si la charge est nulle.

### 2. Conflit avec le scaling manuel

**Ne faites pas ceci :**
```bash
# HPA est activé...
kubectl scale deployment mon-app --replicas=5  # Conflit !
```

Le HPA reprendra le contrôle et ajustera selon ses règles. Si vous voulez scaler manuellement, désactivez d'abord le HPA :

```bash
kubectl delete hpa mon-app-hpa
```

### 3. Métriques non disponibles

Si Metrics Server ne fonctionne pas ou si les pods n'ont pas de `requests`, vous verrez :

```
TARGETS: <unknown>/50%
```

Solution : Vérifiez Metrics Server et les resource requests.

### 4. Flapping (oscillations)

Le HPA peut osciller entre deux valeurs (ex: 3 ↔ 4 répliques) si les seuils sont trop serrés.

**Solution :** Augmentez `stabilizationWindowSeconds` pour scale down :

```yaml
behavior:
  scaleDown:
    stabilizationWindowSeconds: 300  # 5 minutes
```

### 5. Startup lent des pods

Si vos pods mettent 2 minutes à démarrer, le HPA peut créer trop de répliques avant que les premières soient prêtes.

**Solution :** Configurez des `readinessProbe` appropriées.

### 6. Scaling trop lent face à un pic brutal

Si votre application reçoit un trafic massif instantané, le HPA peut ne pas scaler assez vite.

**Solutions :**
- Augmentez `minReplicas` pour avoir plus de capacité de base
- Utilisez des politiques de scaling agressives
- Combinez avec le Cluster Autoscaler (multi-node)

## Supprimer un HPA

Pour désactiver l'autoscaling :

```bash
kubectl delete hpa mon-app-hpa
```

Le Deployment continue de fonctionner avec son nombre actuel de répliques, mais ne scale plus automatiquement.

## Résumé

Le **Horizontal Pod Autoscaler (HPA)** est un outil puissant pour automatiser le scaling de vos applications Kubernetes.

**Points clés :**

1. Le HPA ajuste automatiquement le nombre de répliques selon les métriques observées
2. Nécessite Metrics Server pour fonctionner
3. Basé principalement sur CPU et mémoire (mais extensible avec métriques custom)
4. Utilise une formule simple : `répliques = répliques_actuelles × (métrique_actuelle / métrique_cible)`
5. Plus réactif pour scale up que pour scale down (par design)
6. Nécessite que les pods aient des `requests` de ressources définies
7. Définissez toujours des limites min/max réalistes

**Comparaison avec le scaling manuel :**

| Aspect | Scaling Manuel | HPA |
|--------|---------------|-----|
| Intervention humaine | Requise | Automatique |
| Réactivité | Lente (dépend de vous) | Rapide (15s-3min) |
| Disponibilité 24/7 | Non | Oui |
| Optimisation coûts | Manuelle | Automatique |
| Complexité | Simple | Moyenne |
| Contrôle | Total | Délégué à K8s |

**Quand utiliser le HPA ?**

✅ Applications web avec trafic variable
✅ APIs avec pics de charge
✅ Workers de files d'attente
✅ Services avec patterns prévisibles mais variables
✅ Environnements de production nécessitant haute disponibilité

❌ Applications avec charge stable et prévisible (le scaling manuel suffit)
❌ Stateful applications complexes (bases de données) - nécessite une attention particulière
❌ Applications avec temps de démarrage très long (>5 minutes)

Dans les prochaines sections, nous découvrirons le **Vertical Pod Autoscaler (VPA)** qui ajuste les ressources CPU/RAM des pods, et comment utiliser des **métriques personnalisées** pour un scaling encore plus intelligent !

---

**Prochaine section :** 19.3 Vertical Pod Autoscaler (VPA) - Ajuster automatiquement les ressources de vos pods !

⏭️ [Vertical Pod Autoscaler (VPA)](/19-scaling-et-autoscaling/03-vertical-pod-autoscaler-vpa.md)
