🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 21.3 Ajout de Nœuds avec microk8s add-node

## Introduction

Maintenant que vous comprenez l'architecture multi-node et les avantages de la haute disponibilité avec Dqlite, passons à la **pratique** ! Dans ce chapitre, nous allons découvrir comment ajouter concrètement des nœuds à votre cluster MicroK8s.

La bonne nouvelle ? C'est **incroyablement simple**. Là où d'autres distributions Kubernetes nécessitent de multiples commandes et fichiers de configuration, MicroK8s vous permet d'ajouter un nœud avec une seule commande : `microk8s add-node`.

Nous allons voir comment fonctionne cette commande, ce qui se passe en coulisses, et comment vérifier que tout fonctionne correctement.

## Vue d'Ensemble du Processus

### Le Principe

Pour ajouter un nœud à un cluster MicroK8s, il faut :

1. **Sur le nœud existant** : Générer un token de jointure avec `microk8s add-node`
2. **Sur le nouveau nœud** : Utiliser ce token pour rejoindre le cluster avec `microk8s join`

C'est tout ! MicroK8s s'occupe de :
- Configurer automatiquement Dqlite
- Établir la communication sécurisée entre nœuds
- Synchroniser la configuration du cluster
- Activer le nouveau nœud

**Analogie :**
C'est comme inviter quelqu'un dans un groupe WhatsApp :
1. Vous générez un lien d'invitation
2. La personne clique sur le lien
3. Elle rejoint automatiquement le groupe

### Schéma du Processus

```
┌──────────────────────────────────────────────────────────────┐
│                    PROCESSUS D'AJOUT                         │
└──────────────────────────────────────────────────────────────┘

ÉTAPE 1 : Sur le nœud existant (Node 1)
┌────────────┐
│   Node 1   │
│  (Master)  │  $ microk8s add-node
│            │  ──────────────────────►  Génère un token
└────────────┘                          avec IP et port

ÉTAPE 2 : Installation sur le nouveau serveur (Node 2)
┌────────────┐
│   Node 2   │
│  (Nouveau) │  $ sudo snap install microk8s --classic
│            │
└────────────┘

ÉTAPE 3 : Jointure avec le token
┌────────────┐                          ┌────────────┐
│   Node 2   │                          │   Node 1   │
│            │  microk8s join <token>   │            │
│            │  ─────────────────────►  │            │
│            │  ◄─────────────────────  │            │
│            │    Handshake + Config    │            │
└────────────┘                          └────────────┘

ÉTAPE 4 : Synchronisation automatique
┌────────────┐                          ┌────────────┐
│   Node 2   │  ◄──── Dqlite ────►      │   Node 1   │
│  Dqlite DB │  Réplication auto        │  Dqlite DB │
│   kubelet  │  des données             │   kubelet  │
└────────────┘                          └────────────┘

RÉSULTAT : Cluster à 2 nœuds opérationnel !
```

## Prérequis Avant d'Ajouter un Nœud

### 1. Réseau et Connectivité

**Communication bidirectionnelle :**
Les nœuds doivent pouvoir se parler dans les deux sens. Vérifiez que :

```bash
# Depuis Node 1, pinger Node 2
ping 192.168.1.11

# Depuis Node 2, pinger Node 1
ping 192.168.1.10
```

Si le ping ne fonctionne pas, vérifiez :
- Les adresses IP sont correctes
- Les machines sont sur le même réseau (ou routage configuré)
- Pas de firewall bloquant ICMP

**Adresses IP statiques recommandées :**
- Utilisez des IPs fixes, pas de DHCP
- Évitez que les IPs changent après un redémarrage
- Configurez vos IPs de manière permanente

**Latence :**
- Recommandé : < 10ms entre nœuds
- Maximum : < 50ms (au-delà, des problèmes peuvent survenir)

```bash
# Tester la latence
ping -c 10 192.168.1.11
# Regardez les valeurs "time=" dans les résultats
```

### 2. Ports à Ouvrir

MicroK8s a besoin de plusieurs ports ouverts entre les nœuds :

