🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 2.3 Installation sur Linux (Ubuntu/Debian)

## Introduction

Félicitations ! Si vous êtes sur Linux (Ubuntu ou Debian), vous êtes dans l'environnement idéal pour MicroK8s. L'installation sera **simple, rapide et native**. En quelques minutes, vous aurez un cluster Kubernetes pleinement fonctionnel sur votre machine.

Cette section vous guidera pas à pas à travers l'installation complète de MicroK8s sur Ubuntu ou Debian. Chaque commande sera expliquée, et nous verrons ensemble comment vérifier que tout fonctionne correctement.

## Pourquoi Ubuntu/Debian est idéal

Avant de commencer, rappelons pourquoi ces distributions sont parfaites pour MicroK8s :

**Ubuntu :**
- Développé par Canonical (même entreprise que MicroK8s)
- Snap pré-installé
- Support officiel complet
- Documentation la plus fournie
- **C'est le choix numéro 1**

**Debian :**
- Base solide et stable
- Ubuntu est basé sur Debian
- Excellente compatibilité
- Support très bon

## Prérequis - Vérification rapide

Avant d'installer, vérifions rapidement que tout est en ordre.

### Vérifier votre distribution et version

```bash
# Afficher les informations système
cat /etc/os-release

# Vous devriez voir quelque chose comme :
# Ubuntu 22.04 LTS ou Ubuntu 24.04 LTS
# ou Debian 11 (Bullseye) ou Debian 12 (Bookworm)
```

**Versions recommandées :**
- Ubuntu : 20.04 LTS, 22.04 LTS, 23.10, 24.04 LTS
- Debian : 10 (Buster), 11 (Bullseye), 12 (Bookworm)

### Vérifier les ressources système

```bash
# CPU - devrait afficher 2 ou plus
nproc

# RAM - devrait afficher au moins 4G disponible
free -h

# Espace disque - devrait afficher au moins 20G libre
df -h /
```

### Vérifier les droits administrateur

```bash
# Tester sudo (vous devrez entrer votre mot de passe)
sudo echo "Sudo fonctionne !"

# Si ça affiche "Sudo fonctionne !", c'est bon !
```

**Important :** Vous aurez besoin des droits sudo (administrateur) pour l'installation.

## Étape 1 : Mettre à jour le système

Commençons par mettre à jour votre système. C'est une bonne pratique avant toute installation.

### Sur Ubuntu

```bash
# Mettre à jour la liste des paquets disponibles
sudo apt update

# Mettre à jour les paquets installés
sudo apt upgrade -y
```

**Ce qui se passe :**
- `apt update` : Télécharge la liste des dernières versions de paquets
- `apt upgrade -y` : Met à jour tous les paquets installés (le `-y` répond automatiquement "oui" aux confirmations)

**Durée :** 2-10 minutes selon votre connexion et le nombre de mises à jour

### Sur Debian

```bash
# Même chose sur Debian
sudo apt update
sudo apt upgrade -y
```

**Note :** Si c'est la première fois que vous utilisez sudo, le système vous demandera votre mot de passe.

## Étape 2 : Installer Snap (Debian uniquement)

**Si vous êtes sur Ubuntu :** Snap est déjà installé, passez directement à l'étape 3 ! ✅

**Si vous êtes sur Debian :** Snap doit être installé manuellement.

### Installation de snapd sur Debian

```bash
# Installer snapd
sudo apt install snapd -y

# Activer le service snapd
sudo systemctl enable --now snapd

# Activer le support des snaps classiques
sudo ln -s /var/lib/snapd/snap /snap
```

**Explication des commandes :**
- `apt install snapd` : Installe le gestionnaire de paquets Snap
- `systemctl enable --now snapd` : Active et démarre le service snapd
- `ln -s` : Crée un lien symbolique pour le support des snaps "classic"

### Vérifier que Snap fonctionne

```bash
# Vérifier la version de snap
snap version

# Vous devriez voir quelque chose comme :
# snap    2.60.4
# snapd   2.60.4
# series  16
# debian  12
```

**Important pour Debian :** Après l'installation de snapd, il est recommandé de **redémarrer votre machine** ou au moins de se déconnecter/reconnecter.

```bash
# Optionnel mais recommandé
sudo reboot
```

## Étape 3 : Installer MicroK8s

C'est l'étape principale ! Nous allons installer MicroK8s via Snap.

### Installation avec une commande

```bash
# Installation de MicroK8s
sudo snap install microk8s --classic
```

