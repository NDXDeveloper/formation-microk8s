🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 21.6 Load Balancing entre Nodes

## Introduction

Maintenant que vous avez un cluster multi-node avec plusieurs workers, une question essentielle se pose : **comment répartir le trafic entre tous ces nœuds** ? Comment faire en sorte que les utilisateurs accèdent à vos applications de manière optimale, peu importe sur quel nœud les pods tournent ?

C'est là qu'intervient le **load balancing** (répartition de charge). Dans ce chapitre, nous allons découvrir :
- Les différents types de Services Kubernetes
- Comment fonctionne le load balancing natif
- MetalLB : la solution pour les clusters bare-metal
- Comment distribuer le trafic équitablement entre vos nœuds
- Les bonnes pratiques de configuration

Le load balancing est crucial pour :
- **Performance** : Répartir la charge entre plusieurs serveurs
- **Disponibilité** : Si un nœud tombe, le trafic va vers les autres
- **Scalabilité** : Ajouter de la capacité sans changer les points d'entrée
- **Simplicité** : Une seule adresse IP pour accéder à l'application

## Qu'est-ce que le Load Balancing ?

### Définition Simple

Le **load balancing** (équilibrage de charge) consiste à **distribuer les requêtes** entre plusieurs serveurs pour éviter qu'un seul soit surchargé.

**Analogie du restaurant :**
Imaginez un restaurant avec plusieurs serveurs :
- **Sans load balancer** : Tous les clients vont vers le même serveur → il est débordé, les autres s'ennuient
- **Avec load balancer** : Un maître d'hôtel (load balancer) répartit les clients entre tous les serveurs → charge équilibrée

**Dans Kubernetes :**
```
                     ┌─────────────┐
                     │   Internet  │
                     └──────┬──────┘
                            │
                     ┌──────▼──────┐
                     │    Load     │
                     │   Balancer  │  ← Répartit le trafic
                     └──────┬──────┘
                            │
        ┌───────────────────┼───────────────────┐
        │                   │                   │
   ┌────▼────┐         ┌────▼────┐         ┌────▼────┐
   │ Node 1  │         │ Node 2  │         │ Node 3  │
   │ Pod A   │         │ Pod B   │         │ Pod C   │
   └─────────┘         └─────────┘         └─────────┘
```

Le load balancer reçoit une requête et la transmet à l'un des pods, peu importe sur quel nœud il se trouve.

### Les Problèmes à Résoudre

**Sans load balancing dans un cluster multi-node :**

1. **Point d'entrée multiple**
   - Chaque nœud a sa propre IP
   - L'utilisateur doit connaître toutes les IPs
   - Si un nœud tombe, les requêtes vers cette IP échouent

2. **Distribution inégale**
   - Un nœud peut recevoir 80% du trafic
   - Les autres nœuds sont sous-utilisés
   - Gaspillage de ressources

3. **Complexité pour l'utilisateur**
   - Quelle IP utiliser ?
   - Comment gérer les pannes ?
   - Comment ajouter/retirer des nœuds ?

**Avec load balancing :**
- Une seule IP virtuelle (VIP) pour accéder à l'application
- Trafic automatiquement distribué
- Résilience automatique (si un nœud tombe)
- Ajout/retrait de nœuds transparent

## Types de Services Kubernetes

Kubernetes propose plusieurs types de **Services** pour exposer vos applications. Comprendre ces types est essentiel pour bien configurer le load balancing.

### 1. ClusterIP (Par Défaut)

**Caractéristiques :**
- IP virtuelle **interne** au cluster uniquement
- Accessible seulement depuis l'intérieur du cluster
- Parfait pour la communication entre microservices

**Exemple :**
```yaml
apiVersion: v1
kind: Service
metadata:
  name: backend-service
spec:
  type: ClusterIP
  selector:
    app: backend
  ports:
  - port: 8080
    targetPort: 8080
```

