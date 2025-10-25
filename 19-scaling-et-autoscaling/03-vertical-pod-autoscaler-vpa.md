🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 19.3 Vertical Pod Autoscaler (VPA)

## Introduction au VPA

Le **Vertical Pod Autoscaler (VPA)** est un contrôleur Kubernetes qui ajuste automatiquement les ressources CPU et mémoire de vos pods en fonction de leur utilisation réelle. Contrairement au HPA qui ajoute ou retire des pods, le VPA modifie les ressources allouées à chaque pod.

### Rappel : Horizontal vs Vertical

Avant d'entrer dans les détails, rappelons la différence fondamentale :

**Horizontal Pod Autoscaler (HPA) :**
- Change le **nombre** de pods
- Exemple : 2 pods → 5 pods
- Chaque pod garde les mêmes ressources

**Vertical Pod Autoscaler (VPA) :**
- Change les **ressources** de chaque pod
- Exemple : 1 pod avec 100m CPU → 1 pod avec 500m CPU
- Le nombre de pods reste le même

### Analogie simple

Imaginez une équipe de déménagement :

**Scaling horizontal (HPA) :**
- Vous engagez plus de déménageurs
- 2 déménageurs → 5 déménageurs
- Chaque déménageur a la même force

**Scaling vertical (VPA) :**
- Vous gardez le même nombre de déménageurs
- Mais vous leur donnez de meilleurs outils (diable, chariot)
- Chaque déménageur devient plus efficace

Le VPA "muscle" vos pods existants au lieu d'en créer de nouveaux !

## Pourquoi utiliser le VPA ?

### Problème courant : Mal dimensionner les ressources

Quand vous créez un Deployment, vous définissez les ressources :

```yaml
resources:
  requests:
    cpu: "100m"
    memory: "128Mi"
  limits:
    cpu: "500m"
    memory: "512Mi"
```

Mais comment savoir si ces valeurs sont bonnes ?

**Problèmes possibles :**

1. **Sous-dimensionnement** :
   - Requests trop faibles → Pod souvent throttlé (ralenti)
   - Limits trop faibles → Pod tué par OOMKill (Out Of Memory)
   - Performance médiocre

2. **Sur-dimensionnement** :
   - Requests trop élevées → Gaspillage de ressources
   - Coûts inutiles
   - Moins de pods peuvent tenir sur le cluster

### Solution : Le VPA

Le VPA résout ce problème en :

1. **Observant** l'utilisation réelle de vos pods
2. **Calculant** les ressources optimales
3. **Recommandant** ou **appliquant** automatiquement ces valeurs

**Avantages :**

- ✅ Dimensionnement optimal automatique
- ✅ Meilleure utilisation des ressources du cluster
- ✅ Réduction des coûts (pas de sur-provisionnement)
- ✅ Meilleures performances (pas de sous-provisionnement)
- ✅ Moins de maintenance (ajustements automatiques)

## Comment fonctionne le VPA ?

Le VPA est composé de trois composants qui travaillent ensemble :

### 1. VPA Recommender

**Rôle :** Observer et calculer

- Surveille l'utilisation CPU et mémoire de tous les pods
- Conserve un historique (par défaut 8 jours)
- Calcule des recommandations basées sur les patterns observés
- Utilise des statistiques (percentiles) pour éviter les valeurs extrêmes

**Exemple :**
Si votre pod utilise en moyenne 150m CPU mais parfois monte à 400m, le Recommender suggérera peut-être 250m (pour couvrir la majorité des cas).

### 2. VPA Updater

**Rôle :** Appliquer les recommandations

- Lit les recommandations du Recommender
- Décide si un pod doit être mis à jour
- Évite les mises à jour trop fréquentes (cooldown period)
- Respecte les PodDisruptionBudgets (PDB)

**Important :** Pour appliquer les nouvelles ressources, le VPA doit **redémarrer le pod** ! Ce n'est pas possible à chaud actuellement dans Kubernetes.

