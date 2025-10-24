🔝 Retour au [Sommaire](/SOMMAIRE.md)

# Annexe A : Scripts et Automatisation

## Introduction à l'Automatisation avec MicroK8s

L'automatisation est l'un des piliers fondamentaux des pratiques DevOps modernes. Dans le contexte de MicroK8s et de Kubernetes en général, l'automatisation vous permet de gérer efficacement votre infrastructure, de réduire les erreurs humaines et d'accélérer considérablement vos opérations.

Cette annexe vous fournit une collection complète de scripts prêts à l'emploi pour automatiser les tâches courantes liées à MicroK8s. Que vous soyez débutant ou utilisateur avancé, vous trouverez ici des outils pour simplifier votre quotidien avec Kubernetes.

---

## Pourquoi Automatiser ?

### 1. Gains de Productivité

L'automatisation transforme des tâches qui prendraient plusieurs minutes (voire heures) en opérations de quelques secondes. Au lieu de saisir manuellement une dizaine de commandes, vous exécutez un seul script.

**Exemple concret :**
- Installation manuelle de MicroK8s : 10-15 minutes
- Installation automatisée : 2-3 minutes
- Configuration d'un cluster HA : 30-45 minutes manuellement vs 5 minutes avec un script

### 2. Reproductibilité

Un script garantit que la même séquence d'opérations est exécutée à chaque fois, exactement de la même manière. C'est essentiel pour :

- **Environnements multiples** : développement, test, production
- **Collaboration d'équipe** : tous les membres obtiennent la même configuration
- **Documentation vivante** : le script documente précisément ce qui est fait
- **Conformité** : respect des standards et des procédures

### 3. Réduction des Erreurs

Les erreurs humaines sont inévitables lors de tâches répétitives. Les scripts éliminent :

- Les fautes de frappe dans les commandes
- Les oublis d'étapes importantes
- Les incohérences de configuration
- Les erreurs de copier-coller

### 4. Scalabilité

Lorsque vous devez gérer plusieurs clusters ou environnements, l'automatisation devient indispensable :

- Déployer 10 clusters prend le même temps qu'en déployer 1
- Les modifications de configuration se propagent instantanément
- La maintenance est centralisée et simplifiée

---

## Types de Scripts dans cette Annexe

Cette annexe couvre trois catégories principales de scripts, chacune répondant à des besoins spécifiques.

### 1. Scripts d'Installation Automatisée

Ces scripts gèrent le déploiement initial de MicroK8s sur différentes plateformes.

**Ce qu'ils font :**
- Vérification des prérequis système
- Installation des dépendances nécessaires
- Déploiement de MicroK8s
- Configuration initiale
- Activation des addons essentiels

**Plateformes couvertes :**
- Ubuntu / Debian
- CentOS / RHEL / Fedora
- Windows (via WSL2)
- macOS (via Multipass)

**Niveaux de complexité :**
- Installation minimale (base seulement)
- Installation standard (addons essentiels)
- Installation complète (environnement complet)
- Installation interactive (assistant guidé)

### 2. Scripts de Maintenance

Ces scripts automatisent les tâches d'administration courantes et la maintenance préventive.

**Tâches couvertes :**
- Mise à jour de MicroK8s
- Nettoyage des ressources inutilisées
- Vérifications de santé du cluster
- Rotation des logs
- Gestion des certificats
- Sauvegarde de configuration
- Surveillance de l'espace disque
- Redémarrage des services problématiques

**Avantages :**
- Automatisation des tâches récurrentes
- Maintenance préventive planifiée
- Détection précoce des problèmes
- Historique des opérations

### 3. Scripts de Monitoring

Ces scripts collectent et analysent les métriques et les logs pour vous donner une vue d'ensemble de votre cluster.

**Fonctionnalités :**
- Collection de métriques système
- Analyse des ressources (CPU, RAM, disque)
- Vérification de l'état des pods
- Surveillance des nœuds
- Alertes simples
- Génération de rapports
- Export des données pour analyse

**Types de monitoring :**
- Monitoring en temps réel
- Rapports périodiques
- Alertes automatiques
- Dashboards textuels

