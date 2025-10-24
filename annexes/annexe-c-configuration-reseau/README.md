🔝 Retour au [Sommaire](/SOMMAIRE.md)

# Annexe C : Configuration Réseau

## Introduction à la Configuration Réseau dans MicroK8s

Le réseau est l'un des aspects les plus critiques et parfois les plus complexes de Kubernetes. Cette annexe vous guidera à travers tous les aspects de la configuration réseau dans MicroK8s, de manière progressive et accessible, même si vous débutez avec les concepts de réseau conteneurisé.

### Pourquoi le Réseau est Important ?

Dans un environnement Kubernetes, le réseau permet :

1. **Communication entre pods** : Vos applications doivent pouvoir se parler
2. **Exposition des services** : Rendre vos applications accessibles depuis l'extérieur
3. **Isolation et sécurité** : Contrôler qui peut communiquer avec qui
4. **Scalabilité** : Distribuer la charge entre plusieurs instances
5. **Haute disponibilité** : Maintenir le service même en cas de panne

**Analogie simple :** Imaginez un immeuble de bureaux (votre cluster). Le réseau est comme le système de télécommunication : téléphones internes pour la communication entre bureaux (pods), standard téléphonique pour l'extérieur (services), et système de badges pour contrôler les accès (network policies).

### Vue d'Ensemble du Réseau Kubernetes

Kubernetes repose sur plusieurs concepts réseau interconnectés :

```
┌────────────────────────────────────────────────────────────┐
│                    CLUSTER KUBERNETES                      │
│                                                            │
│  ┌────────────────────────────────────────────────────┐    │
│  │              RÉSEAU DE PODS (Pod Network)          │    │
│  │  Chaque pod a une IP unique                        │    │
│  │  Communication directe pod-to-pod                  │    │
│  │  Géré par le CNI (Calico dans MicroK8s)            │    │
│  └────────────────────────────────────────────────────┘    │
│                           │                                │
│                           ▼                                │
│  ┌────────────────────────────────────────────────────┐    │
│  │               SERVICES KUBERNETES                  │    │
│  │  Abstraction stable pour accéder aux pods          │    │
│  │  Load balancing automatique                        │    │
│  │  DNS interne (CoreDNS)                             │    │
│  └────────────────────────────────────────────────────┘    │
│                           │                                │
│                           ▼                                │
│  ┌────────────────────────────────────────────────────┐    │
│  │           INGRESS / LOAD BALANCER                  │    │
│  │  Exposition vers l'extérieur                       │    │
│  │  Routage HTTP/HTTPS                                │    │
│  │  Certificats SSL/TLS                               │    │
│  └────────────────────────────────────────────────────┘    │
│                                                            │
└────────────────────────────────────────────────────────────┘
                           │
                           ▼
                    INTERNET / RÉSEAU EXTERNE
```

---

## Les Quatre Piliers du Réseau Kubernetes

### 1. Réseau de Pods (Pod Network)

**Concept :** Chaque pod dans Kubernetes reçoit sa propre adresse IP unique.

**Principes fondamentaux :**
- Tous les pods peuvent communiquer entre eux sans NAT
- Chaque pod voit les autres avec leur vraie IP
- Les pods sont éphémères, leurs IPs peuvent changer

**Exemple dans MicroK8s :**

```bash
# Lister les pods avec leurs IPs
microk8s kubectl get pods -o wide

# Résultat typique :
# NAME        READY   STATUS    IP            NODE
# frontend    1/1     Running   10.1.128.5    node1
# backend     1/1     Running   10.1.128.6    node1
```

**Dans la pratique :**

```bash
# Depuis le pod frontend, vous pouvez directement accéder au backend
microk8s kubectl exec frontend -- curl http://10.1.128.6:8080
```

### 2. Services

**Concept :** Une abstraction stable pour accéder à un ensemble de pods.

**Problème résolu :** Les pods meurent et renaissent avec de nouvelles IPs. Les Services fournissent une IP virtuelle stable et un nom DNS.

**Types de Services :**

