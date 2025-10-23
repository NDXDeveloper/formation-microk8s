🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 8.3 Gestion des Échecs et des Reprises

## Introduction

Dans les sections précédentes, nous avons découvert les **Jobs** (8.1) et les **CronJobs** (8.2), qui permettent d'exécuter des tâches ponctuelles ou planifiées. Mais dans le monde réel, les choses ne se passent pas toujours comme prévu :

- Une base de données peut être temporairement indisponible
- Une API externe peut retourner une erreur 503
- Un réseau peut être instable
- Un fichier attendu peut être absent
- Un script peut contenir un bug

C'est là que la **gestion des échecs et des reprises** devient cruciale. Kubernetes offre plusieurs mécanismes pour gérer automatiquement ces situations et rendre vos Jobs plus robustes et fiables.

## Comprendre les Échecs dans Kubernetes

### Qu'est-ce qu'un échec ?

Dans Kubernetes, un Job échoue quand :

1. **Le conteneur retourne un code de sortie non nul**
   ```bash
   exit 1  # Échec
   exit 2  # Échec
   exit 0  # Succès
   ```

2. **Le conteneur plante (crash)**
   ```bash
   # Erreur segmentation, out of memory, etc.
   ```

3. **Le Job dépasse son timeout**
   ```yaml
   activeDeadlineSeconds: 300  # Job tué après 5 minutes
   ```

### Les trois niveaux d'échec

Kubernetes gère les échecs à trois niveaux différents :

```
CronJob (niveau 3)
   │
   ├─── Job (niveau 2)
   │       │
   │       └─── Pod (niveau 1)
   │               │
   │               └─── Conteneur
```

1. **Niveau Conteneur** : Le code s'exécutant dans le conteneur échoue
2. **Niveau Pod** : `restartPolicy` détermine si le conteneur est redémarré
3. **Niveau Job** : `backoffLimit` détermine combien de Pods sont créés en cas d'échec

## Codes de Sortie : Le Langage des Échecs

### Qu'est-ce qu'un code de sortie ?

Chaque programme qui se termine retourne un **code de sortie** (exit code) :

| Code | Signification | Exemple |
|------|---------------|---------|
| **0** | Succès | La tâche s'est terminée correctement |
| **1** | Erreur générale | Script échoué pour une raison quelconque |
| **2** | Mauvaise utilisation | Arguments incorrects |
| **126** | Commande non exécutable | Problème de permissions |
| **127** | Commande introuvable | La commande n'existe pas |
| **130** | Interrompu (Ctrl+C) | Processus terminé par un signal |
| **137** | Tué (SIGKILL) | Processus tué par l'OS (souvent OOM) |
| **143** | Terminé (SIGTERM) | Arrêt gracieux demandé |

### Exemples de codes de sortie

```bash
# Succès
#!/bin/bash
echo "Tout va bien"
exit 0

# Échec
#!/bin/bash
echo "Erreur détectée"
exit 1

# Sans exit explicite, le code de sortie est celui de la dernière commande
#!/bin/bash
ls /fichier-inexistant  # Retourne 1 si le fichier n'existe pas
# Le script retourne automatiquement 1
```

### Comment Kubernetes interprète les codes de sortie

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: test-exit-codes
spec:
  template:
    spec:
      containers:
      - name: test
        image: busybox
        command:
        - sh
        - -c
        - |
          echo "Test en cours..."
          # Simuler une erreur
          exit 1
      restartPolicy: Never
```

Dans cet exemple, le conteneur retourne `exit 1`, donc Kubernetes considère que le Job a **échoué**.

## RestartPolicy : Gestion des Échecs au Niveau du Pod

Le paramètre `restartPolicy` détermine comment Kubernetes réagit quand un conteneur échoue.

### Les trois valeurs possibles

#### 1. `Never` - Ne jamais redémarrer

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: job-never-restart
spec:
  template:
    spec:
      restartPolicy: Never
      containers:
      - name: task
        image: busybox
        command: ["sh", "-c", "exit 1"]
```

