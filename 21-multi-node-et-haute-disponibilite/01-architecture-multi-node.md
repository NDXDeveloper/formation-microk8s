🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 21.1 Architecture Multi-Node

## Introduction

Jusqu'à présent, nous avons travaillé avec MicroK8s en mode **single-node**, c'est-à-dire avec un seul serveur hébergeant l'ensemble du cluster Kubernetes. Cette configuration est parfaite pour l'apprentissage, le développement et les tests. Cependant, pour des environnements plus robustes ou pour comprendre le fonctionnement réel de Kubernetes en production, il est essentiel de découvrir l'architecture **multi-node**.

Dans ce chapitre, nous allons explorer les concepts fondamentaux d'un cluster Kubernetes composé de plusieurs machines, sans encore aborder les aspects techniques de configuration (qui viendront dans les sections suivantes).

## Qu'est-ce qu'une Architecture Multi-Node ?

Une architecture multi-node signifie simplement que votre cluster Kubernetes est réparti sur **plusieurs machines physiques ou virtuelles** au lieu d'une seule. Chaque machine dans le cluster est appelée un **nœud** (node en anglais).

### Analogie Simple

Imaginez une entreprise :
- **Single-node** : Une seule personne fait tout le travail (comptabilité, ventes, production, service client)
- **Multi-node** : Une équipe où chaque personne a un rôle spécifique et collabore avec les autres

Dans Kubernetes, c'est pareil : au lieu d'un seul serveur qui gère tout, vous avez plusieurs serveurs qui se répartissent le travail.

## Les Deux Types de Nœuds

Dans un cluster Kubernetes, il existe deux grands types de nœuds :

### 1. Control Plane Nodes (Nœuds de Contrôle)

Le **control plane** est le "cerveau" du cluster. Il prend toutes les décisions importantes :

**Responsabilités :**
- **API Server** : Point d'entrée pour toutes les commandes (quand vous tapez `kubectl`, vous parlez à l'API Server)
- **Scheduler** : Décide sur quel nœud déployer chaque pod
- **Controller Manager** : Surveille l'état du cluster et corrige les écarts (par exemple, si un pod crashe, il en relance un nouveau)
- **etcd ou Dqlite** : Base de données qui stocke toute la configuration du cluster

**MicroK8s et le Control Plane :**
MicroK8s utilise **Dqlite** au lieu d'etcd traditionnel. Dqlite est une version simplifiée et plus légère, parfaitement adaptée aux petits clusters. C'est l'un des grands avantages de MicroK8s : la simplicité !

### 2. Worker Nodes (Nœuds de Travail)

Les **worker nodes** sont les "muscles" du cluster. Ils exécutent vos applications.

**Responsabilités :**
- **Kubelet** : Agent qui tourne sur chaque worker, reçoit les instructions du control plane et les exécute
- **Container Runtime** : Exécute les conteneurs (Docker, containerd, etc.)
- **Kube-proxy** : Gère le réseau et les communications entre les pods

Les workers ne prennent pas de décisions, ils exécutent simplement ce que le control plane leur demande de faire.

## Topologies Courantes

### Configuration Minimale (Dev/Test)

```
┌────────────────────────────┐
│   Control Plane + Worker   │
│   (1 nœud - single-node)   │
└────────────────────────────┘
```

**Avantages :**
- Simple à installer
- Peu de ressources nécessaires
- Parfait pour apprendre

**Inconvénients :**
- Pas de haute disponibilité
- Si le nœud tombe, tout s'arrête

### Configuration Petite Production (3 nœuds)

```
┌──────────────┐  ┌──────────────┐  ┌──────────────┐
│Control Plane │  │Control Plane │  │Control Plane │
│   + Worker   │  │   + Worker   │  │   + Worker   │
│   (Node 1)   │  │   (Node 2)   │  │   (Node 3)   │
└──────────────┘  └──────────────┘  └──────────────┘
```