| Type | Description | Cas d'usage |
|------|-------------|-------------|
| **ClusterIP** | IP interne au cluster (défaut) | Communication interne uniquement |
| **NodePort** | Expose un port sur chaque nœud | Accès externe simple |
| **LoadBalancer** | Crée un load balancer externe | Production avec trafic externe |
| **ExternalName** | Alias DNS vers un service externe | Intégration services externes |

**Schéma ClusterIP :**

```
┌──────────────────────────────────────┐
│  Service: backend                    │
│  Type: ClusterIP                     │
│  IP: 10.152.183.100                  │
│  Port: 8080                          │
│  Selector: app=backend               │
└──────────┬───────────────────────────┘
           │ Load balance entre :
           │
    ┌──────┴───────┬─────────────┐
    │              │             │
    ▼              ▼             ▼
┌─────────┐  ┌─────────┐  ┌─────────┐
│Backend-1│  │Backend-2│  │Backend-3│
│Pod      │  │Pod      │  │Pod      │
└─────────┘  └─────────┘  └─────────┘
```

### 3. Ingress

**Concept :** Routage HTTP/HTTPS intelligent pour exposer plusieurs services via un seul point d'entrée.

**Avantages :**
- Un seul point d'entrée (IP/domaine) pour toutes vos applications
- Routage basé sur le nom d'hôte ou le chemin
- Terminaison SSL/TLS
- Réduction des coûts (une seule IP publique)

**Exemple de routage :**

```
                   ┌─────────────┐
                   │   Ingress   │
                   │ 203.0.113.50│
                   └──────┬──────┘
                          │
        ┌─────────────────┼─────────────────┐
        │                 │                 │
   Host: app1.com    Host: app2.com    Path: /api
        │                 │                 │
        ▼                 ▼                 ▼
   ┌────────┐        ┌────────┐       ┌────────┐
   │Service │        │Service │       │Service │
   │  App1  │        │  App2  │       │  API   │
   └────────┘        └────────┘       └────────┘
```

### 4. Network Policies

**Concept :** Firewall au niveau des pods pour contrôler le trafic.

**Par défaut :** Kubernetes autorise toute communication entre pods.

**Avec Network Policies :** Vous définissez explicitement qui peut parler à qui.

**Exemple simple :**

```
Sans Network Policy :
┌──────────┐     ┌──────────┐     ┌──────────┐
│Frontend  │────►│Backend   │────►│Database  │
└──────────┘  ✓  └──────────┘  ✓  └──────────┘
     │                                   ▲
     └───────────────────────────────────┘
                    ✓

Avec Network Policy :
┌──────────┐     ┌──────────┐     ┌──────────┐
│Frontend  │────►│Backend   │────►│Database  │
└──────────┘  ✓  └──────────┘  ✓  └──────────┘
     │                                   ▲
     └───────────────────────────────────┘
                    ✗ (bloqué)
```

---

## Spécificités MicroK8s

### CNI : Calico par Défaut

MicroK8s utilise **Calico** comme plugin CNI (Container Network Interface). Calico est responsable de :

1. **Attribution des IPs** aux pods
2. **Routage** du trafic entre pods
3. **Application des Network Policies**
4. **Support de BGP** pour du routage avancé

**Vérifier Calico :**

```bash
microk8s kubectl get pods -n kube-system -l k8s-app=calico-node

# Devrait afficher un pod calico-node par nœud
```

### Architecture Réseau MicroK8s

#### Single Node

```
┌────────────────────────────────────────────────┐
│              Machine Hôte                      │
│  IP: 192.168.1.100                             │
│                                                │
│  ┌───────────────────────────────────────────┐ │
│  │         MicroK8s Cluster                  │ │
│  │                                           │ │
│  │  ┌──────────────────────┐                 │ │
│  │  │  Calico              │                 │ │
│  │  │  Réseau: 10.1.0.0/16 │                 │ │
│  │  └─────────┬────────────┘                 │ │
│  │            │                              │ │
│  │     ┌──────┴────────┐                     │ │
│  │     │               │                     │ │
│  │  ┌──▼───┐       ┌───▼──┐                  │ │
│  │  │ Pod1 │       │ Pod2 │                  │ │
│  │  │ IP:  │       │ IP:  │                  │ │
│  │  │10.1.1│       │10.1.2│                  │ │
│  │  └──────┘       └──────┘                  │ │
│  │                                           │ │
│  └───────────────────────────────────────────┘ │
└────────────────────────────────────────────────┘
```

