🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 3. Concepts Kubernetes Essentiels

## Introduction au chapitre

Félicitations ! Vous avez installé MicroK8s et vous êtes prêt à découvrir le cœur de Kubernetes. Dans ce chapitre, nous allons explorer les **concepts fondamentaux** qui constituent la base de toute utilisation de Kubernetes.

Ces concepts ne sont pas seulement théoriques : ce sont les **briques de construction** que vous utiliserez quotidiennement pour déployer, gérer et faire évoluer vos applications. Maîtriser ces notions est essentiel pour devenir autonome avec Kubernetes.

## Pourquoi ces concepts sont-ils essentiels ?

Kubernetes peut sembler complexe au premier abord, avec son vocabulaire spécifique et ses nombreux objets. Cependant, une fois que vous comprendrez les concepts de base présentés dans ce chapitre, tout le reste deviendra beaucoup plus clair.

Pensez à Kubernetes comme à un **langage** :
- Les concepts de ce chapitre sont le **vocabulaire de base**
- Sans eux, vous ne pourriez pas "parler" à Kubernetes
- Une fois maîtrisés, vous pourrez exprimer des architectures complexes de manière simple

## Vue d'ensemble du chapitre

Ce chapitre couvre **sept concepts essentiels** que vous devez absolument maîtriser :

### 1. Les Pods : la brique élémentaire
Le Pod est l'unité de base dans Kubernetes. C'est le plus petit objet déployable, une sorte de "conteneur de conteneurs". Vous apprendrez :
- Ce qu'est un Pod et pourquoi il existe
- Comment les conteneurs cohabitent dans un Pod
- Le cycle de vie d'un Pod
- Pourquoi on ne crée presque jamais de Pods directement

### 2. Les Deployments et ReplicaSets : la gestion automatisée
Les Deployments sont les contrôleurs qui gèrent vos Pods de manière intelligente et automatisée. Vous découvrirez :
- Comment maintenir automatiquement un nombre désiré de Pods
- Les mises à jour progressives sans interruption de service
- Le rollback en cas de problème
- Le scaling (augmenter/diminuer le nombre de Pods)

### 3. Les Services : la découverte et l'exposition
Les Services résolvent le problème de communication entre les Pods et vers l'extérieur. Vous comprendrez :
- Comment obtenir une IP stable pour accéder à vos applications
- Les trois types de Services (ClusterIP, NodePort, LoadBalancer)
- La résolution DNS automatique
- Comment exposer vos applications au monde extérieur

### 4. Les Namespaces : l'organisation logique
Les Namespaces permettent de diviser votre cluster en espaces logiques isolés. Vous apprendrez :
- Comment organiser vos ressources de manière claire
- La séparation des environnements (dev, staging, production)
- Les quotas de ressources par namespace
- Les bonnes pratiques d'organisation

### 5. Les Labels et Selectors : le système d'étiquetage
Les Labels sont le système d'organisation et de sélection de Kubernetes. Vous verrez :
- Comment étiqueter vos ressources pour les identifier
- Comment les Selectors permettent de filtrer et sélectionner des ressources
- Le lien vital entre Services, Deployments et Pods
- Les patterns d'organisation avancés

### 6. Les ConfigMaps : la configuration externalisée
Les ConfigMaps séparent la configuration du code de l'application. Vous découvrirez :
- Comment externaliser la configuration de vos applications
- La réutilisation de la même image Docker dans tous les environnements
- L'injection de configuration via variables d'environnement ou fichiers
- La gestion multi-environnements simplifiée

### 7. Les Secrets : la gestion des données sensibles
Les Secrets sont dédiés au stockage des informations confidentielles. Vous comprendrez :
- La différence entre ConfigMaps et Secrets
- Comment stocker des mots de passe, clés API et certificats
- Les bonnes pratiques de sécurité
- Les limites et comment aller plus loin

## La hiérarchie des concepts

Ces sept concepts ne sont pas isolés, ils forment un **écosystème cohérent** :

