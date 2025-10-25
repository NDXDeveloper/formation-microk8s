🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 17.1 Principes DevOps et GitOps

## Introduction

Avant de plonger dans les outils et les techniques de déploiement automatisé sur MicroK8s, il est essentiel de comprendre les philosophies qui les sous-tendent : **DevOps** et **GitOps**. Ces approches ont révolutionné la façon dont les équipes développent, déploient et maintiennent les applications modernes.

Dans ce chapitre, nous allons explorer ces concepts fondamentaux de manière accessible, en expliquant pourquoi ils sont importants et comment ils s'appliquent concrètement à votre environnement Kubernetes.

---

## Qu'est-ce que DevOps ?

### Définition

**DevOps** est la contraction de "Development" (Développement) et "Operations" (Opérations). C'est avant tout une **culture** et un ensemble de **pratiques** qui visent à rapprocher les équipes de développement et les équipes d'exploitation informatique.

Traditionnellement, ces deux équipes travaillaient en silos :
- Les **développeurs** créaient du code et voulaient déployer rapidement de nouvelles fonctionnalités
- Les **opérationnels** géraient l'infrastructure et privilégiaient la stabilité et la fiabilité

Cette séparation créait souvent des frictions, des délais et des incompréhensions.

### Objectifs du DevOps

DevOps cherche à briser ces silos en établissant :

1. **Une collaboration étroite** entre les équipes
2. **Une automatisation maximale** des processus
3. **Une amélioration continue** basée sur le feedback
4. **Une responsabilité partagée** sur l'ensemble du cycle de vie de l'application

### Les piliers du DevOps

#### 1. Culture et collaboration

Le DevOps encourage une culture où :
- Les développeurs comprennent les contraintes opérationnelles
- Les opérationnels participent dès les phases de conception
- L'échec est considéré comme une opportunité d'apprentissage
- La communication est transparente et continue

#### 2. Automatisation

L'automatisation est au cœur du DevOps. Elle concerne :
- **L'intégration continue (CI)** : automatisation des tests et de la construction du code
- **Le déploiement continu (CD)** : automatisation du déploiement vers les environnements
- **L'infrastructure as Code (IaC)** : gestion de l'infrastructure via du code versionné
- **Les tests automatisés** : validation continue de la qualité
- **Le monitoring automatisé** : surveillance et alertes

#### 3. Mesure et feedback

DevOps s'appuie sur des métriques pour :
- Mesurer les performances des applications
- Suivre la santé de l'infrastructure
- Évaluer la satisfaction des utilisateurs
- Identifier les axes d'amélioration

Exemples de métriques importantes :
- **Deployment Frequency** : fréquence de déploiement
- **Lead Time for Changes** : temps entre le commit et le déploiement
- **Mean Time to Recovery (MTTR)** : temps moyen de résolution d'incident
- **Change Failure Rate** : taux d'échec des changements

#### 4. Amélioration continue

Le cycle DevOps est itératif :
1. **Plan** : planifier les fonctionnalités et améliorations
2. **Code** : développer le code
3. **Build** : construire l'application
4. **Test** : tester automatiquement
5. **Release** : préparer la release
6. **Deploy** : déployer en production
7. **Operate** : exploiter et monitorer
8. **Monitor** : surveiller et collecter du feedback

Ce cycle se répète continuellement, permettant des livraisons fréquentes et de petite taille.

### Bénéfices du DevOps

En adoptant DevOps, les organisations constatent :
- **Des déploiements plus fréquents** : de plusieurs fois par jour au lieu de plusieurs fois par an
- **Une réduction des échecs** : moins de bugs en production grâce aux tests automatisés
- **Une récupération plus rapide** : capacité à corriger rapidement les problèmes
- **Une meilleure qualité** : feedback constant et amélioration continue
- **Une satisfaction accrue** : pour les équipes (moins de stress) et les utilisateurs (nouvelles fonctionnalités plus rapides)

---

## Qu'est-ce que GitOps ?

### Définition

**GitOps** est une évolution du DevOps qui place **Git au centre** de tous les processus de déploiement et d'exploitation. C'est une méthode de livraison continue qui utilise Git comme **source unique de vérité** pour l'infrastructure et les applications.

