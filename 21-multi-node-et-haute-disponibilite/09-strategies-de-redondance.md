🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 21.9 Stratégies de Redondance

## Introduction

Vous avez construit un cluster multi-node avec haute disponibilité, backups automatiques et stockage distribué. Mais la vraie résilience ne vient pas d'un seul mécanisme - elle vient d'une **stratégie de redondance complète** qui couvre tous les niveaux de votre infrastructure.

Dans ce chapitre, nous allons explorer :
- Les différents niveaux de redondance (infrastructure, réseau, données, applications)
- Comment concevoir une architecture véritablement résiliente
- Les points de défaillance uniques (SPOF) et comment les éliminer
- Les compromis entre coût, complexité et résilience
- Comment tester et valider votre redondance

La redondance est l'art de **ne jamais avoir un seul point de défaillance**. C'est ce qui fait la différence entre un cluster qui "tient" et un cluster qui **tient vraiment** en production.

**Principe fondamental :** "N+1" ou "N+2" - Toujours avoir au moins une ressource de plus que le minimum nécessaire.

## Qu'est-ce que la Redondance ?

### Définition

La **redondance** consiste à avoir plusieurs composants identiques ou équivalents, de sorte que si l'un tombe en panne, les autres peuvent prendre le relais sans interruption de service.

**Analogie de l'avion :**
Un avion de ligne a :
- 2 ou 4 moteurs (si un tombe, les autres compensent)
- 2 pilotes (si l'un est incapacité, l'autre prend le relais)
- Systèmes électriques redondants
- Systèmes hydrauliques redondants
- Multiples pompes à carburant

**Pourquoi ?** Parce que la vie des passagers en dépend.

Dans votre cluster, c'est pareil : la disponibilité de vos applications (et peut-être votre business) en dépend.

### Types de Redondance

**1. Redondance Active-Active**
Tous les composants sont actifs en même temps et partagent la charge.

```
┌──────────┐      ┌──────────┐      ┌──────────┐
│ Node 1   │      │ Node 2   │      │ Node 3   │
│ ACTIF    │◄────►│ ACTIF    │◄────►│ ACTIF    │
│ Pod A    │      │ Pod B    │      │ Pod C    │
└──────────┘      └──────────┘      └──────────┘
     ▲                 ▲                 ▲
     └─────────────────┼─────────────────┘
                   Trafic réparti
```

**Avantages :**
- Utilisation optimale des ressources
- Performance maximale
- Pas de ressource "gaspillée"

**Inconvénients :**
- Si charge élevée, la perte d'un composant peut surcharger les autres

**2. Redondance Active-Passive**
Un composant est actif, les autres en standby (attente).

```
┌──────────┐      ┌──────────┐
│ Node 1   │      │ Node 2   │
│ ACTIF    │      │ PASSIF   │
│ Leader   │◄────►│ Standby  │
│ Tout le  │      │ Prêt mais│
│ trafic   │      │ inactif  │
└──────────┘      └──────────┘
```

**Avantages :**
- Bascule simple
- Charge prévisible

**Inconvénients :**
- Ressources sous-utilisées (standby ne fait rien)
- Temps de bascule (quelques secondes)

**3. Redondance N+1**
Vous avez N composants nécessaires + 1 de secours.

Exemple : Si vous avez besoin de 3 nœuds pour le quorum (N=3), avoir 4 nœuds (N+1=4) vous protège.

**4. Redondance N+2**
N composants + 2 de secours (plus coûteux mais plus résilient).

### Points de Défaillance Uniques (SPOF)

Un **SPOF** (Single Point of Failure) est un composant unique dont la défaillance entraîne l'arrêt du système entier.

**Exemples de SPOF :**
- Un seul switch réseau
- Un seul serveur NFS
- Un seul nœud Kubernetes (cluster single-node)
- Une seule alimentation électrique
- Un seul uplink Internet

**Objectif :** Identifier et **éliminer tous les SPOF**.

## Niveaux de Redondance

### Niveau 1 : Infrastructure Physique