```
┌───────────────────────────────────────────────────────────┐
│                      CLUSTER KUBERNETES                   │
│                                                           │
│  ┌─────────────────────────────────────────────────────┐  │
│  │                    NAMESPACE                        │  │
│  │                                                     │  │
│  │  ┌─────────────────┐          ┌──────────────────┐  │  │
│  │  │   DEPLOYMENT    │          │     SERVICE      │  │  │
│  │  │  (Gère les Pods)│          │  (Expose les Pods│  │  │
│  │  │                 │          │                  │  │  │
│  │  │  Labels ────────┼──────────┼─→ Selectors      │  │  │
│  │  │                 │          │                  │  │  │
│  │  │   ┌──────────┐  │          └──────────────────┘  │  │
│  │  │   │   POD    │  │                                │  │
│  │  │   │          │  │                                │  │
│  │  │   │ ┌──────┐ │  │    ┌────────────┐              │  │
│  │  │   │ │ App  │ │  │    │ ConfigMap  │──────┐       │  │
│  │  │   │ └──────┘ │  │    └────────────┘      │       │  │
│  │  │   │          │  │                        │       │  │
│  │  │   │ Config ←─┼──┼────────────────────────┘       │  │
│  │  │   │ Secret ←─┼──┼────┐                           │  │
│  │  │   └──────────┘  │    │  ┌────────────┐           │  │
│  │  └─────────────────┘    └──│   Secret   │           │  │
│  │                            └────────────┘           │  │
│  └─────────────────────────────────────────────────────┘  │
└───────────────────────────────────────────────────────────┘
```

**Lecture du schéma** :
1. Le **Cluster** contient tout
2. Les **Namespaces** organisent les ressources
3. Les **Deployments** créent et gèrent des **Pods**
4. Les **Services** exposent les Pods sélectionnés via les **Labels**
5. Les **ConfigMaps** et **Secrets** injectent la configuration dans les Pods

## Comment aborder ce chapitre ?

### Approche progressive

Ce chapitre est conçu pour être parcouru **dans l'ordre**. Chaque concept s'appuie sur les précédents :

1. **Commencez par les Pods** : la brique de base
2. **Puis les Deployments** : comment gérer les Pods
3. **Ensuite les Services** : comment exposer les Pods
4. **Les Namespaces** : comment organiser tout ça
5. **Les Labels et Selectors** : comment tout connecter
6. **Les ConfigMaps** : comment configurer vos applications
7. **Enfin les Secrets** : comment sécuriser les données sensibles

### Ne vous précipitez pas

**Important** : Prenez le temps de bien comprendre chaque concept avant de passer au suivant. Ces notions sont utilisées dans **100%** des déploiements Kubernetes, il est donc crucial de bien les maîtriser.

Ne vous inquiétez pas si certains concepts semblent abstraits au début. Au fur et à mesure que vous avancerez dans le chapitre, les pièces du puzzle s'assembleront naturellement.

### Approche pratique

Bien que ce tutoriel n'inclue pas d'exercices pratiques formels, nous vous encourageons vivement à :
- **Tester les exemples** fournis dans chaque section
- **Expérimenter** avec vos propres variations
- **Observer** ce qui se passe quand vous modifiez les configurations
- **Faire des erreurs** : c'est la meilleure façon d'apprendre !

## À quoi vous attendre

### Ce que vous saurez faire à la fin de ce chapitre

Après avoir parcouru ce chapitre, vous serez capable de :

✅ **Comprendre** l'architecture et les concepts fondamentaux de Kubernetes

✅ **Déployer** des applications conteneurisées de manière fiable et scalable

✅ **Exposer** vos applications pour les rendre accessibles

✅ **Organiser** vos ressources de manière logique et professionnelle

✅ **Configurer** vos applications pour différents environnements

✅ **Gérer** les données sensibles de manière sécurisée

✅ **Lire et comprendre** des manifestes YAML Kubernetes

✅ **Déboguer** les problèmes de base dans vos déploiements

### Les compétences transversales

Au-delà des concepts techniques, vous développerez également :

- **La pensée déclarative** : décrire l'état désiré plutôt que les étapes
- **La compréhension des systèmes distribués** : comment les composants communiquent
- **Les bonnes pratiques DevOps** : organisation, versioning, documentation
- **Le vocabulaire Kubernetes** : pour communiquer avec la communauté

## Manifestes YAML : le langage de Kubernetes

Tout au long de ce chapitre, vous rencontrerez de nombreux **fichiers YAML**. C'est le format utilisé pour décrire vos ressources Kubernetes.

### Structure typique d'un manifeste

Tous les manifestes Kubernetes suivent une structure similaire :

```yaml
apiVersion: <version-api>      # Version de l'API Kubernetes
kind: <type-ressource>         # Type d'objet (Pod, Deployment, Service, etc.)
metadata:                      # Métadonnées (nom, labels, etc.)
  name: <nom>
  labels:
    <clé>: <valeur>
spec:                         # Spécification (ce que vous voulez)
  <configuration>
```

**Exemple simple** :
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: mon-premier-pod
  labels:
    app: nginx
