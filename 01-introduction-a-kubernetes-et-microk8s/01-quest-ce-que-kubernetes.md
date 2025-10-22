🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 1.1 Qu'est-ce que Kubernetes ?

## Introduction

Kubernetes (souvent abrégé **K8s**, car il y a 8 lettres entre le "K" et le "s") est un système open-source d'orchestration de conteneurs. Mais qu'est-ce que cela signifie concrètement ? Décomposons cette définition pour la rendre accessible.

## Comprendre les conteneurs avant Kubernetes

Avant de parler de Kubernetes, il faut comprendre ce qu'est un conteneur. Imaginez que vous ayez développé une application web. Cette application a besoin de :
- Un système d'exploitation spécifique
- Des bibliothèques logicielles particulières
- Des dépendances précises
- Une configuration spécifique

Traditionnellement, déployer cette application sur différents serveurs pouvait être compliqué. C'est là qu'interviennent les **conteneurs** (comme Docker). Un conteneur est comme une "boîte" qui contient votre application et tout ce dont elle a besoin pour fonctionner. Cette boîte peut être déplacée et exécutée n'importe où de manière identique.

## Le défi : gérer des dizaines ou centaines de conteneurs

Les conteneurs ont révolutionné le développement et le déploiement d'applications. Mais imaginez maintenant que vous ayez :
- 50 applications différentes
- Chacune tournant dans plusieurs conteneurs pour gérer la charge
- Des conteneurs qui tombent en panne et doivent être redémarrés
- Des besoins de mise à l'échelle (plus ou moins de conteneurs selon la charge)
- Des mises à jour à déployer sans interruption de service

Gérer tout cela manuellement devient rapidement impossible. **C'est exactement le problème que Kubernetes résout.**

## Kubernetes : le chef d'orchestre de vos conteneurs

Kubernetes agit comme un **chef d'orchestre** pour vos conteneurs. Tout comme un chef d'orchestre coordonne les musiciens pour créer une symphonie harmonieuse, Kubernetes coordonne vos conteneurs pour faire fonctionner vos applications de manière fluide et efficace.

### Les responsabilités de Kubernetes

Kubernetes prend en charge automatiquement :

**1. Le déploiement**
- Déployer vos applications sur les serveurs disponibles
- Distribuer intelligemment les conteneurs sur votre infrastructure

**2. La haute disponibilité**
- Surveiller constamment l'état de vos conteneurs
- Redémarrer automatiquement les conteneurs qui tombent en panne
- Remplacer les conteneurs défaillants sans intervention humaine

**3. La mise à l'échelle**
- Augmenter le nombre de conteneurs quand la charge augmente
- Réduire le nombre de conteneurs quand la charge diminue
- Adapter automatiquement les ressources selon les besoins

**4. L'équilibrage de charge**
- Distribuer le trafic entre plusieurs conteneurs
- Diriger les requêtes vers les conteneurs disponibles et en bonne santé

**5. Les mises à jour**
- Déployer de nouvelles versions sans interruption de service
- Effectuer des retours en arrière (rollback) si une mise à jour pose problème

**6. La gestion des ressources**
- Allouer de la mémoire et du CPU aux applications
- Optimiser l'utilisation de votre infrastructure

## Une analogie simple : Kubernetes comme un gestionnaire d'immeubles

Pensez à Kubernetes comme à un **gestionnaire d'immeubles automatisé** :

- **Vos applications** = les locataires
- **Les conteneurs** = les appartements où vivent les locataires
- **Les serveurs** = les immeubles contenant les appartements
- **Kubernetes** = le gestionnaire qui s'occupe de tout

Le gestionnaire (Kubernetes) :
- Attribue automatiquement des appartements (conteneurs) aux locataires (applications)
- Répare immédiatement les appartements endommagés
- Construit de nouveaux appartements quand il y a plus de demande
- S'assure que le courrier (le trafic réseau) arrive au bon appartement
- Gère les déménagements (migrations) sans déranger les locataires

## Pourquoi Kubernetes est devenu incontournable

Kubernetes s'est imposé comme la solution standard pour l'orchestration de conteneurs pour plusieurs raisons :

### Portable et flexible
Kubernetes fonctionne partout : sur votre ordinateur portable, dans votre datacenter, ou chez n'importe quel fournisseur cloud (AWS, Google Cloud, Azure, etc.). Vos applications peuvent être déplacées facilement d'un environnement à un autre.

