🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 23.8 Performance et Ressources

## Introduction

La gestion des ressources (CPU et mémoire) est cruciale dans Kubernetes. Une mauvaise configuration peut entraîner des pods qui ne démarrent pas, des applications lentes, des crashs, ou un cluster entier qui devient instable. Comprendre comment Kubernetes gère les ressources et savoir diagnostiquer les problèmes de performance est essentiel pour maintenir un cluster sain.

> **Pour les débutants** : Imaginez votre cluster Kubernetes comme un hôtel. Chaque pod est comme un client qui réserve une chambre (ressources). Si un client demande une suite présidentielle (beaucoup de ressources) mais qu'il n'y en a plus de disponible, il ne peut pas s'enregistrer (le pod ne démarre pas). Si certains clients utilisent trop de ressources communes (salle de sport, restaurant), cela dégrade l'expérience pour tous (ralentissement du cluster). Cette section vous apprend à gérer efficacement cet "hôtel".

## Comprendre les Ressources dans Kubernetes

### Les Deux Types de Ressources

Kubernetes gère principalement deux types de ressources :

#### 1. CPU (Processeur)

**Unités** :
- Mesurée en "cores" ou "millicores" (m)
- 1 core = 1000 millicores (1000m)
- 100m = 0.1 core = 10% d'un CPU

**Exemples** :
```yaml
resources:
  requests:
    cpu: 100m      # 0.1 core
  limits:
    cpu: 500m      # 0.5 core
```

**Caractéristique importante** : La CPU est une ressource **compressible**
- Si un pod demande plus de CPU qu'alloué, il sera simplement ralenti (throttled)
- Le pod ne crashe pas, il tourne juste plus lentement

#### 2. Mémoire (RAM)

**Unités** :
- Mesurée en bytes, mais on utilise généralement :
  - `Ki` = Kibibytes (1024 bytes)
  - `Mi` = Mebibytes (1024 Ki)
  - `Gi` = Gibibytes (1024 Mi)

**Exemples** :
```yaml
resources:
  requests:
    memory: 128Mi   # 128 Mebibytes
  limits:
    memory: 512Mi   # 512 Mebibytes
```

**Caractéristique importante** : La mémoire est une ressource **incompressible**
- Si un pod dépasse sa limite de mémoire, il est **tué** (OOMKilled - Out Of Memory Killed)
- Le pod crashe et redémarre

### Requests vs Limits

Deux concepts fondamentaux :

#### Requests (Demandes)

**Définition** : La quantité **minimale** de ressources garantie au pod.

**Utilisation** :
- Kubernetes utilise les requests pour **décider sur quel nœud placer le pod**
- Le scheduler cherche un nœud avec suffisamment de ressources disponibles

**Exemple** :
```yaml
resources:
  requests:
    cpu: 250m
    memory: 256Mi
```

Kubernetes garantit que ce pod aura au moins 250m CPU et 256Mi de mémoire disponibles.

#### Limits (Limites)

**Définition** : La quantité **maximale** de ressources qu'un pod peut utiliser.

**Utilisation** :
- Empêche un pod de monopoliser toutes les ressources
- Protection contre les fuites mémoire ou boucles infinies

**Exemple** :
```yaml
resources:
  limits:
    cpu: 1000m
    memory: 1Gi
```

Le pod ne pourra jamais utiliser plus d'1 CPU core ou 1 Gi de mémoire.

### Différence entre Requests et Limits

**Analogie** :
- **Request** = La taille de chambre d'hôtel que vous avez réservée (garantie)
- **Limit** = La taille maximale que votre chambre peut avoir (vous ne pouvez pas agrandir au-delà)

**Configuration recommandée** :
```yaml
resources:
  requests:
    cpu: 100m        # Ce dont j'ai besoin minimum
    memory: 128Mi
  limits:
    cpu: 500m        # Ce que je peux utiliser maximum
    memory: 512Mi
```

**Cas spéciaux** :

1. **Pas de requests ni limits** :
```yaml
# Aucune garantie, aucune limite
# Mauvaise pratique en production !
```

2. **Requests seulement** :
```yaml
resources:
  requests:
    cpu: 250m
    memory: 256Mi
# Pas de limits = peut utiliser autant que disponible
```

