🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 9.3 Configuration MetalLB : Définir le Pool d'Adresses IP

## Introduction

La configuration du pool d'adresses IP est le cœur de MetalLB. C'est ce qui détermine quelles adresses IP seront disponibles pour vos Services LoadBalancer. Une bonne planification de ce pool est essentielle pour éviter les conflits réseau et assurer un fonctionnement optimal.

Dans cette section, nous allons approfondir la configuration du pool d'IP, comprendre les différentes options disponibles, et apprendre à gérer cette configuration de manière avancée.

## Rappel : Qu'est-ce qu'un Pool d'Adresses IP ?

Un **pool d'adresses IP** est un ensemble d'adresses IP que MetalLB peut assigner aux Services de type LoadBalancer. Pensez-y comme à un réservoir d'IP disponibles :

- Quand vous créez un Service LoadBalancer, MetalLB pioche une IP dans ce pool
- L'IP est assignée au Service et annoncée sur le réseau
- Quand vous supprimez le Service, l'IP retourne dans le pool

## Planification du Pool d'Adresses IP

Avant de configurer MetalLB, prenez le temps de planifier votre pool d'IP. C'est une étape cruciale qui évitera bien des problèmes par la suite.

### Étape 1 : Comprendre Votre Réseau

Identifiez les informations suivantes sur votre réseau local :

**1. Votre sous-réseau**
```bash
# Sur Linux, pour voir votre configuration réseau
ip addr show

# Exemple de sortie :
# inet 192.168.1.50/24
```

Ici, `192.168.1.50/24` signifie :
- Votre machine est sur le réseau `192.168.1.0/24`
- Les IP vont de `192.168.1.1` à `192.168.1.254`
- Le masque est `255.255.255.0`

**2. La passerelle (gateway)**
```bash
# Votre routeur, généralement
ip route show | grep default

# Souvent : 192.168.1.1 ou 192.168.0.1
```

**3. Le pool DHCP de votre routeur**

Connectez-vous à l'interface de votre routeur pour voir :
- Quelle plage d'IP le DHCP utilise
- Exemple : `192.168.1.2` à `192.168.1.199`

### Étape 2 : Choisir Votre Plage d'IP

**Règles d'or** :

1. **Les IP doivent être sur le même sous-réseau** que vos machines
2. **Les IP ne doivent PAS être dans le pool DHCP**
3. **Les IP ne doivent PAS être déjà utilisées** par d'autres équipements

**Stratégie recommandée** :

Si votre DHCP utilise `192.168.1.2-199`, réservez la fin du range pour MetalLB :
- **Pour un petit lab** : `192.168.1.200-210` (11 IP)
- **Pour un lab moyen** : `192.168.1.200-220` (21 IP)
- **Pour un lab important** : `192.168.1.200-250` (51 IP)

### Étape 3 : Documenter Votre Choix

Créez un document (même simple) pour noter :
```
Réseau MicroK8s : 192.168.1.0/24
IP du serveur : 192.168.1.50
Pool DHCP : 192.168.1.2-199
Pool MetalLB : 192.168.1.200-210

Services prévus :
- Web dashboard : besoin d'1 IP
- API backend : besoin d'1 IP
- Base de données : besoin d'1 IP
Total : 3 IP (reste 8 IP libres)
```

## Configuration Initiale lors de l'Installation

Lors de l'exécution de `microk8s enable metallb`, vous avez été invité à entrer un pool d'IP. MetalLB a créé automatiquement deux ressources pour vous :

### 1. IPAddressPool

C'est la ressource qui définit les adresses disponibles.

### 2. L2Advertisement

C'est la ressource qui indique à MetalLB comment annoncer ces adresses sur le réseau (en mode Layer 2).

Examinons ces ressources en détail.

## Les Ressources MetalLB

MetalLB utilise des ressources Kubernetes personnalisées (Custom Resources) pour sa configuration. Les deux principales sont :

### IPAddressPool : Définir les Adresses

Visualisez la configuration actuelle :

```bash
microk8s kubectl get ipaddresspool -n metallb-system
```

