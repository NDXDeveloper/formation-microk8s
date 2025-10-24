🔝 Retour au [Sommaire](/SOMMAIRE.md)

# Annexe A : Scripts de Maintenance MicroK8s

## Introduction

La maintenance régulière de votre cluster MicroK8s est essentielle pour garantir sa performance, sa stabilité et sa sécurité. Les scripts de maintenance automatisent les tâches récurrentes qui, effectuées manuellement, seraient chronophages et sources d'erreurs.

Cette section présente une collection complète de scripts de maintenance pour MicroK8s, couvrant tous les aspects du cycle de vie opérationnel : nettoyage, mises à jour, vérifications de santé, gestion des certificats, sauvegardes et bien plus encore.

**Ce que vous apprendrez :**
- Comment automatiser les tâches de maintenance quotidiennes
- Comment maintenir votre cluster en bonne santé
- Comment gérer les mises à jour en toute sécurité
- Comment prévenir les problèmes avant qu'ils ne surviennent
- Comment libérer de l'espace disque et optimiser les ressources

---

## 1. Script de Nettoyage Quotidien

Ce script effectue un nettoyage automatique des ressources inutilisées pour maintenir votre cluster propre et performant.

### 1.1 Script Complet

```bash
#!/bin/bash
# Script de nettoyage quotidien MicroK8s
# Nettoie les ressources inutilisées et libère de l'espace disque
# À exécuter quotidiennement via cron

set -e

# ============================================
# CONFIGURATION
# ============================================

# Répertoire des logs
LOG_DIR="/var/log/microk8s-maintenance"
LOG_FILE="${LOG_DIR}/cleanup-$(date +%Y%m%d-%H%M%S).log"

# Seuil de conservation des images (en jours)
IMAGE_RETENTION_DAYS=7

# Seuil de conservation des logs (en jours)
LOG_RETENTION_DAYS=30

# Activer/désactiver certaines opérations
CLEANUP_IMAGES=true
CLEANUP_CONTAINERS=true
CLEANUP_PODS=true
CLEANUP_LOGS=true
CLEANUP_CACHE=true

# ============================================
# FONCTIONS
# ============================================

# Couleurs pour les messages
RED='\033[0;31m'
GREEN='\033[0;32m'
YELLOW='\033[1;33m'
BLUE='\033[0;34m'
NC='\033[0m'

log_info() {
    echo -e "${GREEN}[INFO]${NC} $(date '+%Y-%m-%d %H:%M:%S') - $1" | tee -a "$LOG_FILE"
}

log_warn() {
    echo -e "${YELLOW}[WARN]${NC} $(date '+%Y-%m-%d %H:%M:%S') - $1" | tee -a "$LOG_FILE"
}

log_error() {
    echo -e "${RED}[ERROR]${NC} $(date '+%Y-%m-%d %H:%M:%S') - $1" | tee -a "$LOG_FILE"
}

log_success() {
    echo -e "${BLUE}[✓]${NC} $(date '+%Y-%m-%d %H:%M:%S') - $1" | tee -a "$LOG_FILE"
}

# Vérifier que MicroK8s est en cours d'exécution
check_microk8s_running() {
    if ! microk8s status --wait-ready --timeout 30 > /dev/null 2>&1; then
        log_error "MicroK8s n'est pas en cours d'exécution"
        exit 1
    fi
    log_info "MicroK8s est opérationnel"
}

# Afficher l'utilisation du disque avant nettoyage
show_disk_usage_before() {
    log_info "Utilisation du disque AVANT nettoyage :"
    df -h / | tee -a "$LOG_FILE"

    # Espace utilisé par containerd
    if [ -d /var/snap/microk8s/common/var/lib/containerd ]; then
        CONTAINERD_SIZE=$(du -sh /var/snap/microk8s/common/var/lib/containerd 2>/dev/null | cut -f1)
        log_info "Espace utilisé par containerd : $CONTAINERD_SIZE"
    fi
}

# Nettoyer les images Docker inutilisées
cleanup_images() {
    if [ "$CLEANUP_IMAGES" = false ]; then
        log_info "Nettoyage des images désactivé"
        return
    fi

    log_info "Nettoyage des images inutilisées..."

    # Lister les images avant nettoyage
    IMAGE_COUNT_BEFORE=$(microk8s ctr images ls -q | wc -l)
    log_info "Nombre d'images avant nettoyage : $IMAGE_COUNT_BEFORE"

    # Supprimer les images non utilisées de plus de X jours
    # Note : Cette commande est sûre, elle ne supprime que les images non référencées
    microk8s ctr images ls -q | while read image; do
        # Vérifier si l'image est utilisée par un conteneur
        if ! microk8s ctr containers ls -q | xargs -I {} microk8s ctr containers info {} 2>/dev/null | grep -q "$image"; then
            log_info "Suppression de l'image non utilisée : $image"
            microk8s ctr images rm "$image" 2>/dev/null || true
        fi
    done

    IMAGE_COUNT_AFTER=$(microk8s ctr images ls -q | wc -l)
    IMAGES_REMOVED=$((IMAGE_COUNT_BEFORE - IMAGE_COUNT_AFTER))
    log_success "Images supprimées : $IMAGES_REMOVED"
}

# Nettoyer les conteneurs arrêtés
cleanup_containers() {
    if [ "$CLEANUP_CONTAINERS" = false ]; then
        log_info "Nettoyage des conteneurs désactivé"
        return
    fi

    log_info "Nettoyage des conteneurs arrêtés..."

    # Compter les conteneurs avant nettoyage
    CONTAINER_COUNT_BEFORE=$(microk8s ctr containers ls -q | wc -l)
    log_info "Nombre de conteneurs avant : $CONTAINER_COUNT_BEFORE"

    # Note : containerd nettoie généralement automatiquement
    # Cette section est plus pour information

    log_success "Vérification des conteneurs terminée"
}

# Nettoyer les Pods en état Failed ou Completed
cleanup_pods() {
    if [ "$CLEANUP_PODS" = false ]; then
        log_info "Nettoyage des pods désactivé"
        return
    fi

    log_info "Nettoyage des pods terminés..."

    # Supprimer les pods Completed (jobs terminés)
    COMPLETED_PODS=$(microk8s kubectl get pods --all-namespaces --field-selector=status.phase==Succeeded -o json | jq -r '.items | length')
    if [ "$COMPLETED_PODS" -gt 0 ]; then
        log_info "Suppression de $COMPLETED_PODS pods Completed"
        microk8s kubectl delete pods --all-namespaces --field-selector=status.phase==Succeeded --ignore-not-found=true
    fi

    # Supprimer les pods Failed (plus anciens que 24h pour éviter de perdre les logs)
    FAILED_PODS=$(microk8s kubectl get pods --all-namespaces --field-selector=status.phase==Failed -o json | jq -r '.items | length')
    if [ "$FAILED_PODS" -gt 0 ]; then
        log_warn "Pods en état Failed détectés : $FAILED_PODS"
        log_info "Suppression des pods Failed de plus de 24h"

        # Obtenir la liste des pods Failed
        microk8s kubectl get pods --all-namespaces --field-selector=status.phase==Failed -o json | \
        jq -r '.items[] | select(.metadata.creationTimestamp | fromdateiso8601 < (now - 86400)) |
               "\(.metadata.namespace) \(.metadata.name)"' | \
        while read namespace pod; do
            if [ -n "$pod" ]; then
                log_info "Suppression du pod Failed : $namespace/$pod"
                microk8s kubectl delete pod "$pod" -n "$namespace" --ignore-not-found=true
            fi
        done
    fi

    # Supprimer les pods Evicted
    EVICTED_PODS=$(microk8s kubectl get pods --all-namespaces -o json | jq -r '.items[] | select(.status.reason == "Evicted") | "\(.metadata.namespace) \(.metadata.name)"' | wc -l)
    if [ "$EVICTED_PODS" -gt 0 ]; then
        log_info "Suppression de $EVICTED_PODS pods Evicted"
        microk8s kubectl get pods --all-namespaces -o json | \
        jq -r '.items[] | select(.status.reason == "Evicted") | "\(.metadata.namespace) \(.metadata.name)"' | \
        while read namespace pod; do
            if [ -n "$pod" ]; then
                microk8s kubectl delete pod "$pod" -n "$namespace" --ignore-not-found=true
            fi
        done
    fi

    log_success "Nettoyage des pods terminé"
}

# Nettoyer les anciens logs
cleanup_logs() {
    if [ "$CLEANUP_LOGS" = false ]; then
        log_info "Nettoyage des logs désactivé"
        return
    fi

    log_info "Nettoyage des anciens logs..."

    # Nettoyer les logs de maintenance
    if [ -d "$LOG_DIR" ]; then
        LOGS_BEFORE=$(find "$LOG_DIR" -type f -name "*.log" | wc -l)
        find "$LOG_DIR" -type f -name "*.log" -mtime +${LOG_RETENTION_DAYS} -delete
        LOGS_AFTER=$(find "$LOG_DIR" -type f -name "*.log" | wc -l)
        LOGS_REMOVED=$((LOGS_BEFORE - LOGS_AFTER))
        log_info "Logs de maintenance supprimés : $LOGS_REMOVED"
    fi

    # Nettoyer les logs système de MicroK8s (avec précaution)
    MICROK8S_LOG_DIR="/var/snap/microk8s/common/var/log"
    if [ -d "$MICROK8S_LOG_DIR" ]; then
        log_info "Rotation des logs système MicroK8s"
        find "$MICROK8S_LOG_DIR" -type f -name "*.log.*" -mtime +${LOG_RETENTION_DAYS} -delete 2>/dev/null || true
    fi

    # Nettoyer les logs de containerd
    CONTAINERD_LOG="/var/snap/microk8s/common/var/log/containerd.log"
    if [ -f "$CONTAINERD_LOG" ]; then
        LOG_SIZE=$(stat -f%z "$CONTAINERD_LOG" 2>/dev/null || stat -c%s "$CONTAINERD_LOG" 2>/dev/null || echo "0")
        # Si le log dépasse 100MB, le tronquer
        if [ "$LOG_SIZE" -gt 104857600 ]; then
            log_info "Rotation du log containerd (taille: $(($LOG_SIZE / 1048576))MB)"
            : > "$CONTAINERD_LOG"
        fi
    fi

    log_success "Nettoyage des logs terminé"
}

# Nettoyer le cache et les fichiers temporaires
cleanup_cache() {
    if [ "$CLEANUP_CACHE" = false ]; then
        log_info "Nettoyage du cache désactivé"
        return
    fi

    log_info "Nettoyage du cache..."

    # Nettoyer le cache apt/yum si disponible
    if command -v apt-get &> /dev/null; then
        log_info "Nettoyage du cache APT"
        sudo apt-get clean 2>/dev/null || true
        sudo apt-get autoclean 2>/dev/null || true
    elif command -v yum &> /dev/null; then
        log_info "Nettoyage du cache YUM"
        sudo yum clean all 2>/dev/null || true
    fi

    # Nettoyer le cache snap
    log_info "Nettoyage des anciennes révisions snap"
    SNAP_REVISIONS=$(snap list --all microk8s | tail -n +2 | wc -l)
    if [ "$SNAP_REVISIONS" -gt 2 ]; then
        log_info "Suppression des anciennes révisions de microk8s snap"
        snap list --all microk8s | awk '/disabled/{print $3}' | while read revision; do
            sudo snap remove microk8s --revision="$revision" 2>/dev/null || true
        done
    fi

    log_success "Nettoyage du cache terminé"
}

# Afficher l'utilisation du disque après nettoyage
show_disk_usage_after() {
    log_info "Utilisation du disque APRÈS nettoyage :"
    df -h / | tee -a "$LOG_FILE"

    if [ -d /var/snap/microk8s/common/var/lib/containerd ]; then
        CONTAINERD_SIZE=$(du -sh /var/snap/microk8s/common/var/lib/containerd 2>/dev/null | cut -f1)
        log_info "Espace utilisé par containerd : $CONTAINERD_SIZE"
    fi
}

# Génération du rapport de nettoyage
generate_report() {
    log_info "=========================================="
    log_info "RAPPORT DE NETTOYAGE QUOTIDIEN"
    log_info "=========================================="
    log_info "Date : $(date '+%Y-%m-%d %H:%M:%S')"
    log_info "Hostname : $(hostname)"
    log_info "MicroK8s version : $(microk8s version --short 2>/dev/null || echo 'N/A')"
    log_info "=========================================="
    log_success "Nettoyage quotidien terminé avec succès"
    log_info "Log complet disponible : $LOG_FILE"
}

# ============================================
# PROGRAMME PRINCIPAL
# ============================================

main() {
    # Créer le répertoire de logs
    mkdir -p "$LOG_DIR"

    log_info "=========================================="
    log_info "DÉMARRAGE DU NETTOYAGE QUOTIDIEN"
    log_info "=========================================="

    # Vérifications préalables
    check_microk8s_running

    # Afficher l'état avant nettoyage
    show_disk_usage_before

    echo "" | tee -a "$LOG_FILE"

    # Exécuter les opérations de nettoyage
    cleanup_pods
    cleanup_images
    cleanup_containers
    cleanup_logs
    cleanup_cache

    echo "" | tee -a "$LOG_FILE"

    # Afficher l'état après nettoyage
    show_disk_usage_after

    echo "" | tee -a "$LOG_FILE"

    # Générer le rapport
    generate_report
}

# Gestion des erreurs
trap 'log_error "Une erreur est survenue. Vérifiez le log : $LOG_FILE"' ERR

# Lancement
main "$@"
```

