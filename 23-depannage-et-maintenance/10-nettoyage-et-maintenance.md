🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 23.10 Nettoyage et Maintenance

## Introduction

Un cluster Kubernetes, même bien géré, accumule avec le temps des ressources inutilisées : images Docker obsolètes, pods terminés, logs volumineux, volumes orphelins. Sans maintenance régulière, ces éléments peuvent saturer le disque, ralentir le cluster, et compliquer le dépannage. Cette section vous apprend à maintenir votre cluster MicroK8s propre et performant.

> **Pour les débutants** : Imaginez votre cluster Kubernetes comme une maison. Sans nettoyage régulier, la poussière s'accumule, les placards se remplissent d'objets inutiles, et vous finissez par manquer d'espace. Le nettoyage et la maintenance, c'est faire le ménage régulièrement pour que tout reste en ordre et fonctionne bien.

## Pourquoi Faire du Nettoyage ?

### Les Conséquences d'un Cluster Non Maintenu

**Problèmes courants** :
- **Saturation disque** : Pods évictés, applications qui crashent
- **Performance dégradée** : Lenteurs, timeouts
- **Difficulté de diagnostic** : Trop de ressources inutiles polluent les résultats
- **Gaspillage de ressources** : CPU/Mémoire utilisés pour rien
- **Risques de sécurité** : Images vulnérables qui traînent

### Bénéfices d'une Maintenance Régulière

**Avantages** :
- Cluster stable et performant
- Meilleure utilisation des ressources
- Diagnostic plus facile
- Sécurité améliorée
- Prévention des pannes

## Nettoyage des Images Docker

Les images Docker sont souvent le plus gros consommateur d'espace disque.

### Diagnostic

#### Voir l'Espace Utilisé par les Images

```bash
# Lister toutes les images
microk8s ctr images ls

# Voir la taille
microk8s ctr images ls -q | xargs microk8s ctr images size

# Espace total utilisé
sudo du -sh /var/snap/microk8s/common/var/lib/containerd
```

#### Identifier les Images Non Utilisées

```bash
# Images actuellement utilisées par des pods
USED_IMAGES=$(microk8s kubectl get pods --all-namespaces -o jsonpath="{.items[*].spec.containers[*].image}" | tr -s '[[:space:]]' '\n' | sort | uniq)

# Toutes les images
ALL_IMAGES=$(microk8s ctr images ls -q)

# Comparer pour trouver les non utilisées
# (nécessite un peu de bash)
echo "$ALL_IMAGES" | while read image; do
    if ! echo "$USED_IMAGES" | grep -q "$image"; then
        echo "Non utilisée: $image"
    fi
done
```

### Nettoyage Manuel

#### Supprimer Toutes les Images Non Utilisées

**Attention** : Cette commande supprime TOUTES les images non utilisées !

```bash
# Avec containerd (MicroK8s)
microk8s ctr images ls -q | xargs microk8s ctr images rm

# Ou avec crictl
microk8s crictl rmi --prune
```

**Résultat typique** :
```
Successfully deleted image sha256:abc123...
Successfully deleted image sha256:def456...
...
```

#### Supprimer des Images Spécifiques

```bash
# Supprimer une image par son nom
microk8s ctr images rm docker.io/library/nginx:1.20

# Supprimer par pattern (toutes les versions nginx)
microk8s ctr images ls | grep nginx | awk '{print $1}' | xargs microk8s ctr images rm
```

#### Supprimer les Images Anciennes

```bash
# Script pour supprimer les images de plus de 30 jours
# (nécessite inspection manuelle, containerd ne stocke pas les dates facilement)

# Alternative : Politique d'images - garder seulement les N dernières versions
# À implémenter dans votre CI/CD
```

### Nettoyage Automatique

#### Script de Nettoyage Hebdomadaire

```bash
#!/bin/bash
# cleanup-images.sh

echo "=== Nettoyage des Images Docker ==="

# Avant nettoyage
BEFORE=$(du -sh /var/snap/microk8s/common/var/lib/containerd | cut -f1)
echo "Espace utilisé avant: $BEFORE"

# Supprimer les images non utilisées
echo "Suppression des images non utilisées..."
microk8s crictl rmi --prune

# Après nettoyage
AFTER=$(du -sh /var/snap/microk8s/common/var/lib/containerd | cut -f1)
echo "Espace utilisé après: $AFTER"

echo "Nettoyage terminé!"
```

