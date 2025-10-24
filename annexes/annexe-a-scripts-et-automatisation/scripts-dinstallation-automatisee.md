🔝 Retour au [Sommaire](/SOMMAIRE.md)

# Annexe A : Scripts d'Installation Automatisée MicroK8s

## Introduction

L'installation automatisée de MicroK8s permet de gagner un temps précieux et d'assurer une configuration cohérente à chaque déploiement. Cette annexe présente des scripts prêts à l'emploi pour différents systèmes d'exploitation, accompagnés d'explications détaillées pour les débutants.

**Pourquoi automatiser l'installation ?**

- **Gain de temps** : une seule commande au lieu de multiples étapes manuelles
- **Reproductibilité** : configuration identique à chaque installation
- **Réduction d'erreurs** : moins de manipulations manuelles = moins de risques
- **Documentation** : le script sert de référence pour votre configuration
- **Partage** : facilite l'installation pour d'autres membres de l'équipe

---

## 1. Script d'Installation pour Ubuntu/Debian

### 1.1 Script de Base

Ce script installe MicroK8s avec une configuration standard sur Ubuntu ou Debian.

```bash
#!/bin/bash
# Script d'installation MicroK8s pour Ubuntu/Debian
# Auteur : Votre nom
# Date : 2025-10-24

set -e  # Arrête le script en cas d'erreur

echo "=========================================="
echo "Installation de MicroK8s"
echo "=========================================="
echo ""

# Vérification des privilèges
if [ "$EUID" -eq 0 ]; then
    echo "Erreur : Ne pas exécuter ce script en tant que root"
    echo "Utilisez votre utilisateur normal, sudo sera appelé quand nécessaire"
    exit 1
fi

# Mise à jour du système
echo "[1/6] Mise à jour du système..."
sudo apt update
sudo apt upgrade -y

# Installation de snapd (si pas déjà installé)
echo "[2/6] Installation de snapd..."
sudo apt install snapd -y

# Installation de MicroK8s
echo "[3/6] Installation de MicroK8s..."
sudo snap install microk8s --classic --channel=1.31/stable

# Ajout de l'utilisateur au groupe microk8s
echo "[4/6] Configuration des permissions utilisateur..."
sudo usermod -a -G microk8s $USER
sudo chown -R $USER ~/.kube

# Configuration de l'alias kubectl
echo "[5/6] Configuration de l'alias kubectl..."
echo "alias kubectl='microk8s kubectl'" >> ~/.bashrc
echo "alias k='microk8s kubectl'" >> ~/.bashrc

# Attente du démarrage de MicroK8s
echo "[6/6] Attente du démarrage de MicroK8s..."
sudo microk8s status --wait-ready

echo ""
echo "=========================================="
echo "Installation terminée avec succès !"
echo "=========================================="
echo ""
echo "IMPORTANT : Pour appliquer les changements de groupe,"
echo "vous devez vous déconnecter et vous reconnecter, ou exécuter :"
echo "  newgrp microk8s"
echo ""
echo "Commandes utiles :"
echo "  microk8s status           # Vérifier le statut"
echo "  microk8s kubectl get nodes # Voir les nœuds"
echo "  microk8s enable dns       # Activer le DNS"
echo ""
```

### 1.2 Explication du Script Ligne par Ligne

**En-tête du script**

```bash
#!/bin/bash
```
Le "shebang" indique que c'est un script bash. Il doit toujours être sur la première ligne.

```bash
set -e
```
Cette commande arrête le script immédiatement si une commande échoue. C'est une sécurité importante.

**Vérification des privilèges**

```bash
if [ "$EUID" -eq 0 ]; then
```
Vérifie si le script est exécuté en tant que root. MicroK8s ne doit PAS être installé en tant que root pour des raisons de sécurité.

**Installation progressive**

Le script est divisé en 6 étapes clairement identifiées pour que l'utilisateur puisse suivre la progression.

**Configuration des permissions**

```bash
sudo usermod -a -G microk8s $USER
```
Ajoute l'utilisateur actuel au groupe `microk8s`, nécessaire pour utiliser MicroK8s sans sudo.

### 1.3 Script d'Installation Complète avec Addons

Ce script va plus loin et active automatiquement les addons essentiels.