Pour voir les détails :

```bash
microk8s kubectl get ipaddresspool -n metallb-system -o yaml
```

**Exemple de configuration** :

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

**Explication des champs** :

- **apiVersion** : Version de l'API MetalLB (v1beta1)
- **kind** : Type de ressource (IPAddressPool)
- **metadata.name** : Nom du pool (ici "default")
- **metadata.namespace** : Toujours dans metallb-system
- **spec.addresses** : Liste des plages d'adresses IP

### L2Advertisement : Annoncer les Adresses

Visualisez la configuration :

```bash
microk8s kubectl get l2advertisement -n metallb-system -o yaml
```

**Exemple** :

```yaml
apiVersion: metallb.io/v1beta1
kind: L2Advertisement
metadata:
  name: default
  namespace: metallb-system
spec:
  ipAddressPools:
  - default
```

**Explication** :

- Cette ressource dit à MetalLB d'annoncer les IP du pool "default" en mode Layer 2
- Sans cette ressource, les IP seraient assignées mais pas annoncées sur le réseau

## Formats d'Adresses Supportés

MetalLB accepte plusieurs formats pour définir les adresses IP. Vous pouvez les combiner dans le même pool.

### 1. Plage (Range)

Format le plus courant et le plus lisible :

```yaml
addresses:
- 192.168.1.200-192.168.1.210
```

**Utilisation** : Parfait quand vous avez un bloc contigu d'adresses.

### 2. Notation CIDR

Utilise la notation réseau standard :

```yaml
addresses:
- 192.168.1.200/29
```

Le `/29` signifie 8 adresses IP dont 6 utilisables (2 réservées pour réseau et broadcast dans le calcul strict, mais MetalLB peut toutes les utiliser).

**Tableau de correspondance CIDR** :

| CIDR | Nombre d'IP | Plage équivalente |
|------|-------------|-------------------|
| /32  | 1           | Une seule IP      |
| /31  | 2           | 2 IP              |
| /30  | 4           | 4 IP              |
| /29  | 8           | 8 IP              |
| /28  | 16          | 16 IP             |
| /27  | 32          | 32 IP             |
| /26  | 64          | 64 IP             |
| /25  | 128         | 128 IP            |
| /24  | 256         | 256 IP            |

**Utilisation** : Pratique pour définir rapidement un bloc, mais moins intuitif pour les débutants.

### 3. Adresse Unique

Pour assigner une seule IP spécifique :

```yaml
addresses:
- 192.168.1.200/32
```

Ou simplement :

```yaml
addresses:
- 192.168.1.200
```

**Utilisation** : Quand vous voulez réserver une IP précise pour un service particulier.

### 4. Combinaison Multiple

Vous pouvez combiner plusieurs formats :

```yaml
addresses:
- 192.168.1.200-192.168.1.210    # 11 IP
- 192.168.1.220/29                # 8 IP
- 192.168.1.250                   # 1 IP
```

Total : 20 adresses IP disponibles.

## Création de Pools Multiples

Vous pouvez créer plusieurs pools d'IP pour différents usages. C'est utile pour :
- Séparer les environnements (dev, test, prod)
- Séparer les types de services (web, databases, APIs)
- Avoir des pools avec des propriétés différentes

### Exemple : Deux Pools Distincts

**Pool pour les applications web** :

```yaml
apiVersion: metallb.io/v1beta1
kind: IPAddressPool
metadata:
  name: web-pool
  namespace: metallb-system
spec:
  addresses:
  - 192.168.1.200-192.168.1.209
```

**Pool pour les bases de données** :

```yaml
apiVersion: metallb.io/v1beta1
kind: IPAddressPool
metadata:
  name: database-pool
  namespace: metallb-system
spec:
  addresses:
  - 192.168.1.210-192.168.1.219
```

### Utiliser un Pool Spécifique

Par défaut, MetalLB utilise le premier pool disponible. Pour spécifier un pool particulier, ajoutez une annotation à votre Service :

