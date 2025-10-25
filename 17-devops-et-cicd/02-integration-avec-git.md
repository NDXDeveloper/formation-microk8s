🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 17.2 Intégration avec Git

## Introduction

Dans la section précédente, nous avons découvert les principes DevOps et GitOps. Maintenant, il est temps de passer à la pratique en intégrant Git dans votre workflow Kubernetes. Git va devenir le **centre nerveux** de votre infrastructure, la source unique de vérité pour tout ce qui concerne vos déploiements.

Dans ce chapitre, nous allons voir comment organiser et structurer vos manifestes Kubernetes dans Git, adopter les bonnes pratiques et préparer le terrain pour l'automatisation complète.

---

## Rappel : Qu'est-ce que Git ?

### Définition simple

**Git** est un système de **contrôle de version distribué**. Cela signifie qu'il garde une trace de tous les changements apportés à vos fichiers au fil du temps, vous permettant de :

- Revenir à une version antérieure si nécessaire
- Voir qui a modifié quoi et quand
- Collaborer avec d'autres personnes sans écraser leur travail
- Travailler sur des fonctionnalités en parallèle (branches)

### Concepts de base

Si vous débutez avec Git, voici les concepts essentiels :

#### Repository (dépôt)
Un **repository** (ou "repo") est un dossier qui contient tous vos fichiers et l'historique complet de leurs modifications.

#### Commit
Un **commit** est un instantané de vos fichiers à un moment donné. Chaque commit a :
- Un identifiant unique (hash)
- Un message descriptif
- Un auteur et une date
- Les changements effectués

#### Branch (branche)
Une **branche** est une ligne de développement indépendante. La branche principale s'appelle généralement `main` ou `master`.

#### Remote (distant)
Un **remote** est une version du repository hébergée ailleurs (GitHub, GitLab, etc.). Il permet de synchroniser votre travail local avec un serveur central.

### Les plateformes Git

Plusieurs plateformes permettent d'héberger vos repositories Git :

- **GitHub** : la plus populaire, gratuite pour les projets publics et privés
- **GitLab** : alternative open-source avec CI/CD intégrée
- **Bitbucket** : orientée entreprise
- **Gitea** : auto-hébergeable, léger et open-source

Pour ce tutoriel, les exemples seront génériques et fonctionneront avec n'importe quelle plateforme.

---

## Pourquoi Git pour Kubernetes ?

### La traçabilité

Avec Git, vous avez un **journal d'audit complet** :

```
commit a1b2c3d
Author: Jean Dupont <jean@example.com>
Date:   Mon Oct 25 14:30:00 2025

    Augmentation de 2 à 3 replicas pour l'application web

    Raison: Pic de trafic anticipé pour le week-end
```

Ce simple commit vous indique :
- **Qui** a fait le changement
- **Quand** il a été fait
- **Quoi** a été modifié
- **Pourquoi** ce changement a été effectué

### Le rollback facilité

Si un déploiement pose problème, vous pouvez revenir en arrière en quelques secondes :

```bash
# Annuler le dernier commit
git revert HEAD

# Revenir à un commit spécifique
git revert a1b2c3d
```

### La collaboration

Git facilite le travail en équipe grâce aux **pull requests** (ou merge requests) :

1. Un développeur crée une branche pour sa modification
2. Il propose ses changements via une pull request
3. Les collègues peuvent réviser le code
4. Des discussions peuvent avoir lieu
5. Une fois approuvé, le changement est fusionné

Même seul sur votre MicroK8s, cette discipline vous aidera à mieux réfléchir à vos changements.

### La documentation implicite

L'historique Git devient une **documentation vivante** de l'évolution de votre infrastructure. Vous pouvez comprendre pourquoi telle décision a été prise en consultant les commits historiques.

---

## Structure d'un repository Kubernetes

### Organisation recommandée

Voici une structure de repository bien organisée pour vos manifestes Kubernetes :