**Avantages :**
- Haute disponibilité du control plane (quorum de 3)
- Si un nœud tombe, le cluster continue de fonctionner
- Les workloads sont automatiquement redistribués

**Inconvénients :**
- Plus de ressources nécessaires
- Configuration légèrement plus complexe

**Note importante :** Avec MicroK8s, les nœuds jouent généralement les deux rôles (control plane + worker), ce qui simplifie l'architecture pour les petits labs.

### Configuration Production Séparée

```
┌──────────────┐  ┌──────────────┐  ┌──────────────┐
│Control Plane │  │Control Plane │  │Control Plane │
│   (Master)   │  │   (Master)   │  │   (Master)   │
└──────────────┘  └──────────────┘  └──────────────┘
        ↓                ↓                ↓
┌──────────────┐  ┌──────────────┐  ┌──────────────┐
│   Worker 1   │  │   Worker 2   │  │   Worker 3   │
└──────────────┘  └──────────────┘  └──────────────┘
```

**Avantages :**
- Séparation des responsabilités
- Control plane dédié (pas de charge applicative)
- Scalabilité : on peut ajouter autant de workers que nécessaire

**Inconvénients :**
- Plus de machines nécessaires
- Plus complexe à gérer

Cette configuration est typique des grandes productions, mais rarement nécessaire pour un lab personnel.

## Pourquoi Passer en Multi-Node ?

### 1. Apprentissage et Compréhension

**Comprendre Kubernetes en profondeur :**
- Voir comment les pods se déplacent entre nœuds
- Observer la répartition de charge
- Comprendre les mécanismes de haute disponibilité
- Expérimenter avec les pannes et la résilience

Travailler en multi-node vous rapproche de ce qui se passe réellement en production.

### 2. Haute Disponibilité (HA)

**Protection contre les pannes :**
Si vous avez un seul nœud et qu'il tombe (panne matérielle, mise à jour, problème réseau), **tout votre cluster est hors service**.

Avec 3 nœuds ou plus :
- Si un nœud tombe, les autres continuent de fonctionner
- Kubernetes redistribue automatiquement les pods sur les nœuds restants
- Vos applications restent accessibles

**Le concept de quorum :**
Pour le control plane, il faut un nombre impair de nœuds (3, 5, 7...) pour maintenir un "quorum" (majorité). Si vous avez 3 nœuds de control plane :
- 2 nœuds actifs = cluster fonctionnel ✅
- 1 seul nœud actif = cluster en lecture seule ⚠️

### 3. Performance et Capacité

**Répartition de la charge :**
- Plus de CPU disponible pour vos applications
- Plus de RAM pour exécuter plus de pods
- Meilleure utilisation des ressources

**Scalabilité horizontale :**
Au lieu d'acheter un serveur plus puissant (scale up), vous ajoutez plus de serveurs (scale out). C'est souvent plus économique et flexible.

### 4. Isolation et Organisation

**Séparation logique :**
Vous pouvez dédier certains nœuds à certains types d'applications :
- Nœuds pour les bases de données (avec stockage SSD rapide)
- Nœuds pour les applications web
- Nœuds pour les jobs de traitement

Cela se fait avec des **labels**, **taints** et **tolerations** (que nous verrons plus tard).

## Comment les Nœuds Communiquent

### Réseau du Cluster

Dans un cluster multi-node, tous les nœuds doivent pouvoir communiquer entre eux. C'est crucial !

**Composants réseau :**

1. **CNI (Container Network Interface)**
   - Calico, Flannel, Weave, ou autre
   - MicroK8s utilise par défaut un CNI simple et efficace
   - Assure que tous les pods peuvent se parler, peu importe sur quel nœud ils sont

2. **DNS Interne (CoreDNS)**
   - Résolution de noms entre les services
   - Fonctionne de manière transparente dans tout le cluster

