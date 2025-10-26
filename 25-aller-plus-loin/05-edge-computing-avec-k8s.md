🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 25.5 Edge computing avec K8s

## Introduction : Qu'est-ce que l'Edge Computing ?

### L'analogie du restaurant

Pour comprendre l'edge computing, imaginez cette analogie :

**Modèle Cloud (centralisé)** :
```
Vous êtes dans un restaurant
   ↓
Vous commandez un plat
   ↓
La cuisine centrale prépare (à 500 km de là)
   ↓
Le plat est livré par avion
   ↓
Vous mangez (2 heures plus tard)
```

**Modèle Edge (distribué)** :
```
Vous êtes dans un restaurant
   ↓
Vous commandez un plat
   ↓
La cuisine locale prépare (dans le restaurant)
   ↓
Le plat arrive sur votre table
   ↓
Vous mangez (5 minutes plus tard)
```

L'edge computing, c'est **rapprocher la puissance de calcul de là où les données sont générées et consommées**.

### Définition formelle

L'**Edge Computing** (informatique en périphérie) est un paradigme où le traitement des données se fait :
- **Au plus près de la source** de génération des données
- **Localement**, plutôt que dans un datacenter distant
- **En temps réel** ou quasi-temps réel

```
Cloud Computing :        Edge Computing :

[Capteurs]              [Capteurs]
    ↓                       ↓
[Internet]              [Edge Device]
    ↓                       ↓
[Datacenter Cloud]      [Traitement local]
    ↓                       ↓
[Traitement]            [Action immédiate]
    ↓
[Résultat]
```

### Pourquoi l'Edge Computing ?

#### 1. Latence réduite

**Problème Cloud** : Les données font des allers-retours vers le datacenter.
```
Caméra de sécurité → Cloud (500 ms) → Analyse → Alerte (500 ms) → Total : 1 seconde
```

**Solution Edge** : Le traitement se fait localement.
```
Caméra de sécurité → Edge Device (10 ms) → Analyse → Alerte (10 ms) → Total : 20 ms
```

Pour une voiture autonome, 1 seconde = 30 mètres parcourus à 100 km/h !

#### 2. Bande passante économisée

**Sans Edge** :
- 100 caméras × 1080p × 24h/jour = **téraoctets** de données vers le cloud
- Coûts de transfert importants

**Avec Edge** :
- Traitement local des vidéos
- Envoi uniquement des alertes et statistiques au cloud
- Économie de **99% de bande passante**

#### 3. Fonctionnement offline

Si la connexion internet est coupée :
- **Cloud** : Système inopérant ❌
- **Edge** : Continue de fonctionner localement ✅

#### 4. Conformité et vie privée

Données sensibles (médicales, vidéosurveillance) :
- **Cloud** : Transit sur Internet, stockage distant
- **Edge** : Données traitées localement, ne quittent jamais le site

## Le spectre Cloud → Edge → Device

L'informatique moderne n'est pas binaire (cloud OU edge), c'est un **continuum** :

```
┌──────────────────────────────────────────────────────────────────┐
│                                                                  │
│  [Cloud]        [Regional Edge]      [Local Edge]      [Device]  │
│  Datacenter     Mini-datacenter      Box locale        Capteur   │
│                                                                  │
│  Latence:       Latence:             Latence:          Latence:  │
│  100-500 ms     20-50 ms             5-20 ms           <5 ms     │
│                                                                  │
│  Puissance:     Puissance:           Puissance:        Puissance:│
│  Très haute     Haute                Moyenne           Faible    │
│                                                                  │
│  Exemples:      Exemples:            Exemples:         Exemples: │
│  AWS            Cloudflare           Raspberry Pi      ESP32     │
│  Azure          Edge locations       NVIDIA Jetson     Sensors   │
│  GCP            5G MEC               Intel NUC         Cameras   │
│                                                                  │
└──────────────────────────────────────────────────────────────────┘
```

### Où Kubernetes intervient-il ?

Kubernetes (et ses variantes légères) peut orchestrer des applications sur :
- **Regional Edge** : Petits datacenters en bordure de réseau
- **Local Edge** : Serveurs dans les usines, magasins, véhicules

## Cas d'usage de l'Edge Computing

### 1. Industrie 4.0 (Smart Manufacturing)

**Scénario** : Une usine avec 1000 capteurs IoT sur les machines.

**Sans Edge** :
- Tous les capteurs envoient leurs données au cloud
- Analyse centralisée
- Latence : 500 ms
- Problème : En cas de défaillance machine, la détection arrive trop tard

**Avec Edge** :
- Kubernetes en edge dans l'usine
- Analyse en temps réel des données capteurs
- Latence : 10 ms
- Avantage : Arrêt automatique de la machine défaillante en <1 seconde

