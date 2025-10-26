🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 22.8 Automatisation des sauvegardes

## Introduction

L'automatisation des sauvegardes est le passage d'un système qui dépend de **votre mémoire** à un système qui fonctionne **tout seul**. C'est la différence entre "je dois penser à faire une sauvegarde" et "mes sauvegardes se font automatiquement pendant que je dors".

### Pourquoi automatiser ?

**Le problème humain** :

```
Lundi : "Je ferai une sauvegarde ce soir"
Mardi : "J'ai oublié hier, je le ferai ce soir"
Mercredi : "Trop fatigué, demain c'est sûr"
Jeudi : "Ah mince, j'ai encore oublié"
Vendredi : Le serveur tombe en panne...
Dernière sauvegarde : Il y a 3 semaines
```

**La solution automatisée** :

```
Tous les jours à 2h00 : Backup automatique
Tous les dimanches : Backup archive automatique
Toutes les heures : Validation automatique
En cas d'erreur : Alerte automatique

Résultat : Vous dormez tranquille
```

### Les avantages de l'automatisation

✅ **Fiabilité** : N'oublie jamais, ne se fatigue pas
✅ **Cohérence** : Toujours au même moment, toujours de la même façon
✅ **Tranquillité d'esprit** : Vous n'avez plus à y penser
✅ **Historique régulier** : Points de restauration fréquents
✅ **Détection rapide** : Les erreurs sont signalées immédiatement
✅ **Gain de temps** : Plus de tâches manuelles répétitives

### Analogie simple

Imaginez deux approches pour prendre vos médicaments :

**Approche manuelle** : Vous devez vous rappeler chaque jour à 8h de prendre votre pilule. Vous oubliez régulièrement.

**Approche automatisée** : Vous avez un pilulier avec alarme. Chaque jour à 8h, il sonne. Vous ne pouvez pas oublier.

L'automatisation des sauvegardes, c'est votre pilulier pour vos données.

## Niveaux d'automatisation

L'automatisation peut se faire à différents niveaux, du plus simple au plus sophistiqué.

### Niveau 1 : Script manuel avec rappel

**Complexité** : ⭐ Très simple
**Automatisation** : 20%

Vous avez un script, mais vous devez penser à le lancer.

```bash
#!/bin/bash
# backup-manual.sh
# À lancer manuellement quand vous y pensez

velero backup create manual-backup-$(date +%Y%m%d)
echo "Backup créé, pensez à vérifier !"
```

**Utilisation** :
```bash
./backup-manual.sh
```

**Avantages** : Simple, vous gardez le contrôle
**Inconvénients** : Dépend de votre mémoire

### Niveau 2 : Cron simple

**Complexité** : ⭐⭐ Simple
**Automatisation** : 50%

Le système lance automatiquement le script à horaire fixe.

```bash
# Ajouter dans crontab
# crontab -e

# Backup tous les jours à 2h du matin
0 2 * * * /home/user/scripts/backup-manual.sh
```

**Avantages** : Totalement automatique
**Inconvénients** : Pas de surveillance, pas d'alertes

### Niveau 3 : Velero Schedules

**Complexité** : ⭐⭐ Simple
**Automatisation** : 70%

Velero gère automatiquement les sauvegardes avec rétention.

```bash
velero schedule create daily-backup \
  --schedule="0 2 * * *" \
  --ttl 168h
```

**Avantages** : Intégré, gestion de la rétention automatique
**Inconvénients** : Pas de notifications natives

### Niveau 4 : Orchestration complète

**Complexité** : ⭐⭐⭐⭐ Avancé
**Automatisation** : 95%

Sauvegardes + vérification + alertes + nettoyage + monitoring.

**Avantages** : Système complet, vous n'intervenez qu'en cas de problème
**Inconvénients** : Plus complexe à mettre en place initialement

## Automatisation avec Cron

Cron est le planificateur de tâches classique sous Linux. Simple et efficace.

### Comprendre la syntaxe Cron

Le format Cron : `minute heure jour mois jour_semaine commande`

```
┌─────── Minute (0-59)
│ ┌───── Heure (0-23)
│ │ ┌─── Jour du mois (1-31)
│ │ │ ┌─ Mois (1-12)
│ │ │ │ ┌ Jour de la semaine (0-7, 0 et 7 = dimanche)
│ │ │ │ │
* * * * * commande
```

**Exemples pratiques** :

```bash
# Tous les jours à 2h00
0 2 * * *

# Tous les jours à 2h30
30 2 * * *

# Toutes les heures
0 * * * *

# Toutes les 6 heures
0 */6 * * *

# Tous les lundis à 3h00
0 3 * * 1

# Premier jour du mois à minuit
0 0 1 * *

# Du lundi au vendredi à 9h00
0 9 * * 1-5

# Samedis et dimanches à 10h00
0 10 * * 6,7
```

### Script de sauvegarde pour Cron

Créez un script optimisé pour Cron :

