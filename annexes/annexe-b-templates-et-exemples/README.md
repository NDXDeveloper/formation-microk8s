🔝 Retour au [Sommaire](/SOMMAIRE.md)

# Annexe B : Templates et Exemples

## Introduction

Bienvenue dans l'Annexe B de la formation MicroK8s ! Cette annexe est votre **bibliothèque de référence pratique** contenant des templates et des exemples prêts à l'emploi pour vos déploiements Kubernetes.

### Pourquoi cette annexe ?

Lorsque vous travaillez avec Kubernetes et MicroK8s, vous allez constamment créer et modifier des fichiers de configuration. Plutôt que de partir de zéro à chaque fois, cette annexe vous fournit :

- 📋 **Des templates éprouvés** que vous pouvez copier et adapter
- 🎯 **Des exemples concrets** basés sur des cas d'usage réels
- ⚡ **Un gain de temps considérable** dans vos développements
- ✅ **Des bonnes pratiques intégrées** dans chaque exemple
- 🎓 **Une référence pédagogique** pour apprendre par l'exemple

### Comment utiliser cette annexe ?

Cette annexe est conçue pour être utilisée comme une **référence pratique** :

1. **En tant que débutant** : Étudiez les exemples pour comprendre comment les différentes ressources fonctionnent ensemble
2. **En développement** : Copiez les templates et adaptez-les à vos besoins spécifiques
3. **En production** : Utilisez les exemples avancés comme base pour vos déploiements
4. **Pour apprendre** : Analysez les commentaires et les explications pour approfondir vos connaissances

**💡 Conseil** : Gardez cette annexe à portée de main ! Elle vous servira de référence tout au long de votre travail avec MicroK8s.

---

## Vue d'ensemble du contenu

Cette annexe est organisée en trois grandes sections, chacune couvrant un aspect essentiel du travail avec Kubernetes et MicroK8s.

### 1. Templates de Manifestes YAML

Les manifestes YAML sont les fichiers de configuration qui décrivent vos ressources Kubernetes. Cette section vous fournit des templates pour :

**Ressources de base** :
- **Pods** : L'unité de déploiement la plus simple
- **Deployments** : Gestion de réplicas et mises à jour progressives
- **Services** : Exposition réseau de vos applications (ClusterIP, NodePort, LoadBalancer)
- **ConfigMaps** : Configuration non sensible
- **Secrets** : Données sensibles (mots de passe, clés)
- **Namespaces** : Isolation logique des ressources

**Stockage** :
- **PersistentVolumeClaims (PVC)** : Demandes de stockage persistant
- **StatefulSets** : Applications avec état (bases de données)

**Tâches et jobs** :
- **Jobs** : Tâches ponctuelles
- **CronJobs** : Tâches planifiées

**Réseau et routage** :
- **Ingress** : Routage HTTP/HTTPS externe
- **NetworkPolicies** : Contrôle du trafic réseau

**Scaling et ressources** :
- **HorizontalPodAutoscaler (HPA)** : Autoscaling automatique
- **ResourceQuotas** : Limites de ressources par namespace
- **LimitRanges** : Contraintes par défaut

**Sécurité et contrôle d'accès** :
- **ServiceAccounts** : Identités pour les Pods
- **Roles et RoleBindings (RBAC)** : Contrôle d'accès

**Avancé** :
- **DaemonSets** : Un Pod par nœud
- **Applications complètes** : Exemples multi-composants

**Ce que vous y trouverez** :
- ✅ Templates commentés ligne par ligne
- ✅ Explications des paramètres importants
- ✅ Exemples d'utilisation
- ✅ Bonnes pratiques intégrées
- ✅ Commandes kubectl associées

**Quand l'utiliser** :
- Lors de la création de nouvelles ressources
- Pour comprendre la structure des manifestes
- Comme référence pour les paramètres disponibles
- Pour valider vos propres manifestes

---

### 2. Exemples de Helm Charts

Helm est le gestionnaire de packages pour Kubernetes. Au lieu de gérer des dizaines de fichiers YAML séparément, Helm vous permet de packager toute votre application dans un **Chart** réutilisable et configurable.

