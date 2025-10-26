🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 23. Dépannage et Maintenance

## Introduction au Chapitre

Bienvenue dans le chapitre le plus pratique de ce tutoriel ! Même avec la meilleure préparation et la configuration la plus soignée, tout cluster Kubernetes rencontrera tôt ou tard des problèmes. Pods qui ne démarrent pas, applications inaccessibles, espace disque saturé, performances dégradées... Ces situations font partie intégrante de la vie d'un administrateur Kubernetes.

Ce chapitre est votre guide de survie pour MicroK8s. Il vous apprendra non seulement à résoudre les problèmes courants, mais surtout à développer une approche méthodique du dépannage qui vous permettra de diagnostiquer et résoudre même les problèmes les plus complexes.

> **Pour les débutants** : Imaginez que vous apprenez à conduire. Savoir démarrer, accélérer et tourner est important, mais savoir quoi faire quand le moteur fait un bruit bizarre, quand un voyant s'allume, ou quand la voiture ne démarre plus est tout aussi crucial. Ce chapitre, c'est votre manuel de dépannage automobile pour Kubernetes. Vous apprendrez à identifier les symptômes, à poser le bon diagnostic, et à appliquer le traitement approprié.

## Pourquoi ce Chapitre est Crucial

### La Réalité des Systèmes de Production

Dans un environnement de production, les problèmes ne surviennent jamais au moment idéal. Ils apparaissent souvent :
- À 2h du matin, quand vous dormez
- Juste avant une démonstration importante
- Pendant un pic de trafic critique
- Quand la personne qui connaît le système est en vacances

**Être préparé fait toute la différence** entre une interruption de service de 5 minutes et un incident majeur de plusieurs heures.

### Au-Delà de la Simple Résolution

Ce chapitre ne vous donne pas simplement des solutions toutes faites. Il vous enseigne :

**1. Une méthodologie** : Comment approcher systématiquement un problème
**2. Des compétences de diagnostic** : Comment lire les symptômes et identifier la cause racine
**3. Des outils** : Quel outil utiliser dans quelle situation
**4. Des réflexes** : Comment réagir rapidement et efficacement
**5. Une maintenance proactive** : Comment prévenir les problèmes avant qu'ils ne surviennent

### Les Trois Piliers du Dépannage

Ce chapitre s'articule autour de trois piliers fondamentaux :

#### 1. **Prévention**
> Mieux vaut prévenir que guérir

- Maintenance régulière
- Surveillance proactive
- Nettoyage planifié
- Mises à jour contrôlées

#### 2. **Diagnostic**
> Comprendre avant d'agir

- Méthodologie systématique
- Outils de diagnostic
- Analyse des symptômes
- Identification de la cause racine

#### 3. **Résolution**
> Agir efficacement

- Solutions éprouvées
- Procédures documentées
- Validation des corrections
- Documentation des incidents

## Structure du Chapitre

Ce chapitre est organisé en 11 sections progressives qui couvrent tous les aspects du dépannage et de la maintenance de MicroK8s.

### Vue d'Ensemble des Sections

#### **23.1 Méthodologie de Dépannage**
*La fondation : comment approcher un problème*

Avant de plonger dans les problèmes spécifiques, cette section établit une approche méthodique et systématique du dépannage. Vous apprendrez à :
- Structurer votre démarche de diagnostic
- Poser les bonnes questions
- Collecter les informations pertinentes
- Éviter les pièges courants
- Documenter vos actions

**Pourquoi commencer ici** : Une bonne méthodologie vous fera gagner du temps sur tous les problèmes, même ceux que nous n'avons pas couverts dans ce tutoriel.

#### **23.2 Problèmes de Pods**
*Les problèmes les plus fréquents*

Les pods sont les briques de base de Kubernetes, et leurs problèmes sont les plus courants. Cette section couvre :
- Pods qui ne démarrent pas (Pending, ImagePullBackOff)
- Pods qui crashent (CrashLoopBackOff, OOMKilled)
- Pods en état d'erreur (Error, Unknown)
- Problèmes de santé (Readiness, Liveness)