```bash
#!/bin/bash
# Script d'installation MicroK8s COMPLÈTE avec addons
# Configuration pour un lab de développement

set -e

echo "=========================================="
echo "Installation Complète de MicroK8s"
echo "=========================================="
echo ""

# Fonction pour afficher les messages
log_info() {
    echo "[INFO] $1"
}

log_success() {
    echo "[✓] $1"
}

log_error() {
    echo "[✗] $1"
}

# Vérification des privilèges
if [ "$EUID" -eq 0 ]; then
    log_error "Ne pas exécuter ce script en tant que root"
    exit 1
fi

# Variables de configuration
CHANNEL="1.31/stable"
ADDONS=(dns dashboard storage registry ingress metallb:10.64.140.43-10.64.140.49)

# Mise à jour du système
log_info "Mise à jour du système..."
sudo apt update && sudo apt upgrade -y
log_success "Système mis à jour"

# Installation de snapd
log_info "Installation de snapd..."
sudo apt install snapd -y
log_success "Snapd installé"

# Installation de MicroK8s
log_info "Installation de MicroK8s (canal: $CHANNEL)..."
sudo snap install microk8s --classic --channel=$CHANNEL
log_success "MicroK8s installé"

# Configuration des permissions
log_info "Configuration des permissions..."
sudo usermod -a -G microk8s $USER
sudo chown -R $USER ~/.kube
log_success "Permissions configurées"

# Attente du démarrage
log_info "Attente du démarrage de MicroK8s..."
sudo microk8s status --wait-ready
log_success "MicroK8s démarré"

# Activation des addons
log_info "Activation des addons essentiels..."
for addon in "${ADDONS[@]}"; do
    log_info "Activation de l'addon: $addon"
    sudo microk8s enable $addon
    log_success "Addon $addon activé"
done

# Configuration de kubectl
log_info "Configuration des alias kubectl..."
cat >> ~/.bashrc << 'EOF'

# Aliases MicroK8s
alias kubectl='microk8s kubectl'
alias k='microk8s kubectl'
alias mk='microk8s'
EOF

# Génération du fichier kubeconfig
log_info "Génération du kubeconfig..."
mkdir -p ~/.kube
sudo microk8s config > ~/.kube/config
chmod 600 ~/.kube/config
log_success "Kubeconfig généré"

echo ""
echo "=========================================="
echo "Installation COMPLÈTE terminée !"
echo "=========================================="
echo ""
echo "Addons activés :"
for addon in "${ADDONS[@]}"; do
    echo "  - $addon"
done
echo ""
echo "Pour activer les alias, exécutez :"
echo "  source ~/.bashrc"
echo "  newgrp microk8s"
echo ""
echo "Pour vérifier l'installation :"
echo "  microk8s status"
echo "  kubectl get all --all-namespaces"
echo ""
```

### 1.4 Personnalisation du Script

Vous pouvez modifier les variables de configuration selon vos besoins :

```bash
# Changer la version de MicroK8s
CHANNEL="1.30/stable"  # ou "latest/edge" pour la dernière version

# Modifier la liste des addons
ADDONS=(
    dns
    dashboard
    storage
    registry
    ingress
    cert-manager
    prometheus
    # metallb:10.64.140.43-10.64.140.49  # Commentez si pas besoin
)

# Ajouter des variables d'environnement
KUBE_EDITOR="nano"  # ou "vim", "code", etc.
```

---

## 2. Script d'Installation pour CentOS/RHEL

### 2.1 Script de Base CentOS/RHEL

