🔝 Retour au [Sommaire](/SOMMAIRE.md)

# Configuration Firewall pour MicroK8s

## Introduction

### Qu'est-ce qu'un firewall ?

Un firewall (pare-feu en français) est un système de sécurité qui contrôle le trafic réseau entrant et sortant en fonction de règles de sécurité prédéfinies. C'est comme un vigile à l'entrée d'un bâtiment qui vérifie les badges et autorise ou refuse l'accès selon des critères définis.

### Pourquoi configurer un firewall pour MicroK8s ?

Même si MicroK8s est destiné à un environnement de développement ou un lab personnel, la sécurité reste importante :

1. **Protection contre les accès non autorisés** : empêcher les intrusions depuis Internet
2. **Limitation de la surface d'attaque** : n'exposer que les services nécessaires
3. **Contrôle du trafic** : gérer finement quelles applications peuvent communiquer
4. **Bonnes pratiques** : apprendre à sécuriser correctement une infrastructure

### Les différents niveaux de firewall

Dans un environnement MicroK8s, il existe plusieurs niveaux de protection :

```
┌─────────────────────────────────────────────┐
│ Internet                                    │
└─────────────────┬───────────────────────────┘
                  │
┌─────────────────▼───────────────────────────┐
│ 1. Firewall Routeur/Box Internet            │
│    (Premier niveau de défense)              │
└─────────────────┬───────────────────────────┘
                  │
┌─────────────────▼───────────────────────────┐
│ 2. Firewall OS (UFW, firewalld, iptables)   │
│    (Protection au niveau du système)        │
└─────────────────┬───────────────────────────┘
                  │
┌─────────────────▼───────────────────────────┐
│ 3. Network Policies Kubernetes              │
│    (Contrôle inter-pods)                    │
└─────────────────────────────────────────────┘
```

Ce guide couvre les niveaux 2 et 3.

---

## Partie 1 : Firewall au Niveau Système d'Exploitation

### Ports utilisés par MicroK8s

Avant de configurer le firewall, vous devez comprendre quels ports MicroK8s utilise :

#### Ports Essentiels

| Port | Protocole | Usage | Accès depuis |
|------|-----------|-------|--------------|
| **16443** | TCP | API Server Kubernetes | Réseau local ou Internet (si administration distante) |
| **10250** | TCP | Kubelet API | Réseau local (communication inter-nœuds) |
| **10255** | TCP | Kubelet Read-only | Réseau local |
| **25000** | TCP | Cluster agent | Réseau local (multi-node) |
| **12379** | TCP | Dqlite | Réseau local (multi-node HA) |

#### Ports pour Services Exposés

| Port | Protocole | Usage | Accès depuis |
|------|-----------|-------|--------------|
| **80** | TCP | HTTP (Ingress) | Internet/Réseau local |
| **443** | TCP | HTTPS (Ingress) | Internet/Réseau local |
| **30000-32767** | TCP/UDP | NodePort Services | Selon besoins |

#### Ports pour Addons

| Port | Protocole | Addon | Accès depuis |
|------|-----------|-------|--------------|
| **16443** | TCP | Dashboard | Via kubectl proxy ou Ingress |
| **5000** | TCP | Registry | Réseau local |
| **9090** | TCP | Prometheus | Via Ingress ou local |
| **3000** | TCP | Grafana | Via Ingress ou local |

### Ports CNI (Container Network Interface)

MicroK8s utilise Calico par défaut :

| Port | Protocole | Usage |
|------|-----------|-------|
| **179** | TCP | BGP (Calico) |
| **4789** | UDP | VXLAN (Calico) |
| **51820** | UDP | Wireguard (si activé) |

---

## Configuration UFW (Ubuntu/Debian)

UFW (Uncomplicated Firewall) est le firewall par défaut sur Ubuntu. Il est simple à utiliser et parfait pour les débutants.

### Installation

UFW est généralement préinstallé sur Ubuntu. Si ce n'est pas le cas :

```bash
sudo apt update
sudo apt install ufw
```

### Vérification du statut

```bash
sudo ufw status
```

Si le firewall est inactif, vous verrez :
```
Status: inactive
```

### Configuration de Base pour MicroK8s Single-Node

#### Étape 1 : Autoriser SSH (IMPORTANT !)

**Avant d'activer le firewall, autorisez toujours SSH pour ne pas vous bloquer l'accès à votre machine :**

```bash
sudo ufw allow 22/tcp comment 'SSH'
```

#### Étape 2 : Autoriser les ports MicroK8s essentiels

```bash
# API Server Kubernetes (si accès distant nécessaire)
sudo ufw allow 16443/tcp comment 'Kubernetes API'

# HTTP et HTTPS pour les applications web
sudo ufw allow 80/tcp comment 'HTTP'
sudo ufw allow 443/tcp comment 'HTTPS'
```

#### Étape 3 : Autoriser le trafic localhost

Le trafic sur l'interface locale (lo) doit toujours être autorisé :

```bash
sudo ufw allow in on lo
sudo ufw allow out on lo
```

#### Étape 4 : Activer le firewall

```bash
sudo ufw enable
```

Vous verrez un avertissement. Tapez `y` pour confirmer.

#### Étape 5 : Vérifier la configuration

```bash
sudo ufw status verbose
```

Résultat attendu :

