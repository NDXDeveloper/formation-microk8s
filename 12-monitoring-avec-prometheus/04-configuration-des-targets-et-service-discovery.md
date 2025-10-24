🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 12.4 Configuration des targets et service discovery

## Introduction

Dans un environnement traditionnel, vous devez dire manuellement à votre système de monitoring : "Surveille cette machine à l'adresse 192.168.1.10, et celle-ci à 192.168.1.11, etc." C'est fastidieux et source d'erreurs.

Dans Kubernetes, les choses changent constamment. Les pods sont créés, détruits, déplacés. Leurs adresses IP changent. Comment Prometheus peut-il suivre tout cela automatiquement ?

C'est tout l'enjeu du **Service Discovery** (découverte de services).

## Le problème : Un environnement dynamique

Imaginez ce scénario typique dans Kubernetes :

```
09:00 - Vous déployez une application avec 3 réplicas
      → 3 pods créés avec des IPs : 10.1.1.5, 10.1.1.6, 10.1.1.7

10:30 - Un pod crash, Kubernetes le recrée
      → Nouvelle IP : 10.1.1.12

14:00 - Vous scalez à 5 réplicas
      → 2 nouveaux pods : 10.1.1.18, 10.1.1.19

16:00 - Mise à jour de l'application (rolling update)
      → Tous les pods sont recréés avec de nouvelles IPs
```

**Question** : Comment Prometheus peut-il suivre toutes ces applications sans intervention manuelle ?

**Réponse** : Le Service Discovery automatique !

## Qu'est-ce que le Service Discovery ?

Le **Service Discovery** est le mécanisme par lequel Prometheus découvre automatiquement :
- **Quoi surveiller** : Quels pods, services, ou endpoints scraper
- **Où les trouver** : Leurs adresses IP et ports
- **Comment les identifier** : Leurs labels et métadonnées

### Analogie simple

Imaginez un facteur (Prometheus) qui doit distribuer du courrier (collecter des métriques). Dans un quartier traditionnel, il a une liste fixe d'adresses. Mais dans un quartier Kubernetes, les maisons apparaissent et disparaissent constamment, et changent d'adresse !

Le Service Discovery, c'est comme si le facteur avait accès à un registre en temps réel de toutes les maisons du quartier, avec leurs nouvelles adresses. Il n'a plus besoin d'une liste manuelle, il consulte simplement le registre.

## Comment fonctionne le Service Discovery dans Kubernetes ?

Prometheus utilise l'**API Kubernetes** comme source de vérité. Voici le processus :

```
1. Prometheus interroge l'API Kubernetes
   "Donne-moi la liste de tous les pods/services/endpoints"

2. L'API Kubernetes répond avec tous les objets et leurs métadonnées
   (noms, labels, IPs, ports, namespaces, etc.)

3. Prometheus filtre selon les règles configurées
   "Je veux uniquement les pods avec le label 'monitoring=enabled'"

4. Prometheus extrait les informations nécessaires
   (IP, port, chemin /metrics)

5. Prometheus commence à scraper ces targets

6. Ce processus se répète régulièrement (toutes les 30 secondes par défaut)
```

### Le rôle de Prometheus Operator

MicroK8s utilise **Prometheus Operator**, qui simplifie énormément la configuration. Au lieu de modifier directement la configuration Prometheus (fichier YAML complexe), vous créez simplement des objets Kubernetes spéciaux :

- **ServiceMonitor** : Pour surveiller des Services Kubernetes
- **PodMonitor** : Pour surveiller des Pods directement
- **Probe** : Pour surveiller des endpoints externes

Ces objets sont automatiquement détectés par Prometheus Operator qui met à jour la configuration de Prometheus.

## ServiceMonitor : Surveiller via les Services

### Qu'est-ce qu'un ServiceMonitor ?

Un **ServiceMonitor** est un objet Kubernetes qui dit à Prometheus :

> "Surveille tous les Pods derrière ce Service Kubernetes"

C'est la méthode recommandée car elle utilise les Services, qui sont l'abstraction standard de Kubernetes pour exposer des applications.

### Anatomie d'un ServiceMonitor