Le terme a été popularisé par Weaveworks en 2017, spécifiquement dans le contexte de Kubernetes.

### Le principe fondamental

Avec GitOps, l'état désiré de votre infrastructure et de vos applications est **décrit dans Git** sous forme de code. Le système se charge ensuite automatiquement de **synchroniser** l'état réel avec l'état désiré.

```
État désiré dans Git → Système automatisé → État réel dans le cluster
```

### Les quatre principes de GitOps

#### 1. Déclaratif

L'ensemble du système est décrit de **manière déclarative**. Plutôt que de dire "comment" faire quelque chose (impératif), on décrit "ce que" l'on veut obtenir.

Exemple :
- **Impératif** : "Exécute ces 10 commandes pour déployer l'application"
- **Déclaratif** : "Je veux 3 replicas de cette application avec cette version"

Dans Kubernetes, les manifestes YAML sont des descriptions déclaratives.

#### 2. Versionné et immuable

Tout est stocké dans **Git**, ce qui signifie :
- **Historique complet** : on peut voir qui a fait quoi et quand
- **Retour en arrière facile** : git revert pour annuler un changement
- **Auditabilité** : traçabilité complète des modifications
- **Immutabilité** : les commits sont immuables et signés

#### 3. Pull automatique

Le système cible (votre cluster Kubernetes) **récupère automatiquement** les changements depuis Git, plutôt que de recevoir des push externes.

Cela signifie :
- Pas besoin de credentials de cluster en dehors du cluster
- Plus de sécurité : le cluster n'expose pas d'API de déploiement
- Synchronisation continue : le cluster vérifie régulièrement Git

#### 4. Réconciliation continue

Des **agents logiciels** surveillent en permanence l'état désiré (dans Git) et l'état réel (dans le cluster) pour assurer leur convergence.

Si quelqu'un modifie manuellement quelque chose dans le cluster, l'agent le détectera et restaurera l'état défini dans Git. C'est ce qu'on appelle la **drift detection** (détection de dérive).

### Flux de travail GitOps

Voici comment fonctionne un workflow GitOps typique :

1. **Développement** : un développeur modifie le code de l'application
2. **Pull Request** : le développeur crée une PR avec les changements
3. **Tests automatisés** : la CI exécute les tests
4. **Revue de code** : les pairs examinent le code
5. **Merge** : la PR est fusionnée dans la branche principale
6. **Build** : la CI construit une nouvelle image Docker
7. **Update du manifeste** : la CI met à jour le manifeste Kubernetes dans Git avec la nouvelle version d'image
8. **Détection** : l'agent GitOps (comme ArgoCD ou Flux) détecte le changement dans Git
9. **Synchronisation** : l'agent applique automatiquement les changements au cluster
10. **Validation** : l'état du cluster correspond maintenant à l'état désiré dans Git

### Avantages de GitOps

#### Pour les développeurs
- **Familiarité** : utilisation d'outils qu'ils connaissent (Git, PR)
- **Transparence** : tout est visible dans Git
- **Rollback facile** : simple git revert
- **Pas de kubectl nécessaire** : pas besoin d'accès direct au cluster

#### Pour les opérationnels
- **Auditabilité** : qui a changé quoi et quand
- **Sécurité** : pas de credentials partagés
- **Disaster recovery** : reconstruction facile à partir de Git
- **Drift prevention** : détection et correction automatique des modifications manuelles

#### Pour l'organisation
- **Conformité** : piste d'audit complète
- **Collaboration** : processus de revue standardisé
- **Résilience** : recovery rapide en cas de problème
- **Standardisation** : même processus pour tous les environnements

---

## DevOps vs GitOps : Quelle différence ?

Il est important de comprendre que **GitOps n'est pas un remplacement de DevOps**, mais plutôt une **implémentation spécifique** des principes DevOps.

### DevOps
- **Philosophie générale** : culture, collaboration, automatisation
- **Scope large** : toutes les pratiques de développement et opérations
- **Flexibilité** : peut être implémenté de nombreuses façons

### GitOps
- **Méthodologie spécifique** : implémentation concrète de DevOps
- **Scope ciblé** : déploiement et gestion d'infrastructure
- **Git-centrique** : Git comme source de vérité obligatoire
- **Particulièrement adapté** : environnements Kubernetes

