🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 12.2 Architecture Prometheus

## Introduction

Dans la section précédente, nous avons découvert l'écosystème Prometheus dans son ensemble. Maintenant, plongeons dans l'architecture technique pour comprendre comment tout s'articule. Ne vous inquiétez pas si cela semble complexe au premier abord : nous allons décomposer chaque élément de manière progressive.

## Vue d'ensemble de l'architecture

Voici une représentation simplifiée de l'architecture Prometheus dans un cluster Kubernetes :

```
┌────────────────────────────────────────────────────────────────────┐
│                      CLUSTER KUBERNETES                            │
│                                                                    │
│  ┌──────────────┐      ┌────────────────┐      ┌────────────────┐  │
│  │   Pod App 1  │      │   Pod App 2    │      │   Pod App 3    │  │
│  │  :8080/metrics│     │  :8080/metrics │      │  :8080/metrics │  │
│  └──────────────┘      └────────────────┘      └────────────────┘  │
│         ↑                      ↑                      ↑            │
│         │                      │                      │            │
│         │      Scrape (pull)   │                      │            │
│         └──────────────────────┼──────────────────────┘            │
│                                │                                   │
│                    ┌───────────▼──────────┐                        │
│                    │  PROMETHEUS SERVER   │                        │
│                    │                      │                        │
│                    │  • Service Discovery │                        │
│                    │  • Time Series DB    │                        │
│                    │  • PromQL Engine     │                        │
│                    │  • Alert Rules       │                        │
│                    │  • Web UI            │                        │
│                    └───────────┬──────────┘                        │
│                                │                                   │
│                    ┌───────────▼──────────┐                        │
│                    │   ALERTMANAGER       │                        │
│                    │                      │                        │
│                    │  • Alert Routing     │                        │
│                    │  • Grouping          │                        │
│                    │  • Deduplication     │                        │
│                    └───────────┬──────────┘                        │
│                                │                                   │
└────────────────────────────────┼───────────────────────────────────┘
                                 │
                    ┌────────────▼────────────┐
                    │    NOTIFICATIONS        │
                    │                         │
                    │  • Email                │
                    │  • Slack                │
                    │  • PagerDuty            │
                    │  • Webhook              │
                    └─────────────────────────┘

         ┌──────────────────────┐
         │      GRAFANA         │ ◄─── Requêtes PromQL
         │                      │
         │  • Dashboards        │
         │  • Visualisations    │
         └──────────────────────┘
```

## Les composants principaux en détail

### 1. Prometheus Server : Le cerveau du système

Le serveur Prometheus est le composant central qui orchestre tout. Il est composé de plusieurs modules internes :

#### A. Service Discovery (Découverte de services)

**Qu'est-ce que c'est ?**

Dans un environnement Kubernetes, les pods naissent et meurent constamment. Leurs adresses IP changent. Comment Prometheus peut-il savoir quoi surveiller ?

C'est là qu'intervient le **Service Discovery** (découverte de services). Au lieu de configurer manuellement chaque cible à surveiller, Prometheus interroge l'API Kubernetes pour découvrir automatiquement :
- Les pods
- Les services
- Les endpoints
- Les nodes

**Comment ça fonctionne ?**

```
1. Prometheus interroge l'API Kubernetes
2. Kubernetes répond : "Voici tous les pods avec le label 'monitoring=enabled'"
3. Prometheus extrait les adresses IP et ports
4. Prometheus commence à scraper ces endpoints
```

**Exemple concret** :

Si vous déployez une nouvelle application avec un label `prometheus.io/scrape: "true"`, Prometheus la détectera automatiquement et commencera à collecter ses métriques. Pas besoin de redémarrer Prometheus ou de modifier sa configuration !

#### B. Time Series Database (Base de données temporelle)

**Qu'est-ce que c'est ?**

Une base de données optimisée pour stocker des séries de valeurs dans le temps. Contrairement à une base de données classique (MySQL, PostgreSQL), elle est conçue spécifiquement pour les métriques.