Voici un exemple simple :

```yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: mon-application
  namespace: default
  labels:
    app: mon-app
spec:
  # Quel Service surveiller ?
  selector:
    matchLabels:
      app: mon-app

  # Configuration du scraping
  endpoints:
    - port: metrics        # Nom du port (défini dans le Service)
      path: /metrics       # Chemin de l'endpoint
      interval: 30s        # Fréquence de scraping
```

**Explication ligne par ligne** :

1. **apiVersion** : Version de l'API (fournie par Prometheus Operator)
2. **kind: ServiceMonitor** : Type d'objet
3. **metadata** : Informations sur le ServiceMonitor lui-même
4. **selector.matchLabels** : "Trouve les Services avec ces labels"
5. **endpoints** : Configuration de comment scraper

### Comment ça marche ?

```
1. Vous créez un Service pour votre application :

   Service "mon-app" avec label app=mon-app
   ├── Sélectionne les pods avec app=mon-app
   └── Expose le port "metrics" (9090)

2. Vous créez un ServiceMonitor :

   ServiceMonitor cherche les Services avec label app=mon-app
   → Trouve le Service "mon-app"
   → Découvre les Pods derrière ce Service
   → Configure Prometheus pour les scraper

3. Prometheus scrape automatiquement :

   http://pod-1:9090/metrics
   http://pod-2:9090/metrics
   http://pod-3:9090/metrics
```

### Exemple complet

Voici un exemple complet avec une application :

**1. Déploiement de l'application** :

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mon-app
  namespace: default
spec:
  replicas: 3
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
        image: mon-app:v1.0
        ports:
        - name: metrics      # Important : nom du port
          containerPort: 9090
```

**2. Service associé** :

```yaml
apiVersion: v1
kind: Service
metadata:
  name: mon-app
  namespace: default
  labels:
    app: mon-app           # Label pour le ServiceMonitor
spec:
  selector:
    app: mon-app
  ports:
  - name: metrics          # Important : même nom que dans le pod
    port: 9090
    targetPort: metrics
```

**3. ServiceMonitor** :

```yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: mon-app
  namespace: default
  labels:
    prometheus: kube-prometheus  # Important pour être détecté
spec:
  selector:
    matchLabels:
      app: mon-app
  endpoints:
  - port: metrics
    interval: 30s
    path: /metrics
```

**4. Prometheus scrape automatiquement** :

Après quelques secondes, Prometheus détecte le ServiceMonitor et commence à scraper les 3 pods de l'application.

### Label important : prometheus

Pour que Prometheus Operator détecte votre ServiceMonitor, il doit avoir un label spécifique. Par défaut dans MicroK8s :

```yaml
labels:
  prometheus: kube-prometheus
```

Ou pour surveiller dans tous les namespaces :

```yaml
labels:
  release: prometheus
```

**Comment savoir quel label utiliser ?**

Vérifiez la configuration de votre Prometheus :

```bash
microk8s kubectl get prometheus -n monitoring prometheus-k8s -o yaml | grep serviceMonitorSelector -A5
```

## PodMonitor : Surveiller des Pods directement

### Quand utiliser un PodMonitor ?

Utilisez un **PodMonitor** quand :
- Vous voulez surveiller des pods qui n'ont pas de Service
- Vous voulez plus de contrôle granulaire
- Vous utilisez un StatefulSet avec des endpoints individuels

### Anatomie d'un PodMonitor

```yaml
apiVersion: monitoring.coreos.com/v1
kind: PodMonitor
metadata:
  name: mon-app-pods
  namespace: default
  labels:
    prometheus: kube-prometheus
spec:
  # Quels Pods surveiller ?
  selector:
    matchLabels:
      app: mon-app

  # Configuration du scraping
  podMetricsEndpoints:
  - port: metrics
    path: /metrics
    interval: 30s