**Comportement :**
- Le conteneur échoue → Le Pod reste en état "Failed"
- Kubernetes crée un **nouveau Pod** pour réessayer
- Vous aurez plusieurs Pods visibles (utile pour le débogage)

**Quand l'utiliser :**
- Pendant le développement (pour analyser les Pods échoués)
- Quand vous voulez garder l'historique de chaque tentative
- Quand chaque exécution doit être dans un environnement propre

#### 2. `OnFailure` - Redémarrer en cas d'échec

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: job-restart-on-failure
spec:
  template:
    spec:
      restartPolicy: OnFailure
      containers:
      - name: task
        image: busybox
        command: ["sh", "-c", "exit 1"]
```

**Comportement :**
- Le conteneur échoue → Kubernetes **redémarre le conteneur dans le même Pod**
- Un seul Pod visible, mais plusieurs tentatives à l'intérieur
- Plus propre mais moins facile à déboguer

**Quand l'utiliser :**
- En production (plus propre)
- Quand vous êtes sûr que votre code de gestion d'erreur est bon
- Pour économiser les ressources (pas de création de nouveaux Pods)

#### 3. `Always` - INTERDIT pour les Jobs

```yaml
# ❌ CECI NE FONCTIONNE PAS
restartPolicy: Always  # Erreur : pas compatible avec les Jobs
```

**Pourquoi ?** Un Job doit pouvoir se terminer. `Always` créerait une boucle infinie où le conteneur redémarre continuellement après chaque fin d'exécution.

### Comparaison visuelle

#### Avec `restartPolicy: Never`
```
Job
 ├── Pod-1 (Failed) ❌
 ├── Pod-2 (Failed) ❌
 └── Pod-3 (Succeeded) ✅
```

#### Avec `restartPolicy: OnFailure`
```
Job
 └── Pod-1
      ├── Tentative 1 (Failed) ❌
      ├── Tentative 2 (Failed) ❌
      └── Tentative 3 (Succeeded) ✅
```

## BackoffLimit : Gestion des Échecs au Niveau du Job

Le paramètre `backoffLimit` définit **combien de fois** Kubernetes réessaie d'exécuter un Job avant d'abandonner.

### Configuration de base

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: job-avec-retry
spec:
  backoffLimit: 3  # Réessayer 3 fois maximum
  template:
    spec:
      restartPolicy: Never
      containers:
      - name: task
        image: busybox
        command: ["sh", "-c", "exit 1"]  # Échoue toujours
```

**Comportement :**
1. Tentative 1 : Échec
2. Tentative 2 : Échec
3. Tentative 3 : Échec
4. **Le Job est marqué comme Failed** (pas de 4ème tentative)

### Valeurs courantes

| backoffLimit | Signification | Cas d'usage |
|--------------|---------------|-------------|
| **0** | Ne jamais réessayer | Tâche critique qui doit réussir du premier coup ou être inspectée manuellement |
| **1** | Réessayer une fois | Tâche assez stable, un retry devrait suffire |
| **3** | Réessayer 3 fois (défaut si non spécifié) | Bon équilibre pour la plupart des cas |
| **6** | Valeur par défaut | Bonne tolérance aux erreurs temporaires |
| **10+** | Nombreuses tentatives | Tâches susceptibles d'échouer temporairement (dépendances externes instables) |

### Exemple : Tentative de connexion à une base de données

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: test-connexion-db
spec:
  backoffLimit: 5  # Réessayer jusqu'à 5 fois
  template:
    spec:
      restartPolicy: OnFailure
      containers:
      - name: db-checker
        image: postgres:14
        command:
        - sh
        - -c
        - |
          echo "Tentative de connexion à la base de données..."

          # Tentative de connexion
          if pg_isready -h postgresql.default.svc.cluster.local -p 5432; then
            echo "✅ Connexion réussie"
            exit 0
          else
            echo "❌ Connexion échouée, nouvelle tentative..."
            exit 1
          fi