```
mon-cluster-k8s/
├── README.md
├── .gitignore
├── apps/
│   ├── production/
│   │   ├── web-app/
│   │   │   ├── deployment.yaml
│   │   │   ├── service.yaml
│   │   │   ├── ingress.yaml
│   │   │   └── kustomization.yaml
│   │   ├── api/
│   │   │   ├── deployment.yaml
│   │   │   ├── service.yaml
│   │   │   └── configmap.yaml
│   │   └── database/
│   │       ├── statefulset.yaml
│   │       ├── service.yaml
│   │       └── pvc.yaml
│   ├── staging/
│   │   └── (même structure que production)
│   └── development/
│       └── (même structure que production)
├── infrastructure/
│   ├── namespaces/
│   │   ├── production.yaml
│   │   ├── staging.yaml
│   │   └── development.yaml
│   ├── ingress/
│   │   ├── nginx-config.yaml
│   │   └── certificates.yaml
│   ├── monitoring/
│   │   ├── prometheus/
│   │   └── grafana/
│   └── storage/
│       └── storage-classes.yaml
├── manifests/
│   └── (manifestes génériques réutilisables)
└── scripts/
    ├── deploy.sh
    └── rollback.sh
```

### Détails de la structure

#### Le fichier README.md
Un fichier **README.md** doit toujours être présent à la racine. Il contient :
- Une description du repository
- Les prérequis
- Les instructions d'installation
- La structure du projet
- Les contacts ou liens utiles

Exemple :
```markdown
# Mon Cluster MicroK8s

Ce repository contient tous les manifestes Kubernetes pour mon cluster personnel.

## Environnements
- **production** : applications en production
- **staging** : environnement de pré-production
- **development** : environnement de développement

## Déploiement
Voir le dossier `scripts/` pour les scripts de déploiement.
```

#### Le fichier .gitignore
Le fichier **.gitignore** indique à Git quels fichiers ne pas versionner :

```gitignore
# Secrets locaux (ne JAMAIS commiter de secrets réels)
*.secret
secrets.yaml
*-secret.yaml

# Fichiers temporaires
*.tmp
*.swp
*~

# Fichiers de configuration locale
.env.local
config.local.yaml

# Dossiers de build
build/
dist/

# Logs
*.log
```

⚠️ **Important** : Ne versionnez JAMAIS de vraies données sensibles (mots de passe, tokens, clés API) dans Git !

#### Le dossier apps/
Contient vos **applications** organisées par environnement :
- `production/` : environnement de production
- `staging/` : environnement de test avant production
- `development/` : environnement de développement

Chaque application a son propre sous-dossier avec tous ses manifestes.

#### Le dossier infrastructure/
Contient les ressources **d'infrastructure** :
- Namespaces
- Configurations réseau (Ingress)
- Monitoring (Prometheus, Grafana)
- Stockage (StorageClasses, PV)
- RBAC (Roles, RoleBindings)

#### Le dossier manifests/
Contient des **templates** ou manifestes réutilisables que vous pouvez adapter selon vos besoins.

#### Le dossier scripts/
Contient des **scripts utilitaires** pour faciliter les opérations courantes :
- Déploiement
- Rollback
- Backup
- Tests

---

## Stratégies de branching

### La stratégie simple : Main + Feature Branches

Pour un lab personnel ou une petite équipe, une stratégie simple suffit :

```
main (branche principale, stable)
  ↑
  |-- feature/nouvelle-app
  |-- fix/correction-ingress
  └-- update/prometheus-version
```

**Workflow** :
1. La branche `main` représente l'état actuel du cluster
2. Pour chaque changement, créez une branche spécifique
3. Testez vos changements
4. Fusionnez dans `main` quand tout fonctionne
5. Déployez depuis `main`

### La stratégie Git Flow (pour les environnements complexes)

Si vous gérez plusieurs environnements, Git Flow est plus adapté :

```
main (production)
  ↓
develop (développement)
  ↓
feature branches (nouvelles fonctionnalités)
```

**Workflow** :
1. `main` représente la production
2. `develop` représente l'environnement de développement
3. Les features sont développées dans des branches dédiées
4. Les features sont fusionnées dans `develop`
5. Quand `develop` est stable, on la fusionne dans `main`
6. Un tag est créé pour chaque release