#### Serveurs / Nœuds

**Configuration minimale HA (vue au chapitre 21.4) :**
```
3 nœuds control plane + worker
- Tolérance : 1 panne
- Quorum : 2/3
```

**Configuration renforcée :**
```
3 nœuds control plane + 2 workers purs
- Tolérance : 1 panne control plane + N workers
- Plus de capacité applicative
```

**Configuration maximale :**
```
5 nœuds control plane + 5 workers purs
- Tolérance : 2 pannes simultanées control plane
- Quorum : 3/5
- Haute capacité applicative
```

**Règle :** Toujours nombre impair de nœuds control plane (3, 5, 7).

#### Répartition Géographique

**Problème :** Si tous vos nœuds sont dans la même pièce/rack, une panne (électricité, climatisation, inondation) peut tout arrêter.

**Solution : Zones de disponibilité**

```
┌─────────────────────────────────────────────────────┐
│                DATACENTER / HOMELAB                 │
├─────────────────────────────────────────────────────┤
│                                                     │
│  ┌──────────────┐  ┌──────────────┐  ┌───────────┐  │
│  │   Zone A     │  │   Zone B     │  │  Zone C   │  │
│  │              │  │              │  │           │  │
│  │  Rack 1      │  │  Rack 2      │  │  Rack 3   │  │
│  │  Alim 1      │  │  Alim 2      │  │  Alim 3   │  │
│  │  Switch 1    │  │  Switch 2    │  │  Switch 3 │  │
│  │              │  │              │  │           │  │
│  │  Node 1      │  │  Node 2      │  │  Node 3   │  │
│  │  Worker 1    │  │  Worker 2    │  │  Worker 3 │  │
│  └──────────────┘  └──────────────┘  └───────────┘  │
└─────────────────────────────────────────────────────┘
```