3. **Limits seulement** :
```yaml
resources:
  limits:
    cpu: 500m
    memory: 512Mi
# Kubernetes met automatiquement requests = limits
```

4. **Requests = Limits** :
```yaml
resources:
  requests:
    cpu: 500m
    memory: 512Mi
  limits:
    cpu: 500m
    memory: 512Mi
# QoS = Guaranteed (meilleure garantie)
```

## Problème 1 : Pod Bloqué en Pending (Ressources Insuffisantes)

### Symptômes

Le pod reste en état `Pending` indéfiniment.

```bash
microk8s kubectl get pods
# NAME      READY   STATUS    RESTARTS   AGE
# mon-pod   0/1     Pending   0          5m
```

**Message typique** :
```
0/1 nodes are available: 1 Insufficient cpu.
0/1 nodes are available: 1 Insufficient memory.
```

### Diagnostic

#### Étape 1 : Examiner le Pod

```bash
microk8s kubectl describe pod mon-pod
```

**Chercher dans Events** :
```
Events:
  Type     Reason            Message
  ----     ------            -------
  Warning  FailedScheduling  0/1 nodes are available: 1 Insufficient cpu.
```

#### Étape 2 : Voir les Ressources Demandées

```bash
# Ressources demandées par le pod
microk8s kubectl get pod mon-pod -o jsonpath='{.spec.containers[*].resources}'
```

#### Étape 3 : Vérifier les Ressources Disponibles

```bash
# Vue d'ensemble du nœud
microk8s kubectl describe node

# Chercher les sections :
# - Capacity: Ressources totales du nœud
# - Allocatable: Ressources disponibles pour les pods
# - Allocated resources: Ce qui est déjà utilisé
```

**Exemple de sortie** :
```
Capacity:
  cpu:                4
  memory:             8Gi
Allocatable:
  cpu:                4
  memory:             7.5Gi
Allocated resources:
  Resource           Requests      Limits
  --------           --------      ------
  cpu                3500m (87%)   7000m (175%)
  memory             6Gi (80%)     12Gi (160%)
```

**Interprétation** :
- **Capacity** : Ressources physiques totales
- **Allocatable** : Un peu moins (réservé pour le système)
- **Requests** : Ce qui est garanti aux pods existants
- **Limits** : Ce que les pods peuvent utiliser au maximum

#### Étape 4 : Calculer les Ressources Disponibles

```bash
# Script pour voir les ressources disponibles
microk8s kubectl describe node | grep -A 5 "Allocated resources"
```

**Disponible pour nouveaux pods** = Allocatable - Requests (pas Limits !)

### Solutions Courantes

#### Solution 1 : Réduire les Requests du Pod

Si le pod demande trop de ressources :

```yaml
# Avant
resources:
  requests:
    cpu: 2000m      # 2 cores
    memory: 4Gi

# Après
resources:
  requests:
    cpu: 500m       # 0.5 core
    memory: 1Gi
```

**Comment déterminer les bonnes valeurs** :
1. Commencer avec des valeurs conservatrices
2. Surveiller l'utilisation réelle
3. Ajuster en fonction des besoins

#### Solution 2 : Supprimer des Pods pour Libérer des Ressources

```bash
# Identifier les pods gourmands
microk8s kubectl top pods --sort-by=memory
microk8s kubectl top pods --sort-by=cpu

# Supprimer ou réduire l'échelle de certains pods
microk8s kubectl scale deployment app-non-critique --replicas=0
```

#### Solution 3 : Ajouter de la Capacité au Nœud

**Option 1 : Augmenter les ressources du nœud existant**
- Ajouter de la RAM
- Ajouter des CPU cores
- Redémarrer MicroK8s

**Option 2 : Ajouter un nœud au cluster**
```bash
# Sur le nœud existant
microk8s add-node

# Sur le nouveau nœud (suivre les instructions affichées)
microk8s join <token>
```

#### Solution 4 : Optimiser les Requests Existants

Beaucoup de pods demandent trop de ressources "au cas où" :

```bash
# Voir l'utilisation réelle vs demandée
microk8s kubectl top pods

# Comparer avec les requests
microk8s kubectl get pods -o custom-columns=NAME:.metadata.name,CPU_REQUEST:.spec.containers[0].resources.requests.cpu,MEM_REQUEST:.spec.containers[0].resources.requests.memory
```

