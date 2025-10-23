🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 8.2 CronJobs : Planifier des Tâches Récurrentes

## Introduction

Dans la section précédente (8.1), nous avons découvert les **Jobs**, qui permettent d'exécuter des tâches ponctuelles. Mais que se passe-t-il si vous avez besoin d'exécuter une tâche **régulièrement** ? Par exemple :

- Effectuer une sauvegarde tous les jours à 2h du matin
- Générer un rapport hebdomadaire chaque lundi
- Nettoyer les fichiers temporaires toutes les heures
- Envoyer un email récapitulatif chaque fin de mois
- Synchroniser des données toutes les 15 minutes

C'est exactement le rôle des **CronJobs** !

## Qu'est-ce qu'un CronJob ?

Un **CronJob** est un objet Kubernetes qui crée automatiquement des **Jobs** selon un planning défini. C'est l'équivalent de la commande `cron` sur les systèmes Linux, mais pour Kubernetes.

### Relation entre CronJob et Job

```
CronJob
   │
   ├─── Job 1 (créé à 10:00)
   │       └─── Pod
   │
   ├─── Job 2 (créé à 11:00)
   │       └─── Pod
   │
   └─── Job 3 (créé à 12:00)
           └─── Pod
```

**Hiérarchie :**
1. Le **CronJob** définit le planning
2. À chaque horaire prévu, il crée un **Job**
3. Chaque Job crée un **Pod** qui exécute la tâche
4. Le Pod se termine une fois la tâche accomplie

### Comparaison des objets Kubernetes

| Objet | Fréquence d'exécution | Cas d'usage |
|-------|----------------------|-------------|
| **Deployment** | Continue (24/7) | Applications web, APIs |
| **Job** | Une seule fois | Migration, traitement ponctuel |
| **CronJob** | Planifiée et récurrente | Sauvegardes, rapports, nettoyage |

## Anatomie d'un CronJob

Voici la structure de base d'un manifeste CronJob :

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: mon-cronjob
spec:
  schedule: "*/5 * * * *"  # Toutes les 5 minutes
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: conteneur
            image: busybox
            command: ["echo", "Tâche planifiée"]
          restartPolicy: OnFailure
```

### Éléments clés :

1. **apiVersion: batch/v1** - Les CronJobs font partie de l'API batch
2. **kind: CronJob** - Définit qu'il s'agit d'un CronJob
3. **schedule** - L'expression cron qui définit quand exécuter le Job
4. **jobTemplate** - Le template du Job qui sera créé (identique à la section 8.1)

## Comprendre la Syntaxe Cron

La partie la plus importante d'un CronJob est l'expression `schedule`, qui utilise la syntaxe cron standard.

### Format de l'expression cron

```
 ┌───────────── minute (0 - 59)
 │ ┌───────────── heure (0 - 23)
 │ │ ┌───────────── jour du mois (1 - 31)
 │ │ │ ┌───────────── mois (1 - 12)
 │ │ │ │ ┌───────────── jour de la semaine (0 - 6) (Dimanche = 0)
 │ │ │ │ │
 * * * * *