**Pourquoi Helm ?**

Sans Helm :
```bash
kubectl apply -f deployment.yaml
kubectl apply -f service.yaml
kubectl apply -f configmap.yaml
kubectl apply -f secret.yaml
kubectl apply -f ingress.yaml
kubectl apply -f pvc.yaml
# ... et gérer les différences entre dev, staging, production
```

Avec Helm :
```bash
helm install mon-app ./mon-chart -f values-production.yaml
# Tout est déployé en une seule commande !
```

**Exemples de Charts inclus** :

1. **Chart Simple** - Application web NGINX
   - Idéal pour commencer avec Helm
   - Structure de base d'un Chart
   - Utilisation des values et templates

2. **Chart Full Stack** - Application avec base de données
   - Frontend + Backend + PostgreSQL
   - Gestion des dépendances
   - Configuration multi-services

3. **Chart Microservices** - Architecture microservices
   - Gestion de plusieurs services
   - ConfigMaps et Secrets avancés
   - Service discovery

4. **Chart avec Jobs** - Migrations et Init Containers
   - Jobs de migration de base de données
   - Init Containers pour l'initialisation
   - Hooks Helm

5. **Chart avec HPA** - Autoscaling et ressources
   - Horizontal Pod Autoscaler
   - PodDisruptionBudget
   - Gestion avancée des ressources

6. **Chart Multi-Environnements**
   - Développement, Staging, Production
   - Fichiers values séparés
   - Configuration par environnement

**Ce que vous apprendrez** :
- 📦 Comment structurer un Helm Chart
- 🔧 Comment utiliser les templates et les values
- 🎯 Comment gérer plusieurs environnements
- 🔄 Comment versionner et partager vos Charts
- 🚀 Comment simplifier vos déploiements

**Ce que vous y trouverez** :
- ✅ Charts complets et fonctionnels
- ✅ Fichiers Chart.yaml, values.yaml, templates/
- ✅ Helpers et fonctions réutilisables
- ✅ NOTES.txt pour les instructions post-installation
- ✅ Bonnes pratiques Helm

**Quand l'utiliser** :
- Pour packager vos applications
- Pour gérer des configurations complexes
- Pour déployer sur plusieurs environnements
- Pour partager des applications avec votre équipe

---

### 3. Exemples de Pipelines CI/CD

Le CI/CD (Continuous Integration / Continuous Deployment) automatise le cycle de développement, des tests au déploiement en production. Ces pipelines transforment votre workflow de développement.

**Qu'est-ce que le CI/CD ?**

**Sans CI/CD** (processus manuel) :
```
Développeur écrit du code
↓
Commit → Pull → Build local
↓
Tests manuels
↓
Build Docker manuel
↓
Push manuel vers registry
↓
Déploiement manuel sur Kubernetes
↓
Vérification manuelle
↓
Si problème → Rollback manuel 😰
```

**Avec CI/CD** (processus automatique) :
```
Développeur commit
↓
🤖 Pipeline CI/CD automatique :
   ✅ Tests automatiques
   ✅ Build Docker
   ✅ Scan de sécurité
   ✅ Push vers registry
   ✅ Déploiement Kubernetes
   ✅ Tests de santé
   ✅ Rollback automatique si échec
↓
Application en production ! 🚀
```

**Exemples de Pipelines inclus** :

1. **GitLab CI/CD**
   - Pipeline simple pour Node.js
   - Pipeline avec tests de sécurité
   - Déploiement multi-environnements
   - Format : `.gitlab-ci.yml`

2. **GitHub Actions**
   - Workflow complet avec matrix testing
   - Tests d'intégration avec services
   - Analyse de sécurité (CodeQL, Trivy)
   - Déploiement progressif
   - Format : `.github/workflows/`

3. **Jenkins**
   - Pipeline déclaratif
   - Pipeline scripté avancé
   - Gestion des environnements
   - Rollback automatique
   - Format : `Jenkinsfile`