```

**Différence avec ServiceMonitor** :

- **ServiceMonitor** → Sélectionne des Services → Surveille les Pods derrière
- **PodMonitor** → Sélectionne directement des Pods

### Exemple : Surveiller un StatefulSet

Pour un StatefulSet (comme une base de données), vous pourriez vouloir surveiller chaque instance individuellement :

```yaml
apiVersion: monitoring.coreos.com/v1
kind: PodMonitor
metadata:
  name: postgres-monitoring
  namespace: default
  labels:
    prometheus: kube-prometheus
spec:
  selector:
    matchLabels:
      app: postgres
  podMetricsEndpoints:
  - port: metrics
    path: /metrics
    interval: 15s
  namespaceSelector:
    matchNames:
    - default
```

## Configuration avancée des endpoints

### Plusieurs endpoints par service

Vous pouvez scraper plusieurs endpoints sur le même pod :

```yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: multi-endpoint-app
spec:
  selector:
    matchLabels:
      app: mon-app
  endpoints:
  - port: metrics        # Métriques de l'application
    path: /metrics
    interval: 30s
  - port: metrics        # Métriques business
    path: /business-metrics
    interval: 60s
  - port: http           # Métriques HTTP
    path: /actuator/prometheus
    interval: 30s
```

### Configuration du timeout

Par défaut, Prometheus attend 10 secondes pour chaque scrape. Vous pouvez ajuster :

```yaml
endpoints:
- port: metrics
  path: /metrics
  interval: 30s
  scrapeTimeout: 5s     # Timeout de 5 secondes
```

### Relabeling : Transformer les labels

Le relabeling permet de modifier, ajouter, ou supprimer des labels des métriques collectées.

**Exemple 1 : Ajouter un label personnalisé**

```yaml
endpoints:
- port: metrics
  path: /metrics
  relabelings:
  - sourceLabels: [__meta_kubernetes_pod_name]
    targetLabel: pod_name
  - targetLabel: environment
    replacement: production
```

**Exemple 2 : Renommer un label**

```yaml
endpoints:
- port: metrics
  relabelings:
  - sourceLabels: [__meta_kubernetes_namespace]
    targetLabel: k8s_namespace
```

**Exemple 3 : Filtrer certaines métriques**

```yaml
endpoints:
- port: metrics
  metricRelabelings:
  - sourceLabels: [__name__]
    regex: 'go_.*'        # Supprime toutes les métriques go_*
    action: drop
```

### Authentication et TLS

Pour les endpoints sécurisés :

```yaml
endpoints:
- port: metrics
  path: /metrics
  scheme: https           # Utilise HTTPS
  tlsConfig:
    insecureSkipVerify: true   # Pour les certificats auto-signés
  bearerTokenFile: /var/run/secrets/kubernetes.io/serviceaccount/token
```

## Labels Kubernetes utilisés par le Service Discovery

Prometheus utilise des **meta labels** provenant de Kubernetes pour découvrir et labelliser les targets. Voici les plus importants :

### Meta labels pour les Pods

```
__meta_kubernetes_pod_name                  # Nom du pod
__meta_kubernetes_pod_namespace             # Namespace du pod
__meta_kubernetes_pod_ip                    # IP du pod
__meta_kubernetes_pod_label_<labelname>     # Labels du pod
__meta_kubernetes_pod_annotation_<name>     # Annotations du pod
__meta_kubernetes_pod_node_name             # Node où s'exécute le pod
__meta_kubernetes_pod_container_name        # Nom du conteneur
__meta_kubernetes_pod_container_port_name   # Nom du port
__meta_kubernetes_pod_container_port_number # Numéro du port
__meta_kubernetes_pod_ready                 # État ready du pod
```

### Meta labels pour les Services

```
__meta_kubernetes_service_name              # Nom du service
__meta_kubernetes_service_namespace         # Namespace du service
__meta_kubernetes_service_label_<labelname> # Labels du service
__meta_kubernetes_service_annotation_<name> # Annotations du service
```

### Exemple d'utilisation

Ces meta labels sont automatiquement ajoutés par Prometheus, mais vous pouvez les utiliser pour créer vos propres labels :

```yaml
endpoints:
- port: metrics
  relabelings:
  # Créer un label avec le nom du namespace
  - sourceLabels: [__meta_kubernetes_namespace]
    targetLabel: kubernetes_namespace

  # Créer un label avec le nom du pod
  - sourceLabels: [__meta_kubernetes_pod_name]
    targetLabel: pod

  # Ajouter le nom du node
  - sourceLabels: [__meta_kubernetes_pod_node_name]
    targetLabel: node