**Explication :**
- `snap install` : Commande d'installation Snap
- `microk8s` : Le paquet à installer
- `--classic` : Mode "classic" qui donne plus de permissions à MicroK8s (nécessaire pour accéder au système)

**Ce qui se passe :**
1. Snap télécharge MicroK8s (~200-300 Mo)
2. Snap installe tous les composants Kubernetes
3. MicroK8s configure automatiquement tout ce qui est nécessaire
4. Les services Kubernetes démarrent automatiquement

**Durée :** 2-5 minutes selon votre connexion internet

### Sortie attendue

```
microk8s (1.28/stable) v1.28.3 from Canonical✓ installed
```

Le numéro de version peut varier, c'est normal !

### Choisir une version spécifique (optionnel)

Par défaut, Snap installe la version stable recommandée. Si vous voulez une version spécifique :

```bash
# Installer une version Kubernetes spécifique
sudo snap install microk8s --classic --channel=1.28/stable

# Autres canaux disponibles :
# - 1.27/stable, 1.28/stable, 1.29/stable, 1.30/stable
# - latest/stable (dernière version)
# - latest/edge (version de développement)
```

**Recommandation pour débutants :** Gardez l'installation par défaut, c'est la version testée et stable.

## Étape 4 : Ajouter votre utilisateur au groupe MicroK8s

Pour utiliser MicroK8s sans `sudo` à chaque commande, ajoutez votre utilisateur au groupe `microk8s`.

### Ajouter l'utilisateur au groupe

```bash
# Ajouter votre utilisateur actuel au groupe microk8s
sudo usermod -a -G microk8s $USER

# Également au groupe snap_microk8s si présent
sudo usermod -a -G snap_microk8s $USER
```

**Explication :**
- `usermod` : Commande pour modifier un utilisateur
- `-a -G` : Ajouter aux groupes (sans retirer des autres)
- `microk8s` : Le groupe à rejoindre
- `$USER` : Variable qui contient votre nom d'utilisateur actuel

### Changer les permissions du dossier de configuration

```bash
# Créer le dossier .kube s'il n'existe pas
mkdir -p ~/.kube

# Donner les bonnes permissions
sudo chown -R $USER ~/.kube
```

### Appliquer les changements

Pour que les changements de groupe prennent effet, vous devez :

**Option 1 : Se reconnecter (recommandé)**
```bash
# Se déconnecter et reconnecter
# Ou simplement fermer et rouvrir le terminal
```

**Option 2 : Nouvelle session de groupe**
```bash
# Créer une nouvelle session avec les nouveaux groupes
su - $USER
```

**Option 3 : Redémarrer**
```bash
# Le plus sûr mais pas toujours nécessaire
sudo reboot
```

### Vérifier l'appartenance au groupe

```bash
# Voir les groupes de votre utilisateur
groups

# Vous devriez voir "microk8s" dans la liste
# Exemple : username adm cdrom sudo dip plugdev lxd microk8s
```

**Important :** Si vous ne voyez pas `microk8s` dans la liste, assurez-vous d'avoir bien exécuté `su - $USER` ou de vous être reconnecté.

## Étape 5 : Vérifier que MicroK8s fonctionne

Maintenant, vérifions que tout fonctionne correctement.

### Vérifier le statut de MicroK8s

```bash
# Voir le statut du cluster
microk8s status --wait-ready
```

**L'option `--wait-ready` :**
- Attend que MicroK8s soit complètement prêt
- Peut prendre 30-60 secondes au premier démarrage
- Affiche "microk8s is running" quand c'est prêt

**Sortie attendue :**
```
microk8s is running
high-availability: no
  datastore master nodes: 127.0.0.1:19001
  datastore standby nodes: none
addons:
  enabled:
    ha-cluster           # (core) Configure high availability on the current node
  disabled:
    ambassador           # (community) Ambassador API Gateway
    cilium               # (community) SDN, fast with full network policy
    ...
```

**Indicateurs importants :**
- ✅ "microk8s is running" → Tout fonctionne !
- ⚠️ "microk8s is not running" → Problème (voir section troubleshooting)

### Vérifier le nœud Kubernetes

```bash
# Voir les nœuds du cluster
microk8s kubectl get nodes
```

**Sortie attendue :**
```
NAME          STATUS   ROLES    AGE   VERSION
votre-machine Ready    <none>   30s   v1.28.3
```

**Indicateurs :**
- ✅ STATUS = Ready → Le nœud est prêt !
- ⚠️ STATUS = NotReady → Attendre quelques secondes, il démarre encore

