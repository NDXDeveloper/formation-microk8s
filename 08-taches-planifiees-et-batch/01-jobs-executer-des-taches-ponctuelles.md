🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 8.1 Jobs : Exécuter des Tâches Ponctuelles

## Introduction

Dans Kubernetes, nous avons jusqu'à présent principalement travaillé avec des **Deployments**, qui sont conçus pour faire tourner des applications en continu. Mais que se passe-t-il lorsque vous avez besoin d'exécuter une tâche qui doit se terminer ? Par exemple :

- Effectuer un traitement de données
- Générer un rapport
- Exécuter un script de migration de base de données
- Effectuer une sauvegarde ponctuelle
- Traiter un batch de fichiers

C'est là qu'interviennent les **Jobs**.

## Qu'est-ce qu'un Job ?

Un **Job** dans Kubernetes est un objet qui crée un ou plusieurs Pods pour exécuter une tâche jusqu'à son accomplissement réussi. Contrairement aux Deployments qui maintiennent vos Pods en vie en permanence, un Job :

1. **Lance un Pod** pour exécuter une tâche
2. **Surveille l'exécution** de cette tâche
3. **Considère le travail terminé** quand le Pod se termine avec succès (code de sortie 0)
4. **Peut réessayer** en cas d'échec
5. **Se termine** une fois la tâche accomplie

### Différence avec les autres objets Kubernetes

| Objet | Comportement | Cas d'usage |
|-------|-------------|-------------|
| **Deployment** | Maintient les Pods en vie en permanence | Applications web, APIs, services continus |
| **Job** | Exécute une tâche puis se termine | Traitements ponctuels, scripts, migrations |
| **CronJob** | Exécute des Jobs selon un planning | Tâches planifiées régulières (voir section 8.2) |

## Anatomie d'un Job

Voici la structure de base d'un manifeste Job :

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: mon-job
spec:
  template:
    spec:
      containers:
      - name: conteneur-job
        image: mon-image:latest
        command: ["mon-script.sh"]
      restartPolicy: Never
```

### Points importants à noter :

1. **apiVersion: batch/v1** - Les Jobs font partie de l'API batch de Kubernetes
2. **kind: Job** - Définit qu'il s'agit d'un Job
3. **spec.template** - Contient la spécification du Pod qui sera créé
4. **restartPolicy** - **Doit être `Never` ou `OnFailure`**, jamais `Always`

> **⚠️ Note importante** : La politique de redémarrage `Always` n'est pas compatible avec les Jobs car elle créerait une boucle infinie (le Pod redémarrerait continuellement après chaque terminaison).

## Exemples de Jobs Simples

### Exemple 1 : Job qui affiche un message

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: hello-job
spec:
  template:
    spec:
      containers:
      - name: hello
        image: busybox
        command: ["echo", "Bonjour depuis mon premier Job !"]
      restartPolicy: Never
```

Ce Job simple lance un conteneur qui affiche un message puis se termine.

### Exemple 2 : Job qui effectue un calcul

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: calcul-pi
spec:
  template:
    spec:
      containers:
      - name: pi
        image: perl:5.34
        command: ["perl", "-Mbignum=bpi", "-wle", "print bpi(2000)"]
      restartPolicy: Never
```

Ce Job calcule les 2000 premières décimales de Pi et affiche le résultat.

### Exemple 3 : Job avec traitement de données

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: traitement-fichiers
spec:
  template:
    spec:
      containers:
      - name: processeur
        image: python:3.9
        command:
        - python
        - -c
        - |
          import time
          print("Début du traitement...")
          time.sleep(5)
          print("Traitement des fichiers en cours...")
          time.sleep(5)
          print("Traitement terminé avec succès !")
      restartPolicy: Never
```

## Paramètres Importants des Jobs

### 1. Gestion des échecs avec `backoffLimit`

Par défaut, Kubernetes réessaiera jusqu'à 6 fois si votre Job échoue. Vous pouvez contrôler ce comportement :

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: job-avec-retry
spec:
  backoffLimit: 3  # Réessayer maximum 3 fois en cas d'échec
  template:
    spec:
      containers:
      - name: tache
        image: busybox
        command: ["sh", "-c", "exit 1"]  # Commande qui échoue
      restartPolicy: Never
```

- **backoffLimit: 0** - Ne jamais réessayer
- **backoffLimit: 3** - Réessayer jusqu'à 3 fois
- **backoffLimit: 6** - Valeur par défaut

### 2. Définir un délai d'expiration avec `activeDeadlineSeconds`

Pour éviter qu'un Job ne tourne indéfiniment :

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: job-avec-timeout
spec:
  activeDeadlineSeconds: 300  # Le Job sera arrêté après 5 minutes
  template:
    spec:
      containers:
      - name: longue-tache
        image: busybox
        command: ["sleep", "1000"]
      restartPolicy: Never
```

Si le Job n'est pas terminé après 300 secondes, il sera automatiquement arrêté.

### 3. Exécutions parallèles avec `parallelism` et `completions`