```yaml
# Application edge de monitoring industriel
apiVersion: apps/v1
kind: Deployment
metadata:
  name: machine-monitoring
spec:
  replicas: 1
  selector:
    matchLabels:
      app: machine-monitor
  template:
    spec:
      containers:
      - name: monitor
        image: factory/machine-monitor:v1
        env:
        - name: SENSOR_ENDPOINT
          value: "http://localhost:9090/metrics"
        - name: ALERT_THRESHOLD
          value: "85"  # Température critique
        resources:
          requests:
            memory: "256Mi"
            cpu: "500m"
```

### 2. Retail (Commerce de détail)

**Scénario** : Chaîne de 500 magasins avec analyse vidéo.

**Architecture Edge** :
```
Chaque magasin :
├── 20 caméras
├── 1 serveur edge avec Kubernetes
└── Applications :
    ├── Comptage de personnes
    ├── Détection de vol
    ├── Analyse de parcours clients
    └── Gestion des stocks en temps réel

Cloud centralisé :
└── Agrégation des statistiques
    └── Business Intelligence
```

**Bénéfices** :
- Analyse instantanée (alerte vol en temps réel)
- 99% de réduction de bande passante
- Respect RGPD (vidéos ne quittent pas le magasin)

### 3. Véhicules connectés

**Scénario** : Flottes de véhicules autonomes ou semi-autonomes.

```
Chaque véhicule :
├── Kubernetes léger (K3s)
├── Applications :
│   ├── Traitement des images (caméras)
│   ├── Fusion de capteurs (LIDAR, radar)
│   ├── Prise de décision locale
│   └── Maintenance prédictive
└── Synchro cloud quand WiFi disponible
```

**Pourquoi Edge ?** : Une voiture ne peut pas dépendre d'une connexion 4G/5G pour freiner !

### 4. Santé connectée

**Scénario** : Hôpital avec équipements médicaux connectés.

```
Edge cluster dans l'hôpital :
├── Monitoring patient temps réel
├── Analyse ECG locale
├── Alertes critiques instantanées
├── Données sensibles restent sur site
└── Anonymisation avant envoi au cloud
```

**Conformité** : Données médicales traitées localement (HIPAA, RGPD).

### 5. Télécommunications (5G Edge)

**Scénario** : Multi-access Edge Computing (MEC) dans les antennes 5G.

```
Antenne 5G :
├── Edge computing node
├── Applications à ultra-faible latence :
│   ├── Gaming cloud (<10 ms)
│   ├── AR/VR streaming
│   ├── Vidéo 4K/8K
│   └── Conduite assistée
```

**Avantage** : Latence réduite de 100 ms (cloud) à 5-10 ms (edge).

### 6. Agriculture de précision

**Scénario** : Ferme connectée avec capteurs et drones.

```
Edge device (Raspberry Pi + K3s) :
├── Collecte données capteurs (humidité, température)
├── Traitement images drones
├── Décision d'irrigation automatique
├── Détection précoce de maladies
└── Fonctionnement même sans Internet
```

### 7. Smart Cities

**Scénario** : Ville intelligente avec infrastructure connectée.

```
Edge nodes par quartier :
├── Gestion du trafic (feux intelligents)
├── Éclairage public adaptatif
├── Détection d'incidents
├── Qualité de l'air en temps réel
└── Agrégation vers le cloud municipal
```

## Défis spécifiques de l'Edge

### 1. Ressources limitées

Les edge devices ont moins de puissance qu'un datacenter cloud :

```
Cloud Server :          Edge Device :
- 64 GB RAM            - 4-8 GB RAM
- 32 CPU cores         - 4 CPU cores
- SSD illimité         - 256 GB SSD
- Réseau 10 Gbps       - Réseau 100 Mbps
```

**Conséquences** :
- Kubernetes standard est trop lourd
- Besoin de distributions légères (K3s, MicroK8s)
- Applications doivent être optimisées

### 2. Connectivité intermittente

Les edge devices peuvent perdre la connexion :

```
Edge Device en mer sur un bateau :
├── 80% du temps : Pas de connexion satellite
├── 15% du temps : Connexion satellite lente
└── 5% du temps : Connexion au port

→ Doit fonctionner en mode autonome !
```

### 3. Gestion à grande échelle

Gérer 1000 edge locations vs 3 datacenters cloud :

```
Datacenter Cloud :
- 3 clusters à gérer
- Accès SSH facile
- Équipe IT sur place

Edge :
- 1000 edge devices à gérer
- Accès physique difficile
- Souvent aucun IT sur place
- Mises à jour OTA (Over The Air) obligatoires
```

### 4. Hétérogénéité du matériel

```
Edge Device 1 : Raspberry Pi 4 (ARM)
Edge Device 2 : Intel NUC (x86)
Edge Device 3 : NVIDIA Jetson (ARM + GPU)
Edge Device 4 : PC industriel (x86)

→ Images Docker multi-arch nécessaires
```