3. **Service ClusterIP**
   - Adresse IP virtuelle stable pour chaque service
   - Le trafic est automatiquement routé vers les bons pods, même s'ils changent de nœud

### Flux de Communication Typique

```
┌─────────────────────────────────────────────────────────────┐
│                         Internet                            │
└────────────────────────────┬────────────────────────────────┘
                             │
                    ┌────────▼────────┐
                    │  Load Balancer  │
                    │   (MetalLB)     │
                    └────────┬────────┘
                             │
        ┌────────────────────┼────────────────────┐
        │                    │                    │
   ┌────▼─────┐        ┌────▼─────┐        ┌────▼─────┐
   │  Node 1  │────────│  Node 2  │────────│  Node 3  │
   │          │        │          │        │          │
   │ Pod A    │        │ Pod B    │        │ Pod C    │
   └──────────┘        └──────────┘        └──────────┘
```

**Explication :**
1. Une requête arrive de l'extérieur
2. Le Load Balancer la reçoit
3. Le Load Balancer la transmet à l'un des pods, peu importe son nœud
4. Les pods peuvent communiquer entre eux directement
5. Le réseau CNI s'occupe de tout le routage

## Dqlite : La Simplicité de MicroK8s

### Qu'est-ce que Dqlite ?

Dqlite est une base de données distribuée utilisée par MicroK8s pour stocker l'état du cluster. C'est une alternative à **etcd**, utilisé par les distributions Kubernetes classiques.

**Avantages de Dqlite :**
- **Léger** : Consomme moins de ressources qu'etcd
- **Simple** : Moins de configuration nécessaire
- **Intégré** : Pas besoin de l'installer séparément
- **Automatique** : La réplication est gérée automatiquement

**Comment ça fonctionne :**
Quand vous ajoutez un nœud à votre cluster MicroK8s, Dqlite se réplique automatiquement. Chaque nœud du control plane a une copie de la base de données, et elles restent synchronisées.

### Comparaison avec etcd

| Caractéristique | etcd (Kubernetes classique) | Dqlite (MicroK8s) |
|-----------------|-----------------------------|--------------------|
| Consommation RAM | Moyenne à élevée | Faible |
| Configuration | Manuelle et complexe | Automatique |
| Maintenance | Backups manuels | Simplifié |
| Idéal pour | Gros clusters production | Petits à moyens clusters, labs |