**Structure des données** :

Chaque métrique est stockée comme une série temporelle :

```
métrique{labels} valeur timestamp

Exemple :
http_requests_total{method="GET", status="200", instance="app-1"} 1547 1698765432
http_requests_total{method="GET", status="200", instance="app-1"} 1552 1698765447
http_requests_total{method="GET", status="200", instance="app-1"} 1560 1698765462
```

**Pourquoi une base de données spécialisée ?**

- **Compression efficace** : Les métriques ont souvent des patterns répétitifs
- **Requêtes rapides** : Optimisée pour les agrégations temporelles
- **Retention automatique** : Suppression des anciennes données selon la configuration
- **Stockage en blocks** : Les données sont organisées en blocs de 2h (par défaut) pour une meilleure performance

**Caractéristiques importantes** :

- **Stockage local** : Par défaut, Prometheus stocke sur le disque local (pas de base externe)
- **Rétention configurable** : Par défaut 15 jours, mais vous pouvez ajuster selon votre espace disque
- **Pas de haute disponibilité native** : Un serveur Prometheus = un stockage isolé

#### C. PromQL Engine (Moteur de requêtes)

**Qu'est-ce que c'est ?**

C'est le moteur qui exécute vos requêtes PromQL pour extraire et transformer les données. Il peut :
- Filtrer les séries temporelles
- Effectuer des agrégations (somme, moyenne, max, min)
- Calculer des taux de changement
- Combiner plusieurs métriques
- Appliquer des fonctions mathématiques

**Exemple simple** :

```promql
# Taux de requêtes HTTP par seconde
rate(http_requests_total[5m])

# Utilisation CPU moyenne par pod
avg(container_cpu_usage_seconds_total) by (pod)

# Pods utilisant plus de 80% de mémoire
container_memory_usage_bytes / container_memory_limit_bytes > 0.8
```

#### D. Alert Rules Engine (Moteur de règles d'alerte)

**Qu'est-ce que c'est ?**

Ce module évalue périodiquement (par défaut toutes les minutes) les règles d'alerte que vous avez définies.

**Cycle de vie d'une alerte** :

```
1. Règle définie : "CPU > 80% pendant 5 minutes"
2. Évaluation : Le moteur exécute la requête PromQL
3. Condition vraie : L'alerte passe en état "Pending"
4. Durée atteinte : L'alerte passe en état "Firing"
5. Envoi : Alertmanager reçoit l'alerte
6. Résolution : Quand la condition n'est plus vraie, alerte résolue
```

**États d'une alerte** :

- **Inactive** : La condition n'est pas remplie
- **Pending** : La condition est remplie, mais pas encore assez longtemps
- **Firing** : La condition est remplie depuis la durée définie, alerte active

#### E. HTTP Server et Web UI

**Qu'est-ce que c'est ?**

Prometheus expose une interface web simple sur le port 9090 qui permet de :
- Exécuter des requêtes PromQL manuellement
- Visualiser les graphiques basiques
- Voir l'état des targets (cibles surveillées)
- Consulter les règles d'alerte et leur état
- Vérifier la configuration

**Utilité** :

C'est principalement un outil de débogage et d'exploration. Pour de vraies visualisations, on utilise Grafana, mais l'UI Prometheus est pratique pour tester rapidement des requêtes.

### 2. Retrieval (Module de collecte)

**Qu'est-ce que c'est ?**

C'est le composant qui effectue le travail de scraping, c'est-à-dire qui va chercher les métriques auprès des targets.

**Processus de scraping** :

```
1. Service Discovery fournit la liste des targets
2. Retrieval lit la configuration (intervalle, timeout, labels)
3. Pour chaque target, Retrieval envoie une requête HTTP GET vers /metrics
4. La target répond avec ses métriques au format Prometheus
5. Retrieval parse les métriques et ajoute les labels configurés
6. Les métriques sont envoyées au stockage
```

**Configuration du scraping** :