```
Status: active
Logging: on (low)
Default: deny (incoming), allow (outgoing), disabled (routed)

To                         Action      From
--                         ------      ----
22/tcp                     ALLOW IN    Anywhere                   # SSH
16443/tcp                  ALLOW IN    Anywhere                   # Kubernetes API
80/tcp                     ALLOW IN    Anywhere                   # HTTP
443/tcp                    ALLOW IN    Anywhere                   # HTTPS
Anywhere on lo             ALLOW IN    Anywhere
```

### Configuration pour MicroK8s Multi-Node

Si vous avez un cluster multi-node, les nœuds doivent communiquer entre eux.

#### Autoriser la communication entre nœuds du cluster

Supposons que vos nœuds ont les IPs suivantes :
- Nœud 1 : 192.168.1.101
- Nœud 2 : 192.168.1.102
- Nœud 3 : 192.168.1.103

**Sur chaque nœud, autoriser les autres nœuds :**

```bash
# Sur le nœud 1
sudo ufw allow from 192.168.1.102 comment 'MicroK8s Node 2'
sudo ufw allow from 192.168.1.103 comment 'MicroK8s Node 3'

# Sur le nœud 2
sudo ufw allow from 192.168.1.101 comment 'MicroK8s Node 1'
sudo ufw allow from 192.168.1.103 comment 'MicroK8s Node 3'

# Sur le nœud 3
sudo ufw allow from 192.168.1.101 comment 'MicroK8s Node 1'
sudo ufw allow from 192.168.1.102 comment 'MicroK8s Node 2'
```

#### Autoriser les ports spécifiques inter-nœuds

Pour plus de finesse, autoriser uniquement les ports nécessaires :

```bash
# Sur tous les nœuds
sudo ufw allow from 192.168.1.0/24 to any port 16443 proto tcp comment 'K8s API'
sudo ufw allow from 192.168.1.0/24 to any port 10250 proto tcp comment 'Kubelet'
sudo ufw allow from 192.168.1.0/24 to any port 25000 proto tcp comment 'Cluster Agent'
sudo ufw allow from 192.168.1.0/24 to any port 12379 proto tcp comment 'Dqlite'
sudo ufw allow from 192.168.1.0/24 to any port 179 proto tcp comment 'BGP Calico'
sudo ufw allow from 192.168.1.0/24 to any port 4789 proto udp comment 'VXLAN Calico'
```

### Règles Avancées UFW

#### Limiter les tentatives SSH (protection brute-force)

```bash
sudo ufw limit 22/tcp comment 'SSH rate limit'
```

Cette règle bloque temporairement les IPs qui tentent plus de 6 connexions en 30 secondes.

#### Autoriser l'accès uniquement depuis un réseau spécifique

Si vous voulez que seules les machines de votre réseau local accèdent au Dashboard :

```bash
sudo ufw allow from 192.168.1.0/24 to any port 16443 proto tcp comment 'K8s API local only'
```

#### Bloquer une IP spécifique

```bash
sudo ufw deny from 203.0.113.50 comment 'Blocked suspicious IP'
```

#### Autoriser un addon spécifique (Registry)

```bash
sudo ufw allow from 192.168.1.0/24 to any port 5000 proto tcp comment 'Registry local'
```

### Logging UFW

Activer les logs pour surveiller les tentatives de connexion :

```bash
# Activer le logging
sudo ufw logging on

# Niveau de log (low, medium, high, full)
sudo ufw logging medium
```

Les logs se trouvent dans `/var/log/ufw.log`.

**Visualiser les logs :**

```bash
sudo tail -f /var/log/ufw.log
```

### Commandes UFW Utiles

| Commande | Description |
|----------|-------------|
| `sudo ufw status numbered` | Afficher les règles avec numéros |
| `sudo ufw delete [numéro]` | Supprimer une règle par numéro |
| `sudo ufw delete allow 80/tcp` | Supprimer une règle par description |
| `sudo ufw reset` | Réinitialiser toutes les règles |
| `sudo ufw disable` | Désactiver le firewall |
| `sudo ufw reload` | Recharger les règles |

### Exemple de Script de Configuration UFW Complet

Créez un script pour automatiser la configuration :

```bash
#!/bin/bash
# configure-ufw-microk8s.sh

echo "Configuration UFW pour MicroK8s..."

# Réinitialiser UFW
sudo ufw --force reset

# Politiques par défaut
sudo ufw default deny incoming
sudo ufw default allow outgoing

# Autoriser localhost
sudo ufw allow in on lo
sudo ufw allow out on lo

# SSH avec limitation
sudo ufw limit 22/tcp comment 'SSH'

# Kubernetes API (administration distante)
sudo ufw allow 16443/tcp comment 'Kubernetes API'

# HTTP et HTTPS
sudo ufw allow 80/tcp comment 'HTTP'
sudo ufw allow 443/tcp comment 'HTTPS'

# Si cluster multi-node, décommenter :
# CLUSTER_NETWORK="192.168.1.0/24"
# sudo ufw allow from $CLUSTER_NETWORK to any port 10250 proto tcp comment 'Kubelet'
# sudo ufw allow from $CLUSTER_NETWORK to any port 25000 proto tcp comment 'Cluster Agent'
# sudo ufw allow from $CLUSTER_NETWORK to any port 12379 proto tcp comment 'Dqlite'
# sudo ufw allow from $CLUSTER_NETWORK to any port 179 proto tcp comment 'BGP'
# sudo ufw allow from $CLUSTER_NETWORK to any port 4789 proto udp comment 'VXLAN'

# Activer le logging
sudo ufw logging medium

# Activer UFW
sudo ufw --force enable

echo "Configuration terminée !"
sudo ufw status verbose
```