```yaml
apiVersion: v1
kind: Service
metadata:
  name: mon-app-web
  annotations:
    metallb.universe.tf/address-pool: web-pool
spec:
  type: LoadBalancer
  selector:
    app: mon-app-web
  ports:
  - port: 80
    targetPort: 8080
```

L'annotation `metallb.universe.tf/address-pool: web-pool` force l'utilisation du pool "web-pool".

## Modifier la Configuration du Pool

Il y a plusieurs façons de modifier la configuration du pool d'adresses IP après l'installation initiale.

### Méthode 1 : Édition Manuelle (Recommandée)

Éditez directement la ressource IPAddressPool :

```bash
microk8s kubectl edit ipaddresspool default -n metallb-system
```

Cela ouvre un éditeur (généralement `vi` ou `nano`) avec le YAML. Modifiez la section `addresses` :

**Avant** :
```yaml
spec:
  addresses:
  - 192.168.1.200-192.168.1.210
```

**Après** :
```yaml
spec:
  addresses:
  - 192.168.1.200-192.168.1.220
```

Sauvegardez et quittez. La modification est appliquée immédiatement.

### Méthode 2 : Application d'un Fichier YAML

Créez un fichier `metallb-pool.yaml` :

```yaml
apiVersion: metallb.io/v1beta1
kind: IPAddressPool
metadata:
  name: default
  namespace: metallb-system
spec:
  addresses:
  - 192.168.1.200-192.168.1.220
```

Appliquez-le :

```bash
microk8s kubectl apply -f metallb-pool.yaml
```

### Méthode 3 : Désactivation et Réactivation

La méthode la plus simple mais destructive :

```bash
microk8s disable metallb
microk8s enable metallb
# Entrez la nouvelle plage d'IP
```

**⚠️ Attention** : Cette méthode supprime toutes les IP assignées aux Services existants. À utiliser uniquement si vous n'avez pas de Services LoadBalancer actifs ou si vous êtes prêt à les recréer.

## Configuration Avancée du Pool

### Auto-Assignment (Assignation Automatique)

Par défaut, MetalLB assigne automatiquement des IP du pool. Vous pouvez désactiver cette fonctionnalité pour certains pools :

```yaml
apiVersion: metallb.io/v1beta1
kind: IPAddressPool
metadata:
  name: reserved-pool
  namespace: metallb-system
spec:
  addresses:
  - 192.168.1.230-192.168.1.240
  autoAssign: false
```

**Utilisation** : Pour des IP que vous voulez assigner manuellement à des Services spécifiques.

Pour utiliser une IP de ce pool, vous devez spécifier l'IP exacte dans le Service :

```yaml
apiVersion: v1
kind: Service
metadata:
  name: mon-service-special
  annotations:
    metallb.universe.tf/address-pool: reserved-pool
spec:
  type: LoadBalancer
  loadBalancerIP: 192.168.1.235  # IP spécifique
  # ... reste de la configuration
```

### Éviter Certaines Adresses (Avoid Buggy IPs)

Si certaines adresses causent des problèmes, vous pouvez les exclure en créant des plages discontinues :

```yaml
addresses:
- 192.168.1.200-192.168.1.209    # OK
# On saute 192.168.1.210 (problématique)
- 192.168.1.211-192.168.1.220    # OK
```

## L2Advertisement Avancé

La ressource L2Advertisement contrôle comment MetalLB annonce les IP sur le réseau.

### Configuration Basique

La configuration par défaut annonce tous les pools :

```yaml
apiVersion: metallb.io/v1beta1
kind: L2Advertisement
metadata:
  name: default
  namespace: metallb-system
spec:
  ipAddressPools:
  - default
```

### Limiter aux Nœuds Spécifiques

Dans un cluster multi-nœud, vous pouvez limiter quels nœuds annoncent les IP :

```yaml
apiVersion: metallb.io/v1beta1
kind: L2Advertisement
metadata:
  name: selective-l2
  namespace: metallb-system
spec:
  ipAddressPools:
  - default
  nodeSelectors:
  - matchLabels:
      kubernetes.io/hostname: node1
  - matchLabels:
      kubernetes.io/hostname: node2
```