#### Multi-Node

```
┌──────────────────────┐     ┌──────────────────────┐
│   Node 1 (Control)   │     │   Node 2 (Worker)    │
│  IP: 192.168.1.101   │     │  IP: 192.168.1.102   │
│                      │     │                      │
│  Calico: 10.1.0.0/24 │────►│ Calico: 10.1.1.0/24  │
│                      │ BGP │                      │
│  ┌────┐   ┌────┐     │     │  ┌────┐   ┌────┐     │
│  │Pod1│   │Pod2│     │     │  │Pod3│   │Pod4│     │
│  └────┘   └────┘     │     │  └────┘   └────┘     │
└──────────────────────┘     └──────────────────────┘
           │                           │
           └──────────────┬────────────┘
                          │
                   Réseau Physique
                   192.168.1.0/24
```

### Plages IP par Défaut

MicroK8s utilise les plages suivantes :

| Ressource | Plage IP par défaut | Modifiable |
|-----------|---------------------|------------|
| **Pods** | 10.1.0.0/16 | Oui (via Calico config) |
| **Services** | 10.152.183.0/24 | Oui (kube-apiserver config) |
| **DNS (CoreDNS)** | 10.152.183.10 | Non (dynamique) |

**Voir les plages actuelles :**

```bash
# Plage des pods
microk8s kubectl cluster-info dump | grep -i cluster-cidr

# Plage des services
microk8s kubectl cluster-info dump | grep -i service-cluster-ip-range
```

### Addons Réseau MicroK8s

MicroK8s propose plusieurs addons pour enrichir les capacités réseau :

| Addon | Fonction | Commande |
|-------|----------|----------|
| **dns** | Résolution de noms interne (CoreDNS) | `microk8s enable dns` |
| **ingress** | Contrôleur Ingress NGINX | `microk8s enable ingress` |
| **metallb** | Load Balancer pour bare metal | `microk8s enable metallb` |
| **cert-manager** | Gestion automatique des certificats | `microk8s enable cert-manager` |

---

## Concepts Clés de Réseau

### 1. CIDR (Classless Inter-Domain Routing)

**Définition :** Notation pour définir une plage d'adresses IP.

**Format :** `IP/masque`

**Exemples :**

| CIDR | Nombre d'IPs | Usage typique |
|------|--------------|---------------|
| `192.168.1.0/24` | 256 (254 utilisables) | Réseau local standard |
| `10.0.0.0/8` | 16 777 216 | Grand réseau privé |
| `10.1.0.0/16` | 65 536 | Réseau de pods Kubernetes |
| `203.0.113.50/32` | 1 | IP unique |

**Calculateur en ligne :** https://www.ipaddressguide.com/cidr

### 2. NAT (Network Address Translation)

**Définition :** Traduction d'adresses IP privées en adresses publiques.

**Dans MicroK8s :**

```
Pod interne          →  NAT  →  Internet
10.1.128.5:45678     →       →  203.0.113.50:45678
(IP privée)                     (IP publique)
```

**Pourquoi c'est important :**
- Permet aux pods d'accéder à Internet
- Économise les adresses IP publiques
- Ajoute une couche de sécurité

### 3. Port Forwarding

**Définition :** Redirection de trafic d'un port vers un autre.

**Cas d'usage dans Kubernetes :**

```
Port 80 externe  →  Port forwarding  →  Port 8080 interne
(Ingress)                                 (Application)
```

**Exemple kubectl port-forward :**

```bash
# Accéder à un pod localement
microk8s kubectl port-forward pod/backend 8080:8080

# Maintenant http://localhost:8080 accède au pod
```

### 4. Load Balancing

**Définition :** Distribution du trafic entre plusieurs instances.

**Types dans Kubernetes :**

1. **Service Load Balancing** : Distribution automatique entre pods
2. **Ingress Load Balancing** : Distribution entre services
3. **External Load Balancer** : Load balancer fourni par le cloud provider

