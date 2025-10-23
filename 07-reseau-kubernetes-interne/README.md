🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 7. Réseau Kubernetes Interne

## Introduction au chapitre

Bienvenue dans l'un des chapitres les plus importants de cette formation : le **réseau Kubernetes**. Si Kubernetes était une ville, le réseau serait son système de routes, de ponts et de communications. Sans un réseau fonctionnel, vos applications ne pourraient tout simplement pas communiquer entre elles, et votre cluster ne serait qu'un ensemble de conteneurs isolés et inutiles.

Le réseau est souvent perçu comme l'aspect le plus complexe de Kubernetes, et cette réputation n'est pas totalement injustifiée. Cependant, avec une approche progressive et pédagogique, vous découvrirez que les principes sous-jacents sont élégants et logiques. Ce chapitre est conçu pour démystifier le réseau Kubernetes et vous donner une compréhension solide, accessible même si vous n'êtes pas un expert réseau.

## Pourquoi le réseau est-il si important dans Kubernetes ?

### Le défi de la communication distribuée

Dans un environnement traditionnel, les applications s'exécutent sur des serveurs fixes avec des adresses IP stables. Si votre application web doit contacter une base de données, vous configurez simplement l'IP de la base de données, et tout fonctionne.

**Mais dans Kubernetes, tout est différent :**

```
Environnement traditionnel :
┌─────────────┐         ┌─────────────┐
│   Web App   │────────▶│  Database   │
│ 192.168.1.5 │         │ 192.168.1.6 │
│  (fixe)     │         │  (fixe)     │
└─────────────┘         └─────────────┘
Configuration stable, IPs permanentes


Environnement Kubernetes :
┌─────────────┐         ┌─────────────┐
│  Web Pod    │────?───▶│   DB Pod    │
│  10.1.1.5   │         │  10.1.1.8   │
│  (éphémère) │         │  (éphémère) │
└─────────────┘         └─────────────┘
    │                        │
    │ Redémarre              │ Redémarre
    ▼                        ▼
│  10.1.1.23  │         │  10.1.1.17  │
└─────────────┘         └─────────────┘
IPs changeantes, comment rester connecté ?
```

**Les défis spécifiques à Kubernetes :**

1. **Éphémérité** : Les Pods peuvent être créés, détruits et recréés à tout moment, changeant d'IP à chaque fois
2. **Distribution** : Les Pods peuvent être sur différents serveurs (Nodes) et doivent pouvoir communiquer entre eux
3. **Scalabilité** : Une application peut avoir 1, 10, ou 100 Pods qui doivent tous être accessibles
4. **Découverte** : Comment une application trouve-t-elle les autres applications dont elle a besoin ?
5. **Isolation** : Comment garantir que seules les applications autorisées peuvent communiquer entre elles ?

### La solution Kubernetes

Kubernetes résout ces défis avec une architecture réseau élégante qui repose sur trois piliers :

**1. Un réseau plat et universel**
- Chaque Pod a sa propre adresse IP
- Tous les Pods peuvent communiquer avec tous les autres Pods
- Pas de NAT compliqué entre les Pods

**2. Des Services comme abstraction**
- Les Services fournissent des points d'entrée stables
- Redirection automatique vers les Pods disponibles
- Load balancing intégré

**3. Un DNS intégré**
- Découverte automatique des services par nom
- Plus besoin de gérer les IPs manuellement
- Configuration portable entre environnements

## Ce que vous allez apprendre dans ce chapitre

Ce chapitre est divisé en cinq sections progressives, chacune construisant sur la précédente :

### 7.1 Concepts de base du réseau Kubernetes

Vous apprendrez les **fondations théoriques** du réseau Kubernetes :

- **Les quatre règles d'or** qui régissent le réseau Kubernetes
- **Les différents types d'adresses IP** : Pod IP, Service IP, Node IP
- **Le concept de réseau plat** et pourquoi c'est génial
- **Les plages d'adresses (CIDR)** et comment elles sont organisées
- **Le rôle du CNI** (Container Network Interface)

**À la fin de cette section**, vous comprendrez *pourquoi* le réseau Kubernetes fonctionne comme il fonctionne, et vous aurez un modèle mental clair de l'architecture globale.