---

## Prérequis pour Utiliser ces Scripts

### Connaissances de Base

Pour tirer le meilleur parti de ces scripts, vous devriez avoir une compréhension de base de :

**1. Ligne de commande Linux**
- Navigation dans le système de fichiers (`cd`, `ls`, `pwd`)
- Manipulation de fichiers (`cp`, `mv`, `rm`, `cat`)
- Édition de fichiers (`nano`, `vim`)
- Gestion des permissions (`chmod`, `chown`)

**2. Bash scripting (notions)**
- Variables
- Conditions (`if`, `else`)
- Boucles (`for`, `while`)
- Fonctions

**3. Concepts MicroK8s/Kubernetes**
- Architecture de base
- Pods, Services, Deployments
- Namespaces
- kubectl (commandes essentielles)

> **Note pour les débutants :** Ne vous inquiétez pas si vous n'êtes pas expert ! Chaque script est accompagné d'explications détaillées. Vous pouvez les utiliser sans comprendre tous les détails techniques, puis progressivement approfondir vos connaissances.

### Outils Nécessaires

**Sur votre système Linux :**
```bash
# Éditeur de texte
nano, vim, ou votre éditeur préféré

# Outils de base (généralement pré-installés)
bash
curl ou wget
grep, sed, awk
```

**Permissions :**
- Accès utilisateur normal (pas root)
- Capacité à utiliser `sudo`
- Appartenance au groupe `microk8s` (après installation)

---

## Structure et Organisation des Scripts

Tous les scripts de cette annexe suivent une structure cohérente pour faciliter la lecture et la maintenance.

### Anatomie d'un Script Type

```bash
#!/bin/bash
# [Titre et description du script]
# Auteur : [Nom]
# Date : [Date]
# Version : [Numéro de version]

# Configuration stricte des erreurs
set -e  # Arrête en cas d'erreur
set -u  # Erreur si variable non définie (optionnel)
set -o pipefail  # Erreur dans les pipes (optionnel)

# ============================================
# CONFIGURATION
# ============================================
# Variables globales modifiables
VARIABLE_1="valeur1"
VARIABLE_2="valeur2"

# ============================================
# FONCTIONS
# ============================================
# Fonctions réutilisables
function_name() {
    # Corps de la fonction
}

# ============================================
# VÉRIFICATIONS PRÉALABLES
# ============================================
# Vérifications avant exécution

# ============================================
# PROGRAMME PRINCIPAL
# ============================================
main() {
    # Logique principale du script
}

# Lancement du programme
main "$@"
```

### Conventions de Nommage

**Fichiers de scripts :**
```
install-microk8s.sh           # Installation
maintenance-cleanup.sh         # Maintenance
monitor-cluster-health.sh      # Monitoring
backup-config.sh              # Sauvegarde
```

**Variables :**
```bash
# Constantes (majuscules)
MICROK8S_CHANNEL="1.31/stable"
DEFAULT_ADDONS=(dns storage)

# Variables (minuscules avec underscores)
cluster_name="prod-cluster"
backup_dir="/var/backups/microk8s"
```

**Fonctions :**
```bash
# Verbes d'action en snake_case
check_prerequisites()
install_addons()
backup_configuration()
log_info()
```

---

## Comment Utiliser les Scripts

### Étape 1 : Téléchargement ou Création

**Option A : Télécharger depuis un dépôt**
```bash
# Cloner un dépôt Git
git clone https://github.com/votre-repo/microk8s-scripts.git
cd microk8s-scripts

# Ou télécharger un script individuel
wget https://exemple.com/scripts/install-microk8s.sh
```

**Option B : Créer localement**
```bash
# Créer un nouveau fichier
nano mon-script.sh

# Coller le contenu du script
# Sauvegarder avec Ctrl+O, Quitter avec Ctrl+X
```

### Étape 2 : Rendre Exécutable

```bash
# Donner les permissions d'exécution
chmod +x mon-script.sh

# Vérifier les permissions
ls -l mon-script.sh
# Devrait afficher : -rwxr-xr-x
```

### Étape 3 : Exécution