```

Dans cet exemple, si la base de données n'est pas prête, le Job réessaiera jusqu'à 5 fois.

## Le Backoff Exponentiel

Quand un Job échoue et est réessayé, Kubernetes n'essaie pas immédiatement. Il attend un certain temps qui **augmente progressivement**.

### Comment fonctionne le backoff exponentiel ?

```
Tentative 1 : Échec → Attendre 10 secondes
Tentative 2 : Échec → Attendre 20 secondes
Tentative 3 : Échec → Attendre 40 secondes
Tentative 4 : Échec → Attendre 80 secondes
Tentative 5 : Échec → Attendre 160 secondes
Tentative 6 : Échec → Attendre 300 secondes (max)
```

**Formule simplifiée :**
```
Délai = min(10s × 2^(tentatives-1), 300s)
```

### Pourquoi le backoff exponentiel ?

1. **Éviter de surcharger les systèmes** : Si une API est en panne, la bombarder de requêtes n'aidera pas
2. **Laisser le temps aux problèmes temporaires de se résoudre** : Un redémarrage de service peut prendre 1 minute
3. **Optimiser les ressources** : Pas besoin de réessayer immédiatement

### Visualisation

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: job-backoff-example
spec:
  backoffLimit: 4
  template:
    spec:
      restartPolicy: Never
      containers:
      - name: task
        image: busybox
        command:
        - sh
        - -c
        - |
          echo "Exécution à $(date)"
          echo "Cette tentative va échouer"
          exit 1
```

**Timeline d'exécution :**
```
00:00:00 - Tentative 1 → Échec
00:00:10 - Tentative 2 → Échec (10s plus tard)
00:00:30 - Tentative 3 → Échec (20s plus tard)
00:01:10 - Tentative 4 → Échec (40s plus tard)
00:02:30 - Job marqué comme Failed (80s plus tard)
```

## Stratégies de Gestion d'Erreurs

### Stratégie 1 : Retry avec Timeout

Combiner `backoffLimit` avec `activeDeadlineSeconds` pour limiter le temps total.

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: job-avec-timeout
spec:
  backoffLimit: 10                    # Jusqu'à 10 tentatives
  activeDeadlineSeconds: 600          # Mais arrêter après 10 minutes max
  template:
    spec:
      restartPolicy: OnFailure
      containers:
      - name: task
        image: mon-app:latest
        command: ["process-data.sh"]
```

**Comportement :**
- Le Job réessaiera jusqu'à 10 fois
- **MAIS** s'il n'a pas réussi après 10 minutes, il sera arrêté
- Utile pour éviter qu'un Job problématique ne tourne indéfiniment

### Stratégie 2 : Échec Rapide (Fail Fast)

Pour les tâches critiques qui doivent réussir du premier coup.

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: migration-critique
spec:
  backoffLimit: 0  # Pas de retry
  template:
    spec:
      restartPolicy: Never
      containers:
      - name: migration
        image: migration-tool:latest
        command: ["migrate-database.sh"]
```

**Quand l'utiliser :**
- Migrations de base de données (ne doivent pas être exécutées plusieurs fois)
- Transactions financières
- Opérations qui ne sont pas idempotentes

### Stratégie 3 : Retry Agressif

Pour les dépendances externes peu fiables.

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: sync-api-externe
spec:
  backoffLimit: 20  # Beaucoup de tentatives
  template:
    spec:
      restartPolicy: OnFailure
      containers:
      - name: sync
        image: sync-tool:latest
        command: ["sync-from-api.sh"]