```bash
#!/bin/bash
# Script d'installation MicroK8s pour CentOS/RHEL
# Compatible CentOS 7/8/9 et RHEL 7/8/9

set -e

echo "=========================================="
echo "Installation de MicroK8s sur CentOS/RHEL"
echo "=========================================="
echo ""

# Détection de la version
if [ -f /etc/os-release ]; then
    . /etc/os-release
    OS_VERSION=$VERSION_ID
    echo "Système détecté : $NAME $VERSION"
fi

# Vérification des privilèges
if [ "$EUID" -eq 0 ]; then
    echo "[✗] Ne pas exécuter ce script en tant que root"
    exit 1
fi

# Installation des dépendances
echo "[1/7] Installation des dépendances..."
sudo yum install -y epel-release
sudo yum update -y
sudo yum install -y snapd

# Activation de snapd
echo "[2/7] Activation du service snapd..."
sudo systemctl enable --now snapd.socket
sudo systemctl start snapd

# Création du lien symbolique classique
echo "[3/7] Configuration de snap..."
sudo ln -sf /var/lib/snapd/snap /snap

# Attente de la disponibilité de snapd
echo "[4/7] Attente de l'initialisation de snapd..."
sleep 10

# Installation de MicroK8s
echo "[5/7] Installation de MicroK8s..."
sudo snap install microk8s --classic --channel=1.31/stable

# Configuration des permissions
echo "[6/7] Configuration des permissions..."
sudo usermod -a -G microk8s $USER
sudo chown -R $USER ~/.kube

# Configuration du firewall (si firewalld est actif)
if systemctl is-active --quiet firewalld; then
    echo "[7/7] Configuration du firewall..."
    sudo firewall-cmd --permanent --add-port=16443/tcp  # API server
    sudo firewall-cmd --permanent --add-port=10250/tcp  # Kubelet
    sudo firewall-cmd --permanent --add-port=10255/tcp  # Read-only kubelet
    sudo firewall-cmd --permanent --add-port=10251/tcp  # Scheduler
    sudo firewall-cmd --permanent --add-port=10252/tcp  # Controller
    sudo firewall-cmd --permanent --add-port=2379-2380/tcp  # etcd
    sudo firewall-cmd --reload
    echo "[✓] Règles firewall configurées"
else
    echo "[7/7] Firewall non actif, étape ignorée"
fi

# Attente du démarrage
echo "Attente du démarrage de MicroK8s..."
sudo microk8s status --wait-ready

# Configuration SELinux (si actif)
if command -v getenforce &> /dev/null; then
    if [ "$(getenforce)" != "Disabled" ]; then
        echo ""
        echo "ATTENTION : SELinux est actif"
        echo "Pour une meilleure compatibilité, vous pouvez le mettre en mode permissif :"
        echo "  sudo setenforce 0"
        echo "  sudo sed -i 's/^SELINUX=enforcing/SELINUX=permissive/' /etc/selinux/config"
    fi
fi

echo ""
echo "=========================================="
echo "Installation terminée avec succès !"
echo "=========================================="
echo ""
echo "IMPORTANT : Déconnectez-vous et reconnectez-vous, ou exécutez :"
echo "  newgrp microk8s"
echo ""
```

### 2.2 Spécificités CentOS/RHEL

**Gestion du firewall**

Sur CentOS/RHEL, `firewalld` est souvent actif par défaut. Le script ouvre automatiquement les ports nécessaires.

**SELinux**

SELinux peut causer des problèmes avec certaines fonctionnalités. Le script le détecte et vous informe.

**Snapd**

Sur CentOS/RHEL, snapd nécessite une configuration supplémentaire (lien symbolique `/snap`).

---

## 3. Script d'Installation pour Windows (WSL2)

### 3.1 Prérequis Windows

Avant d'exécuter le script, assurez-vous que :

1. WSL2 est installé et activé
2. Une distribution Ubuntu est installée dans WSL2
3. Vous utilisez un terminal WSL (Ubuntu)

### 3.2 Script pour WSL2

```bash
#!/bin/bash
# Script d'installation MicroK8s pour Windows WSL2
# À exécuter dans votre distribution WSL Ubuntu

set -e

echo "=========================================="
echo "Installation de MicroK8s sur WSL2"
echo "=========================================="
echo ""

# Vérification que nous sommes bien dans WSL
if ! grep -qi microsoft /proc/version; then
    echo "[✗] Ce script doit être exécuté dans WSL2"
    exit 1
fi

echo "[✓] Environnement WSL2 détecté"

# Vérification des privilèges
if [ "$EUID" -eq 0 ]; then
    echo "[✗] Ne pas exécuter ce script en tant que root"
    exit 1
fi

# Mise à jour du système
echo "[1/6] Mise à jour du système..."
sudo apt update
sudo apt upgrade -y

# Installation de snapd
echo "[2/6] Installation de snapd..."
sudo apt install -y snapd

# Démarrage de snapd (spécifique WSL)
echo "[3/6] Démarrage du service snapd..."
sudo systemctl enable snapd
sudo systemctl start snapd

# Attente de l'initialisation de snapd
echo "Attente de l'initialisation de snapd (30 secondes)..."
sleep 30

# Installation de MicroK8s
echo "[4/6] Installation de MicroK8s..."
sudo snap install microk8s --classic --channel=1.31/stable

# Configuration des permissions
echo "[5/6] Configuration des permissions..."
sudo usermod -a -G microk8s $USER
sudo chown -R $USER ~/.kube

# Configuration spécifique WSL2
echo "[6/6] Configuration spécifique WSL2..."

# Augmentation des limites de fichiers (important pour WSL2)
sudo sh -c 'echo "fs.inotify.max_user_watches=524288" >> /etc/sysctl.conf'
sudo sh -c 'echo "fs.inotify.max_user_instances=512" >> /etc/sysctl.conf'
sudo sysctl -p

# Attente du démarrage
echo "Attente du démarrage de MicroK8s..."
sudo microk8s status --wait-ready

# Configuration des alias
echo "Configuration des alias..."
cat >> ~/.bashrc << 'EOF'

# Aliases MicroK8s
alias kubectl='microk8s kubectl'
alias k='microk8s kubectl'
EOF

echo ""
echo "=========================================="
echo "Installation WSL2 terminée !"
echo "=========================================="
echo ""
echo "IMPORTANT pour WSL2 :"
echo "1. Fermez et rouvrez votre terminal WSL"
echo "2. Ou exécutez : newgrp microk8s"
echo ""
echo "Note : Sur WSL2, certains addons comme metallb"
echo "peuvent nécessiter une configuration réseau avancée."
echo ""
```