```bash
#!/bin/bash
# /home/user/scripts/automated-backup.sh

# Configuration
LOG_FILE="/var/log/microk8s-backup.log"
BACKUP_NAME="daily-backup-$(date +%Y%m%d-%H%M%S)"
ALERT_EMAIL="admin@example.com"

# Fonction de logging
log() {
  echo "[$(date '+%Y-%m-%d %H:%M:%S')] $1" | tee -a "$LOG_FILE"
}

# Début
log "=== Début de la sauvegarde automatique ==="

# Vérifier que Velero est accessible
if ! command -v velero &> /dev/null; then
  log "ERREUR: Velero CLI non trouvé"
  exit 1
fi

# Créer le backup
log "Création du backup: $BACKUP_NAME"
velero backup create "$BACKUP_NAME" \
  --default-volumes-to-fs-backup \
  --wait 2>&1 | tee -a "$LOG_FILE"

# Vérifier le résultat
if [ $? -eq 0 ]; then
  log "✓ Backup créé avec succès"

  # Obtenir des statistiques
  STATS=$(velero backup describe "$BACKUP_NAME" --details)
  log "Statistiques: $STATS"

  # Nettoyage des anciens backups (optionnel)
  log "Nettoyage des backups de plus de 30 jours..."
  # (implémenté dans une section suivante)

  exit 0
else
  log "✗ ERREUR: Échec de la création du backup"

  # Envoyer une alerte (si mail configuré)
  if command -v mail &> /dev/null; then
    echo "Le backup automatique a échoué. Voir $LOG_FILE" | \
      mail -s "ALERTE: Échec backup MicroK8s" "$ALERT_EMAIL"
  fi

  exit 1
fi
```

**Rendre le script exécutable** :

```bash
chmod +x /home/user/scripts/automated-backup.sh
```

### Configurer Cron

```bash
# Éditer la crontab
crontab -e

# Ajouter les lignes suivantes :

# Variables d'environnement
PATH=/usr/local/bin:/usr/bin:/bin
SHELL=/bin/bash

# Backup quotidien à 2h00
0 2 * * * /home/user/scripts/automated-backup.sh

# Backup hebdomadaire archive (dimanche 1h00)
0 1 * * 0 /home/user/scripts/automated-backup-archive.sh

# Vérification quotidienne des backups à 3h00
0 3 * * * /home/user/scripts/verify-backups.sh

# Nettoyage mensuel (1er du mois)
0 4 1 * * /home/user/scripts/cleanup-old-backups.sh
```

### Vérifier les Cron jobs

```bash
# Lister vos cron jobs
crontab -l

# Voir les logs Cron (Ubuntu/Debian)
grep CRON /var/log/syslog

# Ou sur d'autres systèmes
journalctl -u cron

# Vérifier votre log personnalisé
tail -f /var/log/microk8s-backup.log
```

### Tester votre Cron

Ne pas attendre 2h du matin pour savoir si ça marche !

```bash
# Test immédiat du script
/home/user/scripts/automated-backup.sh

# Vérifier les logs
tail -20 /var/log/microk8s-backup.log

# Pour tester le cron lui-même, temporairement mettre :
# */5 * * * * /home/user/scripts/automated-backup.sh
# (Toutes les 5 minutes)
# Puis remettre l'horaire normal une fois validé
```

## Automatisation avec Systemd Timers

Systemd timers est l'alternative moderne à Cron, plus flexible et mieux intégrée.

### Pourquoi Systemd Timers ?

**Avantages sur Cron** :
- Logs intégrés avec journalctl
- Gestion des dépendances (ne lance pas si le système vient de démarrer)
- Randomisation possible (éviter que tous les serveurs fassent leur backup en même temps)
- Meilleure gestion des erreurs
- Peut déclencher sur événements système

### Créer un Service Systemd

Créez le fichier `/etc/systemd/system/microk8s-backup.service` :

```ini
[Unit]
Description=MicroK8s Automatic Backup
Documentation=https://example.com/backup-docs
After=network-online.target
Wants=network-online.target

[Service]
Type=oneshot
User=root
Group=root

# Le script à exécuter
ExecStart=/home/user/scripts/automated-backup.sh

# Timeout (2 heures max)
TimeoutStartSec=7200

# En cas d'échec, notifier
StandardOutput=journal
StandardError=journal
SyslogIdentifier=microk8s-backup

# Redémarrer si échec
Restart=on-failure
RestartSec=5m

[Install]
WantedBy=multi-user.target
```

### Créer le Timer associé

Créez le fichier `/etc/systemd/system/microk8s-backup.timer` :

```ini
[Unit]
Description=MicroK8s Backup Timer
Documentation=https://example.com/backup-docs
Requires=microk8s-backup.service

[Timer]
# Quand déclencher
OnCalendar=daily
# À quelle heure précise (2h du matin)
OnCalendar=*-*-* 02:00:00

# Ajouter un délai aléatoire de 0-30 min (évite les pics)
RandomizedDelaySec=30min

# Si le système était éteint, rattraper au démarrage
Persistent=true

# Ne pas lancer si le système vient juste de démarrer
OnBootSec=15min

[Install]
WantedBy=timers.target
```

### Activer et gérer le Timer

```bash
# Recharger systemd pour prendre en compte les nouveaux fichiers
sudo systemctl daemon-reload

# Activer le timer (démarre au boot)
sudo systemctl enable microk8s-backup.timer

# Démarrer le timer maintenant
sudo systemctl start microk8s-backup.timer

# Vérifier le statut
sudo systemctl status microk8s-backup.timer

# Lister tous les timers
sudo systemctl list-timers

# Voir quand le prochain backup aura lieu
sudo systemctl list-timers microk8s-backup.timer

# Voir les logs du service
sudo journalctl -u microk8s-backup.service

# Suivre les logs en temps réel
sudo journalctl -u microk8s-backup.service -f

# Tester manuellement le service
sudo systemctl start microk8s-backup.service
```

### Exemples d'horaires OnCalendar

