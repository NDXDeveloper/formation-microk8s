🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 2.6 Installation sur macOS

## Introduction

L'installation de MicroK8s sur macOS présente quelques particularités par rapport aux systèmes Linux. En effet, Kubernetes nécessite un noyau Linux pour fonctionner. Sur macOS, MicroK8s utilise donc **Multipass**, une solution de virtualisation légère développée par Canonical, qui crée une machine virtuelle Linux optimisée pour faire tourner MicroK8s.

Cette approche permet de bénéficier de l'expérience MicroK8s sur macOS tout en conservant une excellente intégration avec le système hôte.

## Prérequis Système

Avant de commencer l'installation, assurez-vous que votre système répond aux exigences suivantes :

### Configuration minimale recommandée

- **Système d'exploitation** : macOS 10.14 (Mojave) ou plus récent
- **Processeur** : Intel ou Apple Silicon (M1/M2/M3)
- **RAM** : 4 Go minimum (8 Go recommandés pour un usage confortable)
- **Espace disque** : 20 Go d'espace libre minimum
- **Droits administrateur** : Nécessaires pour l'installation

### Logiciels requis

- **Homebrew** : Le gestionnaire de paquets pour macOS (nous l'utiliserons pour l'installation)

Si vous n'avez pas encore Homebrew installé, vous pouvez l'installer avec la commande suivante dans le Terminal :

```bash
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"
```

## Méthode d'Installation

Sur macOS, il existe deux approches principales pour installer MicroK8s :

1. **Installation via Homebrew** (recommandée - la plus simple)
2. **Installation manuelle de Multipass puis MicroK8s**

Nous allons détailler la méthode recommandée via Homebrew, qui automatise l'ensemble du processus.

## Installation via Homebrew

### Étape 1 : Installation de Multipass

Multipass est l'outil qui créera et gérera la machine virtuelle Linux nécessaire à MicroK8s.

Ouvrez le Terminal et exécutez :

```bash
brew install multipass
```

Cette commande va télécharger et installer Multipass ainsi que ses dépendances. L'installation peut prendre quelques minutes selon votre connexion internet.

### Étape 2 : Vérification de Multipass

Une fois l'installation terminée, vérifiez que Multipass fonctionne correctement :

```bash
multipass version
```

Vous devriez voir s'afficher la version de Multipass installée, par exemple :

```
multipass   1.13.0
multipassd  1.13.0
```

### Étape 3 : Installation de MicroK8s

Maintenant que Multipass est installé, vous pouvez installer MicroK8s :

```bash
brew install ubuntu/microk8s/microk8s
```

Cette commande va :
- Télécharger l'image Ubuntu optimisée pour MicroK8s
- Créer une machine virtuelle via Multipass
- Installer et configurer MicroK8s dans cette VM
- Configurer les commandes pour qu'elles soient accessibles depuis votre macOS

L'installation peut prendre 5 à 10 minutes. Soyez patient, le processus télécharge plusieurs gigaoctets de données.

### Étape 4 : Démarrage de MicroK8s

Une fois l'installation terminée, démarrez MicroK8s :

```bash
microk8s start
```

Cette commande lance la machine virtuelle et démarre tous les services Kubernetes nécessaires.

### Étape 5 : Vérification de l'installation

Vérifiez que votre cluster MicroK8s fonctionne correctement :

```bash
microk8s status --wait-ready
```

L'option `--wait-ready` attend que tous les services soient prêts avant de retourner le statut. Vous devriez voir un message indiquant que MicroK8s est en cours d'exécution, avec une liste des addons disponibles.

Pour vérifier l'état des nœuds du cluster :

```bash
microk8s kubectl get nodes
```

Vous devriez voir un nœud en état "Ready" :

```
NAME               STATUS   ROLES    AGE   VERSION
microk8s-vm        Ready    <none>   2m    v1.28.3
```

## Configuration Post-Installation

### Alias kubectl

Pour simplifier l'utilisation quotidienne, il est recommandé de créer un alias pour la commande `kubectl`. Ajoutez cette ligne à votre fichier de configuration shell (`~/.zshrc` pour zsh ou `~/.bash_profile` pour bash) :

```bash
alias kubectl='microk8s kubectl'
```

Puis rechargez votre configuration :

```bash
source ~/.zshrc  # pour zsh
# ou
source ~/.bash_profile  # pour bash
```

Vous pourrez maintenant utiliser simplement `kubectl` au lieu de `microk8s kubectl`.

### Configuration de la mémoire et du CPU

Par défaut, Multipass alloue des ressources limitées à la VM. Pour ajuster ces paramètres selon vos besoins, vous pouvez arrêter MicroK8s et reconfigurer la VM :

```bash
microk8s stop
```

Pour modifier les ressources allouées, vous devrez utiliser directement les commandes Multipass. Par exemple, pour allouer 4 CPU et 8 Go de RAM :

```bash
multipass stop microk8s-vm
multipass set local.microk8s-vm.cpus=4
multipass set local.microk8s-vm.memory=8G
multipass start microk8s-vm
```

Puis redémarrez MicroK8s :

```bash
microk8s start
```

## Comprendre l'Architecture sur macOS

Il est important de comprendre comment fonctionne MicroK8s sur macOS :

1. **macOS (Système hôte)** : Votre système d'exploitation principal
2. **Multipass** : Gestionnaire de machines virtuelles légères
3. **Machine virtuelle Ubuntu** : VM créée automatiquement par Multipass
4. **MicroK8s** : Cluster Kubernetes s'exécutant dans la VM Ubuntu

