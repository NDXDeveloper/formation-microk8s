🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 7.1 Concepts de base du réseau Kubernetes

## Introduction

Le réseau dans Kubernetes peut sembler complexe au premier abord, mais il repose sur des principes simples et élégants. Dans ce chapitre, nous allons explorer les concepts fondamentaux qui permettent à vos applications de communiquer entre elles et avec le monde extérieur.

Avant de plonger dans les détails techniques, commençons par une analogie simple pour comprendre pourquoi le réseau Kubernetes est différent des réseaux traditionnels.

## Analogie : La ville et ses quartiers

Imaginez Kubernetes comme une ville moderne :

- **Les Pods** sont comme des immeubles d'habitation
- **Les Services** sont comme des bureaux de poste avec des adresses fixes
- **Le réseau Kubernetes** est comme le système postal et routier de la ville
- **Les Nodes** (serveurs) sont comme des quartiers de la ville

Dans une ville traditionnelle, si vous déménagez, votre adresse change. Mais dans Kubernetes, même si un Pod "déménage" (est recréé), le Service garde la même adresse, comme un bureau de poste permanent qui redirige toujours le courrier vers votre nouvelle adresse.

## Les quatre règles d'or du réseau Kubernetes

Kubernetes suit quatre principes fondamentaux pour son réseau. Ces règles sont essentielles à comprendre :

### 1. Chaque Pod a sa propre adresse IP

Contrairement aux conteneurs Docker classiques qui partagent l'IP de l'hôte, **chaque Pod dans Kubernetes reçoit sa propre adresse IP unique**.

**Pourquoi c'est important ?**
- Simplification de la communication entre applications
- Pas de conflit de ports entre conteneurs
- Migration facile depuis des architectures traditionnelles (chaque application avait sa propre machine/IP)

**Exemple concret :**
```
Pod Nginx     : 10.1.1.5
Pod PostgreSQL: 10.1.1.8
Pod Redis     : 10.1.1.12
```

Chaque Pod peut être contacté directement via son IP, comme si c'était un serveur physique indépendant.

### 2. Les Pods peuvent communiquer entre eux sans NAT

**NAT (Network Address Translation)** est une technique qui modifie les adresses IP lors de la communication. Kubernetes évite cela entre les Pods.

**Qu'est-ce que cela signifie concrètement ?**