Si un pod demande 1Gi mais n'utilise que 200Mi, vous pouvez réduire ses requests.

## Problème 2 : Pod Tué (OOMKilled)

### Symptômes

Le pod est constamment redémarré avec le statut `OOMKilled` (Out Of Memory).

```bash
microk8s kubectl get pods
# NAME      READY   STATUS      RESTARTS   AGE
# mon-pod   0/1     OOMKilled   5          10m

# Ou après redémarrage
# NAME      READY   STATUS      RESTARTS   AGE
# mon-pod   1/1     Running     5          10m
```

**RESTARTS élevé** est un indicateur clé.

### Diagnostic

#### Étape 1 : Confirmer le OOMKill

```bash
# Voir les détails du pod
microk8s kubectl describe pod mon-pod

# Chercher dans Events
# Events:
#   Warning  OOMKilled  Container was OOMKilled
```

**Dans les logs du conteneur précédent** :
```bash
microk8s kubectl logs mon-pod --previous
```

Souvent, vous verrez les derniers messages avant le kill.

#### Étape 2 : Voir les Limites Actuelles

```bash
# Limites du pod
microk8s kubectl get pod mon-pod -o jsonpath='{.spec.containers[*].resources.limits.memory}'
```

#### Étape 3 : Voir l'Utilisation Réelle

```bash
# Utilisation actuelle (si le pod tourne)
microk8s kubectl top pod mon-pod

# Sortie :
# NAME      CPU(cores)   MEMORY(bytes)
# mon-pod   100m         450Mi
```

**Note** : Le pod peut être tué avant que vous ne puissiez voir son pic d'utilisation.

#### Étape 4 : Comprendre Pourquoi

**Causes courantes** :
1. **Limite trop basse** : L'application a réellement besoin de plus de mémoire
2. **Fuite mémoire** : Bug dans l'application qui accumule de la mémoire
3. **Pic temporaire** : Charge inhabituelle qui dépasse les limites
4. **Pas de limite définie** : Le pod utilise toute la mémoire disponible

### Solutions Courantes

#### Solution 1 : Augmenter la Limite Mémoire

Si l'application a besoin de plus de mémoire :

```yaml
# Avant
resources:
  limits:
    memory: 512Mi

# Après
resources:
  limits:
    memory: 1Gi      # Doubler ou augmenter selon besoin
```

**Attention** : Assurez-vous que le nœud a assez de mémoire totale.

#### Solution 2 : Identifier et Corriger une Fuite Mémoire

**Signes d'une fuite** :
- Utilisation mémoire qui augmente constamment
- Ne redescend jamais
- OOMKilled après un certain temps de fonctionnement

**Diagnostic** :
```bash
# Surveiller l'utilisation dans le temps
watch -n 5 microk8s kubectl top pod mon-pod
```

**Solution** : Corriger le code de l'application (hors scope Kubernetes).

#### Solution 3 : Redémarrer Périodiquement

Si vous ne pouvez pas corriger la fuite immédiatement, redémarrez périodiquement :

```yaml
# Ajouter une livenessProbe qui force le redémarrage
livenessProbe:
  httpGet:
    path: /health
    port: 8080
  initialDelaySeconds: 300
  periodSeconds: 60
  timeoutSeconds: 5
  failureThreshold: 3
```

**Ou utiliser un CronJob pour redémarrer** :
```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: restart-leaky-app
spec:
  schedule: "0 2 * * *"  # Tous les jours à 2h
  jobTemplate:
    spec:
      template:
        spec:
          serviceAccountName: pod-restarter
          containers:
          - name: kubectl
            image: bitnami/kubectl:latest
            command:
            - /bin/sh
            - -c
            - kubectl rollout restart deployment/mon-app
          restartPolicy: Never
```

#### Solution 4 : Optimiser l'Application

**Conseils généraux** :
- Utiliser des pools de connexions
- Limiter le cache en mémoire
- Utiliser Redis/Memcached pour cache externe
- Nettoyer les objets non utilisés
- Utiliser des profilers pour identifier les gourmands en mémoire

## Problème 3 : Performance Lente (CPU Throttling)

### Symptômes