### 3. VPA Admission Controller

**Rôle :** Intercepter et modifier

- Webhook qui intercepte la création de nouveaux pods
- Applique automatiquement les recommandations du VPA
- Permet d'appliquer les bonnes ressources dès le démarrage

**Flux :**
```
Nouvelle demande de pod
   ↓
Admission Controller intercepte
   ↓
Consulte les recommandations VPA
   ↓
Modifie les resources requests/limits
   ↓
Pod créé avec les bonnes ressources
```

## Installation du VPA sur MicroK8s

Malheureusement, contrairement au HPA, le VPA n'est pas disponible comme addon MicroK8s natif. Il faut l'installer manuellement.

### Prérequis

1. **Metrics Server** doit être activé :

```bash
microk8s enable metrics-server
```

2. **Git** pour télécharger les manifestes :

```bash
sudo apt install git    # Sur Ubuntu/Debian
```

### Installation manuelle du VPA

**Étape 1 : Cloner le dépôt officiel**

```bash
git clone https://github.com/kubernetes/autoscaler.git
cd autoscaler/vertical-pod-autoscaler
```

**Étape 2 : Déployer le VPA**

```bash
./hack/vpa-up.sh
```

Ce script installe les trois composants du VPA dans le namespace `kube-system`.

**Étape 3 : Vérifier l'installation**

```bash
kubectl get pods -n kube-system | grep vpa
```

Vous devriez voir trois pods :

```
vpa-admission-controller-xxxxxxxxx-xxxxx   1/1     Running   0          2m
vpa-recommender-xxxxxxxxx-xxxxx            1/1     Running   0          2m
vpa-updater-xxxxxxxxx-xxxxx                1/1     Running   0          2m
```

**Étape 4 : Vérifier les CRDs**

```bash
kubectl get crd | grep verticalpodautoscaler
```

Sortie attendue :

```
verticalpodautoscalercheckpoints.autoscaling.k8s.io
verticalpodautoscalers.autoscaling.k8s.io
```

Si vous voyez ces lignes, le VPA est installé correctement !

## Les modes du VPA

Le VPA peut fonctionner en plusieurs modes selon vos besoins :

### 1. Mode "Off" (Recommandations uniquement)

**Comportement :** Le VPA observe et recommande, mais n'applique rien automatiquement.

**Utilisation :** Parfait pour commencer et comprendre ce que le VPA recommande avant de lui donner le contrôle.

```yaml
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
metadata:
  name: mon-app-vpa
spec:
  targetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: mon-app
  updatePolicy:
    updateMode: "Off"    # Mode recommandation uniquement
```

**Consulter les recommandations :**

```bash
kubectl describe vpa mon-app-vpa
```

Vous verrez une section comme :

```
Recommendation:
  Container Recommendations:
    Container Name:  mon-app
    Lower Bound:
      Cpu:     100m
      Memory:  128Mi
    Target:
      Cpu:     250m
      Memory:  256Mi
    Uncapped Target:
      Cpu:     300m
      Memory:  300Mi
    Upper Bound:
      Cpu:     500m
      Memory:  512Mi
```

**Interprétation :**
- **Lower Bound** : Minimum absolu recommandé
- **Target** : Valeur optimale recommandée
- **Upper Bound** : Maximum recommandé
- **Uncapped Target** : Ce que le VPA recommanderait sans limites

### 2. Mode "Initial" (Application au démarrage)

**Comportement :** Le VPA applique les recommandations uniquement lors de la création de nouveaux pods, pas aux pods existants.

**Utilisation :** Moins perturbant que le mode Auto, car ne redémarre pas les pods existants.

```yaml
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
metadata:
  name: mon-app-vpa
spec:
  targetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: mon-app
  updatePolicy:
    updateMode: "Initial"    # Application uniquement aux nouveaux pods
```

**Cas d'usage :**
- Applications où vous faites souvent des déploiements
- Quand vous voulez éviter les redémarrages automatiques
- Environnements de développement