Rendre le script exécutable et l'exécuter :

```bash
chmod +x configure-ufw-microk8s.sh
sudo ./configure-ufw-microk8s.sh
```

---

## Configuration Firewalld (CentOS/RHEL/Fedora)

Firewalld est le firewall par défaut sur les distributions Red Hat.

### Installation

```bash
sudo dnf install firewalld  # Fedora/RHEL 8+
# ou
sudo yum install firewalld  # CentOS 7
```

### Démarrage du service

```bash
sudo systemctl start firewalld
sudo systemctl enable firewalld
```

### Vérification du statut

```bash
sudo firewall-cmd --state
```

### Concepts de Firewalld

Firewalld utilise des **zones** pour organiser les règles. Les principales zones :

| Zone | Niveau de confiance | Usage |
|------|---------------------|-------|
| **drop** | Minimal | Tout est bloqué sauf sortant |
| **block** | Bas | Tout bloqué avec réponse rejet |
| **public** | Moyen | Par défaut, quelques services autorisés |
| **trusted** | Maximum | Tout autorisé |
| **home** | Élevé | Réseau domestique |
| **internal** | Élevé | Réseau interne |

### Configuration de Base pour MicroK8s

#### Étape 1 : Vérifier la zone active

```bash
sudo firewall-cmd --get-active-zones
```

La zone par défaut est généralement `public`.

#### Étape 2 : Autoriser les services de base

```bash
# SSH (généralement déjà autorisé)
sudo firewall-cmd --permanent --add-service=ssh

# HTTP et HTTPS
sudo firewall-cmd --permanent --add-service=http
sudo firewall-cmd --permanent --add-service=https
```

#### Étape 3 : Autoriser les ports MicroK8s

```bash
# API Kubernetes
sudo firewall-cmd --permanent --add-port=16443/tcp

# Kubelet (si multi-node)
sudo firewall-cmd --permanent --add-port=10250/tcp

# Cluster Agent (si multi-node)
sudo firewall-cmd --permanent --add-port=25000/tcp

# Dqlite (si HA)
sudo firewall-cmd --permanent --add-port=12379/tcp
```

#### Étape 4 : Autoriser Calico (si multi-node)

```bash
# BGP
sudo firewall-cmd --permanent --add-port=179/tcp

# VXLAN
sudo firewall-cmd --permanent --add-port=4789/udp
```

#### Étape 5 : Recharger la configuration

```bash
sudo firewall-cmd --reload
```

#### Étape 6 : Vérifier les règles

```bash
sudo firewall-cmd --list-all
```

### Configuration Multi-Node avec Zone Trusted

Pour simplifier la communication entre nœuds, créez une zone trusted pour le réseau du cluster :

```bash
# Créer une zone trusted pour le cluster
sudo firewall-cmd --permanent --new-zone=k8s-cluster

# Ajouter le réseau du cluster à cette zone
sudo firewall-cmd --permanent --zone=k8s-cluster --add-source=192.168.1.0/24

# Autoriser tout le trafic dans cette zone
sudo firewall-cmd --permanent --zone=k8s-cluster --set-target=ACCEPT

# Recharger
sudo firewall-cmd --reload
```

Vérification :

```bash
sudo firewall-cmd --zone=k8s-cluster --list-all
```

### Rich Rules pour Contrôle Avancé

Les "rich rules" permettent des règles plus complexes.

#### Limiter l'accès à l'API Kubernetes à un réseau spécifique

```bash
sudo firewall-cmd --permanent --add-rich-rule='
  rule family="ipv4"
  source address="192.168.1.0/24"
  port protocol="tcp" port="16443"
  accept'
```

#### Bloquer une IP spécifique

```bash
sudo firewall-cmd --permanent --add-rich-rule='
  rule family="ipv4"
  source address="203.0.113.50"
  reject'
```

#### Logger les connexions sur un port

```bash
sudo firewall-cmd --permanent --add-rich-rule='
  rule family="ipv4"
  port protocol="tcp" port="16443"
  log prefix="K8S-API: " level="info"
  accept'
```

### Port Forwarding avec Firewalld

Si vous voulez rediriger un port externe vers un service interne :

```bash
# Rediriger le port 8080 externe vers 80 interne
sudo firewall-cmd --permanent --add-forward-port=port=8080:proto=tcp:toport=80

sudo firewall-cmd --reload
```

### Commandes Firewalld Utiles

| Commande | Description |
|----------|-------------|
| `sudo firewall-cmd --list-all` | Afficher toutes les règles |
| `sudo firewall-cmd --list-ports` | Lister les ports ouverts |
| `sudo firewall-cmd --list-services` | Lister les services autorisés |
| `sudo firewall-cmd --get-zones` | Lister toutes les zones |
| `sudo firewall-cmd --remove-port=80/tcp --permanent` | Supprimer un port |
| `sudo firewall-cmd --runtime-to-permanent` | Sauvegarder les règles temporaires |

### Exemple de Script de Configuration Firewalld