**Schéma :**

```
         Requête
            │
            ▼
     ┌──────────────┐
     │Load Balancer │
     └──────┬───────┘
            │
    ┌───────┼───────┐
    │       │       │
    ▼       ▼       ▼
 ┌────┐  ┌────┐  ┌────┐
 │Pod1│  │Pod2│  │Pod3│
 └────┘  └────┘  └────┘
```

### 5. DNS (Domain Name System)

**Définition :** Système de résolution de noms en adresses IP.

**Deux niveaux dans Kubernetes :**

1. **DNS Interne (CoreDNS)** : Résolution des noms de services
   - Format : `service-name.namespace.svc.cluster.local`
   - Exemple : `backend.default.svc.cluster.local` → `10.152.183.100`

2. **DNS Externe** : Résolution depuis Internet
   - Format : `monapp.mondomaine.com`
   - Exemple : `api.example.com` → `203.0.113.50`

---

## Flux Réseau Typiques

### Flux 1 : Communication Pod-to-Pod

```
┌──────────────────────────────────────────────────┐
│  Pod A                                           │
│  IP: 10.1.128.5                                  │
│                                                  │
│  Application fait: curl http://10.1.128.6:8080   │
└──────────────────┬───────────────────────────────┘
                   │
                   │ Direct (même réseau)
                   │
                   ▼
┌──────────────────────────────────────────────────┐
│  Pod B                                           │
│  IP: 10.1.128.6                                  │
│  Port: 8080                                      │
└──────────────────────────────────────────────────┘
```

**Caractéristiques :**
- Communication directe IP-to-IP
- Pas de NAT
- Très rapide
- Pas recommandé (les IPs changent)

### Flux 2 : Communication via Service

```
┌──────────────────────────────────────────────────┐
│  Pod Frontend                                    │
│                                                  │
│  Application fait: curl http://backend:8080      │
└──────────────────┬───────────────────────────────┘
                   │
                   │ 1. Résolution DNS
                   ▼
            ┌─────────────┐
            │   CoreDNS   │
            │   "backend" │
            │   → IP Svc  │
            └──────┬──────┘
                   │ 2. Retour IP Service
                   │
                   ▼
            ┌─────────────┐
            │   Service   │
            │   backend   │
            │ Load Balance│
            └──────┬──────┘
                   │ 3. Sélection d'un pod
                   │
         ┌─────────┼─────────┐
         │         │         │
         ▼         ▼         ▼
     ┌────┐    ┌────┐    ┌────┐
     │Pod1│    │Pod2│    │Pod3│
     └────┘    └────┘    └────┘
```

**Caractéristiques :**
- Recommandé (abstraction stable)
- Load balancing automatique
- Resilient aux changements de pods

### Flux 3 : Accès depuis Internet

```
    Internet
       │
       │ 1. DNS externe
       │    app.example.com → 203.0.113.50
       │
       ▼
┌─────────────┐
│   Routeur   │
│Port Forward │
└──────┬──────┘
       │ 2. Redirection :80 → Node:80
       │
       ▼
┌─────────────┐
│   Node      │
│192.168.1.100│
└──────┬──────┘
       │ 3. Ingress Controller écoute sur :80
       │
       ▼
┌─────────────┐
│  Ingress    │
│ Controller  │
└──────┬──────┘
       │ 4. Routage selon Host header
       │    app.example.com → service-app
       │
       ▼
┌─────────────┐
│  Service    │
│   app       │
└──────┬──────┘
       │ 5. Load balance vers pods
       │
       ▼
┌─────────────┐
│    Pod      │
│ Application │
└─────────────┘
```

**Composants requis :**
1. DNS externe configuré
2. Routeur avec port forwarding
3. Ingress Controller actif
4. Ingress resource créée
5. Service et Pods déployés

---

## Démarrage Rapide - Configuration Minimale

### Étape 1 : Activer les Addons Essentiels

```bash
# DNS pour la résolution interne
microk8s enable dns

# Ingress pour l'exposition HTTP/HTTPS
microk8s enable ingress

# Vérification
microk8s status
```