```

### Exemples d'expressions cron courantes

| Expression | Signification |
|------------|---------------|
| `* * * * *` | Toutes les minutes |
| `*/5 * * * *` | Toutes les 5 minutes |
| `0 * * * *` | Toutes les heures (à la minute 0) |
| `0 0 * * *` | Tous les jours à minuit |
| `0 2 * * *` | Tous les jours à 2h du matin |
| `0 9 * * 1` | Tous les lundis à 9h |
| `0 0 1 * *` | Le 1er de chaque mois à minuit |
| `0 0 * * 0` | Tous les dimanches à minuit |
| `*/15 * * * *` | Toutes les 15 minutes |
| `0 */6 * * *` | Toutes les 6 heures |
| `30 8 * * 1-5` | Du lundi au vendredi à 8h30 |
| `0 9,17 * * *` | Tous les jours à 9h et à 17h |

### Caractères spéciaux

- **`*`** (astérisque) : Toutes les valeurs possibles
- **`,`** (virgule) : Liste de valeurs (ex: `1,3,5`)
- **`-`** (tiret) : Plage de valeurs (ex: `1-5` pour lundi à vendredi)
- **`/`** (slash) : Intervalle (ex: `*/10` pour toutes les 10 unités)

### Exemples détaillés

#### Toutes les 30 minutes
```yaml
schedule: "*/30 * * * *"
```
Exécution : 00:00, 00:30, 01:00, 01:30, 02:00, 02:30...

#### Tous les jours à 3h15 du matin
```yaml
schedule: "15 3 * * *"
```
Exécution : 03:15 chaque jour

#### Tous les lundis à 9h
```yaml
schedule: "0 9 * * 1"
```
Exécution : 09:00 uniquement les lundis

#### Le premier jour de chaque mois à minuit
```yaml
schedule: "0 0 1 * *"
```
Exécution : 00:00 le 1er janvier, 1er février, 1er mars...

## Exemples de CronJobs

### Exemple 1 : Sauvegarde quotidienne

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: backup-quotidien
spec:
  schedule: "0 2 * * *"  # Tous les jours à 2h du matin
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: backup
            image: backup-tool:latest
            command:
            - /bin/sh
            - -c
            - |
              echo "Début de la sauvegarde à $(date)"
              backup-script.sh
              echo "Sauvegarde terminée à $(date)"
          restartPolicy: OnFailure
```

### Exemple 2 : Nettoyage des fichiers temporaires

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: nettoyage-temp
spec:
  schedule: "0 */6 * * *"  # Toutes les 6 heures
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: cleanup
            image: busybox
            command:
            - /bin/sh
            - -c
            - |
              echo "Nettoyage des fichiers temporaires..."
              find /tmp -type f -mtime +7 -delete
              echo "Nettoyage terminé"
            volumeMounts:
            - name: temp
              mountPath: /tmp
          volumes:
          - name: temp
            hostPath:
              path: /tmp
          restartPolicy: OnFailure
```

### Exemple 3 : Rapport hebdomadaire

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: rapport-hebdomadaire
spec:
  schedule: "0 9 * * 1"  # Tous les lundis à 9h
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: rapport
            image: python:3.9
            command:
            - python
            - -c
            - |
              import datetime
              print(f"Génération du rapport hebdomadaire - {datetime.datetime.now()}")
              # Code de génération du rapport ici
              print("Rapport généré et envoyé avec succès")
          restartPolicy: OnFailure
```

### Exemple 4 : Synchronisation de données

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: sync-donnees
spec:
  schedule: "*/15 * * * *"  # Toutes les 15 minutes
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: sync
            image: rsync:latest
            command: ["rsync", "-avz", "source/", "destination/"]
            env:
            - name: SOURCE_URL
              value: "https://api.example.com/data"
          restartPolicy: OnFailure
```

## Paramètres Importants des CronJobs

### 1. Politique d'exécution concurrente : `concurrencyPolicy`

Ce paramètre contrôle ce qui se passe si une nouvelle exécution doit démarrer alors que la précédente n'est pas terminée.

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: mon-cronjob
spec:
  schedule: "*/5 * * * *"
  concurrencyPolicy: Forbid  # Allow, Forbid, ou Replace
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: tache
            image: busybox
            command: ["sleep", "600"]  # Tâche longue de 10 minutes
          restartPolicy: OnFailure
```

**Valeurs possibles :**

| Valeur | Comportement | Quand l'utiliser |
|--------|--------------|------------------|
| **Allow** (défaut) | Autorise plusieurs Jobs à s'exécuter en parallèle | Quand les tâches sont indépendantes et peuvent tourner en parallèle |
| **Forbid** | Saute la nouvelle exécution si la précédente tourne encore | Quand l'exécution simultanée pourrait causer des problèmes (corruption de données, conflits) |
| **Replace** | Arrête la tâche en cours et lance la nouvelle | Quand seule la dernière exécution compte |