- Application anormalement lente
- Timeouts fréquents
- Requêtes qui prennent beaucoup de temps

### Diagnostic

#### Étape 1 : Vérifier l'Utilisation CPU

```bash
# Utilisation actuelle
microk8s kubectl top pod mon-pod

# Sortie :
# NAME      CPU(cores)   MEMORY(bytes)
# mon-pod   490m         256Mi
```

#### Étape 2 : Comparer avec les Limits

```bash
# Voir les limits CPU
microk8s kubectl get pod mon-pod -o jsonpath='{.spec.containers[*].resources.limits.cpu}'
# Sortie : 500m
```

**Si l'utilisation est proche ou égale à la limit** : Le pod est probablement throttled (bridé).

#### Étape 3 : Vérifier le Throttling

Sur le nœud, vous pouvez voir les statistiques de throttling :

```bash
# Trouver le container ID
CONTAINER_ID=$(microk8s kubectl get pod mon-pod -o jsonpath='{.status.containerStatuses[0].containerID}' | cut -d'/' -f3)

# Voir les stats de throttling
cat /sys/fs/cgroup/cpu/kubepods.slice/*/cri-containerd-${CONTAINER_ID}.scope/cpu.stat

# Chercher :
# nr_throttled: nombre de fois throttlé
# throttled_time: temps total de throttling
```

#### Étape 4 : Profiler l'Application

**Indices de throttling** :
- Haute utilisation CPU constante
- Performance qui varie beaucoup
- Pics de latence réguliers

### Solutions Courantes

#### Solution 1 : Augmenter la Limite CPU

```yaml
# Avant
resources:
  limits:
    cpu: 500m

# Après
resources:
  limits:
    cpu: 1000m    # 1 core
```

#### Solution 2 : Supprimer la Limite CPU

Pour certaines applications, supprimer la limite peut être approprié :

```yaml
resources:
  requests:
    cpu: 250m     # Gardez les requests
  limits:
    # Pas de limite CPU
    memory: 512Mi # Gardez la limite mémoire !
```

**Attention** : Le pod pourra utiliser plus de CPU si disponible, mais partagera équitablement avec les autres.

#### Solution 3 : Optimiser l'Application

- Utiliser des caches
- Optimiser les requêtes base de données
- Utiliser des connexions async/non-bloquantes
- Paralléliser les tâches
- Profiler et optimiser le code

#### Solution 4 : Augmenter le Nombre de Réplicas

Au lieu d'un pod puissant, plusieurs pods moins puissants :

```yaml
# Avant
replicas: 1
resources:
  requests:
    cpu: 1000m

# Après
replicas: 4
resources:
  requests:
    cpu: 250m     # Charge répartie sur 4 pods
```

## Problème 4 : Pods Évictés (Evicted)

### Symptômes

Des pods sont expulsés du nœud et apparaissent avec le statut `Evicted`.

```bash
microk8s kubectl get pods
# NAME          READY   STATUS    RESTARTS   AGE
# mon-pod-abc   0/1     Evicted   0          10m
```

### Diagnostic

#### Étape 1 : Voir la Raison de l'Éviction

```bash
microk8s kubectl describe pod mon-pod-abc

# Chercher :
# Reason:  Evicted
# Message: The node was low on resource: memory
# ou
# Message: The node had condition: [DiskPressure]
```

**Raisons courantes** :
- **Memory Pressure** : Mémoire du nœud faible
- **Disk Pressure** : Espace disque faible
- **PID Pressure** : Trop de processus

#### Étape 2 : Vérifier l'État du Nœud

```bash
# État du nœud
microk8s kubectl describe node | grep -A 5 "Conditions:"

# Chercher :
# MemoryPressure   False
# DiskPressure     False
# PIDPressure      False
```

**Si une condition est True** : Le nœud est sous pression.

#### Étape 3 : Vérifier les Ressources du Nœud

```bash
# Utilisation globale
microk8s kubectl top node

# Sur le nœud directement
free -h    # Mémoire
df -h      # Disque
```

### Solutions Courantes

#### Solution 1 : Nettoyer les Pods Évictés

Ils ne servent plus à rien :

```bash
# Supprimer tous les pods évictés
microk8s kubectl get pods -A | grep Evicted | awk '{print $2 " -n " $1}' | xargs -L1 microk8s kubectl delete pod
```