```bash
#!/bin/bash
# configure-firewalld-microk8s.sh

echo "Configuration Firewalld pour MicroK8s..."

# Services de base
sudo firewall-cmd --permanent --add-service=ssh
sudo firewall-cmd --permanent --add-service=http
sudo firewall-cmd --permanent --add-service=https

# Ports MicroK8s
sudo firewall-cmd --permanent --add-port=16443/tcp  # K8s API
sudo firewall-cmd --permanent --add-port=10250/tcp  # Kubelet
sudo firewall-cmd --permanent --add-port=25000/tcp  # Cluster Agent
sudo firewall-cmd --permanent --add-port=12379/tcp  # Dqlite

# Calico CNI
sudo firewall-cmd --permanent --add-port=179/tcp    # BGP
sudo firewall-cmd --permanent --add-port=4789/udp   # VXLAN

# Zone trusted pour le cluster (optionnel)
CLUSTER_NETWORK="192.168.1.0/24"
sudo firewall-cmd --permanent --new-zone=k8s-cluster 2>/dev/null || true
sudo firewall-cmd --permanent --zone=k8s-cluster --add-source=$CLUSTER_NETWORK
sudo firewall-cmd --permanent --zone=k8s-cluster --set-target=ACCEPT

# Recharger
sudo firewall-cmd --reload

echo "Configuration terminée !"
sudo firewall-cmd --list-all
```

---

## Configuration Windows Firewall (pour WSL2)

Si vous utilisez MicroK8s dans WSL2 sur Windows, vous devez configurer le firewall Windows.

### Via PowerShell (Administrateur)

#### Autoriser les ports essentiels

```powershell
# HTTP
New-NetFirewallRule -DisplayName "MicroK8s HTTP" -Direction Inbound -Protocol TCP -LocalPort 80 -Action Allow

# HTTPS
New-NetFirewallRule -DisplayName "MicroK8s HTTPS" -Direction Inbound -Protocol TCP -LocalPort 443 -Action Allow

# Kubernetes API
New-NetFirewallRule -DisplayName "MicroK8s API" -Direction Inbound -Protocol TCP -LocalPort 16443 -Action Allow
```

#### Autoriser un réseau spécifique

```powershell
New-NetFirewallRule -DisplayName "MicroK8s from local network" `
  -Direction Inbound `
  -Protocol TCP `
  -LocalPort 80,443,16443 `
  -RemoteAddress 192.168.1.0/24 `
  -Action Allow
```

### Via Interface Graphique

1. Ouvrir **Paramètres Windows** → **Réseau et Internet** → **Pare-feu Windows Defender**
2. Cliquer sur **Paramètres avancés**
3. Sélectionner **Règles de trafic entrant**
4. Cliquer sur **Nouvelle règle...**
5. Choisir **Port** → **Suivant**
6. Sélectionner **TCP** et entrer les ports : `80, 443, 16443`
7. Choisir **Autoriser la connexion**
8. Appliquer à tous les profils (Domaine, Privé, Public)
9. Donner un nom : "MicroK8s Ports"
10. Terminer

### Port Forwarding WSL2 vers Windows

WSL2 utilise une interface réseau virtuelle. Pour exposer les services au réseau :

**Script PowerShell de port forwarding :**

```powershell
# Obtenir l'IP de WSL2
$wsl_ip = (wsl hostname -I).Trim()

# Rediriger les ports
netsh interface portproxy add v4tov4 listenport=80 listenaddress=0.0.0.0 connectport=80 connectaddress=$wsl_ip
netsh interface portproxy add v4tov4 listenport=443 listenaddress=0.0.0.0 connectport=443 connectaddress=$wsl_ip
netsh interface portproxy add v4tov4 listenport=16443 listenaddress=0.0.0.0 connectport=16443 connectaddress=$wsl_ip

# Afficher les redirections
netsh interface portproxy show all
```

**Pour supprimer les redirections :**

```powershell
netsh interface portproxy delete v4tov4 listenport=80 listenaddress=0.0.0.0
netsh interface portproxy delete v4tov4 listenport=443 listenaddress=0.0.0.0
netsh interface portproxy delete v4tov4 listenport=16443 listenaddress=0.0.0.0
```

---

## Partie 2 : Network Policies Kubernetes

### Qu'est-ce qu'une Network Policy ?

Les Network Policies sont des règles au niveau Kubernetes qui contrôlent le trafic entre les pods. C'est le troisième niveau de défense, à l'intérieur même du cluster.

### Pourquoi utiliser des Network Policies ?

Par défaut, dans Kubernetes, **tous les pods peuvent communiquer avec tous les autres pods**. Les Network Policies permettent de :

1. **Isoler les applications sensibles** : par exemple, isoler la base de données
2. **Appliquer le principe du moindre privilège** : chaque pod n'a accès qu'à ce dont il a besoin
3. **Segmenter le réseau** : créer des zones de sécurité (DMZ, backend, database)
4. **Se conformer aux normes de sécurité** : PCI-DSS, HIPAA, etc.

### Prérequis

Les Network Policies nécessitent un plugin CNI qui les supporte. MicroK8s utilise **Calico** par défaut, qui supporte pleinement les Network Policies.

Vérifier que Calico est actif :

```bash
microk8s kubectl get pods -n kube-system -l k8s-app=calico-node
```

### Concept de Base

Une Network Policy définit :

1. **À qui elle s'applique** : sélection des pods via labels
2. **Type de trafic** : Ingress (entrant) et/ou Egress (sortant)
3. **Règles** : qui peut communiquer et sur quels ports

### Comportement par Défaut

**Important :** Dès qu'une Network Policy sélectionne un pod, le comportement change :

- **Sans Network Policy** : tout est autorisé
- **Avec Network Policy** : tout est refusé sauf ce qui est explicitement autorisé

### Exemple 1 : Bloquer Tout le Trafic Entrant

La Network Policy la plus restrictive :

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: deny-all-ingress
  namespace: production
spec:
  podSelector: {}  # Sélectionne tous les pods du namespace
  policyTypes:
  - Ingress
```