Quand le Pod A (10.1.1.5) envoie un message au Pod B (10.1.1.8) :
- Pod B voit que le message vient de **10.1.1.5** (l'IP réelle de Pod A)
- Il n'y a pas d'intermédiaire qui masque ou modifie l'adresse source
- C'est comme si deux personnes se parlaient directement, sans passer par un standardiste

**Avantages :**
- Communication simple et transparente
- Facilite le débogage (vous voyez les vraies adresses)
- Meilleure performance (pas de traduction d'adresse)

### 3. Les Nodes peuvent communiquer avec tous les Pods

Les **Nodes** (serveurs physiques ou virtuels qui hébergent vos Pods) peuvent communiquer directement avec n'importe quel Pod du cluster, sans NAT.

**Cas d'usage :**
- Monitoring : les outils de surveillance sur le Node peuvent interroger tous les Pods
- Logging : collecter les logs de tous les conteneurs
- Health checks : vérifier que les Pods fonctionnent correctement

### 4. Les Pods voient leur propre adresse IP de la même manière que les autres la voient

Cette règle peut sembler abstraite, mais elle est cruciale. Quand un Pod se demande "quelle est mon adresse IP ?", il obtient la même réponse que celle vue par les autres Pods.

**Pourquoi c'est important ?**

Dans certains systèmes réseau, votre adresse "vue de l'intérieur" peut être différente de celle "vue de l'extérieur" (à cause du NAT). Kubernetes élimine cette complexité.

## Le réseau plat : tous égaux, tous connectés

Ces quatre règles créent ce qu'on appelle un **réseau plat** (flat network). Imaginez un grand terrain de jeu où tous les enfants (Pods) peuvent se parler directement, sans avoir à passer par des adultes (NAT) pour traduire leurs messages.

### Schéma mental

```
┌────────────────────────────────────────────────────┐
│                  Cluster Kubernetes                │
│                                                    │
│  ┌──────────┐    ┌──────────┐    ┌──────────┐      │
│  │  Pod A   │────│  Pod B   │────│  Pod C   │      │
│  │10.1.1.5  │    │10.1.1.8  │    │10.1.1.12 │      │
│  └──────────┘    └──────────┘    └──────────┘      │
│       │               │                │           │
│       └───────────────┴────────────────┘           │
│              Communication directe                 │
│              (sans intermédiaire NAT)              │
└────────────────────────────────────────────────────┘
```

## Types d'adresses IP dans Kubernetes

Il existe trois types principaux d'adresses IP que vous rencontrerez :

### 1. Pod IP (Adresse de Pod)

- **Durée de vie :** Éphémère (temporaire)
- **Changement :** Change à chaque fois que le Pod est recréé
- **Utilisation :** Communication directe entre Pods (rare en pratique)
- **Exemple :** 10.1.1.5

**Analogie :** C'est comme un numéro de téléphone portable temporaire. Si vous cassez votre téléphone et en achetez un nouveau, vous aurez un nouveau numéro.

### 2. Service IP (ClusterIP)

- **Durée de vie :** Stable et permanent
- **Changement :** Ne change pas (sauf si vous supprimez le Service)
- **Utilisation :** Point d'entrée stable pour accéder à un groupe de Pods
- **Exemple :** 10.96.0.15

**Analogie :** C'est comme le numéro de téléphone d'une entreprise. Même si les employés (Pods) changent, le numéro de l'entreprise reste le même.

### 3. Node IP (Adresse du serveur)

- **Durée de vie :** Stable (liée au serveur physique/virtuel)
- **Changement :** Ne change pas tant que le serveur existe
- **Utilisation :** Accès depuis l'extérieur du cluster
- **Exemple :** 192.168.1.100 (IP de votre serveur)

## Pourquoi les Pods ont des IPs éphémères ?

Vous vous demandez peut-être : "Pourquoi les Pods ont-ils des IPs qui changent ?"

**Raisons techniques :**

1. **Scalabilité automatique :** Kubernetes crée et détruit des Pods constamment pour s'adapter à la charge
2. **Résilience :** Si un Pod plante, Kubernetes en crée un nouveau (avec une nouvelle IP)
3. **Mises à jour :** Lors d'une mise à jour, les anciens Pods sont remplacés par de nouveaux

**La solution : les Services**

C'est précisément pourquoi les **Services** existent ! Ils fournissent une adresse stable (ClusterIP) qui redirige automatiquement le trafic vers les Pods disponibles, peu importe combien de fois ils sont recréés.

## Les plages d'adresses (CIDR)

Kubernetes utilise des plages d'adresses IP pour organiser son réseau. Une plage est notée en **CIDR** (Classless Inter-Domain Routing).

### Qu'est-ce qu'un CIDR ?

Un CIDR ressemble à ceci : **10.1.0.0/16**

Décomposons :
- **10.1.0.0** : Adresse de base
- **/16** : Nombre de bits "fixes" (les autres bits peuvent varier)

**Traduction simple :**
- **/16** signifie que les deux premiers nombres (10.1) sont fixes
- Les deux derniers (0.0) peuvent varier de 0.0 à 255.255
- Cela donne **65 536 adresses possibles** (10.1.0.0 à 10.1.255.255)

### Les trois plages principales dans MicroK8s

MicroK8s configure automatiquement trois plages d'adresses :

#### 1. Pod Network (Réseau des Pods)
```
Plage par défaut : 10.1.0.0/16
Nombre d'adresses : ~65 000
Utilisation : IPs attribuées aux Pods
```

#### 2. Service Network (Réseau des Services)
```
Plage par défaut : 10.152.183.0/24
Nombre d'adresses : ~256
Utilisation : IPs attribuées aux Services (ClusterIP)
```

#### 3. Node Network (Réseau physique)
```
Plage : Celle de votre réseau local
Exemple : 192.168.1.0/24
Utilisation : IPs des serveurs physiques/virtuels
```

### Pourquoi des plages séparées ?

**Organisation et isolation :**
- Évite les conflits d'adresses
- Facilite le routage (Kubernetes sait où diriger chaque paquet)
- Simplifie la configuration du firewall

**Analogie :** C'est comme les codes postaux d'une ville :
- 10.1.x.x = quartier résidentiel (Pods)
- 10.152.x.x = quartier commercial (Services)
- 192.168.x.x = routes principales (Nodes)

## La communication Pod-à-Pod : comment ça marche ?

Voyons concrètement comment deux Pods communiquent.

### Scénario : Pod Frontend appelle Pod Backend

```
┌─────────────┐                           ┌─────────────┐
│   Frontend  │                           │   Backend   │
│   (Pod A)   │      1. Requête HTTP      │   (Pod B)   │
│  10.1.1.5   │──────────────────────────▶│  10.1.1.8   │
│             │                           │             │
│             │◀──────────────────────────│             │
│             │      2. Réponse HTTP      │             │
└─────────────┘                           └─────────────┘
```

**Étapes détaillées :**

1. **Le Frontend veut contacter le Backend**
   - Frontend connaît l'IP du Backend : 10.1.1.8

2. **Résolution du chemin réseau**
   - Le Pod Frontend envoie un paquet à 10.1.1.8
   - Le plugin réseau CNI (nous verrons ça plus tard) achemine le paquet

3. **Traversée du réseau**
   - Le paquet peut passer par un ou plusieurs Nodes
   - Mais pour les Pods, c'est transparent (communication directe)

4. **Réception par le Backend**
   - Le Backend reçoit le paquet
   - Il voit que l'expéditeur est 10.1.1.5 (pas d'IP masquée)

5. **Réponse**
   - Le Backend répond directement à 10.1.1.5

## Le rôle du CNI (Container Network Interface)

Le **CNI** est le plugin réseau qui implémente concrètement les règles de réseau Kubernetes.

### Qu'est-ce qu'un CNI ?

Le CNI est un **pilote réseau** pour Kubernetes, comme un pilote d'imprimante pour votre ordinateur.

**Responsabilités du CNI :**
- Attribuer une IP unique à chaque nouveau Pod
- Configurer les routes réseau pour que les Pods puissent communiquer
- Gérer les interfaces réseau virtuelles
- Implémenter les Network Policies (règles de sécurité réseau)

### CNI dans MicroK8s

MicroK8s utilise par défaut **Calico** comme CNI, qui est l'un des plugins les plus populaires et robustes.

**Autres CNI populaires (pour votre culture générale) :**
- Flannel : Simple et léger
- Weave : Chiffrement natif
- Cilium : Basé sur eBPF, très performant
- Canal : Combinaison de Flannel et Calico

**Bonne nouvelle :** Avec MicroK8s, vous n'avez rien à configurer ! Le CNI est installé et configuré automatiquement.

## Les interfaces réseau virtuelles

Chaque Pod dispose de sa propre interface réseau virtuelle, comme une carte réseau virtuelle.

### Dans le Pod

Quand vous exécutez `ifconfig` ou `ip addr` dans un Pod, vous voyez :

```
eth0: Interface réseau principale du Pod
lo:   Interface loopback (localhost - 127.0.0.1)
```

**eth0** est l'interface que le Pod utilise pour communiquer avec le monde extérieur.

### Sur le Node

Le Node (serveur) héberge de nombreux Pods, donc il a de nombreuses interfaces virtuelles :

```
eth0:           Interface physique du serveur
vethXXXXX:      Interface virtuelle pour Pod 1
vethYYYYY:      Interface virtuelle pour Pod 2
caliXXXXX:      Interface Calico (CNI) pour Pod 1
```

Ces interfaces virtuelles agissent comme des "câbles réseau virtuels" qui relient chaque Pod au réseau du cluster.

## Le DNS : le système de noms dans Kubernetes

Tout comme Internet utilise des noms de domaine (google.com) au lieu d'adresses IP, Kubernetes a son propre système DNS interne.

### Pourquoi le DNS est crucial ?

Rappelez-vous : les IPs des Pods changent constamment. Le DNS permet d'utiliser des **noms stables** au lieu des IPs.

### Le format des noms DNS Kubernetes

Dans Kubernetes, chaque Service reçoit automatiquement un nom DNS selon ce format :

```
<nom-du-service>.<namespace>.svc.cluster.local
```

**Exemple concret :**

Si vous avez un Service nommé `database` dans le namespace `production` :

```
Nom DNS complet : database.production.svc.cluster.local
Nom court (dans le même namespace) : database
IP du Service : 10.152.183.15 (automatique)
```

### Utilisation pratique

**Depuis votre application :**

Au lieu de coder en dur l'IP :
```python
# ❌ Mauvaise pratique (IP qui change)
db_host = "10.1.1.8"
```

Utilisez le nom DNS :
```python
# ✅ Bonne pratique (nom stable)
db_host = "database"  # Dans le même namespace
# ou
db_host = "database.production.svc.cluster.local"  # Nom complet
```

Kubernetes résoudra automatiquement `database` vers l'IP actuelle du Service, même si les Pods derrière changent.

## CoreDNS : le serveur DNS de Kubernetes

**CoreDNS** est le serveur DNS qui tourne dans votre cluster Kubernetes (activé par défaut dans MicroK8s avec l'addon DNS).

### Rôle de CoreDNS

CoreDNS maintient une base de données dynamique :
- Noms de tous les Services → IPs correspondantes
- Noms de tous les Pods (optionnel) → IPs correspondantes
- Mise à jour automatique quand Services/Pods changent

### Comment les Pods utilisent CoreDNS ?

Chaque Pod est automatiquement configuré pour utiliser CoreDNS :

**Fichier /etc/resolv.conf dans le Pod :**
```
nameserver 10.152.183.10
search default.svc.cluster.local svc.cluster.local cluster.local
```

- **nameserver** : IP du service CoreDNS
- **search** : Suffixes DNS ajoutés automatiquement

Quand votre application fait une requête DNS pour "database", le Pod interroge automatiquement CoreDNS.

## La communication Service-à-Pod

Les Services agissent comme des **load balancers** (répartiteurs de charge) internes.

### Scénario : Service avec 3 Pods backend

```
                    ┌──────────────┐
     Requête        │   Service    │
    ─────────────▶  │  backend     │
                    │10.152.183.20 │
                    └──────┬───────┘
                           │
              ┌────────────┼────────────┐
              │            │            │
              ▼            ▼            ▼
        ┌─────────┐  ┌─────────┐  ┌─────────┐
        │ Pod 1   │  │ Pod 2   │  │ Pod 3   │
        │10.1.1.5 │  │10.1.1.8 │  │10.1.1.12│
        └─────────┘  └─────────┘  └─────────┘
```

### Mécanisme de distribution

1. **Client envoie une requête** au Service (10.152.183.20)
2. **Le Service sélectionne un Pod** (round-robin par défaut)
3. **Redirection transparente** vers l'IP du Pod choisi
4. **Le Pod traite la requête** et répond
5. **La réponse passe par le Service** et revient au client

**Important :** Le client ne sait pas qu'il y a plusieurs Pods derrière. Il pense parler à une seule entité (le Service).

## Les ports dans Kubernetes

Les Pods et Services utilisent des ports, comme des portes d'entrée numérotées.

### Anatomie d'une adresse réseau complète

```
10.1.1.5:8080
   │      │
   │      └─ Port (la "porte" spécifique)
   └──────── Adresse IP (l'"immeuble")
```

### Ports dans un Service

Un Service peut exposer un ou plusieurs ports :

```yaml
Port du Service : 80  (port exposé par le Service)
    ↓
Target Port : 8080  (port du conteneur dans le Pod)
```

**Traduction :**
- Le Service écoute sur le port **80**
- Quand une requête arrive sur le port 80, elle est redirigée vers le port **8080** du Pod

**Avantage :** Vous pouvez standardiser les ports des Services (toujours 80 pour HTTP) même si vos applications écoutent sur des ports différents (8080, 3000, 5000, etc.).

## Isolation réseau : les Namespaces

Les **Namespaces** sont comme des appartements séparés dans un immeuble. Ils permettent d'isoler logiquement les ressources.

### Impact sur le réseau

**Par défaut :** Les Pods de différents namespaces **peuvent** communiquer entre eux.

```
Namespace "dev"                Namespace "prod"
┌─────────────┐               ┌─────────────┐
│   Pod A     │──────────────▶│   Pod B     │
│  10.1.1.5   │   Autorisé    │  10.1.2.8   │
└─────────────┘               └─────────────┘
```

**Avec Network Policies :** Vous pouvez bloquer ou autoriser des communications spécifiques (sujet avancé, nous verrons ça dans un chapitre ultérieur).

### DNS et Namespaces

Pour accéder à un Service dans un autre namespace :

```
Service dans le même namespace : database
Service dans un autre namespace : database.production
Nom complet : database.production.svc.cluster.local
```

## Le trafic externe : entrer et sortir du cluster

Jusqu'ici, nous avons parlé de la communication **à l'intérieur** du cluster. Mais comment le monde extérieur peut-il accéder à vos applications ?

### Trafic entrant (Ingress)

Le trafic qui vient de l'extérieur vers vos Pods passe par plusieurs couches :

```
Internet
   │
   ▼
Router / Firewall (votre box)
   │
   ▼
Node IP (192.168.1.100)
   │
   ▼
Service NodePort ou LoadBalancer
   │
   ▼
Service ClusterIP
   │
   ▼
Pods
```

**Nous explorerons en détail les différents types de Services (NodePort, LoadBalancer) et les Ingress dans les prochains chapitres.**

### Trafic sortant (Egress)

Quand un Pod veut accéder à Internet (par exemple, télécharger une dépendance), le trafic sort via le Node :

```
Pod (10.1.1.5)
   │
   ▼
Node (192.168.1.100)
   │
   ▼  NAT appliqué ici !
   │
   ▼
Internet
```

**Note :** Contrairement à la communication intra-cluster, le trafic vers Internet **passe par NAT**. Le serveur distant voit l'IP du Node, pas l'IP du Pod.

## Sécurité réseau : aperçu

Quelques concepts de sécurité à connaître (détails dans le chapitre Sécurité) :

### Network Policies

Les **Network Policies** sont comme des règles de firewall pour vos Pods :
- Autoriser/bloquer le trafic entrant (ingress)
- Autoriser/bloquer le trafic sortant (egress)
- Règles basées sur les labels, namespaces, ou IPs

### Principe du moindre privilège

**Bonne pratique :** Par défaut, tout est autorisé dans Kubernetes. En production, vous devriez :
1. Créer des Network Policies restrictives
2. N'autoriser que les communications nécessaires
3. Bloquer tout le reste

## Schéma récapitulatif : architecture réseau complète

Voyons maintenant une vue d'ensemble de tous les concepts ensemble :

```
┌──────────────────────────────────────────────────────────────┐
│                       Cluster Kubernetes                     │
│                                                              │
│  ┌─────────────────────────────────────────────────────────┐ │
│  │                 Namespace: production                   │ │
│  │                                                         │ │
│  │  ┌─────────────────┐          ┌─────────────────┐       │ │
│  │  │ Service: web    │          │ Service: api    │       │ │
│  │  │ 10.152.183.10   │          │ 10.152.183.11   │       │ │
│  │  │ DNS: web        │          │ DNS: api        │       │ │
│  │  └────────┬────────┘          └────────┬────────┘       │ │
│  │           │                            │                │ │
│  │     ┌─────┴─────┐                ┌─────┴───┐            │ │
│  │     │           │                │         │            │ │
│  │  ┌──▼───┐   ┌──▼───┐        ┌────▼─┐  ┌────▼─┐          │ │
│  │  │Pod 1 │   │Pod 2 │        │Pod 3 │  │Pod 4 │          │ │
│  │  │10.1  │   │10.1  │        │10.1  │  │10.1  │          │ │
│  │  │.1.10 │   │.1.11 │        │.1.20 │  │.1.21 │          │ │
│  │  └──────┘   └──────┘        └──────┘  └──────┘          │ │
│  │                                                         │ │
│  └─────────────────────────────────────────────────────────┘ │
│                                                              │
│  ┌──────────────────────────────────────┐                    │
│  │     CoreDNS (Service DNS)            │                    │
│  │     10.152.183.10                    │                    │
│  └──────────────────────────────────────┘                    │
│                                                              │
│  Infrastructure réseau : CNI (Calico)                        │
│                                                              │
└────────────────────────────────┬─────────────────────────────┘
                                 │
                    ┌────────────┴────────────┐
                    │   Node (Serveur)        │
                    │   192.168.1.100         │
                    └─────────────────────────┘
                                 │
                                 ▼
                            Internet
```

## Points clés à retenir

Avant de passer à la suite, assurez-vous de bien comprendre ces concepts fondamentaux :

### ✅ Les quatre règles d'or
1. Chaque Pod a sa propre IP unique
2. Communication Pod-à-Pod sans NAT
3. Les Nodes peuvent contacter tous les Pods
4. Les Pods voient leur IP de la même façon que les autres

### ✅ Les trois types d'IP
1. **Pod IP** : Éphémère, change à chaque recréation
2. **Service IP (ClusterIP)** : Stable, point d'entrée permanent
3. **Node IP** : IP du serveur physique/virtuel

### ✅ Le DNS Kubernetes
- Format : `<service>.<namespace>.svc.cluster.local`
- Géré par CoreDNS
- Toujours utiliser les noms DNS plutôt que les IPs

### ✅ Le CNI
- Plugin réseau qui implémente les règles Kubernetes
- MicroK8s utilise Calico par défaut
- Transparent pour l'utilisateur

### ✅ Les Services
- Load balancers internes
- Redistribuent le trafic vers les Pods backend
- Exposent une IP et un DNS stables

## Pourquoi ces concepts sont importants ?

Comprendre le réseau Kubernetes vous permettra de :

1. **Déboguer efficacement** : Quand une application ne peut pas en contacter une autre, vous saurez où chercher
2. **Concevoir de meilleures architectures** : Vous saurez comment organiser vos Services et Pods
3. **Optimiser les performances** : Vous comprendrez l'impact des choix réseau
4. **Sécuriser votre cluster** : Vous pourrez implémenter des Network Policies appropriées

## Prochaines étapes

Maintenant que vous maîtrisez les concepts fondamentaux, les prochains chapitres approfondiront :

- **7.2 Networking CNI** : Détails sur le fonctionnement du plugin réseau
- **7.3 DNS interne avec CoreDNS** : Configuration et utilisation avancée
- **7.4 Communication Pod-to-Pod et Service-to-Pod** : Exemples pratiques
- **7.5 Test de connectivité interne** : Outils et techniques de diagnostic

Le réseau Kubernetes peut sembler intimidant au début, mais avec ces fondations solides, vous êtes bien préparé pour explorer les aspects plus avancés !

---

**Note :** Ce chapitre introduit les concepts de base. Les configurations pratiques et les commandes seront abordées dans les chapitres suivants avec des exemples concrets dans MicroK8s.

⏭️ [Networking CNI (Container Network Interface)](/07-reseau-kubernetes-interne/02-networking-cni.md)