**Ports essentiels pour la jointure :**
| Port | Protocole | Direction | Description |
|------|-----------|-----------|-------------|
| 16443 | TCP | Bidirectionnel | API Server Kubernetes |
| 25000 | TCP | Bidirectionnel | Cluster agent (jointure) |
| 12379 | TCP | Bidirectionnel | Dqlite database |
| 10250 | TCP | Bidirectionnel | Kubelet API |
| 10255 | TCP | Bidirectionnel | Kubelet read-only |
| 10257 | TCP | Bidirectionnel | Kube-controller-manager |
| 10259 | TCP | Bidirectionnel | Kube-scheduler |

**Exemple de configuration firewall (UFW sur Ubuntu) :**
```bash
# Sur CHAQUE nœud, autoriser depuis l'IP du/des autre(s) nœud(s)

# Si Node 1 = 192.168.1.10 et Node 2 = 192.168.1.11

# Sur Node 1 :
sudo ufw allow from 192.168.1.11 to any port 16443
sudo ufw allow from 192.168.1.11 to any port 25000
sudo ufw allow from 192.168.1.11 to any port 12379
sudo ufw allow from 192.168.1.11 to any port 10250:10259/tcp

# Sur Node 2 :
sudo ufw allow from 192.168.1.10 to any port 16443
sudo ufw allow from 192.168.1.10 to any port 25000
sudo ufw allow from 192.168.1.10 to any port 12379
sudo ufw allow from 192.168.1.10 to any port 10250:10259/tcp
```

**Astuce :** Si vous êtes sur un réseau local de confiance, vous pouvez autoriser tout le trafic entre les nœuds :
```bash
# Sur Node 1
sudo ufw allow from 192.168.1.11

# Sur Node 2
sudo ufw allow from 192.168.1.10
```

### 3. Ressources Matérielles

**Minimum par nœud :**
- **CPU** : 2 cœurs
- **RAM** : 4 GB
- **Stockage** : 20 GB (40 GB recommandé)
- **Réseau** : 100 Mbps (1 Gbps recommandé)

**Vérifier les ressources disponibles :**
```bash
# CPU
nproc

# RAM
free -h

# Espace disque
df -h /
```

### 4. Système d'Exploitation

**OS identique recommandé :**
- Tous les nœuds sur Ubuntu 22.04 (recommandé)
- Ou tous sur Ubuntu 20.04
- Ou tous sur Debian, CentOS, etc.

**Version de MicroK8s :**
- **Même version** sur tous les nœuds !
- Vérifier : `snap info microk8s`

```bash
# Installer la même version sur tous les nœuds
sudo snap install microk8s --classic --channel=1.28/stable
```

### 5. Nom d'Hôte Unique

Chaque nœud doit avoir un nom d'hôte (hostname) **unique**.

```bash
# Vérifier le hostname
hostname

# Si nécessaire, le changer (exemple)
sudo hostnamectl set-hostname node2
```

**Convention de nommage recommandée :**
- node1, node2, node3
- k8s-node1, k8s-node2, k8s-node3
- master1, worker1, worker2

Évitez les noms génériques comme "localhost" ou "ubuntu".

## La Commande microk8s add-node

### Syntaxe de Base

Sur le nœud **existant** du cluster :

```bash
microk8s add-node
```

C'est tout ! Cette commande génère un token de jointure.

### Options Avancées

```bash
# Spécifier l'IP du nœud actuel (utile si plusieurs interfaces réseau)
microk8s add-node --interface eth0

# Spécifier une IP précise
microk8s add-node --interface 192.168.1.10

# Générer un token pour un worker (pas control plane)
microk8s add-node --worker
```

### Sortie de la Commande

Quand vous exécutez `microk8s add-node`, vous obtenez une sortie similaire à :