### 1.2 Explication du Script

**Structure générale :**

Le script est divisé en sections logiques :
1. **Configuration** : variables modifiables
2. **Fonctions** : opérations réutilisables
3. **Programme principal** : orchestration des tâches

**Opérations de nettoyage :**

1. **Nettoyage des images** : supprime les images Docker non utilisées
2. **Nettoyage des conteneurs** : vérifie les conteneurs arrêtés
3. **Nettoyage des pods** : supprime les pods Failed, Completed et Evicted
4. **Nettoyage des logs** : rotation et suppression des anciens logs
5. **Nettoyage du cache** : libère l'espace du cache système

**Fonctionnalités de sécurité :**

```bash
set -e  # Arrête en cas d'erreur
trap 'log_error "..."' ERR  # Capture les erreurs
```

### 1.3 Configuration du Nettoyage Automatique

Pour exécuter ce script automatiquement chaque nuit :

```bash
# Éditer le crontab
crontab -e

# Ajouter cette ligne pour exécution à 2h du matin
0 2 * * * /chemin/vers/cleanup-daily.sh >> /var/log/microk8s-cron.log 2>&1
```

---

## 2. Script de Mise à Jour

Ce script gère les mises à jour de MicroK8s de manière sécurisée avec des vérifications avant et après.

### 2.1 Script de Mise à Jour Sécurisée

