🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 8. Tâches Planifiées et Batch - Introduction

## Bienvenue dans le Monde des Tâches Automatisées

Jusqu'à présent dans cette formation, nous avons principalement travaillé avec des **applications qui tournent en continu** : serveurs web, APIs, bases de données... Ces applications sont conçues pour rester actives 24h/24 et 7j/7, répondant aux requêtes à tout moment.

Mais il existe une autre catégorie d'applications tout aussi importante : **les tâches qui s'exécutent puis se terminent**. Ce sont les tâches batch et les tâches planifiées, et ce chapitre leur est entièrement dédié.

## Qu'est-ce qu'une Tâche Batch ?

Une **tâche batch** (ou traitement par lots) est un programme qui :

1. **Démarre** à un moment donné
2. **Exécute** un travail spécifique
3. **Se termine** une fois le travail accompli
4. **Ne reste pas actif** en permanence

### Exemples de la Vie Réelle

Imaginons quelques scénarios du quotidien :

**Exemple 1 : La Sauvegarde Nocturne**
> Tous les soirs à 2h du matin, un script se réveille, sauvegarde votre base de données, compresse l'archive, la stocke dans un endroit sûr, puis s'arrête. Il n'a pas besoin de tourner en continu.

**Exemple 2 : Le Rapport Mensuel**
> Le premier jour de chaque mois, un programme génère automatiquement un rapport des ventes du mois précédent, l'envoie par email aux dirigeants, puis se termine.

**Exemple 3 : Le Nettoyage Automatique**
> Chaque dimanche, un script parcourt vos fichiers temporaires, supprime ceux qui ont plus de 30 jours, libère de l'espace disque, puis s'arrête.

Ces trois exemples sont des **tâches batch**. Elles ne tournent pas en continu comme un serveur web, mais s'exécutent à des moments précis pour accomplir un travail spécifique.

## Différence entre Applications Continues et Tâches Batch

Prenons un moment pour bien comprendre la différence fondamentale :

### Applications Continues (Ce que nous avons vu jusqu'ici)

```
┌─────────────────────────────────────────────────────────┐
│                    SERVEUR WEB                          │
│                                                         │
│  ┌─────┐  ┌─────┐  ┌─────┐  ┌─────┐  ┌─────┐  ┌─────┐   │
│  │ REQ │  │ REQ │  │ REQ │  │ REQ │  │ REQ │  │ REQ │   │
│  └─────┘  └─────┘  └─────┘  └─────┘  └─────┘  └─────┘   │
│                                                         │
│  Actif en permanence, attend et traite les requêtes     │
│  Ne se termine jamais (sauf crash ou arrêt manuel)      │
└─────────────────────────────────────────────────────────┘
     ↑                                              ↑
  Démarre                                    Tourne toujours
```

**Caractéristiques :**
- ✅ Toujours actif
- ✅ Attend des requêtes
- ✅ Traite en continu
- ✅ Ne se termine pas
- 📦 Déployé avec un **Deployment**

### Tâches Batch (Ce que nous allons découvrir)

```
┌─────────────────────────────────────────────────────────┐
│                    TÂCHE BATCH                          │
│                                                         │
│  ┌────────┐                                             │
│  │ DÉBUT  │                                             │
│  └────┬───┘                                             │
│       │                                                 │
│       ▼                                                 │
│  ┌─────────────┐                                        │
│  │  TRAITEMENT │                                        │
│  └─────┬───────┘                                        │
│        │                                                │
│        ▼                                                │
│  ┌────────┐                                             │
│  │  FIN   │  ← Se termine après le travail              │
│  └────────┘                                             │
│                                                         │
│  Démarre, exécute, se termine                           │
└─────────────────────────────────────────────────────────┘
     ↑                    ↑
  Démarre            Se termine
```

**Caractéristiques :**
- ✅ S'active à un moment précis
- ✅ Exécute un travail défini
- ✅ Se termine après l'exécution
- ✅ N'attend pas de requêtes
- 📦 Déployé avec un **Job** ou **CronJob**

## Pourquoi les Tâches Batch sont Importantes ?

Les tâches batch sont essentielles dans toute infrastructure moderne. Voici pourquoi :

### 1. Automatisation

Sans tâches batch, vous devriez tout faire manuellement :
- Vous connecter à 2h du matin pour lancer la sauvegarde
- Penser à générer le rapport chaque début de mois
- Ne pas oublier de nettoyer les fichiers temporaires

Avec les tâches batch, **tout se fait automatiquement** pendant que vous dormez !

### 2. Fiabilité

