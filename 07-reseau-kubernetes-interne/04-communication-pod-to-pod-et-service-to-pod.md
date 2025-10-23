🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 7.4 Communication Pod-to-Pod et Service-to-Pod

## Introduction

Dans les chapitres précédents, nous avons vu les fondations du réseau Kubernetes : les concepts de base, le CNI (Calico), et le DNS (CoreDNS). Maintenant, il est temps de voir comment tout cela fonctionne ensemble pour permettre aux applications de communiquer.

Dans ce chapitre, nous allons explorer concrètement comment les Pods communiquent entre eux, comment les Services facilitent cette communication, et quels sont les différents patterns de communication que vous rencontrerez.

## Les trois modes de communication dans Kubernetes

Il existe trois façons principales pour les applications de communiquer dans Kubernetes :

### 1. Communication Pod-to-Pod directe

```
┌──────────┐          ┌──────────┐
│  Pod A   │─────────▶│  Pod B   │
│10.1.1.5  │          │10.1.1.8  │
└──────────┘          └──────────┘
```

Le Pod A connaît l'IP du Pod B et communique directement avec lui.

**Quand l'utiliser ?**
- Rarement en pratique
- Principalement pour des cas spéciaux (debugging, tests)
- Dans des StatefulSets où chaque Pod a une identité stable

**Inconvénient majeur :** Si le Pod B redémarre, il change d'IP, et Pod A ne peut plus le joindre.

### 2. Communication via Service (le mode recommandé)

```
┌──────────┐          ┌─────────┐          ┌──────────┐
│  Pod A   │─────────▶│ Service │─────────▶│  Pod B   │
│10.1.1.5  │          │  (DNS)  │          │10.1.1.8  │
└──────────┘          └─────────┘          └──────────┘
```

Le Pod A utilise le nom DNS du Service, qui redirige vers un ou plusieurs Pods backend.

**Pourquoi c'est recommandé ?**
- **Abstraction** : Pod A ne connaît pas l'IP de Pod B
- **Résilience** : Si Pod B change d'IP, le Service s'adapte automatiquement
- **Load balancing** : Si plusieurs Pods backend, le trafic est distribué
- **Découplage** : Pod A et Pod B peuvent évoluer indépendamment

### 3. Communication via Ingress (pour l'accès externe)

```
Internet ──▶ Ingress ──▶ Service ──▶ Pods
```

Pour exposer des applications à l'extérieur du cluster (nous verrons ça dans les chapitres suivants).

## Communication Pod-to-Pod directe

Même si ce n'est pas le mode recommandé, comprenons d'abord comment fonctionne la communication directe entre Pods.

### Scénario 1 : Deux Pods sur le même Node

```
┌──────────────────────────────────────────────────┐
│                    Node 1                        │
│                 192.168.1.10                     │
│                                                  │
│   ┌──────────────┐           ┌──────────────┐    │
│   │   Pod A      │           │   Pod B      │    │
│   │  10.1.1.5    │           │  10.1.1.8    │    │
│   │              │           │              │    │
│   │  eth0        │           │  eth0        │    │
│   └──────┬───────┘           └──────┬───────┘    │
│          │                          │            │
│          │                          │            │
│   ┌──────▼───────┐           ┌──────▼───────┐    │
│   │ cali123456   │           │ cali789012   │    │
│   └──────┬───────┘           └──────┬───────┘    │
│          │                          │            │
│          └──────────┬───────────────┘            │
│                     │                            │
│            ┌────────▼────────┐                   │
│            │  Linux Bridge   │                   │
│            │  (ou routing)   │                   │
│            └─────────────────┘                   │
└──────────────────────────────────────────────────┘
```

**Étapes de communication :**

1. **Pod A crée un paquet** : Source=10.1.1.5, Destination=10.1.1.8
2. **Le paquet sort par eth0** du Pod A vers l'interface cali123456 du Node
3. **Le kernel Linux route** : "10.1.1.8 est local, envoyer via cali789012"
4. **Le paquet entre dans Pod B** via son eth0

**Caractéristiques :**
- ✅ **Très rapide** : Pas de traversée réseau physique
- ✅ **Faible latence** : Communication locale uniquement
- ✅ **Pas de NAT** : L'IP source reste 10.1.1.5

### Scénario 2 : Deux Pods sur des Nodes différents

```
┌──────────────────────────┐      ┌──────────────────────────┐
│       Node 1             │      │       Node 2             │
│    192.168.1.10          │      │    192.168.1.11          │
│                          │      │                          │
│  ┌──────────────┐        │      │        ┌──────────────┐  │
│  │   Pod A      │        │      │        │   Pod B      │  │
│  │  10.1.1.5    │        │      │        │  10.1.2.8    │  │
│  │              │        │      │        │              │  │
│  │  eth0        │        │      │        │  eth0        │  │
│  └──────┬───────┘        │      │        └──────▲───────┘  │
│         │                │      │               │          │
│  ┌──────▼───────┐        │      │        ┌──────┴───────┐  │
│  │ cali123456   │        │      │        │ cali456789   │  │
│  └──────┬───────┘        │      │        └──────▲───────┘  │
│         │                │      │               │          │
│  ┌──────▼───────┐        │      │        ┌──────┴───────┐  │
│  │   Routing    │        │      │        │   Routing    │  │
│  │    Table     │        │      │        │    Table     │  │
│  └──────┬───────┘        │      │        └──────▲───────┘  │
│         │                │      │               │          │
│  ┌──────▼───────┐        │      │        ┌──────┴───────┐  │
│  │     eth0     │        │      │        │     eth0     │  │
│  └──────┬───────┘        │      │        └──────▲───────┘  │
└─────────┼────────────────┘      └───────────────┼──────────┘
          │                                       │
          └───────────── Réseau physique ─────────┘
                    (Switch / Router)
```

