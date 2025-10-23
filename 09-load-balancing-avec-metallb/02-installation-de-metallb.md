🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 9.2 Installation de MetalLB (microk8s enable metallb)

## Introduction

Maintenant que vous comprenez les concepts du load balancing, passons à la pratique ! L'installation de MetalLB sur MicroK8s est étonnamment simple grâce au système d'addons intégré. En quelques commandes, vous aurez un load balancer fonctionnel dans votre cluster.

## Qu'est-ce que MetalLB ?

**MetalLB** est une implémentation de load balancer pour Kubernetes conçue spécifiquement pour les environnements bare-metal (serveurs physiques, labs personnels, on-premise). Son nom vient de "Metal Load Balancer" - un load balancer pour le "métal nu".

### Pourquoi MetalLB ?

Dans les clouds publics (AWS, GCP, Azure), créer un Service de type `LoadBalancer` provisionne automatiquement un load balancer cloud. Dans votre lab personnel ou serveur dédié, ce mécanisme n'existe pas. MetalLB comble ce vide en :

- Assignant des adresses IP externes à vos Services
- Annonçant ces IP sur votre réseau local
- Gérant le trafic entrant vers votre cluster
- Offrant une expérience similaire aux environnements cloud

## Prérequis

Avant d'installer MetalLB, assurez-vous que :

### 1. MicroK8s est Installé et Fonctionnel

Vérifiez l'état de votre cluster :

```bash
microk8s status
```

Vous devriez voir :

```
microk8s is running
high-availability: no
```

### 2. Vous Disposez d'un Pool d'Adresses IP

MetalLB a besoin d'un pool d'adresses IP à distribuer. Ces adresses doivent :

- **Être sur le même réseau** que votre machine MicroK8s
- **Ne pas être utilisées** par d'autres équipements (DHCP, serveurs, etc.)
- **Être accessibles** depuis votre réseau local

**Exemple de configuration réseau** :

Si votre machine MicroK8s a l'IP `192.168.1.50` et que votre réseau est `192.168.1.0/24`, vous pourriez réserver :
- `192.168.1.200` à `192.168.1.250` pour MetalLB

**⚠️ Important** : Assurez-vous que ces IP ne sont pas dans le pool DHCP de votre routeur pour éviter les conflits.

### 3. DNS est Activé (Recommandé)

Le DNS interne doit être actif (normalement activé par défaut) :

```bash
microk8s status | grep dns
```

Si ce n'est pas le cas, activez-le :

```bash
microk8s enable dns
```

## Installation de MetalLB

### Étape 1 : Activer l'Addon MetalLB

MicroK8s simplifie l'installation avec son système d'addons. Une seule commande suffit :

```bash
microk8s enable metallb
```

Après avoir exécuté cette commande, MicroK8s vous demandera de spécifier votre pool d'adresses IP.

### Étape 2 : Définir le Pool d'Adresses IP

Vous verrez un message similaire à :

```
Enter each IP address range delimited by comma (e.g. '10.64.140.43-10.64.140.49,192.168.0.105-192.168.0.111'):
```

**Format accepté** :

Vous pouvez spécifier les adresses de plusieurs façons :

1. **Range (plage)** : `192.168.1.200-192.168.1.210`
2. **CIDR** : `192.168.1.200/29` (8 adresses utilisables)
3. **Adresse unique** : `192.168.1.200`
4. **Combinaison** : `192.168.1.200-192.168.1.205,192.168.1.220`

**Exemples concrets** :

Pour un lab personnel simple, une petite plage suffit :
```
192.168.1.200-192.168.1.210
```

Cela vous donne 11 adresses IP pour vos Services LoadBalancer.

### Étape 3 : Validation de l'Installation

Une fois la configuration terminée, vous verrez :

```
Enabling MetalLB
Applying Metallb manifest
namespace/metallb-system created
...
MetalLB is enabled
```

## Vérification de l'Installation

### Vérifier les Pods MetalLB

MetalLB déploie plusieurs composants dans un namespace dédié. Vérifions qu'ils fonctionnent :

```bash
microk8s kubectl get pods -n metallb-system
```

Vous devriez voir quelque chose comme :

```
NAME                          READY   STATUS    RESTARTS   AGE
controller-xxxxx              1/1     Running   0          2m
speaker-xxxxx                 1/1     Running   0          2m
```

**Composants de MetalLB** :

1. **Controller** : Le cerveau de MetalLB
   - Assigne les IP aux Services
   - Gère le pool d'adresses
   - Un seul pod (pas besoin de plus)