```yaml
scrape_configs:
  - job_name: 'my-app'
    scrape_interval: 15s      # Fréquence de scraping
    scrape_timeout: 10s       # Timeout pour chaque requête
    metrics_path: '/metrics'  # Chemin de l'endpoint
    static_configs:
      - targets: ['app:8080']
```

**Gestion des erreurs** :

Si un scrape échoue :
- Prometheus enregistre l'erreur
- La métrique `up{job="my-app", instance="app:8080"}` passe à 0
- Prometheus réessaiera au prochain intervalle
- Vous pouvez voir les erreurs dans l'UI Prometheus

### 3. Exporters : Les traducteurs de métriques

Les exporters ne font pas techniquement partie de Prometheus, mais ils sont essentiels à l'architecture.

#### Node Exporter

**Rôle** : Expose les métriques du système d'exploitation et du matériel

**Métriques fournies** :
- Utilisation CPU par core
- Mémoire (totale, disponible, utilisée, swap)
- Disque (espace, I/O, latence)
- Réseau (octets transmis/reçus, erreurs, paquets droppés)
- Filesystem (points de montage, espace libre)
- Statistiques système (load average, uptime)

**Déploiement** : Généralement un DaemonSet (un pod par node)

#### kube-state-metrics

**Rôle** : Expose l'état des objets Kubernetes (API Kubernetes → Métriques Prometheus)

**Métriques fournies** :
- État des pods (running, pending, failed)
- État des deployments (replicas desired vs available)
- État des nodes (ready, not ready)
- Ressources requests et limits
- Labels et annotations
- État des PersistentVolumeClaims

**Différence avec les métriques Kubelet** :
- kube-state-metrics = état des objets Kubernetes
- Kubelet/cAdvisor = métriques de performance des conteneurs

#### cAdvisor

**Rôle** : Collecte les métriques au niveau conteneur (intégré dans Kubelet)

**Métriques fournies** :
- Utilisation CPU par conteneur
- Utilisation mémoire par conteneur
- I/O disque par conteneur
- Statistiques réseau par conteneur
- Performance du filesystem

**Point important** : cAdvisor est déjà présent dans chaque node Kubernetes, pas besoin de l'installer.

### 4. Alertmanager : Le gestionnaire d'alertes

Alertmanager reçoit les alertes de Prometheus et les gère intelligemment.

#### Architecture interne d'Alertmanager

```
Alertes Prometheus
       ↓
   ┌────────────────────┐
   │   ALERTMANAGER     │
   │                    │
   │  ┌──────────────┐  │
   │  │  Dispatcher  │  │  ← Réception des alertes
   │  └──────┬───────┘  │
   │         ↓          │
   │  ┌──────────────┐  │
   │  │   Grouping   │  │  ← Regroupe les alertes similaires
   │  └──────┬───────┘  │
   │         ↓          │
   │  ┌──────────────┐  │
   │  │ Deduplication│  │  ← Élimine les doublons
   │  └──────┬───────┘  │
   │         ↓          │
   │  ┌──────────────┐  │
   │  │   Silences   │  │  ← Vérifie si l'alerte est silencée
   │  └──────┬───────┘  │
   │         ↓          │
   │  ┌──────────────┐  │
   │  │   Inhibition │  │  ← Supprime les alertes dépendantes
   │  └──────┬───────┘  │
   │         ↓          │
   │  ┌──────────────┐  │
   │  │   Routing    │  │  ← Détermine où envoyer l'alerte
   │  └──────┬───────┘  │
   │         ↓          │
   │  ┌──────────────┐  │
   │  │  Notifiers   │  │  ← Envoie aux destinations
   │  └──────────────┘  │
   └────────────────────┘
```

#### Fonctionnalités clés

**Grouping (Regroupement)** :

Au lieu de recevoir 50 alertes "Pod down" séparément, vous recevez un seul message :
```
[FIRING:50] Pods are down in namespace production
- pod-1 is down
- pod-2 is down
- pod-3 is down
...
```