**Ce que vous voyez :**
- **NAME** : Le nom de votre machine
- **STATUS** : Ready = le nœud fonctionne
- **ROLES** : Vide pour un single-node (normal)
- **AGE** : Âge du nœud depuis son démarrage
- **VERSION** : Version de Kubernetes

### Vérifier les services système

```bash
# Voir tous les pods système
microk8s kubectl get pods -A
```

**Sortie attendue :**
```
NAMESPACE     NAME                       READY   STATUS    RESTARTS   AGE
kube-system   calico-node-xxxxx          1/1     Running   0          2m
kube-system   calico-kube-controllers-xx 1/1     Running   0          2m
```

**Indicateurs :**
- ✅ STATUS = Running → Les services fonctionnent
- ✅ READY = 1/1 → Le pod est prêt
- ⚠️ STATUS = ContainerCreating → En cours de démarrage (normal les premières minutes)

**Note :** Les premières minutes, vous verrez peut-être des pods en "ContainerCreating". C'est normal, ils démarrent. Attendez 2-3 minutes et relancez la commande.

## Étape 6 : Configuration post-installation

Quelques configurations optionnelles mais recommandées pour améliorer votre expérience.

### Créer un alias pour kubectl

Au lieu de taper `microk8s kubectl` à chaque fois, créons un alias `kubectl` :

```bash
# Ajouter l'alias à votre .bashrc
echo "alias kubectl='microk8s kubectl'" >> ~/.bashrc

# Recharger le .bashrc
source ~/.bashrc
```

**Pour les utilisateurs de zsh :**
```bash
# Même chose pour zsh
echo "alias kubectl='microk8s kubectl'" >> ~/.zshrc
source ~/.zshrc
```

**Test de l'alias :**
```bash
# Maintenant vous pouvez utiliser directement kubectl
kubectl get nodes

# Au lieu de
microk8s kubectl get nodes
```

### Activer l'autocomplétion

L'autocomplétion vous permet de presser TAB pour compléter les commandes.

**Pour bash :**
```bash
# Ajouter l'autocomplétion
echo "source <(microk8s kubectl completion bash)" >> ~/.bashrc
echo "complete -F __start_kubectl kubectl" >> ~/.bashrc
source ~/.bashrc
```

**Pour zsh :**
```bash
# Ajouter l'autocomplétion zsh
echo "source <(microk8s kubectl completion zsh)" >> ~/.zshrc
echo "compdef __start_kubectl kubectl" >> ~/.zshrc
source ~/.zshrc
```

**Test :**
```bash
# Tapez et appuyez sur TAB
kubectl get po[TAB]
# Devrait compléter en : kubectl get pods
```

### Configurer la mémoire des services (optionnel)

Si vous avez peu de RAM (4 Go), vous pouvez réduire la consommation mémoire :

```bash
# Limiter la mémoire de containerd
sudo snap set microk8s containerd-memory-limit=1G

# Redémarrer MicroK8s pour appliquer
microk8s stop
microk8s start
```

**Note :** Cette configuration n'est généralement pas nécessaire avec 8 Go de RAM ou plus.

## Étape 7 : Démarrage automatique

MicroK8s démarre automatiquement au boot par défaut. Vérifions et ajustons si nécessaire.

### Vérifier le démarrage automatique

```bash
# Voir si MicroK8s démarre au boot
systemctl is-enabled snap.microk8s.daemon-kubelite.service
```

**Sortie :**
- `enabled` → Démarrage automatique activé ✅
- `disabled` → Démarrage automatique désactivé

### Activer le démarrage automatique

```bash
# Si désactivé, l'activer
sudo systemctl enable snap.microk8s.daemon-kubelite.service
```

### Désactiver le démarrage automatique (optionnel)

Si vous voulez économiser des ressources et ne démarrer MicroK8s que manuellement :

```bash
# Désactiver le démarrage automatique
sudo systemctl disable snap.microk8s.daemon-kubelite.service

# Vous devrez alors démarrer manuellement
microk8s start
```

## Premiers pas avec MicroK8s

Maintenant que tout est installé, faisons quelques tests rapides !

### Déployer votre première application

```bash
# Créer un déploiement nginx
kubectl create deployment nginx --image=nginx

# Voir le déploiement
kubectl get deployments

# Voir le pod créé
kubectl get pods
```

**Ce qui se passe :**
1. Kubernetes crée un Deployment nommé "nginx"
2. Le Deployment crée un Pod avec l'image nginx
3. L'image nginx est téléchargée depuis Docker Hub
4. Le conteneur nginx démarre dans le pod