2. **Speaker** : L'annonceur réseau
   - Annonce les IP sur le réseau
   - Un pod par nœud du cluster (en DaemonSet)
   - Utilise des protocoles réseau (BGP ou Layer 2)

### Vérifier la Configuration

Pour voir la configuration du pool d'IP :

```bash
microk8s kubectl get ipaddresspool -n metallb-system
```

Ou pour voir toutes les ressources MetalLB :

```bash
microk8s kubectl get all -n metallb-system
```

### Vérifier l'Addon

Confirmez que MetalLB apparaît bien dans les addons activés :

```bash
microk8s status
```

Vous devriez voir `metallb: enabled` dans la liste.

## Architecture de MetalLB

Comprendre l'architecture aide à mieux utiliser MetalLB :

```
┌─────────────────────────────────────────┐
│         Namespace: metallb-system       │
│                                         │
│  ┌──────────────┐     ┌──────────────┐  │
│  │ Controller   │     │   Speaker    │  │
│  │              │     │  (DaemonSet) │  │
│  │ - Assigne IP │     │ - Annonce IP │  │
│  │ - Gère pool  │     │ - ARP/BGP    │  │
│  └──────────────┘     └──────────────┘  │
│                                         │
└─────────────────────────────────────────┘
              │
              ↓
┌─────────────────────────────────────────┐
│        Services LoadBalancer            │
│                                         │
│  Service A: 192.168.1.200               │
│  Service B: 192.168.1.201               │
│  Service C: 192.168.1.202               │
└─────────────────────────────────────────┘
```

## Modes de Fonctionnement de MetalLB

MetalLB peut fonctionner en deux modes pour annoncer les IP sur le réseau :

### Mode Layer 2 (Par Défaut)

**C'est le mode activé par défaut sur MicroK8s.** Il utilise le protocole ARP (Address Resolution Protocol) :

**Avantages** :
- Fonctionne sur n'importe quel réseau Ethernet
- Aucune configuration réseau supplémentaire nécessaire
- Parfait pour les labs et petits déploiements
- Simple à comprendre et à dépanner

**Fonctionnement** :
- Un nœud est élu "leader" pour chaque IP
- Ce nœud répond aux requêtes ARP pour cette IP
- Si le leader tombe, un autre nœud prend le relais

**Limitations** :
- Pas de réelle répartition de charge entre nœuds (tout passe par un seul nœud)
- La bascule (failover) peut prendre quelques secondes

### Mode BGP (Avancé)

Le mode BGP (Border Gateway Protocol) est plus avancé et nécessite une configuration réseau spécifique :

**Avantages** :
- Vraie répartition de charge entre les nœuds
- Failover instantané
- Scalabilité supérieure

**Inconvénients** :
- Nécessite un routeur compatible BGP
- Configuration plus complexe
- Pas nécessaire pour un lab personnel

**Pour un débutant** : Le mode Layer 2 (par défaut) est amplement suffisant et recommandé.

## Configuration du Pool d'Adresses IP

Lors de l'installation, vous avez défini un pool d'IP. Cette configuration est stockée dans une ressource Kubernetes appelée `IPAddressPool`.

### Visualiser le Pool Actuel

```bash
microk8s kubectl get ipaddresspool -n metallb-system -o yaml
```

Vous verrez une configuration similaire à :

```yaml
apiVersion: metallb.io/v1beta1
kind: IPAddressPool
metadata:
  name: default
  namespace: metallb-system
spec:
  addresses:
  - 192.168.1.200-192.168.1.210
```

### Comprendre la Configuration

- **name: default** : Nom du pool (peut avoir plusieurs pools)
- **addresses** : Liste des plages d'IP disponibles
- **namespace** : Toujours dans `metallb-system`

## Reconfiguration Post-Installation

Si vous devez modifier le pool d'IP après l'installation, vous avez plusieurs options :

### Option 1 : Désactiver et Réactiver l'Addon

La méthode la plus simple :

```bash
# Désactiver MetalLB
microk8s disable metallb

# Réactiver avec une nouvelle configuration
microk8s enable metallb
```

**⚠️ Attention** : Cela supprimera toutes les IP assignées aux Services existants.

### Option 2 : Modifier la Ressource Manuellement

Pour les utilisateurs plus avancés :

```bash
microk8s kubectl edit ipaddresspool default -n metallb-system
```

Cela ouvre l'éditeur avec la configuration actuelle que vous pouvez modifier.

## Considérations Réseau Importantes

### 1. Firewall