Une tâche automatisée :
- Ne prend jamais de vacances
- N'oublie jamais de s'exécuter
- Exécute toujours les mêmes étapes de la même manière
- Peut réessayer automatiquement en cas d'échec

### 3. Efficacité des Ressources

Les tâches batch utilisent des ressources **uniquement quand elles en ont besoin** :
- Un serveur web consomme des ressources 24h/24
- Une tâche batch consomme des ressources pendant 5 minutes, puis libère tout

Sur un petit cluster comme votre lab MicroK8s, c'est particulièrement important !

### 4. Planification Optimale

Vous pouvez planifier les tâches lourdes pendant les **heures creuses** :
- Sauvegardes à 2h du matin (quand personne n'utilise le système)
- Rapports générés avant l'arrivée des équipes
- Nettoyages pendant le week-end

## Les Objets Kubernetes pour les Tâches Batch

Kubernetes propose deux objets principaux pour gérer les tâches batch :

### 1. Job - La Tâche Ponctuelle

Un **Job** exécute une tâche **une seule fois**.

```
Job
 │
 ├─ Démarre
 ├─ Exécute la tâche
 ├─ Se termine
 └─ Reste dans l'historique
```

**Exemples d'utilisation :**
- Migration de base de données (une seule fois)
- Traitement d'un batch de fichiers (ponctuel)
- Import de données initial (unique)
- Génération d'un rapport ad-hoc (sur demande)

**Section dans ce chapitre :** 8.1 Jobs

### 2. CronJob - La Tâche Planifiée

Un **CronJob** crée automatiquement des Jobs selon un **planning défini**.

```
CronJob
 │
 ├─ 02:00 → Crée Job 1 → Sauvegarde → Terminé
 ├─ 02:00 (lendemain) → Crée Job 2 → Sauvegarde → Terminé
 ├─ 02:00 (surlendemain) → Crée Job 3 → Sauvegarde → Terminé
 └─ ...et ainsi de suite automatiquement
```

**Exemples d'utilisation :**
- Sauvegarde quotidienne à 2h du matin
- Rapport hebdomadaire tous les lundis
- Nettoyage mensuel le 1er de chaque mois
- Synchronisation de données toutes les 15 minutes

**Section dans ce chapitre :** 8.2 CronJobs

## Comparaison Visuelle : Deployment vs Job vs CronJob

### Deployment (Application Continue)

```
LUNDI      MARDI      MERCREDI   JEUDI      VENDREDI
  │          │          │          │          │
  ▼          ▼          ▼          ▼          ▼
████████████████████████████████████████████████
        Application web tourne en continu
████████████████████████████████████████████████
```

### Job (Tâche Ponctuelle)

```
LUNDI      MARDI      MERCREDI   JEUDI      VENDREDI
  │          │          │          │          │
  ▼          ▼          ▼          ▼          ▼
────────────────────────────█──────────────────
                        Migration DB
                        (une seule fois)
```

### CronJob (Tâche Planifiée)

```
LUNDI      MARDI      MERCREDI   JEUDI      VENDREDI
  │          │          │          │          │
  ▼          ▼          ▼          ▼          ▼
──█────────────█────────────█────────────█──────
  │            │            │            │
Backup      Backup      Backup      Backup
02:00       02:00       02:00       02:00
```

## Cas d'Usage Typiques dans un Lab Personnel

Maintenant que vous comprenez le concept, voici quelques cas d'usage parfaits pour votre lab MicroK8s :

### 🔐 Sécurité et Sauvegarde
- **Sauvegarde de base de données** tous les jours à 2h
- **Sauvegarde des configurations Kubernetes** toutes les semaines
- **Rotation des logs** pour éviter de saturer le disque
- **Snapshots des volumes** pour protection des données

### 📊 Monitoring et Reporting
- **Génération de rapports** de santé du cluster
- **Collecte de métriques** et création de statistiques
- **Envoi d'alertes** récapitulatives par email
- **Génération de dashboards** de performance

### 🧹 Maintenance et Nettoyage
- **Nettoyage des images Docker** inutilisées
- **Suppression des logs anciens** (> 30 jours)
- **Purge des fichiers temporaires** qui s'accumulent
- **Optimisation de la base de données** (VACUUM, REINDEX)

### 🔄 Synchronisation et ETL
- **Synchronisation avec des services externes** (APIs)
- **Import de données** depuis des sources externes
- **Transformation et agrégation** de données (ETL)
- **Export vers des services cloud** (S3, Drive)

### ✅ Tests et Validation
- **Vérifications de santé** périodiques des services
- **Tests de connectivité** vers les dépendances
- **Validation de l'intégrité** des sauvegardes
- **Scans de sécurité** automatisés

## Architecture des Tâches Batch dans Kubernetes

Voici comment Kubernetes gère les tâches batch :

```
┌──────────────────────────────────────────────────────┐
│                   KUBERNETES CLUSTER                 │
│                                                      │
│  ┌────────────────────────────────────────────────┐  │
│  │              SCHEDULER (Planificateur)         │  │
│  │                                                │  │
│  │  ┌─────────────┐         ┌──────────────────┐  │  │
│  │  │   CronJob   │  crée   │       Job        │  │  │
│  │  │             │ ──────> │                  │  │  │
│  │  │ schedule:   │         │  backoffLimit: 3 │  │  │
│  │  │ "0 2 * * *" │         │  completions: 1  │  │  │
│  │  └─────────────┘         └────────┬─────────┘  │  │
│  │                                   │            │  │
│  │                                   │ crée       │  │
│  │                                   ▼            │  │
│  │                          ┌─────────────────┐   │  │
│  │                          │      Pod        │   │  │
│  │                          │                 │   │  │
│  │                          │  ┌───────────┐  │   │  │
│  │                          │  │ Container │  │   │  │
│  │                          │  │  (Task)   │  │   │  │
│  │                          │  └───────────┘  │   │  │
│  │                          │                 │   │  │
│  │                          │  Status: ...    │   │  │
│  │                          └─────────────────┘   │  │
│  └────────────────────────────────────────────────┘  │
│                                                      │
└──────────────────────────────────────────────────────┘
```

**Hiérarchie :**
1. **CronJob** : Définit le planning et crée des Jobs automatiquement
2. **Job** : Gère l'exécution d'une tâche (retry, completion)
3. **Pod** : Exécute réellement la tâche dans un conteneur
4. **Container** : Contient votre code/script qui effectue le travail

## Avantages des Tâches Batch sur Kubernetes

### 1. Gestion Automatique des Échecs

Si votre tâche échoue, Kubernetes peut automatiquement :
- **Réessayer** plusieurs fois
- **Attendre** entre chaque tentative (backoff exponentiel)
- **Logger** toutes les tentatives pour le débogage
- **Notifier** en cas d'échec définitif

### 2. Isolation et Reproductibilité

Chaque tâche s'exécute dans un **environnement isolé** :
- Même image Docker à chaque fois
- Mêmes variables d'environnement
- Mêmes ressources allouées
- Pas d'interférence avec d'autres tâches

### 3. Scalabilité

Vous pouvez facilement :
- **Paralléliser** les tâches (traiter 10 fichiers en même temps)
- **Distribuer** la charge sur plusieurs nœuds
- **Augmenter** le nombre de workers selon les besoins

### 4. Observabilité

Kubernetes garde une trace de tout :
- **Historique** des exécutions
- **Logs** de chaque tentative
- **Statut** de chaque Job (réussi, échoué, en cours)
- **Métriques** de performance (durée, ressources utilisées)

### 5. Intégration Native

Les tâches batch s'intègrent naturellement avec :
- **Secrets** pour les credentials
- **ConfigMaps** pour la configuration
- **PersistentVolumes** pour le stockage
- **RBAC** pour les permissions
- **NetworkPolicies** pour la sécurité

## Ce que Vous Allez Apprendre dans ce Chapitre

Ce chapitre est organisé en 4 sections progressives :

### **8.1 Jobs : Exécuter des Tâches Ponctuelles**
Vous apprendrez à :
- Créer et gérer des Jobs
- Comprendre les paramètres importants (`backoffLimit`, `activeDeadlineSeconds`)
- Exécuter des tâches parallèles
- Déboguer les Jobs qui échouent

### **8.2 CronJobs : Planifier des Tâches Récurrentes**
Vous apprendrez à :
- Comprendre la syntaxe cron
- Créer des tâches planifiées automatiques
- Gérer les politiques de concurrence
- Maintenir l'historique des exécutions

### **8.3 Gestion des Échecs et des Reprises**
Vous apprendrez à :
- Comprendre les codes de sortie
- Configurer les stratégies de retry
- Gérer le backoff exponentiel
- Implémenter des patterns de résilience

### **8.4 Cas d'Usage Pratiques**
Vous découvrirez des exemples concrets de :
- **Backups** : Sauvegardes automatiques de bases de données
- **ETL** : Extraction, transformation et chargement de données
- **Cleanup** : Nettoyage automatique des ressources
- Et bien d'autres cas d'usage réels !

## Prérequis pour ce Chapitre

Avant de commencer, assurez-vous d'être à l'aise avec :

✅ **Les concepts Kubernetes de base** :
- Pods
- Deployments
- Services
- ConfigMaps et Secrets
- PersistentVolumes

✅ **Les commandes kubectl** :
- `kubectl apply`
- `kubectl get`
- `kubectl describe`
- `kubectl logs`
- `kubectl delete`

✅ **Les manifestes YAML** :
- Structure de base
- Spécification de conteneurs
- Variables d'environnement

Si vous avez suivi les chapitres précédents de cette formation, vous avez déjà tous ces prérequis !

## Concepts Clés à Retenir

Avant de plonger dans les détails techniques, gardez en tête ces concepts fondamentaux :

### 🎯 Concept 1 : Tâche vs Service
- **Tâche** : S'exécute puis se termine (Job)
- **Service** : Tourne en continu (Deployment)

### 🎯 Concept 2 : Ponctuel vs Planifié
- **Ponctuel** : Une seule fois, quand vous le décidez (Job)
- **Planifié** : Automatiquement selon un planning (CronJob)

### 🎯 Concept 3 : Succès vs Échec
- **Succès** : Code de sortie 0
- **Échec** : Code de sortie différent de 0

### 🎯 Concept 4 : Retry et Backoff
- Kubernetes réessaie automatiquement en cas d'échec
- Le délai entre les tentatives augmente progressivement

### 🎯 Concept 5 : Idempotence
- Une tâche doit pouvoir être exécutée plusieurs fois sans problème
- Important pour la fiabilité et les retries

## Différences avec d'Autres Systèmes

Si vous avez déjà utilisé d'autres systèmes de planification de tâches, voici comment ils se comparent :

### Cron Unix (Linux traditionnel)

| Aspect | Cron Unix | CronJob Kubernetes |
|--------|-----------|-------------------|
| **Configuration** | Fichier crontab | Manifeste YAML |
| **Isolation** | Processus système | Conteneur isolé |
| **Logs** | syslog ou fichier | `kubectl logs` |
| **Retry** | Manuel (dans le script) | Automatique (configuré) |
| **Distribution** | Machine unique | Cluster (multi-nœuds) |
| **Versioning** | Non | Oui (images Docker) |

### Jenkins / GitLab CI

| Aspect | Jenkins/GitLab | Jobs Kubernetes |
|--------|----------------|-----------------|
| **Cas d'usage** | CI/CD principalement | Tâches générales |
| **Complexité** | Interface graphique riche | Configuration YAML simple |
| **Intégration** | Plugins variés | Native Kubernetes |
| **Ressources** | Serveur dédié | Ressources cluster |
| **Déploiement** | Configuration serveur | Manifeste déclaratif |

### AWS Lambda / Cloud Functions

| Aspect | Serverless | Jobs Kubernetes |
|--------|------------|-----------------|
| **Hébergement** | Cloud provider | Votre cluster |
| **Coût** | Par exécution | Ressources cluster |
| **Durée max** | Limitée (15 min) | Configurable |
| **Contrôle** | Limité | Total |
| **Cold start** | Oui | Non (si préconfiguré) |

## Workflow Typique d'un Job Kubernetes

Voici ce qui se passe en coulisses quand vous créez un Job :

```
1. Vous créez le Job
   │
   ├─> kubectl apply -f job.yaml
   │
   ▼
2. Kubernetes crée un Pod
   │
   ├─> Télécharge l'image Docker
   ├─> Configure l'environnement
   ├─> Monte les volumes
   │
   ▼
3. Le Pod s'exécute
   │
   ├─> Lance le conteneur
   ├─> Exécute votre script/commande
   ├─> Produit des logs
   │
   ▼
4. Deux possibilités :
   │
   ├─> ✅ SUCCÈS (exit 0)
   │    │
   │    ├─> Job marqué comme "Completed"
   │    └─> Pod reste disponible pour consulter les logs
   │
   └─> ❌ ÉCHEC (exit != 0)
        │
        ├─> Kubernetes attend (backoff)
        ├─> Crée un nouveau Pod (si backoffLimit pas atteint)
        └─> Réessaye l'exécution
```

## Bonnes Pratiques Générales

Avant même de commencer à écrire vos premiers Jobs, voici quelques principes à garder en tête :

### 1. ✅ Principe de Responsabilité Unique
Chaque Job doit faire **une seule chose** et la faire bien.

**Mauvais exemple :**
```
Job "tout-faire"
├─ Sauvegarder la base de données
├─ Nettoyer les logs
├─ Envoyer un rapport
└─ Mettre à jour le cache
```

**Bon exemple :**
```
Job "backup-db"     → Sauvegarde uniquement
Job "cleanup-logs"  → Nettoyage uniquement
Job "send-report"   → Rapport uniquement
Job "update-cache"  → Cache uniquement
```

### 2. ✅ Toujours Logger
Vos scripts doivent produire des logs clairs et détaillés.

```bash
echo "Début de la sauvegarde - $(date)"
echo "Base de données : production"
echo "Destination : /backup/..."
# ... votre code ...
echo "Sauvegarde terminée avec succès - $(date)"
```

### 3. ✅ Gérer les Erreurs
Votre code doit toujours retourner le bon code de sortie.

```bash
if backup_successful; then
  echo "✅ Succès"
  exit 0  # Succès
else
  echo "❌ Échec"
  exit 1  # Échec
fi
```

### 4. ✅ Être Idempotent
Votre tâche doit pouvoir être exécutée plusieurs fois sans problème.

```bash
# Mauvais : ajoute à chaque fois
echo "data" >> file.txt

# Bon : remplace à chaque fois
echo "data" > file.txt
```

### 5. ✅ Limiter les Ressources
Définissez toujours des limites de ressources.

```yaml
resources:
  requests:
    memory: "256Mi"
    cpu: "100m"
  limits:
    memory: "512Mi"
    cpu: "500m"
```

### 6. ✅ Utiliser des Secrets
Ne jamais mettre de mots de passe en clair dans les manifestes.

```yaml
env:
- name: DB_PASSWORD
  valueFrom:
    secretKeyRef:
      name: db-credentials
      key: password
```

### 7. ✅ Documenter
Utilisez des annotations pour documenter vos Jobs.

```yaml
metadata:
  name: backup-postgresql
  annotations:
    description: "Sauvegarde quotidienne de PostgreSQL"
    contact: "ops@monentreprise.com"
    runbook: "https://wiki.monentreprise.com/runbooks/backup"
```

## Terminologie à Connaître

Avant de continuer, assurons-nous que vous connaissez ces termes :

| Terme | Définition |
|-------|------------|
| **Batch** | Traitement par lots, exécution de tâches ponctuelles |
| **Job** | Objet Kubernetes pour exécuter une tâche une fois |
| **CronJob** | Objet Kubernetes pour exécuter des tâches planifiées |
| **Backoff** | Délai d'attente entre les tentatives en cas d'échec |
| **Completion** | Nombre d'exécutions réussies requises |
| **Parallelism** | Nombre d'exécutions simultanées |
| **RestartPolicy** | Politique de redémarrage (`Never`, `OnFailure`) |
| **BackoffLimit** | Nombre maximum de tentatives en cas d'échec |
| **ActiveDeadlineSeconds** | Durée maximale d'exécution avant timeout |
| **Schedule** | Expression cron définissant quand exécuter la tâche |
| **ConcurrencyPolicy** | Politique si une nouvelle exécution démarre alors que la précédente tourne |
| **Exit Code** | Code de sortie d'un programme (0 = succès, autre = échec) |
| **Idempotence** | Capacité à être exécuté plusieurs fois avec le même résultat |
| **ETL** | Extract, Transform, Load - Processus de traitement de données |

## Prêt à Commencer ?

Vous avez maintenant une vue d'ensemble complète des tâches planifiées et batch dans Kubernetes. Vous comprenez :

✅ La différence entre applications continues et tâches batch
✅ Pourquoi les tâches automatisées sont essentielles
✅ Les objets Kubernetes dédiés (Job et CronJob)
✅ Les cas d'usage typiques dans un lab personnel
✅ Les bonnes pratiques à suivre

Dans les sections suivantes, nous allons mettre tout cela en pratique :

**→ Section 8.1** : Vous créerez vos premiers **Jobs** pour exécuter des tâches ponctuelles
**→ Section 8.2** : Vous apprendrez à planifier des tâches avec les **CronJobs**
**→ Section 8.3** : Vous maîtriserez la **gestion des échecs** et des reprises
**→ Section 8.4** : Vous découvrirez des **cas d'usage réels** (backups, ETL, cleanup)

Passons maintenant à la section 8.1 pour créer votre premier Job !

---

**Note :** Ce chapitre est conçu pour être accessible aux débutants. Si certains concepts vous semblent complexes au début, ne vous inquiétez pas : ils deviendront clairs au fur et à mesure que vous pratiquerez avec les exemples concrets des sections suivantes.

⏭️ [Jobs : exécuter des tâches ponctuelles](/08-taches-planifiees-et-batch/01-jobs-executer-des-taches-ponctuelles.md)
