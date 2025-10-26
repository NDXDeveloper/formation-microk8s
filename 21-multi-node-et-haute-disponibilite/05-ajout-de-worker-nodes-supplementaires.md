🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 21.5 Ajout de Worker Nodes Supplémentaires

## Introduction

Votre cluster HA est maintenant opérationnel avec 3 nœuds control plane. Mais que faire quand vos applications grandissent et que vous avez besoin de **plus de puissance de calcul** ? C'est là qu'interviennent les **worker nodes supplémentaires**.

Dans ce chapitre, nous allons apprendre à :
- Ajouter des workers purs (sans control plane)
- Comprendre quand et pourquoi les ajouter
- Gérer la capacité et le placement des pods
- Optimiser les ressources du cluster
- Scaler horizontalement votre infrastructure

L'ajout de workers est l'un des grands avantages de Kubernetes : vous pouvez augmenter la capacité de votre cluster **sans toucher au control plane**, qui reste stable et sécurisé.

## Nœuds Mixtes vs Workers Purs : Rappel

### Nœuds Mixtes (Control Plane + Worker)

**Ce que vous avez actuellement (3 nœuds HA) :**
```
┌──────────────┐  ┌──────────────┐  ┌──────────────┐
│   Node 1     │  │   Node 2     │  │   Node 3     │
│              │  │              │  │              │
│ Control      │  │ Control      │  │ Control      │
│ Plane        │  │ Plane        │  │ Plane        │
│ + Worker     │  │ + Worker     │  │ + Worker     │
└──────────────┘  └──────────────┘  └──────────────┘
```

**Avantages :**
- Simple à gérer
- Utilisation optimale des ressources (pas de gaspillage)
- Parfait pour petits clusters (3-7 nœuds)

**Inconvénients :**
- Les pods applicatifs partagent les ressources avec le control plane
- Si forte charge applicative, le control plane peut être impacté
- Moins d'isolation

### Workers Purs (Worker Only)

**Avec workers supplémentaires :**
```
┌──────────────┐  ┌──────────────┐  ┌──────────────┐
│   Node 1     │  │   Node 2     │  │   Node 3     │
│              │  │              │  │              │
│ Control      │  │ Control      │  │ Control      │
│ Plane        │  │ Plane        │  │ Plane        │
│ + Worker     │  │ + Worker     │  │ + Worker     │
└──────────────┘  └──────────────┘  └──────────────┘
        │                 │                 │
        └─────────────────┼─────────────────┘
                          │
        ┌─────────────────┴─────────────────┐
        │                                   │
   ┌────▼────────┐  ┌──────────────┐  ┌─────▼───────┐
   │  Worker 1   │  │  Worker 2    │  │  Worker 3   │
   │             │  │              │  │             │
   │  Worker     │  │  Worker      │  │  Worker     │
   │  Only       │  │  Only        │  │  Only       │
   └─────────────┘  └──────────────┘  └─────────────┘
```

**Avantages :**
- Isolation : control plane dédié, pas impacté par la charge applicative
- Scalabilité : ajout facile de capacité sans complexifier le control plane
- Moins de ressources consommées par worker (pas de Dqlite, API server, etc.)
- Sécurité : control plane séparé physiquement

**Inconvénients :**
- Plus de machines nécessaires
- Légèrement plus complexe à gérer

## Pourquoi Ajouter des Workers Supplémentaires ?

### 1. Augmentation de la Capacité

**Scénario typique :**
Vous avez un cluster 3 nœuds avec :
- 4 CPU et 8 GB RAM par nœud
- Total : 12 CPU, 24 GB RAM

Vos applications grandissent et vous atteignez **80% d'utilisation** :
```bash
microk8s kubectl top nodes
```
```
NAME    CPU    CPU%   MEMORY   MEMORY%
node1   3200m  80%    6.4Gi    80%
node2   3100m  77%    6.2Gi    77%
node3   3300m  82%    6.6Gi    82%
```

**Solution :** Ajouter 2 workers avec 4 CPU et 8 GB chacun.
- Nouveau total : 20 CPU, 40 GB RAM
- Utilisation redescend à ~50%

### 2. Isolation des Workloads