**Schéma :**
```
┌────────────────────────────────────────────┐
│            CLUSTER                         │
│                                            │
│  ┌────────────┐      ┌────────────┐        │
│  │  Frontend  │─────►│  Backend   │        │
│  │  Pod       │      │  Service   │        │
│  └────────────┘      └──────┬─────┘        │
│                             │              │
│                    ┌────────┼────────┐     │
│                    │        │        │     │
│               ┌────▼──┐ ┌──▼────┐ ┌─▼───┐  │
│               │ Pod 1 │ │ Pod 2 │ │Pod 3│  │
│               └───────┘ └───────┘ └─────┘  │
└────────────────────────────────────────────┘
```

**Usage :**
- Bases de données internes
- APIs internes
- Microservices qui ne doivent pas être exposés à l'extérieur

**Limitation :** Pas accessible depuis l'extérieur du cluster.

### 2. NodePort

**Caractéristiques :**
- Ouvre un **port** sur chaque nœud du cluster
- Le port est le même sur tous les nœuds
- Accessible via `<NodeIP>:<NodePort>`

**Exemple :**
```yaml
apiVersion: v1
kind: Service
metadata:
  name: web-service
spec:
  type: NodePort
  selector:
    app: web
  ports:
  - port: 80
    targetPort: 8080
    nodePort: 30080  # Port entre 30000-32767
```

**Schéma :**
```
┌───────────────────────────────────────────────────────┐
│                     CLUSTER                           │
│                                                       │
│  ┌─────────────┐   ┌─────────────┐   ┌─────────────┐  │
│  │   Node 1    │   │   Node 2    │   │   Node 3    │  │
│  │ :30080      │   │ :30080      │   │ :30080      │  │
│  └──────┬──────┘   └──────┬──────┘   └──────┬──────┘  │
│         │                 │                 │         │
│         └─────────────────┼─────────────────┘         │
│                           │                           │
│                  ┌────────▼────────┐                  │
│                  │   Web Service   │                  │
│                  └────────┬────────┘                  │
│                           │                           │
│                  ┌────────┼────────┐                  │
│                  │        │        │                  │
│             ┌────▼──┐ ┌───▼───┐ ┌──▼──┐               │
│             │ Pod 1 │ │ Pod 2 │ │Pod 3│               │
│             └───────┘ └───────┘ └─────┘               │
└───────────────────────────────────────────────────────┘

Internet → http://192.168.1.10:30080 → Web Service → Pods
Internet → http://192.168.1.11:30080 → Web Service → Pods
Internet → http://192.168.1.12:30080 → Web Service → Pods
```

**Usage :**
- Accès externe simple (dev, test)
- Pas besoin de load balancer externe
- Debugging

**Limitations :**
- Port dans la plage 30000-32767 (pas standard)
- Pas de vraie répartition de charge (utilisateur doit choisir une IP)
- Pas professionnel pour la production

### 3. LoadBalancer (La Solution Idéale)

**Caractéristiques :**
- Crée automatiquement un **load balancer externe**
- Une **IP unique** pour accéder au service
- Répartition automatique du trafic entre les nœuds
- Haute disponibilité intégrée

**Exemple :**
```yaml
apiVersion: v1
kind: Service
metadata:
  name: web-service
spec:
  type: LoadBalancer
  selector:
    app: web
  ports:
  - port: 80
    targetPort: 8080
```

**Schéma :**
```
┌──────────────────────────────────────────────────────┐
│                    Internet                          │
└───────────────────────┬──────────────────────────────┘
                        │
                 ┌──────▼────────┐
                 │ Load Balancer │
                 │ 192.168.1.100 │  ← IP Virtuelle (VIP)
                 └──────┬────────┘
                        │
        ┌───────────────┼───────────────┐
        │               │               │
   ┌────▼────┐     ┌────▼────┐     ┌────▼────┐
   │ Node 1  │     │ Node 2  │     │ Node 3  │
   │ Pod A   │     │ Pod B   │     │ Pod C   │
   └─────────┘     └─────────┘     └─────────┘
```

**Avantages :**
- Une seule IP à retenir
- Distribution automatique du trafic
- Haute disponibilité (si un nœud tombe, le trafic va ailleurs)
- Port standard (80, 443, etc.)

**Le Problème :**
Dans les clouds publics (AWS, GCP, Azure), le type `LoadBalancer` crée automatiquement un load balancer cloud. Mais sur **bare-metal** (serveurs physiques, homelab), il ne se passe rien !