### 5. Sécurité physique

Un edge device peut être physiquement accessible :
- Vol du device
- Accès USB direct
- Redémarrage avec autre OS

**Mitigations** :
- Chiffrement du disque
- Secure boot
- TPM (Trusted Platform Module)
- Détection de tampering

### 6. Conditions environnementales

Edge devices opèrent dans des environnements variés :
```
- Usine : Chaleur, poussière, vibrations
- Extérieur : Froid, humidité, température extrême
- Véhicule : Vibrations, variations de tension
```

**Solutions** :
- Matériel industriel (ruggedized)
- Refroidissement passif
- Watchdog matériel

## Distributions Kubernetes pour l'Edge

### 1. K3s : Le champion de l'Edge

**K3s** (prononcé "K-three-s") est une distribution Kubernetes ultra-légère par Rancher.

#### Caractéristiques

- **Binaire unique** : ~70 MB (vs 1+ GB pour Kubernetes standard)
- **Faible empreinte** : ~512 MB RAM minimum
- **SQLite par défaut** : Pas besoin d'etcd
- **Architecture ARM** : Parfait pour Raspberry Pi
- **Installation en 30 secondes** : `curl -sfL https://get.k3s.io | sh -`

#### Ce qui est retiré de K3s vs Kubernetes

- Drivers cloud propriétaires (AWS, Azure, GCP)
- Composants alpha/beta non essentiels
- Plugins de stockage legacy

#### Ce qui est inclus par défaut

- ✅ Traefik Ingress Controller
- ✅ ServiceLB (simple load balancer)
- ✅ Local Path Provisioner (stockage)
- ✅ CoreDNS
- ✅ Metrics Server

#### Installation de K3s

```bash
# Sur un Raspberry Pi, Ubuntu, ou n'importe quel Linux

# Installation simple (single node)
curl -sfL https://get.k3s.io | sh -

# Vérifier
sudo k3s kubectl get nodes

# Alias pour kubectl
alias kubectl='sudo k3s kubectl'

# Installation sans Traefik (si vous voulez NGINX)
curl -sfL https://get.k3s.io | sh -s - --disable traefik

# Installation en mode agent (pour un cluster)
curl -sfL https://get.k3s.io | K3S_URL=https://server:6443 K3S_TOKEN=xxx sh -
```

#### K3s en mode High Availability

```bash
# Server 1
curl -sfL https://get.k3s.io | sh -s - server \
  --cluster-init \
  --tls-san=k3s.example.com

# Server 2 et 3
curl -sfL https://get.k3s.io | sh -s - server \
  --server https://server1:6443 \
  --token=xxx
```

### 2. MicroK8s : Simplicité et addons

**MicroK8s** (que vous connaissez déjà !) est également excellent pour l'edge :

**Avantages pour l'Edge** :
- Installation via snap (simple)
- Addons activables en un clic
- Mises à jour automatiques
- Fonctionne sur ARM (Raspberry Pi)

**Installation sur Raspberry Pi** :

```bash
sudo snap install microk8s --classic --channel=1.28/stable

# Sur Raspberry Pi, limiter les ressources
microk8s enable dns storage

# Désactiver les addons non essentiels pour économiser RAM
```

### 3. KubeEdge : Kubernetes natif pour l'Edge

**KubeEdge** est un projet CNCF spécifiquement conçu pour l'edge computing.

#### Architecture KubeEdge

```
Cloud (Hub) :
┌────────────────────┐
│  Kubernetes        │
│  + CloudCore       │ ← Composant KubeEdge
└────────┬───────────┘
         │
         │ Connexion Internet (peut être intermittente)
         │
Edge :   ↓
┌────────────────────┐
│  EdgeCore          │ ← Agent léger KubeEdge
│  + Edge Runtime    │
│  + Pods            │
└────────────────────┘
```

#### Fonctionnalités uniques

- **Mode autonome** : Edge fonctionne même si la connexion cloud est coupée
- **Device management** : Gestion native de devices IoT (MQTT, Modbus)
- **Edge mesh** : Communication entre edge nodes sans passer par le cloud
- **Bandwidth optimization** : Métadonnées synchronisées, pas de verbosité

#### Installation KubeEdge

```bash
# Sur le cloud cluster (hub)
kubectl apply -f https://raw.githubusercontent.com/kubeedge/kubeedge/master/build/crds/devices/devices_v1alpha2_device.yaml
keadm init --advertise-address=<cloud-ip> --kubeedge-version=v1.15.0

# Sur l'edge node
keadm join --cloudcore-ipport=<cloud-ip>:10000 --token=<token>
```

### 4. K0s : Zero Friction Kubernetes

**K0s** (prononcé "k-zero-s") est une autre distribution légère.

```bash
# Installation
curl -sSLf https://get.k0s.sh | sudo sh

# Démarrer
sudo k0s install controller --single

# kubectl intégré
sudo k0s kubectl get nodes
```

