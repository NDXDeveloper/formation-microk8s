🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 17. DevOps et CI/CD

## Introduction

Félicitations d'être arrivé jusqu'ici ! Vous avez parcouru un long chemin dans votre apprentissage de Kubernetes et MicroK8s. Vous savez maintenant créer des Pods, des Deployments, des Services, gérer le stockage, configurer le réseau, et bien plus encore. Vous avez les **fondations techniques** nécessaires pour faire tourner des applications sur Kubernetes.

Mais une question reste en suspens : **comment passer du code à la production de manière fiable, répétable et automatisée ?**

Jusqu'à présent, vous avez probablement déployé vos applications manuellement :
- Écrire du code sur votre machine
- Construire une image Docker manuellement
- La pousser vers un registry à la main
- Exécuter `kubectl apply -f deployment.yaml`
- Croiser les doigts en espérant que tout fonctionne 🤞

Cette approche manuelle fonctionne pour l'apprentissage et les petits projets personnels, mais elle devient rapidement **problématique en production** :

- ❌ **Erreurs humaines** : Un `kubectl delete` sur le mauvais namespace et c'est la catastrophe
- ❌ **Pas de traçabilité** : Qui a déployé quoi, quand, et pourquoi ?
- ❌ **Lent et répétitif** : Toujours les mêmes étapes manuelles à chaque mise à jour
- ❌ **Pas de tests** : Le code est déployé sans validation préalable
- ❌ **Rollback difficile** : Comment revenir à la version précédente rapidement ?
- ❌ **Pas de collaboration** : Difficile de travailler en équipe
- ❌ **Stress** : Chaque déploiement devient une source d'anxiété

C'est là qu'interviennent **DevOps** et **CI/CD** : un ensemble de pratiques, d'outils et de philosophies qui transforment le déploiement d'applications d'un processus manuel stressant en un workflow **automatisé, fiable et serein**.

---

## Qu'est-ce que DevOps ?

### Définition

**DevOps** est une contraction de **"Development"** (développement) et **"Operations"** (exploitation). C'est une **culture** et un ensemble de pratiques qui visent à :

- **Rapprocher les équipes** de développement et d'exploitation
- **Automatiser** les processus de livraison logicielle
- **Améliorer** la qualité et la vitesse de livraison
- **Réduire** le temps entre l'écriture du code et sa mise en production

### Le problème historique

Avant DevOps, les équipes fonctionnaient en **silos** :

```
┌──────────────────────────────────────────────────────────┐
│                    AVANT DEVOPS                          │
│                                                          │
│  ┌──────────────┐              ┌──────────────┐          │
│  │  Développeurs│              │   Ops/IT     │          │
│  │              │              │              │          │
│  │ • Écrit code │              │ • Déploie    │          │
│  │ • Tests      │              │ • Maintient  │          │
│  │ • Features   │    MUR 🧱    │ • Surveille  │          │
│  │ • Rapidité   │              │ • Stabilité  │          │
│  │              │              │              │          │
│  │ "Ça marche   │              │ "Ça ne       │          │
│  │  sur ma      │              │  marche pas  │          │
│  │  machine!"   │              │  en prod!"   │          │
│  └──────────────┘              └──────────────┘          │
│                                                          │
│  Résultat : Friction, lenteur, bugs en production        │
└──────────────────────────────────────────────────────────┘
```

**Les développeurs** voulaient livrer rapidement de nouvelles fonctionnalités.

**Les Ops** voulaient la stabilité et éviter les pannes.

Ces objectifs contradictoires créaient des **conflits** et des **inefficacités**.

### L'approche DevOps

DevOps casse ces silos et unifie les équipes autour d'objectifs communs :

```
┌──────────────────────────────────────────────────────────┐
│                     AVEC DEVOPS                          │
│                                                          │
│        ┌────────────────────────────────────┐            │
│        │        Équipe DevOps unifiée       │            │
│        │                                    │            │
│        │  Développeurs + Ops + QA + Sécurité│            │
│        │                                    │            │
│        │  • Objectifs communs               │            │
│        │  • Responsabilités partagées       │            │
│        │  • Communication constante         │            │
│        │  • Automatisation maximale         │            │
│        │                                    │            │
│        │  "Nous livrons ensemble,           │            │
│        │   rapidement ET en toute sécurité" │            │
│        └────────────────────────────────────┘            │
│                                                          │
│  Résultat : Livraisons rapides, fiables, sans friction   │
└──────────────────────────────────────────────────────────┘
```

### Les piliers de DevOps