#### Exemple avec `Allow`
```yaml
concurrencyPolicy: Allow
schedule: "*/5 * * * *"
# Durée d'exécution : 10 minutes
# Résultat : Plusieurs Jobs tourneront en parallèle
```

#### Exemple avec `Forbid`
```yaml
concurrencyPolicy: Forbid
schedule: "*/5 * * * *"
# Durée d'exécution : 10 minutes
# Résultat : Si un Job tourne déjà, le nouveau est ignoré
```

#### Exemple avec `Replace`
```yaml
concurrencyPolicy: Replace
schedule: "*/5 * * * *"
# Durée d'exécution : 10 minutes
# Résultat : Le Job en cours est arrêté, le nouveau démarre
```

### 2. Historique des Jobs : `successfulJobsHistoryLimit` et `failedJobsHistoryLimit`

Ces paramètres contrôlent combien de Jobs terminés sont conservés dans l'historique.

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: mon-cronjob
spec:
  schedule: "0 * * * *"
  successfulJobsHistoryLimit: 3  # Garder les 3 derniers Jobs réussis
  failedJobsHistoryLimit: 1      # Garder le dernier Job échoué
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: tache
            image: busybox
            command: ["echo", "Tâche exécutée"]
          restartPolicy: OnFailure
```

**Valeurs par défaut :**
- `successfulJobsHistoryLimit: 3`
- `failedJobsHistoryLimit: 1`

**Pourquoi c'est important :**
- Évite l'accumulation de Jobs terminés
- Permet de garder un historique pour le débogage
- Libère des ressources dans le cluster

**Recommandations :**
- Gardez au moins 1-3 Jobs réussis pour vérifier que tout fonctionne
- Gardez 1-2 Jobs échoués pour le débogage
- Utilisez 0 si vous ne voulez pas d'historique (non recommandé)

### 3. Date de début : `startingDeadlineSeconds`

Définit combien de temps Kubernetes peut "rattraper" une exécution manquée.

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: mon-cronjob
spec:
  schedule: "0 * * * *"
  startingDeadlineSeconds: 300  # 5 minutes
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: tache
            image: busybox
            command: ["echo", "Tâche planifiée"]
          restartPolicy: OnFailure
```

**Scénario :**
- Le CronJob devait s'exécuter à 10:00
- Le nœud Kubernetes était en maintenance à 10:00
- Le nœud revient en ligne à 10:03
- Avec `startingDeadlineSeconds: 300`, le Job sera quand même exécuté
- Sans ce paramètre ou avec une valeur trop courte, le Job serait ignoré

**Cas d'usage :**
- Utile pour les tâches critiques qui doivent s'exécuter même en cas de problème temporaire
- Permet une certaine tolérance aux pannes

### 4. Suspension d'un CronJob : `suspend`

Permet de mettre un CronJob en pause sans le supprimer.

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: mon-cronjob
spec:
  schedule: "*/5 * * * *"
  suspend: true  # Le CronJob ne créera pas de nouveaux Jobs
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: tache
            image: busybox
            command: ["echo", "Tâche planifiée"]
          restartPolicy: OnFailure
```

**Quand utiliser `suspend` :**
- Pendant une maintenance planifiée
- Pour déboguer un problème sans supprimer le CronJob
- Pour désactiver temporairement une tâche

**Commande pour suspendre/reprendre :**
```bash
# Suspendre un CronJob
kubectl patch cronjob mon-cronjob -p '{"spec":{"suspend":true}}'