### Écosystème riche
Une communauté mondiale active développe des milliers d'outils et d'extensions pour Kubernetes. Quelle que soit votre problématique, il existe probablement déjà une solution.

### Architecture déclarative
Au lieu de dire à Kubernetes "fais ceci, puis fais cela", vous lui décrivez simplement l'état souhaité de vos applications, et Kubernetes se charge de faire le nécessaire pour atteindre cet état. C'est comme dire "je veux 3 copies de mon application qui tourne" plutôt que "démarre un conteneur ici, puis un autre là".

### Standard industriel
Adopté par les plus grandes entreprises du monde, Kubernetes est devenu le standard de facto. Apprendre Kubernetes est donc un investissement précieux pour votre carrière.

## Les origines de Kubernetes

Kubernetes a été créé par Google et publié en open-source en 2014. Il est basé sur plus de 15 ans d'expérience de Google dans l'exécution de charges de travail en production à échelle massive. Google utilise des systèmes similaires (appelés Borg et Omega) pour faire fonctionner ses propres services (Gmail, YouTube, Google Search, etc.).

En 2015, Google a donné Kubernetes à la Cloud Native Computing Foundation (CNCF), une organisation neutre qui supervise désormais son développement. Aujourd'hui, des milliers de contributeurs du monde entier participent à l'amélioration continue de Kubernetes.

## Kubernetes en quelques chiffres

Pour comprendre l'ampleur de Kubernetes :
- Plus de **5,6 millions de développeurs** l'utilisent dans le monde
- Plus de **96% des organisations** utilisent ou évaluent Kubernetes
- Des **milliers d'entreprises** contribuent au projet
- Plus de **88 000 commits** sur le projet GitHub
- Utilisé par des géants comme **Spotify, Airbnb, Netflix, Uber, Twitter**

## Ce que Kubernetes n'est pas

Pour bien comprendre Kubernetes, il est aussi important de clarifier ce qu'il n'est **pas** :

**Kubernetes n'est pas une plateforme de conteneurisation**
- Il ne crée pas les conteneurs, il les orchestre
- Vous avez besoin d'un runtime de conteneurs comme Docker ou containerd

**Kubernetes n'est pas un service Platform-as-a-Service (PaaS) traditionnel**
- Il est plus flexible mais aussi plus complexe qu'un PaaS comme Heroku
- Vous gardez plus de contrôle, mais aussi plus de responsabilités

**Kubernetes n'est pas magique**
- Il ne résoudra pas les problèmes d'architecture mal conçue
- Il nécessite un apprentissage et une bonne compréhension pour être utilisé efficacement

**Kubernetes n'est pas obligatoire pour tout le monde**
- Pour de petites applications simples, il peut être trop complexe
- C'est un outil puissant, mais qui doit être utilisé à bon escient

## Quand utiliser Kubernetes ?

Kubernetes est particulièrement adapté si :
- Vous déployez des applications conteneurisées
- Vous avez besoin de haute disponibilité
- Vous devez gérer plusieurs applications ou microservices
- Vous souhaitez automatiser vos déploiements
- Vous avez besoin de mise à l'échelle automatique
- Vous voulez une infrastructure portable entre différents environnements
- Vous développez des applications cloud-native

## Conclusion

Kubernetes est un système d'orchestration de conteneurs qui automatise le déploiement, la mise à l'échelle et la gestion de vos applications conteneurisées. C'est devenu un standard incontournable dans le monde du développement et des opérations modernes.

Dans les prochaines sections, nous verrons comment **MicroK8s** rend Kubernetes accessible même pour un usage personnel ou un lab de développement, en simplifiant considérablement son installation et sa configuration.

Kubernetes peut sembler intimidant au premier abord, mais avec MicroK8s et une approche progressive, vous découvrirez qu'il devient rapidement un outil précieux et accessible. Bienvenue dans votre voyage d'apprentissage de Kubernetes !

---

**Points clés à retenir :**
- Kubernetes orchestre et automatise la gestion de conteneurs
- Il assure haute disponibilité, mise à l'échelle et déploiements automatisés
- C'est le standard industriel pour l'orchestration de conteneurs
- Il est portable, flexible et dispose d'un écosystème très riche
- MicroK8s va nous permettre de l'utiliser facilement pour notre apprentissage

⏭️ [Qu'est-ce que MicroK8s ?](/01-introduction-a-kubernetes-et-microk8s/02-quest-ce-que-microk8s.md)
