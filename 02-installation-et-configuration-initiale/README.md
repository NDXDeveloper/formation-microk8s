🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 2. Installation et Configuration Initiale

## Introduction au Chapitre

Bienvenue dans le chapitre consacré à l'installation et à la configuration initiale de MicroK8s ! C'est ici que votre voyage pratique avec Kubernetes commence réellement. Après avoir compris les concepts théoriques du chapitre précédent, vous allez maintenant mettre les mains dans le cambouis et installer votre propre cluster Kubernetes.

Ce chapitre est conçu pour vous guider pas à pas, quel que soit votre système d'exploitation ou votre niveau d'expérience. Nous avons structuré le contenu de manière progressive pour que même les débutants complets puissent suivre sans difficulté.

## Pourquoi ce Chapitre est Crucial

L'installation et la configuration initiale constituent les fondations de votre apprentissage Kubernetes. Une installation bien réalisée vous permettra de :

**Éviter les problèmes futurs** : Une installation correcte dès le départ élimine 80% des problèmes que vous pourriez rencontrer plus tard.

**Gagner du temps** : En suivant les bonnes pratiques dès le début, vous économiserez des heures de débogage par la suite.

**Comprendre l'architecture** : Le processus d'installation vous permet de découvrir comment les différents composants de Kubernetes s'articulent.

**Acquérir de la confiance** : Réussir votre première installation vous donnera la confiance nécessaire pour explorer des concepts plus avancés.

**Créer un environnement de test fiable** : Un cluster bien configuré est essentiel pour expérimenter en toute sérénité.

## Philosophie de MicroK8s : La Simplicité avant Tout

Avant de commencer, il est important de comprendre pourquoi MicroK8s est particulièrement adapté aux débutants et aux environnements de développement :

### Installation minimale

Contrairement à d'autres distributions Kubernetes qui nécessitent de nombreuses étapes de configuration, MicroK8s adopte une approche "batteries included" :
- Une seule commande d'installation
- Configuration par défaut fonctionnelle
- Démarrage automatique après installation

### Gestion simplifiée

MicroK8s encapsule toute la complexité de Kubernetes derrière des commandes simples :
- `microk8s status` pour vérifier l'état
- `microk8s enable addon` pour ajouter des fonctionnalités
- `microk8s inspect` pour diagnostiquer les problèmes

### Addons intégrés

Une des forces majeures de MicroK8s est son système d'addons. Au lieu de passer des heures à configurer manuellement des composants comme :
- Le DNS (CoreDNS)
- Le stockage persistant
- Les certificats SSL/TLS
- Le monitoring (Prometheus)
- Le load balancing (MetalLB)

...vous pouvez simplement exécuter `microk8s enable dns` ou `microk8s enable prometheus`. C'est cette simplicité qui vous permettra de vous concentrer sur l'apprentissage de Kubernetes plutôt que sur sa configuration.

### Isolation et sécurité

MicroK8s utilise la technologie Snap sur Linux, ce qui apporte :
- **Isolation** : MicroK8s fonctionne dans son propre environnement isolé
- **Mises à jour automatiques** : Les nouvelles versions sont déployées automatiquement (configurable)
- **Rollback facile** : Retour à une version précédente en cas de problème
- **Pas de pollution système** : L'installation et la désinstallation sont propres

## Aperçu du Contenu du Chapitre

Ce chapitre vous guidera à travers toutes les étapes nécessaires pour avoir un cluster MicroK8s pleinement opérationnel. Voici ce que nous allons couvrir :

### 2.1 Prérequis Système

Avant d'installer MicroK8s, vous devez vous assurer que votre machine répond aux exigences minimales. Cette section détaille :
- Les besoins en CPU, RAM et stockage
- Les différences selon les cas d'usage (développement, production, lab)
- Comment vérifier que votre système est compatible

**Pourquoi c'est important** : Installer MicroK8s sur une machine sous-dimensionnée conduira inévitablement à des problèmes de performance et de stabilité.

### 2.2 Systèmes d'Exploitation Supportés

MicroK8s fonctionne sur une grande variété de systèmes d'exploitation. Cette section présente :
- Les distributions Linux supportées
- Les versions Windows compatibles
- macOS et ses particularités
- Les considérations spécifiques à chaque plateforme

**Pourquoi c'est important** : Chaque système d'exploitation a ses spécificités, et connaître les particularités de votre OS vous évitera des surprises.

### 2.3 Installation sur Linux (Ubuntu/Debian)