**Attendre que le pod soit Running :**
```bash
# Surveiller le pod (rafraîchit toutes les 2 secondes)
kubectl get pods -w

# Sortie :
# NAME                     READY   STATUS    RESTARTS   AGE
# nginx-77b4fdf86c-xxxxx   1/1     Running   0          30s

# Pressez Ctrl+C pour arrêter la surveillance
```

### Exposer l'application

```bash
# Créer un service pour exposer nginx
kubectl expose deployment nginx --port=80 --type=NodePort

# Voir le service
kubectl get services nginx
```

**Sortie :**
```
NAME    TYPE       CLUSTER-IP      EXTERNAL-IP   PORT(S)        AGE
nginx   NodePort   10.152.183.xx   <none>        80:31234/TCP   5s
```

Le port NodePort (ex: 31234) est assigné automatiquement entre 30000-32767.

### Tester l'accès à l'application

```bash
# Récupérer le port NodePort
kubectl get service nginx -o jsonpath='{.spec.ports[0].nodePort}'

# Tester avec curl (remplacez XXXXX par votre port)
curl http://localhost:XXXXX

# Vous devriez voir le HTML de la page d'accueil nginx !
```

**Ou dans un navigateur :**
Ouvrez `http://localhost:XXXXX` (remplacez XXXXX par votre NodePort)

### Nettoyer le test

```bash
# Supprimer le service
kubectl delete service nginx

# Supprimer le déploiement
kubectl delete deployment nginx

# Vérifier que tout est supprimé
kubectl get all
```

## Commandes utiles MicroK8s

Voici un récapitulatif des commandes MicroK8s essentielles :

### Gestion du cluster

```bash
# Démarrer MicroK8s
microk8s start

# Arrêter MicroK8s
microk8s stop

# Redémarrer MicroK8s
microk8s stop && microk8s start

# Voir le statut
microk8s status

# Voir des infos détaillées
microk8s inspect
```

### Gestion des addons

```bash
# Lister les addons disponibles
microk8s status

# Activer un addon
microk8s enable dns

# Désactiver un addon
microk8s disable dns

# Activer plusieurs addons
microk8s enable dns dashboard storage
```

### Utilisation de kubectl

```bash
# Via microk8s
microk8s kubectl get pods

# Si vous avez créé l'alias
kubectl get pods

# Voir tous les objets
kubectl get all

# Voir dans tous les namespaces
kubectl get pods -A
```

### Informations et diagnostics

```bash
# Version de MicroK8s
microk8s version

# Logs d'un service
microk8s kubectl logs <nom-du-pod>

# Diagnostic complet (génère un rapport)
microk8s inspect

# Configuration kubectl
microk8s config
```

## Troubleshooting : Problèmes courants

### Problème : "microk8s: command not found"

**Cause :** Le chemin Snap n'est pas dans votre PATH.

**Solution :**
```bash
# Ajouter Snap au PATH
echo 'export PATH="$PATH:/snap/bin"' >> ~/.bashrc
source ~/.bashrc

# Vérifier
which microk8s
```

### Problème : "Permission denied" lors de l'utilisation de microk8s

**Cause :** Vous n'êtes pas dans le groupe microk8s ou les changements ne sont pas pris en compte.

**Solution :**
```bash
# Re-vérifier l'ajout au groupe
sudo usermod -a -G microk8s $USER

# Forcer le rechargement des groupes
su - $USER

# Vérifier
groups | grep microk8s
```

### Problème : Les pods restent en "ContainerCreating"

**Cause :** Images en cours de téléchargement ou problème réseau.

**Solution :**
```bash
# Voir les événements pour plus d'infos
kubectl describe pod <nom-du-pod>

# Vérifier les logs
microk8s kubectl logs <nom-du-pod>

# Attendre 2-3 minutes, surtout au premier lancement
# Les images doivent être téléchargées
```

### Problème : Le nœud reste "NotReady"

**Cause :** Services système pas encore démarrés.

**Solution :**
```bash
# Attendre avec --wait-ready
microk8s status --wait-ready

# Si ça ne fonctionne toujours pas après 5 minutes
microk8s stop
microk8s start
microk8s status --wait-ready

# Vérifier les logs
microk8s inspect
```

### Problème : "snap is not available"

**Cause :** Snapd n'est pas installé (Debian).

**Solution :**
```bash
# Installer snapd
sudo apt update
sudo apt install snapd -y

# Activer et démarrer
sudo systemctl enable --now snapd

# Créer le lien
sudo ln -s /var/lib/snapd/snap /snap

# Redémarrer
sudo reboot
```

### Problème : Erreur "cannot communicate with server"