```bash
# Exécution simple
./mon-script.sh

# Exécution avec paramètres
./mon-script.sh --option valeur

# Exécution avec capture des logs
./mon-script.sh 2>&1 | tee execution.log
```

### Étape 4 : Vérification

Après l'exécution, vérifiez toujours que tout s'est bien passé :

```bash
# Vérifier le code de retour
echo $?  # 0 = succès, autre = erreur

# Vérifier les logs
cat execution.log

# Vérifier l'état de MicroK8s
microk8s status
```

---

## Personnalisation des Scripts

Tous les scripts de cette annexe sont conçus pour être facilement personnalisables.

### Variables de Configuration

Au début de chaque script, vous trouverez une section de configuration :

```bash
# ============================================
# CONFIGURATION - Personnalisez ici
# ============================================

# Version de MicroK8s à installer
MICROK8S_CHANNEL="1.31/stable"  # Modifiez selon vos besoins

# Addons à activer automatiquement
ADDONS=(
    dns
    dashboard
    storage
    # registry      # Décommentez pour activer
    # ingress       # Décommentez pour activer
)

# Répertoire de sauvegarde
BACKUP_DIR="/var/backups/microk8s"

# Adresse email pour les notifications
ADMIN_EMAIL="admin@example.com"
```

### Adapter à vos Besoins

**Exemple : Modifier la version de Kubernetes**
```bash
# Dans install-microk8s.sh
# Changez cette ligne :
MICROK8S_CHANNEL="1.31/stable"
# En :
MICROK8S_CHANNEL="1.30/stable"
```

**Exemple : Ajouter des addons personnalisés**
```bash
# Ajoutez vos propres addons à la liste
ADDONS=(
    dns
    storage
    prometheus    # Nouveau
    grafana       # Nouveau
)
```

---

## Bonnes Pratiques

### 1. Testez Toujours en Environnement de Test

Avant d'exécuter un script en production :

```bash
# Créez une VM de test
multipass launch --name test-k8s --memory 4G --disk 20G

# Testez le script dans la VM
multipass exec test-k8s -- bash -c "$(cat mon-script.sh)"

# Vérifiez les résultats
multipass exec test-k8s -- microk8s status

# Détruisez la VM après test
multipass delete test-k8s
multipass purge
```

### 2. Versionnez vos Scripts

Utilisez Git pour suivre les modifications :

```bash
# Initialiser un dépôt Git
git init
git add *.sh
git commit -m "Initial commit: MicroK8s automation scripts"

# Créer des branches pour les expérimentations
git checkout -b experimental-feature
# Testez vos modifications
git checkout main
git merge experimental-feature
```

### 3. Documentez vos Modifications

Maintenez un fichier CHANGELOG :

```markdown
# Changelog

## [1.2.0] - 2025-10-24
### Ajouté
- Support pour CentOS 9
- Vérification automatique de l'espace disque

### Modifié
- Amélioration de la détection du système d'exploitation
- Messages d'erreur plus explicites

### Corrigé
- Bug lors de l'activation de metallb
```

### 4. Utilisez des Logs

Activez le logging dans vos scripts :

```bash
# Au début du script
LOG_FILE="/var/log/microk8s-scripts/$(date +%Y%m%d-%H%M%S).log"
mkdir -p "$(dirname "$LOG_FILE")"

# Redirection de toutes les sorties
exec 1> >(tee -a "$LOG_FILE")
exec 2>&1

echo "Log de cette exécution : $LOG_FILE"
```

### 5. Gestion des Erreurs

Implémentez une gestion robuste des erreurs :

```bash
# Fonction de nettoyage en cas d'erreur
cleanup() {
    echo "Une erreur s'est produite. Nettoyage..."
    # Actions de nettoyage
}

# Associer la fonction aux signaux d'erreur
trap cleanup ERR EXIT
```

---

## Sécurité des Scripts

### Principes de Sécurité

**1. Ne jamais exécuter en tant que root**
```bash
if [ "$EUID" -eq 0 ]; then
    echo "N'exécutez pas ce script en tant que root"
    exit 1
fi
```

