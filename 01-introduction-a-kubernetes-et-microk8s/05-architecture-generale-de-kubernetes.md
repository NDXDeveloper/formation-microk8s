🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 1.5 Architecture générale de Kubernetes

## Introduction

Maintenant que nous comprenons ce qu'est Kubernetes et pourquoi MicroK8s est un excellent choix pour notre lab, il est temps de regarder sous le capot. Comment Kubernetes fonctionne-t-il réellement ? Quels sont les différents composants qui travaillent ensemble pour orchestrer vos conteneurs ?

Ne vous inquiétez pas si l'architecture semble complexe au premier abord. Nous allons la décomposer étape par étape, en utilisant des analogies simples pour rendre chaque concept clair et accessible.

## Vue d'ensemble : l'analogie de l'entreprise

Imaginez Kubernetes comme une **entreprise bien organisée** :

- **Le control plane** (plan de contrôle) = La direction et les bureaux administratifs
- **Les worker nodes** (nœuds travailleurs) = Les entrepôts où le travail réel est effectué
- **Les conteneurs** = Les produits fabriqués dans les entrepôts
- **L'API Server** = Le PDG qui reçoit toutes les demandes et prend les décisions
- **Le Scheduler** = Le responsable RH qui assigne les tâches aux travailleurs
- **Les Controllers** = Les superviseurs qui vérifient que tout fonctionne correctement

Cette entreprise fonctionne de manière autonome : donnez-lui des instructions sur ce que vous voulez, et elle s'organise pour y arriver !

## Architecture en deux parties

Kubernetes est composé de deux parties principales :

### 1. Le Control Plane (Plan de Contrôle)

C'est le **cerveau** de Kubernetes. Il prend toutes les décisions importantes :
- Où déployer les conteneurs ?
- Quand créer de nouveaux conteneurs ?
- Que faire si un conteneur tombe en panne ?
- Comment router le trafic réseau ?

**Rôle :** Gérer et coordonner tout le cluster

### 2. Les Worker Nodes (Nœuds Travailleurs)

Ce sont les **muscles** de Kubernetes. Ils exécutent le travail réel :
- Faire tourner les conteneurs
- Héberger les applications
- Gérer les ressources locales (CPU, mémoire, stockage)

**Rôle :** Exécuter vos applications conteneurisées

## Visualisation de l'architecture

Voici comment ces éléments s'organisent :

```
┌────────────────────────────────────────────────────────────┐
│                     CONTROL PLANE                          │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐      │
│  │  API Server  │  │  Scheduler   │  │ Controller   │      │
│  │              │  │              │  │  Manager     │      │
│  └──────────────┘  └──────────────┘  └──────────────┘      │
│                                                            │
│         ┌──────────────┐                                   │
│         │     etcd     │  (Base de données)                │
│         └──────────────┘                                   │
└────────────────────────────────────────────────────────────┘
                            │
                            │ Communication
                            │
        ┌───────────────────┴───────────────────┐
        │                                       │
┌───────▼──────────┐                   ┌────────▼─────────┐
│  WORKER NODE 1   │                   │  WORKER NODE 2   │
│                  │                   │                  │
│  ┌────────────┐  │                   │  ┌────────────┐  │
│  │  Kubelet   │  │                   │  │  Kubelet   │  │
│  └────────────┘  │                   │  └────────────┘  │
│  ┌────────────┐  │                   │  ┌────────────┐  │
│  │ Kube-Proxy │  │                   │  │ Kube-Proxy │  │
│  └────────────┘  │                   │  └────────────┘  │
│  ┌────────────┐  │                   │  ┌────────────┐  │
│  │ Container  │  │                   │  │ Container  │  │
│  │  Runtime   │  │                   │  │  Runtime   │  │
│  └────────────┘  │                   │  └────────────┘  │
│                  │                   │                  │
│  [Pods running]  │                   │  [Pods running]  │
└──────────────────┘                   └──────────────────┘
```

## Les composants du Control Plane

Plongeons maintenant dans chaque composant du control plane pour comprendre son rôle.

### 1. API Server (kube-apiserver)