#### Solution 2 : Libérer de la Mémoire

```bash
# Identifier les pods gourmands
microk8s kubectl top pods --all-namespaces --sort-by=memory

# Réduire l'échelle ou supprimer
microk8s kubectl scale deployment app-gourmande --replicas=1
```

#### Solution 3 : Libérer de l'Espace Disque

```bash
# Nettoyer les images
microk8s ctr images rm $(microk8s ctr images ls -q)

# Nettoyer les conteneurs arrêtés
microk8s crictl rmi --prune

# Nettoyer les logs
sudo find /var/snap/microk8s/common/var/log -type f -name "*.log" -mtime +7 -delete
```

#### Solution 4 : Définir des Requests/Limits Appropriées

Pour éviter les évictions futures :

```yaml
resources:
  requests:
    memory: 256Mi     # Réaliste
  limits:
    memory: 512Mi     # Pas trop haut
```

#### Solution 5 : Augmenter les Ressources du Nœud

- Ajouter de la RAM
- Ajouter de l'espace disque
- Ajouter un nouveau nœud

## Quality of Service (QoS) Classes

Kubernetes attribue automatiquement une QoS class à chaque pod selon ses requests/limits.

### Les Trois Classes

#### 1. Guaranteed (Garanti)

**Définition** : Meilleure garantie, derniers à être évictés.

**Conditions** :
- Chaque conteneur a requests ET limits
- requests = limits pour CPU et mémoire

**Exemple** :
```yaml
resources:
  requests:
    cpu: 500m
    memory: 512Mi
  limits:
    cpu: 500m      # = requests
    memory: 512Mi  # = requests
```

**Utilisation** : Applications critiques (bases de données, services essentiels)

#### 2. Burstable (Rafale)

**Définition** : Garantie partielle.

**Conditions** :
- Au moins un conteneur a requests OU limits
- requests ≠ limits

**Exemple** :
```yaml
resources:
  requests:
    cpu: 250m
    memory: 256Mi
  limits:
    cpu: 1000m     # > requests
    memory: 1Gi    # > requests
```

**Utilisation** : La plupart des applications

#### 3. BestEffort (Meilleur Effort)

**Définition** : Aucune garantie, premiers à être évictés.

**Conditions** :
- Aucun requests ni limits définis

**Exemple** :
```yaml
# Pas de resources du tout
containers:
- name: mon-app
  image: nginx
```

**Utilisation** : Jobs non critiques, tâches batch

### Ordre d'Éviction

En cas de pression sur les ressources :

```
1. BestEffort   (évictés en premier)
2. Burstable    (évictés ensuite)
3. Guaranteed   (évictés en dernier recours)
```

### Vérifier la QoS d'un Pod

```bash
microk8s kubectl get pod mon-pod -o jsonpath='{.status.qosClass}'
# Sortie : Burstable, Guaranteed, ou BestEffort
```

## Outils de Monitoring et Diagnostic

### 1. kubectl top

**Installation du Metrics Server** (si pas déjà fait) :
```bash
microk8s enable metrics-server

# Attendre qu'il soit prêt
microk8s kubectl wait --for=condition=ready pod -n kube-system -l k8s-app=metrics-server --timeout=60s
```

**Utilisation** :

```bash
# Utilisation des nœuds
microk8s kubectl top nodes

# Utilisation des pods
microk8s kubectl top pods

# Tous les namespaces
microk8s kubectl top pods --all-namespaces

# Trier par utilisation
microk8s kubectl top pods --sort-by=cpu
microk8s kubectl top pods --sort-by=memory

# Conteneurs individuels
microk8s kubectl top pods --containers
```

### 2. kubectl describe

```bash
# Vue complète d'un nœud
microk8s kubectl describe node

# Vue complète d'un pod
microk8s kubectl describe pod mon-pod
```

**Sections importantes** :
- **Conditions** : État du nœud
- **Capacity/Allocatable** : Ressources disponibles
- **Allocated resources** : Ressources utilisées
- **Events** : Évictions, OOM, etc.

### 3. Prometheus + Grafana

Pour un monitoring avancé :