```bash
#!/bin/bash
# Script de mise à jour sécurisée de MicroK8s
# Effectue une sauvegarde avant mise à jour et vérifie l'état après

set -e

# ============================================
# CONFIGURATION
# ============================================

# Canal de mise à jour
UPDATE_CHANNEL="1.31/stable"  # Modifier selon vos besoins

# Répertoire de sauvegarde
BACKUP_DIR="/var/backups/microk8s"
BACKUP_DATE=$(date +%Y%m%d-%H%M%S)
BACKUP_PATH="${BACKUP_DIR}/pre-update-${BACKUP_DATE}"

# Timeout pour les opérations (en secondes)
TIMEOUT=300

# Mode dry-run (tester sans appliquer)
DRY_RUN=false

# ============================================
# FONCTIONS
# ============================================

RED='\033[0;31m'
GREEN='\033[0;32m'
YELLOW='\033[1;33m'
BLUE='\033[0;34m'
NC='\033[0m'

log_info() {
    echo -e "${GREEN}[INFO]${NC} $(date '+%Y-%m-%d %H:%M:%S') - $1"
}

log_warn() {
    echo -e "${YELLOW}[WARN]${NC} $(date '+%Y-%m-%d %H:%M:%S') - $1"
}

log_error() {
    echo -e "${RED}[ERROR]${NC} $(date '+%Y-%m-%d %H:%M:%S') - $1"
}

log_success() {
    echo -e "${BLUE}[✓]${NC} $(date '+%Y-%m-%d %H:%M:%S') - $1"
}

# Vérifier les prérequis
check_prerequisites() {
    log_info "Vérification des prérequis..."

    # Vérifier que MicroK8s est installé
    if ! command -v microk8s &> /dev/null; then
        log_error "MicroK8s n'est pas installé"
        exit 1
    fi

    # Vérifier que MicroK8s fonctionne
    if ! microk8s status --wait-ready --timeout 30 > /dev/null 2>&1; then
        log_error "MicroK8s n'est pas en cours d'exécution"
        exit 1
    fi

    # Vérifier l'espace disque (minimum 2GB)
    AVAILABLE_SPACE=$(df -BG / | awk 'NR==2 {print $4}' | sed 's/G//')
    if [ "$AVAILABLE_SPACE" -lt 2 ]; then
        log_error "Espace disque insuffisant : ${AVAILABLE_SPACE}GB (minimum 2GB requis)"
        exit 1
    fi

    log_success "Prérequis vérifiés"
}

# Afficher les informations actuelles
show_current_info() {
    log_info "=========================================="
    log_info "INFORMATIONS SYSTÈME ACTUELLES"
    log_info "=========================================="

    # Version actuelle
    CURRENT_VERSION=$(microk8s version --short 2>/dev/null)
    log_info "Version MicroK8s actuelle : $CURRENT_VERSION"

    # Canal actuel
    CURRENT_CHANNEL=$(snap info microk8s | grep "tracking:" | awk '{print $2}')
    log_info "Canal actuel : $CURRENT_CHANNEL"

    # État du cluster
    log_info "État du cluster :"
    microk8s status | head -n 10

    # Nombre de pods
    POD_COUNT=$(microk8s kubectl get pods --all-namespaces --no-headers | wc -l)
    log_info "Nombre de pods actifs : $POD_COUNT"

    # Addons actifs
    log_info "Addons actifs :"
    microk8s status | grep "enabled" | awk '{print "  - " $1}'

    echo ""
}

# Créer une sauvegarde avant mise à jour
create_backup() {
    log_info "Création d'une sauvegarde avant mise à jour..."

    mkdir -p "$BACKUP_PATH"

    # Sauvegarder la configuration kubeconfig
    log_info "Sauvegarde de kubeconfig..."
    microk8s config > "${BACKUP_PATH}/kubeconfig.yaml"

    # Sauvegarder la liste des addons
    log_info "Sauvegarde de la liste des addons..."
    microk8s status > "${BACKUP_PATH}/addons-status.txt"

    # Sauvegarder toutes les ressources Kubernetes
    log_info "Sauvegarde des ressources Kubernetes..."
    for namespace in $(microk8s kubectl get namespaces -o jsonpath='{.items[*].metadata.name}'); do
        log_info "  Sauvegarde du namespace: $namespace"
        mkdir -p "${BACKUP_PATH}/resources/${namespace}"

        # Sauvegarder les différents types de ressources
        for resource in deployments services configmaps secrets pods statefulsets daemonsets; do
            microk8s kubectl get "$resource" -n "$namespace" -o yaml > "${BACKUP_PATH}/resources/${namespace}/${resource}.yaml" 2>/dev/null || true
        done
    done

    # Sauvegarder les informations sur les PV et PVC
    log_info "Sauvegarde des volumes persistants..."
    microk8s kubectl get pv -o yaml > "${BACKUP_PATH}/persistent-volumes.yaml" 2>/dev/null || true
    microk8s kubectl get pvc --all-namespaces -o yaml > "${BACKUP_PATH}/persistent-volume-claims.yaml" 2>/dev/null || true

    # Créer un fichier d'information
    cat > "${BACKUP_PATH}/backup-info.txt" << EOF
Backup créé le : $(date)
Version MicroK8s : $(microk8s version --short 2>/dev/null)
Canal : $(snap info microk8s | grep "tracking:" | awk '{print $2}')
Hostname : $(hostname)
EOF

    log_success "Sauvegarde créée dans : $BACKUP_PATH"
}

# Vérifier les mises à jour disponibles
check_available_updates() {
    log_info "Vérification des mises à jour disponibles..."

    # Rafraîchir les informations snap
    sudo snap refresh --list 2>/dev/null | grep microk8s || log_info "Aucune mise à jour disponible actuellement"

    # Afficher les versions disponibles
    log_info "Versions disponibles sur le canal $UPDATE_CHANNEL :"
    snap info microk8s --channel="$UPDATE_CHANNEL" | grep "  $UPDATE_CHANNEL:"
}

# Effectuer la mise à jour
perform_update() {
    if [ "$DRY_RUN" = true ]; then
        log_warn "Mode DRY-RUN activé - aucune modification ne sera effectuée"
        log_info "Commande qui serait exécutée : sudo snap refresh microk8s --channel=$UPDATE_CHANNEL"
        return
    fi

    log_info "=========================================="
    log_info "DÉMARRAGE DE LA MISE À JOUR"
    log_info "=========================================="

    # Effectuer la mise à jour
    log_info "Mise à jour vers le canal : $UPDATE_CHANNEL"
    sudo snap refresh microk8s --channel="$UPDATE_CHANNEL"

    log_success "Mise à jour snap terminée"
}

# Attendre que MicroK8s soit prêt après mise à jour
wait_for_ready() {
    log_info "Attente du redémarrage de MicroK8s..."

    # Attendre quelques secondes pour que les services démarrent
    sleep 10

    # Attendre que MicroK8s soit prêt
    if microk8s status --wait-ready --timeout "$TIMEOUT"; then
        log_success "MicroK8s est prêt"
    else
        log_error "MicroK8s n'a pas redémarré dans le délai imparti"
        log_error "Vérifiez manuellement avec : microk8s inspect"
        exit 1
    fi
}

# Vérifier l'état après mise à jour
verify_after_update() {
    log_info "=========================================="
    log_info "VÉRIFICATION POST-MISE À JOUR"
    log_info "=========================================="

    # Nouvelle version
    NEW_VERSION=$(microk8s version --short 2>/dev/null)
    log_info "Nouvelle version : $NEW_VERSION"

    # Vérifier l'état du cluster
    log_info "État du cluster après mise à jour :"
    microk8s status

    # Vérifier que tous les nœuds sont prêts
    log_info "Vérification des nœuds..."
    if microk8s kubectl get nodes | grep -q "NotReady"; then
        log_warn "Certains nœuds ne sont pas prêts"
        microk8s kubectl get nodes
    else
        log_success "Tous les nœuds sont prêts"
    fi

    # Vérifier les pods système
    log_info "Vérification des pods système (kube-system)..."
    FAILED_PODS=$(microk8s kubectl get pods -n kube-system --no-headers | grep -v "Running\|Completed" | wc -l)

    if [ "$FAILED_PODS" -gt 0 ]; then
        log_warn "$FAILED_PODS pods système ne sont pas en état Running"
        microk8s kubectl get pods -n kube-system
    else
        log_success "Tous les pods système fonctionnent correctement"
    fi

    # Comparer le nombre de pods
    NEW_POD_COUNT=$(microk8s kubectl get pods --all-namespaces --no-headers | wc -l)
    log_info "Nombre de pods après mise à jour : $NEW_POD_COUNT"

    # Vérifier les addons
    log_info "Vérification des addons..."
    microk8s status | grep "enabled" | awk '{print "  - " $1}'

    echo ""
}

# Test de fonctionnement basique
run_smoke_tests() {
    log_info "=========================================="
    log_info "TESTS DE FONCTIONNEMENT"
    log_info "=========================================="

    # Test 1 : Kubectl fonctionne
    log_info "Test 1 : Vérification de kubectl..."
    if microk8s kubectl version --short &> /dev/null; then
        log_success "kubectl fonctionne"
    else
        log_error "kubectl ne répond pas"
        return 1
    fi

    # Test 2 : API server répond
    log_info "Test 2 : Vérification de l'API server..."
    if microk8s kubectl get --raw /healthz &> /dev/null; then
        log_success "API server répond"
    else
        log_error "API server ne répond pas"
        return 1
    fi

    # Test 3 : Déploiement de test
    log_info "Test 3 : Déploiement d'un pod de test..."
    microk8s kubectl run test-pod --image=busybox --restart=Never --command -- sleep 30 &> /dev/null

    # Attendre que le pod soit prêt
    sleep 5

    if microk8s kubectl get pod test-pod --no-headers | grep -q "Running\|Completed"; then
        log_success "Pod de test déployé avec succès"
        microk8s kubectl delete pod test-pod --ignore-not-found=true &> /dev/null
    else
        log_warn "Le pod de test n'a pas démarré correctement"
        microk8s kubectl delete pod test-pod --ignore-not-found=true &> /dev/null
    fi

    log_success "Tests de fonctionnement terminés"
}

# Générer le rapport de mise à jour
generate_update_report() {
    log_info "=========================================="
    log_info "RAPPORT DE MISE À JOUR"
    log_info "=========================================="
    log_info "Date de mise à jour : $(date '+%Y-%m-%d %H:%M:%S')"
    log_info "Version précédente : $CURRENT_VERSION"
    log_info "Version actuelle : $(microk8s version --short 2>/dev/null)"
    log_info "Canal : $UPDATE_CHANNEL"
    log_info "Sauvegarde disponible : $BACKUP_PATH"
    log_info "=========================================="
    log_success "Mise à jour terminée avec succès"

    echo ""
    log_info "En cas de problème, vous pouvez revenir en arrière avec :"
    log_info "  sudo snap revert microk8s"
    echo ""
}

# ============================================
# PROGRAMME PRINCIPAL
# ============================================

main() {
    echo "=========================================="
    echo "SCRIPT DE MISE À JOUR MICROK8S"
    echo "=========================================="
    echo ""

    # Parse des arguments
    while [[ $# -gt 0 ]]; do
        case $1 in
            --dry-run)
                DRY_RUN=true
                shift
                ;;
            --channel)
                UPDATE_CHANNEL="$2"
                shift 2
                ;;
            *)
                log_error "Argument inconnu : $1"
                echo "Usage: $0 [--dry-run] [--channel CANAL]"
                exit 1
                ;;
        esac
    done

    # Vérifications préalables
    check_prerequisites

    # Afficher les informations actuelles
    show_current_info

    # Vérifier les mises à jour disponibles
    check_available_updates

    echo ""
    if [ "$DRY_RUN" = false ]; then
        read -p "Voulez-vous continuer la mise à jour ? (o/N) : " -n 1 -r
        echo
        if [[ ! $REPLY =~ ^[Oo]$ ]]; then
            log_info "Mise à jour annulée"
            exit 0
        fi
    fi

    # Créer une sauvegarde
    create_backup

    # Effectuer la mise à jour
    perform_update

    if [ "$DRY_RUN" = false ]; then
        # Attendre que MicroK8s soit prêt
        wait_for_ready

        # Vérifier l'état après mise à jour
        verify_after_update

        # Tests de fonctionnement
        run_smoke_tests

        # Générer le rapport
        generate_update_report
    fi
}

# Gestion des erreurs
trap 'log_error "Erreur lors de la mise à jour. Sauvegarde disponible : $BACKUP_PATH"' ERR

# Lancement
main "$@"
```

