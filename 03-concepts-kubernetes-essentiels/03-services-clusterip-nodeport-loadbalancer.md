🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 3.3 Services (ClusterIP, NodePort, LoadBalancer)

## Introduction

Dans les sections précédentes, nous avons appris à créer des Pods avec des Deployments. Maintenant, un nouveau problème se pose : **comment accéder à ces Pods de manière fiable ?**

Les **Services** sont la solution Kubernetes pour exposer vos applications et permettre la communication réseau de manière stable et prévisible.

## Le problème : Pods éphémères et IPs changeantes

### Scénario problématique

Imaginez que vous avez déployé une application web avec 3 Pods :

```
Pod 1: 10.1.5.23
Pod 2: 10.1.5.24
Pod 3: 10.1.5.25
```

**Problèmes rencontrés :**

1. **IPs changeantes** : Quand un Pod meurt et est remplacé, il obtient une nouvelle IP
   ```
   Pod 1 meurt → Pod 4 créé avec IP 10.1.5.99
   ```

2. **Quelle IP utiliser ?** : Si vous avez 3 Pods, comment savoir vers lequel envoyer les requêtes ?

3. **Load balancing** : Comment répartir le trafic équitablement entre les 3 Pods ?

4. **Découverte de service** : Comment les autres applications trouvent-elles votre service ?

### Exemple concret du problème

Vous avez :
- Un **frontend** (application web) avec 3 Pods
- Un **backend** (API) qui doit appeler le frontend

**Sans Service** :
```yaml
# Dans le backend, vous devriez faire quelque chose comme :
http://10.1.5.23/api  # Mais cette IP peut changer !
```

Si le Pod avec l'IP `10.1.5.23` meurt, votre backend ne peut plus accéder au frontend. C'est ingérable !

## Qu'est-ce qu'un Service ?

### Définition simple

Un **Service** est une abstraction qui définit un ensemble logique de Pods et une politique pour y accéder. Il fournit une **IP stable** et un **nom DNS** pour accéder à vos Pods.

### Analogie : le standard téléphonique

Imaginez un **standard téléphonique** d'une entreprise :
- Vous appelez un **numéro unique** (le Service)
- Le standard **redirige** votre appel vers un employé disponible (un Pod)
- Si un employé quitte l'entreprise (Pod qui meurt), le standard redirige vers un autre
- Vous n'avez **pas besoin de connaître** le numéro direct de chaque employé (IP des Pods)

Le Service fonctionne exactement de cette manière !

### Fonctionnement d'un Service

```
┌─────────────────────────────────────────────┐
│           Service "frontend"                │
│                                             │
│  IP stable: 10.96.100.50                    │
│  DNS: frontend.default.svc.cluster.local    │
│                                             │
│  Sélectionne les Pods avec: app=frontend    │
└──────────────┬──────────────────────────────┘
               │ Load balancing automatique
               │
      ┌────────┼────────┐
      │        │        │
      v        v        v
   ┌─────┐ ┌─────┐ ┌─────┐
   │Pod 1│ │Pod 2│ │Pod 3│
   │10.1 │ │10.1 │ │10.1 │
   │.5.23│ │.5.24│ │.5.25│
   └─────┘ └─────┘ └─────┘
```

**Avantages** :
- **IP stable** : Le Service garde toujours la même IP
- **Nom DNS** : Vous pouvez utiliser un nom plutôt qu'une IP
- **Load balancing** : Le trafic est réparti automatiquement
- **Service discovery** : Les applications se trouvent facilement

## Les trois types de Services principaux

Kubernetes propose trois types de Services pour différents cas d'usage :

1. **ClusterIP** (par défaut) : Accessible uniquement depuis l'intérieur du cluster
2. **NodePort** : Accessible depuis l'extérieur via un port sur chaque nœud
3. **LoadBalancer** : Accessible depuis l'extérieur via un load balancer externe

Commençons par le plus simple : ClusterIP.