```
From the node you wish to join to this cluster, run the following:
microk8s join 192.168.1.10:25000/a1b2c3d4e5f6g7h8i9j0k1l2m3n4o5p6

Use the '--worker' flag to join a node as a worker not running the control plane, eg:
microk8s join 192.168.1.10:25000/a1b2c3d4e5f6g7h8i9j0k1l2m3n4o5p6 --worker

If the node you are adding is not reachable through the default interface you can use one of the following:
microk8s join 192.168.1.10:25000/a1b2c3d4e5f6g7h8i9j0k1l2m3n4o5p6
microk8s join 10.0.0.10:25000/a1b2c3d4e5f6g7h8i9j0k1l2m3n4o5p6
```

**Décomposition du token :**
```
microk8s join 192.168.1.10:25000/a1b2c3d4e5f6g7h8i9j0k1l2m3n4o5p6
                │             │      │
                │             │      └─ Token unique (32 caractères alphanumériques)
                │             └──────── Port du cluster agent
                └────────────────────── IP du nœud existant
```

### Comprendre le Token

**Qu'est-ce que le token ?**
C'est une clé d'authentification temporaire qui permet à un nouveau nœud de :
1. S'authentifier auprès du cluster
2. Récupérer la configuration
3. Rejoindre Dqlite
4. Démarrer les composants Kubernetes

**Sécurité du token :**
- Le token est **à usage unique** (ou limité dans le temps)
- Il contient des informations de chiffrement
- Une fois utilisé, il ne peut plus servir
- Si vous devez ajouter plusieurs nœuds, générez un nouveau token à chaque fois

**Durée de validité :**
- Par défaut : le token est valide pendant **24 heures**
- Après expiration, générez-en un nouveau avec `microk8s add-node`

## Ajouter un Nœud : Processus Étape par Étape

### Scénario : Ajouter Node 2 à un Cluster Existant

**Situation initiale :**
- Node 1 (192.168.1.10) : cluster MicroK8s fonctionnel en single-node
- Node 2 (192.168.1.11) : machine vide, Ubuntu 22.04 installé

### Étape 1 : Préparer Node 2

**Sur Node 2, installer MicroK8s :**
```bash
# Mettre à jour le système
sudo apt update && sudo apt upgrade -y

# Installer MicroK8s (même version que Node 1 !)
sudo snap install microk8s --classic --channel=1.28/stable

# Ajouter votre utilisateur au groupe microk8s
sudo usermod -a -G microk8s $USER
sudo chown -f -R $USER ~/.kube

# Recharger les groupes (ou se déconnecter/reconnecter)
newgrp microk8s

# Vérifier l'installation
microk8s status --wait-ready
```