**Principe :** Répartir les nœuds sur :
- Différents racks (panne d'un PDU)
- Différents switchs (panne réseau)
- Différentes alimentations
- Idéalement, différentes pièces ou datacenters

**Labelliser les zones :**
```bash
microk8s kubectl label node node1 topology.kubernetes.io/zone=zone-a
microk8s kubectl label node node2 topology.kubernetes.io/zone=zone-b
microk8s kubectl label node node3 topology.kubernetes.io/zone=zone-c
```

**Utiliser topology spread constraints :**
```yaml
spec:
  topologySpreadConstraints:
  - maxSkew: 1
    topologyKey: topology.kubernetes.io/zone
    whenUnsatisfiable: DoNotSchedule
    labelSelector:
      matchLabels:
        app: myapp
```

Garantit que les pods sont répartis équitablement entre les zones.

#### Alimentation Électrique

**SPOF : Une seule source d'alimentation**

**Solutions :**

**1. Onduleurs (UPS)**
- Protège contre les micro-coupures
- Donne le temps d'éteindre proprement

**2. Alimentations redondantes**
```
                 ┌──────────────┐
                 │  Serveur     │
                 │              │
        ┌────────┤ PSU 1  PSU 2 ├────────┐
        │        └──────────────┘        │
        │                                │
  ┌─────▼─────┐                    ┌─────▼─────┐
  │ Circuit A │                    │ Circuit B │
  │ Tableau 1 │                    │ Tableau 2 │
  └───────────┘                    └───────────┘
```

Si un circuit tombe, le serveur continue via l'autre.

**3. Générateur (pour homelab/PME)**
En cas de coupure prolongée, un générateur prend le relais.

#### Refroidissement

**Serveurs surchauffent → Extinction automatique**

**Solutions :**
- Climatisation redondante (2 unités)
- Surveillance de température
- Alertes automatiques

### Niveau 2 : Réseau

#### Connectivité Réseau

**SPOF : Un seul switch**

**Solution : Topology réseau redondante**

```
                     ┌────────────┐
                     │  Internet  │
                     └─────┬──────┘
                           │
              ┌────────────┴──────────┐
              │                       │
        ┌─────▼─────┐           ┌─────▼─────┐
        │ Router 1  │           │ Router 2  │
        │ (Actif)   │◄─────────►│ (Backup)  │
        └─────┬─────┘           └─────┬─────┘
              │                       │
        ┌─────▼─────┐           ┌─────▼─────┐
        │ Switch 1  │◄─────────►│ Switch 2  │
        └─────┬─────┘           └─────┬─────┘
              │                       │
    ┌─────────┼───────────┬───────────┼─────────┐
    │         │           │           │         │
┌───▼───┐ ┌───▼───┐   ┌───▼───┐   ┌───▼───┐ ┌───▼───┐
│Node 1 │ │Node 2 │   │Node 3 │   │Worker1│ │Worker2│
└───────┘ └───────┘   └───────┘   └───────┘ └───────┘
```

**Caractéristiques :**
- Chaque nœud connecté à 2 switchs (bonding)
- Redondance switch
- Redondance routeur

**Configuration bonding (exemple Ubuntu) :**
```yaml
# /etc/netplan/01-netcfg.yaml
network:
  version: 2
  ethernets:
    eth0:
      dhcp4: false
    eth1:
      dhcp4: false
  bonds:
    bond0:
      interfaces:
      - eth0
      - eth1
      addresses:
      - 192.168.1.10/24
      gateway4: 192.168.1.1
      parameters:
        mode: active-backup
        mii-monitor-interval: 100
```

Si une interface réseau ou un switch tombe, l'autre prend le relais automatiquement.

#### DNS

**SPOF : Un seul serveur DNS**

**Solution : DNS redondants**
```bash
# /etc/netplan/01-netcfg.yaml
nameservers:
  addresses:
  - 8.8.8.8      # Google DNS primaire
  - 8.8.4.4      # Google DNS secondaire
  - 1.1.1.1      # Cloudflare DNS tertiaire
```

**Ou DNS interne redondant :**
- 2 serveurs Pi-hole
- 2 serveurs Bind9
- Configuration automatique via DHCP

#### Uplink Internet

**SPOF : Un seul fournisseur Internet**

**Solution (entreprise) :** Dual WAN
- 2 FAI différents (Fibre + 4G/5G backup)
- Routeur avec failover automatique

**Solution (homelab) :**
- 1 FAI principal (fibre)
- 4G/5G en backup (Starlink, hotspot mobile)

### Niveau 3 : Stockage

#### Stockage Distribué (Longhorn)

**Redondance par réplication (vu au chapitre 21.7)**

**Configuration par défaut :** 3 replicas

```yaml
# StorageClass avec 3 replicas
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: longhorn-redundant
provisioner: driver.longhorn.io
parameters:
  numberOfReplicas: "3"
  staleReplicaTimeout: "2880"
```

**Répartition des replicas :**
```
Volume "database-vol" (10 GB)
├─ Replica 1 : Node 1 (Actif)
├─ Replica 2 : Node 2 (Actif)
└─ Replica 3 : Node 3 (Actif)

Si Node 2 tombe → Replicas 1 et 3 continuent
```

**Configuration renforcée (données critiques) :**
```yaml
parameters:
  numberOfReplicas: "5"  # 5 copies !
```

Nécessite 5 nœuds minimum.

#### Backups des Volumes

**Redondance temporelle (versions)**

**Snapshots Longhorn :**
- Quotidiens : 7 jours
- Hebdomadaires : 4 semaines
- Mensuels : 12 mois

**Backup externe :**
- S3 (AWS, Backblaze B2, Wasabi)
- NFS sur un serveur dédié
- Stockage offline (disque externe)

**Règle 3-2-1 (rappel) :**
- 3 copies
- 2 supports différents
- 1 hors-site

### Niveau 4 : Applications

#### Replicas

**Principe de base :** Toujours au moins 2 réplicas pour les applications critiques.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-app
spec:
  replicas: 3  # ← Minimum 2, recommandé 3+
  selector:
    matchLabels:
      app: web
  template:
    metadata:
      labels:
        app: web
    spec:
      containers:
      - name: nginx
        image: nginx:alpine
```

**Règles selon criticité :**

| Type d'application | Replicas min | Recommandé |
|--------------------|--------------|------------|
| Non-critique (dev) | 1 | 1 |
| Standard | 2 | 3 |
| Importante | 3 | 5 |
| Critique | 3 | 7+ |

#### Pod Anti-Affinity

**Garantir que les replicas sont sur des nœuds différents :**

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: database
spec:
  replicas: 3
  selector:
    matchLabels:
      app: database
  template:
    metadata:
      labels:
        app: database
    spec:
      affinity:
        podAntiAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
          - labelSelector:
              matchExpressions:
              - key: app
                operator: In
                values:
                - database
            topologyKey: kubernetes.io/hostname
      containers:
      - name: postgres
        image: postgres:15
```

**Résultat :**
- Replica 1 → Node 1
- Replica 2 → Node 2
- Replica 3 → Node 3

Si Node 1 tombe, Replicas 2 et 3 continuent. ✅

#### Pod Disruption Budgets (PDB)

**Garantir un nombre minimum de pods disponibles pendant les maintenances :**

```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: web-app-pdb
spec:
  minAvailable: 2  # Au moins 2 pods doivent rester actifs
  selector:
    matchLabels:
      app: web
```

**Ou :**
```yaml
spec:
  maxUnavailable: 1  # Max 1 pod peut être down à la fois
```

**Utilité :**
Quand vous drainez un nœud (`kubectl drain`), Kubernetes respecte le PDB et n'évacue qu'un pod à la fois, garantissant que l'application reste accessible.

#### Health Checks

**Liveness et Readiness probes :**

```yaml
spec:
  containers:
  - name: app
    image: myapp:v1
    livenessProbe:
      httpGet:
        path: /healthz
        port: 8080
      initialDelaySeconds: 30
      periodSeconds: 10
    readinessProbe:
      httpGet:
        path: /ready
        port: 8080
      initialDelaySeconds: 5
      periodSeconds: 5
```

**Liveness :** Si échec, Kubernetes redémarre le pod.
**Readiness :** Si échec, Kubernetes retire le pod du Service (pas de trafic).

**Redondance via auto-healing :**
Si un pod devient unhealthy, Kubernetes le remplace automatiquement.

#### StatefulSets pour Applications Stateful

**Pour bases de données, applications avec état :**

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: postgres
spec:
  serviceName: postgres
  replicas: 3
  selector:
    matchLabels:
      app: postgres
  template:
    metadata:
      labels:
        app: postgres
    spec:
      containers:
      - name: postgres
        image: postgres:15
        volumeMounts:
        - name: data
          mountPath: /var/lib/postgresql/data
  volumeClaimTemplates:
  - metadata:
      name: data
    spec:
      accessModes: [ "ReadWriteOnce" ]
      storageClassName: longhorn
      resources:
        requests:
          storage: 10Gi
```

**Avantages :**
- Noms de pods stables (postgres-0, postgres-1, postgres-2)
- Volumes persistants dédiés par pod
- Ordre de démarrage/arrêt prévisible

### Niveau 5 : Bases de Données

#### Réplication Master-Slave

**Configuration PostgreSQL avec réplication :**

```
┌─────────────┐        ┌─────────────┐        ┌─────────────┐
│   Master    │───────►│   Slave 1   │        │   Slave 2   │
│  (Writes)   │   sync │  (Reads)    │◄──────►│  (Reads)    │
│             │───────►│             │  async │             │
└─────────────┘        └─────────────┘        └─────────────┘
      │                       │                       │
      └───────────────────────┴───────────────────────┘
                   Failover automatique
```

**En cas de panne du Master :**
- Slave 1 est promu Master
- Slave 2 se reconnecte au nouveau Master
- Downtime : 5-30 secondes

**Outils pour PostgreSQL :**
- **Patroni** (HA PostgreSQL)
- **Stolon** (HA PostgreSQL)
- Opérateurs Kubernetes : postgres-operator, zalando operator

#### Clustering avec Consensus

**Pour MySQL, MariaDB :**
- Galera Cluster (Active-Active)
- InnoDB Cluster

**Pour bases NoSQL :**
- MongoDB Replica Set
- Cassandra (naturellement distribué)
- Redis Sentinel

**Principe :** Toutes les instances sont actives, les données répliquées automatiquement.

### Niveau 6 : Services Critiques

#### DNS Interne (CoreDNS)

**Par défaut, CoreDNS tourne en 2 replicas :**

```bash
microk8s kubectl get deployment -n kube-system coredns
```

```
NAME      READY   UP-TO-DATE   AVAILABLE
coredns   2/2     2            2
```

**Si critique, augmenter :**
```bash
microk8s kubectl scale deployment coredns --replicas=3 -n kube-system
```

#### Ingress Controller

**NGINX Ingress Controller en HA :**

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-ingress-controller
  namespace: ingress
spec:
  replicas: 3  # Au moins 3 replicas
  selector:
    matchLabels:
      app: nginx-ingress
  template:
    metadata:
      labels:
        app: nginx-ingress
    spec:
      affinity:
        podAntiAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
          - labelSelector:
              matchExpressions:
              - key: app
                operator: In
                values:
                - nginx-ingress
            topologyKey: kubernetes.io/hostname
      containers:
      - name: nginx-ingress
        image: registry.k8s.io/ingress-nginx/controller:latest
```

**Exposer avec LoadBalancer (MetalLB) :**
Le trafic est automatiquement distribué entre les 3 replicas.

#### Monitoring (Prometheus/Grafana)

**Prometheus HA :**
```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: prometheus
spec:
  replicas: 2  # 2 instances Prometheus
```

**Thanos pour fédération :**
Agrège les données de plusieurs Prometheus instances.

**Grafana HA :**
```yaml
spec:
  replicas: 2
```

Avec base de données externe (PostgreSQL) pour partager les dashboards.

## Stratégies par Niveau de Criticité

### Applications Non-Critiques (Dev/Test)

**Configuration minimale :**
- 1 replica
- Pas de PDB
- Pas d'anti-affinity
- Backups hebdomadaires

**Coût :** Minimal
**Résilience :** Faible (acceptable pour dev)

### Applications Standard (Production non-critique)

**Configuration recommandée :**
- 2-3 replicas
- PDB avec minAvailable: 1
- Anti-affinity optionnelle
- Backups quotidiens
- Health checks (liveness + readiness)

**Coût :** Modéré
**Résilience :** Bonne

### Applications Critiques (Production essentielle)

**Configuration complète :**
- 3-5 replicas minimum
- PDB avec minAvailable: 2
- Anti-affinity obligatoire
- Topology spread constraints (zones)
- Backups multi-versions (quotidien + temps réel)
- Health checks avancés
- Monitoring et alerting 24/7
- Tests de failover réguliers

**Coût :** Élevé
**Résilience :** Très élevée

### Applications Mission-Critiques (Zéro Downtime)

**Configuration maximale :**
- 7+ replicas
- PDB avec minAvailable: 5
- Multi-zones géographiques
- Multi-clusters (si possible)
- Backups en continu + rétention longue
- Monitoring multi-niveaux
- Équipe d'astreinte 24/7
- Procédures de failover testées mensuellement

**Coût :** Très élevé
**Résilience :** Maximale

## Matrice de Redondance

| Composant | Single-Node | HA (3 nodes) | Production | Mission-Critique |
|-----------|-------------|--------------|------------|------------------|
| **Nœuds Control Plane** | 1 | 3 | 3-5 | 5-7 |
| **Workers** | 0 (node=worker) | 0-2 | 3-5 | 5-10 |
| **Stockage Replicas** | 1 | 3 | 3 | 5 |
| **App Replicas** | 1 | 2-3 | 3-5 | 7+ |
| **Ingress Replicas** | 1 | 2 | 3 | 3-5 |
| **DNS Replicas** | 1 | 2 | 3 | 3 |
| **Backups** | Aucun | Quotidien | Multi-versions | Temps réel |
| **Zones** | 1 | 1 | 1-2 | 2-3 |
| **Switch Réseau** | 1 | 1 | 2 | 2+ |
| **Alim Électrique** | 1 | 1 | UPS | UPS + Générateur |

## Compromis : Coût vs Résilience

### Calcul du Coût

**Exemple cluster 3 nœuds :**
```
Matériel :
- 3 mini-PCs @ 300€ = 900€
- 1 switch 24 ports = 150€
- Total initial : 1050€

Électricité :
- 3 machines × 50W × 24h × 365j × 0.20€/kWh = 262€/an

Total 1ère année : 1312€
Total année suivante : 262€/an
```

**Passer à 5 nœuds :**
```
Matériel supplémentaire :
- 2 mini-PCs @ 300€ = 600€

Électricité supplémentaire :
- 2 × 50W × 24h × 365j × 0.20€/kWh = 175€/an

Coût additionnel 1ère année : 775€
Coût additionnel années suivantes : 175€/an
```

**Question :** Votre business justifie-t-il 775€ supplémentaires pour passer de 1 panne tolérée à 2 ?

### ROI de la Redondance

**Calcul du risque sans redondance :**
```
Probabilité de panne nœud : 5% par an
Coût d'une panne (downtime) : 10 000€

Risque = 5% × 10 000€ = 500€/an

Si redondance coûte 175€/an et évite le risque :
ROI = (500 - 175) / 175 = 186% ✅
```

**La redondance est rentable si le coût d'une panne > coût de la redondance.**

### Optimisation des Coûts

**1. Commencer petit, scaler progressivement**
- Démarrer avec 3 nœuds
- Ajouter des workers au besoin
- Augmenter replicas au fur et à mesure

**2. Réutiliser du matériel existant**
- Vieux laptops comme nœuds
- Mini-PCs économiques (Raspberry Pi, Intel NUC d'occasion)

**3. Redondance sélective**
- Redondance complète pour production
- Redondance minimale pour dev/test

**4. Cloud hybride**
- Cluster on-premise pour le quotidien
- Backup dans le cloud (S3) pour le DR

## Tests de Redondance

### Chaos Engineering

**Principe :** Casser volontairement des choses pour vérifier que le système tient.

#### Test 1 : Arrêt d'un Nœud Worker

```bash
# Sur worker1
sudo shutdown -h now
```

**Vérifications :**
- Les pods sont-ils redistribués ?
- Combien de temps pour la redistribution ?
- Les applications restent-elles accessibles ?

**Attendu :**
- Redistribution en 1-2 minutes
- Aucune interruption de service (si replicas ≥ 2)

#### Test 2 : Arrêt du Nœud Leader (Control Plane)

```bash
# Identifier le leader
microk8s kubectl get lease -n kube-system

# Arrêter le leader
# Sur le nœud leader
sudo shutdown -h now
```

**Vérifications :**
- Nouvelle élection de leader ?
- Cluster reste-t-il fonctionnel ?
- Temps de basculement ?

**Attendu :**
- Nouvelle élection en 5-10 secondes
- Cluster continue de fonctionner
- Commandes kubectl fonctionnent (via les autres nœuds)

#### Test 3 : Saturation Réseau

```bash
# Simuler une charge réseau importante
# Sur un nœud
apt install iperf3
iperf3 -s

# Sur un autre nœud
iperf3 -c <ip-du-premier-noeud> -t 300 -P 10
```

**Vérifications :**
- Les pods continuent de communiquer ?
- Latence augmentée mais service maintenu ?

#### Test 4 : Remplissage du Disque

```bash
# Créer un gros fichier pour saturer le disque
dd if=/dev/zero of=/tmp/bigfile bs=1M count=10000
```

**Vérifications :**
- Kubernetes marque-t-il le nœud comme "DiskPressure" ?
- Les nouveaux pods évitent-ils ce nœud ?
- Alertes déclenchées ?

**Nettoyer :**
```bash
rm /tmp/bigfile
```

#### Test 5 : Corruption de Pod

```bash
# Tuer un processus dans un pod
POD_NAME=$(microk8s kubectl get pod -l app=web -o jsonpath="{.items[0].metadata.name}")
microk8s kubectl exec $POD_NAME -- killall -9 nginx
```

**Vérifications :**
- Kubernetes redémarre-t-il le pod ?
- Temps de redémarrage ?
- Service interrompu pour les autres replicas ?

**Attendu :**
- Pod redémarré en 10-30 secondes
- Autres replicas continuent de servir le trafic

### Tests Réguliers

**Fréquence recommandée :**
- **Tests basiques (arrêt 1 nœud)** : Mensuel
- **Tests avancés (chaos engineering)** : Trimestriel
- **Tests de restauration complète** : Semestriel

**Documenter chaque test :**
```markdown
# Test de Redondance - 2025-01-15

## Test Effectué
Arrêt du nœud worker2 pendant 10 minutes

## Résultats
- ✅ Pods redistribués en 1m30s
- ✅ Aucune interruption de service
- ✅ Alertes Prometheus déclenchées
- ⚠️ Un pod en CrashLoopBackOff (bug indépendant)

## Actions
- Investiguer le CrashLoopBackOff
- RAS sur la redondance

## Temps de Rétablissement
- Redémarrage de worker2 : 2 minutes
- Rejoindre le cluster : 1 minute
- Pods retournent sur worker2 : 5 minutes
- Total : 8 minutes

## Conclusion
✅ Redondance fonctionnelle
```

## Surveillance de la Redondance

### Métriques Clés

**1. Nombre de nœuds Ready**
```promql
count(kube_node_status_condition{condition="Ready", status="true"})
```

**Alerte si < 3** (cluster HA minimal)

**2. Nombre de replicas disponibles par Deployment**
```promql
kube_deployment_status_replicas_available < kube_deployment_spec_replicas
```

**Alerte si disponible < désiré**

**3. Pods évacués (evicted)**
```promql
kube_pod_status_phase{phase="Failed"} > 0
```

**4. Utilisation disque par nœud**
```promql
(node_filesystem_size_bytes - node_filesystem_free_bytes) / node_filesystem_size_bytes > 0.85
```

**Alerte si > 85%**

**5. Santé Longhorn**
```promql
longhorn_volume_robustness != 1
```

**Alerte si volume degraded**

### Dashboards Recommandés

**Grafana - Dashboard "Cluster Health"**
- Statut de chaque nœud
- Nombre de pods par nœud
- Utilisation CPU/RAM/Disk par nœud
- Répartition des pods entre zones
- État des volumes Longhorn

**Grafana - Dashboard "Application Resilience"**
- Nombre de replicas par Deployment
- Pod restarts (dernières 24h)
- Pod evictions
- Temps de réponse applicatif
- Taux d'erreur

## Bonnes Pratiques

### 1. Documentation de l'Architecture

**Maintenir un schéma à jour :**
```markdown
# Architecture Cluster K8s

## Infrastructure
- 3 nœuds control+worker (node1-3)
- 2 workers purs (worker1-2)
- 2 switches (active-backup bonding)
- 1 UPS (autonomie 30 min)

## Redondance
- Control plane : 3 nœuds (tolère 1 panne)
- Stockage : 3 replicas Longhorn
- Applications critiques : 3-5 replicas
- Backups : Quotidien (7j) + Hebdo (4 semaines)

## SPOF Identifiés
- ❌ Switch 1 (en cours : ajout Switch 2)
- ✅ Aucun autre SPOF connu

## Dernière MAJ
2025-01-15 par Jean
```

### 2. Principe du "Cattle, Not Pets"

**Pets (animaux de compagnie) :**
- Serveurs nommés avec soin (prod-01, db-master)
- Configurés manuellement
- Irremplaçables

**Cattle (bétail) :**
- Serveurs numérotés (node1, node2, worker3)
- Configurés automatiquement (scripts, Ansible)
- Remplaçables à tout moment

**Objectif :** Tout doit pouvoir être reconstruit automatiquement.

### 3. Immutabilité

**Ne pas modifier les nœuds en place.**

**Mauvais :**
```bash
# Se connecter au nœud et modifier manuellement
ssh node1
sudo apt install custom-package
sudo nano /etc/config.conf
```

**Bon :**
```bash
# Modifier le script d'installation
# Reconstruire le nœud depuis zéro
# Ou utiliser Ansible/Terraform
```

**Avantage :** Si le nœud doit être reconstruit, vous savez exactement comment.

### 4. Versioning et GitOps

**Tout dans Git :**
- Manifestes YAML
- Scripts de configuration
- Documentation
- Procédures

**Workflow :**
```
Modification → Git commit → Git push → ArgoCD deploy
```

**Avantages :**
- Historique complet
- Rollback facile
- Collaboration
- Backup automatique

### 5. Tester la Redondance Avant d'en Avoir Besoin

**Ne découvrez pas que votre redondance ne fonctionne pas lors d'une vraie panne.**

**Tests réguliers = assurance qualité de votre redondance.**

### 6. Automatisation du Failover

**Éviter les actions manuelles lors d'une panne.**

**Kubernetes fait déjà beaucoup automatiquement :**
- Redémarrage de pods crashés
- Redistribution sur panne nœud
- Élection de nouveau leader

**Compléter avec :**
- Alertes automatiques (PagerDuty, Slack)
- Runbooks automatisés
- Scripts de failover testés

### 7. Mesurer et Améliorer

**KPIs de résilience :**
- **MTBF** (Mean Time Between Failures) : Temps moyen entre deux pannes
- **MTTR** (Mean Time To Recovery) : Temps moyen de récupération
- **Uptime** : % de disponibilité

**Objectif :**
- Augmenter MTBF (pannes moins fréquentes)
- Réduire MTTR (récupération plus rapide)
- Maximiser Uptime (> 99.9%)

**Calcul de l'uptime :**
```
Uptime = (Temps total - Temps d'indisponibilité) / Temps total × 100

Exemple sur 1 mois (30 jours) :
- Downtime total : 1 heure
- Uptime = (30×24 - 1) / (30×24) × 100 = 99.86%
```

**SLA standards :**
- 99% = 7h20min downtime/mois (acceptable pour dev)
- 99.9% = 43min downtime/mois (production standard)
- 99.99% = 4min downtime/mois (production critique)
- 99.999% = 26 secondes/mois (mission-critique)

## Points Clés à Retenir

1. **Redondance = Ne jamais avoir un seul point de défaillance**
2. **N+1** : Toujours une ressource de plus que le minimum
3. **Niveaux de redondance** : Infrastructure, réseau, stockage, applications
4. **Active-Active** > Active-Passive (utilisation optimale)
5. **Replicas ≥ 2** pour toutes les applications critiques
6. **Anti-affinity** pour répartir les pods sur différents nœuds
7. **PDB** pour garantir un minimum de pods pendant les maintenances
8. **Tests réguliers** : Chaos engineering mensuel/trimestriel
9. **Documenter** : Architecture, SPOF, procédures
10. **Mesurer** : MTBF, MTTR, Uptime

## Prochaines Étapes

Vous maîtrisez maintenant les stratégies de redondance complètes ! Le prochain chapitre couvrira :

- **21.10** : Tests de failover et validation complète de la résilience

Avec une stratégie de redondance bien pensée et testée, votre cluster peut désormais affronter les pannes les plus courantes sans interruption de service ! 🚀

---

**Félicitations !** Vous savez maintenant concevoir et implémenter une architecture Kubernetes véritablement résiliente avec redondance à tous les niveaux. C'est la marque des architectures de production sérieuses !

⏭️ [Tests de failover](/21-multi-node-et-haute-disponibilite/10-tests-de-failover.md)