**Cas d'usage :**
- Séparer les workloads critiques (bases de données) des workloads moins critiques (batch jobs)
- Dédier certains workers à certaines applications
- Isolation par tenant (si multi-tenant)

**Exemple :**
```
Control Plane (3 nœuds) : Infrastructure système
Worker 1-2              : Applications web
Worker 3-4              : Bases de données (SSD rapides)
Worker 5-6              : Jobs de traitement (beaucoup de CPU)
```

### 3. Zones de Disponibilité

**Dans un datacenter ou cloud :**
Répartir les workers sur plusieurs zones physiques :
```
Zone A : node1 (control) + worker1
Zone B : node2 (control) + worker2
Zone C : node3 (control) + worker3
```

Si une zone tombe (panne électrique, réseau), les autres continuent.

### 4. Scaling Horizontal vs Vertical

**Vertical Scaling (Scale Up) :**
- Remplacer un serveur 4 CPU / 8 GB par un 8 CPU / 16 GB
- ❌ Nécessite downtime
- ❌ Limite physique (max 128 CPU sur une machine)
- ❌ Plus cher (serveurs puissants = prix exponentiels)

**Horizontal Scaling (Scale Out) :**
- Ajouter un worker supplémentaire 4 CPU / 8 GB
- ✅ Pas de downtime
- ✅ Quasi illimité (ajoutez autant de workers que nécessaire)
- ✅ Moins cher (serveurs standards)

**Kubernetes favorise le scaling horizontal !**

### 5. Machines Hétérogènes

Vous pouvez avoir des workers avec des specs différentes :
```
node1-3   : 4 CPU, 8 GB RAM  (control plane + worker)
worker1-2 : 4 CPU, 8 GB RAM  (workers standards)
worker3-4 : 8 CPU, 16 GB RAM (workers puissants pour DB)
worker5   : 2 CPU, 4 GB RAM  (worker léger pour monitoring)
```

Kubernetes s'adapte et schedule les pods en fonction des ressources disponibles.

## Prérequis pour un Worker Supplémentaire

### 1. Ressources Matérielles

**Minimum par worker :**
```
CPU     : 2 cœurs
RAM     : 4 GB
Stockage: 20 GB
Réseau  : 1 Gbps
```

**Recommandé par worker :**
```
CPU     : 4 cœurs
RAM     : 8 GB
Stockage: 40 GB (SSD)
Réseau  : 1 Gbps
```

**Note :** Un worker pur consomme **moins** de ressources qu'un nœud mixte, car il n'exécute pas le control plane.

### 2. Configuration Réseau

**Connectivité avec les nœuds existants :**
```bash
# Depuis le nouveau worker, pinger les nœuds existants
ping -c 3 192.168.1.10  # node1
ping -c 3 192.168.1.11  # node2
ping -c 3 192.168.1.12  # node3

# Vérifier la latence (< 10ms recommandé)
ping -c 10 192.168.1.10 | grep avg
```

**Adresse IP statique :**
```
node1   : 192.168.1.10  (control plane)
node2   : 192.168.1.11  (control plane)
node3   : 192.168.1.12  (control plane)
worker1 : 192.168.1.20  ← Nouvelle IP statique
worker2 : 192.168.1.21  ← Nouvelle IP statique
```

**Firewall :**
```bash
# Sur worker1, autoriser le trafic depuis les nœuds du cluster
sudo ufw allow from 192.168.1.0/24

# Sur les nœuds existants, autoriser worker1
sudo ufw allow from 192.168.1.20
sudo ufw allow from 192.168.1.21
```

### 3. Hostname Unique

```bash
# Sur worker1
sudo hostnamectl set-hostname worker1

# Vérifier
hostname
```

**Ajouter dans /etc/hosts (optionnel mais recommandé) :**
```bash
sudo nano /etc/hosts
```
```
192.168.1.10    node1
192.168.1.11    node2
192.168.1.12    node3
192.168.1.20    worker1
192.168.1.21    worker2
```

### 4. Même Version de MicroK8s

**Sur le worker, installer la même version que le cluster :**
```bash
# Vérifier la version sur le cluster existant
# (depuis node1, node2 ou node3)
microk8s version

# Sur worker1, installer la même version
sudo snap install microk8s --classic --channel=1.28/stable

# Ajouter l'utilisateur au groupe
sudo usermod -a -G microk8s $USER
sudo chown -f -R $USER ~/.kube
newgrp microk8s

# Vérifier
microk8s version
```

