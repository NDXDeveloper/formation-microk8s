🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 7.2 Networking CNI (Container Network Interface)

## Introduction

Dans le chapitre précédent, nous avons vu que Kubernetes suit quatre règles d'or pour son réseau. Mais qui implémente concrètement ces règles ? Comment Kubernetes assigne-t-il des adresses IP aux Pods ? Comment route-t-il le trafic entre eux ? C'est précisément le rôle du **CNI** (Container Network Interface).

Dans ce chapitre, nous allons démystifier le CNI et comprendre comment il fait fonctionner le réseau Kubernetes en coulisses.

## Qu'est-ce que le CNI ?

### Définition simple

Le **CNI** (Container Network Interface) est un **plugin réseau** qui se charge de toute la "plomberie" réseau dans Kubernetes. C'est l'intermédiaire entre Kubernetes et le système réseau du serveur.

### Analogie : l'électricien de votre maison

Imaginez que vous construisez une maison (votre cluster Kubernetes) :

- **Kubernetes** est l'architecte qui définit les plans : "Je veux que chaque pièce (Pod) ait l'électricité (réseau) et qu'elles puissent toutes communiquer"
- **Le CNI** est l'électricien qui réalise concrètement l'installation : il tire les câbles, pose les prises, configure le tableau électrique
- **Vous** (l'utilisateur) utilisez simplement les prises sans vous soucier du câblage derrière les murs

Vous n'avez pas besoin de comprendre l'électricité pour allumer une lampe. De même, vous n'avez pas besoin de comprendre tous les détails du CNI pour déployer vos applications. Mais connaître les bases vous aidera à comprendre comment tout fonctionne et à résoudre les problèmes.

## Pourquoi le CNI existe-t-il ?

### Le problème à résoudre

Kubernetes est conçu pour fonctionner sur différentes plateformes :
- Serveurs physiques
- Machines virtuelles
- Cloud public (AWS, Azure, GCP)
- Cloud privé
- Environnements de développement locaux

Chacun de ces environnements a ses propres particularités réseau. Au lieu d'écrire du code réseau spécifique pour chaque plateforme, Kubernetes a créé une **interface standard** : le CNI.

### La solution : une interface standardisée

Le CNI définit un contrat :

```
Kubernetes dit : "J'ai besoin de créer un réseau pour ce Pod"
Le CNI répond : "D'accord, je m'en occupe avec ma propre méthode"
```

Peu importe comment le CNI implémente le réseau en interne, tant qu'il respecte les règles de Kubernetes.

**Avantages :**
- **Flexibilité** : Vous pouvez choisir le CNI adapté à vos besoins
- **Portabilité** : Kubernetes fonctionne partout grâce à cette abstraction
- **Innovation** : Les développeurs de CNI peuvent innover sans modifier Kubernetes

## Le cycle de vie réseau d'un Pod

Pour vraiment comprendre le CNI, suivons le voyage d'un Pod depuis sa création jusqu'à sa connexion au réseau.

### Étape 1 : Création du Pod

```
Utilisateur : kubectl create -f mon-pod.yaml
    ↓
Kubernetes : "Je dois créer un Pod"
```

### Étape 2 : Assignation à un Node

```
Scheduler : "Je vais placer ce Pod sur le Node 'node-1'"
    ↓
Kubelet (sur node-1) : "Je reçois l'ordre de créer ce Pod"
```

### Étape 3 : Création du conteneur

```
Kubelet : "Je crée le conteneur via le runtime (containerd, docker, etc.)"
    ↓
Conteneur créé, mais PAS ENCORE connecté au réseau
```

### Étape 4 : Appel au CNI (le moment clé)

```
Kubelet : "CNI, j'ai besoin que tu configures le réseau pour ce Pod"
    ↓
CNI : "OK, je m'en charge !"
```

### Étape 5 : Configuration réseau par le CNI

Le CNI effectue plusieurs opérations (nous verrons les détails plus tard) :

1. **Créer une interface réseau virtuelle** dans le Pod
2. **Assigner une adresse IP** au Pod depuis le pool disponible
3. **Configurer les routes réseau** pour que le Pod puisse communiquer
4. **Enregistrer l'IP** dans la base de données du CNI

### Étape 6 : Pod opérationnel

```
CNI : "C'est fait ! Le Pod a l'IP 10.1.1.42"
    ↓
Kubelet : "Parfait, le Pod est prêt"
    ↓
Pod : Opérationnel et connecté au réseau du cluster
```

### Schéma récapitulatif

```
┌──────────────┐
│  Utilisateur │
└──────┬───────┘
       │ kubectl create
       ▼
┌──────────────┐
│  API Server  │
└──────┬───────┘
       │
       ▼
┌──────────────┐
│  Scheduler   │ ─────▶ Assigne à un Node
└──────┬───────┘
       │
       ▼
┌──────────────┐
│   Kubelet    │ ─────▶ Crée le conteneur
└──────┬───────┘
       │
       ▼
┌──────────────┐
│     CNI      │ ─────▶ Configure le réseau
└──────┬───────┘
       │
       ▼
┌──────────────┐
│  Pod prêt !  │
└──────────────┘
```

## Comment fonctionne concrètement le CNI ?

Le CNI est en réalité un ensemble de **programmes exécutables** (binaires) que Kubernetes appelle quand nécessaire.

### Les opérations CNI

Le CNI doit répondre à quatre opérations principales :

#### 1. ADD (Ajouter) - Créer le réseau pour un Pod

Quand un nouveau Pod est créé :

```bash
Kubelet appelle : CNI ADD
Paramètres : ID du conteneur, namespace réseau, configuration
Résultat : IP assignée, routes configurées
```

#### 2. DEL (Supprimer) - Nettoyer le réseau d'un Pod

Quand un Pod est détruit :

```bash
Kubelet appelle : CNI DEL
Paramètres : ID du conteneur
Résultat : IP libérée, interfaces supprimées
```

#### 3. CHECK (Vérifier) - Vérifier l'état du réseau

Périodiquement, Kubernetes vérifie que le réseau fonctionne :

```bash
Kubelet appelle : CNI CHECK
Résultat : OK ou Erreur
```

#### 4. VERSION (Version) - Obtenir la version du CNI

```bash
Requête : CNI VERSION
Résultat : Version du plugin (ex: 1.0.0)
```

### Où sont installés les plugins CNI ?

Sur chaque Node, les binaires CNI sont stockés dans un répertoire standard :

```
/opt/cni/bin/
```

Dans ce dossier, vous trouverez les exécutables de différents plugins :

```
/opt/cni/bin/
├── calico           ← Plugin Calico (utilisé par MicroK8s)
├── flannel          ← Plugin Flannel
├── bridge           ← Plugin bridge simple
├── host-local       ← Gestion d'IPs locales
├── loopback         ← Interface loopback
└── ...
```

### Configuration du CNI

La configuration du CNI se trouve dans :

```
/etc/cni/net.d/
```

C'est un fichier JSON qui indique à Kubernetes quel CNI utiliser et comment le configurer.

**Exemple de configuration (simplifié) :**

```json
{
  "cniVersion": "1.0.0",
  "name": "k8s-pod-network",
  "type": "calico",
  "ipam": {
    "type": "calico-ipam"
  }
}
```

**Décomposition :**
- **cniVersion** : Version de la spécification CNI utilisée
- **name** : Nom du réseau
- **type** : Quel plugin utiliser (ici Calico)
- **ipam** : IP Address Management - comment gérer les IPs

## Les composants clés du CNI

Un plugin CNI moderne (comme Calico) est composé de plusieurs parties :

### 1. Le plugin CNI principal

Le binaire que Kubelet appelle directement pour configurer le réseau d'un Pod.

**Localisation :** `/opt/cni/bin/calico`

### 2. IPAM (IP Address Management)

Le système de gestion des adresses IP.

**Responsabilités :**
- Maintenir un pool d'adresses IP disponibles
- Assigner une IP unique à chaque nouveau Pod
- Libérer les IPs quand les Pods sont supprimés
- Éviter les conflits d'adresses

**Analogie :** C'est comme le guichet qui distribue les numéros dans une salle d'attente. Il s'assure que chaque personne (Pod) reçoit un numéro unique et récupère les numéros quand les gens partent.

### 3. Le système de routage

Configure les routes réseau pour que les paquets trouvent leur chemin.

**Mécanismes possibles :**
- Routes Linux classiques (ip route)
- Overlay networks (tunnels VXLAN, etc.)
- BGP (Border Gateway Protocol)
- eBPF (Extended Berkeley Packet Filter)

### 4. Les agents/démons

Des processus qui tournent en permanence sur chaque Node pour gérer le réseau.

**Exemple avec Calico :**
- **calico-node** : Démon qui tourne sur chaque Node
- **calico-kube-controllers** : Contrôleurs centraux

## Calico : le CNI de MicroK8s

MicroK8s utilise **Calico** comme CNI par défaut. C'est l'un des CNI les plus populaires et robustes de l'écosystème Kubernetes.

### Pourquoi Calico ?

**Avantages de Calico :**

1. **Performance** : Pas d'overhead, utilise des routes Linux natives
2. **Scalabilité** : Supporte des milliers de Nodes
3. **Sécurité** : Network Policies natives et avancées
4. **Flexibilité** : Plusieurs modes de fonctionnement
5. **Maturité** : Utilisé en production depuis des années

### Architecture de Calico

Calico se compose de plusieurs éléments :

```
┌─────────────────────────────────────────────────┐
│                 Cluster Kubernetes              │
│                                                 │
│  ┌──────────────────────────────────────────┐   │
│  │          Calico Controllers              │   │
│  │   (Gestion centralisée des policies)     │   │
│  └──────────────────────────────────────────┘   │
│                                                 │
│  ┌─────────────┐          ┌─────────────┐       │
│  │   Node 1    │          │   Node 2    │       │
│  │             │          │             │       │
│  │ ┌─────────┐ │          │ ┌─────────┐ │       │
│  │ │ calico  │ │          │ │ calico  │ │       │
│  │ │  node   │ │◄────────►│ │  node   │ │       │
│  │ │ daemon  │ │   BGP    │ │ daemon  │ │       │
│  │ └─────────┘ │          │ └─────────┘ │       │
│  │             │          │             │       │
│  │  Pods...    │          │  Pods...    │       │
│  └─────────────┘          └─────────────┘       │
└─────────────────────────────────────────────────┘
```

### Les modes de fonctionnement de Calico

Calico peut fonctionner selon différents modes. Comprenons les deux principaux :

#### Mode 1 : IP-in-IP (Encapsulation)

Dans ce mode, Calico encapsule les paquets dans un tunnel.

**Comment ça marche :**

```
Pod A (Node 1) veut envoyer un paquet à Pod B (Node 2)

1. Paquet original : [IP Pod A → IP Pod B | Données]
2. Calico encapsule : [IP Node 1 → IP Node 2 | [IP Pod A → IP Pod B | Données]]
3. Le paquet traverse le réseau entre les Nodes
4. Calico décapsule : [IP Pod A → IP Pod B | Données]
5. Le paquet arrive à Pod B
```

**Avantages :**
- Fonctionne sur n'importe quel réseau
- Compatible avec des réseaux qui ne supportent pas le routage direct

**Inconvénients :**
- Petit overhead de performance (encapsulation/décapsulation)
- Légère augmentation de la taille des paquets

#### Mode 2 : Direct Routing (Routage direct)

Dans ce mode, Calico utilise le routage BGP natif, sans encapsulation.

**Comment ça marche :**

```
Pod A (Node 1) veut envoyer un paquet à Pod B (Node 2)

1. Paquet : [IP Pod A → IP Pod B | Données]
2. Routage direct via les tables de routage du Node
3. Le paquet arrive directement à Node 2
4. Routé directement vers Pod B
```

**Avantages :**
- Performances maximales (pas d'encapsulation)
- Utilise les mécanismes réseau natifs du kernel Linux

**Inconvénients :**
- Nécessite un réseau qui supporte le routage BGP ou des routes statiques
- Configuration réseau légèrement plus complexe

**MicroK8s utilise principalement le mode Direct Routing avec BGP** car c'est le plus performant pour les installations locales et de test.

### BGP : le protocole de routage de Calico

**BGP** (Border Gateway Protocol) est un protocole de routage utilisé sur Internet pour échanger des informations de routage entre réseaux.

#### BGP simplifié

Imaginez BGP comme un système de panneaux indicateurs sur l'autoroute :

```
Node 1 annonce : "Pour atteindre 10.1.1.0/24 (mes Pods), passez par moi (192.168.1.10)"
Node 2 annonce : "Pour atteindre 10.1.2.0/24 (mes Pods), passez par moi (192.168.1.11)"
```

Chaque Node apprend automatiquement comment atteindre les Pods des autres Nodes.

#### Peering BGP

Les Nodes Calico établissent des connexions BGP (appelées "peering") entre eux :

```
┌─────────┐                    ┌─────────┐
│ Node 1  │◄───── BGP ────────►│ Node 2  │
│         │   Échange de       │         │
│         │   routes réseau    │         │
└─────────┘                    └─────────┘
```

**Types de peering :**

1. **Full Mesh (Maillage complet)** : Chaque Node se connecte à tous les autres
   - Simple, fonctionne bien pour de petits clusters (<100 nodes)
   - C'est le mode par défaut de MicroK8s

2. **Route Reflectors** : Architecture hiérarchique pour de gros clusters
   - Pour des clusters de centaines ou milliers de Nodes

### Les composants Calico dans MicroK8s

Quand vous démarrez MicroK8s, plusieurs composants Calico sont automatiquement lancés :

#### 1. calico-node (DaemonSet)

Un Pod **calico-node** tourne sur **chaque Node** du cluster.

**Responsabilités :**
- Configurer les interfaces réseau pour les nouveaux Pods
- Établir les connexions BGP avec les autres Nodes
- Appliquer les Network Policies
- Maintenir les routes réseau

**Visualisation :**
```
┌────────────────────────────────┐
│           Node                 │
│                                │
│  ┌──────────────────────────┐  │
│  │   calico-node (daemon)   │  │
│  └──────────────────────────┘  │
│              │                 │
│  ┌───────────┼──────────────┐  │
│  │  Pod 1    │   Pod 2      │  │
│  │  10.1.1.5 │   10.1.1.6   │  │
│  └───────────┴──────────────┘  │
└────────────────────────────────┘
```

#### 2. calico-kube-controllers (Deployment)

Un Pod **calico-kube-controllers** tourne dans le cluster (généralement un seul).

**Responsabilités :**
- Surveiller les ressources Kubernetes (Pods, Services, etc.)
- Synchroniser l'état avec Calico
- Gérer l'allocation des blocs d'IPs
- Nettoyer les ressources obsolètes

#### 3. Felix

**Felix** est le composant principal qui tourne dans chaque Pod calico-node.

**Responsabilités :**
- Programmer les règles iptables/eBPF
- Gérer les interfaces réseau
- Maintenir les tables de routage
- Appliquer les politiques de sécurité

### Datastore de Calico : Kubernetes API

Calico doit stocker des informations :
- Quelles IPs sont assignées à quels Pods
- Configuration des Network Policies
- État des connexions BGP

**Dans MicroK8s, Calico utilise l'API Kubernetes comme datastore**, ce qui simplifie l'architecture (pas besoin d'etcd séparé).

## Les interfaces réseau créées par le CNI

Quand Calico configure le réseau d'un Pod, il crée plusieurs interfaces virtuelles. Voyons comment.

### Du point de vue du Pod

Depuis l'intérieur du Pod, vous voyez une interface réseau simple :

```bash
# Depuis l'intérieur d'un Pod
ip addr show

1: lo: <LOOPBACK,UP,LOWER_UP>
   inet 127.0.0.1/8

2: eth0@if42: <BROADCAST,MULTICAST,UP,LOWER_UP>
   inet 10.1.1.5/32
```

**Explication :**
- **lo (loopback)** : Interface locale (localhost)
- **eth0** : Interface réseau principale du Pod
  - Adresse IP : 10.1.1.5
  - Le `@if42` indique que c'est une paire virtuelle (veth pair)

### Du point de vue du Node

Sur le Node qui héberge ce Pod, vous voyez l'autre côté de la paire :

```bash
# Sur le Node
ip addr show

...
42: cali123456789@if2: <BROADCAST,MULTICAST,UP,LOWER_UP>
   link/ether aa:bb:cc:dd:ee:ff
```

**C'est une paire veth (virtual ethernet pair)** :
- Un bout est dans le Pod (eth0 - interface 2)
- L'autre bout est sur le Node (cali123456789 - interface 42)

### Schéma : Paire veth

```
┌─────────────────────────────────────────────────┐
│                     Node                        │
│                                                 │
│   ┌─────────────────────────────────────┐       │
│   │  Namespace réseau du Pod            │       │
│   │                                     │       │
│   │    ┌──────────┐                     │       │
│   │    │   Pod    │                     │       │
│   │    │          │                     │       │
│   │    │  eth0 ◄──┼──────┐              │       │
│   │    │10.1.1.5  │      │              │       │
│   │    └──────────┘      │              │       │
│   └──────────────────────┼──────────────┘       │
│                          │ veth pair            │
│                          │                      │
│        ┌─────────────────┘                      │
│        │                                        │
│        ▼                                        │
│   ┌──────────┐                                  │
│   │cali123456│◄─── Interface sur le Node        │
│   │          │                                  │
│   └────┬─────┘                                  │
│        │                                        │
│        │ Routes vers d'autres Pods/Nodes        │
│        ▼                                        │
│   ┌──────────┐                                  │
│   │  Réseau  │                                  │
│   │  Cluster │                                  │
│   └──────────┘                                  │
└─────────────────────────────────────────────────┘
```

### Pourquoi une paire veth ?

La paire veth est comme un **câble réseau virtuel** avec deux bouts :
- Un bout branché dans le Pod (eth0)
- Un bout branché dans le Node (caliXXX)

Tout ce qui entre par un bout sort par l'autre. C'est ainsi que le Pod communique avec le reste du cluster.

## Le routage avec Calico

Maintenant que nous comprenons les interfaces, voyons comment les paquets sont routés.

### Tables de routage sur le Node

Sur chaque Node, Calico configure des routes pour atteindre tous les Pods du cluster.

**Exemple de table de routage :**

```bash
# Sur Node 1
ip route show

# Route par défaut
default via 192.168.1.1 dev eth0

# Routes vers les Pods locaux (sur ce Node)
10.1.1.5 dev cali123456 scope link
10.1.1.6 dev cali789012 scope link

# Routes vers les Pods sur d'autres Nodes (via BGP)
10.1.2.0/24 via 192.168.1.11 dev eth0
10.1.3.0/24 via 192.168.1.12 dev eth0
```

**Décomposition :**

1. **Routes locales** (`10.1.1.5 dev cali123456`)
   - Pour atteindre un Pod local, utiliser son interface calico directement

2. **Routes distantes** (`10.1.2.0/24 via 192.168.1.11`)
   - Pour atteindre les Pods sur Node 2 (192.168.1.11), router via ce Node
   - Ces routes sont apprises via BGP

### Voyage d'un paquet : exemple concret

Suivons un paquet depuis Pod A (Node 1) vers Pod B (Node 2) :

```
┌─────────────────────────────────────────────────────────┐
│                       Cluster                           │
│                                                         │
│  ┌─────────────────────┐      ┌─────────────────────┐   │
│  │      Node 1         │      │      Node 2         │   │
│  │  192.168.1.10       │      │  192.168.1.11       │   │
│  │                     │      │                     │   │
│  │  ┌──────────┐       │      │  ┌──────────┐       │   │
│  │  │  Pod A   │       │      │  │  Pod B   │       │   │
│  │  │10.1.1.5  │       │      │  │10.1.2.8  │       │   │
│  │  └────┬─────┘       │      │  └────▲─────┘       │   │
│  │       │             │      │       │             │   │
│  │       │ 1. Paquet   │      │       │ 5. Paquet   │   │
│  │       ▼             │      │       │             │   │
│  │  ┌─────────┐        │      │  ┌─────────┐        │   │
│  │  │cali123  │        │      │  │cali456  │        │   │
│  │  └────┬────┘        │      │  └────▲────┘        │   │
│  │       │             │      │       │             │   │
│  │       │ 2. Lookup   │      │       │ 4. Routage  │   │
│  │       │    route    │      │       │    local    │   │
│  │       ▼             │      │       │             │   │
│  │  ┌─────────┐        │      │  ┌─────────┐        │   │
│  │  │ Routing │        │      │  │ Routing │        │   │
│  │  │  Table  │        │      │  │  Table  │        │   │
│  │  └────┬────┘        │      │  └────▲────┘        │   │
│  │       │             │      │       │             │   │
│  └───────┼─────────────┘      └───────┼─────────────┘   │
│          │                            │                 │
│          │ 3. Envoi vers Node 2       │                 │
│          └────────────────────────────┘                 │
│                 Réseau physique                         │
└─────────────────────────────────────────────────────────┘
```

**Étapes détaillées :**

1. **Pod A crée un paquet** destiné à Pod B (10.1.2.8)
   ```
   Source: 10.1.1.5 (Pod A)
   Destination: 10.1.2.8 (Pod B)
   ```

2. **Le paquet sort par eth0 du Pod** et arrive sur l'interface cali123 du Node 1

3. **Node 1 consulte sa table de routage**
   ```
   Destination 10.1.2.8 ?
   → Consulte la route : 10.1.2.0/24 via 192.168.1.11
   → Doit envoyer vers Node 2 (192.168.1.11)
   ```

4. **Le paquet traverse le réseau physique** entre Node 1 et Node 2

5. **Node 2 reçoit le paquet** et consulte sa table de routage
   ```
   Destination 10.1.2.8 ?
   → Route locale : 10.1.2.8 dev cali456
   → Envoyer via l'interface cali456
   ```

6. **Le paquet arrive sur cali456** et entre dans Pod B via son eth0

7. **Pod B reçoit le paquet !**

**Point important :** Notez que le paquet **n'est PAS encapsulé**. L'IP source reste 10.1.1.5 et la destination reste 10.1.2.8 tout au long du voyage. C'est le routage direct de Calico en action.

## IPAM : la gestion des adresses IP

Le plugin IPAM (IP Address Management) de Calico gère l'allocation des IPs aux Pods.

### Pools d'adresses IP

Calico divise la plage d'adresses Pods en **blocs** (IP blocks).

**Exemple avec la plage 10.1.0.0/16 :**

```
Plage totale : 10.1.0.0/16 (65 536 adresses)

Divisée en blocs de /26 (64 adresses par bloc) :
- Bloc 1 : 10.1.0.0/26   → assigné à Node 1 (64 IPs)
- Bloc 2 : 10.1.0.64/26  → assigné à Node 2 (64 IPs)
- Bloc 3 : 10.1.0.128/26 → assigné à Node 3 (64 IPs)
- ...
```

### Pourquoi des blocs ?

**Avantages :**

1. **Performance** : Chaque Node gère son propre bloc sans coordination centralisée
2. **Scalabilité** : Pas de goulot d'étranglement lors de l'allocation d'IPs
3. **Efficacité du routage** : Les routes BGP annoncent des blocs entiers, pas des IPs individuelles

### Allocation d'IP pour un nouveau Pod

Quand un nouveau Pod est créé sur Node 1 :

```
1. Kubelet demande au CNI : "Configure le réseau pour ce Pod"
2. CNI Calico consulte IPAM : "Donne-moi une IP disponible"
3. IPAM répond : "Voici 10.1.0.5 du bloc local"
4. CNI configure l'interface avec cette IP
5. IPAM marque l'IP comme utilisée
```

Si le bloc local est plein, IPAM demande un nouveau bloc au pool global.

### Libération d'IP lors de la suppression d'un Pod

```
1. Kubelet demande au CNI : "Supprime le réseau de ce Pod"
2. CNI Calico informe IPAM : "Libère l'IP 10.1.0.5"
3. IPAM marque l'IP comme disponible
4. L'IP peut être réutilisée pour un futur Pod
```

## Network Policies avec Calico

Un des grands avantages de Calico est son support natif et performant des **Network Policies**.

### Qu'est-ce qu'une Network Policy ?

Une Network Policy est une règle firewall pour Pods :
- Contrôler qui peut se connecter à un Pod (Ingress)
- Contrôler où un Pod peut se connecter (Egress)

### Comment Calico implémente les Network Policies

Calico utilise **iptables** ou **eBPF** pour appliquer les règles.

#### Avec iptables (mode par défaut)

```
┌──────────────────────────────────────┐
│             Node                     │
│                                      │
│  ┌────────────────────────────────┐  │
│  │   Felix (composant Calico)     │  │
│  │                                │  │
│  │   Network Policy définie       │  │
│  │          ▼                     │  │
│  │   Traduit en règles iptables   │  │
│  │          ▼                     │  │
│  │   Applique dans le kernel      │  │
│  └────────────────────────────────┘  │
│                                      │
│  Les paquets passent par iptables    │
│  qui accepte/rejette selon règles    │
└──────────────────────────────────────┘
```

**Exemple de règle générée :**
```bash
# Autoriser le trafic du Pod 10.1.1.5 vers le Pod 10.1.1.8 sur le port 80
iptables -A FORWARD -s 10.1.1.5 -d 10.1.1.8 -p tcp --dport 80 -j ACCEPT

# Bloquer tout le reste
iptables -A FORWARD -d 10.1.1.8 -j DROP
```

#### Avec eBPF (mode avancé, optionnel)

**eBPF** (extended Berkeley Packet Filter) est une technologie récente du kernel Linux qui permet d'exécuter du code directement dans le kernel, de manière sûre et performante.

**Avantages :**
- **Performances supérieures** (pas de traversée de la stack iptables)
- **Visibilité accrue** (métriques détaillées)
- **Scalabilité** (mieux pour de très gros clusters)

## Diagnostiquer le réseau Calico

Quelques commandes utiles pour comprendre et déboguer le réseau Calico.

### 1. Vérifier le statut de Calico

```bash
# Voir les Pods Calico
microk8s kubectl get pods -n kube-system -l k8s-app=calico-node

# Devrait afficher un calico-node par Node, tous en "Running"
```

### 2. Voir les routes BGP

```bash
# Depuis un Pod calico-node
microk8s kubectl exec -n kube-system <calico-node-pod> -- /bin/sh -c "ip route"
```

### 3. Voir les interfaces réseau

```bash
# Sur le Node
ip addr show | grep cali

# Affiche toutes les interfaces caliXXX (une par Pod sur ce Node)
```

### 4. Vérifier les blocs d'IPs

```bash
# Lister les blocs IPAM
microk8s kubectl get ippools -o wide

# Voir les détails d'un bloc
microk8s kubectl describe ippool default-ipv4-ippool
```

### 5. Tester la connectivité

```bash
# Créer un Pod de test
microk8s kubectl run test-pod --image=busybox --restart=Never -- sleep 3600

# Obtenir son IP
microk8s kubectl get pod test-pod -o wide

# Tester la connectivité depuis un autre Pod
microk8s kubectl exec <autre-pod> -- ping <ip-du-test-pod>
```

## Comparaison : Calico vs autres CNI

Pour votre culture générale, voyons comment Calico se compare à d'autres CNI populaires.

### Tableau comparatif

| Caractéristique | Calico | Flannel | Weave | Cilium |
|-----------------|--------|---------|-------|--------|
| **Performance** | ⭐⭐⭐⭐⭐ | ⭐⭐⭐ | ⭐⭐⭐ | ⭐⭐⭐⭐⭐ |
| **Network Policies** | ⭐⭐⭐⭐⭐ | ❌ | ⭐⭐⭐ | ⭐⭐⭐⭐⭐ |
| **Simplicité** | ⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐ | ⭐⭐⭐ |
| **Scalabilité** | ⭐⭐⭐⭐⭐ | ⭐⭐⭐ | ⭐⭐⭐ | ⭐⭐⭐⭐⭐ |
| **Maturité** | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐ | ⭐⭐⭐⭐ |

### Quand choisir quel CNI ?

**Calico** (choix de MicroK8s) :
- Production, lab personnel, ou test
- Besoin de Network Policies robustes
- Performance importante
- **Recommandé pour la plupart des cas d'usage**

**Flannel** :
- Simplicité maximale
- Pas besoin de Network Policies
- Petits clusters de développement

**Weave** :
- Besoin de chiffrement natif du réseau
- Multi-cloud avec réseaux complexes

**Cilium** :
- Besoin de performances maximales (eBPF natif)
- Observabilité réseau avancée
- Service Mesh intégré

## Points clés à retenir

### ✅ CNI : le chef d'orchestre réseau

Le CNI est le plugin qui implémente concrètement les règles réseau de Kubernetes :
- Attribution des IPs aux Pods
- Configuration des interfaces réseau
- Routage des paquets
- Application des Network Policies

### ✅ Calico dans MicroK8s

- **CNI par défaut** de MicroK8s
- Utilise **BGP** pour le routage
- Mode **Direct Routing** (pas d'encapsulation)
- Support complet des **Network Policies**

### ✅ Composants Calico

1. **calico-node** : Démon sur chaque Node
2. **calico-kube-controllers** : Contrôleurs centraux
3. **Felix** : Applique les règles réseau
4. **IPAM** : Gère les adresses IP

### ✅ Interfaces réseau

- **veth pair** : Câble virtuel entre Pod et Node
- **eth0** : Interface réseau du Pod
- **caliXXX** : Interface Calico sur le Node

### ✅ Routage

- Tables de routage configurées par Calico
- BGP pour propager les routes entre Nodes
- Routage direct sans NAT entre Pods

### ✅ IPAM

- Allocation d'IPs par blocs
- Gestion décentralisée pour la performance
- Libération automatique des IPs

## Pourquoi tout cela est important ?

Comprendre le CNI et Calico vous permet de :

1. **Comprendre ce qui se passe réellement** quand vous créez un Pod
2. **Déboguer les problèmes réseau** efficacement
3. **Optimiser les performances** de votre cluster
4. **Implémenter des Network Policies** en connaissance de cause
5. **Faire des choix architecturaux** éclairés

## Prochaines étapes

Maintenant que vous maîtrisez le CNI et Calico, les prochains chapitres approfondissent :

- **7.3 DNS interne avec CoreDNS** : Comment les Pods se trouvent par leur nom
- **7.4 Communication Pod-to-Pod et Service-to-Pod** : Exemples pratiques de communication
- **7.5 Test de connectivité interne** : Outils et techniques pour vérifier le réseau

Le CNI est le cœur du réseau Kubernetes. Avec ces connaissances, vous avez une compréhension solide de ce qui se passe "sous le capot" !

---

**Note :** Ce chapitre présente les concepts théoriques du CNI. Les configurations avancées et les Network Policies seront détaillées dans les chapitres dédiés à la sécurité réseau.

⏭️ [DNS interne avec CoreDNS](/07-reseau-kubernetes-interne/03-dns-interne-avec-coredns.md)
