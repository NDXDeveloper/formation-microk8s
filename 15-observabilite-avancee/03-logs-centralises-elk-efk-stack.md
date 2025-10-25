🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 15.3 Logs Centralisés (ELK/EFK Stack)

## Introduction aux Logs Centralisés

### Le Problème des Logs Distribués

Dans un cluster Kubernetes comme MicroK8s, vous avez potentiellement :
- Plusieurs pods répartis sur différents nœuds
- Des pods qui redémarrent et perdent leurs logs
- Des applications qui écrivent dans différents formats
- Des logs éparpillés difficiles à corréler

**Exemple du problème** :
```
Utilisateur : "Mon achat a échoué à 14h32"
Vous : "Voyons voir..."
  → Logs du pod frontend : ❌ Pod redémarré, logs perdus
  → Logs du pod backend : ✅ "Error: payment timeout"
  → Logs du pod database : ✅ "Slow query: 3.2s"
  → Logs du pod payment : ❌ Pod sur un autre nœud

Comment tout relier ? 🤔
```

### La Solution : Centralisation des Logs

L'idée est simple : **collecter tous les logs au même endroit**.

```
┌──────────┐  ┌──────────┐  ┌──────────┐
│  Pod 1   │  │  Pod 2   │  │  Pod 3   │
│  logs    │  │  logs    │  │  logs    │
└────┬─────┘  └────┬─────┘  └────┬─────┘
     │             │             │
     └─────────────┴─────────────┘
                   ↓
           ┌──────────────┐
           │  Collecteur  │
           │  (Fluentd)   │
           └──────┬───────┘
                  ↓
           ┌──────────────┐
           │ Elasticsearch│ ← Stockage centralisé
           └──────┬───────┘
                  ↓
           ┌──────────────┐
           │   Kibana     │ ← Interface de recherche
           └──────────────┘
```

**Avantages** :
- ✅ Tous les logs accessibles depuis un seul endroit
- ✅ Logs préservés même si les pods redémarrent
- ✅ Recherche puissante sur tous les logs
- ✅ Corrélation facile entre services
- ✅ Rétention configurable (garder les logs X jours)
- ✅ Visualisation et dashboards

## ELK vs EFK : Deux Stacks Similaires

Il existe deux stacks principales pour la centralisation des logs. Elles sont très similaires et utilisent les mêmes principes.

### Stack ELK

**ELK** = **E**lasticsearch + **L**ogstash + **K**ibana

```
Logs → Logstash → Elasticsearch → Kibana
```

**Composants** :
- **Elasticsearch** : Base de données pour stocker et indexer les logs
- **Logstash** : Collecteur et transformateur de logs (écrit en Ruby/JRuby)
- **Kibana** : Interface web pour visualiser et rechercher dans les logs

### Stack EFK

**EFK** = **E**lasticsearch + **F**luentd + **K**ibana

```
Logs → Fluentd → Elasticsearch → Kibana
```

**Composants** :
- **Elasticsearch** : Base de données (identique à ELK)
- **Fluentd** : Collecteur et transformateur de logs (écrit en Ruby/C)
- **Kibana** : Interface web (identique à ELK)

### Comparaison ELK vs EFK

| Aspect | Logstash (ELK) | Fluentd (EFK) |
|--------|---------------|---------------|
| **Langage** | Ruby/JRuby | Ruby + C |
| **Mémoire** | ~200-500 MB | ~40-80 MB |
| **Performance** | Bon | Excellent |
| **Plugins** | ~200+ | ~500+ |
| **Configuration** | DSL complexe | YAML simple |
| **Docker/K8s** | Bon support | Natif, mieux intégré |
| **Popularité** | Très populaire | Standard dans K8s |

### Quelle Stack Choisir ?

**Pour MicroK8s, nous recommandons EFK** pour ces raisons :
- ✅ Plus léger en mémoire (important pour un lab)
- ✅ Configuration plus simple
- ✅ Mieux intégré dans l'écosystème Kubernetes
- ✅ Standard de facto dans Kubernetes
- ✅ DaemonSet natif pour collecter sur tous les nœuds

**Mais** : Les concepts sont identiques, si vous connaissez l'un, vous connaissez l'autre.

## Architecture de la Stack EFK

### Vue d'Ensemble

```
┌─────────────────────────────────────────────────────────┐
│                  Cluster MicroK8s                       │
│                                                         │
│  ┌────────┐  ┌────────┐  ┌────────┐  ┌────────┐         │
│  │ Pod 1  │  │ Pod 2  │  │ Pod 3  │  │ Pod N  │         │
│  │ stdout │  │ stdout │  │ stdout │  │ stdout │         │
│  │ stderr │  │ stderr │  │ stderr │  │ stderr │         │
│  └───┬────┘  └───┬────┘  └───┬────┘  └───┬────┘         │
│      │           │           │           │              │
│      └───────────┴───────────┴───────────┘              │
│                  ↓                                      │
│        ┌──────────────────────┐                         │
│        │   Docker/containerd  │                         │
│        │   (stocke logs)      │                         │
│        └─────────┬────────────┘                         │
│                  ↓                                      │
│        ┌──────────────────────┐                         │
│        │  Fluentd DaemonSet   │ ← Un pod par nœud       │
│        │  (lit les logs)      │                         │
│        └─────────┬────────────┘                         │
└──────────────────┼──────────────────────────────────────┘
                   ↓
         ┌─────────────────────┐
         │   Elasticsearch     │ ← Stockage et indexation
         │   StatefulSet       │
         └─────────┬───────────┘
                   ↓
         ┌─────────────────────┐
         │      Kibana         │ ← Interface utilisateur
         │      Deployment     │
         └─────────────────────┘
```