```bash
microk8s kubectl get svc web-service
```
```
NAME          TYPE           EXTERNAL-IP   PORT(S)
web-service   LoadBalancer   <pending>     80:30080/TCP
```

L'`EXTERNAL-IP` reste en `<pending>` indéfiniment. 😞

**Solution :** Utiliser **MetalLB** !

## MetalLB : Load Balancer pour Bare-Metal

### Qu'est-ce que MetalLB ?

**MetalLB** est un load balancer logiciel spécialement conçu pour les clusters Kubernetes **bare-metal** (non-cloud). Il émule le comportement des load balancers cloud (AWS ELB, GCP Load Balancer, etc.) sur votre propre infrastructure.

**Ce que fait MetalLB :**
1. Attribue une **IP réelle** aux Services de type `LoadBalancer`
2. **Annonce** cette IP sur le réseau (via ARP ou BGP)
3. **Distribue** le trafic entrant entre les nœuds

**Analogie :**
C'est comme avoir votre propre service postal (load balancer cloud) mais à domicile (bare-metal). MetalLB joue le rôle du facteur qui distribue le courrier (trafic) aux bonnes boîtes aux lettres (pods).

### Modes de Fonctionnement

MetalLB supporte deux modes :

#### 1. Mode Layer 2 (L2) - Recommandé pour Débuter

**Principe :**
- MetalLB répond aux requêtes ARP pour l'IP virtuelle
- Un nœud "leader" est élu pour chaque IP
- Le trafic arrive sur ce nœud, puis est distribué vers les pods

**Avantages :**
- Simple à configurer
- Fonctionne sur n'importe quel réseau
- Pas besoin de configuration spéciale sur le switch/routeur

**Inconvénients :**
- Le trafic passe toujours par le même nœud (pas de vraie distribution L2)
- Si le nœud leader tombe, élection d'un nouveau leader (quelques secondes de coupure)

**Schéma Layer 2 :**
```
┌──────────────────────────────────────────────────────┐
│                   Réseau Local                       │
└───────────────────────┬──────────────────────────────┘
                        │
                 ┌──────▼────────┐
                 │ VIP:          │
                 │ 192.168.1.100 │
                 │               │
                 │ MetalLB dit:  │
                 │ "Je suis sur  │
                 │  Node 2!"     │  ← Réponse ARP
                 └──────┬────────┘
                        │
        ┌───────────────┼────────────────┐
        │               │                │
   ┌────▼────┐     ┌────▼────┐ (Leader) ┌──────────┐
   │ Node 1  │     │ Node 2  │◄─────────│  Node 3  │
   │         │     │ ▼ Trafic│          │          │
   │ Pod A   │     │ Pod B   │          │ Pod C    │
   └─────────┘     └─────────┘          └──────────┘
```

#### 2. Mode BGP - Avancé

**Principe :**
- MetalLB annonce les IPs via le protocole BGP
- Le routeur réseau distribue le trafic entre les nœuds
- Vraie distribution L3 (couche réseau)

**Avantages :**
- Vraie répartition de charge au niveau réseau
- Haute disponibilité native
- Pas de single point of failure

**Inconvénients :**
- Nécessite un routeur compatible BGP
- Configuration plus complexe
- Overkill pour un homelab

**Pour la plupart des cas (homelab, PME), le mode Layer 2 est suffisant.**

### Installation de MetalLB avec MicroK8s

**Bonne nouvelle :** MicroK8s intègre MetalLB comme addon ! L'installation est **ultra simple**.

**Sur n'importe quel nœud du cluster :**
```bash
microk8s enable metallb
```

**Question posée :**
```
Enter each IP address range delimited by comma (e.g. '10.64.140.43-10.64.140.49,192.168.0.105-192.168.0.111'):
```

**Exemple de réponse :**
```
192.168.1.100-192.168.1.110
```

**Explication :**
- MetalLB va utiliser les IPs de **192.168.1.100** à **192.168.1.110**
- Il peut attribuer jusqu'à **11 IPs** différentes (une par Service LoadBalancer)
- Ces IPs doivent être :
  - **Dans votre réseau local** (même sous-réseau que les nœuds)
  - **Non utilisées** par d'autres machines
  - **En dehors du range DHCP** de votre routeur

