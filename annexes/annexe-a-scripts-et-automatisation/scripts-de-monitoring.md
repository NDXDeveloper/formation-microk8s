🔝 Retour au [Sommaire](/SOMMAIRE.md)

# Annexe A : Scripts de Monitoring MicroK8s

## Introduction

Le monitoring (surveillance) de votre cluster MicroK8s est essentiel pour anticiper les problèmes, optimiser les performances et maintenir une visibilité constante sur l'état de votre infrastructure. Les scripts de monitoring automatisent la collecte de métriques, l'analyse des tendances et la génération de rapports.

Cette section présente une collection complète de scripts de monitoring pour MicroK8s, allant de la surveillance simple en temps réel jusqu'à la génération de rapports détaillés et l'envoi d'alertes automatiques.

**Ce que vous apprendrez :**
- Comment surveiller l'utilisation des ressources en temps réel
- Comment collecter et analyser des métriques
- Comment générer des rapports automatiques
- Comment configurer des alertes simples
- Comment créer des dashboards textuels
- Comment exporter les données vers des outils externes

---

## 1. Script de Monitoring en Temps Réel

Ce script affiche en temps réel l'état de votre cluster dans votre terminal.

### 1.1 Script de Monitoring Live

```bash
#!/bin/bash
# Script de monitoring en temps réel MicroK8s
# Affiche les métriques du cluster dans le terminal

# ============================================
# CONFIGURATION
# ============================================

# Intervalle de rafraîchissement (en secondes)
REFRESH_INTERVAL=5

# Afficher les détails des pods
SHOW_POD_DETAILS=true

# Nombre de pods à afficher par namespace
MAX_PODS_PER_NS=10

# ============================================
# FONCTIONS
# ============================================

# Couleurs
RED='\033[0;31m'
GREEN='\033[0;32m'
YELLOW='\033[1;33m'
BLUE='\033[0;34m'
CYAN='\033[0;36m'
MAGENTA='\033[0;35m'
BOLD='\033[1m'
NC='\033[0m'

# Effacer l'écran
clear_screen() {
    clear
    tput cup 0 0
}

# Afficher l'en-tête
show_header() {
    local current_time=$(date '+%Y-%m-%d %H:%M:%S')
    local version=$(microk8s version --short 2>/dev/null || echo "N/A")

    echo -e "${BOLD}${BLUE}╔════════════════════════════════════════════════════════════════════════╗${NC}"
    echo -e "${BOLD}${BLUE}║${NC}                  ${BOLD}MicroK8s Live Monitor${NC}                           ${BOLD}${BLUE}║${NC}"
    echo -e "${BOLD}${BLUE}╚════════════════════════════════════════════════════════════════════════╝${NC}"
    echo -e "${CYAN}Hostname:${NC} $(hostname)  ${CYAN}Version:${NC} $version  ${CYAN}Time:${NC} $current_time"
    echo ""
}

# Afficher l'état du cluster
show_cluster_status() {
    echo -e "${BOLD}${MAGENTA}━━━ CLUSTER STATUS ━━━${NC}"

    # Vérifier si MicroK8s fonctionne
    if microk8s status --wait-ready --timeout 5 > /dev/null 2>&1; then
        echo -e "Status: ${GREEN}●${NC} Running"
    else
        echo -e "Status: ${RED}●${NC} Not Ready"
        return
    fi

    # Nombre de nœuds
    local node_count=$(microk8s kubectl get nodes --no-headers 2>/dev/null | wc -l)
    local ready_nodes=$(microk8s kubectl get nodes --no-headers 2>/dev/null | grep -c "Ready" || echo "0")
    echo -e "Nodes: ${GREEN}$ready_nodes${NC}/${node_count} Ready"

    echo ""
}

# Afficher les métriques de ressources système
show_system_resources() {
    echo -e "${BOLD}${MAGENTA}━━━ SYSTEM RESOURCES ━━━${NC}"

    # CPU
    local cpu_usage=$(top -bn1 | grep "Cpu(s)" | awk '{print $2}' | cut -d'%' -f1 | cut -d'.' -f1)
    local cpu_bar=$(generate_progress_bar "$cpu_usage" 50)

    if [ "$cpu_usage" -gt 80 ]; then
        echo -e "CPU:    ${RED}${cpu_bar}${NC} ${RED}${cpu_usage}%${NC}"
    elif [ "$cpu_usage" -gt 60 ]; then
        echo -e "CPU:    ${YELLOW}${cpu_bar}${NC} ${YELLOW}${cpu_usage}%${NC}"
    else
        echo -e "CPU:    ${GREEN}${cpu_bar}${NC} ${GREEN}${cpu_usage}%${NC}"
    fi

    # Mémoire
    local mem_total=$(free -m | awk 'NR==2{print $2}')
    local mem_used=$(free -m | awk 'NR==2{print $3}')
    local mem_percent=$((mem_used * 100 / mem_total))
    local mem_bar=$(generate_progress_bar "$mem_percent" 50)

    if [ "$mem_percent" -gt 85 ]; then
        echo -e "Memory: ${RED}${mem_bar}${NC} ${RED}${mem_used}MB/${mem_total}MB (${mem_percent}%)${NC}"
    elif [ "$mem_percent" -gt 70 ]; then
        echo -e "Memory: ${YELLOW}${mem_bar}${NC} ${YELLOW}${mem_used}MB/${mem_total}MB (${mem_percent}%)${NC}"
    else
        echo -e "Memory: ${GREEN}${mem_bar}${NC} ${GREEN}${mem_used}MB/${mem_total}MB (${mem_percent}%)${NC}"
    fi

    # Disque
    local disk_usage=$(df -h / | tail -1 | awk '{print $5}' | sed 's/%//')
    local disk_used=$(df -h / | tail -1 | awk '{print $3}')
    local disk_total=$(df -h / | tail -1 | awk '{print $2}')
    local disk_bar=$(generate_progress_bar "$disk_usage" 50)

    if [ "$disk_usage" -gt 90 ]; then
        echo -e "Disk:   ${RED}${disk_bar}${NC} ${RED}${disk_used}/${disk_total} (${disk_usage}%)${NC}"
    elif [ "$disk_usage" -gt 75 ]; then
        echo -e "Disk:   ${YELLOW}${disk_bar}${NC} ${YELLOW}${disk_used}/${disk_total} (${disk_usage}%)${NC}"
    else
        echo -e "Disk:   ${GREEN}${disk_bar}${NC} ${GREEN}${disk_used}/${disk_total} (${disk_usage}%)${NC}"
    fi

    echo ""
}

# Générer une barre de progression
generate_progress_bar() {
    local percent=$1
    local width=${2:-20}
    local filled=$((percent * width / 100))
    local empty=$((width - filled))

    printf "["
    printf "%${filled}s" | tr ' ' '█'
    printf "%${empty}s" | tr ' ' '░'
    printf "]"
}

# Afficher le résumé des pods
show_pod_summary() {
    echo -e "${BOLD}${MAGENTA}━━━ PODS SUMMARY ━━━${NC}"

    local total=$(microk8s kubectl get pods --all-namespaces --no-headers 2>/dev/null | wc -l)
    local running=$(microk8s kubectl get pods --all-namespaces --no-headers 2>/dev/null | grep -c "Running" || echo "0")
    local pending=$(microk8s kubectl get pods --all-namespaces --no-headers 2>/dev/null | grep -c "Pending" || echo "0")
    local failed=$(microk8s kubectl get pods --all-namespaces --no-headers 2>/dev/null | grep -cE "Failed|Error|CrashLoopBackOff" || echo "0")
    local completed=$(microk8s kubectl get pods --all-namespaces --no-headers 2>/dev/null | grep -c "Completed" || echo "0")

    echo -e "Total:     ${BOLD}${total}${NC}"
    echo -e "Running:   ${GREEN}${running}${NC}"
    echo -e "Pending:   ${YELLOW}${pending}${NC}"
    echo -e "Failed:    ${RED}${failed}${NC}"
    echo -e "Completed: ${CYAN}${completed}${NC}"

    echo ""
}

# Afficher les détails des pods par namespace
show_pod_details() {
    if [ "$SHOW_POD_DETAILS" = false ]; then
        return
    fi

    echo -e "${BOLD}${MAGENTA}━━━ PODS DETAILS ━━━${NC}"

    # Obtenir les namespaces avec des pods
    local namespaces=$(microk8s kubectl get pods --all-namespaces --no-headers 2>/dev/null | awk '{print $1}' | sort -u)

    for ns in $namespaces; do
        local pod_count=$(microk8s kubectl get pods -n "$ns" --no-headers 2>/dev/null | wc -l)

        echo -e "\n${BOLD}${CYAN}Namespace: $ns${NC} (${pod_count} pods)"
        echo "────────────────────────────────────────────────────────────────"

        # Afficher les pods (limité)
        microk8s kubectl get pods -n "$ns" --no-headers 2>/dev/null | head -n "$MAX_PODS_PER_NS" | while read line; do
            local pod_name=$(echo "$line" | awk '{print $1}')
            local ready=$(echo "$line" | awk '{print $2}')
            local status=$(echo "$line" | awk '{print $3}')
            local restarts=$(echo "$line" | awk '{print $4}')
            local age=$(echo "$line" | awk '{print $5}')

            # Colorer selon le statut
            case "$status" in
                Running)
                    if [ "$restarts" -gt 5 ]; then
                        echo -e "  ${YELLOW}●${NC} $pod_name ${YELLOW}[$status]${NC} ${YELLOW}Restarts: $restarts${NC}"
                    else
                        echo -e "  ${GREEN}●${NC} $pod_name ${GREEN}[$status]${NC} Ready: $ready"
                    fi
                    ;;
                Pending)
                    echo -e "  ${YELLOW}●${NC} $pod_name ${YELLOW}[$status]${NC}"
                    ;;
                Failed|Error|CrashLoopBackOff)
                    echo -e "  ${RED}●${NC} $pod_name ${RED}[$status]${NC} ${RED}Restarts: $restarts${NC}"
                    ;;
                Completed)
                    echo -e "  ${CYAN}●${NC} $pod_name ${CYAN}[$status]${NC}"
                    ;;
                *)
                    echo -e "  ○ $pod_name [$status]"
                    ;;
            esac
        done

        # Indiquer s'il y a plus de pods
        if [ "$pod_count" -gt "$MAX_PODS_PER_NS" ]; then
            echo -e "  ${YELLOW}... et $((pod_count - MAX_PODS_PER_NS)) autres pods${NC}"
        fi
    done

    echo ""
}

# Afficher les services
show_services_summary() {
    echo -e "${BOLD}${MAGENTA}━━━ SERVICES ━━━${NC}"

    local service_count=$(microk8s kubectl get services --all-namespaces --no-headers 2>/dev/null | wc -l)
    local loadbalancer=$(microk8s kubectl get services --all-namespaces --no-headers 2>/dev/null | grep -c "LoadBalancer" || echo "0")
    local nodeport=$(microk8s kubectl get services --all-namespaces --no-headers 2>/dev/null | grep -c "NodePort" || echo "0")
    local clusterip=$(microk8s kubectl get services --all-namespaces --no-headers 2>/dev/null | grep -c "ClusterIP" || echo "0")

    echo -e "Total:        ${BOLD}${service_count}${NC}"
    echo -e "LoadBalancer: ${loadbalancer}"
    echo -e "NodePort:     ${nodeport}"
    echo -e "ClusterIP:    ${clusterip}"

    echo ""
}

# Afficher les événements récents
show_recent_events() {
    echo -e "${BOLD}${MAGENTA}━━━ RECENT EVENTS (Last 5) ━━━${NC}"

    microk8s kubectl get events --all-namespaces --sort-by='.lastTimestamp' 2>/dev/null | tail -n 6 | tail -n 5 | while read line; do
        local event_type=$(echo "$line" | awk '{print $3}')

        case "$event_type" in
            Warning)
                echo -e "${YELLOW}⚠${NC} $(echo "$line" | awk '{$3=""; print $0}')"
                ;;
            Error)
                echo -e "${RED}✗${NC} $(echo "$line" | awk '{$3=""; print $0}')"
                ;;
            Normal)
                echo -e "${GREEN}✓${NC} $(echo "$line" | awk '{$3=""; print $0}')"
                ;;
            *)
                echo -e "  $line"
                ;;
        esac
    done

    echo ""
}

# Afficher le footer
show_footer() {
    echo -e "${BOLD}${BLUE}════════════════════════════════════════════════════════════════════════${NC}"
    echo -e "${CYAN}Refresh:${NC} ${REFRESH_INTERVAL}s  ${CYAN}Press${NC} Ctrl+C ${CYAN}to exit${NC}"
}

# Boucle principale
main_loop() {
    while true; do
        clear_screen
        show_header
        show_cluster_status
        show_system_resources
        show_pod_summary
        show_pod_details
        show_services_summary
        show_recent_events
        show_footer

        sleep "$REFRESH_INTERVAL"
    done
}

# ============================================
# PROGRAMME PRINCIPAL
# ============================================

# Parse des arguments
while [[ $# -gt 0 ]]; do
    case $1 in
        --interval|-i)
            REFRESH_INTERVAL=$2
            shift 2
            ;;
        --no-pods)
            SHOW_POD_DETAILS=false
            shift
            ;;
        --max-pods)
            MAX_PODS_PER_NS=$2
            shift 2
            ;;
        --help|-h)
            echo "Usage: $0 [OPTIONS]"
            echo ""
            echo "Options:"
            echo "  -i, --interval SECONDS   Refresh interval (default: 5)"
            echo "  --no-pods                Don't show pod details"
            echo "  --max-pods NUMBER        Max pods per namespace (default: 10)"
            echo "  -h, --help               Show this help"
            exit 0
            ;;
        *)
            echo "Unknown option: $1"
            echo "Use --help for usage information"
            exit 1
            ;;
    esac
done

# Vérifier que MicroK8s est installé
if ! command -v microk8s &> /dev/null; then
    echo "Error: MicroK8s is not installed"
    exit 1
fi

# Lancement
main_loop
```

