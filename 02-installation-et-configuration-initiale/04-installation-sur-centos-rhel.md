🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 2.4 Installation sur CentOS/RHEL

## Introduction

Si vous utilisez CentOS, Red Hat Enterprise Linux (RHEL), Rocky Linux ou AlmaLinux, cette section est pour vous ! Ces distributions sont très populaires en entreprise pour leur stabilité et leur support à long terme.

L'installation de MicroK8s sur ces systèmes nécessite quelques étapes supplémentaires par rapport à Ubuntu, principalement l'installation de Snap (qui n'est pas présent par défaut). Mais ne vous inquiétez pas, nous allons tout voir étape par étape !

## Comprendre l'écosystème RHEL

Avant de commencer, clarifions les différentes distributions de cette famille :

### Les distributions de la famille RHEL

**Red Hat Enterprise Linux (RHEL)**
- Distribution commerciale de Red Hat
- Support professionnel payant
- Très stable, orientée entreprise
- Cycle de vie : 10 ans de support

**CentOS**
- **CentOS 7** : Clone gratuit de RHEL 7 (fin de vie : 30 juin 2024)
- **CentOS 8** : Fin de vie prématurée (31 décembre 2021)
- **CentOS Stream** : Version rolling release, en amont de RHEL

**Rocky Linux**
- Créé après l'arrêt de CentOS 8
- Clone gratuit de RHEL (comme l'ancien CentOS)
- Fondé par le créateur original de CentOS
- **Recommandé** comme alternative à CentOS

**AlmaLinux**
- Autre clone gratuit de RHEL
- Sponsorisé par CloudLinux
- Très bonne alternative à CentOS
- Support communautaire actif

**Fedora**
- Distribution communautaire en amont de RHEL
- Versions plus récentes, moins stable
- Cycle de vie court (13 mois)
- Si vous êtes sur Fedora, voir section 2.3 (proche d'Ubuntu/Debian)

### Quelle distribution choisir ?

**Pour production/entreprise :**
1. **Rocky Linux 9** (premier choix)
2. **AlmaLinux 9**
3. **RHEL 9** (si vous avez une licence)

**Pour apprentissage/lab :**
1. **Rocky Linux 9**
2. **AlmaLinux 9**
3. **CentOS Stream 9** (si vous voulez tester les nouveautés)

**⚠️ Éviter :**
- CentOS 7 (obsolète, fin de vie dépassée)
- CentOS 8 (fin de vie en 2021)

## Prérequis - Vérification rapide

### Vérifier votre distribution et version

```bash
# Afficher les informations système
cat /etc/os-release

# Ou pour les anciennes versions
cat /etc/redhat-release
```

**Sorties possibles :**
```
# Rocky Linux 9
NAME="Rocky Linux"
VERSION="9.3 (Blue Onyx)"

# AlmaLinux 9
NAME="AlmaLinux"
VERSION="9.3 (Shamrock Pampas Cat)"

# CentOS Stream 9
NAME="CentOS Stream"
VERSION="9"

# RHEL 9
NAME="Red Hat Enterprise Linux"
VERSION="9.3 (Plow)"
```

**Versions recommandées pour MicroK8s :**
- Rocky Linux / AlmaLinux / RHEL : 8.x ou 9.x
- CentOS Stream : 8 ou 9

### Vérifier les ressources système

```bash
# CPU
nproc

# RAM
free -h

# Espace disque
df -h /

# Architecture (doit être x86_64)
uname -m
```

### Vérifier les droits administrateur

```bash
# Tester sudo
sudo echo "Sudo fonctionne !"
```

## Étape 1 : Mettre à jour le système

Commençons par mettre à jour votre système avec yum (CentOS/RHEL 7) ou dnf (CentOS/RHEL 8+).

### Sur CentOS/RHEL 7 (yum)

```bash
# Mettre à jour le cache
sudo yum check-update

# Mettre à jour tous les paquets
sudo yum update -y
```

### Sur CentOS/RHEL/Rocky/Alma 8+ (dnf)

```bash
# Mettre à jour le cache
sudo dnf check-update

# Mettre à jour tous les paquets
sudo dnf update -y
```

**Note :** dnf est la nouvelle génération de yum. Les commandes sont similaires.

**Durée :** 5-15 minutes selon votre connexion et le nombre de mises à jour.

**Important :** Si le noyau est mis à jour, redémarrez votre système :
```bash
sudo reboot
```

## Étape 2 : Installer EPEL (Extra Packages for Enterprise Linux)

EPEL est un dépôt qui contient des paquets supplémentaires non inclus dans les dépôts officiels RHEL/CentOS. Nous en avons besoin pour installer snapd.

### Sur CentOS/RHEL 7

```bash
# Installer EPEL
sudo yum install epel-release -y

# Vérifier
yum repolist | grep epel
```

### Sur CentOS/RHEL/Rocky/Alma 8+

```bash
# Installer EPEL
sudo dnf install epel-release -y

# Vérifier
dnf repolist | grep epel
```

**Sur RHEL (Red Hat Enterprise Linux) :**

RHEL nécessite une activation manuelle des dépôts :

```bash
# RHEL 9
sudo dnf install https://dl.fedoraproject.org/pub/epel/epel-release-latest-9.noarch.rpm -y

# RHEL 8
sudo dnf install https://dl.fedoraproject.org/pub/epel/epel-release-latest-8.noarch.rpm -y

# Activer le dépôt CodeReady Builder (RHEL 9)
sudo subscription-manager repos --enable codeready-builder-for-rhel-9-$(arch)-rpms

# Ou pour RHEL 8
sudo subscription-manager repos --enable codeready-builder-for-rhel-8-$(arch)-rpms
```

**Explication :**
- EPEL (Extra Packages for Enterprise Linux) est maintenu par la communauté Fedora
- Il fournit des milliers de paquets supplémentaires
- C'est nécessaire pour snapd

**Sortie attendue pour la vérification :**
```
epel          Extra Packages for Enterprise Linux 9 - x86_64
epel-modular  Extra Packages for Enterprise Linux Modular 9 - x86_64
```

## Étape 3 : Installer snapd

Maintenant que nous avons EPEL, installons snapd (le gestionnaire de paquets Snap).

### Installation de snapd

**Sur CentOS/RHEL 7 :**
```bash
sudo yum install snapd -y
```

**Sur CentOS/RHEL/Rocky/Alma 8+ :**
```bash
sudo dnf install snapd -y
```

### Activer et démarrer le service snapd

```bash
# Activer snapd au démarrage
sudo systemctl enable --now snapd.socket

# Démarrer snapd
sudo systemctl start snapd

# Vérifier le statut
sudo systemctl status snapd
```

**Sortie attendue :**
```
● snapd.service - Snap Daemon
   Loaded: loaded (/usr/lib/systemd/system/snapd.service; enabled)
   Active: active (running) since ...
```

Si vous voyez "active (running)", c'est bon ! ✅

### Créer le lien symbolique pour les snaps classiques

```bash
# Créer le lien /snap
sudo ln -s /var/lib/snapd/snap /snap
```

**Explication :** Ce lien permet aux snaps "classic" (comme MicroK8s) de fonctionner correctement.

### Vérifier que Snap fonctionne

```bash
# Vérifier la version de snap
snap version

# Si vous voyez une erreur "snap: command not found"
# Ajoutez snap au PATH :
echo 'export PATH=$PATH:/var/lib/snapd/snap/bin' >> ~/.bashrc
source ~/.bashrc
```

**Sortie attendue :**
```
snap    2.60.4
snapd   2.60.4
series  16
centos  9
```

**Important :** Après l'installation de snapd, il est fortement recommandé de **redémarrer** :

```bash
sudo reboot
```

Après le redémarrage, vérifiez que snapd fonctionne :
```bash
sudo systemctl status snapd
snap version
```

## Étape 4 : Configurer SELinux (Important !)

SELinux (Security-Enhanced Linux) est activé par défaut sur RHEL/CentOS et peut bloquer certaines fonctionnalités de MicroK8s.

### Vérifier l'état de SELinux

```bash
# Voir l'état de SELinux
getenforce
```

**Sorties possibles :**
- `Enforcing` : SELinux est actif et applique les règles
- `Permissive` : SELinux est actif mais ne bloque pas (log seulement)
- `Disabled` : SELinux est désactivé

### Option 1 : Mode Permissive (Recommandé pour apprentissage)

Le mode permissive permet à tout de fonctionner tout en journalisant les violations.

```bash
# Passer en mode permissive temporairement
sudo setenforce 0

# Vérifier
getenforce
# Devrait afficher : Permissive
```

**Pour rendre permanent (survit au redémarrage) :**
```bash
# Éditer la configuration SELinux
sudo sed -i 's/^SELINUX=enforcing/SELINUX=permissive/' /etc/selinux/config

# Vérifier
cat /etc/selinux/config | grep SELINUX=
```

### Option 2 : Désactiver SELinux complètement (Non recommandé en production)

```bash
# Désactiver temporairement
sudo setenforce 0

# Désactiver de façon permanente
sudo sed -i 's/^SELINUX=enforcing/SELINUX=disabled/' /etc/selinux/config

# Redémarrer pour appliquer
sudo reboot
```

**⚠️ Attention :** Désactiver SELinux réduit la sécurité de votre système. Pour un lab personnel, c'est acceptable, mais pas pour la production.

### Option 3 : Configurer les règles SELinux (Avancé)

Pour les utilisateurs avancés qui veulent garder SELinux en mode enforcing :

```bash
# Installer les outils de gestion SELinux
sudo dnf install policycoreutils-python-utils -y

# Les règles SELinux pour MicroK8s peuvent nécessiter une configuration spécifique
# Consultez la documentation officielle pour votre cas d'usage
```

**Pour débutants :** Restez en mode **permissive** pour cette formation.

## Étape 5 : Configurer le firewall

CentOS/RHEL utilise firewalld par défaut. Nous devons autoriser les ports nécessaires à Kubernetes.

### Vérifier l'état du firewall

```bash
# Voir si firewalld est actif
sudo systemctl status firewalld
```

### Option 1 : Autoriser les ports nécessaires (Recommandé)

```bash
# Autoriser le traffic API Server (port 16443)
sudo firewall-cmd --permanent --add-port=16443/tcp

# Autoriser les NodePorts (30000-32767)
sudo firewall-cmd --permanent --add-port=30000-32767/tcp

# Si vous voulez utiliser le dashboard
sudo firewall-cmd --permanent --add-port=10443/tcp

# Recharger la configuration
sudo firewall-cmd --reload

# Vérifier les ports ouverts
sudo firewall-cmd --list-ports
```

### Option 2 : Désactiver temporairement le firewall (Pour tests)

```bash
# Arrêter firewalld
sudo systemctl stop firewalld

# Désactiver au démarrage
sudo systemctl disable firewalld
```

**⚠️ Note :** Pour un lab personnel isolé, désactiver le firewall est acceptable. En production ou si votre machine est exposée sur le réseau, gardez le firewall actif et configurez les règles.

### Option 3 : Ajouter une zone de confiance pour Kubernetes

```bash
# Créer une zone trusted pour Kubernetes
sudo firewall-cmd --permanent --new-zone=kubernetes
sudo firewall-cmd --permanent --zone=kubernetes --set-target=ACCEPT
sudo firewall-cmd --permanent --zone=kubernetes --add-source=10.0.0.0/8
sudo firewall-cmd --permanent --zone=kubernetes --add-source=172.16.0.0/12
sudo firewall-cmd --reload
```

## Étape 6 : Installer MicroK8s

Nous sommes maintenant prêts pour l'installation de MicroK8s !

### Installation

```bash
# Installer MicroK8s
sudo snap install microk8s --classic
```

**Sortie attendue :**
```
microk8s (1.28/stable) v1.28.3 from Canonical✓ installed
```

**Durée :** 2-5 minutes selon votre connexion internet.

**Ce qui se passe :**
1. Snap télécharge MicroK8s (~300 Mo)
2. Tous les composants Kubernetes sont installés
3. Les services démarrent automatiquement

### Choisir une version spécifique (optionnel)

```bash
# Installer une version Kubernetes spécifique
sudo snap install microk8s --classic --channel=1.28/stable

# Voir les versions disponibles
snap info microk8s
```

## Étape 7 : Ajouter votre utilisateur au groupe MicroK8s

```bash
# Ajouter votre utilisateur au groupe microk8s
sudo usermod -a -G microk8s $USER

# Changer le propriétaire du dossier .kube
mkdir -p ~/.kube
sudo chown -R $USER ~/.kube
```

### Appliquer les changements de groupe

**Option 1 : Nouvelle session (recommandé)**
```bash
su - $USER
```

**Option 2 : Redémarrer**
```bash
sudo reboot
```

### Vérifier l'appartenance au groupe

```bash
# Voir vos groupes
groups

# Devrait contenir "microk8s"
```

## Étape 8 : Vérifier que MicroK8s fonctionne

### Vérifier le statut

```bash
# Attendre que MicroK8s soit prêt
microk8s status --wait-ready
```

**Sortie attendue :**
```
microk8s is running
high-availability: no
  datastore master nodes: 127.0.0.1:19001
  datastore standby nodes: none
```

Si vous voyez "microk8s is running", tout fonctionne ! ✅

### Vérifier le nœud

```bash
# Voir le nœud
microk8s kubectl get nodes
```

**Sortie attendue :**
```
NAME           STATUS   ROLES    AGE   VERSION
votre-machine  Ready    <none>   1m    v1.28.3
```

**STATUS = Ready** → Parfait ! ✅

### Vérifier les pods système

```bash
# Voir tous les pods
microk8s kubectl get pods -A
```

Les pods devraient être en état "Running" après 2-3 minutes.

## Étape 9 : Configuration post-installation

### Créer l'alias kubectl

```bash
# Pour bash
echo "alias kubectl='microk8s kubectl'" >> ~/.bashrc
source ~/.bashrc

# Pour zsh (si vous l'utilisez)
echo "alias kubectl='microk8s kubectl'" >> ~/.zshrc
source ~/.zshrc
```

### Activer l'autocomplétion

```bash
# Pour bash
echo "source <(microk8s kubectl completion bash)" >> ~/.bashrc
echo "complete -F __start_kubectl kubectl" >> ~/.bashrc
source ~/.bashrc
```

### Test rapide

```bash
# Avec l'alias
kubectl get nodes

# Devrait fonctionner sans "microk8s" devant
```

## Test de déploiement

Testons avec une application simple pour vérifier que tout fonctionne.

### Déployer nginx

```bash
# Créer un déploiement
kubectl create deployment nginx --image=nginx

# Attendre que le pod soit prêt
kubectl wait --for=condition=ready pod -l app=nginx --timeout=300s

# Voir le pod
kubectl get pods
```

### Exposer le service

```bash
# Créer un service NodePort
kubectl expose deployment nginx --port=80 --type=NodePort

# Voir le service
kubectl get service nginx
```

### Tester l'accès

```bash
# Récupérer le port NodePort
PORT=$(kubectl get service nginx -o jsonpath='{.spec.ports[0].nodePort}')

# Tester avec curl
curl http://localhost:$PORT

# Vous devriez voir le HTML de nginx !
```

### Nettoyer

```bash
kubectl delete service nginx
kubectl delete deployment nginx
```

Si tout a fonctionné, votre installation est complète ! 🎉

## Troubleshooting : Problèmes spécifiques à CentOS/RHEL

### Problème : "snap: command not found"

**Cause :** Snap n'est pas dans le PATH.

**Solution :**
```bash
# Ajouter snap au PATH
echo 'export PATH=$PATH:/var/lib/snapd/snap/bin' >> ~/.bashrc
source ~/.bashrc

# Vérifier
which snap
```

### Problème : SELinux bloque MicroK8s

**Symptômes :**
- Pods qui ne démarrent pas
- Erreurs de permission dans les logs
- Messages "Permission denied" dans `microk8s inspect`

**Solution :**
```bash
# Passer en mode permissive
sudo setenforce 0

# Rendre permanent
sudo sed -i 's/^SELINUX=enforcing/SELINUX=permissive/' /etc/selinux/config

# Redémarrer MicroK8s
microk8s stop
microk8s start
```

### Problème : Le firewall bloque les connexions

**Symptômes :**
- Impossible d'accéder aux services NodePort
- Timeout lors de l'accès aux applications

**Solution :**
```bash
# Autoriser les ports Kubernetes
sudo firewall-cmd --permanent --add-port=16443/tcp
sudo firewall-cmd --permanent --add-port=30000-32767/tcp
sudo firewall-cmd --reload

# Ou temporairement désactiver pour tester
sudo systemctl stop firewalld
```

### Problème : "cannot install snap: classic confinement requires snaps under /snap"

**Cause :** Le lien symbolique /snap n'existe pas.

**Solution :**
```bash
# Créer le lien
sudo ln -s /var/lib/snapd/snap /snap

# Réinstaller MicroK8s
sudo snap install microk8s --classic
```

### Problème : snapd ne démarre pas après redémarrage

**Solution :**
```bash
# Réactiver et redémarrer snapd
sudo systemctl enable snapd.socket
sudo systemctl enable snapd
sudo systemctl start snapd

# Vérifier
sudo systemctl status snapd
```

### Problème : EPEL n'est pas disponible

**Sur RHEL :**
```bash
# Vérifier que subscription-manager est configuré
sudo subscription-manager repos --list

# Activer les dépôts nécessaires
sudo subscription-manager repos --enable codeready-builder-for-rhel-9-$(arch)-rpms

# Installer EPEL
sudo dnf install epel-release -y
```

### Problème : Les pods restent en "Pending"

**Cause possible :** Problèmes SELinux ou de réseau.

**Solution :**
```bash
# Vérifier les événements
kubectl describe pod <nom-du-pod>

# Vérifier SELinux
getenforce

# Si "Enforcing", passer en "Permissive"
sudo setenforce 0

# Redémarrer MicroK8s
microk8s stop
microk8s start
```

### Problème : "Error: snap "microk8s" has running apps"

**Cause :** MicroK8s est en cours d'exécution.

**Solution :**
```bash
# Arrêter MicroK8s d'abord
microk8s stop

# Puis effectuer l'opération (ex: désinstallation)
sudo snap remove microk8s
```

## Commandes utiles spécifiques à RHEL/CentOS

### Gestion des services avec systemctl

```bash
# Voir tous les services snap
systemctl list-units | grep snap

# Redémarrer snapd
sudo systemctl restart snapd

# Voir les logs de snapd
sudo journalctl -u snapd -f
```

### Vérifier les logs système

```bash
# Logs généraux
sudo journalctl -xe

# Logs SELinux
sudo ausearch -m avc -ts recent

# Logs firewall
sudo journalctl -u firewalld
```

### Gestion des paquets

```bash
# Lister les paquets installés (dnf)
dnf list installed | grep -i snap

# Nettoyer le cache
sudo dnf clean all

# Mettre à jour tout le système
sudo dnf update -y
```

## Différences avec Ubuntu/Debian

Pour votre information, voici les principales différences :

| Aspect | Ubuntu/Debian | CentOS/RHEL |
|--------|---------------|-------------|
| **Gestionnaire de paquets** | apt | yum/dnf |
| **Snap pré-installé** | Oui (Ubuntu) | Non |
| **EPEL requis** | Non | Oui |
| **SELinux** | Désactivé par défaut | Activé |
| **Firewall** | ufw | firewalld |
| **Configuration supplémentaire** | Minimale | Plus importante |
| **Complexité installation** | Simple | Moyenne |

## Optimisations pour CentOS/RHEL

### Désactiver les services inutiles

```bash
# Voir les services actifs
systemctl list-units --type=service --state=running

# Désactiver des services non nécessaires (exemples)
sudo systemctl disable postfix
sudo systemctl stop postfix
```

### Optimiser le swappiness

```bash
# Réduire l'utilisation du swap (mieux pour Kubernetes)
echo "vm.swappiness=10" | sudo tee -a /etc/sysctl.conf
sudo sysctl -p
```

### Activer les mises à jour automatiques de sécurité (optionnel)

```bash
# Installer yum-cron (CentOS/RHEL 7)
sudo yum install yum-cron -y
sudo systemctl enable yum-cron
sudo systemctl start yum-cron

# Ou dnf-automatic (RHEL 8+)
sudo dnf install dnf-automatic -y
sudo systemctl enable dnf-automatic.timer
sudo systemctl start dnf-automatic.timer
```

## Désinstallation complète

Si nécessaire, voici comment désinstaller complètement MicroK8s :

```bash
# Arrêter MicroK8s
microk8s stop

# Désinstaller
sudo snap remove microk8s

# Supprimer les données
sudo rm -rf /var/snap/microk8s

# Désinstaller snapd (optionnel)
sudo dnf remove snapd -y  # ou yum sur CentOS 7

# Supprimer le lien
sudo rm /snap
```

## Checklist de vérification finale

**✅ Prérequis :**
- [ ] Système à jour (`dnf update` ou `yum update`)
- [ ] EPEL installé (`dnf repolist | grep epel`)
- [ ] snapd installé et actif (`systemctl status snapd`)
- [ ] SELinux en permissive ou disabled (`getenforce`)
- [ ] Firewall configuré (ports autorisés ou désactivé)

**✅ Installation :**
- [ ] MicroK8s installé (`microk8s version`)
- [ ] Utilisateur dans le groupe microk8s (`groups | grep microk8s`)
- [ ] Status running (`microk8s status`)

**✅ Fonctionnement :**
- [ ] Node Ready (`kubectl get nodes`)
- [ ] Pods système Running (`kubectl get pods -A`)
- [ ] Déploiement test nginx réussi

**Si toutes les cases sont cochées : Installation réussie ! 🎉**

## Prochaines étapes

Votre installation MicroK8s sur CentOS/RHEL est maintenant complète !

**Recommandations :**
1. Familiarisez-vous avec les commandes de base
2. Activez les addons essentiels : `microk8s enable dns dashboard storage`
3. Explorez le Dashboard Kubernetes
4. Suivez les sections suivantes du tutoriel

**Particularités à retenir :**
- SELinux en mode permissive pour éviter les blocages
- Firewalld configuré pour les ports Kubernetes
- EPEL nécessaire pour snapd
- Redémarrage recommandé après installation de snapd

**Vous avez maintenant un cluster Kubernetes fonctionnel sur votre distribution Red Hat ! 🚀**

---

**Points clés à retenir :**
- **EPEL requis** pour installer snapd : `dnf install epel-release`
- **snapd doit être installé** : `dnf install snapd`
- **SELinux en permissive** recommandé : `sudo setenforce 0`
- **Firewall à configurer** : Autoriser ports 16443 et 30000-32767
- **Redémarrage recommandé** après installation de snapd
- **Lien /snap nécessaire** : `sudo ln -s /var/lib/snapd/snap /snap`
- Rocky Linux et AlmaLinux sont les meilleures alternatives à CentOS
- Installation plus complexe qu'Ubuntu mais fonctionne parfaitement
- Même fonctionnalités MicroK8s une fois installé
- Documentation RHEL officielle disponible pour configurations avancées

⏭️ [Installation sur Windows (WSL2)](/02-installation-et-configuration-initiale/05-installation-sur-windows-wsl2.md)