**Vérifier l'installation :**
```bash
microk8s kubectl get pods -n metallb-system
```

```
NAME                          READY   STATUS    RESTARTS
controller-xxx                1/1     Running   0
speaker-yyy                   1/1     Running   0
speaker-zzz                   1/1     Running   0
```

- **controller** : Gère l'attribution des IPs aux Services
- **speaker** : Un par nœud, annonce les IPs sur le réseau (ARP)

**MetalLB est maintenant opérationnel !** 🎉

## Configuration de MetalLB

### Choisir le Bon Range d'IPs

**Étapes pour déterminer le range :**

**1. Identifier votre réseau local :**
```bash
ip addr show
```

Exemple de sortie :
```
eth0: inet 192.168.1.10/24
```

Votre réseau est `192.168.1.0/24` (de 192.168.1.1 à 192.168.1.254).

**2. Vérifier le range DHCP de votre routeur :**
Connectez-vous à l'interface web de votre routeur (souvent `192.168.1.1`).

Exemple de range DHCP :
```
DHCP Range: 192.168.1.50 - 192.168.1.200
```

**3. Choisir un range en dehors du DHCP :**
```
Range MetalLB: 192.168.1.100 - 192.168.1.110
```

**Attention :** Ce range est **dans** le range DHCP de l'exemple ci-dessus ! Il faut soit :
- **Option A** : Réduire le range DHCP dans le routeur (ex: 192.168.1.50-192.168.1.99)
- **Option B** : Utiliser un range plus bas (ex: 192.168.1.10-192.168.1.20)
- **Option C** : Utiliser un range plus haut (ex: 192.168.1.201-192.168.1.210)

**Recommandation pour un homelab :**
```
Range MetalLB: 192.168.1.200 - 192.168.1.210
```

### Exemple de Configuration Réseau Complète

```
Routeur:             192.168.1.1
Range DHCP:          192.168.1.50 - 192.168.1.149
Nœuds Kubernetes:
  - node1:           192.168.1.10 (IP statique)
  - node2:           192.168.1.11 (IP statique)
  - node3:           192.168.1.12 (IP statique)
  - worker1:         192.168.1.20 (IP statique)
  - worker2:         192.168.1.21 (IP statique)
Range MetalLB:       192.168.1.200 - 192.168.1.210
```

**Avec cette configuration :**
- Les nœuds Kubernetes ont des IPs fixes en dehors du DHCP
- MetalLB peut attribuer 11 IPs (200 à 210)
- Pas de conflit avec le DHCP

### Configuration Avancée (ConfigMap)

Si vous voulez modifier la configuration après l'installation :

```bash
microk8s kubectl get configmap -n metallb-system config -o yaml
```

**Exemple de ConfigMap MetalLB :**
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  namespace: metallb-system
  name: config
data:
  config: |
    address-pools:
    - name: default
      protocol: layer2
      addresses:
      - 192.168.1.200-192.168.1.210
```

**Modifier le range :**
```bash
microk8s kubectl edit configmap -n metallb-system config
```

Changez les adresses, sauvegardez. MetalLB applique automatiquement les changements.

### Plusieurs Pools d'Adresses

Vous pouvez définir plusieurs pools pour différents usages :

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  namespace: metallb-system
  name: config
data:
  config: |
    address-pools:
    - name: production
      protocol: layer2
      addresses:
      - 192.168.1.200-192.168.1.205
    - name: development
      protocol: layer2
      addresses:
      - 192.168.1.206-192.168.1.210
```

**Utiliser un pool spécifique dans un Service :**
```yaml
apiVersion: v1
kind: Service
metadata:
  name: web-prod
  annotations:
    metallb.universe.tf/address-pool: production
spec:
  type: LoadBalancer
  selector:
    app: web
  ports:
  - port: 80
```

## Utiliser MetalLB : Premier Service LoadBalancer

### Déployer une Application de Test

**1. Créer un Deployment :**
```yaml
# deployment-nginx.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-lb-test
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:alpine
        ports:
        - containerPort: 80
```

```bash
microk8s kubectl apply -f deployment-nginx.yaml
```