Seuls les nœuds `node1` et `node2` annonceront les IP.

### Annoncer Plusieurs Pools

Vous pouvez avoir une seule L2Advertisement pour plusieurs pools :

```yaml
apiVersion: metallb.io/v1beta1
kind: L2Advertisement
metadata:
  name: all-pools
  namespace: metallb-system
spec:
  ipAddressPools:
  - web-pool
  - database-pool
  - api-pool
```

Ou créer des L2Advertisement séparées pour chaque pool avec des configurations différentes.

## Vérification de la Configuration

### Voir tous les Pools

```bash
microk8s kubectl get ipaddresspool -n metallb-system
```

### Voir le Détail d'un Pool

```bash
microk8s kubectl describe ipaddresspool default -n metallb-system
```

### Voir les IP Assignées

Pour voir quels Services utilisent quelles IP :

```bash
microk8s kubectl get svc --all-namespaces -o wide | grep LoadBalancer
```

Exemple de sortie :
```
NAMESPACE   NAME           TYPE           EXTERNAL-IP      PORT(S)
default     mon-app        LoadBalancer   192.168.1.200    80:30123/TCP
default     mon-api        LoadBalancer   192.168.1.201    443:30456/TCP
```

### Calculer les IP Disponibles

Si vous avez `192.168.1.200-210` (11 IP) et 3 Services, il vous reste 8 IP disponibles.

## Exemples de Configurations Complètes

### Configuration Simple : Lab Personnel

Pour un lab de test basique :

```yaml
apiVersion: metallb.io/v1beta1
kind: IPAddressPool
metadata:
  name: default
  namespace: metallb-system
spec:
  addresses:
  - 192.168.1.200-192.168.1.210
---
apiVersion: metallb.io/v1beta1
kind: L2Advertisement
metadata:
  name: default
  namespace: metallb-system
spec:
  ipAddressPools:
  - default
```

### Configuration Avancée : Plusieurs Pools

Pour un environnement plus structuré :

```yaml
# Pool pour le développement
apiVersion: metallb.io/v1beta1
kind: IPAddressPool
metadata:
  name: dev-pool
  namespace: metallb-system
spec:
  addresses:
  - 192.168.1.200-192.168.1.209
---
# Pool pour la production
apiVersion: metallb.io/v1beta1
kind: IPAddressPool
metadata:
  name: prod-pool
  namespace: metallb-system
spec:
  addresses:
  - 192.168.1.210-192.168.1.219
---
# Pool réservé (assignation manuelle uniquement)
apiVersion: metallb.io/v1beta1
kind: IPAddressPool
metadata:
  name: reserved-pool
  namespace: metallb-system
spec:
  addresses:
  - 192.168.1.220-192.168.1.225
  autoAssign: false
---
# Annonce de tous les pools
apiVersion: metallb.io/v1beta1
kind: L2Advertisement
metadata:
  name: all-pools-l2
  namespace: metallb-system
spec:
  ipAddressPools:
  - dev-pool
  - prod-pool
  - reserved-pool
```

## Scénarios Courants

### Scénario 1 : Manque d'IP

**Symptôme** : Vos nouveaux Services restent en `<pending>`.

**Diagnostic** :
```bash
# Comptez les Services LoadBalancer existants
microk8s kubectl get svc --all-namespaces | grep LoadBalancer | wc -l

# Vérifiez la taille de votre pool
microk8s kubectl get ipaddresspool default -n metallb-system -o yaml
```

**Solution** : Étendez votre pool :
```bash
microk8s kubectl edit ipaddresspool default -n metallb-system
# Modifiez la plage, par exemple de 200-210 à 200-220
```

### Scénario 2 : Conflit d'IP

**Symptôme** : Une IP assignée par MetalLB est déjà utilisée sur le réseau.

**Solution** :
1. Identifiez l'IP en conflit
2. Retirez-la du pool en créant une plage discontinue
3. Supprimez et recréez le Service qui utilisait cette IP

### Scénario 3 : Réorganisation du Pool

**Besoin** : Vous voulez réorganiser complètement vos IP.

