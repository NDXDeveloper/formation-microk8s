🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 21.4 Configuration d'un Cluster HA (3+ Nœuds)

## Introduction

Vous savez maintenant comment ajouter des nœuds à un cluster MicroK8s. Dans ce chapitre, nous allons passer à l'étape supérieure : configurer un **cluster en haute disponibilité (HA) complet** avec 3 nœuds ou plus.

Un cluster HA est conçu pour **ne jamais s'arrêter**, même si un ou plusieurs serveurs tombent en panne. C'est la configuration que vous utiliseriez pour :
- Des applications critiques en production
- Un environnement de test réaliste
- Apprendre les concepts avancés de Kubernetes
- Héberger des services personnels avec redondance

Nous allons voir comment construire ce cluster de A à Z, le configurer correctement, et vérifier qu'il est vraiment résilient.

## Qu'est-ce qu'un Cluster HA ?

### Rappel des Concepts

Un cluster **Haute Disponibilité (High Availability)** est un cluster qui :
1. **Continue de fonctionner** même si un ou plusieurs nœuds tombent
2. **Redistribue automatiquement** les workloads en cas de panne
3. **Maintient le quorum** du control plane avec Dqlite
4. **Offre zéro ou très peu de downtime** lors des incidents

**Analogie avec un avion :**
Un avion de ligne a plusieurs moteurs. Si un moteur tombe en panne, les autres compensent et l'avion peut continuer son vol et atterrir en sécurité. Un cluster HA fonctionne de la même manière.

### Les Composants en HA

Dans un cluster MicroK8s HA, plusieurs composants sont redondants :

**1. Control Plane (Dqlite)**
- Répliqué sur 3 nœuds minimum
- Tolérance à la panne d'un nœud
- Élection automatique d'un nouveau Leader

**2. API Server**
- Actif sur chaque nœud control plane
- Load balancing automatique entre les instances
- N'importe quel API Server peut traiter vos requêtes

**3. Scheduler et Controllers**
- Actifs sur tous les nœuds control plane
- Élection de leader automatique
- Basculement en quelques secondes

**4. Workloads (Pods)**
- Répartis sur tous les nœuds workers
- Redémarrage automatique sur un autre nœud si panne
- Load balancing entre réplicas

## Pourquoi 3 Nœuds Minimum ?

### Le Concept de Quorum

Pour comprendre pourquoi 3 nœuds est le minimum, il faut comprendre le **quorum**.

**Quorum = Majorité simple**