**2. Valider les entrées utilisateur**
```bash
read -p "Entrez le nom du cluster : " cluster_name
if [[ ! "$cluster_name" =~ ^[a-zA-Z0-9-]+$ ]]; then
    echo "Nom invalide. Utilisez uniquement a-z, A-Z, 0-9, -"
    exit 1
fi
```

**3. Protéger les informations sensibles**
```bash
# Ne jamais stocker de mots de passe en clair dans les scripts
# Utilisez des variables d'environnement ou des fichiers de configuration sécurisés

# Mauvais :
PASSWORD="mon_mot_de_passe"

# Bon :
PASSWORD="${ADMIN_PASSWORD:-}"
if [ -z "$PASSWORD" ]; then
    read -s -p "Entrez le mot de passe : " PASSWORD
    echo
fi
```

**4. Permissions de fichiers**
```bash
# Scripts exécutables mais pas modifiables par tous
chmod 755 script.sh

# Fichiers de configuration sensibles
chmod 600 config.conf
```

### Vérification de l'Intégrité

Avant d'exécuter un script téléchargé :

```bash
# Calculer le hash SHA256
sha256sum script.sh

# Comparer avec le hash officiel
echo "abc123def456... script.sh" | sha256sum --check

# Inspecter le contenu avant exécution
less script.sh
# ou
cat script.sh | less
```

---

## Débogage des Scripts

### Mode Debug

Activez le mode verbose pour voir chaque commande exécutée :

```bash
# Au début du script
set -x  # Active le mode trace

# Ou en ligne de commande
bash -x mon-script.sh
```

### Points de Contrôle

Ajoutez des points de contrôle pour le débogage :

```bash
function debug_checkpoint() {
    if [ "${DEBUG:-false}" = "true" ]; then
        echo "[DEBUG] $1"
        read -p "Appuyez sur Entrée pour continuer..."
    fi
}

# Utilisation
debug_checkpoint "Avant installation des addons"
install_addons
debug_checkpoint "Après installation des addons"
```

### Exécution par Étapes

Créez des scripts modulaires qui peuvent être exécutés étape par étape :

```bash
#!/bin/bash
# Script avec étapes numérotées

run_step_1() {
    echo "Étape 1 : Mise à jour du système"
    # ...
}

run_step_2() {
    echo "Étape 2 : Installation de MicroK8s"
    # ...
}

# Exécution
if [ "${STEP:-all}" = "all" ]; then
    run_step_1
    run_step_2
else
    run_step_${STEP}
fi
```

Usage :
```bash
# Exécuter toutes les étapes
./script.sh

# Exécuter seulement l'étape 2
STEP=2 ./script.sh
```

---

## Automatisation avec Cron

Pour exécuter des scripts de maintenance automatiquement :

### Configuration de base

```bash
# Éditer le crontab
crontab -e

# Exemples de planification
# Tous les jours à 2h du matin
0 2 * * * /chemin/vers/maintenance-daily.sh >> /var/log/cron-maintenance.log 2>&1

# Tous les lundis à 3h
0 3 * * 1 /chemin/vers/maintenance-weekly.sh >> /var/log/cron-maintenance.log 2>&1

# Toutes les heures
0 * * * * /chemin/vers/monitor-health.sh >> /var/log/cron-monitor.log 2>&1
```

### Script compatible Cron

Les scripts exécutés par cron nécessitent quelques adaptations :

```bash
#!/bin/bash
# Script compatible cron

# Définir le PATH complet (cron a un PATH minimal)
export PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin

# Définir les variables d'environnement nécessaires
export HOME=/home/utilisateur
export USER=utilisateur

# Charger le profil si nécessaire
source ~/.profile

# Logique du script
# ...
```

---

## Organisation de vos Scripts

### Structure de Dossiers Recommandée

```
microk8s-scripts/
├── README.md
├── install/
│   ├── install-ubuntu.sh
│   ├── install-centos.sh
│   ├── install-wsl2.sh
│   └── install-macos.sh
├── maintenance/
│   ├── daily-cleanup.sh
│   ├── weekly-update.sh
│   ├── certificate-renewal.sh
│   └── backup-config.sh
├── monitoring/
│   ├── cluster-health.sh
│   ├── resource-usage.sh
│   └── generate-report.sh
├── utils/
│   ├── common-functions.sh
│   └── colors.sh
├── config/
│   ├── default.conf
│   └── production.conf
└── logs/
    └── .gitkeep
```

