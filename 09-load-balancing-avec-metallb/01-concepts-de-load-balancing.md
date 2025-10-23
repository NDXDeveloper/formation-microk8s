🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 9.1 Concepts de Load Balancing

## Introduction

Le **load balancing** (ou répartition de charge en français) est un concept fondamental dans le monde des infrastructures informatiques modernes. Dans le contexte de Kubernetes et MicroK8s, comprendre ce concept est essentiel pour exposer vos applications de manière fiable et performante.

## Qu'est-ce que le Load Balancing ?

Imaginez un restaurant très populaire. Si tous les clients devaient être servis par un seul serveur, le service serait lent et chaotique. La solution ? Avoir plusieurs serveurs et un maître d'hôtel qui répartit équitablement les clients entre eux. Le load balancing fonctionne exactement sur ce principe pour vos applications.

Un **load balancer** est un composant qui :
- Reçoit les requêtes entrantes (trafic réseau)
- Les distribue intelligemment entre plusieurs instances de votre application (les Pods dans Kubernetes)
- S'assure que chaque instance reçoit une charge de travail équilibrée
- Redirige automatiquement le trafic en cas de défaillance d'une instance

## Pourquoi le Load Balancing est-il Nécessaire ?

### 1. **Haute Disponibilité**

Si vous n'avez qu'une seule instance de votre application et qu'elle tombe en panne, votre service devient indisponible. Avec le load balancing et plusieurs instances :
- Si une instance échoue, les autres continuent de fonctionner
- Le load balancer détecte l'instance défaillante et cesse de lui envoyer du trafic
- Vos utilisateurs ne remarquent même pas le problème

### 2. **Performance et Scalabilité**

Une seule instance d'application a des limites (CPU, mémoire, capacité de traitement). Le load balancing permet de :
- Répartir les requêtes entre plusieurs instances
- Augmenter la capacité globale en ajoutant simplement plus d'instances
- Éviter la surcharge d'une seule instance
- Améliorer les temps de réponse pour vos utilisateurs

### 3. **Maintenance sans Interruption**

Avec le load balancing, vous pouvez :
- Mettre à jour vos applications progressivement (rolling updates)
- Retirer temporairement des instances pour maintenance
- Maintenir le service actif pendant les opérations de maintenance

## Types de Load Balancing

### Load Balancing au Niveau 4 (Transport)

Le load balancer travaille au niveau TCP/UDP (couche transport du modèle OSI) :
- **Avantages** : Très rapide, faible latence, simple
- **Limitations** : Ne peut pas prendre de décisions basées sur le contenu HTTP
- **Cas d'usage** : Applications non-HTTP, bases de données, trafic générique

### Load Balancing au Niveau 7 (Application)

Le load balancer comprend le protocole HTTP/HTTPS :
- **Avantages** : Routage intelligent basé sur l'URL, les headers, les cookies
- **Capacités** : Terminaison SSL/TLS, réécriture d'URL, routage par chemin
- **Cas d'usage** : Applications web, APIs REST, microservices

## Algorithmes de Répartition de Charge

Un load balancer utilise différents algorithmes pour décider quelle instance doit recevoir la prochaine requête :

### 1. **Round Robin (Tour à tour)**
Les requêtes sont distribuées séquentiellement à chaque instance, l'une après l'autre.

```
Requête 1 → Instance A
Requête 2 → Instance B
Requête 3 → Instance C
Requête 4 → Instance A (on recommence)
```

**Avantage** : Simple et équitable
**Inconvénient** : Ne prend pas en compte la charge actuelle de chaque instance

### 2. **Least Connections (Moins de connexions)**
Les requêtes sont envoyées à l'instance ayant le moins de connexions actives.

**Avantage** : Meilleure répartition si les requêtes ont des durées variables
**Cas d'usage** : Applications avec des sessions longues

### 3. **IP Hash**
La décision est basée sur l'adresse IP du client (via une fonction de hachage).

**Avantage** : Un même client arrive toujours sur la même instance (affinité de session)
**Cas d'usage** : Applications avec état (stateful) nécessitant la persistence de session

### 4. **Weighted (Pondéré)**
Chaque instance reçoit un poids, et les requêtes sont distribuées proportionnellement.

**Cas d'usage** : Quand certaines instances sont plus puissantes que d'autres

## Load Balancing dans Kubernetes

Dans Kubernetes, le load balancing est intégré de plusieurs façons :

### Services de Type ClusterIP (Par défaut)

- Load balancing **interne** uniquement
- Accessible uniquement depuis l'intérieur du cluster
- Kubernetes distribue automatiquement le trafic entre les Pods correspondants
- Utilise kube-proxy pour la répartition

### Services de Type NodePort

- Expose un port sur chaque nœud du cluster
- Kubernetes fait du load balancing entre les Pods
- Accessible de l'extérieur via `<NodeIP>:<NodePort>`
- Limitation : Nécessite de connaître l'IP des nœuds et utilise des ports hauts (30000-32767)

### Services de Type LoadBalancer