**Procédure** :
1. Notez tous vos Services LoadBalancer existants
2. Supprimez tous ces Services
3. Modifiez la configuration du pool
4. Recréez les Services (ils obtiendront de nouvelles IP)

## Bonnes Pratiques

### 1. Documentation

Maintenez un document à jour listant :
- Votre pool d'IP MetalLB
- Les Services et leurs IP assignées
- Les IP réservées pour un usage futur

Exemple :
```
Pool MetalLB : 192.168.1.200-220

IP assignées :
- 192.168.1.200 : ingress-nginx (Ingress Controller)
- 192.168.1.201 : dashboard (Kubernetes Dashboard)
- 192.168.1.202 : monitoring (Prometheus/Grafana)
- 192.168.1.203 : registry (Docker Registry privé)

IP disponibles : 192.168.1.204-220 (17 IP)
```

### 2. Planification de Croissance

Ne dimensionnez pas votre pool au strict minimum :
- **Petit lab** : Prévoyez 10-15 IP même si vous n'en utilisez que 3-4
- **Lab moyen** : 20-30 IP
- **Lab important** : 50+ IP

### 3. Séparation Logique

Si vous gérez plusieurs environnements ou types de services, utilisez plusieurs pools pour :
- Faciliter la gestion
- Éviter les conflits
- Améliorer la lisibilité

### 4. Versionning

Stockez vos configurations MetalLB dans Git :

```bash
# Exporter la configuration actuelle
microk8s kubectl get ipaddresspool -n metallb-system -o yaml > metallb-pools.yaml
microk8s kubectl get l2advertisement -n metallb-system -o yaml > metallb-l2adv.yaml

# Versionner avec Git
git add metallb-*.yaml
git commit -m "Configuration MetalLB - ajout de 10 IP"
```

### 5. Tests avant Déploiement

Avant de modifier le pool en production :
- Testez la configuration dans un environnement de dev
- Vérifiez qu'il n'y a pas de conflits réseau
- Documentez la procédure de rollback

## Résolution de Problèmes

### Les Modifications ne Sont Pas Prises en Compte

**Vérifications** :
```bash
# Vérifiez que les pods MetalLB tournent
microk8s kubectl get pods -n metallb-system

# Consultez les logs du controller
microk8s kubectl logs -n metallb-system -l component=controller

# Vérifiez la syntaxe YAML
microk8s kubectl get ipaddresspool default -n metallb-system -o yaml
```

**Solution** : Redémarrez le controller si nécessaire :
```bash
microk8s kubectl rollout restart deployment controller -n metallb-system
```

### Erreurs de Format d'Adresse

Si vous voyez des erreurs comme `invalid address format`, vérifiez :
- La syntaxe de votre plage (tiret, pas d'espaces)
- Que les IP sont bien dans l'ordre croissant
- Que vous n'avez pas mélangé IPv4 et IPv6

### IP Non Accessibles

Si une IP est assignée mais non accessible :
1. Vérifiez avec `ping` depuis une autre machine
2. Vérifiez les règles firewall
3. Vérifiez que l'IP est bien dans votre sous-réseau
4. Consultez les logs du speaker : `microk8s kubectl logs -n metallb-system -l component=speaker`

## Résumé

La configuration du pool d'adresses IP de MetalLB est une étape cruciale qui demande :

✅ **Planification** : Comprendre votre réseau et choisir les bonnes IP
✅ **Documentation** : Garder une trace de vos choix
✅ **Flexibilité** : Savoir modifier la configuration quand nécessaire
✅ **Organisation** : Utiliser plusieurs pools si besoin
✅ **Surveillance** : Monitorer l'utilisation des IP

Avec une bonne configuration du pool d'IP, MetalLB devient un outil puissant et fiable pour exposer vos Services Kubernetes. Vous disposez maintenant d'une base solide pour créer des Services LoadBalancer professionnels.

---

**Prochaine étape** : Services de type LoadBalancer (Section 9.4)

⏭️ [Services de type LoadBalancer](/09-load-balancing-avec-metallb/04-services-de-type-loadbalancer.md)