### Étape 2 : Vérifier le Réseau de Base

```bash
# Vérifier Calico
microk8s kubectl get pods -n kube-system -l k8s-app=calico-node

# Vérifier CoreDNS
microk8s kubectl get pods -n kube-system -l k8s-app=kube-dns

# Vérifier Ingress
microk8s kubectl get pods -n ingress
```

### Étape 3 : Déployer une Application Test

```yaml
# test-deployment.yaml
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: test-app
spec:
  replicas: 2
  selector:
    matchLabels:
      app: test
  template:
    metadata:
      labels:
        app: test
    spec:
      containers:
      - name: nginx
        image: nginx
        ports:
        - containerPort: 80

---
apiVersion: v1
kind: Service
metadata:
  name: test-service
spec:
  selector:
    app: test
  ports:
  - port: 80
    targetPort: 80
  type: ClusterIP
```

Déployer :

```bash
microk8s kubectl apply -f test-deployment.yaml
```

### Étape 4 : Tester la Communication

```bash
# Obtenir l'IP du service
microk8s kubectl get svc test-service

# Tester depuis un pod temporaire
microk8s kubectl run test-client --image=busybox --rm -it --restart=Never -- wget -O- http://test-service
```

Si vous voyez la page d'accueil nginx, votre réseau de base fonctionne ! ✓

---

## Dépannage Réseau - Commandes de Base

### Vérifier la Connectivité Pod

```bash
# Voir les IPs des pods
microk8s kubectl get pods -o wide

# Tester un pod depuis un autre
microk8s kubectl exec pod-source -- ping pod-destination-ip

# Test HTTP
microk8s kubectl exec pod-source -- curl http://pod-destination-ip:8080
```

### Vérifier les Services

```bash
# Lister tous les services
microk8s kubectl get svc --all-namespaces

# Détails d'un service
microk8s kubectl describe svc nom-service

# Vérifier les endpoints (pods derrière le service)
microk8s kubectl get endpoints nom-service
```

### Vérifier DNS

```bash
# Tester la résolution DNS depuis un pod
microk8s kubectl run dnstest --image=busybox:1.28 --rm -it --restart=Never -- nslookup kubernetes.default

# Test sur un service personnalisé
microk8s kubectl run dnstest --image=busybox:1.28 --rm -it --restart=Never -- nslookup mon-service.default.svc.cluster.local
```

### Vérifier Calico

```bash
# Status des nœuds Calico
microk8s kubectl get pods -n kube-system -l k8s-app=calico-node

# Logs Calico
microk8s kubectl logs -n kube-system -l k8s-app=calico-node --tail=50
```

### Vérifier Ingress

```bash
# Pods Ingress Controller
microk8s kubectl get pods -n ingress

# Services Ingress
microk8s kubectl get svc -n ingress

# Ingress resources
microk8s kubectl get ingress --all-namespaces
```

---

## Problèmes Courants et Solutions Rapides

### Problème 1 : Les Pods ne Peuvent pas se Connecter

**Symptômes :**
```bash
microk8s kubectl exec pod-a -- curl http://pod-b-ip:8080
# Timeout ou Connection refused
```

**Causes possibles :**
1. Network Policy bloque le trafic
2. Application pas en écoute sur le bon port
3. Calico ne fonctionne pas correctement

**Solutions :**

```bash
# 1. Vérifier les Network Policies
microk8s kubectl get networkpolicies --all-namespaces

# 2. Vérifier le port du pod
microk8s kubectl describe pod pod-b | grep Port

# 3. Vérifier Calico
microk8s kubectl get pods -n kube-system -l k8s-app=calico-node
```

### Problème 2 : DNS ne Fonctionne pas

**Symptômes :**
```bash
nslookup mon-service
# Server can't find mon-service: NXDOMAIN
```

**Solutions :**

```bash
# Vérifier que l'addon dns est activé
microk8s status

# Activer si nécessaire
microk8s enable dns

# Vérifier CoreDNS
microk8s kubectl get pods -n kube-system -l k8s-app=kube-dns
```

### Problème 3 : Service Inaccessible

