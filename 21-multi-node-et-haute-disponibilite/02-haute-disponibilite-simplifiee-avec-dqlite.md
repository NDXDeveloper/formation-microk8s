🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 21.2 Haute Disponibilité Simplifiée avec Dqlite

## Introduction

La **Haute Disponibilité** (High Availability ou HA en anglais) est un concept fondamental en informatique : garantir qu'un système reste accessible et fonctionnel même en cas de panne. Dans le contexte de Kubernetes, cela signifie que votre cluster continue de tourner même si un ou plusieurs serveurs tombent en panne.

Traditionnellement, mettre en place la haute disponibilité dans Kubernetes était une tâche complexe nécessitant de configurer **etcd** (la base de données distribuée) avec beaucoup de paramètres et de précautions. C'est là que **Dqlite** et MicroK8s changent la donne : ils rendent la haute disponibilité **simple et automatique**.

Dans ce chapitre, nous allons comprendre ce qu'est la haute disponibilité, comment Dqlite la simplifie, et pourquoi c'est un atout majeur de MicroK8s.

## Qu'est-ce que la Haute Disponibilité ?

### Définition Simple

La haute disponibilité, c'est la capacité d'un système à **continuer de fonctionner malgré des pannes**.

**Analogie du restaurant :**
Imaginez un restaurant avec :
- **Sans HA** : Un seul cuisinier. S'il tombe malade, le restaurant ferme.
- **Avec HA** : Trois cuisiniers. Si l'un tombe malade, les deux autres assurent le service.

Dans Kubernetes, c'est pareil : avec plusieurs nœuds, si l'un tombe, les autres continuent.

### Les Composants Critiques à Protéger

Dans un cluster Kubernetes, deux éléments sont absolument critiques :

1. **Le Control Plane** (le cerveau)
   - Sans lui, impossible de déployer ou modifier quoi que ce soit
   - Contient l'API Server, le Scheduler, les Controllers
   - **C'est là que Dqlite intervient !**

2. **Les Workloads** (vos applications)
   - Si elles ne tournent que sur un seul nœud, elles deviennent un point de défaillance unique
   - Kubernetes peut les redistribuer automatiquement entre plusieurs nœuds

### Les Types de Pannes

Voici les pannes qu'une architecture HA doit supporter :

**Pannes matérielles :**
- Disque dur qui lâche
- RAM défectueuse
- Panne de carte mère
- Coupure électrique locale

**Pannes logicielles :**
- Crash d'un processus système
- Kernel panic
- Bug dans un composant Kubernetes

**Pannes réseau :**
- Perte de connectivité
- Switch qui tombe
- Câble débranché

**Maintenance planifiée :**
- Mises à jour du système
- Redémarrage d'un serveur
- Remplacement de matériel

Avec un cluster HA correctement configuré, **aucune de ces situations ne stoppe votre cluster**.

## Le Problème avec etcd Traditionnel

### Qu'est-ce qu'etcd ?

**etcd** est une base de données distribuée utilisée par Kubernetes "classique" pour stocker :
- Configuration du cluster
- État de tous les objets (pods, services, deployments, etc.)
- Secrets et ConfigMaps
- Network policies
- Tout ce qui fait tourner le cluster

C'est le **cœur du control plane**. Sans etcd, pas de Kubernetes.

### La Complexité d'etcd en HA

Pour avoir etcd en haute disponibilité, il faut :

**1. Installer etcd sur chaque nœud control plane**
```bash
# Configuration manuelle complexe
# Certificats TLS pour chaque membre
# Fichiers de configuration YAML longs et techniques
```

**2. Configurer la réplication**
- Définir les "peers" (autres membres du cluster etcd)
- Configurer les endpoints
- Gérer les certificats pour la communication sécurisée

**3. Gérer le quorum**
- Comprendre les concepts de leader/follower
- Définir le nombre de nœuds (toujours impair : 3, 5, 7...)
- Gérer manuellement la perte de quorum

**4. Maintenir et monitorer**
- Backups réguliers
- Surveillance de la santé
- Défragmentation périodique
- Debugging de problèmes de synchronisation

**5. Ressources importantes**
- etcd consomme beaucoup de RAM
- Besoin de disques SSD rapides
- Latence réseau critique

Pour un administrateur Kubernetes expérimenté, c'est gérable. Pour un débutant ou pour un lab personnel, **c'est décourageant**.

### Exemple de Complexité

Voici à quoi ressemble une configuration etcd manuelle (extrait simplifié) :