```bash
# Activer Prometheus (si pas déjà fait)
microk8s enable prometheus

# Accéder à Grafana
microk8s kubectl port-forward -n prometheus service/prometheus-grafana 3000:80

# Ouvrir : http://localhost:3000
# Login par défaut : admin / prom-operator
```

**Métriques disponibles** :
- Utilisation CPU/mémoire dans le temps
- Historique des performances
- Alertes sur seuils
- Dashboards pré-configurés

### 4. Commandes Système (sur le Nœud)

```bash
# Vue générale
htop

# Utilisation CPU/mémoire
top

# Statistiques I/O
iostat -x 5

# Processus qui consomment le plus
ps aux --sort=-%mem | head -10
ps aux --sort=-%cpu | head -10
```

## Optimisation des Ressources

### Stratégie 1 : Right-Sizing

**Processus** :

1. **Commencer conservateur** :
```yaml
resources:
  requests:
    cpu: 100m
    memory: 128Mi
  limits:
    cpu: 500m
    memory: 512Mi
```

2. **Surveiller pendant quelques jours** :
```bash
watch -n 60 microk8s kubectl top pod mon-pod
```

3. **Ajuster selon l'utilisation réelle** :

```yaml
# Si utilisation moyenne : CPU 250m, Mémoire 300Mi
resources:
  requests:
    cpu: 250m        # Basé sur moyenne
    memory: 300Mi
  limits:
    cpu: 500m        # 2x la moyenne pour les pics
    memory: 600Mi    # 2x la moyenne
```

### Stratégie 2 : Horizontal Pod Autoscaling (HPA)

**Principe** : Ajuster automatiquement le nombre de réplicas selon la charge.

**Configuration** :

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: mon-app-hpa
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
        averageUtilization: 70  # Scaler si CPU > 70%
```

**Créer via commande** :
```bash
microk8s kubectl autoscale deployment mon-app --cpu-percent=70 --min=2 --max=10
```

**Vérifier** :
```bash
microk8s kubectl get hpa
```

### Stratégie 3 : Vertical Pod Autoscaling (VPA)

**Principe** : Ajuster automatiquement les requests/limits selon l'utilisation.

**Installation** :
```bash
# VPA n'est pas inclus dans MicroK8s par défaut
# Installation manuelle nécessaire
```

**Configuration** :
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
    updateMode: "Auto"  # ou "Initial", "Off"
```

### Stratégie 4 : Resource Quotas

**Principe** : Limiter les ressources totales d'un namespace.

**Configuration** :
```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: dev-quota
  namespace: development
spec:
  hard:
    requests.cpu: "4"       # Total requests CPU
    requests.memory: 8Gi    # Total requests mémoire
    limits.cpu: "8"         # Total limits CPU
    limits.memory: 16Gi     # Total limits mémoire
    pods: "20"              # Nombre max de pods
```

**Appliquer** :
```bash
microk8s kubectl apply -f quota.yaml

# Vérifier
microk8s kubectl describe quota -n development
```

### Stratégie 5 : LimitRange

**Principe** : Définir des valeurs par défaut et des contraintes par pod.

**Configuration** :
```yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: limit-range
  namespace: development
spec:
  limits:
  - max:
      cpu: "2"          # Un pod ne peut pas demander plus
      memory: 2Gi
    min:
      cpu: 100m         # Un pod doit demander au moins
      memory: 128Mi
    default:
      cpu: 500m         # Valeur par défaut si non spécifié
      memory: 512Mi
    defaultRequest:
      cpu: 250m
      memory: 256Mi
    type: Container
```

**Avantage** : Les développeurs n'ont plus besoin de spécifier les ressources (valeurs par défaut appliquées).

## Workflows de Dépannage

### Workflow 1 : Pod Pending (Ressources Insuffisantes)

```bash
# 1. Vérifier le problème
microk8s kubectl describe pod mon-pod | grep -A 5 Events

# 2. Voir les ressources demandées
microk8s kubectl get pod mon-pod -o jsonpath='{.spec.containers[*].resources}'

# 3. Voir les ressources disponibles
microk8s kubectl describe node | grep -A 5 "Allocated resources"

# 4. Solutions possibles :

# Option A : Réduire les requests
microk8s kubectl edit deployment mon-app
# Modifier resources.requests

# Option B : Libérer des ressources
microk8s kubectl scale deployment app-non-critique --replicas=0

# Option C : Ajouter un nœud
microk8s add-node
```