# Reprendre un CronJob
kubectl patch cronjob mon-cronjob -p '{"spec":{"suspend":false}}'
```

## Exemple Complet : Sauvegarde de Base de Données

Voici un exemple réaliste et complet de CronJob pour effectuer des sauvegardes de base de données :

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: backup-database
  labels:
    app: mon-application
    type: backup
spec:
  # Exécution tous les jours à 3h du matin
  schedule: "0 3 * * *"

  # Ne jamais exécuter en parallèle (évite la corruption)
  concurrencyPolicy: Forbid

  # Garder l'historique des 7 dernières sauvegardes réussies
  successfulJobsHistoryLimit: 7

  # Garder l'historique des 2 derniers échecs
  failedJobsHistoryLimit: 2

  # Tolérer un délai de 10 minutes si le système est indisponible
  startingDeadlineSeconds: 600

  jobTemplate:
    metadata:
      labels:
        app: mon-application
        job-type: backup
    spec:
      # Réessayer jusqu'à 2 fois en cas d'échec
      backoffLimit: 2

      # Timeout de 30 minutes
      activeDeadlineSeconds: 1800

      template:
        spec:
          restartPolicy: OnFailure

          containers:
          - name: backup
            image: postgres:14

            env:
            # Configuration de la base de données depuis un Secret
            - name: PGHOST
              value: "postgresql.default.svc.cluster.local"
            - name: PGDATABASE
              value: "production"
            - name: PGUSER
              valueFrom:
                secretKeyRef:
                  name: db-credentials
                  key: username
            - name: PGPASSWORD
              valueFrom:
                secretKeyRef:
                  name: db-credentials
                  key: password

            command:
            - /bin/bash
            - -c
            - |
              set -e

              # Nom du fichier de sauvegarde avec timestamp
              BACKUP_FILE="backup-$(date +%Y%m%d-%H%M%S).sql.gz"

              echo "=========================================="
              echo "Début de la sauvegarde : $(date)"
              echo "Fichier : $BACKUP_FILE"
              echo "=========================================="

              # Création de la sauvegarde
              pg_dump --clean --if-exists | gzip > /backup/$BACKUP_FILE

              # Vérification de la sauvegarde
              if [ -f /backup/$BACKUP_FILE ]; then
                SIZE=$(du -h /backup/$BACKUP_FILE | cut -f1)
                echo "✓ Sauvegarde réussie : $BACKUP_FILE ($SIZE)"
              else
                echo "✗ Erreur : fichier de sauvegarde non créé"
                exit 1
              fi

              # Nettoyage des anciennes sauvegardes (garder 30 jours)
              echo "Nettoyage des anciennes sauvegardes..."
              find /backup -name "backup-*.sql.gz" -mtime +30 -delete

              echo "=========================================="
              echo "Sauvegarde terminée : $(date)"
              echo "=========================================="

            volumeMounts:
            - name: backup-storage
              mountPath: /backup

          volumes:
          - name: backup-storage
            persistentVolumeClaim:
              claimName: backup-pvc
```

### Pourquoi cet exemple est-il complet ?

1. **Planning clair** : Sauvegarde quotidienne à 3h du matin
2. **Sécurité** : `concurrencyPolicy: Forbid` évite les sauvegardes simultanées
3. **Historique** : Garde 7 sauvegardes réussies et 2 échecs pour le débogage
4. **Résilience** :
   - Réessaye jusqu'à 2 fois si échec
   - Timeout de 30 minutes
   - Tolérance de 10 minutes pour le démarrage
5. **Gestion des données sensibles** : Utilise des Secrets pour les credentials
6. **Stockage persistant** : Sauvegarde sur un PVC
7. **Nettoyage automatique** : Supprime les sauvegardes de plus de 30 jours
8. **Logs détaillés** : Permet de suivre l'exécution facilement

## Créer et Gérer des CronJobs

### Créer un CronJob

```bash
# Depuis un fichier YAML
kubectl apply -f mon-cronjob.yaml

# Créer un CronJob simple en ligne de commande
kubectl create cronjob test-cron --image=busybox --schedule="*/5 * * * *" -- echo "Hello"
```

### Lister les CronJobs