## Ajouter un Worker : Processus Détaillé

### Étape 1 : Préparer le Worker

**Sur la nouvelle machine (worker1) :**

```bash
# 1. Mettre à jour le système
sudo apt update && sudo apt upgrade -y

# 2. Configurer l'IP statique (si DHCP)
# Exemple avec netplan (Ubuntu)
sudo nano /etc/netplan/01-netcfg.yaml
```

```yaml
network:
  version: 2
  ethernets:
    eth0:
      addresses:
        - 192.168.1.20/24
      gateway4: 192.168.1.1
      nameservers:
        addresses: [8.8.8.8, 8.8.4.4]
```

```bash
sudo netplan apply

# 3. Configurer le hostname
sudo hostnamectl set-hostname worker1

# 4. Installer MicroK8s
sudo snap install microk8s --classic --channel=1.28/stable

# 5. Configuration utilisateur
sudo usermod -a -G microk8s $USER
sudo chown -f -R $USER ~/.kube
newgrp microk8s

# 6. Attendre que MicroK8s soit prêt
microk8s status --wait-ready
```

**À ce stade :** worker1 a MicroK8s installé, mais tourne en standalone (pas encore connecté au cluster).

### Étape 2 : Générer le Token Worker

**Sur un nœud existant du cluster (node1, node2 ou node3) :**

```bash
microk8s add-node --worker
```

**Sortie attendue :**
```
From the node you wish to join to this cluster, run the following:
microk8s join 192.168.1.10:25000/abcd1234efgh5678ijkl9012mnop3456qrst7890 --worker

Use the '--worker' flag to join a node as a worker not running the control plane.
```

**Explication :**
- `--worker` : indique que c'est un worker pur (pas de control plane)
- Le token généré est différent d'un token normal
- Le token est valide 24 heures

**Copier la commande complète.**

### Étape 3 : Joindre le Cluster comme Worker

**Sur worker1 :**

```bash
microk8s join 192.168.1.10:25000/abcd1234efgh5678ijkl9012mnop3456qrst7890 --worker
```

**Ce qui se passe :**
```
Contacting cluster at 192.168.1.10...
Waiting for this node to finish joining the cluster. .. .. ..
Successfully joined the cluster as a worker node.
```

