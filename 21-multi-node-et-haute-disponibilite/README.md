🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 21. Multi-Node et Haute Disponibilité

## Introduction au Chapitre

Jusqu'à présent, vous avez exploré Kubernetes sur un **nœud unique** - une seule machine qui fait tourner tous les composants de votre cluster. C'est parfait pour apprendre, expérimenter et développer. Mais que se passe-t-il si :

- Votre serveur tombe en panne ? 💥
- Vous devez faire une mise à jour système ? 🔄
- Vous manquez de ressources (CPU, RAM) ? 📈
- Un disque dur lâche ? 💾
- Une application gourmande monopolise tout ? 🐷

Sur un cluster mono-node, **la réponse est la même : tout s'arrête**. Vos applications deviennent inaccessibles. Vos utilisateurs voient des pages d'erreur. Votre business est impacté.

C'est là qu'intervient le **multi-node** et la **haute disponibilité** (HA - High Availability).

## Qu'est-ce que le Multi-Node ?

### Définition Simple

Un cluster **multi-node** est composé de **plusieurs machines** (physiques ou virtuelles) qui travaillent ensemble comme un seul système unifié.

**Analogie du restaurant :**

**Restaurant mono-node :**
```
🏠 Restaurant avec 1 seul cuisinier
   ↓
   Si le cuisinier est malade → Restaurant fermé ❌
```

**Restaurant multi-node :**
```
🏠 Restaurant avec 3 cuisiniers
   ↓
   Si un cuisinier est malade → Les 2 autres continuent ✅
   Restaurant ouvert !
```

**Dans Kubernetes, c'est pareil :**
- Plusieurs machines (nœuds) = Plusieurs cuisiniers
- Si une machine tombe = Les autres continuent
- Vos applications restent accessibles ✅

### Architecture Visuelle

**Cluster Single-Node (actuel) :**
```
┌────────────────────────────┐
│        UNE SEULE MACHINE   │
│                            │
│  ┌──────────────────────┐  │
│  │   Control Plane      │  │
│  │   (Chef orchestre)   │  │
│  └──────────────────────┘  │
│                            │
│  ┌──────────────────────┐  │
│  │   Worker             │  │
│  │   (Vos applications) │  │
│  └──────────────────────┘  │
│                            │
│  ┌──────────────────────┐  │
│  │   Stockage           │  │
│  └──────────────────────┘  │
└────────────────────────────┘
    ⚠️ Point unique de défaillance
```

**Cluster Multi-Node (objectif) :**
```
┌──────────────┐  ┌──────────────┐  ┌──────────────┐
│   Node 1     │  │   Node 2     │  │   Node 3     │
│              │  │              │  │              │
│ ┌──────────┐ │  │ ┌──────────┐ │  │ ┌──────────┐ │
│ │ Control  │◄┼──┼►│ Control  │◄┼──┼►│ Control  │ │
│ │ Plane    │ │  │ │ Plane    │ │  │ │ Plane    │ │
│ └──────────┘ │  │ └──────────┘ │  │ └──────────┘ │
│              │  │              │  │              │
│ ┌──────────┐ │  │ ┌──────────┐ │  │ ┌──────────┐ │
│ │ Worker   │ │  │ │ Worker   │ │  │ │ Worker   │ │
│ │ App A    │ │  │ │ App B    │ │  │ │ App A    │ │
│ └──────────┘ │  │ └──────────┘ │  │ └──────────┘ │
└──────────────┘  └──────────────┘  └──────────────┘
         ✅ Résilience et redondance
```

**Avantages immédiats :**
- 🛡️ **Résilience** : Une machine tombe ? Le cluster continue
- 📊 **Performance** : Répartir la charge sur plusieurs machines
- 🔧 **Maintenance** : Mise à jour d'un nœud sans downtime
- 📈 **Scalabilité** : Ajouter des nœuds = plus de capacité

## Qu'est-ce que la Haute Disponibilité (HA) ?

### Définition

La **Haute Disponibilité** (High Availability - HA) est la capacité d'un système à rester **opérationnel et accessible** même en cas de panne d'un ou plusieurs composants.

**Objectif chiffré :**
- 99% = 7h20min d'indisponibilité par mois (pas terrible)
- 99.9% = 43 minutes par mois (bien)
- **99.95% = 21 minutes par mois (très bien)**
- 99.99% = 4 minutes par mois (excellent)
- 99.999% = 26 secondes par mois (exceptionnel)