### Flux de Données

1. **Application** écrit des logs sur `stdout/stderr`
2. **Container runtime** (Docker/containerd) capture et stocke dans des fichiers
3. **Fluentd** (DaemonSet) lit ces fichiers de logs
4. **Fluentd** transforme et enrichit les logs (ajout namespace, pod, labels)
5. **Fluentd** envoie vers **Elasticsearch**
6. **Elasticsearch** indexe et stocke les logs
7. **Kibana** permet de rechercher et visualiser

### Pourquoi un DaemonSet pour Fluentd ?

Un **DaemonSet** garantit qu'un pod Fluentd tourne sur **chaque nœud** du cluster :
```
Nœud 1                      Nœud 2                    Nœud 3
┌─────────────┐            ┌─────────────┐          ┌─────────────┐
│  App Pods   │            │  App Pods   │          │  App Pods   │
├─────────────┤            ├─────────────┤          ├─────────────┤
│ Fluentd Pod │ ← Lit      │ Fluentd Pod │← Lit     │ Fluentd Pod │← Lit
│  (DaemonSet)│   logs     │  (DaemonSet)│  logs    │  (DaemonSet)│  logs
└─────────────┘            └─────────────┘          └─────────────┘
       │                          │                        │
       └──────────────────────────┴────────────────────────┘
                                  ↓
                          Elasticsearch
```

Chaque Fluentd collecte les logs des pods sur son nœud.

## Les Composants en Détail

### 1. Elasticsearch - Le Stockage

#### Qu'est-ce que c'est ?

Elasticsearch est une **base de données de recherche** distribuée et scalable, optimisée pour :
- Stocker de grandes quantités de données textuelles
- Indexer rapidement pour des recherches ultra-rapides
- Recherches full-text avec scoring de pertinence

#### Concepts Clés

**Index** : Un conteneur logique pour vos données (comme une table SQL)
```
Index "logs-2025.10.25" contient tous les logs du 25 octobre 2025
Index "logs-2025.10.24" contient tous les logs du 24 octobre 2025
```

**Document** : Une entrée de log individuelle (comme une ligne SQL)
```json
{
  "timestamp": "2025-10-25T14:32:15Z",
  "level": "ERROR",
  "message": "Payment failed: timeout",
  "kubernetes": {
    "namespace": "production",
    "pod": "payment-service-abc123",
    "container": "payment"
  }
}
```

**Mapping** : Le schéma des champs (types de données)
```json
{
  "timestamp": "date",
  "level": "keyword",
  "message": "text",
  "kubernetes.namespace": "keyword"
}
```

#### Caractéristiques

- **Quasi temps-réel** : Les logs sont recherchables en ~1 seconde
- **Distribué** : Peut s'étendre sur plusieurs nœuds
- **RESTful** : API HTTP pour tout
- **Gourmand en ressources** : Nécessite mémoire et CPU
- **Gestion des index** : Rotation automatique par jour/semaine

### 2. Fluentd - Le Collecteur

#### Qu'est-ce que c'est ?

Fluentd est un **collecteur de logs unifié** qui :
- Lit les logs depuis diverses sources
- Les transforme et les enrichit
- Les envoie vers diverses destinations

#### Architecture de Fluentd

```
┌────────────────────────────────────┐
│            Fluentd                 │
│                                    │
│  ┌────────────────────────────┐    │
│  │       INPUT plugins        │    │
│  │  - tail (fichiers)         │    │
│  │  - forward (réseau)        │    │
│  │  - http (API)              │    │
│  └────────────┬───────────────┘    │
│               ↓                    │
│  ┌────────────────────────────┐    │
│  │      FILTER plugins        │    │
│  │  - parser (JSON, regex)    │    │
│  │  - record_transformer      │    │
│  │  - kubernetes_metadata     │    │
│  └────────────┬───────────────┘    │
│               ↓                    │
│  ┌────────────────────────────┐    │
│  │      OUTPUT plugins        │    │
│  │  - elasticsearch           │    │
│  │  - file                    │    │
│  │  - s3, kafka, etc.         │    │
│  └────────────────────────────┘    │
└────────────────────────────────────┘
```

#### Plugin Kubernetes Metadata

C'est le plugin **star** pour Kubernetes. Il enrichit chaque log avec des métadonnées :