**En arrière-plan, MicroK8s :**
1. Authentifie le worker avec le token
2. Télécharge la configuration du cluster
3. Démarre le kubelet (agent du worker)
4. N'installe PAS Dqlite ni l'API Server (c'est un worker pur)
5. Enregistre le nœud dans l'API Server du cluster

**Temps d'attente :** 30 secondes à 2 minutes.

### Étape 4 : Vérifier que le Worker est Ajouté

**Sur n'importe quel nœud du cluster :**

```bash
microk8s kubectl get nodes
```

**Sortie attendue :**
```
NAME      STATUS   ROLES    AGE     VERSION
node1     Ready    <none>   5d      v1.28.3
node2     Ready    <none>   5d      v1.28.3
node3     Ready    <none>   5d      v1.28.3
worker1   Ready    <none>   2m      v1.28.3
```

**Vérifier les détails du worker :**
```bash
microk8s kubectl get node worker1 -o wide
```

```
NAME      STATUS   ROLES    AGE   VERSION   INTERNAL-IP     OS-IMAGE
worker1   Ready    <none>   2m    v1.28.3   192.168.1.20    Ubuntu 22.04
```

**Vérifier que worker1 n'est PAS dans Dqlite :**
```bash
microk8s status
```

```
microk8s is running
high-availability: yes
  datastore master nodes: 192.168.1.10:19001 192.168.1.11:19001 192.168.1.12:19001
  datastore standby nodes: none
```

worker1 n'apparaît **pas** dans la liste des "datastore master nodes". C'est normal et attendu ! ✅

### Étape 5 : Labelliser le Worker (Optionnel mais Recommandé)

**Ajouter des labels pour identifier le worker :**

```bash
# Label de rôle
microk8s kubectl label node worker1 node-role.kubernetes.io/worker=true

# Label de type
microk8s kubectl label node worker1 node-type=worker

# Label de zone (si applicable)
microk8s kubectl label node worker1 topology.kubernetes.io/zone=zone-a

# Vérifier
microk8s kubectl get nodes --show-labels
```

Ces labels seront utiles pour :
- Node selectors
- Affinity rules
- Visualisation dans les dashboards

## Ajouter Plusieurs Workers

### Ajouter Worker 2

**Répéter le processus :**

1. **Sur un nœud existant, générer un nouveau token :**
```bash
microk8s add-node --worker
```

2. **Sur worker2 (après installation MicroK8s) :**
```bash
microk8s join 192.168.1.10:25000/<nouveau_token> --worker
```

3. **Vérifier :**
```bash
microk8s kubectl get nodes
```

```
NAME      STATUS   ROLES    AGE
node1     Ready    <none>   5d
node2     Ready    <none>   5d
node3     Ready    <none>   5d
worker1   Ready    <none>   10m
worker2   Ready    <none>   1m
```

### Ajouter Worker 3, 4, 5...

**Même processus pour chaque worker :**
- Générer un token unique à chaque fois
- Ne pas réutiliser un ancien token
- Vérifier que chaque worker passe en "Ready"

**Combien de workers peut-on ajouter ?**
- MicroK8s supporte jusqu'à **20-30 workers** confortablement
- Au-delà, envisager d'autres solutions (kubeadm, cloud managé)

## Distribution des Pods sur les Workers

### Comportement par Défaut

Une fois les workers ajoutés, Kubernetes **distribue automatiquement** les nouveaux pods :

```bash
# Créer un deployment avec 10 réplicas
microk8s kubectl create deployment nginx-test --image=nginx --replicas=10

# Voir où les pods sont schedulés
microk8s kubectl get pods -o wide
```

**Résultat typique (cluster avec 3 control+worker + 2 workers purs) :**
```
NAME                 NODE      STATUS
nginx-test-xxx-1     node1     Running
nginx-test-xxx-2     node2     Running
nginx-test-xxx-3     node3     Running
nginx-test-xxx-4     worker1   Running
nginx-test-xxx-5     worker1   Running
nginx-test-xxx-6     worker2   Running
nginx-test-xxx-7     worker2   Running
nginx-test-xxx-8     node1     Running
nginx-test-xxx-9     worker1   Running
nginx-test-xxx-10    node3     Running
```

Kubernetes répartit intelligemment les pods sur **tous** les nœuds disponibles.

### Le Scheduler Kubernetes

**Comment Kubernetes décide où placer un pod ?**

Le **Scheduler** évalue plusieurs critères :
1. **Ressources disponibles** (CPU, RAM)
2. **Affinity / Anti-affinity** (préférences de placement)
3. **Taints / Tolerations** (restrictions)
4. **Priorités** des pods
5. **Topology spread** (répartition géographique)

**Algorithme simplifié :**
```
Pour chaque pod à scheduler:
  1. Filtrer les nœuds (supprimer ceux qui ne peuvent pas héberger le pod)
  2. Scorer les nœuds restants (donner une note à chaque nœud)
  3. Choisir le nœud avec le meilleur score
  4. Attribuer le pod à ce nœud
```

### Forcer le Placement sur un Worker

**Option 1 : Node Selector (Simple)**

```yaml
# deployment-on-worker.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app-on-worker
spec:
  replicas: 3
  selector:
    matchLabels:
      app: myapp
  template:
    metadata:
      labels:
        app: myapp
    spec:
      nodeSelector:
        node-type: worker    # ← Forcer sur les workers uniquement
      containers:
      - name: nginx
        image: nginx
```

```bash
microk8s kubectl apply -f deployment-on-worker.yaml

# Vérifier que les pods sont bien sur worker1 et worker2
microk8s kubectl get pods -o wide
```

**Option 2 : Node Affinity (Plus Flexible)**

```yaml
# deployment-prefer-worker.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app-prefer-worker
spec:
  replicas: 5
  selector:
    matchLabels:
      app: myapp2
  template:
    metadata:
      labels:
        app: myapp2
    spec:
      affinity:
        nodeAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
          - weight: 100
            preference:
              matchExpressions:
              - key: node-type
                operator: In
                values:
                - worker
      containers:
      - name: nginx
        image: nginx
```

**Différence :**
- `nodeSelector` : **Obligatoire** (hard requirement)
- `nodeAffinity preferred` : **Préféré** mais pas obligatoire (soft requirement)

### Exclure les Control Plane (Taint)

Si vous voulez que les workers purs hébergent **uniquement** les pods applicatifs (pas les pods système), vous pouvez "tainter" les nœuds control plane :

```bash
# Empêcher les pods applicatifs sur les nœuds control plane
microk8s kubectl taint nodes node1 node-role.kubernetes.io/control-plane=:NoSchedule
microk8s kubectl taint nodes node2 node-role.kubernetes.io/control-plane=:NoSchedule
microk8s kubectl taint nodes node3 node-role.kubernetes.io/control-plane=:NoSchedule
```

**Résultat :**
- Les nouveaux pods seront schedulés **uniquement** sur worker1, worker2, etc.
- Les pods système (avec tolerations) peuvent toujours tourner sur les control plane
- Les pods existants sur control plane ne sont **pas** évacués

**Retirer le taint (si besoin) :**
```bash
microk8s kubectl taint nodes node1 node-role.kubernetes.io/control-plane-
```

## Gestion de la Capacité du Cluster

### Surveiller l'Utilisation des Ressources

**Voir l'utilisation par nœud :**
```bash
microk8s kubectl top nodes
```

```
NAME      CPU(cores)   CPU%   MEMORY(bytes)   MEMORY%
node1     1200m        30%    3.2Gi           40%
node2     1100m        27%    3.0Gi           37%
node3     1300m        32%    3.4Gi           42%
worker1   800m         20%    2.1Gi           26%
worker2   700m         17%    1.9Gi           23%
```

**Analyse :**
- Les control plane (node1-3) sont plus chargés (control plane + apps)
- Les workers purs (worker1-2) sont moins chargés
- **Distribution saine** : chaque nœud utilise 20-40% des ressources

### Capacité Totale du Cluster

**Calculer la capacité :**
```bash
# Capacité CPU totale (en millicores)
microk8s kubectl get nodes -o json | jq '.items[] | .status.capacity.cpu' | awk '{sum+=$1} END {print sum " cores"}'

# Capacité RAM totale
microk8s kubectl get nodes -o json | jq '.items[] | .status.capacity.memory'
```

**Exemple de sortie :**
```
Cluster 3 control+worker + 2 workers:
- CPU: 5 nœuds × 4 cores = 20 cores
- RAM: 5 nœuds × 8 GB = 40 GB
```

### Requests vs Limits

**Concepts importants :**

**Requests (Demandes) :**
- Ressources **garanties** pour le pod
- Le Scheduler vérifie que le nœud a les requests disponibles avant de placer le pod
- Exemple : `requests: cpu: 500m, memory: 512Mi`

**Limits (Limites) :**
- Ressources **maximales** que le pod peut utiliser
- Si dépassé, le pod peut être tué (OOMKilled pour la RAM)
- Exemple : `limits: cpu: 1000m, memory: 1Gi`

**Exemple de deployment avec requests/limits :**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app-with-resources
spec:
  replicas: 3
  selector:
    matchLabels:
      app: myapp
  template:
    metadata:
      labels:
        app: myapp
    spec:
      containers:
      - name: app
        image: nginx
        resources:
          requests:
            cpu: "250m"       # 0.25 CPU garanti
            memory: "256Mi"   # 256 MB garantis
          limits:
            cpu: "500m"       # Max 0.5 CPU
            memory: "512Mi"   # Max 512 MB
```

**Bonnes pratiques :**
- Toujours définir des **requests** (permet au Scheduler de bien placer les pods)
- Définir des **limits** pour éviter qu'un pod consomme tout
- Ratio limits/requests typique : 2x (exemple: request 250m, limit 500m)

### Seuil d'Alerte et Scaling

**Règles de thumb :**
- **< 60% d'utilisation** : Cluster confortable
- **60-75%** : OK, mais surveiller
- **75-85%** : Zone d'alerte, prévoir d'ajouter des workers
- **> 85%** : Critique, ajouter des workers rapidement

**Quand ajouter un worker ?**
```bash
# Si la moyenne d'utilisation dépasse 75% pendant plusieurs heures
microk8s kubectl top nodes
```

Si vous voyez :
```
NAME      CPU%   MEMORY%
node1     78%    82%
node2     80%    79%
node3     76%    81%
worker1   75%    77%
worker2   77%    78%
```

**→ Il est temps d'ajouter worker3 !**

## Stratégies de Placement Avancées

### 1. Topology Spread Constraints

**Répartir équitablement les pods sur les workers :**

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app-spread
spec:
  replicas: 6
  selector:
    matchLabels:
      app: myapp
  template:
    metadata:
      labels:
        app: myapp
    spec:
      topologySpreadConstraints:
      - maxSkew: 1
        topologyKey: kubernetes.io/hostname
        whenUnsatisfiable: DoNotSchedule
        labelSelector:
          matchLabels:
            app: myapp
      containers:
      - name: nginx
        image: nginx
```

**Résultat :**
- 6 réplicas réparties sur 5 nœuds (3 control+worker + 2 workers)
- Chaque nœud héberge 1 ou 2 pods (écart max = 1)

### 2. Pod Anti-Affinity

**Éviter de placer 2 réplicas du même pod sur le même nœud :**

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app-antiaffinity
spec:
  replicas: 5
  selector:
    matchLabels:
      app: critical-app
  template:
    metadata:
      labels:
        app: critical-app
    spec:
      affinity:
        podAntiAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
          - labelSelector:
              matchExpressions:
              - key: app
                operator: In
                values:
                - critical-app
            topologyKey: kubernetes.io/hostname
      containers:
      - name: app
        image: myapp:v1
```

**Résultat :**
- Chaque réplica est sur un nœud différent
- Garantit la résilience (si un nœud tombe, pas tous les pods)

### 3. Dédier des Workers à Certaines Apps

**Scénario :** Vous avez des workers avec GPU pour du machine learning.

```bash
# Labelliser les workers avec GPU
microk8s kubectl label node worker3 gpu=nvidia-rtx-3090
microk8s kubectl label node worker4 gpu=nvidia-rtx-3090
```

**Dans votre deployment ML :**
```yaml
spec:
  template:
    spec:
      nodeSelector:
        gpu: nvidia-rtx-3090
```

Les pods ML seront schedulés **uniquement** sur worker3 et worker4.

## Retirer un Worker du Cluster

### Méthode Propre (Graceful)

**1. Drainer le worker (évacuer les pods) :**
```bash
microk8s kubectl drain worker1 --ignore-daemonsets --delete-emptydir-data
```

**Ce qui se passe :**
- worker1 est marqué comme "non-schedulable"
- Tous les pods sur worker1 sont gracefully terminés
- Ils sont recréés sur d'autres nœuds
- Les DaemonSets restent (d'où `--ignore-daemonsets`)

**2. Retirer le worker du cluster :**
```bash
# Depuis n'importe quel nœud du cluster
microk8s remove-node worker1
```

**3. Sur worker1, faire leave (optionnel) :**
```bash
# Sur worker1
microk8s leave
```

**4. Vérifier :**
```bash
microk8s kubectl get nodes
```

worker1 n'apparaît plus dans la liste.

### Méthode Forcée (Nœud Mort)

Si le worker est mort (panne matérielle, inaccessible) :

```bash
# Forcer la suppression
microk8s remove-node worker1 --force
```

Les pods qui étaient sur worker1 sont automatiquement redémarrés ailleurs après ~5 minutes (grace period).

## Monitoring des Workers

### Commandes Essentielles

**État de tous les nœuds :**
```bash
watch microk8s kubectl get nodes -o wide
```

**Utilisation CPU/RAM par nœud :**
```bash
watch microk8s kubectl top nodes
```

**Pods par nœud :**
```bash
microk8s kubectl get pods --all-namespaces -o wide --sort-by=.spec.nodeName
```

**Compter les pods par nœud :**
```bash
for node in $(microk8s kubectl get nodes -o name); do
  echo "$node: $(microk8s kubectl get pods --all-namespaces --field-selector spec.nodeName=$(basename $node) --no-headers | wc -l) pods"
done
```

### Dashboard Kubernetes

**Activer le dashboard (si pas déjà fait) :**
```bash
microk8s enable dashboard
```

**Accéder :**
```bash
# Port-forward
microk8s kubectl port-forward -n kube-system service/kubernetes-dashboard 10443:443

# Obtenir le token
microk8s kubectl describe secret -n kube-system microk8s-dashboard-token
```

Ouvrez https://localhost:10443 dans votre navigateur.

**Visualisation :**
- Vue d'ensemble des nœuds
- Utilisation des ressources en temps réel
- Pods par nœud

### Prometheus + Grafana (Recommandé)

**Activer Prometheus :**
```bash
microk8s enable prometheus
```

**Dashboards utiles :**
- "Kubernetes / Compute Resources / Cluster"
- "Kubernetes / Compute Resources / Node (Pods)"
- "Node Exporter / Nodes"

**Voir la distribution des pods par nœud :**
Dans Grafana, requête PromQL :
```promql
count(kube_pod_info) by (node)
```

## Bonnes Pratiques pour les Workers

### 1. Standardiser les Workers

**Recommandation :** Utilisez des workers avec les **mêmes specs** autant que possible.

**Avantages :**
- Prédictibilité (chaque worker peut héberger la même charge)
- Simplification de la gestion
- Facilite les calculs de capacité

**Exceptions acceptables :**
- Workers GPU pour ML
- Workers avec stockage SSD rapide pour DB

### 2. Documenter les Workers

**Maintenir un inventaire :**
```markdown
# Inventaire Cluster

## Control Plane
- node1: 192.168.1.10 - 4 CPU, 8 GB RAM - Control+Worker
- node2: 192.168.1.11 - 4 CPU, 8 GB RAM - Control+Worker
- node3: 192.168.1.12 - 4 CPU, 8 GB RAM - Control+Worker

## Workers Purs
- worker1: 192.168.1.20 - 4 CPU, 8 GB RAM - Worker - Apps générales
- worker2: 192.168.1.21 - 4 CPU, 8 GB RAM - Worker - Apps générales
- worker3: 192.168.1.22 - 8 CPU, 16 GB RAM - Worker - Bases de données
- worker4: 192.168.1.23 - 8 CPU, 16 GB RAM - Worker - Bases de données

## Capacité Totale
- CPU: 36 cores (12 control + 24 workers)
- RAM: 72 GB (24 control + 48 workers)
```

### 3. Labelliser et Organiser

**Système de labels cohérent :**
```bash
# Rôle
microk8s kubectl label node worker1 node-role.kubernetes.io/worker=true

# Type de workload
microk8s kubectl label node worker1 workload-type=general
microk8s kubectl label node worker3 workload-type=database

# Environnement (si multi-env sur même cluster)
microk8s kubectl label node worker1 env=production
microk8s kubectl label node worker5 env=staging

# Zone physique
microk8s kubectl label node worker1 zone=datacenter-a
```

### 4. Requests et Limits Partout

**Ne jamais déployer sans requests :**
```yaml
# ❌ MAUVAIS
containers:
- name: app
  image: myapp

# ✅ BON
containers:
- name: app
  image: myapp
  resources:
    requests:
      cpu: "250m"
      memory: "256Mi"
    limits:
      cpu: "500m"
      memory: "512Mi"
```

Sans requests, le Scheduler ne peut pas bien répartir la charge.

### 5. Prévoir de la Marge

**Règle du 70% :**
Ne planifiez pas d'utiliser plus de **70% des ressources** en conditions normales.

**Pourquoi ?**
- Pics de charge inattendus
- Redéploiements (besoin de place pour les nouveaux pods)
- Maintenance (drain d'un worker)

**Exemple :**
Si vous avez besoin de 16 cores pour vos apps :
- 16 / 0.7 = ~23 cores nécessaires
- Avec des workers de 4 cores : 23 / 4 = 6 workers

### 6. Automatiser l'Ajout (Avancé)

Pour les environnements cloud, vous pouvez scripter l'ajout de workers :

```bash
#!/bin/bash
# add-worker.sh

# 1. Créer la VM (Terraform, cloud CLI, etc.)
# 2. Installer MicroK8s
# 3. Générer token et joindre
TOKEN=$(microk8s add-node --worker | grep "microk8s join" | awk '{print $3" "$4}')
ssh worker-new "sudo snap install microk8s --classic && microk8s $TOKEN"
```

## Scénarios d'Usage Pratiques

### Scénario 1 : Startup en Croissance

**Phase 1 (Démarrage) :**
```
3 nœuds control+worker (12 CPU, 24 GB)
```

**Phase 2 (Croissance) :**
```
3 control+worker + 2 workers purs (20 CPU, 40 GB)
Coût: +2 serveurs
```

**Phase 3 (Scale Up) :**
```
3 control+worker + 5 workers purs (32 CPU, 64 GB)
Coût: +3 serveurs
```

**Phase 4 (Mature) :**
```
3 control+worker + 10 workers purs (52 CPU, 104 GB)
Coût: +5 serveurs
```

### Scénario 2 : Homelab Multi-Usage

**Configuration :**
```
3 nœuds control+worker : Infrastructure système
worker1-2             : Services web (Nextcloud, Wiki, etc.)
worker3               : Media server (Plex, Jellyfin)
worker4               : Bases de données
```

Isolation par type d'application = meilleure gestion.

### Scénario 3 : Environnements Séparés

**Sur le même cluster physique :**
```
worker1-2 : namespace "production" (taints)
worker3-4 : namespace "staging" (taints)
worker5   : namespace "development"
```

Avec des taints et tolerations, vous isolez les environnements.

## Limites et Considérations

### Limites de MicroK8s

**Scalabilité recommandée :**
- **Optimal** : 3-10 nœuds (control + workers)
- **Maximum raisonnable** : 20-30 nœuds total
- **Au-delà** : Envisager Kubernetes "standard" (kubeadm) ou managé (EKS, GKE, AKS)

**Pourquoi ?**
- Dqlite optimisé pour petits clusters
- Au-delà de 30 nœuds, etcd est préférable

### Coûts et ROI

**Calcul du coût :**
```
1 worker = 1 serveur = €X/mois (si cloud) ou coût matériel + électricité

Exemple cloud (DigitalOcean) :
- 4 CPU, 8 GB = ~$48/mois
- 10 workers = $480/mois

Exemple on-premise :
- Mini PC 4 CPU, 8 GB = €300 + €5/mois électricité
- 10 workers = €3000 initial + €50/mois
- Amortissement sur 3 ans : ~€133/mois
```

**Optimiser le ROI :**
- N'ajoutez des workers que quand nécessaire
- Utilisez bien les resources (requests/limits)
- Surveillez l'utilisation réelle

## Points Clés à Retenir

1. **Workers purs** = nœuds sans control plane, dédiés aux workloads
2. **Scaling horizontal** : Ajoutez des workers plutôt que d'upgrader les machines
3. **Token worker** : `microk8s add-node --worker` sur un nœud existant
4. **Join worker** : `microk8s join <token> --worker` sur le nouveau nœud
5. **Isolation** : Workers purs isolent le control plane des charges applicatives
6. **Labels** : Organisez vos workers avec des labels cohérents
7. **Requests/Limits** : Toujours définir pour une bonne répartition
8. **Monitoring** : `kubectl top nodes` et Prometheus/Grafana
9. **Capacité** : Gardez 30% de marge pour les pics et maintenances
10. **Limites** : MicroK8s optimal jusqu'à ~20-30 nœuds

## Prochaines Étapes

Maintenant que vous savez gérer les workers, les prochains chapitres couvriront :

- **21.6** : Load balancing entre nœuds avec MetalLB
- **21.7** : Stockage distribué pour partager les données entre workers
- **21.8** : Backup et restauration d'un cluster multi-node
- **21.9** : Stratégies de redondance avancées
- **21.10** : Tests de failover et chaos engineering

Vous maîtrisez maintenant l'art du scaling horizontal avec MicroK8s ! Vous pouvez faire grandir votre cluster au rythme de vos besoins, sans limite (ou presque) ! 🚀

---

**Félicitations !** Vous savez maintenant comment ajouter des workers supplémentaires pour scaler votre cluster Kubernetes horizontalement. C'est une compétence essentielle pour gérer la croissance et optimiser les ressources !

⏭️ [Load balancing entre nodes](/21-multi-node-et-haute-disponibilite/06-load-balancing-entre-nodes.md)