**Ce que vous apprendrez** : Diagnostiquer et résoudre 90% des problèmes liés aux pods que vous rencontrerez.

#### **23.3 Problèmes de Réseau**
*Quand la connectivité fait défaut*

Les problèmes réseau sont parmi les plus frustrants à diagnostiquer. Cette section vous guide à travers :
- Services inaccessibles
- Communication inter-pods impossible
- Problèmes DNS
- Problèmes d'Ingress
- Isolement réseau (NetworkPolicies)

**Ce que vous apprendrez** : Méthodologie de diagnostic réseau et outils pour tracer les problèmes de connectivité.

#### **23.4 Problèmes de Configuration**
*Quand les manifests posent problème*

Les erreurs de configuration sont fréquentes et parfois subtiles. Cette section aborde :
- ConfigMaps et Secrets mal configurés
- Variables d'environnement incorrectes
- Erreurs dans les manifests YAML
- Problèmes de syntaxe et de validation
- Incompatibilités de versions d'API

**Ce que vous apprendrez** : Valider, corriger et gérer efficacement vos configurations Kubernetes.

#### **23.5 Problèmes de Certificats**
*Sécurité et authentification*

Les certificats expirés ou mal configurés peuvent paralyser un cluster. Cette section couvre :
- Certificats expirés
- Problèmes d'authentification
- Erreurs TLS/SSL
- Renouvellement de certificats
- Diagnostic d'erreurs de certificats

**Ce que vous apprendrez** : Gérer le cycle de vie des certificats et résoudre les problèmes d'authentification.

#### **23.6 Problèmes de Nœuds**
*Quand l'infrastructure pose problème*

Le nœud est la fondation de votre cluster. Les problèmes incluent :
- Nœuds en état NotReady
- Ressources système insuffisantes
- Problèmes de kubelet
- Pression mémoire/disque/PID
- Problèmes matériels

**Ce que vous apprendrez** : Diagnostiquer et résoudre les problèmes au niveau infrastructure.

#### **23.7 Problèmes de Stockage**
*Persistance et volumes*

Le stockage persistant a ses propres défis. Cette section traite :
- PVC bloqués en Pending
- Pods ne pouvant pas monter les volumes
- Données perdues après redémarrage
- Problèmes de permissions
- Espace disque insuffisant

**Ce que vous apprendrez** : Maîtriser le stockage persistant et résoudre ses problèmes spécifiques.

#### **23.8 Performance et Ressources**
*Optimisation et limites*

Les problèmes de performance nécessitent une approche particulière. Cette section explore :
- Pods bloqués en Pending (ressources insuffisantes)
- OOMKilled (Out Of Memory)
- CPU Throttling
- Pods évictés
- Quality of Service (QoS)

**Ce que vous apprendrez** : Optimiser l'utilisation des ressources et diagnostiquer les problèmes de performance.

#### **23.9 Mise à Jour de MicroK8s**
*Évolution sécurisée*

Les mises à jour sont critiques mais risquées. Cette section guide :
- Comprendre les versions et canaux
- Préparer une mise à jour
- Processus de mise à jour
- Problèmes courants post-mise à jour
- Rollback et restauration

**Ce que vous apprendrez** : Mettre à jour MicroK8s en toute sécurité avec un plan de contingence.

#### **23.10 Nettoyage et Maintenance**
*Hygiène du cluster*

Un cluster propre est un cluster sain. Cette section couvre :
- Nettoyage des images Docker
- Suppression des pods terminés
- Gestion des logs
- Nettoyage des volumes
- Planification de la maintenance

**Ce que vous apprendrez** : Maintenir un cluster propre et performant avec des processus automatisés.

#### **23.11 Outils de Diagnostic Avancés**
*Votre boîte à outils*