### 7.2 Networking CNI (Container Network Interface)

Vous plongerez dans le **fonctionnement concret du CNI**, le plugin qui implémente le réseau :

- **Qu'est-ce que le CNI** et pourquoi il est nécessaire
- **Comment Calico fonctionne** (le CNI par défaut de MicroK8s)
- **Le cycle de vie réseau d'un Pod** depuis sa création
- **Les interfaces réseau virtuelles** (veth pairs)
- **Le routage BGP** et comment les Nodes communiquent
- **L'IPAM** (gestion des adresses IP)

**À la fin de cette section**, vous comprendrez *comment* le réseau est réellement implémenté au niveau technique, et vous pourrez diagnostiquer les problèmes réseau de base.

### 7.3 DNS interne avec CoreDNS

Vous explorerez le **système DNS de Kubernetes**, crucial pour la découverte de services :

- **Pourquoi le DNS est essentiel** dans Kubernetes
- **Comment CoreDNS fonctionne** et son architecture
- **Le format des noms DNS** dans Kubernetes
- **Le parcours d'une requête DNS** étape par étape
- **Le Corefile** et la configuration de CoreDNS
- **Comment déboguer les problèmes DNS**

**À la fin de cette section**, vous saurez utiliser le DNS pour faire communiquer vos applications, et vous pourrez résoudre les problèmes de résolution de noms.

### 7.4 Communication Pod-to-Pod et Service-to-Pod

Vous verrez des **exemples concrets** de communication entre applications :