**Cause :** MicroK8s n'est pas démarré ou les services ne sont pas prêts.

**Solution :**
```bash
# Vérifier si MicroK8s tourne
microk8s status

# Si pas démarré
microk8s start

# Attendre qu'il soit prêt
microk8s status --wait-ready
```

### Problème : Manque d'espace disque

**Cause :** Les images Docker consomment beaucoup d'espace.

**Solution :**
```bash
# Voir l'espace utilisé
df -h /

# Nettoyer les images inutilisées
microk8s ctr images ls | grep -v 'registry.k8s.io\|docker.io/library' | awk '{print $1}' | xargs -n1 microk8s ctr images rm

# Nettoyer les conteneurs arrêtés
microk8s ctr containers rm $(microk8s ctr containers ls -q)
```

## Désinstaller MicroK8s (si nécessaire)

Si vous devez désinstaller MicroK8s complètement :

### Désinstallation complète

```bash
# Arrêter MicroK8s
microk8s stop

# Désinstaller le snap
sudo snap remove microk8s

# Supprimer les données persistantes (optionnel)
sudo rm -rf /var/snap/microk8s

# Supprimer le groupe (optionnel)
sudo groupdel microk8s
```

**Attention :** Cette opération supprime TOUT, y compris vos applications déployées et vos données !

### Réinstallation propre

Après désinstallation, pour réinstaller :
```bash
# Réinstaller
sudo snap install microk8s --classic

# Reconfigurer
sudo usermod -a -G microk8s $USER
su - $USER
```

## Mise à jour de MicroK8s

Snap met à jour MicroK8s automatiquement, mais vous pouvez le faire manuellement :

### Mise à jour manuelle

```bash
# Voir les mises à jour disponibles
snap refresh --list

# Mettre à jour MicroK8s
sudo snap refresh microk8s

# Ou vers un canal spécifique
sudo snap refresh microk8s --channel=1.29/stable
```

### Contrôler les mises à jour automatiques

```bash
# Désactiver les mises à jour auto (non recommandé)
sudo snap refresh --hold microk8s

# Réactiver les mises à jour auto
sudo snap refresh --unhold microk8s
```

## Vérification finale : Checklist

Assurez-vous que tout fonctionne avec cette checklist :

**✅ Installation :**
- [ ] `microk8s version` affiche une version
- [ ] `microk8s status` affiche "microk8s is running"
- [ ] `groups` contient "microk8s"

**✅ Fonctionnement :**
- [ ] `kubectl get nodes` affiche votre nœud en "Ready"
- [ ] `kubectl get pods -A` affiche des pods en "Running"
- [ ] Déploiement nginx de test a fonctionné

**✅ Configuration :**
- [ ] Alias `kubectl` fonctionne (optionnel)
- [ ] Autocomplétion fonctionne (optionnel)
- [ ] Démarrage automatique configuré selon vos préférences

**Si toutes les cases sont cochées : Félicitations, MicroK8s est installé et fonctionnel ! 🎉**

## Prochaines étapes

Maintenant que MicroK8s est installé sur votre système Linux :

1. **Familiarisez-vous avec les commandes** essentielles
2. **Explorez le système d'addons** (section 5.x du tutoriel)
3. **Déployez votre première application** complète
4. **Activez des addons** : `microk8s enable dns dashboard storage`

Dans les sections suivantes, nous verrons :
- Comment activer et utiliser les addons
- Comment déployer des applications réelles
- Comment configurer le stockage, réseau, monitoring
- Et bien plus !

**Vous avez maintenant un cluster Kubernetes pleinement fonctionnel sur votre machine Linux. La vraie aventure commence ! 🚀**

---

**Points clés à retenir :**
- **Installation en une commande :** `sudo snap install microk8s --classic`
- **Ajouter au groupe :** `sudo usermod -a -G microk8s $USER` puis `su - $USER`
- **Vérifier le statut :** `microk8s status --wait-ready`
- **Premier nœud :** `kubectl get nodes` doit afficher "Ready"
- **Alias recommandé :** `alias kubectl='microk8s kubectl'`
- **Snap pré-installé sur Ubuntu**, doit être installé sur Debian
- **Démarrage automatique** activé par défaut
- **Troubleshooting :** `microk8s inspect` pour diagnostic complet
- **Mise à jour automatique** via Snap (ou manuelle avec `snap refresh`)
- MicroK8s fonctionne immédiatement après installation !

⏭️ [Installation sur CentOS/RHEL](/02-installation-et-configuration-initiale/04-installation-sur-centos-rhel.md)