Les bons outils font la différence. Cette section présente :
- Kubectl avancé (JSONPath, custom columns)
- K9s (interface interactive)
- Stern (logs multiples)
- Outils réseau (netshoot)
- Outils de sécurité (Trivy, Polaris)
- Scripts personnalisés

**Ce que vous apprendrez** : Utiliser les meilleurs outils pour chaque type de problème.

## Comment Utiliser ce Chapitre

### Pour les Débutants

Si vous débutez avec Kubernetes :

**1. Lisez dans l'ordre** : Les sections s'appuient les unes sur les autres
**2. Commencez par la méthodologie** : La section 23.1 vous donne les fondations
**3. Pratiquez sur un environnement de test** : N'attendez pas d'avoir un problème en production
**4. Gardez ce chapitre comme référence** : Vous y reviendrez souvent

**Temps estimé** : 6-8 heures pour une première lecture complète

### Pour les Utilisateurs Intermédiaires

Si vous avez déjà une expérience Kubernetes :

**1. Parcourez la méthodologie** : Même si vous savez dépanner, une approche structurée aide
**2. Approfondissez vos faiblesses** : Sautez aux sections où vous êtes moins à l'aise
**3. Découvrez les outils** : La section 23.11 vous présentera probablement de nouveaux outils
**4. Adoptez les bonnes pratiques** : Chaque section contient des recommandations précieuses

**Temps estimé** : 3-4 heures pour les sections pertinentes

### Pour les Experts

Si vous êtes déjà un expert Kubernetes :

**1. Utilisez comme référence** : Des solutions rapides pour les problèmes courants
**2. Explorez les spécificités MicroK8s** : Certains problèmes sont spécifiques à MicroK8s
**3. Partagez avec votre équipe** : Excellent matériel de formation
**4. Contribuez** : Vos retours d'expérience enrichiront ce guide

**Temps estimé** : Consultation ponctuelle selon les besoins

## En Situation d'Urgence

### Vous Avez un Problème MAINTENANT ?

**Pas de panique !** Voici un guide de démarrage rapide :

#### 1. **Identifiez le Type de Problème**

**Pods** → Section 23.2
- Mon pod ne démarre pas
- Mon application crashe
- Pod en état d'erreur

**Réseau** → Section 23.3
- Service inaccessible
- DNS ne fonctionne pas
- Problème d'Ingress

**Stockage** → Section 23.7
- PVC en Pending
- Données perdues
- Volume ne monte pas

**Performance** → Section 23.8
- Application lente
- Pod évicté
- OOMKilled

**Nœud** → Section 23.6
- Nœud NotReady
- Cluster inaccessible

#### 2. **Collectez les Informations de Base**

```bash
# État général
microk8s status
microk8s kubectl get nodes
microk8s kubectl get pods --all-namespaces

# Pour un pod spécifique
microk8s kubectl describe pod <nom-pod>
microk8s kubectl logs <nom-pod>

# Événements récents
microk8s kubectl get events --sort-by='.lastTimestamp' | tail -20
```

#### 3. **Consultez la Section Appropriée**

Allez directement à la section correspondant à votre problème. Chaque section contient :
- **Symptômes** : Pour confirmer que c'est bien votre problème
- **Diagnostic** : Étapes pour identifier la cause
- **Solutions** : Corrections éprouvées

#### 4. **Appliquez la Solution**

Suivez les commandes et vérifications proposées. **Important** : Lisez les avertissements avant d'exécuter des commandes.

## Ce Que Vous Allez Apprendre

À la fin de ce chapitre, vous serez capable de :

### Compétences de Diagnostic

✅ **Identifier rapidement** le type de problème (pod, réseau, stockage, etc.)
✅ **Collecter** les informations pertinentes efficacement
✅ **Analyser** les logs et événements pour trouver la cause racine
✅ **Utiliser** les outils appropriés pour chaque situation
✅ **Documenter** vos investigations pour référence future