**Caractéristiques** :
- Binaire unique (~300 MB)
- Zero dépendances (tout inclus)
- Support multi-plateforme
- Bon pour l'edge et pour le lab

### 5. Comparaison des distributions Edge

| Critère | K3s | MicroK8s | KubeEdge | K0s |
|---------|-----|----------|----------|-----|
| **Taille** | 70 MB | 200 MB | Core: 50 MB | 300 MB |
| **RAM min** | 512 MB | 512 MB | 256 MB | 512 MB |
| **Installation** | cURL | Snap | keadm | cURL |
| **ARM support** | ✅ Excellent | ✅ Bon | ✅ Excellent | ✅ Bon |
| **Offline** | ⚠️ Limité | ⚠️ Limité | ✅ Natif | ⚠️ Limité |
| **IoT devices** | ❌ | ❌ | ✅ Natif | ❌ |
| **HA natif** | ✅ Oui | ✅ Oui | ⚠️ Complexe | ✅ Oui |
| **Maturité** | Très mature | Mature | Jeune | Jeune |
| **Communauté** | Grande | Moyenne | Moyenne | Petite |
| **Idéal pour** | Edge général | Lab/Dev | IoT Edge | Lab/Edge |

## Architecture Edge avec Kubernetes

### Modèle Hub and Spoke

L'architecture la plus commune pour l'edge :

```
                    ┌─────────────────┐
                    │   Cloud Hub     │
                    │   (Kubernetes)  │
                    │                 │
                    │  - Gestion      │
                    │  - Monitoring   │
                    │  - Agrégation   │
                    └────────┬────────┘
                             │
              ┌──────────────┼──────────────┐
              │              │              │
         ┌────▼────┐    ┌────▼────┐   ┌─────▼───┐
         │ Edge 1  │    │ Edge 2  │   │ Edge 3  │
         │ (K3s)   │    │ (K3s)   │   │ (K3s)   │
         │         │    │         │   │         │
         │ Usine A │    │ Magasin │   │ Véhicule│
         └────┬────┘    └────┬────┘   └─────┬───┘
              │              │              │
         [Capteurs]     [Caméras]      [GPS/Sensors]
```

### Modèle Regional Edge

Hiérarchie à plusieurs niveaux :

```
                    [Cloud Central]
                          │
          ┌───────────────┼───────────────┐
          │               │               │
    [Regional Edge]  [Regional Edge]  [Regional Edge]
     (EU-West)        (US-East)        (Asia)
          │               │               │
    ┌─────┼─────┐   ┌─────┼─────┐   ┌─────┼─────┐
[Edge 1][Edge 2] [Edge 3][Edge 4] [Edge 5][Edge 6]
```

**Avantages** :
- Réduction de latence multi-niveaux
- Agrégation régionale avant envoi au cloud central
- Résilience (si cloud central down, regional edge continue)

### Modèle Mesh

Les edge nodes communiquent entre eux :

```
     [Edge 1] ←→ [Edge 2]
         ↕  ╲    ╱  ↕
         ↕    ╲╱    ↕
         ↕    ╱╲    ↕
         ↕  ╱    ╲  ↕
     [Edge 3] ←→ [Edge 4]
```

**Cas d'usage** :
- Véhicules communiquant entre eux (V2V)
- Drones en essaim
- Robots collaboratifs

## Gestion centralisée de l'Edge

### 1. Rancher + Fleet pour l'Edge

**Fleet** est le système de GitOps de Rancher optimisé pour l'edge.

```
┌──────────────────────────────┐
│   Rancher Server (Cloud)     │
│   + Fleet Controller         │
└──────────────┬───────────────┘
               │
        Git Repository
               │
    ┌──────────┼──────────┐
    │          │          │
┌───▼───┐  ┌───▼───┐  ┌───▼───┐
│ Fleet │  │ Fleet │  │ Fleet │
│ Agent │  │ Agent │  │ Agent │
├───────┤  ├───────┤  ├───────┤
│ K3s 1 │  │ K3s 2 │  │ K3s 3 │
└───────┘  └───────┘  └───────┘
```

#### Déployer avec Fleet

```yaml
# fleet.yaml dans votre Git repo
defaultNamespace: apps

targetCustomizations:
# Configuration pour les edge devices ARM
- name: arm
  clusterSelector:
    matchLabels:
      arch: arm64
  helm:
    values:
      image:
        repository: myapp/arm64
        tag: v1.0.0

# Configuration pour les edge devices x86
- name: x86
  clusterSelector:
    matchLabels:
      arch: amd64
  helm:
    values:
      image:
        repository: myapp/amd64
        tag: v1.0.0
```

Fleet déploie automatiquement la bonne version selon l'architecture !

### 2. ArgoCD pour l'Edge