4. **ArgoCD (GitOps)**
   - Synchronisation automatique depuis Git
   - Multi-environnements
   - Hooks pour migrations
   - Déploiement déclaratif
   - Format : Applications ArgoCD

5. **CircleCI**
   - Configuration avec orbs
   - Workflows parallèles
   - Déploiement avec approbation
   - Format : `.circleci/config.yml`

**Fonctionnalités couvertes** :
- 🧪 Tests automatisés (unitaires, intégration, E2E)
- 🔍 Analyse de code (linting, qualité)
- 🔒 Scan de sécurité (dépendances, images Docker)
- 🐳 Build et push d'images Docker
- 📦 Déploiement sur Kubernetes/MicroK8s
- 🔄 Stratégies de déploiement (rolling, blue-green, canary)
- 💊 Health checks et rollback automatique
- 📢 Notifications (Slack, email)

**Ce que vous y trouverez** :
- ✅ Pipelines complets et prêts à l'emploi
- ✅ Commentaires détaillés sur chaque étape
- ✅ Intégration avec Kubernetes/MicroK8s
- ✅ Bonnes pratiques de sécurité
- ✅ Stratégies de déploiement avancées
- ✅ Gestion des environnements multiples

**Quand l'utiliser** :
- Pour automatiser vos déploiements
- Pour garantir la qualité du code
- Pour déployer rapidement et en toute confiance
- Pour mettre en place une vraie démarche DevOps

---

## Organisation et navigation

### Structure des sections

Chaque section de cette annexe est organisée de manière progressive :

1. **Introduction** : Présentation du sujet et des concepts
2. **Exemples simples** : Pour commencer et comprendre les bases
3. **Exemples intermédiaires** : Cas d'usage plus réalistes
4. **Exemples avancés** : Configurations complexes et production-ready
5. **Bonnes pratiques** : Conseils et recommandations
6. **Troubleshooting** : Résolution des problèmes courants

### Comment naviguer

**Si vous êtes débutant** :
1. Commencez par les **Templates de Manifestes YAML** pour comprendre les ressources de base
2. Étudiez les exemples simples et lisez tous les commentaires
3. Expérimentez avec les exemples sur votre MicroK8s
4. Progressez vers les **Exemples de Helm Charts** quand vous êtes à l'aise
5. Explorez les **Pipelines CI/CD** pour automatiser vos déploiements

**Si vous avez de l'expérience** :
- Utilisez cette annexe comme référence rapide
- Copiez les templates qui vous intéressent
- Adaptez les exemples à vos besoins
- Inspirez-vous des bonnes pratiques

**Pour un projet spécifique** :
1. Identifiez les ressources nécessaires dans les templates
2. Choisissez un Chart Helm adapté si votre application est complexe
3. Mettez en place un pipeline CI/CD pour l'automatisation
4. Consultez la section troubleshooting en cas de problème

---

## Conventions et format

### Conventions de nommage

Dans tous les exemples, nous utilisons des conventions cohérentes :

**Noms de ressources** :
- `mon-app`, `mon-service` : Noms génériques à remplacer par les vôtres
- `example.com` : Domaine à remplacer par le vôtre
- `production`, `staging`, `dev` : Environnements standards

**Variables et placeholders** :
- `${{ variable }}` : Variables à remplacer (GitHub Actions)
- `${VARIABLE}` : Variables d'environnement
- `<valeur>` : Valeurs à personnaliser

### Format des fichiers

**Manifestes YAML** :
```yaml
# Commentaire expliquant la ressource
apiVersion: v1
kind: Pod
metadata:
  name: mon-pod  # Description du champ
spec:
  # ...
```

**Helm Charts** :
```yaml
# values.yaml avec valeurs par défaut
app:
  name: mon-app  # Nom de l'application
  replicas: 3    # Nombre de réplicas
```

**Pipelines CI/CD** :
```yaml
# .gitlab-ci.yml ou équivalent
stages:
  - test    # Étape de tests
  - build   # Étape de build
  - deploy  # Étape de déploiement
```

### Commentaires

Les exemples contiennent trois types de commentaires :