**Symptômes :**
```bash
curl http://mon-service:8080
# No route to host
```

**Solutions :**

```bash
# Vérifier que le service existe
microk8s kubectl get svc mon-service

# Vérifier qu'il y a des endpoints
microk8s kubectl get endpoints mon-service

# Si endpoints vide, les pods ne matchent pas le selector
microk8s kubectl describe svc mon-service
# Comparer avec les labels des pods
microk8s kubectl get pods --show-labels
```

### Problème 4 : Ingress ne Route pas

**Symptômes :** `curl http://mon-domaine.com` → 404 ou timeout

**Solutions :**

```bash
# Vérifier l'Ingress resource
microk8s kubectl get ingress
microk8s kubectl describe ingress mon-ingress

# Vérifier le service backend existe
microk8s kubectl get svc nom-service-backend

# Vérifier les logs Ingress Controller
microk8s kubectl logs -n ingress -l app.kubernetes.io/name=ingress-nginx
```

---

## Structure de cette Annexe

Cette annexe C est organisée en sections progressives :

### Sections Principales

1. **Configuration DNS Complète** *(voir fichier séparé)*
   - DNS interne (CoreDNS)
   - DNS externe pour accès Internet
   - Configuration de domaines
   - DynamicDNS

2. **Configuration Firewall** *(voir fichier séparé)*
   - UFW (Ubuntu)
   - Firewalld (CentOS/RHEL)
   - Windows Firewall
   - Règles de sécurité

3. **Exemples de Network Policies** *(voir fichier séparé)*
   - Isolation réseau
   - Sécurité par couches
   - Patterns de production

4. **Configuration des Services** *(à venir)*
   - ClusterIP détaillé
   - NodePort configuration
   - LoadBalancer avec MetalLB

5. **Configuration Ingress Avancée** *(à venir)*
   - Routage complexe
   - Rate limiting
   - Authentication

6. **Troubleshooting Réseau** *(à venir)*
   - Outils de diagnostic
   - Méthodes de débogage
   - Résolution de problèmes complexes

---

## Bonnes Pratiques Réseau

### 1. Utiliser les Services plutôt que les IPs

❌ **Mauvais :**
```yaml
# Application configure directement l'IP du pod
backend_url: http://10.1.128.6:8080
```

✅ **Bon :**
```yaml
# Application utilise le nom du service
backend_url: http://backend:8080
```

### 2. Nommer les Ports

✅ **Recommandé :**
```yaml
spec:
  containers:
  - name: app
    ports:
    - containerPort: 8080
      name: http
    - containerPort: 9090
      name: metrics
```

**Avantage :** Vous pouvez changer le numéro de port sans modifier les Services.

### 3. Utiliser des Namespaces

Séparer les environnements et les équipes :

```bash
microk8s kubectl create namespace dev
microk8s kubectl create namespace staging
microk8s kubectl create namespace production
```

### 4. Limiter l'Exposition

Ne pas exposer toutes les applications sur Internet :

```
Internet ─► Ingress ─► Frontend ─► Backend ─► Database
              ✓          ✓           ✓          ✗
         (exposé)   (interne)   (interne)  (isolé)
```

### 5. Utiliser des Network Policies

Même dans un lab, c'est une bonne pratique :

```yaml
# Isoler la base de données
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: database-isolation
spec:
  podSelector:
    matchLabels:
      app: database
  policyTypes:
  - Ingress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: backend
```

### 6. Documenter l'Architecture Réseau

Maintenir un diagramme à jour :

```
┌─────────────────────────────────────┐
│  Architecture Réseau                │
│                                     │
│  Internet                           │
│    │                                │
│    ▼                                │
│  Ingress (80/443)                   │
│    │                                │
│    ├─► Frontend Service             │
│    │     └─► Frontend Pods          │
│    │                                │
│    └─► API Service                  │
│          └─► API Pods               │
│               └─► DB Service        │
│                     └─► DB Pod      │
└─────────────────────────────────────┘
```

---

## Outils Recommandés

### 1. kubectl Plugins