```

**Quand l'utiliser :**
- APIs externes instables
- Services en cours de déploiement
- Opérations qui peuvent échouer temporairement

### Stratégie 4 : Retry Intelligent dans le Code

Implémenter la logique de retry directement dans votre script.

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: job-retry-intelligent
spec:
  backoffLimit: 1  # Kubernetes réessaie 1 fois
  template:
    spec:
      restartPolicy: OnFailure
      containers:
      - name: task
        image: python:3.9
        command:
        - python
        - -c
        - |
          import time
          import sys

          MAX_RETRIES = 5
          RETRY_DELAY = 2  # secondes

          for attempt in range(1, MAX_RETRIES + 1):
              print(f"Tentative {attempt}/{MAX_RETRIES}")

              try:
                  # Votre code ici
                  result = faire_quelque_chose()

                  if result:
                      print("✅ Succès !")
                      sys.exit(0)  # Succès
                  else:
                      raise Exception("Résultat invalide")

              except Exception as e:
                  print(f"❌ Échec : {e}")

                  if attempt < MAX_RETRIES:
                      print(f"Nouvelle tentative dans {RETRY_DELAY}s...")
                      time.sleep(RETRY_DELAY)
                      RETRY_DELAY *= 2  # Backoff exponentiel
                  else:
                      print("Toutes les tentatives ont échoué")
                      sys.exit(1)  # Échec
```

**Avantages :**
- Contrôle précis sur la logique de retry
- Backoff personnalisé
- Logs détaillés de chaque tentative
- Gestion d'erreurs spécifiques

## Gestion d'Erreurs dans les Scripts

### Bonnes pratiques bash

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: job-script-robuste
spec:
  backoffLimit: 3
  template:
    spec:
      restartPolicy: OnFailure
      containers:
      - name: task
        image: busybox
        command:
        - /bin/sh
        - -c
        - |
          # Arrêter en cas d'erreur
          set -e

          # Afficher les commandes exécutées (utile pour le débogage)
          set -x

          # Traiter les variables non définies comme des erreurs
          set -u

          echo "Début du traitement..."

          # Vérification préalable
          if [ ! -f /data/fichier.txt ]; then
            echo "❌ Erreur : fichier manquant"
            exit 1
          fi

          # Traitement avec vérification
          if process-data.sh; then
            echo "✅ Traitement réussi"
          else
            echo "❌ Échec du traitement"
            exit 1
          fi

          echo "Job terminé avec succès"
          exit 0
```

**Explication des options set :**
- `set -e` : Arrête le script dès qu'une commande échoue
- `set -x` : Affiche chaque commande avant de l'exécuter (aide au débogage)
- `set -u` : Traite les variables non définies comme des erreurs

### Gestion d'erreurs en Python

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: job-python-robuste
spec:
  backoffLimit: 3
  template:
    spec:
      restartPolicy: OnFailure
      containers:
      - name: task
        image: python:3.9
        command:
        - python
        - -c
        - |
          import sys
          import logging

          # Configuration des logs
          logging.basicConfig(
              level=logging.INFO,
              format='%(asctime)s - %(levelname)s - %(message)s'
          )

          def main():
              try:
                  logging.info("Début du traitement")

                  # Votre code ici
                  resultat = traiter_donnees()

                  if resultat:
                      logging.info("✅ Traitement réussi")
                      return 0  # Succès
                  else:
                      logging.error("❌ Résultat invalide")
                      return 1  # Échec

              except FileNotFoundError as e:
                  logging.error(f"❌ Fichier manquant : {e}")
                  return 1

              except ConnectionError as e:
                  logging.error(f"❌ Erreur de connexion : {e}")
                  return 1

              except Exception as e:
                  logging.error(f"❌ Erreur inattendue : {e}")
                  return 1

          if __name__ == "__main__":
              sys.exit(main())
```