Guide détaillé d'installation sur les distributions basées sur Debian, incluant :
- Installation via Snap (méthode recommandée)
- Configuration des permissions utilisateur
- Vérification post-installation
- Résolution des problèmes courants

**Pourquoi c'est important** : Ubuntu est la plateforme de prédilection pour MicroK8s, c'est l'environnement le mieux supporté.

### 2.4 Installation sur CentOS/RHEL

Instructions spécifiques pour les distributions Red Hat :
- Particularités de CentOS/RHEL
- Installation de Snap sur ces systèmes
- Configuration SELinux
- Adaptations nécessaires

**Pourquoi c'est important** : Les distributions Red Hat sont très utilisées en entreprise, connaître ces spécificités est donc essentiel.

### 2.5 Installation sur Windows (WSL2)

Guide complet pour utiliser MicroK8s sur Windows :
- Activation et configuration de WSL2
- Installation d'Ubuntu dans WSL2
- Installation de MicroK8s dans l'environnement Linux
- Intégration avec Windows

**Pourquoi c'est important** : De nombreux développeurs travaillent sous Windows, WSL2 offre une excellente expérience Kubernetes sur cette plateforme.

### 2.6 Installation sur macOS

Instructions détaillées pour les utilisateurs Mac :
- Installation de Multipass (gestionnaire de VM)
- Installation de MicroK8s via Homebrew
- Configuration de la machine virtuelle
- Accès depuis macOS

**Pourquoi c'est important** : macOS nécessite une approche particulière avec virtualisation, ce guide vous permettra de l'installer correctement.

### 2.7 Vérification de l'Installation

Étapes essentielles pour confirmer que tout fonctionne :
- Tests de base du cluster
- Vérification des composants système
- Validation de la connectivité
- Interprétation des résultats

**Pourquoi c'est important** : Valider votre installation dès le début vous assure de partir sur de bonnes bases et d'éviter des problèmes plus tard.

### 2.8 Commandes MicroK8s Essentielles

Introduction aux commandes que vous utiliserez quotidiennement :
- `microk8s status` : Vérifier l'état du cluster
- `microk8s inspect` : Diagnostiquer les problèmes
- `microk8s enable` : Activer des fonctionnalités
- Et bien d'autres commandes indispensables

**Pourquoi c'est important** : Maîtriser ces commandes de base vous rendra autonome dans la gestion de votre cluster.

### 2.9 Configuration des Alias kubectl

Optimisation de votre workflow avec des raccourcis :
- Création d'alias pour gagner du temps
- Configuration selon votre shell (Bash, Zsh, Fish)
- Alias recommandés par la communauté
- Bonnes pratiques d'organisation

**Pourquoi c'est important** : Les alias transformeront `microk8s kubectl get pods --all-namespaces` en simple `kgpa`, économisant des heures de frappe.

### 2.10 Premier Diagnostic Complet

Réalisation d'un diagnostic approfondi de votre installation :
- Utilisation de `microk8s inspect`
- Analyse du rapport généré
- Identification des métriques importantes
- Conservation et documentation

**Pourquoi c'est important** : Ce diagnostic établit une "baseline" de votre système sain, essentielle pour le dépannage futur.

## Méthodologie de ce Chapitre

Tout au long de ce chapitre, nous suivrons une méthodologie éprouvée :

### 1. Préparation

Avant chaque installation, nous vérifierons que vous avez tout ce qu'il faut :
- Prérequis système
- Outils nécessaires
- Accès et permissions

### 2. Installation

Étapes détaillées et commentées :
- Commandes exactes à exécuter
- Explication de ce que fait chaque commande
- Sortie attendue de chaque commande
- Temps d'exécution approximatif

### 3. Vérification

Après chaque étape importante :
- Tests de validation
- Interprétation des résultats
- Confirmation que tout fonctionne

### 4. Dépannage

Pour chaque installation, nous fournirons :
- Problèmes courants et leurs solutions
- Messages d'erreur typiques
- Comment obtenir de l'aide

## Recommandations Avant de Commencer

### Choisissez votre plateforme

Si vous avez le choix, voici nos recommandations :

**Pour apprendre Kubernetes** :
- 1er choix : Ubuntu Linux (natif ou VM)
- 2ème choix : macOS avec Multipass
- 3ème choix : Windows avec WSL2

**Pour un lab permanent** :
- Machine dédiée sous Ubuntu Server
- Raspberry Pi avec Ubuntu (ARM)
- VM sur un hyperviseur (Proxmox, ESXi)

**Pour du développement quotidien** :
- Votre système d'exploitation principal avec MicroK8s
- Configuration minimale : 4 Go RAM, 2 CPU, 20 Go disque