### Fichier de Fonctions Communes

Créez un fichier de fonctions réutilisables :

```bash
# utils/common-functions.sh

# Couleurs
RED='\033[0;31m'
GREEN='\033[0;32m'
YELLOW='\033[1;33m'
NC='\033[0m'

# Fonctions de log
log_info() {
    echo -e "${GREEN}[INFO]${NC} $(date '+%Y-%m-%d %H:%M:%S') - $1"
}

log_error() {
    echo -e "${RED}[ERROR]${NC} $(date '+%Y-%m-%d %H:%M:%S') - $1" >&2
}

log_warn() {
    echo -e "${YELLOW}[WARN]${NC} $(date '+%Y-%m-%d %H:%M:%S') - $1"
}

# Vérification utilisateur
check_not_root() {
    if [ "$EUID" -eq 0 ]; then
        log_error "Ne pas exécuter en tant que root"
        exit 1
    fi
}

# Vérification MicroK8s
check_microk8s_installed() {
    if ! command -v microk8s &> /dev/null; then
        log_error "MicroK8s n'est pas installé"
        exit 1
    fi
}
```

Utilisation dans vos scripts :

```bash
#!/bin/bash
# Mon script

# Charger les fonctions communes
source "$(dirname "$0")/../utils/common-functions.sh"

# Utiliser les fonctions
check_not_root
check_microk8s_installed
log_info "Démarrage du script"
```

---

## Prochaines Sections

Cette annexe se compose de trois grandes parties :

1. **Scripts d'Installation Automatisée** (section suivante)
   - Installation sur différents systèmes d'exploitation
   - Scripts avec vérifications avancées
   - Assistant d'installation interactif
   - Scripts de désinstallation et mise à jour

2. **Scripts de Maintenance**
   - Nettoyage automatique
   - Mise à jour et patching
   - Gestion des certificats
   - Sauvegarde et restauration
   - Monitoring de santé

3. **Scripts de Monitoring**
   - Collection de métriques
   - Génération de rapports
   - Alertes automatiques
   - Dashboards textuels

---

## Ressources Complémentaires

### Documentation Officielle

- **MicroK8s Docs** : https://microk8s.io/docs
- **Kubernetes Docs** : https://kubernetes.io/docs
- **Bash Guide** : https://www.gnu.org/software/bash/manual/

### Outils Utiles

```bash
# Vérificateur de syntaxe bash
sudo apt install shellcheck
shellcheck mon-script.sh

# Formateur de code
sudo apt install shfmt
shfmt -w mon-script.sh

# Analyseur de sécurité
# https://github.com/koalaman/shellcheck
```

### Communauté et Support

- **Forum MicroK8s** : https://discuss.kubernetes.io/c/microk8s/
- **GitHub Issues** : https://github.com/canonical/microk8s/issues
- **Stack Overflow** : Tag `microk8s`

---

## Conclusion de l'Introduction

L'automatisation n'est pas réservée aux experts. Avec les scripts fournis dans cette annexe, vous pouvez immédiatement améliorer votre productivité et la fiabilité de vos opérations MicroK8s.

**Commencez simplement :**
1. Utilisez les scripts tels quels
2. Observez comment ils fonctionnent
3. Personnalisez progressivement selon vos besoins
4. Créez vos propres scripts en vous inspirant des exemples

**Rappelez-vous :**
- Testez toujours en environnement de développement
- Documentez vos modifications
- Versionnez vos scripts
- Partagez avec votre équipe

La section suivante détaille les scripts d'installation automatisée pour vous aider à déployer MicroK8s rapidement et efficacement sur n'importe quelle plateforme.

Bonne automatisation ! 🚀

⏭️ [Scripts d'installation automatisée](/annexes/annexe-a-scripts-et-automatisation/scripts-dinstallation-automatisee.md)