```

## Configuration via annotations

Une méthode alternative (plus simple mais moins flexible) consiste à utiliser des **annotations** directement sur vos Pods ou Services.

### Annotations Prometheus standard

Ajoutez ces annotations à votre Deployment/Service :

```yaml
apiVersion: v1
kind: Service
metadata:
  name: mon-app
  annotations:
    prometheus.io/scrape: "true"      # Active le scraping
    prometheus.io/port: "9090"        # Port à scraper
    prometheus.io/path: "/metrics"    # Chemin (défaut: /metrics)
    prometheus.io/scheme: "http"      # http ou https
spec:
  # ... reste de la configuration
```

**Note importante** : Cette méthode nécessite une configuration Prometheus qui recherche ces annotations. Avec Prometheus Operator (utilisé par MicroK8s), **il est préférable d'utiliser des ServiceMonitors**.

## Namespace selection

Par défaut, un ServiceMonitor surveille uniquement les Services dans son propre namespace. Pour surveiller d'autres namespaces :

### Surveiller un namespace spécifique

```yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: mon-app
  namespace: monitoring
spec:
  namespaceSelector:
    matchNames:
    - default
    - production
  selector:
    matchLabels:
      app: mon-app
  endpoints:
  - port: metrics
```

### Surveiller tous les namespaces

```yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: mon-app
  namespace: monitoring
spec:
  namespaceSelector:
    any: true              # Surveille tous les namespaces
  selector:
    matchLabels:
      app: mon-app
  endpoints:
  - port: metrics
```

**Attention** : Surveiller tous les namespaces peut créer beaucoup de targets. Utilisez avec précaution.

## Vérifier le Service Discovery

### Via l'interface web Prometheus

La meilleure façon de vérifier que vos ServiceMonitors fonctionnent :

1. Accédez à Prometheus : `http://localhost:9090`
2. Allez dans **Status > Service Discovery**
3. Cherchez vos ServiceMonitors dans la liste

Vous verrez :
- Les ServiceMonitors découverts
- Les Services/Pods qu'ils ciblent
- Les labels appliqués
- Les endpoints générés

### Via l'interface Targets

1. Allez dans **Status > Targets**
2. Cherchez votre application
3. Vérifiez que l'état est **UP** (vert)

Si l'état est **DOWN** (rouge), cliquez dessus pour voir l'erreur.

### Via kubectl

Lister tous les ServiceMonitors :

```bash
microk8s kubectl get servicemonitors -n monitoring
```

Voir les détails d'un ServiceMonitor :

```bash
microk8s kubectl describe servicemonitor mon-app -n default
```

Vérifier les labels d'un ServiceMonitor :

```bash
microk8s kubectl get servicemonitor mon-app -n default --show-labels
```

## Exemples de ServiceMonitors par défaut

MicroK8s installe plusieurs ServiceMonitors par défaut. Regardons-les :

### ServiceMonitor pour Prometheus lui-même

```bash
microk8s kubectl get servicemonitor prometheus-k8s -n monitoring -o yaml
```

Ce ServiceMonitor indique à Prometheus de se surveiller lui-même !

### ServiceMonitor pour node-exporter

```bash
microk8s kubectl get servicemonitor node-exporter -n monitoring -o yaml
```

Surveille les métriques système de tous les nodes.

### ServiceMonitor pour kube-state-metrics

```bash
microk8s kubectl get servicemonitor kube-state-metrics -n monitoring -o yaml
```

Surveille l'état des objets Kubernetes.

**Conseil** : Étudiez ces ServiceMonitors existants pour comprendre comment les configurer.

## Bonnes pratiques

### 1. Utilisez des noms de ports explicites

```yaml
# ✅ Bon
ports:
- name: metrics
  containerPort: 9090

# ❌ Moins bon
ports:
- containerPort: 9090
```