### Résolution de Problèmes

✅ **Résoudre** 95% des problèmes courants de MicroK8s
✅ **Appliquer** des solutions éprouvées rapidement
✅ **Valider** que le problème est bien résolu
✅ **Prévenir** la récurrence du même problème

### Maintenance Proactive

✅ **Planifier** la maintenance régulière
✅ **Automatiser** les tâches de nettoyage
✅ **Surveiller** les indicateurs de santé
✅ **Mettre à jour** MicroK8s en toute sécurité
✅ **Optimiser** l'utilisation des ressources

### Outils et Automatisation

✅ **Maîtriser** kubectl avancé (JSONPath, custom columns)
✅ **Utiliser** les outils tiers essentiels (k9s, stern, etc.)
✅ **Créer** vos propres scripts de diagnostic
✅ **Automatiser** les vérifications de santé

## Prérequis pour ce Chapitre

### Connaissances Requises

Ce chapitre suppose que vous avez :
- ✅ Installé et configuré MicroK8s (Chapitres 1-5)
- ✅ Compris les concepts de base de Kubernetes (Chapitre 6)
- ✅ Déployé au moins une application (Chapitres 7-10)
- ✅ Une familiarité avec la ligne de commande Linux

Si ce n'est pas le cas, nous vous recommandons de revenir aux chapitres précédents.

### Environnement de Test

Pour pratiquer efficacement :
- **Recommandé** : Un cluster MicroK8s dédié aux tests
- **Pourquoi** : Vous pourrez "casser" des choses sans risque
- **Comment** : Installer MicroK8s sur une VM ou un serveur de test

**Ne testez JAMAIS les commandes destructives sur un environnement de production !**

## Philosophie de ce Chapitre

### Apprendre par la Pratique

Ce chapitre privilégie :
- ✅ **Exemples concrets** : Situations réelles avec solutions détaillées
- ✅ **Commandes complètes** : Copiez-collez directement
- ✅ **Explications claires** : Comprenez ce que vous faites
- ✅ **Bonnes pratiques** : Apprenez la bonne façon de faire

### Approche Progressive

Chaque section suit la même structure :
1. **Introduction** : Comprendre le problème
2. **Symptômes** : Reconnaître les signes
3. **Diagnostic** : Identifier la cause
4. **Solutions** : Résoudre le problème
5. **Prévention** : Éviter la récurrence

### Pas de Magie Noire

Nous expliquons **pourquoi** les solutions fonctionnent, pas seulement **comment** les appliquer. Vous comprendrez les mécanismes sous-jacents.

## Conventions Utilisées

### Symboles et Icônes

Dans ce chapitre, vous verrez ces symboles :

- ✅ **Recommandation** : Bonne pratique à suivre
- ❌ **À éviter** : Erreur commune
- ⚠️ **Attention** : Point important ou risque
- 💡 **Astuce** : Conseil pratique
- 🔍 **Diagnostic** : Étape d'investigation
- 🔧 **Solution** : Correction à appliquer
- 📝 **Note** : Information supplémentaire

### Blocs de Commandes

**Commandes à exécuter** :
```bash
# Cette commande est à exécuter
microk8s kubectl get pods
```

**Résultats attendus** :
```
# Ceci est un exemple de sortie
NAME      READY   STATUS    RESTARTS   AGE
mon-pod   1/1     Running   0          5m
```

**Exemples de configuration** :
```yaml
# Fichier YAML à créer
apiVersion: v1
kind: Pod
metadata:
  name: mon-pod
```

## Conseils Avant de Commencer

### 1. Préparez votre Environnement

**Installez les outils essentiels** :
```bash
# kubectl (inclus avec MicroK8s)
microk8s kubectl version

# jq pour parsing JSON
sudo apt install jq

# yq pour parsing YAML (optionnel)
sudo snap install yq
```

### 2. Configurez les Alias