**2. Créer un Service LoadBalancer :**
```yaml
# service-nginx.yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-lb
spec:
  type: LoadBalancer
  selector:
    app: nginx
  ports:
  - port: 80
    targetPort: 80
```

```bash
microk8s kubectl apply -f service-nginx.yaml
```

**3. Vérifier l'IP attribuée par MetalLB :**
```bash
microk8s kubectl get svc nginx-lb
```

**Avant MetalLB :**
```
NAME       TYPE           EXTERNAL-IP   PORT(S)
nginx-lb   LoadBalancer   <pending>     80:30123/TCP
```

**Avec MetalLB :**
```
NAME       TYPE           EXTERNAL-IP     PORT(S)
nginx-lb   LoadBalancer   192.168.1.200   80:30123/TCP
```

**L'IP 192.168.1.200 est attribuée !** 🎉

**4. Tester l'accès :**
```bash
curl http://192.168.1.200
```

Vous devriez voir la page d'accueil de Nginx.

**Depuis n'importe quelle machine du réseau local :**
Ouvrez un navigateur et allez sur `http://192.168.1.200` → Nginx fonctionne !

## Comment Fonctionne le Load Balancing ?

### Distribution du Trafic

Avec 3 réplicas de Nginx sur 3 nœuds différents :

```bash
microk8s kubectl get pods -o wide
```

```
NAME                  NODE     STATUS
nginx-lb-test-abc     node1    Running
nginx-lb-test-def     node2    Running
nginx-lb-test-ghi     worker1  Running
```

**Quand vous faites une requête à `192.168.1.200` :**

1. **Requête ARP** : Votre machine demande "Qui a 192.168.1.200 ?"
2. **MetalLB speaker** sur le nœud leader répond : "C'est moi (node2) !"
3. **Trafic arrive sur node2**
4. **kube-proxy** sur node2 distribue vers l'un des 3 pods (round-robin par défaut)
5. Le pod traite la requête et répond

**Important :** Même si le pod est sur node1 ou worker1, le trafic peut arriver sur node2, puis être redirigé. C'est transparent.

### Algorithmes de Répartition

**Par défaut :** Kubernetes utilise **round-robin** (chacun son tour).

**Exemple de 6 requêtes :**
```
Requête 1 → Pod sur node1
Requête 2 → Pod sur node2
Requête 3 → Pod sur worker1
Requête 4 → Pod sur node1
Requête 5 → Pod sur node2
Requête 6 → Pod sur worker1
```

**Session Affinity (Sticky Sessions) :**
Si vous voulez qu'un client soit toujours redirigé vers le même pod :

```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-lb-sticky
spec:
  type: LoadBalancer
  sessionAffinity: ClientIP
  sessionAffinityConfig:
    clientIP:
      timeoutSeconds: 3600  # 1 heure
  selector:
    app: nginx
  ports:
  - port: 80
```

Maintenant, chaque client (IP source) sera toujours envoyé vers le même pod pendant 1 heure.

### Haute Disponibilité

**Scénario : Un pod crash**

```bash
# Simuler un crash
microk8s kubectl delete pod nginx-lb-test-abc
```

**Ce qui se passe :**
1. Kubernetes détecte que le pod est mort
2. Un nouveau pod est automatiquement créé (Deployment avec replicas=3)
3. Le Service redirige le trafic vers les 2 pods restants pendant la recréation
4. Une fois le nouveau pod prêt, il est ajouté à la rotation

**Downtime pour l'utilisateur : 0 seconde** (si au moins 2 réplicas fonctionnent)

**Scénario : Un nœud tombe**

```bash
# Sur node2
sudo shutdown -h now
```

**Ce qui se passe :**
1. MetalLB détecte que node2 (leader) ne répond plus
2. Un nouveau leader est élu (node1 ou worker1) en quelques secondes
3. Le nouveau leader répond aux requêtes ARP
4. Kubernetes reschedule les pods de node2 sur d'autres nœuds
5. Le Service continue de fonctionner