### Les Trois Piliers de la HA

**1. Redondance**
Avoir plusieurs exemplaires de chaque composant critique.

```
Component unique ❌     Composants redondants ✅
     │                        │    │    │
     Pod A                  Pod A Pod A Pod A
     │                        │    │    │
     └─ Tombe → Service HS   Un tombe → 2 autres continuent
```

**2. Réplication**
Copier les données sur plusieurs machines.

```
Stockage unique ❌              Stockage répliqué ✅
┌──────────┐                  ┌──────┐ ┌──────┐ ┌──────┐
│ Données  │                  │Data  │ │Data  │ │Data  │
│ Volume   │                  │Copy 1│ │Copy 2│ │Copy 3│
└──────────┘                  └──────┘ └──────┘ └──────┘
Disque HS → Tout perdu        1 disque HS → 2 copies restent
```

**3. Failover Automatique**
Basculement automatique vers un composant de secours en cas de panne.

```
         Node 1                Node 2
         (Actif)               (Standby)
            │                     │
            ├─────────────────────┤
            │   Heartbeat         │
            ├─────────────────────┤
            │                     │
         Tombe ! ❌               │
                                  │
                          Prend le relais ✅
                          (nouveau actif)
```

### HA vs Backup : Quelle Différence ?

Beaucoup de débutants confondent **HA** et **Backup**. Voici la différence :

**Haute Disponibilité (HA) :**
- **Objectif** : Éviter les interruptions de service
- **Protection contre** : Pannes matérielles, crashes, surcharges
- **Temps de récupération** : Secondes à minutes
- **Exemple** : Node1 tombe → Pods redistribués automatiquement sur Node2

**Backup (Sauvegarde) :**
- **Objectif** : Protéger contre la perte de données
- **Protection contre** : Suppressions accidentelles, corruptions, catastrophes
- **Temps de récupération** : Minutes à heures
- **Exemple** : Suppression accidentelle de la base → Restauration depuis backup

**Les deux sont complémentaires !**
```
┌───────────────────────────────────────────┐
│         STRATÉGIE COMPLÈTE                │
├───────────────────────────────────────────┤
│                                           │
│  HA (Haute Disponibilité)                 │
│  └─► Cluster multi-node                   │
│      Réplication des pods                 │
│      Stockage distribué                   │
│      ↓                                    │
│      Protège contre : Pannes courantes    │
│      Downtime : ~0 secondes               │
│                                           │
│              +                            │
│                                           │
│  Backup (Sauvegardes)                     │
│  └─► Backups quotidiens                   │
│      Snapshots                            │
│      Réplication hors-site                │
│      ↓                                    │
│      Protège contre : Catastrophes        │
│      Downtime : Minutes à heures          │
│                                           │
│  = Cluster de production robuste ✅       │
└───────────────────────────────────────────┘
```

## Pourquoi Passer au Multi-Node ?

### Scénarios Où le Single-Node Pose Problème

**Scénario 1 : Panne Matérielle**
```
📅 Vendredi 23h - Votre serveur unique
🔥 La carte mère grille

Conséquence avec single-node :
- Tous les services down ❌
- Impossible de déployer des correctifs
- Vous devez commander une nouvelle carte, attendre la livraison...
- Downtime : Plusieurs jours 😱

Conséquence avec multi-node :
- Node1 tombe, mais Node2 et Node3 continuent ✅
- Les pods sont automatiquement redistribués
- Downtime : 0 seconde
- Vous réparez tranquillement Node1 plus tard
```

**Scénario 2 : Maintenance Système**
```
🔧 Vous devez installer des mises à jour de sécurité urgentes
📦 Nécessite un reboot du serveur

Avec single-node :
- Planifier un créneau de maintenance (ex: 2h-4h du matin)
- Prévenir les utilisateurs
- Stopper toutes les applications
- Rebooter
- Espérer que tout redémarre correctement
- Downtime : 15-30 minutes

Avec multi-node :
- Drain Node1 (évacuer les pods vers Node2/Node3)
- Mettre à jour et rebooter Node1
- Faire pareil avec Node2, puis Node3
- Downtime : 0 seconde ✅
```