### 3.3 Particularités WSL2

**Limites de fichiers**

WSL2 a des limites par défaut plus basses que Linux natif. Le script les augmente automatiquement.

**Réseau**

Le réseau WSL2 est isolé de Windows. Pour accéder aux services depuis Windows, vous devrez :
- Utiliser NodePort plutôt que LoadBalancer
- Configurer le port forwarding manuellement

**Services**

Systemd ne démarre pas automatiquement dans WSL2. Le script s'assure que snapd est bien démarré.

---

## 4. Script d'Installation pour macOS

### 4.1 Script de Base macOS

```bash
#!/bin/bash
# Script d'installation MicroK8s pour macOS
# Utilise Multipass pour créer une VM Ubuntu

set -e

echo "=========================================="
echo "Installation de MicroK8s sur macOS"
echo "=========================================="
echo ""

# Vérification que nous sommes sur macOS
if [[ "$OSTYPE" != "darwin"* ]]; then
    echo "[✗] Ce script est conçu pour macOS uniquement"
    exit 1
fi

echo "[✓] Système macOS détecté"

# Vérification de Homebrew
if ! command -v brew &> /dev/null; then
    echo "[1/5] Installation de Homebrew..."
    /bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"
else
    echo "[1/5] Homebrew déjà installé"
fi

# Installation de Multipass
echo "[2/5] Installation de Multipass..."
if ! command -v multipass &> /dev/null; then
    brew install --cask multipass
    echo "[✓] Multipass installé"
else
    echo "[✓] Multipass déjà installé"
fi

# Création de la VM Ubuntu
echo "[3/5] Création de la VM Ubuntu pour MicroK8s..."
VM_NAME="microk8s-vm"

# Vérification si la VM existe déjà
if multipass list | grep -q "$VM_NAME"; then
    echo "[!] La VM $VM_NAME existe déjà"
    read -p "Voulez-vous la supprimer et la recréer ? (o/N) : " -n 1 -r
    echo
    if [[ $REPLY =~ ^[Oo]$ ]]; then
        multipass delete $VM_NAME
        multipass purge
    else
        echo "Installation annulée"
        exit 0
    fi
fi

# Création de la VM avec ressources suffisantes
multipass launch --name $VM_NAME \
    --cpus 2 \
    --memory 4G \
    --disk 40G \
    ubuntu

echo "[✓] VM créée"

# Installation de MicroK8s dans la VM
echo "[4/5] Installation de MicroK8s dans la VM..."
multipass exec $VM_NAME -- sudo snap install microk8s --classic --channel=1.31/stable

# Configuration de la VM
multipass exec $VM_NAME -- sudo usermod -a -G microk8s ubuntu
multipass exec $VM_NAME -- sudo chown -R ubuntu ~/.kube
multipass exec $VM_NAME -- sudo microk8s status --wait-ready

echo "[✓] MicroK8s installé dans la VM"

# Récupération de la configuration kubectl
echo "[5/5] Configuration de kubectl sur macOS..."
mkdir -p ~/.kube
multipass exec $VM_NAME -- sudo microk8s config > ~/.kube/config-microk8s

# Installation de kubectl sur macOS
if ! command -v kubectl &> /dev/null; then
    echo "Installation de kubectl..."
    brew install kubectl
fi

# Création d'un alias pour faciliter l'accès
cat >> ~/.zshrc << EOF

# MicroK8s sur Multipass
alias mk-shell='multipass shell $VM_NAME'
alias mk-stop='multipass stop $VM_NAME'
alias mk-start='multipass start $VM_NAME'
alias mk-status='multipass exec $VM_NAME -- sudo microk8s status'
export KUBECONFIG=~/.kube/config-microk8s
EOF

# Récupération de l'IP de la VM
VM_IP=$(multipass info $VM_NAME | grep IPv4 | awk '{print $2}')

echo ""
echo "=========================================="
echo "Installation macOS terminée !"
echo "=========================================="
echo ""
echo "Informations de la VM :"
echo "  Nom : $VM_NAME"
echo "  IP  : $VM_IP"
echo ""
echo "Commandes utiles :"
echo "  multipass shell $VM_NAME    # Accéder à la VM"
echo "  multipass stop $VM_NAME     # Arrêter la VM"
echo "  multipass start $VM_NAME    # Démarrer la VM"
echo ""
echo "Pour activer les alias, exécutez :"
echo "  source ~/.zshrc"
echo ""
echo "Vous pouvez maintenant utiliser kubectl depuis macOS !"
echo "  kubectl get nodes"
echo ""
```