Cette politique bloque tout le trafic entrant vers tous les pods du namespace `production`.

### Exemple 2 : Autoriser Uniquement un Pod Spécifique

Autoriser uniquement le pod `frontend` à communiquer avec le pod `backend` :

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-frontend-to-backend
  namespace: production
spec:
  podSelector:
    matchLabels:
      app: backend  # S'applique aux pods avec label app=backend
  policyTypes:
  - Ingress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: frontend  # Autorise uniquement les pods avec label app=frontend
    ports:
    - protocol: TCP
      port: 8080  # Sur le port 8080 uniquement
```

**Schéma logique :**

```
┌──────────────┐           ┌──────────────┐
│  Frontend    │  ✓        │   Backend    │
│ app=frontend │ ────────> │ app=backend  │
└──────────────┘  Port 8080└──────────────┘
                              │
┌──────────────┐              │ ✗
│ Autre Pod    │ ─────────────┘
└──────────────┘
```

### Exemple 3 : Autoriser le Trafic depuis un Namespace

Autoriser tous les pods du namespace `monitoring` à accéder au namespace `production` :

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-monitoring
  namespace: production
spec:
  podSelector: {}  # Tous les pods de production
  policyTypes:
  - Ingress
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          name: monitoring
```

**Note :** Le namespace `monitoring` doit avoir le label `name: monitoring`. Pour l'ajouter :

```bash
microk8s kubectl label namespace monitoring name=monitoring
```

### Exemple 4 : Isolation Complète d'une Base de Données

Scénario réaliste : une base de données PostgreSQL qui ne doit être accessible que par l'API backend.

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: postgres-isolation
  namespace: production
spec:
  podSelector:
    matchLabels:
      app: postgres
  policyTypes:
  - Ingress
  - Egress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: api-backend  # Seul le backend peut accéder
    ports:
    - protocol: TCP
      port: 5432  # Port PostgreSQL
  egress:
  - to:
    - podSelector:
        matchLabels:
          k8s-app: kube-dns  # Autoriser la résolution DNS
    ports:
    - protocol: UDP
      port: 53
```

Cette politique :
- ✓ Autorise le backend à se connecter sur le port 5432
- ✓ Autorise la base de données à résoudre des DNS
- ✗ Bloque tout autre trafic

### Exemple 5 : Autoriser le Trafic depuis une IP Externe

Autoriser un serveur de monitoring externe (ex: Prometheus externe) à scraper les métriques :

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-external-monitoring
  namespace: production
spec:
  podSelector:
    matchLabels:
      metrics: "true"
  policyTypes:
  - Ingress
  ingress:
  - from:
    - ipBlock:
        cidr: 203.0.113.0/24  # Réseau du monitoring externe
    ports:
    - protocol: TCP
      port: 9090  # Port métriques Prometheus
```

### Exemple 6 : Politique Egress (Sortant)

Contrôler les connexions sortantes d'un pod. Par exemple, empêcher une application de se connecter à Internet sauf vers des API spécifiques :

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: restrict-egress
  namespace: production
spec:
  podSelector:
    matchLabels:
      app: webapp
  policyTypes:
  - Egress
  egress:
  - to:
    - podSelector:
        matchLabels:
          app: backend  # Autorise connexion au backend interne
  - to:
    - namespaceSelector:
        matchLabels:
          name: kube-system
    - podSelector:
        matchLabels:
          k8s-app: kube-dns  # Autorise la résolution DNS
    ports:
    - protocol: UDP
      port: 53
  - to:
    - ipBlock:
        cidr: 0.0.0.0/0
        except:
        - 10.0.0.0/8
        - 192.168.0.0/16
        - 172.16.0.0/12  # Bloque les IPs privées
    ports:
    - protocol: TCP
      port: 443  # Autorise HTTPS externe uniquement
```

Cette politique permet au pod `webapp` de :
- ✓ Communiquer avec le backend interne
- ✓ Résoudre des DNS
- ✓ Se connecter en HTTPS vers Internet
- ✗ Accéder à d'autres réseaux privés

### Architecture de Sécurité en Couches

Exemple complet d'une architecture 3-tiers sécurisée :

```yaml
---
# Namespace avec label
apiVersion: v1
kind: Namespace
metadata:
  name: secure-app
  labels:
    name: secure-app

---
# 1. Frontend : accessible depuis Internet
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: frontend-policy
  namespace: secure-app
spec:
  podSelector:
    matchLabels:
      tier: frontend
  policyTypes:
  - Ingress
  - Egress
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          name: ingress-nginx  # Ingress Controller
    ports:
    - protocol: TCP
      port: 80
  egress:
  - to:
    - podSelector:
        matchLabels:
          tier: backend
    ports:
    - protocol: TCP
      port: 8080
  - to:
    - namespaceSelector:
        matchLabels:
          name: kube-system
    - podSelector:
        matchLabels:
          k8s-app: kube-dns
    ports:
    - protocol: UDP
      port: 53

---
# 2. Backend : accessible uniquement depuis frontend
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: backend-policy
  namespace: secure-app
spec:
  podSelector:
    matchLabels:
      tier: backend
  policyTypes:
  - Ingress
  - Egress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          tier: frontend
    ports:
    - protocol: TCP
      port: 8080
  egress:
  - to:
    - podSelector:
        matchLabels:
          tier: database
    ports:
    - protocol: TCP
      port: 5432
  - to:
    - namespaceSelector:
        matchLabels:
          name: kube-system
    - podSelector:
        matchLabels:
          k8s-app: kube-dns
    ports:
    - protocol: UDP
      port: 53