### 2.2 Utilisation du Script

**Mode normal (avec confirmation) :**
```bash
./update-microk8s.sh
```

**Mode dry-run (simulation) :**
```bash
./update-microk8s.sh --dry-run
```

**Spécifier un canal différent :**
```bash
./update-microk8s.sh --channel 1.30/stable
```

### 2.3 Planification des Mises à Jour

Pour automatiser les mises à jour hebdomadaires :

```bash
# Éditer le crontab
crontab -e

# Exécuter tous les dimanches à 3h du matin
0 3 * * 0 /chemin/vers/update-microk8s.sh >> /var/log/microk8s-update.log 2>&1
```

---

## 3. Script de Vérification de Santé

Ce script effectue des vérifications complètes de l'état de votre cluster.

### 3.1 Script de Health Check

```bash
#!/bin/bash
# Script de vérification de santé MicroK8s
# Effectue des vérifications complètes de l'état du cluster

set -e

# ============================================
# CONFIGURATION
# ============================================

# Seuils d'alerte
CPU_THRESHOLD=80         # Pourcentage
MEMORY_THRESHOLD=85      # Pourcentage
DISK_THRESHOLD=90        # Pourcentage
POD_RESTART_THRESHOLD=5  # Nombre de redémarrages

# Mode verbeux
VERBOSE=false

# Envoyer un email en cas de problème
SEND_EMAIL=false
EMAIL_TO="admin@example.com"

# ============================================
# FONCTIONS
# ============================================

RED='\033[0;31m'
GREEN='\033[0;32m'
YELLOW='\033[1;33m'
NC='\033[0m'

# Compteurs de problèmes
WARNINGS=0
ERRORS=0

log_info() {
    echo -e "${GREEN}[INFO]${NC} $1"
}

log_warn() {
    echo -e "${YELLOW}[WARN]${NC} $1"
    ((WARNINGS++))
}

log_error() {
    echo -e "${RED}[ERROR]${NC} $1"
    ((ERRORS++))
}

log_success() {
    echo -e "${GREEN}[✓]${NC} $1"
}

# Vérifier que MicroK8s fonctionne
check_microk8s_running() {
    echo "=========================================="
    echo "1. VÉRIFICATION DE MICROK8S"
    echo "=========================================="

    if microk8s status --wait-ready --timeout 30 > /dev/null 2>&1; then
        log_success "MicroK8s est en cours d'exécution"

        # Afficher la version
        VERSION=$(microk8s version --short 2>/dev/null)
        log_info "Version : $VERSION"
    else
        log_error "MicroK8s n'est pas en cours d'exécution"
        return 1
    fi

    echo ""
}

# Vérifier l'état des nœuds
check_nodes() {
    echo "=========================================="
    echo "2. VÉRIFICATION DES NŒUDS"
    echo "=========================================="

    # Obtenir le nombre de nœuds
    NODE_COUNT=$(microk8s kubectl get nodes --no-headers | wc -l)
    log_info "Nombre de nœuds : $NODE_COUNT"

    # Vérifier que tous les nœuds sont Ready
    NOT_READY=$(microk8s kubectl get nodes --no-headers | grep -v "Ready" | wc -l)

    if [ "$NOT_READY" -gt 0 ]; then
        log_error "$NOT_READY nœud(s) ne sont pas Ready"
        microk8s kubectl get nodes
    else
        log_success "Tous les nœuds sont Ready"
    fi

    # Afficher les informations détaillées si mode verbeux
    if [ "$VERBOSE" = true ]; then
        echo ""
        log_info "Détails des nœuds :"
        microk8s kubectl get nodes -o wide
    fi

    echo ""
}

# Vérifier l'utilisation des ressources système
check_system_resources() {
    echo "=========================================="
    echo "3. VÉRIFICATION DES RESSOURCES SYSTÈME"
    echo "=========================================="

    # CPU
    CPU_USAGE=$(top -bn1 | grep "Cpu(s)" | awk '{print $2}' | cut -d'%' -f1 | cut -d'.' -f1)
    log_info "Utilisation CPU : ${CPU_USAGE}%"

    if [ "$CPU_USAGE" -gt "$CPU_THRESHOLD" ]; then
        log_warn "Utilisation CPU élevée (seuil: ${CPU_THRESHOLD}%)"
    else
        log_success "Utilisation CPU normale"
    fi

    # Mémoire
    MEMORY_USAGE=$(free | grep Mem | awk '{print int($3/$2 * 100)}')
    log_info "Utilisation mémoire : ${MEMORY_USAGE}%"

    if [ "$MEMORY_USAGE" -gt "$MEMORY_THRESHOLD" ]; then
        log_warn "Utilisation mémoire élevée (seuil: ${MEMORY_THRESHOLD}%)"
    else
        log_success "Utilisation mémoire normale"
    fi

    # Disque
    DISK_USAGE=$(df -h / | tail -1 | awk '{print $5}' | sed 's/%//')
    log_info "Utilisation disque : ${DISK_USAGE}%"

    if [ "$DISK_USAGE" -gt "$DISK_THRESHOLD" ]; then
        log_error "Utilisation disque critique (seuil: ${DISK_THRESHOLD}%)"
    else
        log_success "Utilisation disque normale"
    fi

    # Load average
    LOAD_AVG=$(uptime | awk -F'load average:' '{print $2}')
    log_info "Load average :$LOAD_AVG"

    echo ""
}

# Vérifier les pods système
check_system_pods() {
    echo "=========================================="
    echo "4. VÉRIFICATION DES PODS SYSTÈME"
    echo "=========================================="

    # Vérifier les pods dans kube-system
    TOTAL_PODS=$(microk8s kubectl get pods -n kube-system --no-headers | wc -l)
    RUNNING_PODS=$(microk8s kubectl get pods -n kube-system --no-headers | grep "Running" | wc -l)

    log_info "Pods système : $RUNNING_PODS/$TOTAL_PODS en cours d'exécution"

    # Vérifier les pods qui ne sont pas Running
    NOT_RUNNING=$(microk8s kubectl get pods -n kube-system --no-headers | grep -v "Running\|Completed" | wc -l)

    if [ "$NOT_RUNNING" -gt 0 ]; then
        log_error "$NOT_RUNNING pod(s) système ne sont pas Running"
        microk8s kubectl get pods -n kube-system | grep -v "Running\|Completed\|NAME"
    else
        log_success "Tous les pods système fonctionnent correctement"
    fi

    echo ""
}

# Vérifier tous les pods
check_all_pods() {
    echo "=========================================="
    echo "5. VÉRIFICATION DE TOUS LES PODS"
    echo "=========================================="

    # Compter les pods par état
    TOTAL=$(microk8s kubectl get pods --all-namespaces --no-headers | wc -l)
    RUNNING=$(microk8s kubectl get pods --all-namespaces --no-headers | grep "Running" | wc -l)
    PENDING=$(microk8s kubectl get pods --all-namespaces --no-headers | grep "Pending" | wc -l)
    FAILED=$(microk8s kubectl get pods --all-namespaces --no-headers | grep "Failed\|Error\|CrashLoopBackOff" | wc -l)

    log_info "Total de pods : $TOTAL"
    log_info "  - Running : $RUNNING"
    log_info "  - Pending : $PENDING"
    log_info "  - Failed/Error : $FAILED"

    if [ "$FAILED" -gt 0 ]; then
        log_error "$FAILED pod(s) en état Failed/Error"
        microk8s kubectl get pods --all-namespaces | grep "Failed\|Error\|CrashLoopBackOff"
    fi

    if [ "$PENDING" -gt 0 ]; then
        log_warn "$PENDING pod(s) en état Pending"
    fi

    # Vérifier les pods avec trop de redémarrages
    log_info "Vérification des redémarrages de pods..."
    HIGH_RESTARTS=$(microk8s kubectl get pods --all-namespaces --no-headers | awk -v threshold="$POD_RESTART_THRESHOLD" '$4 > threshold' | wc -l)

    if [ "$HIGH_RESTARTS" -gt 0 ]; then
        log_warn "$HIGH_RESTARTS pod(s) ont redémarré plus de $POD_RESTART_THRESHOLD fois"
        microk8s kubectl get pods --all-namespaces --no-headers | awk -v threshold="$POD_RESTART_THRESHOLD" '$4 > threshold {print $0}'
    else
        log_success "Aucun pod avec redémarrages excessifs"
    fi

    echo ""
}

# Vérifier les services
check_services() {
    echo "=========================================="
    echo "6. VÉRIFICATION DES SERVICES"
    echo "=========================================="

    SERVICE_COUNT=$(microk8s kubectl get services --all-namespaces --no-headers | wc -l)
    log_info "Nombre de services : $SERVICE_COUNT"

    # Services sans endpoints
    log_info "Vérification des endpoints..."
    SERVICES_WITHOUT_ENDPOINTS=0

    for namespace in $(microk8s kubectl get namespaces -o jsonpath='{.items[*].metadata.name}'); do
        while IFS= read -r service; do
            if [ -n "$service" ]; then
                ENDPOINTS=$(microk8s kubectl get endpoints "$service" -n "$namespace" -o jsonpath='{.subsets[*].addresses[*].ip}' 2>/dev/null)
                if [ -z "$ENDPOINTS" ]; then
                    log_warn "Service sans endpoint : $namespace/$service"
                    ((SERVICES_WITHOUT_ENDPOINTS++))
                fi
            fi
        done < <(microk8s kubectl get services -n "$namespace" -o jsonpath='{.items[*].metadata.name}')
    done

    if [ "$SERVICES_WITHOUT_ENDPOINTS" -eq 0 ]; then
        log_success "Tous les services ont des endpoints"
    fi

    echo ""
}

# Vérifier les volumes persistants
check_persistent_volumes() {
    echo "=========================================="
    echo "7. VÉRIFICATION DES VOLUMES PERSISTANTS"
    echo "=========================================="

    PV_COUNT=$(microk8s kubectl get pv --no-headers 2>/dev/null | wc -l)
    PVC_COUNT=$(microk8s kubectl get pvc --all-namespaces --no-headers 2>/dev/null | wc -l)

    log_info "PersistentVolumes : $PV_COUNT"
    log_info "PersistentVolumeClaims : $PVC_COUNT"

    # Vérifier les PVC en état Pending
    PENDING_PVC=$(microk8s kubectl get pvc --all-namespaces --no-headers 2>/dev/null | grep "Pending" | wc -l)

    if [ "$PENDING_PVC" -gt 0 ]; then
        log_warn "$PENDING_PVC PVC en état Pending"
        microk8s kubectl get pvc --all-namespaces | grep "Pending"
    else
        log_success "Tous les PVC sont Bound"
    fi

    echo ""
}

# Vérifier les certificats
check_certificates() {
    echo "=========================================="
    echo "8. VÉRIFICATION DES CERTIFICATS"
    echo "=========================================="

    # Vérifier les certificats MicroK8s
    CERT_DIR="/var/snap/microk8s/current/certs"

    if [ -d "$CERT_DIR" ]; then
        log_info "Vérification des certificats dans $CERT_DIR"

        # Vérifier la date d'expiration du certificat API server
        if [ -f "$CERT_DIR/server.crt" ]; then
            EXPIRY_DATE=$(openssl x509 -enddate -noout -in "$CERT_DIR/server.crt" 2>/dev/null | cut -d= -f2)
            EXPIRY_EPOCH=$(date -d "$EXPIRY_DATE" +%s 2>/dev/null || date -j -f "%b %d %H:%M:%S %Y %Z" "$EXPIRY_DATE" +%s 2>/dev/null)
            CURRENT_EPOCH=$(date +%s)
            DAYS_UNTIL_EXPIRY=$(( ($EXPIRY_EPOCH - $CURRENT_EPOCH) / 86400 ))

            log_info "Certificat API server expire dans : $DAYS_UNTIL_EXPIRY jours"

            if [ "$DAYS_UNTIL_EXPIRY" -lt 30 ]; then
                log_error "Certificat API server expire bientôt !"
            elif [ "$DAYS_UNTIL_EXPIRY" -lt 60 ]; then
                log_warn "Certificat API server expire dans moins de 2 mois"
            else
                log_success "Certificat API server valide"
            fi
        fi
    else
        log_warn "Répertoire des certificats non trouvé"
    fi

    echo ""
}

# Vérifier les addons
check_addons() {
    echo "=========================================="
    echo "9. VÉRIFICATION DES ADDONS"
    echo "=========================================="

    log_info "Addons actifs :"
    microk8s status | grep "enabled" | awk '{print "  - " $1}'

    # Vérifier si DNS est activé (recommandé)
    if microk8s status | grep -q "dns.*enabled"; then
        log_success "DNS est activé"
    else
        log_warn "DNS n'est pas activé (recommandé pour la plupart des clusters)"
    fi

    echo ""
}

# Vérifier la connectivité réseau
check_network() {
    echo "=========================================="
    echo "10. VÉRIFICATION DU RÉSEAU"
    echo "=========================================="

    # Test de résolution DNS interne
    log_info "Test de résolution DNS interne..."
    if microk8s kubectl run dns-test --image=busybox:1.28 --restart=Never --rm -i --command -- nslookup kubernetes.default &> /dev/null; then
        log_success "Résolution DNS interne fonctionne"
    else
        log_warn "Problème de résolution DNS interne"
    fi

    # Nettoyer le pod de test
    microk8s kubectl delete pod dns-test --ignore-not-found=true &> /dev/null

    echo ""
}

# Générer le rapport final
generate_report() {
    echo "=========================================="
    echo "RAPPORT DE SANTÉ - RÉSUMÉ"
    echo "=========================================="
    echo "Date : $(date '+%Y-%m-%d %H:%M:%S')"
    echo "Hostname : $(hostname)"
    echo "Version MicroK8s : $(microk8s version --short 2>/dev/null)"
    echo ""
    echo "Résultats :"
    echo "  - Erreurs : $ERRORS"
    echo "  - Avertissements : $WARNINGS"
    echo ""

    if [ "$ERRORS" -eq 0 ] && [ "$WARNINGS" -eq 0 ]; then
        log_success "État du cluster : EXCELLENT ✓"
        return 0
    elif [ "$ERRORS" -eq 0 ]; then
        echo -e "${YELLOW}État du cluster : BON (avec avertissements)${NC}"
        return 0
    else
        log_error "État du cluster : PROBLÈMES DÉTECTÉS"
        return 1
    fi
}

# Envoyer un email si activé
send_email_notification() {
    if [ "$SEND_EMAIL" = true ] && [ "$ERRORS" -gt 0 ]; then
        log_info "Envoi d'une notification par email..."

        SUBJECT="[MicroK8s] Problèmes détectés sur $(hostname)"
        BODY="Des problèmes ont été détectés sur le cluster MicroK8s.

Erreurs : $ERRORS
Avertissements : $WARNINGS

Vérifiez les logs pour plus de détails."

        echo "$BODY" | mail -s "$SUBJECT" "$EMAIL_TO" 2>/dev/null || log_warn "Échec de l'envoi de l'email"
    fi
}

# ============================================
# PROGRAMME PRINCIPAL
# ============================================

main() {
    # Parse des arguments
    while [[ $# -gt 0 ]]; do
        case $1 in
            --verbose|-v)
                VERBOSE=true
                shift
                ;;
            --email)
                SEND_EMAIL=true
                EMAIL_TO="$2"
                shift 2
                ;;
            *)
                echo "Usage: $0 [--verbose] [--email EMAIL]"
                exit 1
                ;;
        esac
    done

    echo "=========================================="
    echo "VÉRIFICATION DE SANTÉ MICROK8S"
    echo "=========================================="
    echo ""

    # Exécuter toutes les vérifications
    check_microk8s_running
    check_nodes
    check_system_resources
    check_system_pods
    check_all_pods
    check_services
    check_persistent_volumes
    check_certificates
    check_addons
    check_network

    # Générer le rapport
    generate_report

    # Envoyer notification si nécessaire
    send_email_notification
}

# Lancement
main "$@"
```