**Analogie :** Le réceptionniste ou le standard téléphonique de l'entreprise

**Rôle :**
C'est le **point d'entrée unique** pour toutes les communications dans Kubernetes. Tout passe par lui :
- Vous voulez créer un pod ? Vous parlez à l'API Server
- Le scheduler veut vérifier les ressources ? Il demande à l'API Server
- Un controller veut modifier un déploiement ? Il passe par l'API Server

**Fonctionnement :**
1. Reçoit les requêtes (via kubectl, dashboard, ou autres outils)
2. Valide les requêtes (est-ce que vous avez le droit ? est-ce valide ?)
3. Traite les requêtes et met à jour l'état dans etcd
4. Retourne les réponses

**Caractéristiques :**
- Interface RESTful (HTTP/HTTPS)
- Seul composant qui parle directement à etcd
- Peut être mis à l'échelle horizontalement (plusieurs instances)
- Point central de sécurité et d'authentification

**Commandes qui parlent à l'API Server :**
```bash
kubectl get pods              # Demande la liste des pods
kubectl create deployment     # Demande de créer un déploiement
kubectl delete service        # Demande de supprimer un service
```

### 2. etcd

**Analogie :** Le système de classement et d'archivage de l'entreprise

**Rôle :**
C'est la **base de données** de Kubernetes. Tout l'état du cluster est stocké ici :
- Quels pods existent ?
- Quelle est leur configuration ?
- Sur quel nœud tournent-ils ?
- Quels sont les services exposés ?
- Toutes les configurations

**Fonctionnement :**
- Base de données clé-valeur distribuée
- Cohérence forte (tous les nœuds voient la même chose)
- Rapide et fiable
- Peut être répliqué pour haute disponibilité

**Caractéristiques importantes :**
- Seul l'API Server communique avec etcd (les autres composants passent par l'API)
- Source de vérité unique (single source of truth)
- Sauvegarder etcd = sauvegarder tout votre cluster
- Performance critique pour le cluster

**Ce qui est stocké dans etcd :**
- État de tous les objets Kubernetes (pods, services, deployments, etc.)
- Configuration du cluster
- Secrets et ConfigMaps
- État des nœuds
- Métadonnées diverses

**Note MicroK8s :** Par défaut, MicroK8s utilise **Dqlite** au lieu d'etcd. C'est une alternative plus légère basée sur SQLite, spécialement conçue pour simplifier la haute disponibilité.

### 3. Scheduler (kube-scheduler)

**Analogie :** Le responsable des ressources humaines qui assigne les employés aux projets

**Rôle :**
Le scheduler décide **où** exécuter chaque pod. Quand vous créez un nouveau pod, c'est lui qui choisit sur quel nœud il va tourner.

**Processus de décision :**
1. **Filtrage** : Élimine les nœuds impossibles
   - Pas assez de CPU disponible ? Éliminé
   - Pas assez de mémoire ? Éliminé
   - Le pod nécessite un GPU et ce nœud n'en a pas ? Éliminé

2. **Scoring** : Note les nœuds restants
   - Quel nœud a le plus de ressources disponibles ?
   - Quel nœud répartira le mieux la charge ?
   - Les affinités et anti-affinités sont-elles respectées ?

3. **Sélection** : Choisit le nœud avec le meilleur score

**Critères de décision :**
- Ressources disponibles (CPU, mémoire)
- Contraintes matérielles (SSD, GPU, etc.)
- Labels et selectors
- Affinités et anti-affinités (placer ensemble ou séparer)
- Taints et tolerations
- Répartition de la charge
- Topologie (zones de disponibilité, régions)

**Exemple concret :**
```
Vous créez un pod qui nécessite 2 CPU et 4 Go RAM

Le scheduler regarde :
- Node1 : 1 CPU disponible ❌ Éliminé
- Node2 : 4 CPU et 8 Go disponibles ✅ Score 8/10
- Node3 : 8 CPU et 16 Go disponibles ✅ Score 10/10

Le pod est placé sur Node3
```

### 4. Controller Manager (kube-controller-manager)

**Analogie :** L'équipe de supervision qui vérifie constamment que tout fonctionne comme prévu