### Environnement = Branche

Une autre approche consiste à avoir **une branche par environnement** :

```
production (branche production)
staging (branche staging)
development (branche development)
```

Les changements progressent de `development` → `staging` → `production` par des merges.

---

## Bonnes pratiques pour versionner Kubernetes

### 1. Un fichier = Une ressource

**Principe** : Chaque ressource Kubernetes dans son propre fichier.

❌ **Évitez** :
```yaml
# all-in-one.yaml
---
apiVersion: v1
kind: Service
metadata:
  name: web
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web
---
```

✅ **Préférez** :
```
web-app/
├── deployment.yaml
├── service.yaml
└── ingress.yaml
```

**Avantages** :
- Plus facile à lire et à comprendre
- Meilleure traçabilité des changements
- Possibilité d'appliquer des ressources indépendamment

### 2. Nommage cohérent

Adoptez une **convention de nommage** claire :

```
<type>-<nom>-<environnement>.yaml

Exemples :
- deployment-web-prod.yaml
- service-api-staging.yaml
- configmap-database-dev.yaml
```

Ou plus simple si vous utilisez des dossiers par environnement :
```
apps/production/web/deployment.yaml
apps/production/web/service.yaml
```

### 3. Utiliser des labels cohérents

Standardisez vos labels pour faciliter la gestion :

```yaml
metadata:
  labels:
    app: web-application
    environment: production
    version: v1.2.3
    component: frontend
    managed-by: gitops
```

### 4. Séparer la configuration du code

Ne mélangez pas configuration et déploiement :

**ConfigMaps et Secrets** séparés :
```
web-app/
├── deployment.yaml          # Définition de l'application
├── configmap.yaml          # Configuration
└── secret.yaml.template    # Template de secret (sans valeurs réelles)
```

### 5. Documenter via les métadonnées

Utilisez les `annotations` pour documenter vos ressources :

```yaml
metadata:
  annotations:
    description: "Application web principale"
    contact: "jean.dupont@example.com"
    documentation: "https://wiki.example.com/web-app"
    deployment-date: "2025-10-25"
```

### 6. Versionner les images

Utilisez toujours des **tags de version spécifiques**, jamais `latest` :

❌ **Évitez** :
```yaml
image: nginx:latest
```

✅ **Préférez** :
```yaml
image: nginx:1.25.3
```

**Pourquoi** : Le tag `latest` peut pointer vers différentes versions selon le moment, créant des comportements imprévisibles.

---

## Workflow Git pour Kubernetes

### Workflow quotidien

Voici un workflow typique pour modifier votre infrastructure :

#### 1. Cloner le repository (première fois uniquement)

```bash
git clone https://github.com/votre-username/mon-cluster-k8s.git
cd mon-cluster-k8s
```

#### 2. S'assurer d'être à jour

```bash
git checkout main
git pull origin main
```

#### 3. Créer une branche pour votre modification

```bash
git checkout -b feature/ajouter-redis
```

Nommage des branches :
- `feature/` : nouvelle fonctionnalité
- `fix/` : correction de bug
- `update/` : mise à jour
- `config/` : changement de configuration

#### 4. Faire vos modifications

Créez ou modifiez vos manifestes YAML :

```bash
mkdir -p apps/production/redis
vim apps/production/redis/deployment.yaml
vim apps/production/redis/service.yaml
```

#### 5. Vérifier vos changements

```bash
# Voir les fichiers modifiés
git status

# Voir les modifications en détail
git diff
```

#### 6. Valider vos manifestes (optionnel mais recommandé)

Avant de commiter, vérifiez que vos YAML sont valides :

```bash
# Validation basique
kubectl apply --dry-run=client -f apps/production/redis/

# Validation serveur (sans appliquer)
kubectl apply --dry-run=server -f apps/production/redis/
```

#### 7. Ajouter vos fichiers

```bash
# Ajouter tous les fichiers modifiés
git add apps/production/redis/

# Ou ajouter spécifiquement
git add apps/production/redis/deployment.yaml
```