#### Exécuter plusieurs Pods en parallèle

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: job-parallele
spec:
  parallelism: 3        # Exécuter 3 Pods en même temps
  completions: 10       # Réussir 10 fois au total
  template:
    spec:
      containers:
      - name: worker
        image: busybox
        command: ["sh", "-c", "echo 'Traitement en cours'; sleep 5; echo 'Terminé'"]
      restartPolicy: Never
```

Dans cet exemple :
- 3 Pods s'exécutent simultanément
- Le Job ne sera considéré comme terminé qu'après 10 exécutions réussies
- Kubernetes lancera automatiquement de nouveaux Pods jusqu'à atteindre 10 complétions

#### Pourquoi utiliser le parallélisme ?

Imaginez que vous devez traiter 100 fichiers. Au lieu de les traiter un par un (ce qui prendrait beaucoup de temps), vous pouvez :
- Diviser le travail en 100 tâches
- En exécuter 10 en parallèle à la fois
- Réduire considérablement le temps de traitement total

### 4. Nettoyage automatique avec `ttlSecondsAfterFinished`

Les Jobs terminés restent dans le cluster par défaut, ce qui peut encombrer le système. Vous pouvez les nettoyer automatiquement :

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: job-auto-cleanup
spec:
  ttlSecondsAfterFinished: 3600  # Supprimer automatiquement 1h après la fin
  template:
    spec:
      containers:
      - name: tache
        image: busybox
        command: ["echo", "Job avec nettoyage automatique"]
      restartPolicy: Never
```

- Le Job et ses Pods seront automatiquement supprimés 3600 secondes (1 heure) après la complétion
- Utile pour éviter l'accumulation de Jobs terminés

## Créer et Gérer des Jobs

### Créer un Job

```bash
# Depuis un fichier YAML
kubectl apply -f mon-job.yaml

# Directement en ligne de commande
kubectl create job test-job --image=busybox -- echo "Hello World"
```

### Vérifier l'état d'un Job

```bash
# Lister tous les Jobs
kubectl get jobs

# Voir les détails d'un Job spécifique
kubectl describe job mon-job

# Voir les Pods créés par un Job
kubectl get pods --selector=job-name=mon-job
```

### Suivre les logs d'un Job

```bash
# Voir les logs du Pod créé par le Job
kubectl logs job/mon-job

# Suivre les logs en temps réel
kubectl logs -f job/mon-job
```

### Supprimer un Job

```bash
# Supprimer un Job (supprime aussi les Pods associés)
kubectl delete job mon-job

# Supprimer tous les Jobs terminés
kubectl delete jobs --field-selector status.successful=1
```

## Comprendre les États d'un Job

Un Job peut avoir plusieurs états pendant son cycle de vie :

### 1. **Active** (En cours d'exécution)
```bash
kubectl get jobs
NAME       COMPLETIONS   DURATION   AGE
mon-job    0/1           5s         5s
```
Le Job est en cours, aucun Pod n'a encore terminé avec succès.

### 2. **Succeeded** (Réussi)
```bash
kubectl get jobs
NAME       COMPLETIONS   DURATION   AGE
mon-job    1/1           10s        15s
```
Le Job s'est terminé avec succès.

### 3. **Failed** (Échoué)
```bash
kubectl get jobs
NAME       COMPLETIONS   DURATION   AGE
mon-job    0/1           20s        25s
```
Le Job a dépassé le nombre maximum de tentatives (`backoffLimit`) sans succès.

## Exemple Complet : Migration de Base de Données

Voici un exemple réaliste d'utilisation d'un Job pour exécuter une migration de base de données :

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: migration-db-v2
  labels:
    app: mon-application
    type: migration
spec:
  # Paramètres de gestion
  backoffLimit: 2                    # Réessayer 2 fois maximum
  activeDeadlineSeconds: 600         # Timeout de 10 minutes
  ttlSecondsAfterFinished: 86400     # Garder 24h après complétion

  template:
    metadata:
      labels:
        app: mon-application
        job: migration
    spec:
      restartPolicy: Never

      containers:
      - name: migration
        image: mon-app/migrator:2.0

        env:
        - name: DB_HOST
          value: "postgresql.default.svc.cluster.local"
        - name: DB_USER
          valueFrom:
            secretKeyRef:
              name: db-credentials
              key: username
        - name: DB_PASSWORD
          valueFrom:
            secretKeyRef:
              name: db-credentials
              key: password

        command:
        - /bin/sh
        - -c
        - |
          echo "Connexion à la base de données..."
          echo "Exécution des migrations..."
          python manage.py migrate
          echo "Migration terminée avec succès"
```

### Pourquoi utiliser un Job pour une migration ?

1. **Exécution unique** : Une migration doit s'exécuter une seule fois, pas en continu
2. **Gestion des erreurs** : Si la migration échoue, Kubernetes peut réessayer automatiquement
3. **Traçabilité** : Vous gardez l'historique des migrations exécutées
4. **Isolation** : La migration s'exécute dans son propre environnement isolé

## Cas d'Usage Courants

### 1. Traitement de Données en Batch
```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: traitement-mensuel
spec:
  parallelism: 5
  completions: 100
  template:
    spec:
      containers:
      - name: processeur
        image: mon-processeur:latest
        command: ["process-data.sh"]
      restartPolicy: OnFailure