**Rôle :**
Le Controller Manager est en fait une **collection de contrôleurs** qui surveillent l'état du cluster et effectuent les actions nécessaires pour atteindre l'état désiré.

**Principe fondamental : La boucle de contrôle**
```
1. Observer l'état actuel
2. Comparer avec l'état désiré
3. Si différent : effectuer une action corrective
4. Répéter en permanence
```

**Contrôleurs principaux :**

**Node Controller**
- Surveille l'état des nœuds
- Détecte quand un nœud tombe en panne
- Décide quand considérer un nœud comme "mort"
- Prend des mesures correctives (évacuer les pods)

**Replication Controller**
- Assure le bon nombre de réplicas de pods
- Vous voulez 3 réplicas ? Il s'assure qu'il y en a toujours 3
- En crée de nouveaux si certains meurent
- En supprime s'il y en a trop

**Endpoints Controller**
- Gère les endpoints (connexions entre services et pods)
- Met à jour les endpoints quand des pods démarrent/arrêtent
- Assure que le trafic va aux bons pods

**Service Account & Token Controllers**
- Gèrent les comptes de service
- Créent les tokens d'authentification
- Gèrent les permissions

**Autres contrôleurs :**
- Deployment Controller
- Job Controller
- DaemonSet Controller
- Et bien d'autres...

**Exemple de boucle de contrôle :**
```
État désiré : 3 réplicas de mon application
État actuel : 2 réplicas (un a crashé)

Le Replication Controller détecte la différence
→ Il demande au scheduler de placer un nouveau pod
→ Le scheduler choisit un nœud
→ Le kubelet sur ce nœud démarre le pod
→ État actuel : 3 réplicas ✅
```

### 5. Cloud Controller Manager (optionnel)

**Rôle :**
Gère l'intégration avec les fournisseurs cloud (AWS, Azure, GCP). Il traduit les concepts Kubernetes en ressources cloud.

**Exemples :**
- Service LoadBalancer → Créer un load balancer AWS ELB
- PersistentVolume → Créer un volume EBS
- Nodes → Gérer les instances EC2

**Note :** Pas nécessaire pour MicroK8s dans un lab local, mais important en production cloud.

## Les composants des Worker Nodes

Les worker nodes sont là où vos applications tournent réellement. Chaque worker node contient plusieurs composants.

### 1. Kubelet

**Analogie :** Le contremaître de l'entrepôt

**Rôle :**
Le kubelet est l'**agent Kubernetes** qui tourne sur chaque nœud. C'est lui qui fait réellement le travail d'exécution des pods.

**Responsabilités :**
1. **Communication avec le control plane**
   - S'enregistre auprès de l'API Server
   - Reçoit les instructions sur les pods à exécuter
   - Rapporte l'état du nœud et des pods

2. **Gestion des pods**
   - Démarre les conteneurs via le container runtime
   - Surveille l'état des conteneurs
   - Redémarre les conteneurs qui crashent
   - Arrête les pods supprimés

3. **Health checks**
   - Exécute les liveness probes (le conteneur est-il vivant ?)
   - Exécute les readiness probes (le conteneur est-il prêt à recevoir du trafic ?)
   - Rapporte les résultats à l'API Server

4. **Gestion des ressources**
   - Monte les volumes
   - Applique les limites CPU/mémoire
   - Gère les secrets et ConfigMaps

**Fonctionnement :**
```
1. Kubelet demande à l'API Server : "Quels pods dois-je exécuter ?"
2. API Server répond : "Tu dois exécuter le pod nginx-xyz"
3. Kubelet demande au container runtime de démarrer les conteneurs
4. Kubelet surveille constamment les conteneurs
5. Kubelet rapporte l'état à l'API Server
```

**Important :** Le kubelet ne gère PAS directement les conteneurs. Il demande au container runtime de le faire.

### 2. Kube-Proxy

**Analogie :** Le système de routage et d'aiguillage postal de l'entrepôt

**Rôle :**
Kube-proxy gère le **réseau** au niveau du nœud. Il s'assure que les connexions réseau arrivent aux bons pods.