#### 8. Commiter avec un message clair

```bash
git commit -m "Ajout du cache Redis pour l'application web

- Déploiement Redis 7.2
- Service ClusterIP
- PersistentVolumeClaim de 5Gi
- Configuration optimisée pour le caching"
```

**Anatomie d'un bon message de commit** :
- Ligne 1 : Titre court et descriptif (50 caractères max)
- Ligne 2 : Ligne vide
- Lignes suivantes : Description détaillée si nécessaire

#### 9. Pousser votre branche

```bash
git push origin feature/ajouter-redis
```

#### 10. Créer une pull request

Sur votre plateforme Git (GitHub, GitLab), créez une **pull request** de votre branche vers `main`.

Même si vous travaillez seul, cette étape vous force à :
- Réviser vos changements
- Documenter pourquoi vous faites ce changement
- Avoir un point de validation avant déploiement

#### 11. Fusionner et déployer

Une fois la PR approuvée et fusionnée :

```bash
# Revenir sur main
git checkout main

# Récupérer les derniers changements
git pull origin main

# Déployer sur le cluster
kubectl apply -f apps/production/redis/
```

### Workflow de rollback

Si un déploiement pose problème, vous pouvez revenir en arrière :

#### Option 1 : Revert du commit

```bash
# Trouver le commit problématique
git log --oneline

# Créer un commit qui annule les changements
git revert abc123

# Pousser
git push origin main

# Re-déployer
kubectl apply -f apps/production/redis/
```

#### Option 2 : Revenir à un commit précédent

```bash
# Revenir à un commit spécifique
git checkout def456 apps/production/redis/

# Commiter
git commit -m "Rollback de Redis à la version stable"

# Pousser et déployer
git push origin main
kubectl apply -f apps/production/redis/
```

---

## Gestion des secrets dans Git

### ⚠️ LA RÈGLE D'OR : Ne JAMAIS commiter de secrets

**JAMAIS** de secrets en clair dans Git :

❌ **MAUVAIS** :
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: database-credentials
stringData:
  username: admin
  password: MonMotDePasseSecret123!  # ❌ NE FAITES JAMAIS ÇA
```

### Solutions pour gérer les secrets

#### Option 1 : Fichiers template

Commitez des **templates** sans valeurs réelles :

```yaml
# secret-database.yaml.template
apiVersion: v1
kind: Secret
metadata:
  name: database-credentials
stringData:
  username: REPLACEME
  password: REPLACEME
```

Puis créez le vrai secret localement (non versionné) :
```bash
cp secret-database.yaml.template secret-database.yaml
vim secret-database.yaml  # Remplir les vraies valeurs
kubectl apply -f secret-database.yaml
rm secret-database.yaml  # Supprimer après application
```

#### Option 2 : Variables d'environnement

Utilisez des variables d'environnement pour les valeurs sensibles :

```bash
# Dans votre shell ou CI/CD
export DB_PASSWORD="MonMotDePasseSecret123!"

# Créer le secret
kubectl create secret generic database-credentials \
  --from-literal=username=admin \
  --from-literal=password=$DB_PASSWORD
```

#### Option 3 : Sealed Secrets

**Sealed Secrets** permet de chiffrer les secrets pour les versionner en toute sécurité. Nous verrons cet outil plus en détail dans les sections avancées.

#### Option 4 : External Secrets Operator

Stocker les secrets dans un gestionnaire externe (Vault, AWS Secrets Manager) et les synchroniser automatiquement.

#### Option 5 : Git-crypt ou SOPS

Outils de chiffrement de fichiers dans Git :
- **git-crypt** : chiffre automatiquement certains fichiers
- **SOPS** : chiffre les valeurs YAML tout en gardant les clés lisibles

### Bonnes pratiques secrets

1. **Utilisez .gitignore** pour exclure les fichiers sensibles
2. **Scannez votre historique** régulièrement (outils : git-secrets, truffleHog)
3. **Révoquez immédiatement** tout secret committé accidentellement
4. **Utilisez des secrets différents** par environnement
5. **Rotez les secrets** régulièrement

---

## Organiser les environnements

### Approche 1 : Dossiers séparés

```
apps/
├── production/
│   └── web/
├── staging/
│   └── web/
└── development/
    └── web/