### 3. Mode "Recreate" (Application avec redémarrage)

**Comportement :** Le VPA peut redémarrer les pods existants pour appliquer les nouvelles ressources.

**Attention :** Ce mode cause des interruptions de service ! À utiliser avec prudence.

```yaml
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
metadata:
  name: mon-app-vpa
spec:
  targetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: mon-app
  updatePolicy:
    updateMode: "Recreate"    # Peut redémarrer les pods
```

**Précautions :**
- Assurez-vous d'avoir plusieurs répliques (haute disponibilité)
- Utilisez des PodDisruptionBudgets (PDB) pour limiter les interruptions
- Testez d'abord en environnement de dev

### 4. Mode "Auto" (Non encore disponible)

**Statut :** Mode futur qui permettrait de mettre à jour les ressources sans redémarrer les pods.

**Actuellement :** Équivalent à "Recreate" dans la plupart des implémentations.

## Configuration de base du VPA

### Exemple complet

Créons un VPA pour une application web :

**1. Le Deployment à surveiller :**

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-web
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx-web
  template:
    metadata:
      labels:
        app: nginx-web
    spec:
      containers:
      - name: nginx
        image: nginx:latest
        resources:
          requests:
            cpu: "100m"
            memory: "128Mi"
          limits:
            cpu: "500m"
            memory: "512Mi"
```

**2. Le VPA associé :**

```yaml
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
metadata:
  name: nginx-web-vpa
spec:
  targetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: nginx-web
  updatePolicy:
    updateMode: "Off"    # Commencez par "Off" pour observer
  resourcePolicy:
    containerPolicies:
    - containerName: nginx
      minAllowed:
        cpu: "50m"
        memory: "64Mi"
      maxAllowed:
        cpu: "1000m"
        memory: "1Gi"
      controlledResources:
      - cpu
      - memory
```

**Appliquer les configurations :**

```bash
kubectl apply -f nginx-deployment.yaml
kubectl apply -f nginx-vpa.yaml
```

### Décomposition de la configuration VPA

**targetRef :** Quel workload surveiller ?

```yaml
targetRef:
  apiVersion: apps/v1
  kind: Deployment       # Peut être Deployment, StatefulSet, DaemonSet, etc.
  name: nginx-web        # Nom exact du workload
```

**updatePolicy :** Comment appliquer les recommandations ?

```yaml
updatePolicy:
  updateMode: "Off"      # Off, Initial, Recreate, ou Auto
```

**resourcePolicy :** Contraintes et limites

```yaml
resourcePolicy:
  containerPolicies:
  - containerName: nginx      # Nom du conteneur dans le pod
    minAllowed:               # Ressources minimum
      cpu: "50m"
      memory: "64Mi"
    maxAllowed:               # Ressources maximum
      cpu: "1000m"
      memory: "1Gi"
    controlledResources:      # Quelles ressources contrôler
    - cpu
    - memory
```

**controlledResources** peut être :
- `cpu` : Contrôler uniquement le CPU
- `memory` : Contrôler uniquement la mémoire
- `cpu` et `memory` : Contrôler les deux (par défaut)

## Consulter les recommandations du VPA

Une fois le VPA créé, attendez quelques minutes (5-10 minutes minimum) pour que le Recommender collecte suffisamment de données.

### Commande de base

```bash
kubectl describe vpa nginx-web-vpa
```

### Sortie détaillée

```yaml
Name:         nginx-web-vpa
Namespace:    default
API Version:  autoscaling.k8s.io/v1
Kind:         VerticalPodAutoscaler

Spec:
  Target Ref:
    API Version:  apps/v1
    Kind:         Deployment
    Name:         nginx-web
  Update Policy:
    Update Mode:  Off