```bash
# Installation de krew (gestionnaire de plugins)
(
  set -x; cd "$(mktemp -d)" &&
  OS="$(uname | tr '[:upper:]' '[:lower:]')" &&
  ARCH="$(uname -m | sed -e 's/x86_64/amd64/' -e 's/\(arm\)\(64\)\?.*/\1\2/' -e 's/aarch64$/arm64/')" &&
  KREW="krew-${OS}_${ARCH}" &&
  curl -fsSLO "https://github.com/kubernetes-sigs/krew/releases/latest/download/${KREW}.tar.gz" &&
  tar zxvf "${KREW}.tar.gz" &&
  ./"${KREW}" install krew
)

# Plugins utiles
kubectl krew install net-forward   # Port forwarding avancé
kubectl krew install netshoot       # Pod debug réseau
kubectl krew install doctor         # Diagnostic cluster
```

### 2. Outils de Diagnostic Réseau

| Outil | Description | Installation |
|-------|-------------|--------------|
| **curl** | Test HTTP/HTTPS | `apt install curl` |
| **netcat (nc)** | Test TCP/UDP | `apt install netcat` |
| **nmap** | Scanner de ports | `apt install nmap` |
| **tcpdump** | Capture paquets | `apt install tcpdump` |
| **dig** | Test DNS | `apt install dnsutils` |

### 3. Images Docker Utiles

```bash
# Pod avec tous les outils réseau
microk8s kubectl run netshoot --image=nicolaka/netshoot -it --rm

# Pod simple pour tests
microk8s kubectl run busybox --image=busybox:1.28 -it --rm

# Pod avec curl
microk8s kubectl run curlpod --image=curlimages/curl:latest -it --rm
```

---

## Checklist de Configuration Réseau

### Configuration Initiale

- [ ] Addon `dns` activé
- [ ] CoreDNS fonctionne (pods Running)
- [ ] Calico fonctionne (pods Running)
- [ ] Test de communication pod-to-pod réussi
- [ ] Test de résolution DNS réussi

### Pour Exposition Externe

- [ ] Addon `ingress` activé
- [ ] Ingress Controller fonctionne
- [ ] Nom de domaine configuré (ou DynamicDNS)
- [ ] DNS externe pointe vers votre IP
- [ ] Port forwarding configuré sur le routeur
- [ ] Firewall autorise ports 80/443
- [ ] Test depuis Internet réussi

### Pour Production

- [ ] Network Policies définies
- [ ] Isolation entre environnements
- [ ] Certificats SSL configurés
- [ ] Monitoring réseau en place
- [ ] Documentation à jour
- [ ] Plan de reprise testé

---

## Ressources et Documentation

### Documentation Officielle

- **Kubernetes Networking** : https://kubernetes.io/docs/concepts/services-networking/
- **Calico Documentation** : https://docs.tigera.io/calico/latest/about/
- **NGINX Ingress** : https://kubernetes.github.io/ingress-nginx/

### Tutoriels Recommandés

- **Kubernetes Networking Demystified** : https://www.youtube.com/watch?v=0Omvgd7Hg1I
- **Life of a Packet in Kubernetes** : https://www.youtube.com/watch?v=0Omvgd7Hg1I

### Livres

- **Kubernetes Up & Running** - Brendan Burns et al.
- **Kubernetes Networking** - James Strong & Vallery Lancey

---

## Conclusion

Le réseau dans Kubernetes peut sembler complexe au début, mais en comprenant les concepts fondamentaux, vous serez capable de :

1. **Connecter vos applications** de manière fiable
2. **Exposer vos services** vers l'extérieur de manière sécurisée
3. **Isoler et protéger** vos composants critiques
4. **Diagnostiquer et résoudre** les problèmes réseau

Cette annexe vous accompagne dans tous ces aspects, du plus simple au plus avancé.

**Prochaines étapes recommandées :**

1. Lire la section Configuration DNS pour comprendre la résolution de noms
2. Configurer le firewall pour sécuriser votre cluster
3. Expérimenter avec les Network Policies
4. Mettre en place une architecture réseau complète pour votre lab

Bonne configuration ! 🚀

⏭️ [Configuration DNS complète](/annexes/annexe-c-configuration-reseau/configuration-dns-complete.md)