### 3.2 Utilisation du Health Check

**Vérification simple :**
```bash
./health-check.sh
```

**Mode verbeux :**
```bash
./health-check.sh --verbose
```

**Avec notification email :**
```bash
./health-check.sh --email admin@example.com
```

### 3.3 Automatisation des Vérifications

```bash
# Vérification toutes les heures
0 * * * * /chemin/vers/health-check.sh >> /var/log/microk8s-health.log 2>&1

# Vérification détaillée quotidienne avec email
0 8 * * * /chemin/vers/health-check.sh --verbose --email admin@example.com >> /var/log/microk8s-health-daily.log 2>&1
```

---

## 4. Script de Gestion des Certificats

Ce script gère le renouvellement et la vérification des certificats.

### 4.1 Script de Gestion des Certificats

```bash
#!/bin/bash
# Script de gestion des certificats MicroK8s
# Vérifie et renouvelle les certificats si nécessaire

set -e

# ============================================
# CONFIGURATION
# ============================================

# Seuil de renouvellement (en jours)
RENEWAL_THRESHOLD=30

# Répertoire des certificats
CERT_DIR="/var/snap/microk8s/current/certs"

# Backup des certificats
BACKUP_DIR="/var/backups/microk8s/certificates"

# ============================================
# FONCTIONS
# ============================================

GREEN='\033[0;32m'
YELLOW='\033[1;33m'
RED='\033[0;31m'
NC='\033[0m'

log_info() {
    echo -e "${GREEN}[INFO]${NC} $(date '+%Y-%m-%d %H:%M:%S') - $1"
}

log_warn() {
    echo -e "${YELLOW}[WARN]${NC} $(date '+%Y-%m-%d %H:%M:%S') - $1"
}

log_error() {
    echo -e "${RED}[ERROR]${NC} $(date '+%Y-%m-%d %H:%M:%S') - $1"
}

log_success() {
    echo -e "${GREEN}[✓]${NC} $(date '+%Y-%m-%d %H:%M:%S') - $1"
}

# Vérifier l'expiration d'un certificat
check_certificate_expiry() {
    local cert_file=$1
    local cert_name=$2

    if [ ! -f "$cert_file" ]; then
        log_warn "Certificat non trouvé : $cert_file"
        return 1
    fi

    # Extraire la date d'expiration
    EXPIRY_DATE=$(openssl x509 -enddate -noout -in "$cert_file" 2>/dev/null | cut -d= -f2)

    if [ -z "$EXPIRY_DATE" ]; then
        log_error "Impossible de lire la date d'expiration de $cert_name"
        return 1
    fi

    # Convertir en epoch
    EXPIRY_EPOCH=$(date -d "$EXPIRY_DATE" +%s 2>/dev/null || date -j -f "%b %d %H:%M:%S %Y %Z" "$EXPIRY_DATE" +%s 2>/dev/null)
    CURRENT_EPOCH=$(date +%s)
    DAYS_UNTIL_EXPIRY=$(( ($EXPIRY_EPOCH - $CURRENT_EPOCH) / 86400 ))

    log_info "$cert_name expire dans : $DAYS_UNTIL_EXPIRY jours"

    if [ "$DAYS_UNTIL_EXPIRY" -lt 0 ]; then
        log_error "$cert_name a EXPIRÉ !"
        return 2
    elif [ "$DAYS_UNTIL_EXPIRY" -lt "$RENEWAL_THRESHOLD" ]; then
        log_warn "$cert_name expire bientôt (renouvellement recommandé)"
        return 3
    else
        log_success "$cert_name est valide"
        return 0
    fi
}

# Créer une sauvegarde des certificats
backup_certificates() {
    log_info "Sauvegarde des certificats..."

    BACKUP_PATH="${BACKUP_DIR}/$(date +%Y%m%d-%H%M%S)"
    mkdir -p "$BACKUP_PATH"

    if [ -d "$CERT_DIR" ]; then
        cp -r "$CERT_DIR"/* "$BACKUP_PATH/"
        log_success "Certificats sauvegardés dans : $BACKUP_PATH"
    else
        log_error "Répertoire des certificats non trouvé"
        return 1
    fi
}

# Renouveler les certificats MicroK8s
renew_certificates() {
    log_info "Renouvellement des certificats MicroK8s..."

    # Sauvegarder avant renouvellement
    backup_certificates

    # MicroK8s gère automatiquement le renouvellement
    # Il suffit de redémarrer les services
    log_info "Redémarrage des services MicroK8s..."
    microk8s stop
    sleep 5
    microk8s start

    # Attendre que le cluster soit prêt
    log_info "Attente du redémarrage..."
    if microk8s status --wait-ready --timeout 120; then
        log_success "Services redémarrés avec succès"
    else
        log_error "Échec du redémarrage des services"
        return 1
    fi

    log_success "Certificats renouvelés"
}

# Vérifier tous les certificats
check_all_certificates() {
    log_info "=========================================="
    log_info "VÉRIFICATION DE TOUS LES CERTIFICATS"
    log_info "=========================================="

    NEEDS_RENEWAL=false

    # Liste des certificats à vérifier
    declare -A CERTIFICATES=(
        ["$CERT_DIR/server.crt"]="API Server"
        ["$CERT_DIR/front-proxy-client.crt"]="Front Proxy Client"
        ["$CERT_DIR/ca.crt"]="Certificate Authority"
    )

    for cert_file in "${!CERTIFICATES[@]}"; do
        cert_name="${CERTIFICATES[$cert_file]}"
        check_certificate_expiry "$cert_file" "$cert_name"
        RESULT=$?

        if [ "$RESULT" -eq 2 ] || [ "$RESULT" -eq 3 ]; then
            NEEDS_RENEWAL=true
        fi

        echo ""
    done

    if [ "$NEEDS_RENEWAL" = true ]; then
        log_warn "Certains certificats nécessitent un renouvellement"
        return 1
    else
        log_success "Tous les certificats sont valides"
        return 0
    fi
}

# Générer un rapport
generate_report() {
    log_info "=========================================="
    log_info "RAPPORT SUR LES CERTIFICATS"
    log_info "=========================================="
    log_info "Date : $(date '+%Y-%m-%d %H:%M:%S')"
    log_info "Seuil de renouvellement : $RENEWAL_THRESHOLD jours"
    log_info "=========================================="
}

# ============================================
# PROGRAMME PRINCIPAL
# ============================================

main() {
    echo "=========================================="
    echo "GESTION DES CERTIFICATS MICROK8S"
    echo "=========================================="
    echo ""

    # Parse des arguments
    AUTO_RENEW=false

    while [[ $# -gt 0 ]]; do
        case $1 in
            --auto-renew)
                AUTO_RENEW=true
                shift
                ;;
            --threshold)
                RENEWAL_THRESHOLD=$2
                shift 2
                ;;
            *)
                echo "Usage: $0 [--auto-renew] [--threshold JOURS]"
                exit 1
                ;;
        esac
    done

    # Vérifier tous les certificats
    check_all_certificates
    NEEDS_RENEWAL=$?

    # Renouveler automatiquement si demandé
    if [ "$AUTO_RENEW" = true ] && [ "$NEEDS_RENEWAL" -eq 1 ]; then
        echo ""
        log_info "Renouvellement automatique activé"
        renew_certificates

        # Vérifier à nouveau après renouvellement
        echo ""
        check_all_certificates
    elif [ "$NEEDS_RENEWAL" -eq 1 ]; then
        echo ""
        log_warn "Renouvellement nécessaire. Exécutez avec --auto-renew pour renouveler automatiquement"
    fi

    echo ""
    generate_report
}

# Lancement
main "$@"
```