Status:
  Conditions:
    Last Transition Time:  2025-10-25T10:00:00Z
    Status:                True
    Type:                  RecommendationProvided

  Recommendation:
    Container Recommendations:
      Container Name:  nginx
      Lower Bound:
        Cpu:     50m
        Memory:  100Mi
      Target:
        Cpu:     150m
        Memory:  200Mi
      Uncapped Target:
        Cpu:     180m
        Memory:  220Mi
      Upper Bound:
        Cpu:     300m
        Memory:  400Mi
```

### Interpréter les recommandations

**Lower Bound (Limite inférieure) :**
- Ressources minimales observées
- Ne descendez jamais en dessous de ces valeurs
- Exemple : Cpu: 50m, Memory: 100Mi

**Target (Cible recommandée) :**
- **C'est la valeur à utiliser !**
- Basée sur l'analyse des patterns d'utilisation
- Généralement au percentile 90-95%
- Exemple : Cpu: 150m, Memory: 200Mi

**Upper Bound (Limite supérieure) :**
- Ressources maximales recommandées
- Pour couvrir les pics exceptionnels
- Exemple : Cpu: 300m, Memory: 400Mi

**Uncapped Target :**
- Recommandation sans tenir compte de vos contraintes (minAllowed/maxAllowed)
- Utile pour voir ce que le VPA recommanderait idéalement

### Appliquer manuellement les recommandations

En mode "Off", vous devez appliquer manuellement. Modifiez votre Deployment :

```yaml
resources:
  requests:
    cpu: "150m"        # Était 100m, maintenant 150m (Target)
    memory: "200Mi"    # Était 128Mi, maintenant 200Mi (Target)
  limits:
    cpu: "300m"        # Utilisez Upper Bound comme limite
    memory: "400Mi"
```

Puis appliquez :

```bash
kubectl apply -f nginx-deployment.yaml
```

## VPA en mode automatique (Recreate)

Si vous êtes prêt à laisser le VPA gérer automatiquement, changez le mode :

```yaml
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
metadata:
  name: nginx-web-vpa
spec:
  targetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: nginx-web
  updatePolicy:
    updateMode: "Recreate"    # Mode automatique
  resourcePolicy:
    containerPolicies:
    - containerName: nginx
      minAllowed:
        cpu: "50m"
        memory: "64Mi"
      maxAllowed:
        cpu: "1000m"
        memory: "1Gi"
```

**Comportement :**

Le VPA va automatiquement :
1. Observer l'utilisation des pods
2. Calculer les ressources optimales
3. Redémarrer les pods un par un avec les nouvelles ressources
4. Respecter les contraintes de disponibilité (si configurées)

**Observer les changements :**

```bash
# Voir les événements
kubectl get events --sort-by=.metadata.creationTimestamp

# Voir les pods redémarrer
kubectl get pods -w
```

Vous verrez des messages comme :
```
pod/nginx-web-xxxxx   Terminating
pod/nginx-web-yyyyy   Creating
pod/nginx-web-yyyyy   Running
```

## Utiliser le VPA avec des PodDisruptionBudgets

Pour éviter trop de perturbations lors des mises à jour automatiques, utilisez un **PodDisruptionBudget (PDB)** :

```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: nginx-web-pdb
spec:
  minAvailable: 2    # Au moins 2 pods doivent rester disponibles
  selector:
    matchLabels:
      app: nginx-web
```

Cela garantit que le VPA ne redémarrera jamais plus d'un pod à la fois (si vous avez 3 répliques).

## Configuration avancée

### Contrôler uniquement certaines ressources

Vous pouvez demander au VPA de gérer uniquement le CPU ou uniquement la mémoire :

**CPU uniquement :**

```yaml
resourcePolicy:
  containerPolicies:
  - containerName: mon-app
    controlledResources:
    - cpu    # Seulement le CPU
```

**Mémoire uniquement :**

```yaml
resourcePolicy:
  containerPolicies:
  - containerName: mon-app
    controlledResources:
    - memory    # Seulement la mémoire