---
# 3. Database : accessible uniquement depuis backend
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: database-policy
  namespace: secure-app
spec:
  podSelector:
    matchLabels:
      tier: database
  policyTypes:
  - Ingress
  - Egress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          tier: backend
    ports:
    - protocol: TCP
      port: 5432
  egress:
  - to:
    - namespaceSelector:
        matchLabels:
          name: kube-system
    - podSelector:
        matchLabels:
          k8s-app: kube-dns
    ports:
    - protocol: UDP
      port: 53
```

**Flux de trafic autorisé :**

```
Internet → Ingress → Frontend → Backend → Database
                        │                     │
                        └──── DNS ────────────┘
```

### Tester les Network Policies

#### Déployer des pods de test

```bash
# Pod frontend
microk8s kubectl run frontend --image=nginx --labels="tier=frontend" -n secure-app

# Pod backend
microk8s kubectl run backend --image=nginx --labels="tier=backend" -n secure-app

# Pod database
microk8s kubectl run database --image=nginx --labels="tier=database" -n secure-app

# Pod non autorisé
microk8s kubectl run unauthorized --image=nginx -n secure-app
```

#### Tester la connectivité

```bash
# Depuis frontend vers backend (devrait fonctionner)
microk8s kubectl exec -it frontend -n secure-app -- curl backend:80

# Depuis unauthorized vers database (devrait échouer)
microk8s kubectl exec -it unauthorized -n secure-app -- curl database:80
```

Si les Network Policies fonctionnent, la deuxième commande devrait échouer avec un timeout.

### Visualiser les Network Policies

```bash
# Lister toutes les Network Policies
microk8s kubectl get networkpolicies --all-namespaces

# Détails d'une politique
microk8s kubectl describe networkpolicy frontend-policy -n secure-app
```

---

## Bonnes Pratiques de Sécurité Firewall

### 1. Principe du Moindre Privilège

N'ouvrez **que les ports strictement nécessaires**. Si vous n'avez pas besoin d'exposer le Dashboard, ne l'exposez pas.

### 2. Défense en Profondeur

Combinez plusieurs niveaux de sécurité :

```
Internet
   ↓
Firewall Routeur (filtrage grossier)
   ↓
Firewall OS (UFW/firewalld)
   ↓
Network Policies (micro-segmentation)
   ↓
Application (authentification)
```

### 3. Séparation des Environnements

Utilisez des namespaces différents pour les environnements :

```bash
microk8s kubectl create namespace dev
microk8s kubectl create namespace staging
microk8s kubectl create namespace production
```

Appliquez des Network Policies différentes à chaque namespace.

### 4. Whitelist plutôt que Blacklist

**Mauvaise approche :** "Bloquer les IPs malveillantes connues"
**Bonne approche :** "N'autoriser que les IPs/réseaux de confiance"

### 5. Monitoring et Alertes

Surveillez les connexions bloquées par le firewall :

**UFW :**
```bash
sudo tail -f /var/log/ufw.log | grep BLOCK
```

**Firewalld :**
```bash
sudo journalctl -f -u firewalld | grep REJECT
```

**Kubernetes Network Policies :**

Utiliser Calico pour voir les logs :
```bash
microk8s kubectl logs -n kube-system -l k8s-app=calico-node | grep DROP
```

### 6. Documentation

Maintenez un document listant :
- Tous les ports ouverts et leur justification
- Les Network Policies appliquées
- Les IPs/réseaux autorisés

### 7. Tests Réguliers

Testez périodiquement votre configuration :

```bash
# Scanner de ports externe
nmap -p 1-65535 votre-ip-publique

# Depuis une machine externe
curl http://votre-domaine.com
curl https://votre-domaine.com
```

### 8. Mise à Jour Régulière

```bash
# Ubuntu
sudo apt update && sudo apt upgrade

# CentOS/RHEL
sudo dnf update
```

Les mises à jour incluent souvent des correctifs de sécurité pour le firewall.

---

## Scénarios Courants et Solutions

### Scénario 1 : Lab de Développement Local

**Objectif :** Sécurité basique, facilité d'utilisation

**Configuration UFW :**

```bash
sudo ufw allow 22/tcp     # SSH
sudo ufw allow 80/tcp     # HTTP
sudo ufw allow 443/tcp    # HTTPS
sudo ufw allow 16443/tcp  # K8s API (accès local)
sudo ufw enable
```

**Network Policies :** Aucune (développement rapide)

### Scénario 2 : Serveur de Production Internet

**Objectif :** Sécurité maximale

**Configuration UFW :**

```bash
sudo ufw allow 22/tcp                                    # SSH
sudo ufw limit 22/tcp                                    # Protection brute-force
sudo ufw allow 80/tcp                                    # HTTP
sudo ufw allow 443/tcp                                   # HTTPS
sudo ufw allow from 192.168.1.0/24 to any port 16443    # K8s API local uniquement
sudo ufw logging high
sudo ufw enable
```

**Network Policies :** Isolation complète de chaque application

### Scénario 3 : Cluster Multi-Node en Datacenter

**Objectif :** Permettre la communication inter-nœuds, bloquer tout le reste

**Configuration UFW (sur chaque nœud) :**

```bash
sudo ufw allow from 10.0.1.0/24  # Réseau interne datacenter
sudo ufw allow 80/tcp from any
sudo ufw allow 443/tcp from any
sudo ufw deny from any           # Bloquer tout le reste
sudo ufw enable
```

**Network Policies :** Segmentation par application

### Scénario 4 : Accès VPN Uniquement

**Objectif :** N'autoriser l'accès qu'aux utilisateurs VPN

**Configuration UFW :**

```bash
# OpenVPN écoute sur UDP 1194
sudo ufw allow 1194/udp