## 1. Service ClusterIP

### Description

**ClusterIP** est le type de Service **par défaut**. Il expose votre application uniquement à l'intérieur du cluster Kubernetes. Les Pods peuvent communiquer entre eux via ce Service, mais il n'est **pas accessible depuis l'extérieur**.

### Cas d'usage

- Communication entre microservices internes
- Bases de données accessibles uniquement par les applications internes
- APIs internes
- Services backend qui ne doivent pas être exposés publiquement

### Schéma de fonctionnement

```
┌───────────────────────────────────────────────────────┐
│              Cluster Kubernetes                       │
│                                                       │
│  ┌──────────────────┐                                 │
│  │  Service         │                                 │
│  │  Type: ClusterIP │                                 │
│  │  IP: 10.96.1.50  │                                 │
│  └────────┬─────────┘                                 │
│           │ Accessible uniquement dans le cluster     │
│           │                                           │
│      ┌────┼────┐                                      │
│      v    v    v                                      │
│   ┌────┐┌────┐┌────┐                                  │
│   │Pod1││Pod2││Pod3│                                  │
│   └────┘└────┘└────┘                                  │
│                                                       │
└───────────────────────────────────────────────────────┘
         ↑
         │ ❌ Pas accessible depuis l'extérieur
         │
    Internet/Utilisateurs externes
```

### Manifeste YAML d'un Service ClusterIP

```yaml
apiVersion: v1
kind: Service
metadata:
  name: mon-service-backend
  labels:
    app: backend
spec:
  type: ClusterIP              # Type de Service (peut être omis car c'est le défaut)
  selector:                    # Sélectionne les Pods à exposer
    app: backend
    tier: api
  ports:
  - name: http                 # Nom du port (optionnel mais recommandé)
    protocol: TCP              # Protocole (TCP par défaut)
    port: 80                   # Port du Service
    targetPort: 8080           # Port sur le conteneur du Pod
```

### Explication détaillée