1. **Commentaires explicatifs** : Expliquent ce que fait le code
2. **Commentaires pédagogiques** : Enseignent les concepts
3. **Commentaires de configuration** : Indiquent les valeurs à modifier

---

## Utilisation pratique

### Copier et adapter

Tous les templates et exemples sont conçus pour être copiés et adaptés :

```bash
# 1. Copier un template
cp template-deployment.yaml mon-deployment.yaml

# 2. Remplacer les valeurs génériques
sed -i 's/mon-app/mon-application-reelle/g' mon-deployment.yaml

# 3. Adapter la configuration à vos besoins
nano mon-deployment.yaml

# 4. Valider avant application
kubectl apply --dry-run=client -f mon-deployment.yaml

# 5. Appliquer
kubectl apply -f mon-deployment.yaml
```

### Validation

Avant d'appliquer un template, validez-le toujours :

```bash
# Validation syntaxique
kubectl apply --dry-run=client -f mon-fichier.yaml

# Validation avec le serveur
kubectl apply --dry-run=server -f mon-fichier.yaml

# Linter YAML
yamllint mon-fichier.yaml

# Helm
helm lint mon-chart/
```

### Tests

Testez vos configurations dans un environnement de développement :

```bash
# Créer un namespace de test
kubectl create namespace test

# Déployer dans le namespace de test
kubectl apply -f mon-fichier.yaml -n test

# Vérifier
kubectl get all -n test

# Nettoyer
kubectl delete namespace test
```

---

## Bonnes pratiques générales

### Organisation des fichiers

```
mon-projet/
├── k8s/                          # Manifestes Kubernetes bruts
│   ├── base/                     # Configuration de base
│   │   ├── deployment.yaml
│   │   ├── service.yaml
│   │   └── configmap.yaml
│   └── overlays/                 # Surcharges par environnement
│       ├── dev/
│       ├── staging/
│       └── production/
├── helm/                         # Charts Helm
│   ├── Chart.yaml
│   ├── values.yaml
│   ├── values-dev.yaml
│   ├── values-staging.yaml
│   ├── values-production.yaml
│   └── templates/
├── .github/workflows/            # Pipelines GitHub Actions
│   └── ci-cd.yml
└── README.md                     # Documentation du projet
```

### Versionning

```bash
# Versionner vos configurations
git init
git add k8s/ helm/ .github/
git commit -m "Initial configuration"

# Taguer les releases
git tag -a v1.0.0 -m "Version 1.0.0"
git push origin v1.0.0
```

### Documentation

Documentez toujours vos configurations :

```yaml
# deployment.yaml
# Description: Déploiement de l'application principale
# Maintainer: votre-nom@example.com
# Version: 1.0.0
# Dernière mise à jour: 2024-01-15
apiVersion: apps/v1
kind: Deployment
# ...
```

### Sécurité

**⚠️ Points d'attention** :

```yaml
# ❌ NE JAMAIS faire
apiVersion: v1
kind: Secret
metadata:
  name: mon-secret
stringData:
  password: "mot-de-passe-en-clair"  # DANGER !

# ✅ Utiliser des outils de gestion de secrets
# - Sealed Secrets
# - External Secrets Operator
# - Vault
```

**Règles de sécurité** :
- 🔒 Ne jamais committer de secrets dans Git
- 🔒 Utiliser des ServiceAccounts dédiés
- 🔒 Appliquer le principe du moindre privilège (RBAC)
- 🔒 Scanner les images Docker
- 🔒 Utiliser NetworkPolicies
- 🔒 Définir des ResourceQuotas et LimitRanges

---

## Support et ressources

### Où obtenir de l'aide

Si vous rencontrez des difficultés :

1. **Consultez les commentaires** dans les templates - ils contiennent souvent la réponse
2. **Vérifiez la section troubleshooting** de chaque section
3. **Utilisez la documentation officielle** :
   - Kubernetes : https://kubernetes.io/docs/
   - Helm : https://helm.sh/docs/
   - MicroK8s : https://microk8s.io/docs/

### Ressources complémentaires