### Gestion d'erreurs avec des codes spécifiques

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: job-codes-erreur
spec:
  backoffLimit: 3
  template:
    spec:
      restartPolicy: OnFailure
      containers:
      - name: task
        image: busybox
        command:
        - sh
        - -c
        - |
          # Codes de sortie personnalisés
          EXIT_SUCCESS=0
          EXIT_DB_ERROR=10
          EXIT_API_ERROR=11
          EXIT_FILE_ERROR=12
          EXIT_UNKNOWN_ERROR=99

          echo "Vérification de la base de données..."
          if ! check_database; then
            echo "❌ Erreur de base de données"
            exit $EXIT_DB_ERROR
          fi

          echo "Vérification de l'API..."
          if ! check_api; then
            echo "❌ Erreur d'API"
            exit $EXIT_API_ERROR
          fi

          echo "✅ Tout est OK"
          exit $EXIT_SUCCESS
```

## Monitoring et Débogage des Échecs

### Identifier les échecs

```bash
# Voir l'état de tous les Jobs
kubectl get jobs

# Sortie exemple :
# NAME         COMPLETIONS   DURATION   AGE
# job-ok       1/1           10s        5m
# job-failed   0/1           2m         2m

# Voir les Pods d'un Job échoué
kubectl get pods -l job-name=job-failed

# Sortie exemple :
# NAME               READY   STATUS    RESTARTS   AGE
# job-failed-abc12   0/1     Error     0          2m
# job-failed-def34   0/1     Error     0          1m
```

### Analyser les causes d'échec

#### 1. Voir les détails du Job

```bash
kubectl describe job job-failed
```

**Informations importantes à chercher :**
```
Status:
  Failed:            3      # Nombre de tentatives échouées
  Succeeded:         0

Conditions:
  Type               Status
  ----               ------
  Failed             True   # Le Job a échoué définitivement

Events:
  Type     Reason                Age   Message
  ----     ------                ----  -------
  Warning  BackoffLimitExceeded  1m    Job has reached the specified backoff limit
```

#### 2. Voir les logs des Pods échoués

```bash
# Logs du premier Pod échoué
kubectl logs job-failed-abc12

# Si le conteneur a redémarré, voir les logs précédents
kubectl logs job-failed-abc12 --previous

# Voir les logs de tous les Pods d'un Job
kubectl logs -l job-name=job-failed --all-containers=true
```

#### 3. Voir les détails d'un Pod échoué

```bash
kubectl describe pod job-failed-abc12
```

**Informations à chercher :**
```
State:          Terminated
  Reason:       Error
  Exit Code:    1          # Code de sortie du conteneur

Events:
  Type     Reason     Age   Message
  ----     ------     ----  -------
  Warning  Failed     2m    Error: exit status 1
```

### Scénarios d'échec courants

#### Scénario 1 : Out of Memory (OOM)

```bash
kubectl describe pod mon-pod
```

**Signes :**
```
State:          Terminated
  Reason:       OOMKilled      # Tué pour manque de mémoire
  Exit Code:    137
```

**Solution :**
```yaml
resources:
  limits:
    memory: "512Mi"  # Augmenter la limite
  requests:
    memory: "256Mi"
```

#### Scénario 2 : Commande introuvable

```bash
kubectl logs mon-pod
```

**Signes :**
```
/bin/sh: my-command: not found
Exit Code: 127
```

**Solution :**
- Vérifier que la commande existe dans l'image
- Vérifier le PATH
- Installer les dépendances manquantes

#### Scénario 3 : Problème de permissions

```bash
kubectl logs mon-pod
```

**Signes :**
```
Permission denied
Exit Code: 126
```

**Solution :**
```yaml
securityContext:
  runAsUser: 0  # Exécuter en tant que root (à utiliser avec précaution)
```

#### Scénario 4 : Timeout

```bash
kubectl describe job mon-job
```

**Signes :**
```
Reason:       DeadlineExceeded
Message:      Job was active longer than specified deadline
```

**Solution :**
```yaml
spec:
  activeDeadlineSeconds: 3600  # Augmenter le timeout