**Deduplication (Déduplication)** :

Si Prometheus redémarre ou si vous avez plusieurs serveurs Prometheus, Alertmanager élimine les alertes identiques.

**Silences** :

Vous pouvez temporairement désactiver des alertes :
```
"Je fais une maintenance sur le serveur DB,
ignore toutes les alertes 'Database down' pendant 2 heures"
```

**Inhibition** :

Supprime les alertes dépendantes. Exemple :
```
Si "Node down" est actif,
alors supprime toutes les alertes "Pod down" sur ce node
(car elles sont une conséquence évidente)
```

**Routing** :

Dirige les alertes vers les bonnes personnes :
```yaml
routes:
  - match:
      severity: critical
      team: backend
    receiver: backend-oncall-pagerduty

  - match:
      severity: warning
      team: frontend
    receiver: frontend-slack
```

### 5. Exporters d'application

#### Exporters officiels

Prometheus fournit des exporters officiels pour de nombreuses technologies :

- **PostgreSQL Exporter** : Métriques de la base de données
- **MySQL Exporter** : Métriques MySQL/MariaDB
- **Redis Exporter** : Métriques Redis
- **NGINX Exporter** : Métriques du serveur web
- **HAProxy Exporter** : Métriques du load balancer
- **Elasticsearch Exporter** : Métriques Elasticsearch

#### Bibliothèques client (Instrumentation)

Pour vos propres applications, Prometheus fournit des bibliothèques dans plusieurs langages :

- **Go** : github.com/prometheus/client_golang
- **Python** : prometheus_client
- **Java** : io.prometheus:simpleclient
- **Node.js** : prom-client
- **.NET** : prometheus-net
- **Ruby** : prometheus-client

**Exemple conceptuel** :

```python
# Dans votre application Python
from prometheus_client import Counter, Histogram

# Compteur de requêtes
http_requests = Counter('http_requests_total', 'Total HTTP requests', ['method', 'endpoint'])

# Histogramme de latence
request_latency = Histogram('http_request_duration_seconds', 'HTTP request latency')

# Dans votre code
http_requests.labels(method='GET', endpoint='/api/users').inc()
with request_latency.time():
    # Votre logique métier
    process_request()
```

## Flux de données complet

Voici le parcours d'une métrique de sa création à sa visualisation :

```
1. APPLICATION
   ↓
   Génère une métrique : http_requests_total{method="GET"} = 1547
   ↓

2. EXPOSITION
   ↓
   Expose sur HTTP : GET http://app:8080/metrics
   ↓

3. DÉCOUVERTE
   ↓
   Service Discovery détecte l'application
   ↓

4. COLLECTE
   ↓
   Retrieval scrape l'endpoint toutes les 15s
   ↓

5. STOCKAGE
   ↓
   Time Series DB enregistre :
   http_requests_total{method="GET", instance="app-1"} 1547 @1698765432
   http_requests_total{method="GET", instance="app-1"} 1552 @1698765447
   ↓

6. ÉVALUATION
   ↓
   Alert Rules Engine vérifie les conditions
   rate(http_requests_total[5m]) < 10 → Alerte !
   ↓

7. ALERTE
   ↓
   Alertmanager reçoit, groupe, route
   ↓

8. NOTIFICATION
   ↓
   Email/Slack : "Low traffic detected on app-1"


En parallèle :

6bis. REQUÊTE
   ↓
   Grafana envoie une requête PromQL
   ↓

7bis. TRAITEMENT
   ↓
   PromQL Engine exécute la requête
   ↓

8bis. VISUALISATION
   ↓
   Grafana affiche le graphique
```

## Considérations d'architecture

### Scalabilité

**Limites d'un seul serveur Prometheus** :
- Peut gérer des milliers de targets
- Millions de séries temporelles
- Dépend du CPU, RAM, et I/O disque

**Quand scaler ?**
- Trop de métriques pour un seul serveur
- Nécessité de haute disponibilité
- Séparation par environnement ou équipe