**1. Culture** : Collaboration, confiance mutuelle, apprentissage continu

**2. Automatisation** : Éliminer les tâches manuelles et répétitives

**3. Mesure** : Métriques, monitoring, feedback constant

**4. Partage** : Connaissance, responsabilité, succès et échecs

### Bénéfices concrets

Selon les études de DORA (DevOps Research and Assessment), les organisations DevOps matures ont :

- ✅ **208x plus de déploiements** que les organisations traditionnelles
- ✅ **106x plus rapide** de récupération après incident
- ✅ **7x moins de taux d'échec** sur les changements
- ✅ **2x moins de temps** passé sur le travail non planifié

---

## Qu'est-ce que CI/CD ?

**CI/CD** est l'acronyme de **Continuous Integration / Continuous Delivery** (ou Deployment). C'est la **concrétisation technique** de la philosophie DevOps à travers l'automatisation du pipeline logiciel.

### Les trois composantes

```
┌───────────────────────────────────────────────────────────┐
│                                                           │
│   ┌──────────────┐   ┌──────────────┐   ┌─────────────┐   │
│   │     CI       │ → │     CD       │ → │     CD      │   │
│   │ Continuous   │   │ Continuous   │   │ Continuous  │   │
│   │ Integration  │   │ Delivery     │   │ Deployment  │   │
│   └──────────────┘   └──────────────┘   └─────────────┘   │
│         ↓                   ↓                   ↓         │
│   Automatiser          Automatiser         Déploiement    │
│   la construction      la préparation      automatique    │
│   et les tests         du release          en production  │
│                                                           │
└───────────────────────────────────────────────────────────┘
```

### 1. CI - Continuous Integration (Intégration Continue)

**Principe** : Chaque changement de code est **automatiquement** construit, testé et vérifié.

**Workflow** :

```
Developer push code
        ↓
    Git repository
        ↓
CI système détecte
        ↓
┌───────────────────┐
│ 1. Clone code     │
│ 2. Build          │
│ 3. Run tests      │
│ 4. Code quality   │
│ 5. Security scan  │
└───────────────────┘
        ↓
  ✅ Success ou ❌ Failure
        ↓
   Feedback immédiat
```

**Avantages** :
- Détection précoce des bugs
- Code toujours dans un état fonctionnel
- Feedback rapide pour les développeurs
- Réduction des conflits d'intégration

**Exemple concret** :
```
1. Alice écrit du code et push sur Git
2. En 2 minutes, elle reçoit un email :
   ✅ "Build succeeded, all tests passed"
3. Elle peut continuer sereinement
```

Au lieu de :
```
1. Alice écrit du code pendant 3 jours
2. Elle essaie d'intégrer : ❌ Conflits partout!
3. 2 jours pour résoudre les problèmes
```

### 2. CD - Continuous Delivery (Livraison Continue)

**Principe** : Le code qui passe le CI est **automatiquement** préparé et prêt à être déployé, mais nécessite une **approbation manuelle** pour la production.

**Workflow** :

```
CI réussit
    ↓
┌──────────────────────┐
│ 1. Build artifact    │
│ 2. Push to registry  │
│ 3. Deploy to staging │
│ 4. Integration tests │
│ 5. Security scan     │
└──────────────────────┘
    ↓
Release prêt
    ↓
👤 Approbation manuelle
    ↓
Déploiement production
```

**Avantages** :
- Release toujours prêt
- Déploiement rapide quand décidé
- Contrôle humain avant production
- Tests exhaustifs avant release

### 3. CD - Continuous Deployment (Déploiement Continu)

**Principe** : Le code qui passe tous les tests est **automatiquement** déployé en production, **sans intervention humaine**.

**Workflow** :

```
CI réussit
    ↓
┌──────────────────────┐
│ 1. Build artifact    │
│ 2. Push to registry  │
│ 3. Deploy to staging │
│ 4. Integration tests │
│ 5. Security scan     │
│ 6. Deploy to prod    │ ← Automatique !
└──────────────────────┘
    ↓
En production
    ↓
Monitoring
```

**Avantages** :
- Délai minimal code → production
- Déploiements fréquents (plusieurs fois par jour)
- Réduction du risque (petits changements)
- Feedback utilisateur rapide

**Différence clé** :
- **Continuous Delivery** : Push button pour déployer
- **Continuous Deployment** : Pas de bouton, c'est automatique

---

## Pourquoi DevOps/CI/CD avec Kubernetes ?

Kubernetes et DevOps/CI/CD sont **faits l'un pour l'autre**. Voici pourquoi :

### 1. Infrastructure as Code