```

#### Scénario 5 : Dépendance externe indisponible

```bash
kubectl logs mon-pod
```

**Signes :**
```
Connection refused
Connection timeout
Exit Code: 1
```

**Solution :**
```yaml
spec:
  backoffLimit: 10  # Augmenter les tentatives
  template:
    spec:
      initContainers:  # Attendre que la dépendance soit prête
      - name: wait-for-db
        image: busybox
        command:
        - sh
        - -c
        - |
          until nc -z postgresql 5432; do
            echo "En attente de la base de données..."
            sleep 2
          done
```

## Patterns Avancés de Gestion d'Erreurs

### Pattern 1 : InitContainer pour les Prérequis

Vérifier que toutes les dépendances sont disponibles avant de démarrer le Job principal.

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: job-avec-init
spec:
  template:
    spec:
      # InitContainer : s'exécute AVANT le conteneur principal
      initContainers:
      - name: check-dependencies
        image: busybox
        command:
        - sh
        - -c
        - |
          echo "Vérification des dépendances..."

          # Attendre que la base de données soit prête
          until nc -z postgresql.default.svc.cluster.local 5432; do
            echo "⏳ En attente de PostgreSQL..."
            sleep 2
          done
          echo "✅ PostgreSQL est prêt"

          # Attendre que l'API soit prête
          until wget -q -O- http://api.default.svc.cluster.local/health; do
            echo "⏳ En attente de l'API..."
            sleep 2
          done
          echo "✅ API est prête"

          echo "✅ Toutes les dépendances sont prêtes"

      # Conteneur principal : s'exécute SEULEMENT si l'init réussit
      containers:
      - name: main-task
        image: mon-app:latest
        command: ["process-data.sh"]

      restartPolicy: OnFailure
```

**Avantages :**
- Évite des échecs inutiles si les dépendances ne sont pas prêtes
- Logs plus clairs (séparation des vérifications et du traitement)
- Meilleure gestion des ressources

### Pattern 2 : Liveness et Readiness pour Jobs Longs

Pour les Jobs qui tournent longtemps, ajouter des vérifications de santé.

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: job-long-avec-health-check
spec:
  template:
    spec:
      containers:
      - name: long-task
        image: mon-app:latest
        command: ["long-process.sh"]

        # Vérification que le processus est toujours vivant
        livenessProbe:
          exec:
            command:
            - sh
            - -c
            - "ps aux | grep -v grep | grep long-process"
          initialDelaySeconds: 30
          periodSeconds: 60
          failureThreshold: 3

      restartPolicy: OnFailure
```

### Pattern 3 : Sidecars pour Monitoring

Ajouter un conteneur pour surveiller l'exécution du Job.

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: job-avec-monitoring
spec:
  template:
    spec:
      containers:
      # Conteneur principal
      - name: main-task
        image: mon-app:latest
        command: ["process-data.sh"]
        volumeMounts:
        - name: shared-logs
          mountPath: /var/log

      # Conteneur de monitoring (sidecar)
      - name: monitor
        image: busybox
        command:
        - sh
        - -c
        - |
          while true; do
            if [ -f /var/log/error.log ]; then
              echo "⚠️  Erreurs détectées dans /var/log/error.log"
              tail -n 10 /var/log/error.log
            fi
            sleep 10
          done
        volumeMounts:
        - name: shared-logs
          mountPath: /var/log

      volumes:
      - name: shared-logs
        emptyDir: {}

      restartPolicy: OnFailure
```

### Pattern 4 : Circuit Breaker

