🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 19.5 Metrics Server

## Introduction au Metrics Server

Le **Metrics Server** est un composant essentiel de Kubernetes qui collecte les métriques de consommation de ressources (CPU et mémoire) de tous les nœuds et pods de votre cluster. C'est la **fondation** sur laquelle reposent le HPA, le VPA, et la commande `kubectl top`.

### Analogie simple

Imaginez un immeuble de bureaux :

**Sans Metrics Server :**
- Vous savez combien de bureaux vous avez (nœuds)
- Vous savez combien d'employés travaillent (pods)
- Mais vous ne savez **pas** combien chaque employé consomme d'électricité ou d'espace

**Avec Metrics Server :**
- Un compteur intelligent mesure en temps réel la consommation de chaque bureau
- Vous pouvez voir qui consomme beaucoup ou peu
- Vous pouvez prendre des décisions intelligentes (ajouter/retirer des bureaux)

Le Metrics Server est ce "compteur intelligent" pour votre cluster Kubernetes !

## Pourquoi le Metrics Server est essentiel ?

### Sans Metrics Server

Voici ce qui **ne fonctionne pas** sans Metrics Server :

```bash
# Cette commande échoue
kubectl top nodes
Error from server (ServiceUnavailable): the server is currently unable to handle the request

# Cette commande échoue aussi
kubectl top pods
Error from server (ServiceUnavailable): the server is currently unable to handle the request

# Le HPA ne peut pas fonctionner
kubectl get hpa
NAME        REFERENCE          TARGETS         MINPODS   MAXPODS   REPLICAS
my-app-hpa  Deployment/my-app  <unknown>/50%   1         10        1

# Le VPA n'a pas de données
kubectl describe vpa my-app-vpa
Status:
  Conditions:
    Message: Waiting for metrics...
```

**Résultat :** Vous ne pouvez pas utiliser l'autoscaling automatique !

### Avec Metrics Server

Une fois le Metrics Server activé :