```

**Avantages** :
- Séparation claire
- Facile à comprendre
- Différences explicites entre environnements

**Inconvénients** :
- Duplication de code
- Maintenance de plusieurs versions

### Approche 2 : Overlays avec Kustomize

Une seule base + des overlays par environnement :

```
apps/web/
├── base/
│   ├── deployment.yaml
│   ├── service.yaml
│   └── kustomization.yaml
└── overlays/
    ├── production/
    │   └── kustomization.yaml
    ├── staging/
    │   └── kustomization.yaml
    └── development/
        └── kustomization.yaml
```

Nous explorerons Kustomize plus en détail dans une section ultérieure.

### Approche 3 : Repository par environnement

Certaines organisations préfèrent avoir un repository Git distinct pour chaque environnement :

```
cluster-production/
cluster-staging/
cluster-development/
```

**Avantages** :
- Isolation totale
- Permissions Git différentes
- Pas de risque de confusion

**Inconvénients** :
- Plus de repositories à maintenir
- Synchronisation manuelle entre environnements

---

## Commandes Git essentielles pour Kubernetes

### Commandes de base

```bash
# Initialiser un nouveau repository
git init

# Cloner un repository existant
git clone <url>

# Voir l'état actuel
git status

# Voir les différences
git diff

# Ajouter des fichiers
git add <fichier>
git add .  # Tous les fichiers

# Commiter
git commit -m "Message"

# Pousser vers le remote
git push origin main

# Récupérer les changements
git pull origin main
```

### Gestion des branches

```bash
# Créer une branche
git branch feature/nouvelle-app

# Changer de branche
git checkout feature/nouvelle-app

# Créer et changer de branche en une commande
git checkout -b feature/nouvelle-app

# Lister les branches
git branch

# Supprimer une branche
git branch -d feature/nouvelle-app

# Fusionner une branche dans la branche courante
git merge feature/nouvelle-app
```

### Historique et inspection

```bash
# Voir l'historique
git log
git log --oneline  # Format compact
git log --graph    # Visualisation graphique

# Voir les changements d'un commit
git show <commit-hash>

# Voir qui a modifié quoi
git blame <fichier>

# Chercher dans l'historique
git log --grep="redis"
git log --author="Jean"
```

### Annulation et correction

```bash
# Défaire les modifications non commitées
git checkout -- <fichier>

# Annuler le dernier commit (garde les changements)
git reset --soft HEAD~1

# Annuler le dernier commit (supprime les changements)
git reset --hard HEAD~1

# Créer un commit qui annule un autre commit
git revert <commit-hash>

# Modifier le dernier commit
git commit --amend
```

### Tags (pour les versions)

```bash
# Créer un tag
git tag v1.0.0

# Créer un tag annoté (recommandé)
git tag -a v1.0.0 -m "Release version 1.0.0"

# Lister les tags
git tag

# Pousser un tag
git push origin v1.0.0

# Pousser tous les tags
git push origin --tags

# Supprimer un tag
git tag -d v1.0.0
```

---

## Intégration Git + kubectl

### Déployer depuis Git

Une fois vos manifestes dans Git, déployer devient simple :

```bash
# Cloner le repository
git clone https://github.com/votre-username/mon-cluster-k8s.git
cd mon-cluster-k8s

# Déployer une application
kubectl apply -f apps/production/web/

# Déployer tout un dossier récursivement
kubectl apply -f apps/production/ -R

# Déployer avec Kustomize
kubectl apply -k apps/web/overlays/production/
```

### Script de déploiement automatisé

Créez un script pour standardiser vos déploiements :

```bash
#!/bin/bash
# scripts/deploy.sh

set -e  # Arrêt en cas d'erreur

ENVIRONMENT=$1
APP=$2