```

### Différentes politiques par conteneur

Si votre pod a plusieurs conteneurs :

```yaml
resourcePolicy:
  containerPolicies:
  - containerName: nginx
    minAllowed:
      cpu: "100m"
      memory: "128Mi"
    maxAllowed:
      cpu: "1000m"
      memory: "1Gi"
  - containerName: sidecar
    minAllowed:
      cpu: "50m"
      memory: "64Mi"
    maxAllowed:
      cpu: "500m"
      memory: "512Mi"
```

### Mode de mise à jour par conteneur

```yaml
resourcePolicy:
  containerPolicies:
  - containerName: nginx
    mode: "Auto"          # Mode par conteneur
    controlledResources:
    - cpu
    - memory
```

## VPA et HPA : Peut-on les utiliser ensemble ?

### Attention : Conflit potentiel !

**Le problème :**

- **HPA** : Ajoute/retire des pods basé sur l'utilisation CPU/mémoire
- **VPA** : Modifie l'utilisation CPU/mémoire des pods

Si les deux ciblent les mêmes métriques (CPU ou mémoire), ils peuvent entrer en conflit :

```
1. HPA voit CPU élevé → ajoute des pods
2. VPA voit CPU élevé → augmente les ressources par pod
3. Utilisation CPU baisse (plus de ressources)
4. HPA retire des pods (CPU trop bas)
5. CPU remonte → retour à l'étape 1
```

Cela crée une boucle instable !

### Solutions pour les utiliser ensemble

**Option 1 : Séparer les métriques**

- **HPA** : Basé sur CPU
- **VPA** : Contrôle uniquement la mémoire

```yaml
# HPA
metrics:
- type: Resource
  resource:
    name: cpu
    target:
      type: Utilization
      averageUtilization: 50

---
# VPA
resourcePolicy:
  containerPolicies:
  - containerName: mon-app
    controlledResources:
    - memory    # Seulement la mémoire
```

**Option 2 : HPA avec métriques custom**

- **HPA** : Basé sur métriques métier (requêtes/sec, longueur queue)
- **VPA** : Gère CPU et mémoire

```yaml
# HPA avec métrique custom
metrics:
- type: Pods
  pods:
    metric:
      name: http_requests_per_second
    target:
      type: AverageValue
      averageValue: "1000"

---
# VPA normal
resourcePolicy:
  containerPolicies:
  - containerName: mon-app
    controlledResources:
    - cpu
    - memory
```

**Option 3 : VPA en mode Off + HPA**

- **VPA** : Mode "Off" pour les recommandations
- **HPA** : Gère le scaling automatique
- Vous : Appliquez manuellement les recommandations VPA périodiquement

C'est souvent l'approche la plus sûre pour commencer.

### Recommandation générale

Pour la plupart des cas, choisissez **l'un ou l'autre** :

- **Applications stateless avec trafic variable** → HPA
- **Applications avec peu de répliques mais besoins variables** → VPA
- **Cas avancés avec expertise** → Les deux (mais attention aux conflits)

## Cas d'usage du VPA

### 1. Applications avec besoins variables dans le temps

Une application de traitement de données qui a besoin de plus de ressources pendant la journée et moins la nuit.

```yaml
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
metadata:
  name: data-processor-vpa
spec:
  targetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: data-processor
  updatePolicy:
    updateMode: "Auto"
  resourcePolicy:
    containerPolicies:
    - containerName: processor
      minAllowed:
        cpu: "200m"
        memory: "256Mi"
      maxAllowed:
        cpu: "4000m"
        memory: "8Gi"
```

### 2. StatefulSets avec 1-3 répliques

Pour les bases de données ou applications stateful où vous ne pouvez pas facilement scaler horizontalement :

```yaml
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
metadata:
  name: postgres-vpa
spec:
  targetRef:
    apiVersion: apps/v1
    kind: StatefulSet
    name: postgres
  updatePolicy:
    updateMode: "Initial"    # Moins perturbant pour des bases de données
  resourcePolicy:
    containerPolicies:
    - containerName: postgres
      minAllowed:
        cpu: "500m"
        memory: "1Gi"
      maxAllowed:
        cpu: "4000m"
        memory: "16Gi"