```ini
# Tous les jours à 2h00
OnCalendar=*-*-* 02:00:00

# Toutes les heures
OnCalendar=hourly

# Toutes les 6 heures
OnCalendar=0/6:00:00

# Lundis à 3h00
OnCalendar=Mon *-*-* 03:00:00

# 1er de chaque mois à minuit
OnCalendar=*-*-01 00:00:00

# Du lundi au vendredi à 9h
OnCalendar=Mon..Fri *-*-* 09:00:00

# Tous les quarts d'heure
OnCalendar=*:0/15
```

## Automatisation avec Velero Schedules

La méthode native et la plus simple pour Kubernetes.

### Créer un Schedule de base

```bash
# Backup quotidien simple
velero schedule create daily-backup \
  --schedule="0 2 * * *" \
  --ttl 168h

# Vérifier
velero schedule get
```

### Schedule avancé avec toutes les options

```bash
velero schedule create production-backup \
  --schedule="0 */6 * * *" \
  --include-namespaces production,database \
  --exclude-resources events,endpoints \
  --default-volumes-to-fs-backup \
  --ttl 720h \
  --labels environment=production,backup-type=automated
```

**Options expliquées** :
- `--schedule` : Expression cron
- `--include-namespaces` : Seulement ces namespaces
- `--exclude-resources` : Ne pas sauvegarder ces types
- `--default-volumes-to-fs-backup` : Sauvegarder les volumes avec Restic
- `--ttl` : Durée de rétention (720h = 30 jours)
- `--labels` : Labels pour organiser/filtrer

### Stratégie multi-schedules

Combinez plusieurs schedules pour différents besoins :

```bash
# 1. Backup fréquent des données critiques
velero schedule create critical-data \
  --schedule="0 */2 * * *" \
  --selector criticality=high \
  --default-volumes-to-fs-backup \
  --ttl 48h

# 2. Backup quotidien complet
velero schedule create daily-full \
  --schedule="0 2 * * *" \
  --exclude-namespaces kube-system,kube-public,velero \
  --default-volumes-to-fs-backup \
  --ttl 168h

# 3. Backup hebdomadaire archive
velero schedule create weekly-archive \
  --schedule="0 1 * * 0" \
  --default-volumes-to-fs-backup \
  --ttl 2160h

# 4. Backup mensuel longue rétention
velero schedule create monthly-archive \
  --schedule="0 0 1 * *" \
  --default-volumes-to-fs-backup \
  --ttl 8760h
```

### Gérer les Schedules

```bash
# Lister tous les schedules
velero schedule get

# Voir les détails d'un schedule
velero schedule describe daily-backup

# Suspendre temporairement
velero schedule pause daily-backup

# Reprendre
velero schedule unpause daily-backup

# Modifier (supprimer puis recréer)
velero schedule delete daily-backup
velero schedule create daily-backup --schedule="0 3 * * *" ...

# Voir les backups créés par un schedule
velero backup get -l velero.io/schedule-name=daily-backup
```

### Schedule via YAML

Pour une gestion GitOps, créez des fichiers YAML :

```yaml
# daily-backup-schedule.yaml
apiVersion: velero.io/v1
kind: Schedule
metadata:
  name: daily-backup
  namespace: velero
spec:
  # Expression cron
  schedule: "0 2 * * *"

  # Template du backup
  template:
    # Rétention
    ttl: 168h0m0s

    # Namespaces inclus
    includedNamespaces:
    - production
    - staging

    # Namespaces exclus
    excludedNamespaces:
    - kube-system
    - kube-public

    # Types de ressources exclus
    excludedResources:
    - events
    - events.events.k8s.io

    # Sauvegarder les volumes
    defaultVolumesToFsBackup: true

    # Backup location
    storageLocation: default

    # Labels
    labels:
      backup-type: automated
      frequency: daily

    # Hooks (exemple pour BDD)
    hooks:
      resources:
      - name: postgres-backup
        includedNamespaces:
        - database
        labelSelector:
          matchLabels:
            app: postgres
        pre:
        - exec:
            container: postgres
            command:
            - /bin/bash
            - -c
            - pg_dump mydb > /tmp/backup.sql
            onError: Fail
            timeout: 5m
```

Appliquer :

```bash
kubectl apply -f daily-backup-schedule.yaml
```

## Vérification automatique des backups

Automatiser non seulement la création, mais aussi la vérification.

### Script de vérification