```bash
# Lister tous les CronJobs
kubectl get cronjobs

# Lister avec plus de détails
kubectl get cronjobs -o wide

# Format de sortie :
# NAME          SCHEDULE      SUSPEND   ACTIVE   LAST SCHEDULE   AGE
# mon-cronjob   */5 * * * *   False     0        2m              10m
```

### Voir les détails d'un CronJob

```bash
kubectl describe cronjob mon-cronjob
```

Vous verrez :
- Le planning (schedule)
- Les paramètres de concurrence
- L'historique des Jobs créés
- Les événements récents

### Voir les Jobs créés par un CronJob

```bash
# Lister les Jobs d'un CronJob spécifique
kubectl get jobs -l job-name=mon-cronjob

# Ou avec un filtre sur le nom
kubectl get jobs | grep mon-cronjob
```

### Voir les Pods créés par un CronJob

```bash
# Lister les Pods
kubectl get pods -l job-name=mon-cronjob-<timestamp>

# Voir les logs du dernier Job
kubectl logs job/mon-cronjob-<timestamp>
```

### Suspendre et reprendre un CronJob

```bash
# Suspendre (arrêter les nouvelles exécutions)
kubectl patch cronjob mon-cronjob -p '{"spec":{"suspend":true}}'

# Reprendre
kubectl patch cronjob mon-cronjob -p '{"spec":{"suspend":false}}'
```

### Déclencher manuellement une exécution

```bash
# Créer un Job à partir d'un CronJob
kubectl create job --from=cronjob/mon-cronjob execution-manuelle
```

Cette commande est très utile pour :
- Tester un CronJob sans attendre le planning
- Exécuter une sauvegarde supplémentaire
- Déboguer un problème

### Modifier un CronJob existant

```bash
# Modifier interactivement
kubectl edit cronjob mon-cronjob

# Ou appliquer les changements depuis un fichier
kubectl apply -f mon-cronjob-modifie.yaml
```

### Supprimer un CronJob

```bash
# Supprimer un CronJob (supprime aussi tous les Jobs et Pods associés)
kubectl delete cronjob mon-cronjob

# Supprimer plusieurs CronJobs
kubectl delete cronjob cronjob1 cronjob2

# Supprimer tous les CronJobs d'un namespace
kubectl delete cronjobs --all
```

## Surveillance et Débogage

### Vérifier qu'un CronJob fonctionne

```bash
# 1. Vérifier que le CronJob existe et n'est pas suspendu
kubectl get cronjob mon-cronjob

# 2. Voir la dernière exécution
kubectl get cronjob mon-cronjob -o jsonpath='{.status.lastScheduleTime}'

# 3. Voir les Jobs récents
kubectl get jobs | grep mon-cronjob

# 4. Vérifier les événements
kubectl describe cronjob mon-cronjob
```

### Déboguer un CronJob qui ne s'exécute pas

Si votre CronJob ne crée pas de Jobs, vérifiez :

1. **Le CronJob n'est pas suspendu**
```bash
kubectl get cronjob mon-cronjob -o jsonpath='{.spec.suspend}'
# Doit retourner "false" ou rien
```

2. **L'expression cron est valide**
```bash
kubectl describe cronjob mon-cronjob
# Regardez la section "Schedule"
```

3. **Vérifiez les événements**
```bash
kubectl describe cronjob mon-cronjob
# Regardez la section "Events"
```

4. **Vérifiez l'heure du système**
```bash
# L'heure dans le cluster Kubernetes
kubectl run test-time --image=busybox --restart=Never --rm -it -- date

# Compare avec votre heure locale
date
```

> **Important** : Kubernetes utilise l'heure UTC pour les CronJobs. Si vous voulez une exécution à 10h heure locale (Paris, UTC+1), vous devez mettre `9` dans l'expression cron !

### Déboguer un Job créé par un CronJob qui échoue