- **C'est ici que MetalLB entre en jeu !**
- Fournit une IP externe unique pour accéder au service
- Le load balancer externe distribue le trafic vers les nœuds
- Kubernetes distribue ensuite le trafic vers les Pods

## Le Problème que MetalLB Résout

Kubernetes a été conçu principalement pour les environnements cloud (AWS, GCP, Azure). Dans ces environnements, quand vous créez un Service de type `LoadBalancer`, le cloud provider provisionne automatiquement un load balancer (payant) avec une IP publique.

**Problème** : Dans un environnement bare-metal (serveurs physiques) ou dans un lab personnel, vous n'avez pas ce fournisseur cloud automatique.

**Solution** : MetalLB est une implémentation de load balancer pour Kubernetes en bare-metal. Il permet d'assigner des IP externes à vos Services de type LoadBalancer, même sans cloud provider.

## Flux de Trafic avec Load Balancing

Voici comment le trafic circule avec un load balancer :

```
1. Client (Utilisateur)
   ↓
2. Load Balancer (IP externe unique)
   ↓
3. Distribution vers les Nœuds du cluster
   ↓
4. Service Kubernetes (répartition vers les Pods)
   ↓
5. Pods de l'application (instances multiples)
```

### Exemple Concret

Imaginez une application web déployée avec 3 replicas :

```
Utilisateur → http://mon-app.com (IP: 192.168.1.100)
                ↓
         Load Balancer (MetalLB)
        /        |        \
       /         |         \
   Pod 1      Pod 2      Pod 3
```

Le load balancer :
1. Reçoit toutes les requêtes sur l'IP 192.168.1.100
2. Vérifie quels Pods sont en bonne santé (health checks)
3. Distribue les requêtes entre les Pods disponibles
4. Si Pod 2 tombe, il redirige tout le trafic vers Pod 1 et Pod 3

## Health Checks (Vérifications de Santé)

Un load balancer intelligent ne se contente pas de distribuer le trafic, il vérifie aussi que les instances sont en bonne santé :

### Liveness Probes (Sondes de Vivacité)
- "L'application est-elle vivante ?"
- Si la sonde échoue, Kubernetes redémarre le Pod
- Détecte les applications bloquées ou crashées

### Readiness Probes (Sondes de Disponibilité)
- "L'application est-elle prête à recevoir du trafic ?"
- Si la sonde échoue, le load balancer arrête d'envoyer du trafic à ce Pod
- Ne redémarre pas le Pod, attend juste qu'il soit prêt

**Exemple** : Une application qui démarre peut prendre 30 secondes pour se connecter à une base de données. Pendant ce temps, elle est "alive" mais pas "ready".

## Bénéfices du Load Balancing dans MicroK8s

Pour votre lab personnel ou environnement de développement, le load balancing avec MetalLB vous apporte :

1. **Une expérience similaire au cloud** : Vous pouvez créer des Services LoadBalancer comme sur AWS/GCP
2. **Des IP stables et accessibles** : Accédez à vos applications via des IP dédiées
3. **Une préparation pour la production** : Les compétences apprises sont directement transférables
4. **Gestion simplifiée** : Plus besoin de jongler avec des NodePorts ou des configurations complexes
5. **Haute disponibilité locale** : Testez et apprenez les concepts de HA sur votre propre infrastructure

## Limitations et Considérations

### Limitations du Load Balancing L4

- Pas de routage basé sur l'URL (pas de `/api` vs `/web`)
- Pas de terminaison SSL/TLS au niveau du load balancer
- Pour ces fonctionnalités avancées, vous utiliserez un **Ingress Controller** (voir chapitre 10)

### Différence Load Balancer vs Ingress

Il est important de comprendre la différence :

**Load Balancer (MetalLB)** :
- Une IP par Service
- Load balancing niveau 4 (TCP/UDP)
- Simple mais peut devenir coûteux (une IP par service)

**Ingress** :
- Une seule IP pour plusieurs services
- Routage niveau 7 (HTTP/HTTPS)
- Routage intelligent basé sur les noms de domaine et chemins
- Terminaison SSL/TLS centralisée

**En pratique** : On utilise souvent les deux ensemble :
- MetalLB fournit l'IP externe à l'Ingress Controller
- L'Ingress Controller fait ensuite le routage intelligent vers les différents Services

## Résumé

Le load balancing est un mécanisme essentiel pour :
- Distribuer le trafic entre plusieurs instances d'une application
- Assurer la haute disponibilité et la performance
- Permettre la scalabilité horizontale (ajouter plus d'instances)
- Gérer automatiquement les pannes

Dans MicroK8s, MetalLB permet de bénéficier de Services de type LoadBalancer même sans cloud provider, en assignant des IP du réseau local à vos Services. C'est une brique fondamentale pour exposer vos applications de manière professionnelle.

Dans les sections suivantes, nous verrons comment installer et configurer MetalLB concrètement dans votre cluster MicroK8s.

---

**Prochaine étape** : Installation de MetalLB (Section 9.2)

⏭️ [Installation de MetalLB (microk8s enable metallb)](/09-load-balancing-avec-metallb/02-installation-de-metallb.md)
