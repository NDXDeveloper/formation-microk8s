🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 4. Premiers Déploiements

## Introduction au Chapitre

Bienvenue dans le chapitre le plus pratique de la formation ! Après avoir installé MicroK8s et compris les concepts fondamentaux de Kubernetes, il est temps de **passer à l'action** et de déployer vos premières applications.

Ce chapitre est conçu comme un parcours progressif où vous découvrirez, étape par étape, comment déployer et gérer des applications dans Kubernetes. Vous commencerez par comprendre la structure des fichiers de configuration, puis vous déploierez une application web simple, l'exposerez au monde extérieur, et apprendrez à la configurer de manière sécurisée.

## Objectifs du Chapitre

À la fin de ce chapitre, vous serez capable de :

✅ **Lire et écrire** des manifestes YAML pour Kubernetes
✅ **Déployer** une application web avec Kubernetes
✅ **Exposer** votre application via différents types de Services
✅ **Configurer** vos applications avec des variables d'environnement
✅ **Gérer les secrets** et les données sensibles de manière sécurisée
✅ **Maîtriser** les commandes kubectl essentielles
✅ **Diagnostiquer et résoudre** les problèmes courants

## Pourquoi ce Chapitre est Important

Dans Kubernetes, **tout est déclaratif**. Vous ne cliquez pas dans une interface graphique pour créer vos applications, vous écrivez des fichiers de configuration (manifestes YAML) qui décrivent l'état désiré de votre système. Kubernetes se charge ensuite de faire en sorte que la réalité corresponde à cette description.

**Analogie :** Imaginez que vous engagez un assistant personnel. Au lieu de lui dire chaque matin « prépare le café, achète le journal, promène le chien », vous lui donnez une liste de tâches à effectuer quotidiennement. Kubernetes est cet assistant : vous lui donnez une description de ce que vous voulez, et il s'assure que c'est toujours le cas.

## Vue d'Ensemble des Sections

### 4.1 Anatomie d'un Manifeste YAML

La **fondation** de tout ce que vous ferez dans Kubernetes. Vous découvrirez :
- La structure des fichiers YAML
- Les quatre sections essentielles d'un manifeste
- Les règles d'indentation et de syntaxe
- Les bonnes pratiques de rédaction

**Pourquoi c'est important :** Sans comprendre YAML, impossible de travailler efficacement avec Kubernetes. Cette section vous donnera les bases solides nécessaires.

### 4.2 Déploiement d'une Application Web Simple

Votre **premier déploiement pratique** ! Vous apprendrez à :
- Créer un Pod simple
- Utiliser des Deployments pour la production
- Gérer les répliques et la résilience
- Effectuer des mises à jour progressives
- Revenir en arrière en cas de problème

**Ce que vous déploierez :** Une application Nginx, le serveur web le plus populaire au monde. Simple mais suffisamment réaliste pour comprendre tous les concepts.

### 4.3 Exposition d'une Application

Rendre votre application **accessible** ! Vous découvrirez :
- Le concept de Service dans Kubernetes
- Les trois types de Services : ClusterIP, NodePort, LoadBalancer
- Comment choisir le bon type selon vos besoins
- Le load balancing automatique
- L'activation de MetalLB pour les environnements locaux

**Le problème résolu :** Les Pods ont des IPs qui changent constamment. Les Services fournissent une adresse stable pour accéder à vos applications.

### 4.4 Variables d'Environnement

**Configurer** vos applications de manière flexible :
- Définir des variables d'environnement statiques
- Utiliser la Downward API pour injecter des métadonnées
- Gérer différents environnements (dev, staging, production)
- Suivre les meilleures pratiques de configuration

**Principe clé :** Séparer la configuration du code. Même image Docker, configuration différente selon l'environnement.

### 4.5 Gestion des Données Sensibles

**Sécuriser** vos secrets ! Section cruciale sur :
- Pourquoi ne jamais mettre de secrets en clair
- Créer et utiliser des Secrets Kubernetes
- Les différents types de Secrets
- Les bonnes pratiques de sécurité
- Les limitations et comment les dépasser

**Règle d'or :** Les mots de passe, clés API et certificats ne doivent JAMAIS apparaître en clair dans vos fichiers de configuration.