# Autoriser tout depuis le réseau VPN
sudo ufw allow from 10.8.0.0/24

# Bloquer l'accès direct depuis Internet
sudo ufw deny 16443/tcp
sudo ufw deny 80/tcp
sudo ufw deny 443/tcp

sudo ufw enable
```

Les utilisateurs doivent d'abord se connecter au VPN pour accéder au cluster.

---

## Dépannage Firewall

### Problème : Impossible d'accéder à l'API Kubernetes

**Symptôme :**
```bash
Unable to connect to the server: dial tcp 192.168.1.100:16443: i/o timeout
```

**Vérifications :**

1. **Le port est-il ouvert dans le firewall ?**
   ```bash
   sudo ufw status | grep 16443
   ```

2. **L'API Kubernetes est-elle en cours d'exécution ?**
   ```bash
   microk8s status
   ```

3. **Test de connectivité depuis la machine locale :**
   ```bash
   curl -k https://localhost:16443
   ```

4. **Test depuis une machine distante :**
   ```bash
   telnet 192.168.1.100 16443
   ```

**Solution :**
```bash
sudo ufw allow 16443/tcp
```

### Problème : Les applications web ne sont pas accessibles

**Symptôme :** `curl http://monapp.com` timeout

**Vérifications :**

1. **Ports 80/443 ouverts ?**
   ```bash
   sudo ufw status | grep -E "80|443"
   ```

2. **Ingress Controller fonctionne ?**
   ```bash
   microk8s kubectl get pods -n ingress
   microk8s kubectl get svc -n ingress
   ```

3. **LoadBalancer IP assignée ?**
   ```bash
   microk8s kubectl get svc -n ingress
   # EXTERNAL-IP doit avoir une vraie IP
   ```

4. **Test local :**
   ```bash
   curl -H "Host: monapp.com" http://localhost
   ```

**Solution :**
```bash
sudo ufw allow 80/tcp
sudo ufw allow 443/tcp
```

### Problème : Les pods ne peuvent pas communiquer entre eux

**Symptôme :** Un service ne peut pas atteindre un autre service

**Vérifications :**

1. **Y a-t-il des Network Policies appliquées ?**
   ```bash
   microk8s kubectl get networkpolicies --all-namespaces
   ```

2. **Logs du pod :**
   ```bash
   microk8s kubectl logs <pod-name>
   ```

3. **Test de connectivité depuis un pod :**
   ```bash
   microk8s kubectl run test --rm -it --image=busybox -- wget -O- http://backend-service
   ```

**Solution :**

Vérifier ou créer une Network Policy autorisant la communication :

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-internal
  namespace: default
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  ingress:
  - from:
    - podSelector: {}  # Autoriser tous les pods du même namespace
```

### Problème : Cluster multi-node - Les nœuds ne communiquent pas

**Symptôme :** `microk8s add-node` échoue ou les nœuds apparaissent comme `NotReady`

**Vérifications :**

1. **Ports inter-nœuds ouverts ?**
   ```bash
   sudo ufw status | grep -E "10250|25000|12379"
   ```

2. **Test de connectivité :**
   ```bash
   # Depuis le nœud 1 vers le nœud 2
   telnet 192.168.1.102 25000
   ```

3. **Vérifier l'état des nœuds :**
   ```bash
   microk8s kubectl get nodes
   ```

**Solution :**

Sur tous les nœuds :
```bash
sudo ufw allow from 192.168.1.0/24 to any port 10250 proto tcp
sudo ufw allow from 192.168.1.0/24 to any port 25000 proto tcp
sudo ufw allow from 192.168.1.0/24 to any port 12379 proto tcp
sudo ufw allow from 192.168.1.0/24 to any port 179 proto tcp
sudo ufw allow from 192.168.1.0/24 to any port 4789 proto udp
```

### Problème : Logs firewall inondés

**Symptôme :** Des milliers de lignes de logs par seconde

**Cause :** Souvent du trafic légitime (health checks, métriques) qui est logué

**Solution :**

Réduire le niveau de logging :
```bash
sudo ufw logging low
```

Ou créer des règles spécifiques sans logging pour les health checks :

```bash
# Désactiver le logging pour les connexions depuis le réseau local
sudo ufw allow from 192.168.1.0/24 comment 'Internal traffic - no log'
```

---

## Outils de Test et Validation

### 1. nmap - Scanner de ports

Installation :
```bash
sudo apt install nmap  # Ubuntu
sudo dnf install nmap  # CentOS
```

Scanner votre machine :
```bash
# Scan basique
nmap localhost

# Scan complet (tous les ports)
nmap -p 1-65535 localhost

# Scan depuis l'extérieur
nmap votre-ip-publique
```

### 2. netcat (nc) - Test de connectivité

```bash
# Tester si un port est ouvert
nc -zv 192.168.1.100 16443

# -z : mode scan (ne pas envoyer de données)
# -v : verbose
```

### 3. tcpdump - Capturer le trafic réseau

```bash
# Capturer tout le trafic sur le port 80
sudo tcpdump -i any port 80