```bash
#!/bin/bash
# verify-backups.sh

LOG_FILE="/var/log/backup-verification.log"
ALERT_EMAIL="admin@example.com"

log() {
  echo "[$(date '+%Y-%m-%d %H:%M:%S')] $1" | tee -a "$LOG_FILE"
}

log "=== Vérification des backups ==="

# 1. Vérifier que Velero fonctionne
if ! kubectl get pods -n velero | grep -q "Running"; then
  log "✗ ERREUR: Pods Velero ne sont pas en Running"
  exit 1
fi

# 2. Vérifier le backup-location
LOCATION_STATUS=$(velero backup-location get -o json | jq -r '.items[0].status.phase')
if [ "$LOCATION_STATUS" != "Available" ]; then
  log "✗ ERREUR: Backup location n'est pas disponible: $LOCATION_STATUS"
  exit 1
fi
log "✓ Backup location OK"

# 3. Vérifier le dernier backup
LATEST_BACKUP=$(velero backup get -o json | jq -r '.items | sort_by(.status.startTimestamp) | last | .metadata.name')
BACKUP_STATUS=$(velero backup describe "$LATEST_BACKUP" | grep "Phase:" | awk '{print $2}')
BACKUP_AGE=$(velero backup get "$LATEST_BACKUP" -o json | jq -r '.status.startTimestamp')

log "Dernier backup: $LATEST_BACKUP"
log "Status: $BACKUP_STATUS"
log "Date: $BACKUP_AGE"

if [ "$BACKUP_STATUS" != "Completed" ]; then
  log "✗ ERREUR: Dernier backup en échec: $BACKUP_STATUS"

  # Envoyer alerte
  if command -v mail &> /dev/null; then
    velero backup logs "$LATEST_BACKUP" | \
      mail -s "ALERTE: Backup échoué - $LATEST_BACKUP" "$ALERT_EMAIL"
  fi

  exit 1
fi

# 4. Vérifier que le backup n'est pas trop vieux
BACKUP_TIMESTAMP=$(date -d "$BACKUP_AGE" +%s)
NOW=$(date +%s)
AGE_HOURS=$(( (NOW - BACKUP_TIMESTAMP) / 3600 ))

if [ $AGE_HOURS -gt 30 ]; then
  log "⚠ ATTENTION: Dernier backup date de $AGE_HOURS heures"
  exit 1
fi
log "✓ Backup récent (${AGE_HOURS}h)"

# 5. Vérifier la taille du backup
BACKUP_SIZE=$(velero backup describe "$LATEST_BACKUP" --details | grep "Total items" | awk '{print $6}')
log "Ressources sauvegardées: $BACKUP_SIZE"

if [ "$BACKUP_SIZE" -lt 10 ]; then
  log "⚠ ATTENTION: Backup semble petit ($BACKUP_SIZE ressources)"
fi

# 6. Vérifier l'espace disque sur MinIO/stockage
# (Exemple pour MinIO accessible localement)
if command -v df &> /dev/null; then
  DISK_USAGE=$(df -h /mnt/backups | tail -1 | awk '{print $5}' | sed 's/%//')
  if [ "$DISK_USAGE" -gt 80 ]; then
    log "⚠ ATTENTION: Espace disque à ${DISK_USAGE}%"
  else
    log "✓ Espace disque OK (${DISK_USAGE}%)"
  fi
fi

log "=== Vérification terminée avec succès ==="
exit 0
```

### Automatiser la vérification

**Avec Cron** :

```bash
# Vérifier tous les jours à 3h (après le backup de 2h)
0 3 * * * /home/user/scripts/verify-backups.sh
```

**Avec Systemd Timer** :

```ini
# /etc/systemd/system/backup-verification.timer
[Unit]
Description=Backup Verification Timer

[Timer]
OnCalendar=*-*-* 03:00:00
Persistent=true

[Install]
WantedBy=timers.target
```

**Avec Kubernetes CronJob** :

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: backup-verification
  namespace: velero
spec:
  schedule: "0 3 * * *"
  jobTemplate:
    spec:
      template:
        spec:
          serviceAccountName: velero
          containers:
          - name: verifier
            image: bitnami/kubectl:latest
            command:
            - /bin/bash
            - -c
            - |
              # Script de vérification inline
              echo "Vérification des backups..."

              LATEST=$(velero backup get -o json | jq -r '.items | sort_by(.status.startTimestamp) | last | .metadata.name')
              STATUS=$(velero backup describe "$LATEST" | grep "Phase:" | awk '{print $2}')

              if [ "$STATUS" = "Completed" ]; then
                echo "✓ Backup OK: $LATEST"
                exit 0
              else
                echo "✗ Backup en échec: $LATEST"
                exit 1
              fi
          restartPolicy: OnFailure
```

## Nettoyage automatique

Éviter que les backups prennent tout l'espace disque.

### Script de nettoyage intelligent

```bash
#!/bin/bash
# cleanup-old-backups.sh

LOG_FILE="/var/log/backup-cleanup.log"

log() {
  echo "[$(date '+%Y-%m-%d %H:%M:%S')] $1" | tee -a "$LOG_FILE"
}

log "=== Nettoyage des anciens backups ==="

# Stratégie de rétention :
# - Garder tous les backups des 7 derniers jours
# - Garder 1 backup par semaine pour le dernier mois
# - Garder 1 backup par mois pour la dernière année

# 1. Supprimer les backups expirés (TTL)
log "Suppression des backups expirés..."
EXPIRED=$(velero backup get -o json | jq -r '.items[] | select(.status.expiration != null and (.status.expiration | fromdateiso8601) < now) | .metadata.name')

if [ -n "$EXPIRED" ]; then
  for backup in $EXPIRED; do
    log "Suppression: $backup (expiré)"
    velero backup delete "$backup" --confirm
  done
else
  log "Aucun backup expiré"
fi

# 2. Appliquer la stratégie GFS (Grandfather-Father-Son)
CUTOFF_DAILY=$(date -d '7 days ago' +%s)
CUTOFF_WEEKLY=$(date -d '30 days ago' +%s)
CUTOFF_MONTHLY=$(date -d '1 year ago' +%s)

# Obtenir tous les backups
ALL_BACKUPS=$(velero backup get -o json)