### 1.2 Utilisation du Monitoring Live

**Mode standard :**
```bash
./live-monitor.sh
```

**Avec intervalle personnalisé :**
```bash
./live-monitor.sh --interval 10
```

**Sans détails des pods :**
```bash
./live-monitor.sh --no-pods
```

---

## 2. Script de Collecte de Métriques

Ce script collecte des métriques détaillées et les enregistre dans des fichiers CSV pour analyse ultérieure.

### 2.1 Script de Collecte

```bash
#!/bin/bash
# Script de collecte de métriques MicroK8s
# Collecte et enregistre les métriques dans des fichiers CSV

set -e

# ============================================
# CONFIGURATION
# ============================================

# Répertoire de sortie des métriques
METRICS_DIR="/var/log/microk8s-metrics"
TIMESTAMP=$(date +%Y%m%d-%H%M%S)

# Fichiers de sortie
SYSTEM_METRICS="${METRICS_DIR}/system-metrics.csv"
POD_METRICS="${METRICS_DIR}/pod-metrics.csv"
NODE_METRICS="${METRICS_DIR}/node-metrics.csv"
RESOURCE_METRICS="${METRICS_DIR}/resource-metrics.csv"

# Initialiser les fichiers CSV si nécessaire
INIT_FILES=false

# ============================================
# FONCTIONS
# ============================================

GREEN='\033[0;32m'
YELLOW='\033[1;33m'
NC='\033[0m'

log_info() {
    echo -e "${GREEN}[INFO]${NC} $(date '+%Y-%m-%d %H:%M:%S') - $1"
}

log_warn() {
    echo -e "${YELLOW}[WARN]${NC} $(date '+%Y-%m-%d %H:%M:%S') - $1"
}

# Créer la structure de répertoires
setup_directories() {
    mkdir -p "$METRICS_DIR"
    log_info "Répertoire de métriques : $METRICS_DIR"
}

# Initialiser les fichiers CSV avec en-têtes
initialize_csv_files() {
    if [ "$INIT_FILES" = true ] || [ ! -f "$SYSTEM_METRICS" ]; then
        echo "timestamp,hostname,cpu_percent,memory_used_mb,memory_total_mb,memory_percent,disk_used_gb,disk_total_gb,disk_percent,load_1min,load_5min,load_15min" > "$SYSTEM_METRICS"
        log_info "Fichier système initialisé : $SYSTEM_METRICS"
    fi

    if [ "$INIT_FILES" = true ] || [ ! -f "$POD_METRICS" ]; then
        echo "timestamp,namespace,pod_name,status,restarts,age,cpu_request,cpu_limit,memory_request,memory_limit" > "$POD_METRICS"
        log_info "Fichier pods initialisé : $POD_METRICS"
    fi

    if [ "$INIT_FILES" = true ] || [ ! -f "$NODE_METRICS" ]; then
        echo "timestamp,node_name,status,cpu_capacity,memory_capacity,pods_capacity,cpu_allocatable,memory_allocatable,pods_allocatable" > "$NODE_METRICS"
        log_info "Fichier nœuds initialisé : $NODE_METRICS"
    fi

    if [ "$INIT_FILES" = true ] || [ ! -f "$RESOURCE_METRICS" ]; then
        echo "timestamp,total_pods,running_pods,pending_pods,failed_pods,total_services,total_deployments,total_namespaces,total_pvcs" > "$RESOURCE_METRICS"
        log_info "Fichier ressources initialisé : $RESOURCE_METRICS"
    fi
}

# Collecter les métriques système
collect_system_metrics() {
    log_info "Collecte des métriques système..."

    local timestamp=$(date '+%Y-%m-%d %H:%M:%S')
    local hostname=$(hostname)

    # CPU
    local cpu_percent=$(top -bn1 | grep "Cpu(s)" | awk '{print $2}' | cut -d'%' -f1)

    # Mémoire
    local mem_used=$(free -m | awk 'NR==2{print $3}')
    local mem_total=$(free -m | awk 'NR==2{print $2}')
    local mem_percent=$((mem_used * 100 / mem_total))

    # Disque
    local disk_used=$(df -BG / | tail -1 | awk '{print $3}' | sed 's/G//')
    local disk_total=$(df -BG / | tail -1 | awk '{print $2}' | sed 's/G//')
    local disk_percent=$(df / | tail -1 | awk '{print $5}' | sed 's/%//')

    # Load average
    local load_avg=($(uptime | awk -F'load average:' '{print $2}' | sed 's/,//g'))
    local load_1min=${load_avg[0]}
    local load_5min=${load_avg[1]}
    local load_15min=${load_avg[2]}

    # Écrire dans le CSV
    echo "$timestamp,$hostname,$cpu_percent,$mem_used,$mem_total,$mem_percent,$disk_used,$disk_total,$disk_percent,$load_1min,$load_5min,$load_15min" >> "$SYSTEM_METRICS"

    log_info "Métriques système collectées"
}

# Collecter les métriques des pods
collect_pod_metrics() {
    log_info "Collecte des métriques des pods..."

    local timestamp=$(date '+%Y-%m-%d %H:%M:%S')
    local count=0

    # Obtenir les informations des pods
    microk8s kubectl get pods --all-namespaces -o json 2>/dev/null | jq -r '.items[] |
        "\(.metadata.namespace),\(.metadata.name),\(.status.phase),\(.status.containerStatuses[0].restartCount // 0),\(.metadata.creationTimestamp)"' |
    while IFS=',' read -r namespace pod_name status restarts created_at; do
        # Calculer l'âge
        local age=$(calculate_age "$created_at")

        # Obtenir les ressources du pod
        local resources=$(microk8s kubectl get pod "$pod_name" -n "$namespace" -o json 2>/dev/null | jq -r '
            .spec.containers[0].resources |
            "\(.requests.cpu // "N/A"),\(.limits.cpu // "N/A"),\(.requests.memory // "N/A"),\(.limits.memory // "N/A")"')

        local cpu_req=$(echo "$resources" | cut -d',' -f1)
        local cpu_limit=$(echo "$resources" | cut -d',' -f2)
        local mem_req=$(echo "$resources" | cut -d',' -f3)
        local mem_limit=$(echo "$resources" | cut -d',' -f4)

        # Écrire dans le CSV
        echo "$timestamp,$namespace,$pod_name,$status,$restarts,$age,$cpu_req,$cpu_limit,$mem_req,$mem_limit" >> "$POD_METRICS"

        ((count++))
    done

    log_info "Métriques de $count pods collectées"
}

# Calculer l'âge d'une ressource
calculate_age() {
    local created=$1
    local created_epoch=$(date -d "$created" +%s 2>/dev/null || echo "0")
    local current_epoch=$(date +%s)
    local age_seconds=$((current_epoch - created_epoch))

    if [ "$age_seconds" -lt 3600 ]; then
        echo "$((age_seconds / 60))m"
    elif [ "$age_seconds" -lt 86400 ]; then
        echo "$((age_seconds / 3600))h"
    else
        echo "$((age_seconds / 86400))d"
    fi
}

# Collecter les métriques des nœuds
collect_node_metrics() {
    log_info "Collecte des métriques des nœuds..."

    local timestamp=$(date '+%Y-%m-%d %H:%M:%S')

    microk8s kubectl get nodes -o json 2>/dev/null | jq -r '.items[] |
        "\(.metadata.name),\(.status.conditions[] | select(.type=="Ready") | .status),
        \(.status.capacity.cpu),\(.status.capacity.memory),\(.status.capacity.pods),
        \(.status.allocatable.cpu),\(.status.allocatable.memory),\(.status.allocatable.pods)"' |
    while IFS=',' read -r node_name status cpu_cap mem_cap pods_cap cpu_alloc mem_alloc pods_alloc; do
        # Nettoyer les valeurs
        status=$(echo "$status" | tr -d ' ')

        echo "$timestamp,$node_name,$status,$cpu_cap,$mem_cap,$pods_cap,$cpu_alloc,$mem_alloc,$pods_alloc" >> "$NODE_METRICS"
    done

    log_info "Métriques des nœuds collectées"
}

# Collecter les métriques des ressources
collect_resource_metrics() {
    log_info "Collecte des métriques des ressources..."

    local timestamp=$(date '+%Y-%m-%d %H:%M:%S')

    # Compter les ressources
    local total_pods=$(microk8s kubectl get pods --all-namespaces --no-headers 2>/dev/null | wc -l)
    local running_pods=$(microk8s kubectl get pods --all-namespaces --no-headers 2>/dev/null | grep -c "Running" || echo "0")
    local pending_pods=$(microk8s kubectl get pods --all-namespaces --no-headers 2>/dev/null | grep -c "Pending" || echo "0")
    local failed_pods=$(microk8s kubectl get pods --all-namespaces --no-headers 2>/dev/null | grep -cE "Failed|Error" || echo "0")

    local total_services=$(microk8s kubectl get services --all-namespaces --no-headers 2>/dev/null | wc -l)
    local total_deployments=$(microk8s kubectl get deployments --all-namespaces --no-headers 2>/dev/null | wc -l)
    local total_namespaces=$(microk8s kubectl get namespaces --no-headers 2>/dev/null | wc -l)
    local total_pvcs=$(microk8s kubectl get pvc --all-namespaces --no-headers 2>/dev/null | wc -l)

    # Écrire dans le CSV
    echo "$timestamp,$total_pods,$running_pods,$pending_pods,$failed_pods,$total_services,$total_deployments,$total_namespaces,$total_pvcs" >> "$RESOURCE_METRICS"

    log_info "Métriques des ressources collectées"
}

# Générer un résumé
generate_summary() {
    log_info "=========================================="
    log_info "RÉSUMÉ DE LA COLLECTE"
    log_info "=========================================="
    log_info "Timestamp : $TIMESTAMP"
    log_info "Fichiers générés :"
    log_info "  - Système : $SYSTEM_METRICS"
    log_info "  - Pods : $POD_METRICS"
    log_info "  - Nœuds : $NODE_METRICS"
    log_info "  - Ressources : $RESOURCE_METRICS"
    log_info "=========================================="
}

# ============================================
# PROGRAMME PRINCIPAL
# ============================================

main() {
    # Parse des arguments
    while [[ $# -gt 0 ]]; do
        case $1 in
            --init)
                INIT_FILES=true
                shift
                ;;
            --dir)
                METRICS_DIR=$2
                shift 2
                ;;
            *)
                echo "Usage: $0 [--init] [--dir DIRECTORY]"
                exit 1
                ;;
        esac
    done

    log_info "Démarrage de la collecte de métriques"

    # Vérifier que MicroK8s fonctionne
    if ! microk8s status --wait-ready --timeout 30 > /dev/null 2>&1; then
        log_warn "MicroK8s n'est pas prêt"
        exit 1
    fi

    # Configuration
    setup_directories
    initialize_csv_files

    # Collecte des métriques
    collect_system_metrics
    collect_pod_metrics
    collect_node_metrics
    collect_resource_metrics

    # Résumé
    generate_summary

    log_info "Collecte terminée avec succès"
}

# Lancement
main "$@"
```