**Important :** À ce stade, Node 2 a MicroK8s installé, mais il tourne en mode standalone (cluster d'un seul nœud).

### Étape 2 : Générer le Token sur Node 1

**Sur Node 1 :**
```bash
microk8s add-node
```

**Copier la commande affichée.** Elle ressemble à :
```bash
microk8s join 192.168.1.10:25000/abc123def456ghi789jkl012mno345pqr678
```

**Astuce :** Utilisez SSH pour copier-coller facilement :
```bash
# Depuis votre PC, se connecter à Node 1
ssh user@192.168.1.10

# Exécuter add-node et copier le résultat
microk8s add-node
```

### Étape 3 : Joindre le Cluster depuis Node 2

**Sur Node 2, coller et exécuter la commande :**
```bash
microk8s join 192.168.1.10:25000/abc123def456ghi789jkl012mno345pqr678
```

**Ce qui se passe alors :**
```
Contacting cluster at 192.168.1.10...
Waiting for this node to finish joining the cluster. ..
```

Le processus peut prendre **30 secondes à 2 minutes**. Vous verrez des messages comme :
```
Waiting for node to finish joining the cluster. .. .. .. ..
Node joined successfully!
```

**Que fait MicroK8s en arrière-plan ?**
1. Authentification avec le token
2. Téléchargement de la configuration du cluster
3. Configuration de Dqlite (réplication)
4. Démarrage du kubelet
5. Enregistrement du nœud dans l'API server
6. Synchronisation des ressources

### Étape 4 : Vérifier que Node 2 a Rejoint

**Sur n'importe quel nœud du cluster :**
```bash
microk8s kubectl get nodes
```

**Sortie attendue :**
```
NAME    STATUS   ROLES    AGE     VERSION
node1   Ready    <none>   10d     v1.28.3
node2   Ready    <none>   2m      v1.28.3
```

**Vérifications importantes :**
- Les deux nœuds affichent **STATUS = Ready**
- Les deux nœuds ont la même **VERSION**
- Les noms sont uniques (node1, node2)

**Si STATUS = NotReady**, attendez quelques minutes. Les images Docker doivent être téléchargées.

### Étape 5 : Vérifier Dqlite

**Sur Node 1 ou Node 2 :**
```bash
microk8s status
```

Vous devriez voir quelque chose comme :
```
microk8s is running
high-availability: yes
  datastore master nodes: 192.168.1.10:19001 192.168.1.11:19001
  datastore standby nodes: none
```

**"high-availability: yes"** confirme que Dqlite est répliqué ! 🎉

## Ajouter un Troisième Nœud (Cluster HA)

Pour atteindre la vraie haute disponibilité, ajoutez un 3ème nœud.

### Processus Identique

**Sur Node 1 ou Node 2, générer un nouveau token :**
```bash
microk8s add-node
```

**Sur Node 3 (après installation de MicroK8s) :**
```bash
microk8s join 192.168.1.10:25000/xyz789uvw456rst123abc098def765mno432
```

**Vérifier les 3 nœuds :**
```bash
microk8s kubectl get nodes
```

```
NAME    STATUS   ROLES    AGE     VERSION
node1   Ready    <none>   10d     v1.28.3
node2   Ready    <none>   1h      v1.28.3
node3   Ready    <none>   2m      v1.28.3
```

**Vérifier Dqlite HA :**
```bash
microk8s status
```

```
microk8s is running
high-availability: yes
  datastore master nodes: 192.168.1.10:19001 192.168.1.11:19001 192.168.1.12:19001
  datastore standby nodes: none
```

**Vous avez maintenant un cluster 3 nœuds en haute disponibilité !** 🚀

## Ajouter un Worker (Sans Control Plane)

Par défaut, chaque nœud ajouté devient un **nœud mixte** (control plane + worker). Mais vous pouvez ajouter un nœud **worker pur**, qui n'exécutera pas le control plane.

### Quand Utiliser un Worker Pur ?

**Avantages :**
- Moins de ressources consommées (pas de Dqlite, API server, etc.)
- Isolation des workloads critiques du control plane
- Scalabilité (ajout de capacité de calcul sans complexifier le control plane)

**Cas d'usage :**
- Vous avez déjà 3 nœuds control plane (quorum OK)
- Vous voulez ajouter de la capacité pour vos applications
- Vous avez des machines moins puissantes à ajouter

### Commandes

**Sur un nœud existant, générer un token worker :**
```bash
microk8s add-node --worker
```

**Sortie :**
```
From the node you wish to join to this cluster, run the following:
microk8s join 192.168.1.10:25000/worker123abc456def789ghi012jkl345mno678 --worker
```

**Sur le nouveau nœud (après installation MicroK8s) :**
```bash
microk8s join 192.168.1.10:25000/worker123abc456def789ghi012jkl345mno678 --worker
```

**Vérifier :**
```bash
microk8s kubectl get nodes
```

```
NAME      STATUS   ROLES    AGE     VERSION
node1     Ready    <none>   10d     v1.28.3
node2     Ready    <none>   2h      v1.28.3
node3     Ready    <none>   1h      v1.28.3
worker1   Ready    <none>   2m      v1.28.3
```

Le worker n'apparaît **pas** dans les "datastore master nodes" de Dqlite :
```bash
microk8s status
```

```
microk8s is running
high-availability: yes
  datastore master nodes: 192.168.1.10:19001 192.168.1.11:19001 192.168.1.12:19001
  datastore standby nodes: none
```

## Comprendre les Rôles des Nœuds

### Nœuds Mixtes (Par Défaut)

Quand vous ajoutez un nœud sans `--worker`, c'est un **nœud mixte** :
- **Control Plane** : Exécute API Server, Scheduler, Controllers, Dqlite
- **Worker** : Peut héberger des pods applicatifs

**Avantages :**
- Simplicité (pas de distinction à gérer)
- Parfait pour les petits clusters (3 à 7 nœuds)

**Inconvénients :**
- Plus de ressources consommées par nœud
- Pods applicatifs partagent les ressources avec le control plane

### Workers Purs

Quand vous ajoutez un nœud avec `--worker` :
- **Pas de Control Plane**
- **Worker uniquement** : Héberge des pods applicatifs

**Avantages :**
- Moins de ressources consommées (pas de Dqlite, etc.)
- Isolation control plane / workloads

**Inconvénients :**
- Ne contribue pas à la HA du control plane

### Architecture Recommandée

**Petit cluster (3-5 nœuds) :**
```
3 nœuds mixtes (control plane + worker)
```
Simple et efficace.

**Cluster moyen (7-15 nœuds) :**
```
3 nœuds mixtes (control plane + worker)
+ 4-12 workers purs
```
HA du control plane + scalabilité des workloads.

**Grand cluster (15+ nœuds) :**
```
3-5 nœuds control plane dédiés (sans workloads)
+ 10+ workers purs
```
Séparation totale, mais plus complexe à gérer.

## Que Faire si la Jointure Échoue ?

### Problèmes Courants et Solutions

#### 1. Erreur "Cannot connect to 192.168.1.10:25000"

**Symptôme :**
```
Contacting cluster at 192.168.1.10...
error: cannot connect to 192.168.1.10:25000
```

**Causes possibles :**
- Firewall bloque le port 25000
- IP incorrecte ou non accessible
- Nœud existant éteint

**Solutions :**
```bash
# Vérifier le firewall sur Node 1
sudo ufw status

# Vérifier la connectivité depuis Node 2
telnet 192.168.1.10 25000
# Ou :
nc -zv 192.168.1.10 25000

# Si échec, ouvrir le port sur Node 1
sudo ufw allow 25000/tcp
```

#### 2. Erreur "Token has expired"

**Symptôme :**
```
error: token has expired
```

**Cause :**
Le token généré est valide pendant 24 heures seulement.

**Solution :**
Générer un nouveau token sur Node 1 :
```bash
microk8s add-node
# Copier le nouveau token et réessayer sur Node 2
```

#### 3. Nœud Reste en "NotReady"

**Symptôme :**
```bash
microk8s kubectl get nodes
```
```
NAME    STATUS      ROLES    AGE     VERSION
node1   Ready       <none>   10d     v1.28.3
node2   NotReady    <none>   5m      v1.28.3
```

**Causes possibles :**
- Images Docker encore en téléchargement
- Problème de réseau CNI
- Kubelet pas complètement démarré

**Solutions :**
```bash
# Attendre 2-5 minutes et vérifier à nouveau

# Voir les logs du kubelet sur Node 2
sudo journalctl -u snap.microk8s.daemon-kubelet -f

# Vérifier les pods système sur Node 2
microk8s kubectl get pods -n kube-system -o wide

# Redémarrer MicroK8s sur Node 2 si nécessaire
microk8s stop
microk8s start
```

#### 4. Versions MicroK8s Différentes

**Symptôme :**
```
error: version mismatch
```

**Cause :**
Node 1 sur MicroK8s 1.28, Node 2 sur MicroK8s 1.27 (ou inverse).

**Solution :**
Installer la **même version** sur tous les nœuds :
```bash
# Vérifier la version sur Node 1
microk8s version

# Sur Node 2, installer la même version
sudo snap refresh microk8s --channel=1.28/stable
```

#### 5. Nom d'Hôte Dupliqué

**Symptôme :**
Les deux nœuds ont le même nom dans `kubectl get nodes`.

**Cause :**
Hostname identique sur plusieurs machines.

**Solution :**
```bash
# Sur Node 2, changer le hostname
sudo hostnamectl set-hostname node2

# Redémarrer MicroK8s
microk8s stop
microk8s start
```

### Logs de Diagnostic

**Sur le nœud qui rejoint (Node 2) :**
```bash
# Logs du daemon principal
sudo journalctl -u snap.microk8s.daemon-kubelite -f

# Logs du kubelet
sudo journalctl -u snap.microk8s.daemon-kubelet -f

# Logs généraux MicroK8s
sudo snap logs microk8s -f
```

**Sur le nœud existant (Node 1) :**
```bash
# Vérifier l'API server
sudo journalctl -u snap.microk8s.daemon-apiserver -f

# Vérifier les événements Kubernetes
microk8s kubectl get events --all-namespaces --sort-by='.lastTimestamp'
```

## Vérifications Post-Jointure

### 1. Nœuds Visibles et Ready

```bash
microk8s kubectl get nodes -o wide
```

Vérifiez :
- Tous les nœuds sont **Ready**
- Les IPs sont correctes
- Les versions Kubernetes sont identiques

### 2. Pods Système Distribués

```bash
microk8s kubectl get pods -n kube-system -o wide
```

Les pods système (CoreDNS, etc.) devraient être répartis sur les différents nœuds.

### 3. Communication Inter-Nœuds

Tester qu'un pod sur Node 1 peut communiquer avec un pod sur Node 2 :

```bash
# Créer un pod de test sur Node 1
microk8s kubectl run test-node1 --image=nginx --restart=Never

# Créer un pod de test sur Node 2 (en forçant le placement)
microk8s kubectl run test-node2 --image=nginx --restart=Never --overrides='{"spec":{"nodeName":"node2"}}'

# Obtenir les IPs des pods
microk8s kubectl get pods -o wide

# Depuis test-node1, pinger test-node2
microk8s kubectl exec test-node1 -- ping -c 3 <IP_du_pod_node2>
```

Si le ping réussit, la communication inter-nœuds fonctionne ! ✅

### 4. Haute Disponibilité Dqlite

```bash
microk8s status
```

Vérifiez :
```
high-availability: yes
  datastore master nodes: <liste_des_IPs>
```

Si "yes" et plusieurs IPs listées, la HA est active.

### 5. État des Addons

Les addons activés sur Node 1 devraient être automatiquement disponibles sur Node 2 :

```bash
microk8s status
```

Exemple :
```
microk8s is running
high-availability: yes
  datastore master nodes: 192.168.1.10:19001 192.168.1.11:19001
  datastore standby nodes: none
addons:
  enabled:
    dns                  # CoreDNS
    ha-cluster           # High availability
    storage              # Stockage persistant
  disabled:
    ...
```

## Bonnes Pratiques

### 1. Toujours Utiliser des IPs Statiques

**Pourquoi ?**
Si une IP change (DHCP), le cluster perd la connexion avec ce nœud.

**Comment ?**
Configurez des IPs fixes dans `/etc/netplan/` (Ubuntu) ou équivalent.

### 2. Documenter vos Nœuds

Maintenez un fichier avec les informations de chaque nœud :

```
# cluster-nodes.txt
Node 1 : 192.168.1.10 - node1 - Control Plane + Worker - 4 CPU, 8 GB RAM
Node 2 : 192.168.1.11 - node2 - Control Plane + Worker - 4 CPU, 8 GB RAM
Node 3 : 192.168.1.12 - node3 - Control Plane + Worker - 4 CPU, 8 GB RAM
Worker 1 : 192.168.1.20 - worker1 - Worker only - 2 CPU, 4 GB RAM
```

### 3. Ajouter les Nœuds un par un

N'ajoutez pas plusieurs nœuds simultanément. Attendez que chaque nœud soit **Ready** avant d'ajouter le suivant.

**Pourquoi ?**
Dqlite a besoin de se stabiliser entre chaque ajout.

### 4. Sauvegarder la Configuration

Avant d'ajouter des nœuds, faites un backup :

```bash
# Sur Node 1, exporter la config
microk8s kubectl get all --all-namespaces -o yaml > backup-before-scale.yaml
```

En cas de problème, vous pourrez restaurer.

### 5. Surveiller les Ressources

Après chaque ajout, vérifiez les ressources :

```bash
# Ressources des nœuds
microk8s kubectl top nodes

# Ressources des pods
microk8s kubectl top pods --all-namespaces
```

### 6. Tester la Résilience

Une fois vos nœuds ajoutés, testez :

```bash
# Éteindre Node 2
# Sur Node 2 :
sudo shutdown -h now

# Sur Node 1 ou 3, vérifier que le cluster continue de fonctionner
microk8s kubectl get nodes
microk8s kubectl get pods --all-namespaces
```

Les pods de Node 2 devraient être automatiquement redémarrés sur Node 1 ou Node 3.

## Retirer un Nœud du Cluster

Si vous devez retirer un nœud (maintenance, remplacement, etc.) :

### Méthode Propre

**Sur le nœud à retirer :**
```bash
microk8s leave
```

**Sur un autre nœud du cluster :**
```bash
microk8s remove-node <nom_du_noeud>
```

**Exemple :**
```bash
# Sur Node 2
microk8s leave

# Sur Node 1
microk8s remove-node node2

# Vérifier
microk8s kubectl get nodes
```

### En Cas de Nœud Inaccessible

Si le nœud est mort et ne répond plus :

```bash
# Sur un nœud fonctionnel
microk8s remove-node node2 --force
```

Cela retire de force le nœud du cluster, même s'il ne répond pas.

## Récapitulatif du Processus

**Workflow complet pour ajouter un nœud :**

1. **Préparer le nouveau serveur** (IP fixe, firewall, hostname unique)
2. **Installer MicroK8s** (même version que les nœuds existants)
3. **Générer un token** sur un nœud existant : `microk8s add-node`
4. **Joindre le cluster** depuis le nouveau nœud : `microk8s join <token>`
5. **Vérifier** : `microk8s kubectl get nodes`
6. **Attendre le statut Ready** (30s à 2 min)
7. **Vérifier la HA** : `microk8s status`
8. **Tester la communication** entre pods sur différents nœuds

**Temps total : 10-15 minutes par nœud.**

## Points Clés à Retenir

1. **`microk8s add-node`** génère un token de jointure sur le nœud existant
2. **`microk8s join <token>`** joint le nouveau nœud au cluster
3. **Le token est à usage unique** et valide 24h
4. **Prérequis critiques** : connectivité réseau, ports ouverts, même version
5. **Par défaut**, les nœuds ajoutés sont mixtes (control plane + worker)
6. **Option `--worker`** pour ajouter un worker pur (sans control plane)
7. **Dqlite se réplique automatiquement** lors de la jointure
8. **HA nécessite 3+ nœuds** de control plane
9. **Vérifier toujours** : `kubectl get nodes` et `microk8s status`
10. **Troubleshooting** : firewall, versions, logs

## Prochaines Étapes

Maintenant que vous savez ajouter des nœuds, les prochains chapitres vous montreront :

- **21.4** : Configurer un cluster HA complet de 3 nœuds avec tous les détails
- **21.5** : Ajouter des workers supplémentaires et scaler votre cluster
- **21.6** : Gérer le load balancing entre nœuds
- **21.7** : Configurer le stockage distribué

Avec `microk8s add-node`, ce qui était complexe devient **simple et rapide**. Vous êtes maintenant capable de construire votre propre cluster Kubernetes multi-node en quelques minutes !

---

**Félicitations !** Vous maîtrisez maintenant l'ajout de nœuds à un cluster MicroK8s. C'est une compétence fondamentale pour construire des infrastructures Kubernetes robustes et scalables ! 🚀

⏭️ [Configuration d'un cluster HA (3+ nœuds)](/21-multi-node-et-haute-disponibilite/04-configuration-dun-cluster-ha-3-noeuds.md)