✅ `kubectl top nodes` fonctionne (voir l'utilisation CPU/mémoire par nœud)
✅ `kubectl top pods` fonctionne (voir l'utilisation CPU/mémoire par pod)
✅ HPA peut prendre des décisions basées sur les métriques réelles
✅ VPA peut calculer des recommandations précises
✅ Dashboard Kubernetes affiche les graphiques de ressources
✅ Vous pouvez surveiller votre cluster efficacement

## Architecture du Metrics Server

Le Metrics Server suit une architecture simple :

```
┌─────────────────────────────────────────────────────┐
│                    API Server                       │
│  (kubectl top, HPA, VPA consultent les métriques)   │
└─────────────────────────────────────────────────────┘
                         ↑
                         │ Expose les métriques via
                         │ l'API Metrics (metrics.k8s.io)
                         │
┌─────────────────────────────────────────────────────┐
│                  Metrics Server                     │
│  (Collecte et agrège les métriques)                 │
└─────────────────────────────────────────────────────┘
                         ↑
                         │ Interroge toutes les 60s
                         │
┌─────────────────────────────────────────────────────┐
│                     Kubelet                         │
│  (Agent sur chaque nœud, expose les métriques       │
│   via cAdvisor - Container Advisor)                 │
└─────────────────────────────────────────────────────┘
                         ↑
                         │ Lit les statistiques
                         │
┌─────────────────────────────────────────────────────┐
│              Conteneurs (Pods)                      │
│  (Runtime collecte CPU/RAM utilisés)                │
└─────────────────────────────────────────────────────┘
```

### Composants expliqués

**1. Conteneurs et Runtime**
- Chaque conteneur consomme du CPU et de la mémoire
- Le runtime (containerd, CRI-O) collecte ces statistiques via cgroups

**2. Kubelet et cAdvisor**
- Le Kubelet (agent K8s sur chaque nœud) intègre cAdvisor
- cAdvisor collecte les métriques des conteneurs
- Expose ces métriques via une API HTTP sur le Kubelet

**3. Metrics Server**
- Interroge tous les Kubelets toutes les **60 secondes** (par défaut)
- Collecte les métriques de tous les nœuds et pods
- Agrège et stocke temporairement ces données (en mémoire uniquement)
- Calcule des moyennes mobiles

**4. Metrics API**
- Le Metrics Server expose une API standard (`metrics.k8s.io`)
- Accessible via l'API Server de Kubernetes
- Permet à `kubectl top`, HPA, VPA de consulter les métriques

### Caractéristiques importantes

**Stockage en mémoire uniquement :**
- Les métriques ne sont **pas** persistées sur disque
- Conservées pour une courte durée (~5-15 minutes)
- Suffisant pour l'autoscaling, mais pas pour l'historique long terme

**Fréquence de collecte :**
- Collecte toutes les 60 secondes par défaut
- Les métriques ont donc un léger retard (~1 minute)
- Suffisant pour l'autoscaling (pas besoin de temps réel strict)

**Métriques limitées :**
- Uniquement CPU et mémoire
- Pas de métriques réseau, disque, ou custom
- Pour des métriques avancées, utilisez Prometheus (section 12)

## Installation sur MicroK8s

Bonne nouvelle ! Sur MicroK8s, l'installation du Metrics Server est **extrêmement simple** grâce à l'addon intégré.

### Méthode simple : Addon MicroK8s

**Étape 1 : Activer l'addon**

```bash
microk8s enable metrics-server
```

Sortie :
```
Infer repository core for addon metrics-server
Enabling Metrics-Server
serviceaccount/metrics-server created
clusterrole.rbac.authorization.k8s.io/system:aggregated-metrics-reader created
clusterrole.rbac.authorization.k8s.io/system:metrics-server created
rolebinding.rbac.authorization.k8s.io/metrics-server-auth-reader created
clusterrolebinding.rbac.authorization.k8s.io/metrics-server:system:auth-delegator created
clusterrolebinding.rbac.authorization.k8s.io/system:metrics-server created
service/metrics-server created
deployment.apps/metrics-server created
apiservice.apiregistration.k8s.io/v1beta1.metrics.k8s.io created
Metrics-Server is enabled
```

**C'est tout !** Le Metrics Server est installé et fonctionnel.

**Étape 2 : Vérifier l'installation**

```bash
kubectl get pods -n kube-system | grep metrics-server
```

Sortie attendue :
```
metrics-server-xxxxxxxxx-xxxxx   1/1     Running   0          1m
```

Le pod doit être en statut `Running`.

**Étape 3 : Vérifier l'API Metrics**

```bash
kubectl get apiservices | grep metrics
```

Sortie :
```
v1beta1.metrics.k8s.io   kube-system/metrics-server   True   2m
```

Le statut doit être `True`.

### Attendre la collecte des premières métriques

Le Metrics Server a besoin de quelques minutes pour collecter les premières données.

**Attendez 2-3 minutes**, puis testez :

```bash
kubectl top nodes
```

Si vous obtenez une erreur, attendez encore 1-2 minutes. Si l'erreur persiste après 5 minutes, voir la section "Dépannage".

## Utilisation du Metrics Server

### Commande kubectl top nodes

Affiche l'utilisation des ressources **par nœud** :

```bash
kubectl top nodes
```

Sortie exemple :
```
NAME          CPU(cores)   CPU%   MEMORY(bytes)   MEMORY%
microk8s-vm   450m         22%    2048Mi          51%
```

**Interprétation :**

- **CPU(cores)** : 450m = 450 millicores = 0.45 CPU
- **CPU%** : 22% de la capacité totale du nœud
- **MEMORY(bytes)** : 2048Mi = ~2 GB de RAM utilisée
- **MEMORY%** : 51% de la mémoire totale du nœud

### Commande kubectl top pods

Affiche l'utilisation des ressources **par pod** :

```bash
kubectl top pods
```

Sortie exemple :
```
NAME                        CPU(cores)   MEMORY(bytes)
nginx-web-7d8c9f5b6d-abcd   5m           10Mi
nginx-web-7d8c9f5b6d-efgh   3m           8Mi
postgres-0                  25m          150Mi
```

**Interprétation :**

- **nginx-web-abcd** : Utilise 5 millicores CPU et 10 Mi de RAM
- **postgres-0** : Utilise 25 millicores CPU et 150 Mi de RAM

### Options utiles de kubectl top

**Voir tous les namespaces :**
```bash
kubectl top pods --all-namespaces
```

**Trier par CPU :**
```bash
kubectl top pods --sort-by=cpu
```

Sortie (pods triés du plus consommateur au moins) :
```
NAME                        CPU(cores)   MEMORY(bytes)
postgres-0                  25m          150Mi
nginx-web-7d8c9f5b6d-abcd   5m           10Mi
nginx-web-7d8c9f5b6d-efgh   3m           8Mi
```

**Trier par mémoire :**
```bash
kubectl top pods --sort-by=memory
```

**Filtrer par label :**
```bash
kubectl top pods -l app=nginx
```

**Voir les conteneurs individuellement :**
```bash
kubectl top pods --containers
```

Sortie :
```
POD                         NAME       CPU(cores)   MEMORY(bytes)
nginx-web-7d8c9f5b6d-abcd   nginx      5m           10Mi
nginx-web-7d8c9f5b6d-abcd   sidecar    2m           5Mi
```

Utile quand un pod a plusieurs conteneurs.

### Surveiller en temps réel

**Commande avec rafraîchissement automatique :**

```bash
watch -n 2 kubectl top pods
```

Cela rafraîchit l'affichage toutes les 2 secondes.

**Alternative avec kubectl :**

```bash
kubectl top pods --watch
```

Malheureusement, `--watch` n'existe pas pour `kubectl top`, utilisez donc `watch`.

## Utilisation par le HPA

Le HPA interroge le Metrics Server pour obtenir les métriques de CPU et mémoire.

### Exemple : HPA basé sur CPU

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: nginx-hpa
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

**Flux de décision du HPA :**

1. HPA interroge le Metrics Server : "Quelle est l'utilisation CPU moyenne des pods nginx-web ?"
2. Metrics Server répond : "Utilisation moyenne : 75%"
3. HPA calcule : "Cible = 50%, Actuel = 75%, donc scale up nécessaire"
4. HPA augmente le nombre de répliques

**Visualiser ce qui se passe :**

```bash
kubectl get hpa nginx-hpa
```

Sortie :
```
NAME        REFERENCE              TARGETS   MINPODS   MAXPODS   REPLICAS   AGE
nginx-hpa   Deployment/nginx-web   75%/50%   1         10        3          5m
```

**TARGETS** montre `75%/50%` :
- **75%** : Utilisation actuelle (données du Metrics Server)
- **50%** : Cible configurée

### Exemple : HPA basé sur mémoire

```yaml
metrics:
- type: Resource
  resource:
    name: memory
    target:
      type: Utilization
      averageUtilization: 70
```

Le HPA interroge le Metrics Server pour l'utilisation mémoire.

### Si Metrics Server est absent

Si vous créez un HPA sans Metrics Server :

```bash
kubectl get hpa
```

Sortie :
```
NAME        REFERENCE              TARGETS         MINPODS   MAXPODS   REPLICAS
nginx-hpa   Deployment/nginx-web   <unknown>/50%   1         10        1
```

**TARGETS** affiche `<unknown>/50%` car le HPA ne peut pas obtenir les métriques.

Le HPA ne peut pas fonctionner et ne scale pas !

## Utilisation par le VPA

Le VPA utilise aussi le Metrics Server, mais de manière différente.

### Collecte d'historique

Le VPA Recommender collecte les métriques en continu pour construire un historique :

```
Jour 1 : Pod utilise 100m CPU, 200Mi RAM
Jour 2 : Pod utilise 150m CPU, 250Mi RAM
Jour 3 : Pod utilise 120m CPU, 220Mi RAM
...
Jour 8 : Moyenne calculée : ~130m CPU, ~230Mi RAM
```

Le VPA recommande alors des `requests` basés sur ces patterns.

### Recommandations basées sur les percentiles

Le VPA ne prend pas simplement la moyenne, mais utilise des **percentiles** :

- **Lower Bound** : Basé sur le minimum observé (percentile 5-10%)
- **Target** : Basé sur un percentile élevé (90-95%)
- **Upper Bound** : Basé sur le maximum observé (percentile 99%)

**Pourquoi les percentiles ?**

Imaginez ces mesures :
```
Utilisation CPU : 100m, 120m, 110m, 800m, 115m, 105m, 125m
```

- **Moyenne** : ~200m (faussée par le pic de 800m)
- **Percentile 95%** : ~125m (ignore les valeurs extrêmes)

Le percentile 95% donne une meilleure recommandation !

## Métriques disponibles

Le Metrics Server fournit **uniquement** deux types de métriques :

### 1. Métriques CPU

**Unités :**
- **Millicores (m)** : 1000m = 1 CPU core
- Exemples : 100m, 500m, 1500m

**Ce qui est mesuré :**
- Temps CPU réel consommé par les conteneurs
- Agrégé sur la fenêtre de collecte (dernière minute)

**Calcul du pourcentage :**
```
% CPU = (CPU utilisé / CPU request) × 100
```

Exemple :
- Request : 200m
- Utilisation : 150m
- Pourcentage : (150/200) × 100 = 75%

### 2. Métriques Mémoire

**Unités :**
- **Bytes** : octets
- **Ki, Mi, Gi** : Kibibytes, Mebibytes, Gibibytes
- Exemples : 128Mi, 1Gi, 500Mi

**Ce qui est mesuré :**
- Mémoire résidente (RSS - Resident Set Size)
- Mémoire réellement utilisée par le processus
- **Ne compte pas** le cache du système d'exploitation

**Calcul du pourcentage :**
```
% Mémoire = (Mémoire utilisée / Mémoire request) × 100
```

Exemple :
- Request : 256Mi
- Utilisation : 200Mi
- Pourcentage : (200/256) × 100 = 78%

### Ce que le Metrics Server ne fournit PAS

❌ Métriques réseau (bande passante, paquets)
❌ Métriques disque (IOPS, latence, espace)
❌ Métriques custom (requêtes/sec, latence HTTP, etc.)
❌ Historique long terme (pas de persistence)
❌ Métriques applicatives (threads, connexions DB, etc.)

**Pour ces métriques, utilisez :**
- **Prometheus** (voir section 12) : Métriques avancées et historique
- **Custom Metrics API** : Pour le HPA avec métriques custom (voir section 19.6)

## Comprendre les requests et limits

Le Metrics Server mesure l'utilisation réelle, mais le HPA calcule les pourcentages basés sur les **requests**.

### Importance des requests

**Sans requests définis :**

```yaml
containers:
- name: nginx
  image: nginx:latest
  # Pas de resources !
```

Le HPA ne peut pas calculer de pourcentage :
```
Utilisation actuelle : 100m
Request : non défini
Pourcentage : impossible à calculer !
```

**Avec requests définis :**

```yaml
containers:
- name: nginx
  image: nginx:latest
  resources:
    requests:
      cpu: "200m"
      memory: "128Mi"
```

Le HPA peut calculer :
```
Utilisation actuelle : 100m
Request : 200m
Pourcentage : 50%
```

### Recommandation importante

**Toujours définir les requests** pour tous les conteneurs si vous utilisez :
- HPA
- VPA
- Scheduling basé sur les ressources

```yaml
resources:
  requests:
    cpu: "100m"      # Minimum garanti
    memory: "128Mi"
  limits:
    cpu: "500m"      # Maximum autorisé
    memory: "512Mi"
```

## Configuration avancée du Metrics Server

### Paramètres de configuration

Sur MicroK8s, le Metrics Server est déjà bien configuré, mais vous pouvez modifier certains paramètres si nécessaire.

**Voir la configuration actuelle :**

```bash
kubectl get deployment metrics-server -n kube-system -o yaml
```

**Paramètres intéressants :**

```yaml
spec:
  containers:
  - name: metrics-server
    args:
    - --cert-dir=/tmp
    - --secure-port=4443
    - --kubelet-preferred-address-types=InternalIP,ExternalIP,Hostname
    - --kubelet-use-node-status-port
    - --metric-resolution=60s        # Fréquence de collecte
    - --kubelet-insecure-tls         # Pour environnements de dev
```

### Fréquence de collecte

**Par défaut :** 60 secondes

Pour collecter plus fréquemment (utilise plus de ressources) :

```yaml
args:
- --metric-resolution=15s    # Collecte toutes les 15 secondes
```

**Note :** Rarement nécessaire. 60 secondes est suffisant pour l'autoscaling.

### Ressources du Metrics Server

Le Metrics Server lui-même consomme des ressources :

```yaml
resources:
  requests:
    cpu: 100m
    memory: 200Mi
  limits:
    cpu: 100m
    memory: 200Mi
```

**Impact typique :**
- CPU : ~50-100m pour un cluster de 10-20 nœuds
- Mémoire : ~100-300Mi selon le nombre de pods

Sur un petit cluster MicroK8s (1-3 nœuds, <50 pods), l'impact est négligeable.

## Comparaison : Metrics Server vs Prometheus

Beaucoup de débutants confondent Metrics Server et Prometheus. Voici les différences :

| Aspect | Metrics Server | Prometheus |
|--------|---------------|------------|
| **Objectif** | Autoscaling et `kubectl top` | Monitoring complet et alerting |
| **Métriques** | CPU et mémoire uniquement | Toutes métriques possibles |
| **Stockage** | En mémoire (~15 min) | Persistant (jours/mois/années) |
| **Historique** | Non | Oui |
| **Fréquence** | 60 secondes | Configurable (15s typique) |
| **Overhead** | Très faible | Moyen |
| **Custom metrics** | Non | Oui |
| **Visualisation** | `kubectl top` uniquement | Grafana dashboards |
| **Alerting** | Non | Oui (Alertmanager) |
| **Requis pour HPA** | Oui (pour CPU/mémoire) | Non (sauf métriques custom) |
| **Complexité** | Simple | Moyenne-élevée |

**Règle simple :**

- **Metrics Server** : Nécessaire pour l'autoscaling basique
- **Prometheus** : Optionnel mais fortement recommandé pour le monitoring complet

**Ils sont complémentaires, pas exclusifs !**

Dans un cluster de production, vous aurez typiquement **les deux** :
- Metrics Server pour HPA/VPA
- Prometheus pour monitoring, dashboards, et alertes

## Dépannage

### Problème 1 : Metrics Server ne démarre pas

**Symptôme :**

```bash
kubectl get pods -n kube-system | grep metrics-server
metrics-server-xxx   0/1   CrashLoopBackOff   5   5m
```

**Cause possible :** Problème de certificats TLS

**Solution :**

Vérifier les logs :
```bash
kubectl logs -n kube-system deployment/metrics-server
```

Si vous voyez des erreurs de certificat, le Metrics Server sur MicroK8s inclut déjà `--kubelet-insecure-tls` donc cela devrait fonctionner. Si problème persiste :

```bash
microk8s disable metrics-server
microk8s enable metrics-server
```

### Problème 2 : kubectl top retourne "unavailable"

**Symptôme :**

```bash
kubectl top nodes
Error from server (ServiceUnavailable): the server is currently unable to handle the request
```

**Solutions :**

**1. Vérifier que le Metrics Server tourne :**
```bash
kubectl get pods -n kube-system | grep metrics-server
```

Le pod doit être `Running`.

**2. Vérifier l'API Service :**
```bash
kubectl get apiservices | grep metrics
```

Doit afficher `True`.

**3. Attendre 2-3 minutes :**
Le Metrics Server a besoin de collecter les premières métriques.

**4. Vérifier la connectivité au Kubelet :**
```bash
kubectl logs -n kube-system deployment/metrics-server
```

Cherchez des erreurs de connexion au Kubelet.

### Problème 3 : Métriques toujours à 0

**Symptôme :**

```bash
kubectl top pods
NAME        CPU(cores)   MEMORY(bytes)
my-pod      0m           0Mi
```

**Cause :** Les conteneurs ne consomment vraiment rien, ou le pod vient juste de démarrer.

**Solutions :**

- Attendre 1-2 minutes pour que les métriques s'accumulent
- Vérifier que le pod fait effectivement quelque chose
- Relancer la collecte : `kubectl delete pod -n kube-system -l k8s-app=metrics-server`

### Problème 4 : HPA affiche "unknown"

**Symptôme :**

```bash
kubectl get hpa
NAME     REFERENCE        TARGETS         MINPODS   MAXPODS   REPLICAS
my-hpa   Deployment/app   <unknown>/50%   1         10        1
```

**Causes possibles :**

**1. Pas de requests définis dans les pods :**

Vérifiez :
```bash
kubectl get deployment app -o yaml | grep -A 5 resources
```

Si absent, ajoutez :
```yaml
resources:
  requests:
    cpu: "100m"
    memory: "128Mi"
```

**2. Pods pas encore prêts :**

```bash
kubectl get pods -l app=app
```

Tous les pods doivent être `Running` et `Ready`.

**3. Metrics Server pas encore prêt :**

Attendez 2-3 minutes après avoir activé le Metrics Server.

### Problème 5 : Métriques incohérentes

**Symptôme :**

Les métriques fluctuent énormément ou semblent incorrectes.

**Explications :**

**1. Les métriques sont des moyennes mobiles :**
L'utilisation peut varier beaucoup d'une seconde à l'autre. Le Metrics Server lisse ces variations.

**2. Délai de collecte :**
Les métriques ont un retard de ~60 secondes. Ce que vous voyez n'est pas instantané.

**3. Bursts temporaires :**
Un pic de 2 secondes toutes les minutes ne sera pas très visible dans les moyennes.

**Solution :**

Pour des métriques plus détaillées, utilisez Prometheus qui collecte toutes les 15 secondes et conserve l'historique.

## Monitoring du Metrics Server

### Santé du Metrics Server

```bash
# Vérifier le statut
kubectl get pods -n kube-system -l k8s-app=metrics-server

# Voir les logs
kubectl logs -n kube-system -l k8s-app=metrics-server -f

# Vérifier l'API
kubectl get apiservices v1beta1.metrics.k8s.io
```

### Métriques du Metrics Server lui-même

Le Metrics Server expose ses propres métriques (pour Prometheus) :

```bash
kubectl port-forward -n kube-system service/metrics-server 4443:443
```

Puis :
```bash
curl -k https://localhost:4443/metrics
```

Métriques intéressantes :
- `metrics_server_manager_tick_duration_seconds` : Durée de collecte
- `metrics_server_scraper_duration_seconds` : Durée d'interrogation des Kubelets
- `metrics_server_storage_points` : Nombre de points de données stockés

### Alertes recommandées

Si vous utilisez Prometheus, configurez des alertes pour :

```yaml
- alert: MetricsServerDown
  expr: up{job="metrics-server"} == 0
  for: 5m
  annotations:
    summary: "Metrics Server is down"

- alert: MetricsServerHighLatency
  expr: metrics_server_manager_tick_duration_seconds > 10
  for: 10m
  annotations:
    summary: "Metrics Server taking too long to collect metrics"
```

## Bonnes pratiques

### 1. Activer immédiatement le Metrics Server

Dès que vous créez un cluster MicroK8s :

```bash
microk8s enable metrics-server
```

C'est un prérequis pour presque tout ce qui concerne l'autoscaling.

### 2. Toujours définir les requests

Pour **tous** vos Deployments :

```yaml
resources:
  requests:
    cpu: "100m"
    memory: "128Mi"
  limits:
    cpu: "500m"
    memory: "512Mi"
```

Sans cela, le HPA ne peut pas fonctionner correctement.

### 3. Ne pas confondre Metrics Server et Prometheus

- **Metrics Server** : Pour l'autoscaling (requis)
- **Prometheus** : Pour le monitoring complet (fortement recommandé)

Activez les deux dans un cluster de production.

### 4. Comprendre les limites

Le Metrics Server fournit :
- ✅ Métriques actuelles (dernière minute)
- ✅ CPU et mémoire uniquement
- ❌ Pas d'historique
- ❌ Pas de métriques custom

Pour plus, ajoutez Prometheus.

### 5. Vérifier régulièrement

Incluez dans vos checks de santé réguliers :

```bash
#!/bin/bash
# healthcheck.sh

echo "Checking Metrics Server..."
kubectl get pods -n kube-system -l k8s-app=metrics-server | grep Running
if [ $? -eq 0 ]; then
  echo "✓ Metrics Server is running"
else
  echo "✗ Metrics Server is not running!"
  exit 1
fi

echo "Checking metrics API..."
kubectl top nodes > /dev/null 2>&1
if [ $? -eq 0 ]; then
  echo "✓ Metrics API is working"
else
  echo "✗ Metrics API is not working!"
  exit 1
fi
```

### 6. Ne pas surcharger avec des top fréquents

La commande `kubectl top` interroge le Metrics Server. Si vous l'appelez trop souvent (ex: toutes les secondes dans un script), vous pouvez surcharger le Metrics Server.

**Bon :**
```bash
watch -n 5 kubectl top pods    # Toutes les 5 secondes
```

**Mauvais :**
```bash
while true; do kubectl top pods; done    # Trop fréquent !
```

### 7. Surveiller l'utilisation du Metrics Server

Sur de gros clusters (100+ nœuds, 1000+ pods), le Metrics Server peut consommer plus de ressources.

Surveillez :
```bash
kubectl top pods -n kube-system -l k8s-app=metrics-server
```

Si nécessaire, augmentez les ressources :

```bash
kubectl edit deployment metrics-server -n kube-system
```

Modifiez :
```yaml
resources:
  requests:
    cpu: 200m        # Augmenté de 100m
    memory: 400Mi    # Augmenté de 200Mi
```

## Résumé

Le **Metrics Server** est un composant fondamental pour l'autoscaling dans Kubernetes.

**Points clés :**

1. Collecte les métriques CPU et mémoire de tous les nœuds et pods
2. Nécessaire pour que le HPA et le VPA fonctionnent
3. Permet d'utiliser `kubectl top nodes` et `kubectl top pods`
4. Très simple à activer sur MicroK8s : `microk8s enable metrics-server`
5. Collecte les métriques toutes les 60 secondes par défaut
6. Stocke les données en mémoire uniquement (pas d'historique long terme)
7. Fournit uniquement CPU et mémoire (pas de métriques custom ou réseau)

**Relation avec les autres outils :**

```
Metrics Server → Fournit les métriques de base
      ↓
HPA → Utilise ces métriques pour scaler horizontalement
VPA → Utilise ces métriques pour recommander des ressources
kubectl top → Affiche ces métriques pour l'utilisateur
```

**À retenir :**

✅ Activez toujours le Metrics Server (c'est gratuit et sans impact)
✅ Définissez toujours des `requests` pour vos conteneurs
✅ Utilisez `kubectl top` pour surveiller votre cluster
✅ Combinez avec Prometheus pour un monitoring complet
✅ C'est la fondation de tout autoscaling efficace

Sans Metrics Server, vous êtes "aveugle" sur les ressources réelles consommées par votre cluster. C'est comme conduire une voiture sans tableau de bord : vous savez que le moteur tourne, mais vous ne savez ni la vitesse, ni le niveau d'essence, ni la température !

---

**Prochaine section :** 19.6 Custom Metrics - Utiliser des métriques personnalisées pour un autoscaling encore plus intelligent !

⏭️ [Custom metrics](/19-scaling-et-autoscaling/06-custom-metrics.md)