**Utilisation** :
```bash
chmod +x cleanup-images.sh
./cleanup-images.sh
```

#### Automatisation avec Cron

```bash
# Éditer la crontab
crontab -e

# Ajouter une ligne pour nettoyer tous les dimanches à 2h du matin
0 2 * * 0 /home/user/cleanup-images.sh >> /var/log/cleanup-images.log 2>&1
```

### Politique d'Images

#### Utiliser ImagePullPolicy Appropriée

**Dans vos Deployments** :

```yaml
spec:
  containers:
  - name: mon-app
    image: nginx:1.21
    imagePullPolicy: IfNotPresent  # Ne télécharge que si absent
    # Ou : Always, Never
```

**Valeurs** :
- **Always** : Toujours télécharger (utilise plus de bande passante)
- **IfNotPresent** : Télécharger seulement si absent (recommandé)
- **Never** : Ne jamais télécharger (l'image doit déjà exister)

#### Limiter le Nombre d'Images Conservées

**Configuration du garbage collector** :

Éditer la configuration kubelet (avancé) :

```bash
# Dans /var/snap/microk8s/current/args/kubelet
# Ajouter ou modifier :
--image-gc-high-threshold=80
--image-gc-low-threshold=70
```

**Explication** :
- Quand l'espace disque atteint 80%, commence à supprimer les vieilles images
- Continue jusqu'à descendre à 70%

## Nettoyage des Pods Terminés

### Diagnostic

#### Voir les Pods Terminés

```bash
# Pods Completed (jobs terminés)
microk8s kubectl get pods --all-namespaces --field-selector=status.phase=Succeeded

# Pods Failed
microk8s kubectl get pods --all-namespaces --field-selector=status.phase=Failed

# Pods Evicted
microk8s kubectl get pods --all-namespaces | grep Evicted
```

#### Compter les Pods à Nettoyer

```bash
# Nombre de pods Completed
microk8s kubectl get pods --all-namespaces --field-selector=status.phase=Succeeded --no-headers | wc -l

# Nombre de pods Failed
microk8s kubectl get pods --all-namespaces --field-selector=status.phase=Failed --no-headers | wc -l

# Nombre de pods Evicted
microk8s kubectl get pods --all-namespaces | grep -c Evicted
```

### Nettoyage Manuel

#### Supprimer les Pods Completed

```bash
# Supprimer tous les pods Completed
microk8s kubectl delete pods --all-namespaces --field-selector=status.phase=Succeeded

# Ou par namespace
microk8s kubectl delete pods -n mon-namespace --field-selector=status.phase=Succeeded
```

#### Supprimer les Pods Failed

```bash
# Supprimer tous les pods Failed
microk8s kubectl delete pods --all-namespaces --field-selector=status.phase=Failed
```

#### Supprimer les Pods Evicted

```bash
# Supprimer tous les pods Evicted
microk8s kubectl get pods --all-namespaces | grep Evicted | awk '{print $2 " -n " $1}' | xargs -L1 microk8s kubectl delete pod
```

### Automatisation

#### Script de Nettoyage des Pods

```bash
#!/bin/bash
# cleanup-pods.sh

echo "=== Nettoyage des Pods Terminés ==="

# Compter avant
COMPLETED=$(microk8s kubectl get pods --all-namespaces --field-selector=status.phase=Succeeded --no-headers | wc -l)
FAILED=$(microk8s kubectl get pods --all-namespaces --field-selector=status.phase=Failed --no-headers | wc -l)
EVICTED=$(microk8s kubectl get pods --all-namespaces | grep -c Evicted)

echo "Pods à nettoyer:"
echo "  - Completed: $COMPLETED"
echo "  - Failed: $FAILED"
echo "  - Evicted: $EVICTED"

# Nettoyer
echo "Nettoyage en cours..."

microk8s kubectl delete pods --all-namespaces --field-selector=status.phase=Succeeded 2>/dev/null
microk8s kubectl delete pods --all-namespaces --field-selector=status.phase=Failed 2>/dev/null
microk8s kubectl get pods --all-namespaces | grep Evicted | awk '{print $2 " -n " $1}' | xargs -L1 microk8s kubectl delete pod 2>/dev/null

echo "Nettoyage terminé!"
```

#### Utiliser ttlSecondsAfterFinished pour les Jobs

**Automatiser le nettoyage des Jobs** :

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: mon-job
spec:
  ttlSecondsAfterFinished: 3600  # Supprimer après 1 heure
  template:
    spec:
      containers:
      - name: worker
        image: busybox
        command: ["sh", "-c", "echo Hello && sleep 10"]
      restartPolicy: Never
```

**CronJob avec nettoyage automatique** :

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: backup-job
spec:
  schedule: "0 2 * * *"
  successfulJobsHistoryLimit: 3  # Garder les 3 derniers succès
  failedJobsHistoryLimit: 1      # Garder le dernier échec
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: backup
            image: backup-tool:latest
          restartPolicy: OnFailure
```

## Nettoyage des Logs

Les logs peuvent rapidement occuper beaucoup d'espace.

### Diagnostic

#### Voir l'Espace Utilisé par les Logs

```bash
# Logs des conteneurs
sudo du -sh /var/snap/microk8s/common/var/log

# Détail par fichier
sudo du -h /var/snap/microk8s/common/var/log | sort -rh | head -20

# Logs système MicroK8s
sudo du -sh /var/log/microk8s
```

#### Identifier les Gros Fichiers de Logs

```bash
# Trouver les fichiers de plus de 100MB
sudo find /var/snap/microk8s/common/var/log -type f -size +100M

# Les 10 plus gros fichiers
sudo find /var/snap/microk8s/common/var/log -type f -exec du -h {} + | sort -rh | head -10
```

### Nettoyage Manuel

#### Supprimer les Vieux Logs

```bash
# Supprimer les logs de plus de 7 jours
sudo find /var/snap/microk8s/common/var/log -type f -name "*.log" -mtime +7 -delete

# Supprimer les logs de plus de 30 jours
sudo find /var/snap/microk8s/common/var/log -type f -mtime +30 -delete

# Vider un fichier de log spécifique (sans le supprimer)
sudo truncate -s 0 /var/snap/microk8s/common/var/log/containers/mon-pod.log
```

#### Nettoyer les Logs Journald

```bash
# Voir l'espace utilisé par journald
sudo journalctl --disk-usage

# Nettoyer les logs de plus de 3 jours
sudo journalctl --vacuum-time=3d

# Garder seulement 500MB de logs
sudo journalctl --vacuum-size=500M

# Nettoyer les logs MicroK8s spécifiquement
sudo journalctl --vacuum-time=7d -u snap.microk8s.*
```

### Configuration de la Rotation des Logs

#### Rotation Automatique avec Logrotate

Créer une configuration logrotate pour MicroK8s :

```bash
# Créer le fichier de configuration
sudo nano /etc/logrotate.d/microk8s
```

**Contenu** :

```
/var/snap/microk8s/common/var/log/containers/*.log {
    daily                    # Rotation quotidienne
    rotate 7                 # Garder 7 jours
    compress                 # Compresser les vieux logs
    delaycompress            # Ne pas compresser le dernier
    missingok               # Pas d'erreur si fichier absent
    notifempty              # Ne pas tourner si vide
    create 0644 root root   # Permissions des nouveaux fichiers
    sharedscripts           # Exécuter les scripts une fois
    postrotate
        /bin/kill -USR1 $(cat /var/run/rsyslogd.pid 2>/dev/null) 2>/dev/null || true
    endscript
}
```

**Tester la configuration** :

```bash
# Test sans exécuter
sudo logrotate -d /etc/logrotate.d/microk8s

# Forcer une rotation manuelle
sudo logrotate -f /etc/logrotate.d/microk8s
```

#### Limiter la Taille des Logs de Conteneurs

**Configuration Kubernetes** (avancé) :

Dans `/var/snap/microk8s/current/args/kubelet`, ajouter :

```
--container-log-max-size=10Mi
--container-log-max-files=5
```

Redémarrer MicroK8s :
```bash
microk8s stop
microk8s start
```

### Script de Nettoyage des Logs

```bash
#!/bin/bash
# cleanup-logs.sh

echo "=== Nettoyage des Logs ==="

# Avant
BEFORE=$(sudo du -sh /var/snap/microk8s/common/var/log | cut -f1)
echo "Espace utilisé avant: $BEFORE"

# Nettoyer les vieux logs (plus de 7 jours)
echo "Suppression des logs de plus de 7 jours..."
sudo find /var/snap/microk8s/common/var/log -type f -name "*.log" -mtime +7 -delete

# Nettoyer journald
echo "Nettoyage de journald (garder 7 jours)..."
sudo journalctl --vacuum-time=7d

# Après
AFTER=$(sudo du -sh /var/snap/microk8s/common/var/log | cut -f1)
echo "Espace utilisé après: $AFTER"

echo "Nettoyage terminé!"
```

## Nettoyage des Volumes Persistants

### Diagnostic

#### Voir les PV et PVC

```bash
# Lister tous les PVC
microk8s kubectl get pvc --all-namespaces

# Lister tous les PV
microk8s kubectl get pv

# PV non liés (Released, Available)
microk8s kubectl get pv | grep -E "Released|Available"

# PVC sans pod qui les utilise
# (nécessite inspection manuelle)
```

#### Voir l'Espace Utilisé

```bash
# Espace total des volumes
sudo du -sh /var/snap/microk8s/common/default-storage

# Par volume
sudo du -sh /var/snap/microk8s/common/default-storage/*

# Les 10 plus gros
sudo du -h /var/snap/microk8s/common/default-storage | sort -rh | head -10
```

### Nettoyage Manuel

#### Supprimer les PV Released

**PV en état Released** = Plus attaché à un PVC mais contient encore des données.

```bash
# Lister les PV Released
microk8s kubectl get pv | grep Released

# Supprimer un PV Released spécifique
microk8s kubectl delete pv <pv-name>

# Supprimer tous les PV Released (ATTENTION AUX DONNÉES !)
microk8s kubectl get pv | grep Released | awk '{print $1}' | xargs microk8s kubectl delete pv
```

**Important** : Vérifiez que vous n'avez pas besoin des données avant de supprimer !

#### Nettoyer les Données d'un PV Released

Si vous voulez réutiliser le PV :

```bash
# Trouver le chemin du volume
PATH_VOLUME=$(microk8s kubectl get pv <pv-name> -o jsonpath='{.spec.hostPath.path}')

# Nettoyer les données
sudo rm -rf $PATH_VOLUME/*

# Retirer la référence au PVC
microk8s kubectl patch pv <pv-name> -p '{"spec":{"claimRef": null}}'

# Le PV est maintenant Available
```

#### Supprimer les PVC Non Utilisés

```bash
# Identifier manuellement les PVC non utilisés
# Puis les supprimer
microk8s kubectl delete pvc <pvc-name> -n <namespace>
```

### Script d'Audit des Volumes

```bash
#!/bin/bash
# audit-volumes.sh

echo "=== Audit des Volumes ==="

# PV par statut
echo "PersistentVolumes:"
echo "  - Bound: $(microk8s kubectl get pv | grep -c Bound)"
echo "  - Available: $(microk8s kubectl get pv | grep -c Available)"
echo "  - Released: $(microk8s kubectl get pv | grep -c Released)"

# PVC par namespace
echo ""
echo "PersistentVolumeClaims par namespace:"
for ns in $(microk8s kubectl get ns -o jsonpath='{.items[*].metadata.name}'); do
    count=$(microk8s kubectl get pvc -n $ns --no-headers 2>/dev/null | wc -l)
    if [ $count -gt 0 ]; then
        echo "  - $ns: $count PVC"
    fi
done

# Espace utilisé
echo ""
echo "Espace disque utilisé:"
sudo du -sh /var/snap/microk8s/common/default-storage

# PV Released
echo ""
RELEASED=$(microk8s kubectl get pv | grep -c Released)
if [ $RELEASED -gt 0 ]; then
    echo "⚠️  $RELEASED PV en état Released (à nettoyer)"
    microk8s kubectl get pv | grep Released
fi

echo ""
echo "Audit terminé!"
```

## Nettoyage des Ressources Orphelines

### Namespaces en Terminating

Parfois, un namespace reste bloqué en état `Terminating`.

#### Diagnostic

```bash
# Voir les namespaces Terminating
microk8s kubectl get ns | grep Terminating
```

#### Solution

```bash
# Supprimer les finaliseurs pour forcer la suppression
NS=mon-namespace
microk8s kubectl get namespace $NS -o json | jq '.spec.finalizers = []' | microk8s kubectl replace --raw "/api/v1/namespaces/$NS/finalize" -f -
```

### Pods Orphelins

Pods sans Deployment/ReplicaSet parent.

#### Identifier

```bash
# Lister tous les pods
microk8s kubectl get pods --all-namespaces -o json | jq -r '.items[] | select(.metadata.ownerReferences == null) | "\(.metadata.namespace)/\(.metadata.name)"'
```

#### Nettoyer

Supprimer manuellement si vraiment orphelins :
```bash
microk8s kubectl delete pod <pod-name> -n <namespace>
```

### Services Sans Endpoints

Services qui ne pointent vers aucun pod.

#### Identifier

```bash
# Script pour trouver les services sans endpoints
for ns in $(microk8s kubectl get ns -o jsonpath='{.items[*].metadata.name}'); do
    for svc in $(microk8s kubectl get svc -n $ns -o jsonpath='{.items[*].metadata.name}'); do
        endpoints=$(microk8s kubectl get endpoints $svc -n $ns -o jsonpath='{.subsets[*].addresses[*].ip}')
        if [ -z "$endpoints" ]; then
            echo "Service sans endpoints: $ns/$svc"
        fi
    done
done
```

## Maintenance Préventive

### Vérifications Régulières

#### Check Quotidien

**Script de vérification quotidienne** :

```bash
#!/bin/bash
# daily-check.sh

echo "=== Vérification Quotidienne - $(date) ==="

# État du cluster
echo "1. État du cluster:"
if microk8s status --wait-ready --timeout 30 >/dev/null 2>&1; then
    echo "   ✓ MicroK8s Running"
else
    echo "   ✗ MicroK8s NOT Running"
fi

# Nœuds
echo "2. Nœuds:"
NOT_READY=$(microk8s kubectl get nodes | grep -v Ready | grep -c "<none>")
if [ $NOT_READY -eq 0 ]; then
    echo "   ✓ Tous les nœuds Ready"
else
    echo "   ✗ $NOT_READY nœud(s) NOT Ready"
fi

# Pods problématiques
echo "3. Pods:"
ISSUES=$(microk8s kubectl get pods --all-namespaces | grep -v "Running\|Completed" | grep -v NAME | wc -l)
if [ $ISSUES -eq 0 ]; then
    echo "   ✓ Tous les pods OK"
else
    echo "   ⚠  $ISSUES pod(s) avec problèmes"
    microk8s kubectl get pods --all-namespaces | grep -v "Running\|Completed" | grep -v NAME
fi

# Espace disque
echo "4. Espace disque:"
DISK_USAGE=$(df -h /var/snap/microk8s | tail -1 | awk '{print $5}' | sed 's/%//')
if [ $DISK_USAGE -lt 80 ]; then
    echo "   ✓ Espace disque OK ($DISK_USAGE%)"
else
    echo "   ⚠  Espace disque élevé ($DISK_USAGE%)"
fi

# Certificats
echo "5. Certificats:"
CERT_DAYS=$(echo | openssl s_client -connect localhost:16443 2>/dev/null | openssl x509 -noout -dates 2>/dev/null | grep notAfter | cut -d'=' -f2)
if [ ! -z "$CERT_DAYS" ]; then
    echo "   ✓ Certificats valides jusqu'au $CERT_DAYS"
else
    echo "   ⚠  Impossible de vérifier les certificats"
fi

echo ""
echo "Vérification terminée!"
```

**Automatisation** :

```bash
# Ajouter à crontab pour exécution quotidienne
crontab -e

# Ajouter :
0 9 * * * /home/user/daily-check.sh >> /var/log/microk8s-daily-check.log 2>&1
```

#### Check Hebdomadaire

**Script de maintenance hebdomadaire** :

```bash
#!/bin/bash
# weekly-maintenance.sh

echo "=== Maintenance Hebdomadaire - $(date) ==="

# Espace disque avant
DISK_BEFORE=$(df -h /var/snap/microk8s | tail -1 | awk '{print $3}')
echo "Espace utilisé avant: $DISK_BEFORE"

# 1. Nettoyer les images
echo ""
echo "1. Nettoyage des images..."
microk8s crictl rmi --prune 2>/dev/null

# 2. Nettoyer les pods terminés
echo "2. Nettoyage des pods terminés..."
microk8s kubectl delete pods --all-namespaces --field-selector=status.phase=Succeeded 2>/dev/null
microk8s kubectl delete pods --all-namespaces --field-selector=status.phase=Failed 2>/dev/null

# 3. Nettoyer les pods évictés
echo "3. Nettoyage des pods évictés..."
microk8s kubectl get pods --all-namespaces | grep Evicted | awk '{print $2 " -n " $1}' | xargs -L1 microk8s kubectl delete pod 2>/dev/null

# 4. Nettoyer les logs
echo "4. Nettoyage des logs..."
sudo find /var/snap/microk8s/common/var/log -type f -name "*.log" -mtime +7 -delete 2>/dev/null
sudo journalctl --vacuum-time=7d >/dev/null 2>&1

# 5. Vérifier les mises à jour
echo "5. Vérification des mises à jour..."
UPDATES=$(snap refresh --list 2>/dev/null | grep microk8s | wc -l)
if [ $UPDATES -gt 0 ]; then
    echo "   ⚠  Mise à jour disponible pour MicroK8s"
else
    echo "   ✓ MicroK8s à jour"
fi

# Espace disque après
DISK_AFTER=$(df -h /var/snap/microk8s | tail -1 | awk '{print $3}')
echo ""
echo "Espace utilisé après: $DISK_AFTER"

echo ""
echo "Maintenance hebdomadaire terminée!"
```

### Surveillance Continue

#### Métriques à Surveiller

**1. Espace Disque**
```bash
# Alerte si > 80%
DISK_USAGE=$(df /var/snap/microk8s | tail -1 | awk '{print $5}' | sed 's/%//')
if [ $DISK_USAGE -gt 80 ]; then
    echo "ALERTE: Espace disque à ${DISK_USAGE}%"
fi
```

**2. Utilisation Mémoire**
```bash
# Alerte si > 85%
MEM_USAGE=$(free | grep Mem | awk '{print int($3/$2 * 100)}')
if [ $MEM_USAGE -gt 85 ]; then
    echo "ALERTE: Mémoire à ${MEM_USAGE}%"
fi
```

**3. Nombre de Pods**
```bash
# Nombre total de pods
TOTAL_PODS=$(microk8s kubectl get pods --all-namespaces --no-headers | wc -l)
echo "Nombre de pods: $TOTAL_PODS"
```

**4. Pods Redémarrant**
```bash
# Pods avec beaucoup de redémarrages
microk8s kubectl get pods --all-namespaces | awk '$5 > 5 {print $0}'
```

### Dashboard de Santé

**Script de tableau de bord** :

```bash
#!/bin/bash
# health-dashboard.sh

clear
echo "╔════════════════════════════════════════════════════════════════╗"
echo "         DASHBOARD SANTÉ MICROK8S - $(date +%Y-%m-%d\ %H:%M)         "
echo "╚════════════════════════════════════════════════════════════════╝"

# Version
echo ""
echo "VERSION"
microk8s version | head -2

# État
echo ""
echo "ÉTAT"
microk8s status | head -1

# Nœuds
echo ""
echo "NŒUDS"
microk8s kubectl get nodes

# Utilisation ressources
echo ""
echo "UTILISATION RESSOURCES"
microk8s kubectl top nodes 2>/dev/null || echo "Metrics server non disponible"

# Pods par statut
echo ""
echo "PODS PAR STATUT"
echo "  Running:   $(microk8s kubectl get pods --all-namespaces | grep -c Running)"
echo "  Pending:   $(microk8s kubectl get pods --all-namespaces | grep -c Pending)"
echo "  Failed:    $(microk8s kubectl get pods --all-namespaces | grep -c Failed)"
echo "  Completed: $(microk8s kubectl get pods --all-namespaces | grep -c Completed)"

# Espace disque
echo ""
echo "ESPACE DISQUE"
df -h /var/snap/microk8s | tail -1

# Derniers événements
echo ""
echo "DERNIERS ÉVÉNEMENTS"
microk8s kubectl get events --all-namespaces --sort-by='.lastTimestamp' | tail -5

echo ""
echo "════════════════════════════════════════════════════════════════"
```

**Utilisation** :

```bash
chmod +x health-dashboard.sh
watch -n 30 ./health-dashboard.sh  # Rafraîchir toutes les 30 secondes
```

## Bonnes Pratiques de Maintenance

### 1. Planifier la Maintenance

**Créer un calendrier de maintenance** :

```
QUOTIDIEN (automatisé)
├── Vérification santé du cluster
├── Alerte sur anomalies
└── Logs des événements

HEBDOMADAIRE (semi-automatisé)
├── Nettoyage images
├── Nettoyage pods terminés
├── Nettoyage logs
└── Audit volumes

MENSUEL (manuel)
├── Revue sécurité
├── Audit complet ressources
├── Optimisation configurations
└── Documentation mise à jour

TRIMESTRIEL (planifié)
├── Mise à jour MicroK8s
├── Backup complet
├── Test de restauration
└── Revue architecture
```

### 2. Automatiser le Nettoyage

**Script maître de nettoyage** :

```bash
#!/bin/bash
# master-cleanup.sh

LOG_FILE="/var/log/microk8s-cleanup.log"

log() {
    echo "[$(date '+%Y-%m-%d %H:%M:%S')] $1" | tee -a $LOG_FILE
}

log "=== Début du nettoyage automatique ==="

# Images
log "Nettoyage des images..."
microk8s crictl rmi --prune 2>&1 | tee -a $LOG_FILE

# Pods
log "Nettoyage des pods terminés..."
microk8s kubectl delete pods --all-namespaces --field-selector=status.phase=Succeeded 2>&1 | tee -a $LOG_FILE
microk8s kubectl delete pods --all-namespaces --field-selector=status.phase=Failed 2>&1 | tee -a $LOG_FILE

# Évictés
log "Nettoyage des pods évictés..."
EVICTED=$(microk8s kubectl get pods --all-namespaces | grep Evicted | wc -l)
if [ $EVICTED -gt 0 ]; then
    microk8s kubectl get pods --all-namespaces | grep Evicted | awk '{print $2 " -n " $1}' | xargs -L1 microk8s kubectl delete pod 2>&1 | tee -a $LOG_FILE
fi

# Logs
log "Nettoyage des logs..."
sudo find /var/snap/microk8s/common/var/log -type f -name "*.log" -mtime +7 -delete 2>&1 | tee -a $LOG_FILE
sudo journalctl --vacuum-time=7d 2>&1 | tee -a $LOG_FILE

# Espace libéré
DISK_USAGE=$(df -h /var/snap/microk8s | tail -1 | awk '{print $5}')
log "Espace disque utilisé: $DISK_USAGE"

log "=== Fin du nettoyage automatique ==="
```

**Cron pour automatisation** :

```bash
# /etc/cron.d/microk8s-cleanup
0 2 * * 0 root /usr/local/bin/master-cleanup.sh
```

### 3. Surveiller et Alerter

**Script d'alerte simple** :

```bash
#!/bin/bash
# alert-check.sh

EMAIL="admin@example.com"
ALERT=0

# Disque > 85%
DISK=$(df /var/snap/microk8s | tail -1 | awk '{print $5}' | sed 's/%//')
if [ $DISK -gt 85 ]; then
    echo "ALERTE: Espace disque à ${DISK}%" | mail -s "Alerte MicroK8s: Disque" $EMAIL
    ALERT=1
fi

# Pods en erreur
FAILED=$(microk8s kubectl get pods --all-namespaces | grep -v "Running\|Completed" | grep -v NAME | wc -l)
if [ $FAILED -gt 5 ]; then
    echo "ALERTE: $FAILED pods en erreur" | mail -s "Alerte MicroK8s: Pods" $EMAIL
    ALERT=1
fi

# Cluster down
if ! microk8s status --wait-ready --timeout 10 >/dev/null 2>&1; then
    echo "ALERTE: MicroK8s ne répond pas" | mail -s "Alerte MicroK8s: Cluster Down" $EMAIL
    ALERT=1
fi

if [ $ALERT -eq 0 ]; then
    echo "Tout va bien"
fi
```

### 4. Documenter les Opérations

**Tenir un journal de maintenance** :

```markdown
# Journal de Maintenance MicroK8s

## 2025-10-26

### Actions
- Nettoyage images: 5GB libérés
- Suppression 15 pods évictés
- Logs nettoyés: 2GB libérés

### Observations
- Augmentation utilisation disque sur namespace production
- Pod redis redémarre fréquemment (à investiguer)

### Actions à Prévoir
- Augmenter limite mémoire pod redis
- Auditer volumes namespace production

## 2025-10-19

[...]
```

### 5. Tester les Restaurations

**Régulièrement, testez vos backups** :

```bash
# 1. Faire un backup
microk8s kubectl get all -n production -o yaml > backup-prod.yaml

# 2. Supprimer quelque chose
microk8s kubectl delete deployment test-app -n production

# 3. Restaurer
microk8s kubectl apply -f backup-prod.yaml

# 4. Vérifier
microk8s kubectl get all -n production
```

### 6. Optimiser les Ressources

**Audit régulier** :

```bash
# Pods sur-dimensionnés
microk8s kubectl top pods --all-namespaces | awk 'NR>1 {if ($2+0 < 100) print $0}'

# Pods sous-dimensionnés
microk8s kubectl top pods --all-namespaces | awk 'NR>1 {if ($2+0 > 900) print $0}'
```

### 7. Maintenir la Documentation

**Documenter** :
- Configuration actuelle
- Addons activés
- Applications déployées
- Procédures de maintenance
- Contacts d'urgence

### 8. Prévoir les Capacités

**Anticiper la croissance** :

```bash
# Tendance d'utilisation (à exécuter régulièrement)
echo "$(date),$(df /var/snap/microk8s | tail -1 | awk '{print $5}'),$(free | grep Mem | awk '{print int($3/$2 * 100)}')" >> /var/log/capacity-trend.csv

# Analyser la tendance pour planifier les upgrades
```

## Checklist de Maintenance

### Checklist Quotidienne

- [ ] Vérifier l'état du cluster (`microk8s status`)
- [ ] Vérifier les nœuds (`kubectl get nodes`)
- [ ] Vérifier les pods problématiques (`kubectl get pods -A`)
- [ ] Vérifier l'espace disque (`df -h`)
- [ ] Consulter les événements récents (`kubectl get events`)

### Checklist Hebdomadaire

- [ ] Nettoyer les images non utilisées
- [ ] Nettoyer les pods terminés/évictés
- [ ] Nettoyer les logs anciens
- [ ] Vérifier les mises à jour disponibles
- [ ] Audit des volumes
- [ ] Vérifier les performances

### Checklist Mensuelle

- [ ] Revue complète des ressources
- [ ] Optimisation des configurations
- [ ] Test des backups
- [ ] Audit de sécurité
- [ ] Mise à jour documentation
- [ ] Planification capacités

### Checklist Trimestrielle

- [ ] Mise à jour MicroK8s
- [ ] Backup complet du cluster
- [ ] Test de disaster recovery
- [ ] Revue architecture
- [ ] Formation équipe
- [ ] Revue des SLA

## Résumé

### Commandes Essentielles de Nettoyage

```bash
# Images
microk8s crictl rmi --prune

# Pods terminés
kubectl delete pods --all-namespaces --field-selector=status.phase=Succeeded
kubectl delete pods --all-namespaces --field-selector=status.phase=Failed

# Pods évictés
kubectl get pods -A | grep Evicted | awk '{print $2 " -n " $1}' | xargs -L1 kubectl delete pod

# Logs
sudo find /var/snap/microk8s/common/var/log -type f -mtime +7 -delete
sudo journalctl --vacuum-time=7d

# PV Released
kubectl get pv | grep Released | awk '{print $1}' | xargs kubectl delete pv
```

### Fréquence Recommandée

| Tâche | Fréquence | Automatisation |
|-------|-----------|----------------|
| Vérification santé | Quotidienne | Automatique |
| Nettoyage images | Hebdomadaire | Automatique |
| Nettoyage pods | Hebdomadaire | Automatique |
| Nettoyage logs | Hebdomadaire | Automatique |
| Audit volumes | Mensuelle | Semi-automatique |
| Optimisation | Mensuelle | Manuelle |
| Mise à jour | Trimestrielle | Manuelle |

### Scripts à Conserver

1. **daily-check.sh** - Vérification quotidienne
2. **master-cleanup.sh** - Nettoyage automatique
3. **health-dashboard.sh** - Tableau de bord santé
4. **alert-check.sh** - Système d'alertes

### Points Clés

1. **Automatiser** le maximum de tâches répétitives
2. **Surveiller** régulièrement l'espace disque
3. **Nettoyer** proactivement, pas réactivement
4. **Documenter** toutes les opérations
5. **Tester** les procédures de restauration
6. **Planifier** la maintenance, ne pas la subir
7. **Alerter** sur les anomalies avant qu'elles deviennent critiques
8. **Optimiser** continuellement les ressources

### Outils Recommandés

- **Prometheus/Grafana** : Monitoring avancé
- **Logrotate** : Rotation automatique des logs
- **Cron** : Automatisation des tâches
- **Scripts shell** : Maintenance personnalisée
- **Velero** : Backups automatisés (addon disponible)

---


⏭️ [Outils de diagnostic avancés](/23-depannage-et-maintenance/11-outils-de-diagnostic-avances.md)