**Scénario 3 : Charge Croissante**
```
📈 Votre application devient populaire
🚀 Le trafic augmente de 300%

Avec single-node :
- CPU/RAM saturés
- Applications ralentissent ou crashent
- Impossible d'ajouter plus de ressources (limite physique)
- Solution : Acheter un serveur plus gros (coûteux, downtime)

Avec multi-node :
- Ajouter simplement un nouveau nœud (Node4)
- Kubernetes répartit automatiquement la charge
- Scaling horizontal (+ de machines) vs vertical (machine + grosse)
- Coût optimisé, pas de downtime ✅
```

**Scénario 4 : Déploiement Raté**
```
🚢 Vous déployez une nouvelle version de l'app
🐛 Bug critique découvert en production

Avec single-node :
- Le mauvais pod crashe et redémarre en boucle
- Pendant ce temps, service inaccessible
- Stress maximal pour faire un rollback rapide

Avec multi-node + réplication :
- Vous avez 3 pods de l'ancienne version qui tournent
- Vous déployez 3 pods de la nouvelle version
- Si la nouvelle version crashe → L'ancienne continue de servir
- Rolling update progressif et sûr ✅
```

### Pour Qui le Multi-Node ?

**Vous DEVRIEZ passer au multi-node si :**
- ✅ Vous préparez une application pour la production
- ✅ Vous avez des utilisateurs qui comptent sur votre service
- ✅ Votre business dépend de la disponibilité (e-commerce, SaaS, API)
- ✅ Vous voulez pouvoir faire des maintenances sans downtime
- ✅ Vous voulez apprendre les vraies pratiques DevOps/SRE
- ✅ Vous avez 2-3 machines (ou plus) disponibles

**Le single-node reste OK pour :**
- ✅ Apprentissage et expérimentation
- ✅ Environnement de développement personnel
- ✅ Proof of concept (POC)
- ✅ Homelab sans enjeu de disponibilité
- ✅ Budget limité (une seule machine)

## Ce Que Vous Allez Apprendre

Ce chapitre 21 est structuré en **10 sections progressives** qui vous guideront de votre cluster single-node actuel vers un cluster multi-node hautement disponible, résilient et production-ready.

### Vue d'Ensemble du Chapitre

**Phase 1 : Comprendre (Sections 21.1 - 21.2)**
```
21.1 Architecture Multi-Node
     └─► Comprendre les rôles (control plane, worker, etcd)
         Schémas d'architecture
         Différence avec single-node

21.2 Haute Disponibilité avec Dqlite
     └─► Découvrir Dqlite (alternative à etcd)
         Consensus et quorum
         Comment MicroK8s gère la HA
```

**Phase 2 : Construire (Sections 21.3 - 21.5)**
```
21.3 Ajout de Nœuds à un Cluster MicroK8s
     └─► Préparer les machines
         Commandes join/add-node
         Vérifications

21.4 Configuration d'un Cluster HA
     └─► Former un cluster 3 nœuds control plane
         Tests de quorum
         Validation HA

21.5 Ajout de Workers Supplémentaires
     └─► Séparer control plane et workers
         Scaling horizontal
         Gestion des labels
```

**Phase 3 : Optimiser (Sections 21.6 - 21.7)**
```
21.6 Load Balancing Entre Nœuds
     └─► MetalLB pour IP externes
         HAProxy pour API server
         Répartition de charge

21.7 Stockage Distribué
     └─► Longhorn pour volumes persistants répliqués
         Snapshots et backups
         Résilience des données
```

**Phase 4 : Sécuriser (Sections 21.8 - 21.10)**
```
21.8 Backup du Control Plane
     └─► Sauvegarder Dqlite et l'état du cluster
         Velero pour backups automatiques
         Procédures de restauration

21.9 Stratégies de Redondance
     └─► Redondance à tous les niveaux
         Éliminer les SPOF (Single Points of Failure)
         Matrice de résilience

21.10 Tests de Failover
      └─► Valider que tout fonctionne réellement
          Chaos engineering
          Mesurer RTO et RPO
```

### Progression Pédagogique