**Gagnez du temps** :
```bash
# Ajouter à ~/.bashrc
alias k='microk8s kubectl'
alias kgp='microk8s kubectl get pods'
alias kgn='microk8s kubectl get nodes'
alias kdp='microk8s kubectl describe pod'
alias kl='microk8s kubectl logs'
```

### 3. Activez l'Auto-complétion

```bash
# Pour bash
source <(microk8s kubectl completion bash)
echo "source <(microk8s kubectl completion bash)" >> ~/.bashrc

# Avec alias 'k'
complete -F __start_kubectl k
```

### 4. Créez un Namespace de Test

```bash
# Pour vos expérimentations
microk8s kubectl create namespace test-debug
```

### 5. Bookmarkez ce Chapitre

Vous y reviendrez souvent ! Ajoutez-le à vos favoris ou imprimez-le pour référence rapide.

## Ressources Complémentaires

### Documentation Officielle

- [Kubernetes Troubleshooting](https://kubernetes.io/docs/tasks/debug/)
- [MicroK8s Documentation](https://microk8s.io/docs)
- [Kubernetes Issues GitHub](https://github.com/kubernetes/kubernetes/issues)

### Communauté

- [Kubernetes Slack](https://slack.k8s.io/)
- [MicroK8s Forum](https://discuss.kubernetes.io/)
- [Stack Overflow - Kubernetes](https://stackoverflow.com/questions/tagged/kubernetes)

### Outils Mentionnés

Nous couvrons ces outils dans la section 23.11, mais vous pouvez les découvrir maintenant :
- [k9s](https://k9scli.io/) - Interface interactive
- [stern](https://github.com/stern/stern) - Logs multiples
- [kubectx/kubens](https://github.com/ahmetb/kubectx) - Navigation rapide

## Structure d'une Section Type

Pour vous aider à naviguer, chaque section problème suit généralement cette structure :

```
1. Introduction
   └── Contexte et importance

2. Comprendre le Problème
   ├── Définitions
   ├── Concepts clés
   └── Contexte technique

3. Diagnostic
   ├── Symptômes
   ├── Commandes de diagnostic
   └── Analyse des résultats

4. Solutions
   ├── Solution 1 (la plus courante)
   ├── Solution 2
   └── Solution N

5. Prévention
   ├── Bonnes pratiques
   ├── Configuration recommandée
   └── Surveillance

6. Checklist
   └── Points de vérification

7. Résumé
   ├── Commandes essentielles
   └── Points clés
```

## Un Dernier Mot Avant de Commencer

Le dépannage est un art autant qu'une science. Avec ce chapitre, vous avez les connaissances scientifiques : les commandes, les outils, les procédures. L'art viendra avec la pratique et l'expérience.

**Ne vous découragez pas** si un problème vous semble insurmontable au début. Même les experts Kubernetes passent du temps à débugger. La différence est qu'ils ont une méthodologie, des outils, et de l'expérience - exactement ce que ce chapitre vous apporte.

**Adoptez une mentalité d'apprentissage** : Chaque problème est une opportunité d'apprendre quelque chose de nouveau sur Kubernetes et sur votre système.

**Documentez vos solutions** : Quand vous résolvez un problème, notez-le. Vous (ou vos collègues) rencontrerez probablement le même problème plus tard.

**Partagez vos connaissances** : La communauté Kubernetes est basée sur le partage. Quand vous trouvez une solution, partagez-la.

## Prêt à Commencer ?

Vous avez maintenant une vue d'ensemble de ce qui vous attend. Ce chapitre est dense, pratique, et regorge d'informations précieuses qui vous serviront tout au long de votre parcours avec Kubernetes.

**Commencez par la section 23.1** pour apprendre la méthodologie de base, puis progressez selon vos besoins et votre niveau.

Bon dépannage !

---


⏭️ [Méthodologie de dépannage](/23-depannage-et-maintenance/01-methodologie-de-depannage.md)