Kubernetes fonctionne entièrement avec des **manifestes YAML déclaratifs** :

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mon-app
spec:
  replicas: 3
  ...
```

Ces fichiers peuvent être :
- Versionnés dans Git
- Testés automatiquement
- Déployés automatiquement
- Audités et tracés

### 2. Déploiements automatisés et contrôlés

Kubernetes offre des stratégies de déploiement sophistiquées :
- **Rolling updates** : Mise à jour progressive sans downtime
- **Blue-Green** : Bascule instantanée entre versions
- **Canary** : Exposition progressive à un % d'utilisateurs

### 3. Self-healing et résilience

Kubernetes redémarre automatiquement les conteneurs défaillants, ce qui permet :
- Des déploiements plus confiants
- Une récupération automatique après problème
- Moins de stress lors des mises à jour

### 4. Scalabilité automatique

Couplé au CI/CD, Kubernetes peut :
- Scaler automatiquement selon la charge
- Gérer des milliers de déploiements par jour
- Supporter une croissance massive

### 5. Écosystème riche

L'écosystème Kubernetes offre des outils dédiés au DevOps :
- **Helm** : Package manager
- **ArgoCD** : GitOps pour Kubernetes
- **Flux** : Alternative GitOps
- **Tekton** : Pipelines natifs Kubernetes
- **Jenkins X** : CI/CD cloud-native

---

## Le workflow DevOps complet

Voici à quoi ressemble un workflow DevOps/CI/CD moderne avec Kubernetes :

```
┌─────────────────────────────────────────────────────────────────┐
│                    WORKFLOW DEVOPS COMPLET                      │
└─────────────────────────────────────────────────────────────────┘

1. DÉVELOPPEMENT
   ┌─────────────────┐
   │ Developer       │
   │ écrit du code   │
   └────────┬────────┘
            │
            ▼
   ┌─────────────────┐
   │ Git commit      │
   │ git push        │
   └────────┬────────┘
            │
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
            │
2. CONTINUOUS INTEGRATION (CI)
            ▼
   ┌─────────────────┐
   │ Git détecte     │
   │ le changement   │
   └────────┬────────┘
            │
            ▼
   ┌─────────────────┐
   │ CI Pipeline     │
   │ (GitLab/GitHub) │
   └────────┬────────┘
            │
            ├──► 1. Lint du code
            │
            ├──► 2. Unit tests
            │
            ├──► 3. Build Docker image
            │
            ├──► 4. Security scan
            │
            ├──► 5. Push to registry
            │
            └──► 6. Integration tests
                     │
                     ▼
              ✅ Tests réussis
                     │
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
                     │
3. CONTINUOUS DELIVERY (CD)
                     ▼
           ┌─────────────────┐
           │ Deploy Staging  │
           │ (Kubernetes)    │
           └────────┬────────┘
                    │
                    ├──► Tests E2E
                    │
                    ├──► Tests de charge
                    │
                    └──► Smoke tests
                             │
                             ▼
                      ✅ Validation OK
                             │
                             ▼
                    ┌─────────────────┐
                    │ ArgoCD/Flux     │
                    │ détecte changes │
                    └────────┬────────┘
                             │
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
                             │
4. DÉPLOIEMENT PRODUCTION
                             ▼
                    ┌──────────────────┐
                    │ Deploy Strategy  │
                    │ (Blue-Green/     │
                    │  Canary/Rolling) │
                    └────────┬─────────┘
                             │
                             ▼
                    ┌──────────────────┐
                    │ Kubernetes       │
                    │ Cluster Prod     │
                    └────────┬─────────┘
                             │
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
                             │
5. MONITORING & FEEDBACK
                             ▼
                    ┌──────────────────┐
                    │ Prometheus       │
                    │ + Grafana        │
                    └────────┬─────────┘
                             │
                             ├──► Métriques
                             │
                             ├──► Logs
                             │
                             ├──► Alertes
                             │
                             └──► Feedback loop
                                       │
                                       ▼
                              Retour aux développeurs