### 4.2 Utilisation du Script de Certificats

**Vérification seulement :**
```bash
./certificate-manager.sh
```

**Avec renouvellement automatique :**
```bash
./certificate-manager.sh --auto-renew
```

**Personnaliser le seuil :**
```bash
./certificate-manager.sh --threshold 60
```

### 4.3 Automatisation

```bash
# Vérification hebdomadaire
0 0 * * 0 /chemin/vers/certificate-manager.sh >> /var/log/microk8s-certs.log 2>&1

# Avec renouvellement automatique mensuel
0 0 1 * * /chemin/vers/certificate-manager.sh --auto-renew >> /var/log/microk8s-certs.log 2>&1
```

---

## 5. Script de Sauvegarde Complète

Ce script effectue une sauvegarde complète de la configuration du cluster.

### 5.1 Script de Backup Complet

```bash
#!/bin/bash
# Script de sauvegarde complète MicroK8s
# Sauvegarde la configuration, les ressources et les données

set -e

# ============================================
# CONFIGURATION
# ============================================

# Répertoire de sauvegarde
BACKUP_BASE_DIR="/var/backups/microk8s"
BACKUP_DATE=$(date +%Y%m%d-%H%M%S)
BACKUP_DIR="${BACKUP_BASE_DIR}/${BACKUP_DATE}"

# Rétention des sauvegardes (en jours)
RETENTION_DAYS=30

# Compression
COMPRESS=true
COMPRESSION_LEVEL=6  # 1-9

# Inclure les volumes persistants (attention à la taille)
BACKUP_VOLUMES=false

# ============================================
# FONCTIONS
# ============================================

GREEN='\033[0;32m'
YELLOW='\033[1;33m'
BLUE='\033[0;34m'
NC='\033[0m'

log_info() {
    echo -e "${GREEN}[INFO]${NC} $(date '+%Y-%m-%d %H:%M:%S') - $1"
}

log_warn() {
    echo -e "${YELLOW}[WARN]${NC} $(date '+%Y-%m-%d %H:%M:%S') - $1"
}

log_success() {
    echo -e "${BLUE}[✓]${NC} $(date '+%Y-%m-%d %H:%M:%S') - $1"
}

# Créer la structure de sauvegarde
create_backup_structure() {
    log_info "Création de la structure de sauvegarde..."

    mkdir -p "$BACKUP_DIR"/{config,resources,addons,certificates,volumes}

    log_success "Structure créée : $BACKUP_DIR"
}

# Sauvegarder la configuration kubeconfig
backup_kubeconfig() {
    log_info "Sauvegarde de kubeconfig..."

    microk8s config > "${BACKUP_DIR}/config/kubeconfig.yaml"

    log_success "Kubeconfig sauvegardé"
}

# Sauvegarder l'état des addons
backup_addons() {
    log_info "Sauvegarde de l'état des addons..."

    microk8s status > "${BACKUP_DIR}/addons/status.txt"
    microk8s status | grep "enabled" | awk '{print $1}' > "${BACKUP_DIR}/addons/enabled-addons.txt"

    log_success "État des addons sauvegardé"
}

# Sauvegarder toutes les ressources Kubernetes
backup_kubernetes_resources() {
    log_info "Sauvegarde des ressources Kubernetes..."

    # Liste de tous les namespaces
    for namespace in $(microk8s kubectl get namespaces -o jsonpath='{.items[*].metadata.name}'); do
        log_info "  Sauvegarde du namespace: $namespace"

        mkdir -p "${BACKUP_DIR}/resources/${namespace}"

        # Ressources à sauvegarder
        RESOURCES=(
            deployments
            services
            configmaps
            secrets
            pods
            statefulsets
            daemonsets
            jobs
            cronjobs
            ingresses
            persistentvolumeclaims
            serviceaccounts
            roles
            rolebindings
        )

        for resource in "${RESOURCES[@]}"; do
            microk8s kubectl get "$resource" -n "$namespace" -o yaml > \
                "${BACKUP_DIR}/resources/${namespace}/${resource}.yaml" 2>/dev/null || true
        done
    done

    # Sauvegarder les ressources cluster-wide
    log_info "  Sauvegarde des ressources cluster-wide..."

    CLUSTER_RESOURCES=(
        nodes
        persistentvolumes
        storageclasses
        clusterroles
        clusterrolebindings
        namespaces
    )

    for resource in "${CLUSTER_RESOURCES[@]}"; do
        microk8s kubectl get "$resource" -o yaml > \
            "${BACKUP_DIR}/resources/${resource}.yaml" 2>/dev/null || true
    done

    log_success "Ressources Kubernetes sauvegardées"
}

# Sauvegarder les certificats
backup_certificates() {
    log_info "Sauvegarde des certificats..."

    CERT_DIR="/var/snap/microk8s/current/certs"

    if [ -d "$CERT_DIR" ]; then
        cp -r "$CERT_DIR"/* "${BACKUP_DIR}/certificates/"
        log_success "Certificats sauvegardés"
    else
        log_warn "Répertoire des certificats non trouvé"
    fi
}

# Sauvegarder les volumes persistants
backup_persistent_volumes() {
    if [ "$BACKUP_VOLUMES" = false ]; then
        log_info "Sauvegarde des volumes désactivée"
        return
    fi

    log_info "Sauvegarde des volumes persistants..."
    log_warn "Cette opération peut prendre du temps selon la taille des données"

    # Répertoire de stockage MicroK8s
    STORAGE_DIR="/var/snap/microk8s/common/default-storage"

    if [ -d "$STORAGE_DIR" ]; then
        cp -r "$STORAGE_DIR" "${BACKUP_DIR}/volumes/" || log_warn "Erreur lors de la copie des volumes"
        log_success "Volumes persistants sauvegardés"
    else
        log_warn "Répertoire de stockage non trouvé"
    fi
}

# Créer un fichier d'informations
create_info_file() {
    log_info "Création du fichier d'informations..."

    cat > "${BACKUP_DIR}/backup-info.txt" << EOF
========================================
INFORMATIONS DE SAUVEGARDE MICROK8S
========================================

Date de sauvegarde : $(date '+%Y-%m-%d %H:%M:%S')
Hostname : $(hostname)
Version MicroK8s : $(microk8s version --short 2>/dev/null)
Version Kubernetes : $(microk8s kubectl version --short 2>/dev/null | grep Server)

Contenu de la sauvegarde :
- Configuration kubeconfig
- État des addons
- Ressources Kubernetes (tous les namespaces)
- Certificats
$([ "$BACKUP_VOLUMES" = true ] && echo "- Volumes persistants")

Nombre de namespaces : $(microk8s kubectl get namespaces --no-headers | wc -l)
Nombre de pods : $(microk8s kubectl get pods --all-namespaces --no-headers | wc -l)
Nombre de services : $(microk8s kubectl get services --all-namespaces --no-headers | wc -l)

========================================
EOF

    log_success "Fichier d'informations créé"
}

# Compresser la sauvegarde
compress_backup() {
    if [ "$COMPRESS" = false ]; then
        log_info "Compression désactivée"
        return
    fi

    log_info "Compression de la sauvegarde..."

    cd "$BACKUP_BASE_DIR"
    tar czf "${BACKUP_DATE}.tar.gz" -C "$BACKUP_BASE_DIR" "$BACKUP_DATE" --remove-files

    BACKUP_SIZE=$(du -h "${BACKUP_DATE}.tar.gz" | cut -f1)
    log_success "Sauvegarde compressée : ${BACKUP_SIZE}"

    # Mettre à jour le chemin de sauvegarde
    BACKUP_DIR="${BACKUP_BASE_DIR}/${BACKUP_DATE}.tar.gz"
}

# Nettoyer les anciennes sauvegardes
cleanup_old_backups() {
    log_info "Nettoyage des sauvegardes anciennes (>${RETENTION_DAYS} jours)..."

    BACKUPS_BEFORE=$(find "$BACKUP_BASE_DIR" -maxdepth 1 \( -type f -name "*.tar.gz" -o -type d \) ! -path "$BACKUP_BASE_DIR" | wc -l)

    # Supprimer les anciennes sauvegardes compressées
    find "$BACKUP_BASE_DIR" -maxdepth 1 -type f -name "*.tar.gz" -mtime +${RETENTION_DAYS} -delete

    # Supprimer les anciens répertoires non compressés
    find "$BACKUP_BASE_DIR" -maxdepth 1 -type d ! -path "$BACKUP_BASE_DIR" -mtime +${RETENTION_DAYS} -exec rm -rf {} +

    BACKUPS_AFTER=$(find "$BACKUP_BASE_DIR" -maxdepth 1 \( -type f -name "*.tar.gz" -o -type d \) ! -path "$BACKUP_BASE_DIR" | wc -l)
    BACKUPS_REMOVED=$((BACKUPS_BEFORE - BACKUPS_AFTER))

    log_info "Sauvegardes supprimées : $BACKUPS_REMOVED"
    log_info "Sauvegardes conservées : $BACKUPS_AFTER"
}

# Générer le rapport
generate_report() {
    log_info "=========================================="
    log_info "RAPPORT DE SAUVEGARDE"
    log_info "=========================================="
    log_info "Date : $(date '+%Y-%m-%d %H:%M:%S')"
    log_info "Emplacement : $BACKUP_DIR"

    if [ -f "$BACKUP_DIR" ]; then
        BACKUP_SIZE=$(du -h "$BACKUP_DIR" | cut -f1)
        log_info "Taille : $BACKUP_SIZE"
    fi

    log_info "=========================================="
    log_success "Sauvegarde complétée avec succès"
}

# ============================================
# PROGRAMME PRINCIPAL
# ============================================

main() {
    echo "=========================================="
    echo "SAUVEGARDE COMPLÈTE MICROK8S"
    echo "=========================================="
    echo ""

    # Parse des arguments
    while [[ $# -gt 0 ]]; do
        case $1 in
            --no-compress)
                COMPRESS=false
                shift
                ;;
            --with-volumes)
                BACKUP_VOLUMES=true
                shift
                ;;
            --retention)
                RETENTION_DAYS=$2
                shift 2
                ;;
            *)
                echo "Usage: $0 [--no-compress] [--with-volumes] [--retention JOURS]"
                exit 1
                ;;
        esac
    done

    # Vérifier que MicroK8s fonctionne
    if ! microk8s status --wait-ready --timeout 30 > /dev/null 2>&1; then
        log_error "MicroK8s n'est pas en cours d'exécution"
        exit 1
    fi

    # Créer la structure
    create_backup_structure

    # Sauvegarder tous les composants
    backup_kubeconfig
    backup_addons
    backup_kubernetes_resources
    backup_certificates
    backup_persistent_volumes

    # Créer le fichier d'informations
    create_info_file

    # Compresser si demandé
    compress_backup

    echo ""

    # Nettoyer les anciennes sauvegardes
    cleanup_old_backups

    echo ""

    # Générer le rapport
    generate_report
}

# Gestion des erreurs
trap 'log_error "Erreur lors de la sauvegarde"' ERR

# Lancement
main "$@"
```