**Log brut du container** :
```
2025-10-25 14:32:15 ERROR Payment failed
```

**Après enrichissement Fluentd** :
```json
{
  "log": "2025-10-25 14:32:15 ERROR Payment failed",
  "stream": "stderr",
  "time": "2025-10-25T14:32:15Z",
  "kubernetes": {
    "namespace_name": "production",
    "pod_name": "payment-service-7d8c9-x4b2k",
    "container_name": "payment",
    "labels": {
      "app": "payment-service",
      "version": "v1.2.3"
    }
  }
}
```

Maintenant vous pouvez filtrer par namespace, pod, labels, etc. !

#### Configuration Fluentd

Exemple de configuration (format YAML simplifié) :

```xml
<source>
  @type tail
  path /var/log/containers/*.log
  pos_file /var/log/fluentd-containers.log.pos
  tag kubernetes.*
  read_from_head true
  <parse>
    @type json
    time_format %Y-%m-%dT%H:%M:%S.%NZ
  </parse>
</source>

<filter kubernetes.**>
  @type kubernetes_metadata
  @id filter_kube_metadata
</filter>

<match **>
  @type elasticsearch
  host elasticsearch.logging.svc.cluster.local
  port 9200
  logstash_format true
  logstash_prefix logs
  <buffer>
    @type file
    path /var/log/fluentd-buffers/kubernetes.system.buffer
    flush_mode interval
    flush_interval 5s
  </buffer>
</match>
```

**Explication** :
- `<source>` : Lit les logs des containers dans `/var/log/containers/`
- `<filter>` : Enrichit avec les métadonnées Kubernetes
- `<match>` : Envoie vers Elasticsearch avec format Logstash

### 3. Kibana - L'Interface de Visualisation

#### Qu'est-ce que c'est ?

Kibana est l'**interface web** pour Elasticsearch. C'est votre outil principal pour :
- Rechercher dans les logs
- Créer des dashboards
- Analyser les patterns
- Configurer des alertes

#### Fonctionnalités Principales

**Discover** : Recherche et exploration des logs
- Interface de recherche principale
- Filtres et requêtes
- Analyse des champs

**Dashboard** : Tableaux de bord personnalisés
- Graphiques de volumes de logs
- Top erreurs
- Timeline des événements

**Visualize** : Création de visualisations
- Graphiques, tableaux, cartes
- Agrégations de données

**Dev Tools** : Console pour requêtes directes
- Requêtes Elasticsearch brutes
- Débogage

## Déploiement dans MicroK8s

### Prérequis

Avant de déployer la stack EFK :

```bash
# Vérifier les ressources disponibles
microk8s kubectl top nodes

# Recommandations minimales :
# - CPU : 2 cores disponibles
# - RAM : 4 GB disponibles
# - Stockage : 20 GB disponibles
```

### Créer le Namespace

```bash
microk8s kubectl create namespace logging
```

### Architecture de Déploiement

Nous allons déployer :
1. **Elasticsearch** : StatefulSet avec PersistentVolume (pour garder les logs)
2. **Fluentd** : DaemonSet (un pod par nœud)
3. **Kibana** : Deployment (interface web)

### 1. Déployer Elasticsearch

#### ConfigMap pour la Configuration

Fichier `elasticsearch-config.yaml` :

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: elasticsearch-config
  namespace: logging
data:
  elasticsearch.yml: |
    cluster.name: "k8s-logs"
    network.host: 0.0.0.0
    discovery.type: single-node
    xpack.security.enabled: false
    # Pour un lab, désactivation de la sécurité
    # En production, TOUJOURS activer xpack.security !
```

#### Service Elasticsearch

Fichier `elasticsearch-service.yaml` :

```yaml
apiVersion: v1
kind: Service
metadata:
  name: elasticsearch
  namespace: logging
  labels:
    app: elasticsearch
spec:
  selector:
    app: elasticsearch
  ports:
  - name: http
    port: 9200
    targetPort: 9200
  - name: transport
    port: 9300
    targetPort: 9300
  type: ClusterIP
```

#### StatefulSet Elasticsearch

Fichier `elasticsearch-statefulset.yaml` :

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: elasticsearch
  namespace: logging
spec:
  serviceName: elasticsearch
  replicas: 1
  selector:
    matchLabels:
      app: elasticsearch
  template:
    metadata:
      labels:
        app: elasticsearch
    spec:
      containers:
      - name: elasticsearch
        image: docker.elastic.co/elasticsearch/elasticsearch:8.11.0
        resources:
          requests:
            memory: "2Gi"
            cpu: "500m"
          limits:
            memory: "2Gi"
            cpu: "1000m"
        ports:
        - containerPort: 9200
          name: http
        - containerPort: 9300
          name: transport
        env:
        - name: discovery.type
          value: "single-node"
        - name: ES_JAVA_OPTS
          value: "-Xms1g -Xmx1g"
        - name: xpack.security.enabled
          value: "false"
        volumeMounts:
        - name: elasticsearch-data
          mountPath: /usr/share/elasticsearch/data
  volumeClaimTemplates:
  - metadata:
      name: elasticsearch-data
    spec:
      accessModes: [ "ReadWriteOnce" ]
      storageClassName: "microk8s-hostpath"
      resources:
        requests:
          storage: 10Gi
```