```
┌────────────────────────────────────────────────┐
│            VOTRE PARCOURS D'APPRENTISSAGE      │
├────────────────────────────────────────────────┤
│                                                │
│  Vous êtes ici                                 │
│       ↓                                        │
│  ┌─────────────┐                               │
│  │ Single-Node │  Cluster 1 machine            │
│  └──────┬──────┘                               │
│         │                                      │
│  (21.1 - 21.2) Apprendre les concepts          │
│         │                                      │
│         ↓                                      │
│  ┌─────────────┐                               │
│  │ Multi-Node  │  Cluster 3 machines           │
│  │ (basique)   │  sans HA                      │
│  └──────┬──────┘                               │
│         │                                      │
│  (21.3 - 21.4) Former le cluster HA            │
│         │                                      │
│         ↓                                      │
│  ┌─────────────┐                               │
│  │ Cluster HA  │  3 control plane              │
│  │ (production)│  + workers                    │
│  └──────┬──────┘                               │
│         │                                      │
│  (21.5 - 21.7) Optimiser et sécuriser          │
│         │                                      │
│         ↓                                      │
│  ┌─────────────┐                               │
│  │ Cluster     │  Stockage distribué           │
│  │ Production  │  Load balancing               │
│  │ Complet     │  Backups auto                 │
│  └──────┬──────┘                               │
│         │                                      │
│  (21.8 - 21.10) Valider et tester              │
│         │                                      │
│         ↓                                      │
│  ┌─────────────┐                               │
│  │ Cluster     │  Résilient, testé             │
│  │ Production  │  et documenté ✅              │
│  │ Ready       │                               │
│  └─────────────┘                               │
│                                                │
└────────────────────────────────────────────────┘
```

## Prérequis Matériels

Pour suivre ce chapitre, vous aurez besoin de **plusieurs machines**. Voici les configurations recommandées selon votre situation :

### Configuration Minimale (Apprentissage)

**Pour tester les concepts :**
- 3 machines virtuelles (VMs)
- 2 GB RAM par VM
- 2 vCPU par VM
- 20 GB disque par VM
- Réseau local entre les VMs

**Exemples d'environnements :**
- VirtualBox / VMware sur votre PC
- Proxmox (hyperviseur open-source)
- Cloud : AWS EC2 t3.micro, Google Cloud e2-micro (attention aux coûts)

### Configuration Standard (Homelab)

**Pour un vrai cluster fonctionnel :**
- 3 mini-PCs ou Raspberry Pi 4
- 4 GB RAM par machine (8 GB recommandé)
- 4 cores CPU par machine
- 50-100 GB SSD par machine
- Switch gigabit
- Alimentations et câbles

**Budget estimé :** 600-900€ pour 3 mini-PCs d'occasion

### Configuration Production (Petite Entreprise)

**Pour un usage professionnel :**
- 5+ serveurs (3 control plane + 2+ workers)
- 16-32 GB RAM par serveur
- 8+ cores CPU par serveur
- SSD NVMe 250 GB+ par serveur
- Switch manageable 10GbE
- UPS (onduleur)
- Redondance réseau/alimentation

**Budget estimé :** 3000-8000€

### Ce Que Vous Pouvez Réutiliser

**Matériel que vous avez peut-être déjà :**
- ✅ Vieux laptops (retirez la batterie si gonflée)
- ✅ Mini-PCs type Intel NUC
- ✅ Raspberry Pi (3B+, 4, 5)
- ✅ Anciens serveurs
- ✅ Desktop PC avec VMs

**Astuce budget :** Commencez avec 3 machines modestes pour apprendre, puis investissez dans du matériel plus sérieux quand vous maîtrisez.

## Prérequis Logiciels et Connaissances

### Connaissances Requises

Avant de commencer ce chapitre, vous devriez être à l'aise avec :
- ✅ Utilisation de base de Kubernetes (Pods, Deployments, Services)
- ✅ Commandes kubectl
- ✅ Linux et ligne de commande (SSH, apt, systemctl)
- ✅ Concepts réseau de base (IP, ports, DNS)
- ✅ MicroK8s sur single-node (chapitres précédents)

**Si vous êtes bloqué sur ces concepts, revoyez les chapitres précédents du tutoriel !**

### Logiciels à Préparer

**Sur chaque machine du cluster :**
- Ubuntu 22.04 LTS ou 24.04 LTS (recommandé)
- Accès SSH configuré
- Utilisateur avec droits sudo
- IPs statiques configurées
- Pare-feu ouvert entre les nœuds

**Sur votre poste de travail :**
- Client SSH (terminal Linux/Mac, PuTTY sur Windows)
- kubectl (ou utilisation via microk8s kubectl)
- Éditeur de texte (nano, vim, VS Code)

## Comment Aborder Ce Chapitre

### Approche Recommandée

**1. Lecture complète d'abord**
Lisez tout le chapitre une première fois pour comprendre le "big picture" avant de commencer à manipuler.