### 4.2 Explication de l'Approche macOS

**Pourquoi Multipass ?**

MicroK8s n'est pas disponible directement sur macOS (qui n'utilise pas snap). Multipass crée une VM Ubuntu légère et optimisée.

**Ressources de la VM**

Le script alloue :
- 2 CPUs (minimum pour un bon fonctionnement)
- 4 GB de RAM (recommandé pour des applications réelles)
- 40 GB de disque (pour les images Docker et les données)

**Configuration kubectl**

Le script configure kubectl sur macOS pour qu'il se connecte directement à MicroK8s dans la VM.

---

## 5. Script d'Installation avec Vérifications Avancées

Ce script universel inclut de nombreuses vérifications et est plus robuste.

```bash
#!/bin/bash
# Script d'installation MicroK8s UNIVERSEL avec vérifications
# Compatible Ubuntu/Debian/CentOS/RHEL

set -e

# Couleurs pour les messages
RED='\033[0;31m'
GREEN='\033[0;32m'
YELLOW='\033[1;33m'
NC='\033[0m' # No Color

# Fonctions d'affichage
log_info() {
    echo -e "${GREEN}[INFO]${NC} $1"
}

log_warn() {
    echo -e "${YELLOW}[WARN]${NC} $1"
}

log_error() {
    echo -e "${RED}[ERROR]${NC} $1"
}

log_success() {
    echo -e "${GREEN}[✓]${NC} $1"
}

# Fonction de vérification des prérequis
check_prerequisites() {
    log_info "Vérification des prérequis système..."

    # Vérification de l'espace disque (minimum 20GB)
    AVAILABLE_SPACE=$(df -BG / | awk 'NR==2 {print $4}' | sed 's/G//')
    if [ "$AVAILABLE_SPACE" -lt 20 ]; then
        log_error "Espace disque insuffisant : ${AVAILABLE_SPACE}GB disponibles (minimum 20GB requis)"
        exit 1
    fi
    log_success "Espace disque suffisant : ${AVAILABLE_SPACE}GB"

    # Vérification de la RAM (minimum 2GB)
    TOTAL_RAM=$(free -g | awk 'NR==2 {print $2}')
    if [ "$TOTAL_RAM" -lt 2 ]; then
        log_warn "RAM faible : ${TOTAL_RAM}GB (recommandé : 4GB minimum)"
    else
        log_success "RAM suffisante : ${TOTAL_RAM}GB"
    fi

    # Vérification du nombre de CPUs
    CPU_COUNT=$(nproc)
    if [ "$CPU_COUNT" -lt 2 ]; then
        log_warn "Peu de CPUs : ${CPU_COUNT} (recommandé : 2 minimum)"
    else
        log_success "CPUs suffisants : ${CPU_COUNT}"
    fi

    # Vérification de la connexion Internet
    if ! ping -c 1 google.com &> /dev/null; then
        log_error "Pas de connexion Internet détectée"
        exit 1
    fi
    log_success "Connexion Internet active"
}

# Détection du système d'exploitation
detect_os() {
    if [ -f /etc/os-release ]; then
        . /etc/os-release
        OS=$ID
        OS_VERSION=$VERSION_ID
        log_info "Système détecté : $PRETTY_NAME"
    else
        log_error "Impossible de détecter le système d'exploitation"
        exit 1
    fi
}

# Installation selon l'OS
install_microk8s() {
    case $OS in
        ubuntu|debian)
            log_info "Installation pour Ubuntu/Debian..."
            sudo apt update
            sudo apt install -y snapd
            ;;
        centos|rhel|fedora)
            log_info "Installation pour CentOS/RHEL/Fedora..."
            sudo yum install -y epel-release
            sudo yum install -y snapd
            sudo systemctl enable --now snapd.socket
            sudo ln -sf /var/lib/snapd/snap /snap
            sleep 10
            ;;
        *)
            log_error "Système d'exploitation non supporté : $OS"
            exit 1
            ;;
    esac

    # Installation de MicroK8s
    log_info "Installation de MicroK8s..."
    sudo snap install microk8s --classic --channel=1.31/stable

    # Configuration
    log_info "Configuration des permissions..."
    sudo usermod -a -G microk8s $USER
    sudo chown -R $USER ~/.kube

    # Attente du démarrage
    log_info "Attente du démarrage de MicroK8s..."
    sudo microk8s status --wait-ready

    log_success "MicroK8s installé avec succès"
}

# Vérification post-installation
verify_installation() {
    log_info "Vérification de l'installation..."

    # Vérification du statut
    if sudo microk8s status --wait-ready --timeout 60 > /dev/null 2>&1; then
        log_success "MicroK8s fonctionne correctement"
    else
        log_error "MicroK8s ne répond pas"
        return 1
    fi

    # Vérification du nœud
    if sudo microk8s kubectl get nodes | grep -q "Ready"; then
        log_success "Nœud Kubernetes opérationnel"
    else
        log_warn "Le nœud n'est pas encore prêt"
    fi

    # Affichage de la version
    VERSION=$(sudo microk8s version | head -n 1)
    log_info "Version installée : $VERSION"
}

# Configuration post-installation
post_install_config() {
    log_info "Configuration post-installation..."

    # Création des alias
    if ! grep -q "alias kubectl='microk8s kubectl'" ~/.bashrc; then
        cat >> ~/.bashrc << 'EOF'

# Aliases MicroK8s
alias kubectl='microk8s kubectl'
alias k='microk8s kubectl'
alias mk='microk8s'
EOF
        log_success "Alias configurés"
    else
        log_info "Alias déjà configurés"
    fi

    # Génération du kubeconfig
    mkdir -p ~/.kube
    sudo microk8s config > ~/.kube/config 2>/dev/null || true

    if [ -f ~/.kube/config ]; then
        chmod 600 ~/.kube/config
        log_success "Kubeconfig généré"
    fi
}

# Affichage des informations finales
display_summary() {
    echo ""
    echo "=========================================="
    echo "Installation terminée avec succès !"
    echo "=========================================="
    echo ""
    echo "Prochaines étapes :"
    echo "1. Déconnectez-vous et reconnectez-vous (ou exécutez : newgrp microk8s)"
    echo "2. Activez les alias : source ~/.bashrc"
    echo "3. Vérifiez l'installation : microk8s status"
    echo ""
    echo "Addons recommandés :"
    echo "  microk8s enable dns"
    echo "  microk8s enable dashboard"
    echo "  microk8s enable storage"
    echo "  microk8s enable registry"
    echo ""
    echo "Documentation : https://microk8s.io/docs"
    echo ""
}

# PROGRAMME PRINCIPAL
main() {
    echo "=========================================="
    echo "Installation MicroK8s - Version Universelle"
    echo "=========================================="
    echo ""

    # Vérification utilisateur
    if [ "$EUID" -eq 0 ]; then
        log_error "Ne pas exécuter ce script en tant que root"
        exit 1
    fi

    # Exécution des étapes
    check_prerequisites
    detect_os
    install_microk8s
    verify_installation
    post_install_config
    display_summary
}

# Lancement du script
main "$@"
```