### 2.2 Utilisation du Script de Collecte

**Collecte unique :**
```bash
./collect-metrics.sh
```

**Initialiser les fichiers CSV :**
```bash
./collect-metrics.sh --init
```

**Répertoire personnalisé :**
```bash
./collect-metrics.sh --dir /custom/path
```

### 2.3 Collecte Automatique

Pour collecter des métriques toutes les 5 minutes :

```bash
# Éditer le crontab
crontab -e

# Ajouter cette ligne
*/5 * * * * /chemin/vers/collect-metrics.sh >> /var/log/metrics-collection.log 2>&1
```

---

## 3. Script de Génération de Rapports

Ce script génère des rapports détaillés à partir des métriques collectées.

### 3.1 Script de Rapport

```bash
#!/bin/bash
# Script de génération de rapports MicroK8s
# Analyse les métriques et génère des rapports HTML

set -e

# ============================================
# CONFIGURATION
# ============================================

# Répertoire des métriques
METRICS_DIR="/var/log/microk8s-metrics"

# Répertoire de sortie des rapports
REPORTS_DIR="/var/reports/microk8s"

# Période d'analyse (en heures)
ANALYSIS_PERIOD=24

# Format de sortie (html, text, json)
OUTPUT_FORMAT="html"

# ============================================
# FONCTIONS
# ============================================

GREEN='\033[0;32m'
BLUE='\033[0;34m'
NC='\033[0m'

log_info() {
    echo -e "${GREEN}[INFO]${NC} $(date '+%Y-%m-%d %H:%M:%S') - $1"
}

log_success() {
    echo -e "${BLUE}[✓]${NC} $(date '+%Y-%m-%d %H:%M:%S') - $1"
}

# Créer la structure de rapports
setup_reports_directory() {
    mkdir -p "$REPORTS_DIR"
    log_info "Répertoire de rapports : $REPORTS_DIR"
}

# Analyser les métriques système
analyze_system_metrics() {
    local metrics_file="${METRICS_DIR}/system-metrics.csv"

    if [ ! -f "$metrics_file" ]; then
        log_info "Aucune métrique système trouvée"
        return
    fi

    log_info "Analyse des métriques système..."

    # Calculer les moyennes sur la période
    local avg_cpu=$(tail -n "$((ANALYSIS_PERIOD * 12))" "$metrics_file" | awk -F',' 'NR>1 {sum+=$3; count++} END {if(count>0) printf "%.2f", sum/count}')
    local avg_mem=$(tail -n "$((ANALYSIS_PERIOD * 12))" "$metrics_file" | awk -F',' 'NR>1 {sum+=$6; count++} END {if(count>0) printf "%.2f", sum/count}')
    local avg_disk=$(tail -n "$((ANALYSIS_PERIOD * 12))" "$metrics_file" | awk -F',' 'NR>1 {sum+=$9; count++} END {if(count>0) printf "%.2f", sum/count}')

    # Pic d'utilisation
    local max_cpu=$(tail -n "$((ANALYSIS_PERIOD * 12))" "$metrics_file" | awk -F',' 'NR>1 {if($3>max) max=$3} END {printf "%.2f", max}')
    local max_mem=$(tail -n "$((ANALYSIS_PERIOD * 12))" "$metrics_file" | awk -F',' 'NR>1 {if($6>max) max=$6} END {printf "%.2f", max}')

    # Stocker les résultats
    SYSTEM_AVG_CPU="$avg_cpu"
    SYSTEM_AVG_MEM="$avg_mem"
    SYSTEM_AVG_DISK="$avg_disk"
    SYSTEM_MAX_CPU="$max_cpu"
    SYSTEM_MAX_MEM="$max_mem"

    log_info "CPU moyen: ${avg_cpu}% (max: ${max_cpu}%)"
    log_info "Mémoire moyenne: ${avg_mem}% (max: ${max_mem}%)"
    log_info "Disque moyen: ${avg_disk}%"
}

# Analyser les métriques des pods
analyze_pod_metrics() {
    local metrics_file="${METRICS_DIR}/pod-metrics.csv"

    if [ ! -f "$metrics_file" ]; then
        log_info "Aucune métrique de pods trouvée"
        return
    fi

    log_info "Analyse des métriques des pods..."

    # Pods avec le plus de redémarrages
    POD_TOP_RESTARTS=$(tail -n 1000 "$metrics_file" | awk -F',' 'NR>1 {print $2","$3","$5}' | sort -t',' -k3 -rn | head -n 5)

    # Comptage des états
    local latest_timestamp=$(tail -n 1 "$metrics_file" | cut -d',' -f1)
    POD_RUNNING=$(grep "$latest_timestamp" "$metrics_file" | grep -c "Running" || echo "0")
    POD_PENDING=$(grep "$latest_timestamp" "$metrics_file" | grep -c "Pending" || echo "0")
    POD_FAILED=$(grep "$latest_timestamp" "$metrics_file" | grep -c "Failed" || echo "0")

    log_info "Pods Running: $POD_RUNNING, Pending: $POD_PENDING, Failed: $POD_FAILED"
}

# Analyser les métriques des ressources
analyze_resource_metrics() {
    local metrics_file="${METRICS_DIR}/resource-metrics.csv"

    if [ ! -f "$metrics_file" ]; then
        log_info "Aucune métrique de ressources trouvée"
        return
    fi

    log_info "Analyse des métriques des ressources..."

    # Dernières valeurs
    local latest=$(tail -n 1 "$metrics_file")

    RESOURCE_TOTAL_PODS=$(echo "$latest" | cut -d',' -f2)
    RESOURCE_RUNNING_PODS=$(echo "$latest" | cut -d',' -f3)
    RESOURCE_TOTAL_SERVICES=$(echo "$latest" | cut -d',' -f6)
    RESOURCE_TOTAL_DEPLOYMENTS=$(echo "$latest" | cut -d',' -f7)
    RESOURCE_TOTAL_NAMESPACES=$(echo "$latest" | cut -d',' -f8)

    log_info "Total Pods: $RESOURCE_TOTAL_PODS, Services: $RESOURCE_TOTAL_SERVICES"
}

# Générer le rapport HTML
generate_html_report() {
    local report_file="${REPORTS_DIR}/report-$(date +%Y%m%d-%H%M%S).html"

    log_info "Génération du rapport HTML..."

    cat > "$report_file" << EOF
<!DOCTYPE html>
<html lang="fr">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Rapport MicroK8s - $(date '+%Y-%m-%d %H:%M:%S')</title>
    <style>
        * {
            margin: 0;
            padding: 0;
            box-sizing: border-box;
        }

        body {
            font-family: 'Segoe UI', Tahoma, Geneva, Verdana, sans-serif;
            background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
            padding: 20px;
            line-height: 1.6;
        }

        .container {
            max-width: 1200px;
            margin: 0 auto;
            background: white;
            border-radius: 10px;
            box-shadow: 0 10px 40px rgba(0,0,0,0.2);
            overflow: hidden;
        }

        .header {
            background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
            color: white;
            padding: 30px;
            text-align: center;
        }

        .header h1 {
            font-size: 2.5em;
            margin-bottom: 10px;
        }

        .header p {
            font-size: 1.1em;
            opacity: 0.9;
        }

        .content {
            padding: 30px;
        }

        .section {
            margin-bottom: 30px;
            padding: 20px;
            background: #f8f9fa;
            border-radius: 8px;
            border-left: 4px solid #667eea;
        }

        .section h2 {
            color: #667eea;
            margin-bottom: 15px;
            font-size: 1.8em;
        }

        .metric-grid {
            display: grid;
            grid-template-columns: repeat(auto-fit, minmax(200px, 1fr));
            gap: 20px;
            margin-top: 20px;
        }

        .metric-card {
            background: white;
            padding: 20px;
            border-radius: 8px;
            box-shadow: 0 2px 10px rgba(0,0,0,0.1);
            text-align: center;
        }

        .metric-card h3 {
            color: #6c757d;
            font-size: 0.9em;
            text-transform: uppercase;
            margin-bottom: 10px;
        }

        .metric-value {
            font-size: 2.5em;
            font-weight: bold;
            color: #667eea;
        }

        .metric-unit {
            font-size: 0.8em;
            color: #6c757d;
        }

        .metric-detail {
            margin-top: 10px;
            font-size: 0.9em;
            color: #6c757d;
        }

        .progress-bar {
            width: 100%;
            height: 30px;
            background: #e9ecef;
            border-radius: 15px;
            overflow: hidden;
            margin: 10px 0;
        }

        .progress-fill {
            height: 100%;
            background: linear-gradient(90deg, #667eea 0%, #764ba2 100%);
            display: flex;
            align-items: center;
            justify-content: center;
            color: white;
            font-weight: bold;
            transition: width 0.3s ease;
        }

        .status-good { color: #28a745; }
        .status-warning { color: #ffc107; }
        .status-critical { color: #dc3545; }

        .table {
            width: 100%;
            border-collapse: collapse;
            margin-top: 15px;
        }

        .table th {
            background: #667eea;
            color: white;
            padding: 12px;
            text-align: left;
        }

        .table td {
            padding: 12px;
            border-bottom: 1px solid #dee2e6;
        }

        .table tr:hover {
            background: #f8f9fa;
        }

        .footer {
            background: #343a40;
            color: white;
            text-align: center;
            padding: 20px;
            font-size: 0.9em;
        }

        @media print {
            body {
                background: white;
            }

            .container {
                box-shadow: none;
            }
        }
    </style>
</head>
<body>
    <div class="container">
        <div class="header">
            <h1>📊 Rapport MicroK8s</h1>
            <p>Période d'analyse : ${ANALYSIS_PERIOD} heures</p>
            <p>Généré le : $(date '+%Y-%m-%d %H:%M:%S')</p>
        </div>

        <div class="content">
            <!-- Section Système -->
            <div class="section">
                <h2>🖥️ Métriques Système</h2>
                <div class="metric-grid">
                    <div class="metric-card">
                        <h3>CPU Moyen</h3>
                        <div class="metric-value">${SYSTEM_AVG_CPU:-N/A}<span class="metric-unit">%</span></div>
                        <div class="metric-detail">Pic: ${SYSTEM_MAX_CPU:-N/A}%</div>
                        <div class="progress-bar">
                            <div class="progress-fill" style="width: ${SYSTEM_AVG_CPU:-0}%">${SYSTEM_AVG_CPU:-0}%</div>
                        </div>
                    </div>

                    <div class="metric-card">
                        <h3>Mémoire Moyenne</h3>
                        <div class="metric-value">${SYSTEM_AVG_MEM:-N/A}<span class="metric-unit">%</span></div>
                        <div class="metric-detail">Pic: ${SYSTEM_MAX_MEM:-N/A}%</div>
                        <div class="progress-bar">
                            <div class="progress-fill" style="width: ${SYSTEM_AVG_MEM:-0}%">${SYSTEM_AVG_MEM:-0}%</div>
                        </div>
                    </div>

                    <div class="metric-card">
                        <h3>Disque Moyen</h3>
                        <div class="metric-value">${SYSTEM_AVG_DISK:-N/A}<span class="metric-unit">%</span></div>
                        <div class="progress-bar">
                            <div class="progress-fill" style="width: ${SYSTEM_AVG_DISK:-0}%">${SYSTEM_AVG_DISK:-0}%</div>
                        </div>
                    </div>
                </div>
            </div>

            <!-- Section Ressources -->
            <div class="section">
                <h2>📦 Ressources Kubernetes</h2>
                <div class="metric-grid">
                    <div class="metric-card">
                        <h3>Pods Totaux</h3>
                        <div class="metric-value">${RESOURCE_TOTAL_PODS:-N/A}</div>
                        <div class="metric-detail">Running: ${RESOURCE_RUNNING_PODS:-N/A}</div>
                    </div>

                    <div class="metric-card">
                        <h3>Services</h3>
                        <div class="metric-value">${RESOURCE_TOTAL_SERVICES:-N/A}</div>
                    </div>

                    <div class="metric-card">
                        <h3>Deployments</h3>
                        <div class="metric-value">${RESOURCE_TOTAL_DEPLOYMENTS:-N/A}</div>
                    </div>

                    <div class="metric-card">
                        <h3>Namespaces</h3>
                        <div class="metric-value">${RESOURCE_TOTAL_NAMESPACES:-N/A}</div>
                    </div>
                </div>
            </div>

            <!-- Section État du Cluster -->
            <div class="section">
                <h2>🎯 État du Cluster</h2>
                <table class="table">
                    <thead>
                        <tr>
                            <th>Composant</th>
                            <th>État</th>
                            <th>Détails</th>
                        </tr>
                    </thead>
                    <tbody>
                        <tr>
                            <td>MicroK8s</td>
                            <td><span class="status-good">✓ Running</span></td>
                            <td>Version: $(microk8s version --short 2>/dev/null || echo "N/A")</td>
                        </tr>
                        <tr>
                            <td>Pods Running</td>
                            <td><span class="status-good">✓ ${POD_RUNNING:-0}</span></td>
                            <td>Pending: ${POD_PENDING:-0}, Failed: ${POD_FAILED:-0}</td>
                        </tr>
                        <tr>
                            <td>Nœuds</td>
                            <td><span class="status-good">✓ Ready</span></td>
                            <td>$(microk8s kubectl get nodes --no-headers 2>/dev/null | wc -l) nœud(s)</td>
                        </tr>
                    </tbody>
                </table>
            </div>

            <!-- Section Recommandations -->
            <div class="section">
                <h2>💡 Recommandations</h2>
                <ul>
EOF

    # Ajouter des recommandations basées sur les métriques
    if [ "${SYSTEM_AVG_CPU%%.*}" -gt 80 ]; then
        echo "                    <li class=\"status-warning\">⚠️ Utilisation CPU élevée - Envisagez d'augmenter les ressources CPU</li>" >> "$report_file"
    fi

    if [ "${SYSTEM_AVG_MEM%%.*}" -gt 85 ]; then
        echo "                    <li class=\"status-critical\">⚠️ Utilisation mémoire critique - Augmentez la RAM ou réduisez la charge</li>" >> "$report_file"
    fi

    if [ "${SYSTEM_AVG_DISK%%.*}" -gt 90 ]; then
        echo "                    <li class=\"status-critical\">⚠️ Espace disque critique - Nettoyage nécessaire</li>" >> "$report_file"
    fi

    if [ "$POD_FAILED" -gt 0 ]; then
        echo "                    <li class=\"status-warning\">⚠️ Des pods sont en échec - Vérifiez les logs</li>" >> "$report_file"
    fi

    # Recommandation par défaut
    if [ "${SYSTEM_AVG_CPU%%.*}" -lt 80 ] && [ "${SYSTEM_AVG_MEM%%.*}" -lt 85 ]; then
        echo "                    <li class=\"status-good\">✓ Le cluster fonctionne normalement</li>" >> "$report_file"
    fi

    cat >> "$report_file" << EOF
                </ul>
            </div>
        </div>

        <div class="footer">
            <p>Rapport généré automatiquement par le script de monitoring MicroK8s</p>
            <p>Hostname: $(hostname) | Période: ${ANALYSIS_PERIOD}h</p>
        </div>
    </div>
</body>
</html>
EOF

    log_success "Rapport HTML généré : $report_file"
    echo "$report_file"
}

# Générer le rapport texte
generate_text_report() {
    local report_file="${REPORTS_DIR}/report-$(date +%Y%m%d-%H%M%S).txt"

    log_info "Génération du rapport texte..."

    cat > "$report_file" << EOF
========================================
RAPPORT MICROK8S
========================================
Date: $(date '+%Y-%m-%d %H:%M:%S')
Hostname: $(hostname)
Période d'analyse: ${ANALYSIS_PERIOD} heures

----------------------------------------
MÉTRIQUES SYSTÈME
----------------------------------------
CPU Moyen:        ${SYSTEM_AVG_CPU:-N/A}% (Pic: ${SYSTEM_MAX_CPU:-N/A}%)
Mémoire Moyenne:  ${SYSTEM_AVG_MEM:-N/A}% (Pic: ${SYSTEM_MAX_MEM:-N/A}%)
Disque Moyen:     ${SYSTEM_AVG_DISK:-N/A}%

----------------------------------------
RESSOURCES KUBERNETES
----------------------------------------
Pods Totaux:      ${RESOURCE_TOTAL_PODS:-N/A}
Pods Running:     ${RESOURCE_RUNNING_PODS:-N/A}
Services:         ${RESOURCE_TOTAL_SERVICES:-N/A}
Deployments:      ${RESOURCE_TOTAL_DEPLOYMENTS:-N/A}
Namespaces:       ${RESOURCE_TOTAL_NAMESPACES:-N/A}

----------------------------------------
ÉTAT DES PODS
----------------------------------------
Running:          ${POD_RUNNING:-0}
Pending:          ${POD_PENDING:-0}
Failed:           ${POD_FAILED:-0}

========================================
FIN DU RAPPORT
========================================
EOF

    log_success "Rapport texte généré : $report_file"
    echo "$report_file"
}

# ============================================
# PROGRAMME PRINCIPAL
# ============================================

main() {
    # Parse des arguments
    while [[ $# -gt 0 ]]; do
        case $1 in
            --period)
                ANALYSIS_PERIOD=$2
                shift 2
                ;;
            --format)
                OUTPUT_FORMAT=$2
                shift 2
                ;;
            --metrics-dir)
                METRICS_DIR=$2
                shift 2
                ;;
            *)
                echo "Usage: $0 [--period HOURS] [--format html|text] [--metrics-dir DIR]"
                exit 1
                ;;
        esac
    done

    log_info "=========================================="
    log_info "GÉNÉRATION DE RAPPORT MICROK8S"
    log_info "=========================================="

    # Configuration
    setup_reports_directory

    # Analyse des métriques
    analyze_system_metrics
    analyze_pod_metrics
    analyze_resource_metrics

    # Génération du rapport
    case "$OUTPUT_FORMAT" in
        html)
            REPORT_FILE=$(generate_html_report)
            ;;
        text)
            REPORT_FILE=$(generate_text_report)
            ;;
        *)
            log_info "Format inconnu: $OUTPUT_FORMAT (utilisation de html)"
            REPORT_FILE=$(generate_html_report)
            ;;
    esac

    log_info "=========================================="
    log_success "Rapport généré avec succès"
    log_info "Fichier: $REPORT_FILE"
    log_info "=========================================="
}

# Lancement
main "$@"
```