Avec un nom explicite, votre ServiceMonitor est plus clair :

```yaml
endpoints:
- port: metrics  # Clair et lisible
```

### 2. Organisez vos ServiceMonitors par namespace

Gardez vos ServiceMonitors dans le même namespace que les applications qu'ils surveillent :

```
Namespace: default
├── Deployment: mon-app
├── Service: mon-app
└── ServiceMonitor: mon-app

Namespace: production
├── Deployment: api
├── Service: api
└── ServiceMonitor: api
```

### 3. Utilisez des labels cohérents

Adoptez une convention de nommage pour vos labels :

```yaml
labels:
  app: mon-app
  component: backend
  team: platform
  monitoring: enabled
```

### 4. Documentez vos ServiceMonitors

Ajoutez des annotations pour documenter :

```yaml
metadata:
  name: mon-app
  annotations:
    description: "Surveillance de l'API principale"
    contact: "team-platform@example.com"
    documentation: "https://wiki.example.com/monitoring/api"
```

### 5. Limitez le scope

Ne surveillez que ce qui est nécessaire :

```yaml
# ✅ Bon : surveille uniquement le namespace production
namespaceSelector:
  matchNames:
  - production

# ❌ Évitez : surveille tous les namespaces
namespaceSelector:
  any: true
```

### 6. Utilisez un intervalle de scraping adapté

```yaml
# Métriques importantes (APM, erreurs)
endpoints:
- port: metrics
  interval: 15s     # Scrape fréquent

# Métriques moins critiques (statistiques)
endpoints:
- port: stats
  interval: 60s     # Scrape moins fréquent
```

### 7. Ajoutez des labels d'environnement

```yaml
endpoints:
- port: metrics
  relabelings:
  - targetLabel: environment
    replacement: production
  - targetLabel: region
    replacement: eu-west-1
```

## Dépannage du Service Discovery

### Problème 1 : ServiceMonitor créé mais pas de target

**Symptômes** :
- ServiceMonitor existe
- Aucune target dans Prometheus

**Causes possibles** :

1. **Label prometheus manquant**

Vérifiez que votre ServiceMonitor a le bon label :

```bash
microk8s kubectl get servicemonitor mon-app -o yaml | grep prometheus
```

Devrait contenir :

```yaml
labels:
  prometheus: kube-prometheus
```

2. **Selector ne trouve aucun Service**

Vérifiez que les labels correspondent :

```bash
# Labels du ServiceMonitor
microk8s kubectl get servicemonitor mon-app -o yaml | grep -A3 selector

# Labels du Service
microk8s kubectl get service mon-app --show-labels
```

3. **Namespace incorrect**

Si le ServiceMonitor est dans un namespace différent, ajoutez :

```yaml
spec:
  namespaceSelector:
    matchNames:
    - default  # Namespace où se trouve votre Service
```

### Problème 2 : Target DOWN

**Symptômes** :
- Target visible dans Prometheus
- État : DOWN (rouge)

**Vérifications** :

1. **Le pod est-il accessible ?**

```bash
# Test de connectivité depuis un pod test
microk8s kubectl run test --rm -it --image=busybox -- wget -O- http://mon-app:9090/metrics
```

2. **Le port est-il correct ?**

Vérifiez que le port dans le ServiceMonitor correspond :

```bash
# Port dans le Service
microk8s kubectl get service mon-app -o yaml | grep -A5 ports

# Port dans le ServiceMonitor
microk8s kubectl get servicemonitor mon-app -o yaml | grep -A3 endpoints
```

3. **Le chemin /metrics existe-t-il ?**

```bash
# Accédez au pod et testez
microk8s kubectl exec -it mon-app-xxxx -- curl localhost:9090/metrics
```

### Problème 3 : Pas de métriques dans Prometheus

**Symptômes** :
- Target UP
- Mais requêtes PromQL ne retournent rien

**Causes possibles** :

1. **L'application n'expose pas de métriques**

Vérifiez manuellement :

```bash
microk8s kubectl port-forward mon-app-xxxx 9090:9090
curl http://localhost:9090/metrics
```

Vous devriez voir des métriques au format Prometheus :