```yaml
# Fichier de configuration etcd - COMPLEXE
name: 'node1'
data-dir: /var/lib/etcd
listen-peer-urls: https://192.168.1.10:2380
listen-client-urls: https://192.168.1.10:2379,https://127.0.0.1:2379
initial-advertise-peer-urls: https://192.168.1.10:2380
advertise-client-urls: https://192.168.1.10:2379
initial-cluster: node1=https://192.168.1.10:2380,node2=https://192.168.1.11:2380,node3=https://192.168.1.12:2380
initial-cluster-token: 'my-cluster'
initial-cluster-state: 'new'
client-transport-security:
  cert-file: /etc/kubernetes/pki/etcd/server.crt
  key-file: /etc/kubernetes/pki/etcd/server.key
  trusted-ca-file: /etc/kubernetes/pki/etcd/ca.crt
peer-transport-security:
  cert-file: /etc/kubernetes/pki/etcd/peer.crt
  key-file: /etc/kubernetes/pki/etcd/peer.key
  trusted-ca-file: /etc/kubernetes/pki/etcd/ca.crt
```

Et il faut répéter cela pour chaque nœud, en adaptant les IPs et les certificats ! 😰

## Dqlite : La Révolution de la Simplicité

### Qu'est-ce que Dqlite ?