ArgoCD peut gérer des edge clusters, mais nécessite une connexion :

```yaml
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: edge-deployments
spec:
  generators:
  - clusters:
      selector:
        matchLabels:
          environment: edge
  template:
    metadata:
      name: '{{name}}-app'
    spec:
      project: default
      source:
        repoURL: https://github.com/myorg/edge-apps
        path: 'apps/{{metadata.labels.location}}'
      destination:
        server: '{{server}}'
        namespace: default
```

### 3. Flux pour l'Edge

Flux supporte bien l'edge car il fonctionne en "pull" :

```bash
# Sur chaque edge cluster
flux bootstrap github \
  --owner=myorg \
  --repository=edge-fleet \
  --branch=main \
  --path=clusters/{{cluster-name}}
```

Chaque edge cluster "tire" sa configuration depuis Git.

## Stockage à l'Edge

### Défis du stockage Edge

Les edge devices ont des contraintes de stockage :
- ⚠️ Espace limité (souvent <256 GB)
- ⚠️ Pas de SAN/NAS disponible
- ⚠️ Disques locaux peuvent tomber en panne

### Solutions de stockage

#### 1. Local Path Provisioner

Stockage local simple (inclus dans K3s) :

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: local-storage
spec:
  accessModes:
  - ReadWriteOnce
  storageClassName: local-path
  resources:
    requests:
      storage: 10Gi
```

#### 2. Longhorn pour l'Edge

Si vous avez plusieurs edge nodes proches :

```bash
# Installer Longhorn sur K3s
kubectl apply -f https://raw.githubusercontent.com/longhorn/longhorn/master/deploy/longhorn.yaml
```

Longhorn réplique les données entre nodes pour résilience.

#### 3. Edge CSI Drivers

Pour du matériel spécifique :
- USB drives : local-path
- SD cards : local-path avec précautions
- Industrial SATA : standard CSI drivers

#### 4. Stockage éphémère

Pour des données temporaires :

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: edge-processor
spec:
  containers:
  - name: processor
    image: myapp:v1
    volumeMounts:
    - name: temp-storage
      mountPath: /tmp/data
  volumes:
  - name: temp-storage
    emptyDir:
      sizeLimit: 5Gi
```

Données perdues au redémarrage du Pod (acceptable pour du traitement temporaire).

## Réseau et connectivité

### 1. Gestion de la connectivité intermittente

Les applications edge doivent gérer la perte de connexion :

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: edge-config
data:
  connectivity.yaml: |
    # Mode de fonctionnement
    mode: hybrid

    # Quand connecté au cloud
    online:
      sync_interval: 5m
      send_telemetry: true
      fetch_updates: true

    # Quand offline
    offline:
      buffer_data: true
      buffer_max_size: 1GB
      local_processing_only: true

    # Transition online → offline
    on_disconnect:
      - save_state
      - buffer_queue
      - local_mode

    # Transition offline → online
    on_reconnect:
      - sync_buffered_data
      - fetch_missed_updates
      - resume_cloud_sync
```

### 2. VPN/Tunnel pour accès sécurisé

Accéder à vos edge devices depuis le cloud :

#### Option 1 : WireGuard

```bash
# Sur chaque edge device
apt install wireguard

# Configuration WireGuard
cat > /etc/wireguard/wg0.conf <<EOF
[Interface]
PrivateKey = <edge-private-key>
Address = 10.200.1.2/24

[Peer]
PublicKey = <cloud-public-key>
Endpoint = cloud.example.com:51820
AllowedIPs = 10.200.0.0/16
PersistentKeepalive = 25
EOF

# Démarrer
wg-quick up wg0
```

#### Option 2 : Tailscale

Solution plus simple (mais service tiers) :

```bash
# Sur chaque edge device
curl -fsSL https://tailscale.com/install.sh | sh
tailscale up

# Accès automatique via réseau mesh chiffré
```

### 3. Service Mesh pour l'Edge

#### Linkerd Edge

Linkerd fonctionne bien sur l'edge (léger) :

```bash
# Sur cluster edge K3s
linkerd install --crds | kubectl apply -f -
linkerd install | kubectl apply -f -
```

#### K8s Gateway API pour l'Edge

```yaml
apiVersion: gateway.networking.k8s.io/v1beta1
kind: Gateway
metadata:
  name: edge-gateway
spec:
  gatewayClassName: traefik
  listeners:
  - name: http
    protocol: HTTP
    port: 80
```

## Monitoring et Observabilité Edge

### Défis du monitoring Edge

- 1000 edge locations = 1000 dashboards à surveiller ?
- Données de monitoring volumineuses
- Besoin d'agrégation intelligente

### Architecture de monitoring

```
Edge Devices :
[Prometheus Agent] ──┐
[Prometheus Agent] ──┼──→ Remote Write → [Prometheus Central]
[Prometheus Agent] ──┘                            │
                                                  ↓
                                            [Grafana]