### Workflow 2 : OOMKilled

```bash
# 1. Confirmer le problème
microk8s kubectl describe pod mon-pod | grep OOM

# 2. Voir la limite actuelle
microk8s kubectl get pod mon-pod -o jsonpath='{.spec.containers[*].resources.limits.memory}'

# 3. Voir l'utilisation (si pod en cours)
microk8s kubectl top pod mon-pod

# 4. Augmenter la limite
microk8s kubectl edit deployment mon-app
# Modifier resources.limits.memory

# 5. Vérifier
microk8s kubectl rollout status deployment mon-app
microk8s kubectl get pods
```

### Workflow 3 : Performance Lente

```bash
# 1. Vérifier l'utilisation CPU
microk8s kubectl top pod mon-pod

# 2. Comparer avec les limits
microk8s kubectl get pod mon-pod -o jsonpath='{.spec.containers[*].resources.limits.cpu}'

# 3. Si throttlé, augmenter
microk8s kubectl edit deployment mon-app
# Augmenter resources.limits.cpu

# 4. Ou ajouter des réplicas
microk8s kubectl scale deployment mon-app --replicas=3

# 5. Surveiller
watch -n 5 microk8s kubectl top pods -l app=mon-app
```

### Workflow 4 : Optimisation Globale

```bash
# 1. Audit des ressources actuelles
microk8s kubectl top pods --all-namespaces

# 2. Identifier les pods sur/sous-dimensionnés
# Sur-dimensionnés : limits >> utilisation
# Sous-dimensionnés : utilisation proche des limits

# 3. Ajuster les configurations
# Pour chaque deployment problématique :
microk8s kubectl edit deployment <nom>

# 4. Mettre en place HPA pour les apps variables
microk8s kubectl autoscale deployment mon-app --cpu-percent=70 --min=2 --max=10

# 5. Définir des quotas pour les namespaces
kubectl apply -f resource-quota.yaml
```

## Bonnes Pratiques

### 1. Toujours Définir Requests et Limits

❌ **Mauvais** :
```yaml
containers:
- name: mon-app
  image: nginx
  # Pas de resources !
```

✅ **Bon** :
```yaml
containers:
- name: mon-app
  image: nginx
  resources:
    requests:
      cpu: 100m
      memory: 128Mi
    limits:
      cpu: 500m
      memory: 512Mi
```

### 2. Requests Basés sur l'Utilisation Réelle

- Démarrer conservateur
- Surveiller l'utilisation réelle
- Ajuster après observation

### 3. Limits Appropriées

**Pour CPU** :
- Limits pas trop strictes (permet les burst)
- Ou pas de limite du tout si charge variable

**Pour Mémoire** :
- TOUJOURS définir une limite
- 2-3x la moyenne d'utilisation
- Protège contre les fuites mémoire

### 4. Utiliser QoS Guaranteed pour les Services Critiques

```yaml
# Base de données, cache, etc.
resources:
  requests:
    cpu: 1000m
    memory: 2Gi
  limits:
    cpu: 1000m      # = requests
    memory: 2Gi     # = requests
```

### 5. Implémenter HPA pour les Charges Variables

```bash
# Applications web, API, workers
microk8s kubectl autoscale deployment web-app --cpu-percent=70 --min=2 --max=10
```

### 6. Utiliser Resource Quotas par Équipe/Projet

```yaml
# Namespace par équipe avec quotas
apiVersion: v1
kind: ResourceQuota
metadata:
  name: team-a-quota
  namespace: team-a
spec:
  hard:
    requests.cpu: "10"
    requests.memory: 20Gi
    limits.cpu: "20"
    limits.memory: 40Gi
```

### 7. Surveiller Régulièrement

```bash
# Dashboard quotidien
microk8s kubectl top nodes
microk8s kubectl top pods --all-namespaces --sort-by=memory | head -20

# Alertes Prometheus/Grafana
# - CPU > 80%
# - Mémoire > 85%
# - Pods évictés
# - Nœuds sous pression
```

### 8. Documenter les Décisions