# Capturer vers un fichier pour analyse
sudo tcpdump -i any port 80 -w capture.pcap

# Analyser avec Wireshark ensuite
```

### 4. iptables - Voir les règles bas niveau

UFW et firewalld utilisent iptables en arrière-plan :

```bash
# Voir toutes les règles iptables
sudo iptables -L -n -v

# Voir les règles NAT
sudo iptables -t nat -L -n -v
```

### 5. ss - Voir les ports en écoute

```bash
# Tous les ports en écoute
sudo ss -tulpn

# Filtrer par port
sudo ss -tulpn | grep :80
```

Explication des options :
- `-t` : TCP
- `-u` : UDP
- `-l` : ports en écoute (listening)
- `-p` : afficher le processus
- `-n` : ne pas résoudre les noms

---

## Checklist de Sécurité Firewall

### Niveau Système (UFW/Firewalld)

- [ ] Firewall activé et configuré pour démarrage automatique
- [ ] SSH autorisé (sinon vous serez bloqué !)
- [ ] Ports HTTP/HTTPS ouverts si nécessaire
- [ ] API Kubernetes (16443) accessible uniquement depuis réseau de confiance
- [ ] Ports inter-nœuds ouverts (si cluster multi-node)
- [ ] Logging activé (niveau medium ou low)
- [ ] Règles documentées
- [ ] Test de connectivité effectué depuis l'intérieur et l'extérieur

### Niveau Kubernetes (Network Policies)

- [ ] Network Policies créées pour les applications sensibles
- [ ] Base de données isolée (uniquement accessible par backend)
- [ ] Frontend isolé (uniquement accessible par Ingress)
- [ ] Politique par défaut deny-all pour les namespaces de production
- [ ] Autorisation DNS explicite dans les politiques Egress
- [ ] Test de connectivité entre pods effectué
- [ ] Documentation des flux de trafic autorisés

### Monitoring et Maintenance

- [ ] Surveillance des logs firewall configurée
- [ ] Alertes sur connexions suspectes configurées
- [ ] Revue mensuelle des règles firewall
- [ ] Scan de ports périodique depuis l'extérieur
- [ ] Mises à jour de sécurité appliquées régulièrement

---

## Résumé et Points Clés

### Trois Niveaux de Protection

1. **Routeur/Box** : première ligne de défense, redirection de ports
2. **Firewall OS** : UFW, firewalld, Windows Firewall
3. **Network Policies** : micro-segmentation Kubernetes

### Ports MicroK8s Essentiels

| Port | Usage | Exposition |
|------|-------|------------|
| 16443 | API Kubernetes | Réseau de confiance uniquement |
| 10250 | Kubelet | Inter-nœuds uniquement |
| 25000 | Cluster Agent | Inter-nœuds uniquement |
| 80 | HTTP applications | Internet/Local selon besoins |
| 443 | HTTPS applications | Internet/Local selon besoins |

### Commandes Rapides

**UFW :**
```bash
sudo ufw status                    # État
sudo ufw allow 80/tcp              # Ouvrir un port
sudo ufw delete allow 80/tcp       # Fermer un port
sudo ufw reset                     # Réinitialiser
```

**Firewalld :**
```bash
sudo firewall-cmd --list-all                      # État
sudo firewall-cmd --permanent --add-port=80/tcp   # Ouvrir un port
sudo firewall-cmd --reload                        # Recharger
```

**Network Policies :**
```bash
microk8s kubectl get networkpolicies              # Lister
microk8s kubectl describe networkpolicy [nom]     # Détails
microk8s kubectl delete networkpolicy [nom]       # Supprimer
```

### Principe Fondamental

**Commencez restrictif, ouvrez progressivement selon les besoins.**

Il est plus facile d'ouvrir un port que de colmater une brèche de sécurité !

---

## Pour Aller Plus Loin

### Resources Recommandées

**Documentation officielle :**
- [Kubernetes Network Policies](https://kubernetes.io/docs/concepts/services-networking/network-policies/)
- [Calico Network Policy](https://docs.projectcalico.org/security/calico-network-policy)
- [UFW Documentation](https://help.ubuntu.com/community/UFW)

**Outils de sécurité :**
- **kube-bench** : vérification des bonnes pratiques de sécurité K8s
- **Falco** : détection d'intrusion pour Kubernetes
- **OPA (Open Policy Agent)** : politiques de sécurité avancées

### Formations Sécurité Kubernetes

- **Certified Kubernetes Security Specialist (CKS)** : certification officielle
- **Kubernetes Security Best Practices** : cours en ligne

### Conclusion

La configuration d'un firewall pour MicroK8s n'est pas seulement une question de "cocher des cases". C'est une réflexion sur l'architecture de sécurité de votre infrastructure.

**Rappelez-vous :**

1. La sécurité est un processus continu, pas un état
2. Combinez plusieurs couches de protection
3. Testez régulièrement vos configurations
4. Documentez tout
5. Surveillez et auditez

Avec une bonne configuration firewall, votre cluster MicroK8s sera protégé tout en restant fonctionnel et accessible pour vos besoins légitimes.

**Prochaines étapes recommandées :**
1. Implémenter les configurations de base présentées
2. Tester avec vos applications
3. Progressivement ajouter des Network Policies
4. Mettre en place un monitoring
5. Planifier des audits de sécurité réguliers

Bonne sécurisation !

⏭️ [Exemples de Network Policies](/annexes/annexe-c-configuration-reseau/exemples-de-network-policies.md)