Assurez-vous que votre firewall autorise :
- Le trafic vers les IP du pool MetalLB
- Le trafic ARP (pour le mode Layer 2)
- Les ports des Services que vous exposez

### 2. Routeur et DHCP

**Réservation d'IP** : Pour éviter les conflits, configurez votre routeur pour :
- Exclure les IP du pool MetalLB du DHCP
- Ou utiliser des IP en dehors du range DHCP

**Exemple** : Si votre DHCP assigne `192.168.1.2` à `192.168.1.199`, utilisez `192.168.1.200-192.168.1.250` pour MetalLB.

### 3. Réseau Multi-Nœud

Si vous avez un cluster multi-nœud :
- Les IP du pool doivent être accessibles depuis tous les nœuds
- Tous les nœuds doivent être sur le même sous-réseau (pour Layer 2)
- Chaque nœud exécute un pod `speaker`

## Résolution de Problèmes Courants

### MetalLB ne Démarre Pas

**Vérifiez les logs du controller** :
```bash
microk8s kubectl logs -n metallb-system deployment/controller
```

**Vérifiez les logs des speakers** :
```bash
microk8s kubectl logs -n metallb-system daemonset/speaker
```

### Les IP ne Sont Pas Assignées

Si vos Services restent en `<pending>` :

1. **Vérifiez qu'il reste des IP disponibles** dans le pool
2. **Vérifiez la syntaxe** du pool d'adresses
3. **Consultez les logs** du controller pour des erreurs

### Conflit d'IP

Si une IP du pool est déjà utilisée sur le réseau :
- Vous verrez des conflits ARP
- Modifiez le pool pour exclure cette IP
- Utilisez `arp -a` pour voir les IP déjà utilisées

## Bonnes Pratiques

### 1. Planification du Pool d'IP

- **Réservez suffisamment d'adresses** : Comptez le nombre de Services LoadBalancer que vous prévoyez
- **Documentez votre pool** : Notez quelque part quelles IP vous avez réservées
- **Laissez de la marge** : Prévoyez 20-30% d'IP supplémentaires pour la croissance

### 2. Séparation des Environnements

Si vous avez plusieurs environnements (dev, test, prod) :
- Utilisez des pools d'IP différents pour chaque environnement
- Ou utilisez des namespaces séparés

### 3. Monitoring

Gardez un œil sur :
- Le nombre d'IP utilisées vs disponibles
- L'état des pods MetalLB
- Les logs en cas de problème

## Désinstallation de MetalLB

Si vous devez désinstaller MetalLB :

```bash
microk8s disable metallb
```

**Attention** : Tous les Services de type LoadBalancer perdront leur IP externe et repasseront en état `<pending>`.

## Commandes Utiles

Voici un récapitulatif des commandes essentielles pour MetalLB :

```bash
# Activer MetalLB
microk8s enable metallb

# Vérifier l'état des pods
microk8s kubectl get pods -n metallb-system

# Voir le pool d'IP
microk8s kubectl get ipaddresspool -n metallb-system

# Voir tous les Services LoadBalancer
microk8s kubectl get svc --all-namespaces | grep LoadBalancer

# Voir les logs du controller
microk8s kubectl logs -n metallb-system -l component=controller

# Voir les logs des speakers
microk8s kubectl logs -n metallb-system -l component=speaker

# Désactiver MetalLB
microk8s disable metallb
```

## Prochaines Étapes

Maintenant que MetalLB est installé et configuré, vous êtes prêt à :

1. **Configurer le pool d'adresses** en détail (section 9.3)
2. **Créer des Services LoadBalancer** (section 9.4)
3. **Tester l'accès** depuis votre réseau local
4. **Intégrer avec Ingress** pour un routage avancé (chapitre 10)

## Résumé

Dans cette section, vous avez appris à :

✅ Comprendre l'architecture de MetalLB
✅ Installer MetalLB avec `microk8s enable metallb`
✅ Configurer un pool d'adresses IP
✅ Vérifier que l'installation fonctionne correctement
✅ Comprendre les modes Layer 2 et BGP
✅ Gérer les considérations réseau importantes

MetalLB transforme votre cluster MicroK8s en une plateforme capable d'exposer des services avec des IP externes stables, exactement comme dans un environnement cloud professionnel. C'est une brique essentielle pour construire des applications accessibles de manière fiable.

---

**Prochaine étape** : Configuration MetalLB : définir le pool d'adresses IP (Section 9.3)

⏭️ [Configuration MetalLB : définir le pool d'adresses IP](/09-load-balancing-avec-metallb/03-configuration-metallb-pool-dadresses-ip.md)