Arrêter les tentatives si trop d'échecs consécutifs.

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: job-circuit-breaker
spec:
  backoffLimit: 10
  template:
    spec:
      containers:
      - name: task
        image: python:3.9
        command:
        - python
        - -c
        - |
          import sys
          import time

          # État du circuit breaker
          FAILURE_THRESHOLD = 3
          failure_count = 0
          circuit_open = False

          for attempt in range(1, 11):
              print(f"Tentative {attempt}")

              # Vérifier si le circuit est ouvert
              if circuit_open:
                  print("⛔ Circuit breaker ouvert - Arrêt des tentatives")
                  sys.exit(1)

              try:
                  # Votre code ici
                  result = faire_appel_api()

                  # Réinitialiser le compteur en cas de succès
                  failure_count = 0
                  print("✅ Succès")
                  sys.exit(0)

              except Exception as e:
                  failure_count += 1
                  print(f"❌ Échec {failure_count}/{FAILURE_THRESHOLD}")

                  # Ouvrir le circuit si trop d'échecs
                  if failure_count >= FAILURE_THRESHOLD:
                      circuit_open = True
                      print("⛔ Circuit breaker activé")

                  time.sleep(2)

          sys.exit(1)

      restartPolicy: Never
```

## Stratégies pour les CronJobs

### Gestion des échecs dans les CronJobs

Les CronJobs héritent de la gestion d'erreurs des Jobs, mais avec des considérations supplémentaires.

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: cronjob-robuste
spec:
  schedule: "0 2 * * *"  # Tous les jours à 2h

  # Politique de concurrence
  concurrencyPolicy: Forbid  # Ne pas lancer si le précédent tourne

  # Garder l'historique des échecs
  failedJobsHistoryLimit: 3
  successfulJobsHistoryLimit: 3

  # Tolérance aux retards de démarrage
  startingDeadlineSeconds: 300

  jobTemplate:
    spec:
      # Gestion d'erreurs du Job
      backoffLimit: 3
      activeDeadlineSeconds: 3600

      template:
        spec:
          restartPolicy: OnFailure
          containers:
          - name: backup
            image: backup-tool:latest
            command: ["backup.sh"]
```

**Bonnes pratiques pour CronJobs :**

1. **Utiliser `concurrencyPolicy: Forbid`** pour les tâches qui ne doivent pas se chevaucher
2. **Définir `startingDeadlineSeconds`** pour les tâches critiques
3. **Garder l'historique** avec `failedJobsHistoryLimit` pour le débogage
4. **Ajouter des alertes** pour surveiller les échecs consécutifs

### Alerte sur échecs multiples

```yaml
# Exemple de règle Prometheus (pour alerting - voir section 14)
groups:
- name: cronjob_alerts
  rules:
  - alert: CronJobFailing
    expr: kube_job_status_failed{job_name=~".*cronjob.*"} > 0
    for: 5m
    annotations:
      summary: "CronJob {{ $labels.job_name }} a échoué"
```

## Bonnes Pratiques de Gestion d'Erreurs

### 1. Toujours retourner un code de sortie approprié

```bash
# ❌ Mauvais
script.sh
# Le script échoue mais ne retourne rien

# ✅ Bon
if script.sh; then
  exit 0
else
  exit 1
fi
```

### 2. Utiliser des logs détaillés

```yaml
command:
- sh
- -c
- |
  set -x  # Afficher les commandes
  echo "==================================="
  echo "Début du job : $(date)"
  echo "==================================="

  # Votre code avec logs

  echo "==================================="
  echo "Fin du job : $(date)"
  echo "==================================="
```

### 3. Définir des timeouts appropriés

```yaml
spec:
  activeDeadlineSeconds: 1800  # 30 minutes
  template:
    spec:
      containers:
      - name: task
        # Timeout au niveau du conteneur aussi
        command:
        - timeout
        - "1500"  # 25 minutes (moins que le Job)
        - process-data.sh
```

### 4. Implémenter l'idempotence

Vos scripts doivent pouvoir être exécutés plusieurs fois sans problème.

```bash
# ❌ Non idempotent
echo "data" >> file.txt  # Ajoute à chaque exécution

# ✅ Idempotent
echo "data" > file.txt   # Remplace à chaque exécution

# ✅ Encore mieux
if [ ! -f file.txt ] || [ "$(cat file.txt)" != "data" ]; then
  echo "data" > file.txt
fi
```