**Responsabilités :**
1. **Implémentation des Services**
   - Maintient les règles réseau pour les Services
   - Permet aux pods de communiquer via des Services stables
   - Route le trafic vers les bons pods

2. **Load balancing**
   - Distribue le trafic entre plusieurs réplicas d'un pod
   - Effectue du load balancing basique

3. **Gestion des règles réseau**
   - Utilise iptables (ou IPVS, ou userspace selon configuration)
   - Met à jour les règles quand des pods démarrent/arrêtent
   - Gère la traduction d'adresses (NAT)

**Exemple de fonctionnement :**
```
Vous avez un Service "webapp" avec 3 pods backend

Kube-proxy crée des règles qui disent :
"Quand quelqu'un appelle webapp:80,
 redirige vers l'un des 3 pods (round-robin)"

Pod A crashe → Kube-proxy met à jour les règles
Maintenant, le trafic va seulement vers Pod B et Pod C
```

**Modes de fonctionnement :**
- **iptables** (mode par défaut) : Utilise les règles iptables Linux
- **IPVS** : Plus performant pour de nombreux services
- **userspace** : Mode legacy, rarement utilisé

### 3. Container Runtime

**Analogie :** Les machines et outils dans l'entrepôt qui font le travail réel

**Rôle :**
C'est le logiciel qui exécute réellement les conteneurs. Kubernetes ne fait pas tourner les conteneurs directement - il délègue ce travail au container runtime.

**Options de container runtime :**
- **containerd** (le plus courant) : Runtime standard, utilisé par MicroK8s
- **CRI-O** : Runtime spécialement conçu pour Kubernetes
- **Docker** (via dockershim, deprecated) : L'ancien standard

**Responsabilités :**
1. Télécharger les images de conteneurs depuis un registre
2. Démarrer et arrêter les conteneurs
3. Gérer le stockage des conteneurs
4. Gérer le réseau des conteneurs (niveau bas)

**Communication :**
Le kubelet parle au container runtime via une interface standard appelée **CRI (Container Runtime Interface)**.

```
Kubelet → "Hey containerd, démarre un conteneur nginx"
containerd → "OK, je télécharge l'image et je démarre le conteneur"
containerd → "C'est fait, le conteneur est en cours d'exécution"
```

## Comment tout cela fonctionne ensemble

Voyons maintenant un exemple complet pour comprendre comment tous ces composants collaborent.

### Scénario : Déployer une application web

**Vous exécutez :**
```bash
kubectl create deployment webapp --image=nginx --replicas=3
```

**Voici ce qui se passe en coulisses :**

#### Étape 1 : Réception de la commande
```
kubectl → API Server : "Crée un Deployment avec 3 réplicas nginx"
API Server : Valide la requête, authentification OK
API Server → etcd : "Enregistre ce nouveau Deployment"
etcd : "OK, enregistré"
```

#### Étape 2 : Le Deployment Controller détecte
```
Deployment Controller (scrute constamment l'API Server)
Deployment Controller : "Nouveau Deployment détecté !"
Deployment Controller : "Je dois créer un ReplicaSet pour gérer les réplicas"
Deployment Controller → API Server : "Crée un ReplicaSet pour ce Deployment"
```

#### Étape 3 : Le ReplicaSet Controller intervient
```
ReplicaSet Controller : "Nouveau ReplicaSet détecté !"
ReplicaSet Controller : "Je dois créer 3 pods"
ReplicaSet Controller → API Server : "Crée 3 pods"
API Server → etcd : "Enregistre ces 3 pods (état : Pending)"
```

#### Étape 4 : Le Scheduler place les pods
```
Scheduler (scrute constamment les nouveaux pods)
Scheduler : "3 nouveaux pods sans nœud assigné !"
Scheduler : Analyse les nœuds disponibles
Scheduler : "Pod1 → Node1, Pod2 → Node2, Pod3 → Node1"
Scheduler → API Server : "Assigne ces pods à ces nœuds"
```