**2. Environnement de test**
Créez d'abord votre cluster multi-node dans un environnement de test (VMs) avant de toucher à vos machines physiques.

**3. Progression méthodique**
Suivez les sections dans l'ordre :
- Ne sautez pas d'étapes
- Validez chaque section avant de passer à la suivante
- Prenez des notes sur ce qui fonctionne/ne fonctionne pas

**4. Documentation**
Documentez votre installation :
- IPs de chaque nœud
- Commandes utilisées
- Problèmes rencontrés et solutions
- Schéma de votre architecture

**5. Tests réguliers**
Après chaque section, testez que tout fonctionne encore avant d'avancer.

### Temps Estimé

**Par section :**
- 21.1 - 21.2 : 2-3 heures (lecture + compréhension)
- 21.3 - 21.4 : 4-6 heures (installation cluster HA)
- 21.5 - 21.7 : 6-8 heures (workers, load balancing, stockage)
- 21.8 - 21.10 : 4-6 heures (backups, redondance, tests)

**Total chapitre 21 : 16-23 heures**

Étalez sur plusieurs jours/semaines selon votre disponibilité.

### En Cas de Problème

**Checklist de dépannage rapide :**
1. Vérifier les logs : `journalctl -u snap.microk8s.daemon-kubelite`
2. Vérifier la connectivité réseau entre nœuds : `ping`, `telnet`
3. Vérifier le statut : `microk8s status`
4. Consulter la documentation officielle : https://microk8s.io/docs
5. Chercher sur les forums : MicroK8s Discourse, Stack Overflow
6. Revenir à un état stable (snapshot VM ou backup)

**N'hésitez pas à recommencer** : Avec des VMs, vous pouvez détruire et reconstruire rapidement pour repartir sur de bonnes bases.

## Objectifs du Chapitre

À la fin du chapitre 21, vous serez capable de :

✅ **Comprendre** l'architecture multi-node et les concepts de HA
✅ **Construire** un cluster MicroK8s 3+ nœuds avec haute disponibilité
✅ **Configurer** le control plane HA avec Dqlite
✅ **Ajouter** et gérer des workers supplémentaires
✅ **Implémenter** le load balancing avec MetalLB
✅ **Déployer** du stockage distribué avec Longhorn
✅ **Sauvegarder** et restaurer le control plane
✅ **Concevoir** des stratégies de redondance complètes
✅ **Tester** la résilience avec des tests de failover
✅ **Gérer** un cluster Kubernetes de production

**Compétences transférables :**
Les concepts appris ici (HA, redondance, failover) s'appliquent à tous les clusters Kubernetes (kubeadm, k3s, EKS, GKE, AKS) et même au-delà de Kubernetes (bases de données, systèmes distribués).

## État d'Esprit

Construire un cluster multi-node hautement disponible peut sembler intimidant au début. C'est normal ! Vous allez manipuler plusieurs machines, des concepts de systèmes distribués, de la réplication, du consensus...

**Quelques conseils :**

**1. Acceptez que ça ne marchera pas du premier coup**
Vous allez rencontrer des problèmes. C'est **normal** et **formateur**. Chaque erreur est une opportunité d'apprentissage.

**2. Allez-y progressivement**
Ne cherchez pas à tout faire en même temps. Cluster basique d'abord, puis HA, puis optimisations.

**3. Testez chaque étape**
Après chaque modification, vérifiez que ça fonctionne. N'accumulez pas les changements non testés.

**4. Documentez votre parcours**
Prenez des notes. Vous les consulterez souvent et elles aideront les futurs vous.

**5. Célébrez les victoires**
Quand votre cluster HA est opérationnel, c'est une vraie réussite ! Vous aurez acquis des compétences que beaucoup de professionnels n'ont pas.

## Prêt à Commencer ?

Vous avez maintenant une vision claire de ce qui vous attend. Le voyage vers un cluster Kubernetes de production commence maintenant !

**Dans la prochaine section (21.1 - Architecture Multi-Node), vous allez plonger dans les détails techniques de l'architecture distribuée et comprendre comment tous les composants travaillent ensemble.**

---

Prenez une grande inspiration, préparez vos machines, et c'est parti pour l'aventure du multi-node ! 🚀

*Note : Gardez cette introduction à portée de main. Elle vous servira de carte pour naviguer dans ce chapitre dense mais passionnant.*

⏭️ [Architecture multi-node](/21-multi-node-et-haute-disponibilite/01-architecture-multi-node.md)