- **Communication directe Pod-to-Pod** (et pourquoi l'éviter)
- **Communication via Services** (le pattern recommandé)
- **Le rôle de kube-proxy** et comment il implémente les Services
- **Patterns de communication courants** (microservices, 3-tiers, etc.)
- **Les différents types de Services** (ClusterIP, NodePort, LoadBalancer)
- **Load balancing** et distribution du trafic
- **Readiness et Liveness Probes** pour la santé des Pods

**À la fin de cette section**, vous saurez concevoir des architectures applicatives robustes qui tirent parti du réseau Kubernetes.

### 7.5 Test de connectivité interne

Vous acquerrez des **compétences pratiques de diagnostic** :

- **Les outils essentiels** : kubectl, busybox, netshoot
- **Comment tester la connectivité** entre Pods et Services
- **Tests DNS** pour vérifier la résolution de noms
- **Analyse des logs** (Pods, CoreDNS, Calico)
- **Méthodologie de debugging** étape par étape
- **Commandes de diagnostic** pour tous les scénarios
- **Outils avancés** : tcpdump, iperf3, traceroute

**À la fin de cette section**, vous serez capable de diagnostiquer et résoudre pratiquement tous les problèmes réseau que vous rencontrerez.

## Prérequis pour ce chapitre

### Connaissances nécessaires

**Vous devriez être à l'aise avec :**

- Les concepts de base de Kubernetes (Pods, Services, Deployments) - *Chapitres 3 et 4*
- L'utilisation de kubectl pour créer et gérer des ressources - *Chapitre 4*
- Les notions de base sur les conteneurs Docker - *Chapitre 1*

**Connaissances réseau utiles (mais pas obligatoires) :**

- Adresses IP et masques de sous-réseau
- Notion de ports TCP/UDP
- DNS basique (résolution de noms de domaine)

**Ne vous inquiétez pas si vous n'êtes pas un expert réseau !** Ce chapitre est conçu pour être accessible aux débutants, avec des analogies, des schémas et des explications pas à pas.

### Environnement requis

Vous devriez avoir :

- ✅ **MicroK8s installé et fonctionnel** - *Chapitre 2*
- ✅ **L'addon DNS activé** : `microk8s enable dns`
- ✅ **Accès à kubectl** (ou microk8s kubectl)

Optionnel mais recommandé :
- Accès SSH à votre Node MicroK8s (si vous utilisez une VM)
- Au moins 2 GB de RAM disponible

## Philosophie d'apprentissage du réseau

### Approche progressive : de la théorie à la pratique

Ce chapitre suit une approche pédagogique en trois temps :

**1. Comprendre (Théorie)**
- *Sections 7.1 et 7.2* : Modèle mental et concepts
- Analogies et schémas pour visualiser
- Comprendre le "pourquoi" avant le "comment"

**2. Observer (Démonstration)**
- *Sections 7.3 et 7.4* : Voir le réseau en action
- Exemples concrets et cas d'usage réels
- Suivre le parcours des paquets

**3. Pratiquer (Hands-on)**
- *Section 7.5* : Utiliser les outils de diagnostic
- Tester, valider, déboguer
- Construire vos propres réflexes

### Ne pas avoir peur de "casser" des choses

Le meilleur moyen d'apprendre le réseau Kubernetes est d'expérimenter. Dans votre environnement de lab :

- ✅ Créez des Pods et regardez comment ils obtiennent des IPs
- ✅ Supprimez CoreDNS temporairement pour voir ce qui se passe
- ✅ Créez des Services et vérifiez comment ils sont implémentés
- ✅ Cassez volontairement des choses puis réparez-les

**L'échec est une opportunité d'apprentissage.** Plus vous comprendrez pourquoi quelque chose ne fonctionne pas, plus vous maîtriserez le réseau Kubernetes.

## Ce que ce chapitre n'est PAS

Pour gérer vos attentes, voici ce que ce chapitre ne couvrira pas (ces sujets seront abordés dans des chapitres ultérieurs) :

### Sujets avancés (chapitres futurs)

**Network Policies** (Chapitre 16 - Sécurité)
- Règles de firewall pour contrôler le trafic
- Isolation réseau entre namespaces
- Sécurisation des communications

**Ingress et routage HTTP** (Chapitre 10)
- Exposer des applications vers Internet
- Routage basé sur les noms de domaine
- Certificats SSL/TLS

**Load Balancing externe** (Chapitre 9)
- MetalLB pour les IPs externes
- Intégration avec des load balancers cloud

**Service Mesh** (Chapitre 25)
- Istio, Linkerd
- Communication mTLS
- Observabilité avancée

**Multi-cluster networking** (Chapitre 25)
- Communication entre clusters
- Fédération de services

### Focus de ce chapitre

Ce chapitre se concentre exclusivement sur le **réseau interne du cluster** :

- Comment les Pods communiquent entre eux
- Comment les Services fonctionnent
- Comment le DNS fonctionne
- Comment diagnostiquer les problèmes

C'est la fondation absolue que vous devez maîtriser avant de passer aux sujets avancés.

## Le réseau Kubernetes vs le réseau traditionnel

### Différences fondamentales

Pour vous aider à changer de paradigme, voici les principales différences entre un réseau traditionnel et le réseau Kubernetes :

| Aspect | Réseau Traditionnel | Réseau Kubernetes |
|--------|---------------------|-------------------|
| **Adresses IP** | Statiques, configurées manuellement | Dynamiques, assignées automatiquement |
| **Découverte** | Fichiers de config, base de données | DNS automatique |
| **Load balancing** | Matériel externe, HAProxy, nginx | Intégré (Services, kube-proxy) |
| **Isolation** | VLANs, firewalls physiques | Network Policies logicielles |
| **Scalabilité** | Ajouter des serveurs manuellement | Scaler les Pods automatiquement |
| **Résilience** | Redondance matérielle complexe | Auto-guérison intégrée |

### Changement de mentalité

**Ancien paradigme (infra traditionnelle) :**
```
"J'ai besoin d'un serveur web ?"
→ Provisionner un serveur
→ Lui donner une IP fixe
→ Configurer les routes
→ Mettre à jour le DNS
→ Configurer le load balancer
→ Mettre en production
Temps : plusieurs jours
```

**Nouveau paradigme (Kubernetes) :**
```
"J'ai besoin d'un serveur web ?"
→ kubectl apply -f deployment.yaml
→ Kubernetes s'occupe de tout
Temps : quelques secondes
```

**Le réseau Kubernetes automatise ce qui prenait auparavant des jours de travail manuel.**

## Analogies utiles pour comprendre le réseau

Tout au long de ce chapitre, nous utiliserons des analogies pour rendre les concepts abstraits plus concrets :

### L'analogie de la ville

- **Pods** = Maisons/immeubles
- **Services** = Bureaux de poste avec adresse fixe
- **DNS** = Annuaire téléphonique
- **CNI** = Entreprise de construction des routes
- **Kube-proxy** = Panneau indicateur qui redirige le trafic
- **Network Policies** = Murs et barrières de sécurité

### L'analogie postale

- **Pod IP** = Adresse de domicile (change si vous déménagez)
- **Service IP** = Adresse professionnelle (stable)
- **DNS** = Annuaire qui associe un nom à une adresse
- **Packets réseau** = Lettres postales
- **Routes** = Chemins que prend le facteur

Ces analogies seront utilisées pour simplifier les explications techniques.

## Conseils pour maximiser votre apprentissage

### 1. Lisez dans l'ordre

Les sections se construisent les unes sur les autres. Ne sautez pas de section, même si un sujet semble facile.

### 2. Pratiquez en parallèle

Gardez un terminal ouvert avec MicroK8s pendant votre lecture. Testez les commandes au fur et à mesure.

### 3. Dessinez des schémas

Le réseau est visuel. Dessinez vos propres schémas pour représenter comment les Pods communiquent. Cela aide énormément à la compréhension.

### 4. Prenez des notes

Notez les commandes importantes, les concepts clés, et vos propres observations.

### 5. Posez-vous des questions

- "Que se passerait-il si je supprimais ce Pod ?"
- "Comment le Service sait-il vers quel Pod rediriger ?"
- "Pourquoi cette commande retourne cette erreur ?"

### 6. Créez des "cartes mentales"

Organisez les concepts en cartes mentales pour voir les relations entre eux :

```
Réseau Kubernetes
├── Pod Networking
│   ├── CNI (Calico)
│   ├── IP Assignment (IPAM)
│   └── Pod-to-Pod comm
├── Services
│   ├── ClusterIP
│   ├── NodePort
│   └── LoadBalancer
├── DNS
│   ├── CoreDNS
│   ├── Service Discovery
│   └── Name Resolution
└── Debugging
    ├── kubectl
    ├── busybox
    └── netshoot
```

### 7. Expérimentez sans crainte

Votre lab MicroK8s est un environnement sûr. Cassez des choses, observez ce qui se passe, puis réparez. C'est comme cela qu'on apprend vraiment.

## Ressources complémentaires

### Documentation officielle

Bien que ce chapitre soit complet, vous pouvez approfondir avec :

- **Kubernetes Networking Concepts** : https://kubernetes.io/docs/concepts/services-networking/
- **CoreDNS Documentation** : https://coredns.io/manual/toc/
- **Calico Documentation** : https://docs.projectcalico.org/

### Commandes de référence rapide

Gardez cette liste à portée de main :

```bash
# Voir les Pods et leurs IPs
microk8s kubectl get pods -o wide

# Voir les Services
microk8s kubectl get svc

# Tester le DNS
microk8s kubectl run test --image=busybox --restart=Never -- sleep 3600
microk8s kubectl exec test -- nslookup kubernetes

# Voir les logs CoreDNS
microk8s kubectl logs -n kube-system -l k8s-app=kube-dns

# Créer un Pod de debug complet
microk8s kubectl run netshoot --image=nicolaka/netshoot --restart=Never -- sleep 3600
```

## État d'esprit pour réussir

### Pour les débutants complets en réseau

**Ne paniquez pas !** Le réseau peut sembler intimidant, mais :

1. Vous n'avez pas besoin d'être un expert réseau pour comprendre Kubernetes
2. Les concepts sont expliqués depuis zéro
3. Les analogies rendent tout plus accessible
4. La pratique régulière développe l'intuition

**Approche recommandée :**
- Lisez une section par jour
- Pratiquez ce que vous avez appris
- Relisez les parties difficiles
- Ne passez à la section suivante que quand vous êtes à l'aise

### Pour ceux qui ont des bases réseau

Vous avez un avantage, mais attention :

1. Le réseau Kubernetes a ses propres spécificités
2. Oubliez certaines habitudes du réseau traditionnel
3. Embrassez le paradigme cloud-native
4. Les concepts sont simples, mais l'orchestration est puissante

**Approche recommandée :**
- Lisez rapidement les bases
- Concentrez-vous sur les spécificités Kubernetes (CNI, Services, DNS)
- Expérimentez avec les outils de diagnostic avancés
- Approfondissez les aspects techniques (BGP, iptables, etc.)

## Objectifs d'apprentissage du chapitre

À la fin de ce chapitre, vous serez capable de :

### Connaissances théoriques

- ✅ **Expliquer** les quatre règles d'or du réseau Kubernetes
- ✅ **Décrire** comment un Pod obtient son IP et communique
- ✅ **Comprendre** le rôle du CNI et de CoreDNS
- ✅ **Distinguer** les différents types de Services et leurs cas d'usage
- ✅ **Visualiser** le parcours d'un paquet réseau dans le cluster

### Compétences pratiques

- ✅ **Créer** des Pods et Services qui communiquent
- ✅ **Tester** la connectivité entre Pods
- ✅ **Résoudre** les problèmes DNS
- ✅ **Diagnostiquer** les problèmes de communication
- ✅ **Utiliser** les outils de debugging (busybox, netshoot, kubectl)
- ✅ **Analyser** les logs et events pour identifier les problèmes

### Mindset DevOps

- ✅ **Penser** en termes de services, pas de serveurs
- ✅ **Adopter** une approche méthodique de debugging
- ✅ **Comprendre** l'importance de l'observabilité
- ✅ **Apprécier** l'automatisation du réseau dans Kubernetes

## Prêt à plonger ?

Le réseau Kubernetes est un sujet fascinant qui transformera votre façon de penser l'infrastructure. C'est l'un des aspects les plus élégants de Kubernetes : une fois que vous comprenez les principes de base, tout devient logique et cohérent.

**Rappelez-vous :**
- La complexité apparente cache une simplicité élégante
- Chaque concept s'appuie sur le précédent
- La pratique transforme la théorie en intuition
- Les erreurs sont des opportunités d'apprentissage

**Citation inspirante :**

> "Le réseau Kubernetes n'est pas magique, c'est juste très bien conçu. Une fois que vous comprenez les fondations, tout le reste découle naturellement."
>
> — *Kelsey Hightower, Kubernetes Advocate*

## Structure du chapitre

Voici un aperçu visuel de la progression du chapitre :

```
Chapitre 7 : Réseau Kubernetes Interne
│
├── 7.1 Concepts de base ───────────────┐
│   (Théorie fondamentale)              │
│                                       │ Comprendre
├── 7.2 CNI (Calico) ───────────────────┤ les bases
│   (Implémentation technique)          │
│                                       │
├── 7.3 DNS avec CoreDNS ───────────────┘
│   (Découverte de services)
│
├── 7.4 Communication ──────────────────┐
│   (Patterns et pratiques)             │ Appliquer
│                                       │ les concepts
├── 7.5 Tests de connectivité ──────────┘
    (Diagnostic et debugging)
```

## Message de motivation

Vous êtes sur le point de maîtriser l'un des aspects les plus puissants de Kubernetes. Le réseau est ce qui permet à tous les autres composants de fonctionner ensemble harmonieusement.

**Prenez votre temps.** Ce n'est pas une course. La compréhension profonde vaut mieux qu'une lecture rapide.

**Soyez curieux.** Posez-vous des questions. Expérimentez. Cassez des choses (dans votre lab !). C'est comme ça qu'on apprend vraiment.

**Amusez-vous !** Le réseau Kubernetes est un système élégant et bien pensé. Apprécier sa beauté rend l'apprentissage plus agréable.

---

**Prêt ? Commençons par les concepts de base dans la section 7.1 !**

*Le voyage de mille kilomètres commence par un premier pas. Votre premier pas vers la maîtrise du réseau Kubernetes commence maintenant.*

⏭️ [Concepts de base du réseau Kubernetes](/07-reseau-kubernetes-interne/01-concepts-de-base-du-reseau-kubernetes.md)