**Documentation officielle** :
- Kubernetes API Reference : https://kubernetes.io/docs/reference/
- Helm Chart Guide : https://helm.sh/docs/chart_template_guide/
- GitLab CI/CD : https://docs.gitlab.com/ee/ci/
- GitHub Actions : https://docs.github.com/actions
- Jenkins : https://www.jenkins.io/doc/

**Outils utiles** :
- **kubectl** : CLI Kubernetes
- **helm** : Gestionnaire de packages
- **k9s** : Interface TUI pour Kubernetes
- **lens** : IDE Kubernetes
- **kubectx/kubens** : Gestion des contextes et namespaces

**Communautés** :
- Kubernetes Slack : https://slack.k8s.io/
- Reddit r/kubernetes : https://reddit.com/r/kubernetes
- Stack Overflow : tag `kubernetes`

---

## Contribuer et améliorer

### Personnalisation

Cette annexe est un point de départ. N'hésitez pas à :

- ✏️ Adapter les templates à vos besoins
- 📝 Ajouter vos propres exemples
- 🔧 Créer vos propres Charts Helm
- 🚀 Personnaliser les pipelines CI/CD
- 📚 Enrichir la documentation

### Partage

Si vous créez des templates ou exemples utiles :

- Partagez-les avec votre équipe
- Créez un repository interne
- Contribuez aux projets open-source
- Documentez vos découvertes

### Mise à jour

Les technologies évoluent rapidement :

- 🔄 Mettez à jour les versions d'images
- 📊 Suivez les nouvelles fonctionnalités Kubernetes
- 🔒 Appliquez les patches de sécurité
- 📖 Restez informé des bonnes pratiques

---

## Prêt à commencer ?

Maintenant que vous connaissez l'organisation de cette annexe, vous êtes prêt à explorer les différentes sections :

1. 📋 **[Templates de Manifestes YAML](#)** → Pour créer vos ressources Kubernetes
2. 📦 **[Exemples de Helm Charts](#)** → Pour packager vos applications
3. 🚀 **[Exemples de Pipelines CI/CD](#)** → Pour automatiser vos déploiements

**💡 Conseil pour débuter** :
- Si vous découvrez Kubernetes : commencez par les Templates de Manifestes YAML
- Si vous avez des bases : explorez les Exemples de Helm Charts
- Si vous voulez automatiser : plongez dans les Pipelines CI/CD

**Bon apprentissage et bonnes explorations ! 🚀**

---

## Index rapide des ressources

### Par cas d'usage

**Déployer une application web simple** :
- Template Deployment + Service
- Chart Helm simple
- Pipeline GitLab CI basique

**Déployer une application avec base de données** :
- Template StatefulSet + PVC
- Chart Helm Full Stack
- Pipeline avec tests d'intégration

**Exposer une application sur Internet** :
- Template Ingress + TLS
- Chart avec Ingress configuré
- Pipeline avec vérification des certificats

**Gérer des secrets** :
- Template Secret
- Chart avec Secrets management
- Pipeline avec secrets CI/CD

**Autoscaler une application** :
- Template HPA
- Chart avec autoscaling
- Pipeline avec tests de charge

**Sauvegarder des données** :
- Template PVC + Backup Job
- Chart avec backup intégré
- Pipeline avec backup automatique

**Déployer en multi-environnements** :
- Templates avec Kustomize
- Chart Helm multi-environnements
- Pipeline GitOps ArgoCD

### Par niveau

**Débutant** 🌱 :
- Pod simple
- Deployment basique
- Service ClusterIP
- ConfigMap simple
- Chart Helm minimal

**Intermédiaire** 🌿 :
- StatefulSet avec PVC
- Ingress avec TLS
- NetworkPolicy
- HPA
- Chart Full Stack
- Pipeline GitLab CI

**Avancé** 🌳 :
- Application complète multi-tiers
- RBAC avancé
- Chart microservices
- Pipeline multi-environnements
- GitOps avec ArgoCD

---

**Prêt ? Passons à la pratique ! 🎯**

⏭️ [Templates de manifestes YAML](/annexes/annexe-b-templates-et-exemples/templates-de-manifestes-yaml.md)