```bash
# 1. Trouver le Job qui a échoué
kubectl get jobs | grep mon-cronjob

# 2. Voir les détails du Job
kubectl describe job mon-cronjob-xxxxx

# 3. Trouver le Pod qui a échoué
kubectl get pods -l job-name=mon-cronjob-xxxxx

# 4. Voir les logs du Pod
kubectl logs mon-cronjob-xxxxx-yyyyy

# 5. Voir plus de détails sur le Pod
kubectl describe pod mon-cronjob-xxxxx-yyyyy
```

### Logs et monitoring

```bash
# Voir les logs du dernier Job créé par le CronJob
kubectl logs -l job-name=$(kubectl get jobs -l cronjob=mon-cronjob --sort-by=.metadata.creationTimestamp -o jsonpath='{.items[-1].metadata.name}')

# Suivre les logs en temps réel
kubectl logs -f -l job-name=mon-cronjob-xxxxx
```

## Cas d'Usage Pratiques

### 1. Nettoyage des logs applicatifs

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: cleanup-logs
spec:
  schedule: "0 0 * * *"  # Tous les jours à minuit
  concurrencyPolicy: Forbid
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: cleanup
            image: busybox
            command:
            - sh
            - -c
            - |
              echo "Nettoyage des logs de plus de 7 jours..."
              find /var/log/app -name "*.log" -mtime +7 -delete
              echo "Nettoyage terminé"
            volumeMounts:
            - name: logs
              mountPath: /var/log/app
          volumes:
          - name: logs
            persistentVolumeClaim:
              claimName: app-logs-pvc
          restartPolicy: OnFailure
```

### 2. Génération de rapports mensuels

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: rapport-mensuel
spec:
  schedule: "0 8 1 * *"  # Le 1er de chaque mois à 8h
  concurrencyPolicy: Forbid
  successfulJobsHistoryLimit: 12  # Garder 1 an d'historique
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: rapport
            image: reporting-tool:latest
            command: ["generate-report", "--type=monthly", "--format=pdf"]
            env:
            - name: EMAIL_TO
              value: "direction@entreprise.com"
          restartPolicy: OnFailure
```

### 3. Synchronisation de données externes

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: sync-api-externe
spec:
  schedule: "*/30 * * * *"  # Toutes les 30 minutes
  concurrencyPolicy: Replace  # Remplacer l'ancienne si elle tourne encore
  startingDeadlineSeconds: 300
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: sync
            image: python:3.9
            command:
            - python
            - -c
            - |
              import requests
              import json

              print("Récupération des données...")
              response = requests.get('https://api.example.com/data')
              data = response.json()

              print(f"Données récupérées : {len(data)} éléments")
              # Traitement des données ici

              print("Synchronisation terminée")
            env:
            - name: API_KEY
              valueFrom:
                secretKeyRef:
                  name: api-credentials
                  key: key
          restartPolicy: OnFailure
```

### 4. Vérifications de santé périodiques

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: health-check
spec:
  schedule: "*/10 * * * *"  # Toutes les 10 minutes
  concurrencyPolicy: Allow  # Peut tourner en parallèle
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: checker
            image: curlimages/curl:latest
            command:
            - sh
            - -c
            - |
              echo "Vérification des services..."

              # Vérifier le service web
              if curl -f http://mon-service.default.svc.cluster.local/health; then
                echo "✓ Service web OK"
              else
                echo "✗ Service web en échec"
                exit 1
              fi

              # Vérifier la base de données
              if nc -zv postgresql.default.svc.cluster.local 5432; then
                echo "✓ Base de données OK"
              else
                echo "✗ Base de données inaccessible"
                exit 1
              fi

              echo "Tous les services sont opérationnels"
          restartPolicy: OnFailure
```