**Points importants** :
- **StatefulSet** : Pour persistance et identité stable
- **PVC** : Volume persistant de 10GB pour les logs
- **Memory** : 2GB alloués (1GB heap Java)
- **Single-node** : Simplifié pour un lab

#### Déployer Elasticsearch

```bash
microk8s kubectl apply -f elasticsearch-config.yaml
microk8s kubectl apply -f elasticsearch-service.yaml
microk8s kubectl apply -f elasticsearch-statefulset.yaml

# Attendre que le pod soit prêt (peut prendre 2-3 minutes)
microk8s kubectl wait --for=condition=ready pod -l app=elasticsearch -n logging --timeout=300s

# Vérifier
microk8s kubectl get pods -n logging
microk8s kubectl logs elasticsearch-0 -n logging
```

#### Tester Elasticsearch

```bash
# Port-forward pour tester localement
microk8s kubectl port-forward svc/elasticsearch 9200:9200 -n logging

# Dans un autre terminal
curl http://localhost:9200
# Devrait retourner des infos sur le cluster

curl http://localhost:9200/_cluster/health?pretty
# Devrait montrer "status": "green" ou "yellow"
```

### 2. Déployer Fluentd

#### ConfigMap Fluentd

Fichier `fluentd-config.yaml` :

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: fluentd-config
  namespace: logging