```

### 2. Génération de Rapports
```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: rapport-ventes
spec:
  activeDeadlineSeconds: 1800  # 30 minutes max
  template:
    spec:
      containers:
      - name: generateur
        image: generateur-rapports:latest
        command: ["generate-report", "--type=sales", "--month=current"]
      restartPolicy: Never
```

### 3. Sauvegarde Ponctuelle
```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: backup-manuel
spec:
  template:
    spec:
      containers:
      - name: backup
        image: backup-tool:latest
        command: ["backup.sh", "--full", "--compress"]
        volumeMounts:
        - name: data
          mountPath: /data
      volumes:
      - name: data
        persistentVolumeClaim:
          claimName: mon-pvc
      restartPolicy: OnFailure
```

### 4. Test d'Infrastructure
```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: test-connectivite
spec:
  template:
    spec:
      containers:
      - name: tester
        image: busybox
        command:
        - sh
        - -c
        - |
          echo "Test de connectivité..."
          ping -c 5 google.com
          nslookup kubernetes.default
          echo "Tests terminés"
      restartPolicy: Never
```

## Bonnes Pratiques

### 1. Toujours définir `restartPolicy`
```yaml
spec:
  template:
    spec:
      restartPolicy: Never  # ou OnFailure, jamais Always
```

### 2. Utiliser des labels significatifs
```yaml
metadata:
  name: migration-db-v2-20241023
  labels:
    app: mon-app
    type: migration
    version: v2
```

### 3. Gérer les échecs avec `backoffLimit`
```yaml
spec:
  backoffLimit: 3  # Adapter selon la criticité de la tâche
```

### 4. Définir des timeouts appropriés
```yaml
spec:
  activeDeadlineSeconds: 3600  # 1 heure max
```

### 5. Nettoyer les Jobs terminés
```yaml
spec:
  ttlSecondsAfterFinished: 86400  # 24 heures
```

### 6. Utiliser des Secrets pour les informations sensibles
```yaml
env:
- name: API_KEY
  valueFrom:
    secretKeyRef:
      name: api-credentials
      key: key
```

## Différence entre `restartPolicy: Never` et `OnFailure`

### `restartPolicy: Never`
```yaml
restartPolicy: Never
```
- Si le conteneur échoue, Kubernetes crée un **nouveau Pod**
- Vous aurez plusieurs Pods en état "Error" ou "Completed"
- Utile pour le débogage (vous pouvez inspecter les Pods échoués)

### `restartPolicy: OnFailure`
```yaml
restartPolicy: OnFailure
```
- Si le conteneur échoue, Kubernetes **redémarre le conteneur dans le même Pod**
- Un seul Pod sera visible
- Plus propre mais moins facile à déboguer

**Recommandation** : Utilisez `Never` pendant le développement et `OnFailure` en production.

## Surveillance et Débogage

### Voir l'état détaillé d'un Job
```bash
kubectl describe job mon-job
```

Vous verrez des informations comme :
- Nombre de Pods actifs
- Nombre de complétions réussies
- Nombre d'échecs
- Événements liés au Job

### Voir les Pods créés par un Job
```bash
kubectl get pods -l job-name=mon-job
```

### Accéder aux logs
```bash
# Logs du Job
kubectl logs job/mon-job

# Logs d'un Pod spécifique
kubectl logs mon-job-xxxxx
```

### Déboguer un Job qui échoue
```bash
# 1. Voir les événements
kubectl describe job mon-job

# 2. Identifier le Pod qui a échoué
kubectl get pods -l job-name=mon-job

# 3. Voir les logs du Pod échoué
kubectl logs mon-job-xxxxx

# 4. Voir plus de détails sur le Pod
kubectl describe pod mon-job-xxxxx
```

## Résumé

Les **Jobs** Kubernetes sont essentiels pour exécuter des tâches ponctuelles :

✅ **Quand utiliser un Job :**
- Tâches qui doivent se terminer (migrations, traitements batch, sauvegardes)
- Tâches qui nécessitent des retry en cas d'échec
- Traitements parallèles de données

❌ **Quand NE PAS utiliser un Job :**
- Applications qui doivent tourner en continu → utilisez un Deployment
- Tâches planifiées régulières → utilisez un CronJob (section 8.2)

**Points clés à retenir :**
1. Un Job exécute une tâche jusqu'à complétion
2. `restartPolicy` doit être `Never` ou `OnFailure`
3. Utilisez `backoffLimit` pour gérer les échecs
4. Utilisez `activeDeadlineSeconds` pour éviter les Jobs qui tournent indéfiniment
5. Utilisez `ttlSecondsAfterFinished` pour nettoyer automatiquement

Dans la section suivante (8.2), nous verrons comment planifier ces Jobs de manière récurrente avec les **CronJobs**.

⏭️ [CronJobs : planifier des tâches récurrentes](/08-taches-planifiees-et-batch/02-cronjobs-planifier-des-taches-recurrentes.md)