**Étapes de communication :**

1. **Pod A crée un paquet** : Source=10.1.1.5, Destination=10.1.2.8
2. **Le paquet sort par eth0** du Pod A vers cali123456
3. **Node 1 consulte sa table de routage** :
   ```
   Destination 10.1.2.8 ?
   → Route apprise via BGP : 10.1.2.0/24 via 192.168.1.11
   → Envoyer vers Node 2
   ```
4. **Le paquet traverse le réseau physique** avec :
   - IP Source : **10.1.1.5** (toujours l'IP du Pod A, pas de NAT !)
   - IP Destination : **10.1.2.8** (IP du Pod B)
5. **Node 2 reçoit le paquet** et consulte sa table de routage :
   ```
   Destination 10.1.2.8 ?
   → Route locale : 10.1.2.8 dev cali456789
   → C'est un de mes Pods !
   ```
6. **Le paquet entre dans Pod B** via cali456789 puis eth0

**Caractéristiques :**
- ✅ **Pas de NAT** : Pod B voit l'IP réelle de Pod A (10.1.1.5)
- ✅ **Routage direct** : Pas d'encapsulation (avec Calico en mode direct routing)
- ⚠️ **Latence réseau** : Traversée du réseau physique (quelques millisecondes)

### Vérifier la communication directe

Vous pouvez tester la communication directe entre Pods :

```bash
# Créer deux Pods de test
microk8s kubectl run pod-a --image=nginx --restart=Never
microk8s kubectl run pod-b --image=busybox --restart=Never -- sleep 3600

# Obtenir l'IP de pod-a
POD_A_IP=$(microk8s kubectl get pod pod-a -o jsonpath='{.status.podIP}')

# Depuis pod-b, pinguer pod-a
microk8s kubectl exec pod-b -- ping -c 3 $POD_A_IP

# Devrait fonctionner !
```

### Pourquoi ne pas utiliser la communication directe ?

**Problème 1 : IPs éphémères**

```
Situation initiale :
Pod A communique avec Pod B (10.1.1.8)

Pod B plante et redémarre :
Pod B a maintenant l'IP 10.1.1.23

Pod A essaie toujours de contacter 10.1.1.8 → ÉCHEC
```

Votre application devrait détecter le changement d'IP et se reconfigurer. Complexe et fragile !

**Problème 2 : Pas de load balancing**

Si vous avez 3 Pods backend, comment choisir lequel contacter ? Vous devez implémenter votre propre logique de sélection.

**Problème 3 : Couplage fort**

Pod A doit connaître l'existence et l'IP de Pod B. Si vous changez l'architecture (ajouter des Pods, changer de déploiement), Pod A doit être modifié.

**Solution : Utiliser les Services !**

## Communication via Services (le bon pattern)

Les Services résolvent tous les problèmes de la communication directe.

### Qu'est-ce qu'un Service ?

Un **Service** est une abstraction qui représente un ensemble de Pods avec :
- **Un nom DNS stable** : `my-service.default.svc.cluster.local`
- **Une IP stable (ClusterIP)** : `10.152.183.25`
- **Un load balancer interne** : Distribue le trafic entre les Pods backend

### Anatomie d'un Service

```yaml
apiVersion: v1
kind: Service
metadata:
  name: backend
  namespace: production
spec:
  selector:
    app: backend      # Sélectionne les Pods avec ce label
  ports:
  - port: 80          # Port du Service
    targetPort: 8080  # Port du conteneur dans le Pod
  type: ClusterIP     # Type de Service (par défaut)
```

**Explication ligne par ligne :**

- **selector** : Définit quels Pods sont "backend" de ce Service
  - Tous les Pods avec le label `app=backend` sont sélectionnés
- **port: 80** : Le Service écoute sur le port 80
- **targetPort: 8080** : Le trafic est redirigé vers le port 8080 des Pods
- **type: ClusterIP** : Service accessible uniquement depuis le cluster

### Sélection des Pods par labels

Les Services utilisent les **labels** pour trouver leurs Pods backend.

**Exemple : 3 Pods avec le label app=backend**

```
┌──────────────────────────────────────────────────┐
│            Service "backend"                     │
│         ClusterIP: 10.152.183.25                 │
│         selector: app=backend                    │
└────────────────┬─────────────────────────────────┘
                 │
                 │ Sélectionne tous les Pods
                 │ qui ont le label app=backend
                 │
        ┌────────┼────────┐
        │        │        │
        ▼        ▼        ▼
    ┌──────┐ ┌──────┐ ┌──────┐
    │Pod 1 │ │Pod 2 │ │Pod 3 │
    │app:  │ │app:  │ │app:  │
    │back  │ │back  │ │back  │
    │end   │ │end   │ │end   │
    │10.1  │ │10.1  │ │10.1  │
    │.1.10 │ │.1.11 │ │.1.12 │
    └──────┘ └──────┘ └──────┘
```

**Point important :** Si vous scalez (ajoutez ou supprimez des Pods), le Service s'adapte automatiquement. Pas besoin de modifier quoi que ce soit !

### Communication via Service : étape par étape

Suivons une requête depuis un Pod frontend vers un Service backend.

```
┌─────────────────────────────────────────────────────────┐
│                                                         │
│  Étape 1 : Pod Frontend veut appeler "backend"          │
│  ┌──────────────────┐                                   │
│  │   Pod Frontend   │                                   │
│  │   10.1.1.5       │                                   │
│  │                  │                                   │
│  │  Code:           │                                   │
│  │  url = "http://backend:80/api"                       │
│  └──────────┬───────┘                                   │
│             │                                           │
└─────────────┼───────────────────────────────────────────┘
              │
              │ Étape 2 : Résolution DNS
              ▼
┌──────────────────────────────────────────────────────────┐
│           CoreDNS                                        │
│                                                          │
│  Query: "backend"                                        │
│  Avec search path → backend.production.svc.cluster.local │
│                                                          │
│  Réponse : 10.152.183.25 (ClusterIP du Service)          │
└─────────────┬────────────────────────────────────────────┘
              │
              │ Étape 3 : Connexion à l'IP du Service
              ▼
┌─────────────────────────────────────────────────────────┐
│  Pod Frontend envoie le paquet à 10.152.183.25:80       │
│                                                         │
│  Paquet : [Source: 10.1.1.5, Dest: 10.152.183.25:80]    │
└─────────────┬───────────────────────────────────────────┘
              │
              │ Étape 4 : Interception par kube-proxy/iptables
              ▼
┌─────────────────────────────────────────────────────────┐
│             Kube-proxy (iptables rules)                 │
│                                                         │
│  Règle : Si destination = 10.152.183.25:80              │
│  Alors : Choisir un Pod backend (round-robin)           │
│                                                         │
│  Pods disponibles :                                     │
│    - 10.1.1.10:8080                                     │
│    - 10.1.1.11:8080                                     │
│    - 10.1.1.12:8080                                     │
│                                                         │
│  Sélection : 10.1.1.11:8080 (Pod 2)                     │
│                                                         │
│  DNAT (Destination NAT) :                               │
│    Destination 10.152.183.25:80 → 10.1.1.11:8080        │
└─────────────┬───────────────────────────────────────────┘
              │
              │ Étape 5 : Paquet modifié routé vers Pod backend
              ▼
┌─────────────────────────────────────────────────────────┐
│  Paquet après DNAT :                                    │
│  [Source: 10.1.1.5, Dest: 10.1.1.11:8080]               │
│                                                         │
│  Le paquet est routé normalement (Pod-to-Pod)           │
└─────────────┬───────────────────────────────────────────┘
              │
              ▼
┌─────────────────────────────────────────────────────────┐
│         Pod Backend (Pod 2)                             │
│         10.1.1.11:8080                                  │
│                                                         │
│  Étape 6 : Pod Backend reçoit le paquet                 │
│  Voit Source: 10.1.1.5 (Pod Frontend)                   │
│                                                         │
│  Étape 7 : Pod Backend traite la requête                │
│  Génère une réponse                                     │
│                                                         │
│  Étape 8 : Réponse envoyée vers 10.1.1.5                │
│  [Source: 10.1.1.11:8080, Dest: 10.1.1.5]               │
└─────────────┬───────────────────────────────────────────┘
              │
              │ Étape 9 : Retour via iptables (reverse NAT)
              ▼
┌─────────────────────────────────────────────────────────┐
│             Kube-proxy (iptables rules)                 │
│                                                         │
│  Reverse NAT (SNAT) :                                   │
│    Source 10.1.1.11:8080 → 10.152.183.25:80             │
│                                                         │
│  Le Frontend pense que la réponse vient du Service      │
└─────────────┬───────────────────────────────────────────┘
              │
              ▼
┌─────────────────────────────────────────────────────────┐
│         Pod Frontend                                    │
│         10.1.1.5                                        │
│                                                         │
│  Étape 10 : Reçoit la réponse                           │
│  [Source: 10.152.183.25:80, Dest: 10.1.1.5]             │
│                                                         │
│  Frontend ne sait pas qu'il y avait plusieurs Pods !    │
│  Pour lui, il a juste parlé avec "backend"              │
└─────────────────────────────────────────────────────────┘
```

**Points clés à retenir :**

1. **DNS** : Le nom "backend" est résolu en IP du Service (10.152.183.25)
2. **ClusterIP virtuelle** : 10.152.183.25 n'existe pas vraiment sur un interface réseau
3. **Kube-proxy** : Intercepte le trafic via iptables et fait du DNAT
4. **Load balancing** : Choisit automatiquement un Pod backend
5. **Transparence** : Le Frontend ne sait pas quel Pod a traité sa requête

### Le rôle de kube-proxy

**Kube-proxy** est un composant Kubernetes qui s'exécute sur chaque Node. Il est responsable de l'implémentation des Services.

**Ses responsabilités :**

1. **Surveiller** l'API Kubernetes pour détecter les changements de Services/Endpoints
2. **Configurer iptables** (ou IPVS) pour rediriger le trafic vers les Pods
3. **Gérer le load balancing** entre les Pods backend

**Architecture :**

```
┌────────────────────────────────────────────────────┐
│                    Node                            │
│                                                    │
│  ┌──────────────────────────────────────────────┐  │
│  │           Kube-proxy                         │  │
│  │                                              │  │
│  │  1. Watch API Server                         │  │
│  │     ↓                                        │  │
│  │  2. Service/Endpoints changent ?             │  │
│  │     ↓                                        │  │
│  │  3. Mettre à jour les règles iptables        │  │
│  └──────────────────────────────────────────────┘  │
│                     │                              │
│                     ▼                              │
│  ┌──────────────────────────────────────────────┐  │
│  │           iptables / IPVS                    │  │
│  │                                              │  │
│  │  Règles pour tous les Services :             │  │
│  │  - Service A → Pods A1, A2, A3               │  │
│  │  - Service B → Pods B1, B2                   │  │
│  │  - ...                                       │  │
│  └──────────────────────────────────────────────┘  │
│                                                    │
│  Tous les paquets réseau passent par iptables      │
└────────────────────────────────────────────────────┘
```

### Les modes de kube-proxy

Kube-proxy peut fonctionner en plusieurs modes :

#### 1. iptables (mode par défaut dans MicroK8s)

**Avantages :**
- Mature et stable
- Fonctionne partout

**Inconvénients :**
- Performance dégradée avec beaucoup de Services (>1000)
- Load balancing basique (round-robin aléatoire)

#### 2. IPVS (mode avancé)

**Avantages :**
- Très performant même avec des milliers de Services
- Algorithmes de load balancing avancés (round-robin, least connection, etc.)
- Utilise moins de ressources CPU

**Inconvénients :**
- Nécessite le module kernel IPVS
- Légèrement plus complexe à debugger

### Endpoints : le lien entre Services et Pods

Quand vous créez un Service, Kubernetes crée automatiquement un objet **Endpoints** qui liste les IPs des Pods backend.

```bash
# Voir les Endpoints d'un Service
microk8s kubectl get endpoints backend

# Exemple de sortie :
NAME      ENDPOINTS                                    AGE
backend   10.1.1.10:8080,10.1.1.11:8080,10.1.1.12:8080   5m
```

**Fonctionnement :**

```
Service "backend" (selector: app=backend)
    ↓
Kubernetes trouve tous les Pods avec app=backend
    ↓
Crée/Met à jour l'objet Endpoints avec leurs IPs
    ↓
Kube-proxy lit les Endpoints
    ↓
Configure iptables pour rediriger vers ces IPs
```

**Important :** Si un Pod tombe ou démarre, l'objet Endpoints est automatiquement mis à jour, et kube-proxy reconfigure iptables. Tout est automatique !

## Patterns de communication courants

Voyons maintenant des scénarios réels de communication entre applications.

### Pattern 1 : Architecture 3-tiers classique

```
┌─────────────────────────────────────────────────────┐
│                                                     │
│  ┌──────────┐        ┌──────────┐       ┌────────┐  │
│  │ Frontend │───────▶│    API   │──────▶│   DB   │  │
│  │  (nginx) │        │  (node)  │       │(postgres  │
│  └──────────┘        └──────────┘       └────────┘  │
│                                                     │
│  Service:            Service:          Service:     │
│  "frontend"          "api"             "database"   │
│                                                     │
└─────────────────────────────────────────────────────┘
```

**Communication :**

1. **Frontend → API** :
   ```javascript
   // Dans le frontend
   fetch('http://api:8080/users')
   ```

2. **API → Database** :
   ```javascript
   // Dans l'API
   const db = new Client({
     host: 'database',
     port: 5432,
     database: 'myapp'
   });
   ```

**Avantages :**
- Chaque tier peut scaler indépendamment
- Abstraction totale des IPs
- Résilience aux redémarrages

### Pattern 2 : Microservices

```
┌──────────────────────────────────────────────────┐
│                                                  │
│         ┌──────────┐                             │
│     ┌───│  Gateway │───┐                         │
│     │   └──────────┘   │                         │
│     │                  │                         │
│     ▼                  ▼                         │
│  ┌────────┐      ┌──────────┐                    │
│  │ Users  │      │ Orders   │                    │
│  │Service │      │ Service  │                    │
│  └───┬────┘      └────┬─────┘                    │
│      │                │                          │
│      │                ▼                          │
│      │         ┌──────────┐                      │
│      └────────▶│Payments  │                      │
│                │Service   │                      │
│                └──────────┘                      │
│                                                  │
│  Chaque boîte est un Service avec ses Pods       │
└──────────────────────────────────────────────────┘
```

**Communication :**

```javascript
// Gateway appelle Users service
const response = await fetch('http://users-service:8080/user/123');

// Orders service appelle Payments service
const payment = await fetch('http://payments-service:8080/charge', {
  method: 'POST',
  body: JSON.stringify(orderData)
});
```

**Avantages :**
- Chaque service est indépendant
- Peut utiliser des technologies différentes
- Scalabilité fine par service

### Pattern 3 : Workers et Queue

```
┌─────────────────────────────────────────────────────┐
│                                                     │
│  ┌──────────┐      ┌──────────┐      ┌──────────┐   │
│  │   API    │─────▶│  Redis   │◀─────│  Worker  │   │
│  │          │ PUT  │  (Queue) │ GET  │          │   │
│  └──────────┘      └──────────┘      └──────────┘   │
│                                         (x3 Pods)   │
│  Service:          Service:            Service:     │
│  "api"             "redis"             "worker"     │
│                                                     │
└─────────────────────────────────────────────────────┘
```

**Communication :**

```python
# API envoie un job dans la queue
redis_client = Redis(host='redis', port=6379)
redis_client.lpush('jobs', json.dumps(job_data))

# Workers récupèrent les jobs
while True:
    job = redis_client.brpop('jobs')
    process_job(job)
```

**Avantages :**
- Traitement asynchrone
- Résilience : si un worker tombe, un autre prend le relais
- Scalabilité : ajoutez des workers selon la charge

### Pattern 4 : Communication cross-namespace

Parfois, des applications dans différents namespaces doivent communiquer.

```
┌────────────────────────────────────────────────┐
│        Namespace: production                   │
│                                                │
│  ┌──────────┐                                  │
│  │   API    │─────┐                            │
│  └──────────┘     │                            │
│                   │ Appel cross-namespace      │
└───────────────────┼────────────────────────────┘
                    │
                    ▼
┌────────────────────────────────────────────────┐
│        Namespace: shared                       │
│                                                │
│  ┌──────────────┐                              │
│  │  Monitoring  │                              │
│  │   Service    │                              │
│  └──────────────┘                              │
└────────────────────────────────────────────────┘
```

**Communication :**

```python
# Depuis namespace "production", appeler service dans "shared"
monitoring_url = 'http://monitoring.shared.svc.cluster.local:9090/metrics'
```

**Format DNS cross-namespace :**
```
<service>.<namespace>.svc.cluster.local
```

**Important :** Par défaut, les Pods peuvent communiquer entre namespaces. Si vous voulez restreindre cela, utilisez des **Network Policies** (chapitre sécurité).

## Types de Services

Nous avons vu le type ClusterIP jusqu'ici, mais il existe d'autres types de Services.

### 1. ClusterIP (par défaut)

```yaml
type: ClusterIP
```

**Caractéristiques :**
- IP interne au cluster uniquement
- Accessible depuis tous les Pods
- Non accessible depuis l'extérieur

**Cas d'usage :**
- Communication entre microservices
- Bases de données internes
- APIs internes

### 2. NodePort

```yaml
type: NodePort
ports:
- port: 80
  targetPort: 8080
  nodePort: 30080  # Port sur chaque Node (30000-32767)
```

**Caractéristiques :**
- Ouvre un port sur **tous les Nodes** du cluster
- Accessible via `<IP-du-Node>:30080`
- Crée automatiquement un ClusterIP aussi

**Schéma :**

```
Internet
   │
   ▼
Node IP:30080 ──▶ Service (ClusterIP) ──▶ Pods
```

**Cas d'usage :**
- Exposition simple pour développement/test
- Accès externe basique

**Inconvénients :**
- Ports dans une plage limitée (30000-32767)
- Pas de nom de domaine automatique
- Pas de SSL/TLS automatique

### 3. LoadBalancer

```yaml
type: LoadBalancer
ports:
- port: 80
  targetPort: 8080
```

**Caractéristiques :**
- Crée un load balancer externe (nécessite un cloud provider ou MetalLB)
- Obtient une IP externe
- Crée automatiquement NodePort et ClusterIP

**Schéma :**

```
Internet
   │
   ▼
Load Balancer (IP externe) ──▶ NodePort ──▶ Service (ClusterIP) ──▶ Pods
```

**Cas d'usage :**
- Exposition production
- Applications publiques

**Note :** Dans MicroK8s, vous devez activer MetalLB pour que ce type fonctionne (voir chapitre MetalLB).

### 4. ExternalName

```yaml
type: ExternalName
externalName: api.example.com
```

**Caractéristiques :**
- Crée un alias DNS vers un nom externe
- Pas de ClusterIP
- Pas de proxying

**Cas d'usage :**
- Référencer des services externes (ex: base de données managée en dehors du cluster)
- Migration progressive vers Kubernetes

**Exemple :**

```python
# Votre code appelle "database"
db_host = 'database'

# Mais "database" est un ExternalName qui pointe vers "prod-db.rds.amazonaws.com"
# Transparent pour l'application !
```

## Load Balancing : distribution du trafic

Quand un Service a plusieurs Pods backend, comment le trafic est-il distribué ?

### Algorithme : Random Selection (sélection aléatoire)

Dans le mode iptables (par défaut), kube-proxy utilise une **sélection aléatoire** avec probabilités égales.

**Exemple avec 3 Pods :**

```
Service "api" avec 3 Pods backend :
- Pod 1 : 10.1.1.10
- Pod 2 : 10.1.1.11
- Pod 3 : 10.1.1.12

Règles iptables générées :
33% de chance → Pod 1
33% de chance → Pod 2
33% de chance → Pod 3
```

**Conséquence :** Sur une grande quantité de requêtes, la distribution est approximativement égale.

### Session Affinity (affinité de session)

Par défaut, chaque requête peut aller vers un Pod différent. Parfois, vous voulez que toutes les requêtes d'un même client aillent au même Pod.

**Configuration :**

```yaml
apiVersion: v1
kind: Service
metadata:
  name: api
spec:
  sessionAffinity: ClientIP  # ← Active l'affinité par IP client
  sessionAffinityConfig:
    clientIP:
      timeoutSeconds: 10800  # 3 heures
  selector:
    app: api
  ports:
  - port: 80
    targetPort: 8080
```

**Comportement :**

```
Client avec IP 192.168.1.100 envoie une requête
    ↓
Kube-proxy : "Première requête de cette IP, choisir Pod 2"
    ↓
Toutes les requêtes suivantes de 192.168.1.100 vont vers Pod 2
(pendant 3 heures)
```

**Cas d'usage :**
- Applications avec sessions côté serveur
- WebSockets
- Connexions longues

**Attention :** L'affinité est basée sur l'IP **source**. Si plusieurs clients partagent la même IP (NAT), ils iront tous au même Pod (déséquilibre possible).

## Ports : mapping et traduction

Comprendre comment les ports sont mappés est crucial.

### Port du Service vs TargetPort

```yaml
ports:
- port: 80          # Port du Service
  targetPort: 8080  # Port du conteneur
```

**Traduction :**

```
Client appelle → Service:80
                    ↓
             Redirigé vers Pod:8080
```

**Avantage :** Vous pouvez standardiser le port du Service (ex: toujours 80 pour HTTP) même si vos applications écoutent sur des ports différents.

**Exemple multi-applications :**

```yaml
# Service frontend
ports:
- port: 80
  targetPort: 3000  # React dev server

# Service API
ports:
- port: 80
  targetPort: 8080  # Spring Boot

# Service admin
ports:
- port: 80
  targetPort: 5000  # Flask
```

Tous exposent le port 80, mais les applications backend utilisent leurs propres ports.

### Named Ports

Vous pouvez donner des noms aux ports dans les Pods :

```yaml
# Définition du Pod
spec:
  containers:
  - name: app
    ports:
    - name: http
      containerPort: 8080
    - name: metrics
      containerPort: 9090
```

Puis référencer ces noms dans le Service :

```yaml
# Service
spec:
  ports:
  - name: web
    port: 80
    targetPort: http  # ← Référence au port nommé
  - name: monitoring
    port: 9090
    targetPort: metrics
```

**Avantages :**
- Plus lisible
- Plus flexible (changer le port dans le Pod ne nécessite pas de changer le Service)

### Multiple Ports

Un Service peut exposer plusieurs ports :

```yaml
apiVersion: v1
kind: Service
metadata:
  name: fullstack-app
spec:
  selector:
    app: fullstack
  ports:
  - name: http
    port: 80
    targetPort: 8080
  - name: https
    port: 443
    targetPort: 8443
  - name: metrics
    port: 9090
    targetPort: 9090
```

**Usage :**

```python
# Connexion HTTP
http_url = 'http://fullstack-app:80'

# Connexion HTTPS
https_url = 'https://fullstack-app:443'

# Métriques Prometheus
metrics_url = 'http://fullstack-app:9090/metrics'
```

## Protocoles supportés

Les Services Kubernetes supportent plusieurs protocoles.

### TCP (par défaut)

```yaml
ports:
- port: 80
  protocol: TCP  # Peut être omis (défaut)
```

**Usage :** HTTP, HTTPS, bases de données, APIs REST, etc.

### UDP

```yaml
ports:
- port: 53
  protocol: UDP
```

**Usage :** DNS, protocoles de streaming, jeux vidéo

### SCTP (Stream Control Transmission Protocol)

```yaml
ports:
- port: 9090
  protocol: SCTP
```

**Usage :** Télécommunications, protocoles industriels (rarement utilisé)

## Readiness et Liveness : santé des Pods

Les Services ne redirigent le trafic que vers des Pods **sains** (ready).

### Readiness Probe

Une **Readiness Probe** indique si un Pod est prêt à recevoir du trafic.

```yaml
spec:
  containers:
  - name: app
    image: myapp:latest
    readinessProbe:
      httpGet:
        path: /health
        port: 8080
      initialDelaySeconds: 5
      periodSeconds: 10
```

**Fonctionnement :**

```
Pod démarre
    ↓
Kubernetes attend 5 secondes (initialDelaySeconds)
    ↓
Envoie une requête GET http://pod-ip:8080/health
    ↓
Si réponse 200 OK → Pod marqué "Ready"
    ↓
Kube-proxy ajoute le Pod aux Endpoints du Service
    ↓
Le Pod commence à recevoir du trafic
```

**Si la probe échoue :**

```
Pod répond 500 Internal Server Error
    ↓
Kubernetes marque le Pod comme "Not Ready"
    ↓
Kube-proxy retire le Pod des Endpoints
    ↓
Plus aucun trafic n'est envoyé vers ce Pod
```

**Importance :** Sans Readiness Probe, un Pod pourrait recevoir du trafic avant d'être prêt (ex: avant que la base de données soit connectée), causant des erreurs 500.

### Liveness Probe

Une **Liveness Probe** indique si un Pod est vivant ou doit être redémarré.

```yaml
spec:
  containers:
  - name: app
    image: myapp:latest
    livenessProbe:
      httpGet:
        path: /healthz
        port: 8080
      initialDelaySeconds: 15
      periodSeconds: 20
```

**Différence avec Readiness :**

- **Readiness** : "Es-tu prêt à recevoir du trafic ?"
  - Non → Retire des Endpoints, mais ne redémarre pas
- **Liveness** : "Es-tu vivant ?"
  - Non → Redémarre le conteneur

**Cas d'usage Liveness :**
- Deadlocks (application bloquée)
- Fuites mémoire (application trop lente)
- Corruption interne

**Bonnes pratiques :**
- Utilisez **toujours** une Readiness Probe
- Utilisez une Liveness Probe si votre app peut se bloquer
- Ne faites pas de checks trop lourds (risque de faux positifs)

## Découverte de services : patterns avancés

### Service Discovery manuel

Si vous avez vraiment besoin de connaître tous les Pods d'un Service :

```python
import socket

# Résoudre le nom DNS du Service
service_name = "backend.production.svc.cluster.local"
try:
    ips = socket.getaddrinfo(service_name, 8080, socket.AF_INET)
    pod_ips = [ip[4][0] for ip in ips]
    print(f"Pods backend : {pod_ips}")
except socket.gaierror:
    print("Service not found")
```

**Note :** Cela ne fonctionne qu'avec des **Headless Services** (clusterIP: None).

### Variables d'environnement (legacy)

Kubernetes injecte automatiquement des variables d'environnement pour chaque Service dans les Pods.

**Exemple :**

Si un Service "database" existe, les Pods verront :

```bash
DATABASE_SERVICE_HOST=10.152.183.25
DATABASE_SERVICE_PORT=5432
```

**Usage :**

```python
import os

db_host = os.environ.get('DATABASE_SERVICE_HOST')
db_port = os.environ.get('DATABASE_SERVICE_PORT')
```

**Inconvénient :** Les variables ne sont injectées qu'au démarrage du Pod. Si vous créez un Service après un Pod, ce Pod ne verra pas les variables.

**Recommandation :** Utilisez plutôt le DNS (plus fiable et dynamique).

## Performance et optimisation

### Localité du trafic

Par défaut, un Service peut envoyer le trafic vers n'importe quel Pod, même sur un autre Node.

**Avec 2 Nodes :**

```
┌─────────────────┐      ┌─────────────────┐
│     Node 1      │      │     Node 2      │
│                 │      │                 │
│  ┌──────────┐   │      │  ┌──────────┐   │
│  │Frontend  │───┼──────┼─▶│ Backend  │   │
│  │          │   │      │  │  Pod     │   │
│  └──────────┘   │      │  └──────────┘   │
│                 │      │                 │
│  ┌──────────┐   │      │                 │
│  │ Backend  │◄──┼──────┼──Trafic peut    │
│  │  Pod     │   │      │  venir de Node2 │
│  └──────────┘   │      │                 │
└─────────────────┘      └─────────────────┘
```

**Impact :** Latence réseau supplémentaire quand le trafic traverse les Nodes.

**Solution : topologyKeys (déprécié) ou Service Internal Traffic Policy**

```yaml
apiVersion: v1
kind: Service
metadata:
  name: backend
spec:
  internalTrafficPolicy: Local  # ← Trafic local uniquement
  selector:
    app: backend
  ports:
  - port: 80
```

**Comportement :**

Avec `internalTrafficPolicy: Local`, le trafic est envoyé uniquement aux Pods **sur le même Node** que le client.

**Avantages :**
- Latence réduite
- Pas de trafic réseau inter-Node

**Inconvénients :**
- Déséquilibre possible si les Pods ne sont pas uniformément distribués
- Si aucun Pod local, la requête échoue

## Debugging de la communication

Quand la communication ne fonctionne pas, voici comment diagnostiquer.

### Problème 1 : "Connection refused" ou "Connection timeout"

**Diagnostic :**

**Étape 1 : Vérifier que le Service existe**

```bash
microk8s kubectl get svc backend

# Devrait afficher le Service avec une ClusterIP
```

**Étape 2 : Vérifier les Endpoints**

```bash
microk8s kubectl get endpoints backend

# Devrait lister les IPs des Pods
# Si vide → Aucun Pod ne correspond au selector
```

**Étape 3 : Vérifier les Pods**

```bash
microk8s kubectl get pods -l app=backend

# Vérifier que des Pods existent et sont en "Running" et "Ready"
```

**Étape 4 : Tester depuis un Pod**

```bash
# Créer un Pod de test
microk8s kubectl run test --image=busybox --restart=Never -- sleep 3600

# Tester la résolution DNS
microk8s kubectl exec test -- nslookup backend

# Tester la connexion
microk8s kubectl exec test -- wget -O- http://backend:80

# Devrait retourner une réponse
```

### Problème 2 : DNS ne résout pas

**Voir le chapitre 7.3 sur CoreDNS pour le debugging DNS complet.**

**Quick check :**

```bash
# Depuis un Pod, vérifier /etc/resolv.conf
microk8s kubectl exec test -- cat /etc/resolv.conf

# Devrait contenir : nameserver 10.152.183.10
```

### Problème 3 : Le Service résout, mais connexion échoue

**Cause probable : Port incorrect**

```yaml
# Service expose le port 80
ports:
- port: 80
  targetPort: 8080

# Mais votre application écoute sur 3000, pas 8080 !
```

**Solution :** Corriger le `targetPort` pour qu'il corresponde au port de l'application.

**Vérifier le port d'écoute du Pod :**

```bash
# Voir les logs du Pod
microk8s kubectl logs <pod-name>

# Chercher une ligne comme : "Listening on port 3000"
```

### Problème 4 : Certaines requêtes échouent, d'autres réussissent

**Cause probable : Un Pod backend est défaillant**

```bash
# Vérifier l'état de tous les Pods
microk8s kubectl get pods -l app=backend

# Vérifier les logs du Pod défaillant
microk8s kubectl logs <pod-name>

# Vérifier les events
microk8s kubectl describe pod <pod-name>
```

**Solution :** Ajouter ou améliorer les **Readiness Probes** pour que les Pods défaillants soient automatiquement retirés.

## Bonnes pratiques

### 1. Toujours utiliser des Services (pas d'IPs en dur)

❌ **Mauvais :**
```python
api_url = "http://10.1.1.5:8080"
```

✅ **Bon :**
```python
api_url = "http://api:8080"
```

### 2. Utiliser des noms de Services courts dans le même namespace

✅ **Bon :**
```python
# Depuis namespace "production"
db_host = "database"  # Résout vers database.production.svc.cluster.local
```

### 3. Utiliser des FQDN cross-namespace

✅ **Bon :**
```python
# Depuis namespace "app" vers namespace "monitoring"
metrics_url = "http://prometheus.monitoring.svc.cluster.local:9090"
```

### 4. Définir des Readiness Probes

```yaml
readinessProbe:
  httpGet:
    path: /health
    port: 8080
  initialDelaySeconds: 5
  periodSeconds: 10
```

**Toujours définir une Readiness Probe** pour éviter que des Pods non prêts reçoivent du trafic.

### 5. Utiliser des Named Ports

```yaml
# Dans le Pod
ports:
- name: http
  containerPort: 8080

# Dans le Service
ports:
- port: 80
  targetPort: http
```

Plus maintenable et lisible.

### 6. Configurer des limites de ressources

```yaml
resources:
  requests:
    memory: "128Mi"
    cpu: "100m"
  limits:
    memory: "256Mi"
    cpu: "200m"
```

Empêche un Pod défaillant de monopoliser les ressources du Node.

### 7. Utiliser des labels significatifs

```yaml
labels:
  app: backend
  version: v2
  tier: api
  environment: production
```

Facilite la sélection et l'organisation.

### 8. Tester la résilience

Simulez des pannes pour vérifier que votre application tolère les redémarrages :

```bash
# Tuer un Pod
microk8s kubectl delete pod <pod-name>

# Vérifier que le Service continue de fonctionner
# (les autres Pods prennent le relais)
```

## Points clés à retenir

### ✅ Deux modes de communication

1. **Pod-to-Pod direct** : Rarement utilisé, IPs éphémères
2. **Via Services** : Recommandé, abstraction stable

### ✅ Services = Load Balancers internes

- DNS stable
- ClusterIP stable
- Distribution automatique du trafic

### ✅ Kube-proxy implémente les Services

- Via iptables (ou IPVS)
- DNAT (Destination NAT)
- Load balancing automatique

### ✅ Endpoints = Liste des Pods backend

- Mis à jour automatiquement
- Utilisé par kube-proxy

### ✅ Types de Services

- **ClusterIP** : Interne uniquement (par défaut)
- **NodePort** : Accessible via Node IP
- **LoadBalancer** : Avec IP externe (nécessite MetalLB ou cloud)
- **ExternalName** : Alias vers nom externe

### ✅ Readiness Probes

Cruciales pour garantir que seuls les Pods sains reçoivent du trafic

### ✅ DNS pour la découverte

- Nom court : même namespace
- FQDN : `service.namespace.svc.cluster.local`

## Pourquoi tout cela est important ?

La communication est le cœur de toute architecture distribuée. Comprendre comment les Pods et Services communiquent vous permet de :

1. **Concevoir des architectures robustes** : Services découplés et résilients
2. **Déboguer efficacement** : Comprendre où chercher quand ça ne fonctionne pas
3. **Optimiser les performances** : Réduire la latence, améliorer le débit
4. **Sécuriser** : Comprendre les flux pour appliquer les bonnes Network Policies

## Prochaines étapes

Maintenant que vous maîtrisez la communication interne, le prochain chapitre explore :

- **7.5 Test de connectivité interne** : Outils et commandes pratiques pour diagnostiquer et valider le réseau

Avec ces connaissances sur la communication Pod-to-Pod et Service-to-Pod, vous avez les fondations pour construire des applications distribuées robustes dans Kubernetes !

---

**Note :** Ce chapitre se concentre sur la communication interne au cluster. L'exposition externe (Ingress, LoadBalancer externe) sera abordée dans les chapitres suivants.

⏭️ [Test de connectivité interne](/07-reseau-kubernetes-interne/05-test-de-connectivite-interne.md)