---

## 6. Script d'Installation avec Menu Interactif

Pour les utilisateurs qui préfèrent un assistant guidé :

```bash
#!/bin/bash
# Script d'installation MicroK8s INTERACTIF

set -e

# Couleurs
GREEN='\033[0;32m'
BLUE='\033[0;34m'
YELLOW='\033[1;33m'
NC='\033[0m'

# Fonction de menu
show_menu() {
    clear
    echo -e "${BLUE}╔════════════════════════════════════════╗${NC}"
    echo -e "${BLUE}║   Installation MicroK8s - Assistant   ║${NC}"
    echo -e "${BLUE}╔════════════════════════════════════════╗${NC}"
    echo ""
    echo "Sélectionnez votre type d'installation :"
    echo ""
    echo "1) Installation minimale (base seulement)"
    echo "2) Installation standard (avec DNS et storage)"
    echo "3) Installation complète (tous les addons essentiels)"
    echo "4) Installation personnalisée"
    echo "5) Quitter"
    echo ""
}

# Installation minimale
install_minimal() {
    echo -e "${GREEN}Installation minimale...${NC}"
    sudo snap install microk8s --classic --channel=1.31/stable
    configure_permissions
}

# Installation standard
install_standard() {
    echo -e "${GREEN}Installation standard...${NC}"
    install_minimal
    sudo microk8s status --wait-ready
    sudo microk8s enable dns
    sudo microk8s enable storage
}

# Installation complète
install_complete() {
    echo -e "${GREEN}Installation complète...${NC}"
    install_minimal
    sudo microk8s status --wait-ready

    # Liste des addons
    ADDONS=(dns dashboard storage registry ingress)

    for addon in "${ADDONS[@]}"; do
        echo "Activation de $addon..."
        sudo microk8s enable $addon
    done
}

# Installation personnalisée
install_custom() {
    echo -e "${GREEN}Installation personnalisée${NC}"
    echo ""

    # Sélection du canal
    echo "Choisissez le canal de version :"
    echo "1) 1.31/stable (recommandé)"
    echo "2) 1.30/stable"
    echo "3) latest/edge (dernière version)"
    read -p "Votre choix : " channel_choice

    case $channel_choice in
        1) CHANNEL="1.31/stable" ;;
        2) CHANNEL="1.30/stable" ;;
        3) CHANNEL="latest/edge" ;;
        *) CHANNEL="1.31/stable" ;;
    esac

    sudo snap install microk8s --classic --channel=$CHANNEL
    configure_permissions
    sudo microk8s status --wait-ready

    # Sélection des addons
    echo ""
    echo "Sélection des addons (o/n) :"

    declare -A ADDONS=(
        [dns]="DNS (recommandé)"
        [dashboard]="Dashboard web"
        [storage]="Stockage persistant"
        [registry]="Registry d'images privé"
        [ingress]="NGINX Ingress"
        [metallb]="Load Balancer MetalLB"
        [cert-manager]="Gestionnaire de certificats"
        [prometheus]="Monitoring Prometheus"
    )

    for addon in "${!ADDONS[@]}"; do
        read -p "Activer ${ADDONS[$addon]} ? (o/N) : " -n 1 -r
        echo
        if [[ $REPLY =~ ^[Oo]$ ]]; then
            if [ "$addon" = "metallb" ]; then
                read -p "Plage d'IP pour MetalLB (ex: 10.0.0.10-10.0.0.20) : " ip_range
                sudo microk8s enable metallb:$ip_range
            else
                sudo microk8s enable $addon
            fi
        fi
    done
}

# Configuration des permissions
configure_permissions() {
    echo "Configuration des permissions..."
    sudo usermod -a -G microk8s $USER
    sudo chown -R $USER ~/.kube

    # Alias
    if ! grep -q "alias kubectl='microk8s kubectl'" ~/.bashrc; then
        cat >> ~/.bashrc << 'EOF'

# Aliases MicroK8s
alias kubectl='microk8s kubectl'
alias k='microk8s kubectl'
EOF
    fi
}

# Programme principal
main() {
    if [ "$EUID" -eq 0 ]; then
        echo "Ne pas exécuter ce script en tant que root"
        exit 1
    fi

    while true; do
        show_menu
        read -p "Votre choix : " choice

        case $choice in
            1)
                install_minimal
                break
                ;;
            2)
                install_standard
                break
                ;;
            3)
                install_complete
                break
                ;;
            4)
                install_custom
                break
                ;;
            5)
                echo "Installation annulée"
                exit 0
                ;;
            *)
                echo "Choix invalide"
                sleep 2
                ;;
        esac
    done

    echo ""
    echo "========================================"
    echo "Installation terminée !"
    echo "========================================"
    echo ""
    echo "Déconnectez-vous et reconnectez-vous, puis exécutez :"
    echo "  microk8s status"
    echo ""
}

main "$@"
```