### 4.6 Commandes kubectl Essentielles

Votre **guide de référence** des commandes kubectl :
- Commandes de base (CRUD)
- Commandes de déploiement et de scaling
- Commandes d'inspection et de débogage
- Options globales et raccourcis
- Alias pour augmenter votre productivité

**Objectif :** Devenir efficace avec kubectl, l'outil que vous utiliserez tous les jours.

### 4.7 Inspection et Débogage de Base

**Résoudre les problèmes** de manière méthodique :
- Méthodologie de débogage systématique
- Comprendre les états des Pods
- Diagnostiquer les problèmes courants
- Utiliser les bons outils au bon moment
- Check-list de débogage rapide

**Compétence essentielle :** Dans le monde réel, les applications ne fonctionnent pas toujours du premier coup. Savoir déboguer est crucial.

## Prérequis

Avant de commencer ce chapitre, assurez-vous que :

✅ MicroK8s est **installé et fonctionne** sur votre machine
```bash
microk8s status
# Devrait afficher : microk8s is running
```

✅ Vous pouvez exécuter des **commandes kubectl**
```bash
microk8s kubectl get nodes
# Devrait afficher votre nœud
```

✅ Vous avez lu les **chapitres 1-3** (concepts de base de Kubernetes)

Si un de ces points n'est pas validé, revenez aux chapitres précédents pour assurer une bonne compréhension.

## Approche Pédagogique

Ce chapitre suit une approche **progressive et pratique** :

### 1. Apprentissage par l'exemple
Chaque concept est illustré par des exemples concrets que vous pouvez exécuter immédiatement sur votre cluster MicroK8s.

### 2. Du simple au complexe
Nous commençons par déployer un Pod simple, puis évoluons vers des Deployments complets avec toutes les bonnes pratiques.

### 3. Explication détaillée
Chaque ligne de code, chaque commande est expliquée. Vous ne copierez pas aveuglément, vous comprendrez **pourquoi**.

### 4. Mise en contexte
Pour chaque fonctionnalité, nous expliquons :
- **Le problème** qu'elle résout
- **Quand** l'utiliser
- **Comment** l'utiliser
- Les **pièges** à éviter

### 5. Bonnes pratiques dès le début
Nous ne vous apprenons pas seulement à faire fonctionner les choses, mais à les faire **correctement** dès le départ.

## Conventions Utilisées

### Commandes

Toutes les commandes sont préfixées par `microk8s kubectl` :
```bash
microk8s kubectl get pods
```

Si vous avez configuré un alias (`alias kubectl='microk8s kubectl'`), vous pouvez simplement utiliser :
```bash
kubectl get pods
```

### Exemples de code

Les manifestes YAML incluent des commentaires explicatifs :
```yaml
apiVersion: v1              # Version de l'API
kind: Pod                   # Type d'objet
metadata:
  name: mon-app            # Nom unique
```

### Avertissements et notes

**⚠️ Attention :** Indique quelque chose d'important à ne pas manquer

**💡 Astuce :** Conseils pratiques pour vous faciliter la vie

**✅ Bonne pratique :** La manière recommandée de faire les choses

**❌ À éviter :** Ce qu'il ne faut surtout pas faire

## Organisation des Fichiers

Pour suivre ce chapitre efficacement, nous vous recommandons de créer une structure de dossiers organisée :

```bash
# Créer un dossier pour vos fichiers
mkdir -p ~/kubernetes-learning/chapter-4
cd ~/kubernetes-learning/chapter-4

# Créer des sous-dossiers par section
mkdir manifests secrets configs
```

**Structure recommandée :**
```
~/kubernetes-learning/chapter-4/
├── manifests/
│   ├── 4.1-example-pod.yaml
│   ├── 4.2-nginx-deployment.yaml
│   └── 4.3-nginx-service.yaml
├── secrets/
│   └── 4.5-db-credentials.yaml
└── configs/
    └── notes.md
```

Cette organisation vous aidera à retrouver facilement vos fichiers et à maintenir un historique de votre apprentissage.

## Comment Utiliser ce Chapitre