On peut dire que **GitOps est une façon de faire du DevOps** en utilisant Git comme pilier central.

---

## Pourquoi c'est important pour MicroK8s ?

### Kubernetes et le paradigme déclaratif

Kubernetes est **intrinsèquement déclaratif**. Vous décrivez l'état désiré (manifestes YAML) et Kubernetes s'occupe de l'atteindre et de le maintenir. C'est exactement l'esprit GitOps.

### MicroK8s comme environnement d'apprentissage

Votre cluster MicroK8s est l'endroit idéal pour :
- **Expérimenter** les pratiques DevOps/GitOps sans risque
- **Apprendre** les outils modernes (ArgoCD, Flux, Helm)
- **Automatiser** vos déploiements personnels
- **Comprendre** les concepts avant de les appliquer en production

### Du manuel à l'automatisé

Au début de votre apprentissage, vous avez probablement déployé des applications avec `kubectl apply -f`. C'est très bien pour apprendre, mais avec DevOps et GitOps, vous allez passer à :

- **Versionner** vos manifestes dans Git
- **Réviser** les changements via pull requests
- **Automatiser** les déploiements
- **Monitorer** automatiquement l'état du cluster
- **Restaurer** facilement en cas de problème

### Pipeline complet

Sur MicroK8s, vous pouvez mettre en place un pipeline complet :

```
Code → Git → CI (tests, build) → Registry → GitOps → MicroK8s → Monitoring
```

Tout cela sur votre propre machine ou lab, ce qui est parfait pour apprendre et expérimenter.

---

## Les outils de l'écosystème

Dans les prochaines sections de ce chapitre, nous allons explorer concrètement les outils qui permettent d'implémenter ces principes :

### Outils DevOps pour Kubernetes
- **GitLab CI / GitHub Actions / Jenkins** : plateformes d'intégration continue
- **Docker / Buildah** : construction d'images
- **Helm** : gestionnaire de packages Kubernetes
- **Registry** : stockage d'images (addon MicroK8s)

### Outils GitOps
- **ArgoCD** : le plus populaire pour GitOps
- **Flux** : alternative légère
- **Kustomize** : gestion de configurations

### Outils de monitoring
- **Prometheus** : métriques (addon MicroK8s)
- **Grafana** : visualisation
- **Alertmanager** : alertes

---

## Récapitulatif

### Points clés à retenir

**DevOps** est une culture qui vise à :
- Briser les silos entre dev et ops
- Automatiser au maximum
- Livrer fréquemment et rapidement
- Améliorer continuellement

**GitOps** est une méthodologie qui :
- Utilise Git comme source de vérité
- S'appuie sur le déclaratif
- Synchronise automatiquement l'état désiré avec l'état réel
- Apporte traçabilité et sécurité

**Ensemble**, ils permettent de :
- Déployer avec confiance et rapidité
- Maintenir la qualité et la stabilité
- Faciliter la collaboration
- Réduire les erreurs humaines

### Ce qui vous attend

Dans les sections suivantes, vous allez :
1. Mettre en place un **registry privé** sur MicroK8s
2. Configurer un **pipeline CI/CD** complet
3. Déployer **ArgoCD** pour GitOps
4. Créer vos premiers **Helm Charts**
5. Implémenter des **stratégies de déploiement** avancées

Vous allez transformer votre MicroK8s en un véritable **environnement DevOps moderne**, avec tous les outils utilisés en production, mais à l'échelle de votre lab personnel.

---

## Conclusion

DevOps et GitOps ne sont pas de simples buzzwords, mais des approches éprouvées qui ont transformé l'industrie du logiciel. Comprendre ces principes est essentiel avant de plonger dans les aspects techniques.

Avec MicroK8s, vous avez l'opportunité unique d'expérimenter ces pratiques dans un environnement sûr et contrôlé. Vous allez acquérir des compétences directement transférables en environnement professionnel.

Dans la prochaine section, nous allons commencer à mettre en pratique ces concepts en intégrant votre cluster MicroK8s avec Git et en configurant votre premier pipeline automatisé.

**Prêt à automatiser vos déploiements ?** Passons à la section 17.2 : Intégration avec Git !

⏭️ [Intégration avec Git](/17-devops-et-cicd/02-integration-avec-git.md)