### 5. Optimisation de base de données

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: optimize-database
spec:
  schedule: "0 4 * * 0"  # Tous les dimanches à 4h du matin
  concurrencyPolicy: Forbid
  jobTemplate:
    spec:
      activeDeadlineSeconds: 7200  # 2 heures max
      template:
        spec:
          containers:
          - name: optimize
            image: postgres:14
            command:
            - sh
            - -c
            - |
              echo "Optimisation de la base de données..."
              psql -c "VACUUM ANALYZE;"
              psql -c "REINDEX DATABASE production;"
              echo "Optimisation terminée"
            env:
            - name: PGHOST
              value: "postgresql.default.svc.cluster.local"
            - name: PGDATABASE
              value: "production"
            - name: PGUSER
              valueFrom:
                secretKeyRef:
                  name: db-credentials
                  key: username
            - name: PGPASSWORD
              valueFrom:
                secretKeyRef:
                  name: db-credentials
                  key: password
          restartPolicy: OnFailure
```

## Bonnes Pratiques

### 1. Toujours définir `concurrencyPolicy`

```yaml
spec:
  concurrencyPolicy: Forbid  # ou Allow ou Replace selon le cas
```

Réfléchissez à ce qui doit se passer si une exécution prend plus de temps que prévu.

### 2. Gérer l'historique des Jobs

```yaml
spec:
  successfulJobsHistoryLimit: 3
  failedJobsHistoryLimit: 1
```

Évite l'accumulation de Jobs dans le cluster tout en gardant assez d'historique pour le débogage.

### 3. Définir des limites de ressources

```yaml
jobTemplate:
  spec:
    template:
      spec:
        containers:
        - name: tache
          image: mon-image
          resources:
            requests:
              memory: "256Mi"
              cpu: "250m"
            limits:
              memory: "512Mi"
              cpu: "500m"
```

Empêche un Job de consommer toutes les ressources du cluster.

### 4. Utiliser des timeouts

```yaml
jobTemplate:
  spec:
    activeDeadlineSeconds: 3600  # 1 heure max
```

Évite qu'un Job ne tourne indéfiniment en cas de problème.

### 5. Utiliser des labels significatifs

```yaml
metadata:
  name: backup-production-db
  labels:
    app: production
    type: backup
    frequency: daily
```

Facilite la gestion et le filtrage des CronJobs.

### 6. Documenter le CronJob

```yaml
metadata:
  name: backup-database
  annotations:
    description: "Sauvegarde quotidienne de la base de données de production"
    contact: "equipe-ops@entreprise.com"
    runbook: "https://wiki.entreprise.com/runbooks/backup-db"
```

### 7. Utiliser `startingDeadlineSeconds` pour les tâches critiques

```yaml
spec:
  startingDeadlineSeconds: 600  # 10 minutes de tolérance
```

### 8. Tester avant de déployer en production

```bash
# Créer un Job de test à partir du CronJob
kubectl create job --from=cronjob/mon-cronjob test-manuel

# Vérifier les logs
kubectl logs job/test-manuel
```

### 9. Surveiller les échecs

Mettez en place des alertes pour être notifié quand :
- Un CronJob ne s'exécute pas au moment prévu
- Un Job échoue plusieurs fois de suite
- Un Job prend beaucoup plus de temps que d'habitude

### 10. Utiliser des Secrets pour les informations sensibles

```yaml
env:
- name: API_TOKEN
  valueFrom:
    secretKeyRef:
      name: api-credentials
      key: token
```

Ne jamais mettre de mots de passe ou tokens en clair dans les manifestes.

## Pièges Courants et Solutions

### Piège 1 : Confusion avec les fuseaux horaires

**Problème** : Vous voulez une exécution à 10h heure de Paris, mais le Job s'exécute à 11h.

**Solution** : Kubernetes utilise UTC. Paris est UTC+1 (ou UTC+2 en été), donc :
```yaml
schedule: "0 9 * * *"  # Pour 10h à Paris (UTC+1)
```

### Piège 2 : CronJobs qui s'accumulent

**Problème** : De nombreux Jobs terminés s'accumulent dans le cluster.

**Solution** : Configurer les limites d'historique :
```yaml
successfulJobsHistoryLimit: 3
failedJobsHistoryLimit: 1
```

### Piège 3 : Exécutions manquées

**Problème** : Un CronJob n'a pas créé de Job à l'heure prévue.

**Solutions possibles** :
- Le cluster était arrêté : utiliser `startingDeadlineSeconds`
- Le CronJob est suspendu : vérifier `suspend: false`
- L'expression cron est incorrecte : valider la syntaxe

### Piège 4 : Jobs qui ne se terminent jamais

**Problème** : Un Job continue de tourner indéfiniment.

**Solution** : Définir un timeout :
```yaml
jobTemplate:
  spec:
    activeDeadlineSeconds: 3600