**Dqlite** est une base de données distribuée développée par Canonical (les créateurs d'Ubuntu et de MicroK8s). Son objectif : offrir les mêmes garanties qu'etcd, mais en **beaucoup plus simple**.

**Caractéristiques principales :**
- Basé sur SQLite (d'où le nom "Distributed SQLite")
- Implémente le protocole Raft (comme etcd)
- Intégré nativement dans MicroK8s
- Configuration automatique
- Légèreté exceptionnelle

### L'Algorithme de Consensus : Raft

Sans entrer dans les détails techniques, Raft est un algorithme qui permet à plusieurs serveurs de se mettre d'accord sur un état commun. C'est ce qui garantit que tous les nœuds du control plane ont la même vision du cluster.

**Analogie du vote :**
Imaginez 3 personnes qui doivent décider où aller dîner :
- Chacun propose un restaurant
- Ils votent
- La majorité l'emporte
- Tous acceptent la décision

Raft fonctionne pareil : les nœuds votent pour chaque changement, et la majorité décide.

**Concept de quorum :**
Pour qu'une décision soit valide, il faut une **majorité** :
- 3 nœuds → besoin de 2 votes (quorum = 2)
- 5 nœuds → besoin de 3 votes (quorum = 3)
- 7 nœuds → besoin de 4 votes (quorum = 4)

C'est pourquoi on utilise toujours un **nombre impair** de nœuds.

### Dqlite vs etcd : La Comparaison

| Critère | etcd | Dqlite |
|---------|------|--------|
| **Installation** | Manuelle, complexe | Automatique |
| **Configuration** | Fichiers YAML complexes | Aucune configuration nécessaire |
| **Certificats** | Gestion manuelle TLS | Automatique |
| **Consommation RAM** | 500 MB - 2 GB | 100 MB - 500 MB |
| **Consommation CPU** | Moyenne | Faible |
| **Stockage** | Nécessite SSD rapide | Moins exigeant |
| **Backups** | Outils externes | Intégré |
| **Monitoring** | Prometheus exporters à installer | Intégré |
| **Scalabilité** | Jusqu'à 1000+ nœuds | Jusqu'à 20-50 nœuds |
| **Complexité opérationnelle** | Élevée | Très faible |
| **Idéal pour** | Grandes productions | Labs, PME, edge computing |

### Les Avantages de Dqlite pour Vous

**1. Zéro Configuration**

Quand vous ajoutez un nœud à votre cluster MicroK8s, Dqlite se configure **automatiquement**. Pas de fichier YAML à écrire, pas de certificats à générer manuellement.

```bash
# C'est aussi simple que ça !
microk8s add-node
# Dqlite se réplique automatiquement sur le nouveau nœud
```

**2. Légèreté**

Dqlite consomme 3 à 5 fois moins de ressources qu'etcd. Pour un lab sur des machines modestes (Raspberry Pi, vieux laptops, petites VMs), c'est un énorme avantage.

**3. Maintenance Simplifiée**

Vous n'avez pas à :
- Gérer manuellement les backups etcd
- Défragmenter la base
- Surveiller la santé avec des outils externes
- Debugger des problèmes de synchronisation complexes

Tout est **automatique**.

**4. Résistance aux Pannes**

Dqlite utilise le même protocole de consensus (Raft) qu'etcd, donc vous avez les **mêmes garanties de fiabilité** :
- Tolérance aux pannes (tant que vous avez le quorum)
- Cohérence des données
- Réplication automatique

**5. Sécurité Intégrée**

Les communications entre nœuds Dqlite sont **automatiquement chiffrées**. Avec etcd classique, vous devez configurer manuellement les certificats TLS.

## Comment Fonctionne la HA avec Dqlite ?

### Architecture d'un Cluster HA MicroK8s

Prenons un exemple de cluster à 3 nœuds :

```
┌─────────────────────────────────────────────────────────────────┐
│                     CLUSTER MICROK8S HA                         │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│   ┌──────────────┐      ┌──────────────┐      ┌──────────────┐  │
│   │   Node 1     │      │   Node 2     │      │   Node 3     │  │
│   │ (Leader)     │◄────►│ (Follower)   │◄────►│ (Follower)   │  │
│   │              │      │              │      │              │  │
│   │ API Server   │      │ API Server   │      │ API Server   │  │
│   │ Scheduler    │      │ Scheduler    │      │ Scheduler    │  │
│   │ Controllers  │      │ Controllers  │      │ Controllers  │  │
│   │ Dqlite DB    │      │ Dqlite DB    │      │ Dqlite DB    │  │
│   │ Kubelet      │      │ Kubelet      │      │ Kubelet      │  │
│   └──────────────┘      └──────────────┘      └──────────────┘  │
│          ▲                      ▲                      ▲        │
│          │                      │                      │        │
│          └──────────────────────┴──────────────────────┘        │
│                    Communication Raft                           │
└─────────────────────────────────────────────────────────────────┘
```

**Explication :**
- Chaque nœud a une copie de Dqlite
- Un nœud est élu **Leader** (celui qui reçoit les écritures)
- Les autres sont **Followers** (ils répliquent les données)
- Tous les nœuds peuvent servir les lectures
- Si le Leader tombe, un nouveau Leader est élu automatiquement en quelques secondes

### Le Processus de Réplication

Voici ce qui se passe quand vous créez un nouveau deployment :

```
1. kubectl apply -f deployment.yaml
   │
   ├──► API Server (peut être sur n'importe quel nœud)
        │
        ├──► Envoi de la requête au Leader Dqlite
             │
             ├──► Leader enregistre dans Dqlite local
             │
             ├──► Leader réplique vers les Followers
             │
             ├──► Attente du quorum (2 nœuds sur 3 = OK)
             │
             └──► Confirmation à kubectl ✓

2. Le Scheduler (actif sur tous les nœuds) voit le nouveau deployment
   │
   └──► Il décide sur quel nœud placer les pods

3. Les Kubelet (sur chaque nœud) reçoivent les instructions
   │
   └──► Ils démarrent les conteneurs
```

**Tout cela est transparent pour vous !** Vous tapez `kubectl apply`, et Dqlite s'occupe de tout le reste.

### Scénarios de Panne

Voyons comment le cluster réagit à différentes pannes :

#### Scénario 1 : Un Nœud Worker Tombe

```
Avant :
Node 1 (Control+Worker) : 3 pods
Node 2 (Control+Worker) : 3 pods  ← PANNE !
Node 3 (Control+Worker) : 3 pods

Après (automatiquement) :
Node 1 (Control+Worker) : 4-5 pods
Node 2 (Control+Worker) : HORS LIGNE
Node 3 (Control+Worker) : 4-5 pods
```

**Ce qui se passe :**
1. Kubernetes détecte que Node 2 ne répond plus (en ~30 secondes)
2. Les pods qui tournaient sur Node 2 sont marqués "Terminating"
3. Le Scheduler redémarre ces pods sur Node 1 et Node 3
4. Le control plane reste fonctionnel (quorum = 2/3 ✓)

**Vos applications restent accessibles !**

#### Scénario 2 : Le Nœud Leader Tombe

```
Avant :
Node 1 (Leader)    : Dqlite Leader
Node 2 (Follower)  : Dqlite Follower
Node 3 (Follower)  : Dqlite Follower

PANNE de Node 1 !

Après (en ~5 secondes) :
Node 1 (Leader)    : HORS LIGNE
Node 2 (Follower)  : Devient Leader ! ←
Node 3 (Follower)  : Reste Follower
```

**Ce qui se passe :**
1. Les Followers détectent que le Leader ne répond plus
2. Une nouvelle élection Raft est déclenchée
3. Un des Followers (Node 2) devient le nouveau Leader
4. Tout continue de fonctionner normalement

**Temps d'indisponibilité : 0 à 5 secondes** (le temps de l'élection)

#### Scénario 3 : Deux Nœuds Tombent (Perte de Quorum)

```
Avant :
Node 1 : OK
Node 2 : OK
Node 3 : OK

PANNE de Node 1 ET Node 2 !

Après :
Node 1 : HORS LIGNE
Node 2 : HORS LIGNE
Node 3 : Seul survivant (quorum perdu = 1/3 ✗)
```

**Ce qui se passe :**
1. Node 3 détecte qu'il n'a plus le quorum
2. Le control plane passe en **mode lecture seule**
3. Les pods existants continuent de tourner sur Node 3
4. **MAIS** : impossible de créer/modifier/supprimer des ressources

**Solution :** Réparer au moins un des nœuds tombés pour retrouver le quorum (2/3).

**C'est pourquoi on recommande toujours 3 nœuds minimum pour la HA !**

### Nombre de Nœuds et Tolérance aux Pannes

| Nombre de nœuds | Quorum nécessaire | Pannes tolérées | Recommandation |
|-----------------|-------------------|-----------------|----------------|
| 1 | 1 | 0 | Dev/Test uniquement |
| 2 | 2 | 0 | ❌ Ne JAMAIS faire (pire que 1 nœud) |
| **3** | 2 | **1** | ✅ Configuration HA minimale |
| 4 | 3 | 1 | ❌ Pas mieux que 3, coûte plus cher |
| **5** | 3 | **2** | ✅ HA renforcée pour production |
| 6 | 4 | 2 | ❌ Pas mieux que 5 |
| **7** | 4 | **3** | ✅ HA maximale |

**Règle d'or :** Toujours un nombre **impair** de nœuds.

**Pourquoi jamais 2 nœuds ?**
Avec 2 nœuds, il faut l'accord des 2 pour avoir le quorum. Si un tombe, le cluster est inutilisable (lecture seule). C'est pire qu'avoir un seul nœud !

## Les Garanties de Dqlite

### Cohérence des Données

Dqlite garantit la **cohérence forte** (strong consistency) :
- Tous les nœuds voient les mêmes données
- Pas de risque de "split brain" (deux parties du cluster qui divergent)
- Les écritures sont atomiques (tout ou rien)

**Exemple concret :**
Si vous créez un Secret, soit il est créé sur **tous** les nœuds (après réplication), soit sur aucun (en cas d'échec). Vous ne risquez jamais d'avoir des données incohérentes.

### Durabilité des Données

Les données écrites dans Dqlite sont **durables** :
- Écrites sur disque immédiatement
- Répliquées sur plusieurs nœuds
- Survivent aux redémarrages

**Même si tous les nœuds redémarrent en même temps**, vos données sont préservées.

### Isolation et Transactions

Dqlite supporte les **transactions ACID** :
- **A**tomicity : Tout ou rien
- **C**onsistency : Respect des règles
- **I**solation : Les transactions ne se marchent pas dessus
- **D**urability : Persistance garantie

C'est exactement ce qu'on attend d'une base de données critique.

## Monitoring de la Santé Dqlite

### Vérifier l'État du Cluster

MicroK8s fournit des commandes pour surveiller Dqlite :

```bash
# Voir l'état général du cluster
microk8s status

# Inspecter en profondeur (inclut Dqlite)
microk8s inspect
```

### Commandes de Diagnostic Dqlite

```bash
# Voir les membres du cluster Dqlite
microk8s kubectl get nodes -o wide

# Vérifier qui est le Leader
microk8s.kubectl get lease -n kube-system

# Logs de Dqlite (si nécessaire)
sudo journalctl -u snap.microk8s.daemon-k8s-dqlite
```

### Indicateurs de Santé

**Cluster sain :**
- Tous les nœuds affichent "Ready"
- Les commandes `kubectl` répondent rapidement
- Pas de messages d'erreur dans les logs

**Problèmes potentiels :**
- Un nœud en "NotReady" (mais cluster fonctionnel si quorum OK)
- Lenteurs sur les commandes `kubectl` (problème de réseau ?)
- Erreurs "quorum lost" (trop de nœuds down)

## Limitations de Dqlite

Soyons honnêtes : Dqlite n'est pas parfait pour tous les cas d'usage.

### Scalabilité Limitée

**Limites recommandées :**
- Maximum ~20-30 nœuds
- Pour des clusters plus grands, etcd reste préférable

**Raisons :**
- Dqlite n'est pas optimisé pour les très gros clusters
- Au-delà de 20-30 nœuds, etcd a de meilleures performances

**Verdict :** Pour un lab personnel, une startup, ou une PME, Dqlite est largement suffisant. Pour Google ou AWS, non. 😊

### Performances sur Charges Très Élevées

Si vous avez des milliers de pods qui se créent/détruisent chaque minute, etcd pourrait être plus performant.

**Mais honnêtement** : pour 99% des cas d'usage (y compris en production pour des PME), Dqlite tient largement la charge.

### Moins d'Outils Tiers

L'écosystème etcd a plus d'outils de monitoring, backup, et administration. Avec Dqlite, vous dépendez plus de MicroK8s lui-même.

**Mais** : MicroK8s fournit l'essentiel, et c'est intégré !

## Quand Utiliser Dqlite vs etcd ?

### Utilisez Dqlite (MicroK8s) si :

✅ Vous gérez un cluster de moins de 20 nœuds
✅ Vous voulez une solution simple et "zero-config"
✅ Vous avez des ressources limitées (RAM, CPU)
✅ Vous êtes débutant en Kubernetes
✅ Vous montez un lab personnel
✅ Vous êtes une PME avec des besoins standards
✅ Edge computing (clusters légers en périphérie)

### Utilisez etcd (Kubernetes classique) si :

✅ Vous gérez un cluster de 50+ nœuds
✅ Vous avez des équipes dédiées à l'infrastructure
✅ Vous avez besoin d'outils tiers spécifiques à etcd
✅ Vous êtes dans un environnement entreprise avec des contraintes strictes
✅ Vous connaissez déjà etcd et êtes à l'aise avec

## Dqlite dans les Clouds et Edge

### Edge Computing

Dqlite brille particulièrement dans les **scénarios edge** :
- Déploiements sur sites distants (magasins, usines, etc.)
- Ressources limitées
- Besoin de simplicité opérationnelle

**Exemple :** Une chaîne de magasins avec un mini-cluster Kubernetes (3 nœuds) dans chaque magasin pour faire tourner des applications locales. Avec Dqlite, c'est simple à déployer et maintenir.

### Cloud Provider

Certains cloud providers commencent à proposer MicroK8s comme option :
- Amazon EC2
- Google Compute Engine
- Azure VMs
- DigitalOcean Droplets

Vous pouvez monter un cluster HA en quelques minutes avec Dqlite.

## Récapitulatif : Pourquoi Dqlite Change la Donne

**Avant Dqlite (avec etcd) :**
1. Installer etcd manuellement sur chaque nœud
2. Générer des certificats TLS
3. Configurer les fichiers YAML complexes
4. Gérer la réplication manuellement
5. Monitorer avec des outils externes
6. Consommer beaucoup de ressources

**Temps de setup : plusieurs heures à plusieurs jours**
**Complexité : élevée**
**Maintenance : continue**

**Avec Dqlite (MicroK8s) :**
1. Installer MicroK8s
2. Ajouter des nœuds avec `microk8s add-node`

**Temps de setup : 10-15 minutes**
**Complexité : très faible**
**Maintenance : quasi automatique**

## Points Clés à Retenir

1. **Dqlite est une alternative légère à etcd**, spécialement conçue pour MicroK8s
2. **Configuration automatique** : pas de fichiers YAML complexes à écrire
3. **Haute disponibilité simplifiée** : ajouter un nœud suffit pour la réplication
4. **Protocole Raft** : mêmes garanties de cohérence qu'etcd
5. **Légèreté** : 3 à 5 fois moins de ressources qu'etcd
6. **Quorum** : toujours un nombre impair de nœuds (3, 5, 7)
7. **Tolérance aux pannes** : peut perdre (n-1)/2 nœuds
8. **Limites** : jusqu'à ~20-30 nœuds (largement suffisant pour la plupart des cas)
9. **Idéal pour** : labs, PME, edge computing, apprentissage

## Prochaines Étapes

Maintenant que vous comprenez **comment** Dqlite rend la haute disponibilité simple, nous allons passer à la pratique :

- **21.3** : Ajouter concrètement des nœuds avec `microk8s add-node`
- **21.4** : Configurer un cluster HA complet de 3 nœuds
- **21.5** : Tester la résilience et les scénarios de panne
- **21.6** : Optimiser et monitorer votre cluster HA

Vous allez voir qu'avec MicroK8s et Dqlite, mettre en place la haute disponibilité est **étonnamment simple**. Ce qui prenait des jours avec Kubernetes traditionnel se fait maintenant en quelques commandes !

---

**Félicitations !** Vous maîtrisez maintenant les concepts de haute disponibilité et comprenez pourquoi Dqlite est une révolution pour les clusters Kubernetes de petite à moyenne taille. La simplicité sans compromis sur la fiabilité ! 🚀

⏭️ [Ajout de nœuds avec microk8s add-node](/21-multi-node-et-haute-disponibilite/03-ajout-de-noeuds-avec-microk8s-add-node.md)