# Backups à supprimer (plus vieux que 7 jours, pas hebdo/mensuel)
TO_DELETE=$(echo "$ALL_BACKUPS" | jq -r --arg cutoff "$CUTOFF_DAILY" '
  .items[] |
  select(
    (.status.startTimestamp | fromdateiso8601) < ($cutoff | tonumber) and
    (.metadata.labels."backup-type" // "" != "weekly") and
    (.metadata.labels."backup-type" // "" != "monthly")
  ) |
  .metadata.name
')

if [ -n "$TO_DELETE" ]; then
  for backup in $TO_DELETE; do
    log "Suppression: $backup (rotation quotidienne)"
    velero backup delete "$backup" --confirm
  done
fi

# 3. Vérifier l'espace disque
TOTAL_BACKUPS=$(echo "$ALL_BACKUPS" | jq '.items | length')
log "Backups restants: $TOTAL_BACKUPS"

log "=== Nettoyage terminé ==="
```

### Politique de rétention TTL simple

La méthode la plus simple : laisser Velero gérer avec TTL.

```bash
# Backups quotidiens - 7 jours
velero schedule create daily \
  --schedule="0 2 * * *" \
  --ttl 168h

# Backups hebdomadaires - 30 jours
velero schedule create weekly \
  --schedule="0 1 * * 0" \
  --ttl 720h \
  --labels backup-type=weekly

# Backups mensuels - 1 an
velero schedule create monthly \
  --schedule="0 0 1 * *" \
  --ttl 8760h \
  --labels backup-type=monthly
```

Velero supprimera automatiquement les backups expirés.

## Notifications automatiques

Être informé automatiquement des succès et échecs.

### Notifications par email

```bash
#!/bin/bash
# notify-backup-status.sh

BACKUP_NAME=$1
STATUS=$2
EMAIL="admin@example.com"

if [ "$STATUS" = "success" ]; then
  SUBJECT="✓ Backup réussi: $BACKUP_NAME"
  BODY="Le backup $BACKUP_NAME s'est terminé avec succès.

Détails:
$(velero backup describe $BACKUP_NAME)
"
else
  SUBJECT="✗ ALERTE: Backup échoué - $BACKUP_NAME"
  BODY="Le backup $BACKUP_NAME a échoué.

Logs:
$(velero backup logs $BACKUP_NAME)
"
fi

# Envoyer l'email
echo "$BODY" | mail -s "$SUBJECT" "$EMAIL"
```

**Configuration de mail** :

```bash
# Installer mailutils
sudo apt install mailutils

# Configurer (exemple avec Gmail)
sudo nano /etc/ssmtp/ssmtp.conf
```

```ini
root=postmaster
mailhub=smtp.gmail.com:587
AuthUser=votre-email@gmail.com
AuthPass=votre-mot-de-passe-application
UseSTARTTLS=YES
```

### Notifications Slack

```bash
#!/bin/bash
# notify-slack.sh

WEBHOOK_URL="https://hooks.slack.com/services/YOUR/WEBHOOK/URL"
BACKUP_NAME=$1
STATUS=$2

if [ "$STATUS" = "success" ]; then
  COLOR="good"
  EMOJI=":white_check_mark:"
  TEXT="Backup réussi"
else
  COLOR="danger"
  EMOJI=":x:"
  TEXT="BACKUP ÉCHOUÉ"
fi

# Créer le payload JSON
PAYLOAD=$(cat <<EOF
{
  "attachments": [{
    "color": "$COLOR",
    "title": "$EMOJI $TEXT: $BACKUP_NAME",
    "text": "Backup automatique MicroK8s",
    "fields": [
      {
        "title": "Backup",
        "value": "$BACKUP_NAME",
        "short": true
      },
      {
        "title": "Status",
        "value": "$STATUS",
        "short": true
      },
      {
        "title": "Timestamp",
        "value": "$(date)",
        "short": false
      }
    ],
    "footer": "MicroK8s Backup System"
  }]
}
EOF
)

# Envoyer à Slack
curl -X POST -H 'Content-type: application/json' \
  --data "$PAYLOAD" \
  "$WEBHOOK_URL"
```

### Notifications Discord

```bash
#!/bin/bash
# notify-discord.sh

WEBHOOK_URL="https://discord.com/api/webhooks/YOUR/WEBHOOK"
BACKUP_NAME=$1
STATUS=$2

if [ "$STATUS" = "success" ]; then
  COLOR=3066993  # Vert
  TITLE="✅ Backup réussi"
else
  COLOR=15158332  # Rouge
  TITLE="❌ BACKUP ÉCHOUÉ"
fi

PAYLOAD=$(cat <<EOF
{
  "embeds": [{
    "title": "$TITLE",
    "description": "Backup: $BACKUP_NAME",
    "color": $COLOR,
    "fields": [
      {
        "name": "Status",
        "value": "$STATUS",
        "inline": true
      },
      {
        "name": "Timestamp",
        "value": "$(date)",
        "inline": true
      }
    ],
    "footer": {
      "text": "MicroK8s Backup System"
    }
  }]
}
EOF
)

curl -X POST -H 'Content-type: application/json' \
  --data "$PAYLOAD" \
  "$WEBHOOK_URL"
```

### Intégration des notifications

**Dans le script de backup** :

```bash
#!/bin/bash
# automated-backup-with-notifications.sh

BACKUP_NAME="daily-backup-$(date +%Y%m%d-%H%M%S)"

# Créer le backup
velero backup create "$BACKUP_NAME" --wait

# Vérifier le résultat
STATUS=$(velero backup describe "$BACKUP_NAME" | grep "Phase:" | awk '{print $2}')

if [ "$STATUS" = "Completed" ]; then
  # Notifier succès
  /home/user/scripts/notify-slack.sh "$BACKUP_NAME" "success"
else
  # Notifier échec
  /home/user/scripts/notify-slack.sh "$BACKUP_NAME" "failure"
  /home/user/scripts/notify-email.sh "$BACKUP_NAME" "failure"
fi
```

### Webhook custom pour notifications

Créer un serveur de webhook simple :

```python
# webhook-server.py
from flask import Flask, request
import requests

app = Flask(__name__)

@app.route('/backup-notification', methods=['POST'])
def backup_notification():
    data = request.json
    backup_name = data.get('backup_name')
    status = data.get('status')

    # Traiter la notification
    print(f"Backup {backup_name}: {status}")

    # Envoyer à Slack, Discord, email, etc.
    send_to_slack(backup_name, status)

    return {'status': 'ok'}, 200

def send_to_slack(backup, status):
    # Votre logique Slack
    pass

if __name__ == '__main__':
    app.run(host='0.0.0.0', port=5000)
```

**Appeler depuis le script** :

```bash
curl -X POST http://localhost:5000/backup-notification \
  -H 'Content-Type: application/json' \
  -d "{\"backup_name\": \"$BACKUP_NAME\", \"status\": \"$STATUS\"}"
```

## Monitoring avec Prometheus

Surveiller les métriques de backup en continu.

### Métriques Velero

Velero expose des métriques Prometheus sur le port 8085.

**Service pour exposer les métriques** :

```yaml
apiVersion: v1
kind: Service
metadata:
  name: velero-metrics
  namespace: velero
  labels:
    app.kubernetes.io/name: velero
spec:
  selector:
    name: velero
  ports:
  - name: metrics
    port: 8085
    targetPort: 8085
```

**ServiceMonitor (si Prometheus Operator)** :

```yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: velero
  namespace: velero
spec:
  selector:
    matchLabels:
      app.kubernetes.io/name: velero
  endpoints:
  - port: metrics
    interval: 30s
    path: /metrics
```

### Alertes Prometheus

```yaml
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: velero-backup-alerts
  namespace: monitoring
spec:
  groups:
  - name: velero.backups
    interval: 30s
    rules:
    # Alerte si aucun backup réussi depuis 25h
    - alert: NoRecentBackup
      expr: |
        (time() - velero_backup_last_successful_timestamp{schedule!=""} > 90000)
      for: 5m
      labels:
        severity: warning
      annotations:
        summary: "Aucun backup récent pour {{ $labels.schedule }}"
        description: "Le dernier backup réussi date de plus de 25 heures"

    # Alerte si backup en échec
    - alert: BackupFailed
      expr: |
        increase(velero_backup_failure_total[5m]) > 0
      for: 1m
      labels:
        severity: critical
      annotations:
        summary: "Backup Velero en échec"
        description: "Un backup a échoué"

    # Alerte si backup-location indisponible
    - alert: BackupLocationUnavailable
      expr: |
        velero_backup_storage_location_available == 0
      for: 10m
      labels:
        severity: critical
      annotations:
        summary: "Backup storage location indisponible"
        description: "Impossible d'accéder au stockage de backup"

    # Alerte si restauration en échec
    - alert: RestoreFailed
      expr: |
        increase(velero_restore_failed_total[10m]) > 0
      labels:
        severity: warning
      annotations:
        summary: "Restauration Velero en échec"
        description: "Une restauration a échoué"
```

### Dashboard Grafana

Importer le dashboard Velero (ID: 11055) ou créer le vôtre :

```json
{
  "dashboard": {
    "title": "Velero Backup Monitoring",
    "panels": [
      {
        "title": "Backup Success Rate",
        "targets": [{
          "expr": "sum(velero_backup_success_total) / sum(velero_backup_total) * 100"
        }],
        "type": "gauge"
      },
      {
        "title": "Time Since Last Backup",
        "targets": [{
          "expr": "(time() - velero_backup_last_successful_timestamp) / 3600"
        }],
        "unit": "hours"
      },
      {
        "title": "Backup Duration",
        "targets": [{
          "expr": "velero_backup_duration_seconds"
        }],
        "unit": "seconds"
      },
      {
        "title": "Backup Attempts (24h)",
        "targets": [{
          "expr": "increase(velero_backup_attempt_total[24h])"
        }]
      }
    ]
  }
}
```

## Orchestration complète

Combiner tous les éléments dans un système cohérent.

### Architecture globale

```
┌────────────────────────────────────────────────────────┐
│                   ORCHESTRATION                        │
├────────────────────────────────────────────────────────┤
│                                                        │
│  ┌──────────────┐      ┌──────────────┐                │
│  │   Velero     │      │    Cron      │                │
│  │  Schedules   │      │   Scripts    │                │
│  └──────┬───────┘      └──────┬───────┘                │
│         │                     │                        │
│         └─────────┬───────────┘                        │
│                   │                                    │
│            ┌──────▼──────┐                             │
│            │   BACKUPS   │                             │
│            └──────┬──────┘                             │
│                   │                                    │
│         ┌─────────┼─────────┐                          │
│         │         │         │                          │
│    ┌────▼───┐ ┌───▼──┐ ┌────▼──┐                       │
│    │Verify  │ │Clean │ │Notify │                       │
│    └────┬───┘ └──┬───┘ └───┬───┘                       │
│         │        │         │                           │
│         └────────┼─────────┘                           │
│                  │                                     │
│           ┌──────▼──────┐                              │
│           │ Monitoring  │                              │
│           │ (Prometheus)│                              │
│           └─────────────┘                              │
│                                                        │
└────────────────────────────────────────────────────────┘
```

### Script orchestrateur principal

```bash
#!/bin/bash
# backup-orchestrator.sh - Orchestration complète

set -euo pipefail

# Configuration
SCRIPT_DIR="/home/user/scripts"
LOG_DIR="/var/log/backups"
CONFIG_FILE="/etc/backup-config.conf"

# Charger la configuration
source "$CONFIG_FILE"

# Fonctions
log() {
  echo "[$(date '+%Y-%m-%d %H:%M:%S')] $1" | tee -a "$LOG_DIR/orchestrator.log"
}

run_with_notification() {
  local task_name=$1
  local command=$2

  log "Début: $task_name"

  if eval "$command"; then
    log "✓ $task_name réussi"
    notify "success" "$task_name"
    return 0
  else
    log "✗ $task_name échoué"
    notify "failure" "$task_name"
    return 1
  fi
}

notify() {
  local status=$1
  local task=$2

  # Notification Slack
  "$SCRIPT_DIR/notify-slack.sh" "$task" "$status"

  # Si échec, email aussi
  if [ "$status" = "failure" ]; then
    "$SCRIPT_DIR/notify-email.sh" "$task" "$status"
  fi
}

# Workflow principal
main() {
  log "=== Démarrage de l'orchestration de backup ==="

  # 1. Pré-vérifications
  run_with_notification "Pré-vérification" \
    "$SCRIPT_DIR/pre-check.sh"

  # 2. Backup principal
  run_with_notification "Backup principal" \
    "$SCRIPT_DIR/create-backup.sh"

  # 3. Vérification du backup
  run_with_notification "Vérification backup" \
    "$SCRIPT_DIR/verify-backup.sh"

  # 4. Nettoyage
  run_with_notification "Nettoyage anciens backups" \
    "$SCRIPT_DIR/cleanup-backups.sh"

  # 5. Mise à jour des métriques
  run_with_notification "Mise à jour métriques" \
    "$SCRIPT_DIR/update-metrics.sh"

  # 6. Rapport quotidien
  "$SCRIPT_DIR/generate-daily-report.sh"

  log "=== Orchestration terminée ==="
}

# Gestion des erreurs
trap 'log "ERREUR à la ligne $LINENO"; notify "failure" "Orchestration"; exit 1' ERR

# Exécution
main "$@"
```

### Configuration centralisée

```bash
# /etc/backup-config.conf

# Chemins
SCRIPT_DIR="/home/user/scripts"
LOG_DIR="/var/log/backups"
BACKUP_DIR="/mnt/backups"

# Velero
VELERO_NAMESPACE="velero"
BACKUP_PREFIX="auto"

# Rétention
DAILY_RETENTION="7"
WEEKLY_RETENTION="4"
MONTHLY_RETENTION="12"

# Notifications
SLACK_WEBHOOK="https://hooks.slack.com/services/YOUR/WEBHOOK"
EMAIL_ALERT="admin@example.com"
NOTIFY_SUCCESS="true"
NOTIFY_FAILURE="true"

# Monitoring
PROMETHEUS_PUSHGATEWAY="http://prometheus-pushgateway:9091"
METRICS_JOB="backup_automation"

# Limites
MAX_BACKUP_DURATION="7200"  # 2 heures
MIN_BACKUP_SIZE="10"        # 10 ressources minimum
DISK_ALERT_THRESHOLD="80"   # %
```

## Bonnes pratiques d'automatisation

### 1. Principes KISS et DRY

**KISS (Keep It Simple, Stupid)** :
- Commencez simple, complexifiez si nécessaire
- Un script = une fonction claire
- Documentation claire

**DRY (Don't Repeat Yourself)** :
- Fonctions réutilisables
- Configuration centralisée
- Templates

### 2. Gestion des erreurs

```bash
#!/bin/bash
# Toujours activer le mode strict
set -euo pipefail

# Piège pour les erreurs
trap 'echo "Erreur ligne $LINENO"; cleanup; exit 1' ERR

# Fonction de nettoyage
cleanup() {
  # Nettoyage des ressources temporaires
  rm -f /tmp/backup-temp-*
}

# S'assurer du cleanup même en cas de succès
trap cleanup EXIT
```

### 3. Logging exhaustif

```bash
# Fonction de log structuré
log() {
  local level=$1
  shift
  local message="$@"
  local timestamp=$(date '+%Y-%m-%d %H:%M:%S')

  echo "[$timestamp] [$level] $message" | tee -a "$LOG_FILE"

  # Syslog aussi
  logger -t backup-automation "[$level] $message"
}

# Utilisation
log INFO "Démarrage du backup"
log WARN "Espace disque faible"
log ERROR "Échec de connexion"
```

### 4. Idempotence

Les scripts doivent pouvoir être relancés sans problème :

```bash
# Mauvais (crée un doublon)
velero backup create daily-backup

# Bon (vérifie d'abord)
BACKUP_NAME="daily-backup-$(date +%Y%m%d)"
if ! velero backup get "$BACKUP_NAME" &>/dev/null; then
  velero backup create "$BACKUP_NAME"
else
  echo "Backup $BACKUP_NAME existe déjà"
fi
```

### 5. Tests et validation

```bash
# Toujours inclure un mode dry-run
if [ "${DRY_RUN:-false}" = "true" ]; then
  echo "Mode dry-run: velero backup create ..."
  exit 0
fi

# Mode de test
if [ "${TEST_MODE:-false}" = "true" ]; then
  # Utiliser des données de test
  BACKUP_PREFIX="test"
  NOTIFICATION_CHANNEL="#test-alerts"
fi
```

### 6. Documentation intégrée

```bash
#!/bin/bash
# automated-backup.sh
#
# Description: Script d'automatisation des backups MicroK8s
# Auteur: Votre nom
# Date: 2025-01-15
# Version: 2.1
#
# Usage:
#   ./automated-backup.sh [OPTIONS]
#
# Options:
#   -h, --help       Afficher l'aide
#   -n, --dry-run    Mode simulation
#   -t, --test       Mode test
#   -v, --verbose    Mode verbeux
#
# Exemples:
#   ./automated-backup.sh
#   ./automated-backup.sh --dry-run
#   TEST_MODE=true ./automated-backup.sh
#
# Dépendances:
#   - velero CLI
#   - kubectl
#   - jq
#
# Configuration:
#   Voir /etc/backup-config.conf

# Fonction d'aide
show_help() {
  grep '^#' "$0" | grep -v '#!/bin/bash' | sed 's/^# //'
}

# Parse des arguments
while [[ $# -gt 0 ]]; do
  case $1 in
    -h|--help)
      show_help
      exit 0
      ;;
    -n|--dry-run)
      DRY_RUN=true
      shift
      ;;
    *)
      echo "Option inconnue: $1"
      show_help
      exit 1
      ;;
  esac
done

# ... reste du script
```

### 7. Sécurité

```bash
# Ne jamais logger les credentials
log "Connexion à MinIO à $MINIO_URL"  # Bon
log "Connexion avec user=$USER pass=$PASS"  # MAUVAIS !

# Permissions strictes sur les fichiers sensibles
chmod 600 /etc/backup-config.conf
chmod 700 /home/user/scripts/*.sh

# Ne pas stocker les secrets dans les scripts
# Utiliser des variables d'environnement ou des fichiers séparés
source /etc/backup-secrets.conf  # Avec chmod 600
```

## Checklist d'automatisation complète

```markdown
## Configuration initiale

- [ ] Velero schedules configurés
- [ ] Scripts de backup créés et testés
- [ ] Cron ou systemd timers configurés
- [ ] Configuration centralisée créée
- [ ] Logs centralisés configurés

## Vérification automatique

- [ ] Script de vérification créé
- [ ] Vérification quotidienne planifiée
- [ ] Alertes configurées pour échecs
- [ ] Monitoring Prometheus en place
- [ ] Dashboard Grafana configuré

## Nettoyage

- [ ] Politique de rétention définie
- [ ] Script de nettoyage créé
- [ ] Nettoyage automatique planifié
- [ ] Vérification espace disque automatique

## Notifications

- [ ] Notifications succès configurées
- [ ] Notifications échec configurées
- [ ] Canaux multiples (email + Slack)
- [ ] Escalade en cas d'échec répété

## Tests

- [ ] Tests manuels réussis
- [ ] Tests automatiques planifiés
- [ ] Procédure de restauration testée
- [ ] Documentation à jour

## Monitoring

- [ ] Métriques exposées
- [ ] Alertes Prometheus configurées
- [ ] Dashboard créé
- [ ] Rapports automatiques générés

## Sécurité

- [ ] Permissions fichiers correctes (600/700)
- [ ] Secrets externalisés
- [ ] Logs ne contiennent pas de secrets
- [ ] Backup des configurations d'automatisation

## Documentation

- [ ] README créé
- [ ] Scripts commentés
- [ ] Procédures documentées
- [ ] Diagrammes d'architecture créés
```

## Conclusion

L'automatisation des sauvegardes transforme une tâche stressante et sujette aux oublis en un processus fiable qui fonctionne en arrière-plan. C'est un investissement initial en temps qui vous fait gagner énormément par la suite.

### Points clés à retenir

✅ **L'automatisation n'est pas optionnelle** : Les backups manuels ne suffisent pas
✅ **Commencez simple** : Un cron basique vaut mieux que rien
✅ **Itérez progressivement** : Ajoutez vérification, puis notifications, puis monitoring
✅ **Testez régulièrement** : L'automatisation ne garantit pas le fonctionnement
✅ **Surveillez** : L'automatisation sans surveillance est dangereuse
✅ **Documentez** : Même automatisé, vous devez comprendre comment ça marche

### Progression recommandée

**Semaine 1** : Velero schedules de base
**Semaine 2** : Script de vérification automatique
**Semaine 3** : Notifications simples (email)
**Semaine 4** : Nettoyage automatique
**Mois 2** : Monitoring Prometheus
**Mois 3** : Orchestration complète

### La règle d'or

**L'automatisation parfaite n'existe pas. L'automatisation qui fonctionne aujourd'hui est mieux que l'automatisation parfaite qui sera prête "bientôt".**

Commencez maintenant, améliorez continuellement.

---

**Checklist de démarrage rapide** :

- [ ] Créer un schedule Velero quotidien (10 min)
- [ ] Tester que le schedule fonctionne (5 min)
- [ ] Configurer une notification basique (15 min)
- [ ] Planifier une vérification hebdomadaire (10 min)
- [ ] Documenter la configuration (10 min)

**Total : 50 minutes pour une base solide d'automatisation.**

N'attendez pas d'avoir le système parfait. Mettez en place le minimum viable maintenant, et améliorez au fil du temps.

⏭️ [Dépannage et Maintenance](/23-depannage-et-maintenance/README.md)