### Préparez votre environnement

Avant de commencer l'installation :

**Temps nécessaire** : Prévoyez 30 minutes à 1 heure pour l'installation complète et la configuration initiale.

**Connexion internet** : Une connexion stable est nécessaire pour télécharger MicroK8s et les images Docker (plusieurs Go de données).

**Droits administrateur** : Vous aurez besoin des droits sudo/administrateur pour l'installation.

**Terminal/Console** : Familiarisez-vous avec le terminal de votre système d'exploitation, nous l'utiliserons beaucoup.

### Gardez ce guide ouvert

Ayez ce guide sous les yeux pendant l'installation :
- Ne sautez pas d'étapes
- Lisez les explications, pas seulement les commandes
- Vérifiez les résultats après chaque étape
- N'hésitez pas à revenir en arrière si nécessaire

### Prenez des notes

Documentez votre installation :
- Notez les commandes que vous exécutez
- Conservez les messages d'erreur éventuels
- Enregistrez les versions installées
- Créez un journal de bord de votre installation

Ces notes vous seront précieuses pour :
- Reproduire l'installation sur une autre machine
- Dépanner des problèmes futurs
- Partager votre expérience avec la communauté

## Philosophie d'Apprentissage

Ce chapitre suit une approche pédagogique spécifique :

### Apprendre en faisant

Nous privilégions la pratique immédiate. Chaque concept est suivi d'actions concrètes. Vous n'apprendrez pas Kubernetes en lisant seulement, mais en manipulant un vrai cluster.

### Comprendre plutôt que mémoriser

Nous expliquons **pourquoi** nous faisons les choses, pas seulement **comment**. Cette compréhension profonde vous rendra autonome.

### Progressivité

Nous commençons par les bases absolues et augmentons progressivement la complexité. Chaque section s'appuie sur les précédentes.

### Pas de magie noire

Toutes les commandes sont expliquées. Nous ne vous demandons jamais d'exécuter quelque chose "en aveugle". Vous comprendrez ce que fait chaque commande.

### Gestion des erreurs

Les erreurs font partie de l'apprentissage. Nous documentons les problèmes courants et leurs solutions, car vous les rencontrerez probablement.

## Ce qui Vous Attend Après ce Chapitre

Une fois ce chapitre terminé, vous aurez :

✅ Un cluster Kubernetes pleinement fonctionnel
✅ Une compréhension de l'architecture MicroK8s
✅ Les commandes essentielles maîtrisées
✅ Un environnement optimisé avec des alias
✅ Un diagnostic complet de votre installation
✅ La confiance pour passer aux concepts Kubernetes

Vous serez alors prêt pour :
- **Chapitre 3** : Comprendre les concepts Kubernetes (pods, services, deployments)
- **Chapitre 4** : Déployer vos premières applications
- **Chapitre 5** : Explorer les addons MicroK8s

## Structure des Sections

Chaque section de ce chapitre suit une structure cohérente :

**Introduction** : Contexte et objectifs de la section

**Prérequis** : Ce que vous devez avoir/savoir avant de commencer

**Instructions pas à pas** : Procédure détaillée avec commandes et explications

**Vérification** : Comment confirmer que tout fonctionne

**Dépannage** : Solutions aux problèmes courants

**Récapitulatif** : Ce que vous avez appris et accompli

## Symboles et Conventions

Tout au long de ce chapitre, nous utilisons des symboles pour faciliter la lecture :

✅ **Succès** : Résultat attendu, tout va bien
❌ **Erreur** : Problème à résoudre
⚠️ **Attention** : Point important à noter
💡 **Astuce** : Conseil pour gagner du temps ou améliorer votre configuration
📝 **Note** : Information complémentaire utile
🔧 **Technique** : Détail technique pour les curieux
🚀 **Prochaine étape** : Ce qui vient ensuite

Les commandes à exécuter sont toujours présentées dans des blocs comme ceci :

```bash
# Ceci est une commande à exécuter
microk8s status
```

Les sorties attendues sont présentées ainsi :

```
microk8s is running
```

## Adapter le Chapitre à Votre Situation

Ce chapitre couvre plusieurs systèmes d'exploitation. Vous n'avez pas besoin de tout lire :

**Si vous utilisez Ubuntu/Debian** :
- Sections 2.1, 2.2, 2.3, puis 2.7 à 2.10

**Si vous utilisez CentOS/RHEL** :
- Sections 2.1, 2.2, 2.4, puis 2.7 à 2.10

**Si vous utilisez Windows** :
- Sections 2.1, 2.2, 2.5, puis 2.7 à 2.10