### Pour les débutants complets
Suivez les sections **dans l'ordre**, sans sauter d'étapes. Chaque section s'appuie sur la précédente.

### Pour ceux qui ont déjà de l'expérience
Vous pouvez utiliser ce chapitre comme **référence**. Les sections 4.6 (Commandes kubectl) et 4.7 (Débogage) sont particulièrement utiles comme aide-mémoire.

### Conseils pratiques

1. **Tapez les commandes vous-même**
   Ne copiez-collez pas aveuglément. Taper aide à mémoriser.

2. **Expérimentez**
   Modifiez les exemples, cassez des choses, réparez-les. C'est en se trompant qu'on apprend le mieux.

3. **Prenez des notes**
   Documentez ce que vous apprenez avec vos propres mots.

4. **Pratiquez régulièrement**
   Mieux vaut 30 minutes par jour qu'une session marathon de 4 heures.

5. **N'hésitez pas à revenir en arrière**
   Si quelque chose n'est pas clair, relisez la section précédente.

## Ce que Vous Allez Créer

Au fil de ce chapitre, vous construirez progressivement une **application web complète et fonctionnelle** :

**Étape 1 (4.2)** : Déploiement d'un serveur web Nginx
```
┌─────────────┐
│  Nginx Pod  │
└─────────────┘
```

**Étape 2 (4.3)** : Exposition via un Service
```
┌──────────┐      ┌─────────────┐
│ Service  │─────▶│  Nginx Pod  │
└──────────┘      └─────────────┘
```

**Étape 3 (4.2)** : Haute disponibilité avec répliques
```
┌──────────┐      ┌─────────────┐
│          │─────▶│  Nginx Pod  │
│ Service  │─────▶│  Nginx Pod  │
│          │─────▶│  Nginx Pod  │
└──────────┘      └─────────────┘
```

**Étape 4 (4.4-4.5)** : Configuration et secrets
```
┌──────────┐      ┌─────────────┐      ┌────────────┐
│          │      │             │      │ ConfigMaps │
│ Service  │─────▶│  Nginx Pod  │◀─────┤            │
│          │      │             │      │  Secrets   │
└──────────┘      └─────────────┘      └────────────┘
```

À la fin du chapitre, vous aurez une application web résiliente, scalable, configurable et sécurisée !

## Temps Estimé

**Temps total pour le chapitre :** 6-8 heures

Répartition par section :
- 4.1 Anatomie YAML : 1h
- 4.2 Déploiement : 1h30
- 4.3 Exposition : 1h30
- 4.4 Variables d'env : 1h
- 4.5 Secrets : 1h
- 4.6 Commandes kubectl : 1h (référence)
- 4.7 Débogage : 1h30

**Conseil :** Ne vous pressez pas. Prenez le temps de bien comprendre chaque concept avant de passer au suivant.

## Aide et Support

### Si vous êtes bloqué

1. **Relisez attentivement** les messages d'erreur
2. Consultez la **section 4.7 (Débogage)**
3. Vérifiez que vous avez bien suivi **toutes les étapes**
4. Utilisez `microk8s inspect` pour diagnostiquer les problèmes de cluster

### Ressources complémentaires

- **Documentation officielle Kubernetes** : https://kubernetes.io/docs/
- **Documentation MicroK8s** : https://microk8s.io/docs
- **Kubectl Cheat Sheet** : https://kubernetes.io/docs/reference/kubectl/cheatsheet/

## À Vous de Jouer !

Vous avez maintenant une vue d'ensemble complète de ce qui vous attend dans ce chapitre. Chaque section a été conçue pour vous donner les compétences pratiques nécessaires pour déployer et gérer des applications dans Kubernetes.

**Êtes-vous prêt ?** Alors commençons par le commencement : comprendre la structure des manifestes YAML !

---

**Prochaine section :** 4.1 Anatomie d'un manifeste YAML

Nous allons découvrir le langage que Kubernetes comprend : les manifestes YAML. C'est la fondation de tout ce que vous ferez dans Kubernetes, alors prenons le temps de bien comprendre cette structure essentielle.

⏭️ [Anatomie d'un manifeste YAML](/04-premiers-deploiements/01-anatomie-dun-manifeste-yaml.md)