#### Étape 5 : Les Kubelets exécutent les pods
```
Kubelet sur Node1 : "L'API Server dit que je dois exécuter Pod1 et Pod3"
Kubelet → containerd : "Démarre un conteneur nginx pour Pod1"
containerd : Télécharge l'image nginx si nécessaire
containerd : Démarre le conteneur
Kubelet : Surveille le conteneur
Kubelet → API Server : "Pod1 est en cours d'exécution"

(Même processus sur Node2 pour Pod2)
```

#### Étape 6 : État final
```
API Server → etcd : Met à jour l'état des pods (Running)
Vous : kubectl get pods
API Server : Retourne la liste des pods
Résultat : 3 pods nginx en état "Running" ✅
```

### Toute cette orchestration s'est passée en quelques secondes !

## La magie de la déclaration d'état

Un concept clé de Kubernetes que nous venons d'illustrer :

**Vous ne dites pas à Kubernetes COMMENT faire les choses**
```bash
# Vous ne dites PAS :
# "Démarre un conteneur nginx sur le nœud 1"
# "Puis démarre-en un autre sur le nœud 2"
# "Surveille-les et redémarre-les s'ils crashent"
```

**Vous dites à Kubernetes CE QUE vous voulez**
```bash
# Vous dites :
"Je veux 3 réplicas de nginx"
```

**Kubernetes se débrouille pour y arriver et maintenir cet état.**

C'est ce qu'on appelle le **paradigme déclaratif** :
- Vous déclarez l'état désiré
- Kubernetes fait converger l'état actuel vers l'état désiré
- En permanence et automatiquement

## Architecture single-node vs multi-node

### Single-Node (configuration MicroK8s par défaut)

Dans un cluster à un seul nœud, **tous les composants sont sur la même machine** :

```
┌────────────────────────────────────────────┐
│            SINGLE NODE                     │
│                                            │
│  ┌───────── CONTROL PLANE ────────┐        │
│  │ API Server | Scheduler | ...   │        │
│  │         etcd/Dqlite            │        │
│  └────────────────────────────────┘        │
│                                            │
│  ┌───────── WORKER COMPONENTS ────┐        │
│  │ Kubelet | Kube-Proxy           │        │
│  │ Container Runtime              │        │
│  │ [Vos pods tournent ici]        │        │
│  └────────────────────────────────┘        │
└────────────────────────────────────────────┘
```

**Avantages :**
- Simple à installer et gérer
- Peu de ressources requises
- Parfait pour développement et apprentissage
- Moins de complexité réseau

**Limitations :**
- Pas de haute disponibilité
- Si le nœud tombe, tout s'arrête
- Limité aux ressources d'une seule machine

### Multi-Node (évolution possible)

Dans un cluster multi-nœuds, le control plane et les workers sont séparés :

```
┌────── CONTROL PLANE NODE(S) ──────┐
│  API Server | Scheduler | ...     │
│         etcd/Dqlite               │
└───────────────┬───────────────────┘
                │
        ┌───────┴────────┐
        │                │
┌───────▼──────┐  ┌──────▼────────┐
│ WORKER NODE 1│  │ WORKER NODE 2 │
│              │  │               │
│ Kubelet      │  │ Kubelet       │
│ Kube-Proxy   │  │ Kube-Proxy    │
│ Container RT │  │ Container RT  │
│ [Pods]       │  │ [Pods]        │
└──────────────┘  └───────────────┘
```

**Avantages :**
- Haute disponibilité (si un nœud tombe, les autres continuent)
- Plus de ressources totales
- Séparation des responsabilités
- Meilleure répartition de la charge

**Complexité ajoutée :**
- Réseau plus complexe
- Plus de machines à gérer
- Configuration plus élaborée

## Communication entre composants

### Communication interne

Tous les composants communiquent via l'**API Server** :

```
┌─────────────────────────────────────────┐
│            API SERVER                   │
│    (Point central de communication)     │
└────┬─────┬────┬─────┬────┬────┬─────────┘
     │     │    │     │    │    │
     ↓     ↓    ↓     ↓    ↓    ↓
  Scheduler │  etcd  │  Kubelet │
      Controller     Kube-Proxy
           Manager
```

**Caractéristiques importantes :**
- Communication sécurisée (TLS)
- Authentification et autorisation
- Format standardisé (REST API, JSON/YAML)