**Downtime : 5-10 secondes** (temps de l'élection du nouveau leader)

## Services LoadBalancer Multiples

MetalLB peut gérer **plusieurs Services LoadBalancer** avec des IPs différentes.

**Exemple avec 3 applications :**

```bash
# Application 1
microk8s kubectl create deployment app1 --image=nginx
microk8s kubectl expose deployment app1 --type=LoadBalancer --port=80

# Application 2
microk8s kubectl create deployment app2 --image=httpd
microk8s kubectl expose deployment app2 --type=LoadBalancer --port=80

# Application 3
microk8s kubectl create deployment app3 --image=caddy
microk8s kubectl expose deployment app3 --type=LoadBalancer --port=80

# Vérifier les IPs
microk8s kubectl get svc
```

**Résultat :**
```
NAME   TYPE           EXTERNAL-IP     PORT(S)
app1   LoadBalancer   192.168.1.200   80:30001/TCP
app2   LoadBalancer   192.168.1.201   80:30002/TCP
app3   LoadBalancer   192.168.1.202   80:30003/TCP
```

Chaque application a sa **propre IP** !

**Accès :**
- `http://192.168.1.200` → Nginx
- `http://192.168.1.201` → Apache
- `http://192.168.1.202` → Caddy

### Limites du Range

Si vous avez défini `192.168.1.200-192.168.1.210` (11 IPs), vous pouvez avoir jusqu'à **11 Services LoadBalancer**.

Si vous créez un 12ème Service :
```bash
microk8s kubectl get svc app12
```
```
NAME    TYPE           EXTERNAL-IP   PORT(S)
app12   LoadBalancer   <pending>     80:30012/TCP
```

L'IP reste en `<pending>` → **plus d'IPs disponibles dans le pool**.

**Solution :** Agrandir le range dans la ConfigMap MetalLB.

## Intégration avec Ingress

MetalLB et Ingress sont **complémentaires** :

**MetalLB** : Fournit une IP externe au Service
**Ingress** : Route le trafic HTTP/HTTPS vers différents Services en fonction du hostname ou du path

**Architecture typique :**
```
┌──────────────────────────────────────────────────────┐
│                    Internet                          │
└───────────────────────┬──────────────────────────────┘
                        │
                 ┌──────▼────────┐
                 │   MetalLB     │
                 │ 192.168.1.200 │
                 └──────┬────────┘
                        │
                 ┌──────▼──────┐
                 │   Ingress   │
                 │  Controller │
                 └──────┬──────┘
                        │
        ┌───────────────┼───────────────┐
        │               │               │
   ┌────▼─────┐   ┌─────▼────┐   ┌──────▼───┐
   │ Service  │   │ Service  │   │ Service  │
   │   App1   │   │   App2   │   │   App3   │
   └──────────┘   └──────────┘   └──────────┘
```

**Exemple de configuration :**

```bash
# 1. Activer Ingress (NGINX)
microk8s enable ingress

# 2. Exposer l'Ingress Controller avec LoadBalancer
# (MicroK8s le fait automatiquement)
```

**Vérifier :**
```bash
microk8s kubectl get svc -n ingress
```

```
NAME                   TYPE           EXTERNAL-IP
nginx-ingress-...      LoadBalancer   192.168.1.200
```

Maintenant, **une seule IP (192.168.1.200)** peut servir plusieurs applications :

**Ingress avec hostnames :**
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: multi-app
spec:
  rules:
  - host: app1.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: app1-service
            port:
              number: 80
  - host: app2.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: app2-service
            port:
              number: 80
```

**Résultat :**
- `http://app1.example.com` → app1-service
- `http://app2.example.com` → app2-service
- Les deux utilisent la même IP (192.168.1.200)

**Économie d'IPs !** Vous n'utilisez qu'une seule IP du pool MetalLB pour plusieurs applications.

## Monitoring et Observabilité

### Vérifier l'État de MetalLB

**Pods MetalLB :**
```bash
microk8s kubectl get pods -n metallb-system
```

Tous les pods doivent être en `Running`.

**Logs du Controller :**
```bash
microk8s kubectl logs -n metallb-system -l component=controller
```

**Logs d'un Speaker :**
```bash
microk8s kubectl logs -n metallb-system -l component=speaker
```

**Événements MetalLB :**
```bash
microk8s kubectl get events -n metallb-system --sort-by='.lastTimestamp'
```

### Voir les Attributions d'IPs

**Liste de tous les Services LoadBalancer :**
```bash
microk8s kubectl get svc --all-namespaces -o wide | grep LoadBalancer
```

**Détails d'un Service spécifique :**
```bash
microk8s kubectl describe svc nginx-lb
```

Cherchez la section `LoadBalancer Ingress` :
```
LoadBalancer Ingress:     192.168.1.200
```

### Tester la Distribution

**Script pour générer du trafic et voir la distribution :**
```bash
#!/bin/bash
# test-load-balance.sh

SERVICE_IP="192.168.1.200"

for i in {1..100}; do
  curl -s http://$SERVICE_IP | grep "Server address" &
done

wait
```

Vous verrez que les requêtes sont distribuées entre différents pods.

### Prometheus et Grafana

Si Prometheus est activé, MetalLB expose des métriques :

**Requêtes PromQL utiles :**
```promql
# Nombre de Services LoadBalancer
count(kube_service_info{type="LoadBalancer"})

# IPs attribuées par MetalLB
metallb_allocator_addresses_in_use_total
```

## Bonnes Pratiques

### 1. Planifier le Range d'IPs

**Évitez de changer le range fréquemment.**

Planifiez en fonction de vos besoins :
- **Homelab petit** : 5-10 IPs suffisent
- **Homelab moyen** : 10-20 IPs
- **PME** : 20-50 IPs

**Laissez de la marge pour la croissance.**

### 2. Documenter les Attributions

**Maintenir un fichier avec les IPs utilisées :**
```markdown
# IPs MetalLB

Range total: 192.168.1.200-192.168.1.210

Attributions:
- 192.168.1.200: nginx-lb (namespace: default)
- 192.168.1.201: ingress-controller (namespace: ingress)
- 192.168.1.202: monitoring-grafana (namespace: observability)
- 192.168.1.203: app-production (namespace: prod)
- 192.168.1.204-210: Disponibles
```

### 3. Utiliser Ingress pour Économiser des IPs

**Au lieu de :**
```
app1 → LoadBalancer → 192.168.1.200
app2 → LoadBalancer → 192.168.1.201
app3 → LoadBalancer → 192.168.1.202
```

**Faire :**
```
Ingress → LoadBalancer → 192.168.1.200
  ├─ app1.example.com → app1
  ├─ app2.example.com → app2
  └─ app3.example.com → app3
```

**Économie : 2 IPs !**

### 4. Définir des Pools Séparés

Séparez les environnements avec des pools :
```yaml
address-pools:
- name: production
  protocol: layer2
  addresses:
  - 192.168.1.200-192.168.1.205
- name: staging
  protocol: layer2
  addresses:
  - 192.168.1.206-192.168.1.208
- name: development
  protocol: layer2
  addresses:
  - 192.168.1.209-192.168.1.210
```

### 5. Monitoring et Alertes

**Alertes recommandées :**
- Pool MetalLB avec < 20% d'IPs disponibles
- Speaker pod en erreur
- Service LoadBalancer en `<pending>` > 5 minutes

### 6. DNS Local

**Créer des enregistrements DNS pour les IPs :**

Dans votre DNS local (Pi-hole, dnsmasq, router) :
```
192.168.1.200    nginx.local
192.168.1.201    grafana.local
192.168.1.202    app.local
```

**Avantage :** Accès par nom plutôt que par IP.

## Dépannage

### Problème : EXTERNAL-IP reste en `<pending>`

**Causes possibles :**

**1. MetalLB pas installé**
```bash
microk8s kubectl get pods -n metallb-system
```

Si vide, installer MetalLB :
```bash
microk8s enable metallb
```

**2. Pool d'IPs vide**
```bash
microk8s kubectl get configmap -n metallb-system config -o yaml
```

Vérifier que le range d'adresses est défini.

**3. Toutes les IPs du pool sont utilisées**
```bash
microk8s kubectl get svc --all-namespaces | grep LoadBalancer
```

Comptez le nombre de Services. Si égal au nombre d'IPs du pool, agrandir le pool.

**4. Speaker pod en erreur**
```bash
microk8s kubectl get pods -n metallb-system
```

Si un speaker est en `Error` ou `CrashLoopBackOff` :
```bash
microk8s kubectl logs -n metallb-system <speaker-pod-name>
```

### Problème : IP attribuée mais pas accessible

**Diagnostic :**

**1. Tester depuis un nœud du cluster**
```bash
# Depuis node1
curl http://192.168.1.200
```

Si ça fonctionne depuis le cluster mais pas depuis l'extérieur :

**2. Vérifier le firewall**
```bash
# Sur chaque nœud
sudo ufw status
```

Autoriser le port :
```bash
sudo ufw allow 80/tcp
```

**3. Vérifier le routeur**
Assurez-vous que le routeur n'a pas de filtrage MAC ou règles qui bloquent les IPs MetalLB.

**4. Tester ARP**
```bash
# Depuis une machine du réseau
arp -a | grep 192.168.1.200
```

Vous devriez voir l'IP avec l'adresse MAC d'un des nœuds.

### Problème : Trafic pas distribué équitablement

**C'est normal avec Layer 2 !**

En mode Layer 2, MetalLB choisit un nœud leader. Tout le trafic passe par ce nœud avant d'être distribué vers les pods.

**Solution pour une vraie distribution :**
- Utiliser le mode BGP (nécessite routeur BGP)
- Ou accepter cette limitation (acceptable pour la plupart des cas)

### Problème : Changement de leader fréquent

**Symptôme :** Les logs MetalLB montrent beaucoup de "leader election".

**Cause possible :** Problème réseau (latence, paquets perdus).

**Diagnostic :**
```bash
# Tester la latence entre nœuds
ping -c 100 192.168.1.10
```

Si > 10% de paquets perdus ou latence > 50ms, investiguer le réseau.

## Alternatives à MetalLB

### 1. NodePort + Reverse Proxy Externe

**Principe :**
- Exposer les apps avec NodePort
- Un reverse proxy externe (HAProxy, Nginx) distribue vers les nœuds

**Avantages :**
- Pas de MetalLB nécessaire
- Contrôle total

**Inconvénients :**
- Plus complexe
- Reverse proxy = point de défaillance unique (SPOF)

### 2. Ingress seul (sans LoadBalancer)

**Principe :**
- Ingress Controller en NodePort
- Accès via `<NodeIP>:30080`

**Avantages :**
- Simple
- Pas besoin de MetalLB

**Inconvénients :**
- Port non-standard
- Pas de distribution L2

### 3. kube-vip

**Alternative à MetalLB :**
- Similaire à MetalLB
- Supporte aussi ARP et BGP
- Moins populaire mais viable

### 4. Pure LB

**Alternative émergente :**
- Plus moderne
- Moins mature que MetalLB

**MetalLB reste la solution la plus éprouvée et recommandée pour bare-metal.**

## Points Clés à Retenir

1. **Load balancing** = Distribution du trafic entre plusieurs serveurs
2. **Service Types** : ClusterIP (interne), NodePort (simple), LoadBalancer (idéal)
3. **MetalLB** = Load balancer logiciel pour bare-metal (non-cloud)
4. **Mode Layer 2** : Simple, recommandé pour débuter
5. **Range d'IPs** : Choisir un range libre, hors DHCP
6. **Installation** : `microk8s enable metallb` (ultra simple)
7. **Une IP par Service** LoadBalancer, ou Ingress pour partager
8. **HA native** : Si un nœud tombe, trafic automatiquement redirigé
9. **Monitoring** : Vérifier les speakers et les attributions d'IPs
10. **Bonnes pratiques** : Planifier le range, documenter, utiliser Ingress

## Prochaines Étapes

Vous maîtrisez maintenant le load balancing entre nœuds ! Les prochains chapitres couvriront :

- **21.7** : Stockage distribué pour la résilience des données
- **21.8** : Backup du control plane et restauration
- **21.9** : Stratégies de redondance avancées
- **21.10** : Tests de failover complets

Avec MetalLB, votre cluster multi-node est maintenant accessible de manière professionnelle, avec une IP unique et une haute disponibilité ! 🚀

---

**Félicitations !** Vous savez maintenant configurer et utiliser le load balancing dans un cluster MicroK8s multi-node. C'est une compétence essentielle pour des déploiements professionnels et résilients !

⏭️ [Stockage distribué](/21-multi-node-et-haute-disponibilite/07-stockage-distribue.md)