```

### 3. Optimisation des coûts en développement

En environnement de dev, utilisez le VPA pour découvrir les vraies ressources nécessaires :

```yaml
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
metadata:
  name: dev-app-vpa
spec:
  targetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: dev-app
  updatePolicy:
    updateMode: "Off"    # Mode recommandation
```

Laissez tourner pendant 1-2 semaines, puis utilisez les recommandations pour dimensionner correctement en production.

### 4. Microservices avec patterns de charge inconnus

Nouveau microservice dont vous ne connaissez pas encore les besoins :

```yaml
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
metadata:
  name: new-service-vpa
spec:
  targetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: new-service
  updatePolicy:
    updateMode: "Initial"
  resourcePolicy:
    containerPolicies:
    - containerName: service
      minAllowed:
        cpu: "100m"
        memory: "128Mi"
      maxAllowed:
        cpu: "2000m"
        memory: "4Gi"
```

## Limitations et précautions du VPA

### 1. Nécessite un redémarrage de pod

**Problème :** Kubernetes ne peut pas changer les ressources d'un pod en cours d'exécution.

**Impact :** Interruption de service momentanée (même brève)

**Mitigation :**
- Ayez plusieurs répliques
- Utilisez des PodDisruptionBudgets
- Ou utilisez le mode "Initial" pour éviter les redémarrages

### 2. Pas adapté à toutes les applications

**Applications inadaptées au VPA :**
- Applications avec une seule réplique (single point of failure)
- Bases de données critiques sans réplication
- Applications avec startup très long (>5 minutes)
- Workloads batch/jobs ponctuels

**Préférez le HPA pour :**
- Applications complètement stateless
- Haute disponibilité requise
- Scaling rapide nécessaire

### 3. Temps de convergence

Le VPA prend du temps pour collecter des données :
- **Premières recommandations** : 1-2 minutes
- **Recommandations fiables** : Quelques heures à quelques jours
- **Historique complet** : 8 jours par défaut

**Ne vous attendez pas à des recommandations précises immédiatement !**

### 4. Consommation de ressources du VPA lui-même

Les trois composants du VPA consomment des ressources :
- **Recommender** : ~50-100m CPU, ~200-500Mi mémoire
- **Updater** : ~25-50m CPU, ~100-200Mi mémoire
- **Admission Controller** : ~25-50m CPU, ~100-200Mi mémoire

Sur un petit cluster, cela peut être significatif.

### 5. Manque de support natif

Le VPA n'est pas aussi mature que le HPA :
- Pas d'addon MicroK8s natif
- Documentation moins complète
- Moins de support communautaire
- Évolution plus lente

## Monitoring du VPA

### Vérifier le statut

```bash
# Lister tous les VPA
kubectl get vpa

# Détails d'un VPA spécifique
kubectl describe vpa mon-app-vpa

# Voir les recommandations en format YAML
kubectl get vpa mon-app-vpa -o yaml
```

### Voir les événements

```bash
kubectl get events --field-selector involvedObject.name=mon-app-vpa --sort-by=.metadata.creationTimestamp
```

### Logs des composants VPA

```bash
# Logs du Recommender
kubectl logs -n kube-system deployment/vpa-recommender

# Logs de l'Updater
kubectl logs -n kube-system deployment/vpa-updater

# Logs de l'Admission Controller
kubectl logs -n kube-system deployment/vpa-admission-controller
```

## Désinstaller le VPA

Si vous voulez retirer le VPA de votre cluster :

```bash
cd autoscaler/vertical-pod-autoscaler
./hack/vpa-down.sh
```

Cela supprime tous les composants VPA du cluster.

## Bonnes pratiques

### 1. Commencez toujours en mode "Off"

Observez d'abord avant d'automatiser :

```yaml
updatePolicy:
  updateMode: "Off"    # Observer d'abord