### Ports réseau standard

Pour votre culture générale, voici les ports utilisés :

**Control Plane :**
- API Server : 6443 (HTTPS)
- etcd : 2379-2380
- Scheduler : 10251
- Controller Manager : 10252

**Worker Nodes :**
- Kubelet API : 10250
- NodePort Services : 30000-32767

**Note MicroK8s :** Ces ports sont gérés automatiquement, vous n'avez généralement pas besoin de vous en préoccuper !

## Spécificités de MicroK8s

MicroK8s simplifie cette architecture de plusieurs façons :

### 1. Tout-en-un
- Control plane et worker sur le même nœud par défaut
- Parfait pour commencer simple

### 2. Dqlite au lieu d'etcd
- Base de données plus légère
- Plus simple pour la haute disponibilité
- Basée sur SQLite

### 3. Installation intégrée
- Tous les composants installés d'un coup
- Pas de configuration manuelle de chaque composant
- Démarrage automatique

### 4. Addons préconfigurés
- Composants supplémentaires activables facilement
- Dashboard, DNS, Registry, etc.
- Installation et configuration automatiques

## Concepts clés à retenir

### 1. Separation of Concerns (Séparation des responsabilités)
Chaque composant a un rôle précis et ne fait que ce rôle. C'est ce qui rend Kubernetes robuste et maintenable.

### 2. État déclaratif
Vous déclarez ce que vous voulez, Kubernetes s'arrange pour y arriver.

### 3. Boucles de contrôle
Les contrôleurs surveillent constamment l'état et corrigent les écarts.

### 4. Communication centralisée
Tout passe par l'API Server, ce qui simplifie la sécurité et le monitoring.

### 5. Composants interchangeables
Vous pouvez remplacer certains composants (container runtime, réseau, etc.) selon vos besoins.

## Analogie complète : l'orchestre symphonique

Pour conclure, voici une analogie complète :

**Kubernetes = un orchestre symphonique**

- **API Server** = Le chef d'orchestre qui coordonne tout
- **etcd** = La partition musicale (la source de vérité)
- **Scheduler** = Le régisseur qui place les musiciens
- **Controllers** = Les chefs de section qui surveillent leur groupe
- **Kubelet** = Chaque musicien qui joue sa partition
- **Kube-Proxy** = L'acoustique de la salle qui distribue le son
- **Container Runtime** = Les instruments de musique
- **Vos applications** = La musique produite

Vous (l'utilisateur) donnez juste la partition (vos manifestes YAML), et l'orchestre se coordonne automatiquement pour jouer la symphonie !

## Conclusion

L'architecture de Kubernetes peut sembler complexe avec tous ces composants, mais chacun a un rôle simple et bien défini. Ensemble, ils créent un système robuste, flexible et auto-réparateur.

**Les points essentiels :**
- **Control Plane** : Le cerveau qui prend les décisions
- **Worker Nodes** : Les muscles qui exécutent le travail
- **API Server** : Le point central de communication
- **Paradigme déclaratif** : Vous déclarez ce que vous voulez, Kubernetes s'en occupe
- **MicroK8s** : Simplifie tout cela pour vous

Dans la prochaine section, nous explorerons les spécificités de MicroK8s, notamment son système d'addons et Dqlite, qui rendent Kubernetes encore plus accessible !

---

**Points clés à retenir :**
- Architecture en deux parties : Control Plane (cerveau) + Worker Nodes (muscles)
- Control Plane : API Server, etcd/Dqlite, Scheduler, Controller Manager
- Worker Nodes : Kubelet, Kube-Proxy, Container Runtime
- Communication centralisée via l'API Server
- Paradigme déclaratif : déclarez l'état désiré, Kubernetes y arrive
- Boucles de contrôle : surveillance et correction continues
- MicroK8s simplifie cette architecture pour la rendre accessible
- Single-node parfait pour débuter, évolutif vers multi-node

⏭️ [Spécificités MicroK8s : addons, Dqlite et simplicité](/01-introduction-a-kubernetes-et-microk8s/06-specificites-microk8s-addons-dqlite-et-simplicite.md)