### 3.2 Utilisation du Générateur de Rapports

**Rapport HTML (par défaut) :**
```bash
./generate-report.sh
```

**Rapport texte :**
```bash
./generate-report.sh --format text
```

**Période personnalisée :**
```bash
./generate-report.sh --period 168  # 7 jours
```

### 3.3 Automatisation des Rapports

```bash
# Rapport quotidien à 23h
0 23 * * * /chemin/vers/generate-report.sh --format html >> /var/log/report-generation.log 2>&1

# Rapport hebdomadaire détaillé
0 0 * * 0 /chemin/vers/generate-report.sh --period 168 --format html >> /var/log/weekly-report.log 2>&1
```

---

## 4. Script d'Alertes Simples

Ce script surveille les métriques et envoie des alertes en cas de dépassement de seuils.

### 4.1 Script d'Alerting

```bash
#!/bin/bash
# Script d'alertes MicroK8s
# Surveille les métriques et envoie des alertes

set -e

# ============================================
# CONFIGURATION
# ============================================

# Seuils d'alerte
CPU_WARNING=70
CPU_CRITICAL=90
MEMORY_WARNING=75
MEMORY_CRITICAL=90
DISK_WARNING=80
DISK_CRITICAL=95

# Configuration email
SEND_EMAIL=false
EMAIL_TO="admin@example.com"
EMAIL_FROM="microk8s@$(hostname)"

# Configuration Slack (optionnel)
SEND_SLACK=false
SLACK_WEBHOOK_URL=""

# Fichier de suivi des alertes
ALERT_STATE_FILE="/var/tmp/microk8s-alerts-state"

# ============================================
# FONCTIONS
# ============================================

RED='\033[0;31m'
YELLOW='\033[1;33m'
GREEN='\033[0;32m'
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

# Initialiser le fichier d'état
initialize_alert_state() {
    if [ ! -f "$ALERT_STATE_FILE" ]; then
        cat > "$ALERT_STATE_FILE" << EOF
cpu_warning=false
cpu_critical=false
memory_warning=false
memory_critical=false
disk_warning=false
disk_critical=false
cluster_down=false
EOF
    fi

    # Charger l'état précédent
    source "$ALERT_STATE_FILE"
}

# Sauvegarder l'état
save_alert_state() {
    cat > "$ALERT_STATE_FILE" << EOF
cpu_warning=$cpu_warning
cpu_critical=$cpu_critical
memory_warning=$memory_warning
memory_critical=$memory_critical
disk_warning=$disk_warning
disk_critical=$disk_critical
cluster_down=$cluster_down
EOF
}

# Envoyer une alerte par email
send_email_alert() {
    if [ "$SEND_EMAIL" = false ]; then
        return
    fi

    local subject=$1
    local message=$2

    echo "$message" | mail -s "$subject" -r "$EMAIL_FROM" "$EMAIL_TO" 2>/dev/null || \
        log_warn "Échec de l'envoi de l'email"
}

# Envoyer une alerte Slack
send_slack_alert() {
    if [ "$SEND_SLACK" = false ] || [ -z "$SLACK_WEBHOOK_URL" ]; then
        return
    fi

    local severity=$1
    local message=$2

    local color="warning"
    if [ "$severity" = "critical" ]; then
        color="danger"
    fi

    local payload=$(cat <<EOF
{
    "attachments": [
        {
            "color": "$color",
            "title": "🚨 Alerte MicroK8s - $(hostname)",
            "text": "$message",
            "footer": "MicroK8s Monitoring",
            "ts": $(date +%s)
        }
    ]
}
EOF
)

    curl -X POST -H 'Content-type: application/json' \
        --data "$payload" "$SLACK_WEBHOOK_URL" 2>/dev/null || \
        log_warn "Échec de l'envoi Slack"
}

# Vérifier l'utilisation CPU
check_cpu() {
    local cpu_usage=$(top -bn1 | grep "Cpu(s)" | awk '{print $2}' | cut -d'%' -f1 | cut -d'.' -f1)

    log_info "CPU: ${cpu_usage}%"

    if [ "$cpu_usage" -ge "$CPU_CRITICAL" ]; then
        if [ "$cpu_critical" = false ]; then
            log_error "CPU CRITIQUE: ${cpu_usage}%"
            send_email_alert "🔴 CPU CRITIQUE sur $(hostname)" \
                "L'utilisation CPU a atteint un niveau critique: ${cpu_usage}%"
            send_slack_alert "critical" "CPU critique: ${cpu_usage}% (seuil: ${CPU_CRITICAL}%)"
            cpu_critical=true
        fi
    elif [ "$cpu_usage" -ge "$CPU_WARNING" ]; then
        if [ "$cpu_warning" = false ]; then
            log_warn "CPU ÉLEVÉ: ${cpu_usage}%"
            send_email_alert "⚠️ CPU élevé sur $(hostname)" \
                "L'utilisation CPU dépasse le seuil d'avertissement: ${cpu_usage}%"
            send_slack_alert "warning" "CPU élevé: ${cpu_usage}% (seuil: ${CPU_WARNING}%)"
            cpu_warning=true
        fi
    else
        # Réinitialiser les alertes si retour à la normale
        if [ "$cpu_warning" = true ] || [ "$cpu_critical" = true ]; then
            log_info "CPU revenu à la normale: ${cpu_usage}%"
            send_email_alert "✅ CPU normal sur $(hostname)" \
                "L'utilisation CPU est revenue à la normale: ${cpu_usage}%"
            send_slack_alert "good" "CPU normal: ${cpu_usage}%"
            cpu_warning=false
            cpu_critical=false
        fi
    fi
}

# Vérifier l'utilisation mémoire
check_memory() {
    local mem_total=$(free -m | awk 'NR==2{print $2}')
    local mem_used=$(free -m | awk 'NR==2{print $3}')
    local mem_percent=$((mem_used * 100 / mem_total))

    log_info "Mémoire: ${mem_percent}%"

    if [ "$mem_percent" -ge "$MEMORY_CRITICAL" ]; then
        if [ "$memory_critical" = false ]; then
            log_error "MÉMOIRE CRITIQUE: ${mem_percent}%"
            send_email_alert "🔴 MÉMOIRE CRITIQUE sur $(hostname)" \
                "L'utilisation mémoire a atteint un niveau critique: ${mem_percent}% (${mem_used}MB/${mem_total}MB)"
            send_slack_alert "critical" "Mémoire critique: ${mem_percent}% (seuil: ${MEMORY_CRITICAL}%)"
            memory_critical=true
        fi
    elif [ "$mem_percent" -ge "$MEMORY_WARNING" ]; then
        if [ "$memory_warning" = false ]; then
            log_warn "MÉMOIRE ÉLEVÉE: ${mem_percent}%"
            send_email_alert "⚠️ Mémoire élevée sur $(hostname)" \
                "L'utilisation mémoire dépasse le seuil d'avertissement: ${mem_percent}%"
            send_slack_alert "warning" "Mémoire élevée: ${mem_percent}% (seuil: ${MEMORY_WARNING}%)"
            memory_warning=true
        fi
    else
        if [ "$memory_warning" = true ] || [ "$memory_critical" = true ]; then
            log_info "Mémoire revenue à la normale: ${mem_percent}%"
            send_email_alert "✅ Mémoire normale sur $(hostname)" \
                "L'utilisation mémoire est revenue à la normale: ${mem_percent}%"
            send_slack_alert "good" "Mémoire normale: ${mem_percent}%"
            memory_warning=false
            memory_critical=false
        fi
    fi
}

# Vérifier l'utilisation disque
check_disk() {
    local disk_usage=$(df -h / | tail -1 | awk '{print $5}' | sed 's/%//')

    log_info "Disque: ${disk_usage}%"

    if [ "$disk_usage" -ge "$DISK_CRITICAL" ]; then
        if [ "$disk_critical" = false ]; then
            log_error "DISQUE CRITIQUE: ${disk_usage}%"
            send_email_alert "🔴 DISQUE CRITIQUE sur $(hostname)" \
                "L'espace disque a atteint un niveau critique: ${disk_usage}%"
            send_slack_alert "critical" "Disque critique: ${disk_usage}% (seuil: ${DISK_CRITICAL}%)"
            disk_critical=true
        fi
    elif [ "$disk_usage" -ge "$DISK_WARNING" ]; then
        if [ "$disk_warning" = false ]; then
            log_warn "DISQUE ÉLEVÉ: ${disk_usage}%"
            send_email_alert "⚠️ Espace disque faible sur $(hostname)" \
                "L'espace disque dépasse le seuil d'avertissement: ${disk_usage}%"
            send_slack_alert "warning" "Disque élevé: ${disk_usage}% (seuil: ${DISK_WARNING}%)"
            disk_warning=true
        fi
    else
        if [ "$disk_warning" = true ] || [ "$disk_critical" = true ]; then
            log_info "Espace disque revenu à la normale: ${disk_usage}%"
            send_email_alert "✅ Disque normal sur $(hostname)" \
                "L'espace disque est revenu à la normale: ${disk_usage}%"
            send_slack_alert "good" "Disque normal: ${disk_usage}%"
            disk_warning=false
            disk_critical=false
        fi
    fi
}

# Vérifier l'état du cluster
check_cluster_status() {
    if ! microk8s status --wait-ready --timeout 10 > /dev/null 2>&1; then
        if [ "$cluster_down" = false ]; then
            log_error "CLUSTER DOWN: MicroK8s n'est pas accessible"
            send_email_alert "🔴 CLUSTER DOWN sur $(hostname)" \
                "Le cluster MicroK8s ne répond pas"
            send_slack_alert "critical" "Cluster MicroK8s DOWN"
            cluster_down=true
        fi
        return 1
    else
        if [ "$cluster_down" = true ]; then
            log_info "Cluster revenu en ligne"
            send_email_alert "✅ Cluster en ligne sur $(hostname)" \
                "Le cluster MicroK8s est de nouveau accessible"
            send_slack_alert "good" "Cluster MicroK8s UP"
            cluster_down=false
        fi
    fi

    # Vérifier les pods Failed
    local failed_pods=$(microk8s kubectl get pods --all-namespaces --no-headers 2>/dev/null | grep -cE "Failed|Error|CrashLoopBackOff" || echo "0")

    if [ "$failed_pods" -gt 0 ]; then
        log_warn "$failed_pods pod(s) en échec"
        # Ne pas spammer - alerte uniquement si le nombre augmente
    fi
}

# ============================================
# PROGRAMME PRINCIPAL
# ============================================

main() {
    log_info "=========================================="
    log_info "VÉRIFICATION DES ALERTES MICROK8S"
    log_info "=========================================="

    # Initialiser l'état des alertes
    initialize_alert_state

    # Vérifications
    check_cpu
    check_memory
    check_disk
    check_cluster_status

    # Sauvegarder le nouvel état
    save_alert_state

    log_info "Vérification terminée"
}

# Lancement
main "$@"
```