spec:
  containers:
  - name: nginx
    image: nginx:1.25
```

Ne vous inquiétez pas si cela vous semble cryptique pour l'instant. À la fin de ce chapitre, vous lirez et écrirez du YAML naturellement !

## Commandes kubectl de base

Vous utiliserez fréquemment la commande `kubectl` (ou `microk8s kubectl`) pour interagir avec votre cluster. Voici quelques commandes que vous rencontrerez souvent :

```bash
# Créer une ressource depuis un fichier YAML
microk8s kubectl apply -f mon-fichier.yaml

# Lister des ressources
microk8s kubectl get pods
microk8s kubectl get deployments
microk8s kubectl get services

# Voir les détails d'une ressource
microk8s kubectl describe pod mon-pod

# Voir les logs d'un Pod
microk8s kubectl logs mon-pod

# Supprimer une ressource
microk8s kubectl delete pod mon-pod
```

Ces commandes deviendront rapidement naturelles au fil du chapitre.

## État d'esprit pour apprendre Kubernetes

### Penser "déclaratif"

Kubernetes fonctionne de manière **déclarative** :
- Vous décrivez **l'état désiré** (ce que vous voulez)
- Kubernetes s'occupe de **réaliser** cet état
- Kubernetes **maintient** continuellement cet état

**Exemple** :
```yaml
# Vous déclarez : "Je veux 3 Pods nginx"
spec:
  replicas: 3
```

Kubernetes :
1. Crée 3 Pods
2. Si un Pod meurt → en recrée un automatiquement
3. Si vous changez à `replicas: 5` → ajoute 2 Pods
4. Surveille et maintient continuellement l'état désiré

C'est très différent des approches impératives ("fais ceci, puis fais cela"), et c'est l'un des grands avantages de Kubernetes.

### Accepter la courbe d'apprentissage

Kubernetes a une réputation de complexité, et c'est vrai qu'il y a beaucoup à apprendre. Cependant :

- Les concepts de base (ce chapitre) sont **abordables** et logiques
- Une fois les fondations posées, le reste suit naturellement
- La communauté est **immense** et toujours prête à aider
- Les bénéfices en termes de scalabilité et de fiabilité sont **énormes**

**Notre promesse** : Si vous suivez ce chapitre attentivement et testez les exemples, vous aurez une base solide pour continuer votre apprentissage.

## Conventions utilisées dans ce chapitre

### Exemples de code

Les exemples de code sont clairement marqués :

```yaml
# Ceci est un exemple de fichier YAML
apiVersion: v1
kind: Pod
metadata:
  name: exemple
```

```bash
# Ceci est une commande à exécuter
microk8s kubectl get pods
```

### Avertissements importants

Les points critiques sont marqués ainsi :

**⚠️ Attention** : Ceci est important et peut causer des problèmes si ignoré

### Bonnes et mauvaises pratiques

Nous utilisons des icônes pour clarifier :

**✅ Bon** : Bonne pratique recommandée
**❌ Mauvais** : À éviter

### Informations supplémentaires

Les notes et astuces sont formatées ainsi :

**Note** : Information complémentaire utile

**Astuce** : Conseil pratique pour gagner du temps

## Ressources complémentaires

Pendant votre lecture de ce chapitre, vous pourriez trouver utiles :

- **Documentation officielle Kubernetes** : https://kubernetes.io/docs/
- **Documentation MicroK8s** : https://microk8s.io/docs
- **Kubernetes Cheat Sheet** : Liste de commandes kubectl courantes
- **Communauté Kubernetes** : Forums, Slack, Stack Overflow

N'hésitez pas à consulter ces ressources si vous souhaitez approfondir un point particulier.

## Prêt à commencer ?

Vous avez maintenant une vue d'ensemble de ce qui vous attend dans ce chapitre. Les sept concepts que vous allez découvrir sont les **piliers fondamentaux** de Kubernetes.

Chaque section est conçue pour être :
- **Progressive** : du simple vers le complexe
- **Pratique** : avec de nombreux exemples concrets
- **Accessible** : même si vous débutez totalement avec Kubernetes

Prenez votre temps, expérimentez, et surtout : amusez-vous ! Kubernetes est un outil puissant qui, une fois maîtrisé, transformera votre manière de déployer et gérer des applications.

---

**Commençons par le concept le plus fondamental : les Pods.**

→ Passez à la section **3.1 Pods : l'unité de base**

⏭️ [Pods : l'unité de base](/03-concepts-kubernetes-essentiels/01-pods-lunite-de-base.md)