```

#### Configuration Prometheus Agent Mode

Sur chaque edge device :

```yaml
# prometheus-agent.yaml
global:
  external_labels:
    edge_location: 'factory-paris'
    edge_id: 'edge-001'

remote_write:
- url: https://prometheus-central.example.com/api/v1/write
  queue_config:
    capacity: 10000
    max_samples_per_send: 1000
    batch_send_deadline: 5s
  # Retry en cas de perte de connexion
  queue_config:
    max_shards: 10
    min_shards: 1
```

### Monitoring local et alertes critiques

Pour les alertes critiques, ne dépendez pas du cloud :

```yaml
# Alertmanager local sur l'edge
apiVersion: v1
kind: ConfigMap
metadata:
  name: alertmanager-config
data:
  alertmanager.yml: |
    route:
      receiver: 'edge-local'
      routes:
      # Alertes critiques → Action locale immédiate
      - match:
          severity: critical
        receiver: 'local-action'
        continue: true
      # Toutes les alertes → Envoi au cloud (si connecté)
      - receiver: 'cloud-hub'

    receivers:
    - name: 'local-action'
      webhook_configs:
      - url: 'http://emergency-handler:8080/alert'

    - name: 'cloud-hub'
      webhook_configs:
      - url: 'https://alerting-hub.example.com/edge-alerts'
        send_resolved: true
```

### Logs Edge

#### Option 1 : Loki avec agrégation

```yaml
# Promtail sur edge (collecte logs)
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: promtail
spec:
  template:
    spec:
      containers:
      - name: promtail
        image: grafana/promtail:latest
        args:
        - -config.file=/etc/promtail/promtail.yaml
        volumeMounts:
        - name: config
          mountPath: /etc/promtail
        - name: varlog
          mountPath: /var/log
```

Configuration Promtail pour buffer offline :

```yaml
positions:
  filename: /tmp/positions.yaml

clients:
- url: https://loki-central.example.com/loki/api/v1/push
  # Buffer important pour gérer les déconnexions
  batchwait: 5s
  batchsize: 102400
  timeout: 10s
  # Retry
  backoff_config:
    min_period: 500ms
    max_period: 5m
    max_retries: 10
```

#### Option 2 : Logs locaux rotatifs

Pour environnements très contraints :

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: app
spec:
  containers:
  - name: app
    image: myapp:v1
    volumeMounts:
    - name: logs
      mountPath: /var/log/app
  # Sidecar pour rotation de logs
  - name: log-rotator
    image: busybox
    command:
    - sh
    - -c
    - |
      while true; do
        find /var/log/app -name "*.log" -size +50M -delete
        sleep 3600
      done
    volumeMounts:
    - name: logs
      mountPath: /var/log/app
  volumes:
  - name: logs
    emptyDir:
      sizeLimit: 500Mi
```

## Sécurité à l'Edge

### Défis de sécurité Edge

Les edge devices sont plus vulnérables :
- ❌ Accès physique possible
- ❌ Réseau local potentiellement non sécurisé
- ❌ Difficile à superviser en temps réel
- ❌ Mises à jour compliquées

### Bonnes pratiques de sécurité

#### 1. Chiffrement des données au repos

```bash
# Chiffrer le disque avec LUKS
cryptsetup luksFormat /dev/sda1
cryptsetup luksOpen /dev/sda1 encrypted-disk
```

#### 2. Secure Boot et Attestation

```bash
# Vérifier que Secure Boot est activé
mokutil --sb-state

# Configuration TPM pour attestation
tpm2_getrandom 16 --hex
```

#### 3. Network Policies strictes

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: edge-security
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  - Egress
  ingress:
  # Autoriser uniquement le trafic local
  - from:
    - podSelector: {}
  egress:
  # Autoriser uniquement cloud hub et DNS
  - to:
    - namespaceSelector:
        matchLabels:
          name: kube-system
    ports:
    - protocol: UDP
      port: 53
  - to:
    - podSelector: {}
  - to:
    - ipBlock:
        cidr: 203.0.113.0/24  # IP du cloud hub
    ports:
    - protocol: TCP
      port: 443
```

#### 4. Secrets chiffrés

Utiliser Sealed Secrets ou SOPS :

```bash
# Chiffrer un secret avec SOPS
sops --encrypt --gcp-kms projects/my-project/locations/global/keyRings/my-ring/cryptoKeys/my-key \
  secret.yaml > secret.enc.yaml

# Déployer sur edge
kubectl apply -f secret.enc.yaml
```

#### 5. Read-only filesystem

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: secure-edge-app
spec:
  securityContext:
    runAsNonRoot: true
    runAsUser: 1000
    fsGroup: 1000
  containers:
  - name: app
    image: myapp:v1
    securityContext:
      allowPrivilegeEscalation: false
      readOnlyRootFilesystem: true
      capabilities:
        drop:
        - ALL
    volumeMounts:
    - name: tmp
      mountPath: /tmp
  volumes:
  - name: tmp
    emptyDir: {}
```