- **type: ClusterIP** : Définit le type de Service (peut être omis car c'est la valeur par défaut)

- **selector** :
  - Définit quels Pods font partie de ce Service
  - Tous les Pods avec les labels `app: backend` et `tier: api` seront exposés par ce Service
  - Les labels doivent correspondre à ceux définis dans votre Deployment

- **ports** :
  - **port: 80** : Port sur lequel le Service écoute (les clients se connectent sur ce port)
  - **targetPort: 8080** : Port sur lequel les Pods écoutent réellement
  - **protocol: TCP** : Protocole utilisé (TCP ou UDP)

### Schéma de redirection des ports

```
Client interne         Service              Pod
     │                   │                   │
     │  GET :80          │                   │
     ├──────────────────>│                   │
     │                   │  GET :8080        │
     │                   ├──────────────────>│
     │                   │                   │
     │                   │  Response         │
     │                   │<──────────────────┤
     │  Response         │                   │
     │<──────────────────┤                   │
```

### Accès au Service depuis un Pod

Une fois le Service créé, il est accessible via :

**1. Par IP** :
```bash
curl http://10.96.1.50:80
```

**2. Par nom DNS (recommandé)** :
```bash
# Format: <nom-service>.<namespace>.svc.cluster.local
curl http://mon-service-backend.default.svc.cluster.local:80

# Ou plus simplement (dans le même namespace) :
curl http://mon-service-backend:80
```

### Exemple complet : Frontend et Backend

**Backend Deployment + Service** :

```yaml
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: backend
spec:
  replicas: 3
  selector:
    matchLabels:
      app: backend
  template:
    metadata:
      labels:
        app: backend
    spec:
      containers:
      - name: api
        image: mon-api:v1
        ports:
        - containerPort: 8080
---
apiVersion: v1
kind: Service
metadata:
  name: backend-service
spec:
  type: ClusterIP
  selector:
    app: backend
  ports:
  - port: 80
    targetPort: 8080
```

**Frontend qui accède au Backend** :

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: frontend
spec:
  replicas: 2
  selector:
    matchLabels:
      app: frontend
  template:
    metadata:
      labels:
        app: frontend
    spec:
      containers:
      - name: web
        image: mon-frontend:v1
        env:
        - name: BACKEND_URL
          value: "http://backend-service:80"    # Utilise le nom DNS du Service
```

Le frontend peut maintenant appeler le backend via `http://backend-service:80` sans se soucier des IPs des Pods !

## 2. Service NodePort

### Description

**NodePort** expose votre Service sur un port statique de **chaque nœud** du cluster. Cela rend votre Service accessible depuis l'extérieur du cluster en utilisant `<IP-du-noeud>:<NodePort>`.

### Cas d'usage

- Exposer une application pour des tests depuis votre machine locale
- Environnements de développement
- Clusters simples sans load balancer externe
- Accès temporaire depuis l'extérieur

### Schéma de fonctionnement

```
┌────────────────────────────────────────────────────────┐
│              Cluster Kubernetes                        │
│                                                        │
│  Nœud 1 (192.168.1.10)      Nœud 2 (192.168.1.11)      │
│    :30080 ────┐                :30080 ────┐            │
│               │                           │            │
│               v                           v            │
│         ┌──────────────────────────────────┐           │
│         │  Service (NodePort)              │           │
│         │  ClusterIP: 10.96.1.50           │           │
│         │  NodePort: 30080                 │           │
│         └────────────┬─────────────────────┘           │
│                      │                                 │
│                 ┌────┼────┐                            │
│                 v    v    v                            │
│              ┌────┐┌────┐┌────┐                        │
│              │Pod1││Pod2││Pod3│                        │
│              └────┘└────┘└────┘                        │
│                                                        │
└────────────────────────────────────────────────────────┘
              ↑                      ↑
              │                      │
              └──────────────────────┘
           Accessible depuis l'extérieur via :
           http://192.168.1.10:30080 ou
           http://192.168.1.11:30080
```

### Manifeste YAML d'un Service NodePort

```yaml
apiVersion: v1
kind: Service
metadata:
  name: mon-service-web
spec:
  type: NodePort              # Type de Service : NodePort
  selector:
    app: web
  ports:
  - name: http
    protocol: TCP
    port: 80                  # Port du Service (ClusterIP)
    targetPort: 8080          # Port sur le Pod
    nodePort: 30080           # Port exposé sur chaque nœud (30000-32767)
```

### Explication des ports

Un Service NodePort utilise **trois ports** :

1. **targetPort: 8080** : Port sur lequel le conteneur écoute dans le Pod
2. **port: 80** : Port du Service (ClusterIP) accessible depuis l'intérieur du cluster
3. **nodePort: 30080** : Port exposé sur chaque nœud du cluster pour l'accès externe

### Schéma des trois niveaux de ports

```
Externe         Nœud          Service         Pod
   │              │              │              │
   │ :30080       │              │              │
   ├─────────────>│              │              │
   │              │ :80          │              │
   │              ├─────────────>│              │
   │              │              │ :8080        │
   │              │              ├─────────────>│
```

### Plage de ports NodePort

Les ports NodePort doivent être dans la plage **30000-32767** par défaut.

**Spécifier un port spécifique** :
```yaml
nodePort: 30080    # Port choisi manuellement
```

**Laisser Kubernetes choisir automatiquement** :
```yaml
# Omettez simplement nodePort, Kubernetes en attribuera un
ports:
- port: 80
  targetPort: 8080
  # nodePort sera attribué automatiquement
```

### Accès au Service NodePort

**Depuis l'intérieur du cluster** :
```bash
curl http://mon-service-web:80
```

**Depuis l'extérieur du cluster** :
```bash
# Via l'IP de n'importe quel nœud du cluster
curl http://192.168.1.10:30080
curl http://192.168.1.11:30080    # Même résultat
```

### Exemple pratique : Application web simple

```yaml
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: webapp
spec:
  replicas: 3
  selector:
    matchLabels:
      app: webapp
  template:
    metadata:
      labels:
        app: webapp
    spec:
      containers:
      - name: nginx
        image: nginx:1.25
        ports:
        - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: webapp-service
spec:
  type: NodePort
  selector:
    app: webapp
  ports:
  - port: 80
    targetPort: 80
    nodePort: 30100
```

Après déploiement, vous pouvez accéder à votre application via :
```bash
# Depuis votre navigateur ou terminal
http://<IP-de-votre-noeud>:30100
```

### Limitations de NodePort

1. **Plage de ports limitée** : 30000-32767 seulement
2. **Ports peu intuitifs** : Les utilisateurs doivent connaître le port spécifique
3. **Un port par Service** : Peut vite devenir compliqué avec beaucoup de services
4. **Pas de vrai load balancing externe** : Vous devez gérer vous-même la répartition entre nœuds
5. **Exposition directe des nœuds** : Problèmes de sécurité potentiels

## 3. Service LoadBalancer

### Description

**LoadBalancer** provisionne un load balancer externe (fourni par le cloud provider ou une solution comme MetalLB) qui expose votre Service avec une IP externe dédiée.

### Cas d'usage

- Applications production accessibles depuis Internet
- Exposition publique d'APIs
- Applications web grand public
- Tout service nécessitant une IP externe stable

### Prérequis

Pour utiliser un Service LoadBalancer avec MicroK8s, vous devez activer **MetalLB** :

```bash
microk8s enable metallb
```

Vous devrez configurer une plage d'IPs que MetalLB peut utiliser (voir le chapitre 9 pour plus de détails).

### Schéma de fonctionnement

```
                    Internet
                       │
                       │ Requêtes HTTP
                       │
                       v
┌──────────────────────────────────────────────────────┐
│            Load Balancer Externe                     │
│            IP Publique: 203.0.113.50                 │
└─────────────────────┬────────────────────────────────┘
                      │
┌─────────────────────┼────────────────────────────────┐
│   Cluster Kubernetes│                                │
│                     v                                │
│              ┌────────────────┐                      │
│              │  Service       │                      │
│              │  Type: LB      │                      │
│              │  ClusterIP:    │                      │
│              │  10.96.1.50    │                      │
│              └────────┬───────┘                      │
│                       │                              │
│                  ┌────┼────┐                         │
│                  v    v    v                         │
│               ┌────┐┌────┐┌────┐                     │
│               │Pod1││Pod2││Pod3│                     │
│               └────┘└────┘└────┘                     │
│                                                      │
└──────────────────────────────────────────────────────┘
```

### Manifeste YAML d'un Service LoadBalancer

```yaml
apiVersion: v1
kind: Service
metadata:
  name: mon-service-public
spec:
  type: LoadBalancer          # Type de Service : LoadBalancer
  selector:
    app: web
  ports:
  - name: http
    protocol: TCP
    port: 80                  # Port exposé par le load balancer
    targetPort: 8080          # Port sur le Pod
  # Optionnel : spécifier une IP externe spécifique (si supporté)
  # loadBalancerIP: 203.0.113.50
```

### Ce qui se passe lors de la création

1. **Kubernetes crée le Service** avec le type LoadBalancer
2. **Un load balancer externe est provisionné** (par MetalLB dans notre cas)
3. **Une IP externe est attribuée** au Service
4. **Le trafic est routé** depuis l'IP externe vers les Pods

### Vérifier l'IP externe attribuée

```bash
microk8s kubectl get service mon-service-public
```

Sortie exemple :
```
NAME                 TYPE           CLUSTER-IP    EXTERNAL-IP     PORT(S)        AGE
mon-service-public   LoadBalancer   10.96.1.50    192.168.1.100   80:31234/TCP   2m
```

L'**EXTERNAL-IP** `192.168.1.100` est l'adresse par laquelle votre service est accessible depuis l'extérieur.

### Accès au Service LoadBalancer

**Depuis l'extérieur** :
```bash
curl http://192.168.1.100
```

**Depuis l'intérieur du cluster** :
```bash
curl http://mon-service-public:80
```

### Exemple complet : Application web publique

```yaml
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: site-web
spec:
  replicas: 4
  selector:
    matchLabels:
      app: site-web
  template:
    metadata:
      labels:
        app: site-web
    spec:
      containers:
      - name: nginx
        image: nginx:1.25
        ports:
        - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: site-web-public
spec:
  type: LoadBalancer
  selector:
    app: site-web
  ports:
  - name: http
    port: 80
    targetPort: 80
  - name: https
    port: 443
    targetPort: 443
```

Ce Service expose votre site web sur les ports 80 (HTTP) et 443 (HTTPS) via une IP externe.

### LoadBalancer vs NodePort

Le Service LoadBalancer **inclut** toutes les fonctionnalités de NodePort :
- Il crée également un NodePort automatiquement
- Le load balancer externe redirige vers ce NodePort
- Vous pouvez toujours accéder via `<IP-noeud>:<NodePort>` si besoin

```
Hiérarchie des types de Services :
LoadBalancer ⊃ NodePort ⊃ ClusterIP

LoadBalancer = ClusterIP + NodePort + Load Balancer externe
NodePort = ClusterIP + Port sur les nœuds
ClusterIP = Service de base
```

### Avec MetalLB (MicroK8s)

MetalLB fournit une implémentation de LoadBalancer pour les clusters "bare metal" (sans cloud provider) :

```bash
# Activer MetalLB
microk8s enable metallb

# Configurer la plage d'IPs (exemple)
# Répondez avec une plage d'IPs disponibles sur votre réseau
# Par exemple : 192.168.1.200-192.168.1.250
```

Ensuite, tous vos Services de type LoadBalancer recevront automatiquement une IP de cette plage.

## Comparaison des trois types de Services

| Caractéristique | ClusterIP | NodePort | LoadBalancer |
|----------------|-----------|----------|--------------|
| **Accessible depuis l'intérieur du cluster** | ✓ | ✓ | ✓ |
| **Accessible depuis l'extérieur** | ✗ | ✓ (via NodePort) | ✓ (via IP externe) |
| **IP stable interne** | ✓ | ✓ | ✓ |
| **IP externe dédiée** | ✗ | ✗ | ✓ |
| **Port exposé** | Port standard | 30000-32767 | Port standard (80, 443, etc.) |
| **Load balancing externe** | ✗ | ✗ | ✓ |
| **Cas d'usage typique** | Communication interne | Dev/Test | Production publique |
| **Complexité** | Simple | Moyenne | Élevée (nécessite LB externe) |

## DNS et découverte de services

### Résolution DNS automatique

Kubernetes crée automatiquement des entrées DNS pour chaque Service :

**Format complet** :
```
<nom-service>.<namespace>.svc.cluster.local
```

**Exemples** :
```bash
# Service "backend" dans le namespace "default"
backend.default.svc.cluster.local

# Service "database" dans le namespace "production"
database.production.svc.cluster.local
```

### Raccourcis DNS

**Depuis le même namespace** :
```bash
# Simplement le nom du service
curl http://backend

# Ou avec le namespace
curl http://backend.default
```

**Depuis un autre namespace** :
```bash
# Doit inclure le namespace
curl http://backend.production
```

### Exemple pratique

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: frontend
spec:
  template:
    spec:
      containers:
      - name: web
        image: mon-frontend:v1
        env:
        - name: API_URL
          value: "http://backend:8080/api"    # Utilise le nom DNS du Service
        - name: DB_HOST
          value: "postgres.database:5432"     # Service dans un autre namespace
```

## Endpoints : les coulisses du Service

### Qu'est-ce qu'un Endpoint ?

Un **Endpoint** est un objet Kubernetes qui liste les IPs des Pods sélectionnés par un Service.

Quand vous créez un Service, Kubernetes crée automatiquement un objet Endpoint correspondant.

### Voir les Endpoints

```bash
microk8s kubectl get endpoints mon-service-backend
```

Sortie exemple :
```
NAME                 ENDPOINTS                                      AGE
mon-service-backend  10.1.5.23:8080,10.1.5.24:8080,10.1.5.25:8080  5m
```

Cela montre les IPs et ports des trois Pods backend accessibles via ce Service.

### Schéma : Service → Endpoints → Pods

```
┌──────────────┐
│   Service    │
│ backend:80   │
└──────┬───────┘
       │ sélectionne les Pods avec app=backend
       v
┌──────────────────┐
│   Endpoints      │
│  10.1.5.23:8080  │
│  10.1.5.24:8080  │  ← Liste mise à jour automatiquement
│  10.1.5.25:8080  │
└──────┬───────────┘
       │
       v
   Pods réels
```

### Debugging : vérifier les Endpoints

Si votre Service ne fonctionne pas, vérifiez les Endpoints :

```bash
microk8s kubectl describe endpoints mon-service-backend
```

**Problème courant** : Endpoints vide
```
Endpoints:  <none>
```

**Cause** : Les labels du Service ne correspondent à aucun Pod. Vérifiez :
- Les labels dans `selector` du Service
- Les labels dans `template.metadata.labels` du Deployment

## Sessions Affinity : clients persistants

### Problème

Par défaut, Kubernetes répartit les requêtes de manière aléatoire entre les Pods. Cela peut poser problème pour :
- Sessions utilisateur (cookies)
- WebSockets
- Applications stateful

### Solution : Session Affinity

```yaml
apiVersion: v1
kind: Service
metadata:
  name: mon-service
spec:
  type: ClusterIP
  selector:
    app: web
  sessionAffinity: ClientIP         # Active la session affinity
  sessionAffinityConfig:
    clientIP:
      timeoutSeconds: 10800         # Durée de la persistance (3 heures)
  ports:
  - port: 80
    targetPort: 8080
```

**Comportement** :
- Les requêtes provenant de la même IP client seront toujours dirigées vers le même Pod
- Valable pendant la durée définie dans `timeoutSeconds`

### Cas d'usage

- Applications web avec sessions
- Applications nécessitant une connexion persistante
- Caches locaux dans les Pods

## Services sans sélecteur

### Cas d'usage

Parfois, vous voulez créer un Service qui pointe vers :
- Une base de données externe (hors cluster)
- Un service dans un autre cluster
- Un service en cours de migration

### Service sans sélecteur

```yaml
apiVersion: v1
kind: Service
metadata:
  name: base-donnees-externe
spec:
  type: ClusterIP
  ports:
  - port: 3306
    targetPort: 3306
  # Pas de selector !
```

### Endpoints manuels

Vous devez créer manuellement l'objet Endpoints :

```yaml
apiVersion: v1
kind: Endpoints
metadata:
  name: base-donnees-externe      # Même nom que le Service
subsets:
- addresses:
  - ip: 192.168.50.10             # IP de la base externe
  ports:
  - port: 3306
```

Maintenant, les Pods peuvent accéder à cette base externe via :
```bash
mysql -h base-donnees-externe -P 3306
```

## Bonnes pratiques

### 1. Choisir le bon type de Service

**Pour la communication interne** :
```yaml
type: ClusterIP    # Toujours préférer ClusterIP pour l'interne
```

**Pour le développement/test** :
```yaml
type: NodePort     # Pratique pour des tests rapides
```

**Pour la production publique** :
```yaml
type: LoadBalancer # Avec MetalLB ou un cloud provider
```

### 2. Utiliser des noms de ports

```yaml
ports:
- name: http        # Nomme vos ports
  port: 80
  targetPort: 8080
- name: https
  port: 443
  targetPort: 8443
```

**Avantage** : Clarté et possibilité de référencer par nom plutôt que par numéro.

### 3. Utiliser le DNS, pas les IPs

**❌ Mauvais** :
```yaml
env:
- name: BACKEND_URL
  value: "http://10.96.1.50:80"
```

**✅ Bon** :
```yaml
env:
- name: BACKEND_URL
  value: "http://backend-service:80"
```

### 4. Grouper Services et Deployments dans le même fichier

```yaml
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mon-app
# ...
---
apiVersion: v1
kind: Service
metadata:
  name: mon-app-service
# ...
```

Utilisez `---` pour séparer les ressources dans un même fichier YAML.

### 5. Vérifier la correspondance des labels

Les labels dans le `selector` du Service DOIVENT correspondre aux labels des Pods :

```yaml
# Deployment
template:
  metadata:
    labels:
      app: backend      # ← Ces labels
      version: v1

# Service
selector:
  app: backend          # ← Doivent correspondre
```

### 6. Exposer plusieurs ports si nécessaire

```yaml
ports:
- name: http
  port: 80
  targetPort: 8080
- name: metrics
  port: 9090
  targetPort: 9090
```

### 7. Utiliser des annotations pour la documentation

```yaml
metadata:
  name: mon-service
  annotations:
    description: "Service backend pour l'API REST"
    owner: "equipe-backend@example.com"
    version: "v1.2.0"
```

## Dépannage des Services

### Problème 1 : Service créé mais pas accessible

**Vérifications** :

```bash
# 1. Vérifier que le Service existe
microk8s kubectl get service mon-service

# 2. Vérifier les Endpoints
microk8s kubectl get endpoints mon-service

# 3. Vérifier les Pods
microk8s kubectl get pods -l app=backend

# 4. Voir les détails du Service
microk8s kubectl describe service mon-service
```

**Causes courantes** :
- Labels ne correspondent pas (Service selector ≠ Pod labels)
- Aucun Pod en état "Ready"
- Mauvais port targetPort

### Problème 2 : Endpoints vide

```bash
microk8s kubectl describe endpoints mon-service
```

Si `Endpoints: <none>`, vérifiez :

```bash
# Labels du Service
microk8s kubectl get service mon-service -o yaml | grep -A 5 selector

# Labels des Pods
microk8s kubectl get pods --show-labels
```

**Solution** : Corriger les labels pour qu'ils correspondent.

### Problème 3 : Service LoadBalancer bloqué en "Pending"

```bash
microk8s kubectl get service
```

```
NAME         TYPE           EXTERNAL-IP   PORT(S)
mon-service  LoadBalancer   <pending>     80:30123/TCP
```

**Causes** :
- MetalLB n'est pas activé
- Pas d'IPs disponibles dans le pool MetalLB
- Problème de configuration MetalLB

**Solution** :
```bash
# Vérifier si MetalLB est activé
microk8s status

# Activer MetalLB si nécessaire
microk8s enable metallb
```

### Problème 4 : Connexion refuse (Connection refused)

**Test de connectivité** :

```bash
# Depuis un Pod de test
microk8s kubectl run test-pod --image=busybox --rm -it -- sh
/ # wget -O- http://mon-service:80
```

Si "Connection refused" :
- Vérifier que les Pods écoutent sur le bon port (targetPort)
- Vérifier les readiness probes
- Vérifier les logs des Pods

### Problème 5 : Timeout

Si la connexion timeout (pas de réponse) :
- Vérifier les Network Policies (si activées)
- Vérifier le firewall des nœuds
- Vérifier que le port est bien exposé dans le conteneur

## Architecture multi-tiers complète

Voici un exemple d'architecture complète avec les trois types de Services :

```yaml
# ===== FRONTEND (accessible publiquement) =====
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: frontend
spec:
  replicas: 3
  selector:
    matchLabels:
      app: frontend
      tier: web
  template:
    metadata:
      labels:
        app: frontend
        tier: web
    spec:
      containers:
      - name: nginx
        image: nginx:1.25
        ports:
        - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: frontend-service
spec:
  type: LoadBalancer         # Accessible depuis Internet
  selector:
    app: frontend
    tier: web
  ports:
  - port: 80
    targetPort: 80

# ===== BACKEND API (accessible uniquement en interne) =====
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: backend
spec:
  replicas: 3
  selector:
    matchLabels:
      app: backend
      tier: api
  template:
    metadata:
      labels:
        app: backend
        tier: api
    spec:
      containers:
      - name: api
        image: mon-api:v1
        ports:
        - containerPort: 8080
        env:
        - name: DATABASE_HOST
          value: "database-service:5432"
---
apiVersion: v1
kind: Service
metadata:
  name: backend-service
spec:
  type: ClusterIP            # Accessible uniquement en interne
  selector:
    app: backend
    tier: api
  ports:
  - port: 80
    targetPort: 8080

# ===== DATABASE (accessible uniquement en interne) =====
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: database
spec:
  replicas: 1
  selector:
    matchLabels:
      app: database
      tier: data
  template:
    metadata:
      labels:
        app: database
        tier: data
    spec:
      containers:
      - name: postgres
        image: postgres:15
        ports:
        - containerPort: 5432
---
apiVersion: v1
kind: Service
metadata:
  name: database-service
spec:
  type: ClusterIP            # Accessible uniquement en interne
  selector:
    app: database
    tier: data
  ports:
  - port: 5432
    targetPort: 5432
```

**Architecture résultante** :

```
Internet
   │
   v
┌─────────────────────┐
│ LoadBalancer        │ ← IP publique
│ (frontend-service)  │
└──────────┬──────────┘
           │
┌──────────┼───────────────────────────────────┐
│ Cluster  v                                   │
│    ┌──────────┐                              │
│    │ Frontend │                              │
│    │ Pods     │                              │
│    └────┬─────┘                              │
│         │ appelle backend-service:80         │
│         v                                    │
│    ┌──────────┐                              │
│    │ Backend  │                              │
│    │ Pods     │                              │
│    └────┬─────┘                              │
│         │ appelle database-service:5432      │
│         v                                    │
│    ┌──────────┐                              │
│    │ Database │                              │
│    │ Pod      │                              │
│    └──────────┘                              │
│                                              │
└──────────────────────────────────────────────┘
```

## Résumé des points clés

- Les **Services** fournissent une IP stable et un nom DNS pour accéder aux Pods
- **ClusterIP** (défaut) : accessible uniquement depuis l'intérieur du cluster
- **NodePort** : accessible depuis l'extérieur via `<IP-noeud>:<NodePort>`
- **LoadBalancer** : accessible depuis l'extérieur via une IP externe dédiée
- Les Services utilisent les **labels** pour sélectionner les Pods à exposer
- Le **DNS** automatique permet d'utiliser des noms plutôt que des IPs
- Les **Endpoints** listent les IPs réelles des Pods derrière un Service
- **Session affinity** permet de maintenir un client sur le même Pod
- Pour la production, utilisez **LoadBalancer** avec MetalLB sur MicroK8s

## Prochaines étapes

Maintenant que vous maîtrisez les Services, vous êtes prêt à découvrir :
- **Les Namespaces** : comment organiser et isoler vos ressources
- **Les ConfigMaps** : comment gérer la configuration de vos applications
- **Les Secrets** : comment stocker et utiliser des données sensibles

Les Services sont essentiels pour toute architecture Kubernetes. Ils permettent de créer des applications modulaires, scalables et résilientes !

⏭️ [Namespaces](/03-concepts-kubernetes-essentiels/04-namespaces.md)