---

## 7. Utilisation des Scripts

### 7.1 Téléchargement et Préparation

```bash
# Télécharger le script (si hébergé en ligne)
wget https://votre-serveur.com/install-microk8s.sh

# Ou créer le fichier localement
nano install-microk8s.sh
# [Coller le contenu du script]

# Rendre le script exécutable
chmod +x install-microk8s.sh
```

### 7.2 Exécution

```bash
# Exécution simple
./install-microk8s.sh

# Avec redirection des logs
./install-microk8s.sh 2>&1 | tee installation.log

# Exécution à distance via SSH
ssh user@serveur 'bash -s' < install-microk8s.sh
```

### 7.3 Vérification après Installation

```bash
# Vérifier le statut
microk8s status

# Vérifier les nœuds
microk8s kubectl get nodes

# Vérifier tous les pods
microk8s kubectl get pods --all-namespaces

# Inspecter le cluster
microk8s inspect
```

---

## 8. Bonnes Pratiques

### 8.1 Versionnement des Scripts

- Conservez vos scripts dans un dépôt Git
- Incluez un numéro de version dans l'en-tête
- Documentez les changements dans un fichier CHANGELOG

### 8.2 Tests

Testez toujours vos scripts :
- Dans une VM jetable
- Sur différents systèmes d'exploitation
- Avec différentes configurations réseau