**Si vous utilisez macOS** :
- Sections 2.1, 2.2, 2.6, puis 2.7 à 2.10

Les sections 2.7 à 2.10 sont communes à tous les systèmes d'exploitation et sont essentielles.

## Objectifs d'Apprentissage du Chapitre

À la fin de ce chapitre, vous serez capable de :

**Compétences techniques** :
- [ ] Vérifier les prérequis système pour MicroK8s
- [ ] Installer MicroK8s sur votre système d'exploitation
- [ ] Démarrer et arrêter le cluster
- [ ] Vérifier l'état du cluster avec `microk8s status`
- [ ] Diagnostiquer des problèmes avec `microk8s inspect`
- [ ] Activer et désactiver des addons
- [ ] Utiliser kubectl via MicroK8s
- [ ] Configurer des alias pour gagner en productivité
- [ ] Interpréter un rapport de diagnostic

**Compétences conceptuelles** :
- [ ] Comprendre l'architecture de MicroK8s
- [ ] Différencier MicroK8s des autres distributions Kubernetes
- [ ] Connaître les composants essentiels d'un cluster
- [ ] Identifier les métriques importantes à surveiller

**Autonomie** :
- [ ] Installer MicroK8s sans assistance
- [ ] Résoudre les problèmes d'installation courants
- [ ] Optimiser votre configuration personnelle
- [ ] Savoir où chercher de l'aide quand nécessaire

## Support et Ressources

Pendant votre parcours dans ce chapitre, vous avez accès à :

**Documentation officielle** :
- Site MicroK8s : https://microk8s.io/
- Documentation : https://microk8s.io/docs/
- GitHub : https://github.com/canonical/microk8s

**Communauté** :
- Forum Kubernetes : https://discuss.kubernetes.io/
- Slack Kubernetes : #microk8s
- Stack Overflow : tag `microk8s`

**Ressources additionnelles** :
- Tutoriels officiels Canonical
- Vidéos et webinaires
- Articles de blog de la communauté

N'hésitez jamais à consulter ces ressources ou à poser des questions. La communauté Kubernetes est réputée pour être accueillante et prête à aider les débutants.

## État d'Esprit pour ce Chapitre

Avant de commencer, adoptez ces principes :

**Patience** : L'installation peut prendre du temps, surtout lors du téléchargement. Ne vous précipitez pas.

**Attention aux détails** : Une petite erreur de frappe peut causer des problèmes. Relisez les commandes avant de les exécuter.

**Expérimentation** : N'ayez pas peur de tester. MicroK8s est conçu pour être facilement réinitialisé si nécessaire.

**Documentation** : Prenez des notes sur ce qui fonctionne (et ce qui ne fonctionne pas) pour vous.

**Persévérance** : Si vous rencontrez un problème, référez-vous à la section dépannage ou demandez de l'aide. Chaque problème a une solution.

## Mot d'Encouragement Final

L'installation et la configuration initiale peuvent sembler intimidantes, surtout si c'est votre première expérience avec Kubernetes. C'est normal ! Tout le monde est passé par là.

Rappelez-vous que :
- **Vous n'êtes pas seul** : Des milliers de personnes ont suivi ce même chemin
- **C'est plus simple qu'il n'y paraît** : MicroK8s est conçu pour simplifier au maximum le processus
- **Chaque expert était un jour débutant** : Même les meilleurs ont commencé par installer leur premier cluster
- **L'erreur est formatrice** : Chaque problème résolu renforce votre compréhension

Prenez votre temps, suivez les étapes, et dans une heure environ, vous aurez votre propre cluster Kubernetes fonctionnel. C'est excitant, non ?

## Prêt à Commencer ?

Vous avez maintenant une vision claire de ce qui vous attend dans ce chapitre. Vous comprenez :
- Pourquoi l'installation est une étape cruciale
- Ce qui rend MicroK8s particulièrement adapté aux débutants
- Ce que vous allez apprendre
- Comment aborder chaque section
- Où trouver de l'aide si nécessaire

Il est temps de passer à l'action ! Commençons par vérifier que votre système répond aux prérequis nécessaires pour faire tourner MicroK8s.

Direction la section 2.1 : Prérequis système ! 🚀

---

**Note** : Si à tout moment vous vous sentez perdu ou avez besoin de clarifications, n'hésitez pas à revenir à cette introduction. Elle est là pour vous guider et vous rappeler la vue d'ensemble.

⏭️ [Prérequis système (CPU, RAM, stockage)](/02-installation-et-configuration-initiale/01-prerequis-systeme.md)