```

**Durée totale** : De quelques minutes à quelques heures selon la complexité

**Fréquence** : Plusieurs fois par jour dans les organisations matures

---

## Ce que vous allez apprendre dans ce chapitre

Ce chapitre 17 est structuré pour vous guider progressivement dans la mise en place d'un workflow DevOps/CI/CD complet sur votre cluster MicroK8s.

### 17.1 Principes DevOps et GitOps
- Comprendre la philosophie DevOps en profondeur
- Découvrir GitOps : l'évolution DevOps native Kubernetes
- Git comme source unique de vérité

### 17.2 Installation d'une registry Docker locale
- Pourquoi une registry locale
- Installation et configuration sur MicroK8s
- Gestion des images Docker

### 17.3 Configuration de GitLab CI/CD
- Installation de GitLab sur MicroK8s
- Configuration des runners
- Création de pipelines CI/CD

### 17.4 Configuration de Jenkins
- Installation de Jenkins
- Configuration des agents
- Pipelines déclaratifs et scriptés

### 17.5 GitHub Actions pour Kubernetes
- Configuration des workflows GitHub
- Secrets et sécurité
- Déploiement automatique

### 17.6 ArgoCD et GitOps
- Installation d'ArgoCD
- Configuration des applications
- Synchronisation automatique

### 17.7 Helm et templating
- Introduction à Helm
- Création de charts
- Gestion des versions

### 17.8 Automated testing
- Tests unitaires pour manifestes
- Tests d'intégration
- Tests de charge et sécurité

### 17.9 Stratégies de déploiement
- Vue d'ensemble des stratégies
- Rolling updates, Blue-Green, Canary
- Choix de la bonne stratégie

### 17.10 Blue-Green deployments
- Implémentation détaillée
- Gestion du rollback
- Cas d'usage avancés

### 17.11 Canary deployments
- Déploiements progressifs
- Analyse automatique avec Flagger
- Métriques et monitoring

---

## Prérequis pour ce chapitre

Avant de commencer, assurez-vous d'avoir :

### Connaissances techniques

✅ **Kubernetes de base** :
- Comprendre Pods, Deployments, Services
- Savoir utiliser `kubectl`
- Connaître les manifestes YAML

✅ **Git** :
- Comprendre les concepts : commit, push, pull, branch
- Savoir utiliser les commandes de base
- (Pas besoin d'être un expert)

✅ **Docker** :
- Comprendre les images et conteneurs
- Savoir créer un Dockerfile basique
- Connaître `docker build` et `docker push`

✅ **Linux/Bash** :
- Commandes de base
- Éditeurs de texte (vim, nano)
- Navigation dans les fichiers

### Infrastructure

✅ **Cluster MicroK8s fonctionnel** :
```bash
microk8s status
# Devrait afficher "microk8s is running"
```

✅ **Ressources suffisantes** :
- CPU : 4+ cœurs recommandés
- RAM : 8+ Go recommandés
- Disk : 50+ Go disponibles

✅ **Accès Internet** :
- Pour télécharger les images
- Pour accéder à Git (GitHub/GitLab)

### Outils à installer

Nous installerons au fur et à mesure, mais préparez-vous à utiliser :
- Git
- Docker
- Helm
- kubectl (déjà présent avec MicroK8s)

---

## Comment aborder ce chapitre

### Pour les débutants

Si vous découvrez DevOps/CI/CD :

1. **Lisez dans l'ordre** : Les sections sont progressives
2. **Prenez votre temps** : Certains concepts sont complexes
3. **Expérimentez** : Testez chaque outil sur votre MicroK8s
4. **Ne paniquez pas** : Il y a beaucoup d'outils, mais chacun a sa place
5. **Commencez simple** : Vous n'avez pas besoin de tout utiliser immédiatement

### Pour les praticiens

Si vous avez déjà de l'expérience :

1. **Choisissez votre outil** : GitLab CI, Jenkins, ou GitHub Actions
2. **Focalisez sur GitOps** : ArgoCD est la star de ce chapitre
3. **Explorez les stratégies** : Blue-Green et Canary sont très utiles
4. **Automatisez** : L'objectif est zéro intervention manuelle

### Approche modulaire

Vous n'êtes **pas obligé** d'utiliser tous les outils présentés :

**Minimum viable** :
- Git + GitHub/GitLab
- GitHub Actions OU GitLab CI
- ArgoCD
- Stratégie Rolling Update (native K8s)

**Stack recommandée** :
- Git + GitHub/GitLab
- CI/CD de votre choix
- Registry Docker locale
- ArgoCD pour GitOps
- Helm pour le templating
- Tests automatisés
- Stratégie Blue-Green ou Canary

**Stack complète (enterprise)** :
- Tout ce qui précède +
- Monitoring avancé (Prometheus/Grafana)
- Security scanning (Trivy, Checkov)
- Policy enforcement (OPA)
- Service mesh (Istio/Linkerd)

---

## Objectifs d'apprentissage

À la fin de ce chapitre, vous serez capable de :

✅ **Comprendre** la philosophie DevOps et son application à Kubernetes

✅ **Mettre en place** un pipeline CI/CD complet de bout en bout

✅ **Automatiser** le déploiement de vos applications sur Kubernetes

✅ **Implémenter** GitOps avec ArgoCD pour une approche déclarative

✅ **Créer** des Helm charts pour packager vos applications

✅ **Tester** automatiquement vos manifestes et applications

✅ **Déployer** avec des stratégies avancées (Blue-Green, Canary)

✅ **Rollback** rapidement en cas de problème

✅ **Travailler en équipe** avec des workflows collaboratifs

✅ **Déployer sereinement** plusieurs fois par jour en production

---

## L'état d'esprit DevOps

Avant de plonger dans les outils, adoptons le bon **état d'esprit** :

### 1. Automatiser tout ce qui peut l'être

Si vous faites quelque chose manuellement plus de 2 fois, **automatisez-le**.

### 2. Fail fast, learn fast

Les erreurs sont normales et attendues. L'important est de :
- Les **détecter rapidement** (CI/CD)
- Les **corriger rapidement** (rollback automatique)
- **Apprendre** pour ne pas les répéter

### 3. Petit et fréquent > Gros et rare

Mieux vaut :
- 10 petits déploiements par jour
- Que 1 gros déploiement par mois

**Pourquoi ?**
- Moins de changements = moins de risque
- Plus facile à débugger
- Rollback plus simple
- Feedback plus rapide

### 4. Mesurer tout

"You can't improve what you don't measure"
- Temps de build
- Fréquence de déploiement
- Taux d'échec
- Temps de récupération
- Métriques applicatives

### 5. Partager la responsabilité

Le déploiement n'est plus "le problème des Ops". Toute l'équipe est responsable :
- Du code
- Des tests
- Du déploiement
- De la production
- Des incidents

### 6. Boucle de feedback courte

Plus vite vous avez du feedback, mieux c'est :
- Tests en < 5 minutes
- Build en < 10 minutes
- Déploiement en < 30 minutes
- Feedback utilisateur immédiat

---

## Vocabulaire DevOps/CI/CD

Avant de commencer, familiarisons-nous avec le vocabulaire :

**Pipeline** : Séquence d'étapes automatisées (build, test, deploy)

**Stage** : Étape dans un pipeline (ex: "test", "build", "deploy")

**Job** : Tâche spécifique dans un stage (ex: "run unit tests")

**Artifact** : Résultat d'un build (image Docker, binaire, archive)

**Registry** : Dépôt d'images Docker (comme DockerHub)

**Manifest** : Fichier YAML décrivant des ressources Kubernetes

**Chart** : Package Helm contenant des templates de manifestes

**Release** : Version déployée d'une application

**Rollback** : Retour à une version précédente

**Deployment Strategy** : Méthode de mise à jour (Rolling, Blue-Green, Canary)

**GitOps** : Pratique où Git est la source de vérité pour l'infrastructure

**Reconciliation** : Process de synchronisation de l'état désiré vs réel

---

## La promesse DevOps

Si vous investissez le temps nécessaire pour mettre en place ces pratiques, vous obtiendrez :

### Avant DevOps/CI/CD :
- 😰 Déploiements stressants le vendredi soir
- 🐛 Bugs découverts en production
- 🕐 Heures/jours pour déployer
- ❌ Peur de casser quelque chose
- 🤔 "Qui a déployé ça?"
- 📱 Téléphone qui sonne le week-end

### Après DevOps/CI/CD :
- 😌 Déploiements automatisés et confiants
- 🧪 Bugs détectés en CI avant production
- ⚡ Minutes pour déployer
- ✅ Rollback automatique si problème
- 📝 Traçabilité complète dans Git
- 🏖️ Week-ends tranquilles

---

## Prêt à commencer ?

Vous avez maintenant une vue d'ensemble de ce qui vous attend dans ce chapitre. C'est un voyage passionnant qui va transformer votre façon de déployer des applications sur Kubernetes.

**N'oubliez pas** :
- Ce chapitre est dense, prenez votre temps
- Testez chaque concept sur votre MicroK8s
- L'objectif n'est pas de tout mémoriser, mais de comprendre les concepts
- Vous pouvez toujours revenir en arrière
- La pratique rend parfait

Dans la section suivante, nous commencerons par les **fondamentaux DevOps et GitOps**, pour bien comprendre la philosophie avant de plonger dans les outils.

**Allons-y ! 🚀**

⏭️ [Principes DevOps et GitOps](/17-devops-et-cicd/01-principes-devops-et-gitops.md)