Avec Dqlite (et l'algorithme Raft), pour qu'une décision soit prise, il faut l'accord de la **majorité** des nœuds.

**Calcul du quorum :**
```
Quorum = (Nombre de nœuds / 2) + 1
```

**Exemples :**
- 1 nœud : quorum = 1 (pas de HA)
- 2 nœuds : quorum = 2 (si 1 tombe, pas de quorum ❌)
- **3 nœuds : quorum = 2** (si 1 tombe, quorum maintenu ✅)
- 4 nœuds : quorum = 3 (si 1 tombe, quorum maintenu, mais pas mieux que 3)
- **5 nœuds : quorum = 3** (si 2 tombent, quorum maintenu ✅)
- 7 nœuds : quorum = 4 (si 3 tombent, quorum maintenu ✅)

### Tableau de Tolérance aux Pannes

| Nœuds | Quorum | Pannes tolérées | HA ? | Coût | Recommandation |
|-------|--------|-----------------|------|------|----------------|
| 1 | 1 | 0 | ❌ | Faible | Dev/Test uniquement |
| 2 | 2 | 0 | ❌ | Moyen | **À ÉVITER** (pire que 1) |
| **3** | 2 | **1** | ✅ | Moyen | **Configuration HA minimale** |
| 4 | 3 | 1 | ✅ | Élevé | ❌ Pas rentable (identique à 3) |
| **5** | 3 | **2** | ✅ | Élevé | **HA renforcée** |
| 6 | 4 | 2 | ✅ | Très élevé | ❌ Pas rentable (identique à 5) |
| **7** | 4 | **3** | ✅ | Très élevé | **HA maximale** |

**Règle d'or :** Utilisez toujours un **nombre impair** de nœuds (3, 5, 7).

### Pourquoi Jamais 2 Nœuds ?

Avec 2 nœuds :
- Quorum = 2
- Si 1 nœud tombe → quorum perdu (1/2 < 50%)
- Cluster en lecture seule (impossible de créer/modifier des ressources)

**C'est pire qu'avoir 1 seul nœud !** Vous consommez le double de ressources sans gain de résilience.

## Architecture Recommandée pour un Cluster HA

### Configuration 3 Nœuds (Recommandée pour Lab/PME)

```
┌─────────────────────────────────────────────────────────────────┐
│              CLUSTER MICROK8S HA - 3 NŒUDS                      │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│   ┌──────────────┐      ┌──────────────┐      ┌──────────────┐  │
│   │   Node 1     │      │   Node 2     │      │   Node 3     │  │
│   │ 192.168.1.10 │◄────►│ 192.168.1.11 │◄────►│ 192.168.1.12 │  │
│   │              │      │              │      │              │  │
│   │ Control      │      │ Control      │      │ Control      │  │
│   │ Plane        │      │ Plane        │      │ Plane        │  │
│   │ + Worker     │      │ + Worker     │      │ + Worker     │  │
│   │              │      │              │      │              │  │
│   │ Dqlite       │      │ Dqlite       │      │ Dqlite       │  │
│   │ (Leader)     │      │ (Follower)   │      │ (Follower)   │  │
│   │              │      │              │      │              │  │
│   │ 4 CPU        │      │ 4 CPU        │      │ 4 CPU        │  │
│   │ 8 GB RAM     │      │ 8 GB RAM     │      │ 8 GB RAM     │  │
│   └──────────────┘      └──────────────┘      └──────────────┘  │
│          │                      │                      │        │
│          └──────────────────────┴──────────────────────┘        │
│                      Réseau Gigabit (1 Gbps)                      │
└─────────────────────────────────────────────────────────────────┘

Tolérance aux pannes : 1 nœud peut tomber
Quorum : 2/3 nœuds nécessaires
```

**Caractéristiques :**
- Chaque nœud fait **control plane ET worker**
- Simplicité de gestion
- Ressources partagées entre control plane et applications
- **Idéal pour :** Labs personnels, PME, environnements de test

### Configuration 5 Nœuds (Production Renforcée)

```
┌────────────────────────────────────────────────────────────────┐
│              CLUSTER MICROK8S HA - 5 NŒUDS                     │
├────────────────────────────────────────────────────────────────┤
│                                                                │
│         CONTROL PLANE (3 nœuds)                                │
│   ┌──────────┐    ┌──────────┐    ┌──────────┐                 │
│   │  Node 1  │◄──►│  Node 2  │◄──►│  Node 3  │                 │
│   │ Control  │    │ Control  │    │ Control  │                 │
│   │ + Worker │    │ + Worker │    │ + Worker │                 │
│   └──────────┘    └──────────┘    └──────────┘                 │
│                                                                │
│         WORKERS PURS (2 nœuds)                                 │
│   ┌──────────┐                    ┌──────────┐                 │
│   │ Worker 1 │                    │ Worker 2 │                 │
│   │ Worker   │                    │ Worker   │                 │
│   │ Only     │                    │ Only     │                 │
│   └──────────┘                    └──────────┘                 │
└────────────────────────────────────────────────────────────────┘

Tolérance aux pannes : 2 nœuds peuvent tomber
Quorum : 3/5 nœuds nécessaires
```

**Caractéristiques :**
- 3 nœuds control plane + worker
- 2 workers purs supplémentaires
- Meilleure tolérance aux pannes (2 nœuds)
- **Idéal pour :** Production, applications critiques

## Prérequis pour un Cluster HA

### 1. Infrastructure Matérielle

**Configuration minimale par nœud (3 nœuds HA) :**
```
CPU     : 2 cœurs (4 recommandés)
RAM     : 4 GB (8 GB recommandés)
Stockage: 40 GB (SSD fortement recommandé)
Réseau  : 1 Gbps
```

**Total pour cluster 3 nœuds :**
```
CPU     : 6-12 cœurs
RAM     : 12-24 GB
Stockage: 120 GB
```

**Vérifier les ressources sur chaque nœud :**
```bash
# CPU
nproc

# RAM (disponible)
free -h

# Espace disque
df -h /

# Interface réseau (vitesse)
ethtool eth0 | grep Speed
```

### 2. Configuration Réseau

**Adresses IP statiques pour chaque nœud :**
```
Node 1: 192.168.1.10
Node 2: 192.168.1.11
Node 3: 192.168.1.12
```

**Résolution DNS (optionnel mais recommandé) :**

Ajouter dans `/etc/hosts` sur **chaque** nœud :
```bash
sudo nano /etc/hosts
```

```
192.168.1.10    node1    node1.cluster.local
192.168.1.11    node2    node2.cluster.local
192.168.1.12    node3    node3.cluster.local
```

**Vérifier la connectivité :**
```bash
# Depuis chaque nœud, pinger les autres
ping -c 3 node1
ping -c 3 node2
ping -c 3 node3

# Vérifier la latence (doit être < 10ms idéalement)
ping -c 10 node2 | grep avg
```

### 3. Ports et Firewall

**Ouvrir les ports nécessaires entre tous les nœuds :**

**Sur chaque nœud, autoriser les autres nœuds :**
```bash
# Sur Node 1
sudo ufw allow from 192.168.1.11
sudo ufw allow from 192.168.1.12

# Sur Node 2
sudo ufw allow from 192.168.1.10
sudo ufw allow from 192.168.1.12

# Sur Node 3
sudo ufw allow from 192.168.1.10
sudo ufw allow from 192.168.1.11
```

**Ou plus précisément (ports spécifiques) :**
```bash
# Ports essentiels pour cluster HA
sudo ufw allow from 192.168.1.0/24 to any port 16443    # API Server
sudo ufw allow from 192.168.1.0/24 to any port 25000    # Cluster agent
sudo ufw allow from 192.168.1.0/24 to any port 12379    # Dqlite
sudo ufw allow from 192.168.1.0/24 to any port 10250    # Kubelet
```

### 4. Synchronisation du Temps

**Critique pour Dqlite !** Tous les nœuds doivent avoir la même heure.

**Installer et configurer NTP :**
```bash
# Sur chaque nœud
sudo apt update
sudo apt install -y chrony

# Vérifier la synchronisation
timedatectl status

# Doit afficher "System clock synchronized: yes"
```

**Vérifier l'écart entre nœuds :**
```bash
# Sur Node 1
date

# Sur Node 2
date

# Sur Node 3
date
```

L'écart doit être de **moins de 1 seconde**.

### 5. Hostnames Uniques

**Sur chaque nœud, définir un hostname unique :**
```bash
# Sur Node 1
sudo hostnamectl set-hostname node1

# Sur Node 2
sudo hostnamectl set-hostname node2

# Sur Node 3
sudo hostnamectl set-hostname node3

# Vérifier
hostname
```

## Construction d'un Cluster HA 3 Nœuds - Étape par Étape

### Phase 1 : Installation de MicroK8s sur Tous les Nœuds

**Sur Node 1, Node 2 ET Node 3 (même version !) :**

```bash
# Mise à jour du système
sudo apt update && sudo apt upgrade -y

# Installation de MicroK8s (utiliser le même channel)
sudo snap install microk8s --classic --channel=1.28/stable

# Ajouter l'utilisateur au groupe
sudo usermod -a -G microk8s $USER
sudo chown -f -R $USER ~/.kube

# Recharger les groupes
newgrp microk8s

# Attendre que MicroK8s soit prêt
microk8s status --wait-ready
```

**Vérifier la version sur chaque nœud :**
```bash
microk8s version
# Doit afficher la même version partout
```

**À ce stade :**
- Node 1 : cluster standalone (1 nœud)
- Node 2 : cluster standalone (1 nœud)
- Node 3 : cluster standalone (1 nœud)

Ils ne se connaissent pas encore.

### Phase 2 : Ajouter Node 2 au Cluster

**Sur Node 1 :**
```bash
microk8s add-node
```

**Copier la commande affichée, exemple :**
```
microk8s join 192.168.1.10:25000/abc123def456ghi789jkl012mno345pqr678
```

**Sur Node 2 :**
```bash
microk8s join 192.168.1.10:25000/abc123def456ghi789jkl012mno345pqr678
```

**Attendre la confirmation :**
```
Contacting cluster at 192.168.1.10...
Waiting for this node to finish joining the cluster. .. .. ..
Successfully joined the cluster.
```

**Vérifier depuis n'importe quel nœud :**
```bash
microk8s kubectl get nodes
```

```
NAME    STATUS   ROLES    AGE     VERSION
node1   Ready    <none>   10m     v1.28.3
node2   Ready    <none>   1m      v1.28.3
```

**Vérifier Dqlite :**
```bash
microk8s status
```

```
microk8s is running
high-availability: yes
  datastore master nodes: 192.168.1.10:19001 192.168.1.11:19001
```

**Important :** À ce stade, vous avez 2 nœuds, mais **pas encore de HA** (quorum = 2, donc besoin des 2 nœuds).

### Phase 3 : Ajouter Node 3 au Cluster (HA Activée)

**Sur Node 1 ou Node 2 :**
```bash
microk8s add-node
```

**Copier le nouveau token (différent du précédent !), exemple :**
```
microk8s join 192.168.1.10:25000/xyz789uvw456rst123abc098def765mno432
```

**Sur Node 3 :**
```bash
microk8s join 192.168.1.10:25000/xyz789uvw456rst123abc098def765mno432
```

**Attendre la confirmation :**
```
Successfully joined the cluster.
```

**Vérifier les 3 nœuds :**
```bash
microk8s kubectl get nodes
```

```
NAME    STATUS   ROLES    AGE     VERSION
node1   Ready    <none>   15m     v1.28.3
node2   Ready    <none>   6m      v1.28.3
node3   Ready    <none>   30s     v1.28.3
```

**Vérifier la HA complète :**
```bash
microk8s status
```

```
microk8s is running
high-availability: yes
  datastore master nodes: 192.168.1.10:19001 192.168.1.11:19001 192.168.1.12:19001
  datastore standby nodes: none
```

**🎉 Vous avez maintenant un cluster HA fonctionnel !**

Quorum = 2/3 → Vous pouvez perdre 1 nœud sans perdre le cluster.

### Phase 4 : Activer les Addons Essentiels

**Sur n'importe quel nœud :**

```bash
# DNS (essentiel)
microk8s enable dns

# Stockage persistant
microk8s enable hostpath-storage

# Dashboard (optionnel, pour monitoring visuel)
microk8s enable dashboard

# Metrics Server (pour kubectl top)
microk8s enable metrics-server
```

**Vérifier que les addons sont activés partout :**
```bash
microk8s status
```

Les addons activés sur un nœud sont **automatiquement disponibles** sur tous les nœuds du cluster.

## Vérifications Post-Configuration

### 1. État des Nœuds

**Tous les nœuds doivent être Ready :**
```bash
microk8s kubectl get nodes -o wide
```

```
NAME    STATUS   ROLES    AGE   VERSION   INTERNAL-IP     OS-IMAGE
node1   Ready    <none>   20m   v1.28.3   192.168.1.10    Ubuntu 22.04
node2   Ready    <none>   11m   v1.28.3   192.168.1.11    Ubuntu 22.04
node3   Ready    <none>   5m    v1.28.3   192.168.1.12    Ubuntu 22.04
```

**Points à vérifier :**
- STATUS = Ready sur tous les nœuds
- VERSION identique
- INTERNAL-IP correcte

### 2. Pods Système Répartis

**Les pods système doivent être distribués :**
```bash
microk8s kubectl get pods -n kube-system -o wide
```

Vous devriez voir des pods sur différents nœuds :
```
NAME                    READY   STATUS    NODE
calico-node-abc         1/1     Running   node1
calico-node-def         1/1     Running   node2
calico-node-ghi         1/1     Running   node3
coredns-xyz             1/1     Running   node2
hostpath-provisioner    1/1     Running   node1
```

C'est normal et souhaitable : la charge système est répartie.

### 3. État de Dqlite

**Vérifier le status HA :**
```bash
microk8s status
```

```
high-availability: yes
  datastore master nodes: 192.168.1.10:19001 192.168.1.11:19001 192.168.1.12:19001
  datastore standby nodes: none
```

**Explication :**
- **master nodes** : Les 3 nœuds participent au quorum Dqlite
- **standby nodes** : Aucun (tous sont actifs dans le quorum)

### 4. Santé Générale

**Inspecter le cluster :**
```bash
microk8s inspect
```

Cette commande génère un rapport complet. Cherchez les sections :
- "Inspection Status" : doit être OK
- "Node Status" : tous Ready
- "Dqlite Status" : HA actif

Si tout est vert, le cluster est en bonne santé ! ✅

### 5. Communication Inter-Nœuds

**Déployer un test multi-réplicas :**
```bash
# Créer un deployment avec 3 réplicas
microk8s kubectl create deployment nginx-test --image=nginx --replicas=3

# Attendre que les pods soient prêts
microk8s kubectl rollout status deployment nginx-test

# Voir sur quels nœuds les pods tournent
microk8s kubectl get pods -o wide
```

Les 3 pods devraient être répartis sur les 3 nœuds (ou au moins 2).

**Tester la communication :**
```bash
# Obtenir les IPs des pods
microk8s kubectl get pods -o wide

# Exec dans un pod et pinger un autre
microk8s kubectl exec -it nginx-test-xxx -- ping -c 3 <IP_du_pod_sur_autre_noeud>
```

Si le ping fonctionne, la communication inter-nœuds est OK !

**Nettoyer :**
```bash
microk8s kubectl delete deployment nginx-test
```

## Configuration Optimale du Cluster HA

### 1. Labels de Nœuds

**Organiser les nœuds avec des labels :**

```bash
# Ajouter des labels pour identifier les rôles
microk8s kubectl label node node1 node-role.kubernetes.io/control-plane=true
microk8s kubectl label node node2 node-role.kubernetes.io/control-plane=true
microk8s kubectl label node node3 node-role.kubernetes.io/control-plane=true

# Ajouter des labels pour la topologie (datacenter, rack, zone, etc.)
microk8s kubectl label node node1 topology.kubernetes.io/zone=zone-a
microk8s kubectl label node node2 topology.kubernetes.io/zone=zone-b
microk8s kubectl label node node3 topology.kubernetes.io/zone=zone-c

# Vérifier
microk8s kubectl get nodes --show-labels
```

Ces labels sont utiles pour :
- Pod affinity/anti-affinity
- Node selectors
- Visualisation dans les dashboards

### 2. Taints (Optionnel)

Si vous voulez réserver certains nœuds au control plane uniquement :

```bash
# Empêcher les pods applicatifs de tourner sur node1
microk8s kubectl taint nodes node1 node-role.kubernetes.io/control-plane=true:NoSchedule
```

**Note :** Dans un cluster 3 nœuds, ce n'est généralement **pas recommandé**, car vous perdez de la capacité de calcul.

### 3. Resource Quotas par Namespace

**Protéger les ressources critiques :**

```yaml
# resource-quota-default.yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: default-quota
  namespace: default
spec:
  hard:
    requests.cpu: "8"
    requests.memory: 16Gi
    limits.cpu: "12"
    limits.memory: 24Gi
```

```bash
microk8s kubectl apply -f resource-quota-default.yaml
```

### 4. Pod Disruption Budgets

**Protéger vos applications critiques :**

```yaml
# pdb-example.yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: nginx-pdb
spec:
  minAvailable: 2
  selector:
    matchLabels:
      app: nginx
```

Cela garantit qu'au moins 2 pods nginx sont toujours disponibles, même pendant les maintenances.

## Tester la Résilience du Cluster

### Test 1 : Arrêt d'un Nœud Worker

**Simuler une panne de Node 3 :**

```bash
# Sur Node 3
sudo shutdown -h now
```

**Sur Node 1 ou Node 2, observer :**
```bash
# Attendre ~30 secondes
microk8s kubectl get nodes
```

```
NAME    STATUS     ROLES    AGE
node1   Ready      <none>   1h
node2   Ready      <none>   50m
node3   NotReady   <none>   44m    ← PANNE
```

**Vérifier que le cluster fonctionne toujours :**
```bash
# Créer un deployment
microk8s kubectl create deployment test-resilience --image=nginx --replicas=5

# Les pods se créent sur node1 et node2 uniquement
microk8s kubectl get pods -o wide
```

**Vérifier Dqlite (quorum maintenu) :**
```bash
microk8s status
```

```
high-availability: yes
  datastore master nodes: 192.168.1.10:19001 192.168.1.11:19001
```

Quorum = 2/2 actifs (sur 3 total) → Cluster fonctionnel ✅

**Rallumer Node 3 :**
```bash
# Sur Node 3, redémarrer la machine
# Après redémarrage, MicroK8s redémarre automatiquement

# Vérifier
microk8s kubectl get nodes
```

Node 3 repasse en **Ready** après ~1-2 minutes. Les pods se redistribuent progressivement.

### Test 2 : Panne du Leader Dqlite

**Identifier le Leader :**
```bash
# Sur n'importe quel nœud
microk8s kubectl get lease -n kube-system
```

Cherchez le lease avec le nom contenant "apiserver" pour identifier le Leader actuel.

**Arrêter le nœud Leader (ex: Node 1) :**
```bash
# Sur Node 1
sudo shutdown -h now
```

**Sur Node 2 ou Node 3, observer l'élection :**
```bash
# Attendre ~5-10 secondes
microk8s kubectl get nodes
```

Le cluster reste fonctionnel ! Dqlite a automatiquement élu un nouveau Leader parmi Node 2 et Node 3.

**Tester la création de ressources :**
```bash
microk8s kubectl run test-after-failover --image=nginx
microk8s kubectl get pods
```

Ça fonctionne ! Le nouveau Leader prend le relais. ✅

### Test 3 : Perte de Quorum (2 Nœuds Down)

**⚠️ Test destructif - à faire en environnement de test uniquement !**

**Arrêter Node 1 ET Node 2 :**
```bash
# Sur Node 1
sudo shutdown -h now

# Sur Node 2
sudo shutdown -h now
```

**Sur Node 3, vérifier le status :**
```bash
microk8s status
```

```
microk8s is running
high-availability: no (quorum lost)
```

**Tenter de créer un pod :**
```bash
microk8s kubectl run test-no-quorum --image=nginx
```

```
Error: cluster is in read-only mode (no quorum)
```

Le cluster est en **mode lecture seule**. Les pods existants continuent de tourner, mais vous ne pouvez plus faire de modifications.

**Solution :** Rallumer au moins 1 des 2 nœuds down pour retrouver le quorum.

**Rallumer Node 1 ou Node 2 :**

Après redémarrage, le quorum est restauré (2/3 actifs). Le cluster redevient fonctionnel.

## Stratégies de Maintenance

### Maintenance Planifiée d'un Nœud

Quand vous devez faire une maintenance sur un nœud (mise à jour OS, remplacement matériel, etc.), suivez ce processus :

**1. Drainer le nœud (évacuer les pods) :**
```bash
# Marquer le nœud comme non-schedulable et évacuer les pods
microk8s kubectl drain node2 --ignore-daemonsets --delete-emptydir-data
```

**2. Faire la maintenance :**
```bash
# Sur Node 2
sudo apt update && sudo apt upgrade -y
sudo reboot
```

**3. Réintégrer le nœud :**
```bash
# Une fois Node 2 de retour
microk8s kubectl uncordon node2
```

Les nouveaux pods peuvent à nouveau être schedulés sur node2.

### Mise à Jour de MicroK8s en HA

**Mettre à jour un nœud à la fois :**

```bash
# Sur Node 1
microk8s kubectl drain node1 --ignore-daemonsets
sudo snap refresh microk8s --channel=1.29/stable
microk8s kubectl uncordon node1

# Attendre que node1 soit Ready, puis répéter pour Node 2
microk8s kubectl drain node2 --ignore-daemonsets
# ... sur node2 ...
sudo snap refresh microk8s --channel=1.29/stable
microk8s kubectl uncordon node2

# Enfin, Node 3
microk8s kubectl drain node3 --ignore-daemonsets
# ... sur node3 ...
sudo snap refresh microk8s --channel=1.29/stable
microk8s kubectl uncordon node3
```

**Zero downtime** si vos applications ont au moins 2 réplicas !

## Monitoring du Cluster HA

### 1. Commandes Essentielles

**Surveillance des nœuds :**
```bash
# État des nœuds
watch microk8s kubectl get nodes

# Utilisation des ressources (CPU/RAM)
watch microk8s kubectl top nodes
```

**Surveillance de Dqlite :**
```bash
# Status général
watch microk8s status

# Logs Dqlite (si problème)
sudo journalctl -u snap.microk8s.daemon-k8s-dqlite -f
```

**Événements du cluster :**
```bash
# Voir les événements récents
microk8s kubectl get events --all-namespaces --sort-by='.lastTimestamp'
```

### 2. Prometheus et Grafana (Recommandé)

**Activer le monitoring avancé :**
```bash
microk8s enable prometheus
```

Cela installe :
- Prometheus (collecte des métriques)
- Grafana (visualisation)
- Alertmanager (alertes)

**Accéder à Grafana :**
```bash
# Obtenir le mot de passe admin
microk8s kubectl get secret -n observability grafana -o jsonpath="{.data.admin-password}" | base64 --decode

# Port-forward pour accéder
microk8s kubectl port-forward -n observability service/grafana 3000:3000
```

Ouvrez http://localhost:3000 dans votre navigateur.

**Dashboards recommandés :**
- Kubernetes / Cluster
- Kubernetes / Nodes
- Dqlite Performance

### 3. Alertes Critiques à Configurer

**Alertes essentielles pour un cluster HA :**
- Nœud down (NotReady > 5 minutes)
- Perte de quorum Dqlite
- CPU > 80% sur un nœud pendant > 10 minutes
- RAM > 90% sur un nœud
- Espace disque < 10% sur un nœud
- Pod crashlooping

## Bonnes Pratiques pour un Cluster HA

### 1. Répartition Physique

**Séparez physiquement vos nœuds si possible :**
- Différents racks (évite la panne d'un PDU)
- Différents switchs réseau (évite un SPOF réseau)
- Différentes alimentations électriques

**Dans un homelab :**
- Au minimum, différents serveurs physiques
- Si VMs, sur différents hyperviseurs

### 2. Backups Réguliers

**Même avec la HA, faites des backups !**

```bash
# Backup de la config du cluster (à faire régulièrement)
microk8s kubectl get all --all-namespaces -o yaml > backup-cluster-$(date +%Y%m%d).yaml

# Backup des secrets
microk8s kubectl get secrets --all-namespaces -o yaml > backup-secrets-$(date +%Y%m%d).yaml
```

**Automatiser avec cron :**
```bash
# Ajouter dans crontab
0 2 * * * /path/to/backup-script.sh
```

### 3. Documentation du Cluster

**Maintenir un document avec :**
- Inventaire des nœuds (IP, hostname, specs)
- Addons activés
- Applications déployées
- Procédures de maintenance
- Contacts en cas d'urgence

**Exemple :**
```markdown
# Cluster K8s Production

## Nœuds
- node1: 192.168.1.10 - 4 CPU, 8 GB RAM - Control Plane + Worker
- node2: 192.168.1.11 - 4 CPU, 8 GB RAM - Control Plane + Worker
- node3: 192.168.1.12 - 4 CPU, 8 GB RAM - Control Plane + Worker

## Addons
- dns, storage, prometheus, ingress, cert-manager

## Applications Critiques
- App Production (namespace: prod)
- Base de données PostgreSQL (namespace: database)

## Procédures
- Restart d'un nœud: drain → maintenance → uncordon
- Contact support: ops@example.com
```

### 4. Tests de Résilience Réguliers

**Testez régulièrement votre HA :**

```bash
# 1x par mois : test de panne d'un nœud
# 1x par trimestre : test de mise à jour
# 1x par an : test de restauration de backup complet
```

**Chaos Engineering (avancé) :**
Utilisez des outils comme **Chaos Mesh** pour simuler des pannes aléatoires et vérifier la résilience.

### 5. Monitoring Proactif

**Configurez des alertes AVANT les problèmes :**
- Disk > 75% → warning
- Disk > 90% → critical
- CPU moyen > 70% pendant 1h → investigation
- Latence réseau > 20ms → vérifier l'infra

## Évolution du Cluster : 3 → 5 → 7 Nœuds

### Passer de 3 à 5 Nœuds

**Ajouter 2 workers purs :**

```bash
# Sur un nœud existant, générer token worker
microk8s add-node --worker

# Sur worker4
microk8s join <token> --worker

# Répéter pour worker5
microk8s add-node --worker
# Sur worker5
microk8s join <token> --worker

# Vérifier
microk8s kubectl get nodes
```

```
NAME      STATUS   ROLES
node1     Ready    <none>
node2     Ready    <none>
node3     Ready    <none>
worker4   Ready    <none>
worker5   Ready    <none>
```

**Avantage :** Plus de capacité de calcul, mais toujours 3 nœuds control plane (tolérance 1 panne).

### Passer de 3 à 5 Nœuds Control Plane

**Ajouter 2 nœuds mixtes (control plane + worker) :**

```bash
# Sur un nœud existant
microk8s add-node

# Sur node4
microk8s join <token>

# Répéter pour node5
```

**Avantage :** Tolérance à 2 pannes simultanées (quorum = 3/5).

```bash
microk8s status
```

```
high-availability: yes
  datastore master nodes:
    192.168.1.10:19001
    192.168.1.11:19001
    192.168.1.12:19001
    192.168.1.13:19001
    192.168.1.14:19001
```

## Dépannage d'un Cluster HA

### Problème : Nœud Reste NotReady

**Diagnostic :**
```bash
# Voir les détails du nœud
microk8s kubectl describe node node2

# Logs kubelet
sudo journalctl -u snap.microk8s.daemon-kubelet -n 100
```

**Causes courantes :**
- Problème réseau (CNI)
- Espace disque plein
- Problème Dqlite

**Solution :**
```bash
# Redémarrer MicroK8s sur le nœud problématique
microk8s stop
microk8s start
```

### Problème : Split-Brain (Très Rare)

**Symptôme :** Deux parties du cluster pensent être le Leader.

**Diagnostic :**
```bash
# Vérifier les Leaders sur chaque nœud
microk8s kubectl get lease -n kube-system
```

**Solution :**
Redémarrer le cluster entièrement (tous les nœuds) pour forcer une nouvelle élection.

### Problème : Performances Dégradées

**Causes possibles :**
- Latence réseau élevée (> 50ms)
- Disques lents (HDD au lieu de SSD)
- Charge CPU/RAM trop élevée

**Diagnostic :**
```bash
# Vérifier latence
ping -c 100 node2 | grep avg

# Vérifier I/O disque
sudo iotop

# Vérifier charge
microk8s kubectl top nodes
```

## Points Clés à Retenir

1. **3 nœuds minimum** pour la vraie HA (quorum = 2/3)
2. **Nombre impair** de nœuds toujours (3, 5, 7)
3. **Ne jamais avoir 2 nœuds** (pire que 1)
4. **Tolérance aux pannes** : (n-1)/2 nœuds
5. **Dqlite gère automatiquement** la réplication et l'élection
6. **Tester régulièrement** la résilience (pannes simulées)
7. **Maintenance** : drainer avant, uncordon après
8. **Monitoring** : Prometheus + Grafana recommandé
9. **Backups** : même avec HA, toujours sauvegarder
10. **Documentation** : maintenir un inventaire à jour

## Prochaines Étapes

Votre cluster HA est maintenant opérationnel ! Les prochains chapitres couvriront :

- **21.5** : Ajouter des workers supplémentaires pour scaler
- **21.6** : Load balancing entre nœuds avec MetalLB
- **21.7** : Stockage distribué pour la HA des données
- **21.8** : Backup du control plane et stratégies DR
- **21.9** : Stratégies de redondance avancées
- **21.10** : Tests de failover complets

Félicitations ! Vous avez construit un cluster Kubernetes en haute disponibilité. Vous êtes maintenant capable de gérer une infrastructure résiliente et prête pour la production ! 🚀

---

**Bravo !** Vous maîtrisez maintenant la configuration d'un cluster MicroK8s HA complet. C'est une compétence avancée et très recherchée dans le monde DevOps et Cloud Native !

⏭️ [Ajout de worker nodes supplémentaires](/21-multi-node-et-haute-disponibilite/05-ajout-de-worker-nodes-supplementaires.md)