**Stratégies de scaling** :
- **Fédération** : Prometheus hiérarchiques (un central agrège plusieurs locaux)
- **Sharding** : Découper par namespace/application
- **Remote Storage** : Utiliser un stockage long-terme (Thanos, Cortex)

### Stockage et rétention

**Combien d'espace disque ?**

Estimation simplifiée :
```
Espace = nb_de_séries × échantillons_par_jour × taille_échantillon × rétention_jours

Exemple :
100 000 séries × 5 760 échantillons/jour × 2 octets × 15 jours
= 17 Go environ
```

**Ajuster la rétention** :
```yaml
# Dans la configuration Prometheus
storage:
  tsdb:
    retention.time: 15d    # Garder 15 jours
    retention.size: 50GB   # Ou 50 Go maximum
```

### Haute disponibilité

**Problème** : Un seul Prometheus = point unique de défaillance

**Solutions** :
1. **Plusieurs Prometheus identiques** : Ils scrapent tous les mêmes targets
2. **Alertmanager en cluster** : Ils se synchronisent pour dédupliquer
3. **Stockage externe** : Pour ne pas perdre l'historique

**Note importante pour MicroK8s** :

Pour un lab, la haute disponibilité n'est généralement pas nécessaire. Une instance Prometheus suffit. Nous couvrirons ce sujet dans la section 12.8 pour ceux qui veulent aller plus loin.

## Architecture dans MicroK8s

Lorsque vous exécutez `microk8s enable prometheus`, voici ce qui est déployé :

```
Namespace: monitoring

Deployments:
├── prometheus-k8s (Prometheus Server)
├── alertmanager-main (Alertmanager)
├── kube-state-metrics (État Kubernetes)
└── prometheus-operator (Gestion des CRDs Prometheus)

DaemonSets:
└── node-exporter (Un par node)

Services:
├── prometheus-k8s (Port 9090)
├── alertmanager-main (Port 9093)
└── kube-state-metrics (Port 8080)

ConfigMaps:
├── prometheus-k8s-rulefiles (Règles d'alerte)
└── alertmanager-main (Configuration Alertmanager)

Secrets:
└── prometheus-k8s (Tokens, certificats)
```

**Prometheus Operator** :

MicroK8s utilise le Prometheus Operator, qui est une extension Kubernetes qui facilite la gestion de Prometheus via des Custom Resources :

- **ServiceMonitor** : Définit quels services scraper
- **PodMonitor** : Définit quels pods scraper
- **PrometheusRule** : Définit les règles d'alerte
- **Alertmanager** : Configuration d'Alertmanager

Pas d'inquiétude si ces concepts sont nouveaux, nous les explorerons en pratique dans les prochaines sections.

## Points clés à retenir

1. **Prometheus Server** est le cœur qui collecte, stocke, et évalue les métriques
2. **Service Discovery** permet la découverte automatique des targets dans Kubernetes
3. **Time Series Database** stocke efficacement les métriques avec leur historique
4. **Exporters** traduisent les métriques de différents systèmes au format Prometheus
5. **Alertmanager** gère intelligemment les alertes (grouping, routing, silences)
6. **PromQL** est le langage pour interroger les données
7. L'architecture est modulaire et chaque composant a un rôle précis

## Prochaines étapes

Maintenant que vous comprenez l'architecture, dans la prochaine section (12.3), nous allons **installer et configurer Prometheus** dans votre cluster MicroK8s et explorer l'interface web.

---

**Conseil** : Ne vous sentez pas obligé de mémoriser tous ces détails immédiatement. Revenez à cette section lorsque vous aurez des questions sur "comment fonctionne telle ou telle partie". L'architecture prendra tout son sens au fur et à mesure de votre pratique.

⏭️ [Installation de Prometheus (microk8s enable prometheus)](/12-monitoring-avec-prometheus/03-installation-de-prometheus-sur-microk8s.md)