Cette architecture en couches signifie que :

- Les commandes `microk8s` depuis votre Terminal macOS communiquent avec la VM
- Les applications déployées dans Kubernetes s'exécutent réellement dans la VM Linux
- Le réseau est géré par Multipass pour permettre l'accès depuis macOS
- Les volumes persistent dans la VM, pas directement sur macOS

## Accès au Shell de la VM

Si vous avez besoin d'accéder directement au shell de la machine virtuelle Ubuntu, vous pouvez utiliser :

```bash
multipass shell microk8s-vm
```

Cette commande ouvre une session shell dans la VM. C'est utile pour le dépannage avancé ou pour accéder directement aux logs système. Pour quitter le shell, tapez `exit`.

## Gestion de l'État de MicroK8s

### Démarrer MicroK8s

```bash
microk8s start
```

### Arrêter MicroK8s

```bash
microk8s stop
```

Cela arrête les services Kubernetes mais maintient la VM en pause.

### Redémarrer MicroK8s

```bash
microk8s stop
microk8s start
```

### Arrêter complètement la VM

```bash
multipass stop microk8s-vm
```

### Démarrer la VM

```bash
multipass start microk8s-vm
```

## Désinstallation

Si vous souhaitez désinstaller MicroK8s de votre système macOS :

### Supprimer MicroK8s et la VM

```bash
microk8s uninstall
```

### Désinstaller le package Homebrew

```bash
brew uninstall microk8s
```

### Supprimer Multipass (optionnel)

Si vous ne prévoyez plus d'utiliser Multipass :

```bash
brew uninstall multipass
```

## Problèmes Courants et Solutions

### La commande microk8s n'est pas reconnue

**Problème** : Après l'installation, le Terminal ne reconnaît pas la commande `microk8s`.

**Solution** :
- Fermez et rouvrez votre Terminal
- Vérifiez que Homebrew est correctement installé : `brew --version`
- Réinstallez avec `brew reinstall ubuntu/microk8s/microk8s`

### La VM ne démarre pas

**Problème** : La machine virtuelle Multipass refuse de démarrer.

**Solution** :
- Vérifiez que la virtualisation est activée dans les paramètres de votre Mac
- Pour les Mac Apple Silicon (M1/M2/M3), assurez-vous d'avoir la dernière version de Multipass
- Essayez de redémarrer Multipass : `sudo killall multipassd` puis `multipass start microk8s-vm`

### Performance insuffisante

**Problème** : MicroK8s est lent ou les déploiements échouent par manque de ressources.

**Solution** :
- Augmentez les ressources allouées à la VM (voir section Configuration de la mémoire et du CPU)
- Fermez d'autres applications gourmandes en ressources
- Vérifiez l'espace disque disponible sur votre Mac

### Problèmes de réseau

**Problème** : Impossible d'accéder aux services déployés dans le cluster.

**Solution** :
- Vérifiez que la VM est bien démarrée : `multipass list`
- Testez la connectivité : `multipass exec microk8s-vm -- ping 8.8.8.8`
- Redémarrez Multipass si nécessaire

## Particularités macOS vs Linux

Il est important de noter quelques différences par rapport à une installation Linux native :

### Avantages

- **Installation simplifiée** : Pas besoin de configurer manuellement la virtualisation
- **Isolation** : La VM isole complètement MicroK8s de votre système macOS
- **Pas de conflit** : Aucun risque de conflit avec d'autres services système

### Inconvénients

- **Overhead de la VM** : Légère perte de performance due à la virtualisation
- **Consommation de ressources** : La VM consomme de la RAM même au repos
- **Accès aux volumes** : Les volumes ne sont pas directement accessibles depuis macOS
- **Réseau** : Une couche supplémentaire de NAT peut compliquer certaines configurations

### Accès aux fichiers de la VM

Si vous devez transférer des fichiers entre macOS et la VM MicroK8s :

**Depuis macOS vers la VM** :

```bash
multipass transfer mon-fichier.yaml microk8s-vm:/home/ubuntu/
```

**Depuis la VM vers macOS** :

```bash
multipass transfer microk8s-vm:/home/ubuntu/mon-fichier.yaml ./
```

## Démarrage Automatique

Par défaut, MicroK8s ne démarre pas automatiquement au démarrage de macOS. Pour activer le démarrage automatique de Multipass :

```bash
brew services start multipass
```

Cela garantit que Multipass démarre au boot de votre Mac, mais vous devrez toujours exécuter `microk8s start` manuellement après le démarrage.

## Vérification Finale

Pour vous assurer que tout fonctionne correctement, exécutez ces commandes de test :

```bash
# Vérifier l'état de MicroK8s
microk8s status

# Vérifier les nœuds
microk8s kubectl get nodes

# Vérifier les services système
microk8s kubectl get pods -n kube-system

# Vérifier la version
microk8s version
```

Si toutes ces commandes s'exécutent sans erreur et retournent des résultats cohérents, votre installation MicroK8s sur macOS est prête à l'emploi !

## Prochaines Étapes

Maintenant que MicroK8s est installé sur votre macOS, vous pouvez :

- Passer à la section **2.7 Vérification de l'installation** pour des tests plus approfondis
- Explorer les **commandes MicroK8s essentielles** dans la section 2.8
- Commencer à déployer vos premières applications dans la **Partie 1 : Fondamentaux**

Félicitations, vous avez maintenant un cluster Kubernetes pleinement fonctionnel sur votre Mac ! 🎉

⏭️ [Vérification de l'installation](/02-installation-et-configuration-initiale/07-verification-de-linstallation.md)