### 5. Valider les prérequis

```bash
#!/bin/bash
set -e

# Vérifications préalables
echo "Vérification des prérequis..."

# Vérifier les variables d'environnement
if [ -z "$DATABASE_URL" ]; then
  echo "❌ Erreur : DATABASE_URL non défini"
  exit 1
fi

# Vérifier les fichiers nécessaires
if [ ! -f /config/app.conf ]; then
  echo "❌ Erreur : Configuration manquante"
  exit 1
fi

# Vérifier la connectivité
if ! nc -z database 5432; then
  echo "❌ Erreur : Base de données inaccessible"
  exit 1
fi

echo "✅ Tous les prérequis sont OK"

# Exécuter la tâche
process-data.sh
```

### 6. Utiliser des ressources appropriées

```yaml
resources:
  requests:
    memory: "256Mi"
    cpu: "250m"
  limits:
    memory: "512Mi"
    cpu: "500m"
```

Évite les OOMKill et les problèmes de performance.

### 7. Documenter les codes d'erreur

```bash
# En haut du script
# Codes de sortie :
# 0  - Succès
# 1  - Erreur générale
# 10 - Base de données inaccessible
# 11 - API externe en erreur
# 12 - Fichier manquant
# 13 - Validation échouée

# Dans les logs
echo "Code de sortie 10 : Vérifier la connexion à la base de données"
exit 10
```

### 8. Séparer les échecs récupérables des échecs critiques

```python
import sys

def main():
    try:
        process_data()
    except RecoverableError as e:
        # Échec temporaire - peut être réessayé
        print(f"Erreur récupérable : {e}")
        return 1
    except CriticalError as e:
        # Échec critique - ne pas réessayer
        print(f"Erreur critique : {e}")
        # Utiliser un code spécial pour indiquer de ne pas réessayer
        return 100
```

## Résumé

La gestion des échecs et des reprises est essentielle pour avoir des Jobs robustes et fiables dans Kubernetes.

**Points clés à retenir :**

### Au niveau du Code
✅ Toujours retourner des codes de sortie appropriés (0 = succès, non-0 = échec)
✅ Implémenter des logs détaillés
✅ Rendre les scripts idempotents
✅ Valider les prérequis avant l'exécution

### Au niveau du Pod
✅ Choisir la bonne `restartPolicy` :
  - `Never` pour le développement et le débogage
  - `OnFailure` pour la production

### Au niveau du Job
✅ Définir `backoffLimit` selon la criticité :
  - 0 pour les opérations critiques (migration)
  - 3-6 pour la plupart des cas
  - 10+ pour les dépendances externes instables
✅ Utiliser `activeDeadlineSeconds` pour éviter les Jobs infinis

### Au niveau du CronJob
✅ Utiliser `concurrencyPolicy: Forbid` pour éviter les chevauchements
✅ Définir `startingDeadlineSeconds` pour les tâches critiques
✅ Garder l'historique avec `failedJobsHistoryLimit`

### Monitoring et Débogage
✅ Utiliser `kubectl describe` pour voir les événements
✅ Utiliser `kubectl logs` pour voir les sorties
✅ Vérifier les codes de sortie dans les descriptions de Pods
✅ Mettre en place des alertes pour les échecs répétés

**Le backoff exponentiel** de Kubernetes aide à gérer les échecs temporaires en espaçant progressivement les tentatives, permettant aux systèmes de récupérer sans être surchargés.

Dans la section suivante (8.4), nous verrons des **cas d'usage pratiques** pour mettre en œuvre tous ces concepts dans des scénarios réels.

⏭️ [Cas d'usage pratiques (backups, ETL, cleanup)](/08-taches-planifiees-et-batch/04-cas-dusage-pratiques.md)