```
# HELP http_requests_total Total HTTP requests
# TYPE http_requests_total counter
http_requests_total{method="GET",status="200"} 1547
```

2. **Les métriques sont filtrées**

Vérifiez s'il y a des règles de metricRelabelings qui suppriment vos métriques.

### Problème 4 : Trop de métriques (cardinality explosion)

**Symptômes** :
- Prometheus utilise beaucoup de RAM
- Ralentissements

**Causes** :
- Trop de labels uniques
- Labels avec des valeurs à haute cardinalité (user_id, timestamp, etc.)

**Solutions** :

Filtrez les métriques problématiques :

```yaml
endpoints:
- port: metrics
  metricRelabelings:
  # Supprimer les labels à haute cardinalité
  - sourceLabels: [user_id]
    action: labeldrop
  # Garder seulement certaines métriques
  - sourceLabels: [__name__]
    regex: 'http_.*|app_.*'
    action: keep
```

## Monitoring de métriques personnalisées

### Exposer des métriques depuis votre application

Pour que Prometheus puisse surveiller votre application, elle doit exposer des métriques.

**Exemple conceptuel en Python** :

```python
from prometheus_client import Counter, Histogram, start_http_server

# Créer des métriques
requests_total = Counter('http_requests_total', 'Total HTTP requests', ['method', 'endpoint'])
request_duration = Histogram('http_request_duration_seconds', 'HTTP request duration')

# Dans votre code
@app.route('/api/users')
def get_users():
    requests_total.labels(method='GET', endpoint='/api/users').inc()
    with request_duration.time():
        # Votre logique métier
        return jsonify(users)

# Exposer les métriques sur le port 9090
start_http_server(9090)
```

**Déploiement dans Kubernetes** :

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mon-api
spec:
  replicas: 3
  selector:
    matchLabels:
      app: mon-api
  template:
    metadata:
      labels:
        app: mon-api
    spec:
      containers:
      - name: api
        image: mon-api:v1.0
        ports:
        - name: http
          containerPort: 8080
        - name: metrics      # Port pour Prometheus
          containerPort: 9090
---
apiVersion: v1
kind: Service
metadata:
  name: mon-api
  labels:
    app: mon-api
spec:
  selector:
    app: mon-api
  ports:
  - name: http
    port: 8080
  - name: metrics          # Port pour Prometheus
    port: 9090
---
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: mon-api
  labels:
    prometheus: kube-prometheus
spec:
  selector:
    matchLabels:
      app: mon-api
  endpoints:
  - port: metrics
    interval: 30s
```

## Résumé

Dans cette section, vous avez appris :

✅ **Le concept de Service Discovery** et pourquoi il est essentiel dans Kubernetes
✅ **Les ServiceMonitors** pour surveiller via les Services Kubernetes
✅ **Les PodMonitors** pour surveiller directement les Pods
✅ **La configuration avancée** (relabeling, namespaces, authentication)
✅ **Les meta labels Kubernetes** utilisés par Prometheus
✅ **Les bonnes pratiques** pour organiser vos configurations
✅ **Le dépannage** des problèmes de Service Discovery
✅ **Comment exposer** des métriques personnalisées

**Points clés à retenir** :

1. Le Service Discovery automatise la découverte des targets
2. ServiceMonitors sont la méthode recommandée avec Prometheus Operator
3. Les labels sont cruciaux pour le matching
4. Vérifiez toujours dans l'UI Prometheus que vos targets sont UP
5. Commencez simple, ajoutez de la complexité progressivement

Dans la prochaine section (12.5), nous plongerons dans **PromQL**, le langage de requête de Prometheus, pour apprendre à interroger et analyser toutes ces métriques que nous collectons.

---

**Conseil pratique** : Créez un ServiceMonitor simple pour une application de test, vérifiez qu'il fonctionne dans l'UI Prometheus, puis ajoutez progressivement des fonctionnalités avancées. L'apprentissage itératif est la meilleure approche !

⏭️ [PromQL : langage de requête Prometheus](/12-monitoring-avec-prometheus/05-promql-langage-de-requete-prometheus.md)