### 4.2 Configuration des Alertes

**Activer les emails :**
```bash
# Éditer le script et modifier
SEND_EMAIL=true
EMAIL_TO="votre-email@example.com"

# Installer mailutils si nécessaire
sudo apt install mailutils
```

**Activer Slack :**
```bash
# Créer un webhook Slack : https://api.slack.com/messaging/webhooks
# Éditer le script et modifier
SEND_SLACK=true
SLACK_WEBHOOK_URL="https://hooks.slack.com/services/YOUR/WEBHOOK/URL"
```

### 4.3 Automatisation des Alertes

```bash
# Vérification toutes les 5 minutes
*/5 * * * * /chemin/vers/alert-monitor.sh >> /var/log/microk8s-alerts.log 2>&1
```

---

## 5. Dashboard Textuel Simple

Un script pour créer un dashboard textuel coloré dans le terminal.

### 5.1 Script de Dashboard

```bash
#!/bin/bash
# Dashboard textuel MicroK8s simplifié

# Fonction principale d'affichage
show_dashboard() {
    clear

    # En-tête
    echo -e "\033[1;34m╔═══════════════════════════════════════════════════════════════╗\033[0m"
    echo -e "\033[1;34m║\033[0m              \033[1mMICROK8S DASHBOARD\033[0m - $(date '+%H:%M:%S')           \033[1;34m║\033[0m"
    echo -e "\033[1;34m╚═══════════════════════════════════════════════════════════════╝\033[0m"
    echo ""

    # Status
    if microk8s status --wait-ready --timeout 5 > /dev/null 2>&1; then
        echo -e "Status: \033[1;32m● RUNNING\033[0m"
    else
        echo -e "Status: \033[1;31m● DOWN\033[0m"
        return
    fi

    # Ressources
    echo ""
    echo -e "\033[1;35m▼ RESOURCES\033[0m"
    echo "───────────────────────────────────────────────────────────────"

    local cpu=$(top -bn1 | grep "Cpu(s)" | awk '{print $2}' | cut -d'%' -f1)
    local mem=$(free | grep Mem | awk '{print int($3/$2 * 100)}')
    local disk=$(df / | tail -1 | awk '{print $5}')

    printf "CPU:    [%-20s] %5s%%\n" "$(generate_bar $cpu 20)" "$cpu"
    printf "Memory: [%-20s] %5s%%\n" "$(generate_bar $mem 20)" "$mem"
    printf "Disk:   [%-20s] %5s\n" "$(generate_bar ${disk%%%} 20)" "$disk"

    # Pods
    echo ""
    echo -e "\033[1;35m▼ PODS\033[0m"
    echo "───────────────────────────────────────────────────────────────"

    local running=$(microk8s kubectl get pods --all-namespaces --no-headers 2>/dev/null | grep -c "Running")
    local total=$(microk8s kubectl get pods --all-namespaces --no-headers 2>/dev/null | wc -l)

    echo -e "Running: \033[1;32m$running\033[0m / $total"

    # Services
    local services=$(microk8s kubectl get services --all-namespaces --no-headers 2>/dev/null | wc -l)
    echo -e "Services: $services"

    echo ""
    echo "Press Ctrl+C to exit | Refresh: 5s"
}

# Générer une barre de progression
generate_bar() {
    local percent=$1
    local width=$2
    local filled=$((percent * width / 100))

    printf "%${filled}s" | tr ' ' '█'
    printf "%$((width - filled))s" | tr ' ' '░'
}

# Boucle
while true; do
    show_dashboard
    sleep 5
done
```