```yaml
# Commentaires dans les YAML
resources:
  requests:
    cpu: 250m      # Basé sur moyenne 200m + 25% marge (testé sur 2 semaines)
    memory: 512Mi  # Utilisation max vue : 400Mi
  limits:
    cpu: 1000m     # Permet burst pour pics de trafic
    memory: 1Gi    # 2x requests pour sécurité
```

## Checklist Performance et Ressources

### Pour Chaque Deployment

- [ ] Requests CPU définis ?
- [ ] Requests mémoire définis ?
- [ ] Limits mémoire définis ? (OBLIGATOIRE)
- [ ] Limits CPU définis ? (recommandé)
- [ ] Requests basés sur utilisation réelle ?
- [ ] QoS class appropriée ?
- [ ] HPA configuré si charge variable ?
- [ ] Readiness/Liveness probes configurées ?

### Pour le Cluster

- [ ] Metrics Server activé ?
- [ ] Utilisation nœud < 80% ?
- [ ] Quotas définis par namespace ?
- [ ] LimitRange configuré ?
- [ ] Monitoring en place (Prometheus/Grafana) ?
- [ ] Alertes configurées ?
- [ ] Plan de scaling documenté ?

### Diagnostics Réguliers

- [ ] `kubectl top nodes` - Nœuds OK ?
- [ ] `kubectl top pods --all-namespaces` - Pods normaux ?
- [ ] `kubectl get pods -A | grep -E 'Evicted|OOMKilled'` - Problèmes ?
- [ ] `kubectl describe nodes | grep Pressure` - Pressions ?

## Résumé

### Commandes Essentielles

```bash
# Monitoring
kubectl top nodes
kubectl top pods --all-namespaces
kubectl describe node

# Diagnostic
kubectl describe pod <pod>
kubectl get pod <pod> -o jsonpath='{.spec.containers[*].resources}'
kubectl get pod <pod> -o jsonpath='{.status.qosClass}'

# Actions
kubectl edit deployment <deploy>
kubectl scale deployment <deploy> --replicas=<n>
kubectl autoscale deployment <deploy> --cpu-percent=70 --min=2 --max=10

# Nettoyage
kubectl delete pod -A --field-selector status.phase=Failed
kubectl get pods -A | grep Evicted | awk '{print $2 " -n " $1}' | xargs -L1 kubectl delete pod
```

### Concepts Clés

| Concept | Définition | Impact |
|---------|------------|--------|
| **Requests** | Ressources garanties | Utilisé par le scheduler |
| **Limits** | Ressources maximales | CPU throttled, Mémoire OOMKilled |
| **QoS Guaranteed** | requests = limits | Derniers évictés |
| **QoS Burstable** | requests ≠ limits | Évictés moyennement |
| **QoS BestEffort** | Pas de resources | Premiers évictés |
| **Throttling** | Limitation CPU | Performance dégradée |
| **OOMKilled** | Dépassement mémoire | Pod tué |
| **Evicted** | Expulsion | Ressources nœud faibles |

### Valeurs Typiques

| Type d'Application | CPU Request | CPU Limit | Memory Request | Memory Limit |
|-------------------|-------------|-----------|----------------|--------------|
| Service web léger | 100m | 500m | 128Mi | 512Mi |
| API backend | 250m | 1000m | 256Mi | 1Gi |
| Worker/Job | 500m | 2000m | 512Mi | 2Gi |
| Base de données | 1000m | 1000m | 1Gi | 1Gi |
| Cache (Redis) | 100m | 500m | 512Mi | 512Mi |

**Note** : Ces valeurs sont indicatives, ajustez selon vos besoins réels !

### Points Clés à Retenir

1. **TOUJOURS définir resources** en production
2. **Requests** = ce dont vous avez besoin minimum
3. **Limits** = protection contre sur-utilisation
4. **Mémoire** : limite OBLIGATOIRE (risque OOMKilled)
5. **CPU** : limite optionnelle (risque throttling)
6. **Surveiller** pour ajuster les valeurs
7. **HPA** pour les charges variables
8. **QoS Guaranteed** pour les services critiques

**Prochaine étape** : Section 23.9 - Mise à jour de MicroK8s

---


⏭️ [Mise à jour de MicroK8s](/23-depannage-et-maintenance/09-mise-a-jour-de-microk8s.md)