data:
  fluent.conf: |
    # Lecture des logs des containers
    <source>
      @type tail
      @id in_tail_container_logs
      path /var/log/containers/*.log
      pos_file /var/log/fluentd-containers.log.pos
      tag kubernetes.*
      read_from_head true
      <parse>
        @type json
        time_format %Y-%m-%dT%H:%M:%S.%NZ
      </parse>
    </source>

    # Enrichissement avec métadonnées Kubernetes
    <filter kubernetes.**>
      @type kubernetes_metadata
      @id filter_kube_metadata
      kubernetes_url "#{ENV['FLUENT_FILTER_KUBERNETES_URL'] || 'https://' + ENV.fetch('KUBERNETES_SERVICE_HOST') + ':' + ENV.fetch('KUBERNETES_SERVICE_PORT') + '/api'}"
      verify_ssl "#{ENV['KUBERNETES_VERIFY_SSL'] || true}"
      ca_file "#{ENV['KUBERNETES_CA_FILE']}"
    </filter>

    # Parser pour extraire les niveaux de log
    <filter kubernetes.**>
      @type parser
      key_name log
      reserve_data true
      remove_key_name_field true
      <parse>
        @type regexp
        expression /^(?<time>[^ ]+) (?<level>[^ ]+) (?<message>.*)$/
        time_format %Y-%m-%dT%H:%M:%S.%NZ
      </parse>
    </filter>

    # Exclusion des logs de la stack logging elle-même (éviter boucle)
    <filter kubernetes.**>
      @type grep
      <exclude>
        key $.kubernetes.namespace_name
        pattern /^(logging|kube-system)$/
      </exclude>
    </filter>

    # Envoi vers Elasticsearch
    <match **>
      @type elasticsearch
      @id out_es
      @log_level info
      include_tag_key true
      host elasticsearch.logging.svc.cluster.local
      port 9200
      scheme http
      logstash_format true
      logstash_prefix logs
      logstash_dateformat %Y.%m.%d
      <buffer>
        @type file
        path /var/log/fluentd-buffers/kubernetes.system.buffer
        flush_mode interval
        retry_type exponential_backoff
        flush_interval 5s
        retry_forever false
        retry_max_interval 30
        chunk_limit_size 2M
        queue_limit_length 8
        overflow_action block
      </buffer>
    </match>
```

#### ServiceAccount et RBAC pour Fluentd

Fichier `fluentd-rbac.yaml` :

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: fluentd
  namespace: logging
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: fluentd
rules:
- apiGroups:
  - ""
  resources:
  - pods
  - namespaces
  verbs:
  - get
  - list
  - watch
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: fluentd
roleRef:
  kind: ClusterRole
  name: fluentd
  apiGroup: rbac.authorization.k8s.io
subjects:
- kind: ServiceAccount
  name: fluentd
  namespace: logging
```

**Pourquoi ces permissions ?**
Fluentd a besoin de lire les métadonnées des pods et namespaces pour enrichir les logs.

#### DaemonSet Fluentd

Fichier `fluentd-daemonset.yaml` :

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: fluentd
  namespace: logging
  labels:
    app: fluentd
spec:
  selector:
    matchLabels:
      app: fluentd
  template:
    metadata:
      labels:
        app: fluentd
    spec:
      serviceAccountName: fluentd
      containers:
      - name: fluentd
        image: fluent/fluentd-kubernetes-daemonset:v1-debian-elasticsearch
        env:
        - name: FLUENT_ELASTICSEARCH_HOST
          value: "elasticsearch.logging.svc.cluster.local"
        - name: FLUENT_ELASTICSEARCH_PORT
          value: "9200"
        - name: FLUENT_ELASTICSEARCH_SCHEME
          value: "http"
        - name: FLUENTD_SYSTEMD_CONF
          value: disable
        resources:
          requests:
            memory: "200Mi"
            cpu: "100m"
          limits:
            memory: "500Mi"
            cpu: "500m"
        volumeMounts:
        - name: varlog
          mountPath: /var/log
        - name: varlibdockercontainers
          mountPath: /var/lib/docker/containers
          readOnly: true
        - name: fluentd-config
          mountPath: /fluentd/etc/fluent.conf
          subPath: fluent.conf
      volumes:
      - name: varlog
        hostPath:
          path: /var/log
      - name: varlibdockercontainers
        hostPath:
          path: /var/lib/docker/containers
      - name: fluentd-config
        configMap:
          name: fluentd-config
```

**Points importants** :
- **DaemonSet** : Un pod sur chaque nœud
- **Volumes hostPath** : Accès aux logs sur le nœud hôte
- **ServiceAccount** : Permissions pour lire métadonnées K8s

#### Déployer Fluentd

```bash
microk8s kubectl apply -f fluentd-config.yaml
microk8s kubectl apply -f fluentd-rbac.yaml
microk8s kubectl apply -f fluentd-daemonset.yaml

# Vérifier (un pod par nœud)
microk8s kubectl get daemonset -n logging
microk8s kubectl get pods -n logging -l app=fluentd

# Voir les logs Fluentd
microk8s kubectl logs -n logging -l app=fluentd --tail=50
```

### 3. Déployer Kibana

#### Service Kibana

Fichier `kibana-service.yaml` :

```yaml
apiVersion: v1
kind: Service
metadata:
  name: kibana
  namespace: logging
  labels:
    app: kibana
spec:
  selector:
    app: kibana
  ports:
  - port: 5601
    targetPort: 5601
    name: http
  type: ClusterIP
```

#### Deployment Kibana

Fichier `kibana-deployment.yaml` :

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: kibana
  namespace: logging
spec:
  replicas: 1
  selector:
    matchLabels:
      app: kibana
  template:
    metadata:
      labels:
        app: kibana
    spec:
      containers:
      - name: kibana
        image: docker.elastic.co/kibana/kibana:8.11.0
        resources:
          requests:
            memory: "1Gi"
            cpu: "500m"
          limits:
            memory: "2Gi"
            cpu: "1000m"
        env:
        - name: ELASTICSEARCH_HOSTS
          value: "http://elasticsearch.logging.svc.cluster.local:9200"
        - name: SERVER_NAME
          value: "kibana"
        - name: SERVER_HOST
          value: "0.0.0.0"
        ports:
        - containerPort: 5601
          name: http
        readinessProbe:
          httpGet:
            path: /api/status
            port: 5601
          initialDelaySeconds: 60
          periodSeconds: 10
        livenessProbe:
          httpGet:
            path: /api/status
            port: 5601
          initialDelaySeconds: 120
          periodSeconds: 30
```

#### Déployer Kibana

```bash
microk8s kubectl apply -f kibana-service.yaml
microk8s kubectl apply -f kibana-deployment.yaml

# Attendre que Kibana soit prêt (peut prendre 2-3 minutes)
microk8s kubectl wait --for=condition=ready pod -l app=kibana -n logging --timeout=300s

# Vérifier
microk8s kubectl get pods -n logging -l app=kibana
microk8s kubectl logs -n logging -l app=kibana
```

#### Accéder à Kibana

**Option 1 : Port-forward (temporaire)**
```bash
microk8s kubectl port-forward -n logging svc/kibana 5601:5601

# Accédez à http://localhost:5601
```

**Option 2 : Ingress (permanent)**

Fichier `kibana-ingress.yaml` :

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: kibana
  namespace: logging
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  ingressClassName: nginx
  rules:
  - host: kibana.votre-domaine.local
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: kibana
            port:
              number: 5601
```

```bash
microk8s kubectl apply -f kibana-ingress.yaml

# Ajoutez dans /etc/hosts :
# 192.168.1.100  kibana.votre-domaine.local

# Accédez à http://kibana.votre-domaine.local
```

### Vérification de la Stack Complète

```bash
# Vérifier tous les composants
microk8s kubectl get all -n logging

# Devrait afficher :
# - elasticsearch-0 (StatefulSet)
# - fluentd-xxxxx (DaemonSet, un par nœud)
# - kibana-xxxxx (Deployment)
```

## Utiliser Kibana pour Explorer les Logs

### Première Connexion

1. Ouvrez Kibana dans votre navigateur
2. Première page : Configuration de l'index pattern

### Créer un Index Pattern

Un **index pattern** définit quels index Elasticsearch vous voulez explorer.

**Étapes** :
1. Dans le menu, allez à **Stack Management** → **Index Patterns**
2. Cliquez sur **Create index pattern**
3. Entrez le pattern : `logs-*`
   - Cela correspondra à `logs-2025.10.25`, `logs-2025.10.26`, etc.
4. Cliquez **Next step**
5. Sélectionnez le champ de temps : `@timestamp`
6. Cliquez **Create index pattern**

### Discover : Explorer les Logs

1. Dans le menu, allez à **Discover**
2. Sélectionnez votre index pattern `logs-*`
3. Vous voyez maintenant vos logs ! 🎉

**Interface Discover** :
```
┌─────────────────────────────────────────────────────┐
│  [Recherche]                           [15 min ▼]   │ ← Barre de recherche et time range
├─────────────────────────────────────────────────────┤
│  ▁▂▄█▇▅▃▂▁                                          │ ← Histogramme temporel
├──────────────┬──────────────────────────────────────┤
│  Champs      │  Logs                                │
│              │  ──────────────────────────────────  │
│  kubernetes  │  Oct 25 14:32:15  ERROR Payment...   │
│  level       │  Oct 25 14:32:14  INFO  User lo...   │
│  message     │  Oct 25 14:32:13  WARN  Slow qu...   │
│  pod_name    │  Oct 25 14:32:12  DEBUG Request...   │
│              │                                      │
└──────────────┴──────────────────────────────────────┘
```

### Recherches Basiques

#### Recherche Full-Text

Dans la barre de recherche :
```
payment failed
```
Trouve tous les logs contenant "payment" ET "failed"

#### Recherche par Champ

```
level: ERROR
```
Trouve tous les logs avec level = ERROR

```
kubernetes.namespace_name: production
```
Trouve tous les logs du namespace production

#### Combiner des Critères

```
level: ERROR AND kubernetes.namespace_name: production
```
Logs d'erreur dans production

```
level: (ERROR OR WARN) AND message: database
```
Erreurs ou warnings contenant "database"

### Filtres Visuels

Au lieu de taper les requêtes, vous pouvez cliquer sur les valeurs dans les logs :

1. Cliquez sur un champ dans la liste (ex: `level`)
2. Survolez une valeur (ex: `ERROR`)
3. Cliquez sur la **loupe +** pour filtrer
4. Cliquez sur la **loupe -** pour exclure

### Time Range (Plage Temporelle)

En haut à droite, sélectionnez la période :
- Last 15 minutes
- Last 1 hour
- Last 24 hours
- Last 7 days
- Custom (période personnalisée)

### Champs Utiles

Affichez les champs pertinents en cliquant dessus dans la liste :

**Champs recommandés** :
- `@timestamp` : Date/heure
- `level` : Niveau de log (INFO, WARN, ERROR)
- `message` ou `log` : Message du log
- `kubernetes.namespace_name` : Namespace
- `kubernetes.pod_name` : Nom du pod
- `kubernetes.container_name` : Nom du container
- `kubernetes.labels.*` : Labels du pod

### Sauvegarder des Recherches

Une fois que vous avez une recherche utile :

1. Cliquez sur **Save** en haut
2. Donnez un nom : "Erreurs Production"
3. Cliquez **Save**

Pour la retrouver :
- Menu **Discover** → **Open** → Sélectionnez votre recherche

## Exemples de Requêtes Utiles

### 1. Toutes les Erreurs

```
level: ERROR
```

### 2. Erreurs dans un Namespace Spécifique

```
level: ERROR AND kubernetes.namespace_name: production
```

### 3. Logs d'un Pod Spécifique

```
kubernetes.pod_name: payment-service-7d8c9-x4b2k
```

### 4. Logs d'une Application (par label)

```
kubernetes.labels.app: payment-service
```

### 5. Recherche dans le Message

```
message: "connection timeout" OR message: "database error"
```

### 6. Exclure les Logs INFO (seulement WARN et ERROR)

```
NOT level: INFO
```

### 7. Logs avec une Durée > 1 Seconde

Si vous parsez la durée dans un champ `duration` :
```
duration: >1000
```

### 8. Requête Complexe

```
(level: ERROR OR level: WARN)
AND kubernetes.namespace_name: production
AND (message: timeout OR message: failed OR message: error)
NOT kubernetes.labels.app: monitoring
```

Traduit en langage naturel :
- Erreurs ou warnings
- Dans le namespace production
- Contenant "timeout", "failed", ou "error"
- Mais pas de l'application monitoring

## Patterns de Logs Courants

### Pattern 1 : Debugging d'une Erreur Utilisateur

**Scénario** : Un utilisateur signale une erreur à 14h32.

**Démarche** :
1. Time range : Autour de 14h32 (ex: 14h25 - 14h35)
2. Recherche : `level: ERROR`
3. Affinez : Ajoutez des mots-clés du rapport utilisateur
4. Trouvez le pod concerné
5. Explorez les logs de ce pod avant et après l'erreur

### Pattern 2 : Analyse d'un Incident

**Scénario** : Le site était lent entre 10h et 11h.

**Démarche** :
1. Time range : 10h - 11h
2. Recherche : `message: slow OR message: timeout`
3. Groupez par pod : Identifiez quel service avait des problèmes
4. Corrélation : Regardez si d'autres services étaient impactés

### Pattern 3 : Suivi d'une Requête

Si vous avez un **request ID** dans vos logs :

```
request_id: "abc-123-xyz"
```

Cela affiche tous les logs de cette requête à travers tous les services.

**Exemple de logs** :
```
14:32:15 frontend  INFO  Request abc-123-xyz received
14:32:16 auth      INFO  Request abc-123-xyz authenticated user 456
14:32:17 api       INFO  Request abc-123-xyz processing order
14:32:18 payment   ERROR Request abc-123-xyz payment timeout
```

Histoire complète de la requête ! 🎯

### Pattern 4 : Top Erreurs

Pour voir les erreurs les plus fréquentes :

1. Recherche : `level: ERROR`
2. Dans la liste des champs, cliquez sur `message`
3. Cliquez sur **Visualize** → **Top values**
4. Vous voyez les messages d'erreur les plus fréquents

## Créer des Dashboards

### Dashboard d'Overview

Créez un dashboard pour avoir une vue d'ensemble :

1. Menu **Dashboard** → **Create dashboard**
2. Ajoutez des panneaux :

**Panneau 1 : Volume de Logs**
- Type : Line chart
- Métrique : Count
- Buckets : Date histogram sur @timestamp

**Panneau 2 : Répartition par Niveau**
- Type : Pie chart
- Buckets : Terms sur `level`

**Panneau 3 : Top 10 Pods avec le Plus d'Erreurs**
- Type : Table
- Filter : `level: ERROR`
- Buckets : Terms sur `kubernetes.pod_name`
- Métrique : Count

**Panneau 4 : Erreurs par Namespace**
- Type : Bar chart
- Filter : `level: ERROR`
- Buckets : Terms sur `kubernetes.namespace_name`

3. Sauvegardez le dashboard : "Logs Overview"

### Dashboard par Application

Créez un dashboard spécifique pour chaque application importante :

1. Filtre global : `kubernetes.labels.app: payment-service`
2. Ajoutez des panneaux pertinents :
   - Volume de logs
   - Taux d'erreur
   - Top erreurs
   - Latence (si parsée)

## Gestion des Index et Rétention

### Rotation des Index

Par défaut, Fluentd crée un index par jour :
```
logs-2025.10.25
logs-2025.10.26
logs-2025.10.27
```

### Politique de Rétention

Les logs prennent de l'espace. Il faut définir combien de temps les garder.

#### Index Lifecycle Management (ILM)

Dans Kibana :
1. **Stack Management** → **Index Lifecycle Policies**
2. **Create policy**
3. Définissez les phases :
   - **Hot** : Index actif (0-1 jour)
   - **Warm** : Lectures occasionnelles (1-7 jours)
   - **Cold** : Archives (7-30 jours)
   - **Delete** : Suppression après X jours

**Exemple de politique simple** :
```json
{
  "policy": {
    "phases": {
      "hot": {
        "actions": {
          "rollover": {
            "max_age": "1d",
            "max_size": "50gb"
          }
        }
      },
      "delete": {
        "min_age": "7d",
        "actions": {
          "delete": {}
        }
      }
    }
  }
}
```

Cette politique :
- Crée un nouvel index chaque jour ou quand l'index atteint 50GB
- Supprime les index de plus de 7 jours

### Suppression Manuelle

Pour supprimer un index :

**Via Kibana** :
1. **Stack Management** → **Index Management**
2. Sélectionnez l'index
3. **Manage** → **Delete index**

**Via API** :
```bash
curl -X DELETE "localhost:9200/logs-2025.10.01"
```

### Monitoring de l'Espace Disque

```bash
# Voir l'utilisation du stockage Elasticsearch
microk8s kubectl exec -n logging elasticsearch-0 -- df -h /usr/share/elasticsearch/data

# Voir la taille des index
curl -X GET "localhost:9200/_cat/indices?v&h=index,store.size&s=store.size:desc"
```

## Bonnes Pratiques

### 1. Structure de Logs

**Utilisez JSON** dans vos applications :

```json
{
  "timestamp": "2025-10-25T14:32:15Z",
  "level": "ERROR",
  "message": "Payment failed",
  "context": {
    "user_id": "12345",
    "order_id": "98765",
    "error_code": "TIMEOUT"
  }
}
```

Avantages :
- ✅ Facile à parser
- ✅ Champs structurés automatiquement
- ✅ Recherches précises possibles

### 2. Niveaux de Log Cohérents

Utilisez des niveaux standards :
- **TRACE** : Détails très verbeux (rarement utilisé en prod)
- **DEBUG** : Informations de débogage
- **INFO** : Informations générales
- **WARN** : Avertissements, situations anormales mais gérables
- **ERROR** : Erreurs nécessitant attention
- **FATAL** : Erreurs critiques causant l'arrêt

### 3. Contexte Riche

Incluez toujours du contexte :

```
❌ "Error"
✅ "Payment processing error: timeout after 30s for order #12345, user 456"

❌ "Request failed"
✅ "API request failed: POST /api/orders, status 500, duration 3.2s, request_id abc-123"
```

### 4. Request ID / Trace ID

Générez un ID unique par requête et propagez-le :

```python
import uuid

@app.before_request
def before_request():
    request.request_id = request.headers.get('X-Request-ID', str(uuid.uuid4()))

@app.route('/api/order')
def create_order():
    logger.info(f"Processing order", extra={'request_id': request.request_id})
```

Permet de suivre une requête à travers tous les services !

### 5. Éviter les Logs Sensibles

**Ne loguez JAMAIS** :
- ❌ Mots de passe
- ❌ Tokens d'authentification
- ❌ Numéros de carte bancaire
- ❌ Données personnelles sensibles (RGPD)

**Masquez les données sensibles** :
```
✅ "User john.doe@example.com logged in"
✅ "Payment with card ending in 1234"
```

### 6. Log Sampling pour Haute Volumétrie

Si vous avez énormément de logs (ex: 1000 req/s) :

```python
import random

if random.random() < 0.1:  # Log 10% des requêtes
    logger.debug(f"Request details: {request}")
```

Ou loguez seulement les requêtes lentes :
```python
if duration > 1.0:  # Plus de 1 seconde
    logger.warn(f"Slow request: {duration}s")
```

### 7. Corrélation avec les Métriques

Ajoutez les mêmes labels dans logs et métriques :

```python
# Dans le log
logger.error("Payment failed", extra={
    'order_id': '12345',
    'payment_method': 'credit_card'
})

# Dans la métrique
payment_failures.labels(payment_method='credit_card').inc()
```

Permet de corréler facilement !

## Dépannage

### Problème : Aucun Log dans Kibana

**Vérifications** :

1. **Elasticsearch est-il accessible ?**
   ```bash
   kubectl exec -n logging -it fluentd-xxxxx -- curl http://elasticsearch.logging.svc.cluster.local:9200/_cluster/health
   ```

2. **Fluentd envoie-t-il des logs ?**
   ```bash
   kubectl logs -n logging fluentd-xxxxx | grep -i error
   ```

3. **Y a-t-il des index dans Elasticsearch ?**
   ```bash
   curl http://localhost:9200/_cat/indices?v
   ```
   Devrait montrer `logs-2025.10.25`

4. **L'index pattern est-il correct dans Kibana ?**
   - Vérifiez que le pattern `logs-*` existe
   - Vérifiez que des documents sont présents

### Problème : Fluentd en CrashLoop

**Causes possibles** :
- Configuration invalide
- Permissions insuffisantes
- Elasticsearch inaccessible

**Débogage** :
```bash
kubectl logs -n logging fluentd-xxxxx
# Cherchez les erreurs de config ou de connexion
```

### Problème : Elasticsearch Out of Memory

**Symptôme** : Pod Elasticsearch redémarre fréquemment

**Solutions** :
1. Augmentez la mémoire allouée dans le StatefulSet
2. Réduisez la heap size Java si nécessaire
3. Activez la politique de rétention pour supprimer les vieux index

### Problème : Kibana Lent

**Causes** :
- Trop de données (index trop gros)
- Requêtes non optimisées
- Ressources insuffisantes

**Solutions** :
1. Utilisez des time ranges plus courts
2. Ajoutez des filtres pour réduire les données
3. Augmentez les ressources Kibana

## Ce Qu'il Faut Retenir

📚 **La stack EFK centralise tous vos logs** :
- **E**lasticsearch : Stocke et indexe
- **F**luentd : Collecte et enrichit
- **K**ibana : Visualise et recherche

🔄 **Architecture distribuée** :
- Fluentd en DaemonSet sur chaque nœud
- Elasticsearch en StatefulSet avec persistance
- Kibana en Deployment pour l'interface

🔍 **Kibana Discover** :
- Recherche full-text et par champs
- Filtres visuels et temporels
- Sauvegarde de recherches

📊 **Dashboards** :
- Créez des vues d'ensemble
- Surveillez les patterns
- Partagez avec l'équipe

⚙️ **Gestion des index** :
- Rotation automatique par jour
- Politique de rétention (ILM)
- Nettoyage régulier nécessaire

✅ **Bonnes pratiques** :
- Logs structurés en JSON
- Niveaux cohérents
- Contexte riche avec request ID
- Protection des données sensibles

🎯 **Prochaine étape** :
- Section 15.4 : Corrélation logs-métriques avec Loki (alternative plus légère)
- Section 15.5 : Tracing distribué avec Jaeger

Avec vos logs centralisés, vous avez maintenant une visibilité complète sur ce qui se passe dans votre cluster MicroK8s ! 🚀

⏭️ [Correlation logs-métriques avec Loki](/15-observabilite-avancee/04-correlation-logs-metriques-avec-loki.md)