if [ -z "$ENVIRONMENT" ] || [ -z "$APP" ]; then
    echo "Usage: ./deploy.sh <environment> <app>"
    echo "Example: ./deploy.sh production web"
    exit 1
fi

echo "🚀 Déploiement de $APP vers $ENVIRONMENT"

# Vérifier que les fichiers existent
APP_DIR="apps/$ENVIRONMENT/$APP"
if [ ! -d "$APP_DIR" ]; then
    echo "❌ Erreur: $APP_DIR n'existe pas"
    exit 1
fi

# Validation des manifestes
echo "✓ Validation des manifestes..."
kubectl apply --dry-run=server -f "$APP_DIR/"

# Déploiement
echo "✓ Déploiement en cours..."
kubectl apply -f "$APP_DIR/"

echo "✅ Déploiement terminé avec succès!"

# Vérification du statut
echo "📊 Statut du déploiement:"
kubectl get pods -l app=$APP
```

Utilisation :
```bash
./scripts/deploy.sh production web
```

---

## Collaboration et revue de code

### Pull Requests / Merge Requests

Même seul, utilisez les PR pour :

1. **Documenter** : Expliquez le pourquoi de votre changement
2. **Vérifier** : Relisez votre code avant de merger
3. **Tester** : Validez que tout fonctionne
4. **Tracer** : Historique clair des changements

**Template de Pull Request** :

```markdown
## Description
Ajout du cache Redis pour améliorer les performances de l'API

## Type de changement
- [ ] Nouveau déploiement
- [x] Mise à jour de configuration
- [ ] Correction de bug
- [ ] Autre

## Tests effectués
- [x] Validation des manifestes YAML
- [x] Déploiement sur environnement de staging
- [x] Tests de connectivité
- [x] Tests de performance

## Checklist
- [x] Documentation mise à jour
- [x] Pas de secrets en clair
- [x] Labels cohérents
- [x] Resource limits définis
```

### Protection de la branche main

Sur votre plateforme Git, configurez des protections :

1. **Require pull request before merging** : Empêche les commits directs
2. **Require status checks** : Validation automatique avant merge
3. **Require linear history** : Historique propre sans merge commits complexes

---

## Récapitulatif

### Points clés

**Git pour Kubernetes, c'est** :
- La source unique de vérité pour votre infrastructure
- Un historique complet et auditable
- La capacité de rollback facile et rapide
- Un workflow de collaboration standard

**Structure recommandée** :
- Organisation par environnement et application
- Un fichier = une ressource
- Documentation dans le README
- Scripts dans un dossier dédié

**Bonnes pratiques** :
- Messages de commit descriptifs
- Branches pour chaque changement
- Pull requests pour la validation
- Jamais de secrets en clair
- Tags pour les versions importantes

**Workflow quotidien** :
1. Pull des derniers changements
2. Créer une branche
3. Faire les modifications
4. Valider et commiter
5. Push et créer une PR
6. Merger et déployer

### Bénéfices immédiats

En adoptant Git pour vos manifestes Kubernetes :
- ✅ Vous ne perdrez plus jamais une configuration
- ✅ Vous pourrez expliquer chaque changement
- ✅ Vous pourrez revenir en arrière en cas de problème
- ✅ Vous aurez une base solide pour l'automatisation

---

## Prochaines étapes

Maintenant que vos manifestes sont versionnés dans Git, vous êtes prêt pour :

1. **Section 17.3** : Mettre en place un registry privé pour vos images Docker
2. **Section 17.4** : Configurer un pipeline CI/CD complet
3. **Section 17.5** : Déployer ArgoCD pour une synchronisation automatique Git → Cluster

Git est la **fondation** de tout workflow DevOps moderne. En maîtrisant ces concepts, vous avez franchi une étape majeure vers l'automatisation complète de vos déploiements.

**Prêt à stocker vos images Docker de manière sécurisée ?** Passons à la section 17.3 : Registry privé et gestion d'images !

⏭️ [Registry privé et gestion d'images](/17-devops-et-cicd/03-registry-prive-et-gestion-dimages.md)