### 5.2 Utilisation du Dashboard

```bash
chmod +x simple-dashboard.sh
./simple-dashboard.sh
```

---

## 6. Bonnes Pratiques de Monitoring

### 6.1 Stratégie de Monitoring

**Niveaux de monitoring :**

1. **Temps réel** : pour le diagnostic et le dépannage
2. **Collecte périodique** : pour l'analyse des tendances
3. **Rapports** : pour la visibilité management
4. **Alertes** : pour la réactivité aux problèmes

### 6.2 Métriques Clés à Surveiller

**Infrastructure :**
- Utilisation CPU, mémoire, disque
- Load average
- Latence réseau

**Kubernetes :**
- Nombre de pods par état
- Redémarrages de pods
- Utilisation des ressources par pod
- État des services

**Application :**
- Temps de réponse
- Taux d'erreur
- Throughput

### 6.3 Rétention des Données

```bash
# Rétention recommandée
Métriques temps réel:     1 heure
Métriques 5 min:          7 jours
Métriques horaires:       30 jours
Rapports quotidiens:      90 jours
Rapports hebdomadaires:   1 an
```

### 6.4 Automatisation Complète

Créez un fichier cron centralisé :

```bash
# /etc/cron.d/microk8s-monitoring

# Collecte de métriques toutes les 5 minutes
*/5 * * * * user /usr/local/bin/collect-metrics.sh

# Rapport quotidien à 23h
0 23 * * * user /usr/local/bin/generate-report.sh

# Alertes toutes les 5 minutes
*/5 * * * * user /usr/local/bin/alert-monitor.sh

# Nettoyage mensuel des anciennes données
0 0 1 * * user find /var/log/microk8s-metrics -name "*.csv" -mtime +90 -delete
```

---

## Conclusion

Les scripts de monitoring présentés dans cette section vous donnent une visibilité complète sur votre cluster MicroK8s. En les combinant avec les scripts d'installation et de maintenance des sections précédentes, vous disposez d'une suite complète d'automatisation.

**Points clés à retenir :**

1. **Surveillez en continu** : automatisez la collecte de métriques
2. **Analysez les tendances** : générez des rapports réguliers
3. **Soyez réactif** : configurez des alertes sur les seuils critiques
4. **Conservez l'historique** : gardez les métriques pour l'analyse
5. **Visualisez** : utilisez les rapports HTML pour une meilleure compréhension

**Pour aller plus loin :**

Pour un monitoring encore plus avancé, considérez l'intégration avec :
- Prometheus + Grafana (voir chapitre 12 et 13)
- ELK/EFK Stack pour les logs
- Alertmanager pour la gestion avancée des alertes

Ces scripts constituent une excellente base de monitoring qui peut être étendue selon vos besoins spécifiques !

---

*Fin de l'Annexe A : Scripts et Automatisation*

⏭️ [Annexe B : Templates et Exemples](/annexes/annexe-b-templates-et-exemples/README.md)