```

### Piège 5 : Multiples exécutions simultanées non voulues

**Problème** : Un Job long crée des exécutions qui se chevauchent.

**Solution** : Utiliser `concurrencyPolicy: Forbid` :
```yaml
concurrencyPolicy: Forbid
```

## Différences avec cron Unix

Si vous connaissez déjà `cron` sur Linux, voici les différences principales :

| Aspect | cron Unix | CronJob Kubernetes |
|--------|-----------|-------------------|
| **Syntaxe** | 5 champs (minute à jour de semaine) | 5 champs (identique) |
| **Exécution** | Sur une machine spécifique | Dans un Pod (n'importe quel nœud) |
| **Isolation** | Processus sur l'OS | Conteneur isolé |
| **Gestion des erreurs** | Dépend du script | Retry automatique configurable |
| **Logs** | Dans syslog ou fichier | kubectl logs |
| **Surveillance** | Scripts personnalisés | API Kubernetes native |
| **Persistance** | Configuration locale | Manifeste YAML versionné |

## Outils pour Générer des Expressions Cron

Si vous avez du mal avec la syntaxe cron, voici quelques outils utiles :

1. **[crontab.guru](https://crontab.guru/)** - Éditeur visuel d'expressions cron en ligne
2. **[crontab-generator.org](https://crontab-generator.org/)** - Générateur d'expressions cron
3. **Commande kubectl** pour tester :
```bash
# Créer un CronJob de test
kubectl create cronjob test --image=busybox --schedule="*/5 * * * *" -- date

# Observer les exécutions
kubectl get cronjobs --watch
```

## Résumé

Les **CronJobs** Kubernetes permettent d'automatiser l'exécution de tâches récurrentes :

✅ **Quand utiliser un CronJob :**
- Sauvegardes régulières
- Génération de rapports périodiques
- Nettoyage automatique de fichiers
- Synchronisation de données
- Tâches de maintenance planifiées

❌ **Quand NE PAS utiliser un CronJob :**
- Tâche unique → utilisez un Job (section 8.1)
- Application continue → utilisez un Deployment
- Besoin de réactivité immédiate → utilisez un événement ou webhook

**Points clés à retenir :**

1. Un CronJob crée des Jobs selon un planning défini
2. La syntaxe cron utilise 5 champs : minute, heure, jour, mois, jour de semaine
3. `concurrencyPolicy` contrôle les exécutions simultanées
4. Kubernetes utilise le fuseau horaire UTC
5. Limitez l'historique avec `successfulJobsHistoryLimit` et `failedJobsHistoryLimit`
6. Utilisez `startingDeadlineSeconds` pour les tâches critiques
7. Testez toujours avec `kubectl create job --from=cronjob/...`

**Prochaines étapes :**
- Dans la section 9, nous verrons comment exposer nos applications avec **MetalLB** pour le load balancing
- Continuez à pratiquer en créant des CronJobs pour automatiser vos tâches courantes

---

**Ressources supplémentaires :**
- [Documentation officielle Kubernetes - CronJobs](https://kubernetes.io/docs/concepts/workloads/controllers/cron-jobs/)
- [crontab.guru](https://crontab.guru/) - Éditeur d'expressions cron

⏭️ [Gestion des échecs et des reprises](/08-taches-planifiees-et-batch/03-gestion-des-echecs-et-des-reprises.md)