Pour un lab personnel ou un petit environnement de production (jusqu'à 10-20 nœuds), Dqlite est largement suffisant et bien plus simple.

## Considérations Réseau

### Adressage IP

Chaque nœud doit avoir :
- **Une adresse IP fixe** (pas de DHCP)
- **Visibilité réseau** sur tous les autres nœuds
- **Ports ouverts** pour la communication (nous verrons les détails plus tard)

**Exemple de plan réseau simple :**
```
Node 1 : 192.168.1.10
Node 2 : 192.168.1.11
Node 3 : 192.168.1.12
```

### Latence

**Recommandation :** Les nœuds doivent être sur le même réseau local (LAN) ou avoir une latence très faible (< 10ms). Des nœuds géographiquement distants peuvent causer des problèmes de synchronisation.

### Bande Passante

Les nœuds échangent constamment des informations :
- État des pods
- Métriques
- Logs
- Réplication de données

Une connexion gigabit (1 Gbps) est recommandée, surtout si vous utilisez du stockage distribué.

## Ressources Matérielles Recommandées

### Pour un Cluster de 3 Nœuds (Lab Personnel)

**Par nœud :**
- **CPU** : 2 cœurs minimum (4 recommandés)
- **RAM** : 4 GB minimum (8 GB recommandés)
- **Stockage** : 40 GB minimum (SSD recommandé)
- **Réseau** : 1 Gbps

**Total pour le cluster :**
- 6 à 12 cœurs CPU
- 12 à 24 GB de RAM
- 120 GB de stockage

Ces valeurs permettent de faire tourner plusieurs applications confortablement.

### Optimisation pour Machines Virtuelles

Si vous utilisez des VMs (sur Proxmox, VMware, VirtualBox...) :
- Activez la **virtualisation nested** si disponible
- Allouez des **CPU dédiés** plutôt que partagés
- Utilisez du **stockage SSD** pour les VMs
- Configurez un **réseau bridged** pour que les VMs se voient

## Scénarios d'Usage Multi-Node

### Scénario 1 : Environnement de Dev/Staging/Prod

Avoir plusieurs clusters pour séparer vos environnements :
- **Cluster Dev** : 1 nœud (expérimentation libre)
- **Cluster Staging** : 3 nœuds (tests de HA)
- **Cluster Prod** : 3+ nœuds (applications critiques)

### Scénario 2 : Lab d'Apprentissage

Cluster de 3 nœuds pour :
- Tester la haute disponibilité
- Simuler des pannes
- Observer la redistribution des pods
- Apprendre les concepts avancés (affinity, taints, etc.)

### Scénario 3 : Homelab avec Services

Héberger vos services personnels avec redondance :
- Serveur media (Plex, Jellyfin)
- Stockage cloud (Nextcloud)
- Outils de productivité (GitLab, Jira)
- Monitoring complet

Si un serveur redémarre (mise à jour), vos services restent accessibles.

## Limitations et Quand Rester en Single-Node

### Quand le Multi-Node n'est PAS Nécessaire

**Vous devriez rester en single-node si :**
- Vous débutez avec Kubernetes (apprentissage des bases)
- Vous n'avez qu'une seule machine disponible
- Vos applications ne sont pas critiques (dev uniquement)
- Vous manquez de ressources (RAM, CPU)

### Limitations de MicroK8s Multi-Node

MicroK8s est optimisé pour les **petits à moyens clusters**. Pour de très gros déploiements :
- Au-delà de 20-30 nœuds, envisagez des solutions comme kubeadm ou des distributions managées
- Dqlite a des limites de scalabilité par rapport à etcd
- Moins de tooling enterprise que des solutions comme Rancher ou OpenShift

**Mais pour 99% des labs et petites productions, MicroK8s est parfait !**

## Points Clés à Retenir

1. **Multi-node = plusieurs machines** dans votre cluster Kubernetes
2. **Deux types de nœuds** : Control Plane (cerveau) et Workers (muscles)
3. **Haute disponibilité** : minimum 3 nœuds pour tolérer une panne
4. **Dqlite** : la force de MicroK8s, simple et automatique
5. **Réseau local** : les nœuds doivent pouvoir se parler (LAN)
6. **Ressources** : prévoir au moins 2 CPU et 4 GB RAM par nœud
7. **Cas d'usage** : apprentissage, HA, performance, ou séparation des environnements

## Prochaines Étapes

Maintenant que vous comprenez **pourquoi** et **comment** fonctionne une architecture multi-node, nous allons dans les sections suivantes :

- **21.2** : Découvrir la simplicité de la haute disponibilité avec Dqlite
- **21.3** : Ajouter concrètement des nœuds avec la commande `microk8s add-node`
- **21.4** : Configurer un cluster HA complet de 3 nœuds ou plus
- **21.5** : Ajouter des workers supplémentaires pour scaler

L'architecture multi-node peut sembler complexe au premier abord, mais avec MicroK8s, c'est étonnamment simple. Vous allez voir qu'ajouter un nœud se fait en quelques commandes seulement !

---

**Félicitations !** Vous maîtrisez maintenant les concepts fondamentaux de l'architecture multi-node. Vous êtes prêt à passer à la pratique ! 🚀

⏭️ [Haute Disponibilité simplifiée avec Dqlite](/21-multi-node-et-haute-disponibilite/02-haute-disponibilite-simplifiee-avec-dqlite.md)