```

Après 1-2 semaines d'observation, passez à "Initial" ou "Recreate".

### 2. Définissez toujours minAllowed et maxAllowed

Évitez les surprises :

```yaml
resourcePolicy:
  containerPolicies:
  - containerName: mon-app
    minAllowed:        # Empêche le sous-dimensionnement
      cpu: "50m"
      memory: "64Mi"
    maxAllowed:        # Empêche le sur-dimensionnement
      cpu: "2000m"
      memory: "4Gi"
```

### 3. Utilisez des PodDisruptionBudgets

Protégez la disponibilité :

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

### 4. Testez en environnement de dev d'abord

Ne testez jamais le VPA directement en production !

### 5. Surveillez les recommandations régulièrement

Même en mode "Off", consultez les recommandations mensuellement :

```bash
kubectl describe vpa --all-namespaces
```

### 6. Documentez vos décisions

```yaml
metadata:
  annotations:
    description: "VPA pour l'API principale. Mode Initial pour éviter les interruptions."
    tuning-date: "2025-10-25"
    notes: "maxAllowed défini à 4Gi après observation de pics à 3.5Gi"
```

### 7. Combinez avec des métriques et alertes

Utilisez Prometheus et Grafana pour surveiller :
- Les recommandations vs les ressources actuelles
- Les évictions et redémarrages
- L'écart entre requests et utilisation réelle

## Comparaison récapitulative : Manual vs HPA vs VPA

| Aspect | Scaling Manuel | HPA | VPA |
|--------|---------------|-----|-----|
| **Qu'est-ce qui change ?** | Nombre de répliques | Nombre de répliques | Ressources par pod |
| **Déclenchement** | Vous | Automatique | Automatique |
| **Basé sur** | Votre décision | Métriques en temps réel | Historique d'utilisation |
| **Redémarrage pods** | Non | Non | Oui (sauf mode Initial) |
| **Adapté pour** | Pics prévisibles | Charge variable | Dimensionnement optimal |
| **Complexité** | Simple | Moyenne | Moyenne-Élevée |
| **Maturité** | N/A | Très mature | Moins mature |
| **Disponibilité 24/7** | Non | Oui | Oui |

## Résumé

Le **Vertical Pod Autoscaler (VPA)** est un outil puissant pour optimiser les ressources de vos pods automatiquement.

**Points clés :**

1. Le VPA ajuste les ressources CPU et mémoire des pods, pas leur nombre
2. Fonctionne en observant l'utilisation réelle sur plusieurs jours
3. Trois composants : Recommender, Updater, Admission Controller
4. Quatre modes : Off, Initial, Recreate, Auto
5. Nécessite généralement un redémarrage des pods (limitation actuelle de K8s)
6. Différent du HPA : vertical vs horizontal scaling
7. Attention aux conflits si utilisé avec HPA sur les mêmes métriques
8. Installation manuelle nécessaire sur MicroK8s

**Quand utiliser le VPA ?**

✅ Applications stateful avec peu de répliques
✅ Workloads avec patterns d'utilisation variables dans le temps
✅ Optimisation des coûts et des ressources
✅ Applications dont vous ne connaissez pas les besoins exacts
✅ Environnements de dev pour découvrir les vrais besoins

❌ Applications nécessitant zéro downtime absolu
❌ Pods avec une seule réplique critique
❌ Si vous avez besoin de scaling très rapide (utilisez HPA)
❌ Workloads batch/jobs ponctuels

**Conseil pour débutants :**

Commencez par le mode "Off" pendant au moins une semaine, observez les recommandations, puis décidez si vous voulez passer en mode automatique. Le VPA est plus un outil d'optimisation à long terme qu'un outil de réactivité immédiate !

---

**Prochaine section :** 19.4 Cluster Autoscaler - Ajuster automatiquement la taille de votre cluster !

⏭️ [Cluster Autoscaler (pour multi-node)](/19-scaling-et-autoscaling/04-cluster-autoscaler-pour-multi-node.md)