### 5.2 Utilisation du Script de Sauvegarde

**Sauvegarde standard :**
```bash
./backup-microk8s.sh
```

**Avec volumes persistants :**
```bash
./backup-microk8s.sh --with-volumes
```

**Sans compression :**
```bash
./backup-microk8s.sh --no-compress
```

**Personnaliser la rétention :**
```bash
./backup-microk8s.sh --retention 60
```

### 5.3 Restauration d'une Sauvegarde

Pour restaurer une sauvegarde :

```bash
#!/bin/bash
# Script de restauration simple

BACKUP_FILE="$1"

if [ -z "$BACKUP_FILE" ]; then
    echo "Usage: $0 /chemin/vers/backup.tar.gz"
    exit 1
fi

# Extraire la sauvegarde
tar xzf "$BACKUP_FILE" -C /tmp

BACKUP_DIR="/tmp/$(basename "$BACKUP_FILE" .tar.gz)"

# Restaurer les ressources
echo "Restauration des ressources..."
find "$BACKUP_DIR/resources" -name "*.yaml" -exec microk8s kubectl apply -f {} \;

echo "Restauration terminée"
```

### 5.4 Automatisation des Sauvegardes

```bash
# Sauvegarde quotidienne à 1h du matin
0 1 * * * /chemin/vers/backup-microk8s.sh >> /var/log/microk8s-backup.log 2>&1

# Sauvegarde hebdomadaire complète avec volumes
0 2 * * 0 /chemin/vers/backup-microk8s.sh --with-volumes >> /var/log/microk8s-backup-weekly.log 2>&1
```