#### 6. Surveillance des intrusions

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: falco
spec:
  template:
    spec:
      containers:
      - name: falco
        image: falcosecurity/falco:latest
        securityContext:
          privileged: true
        volumeMounts:
        - name: docker-socket
          mountPath: /var/run/docker.sock
```

## Mise à jour et maintenance Edge

### Stratégies de mise à jour

#### 1. Over-The-Air (OTA) Updates

Mise à jour sans intervention physique :

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: edge-updater
data:
  update-policy.yaml: |
    schedule: "0 3 * * 0"  # Dimanche à 3h du matin

    strategy:
      type: rolling
      max_unavailable: 20%  # Max 20% des edges en update simultanément

    pre_update:
      - health_check
      - backup_config
      - notify_hub

    post_update:
      - verify_version
      - smoke_tests
      - report_status

    rollback:
      auto: true
      on_failure: true
      timeout: 30m
```

#### 2. Canary Updates

Tester d'abord sur un sous-ensemble :

```yaml
# Fleet GitOps
apiVersion: fleet.cattle.io/v1alpha1
kind: GitRepo
metadata:
  name: edge-apps
spec:
  repo: https://github.com/myorg/edge-fleet
  branch: main

  # Déploiement progressif
  targets:
  # Phase 1: 5% des edges (canary)
  - name: canary
    clusterSelector:
      matchLabels:
        canary: "true"

  # Phase 2: 25% après 24h
  - name: rollout-25
    clusterSelector:
      matchLabels:
        wave: "1"

  # Phase 3: 100% après 48h
  - name: rollout-all
    clusterSelector:
      matchLabels:
        edge: "true"
```

#### 3. Atomic Updates avec OSTree

Pour les mises à jour système complètes :

```bash
# Utiliser Flatcar Linux ou Fedora CoreOS
rpm-ostree status
rpm-ostree upgrade
systemctl reboot
```

Si problème → rollback automatique au boot.

## Exemple complet : Système Edge de détection d'intrusion

### Architecture

```
Edge Location (Usine):
┌────────────────────────────────────┐
│  Intel NUC avec K3s                │
│                                    │
│  ┌──────────────────────────────┐  │
│  │  Caméras IP (10x)            │  │
│  │    ↓                         │  │
│  │  Video Ingestion Service     │  │
│  │    ↓                         │  │
│  │  AI Detection (TensorFlow)   │  │
│  │    ↓                         │  │
│  │  Alert Manager               │  │
│  │    ↓                         │  │
│  │  ├─→ Local Actions           │  │
│  │  └─→ Cloud Notification      │  │
│  └──────────────────────────────┘  │
│                                    │
│  Prometheus + Grafana (local)      │
└────────────────────────────────────┘
```

### Manifestes Kubernetes

#### 1. Video Ingestion

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: video-ingestion
spec:
  replicas: 1
  selector:
    matchLabels:
      app: video-ingestion
  template:
    metadata:
      labels:
        app: video-ingestion
    spec:
      containers:
      - name: ingestion
        image: myapp/video-ingestion:v1
        resources:
          requests:
            memory: "512Mi"
            cpu: "500m"
          limits:
            memory: "1Gi"
            cpu: "1000m"
        env:
        - name: CAMERA_URLS
          valueFrom:
            configMapKeyRef:
              name: camera-config
              key: urls
        - name: FRAME_RATE
          value: "5"  # 5 FPS pour économiser CPU
        volumeMounts:
        - name: video-buffer
          mountPath: /tmp/frames
      volumes:
      - name: video-buffer
        emptyDir:
          sizeLimit: 2Gi
```

#### 2. AI Detection

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: ai-detection
spec:
  replicas: 1
  selector:
    matchLabels:
      app: ai-detection
  template:
    metadata:
      labels:
        app: ai-detection
    spec:
      # Affinity pour GPU si disponible
      affinity:
        nodeAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
          - weight: 100
            preference:
              matchExpressions:
              - key: nvidia.com/gpu
                operator: Exists
      containers:
      - name: detector
        image: myapp/ai-detector:v1
        resources:
          requests:
            memory: "2Gi"
            cpu: "1000m"
          limits:
            memory: "4Gi"
            cpu: "2000m"
            nvidia.com/gpu: 1  # Si GPU disponible
        env:
        - name: MODEL_PATH
          value: "/models/intrusion-detection-v2.pb"
        - name: CONFIDENCE_THRESHOLD
          value: "0.85"
        volumeMounts:
        - name: models
          mountPath: /models
          readOnly: true
        - name: video-buffer
          mountPath: /tmp/frames
      volumes:
      - name: models
        persistentVolumeClaim:
          claimName: ai-models
      - name: video-buffer
        emptyDir:
          sizeLimit: 2Gi
```