### 8.3 Sécurité

```bash
# Vérifier l'intégrité d'un script téléchargé
sha256sum install-microk8s.sh

# Comparer avec le hash officiel
echo "abc123... install-microk8s.sh" | sha256sum --check
```

### 8.4 Logs et Débogage

Ajoutez toujours du logging dans vos scripts :

```bash
# Activer le mode verbose
set -x  # Affiche chaque commande avant exécution

# Rediriger vers un fichier de log
exec 1> >(tee "install-$(date +%Y%m%d-%H%M%S).log")
exec 2>&1
```

---

## 9. Dépannage Courant

### 9.1 Erreur : "snap not found"

```bash
# Ubuntu/Debian
sudo apt update
sudo apt install snapd

# CentOS/RHEL
sudo yum install snapd
sudo systemctl start snapd
```

### 9.2 Erreur : "Permission denied"

```bash
# Vérifier l'appartenance au groupe
groups | grep microk8s

# Réappliquer les permissions
sudo usermod -a -G microk8s $USER
newgrp microk8s
```

### 9.3 MicroK8s ne démarre pas

```bash
# Vérifier les logs
sudo journalctl -u snap.microk8s.daemon-kubelet

# Réinitialiser MicroK8s
microk8s reset
sudo snap restart microk8s
```

### 9.4 Problèmes de ressources

```bash
# Vérifier les ressources système
free -h
df -h
top
```

---

## 10. Scripts Complémentaires

### 10.1 Script de Désinstallation

```bash
#!/bin/bash
# Script de désinstallation complète de MicroK8s

set -e

echo "=========================================="
echo "Désinstallation de MicroK8s"
echo "=========================================="

read -p "Êtes-vous sûr de vouloir désinstaller MicroK8s ? (o/N) : " -n 1 -r
echo
if [[ ! $REPLY =~ ^[Oo]$ ]]; then
    echo "Désinstallation annulée"
    exit 0
fi

# Arrêt de MicroK8s
echo "Arrêt de MicroK8s..."
sudo microk8s stop || true

# Suppression de MicroK8s
echo "Suppression de MicroK8s..."
sudo snap remove microk8s --purge

# Nettoyage des fichiers
echo "Nettoyage des fichiers..."
sudo rm -rf ~/.kube/config
sudo rm -rf /var/snap/microk8s

# Suppression du groupe
echo "Nettoyage des groupes..."
sudo deluser $USER microk8s || true

# Nettoyage des alias
if [ -f ~/.bashrc ]; then
    sed -i '/# Aliases MicroK8s/,+2d' ~/.bashrc
fi

echo ""
echo "Désinstallation terminée !"
echo "Redémarrez votre session pour appliquer les changements."
```

### 10.2 Script de Mise à Jour

```bash
#!/bin/bash
# Script de mise à jour de MicroK8s

set -e

echo "=========================================="
echo "Mise à jour de MicroK8s"
echo "=========================================="

# Vérification de la version actuelle
CURRENT=$(microk8s version --short 2>/dev/null || echo "inconnu")
echo "Version actuelle : $CURRENT"

# Mise à jour via snap
echo "Mise à jour en cours..."
sudo snap refresh microk8s --channel=1.31/stable

# Attente du redémarrage
echo "Attente du redémarrage..."
sudo microk8s status --wait-ready

# Vérification de la nouvelle version
NEW=$(microk8s version --short)
echo "Nouvelle version : $NEW"

echo ""
echo "Mise à jour terminée avec succès !"
```

---

## Conclusion

Ces scripts d'installation automatisée vous permettent de déployer rapidement et de manière fiable MicroK8s sur différentes plateformes. N'hésitez pas à les personnaliser selon vos besoins spécifiques et à les partager avec votre équipe.

**Points clés à retenir :**

- Utilisez toujours `set -e` pour arrêter le script en cas d'erreur
- Vérifiez les prérequis système avant l'installation
- Incluez des messages d'information clairs pour l'utilisateur
- Testez vos scripts dans un environnement de test avant production
- Documentez les variables de configuration modifiables
- Conservez des logs d'installation pour le débogage

Pour aller plus loin, consultez le chapitre suivant sur les scripts de maintenance.

⏭️ [Scripts de maintenance](/annexes/annexe-a-scripts-et-automatisation/scripts-de-maintenance.md)