---

## 6. Bonnes Pratiques de Maintenance

### 6.1 Fréquences Recommandées

**Quotidien :**
- Nettoyage des ressources inutilisées
- Vérification de santé basique
- Monitoring de l'espace disque

**Hebdomadaire :**
- Vérification complète de santé
- Vérification des certificats
- Sauvegarde complète

**Mensuel :**
- Mise à jour de MicroK8s
- Rotation des logs
- Audit de sécurité

### 6.2 Surveillance Continue

Utilisez une combinaison de scripts pour une surveillance continue :

```bash
# Fichier : /etc/cron.d/microk8s-maintenance

# Nettoyage quotidien à 2h
0 2 * * * user /usr/local/bin/cleanup-daily.sh

# Health check toutes les heures
0 * * * * user /usr/local/bin/health-check.sh

# Vérification des certificats hebdomadaire
0 3 * * 0 user /usr/local/bin/certificate-manager.sh

# Sauvegarde quotidienne à 1h
0 1 * * * user /usr/local/bin/backup-microk8s.sh

# Mise à jour mensuelle (premier dimanche)
0 4 1-7 * 0 user /usr/local/bin/update-microk8s.sh
```

### 6.3 Monitoring des Scripts

Créez un script pour surveiller l'exécution des tâches de maintenance :

```bash
#!/bin/bash
# Vérifier que les tâches de maintenance s'exécutent

LOG_DIR="/var/log"

echo "Dernière exécution des scripts de maintenance :"
echo ""
echo "Nettoyage : $(stat -c %y $LOG_DIR/microk8s-maintenance/cleanup-* 2>/dev/null | tail -1 | cut -d'.' -f1)"
echo "Health Check : $(stat -c %y $LOG_DIR/microk8s-health.log 2>/dev/null | cut -d'.' -f1)"
echo "Backup : $(stat -c %y $LOG_DIR/microk8s-backup.log 2>/dev/null | cut -d'.' -f1)"
```

---

## Conclusion

Les scripts de maintenance présentés dans cette section automatisent l'essentiel des tâches opérationnelles de MicroK8s. En les utilisant régulièrement, vous garantissez la stabilité, la performance et la sécurité de votre cluster.

**Points clés à retenir :**

1. **Automatisez tout** : utilisez cron pour exécuter les scripts régulièrement
2. **Testez d'abord** : validez toujours vos scripts en environnement de test
3. **Surveillez les logs** : vérifiez régulièrement que les scripts s'exécutent correctement
4. **Sauvegardez régulièrement** : une sauvegarde quotidienne peut vous sauver la mise
5. **Soyez proactif** : les vérifications de santé permettent de détecter les problèmes avant qu'ils ne deviennent critiques

La section suivante présente les scripts de monitoring pour une visibilité complète sur votre cluster.

⏭️ [Scripts de monitoring](/annexes/annexe-a-scripts-et-automatisation/scripts-de-monitoring.md)