#### 3. Alert Manager

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: alert-manager
spec:
  replicas: 1
  selector:
    matchLabels:
      app: alert-manager
  template:
    spec:
      containers:
      - name: alerter
        image: myapp/alert-manager:v1
        env:
        - name: ALERT_WEBHOOK_LOCAL
          value: "http://local-siren:8080/trigger"
        - name: ALERT_WEBHOOK_CLOUD
          valueFrom:
            secretKeyRef:
              name: cloud-credentials
              key: webhook-url
        - name: OFFLINE_MODE
          value: "true"  # Continue en mode offline
```

#### 4. Configuration des caméras

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: camera-config
data:
  urls: |
    rtsp://camera1.local:554/stream
    rtsp://camera2.local:554/stream
    rtsp://camera3.local:554/stream
    rtsp://camera4.local:554/stream
    rtsp://camera5.local:554/stream
```

### Déploiement et gestion

```bash
# Déploiement initial
kubectl apply -f edge-intrusion-detection/

# Monitoring
kubectl get pods
kubectl logs -f deployment/ai-detection

# Mise à jour du modèle AI
kubectl cp nouveau-model.pb ai-detection-xxx:/models/

# Redémarrage pour charger nouveau modèle
kubectl rollout restart deployment/ai-detection
```

## Conclusion

L'**Edge Computing avec Kubernetes** représente l'avenir de nombreux secteurs industriels. En rapprochant le traitement des données de leur source, vous obtenez des latences ultra-faibles, une résilience accrue et une meilleure utilisation de la bande passante.

### Points clés à retenir

1. **Edge Computing** = Traitement local, près de la source de données
2. **Kubernetes léger** : K3s, MicroK8s pour ressources limitées
3. **Architecture Hub & Spoke** : Gestion centralisée, exécution distribuée
4. **Résilience offline** : Les edge devices doivent fonctionner sans cloud
5. **GitOps** : Fleet, ArgoCD pour déploiement à grande échelle
6. **Monitoring hiérarchique** : Local pour critique, agrégé pour tendances
7. **Sécurité renforcée** : Accès physique possible = protections accrues
8. **Mises à jour OTA** : Essentielles pour maintenir 1000+ devices

### Recommandations pour MicroK8s en Edge

**Pour apprendre l'edge** :
- Installez MicroK8s sur un Raspberry Pi
- Simulez une perte de connexion (débranchez Ethernet)
- Testez que vos apps continuent de fonctionner
- Explorez K3s pour comparer

**Pour un lab edge** :
- 2-3 Raspberry Pi ou Intel NUC
- Un cluster "hub" pour la gestion
- GitOps avec Fleet ou ArgoCD
- Monitoring avec Prometheus/Grafana

**Cas d'usage réalistes** :
- Home automation (domotique)
- Lab de surveillance vidéo
- Monitoring de capteurs IoT
- Traitement local de données

### Évolution future de l'Edge

**Tendances émergentes** :
- **5G MEC** : Edge computing intégré aux antennes 5G
- **WebAssembly** : Alternative aux conteneurs pour l'edge ultra-léger
- **Edge AI** : Modèles ML optimisés pour edge devices
- **Serverless Edge** : Fonctions edge sans serveur (Cloudflare Workers)
- **Edge native apps** : Applications conçues dès le départ pour l'edge

### Quand choisir l'Edge ?

**✅ OUI si** :
- Latence <50ms critique
- Bande passante limitée/coûteuse
- Besoin de fonctionnement offline
- Données sensibles (vie privée)
- IoT avec nombreux capteurs

**❌ NON si** :
- Latence de 500ms acceptable
- Bande passante illimitée
- Connexion stable garantie
- Pas de contraintes de données
- Complexité opérationnelle à éviter

### Prochaines étapes

Après avoir exploré l'Edge Computing, vous pourriez vous intéresser à :
- **25.6 Machine Learning sur Kubernetes** : Intégration de ML/AI dans K8s
- **25.3 Operators** : Automatiser la gestion d'apps complexes en edge
- **19. Scaling et Autoscaling** : Adapter aux ressources limitées

---

**Ressources complémentaires** :

- K3s Documentation : https://docs.k3s.io/
- KubeEdge : https://kubeedge.io/
- CNCF Edge Computing Landscape : https://landscape.cncf.io/card-mode?category=edge-computing
- Rancher Fleet : https://fleet.rancher.io/
- Edge Native Working Group : https://github.com/cncf/tag-runtime/tree/master/wg-edge-native
- Awesome Edge Computing : https://github.com/qijianpeng/awesome-edge-computing

⏭️ [Machine Learning sur Kubernetes](/25-aller-plus-loin/06-machine-learning-sur-kubernetes.md)
