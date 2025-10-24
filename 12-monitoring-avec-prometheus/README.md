🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 12. Monitoring avec Prometheus

## Bienvenue dans le monde du monitoring !

Félicitations ! Vous avez parcouru un long chemin depuis le début de cette formation. Vous savez maintenant :
- Installer et configurer MicroK8s
- Déployer des applications dans Kubernetes
- Gérer le stockage persistant
- Exposer vos services avec Ingress
- Sécuriser vos applications avec des certificats SSL

Mais il manque une pièce essentielle du puzzle : **Comment savoir ce qui se passe réellement dans votre cluster ?**

C'est exactement ce que nous allons découvrir dans ce chapitre.

## Le problème : L'aveuglement opérationnel

### Un scénario courant

Imaginez cette situation :
```
09:00 - Vous déployez une nouvelle version de votre application
10:30 - Des utilisateurs signalent que le site est lent
11:00 - L'application devient complètement inaccessible
11:30 - Vous découvrez que l'application consomme toute la mémoire
12:00 - Après investigation, vous réalisez qu'il y avait des signes précurseurs depuis 10h15
```

**Questions qui se posent** :
- Pourquoi n'avez-vous pas détecté le problème plus tôt ?
- Comment l'utilisation mémoire a-t-elle évolué entre 10h15 et 11h00 ?
- Y a-t-il eu d'autres symptômes que vous avez manqués ?
- Ce problème est-il déjà arrivé dans le passé ?

**Le problème** : Sans monitoring, vous êtes **aveugle**. Vous ne savez pas ce qui se passe dans votre infrastructure jusqu'à ce que quelque chose se casse visiblement.

### L'analogie du pilote d'avion

Imaginez piloter un avion sans aucun instrument :
- ❌ Pas d'altimètre (vous ne savez pas votre altitude)
- ❌ Pas de jauge de carburant (vous ne savez pas quand vous tomberez en panne)
- ❌ Pas d'indicateur de vitesse (vous ne savez pas si vous allez trop vite ou trop lent)
- ❌ Pas de radar (vous ne voyez pas les obstacles)

Vous ne pourriez piloter que par temps clair, en regardant par la fenêtre, et en espérant que rien ne se passe mal.

**C'est exactement ce que c'est de gérer une infrastructure sans monitoring.**

Avec des instruments de bord (monitoring), vous savez :
- ✅ Où vous êtes (état actuel de vos ressources)
- ✅ Où vous allez (tendances et prédictions)
- ✅ Si quelque chose ne va pas (alertes)
- ✅ Ce qui s'est passé dans le passé (historique)

## Qu'est-ce que le monitoring ?

### Définition simple

Le **monitoring** (surveillance) est l'action de collecter, stocker, visualiser et alerter sur des **métriques** de votre infrastructure et de vos applications.

**Métriques** : Mesures numériques qui décrivent l'état d'un système.

**Exemples de métriques** :
- Utilisation CPU : 45%
- Mémoire disponible : 2.5 Go
- Requêtes HTTP par seconde : 1500
- Temps de réponse moyen : 250ms
- Erreurs HTTP 5xx : 12 dans la dernière minute

### Les quatre piliers du monitoring

Un bon système de monitoring répond à quatre questions fondamentales :

#### 1. **Quoi ?** (Métriques)
> "Quelles métriques dois-je surveiller ?"

Les métriques sont les données numériques que vous collectez :
- Infrastructure : CPU, RAM, disque, réseau
- Application : Requêtes, latence, erreurs, trafic
- Business : Inscriptions, ventes, utilisateurs actifs

#### 2. **Comment ?** (Collecte)
> "Comment récupérer ces métriques ?"

La collecte peut se faire de plusieurs manières :
- **Pull** : Le système de monitoring va chercher les métriques (approche Prometheus)
- **Push** : Les applications envoient leurs métriques (approche alternative)
- **Logs** : Extraction de métriques depuis les logs

#### 3. **Visualisation** (Dashboards)
> "Comment présenter ces métriques de manière compréhensible ?"

Les dashboards transforment des chiffres bruts en graphiques et visualisations :
- Graphiques temporels (lignes)
- Gauges (jauges)
- Tableaux
- Heatmaps (cartes de chaleur)

#### 4. **Alerting** (Alertes)
> "Comment être informé quand quelque chose ne va pas ?"

Les alertes vous préviennent automatiquement :
- Email
- SMS
- Slack / Teams
- PagerDuty (pour les équipes d'astreinte)

## Pourquoi le monitoring est crucial ?

### 1. Détection précoce des problèmes

**Sans monitoring** :
```
18:00 - Problème commence (mémoire qui monte)
18:30 - Utilisateurs affectés (site ralentit)
19:00 - Panne complète
19:30 - Vous êtes alerté par les utilisateurs
20:00 - Vous commencez à investiguer
```
**Impact** : 2 heures de panne, utilisateurs mécontents

**Avec monitoring** :
```
18:00 - Problème commence
18:05 - Alerte : "Mémoire > 80%"
18:10 - Vous investiguer
18:15 - Problème résolu (scale up ou redémarrage)
```
**Impact** : 15 minutes, aucun utilisateur affecté

### 2. Compréhension de votre système

Le monitoring vous aide à **comprendre** votre infrastructure :
- Quels sont les patterns normaux d'utilisation ?
- Quels sont les pics de trafic habituels ?
- Combien de ressources consomme réellement chaque application ?
- Où sont les goulots d'étranglement ?

**Exemple** : Vous découvrez que votre base de données consomme 90% de la RAM du serveur, alors que vous pensiez que c'était l'application web.

### 3. Optimisation des coûts

Avec des métriques précises :
- ❌ Ne plus sur-provisionner ("au cas où")
- ✅ Dimensionner exactement selon les besoins réels
- ✅ Identifier les ressources sous-utilisées
- ✅ Détecter les fuites mémoire et les inefficacités

**Exemple** : Vous découvrez qu'une application demande 4 Go de RAM mais n'en utilise que 500 Mo. Vous pouvez réduire les ressources et économiser.

### 4. Planification de la capacité

Les métriques historiques aident à **prédire** les besoins futurs :

```promql
# Prédiction : quand le disque sera-t-il plein ?
predict_linear(node_filesystem_free_bytes[7d], 30 * 24 * 3600)
```

Vous pouvez ainsi :
- Prévoir les besoins en infrastructure
- Éviter les pannes dues à la saturation
- Planifier les upgrades au bon moment

### 5. Post-mortem et apprentissage

Quand un incident se produit, le monitoring fournit les **données** pour comprendre :
- Qu'est-ce qui s'est passé exactement ?
- Quand le problème a-t-il commencé ?
- Quels ont été les symptômes ?
- Qu'est-ce qui a déclenché le problème ?

Ces informations sont cruciales pour :
- Écrire un post-mortem détaillé
- Éviter que le problème se reproduise
- Améliorer vos systèmes

### 6. Confiance et sérénité

Avec un bon monitoring :
- ✅ Vous dormez mieux (les alertes vous réveilleront si nécessaire)
- ✅ Vous partez en vacances sereinement
- ✅ Vous savez que les problèmes seront détectés rapidement
- ✅ Vous avez des preuves objectives de la santé de vos systèmes

## Les types de monitoring

### Monitoring d'infrastructure

**Focus** : Les ressources physiques/virtuelles

**Métriques typiques** :
- CPU, RAM, disque, réseau des serveurs
- État des machines virtuelles
- Espace disque disponible
- Bande passante réseau

**Outils** : Node Exporter (que nous verrons dans ce chapitre)

### Monitoring applicatif (APM)

**Focus** : La performance des applications

**Métriques typiques** :
- Temps de réponse des requêtes
- Nombre de requêtes par seconde
- Taux d'erreur
- Throughput (débit)

**Outils** : Instrumentation de l'application, exporters personnalisés

### Monitoring de conteneurs

**Focus** : Les conteneurs et orchestrateurs (Kubernetes)

**Métriques typiques** :
- Utilisation CPU/mémoire par conteneur
- État des pods (running, pending, failed)
- Nombre de restarts
- Événements Kubernetes

**Outils** : cAdvisor, kube-state-metrics (couverts dans ce chapitre)

### Monitoring business

**Focus** : Les métriques métier

**Métriques typiques** :
- Nombre d'inscriptions
- Montant des ventes
- Panier moyen
- Taux de conversion

**Outils** : Métriques custom dans votre application

### Monitoring synthétique

**Focus** : Simulation d'utilisateurs

**Métriques typiques** :
- Temps de chargement de la page d'accueil
- Disponibilité du site (uptime)
- Performance depuis différentes localisations

**Outils** : Blackbox Exporter, outils externes

## Les concepts clés du monitoring

### Observabilité

L'**observabilité** est la capacité de comprendre l'état interne d'un système à partir de ses sorties externes.

Les **trois piliers de l'observabilité** :

1. **Métriques** : Chiffres agrégés dans le temps
   - Exemple : "Utilisation CPU moyenne : 45%"

2. **Logs** : Événements individuels avec contexte
   - Exemple : "2025-10-24 14:32:15 ERROR Connection failed: timeout"

3. **Traces** : Parcours d'une requête à travers le système
   - Exemple : "Requête GET /api/users a traversé : API Gateway → Service Auth → Service Users → Database"

**Prometheus** se concentre principalement sur les **métriques**, mais peut être complété par des outils de logs (ELK, Loki) et de tracing (Jaeger, Zipkin).

### RED Method

La méthode **RED** définit trois métriques clés pour les services :

- **R**ate : Taux de requêtes par seconde
- **E**rrors : Taux d'erreur
- **D**uration : Temps de réponse

**Exemple** :
```
Service API:
- Rate: 1500 req/s
- Errors: 0.2% (3 erreurs par seconde)
- Duration: p95 = 250ms (95% des requêtes < 250ms)
```

Si vous surveillez ces trois métriques, vous avez une bonne vue de la santé d'un service.

### USE Method

La méthode **USE** définit trois métriques pour les ressources :

- **U**tilization : Pourcentage d'utilisation
- **S**aturation : Degré de surcharge
- **E**rrors : Taux d'erreur

**Exemple pour le CPU** :
```
- Utilization: 75% (le CPU est utilisé à 75%)
- Saturation: Load average = 3.5 sur 4 cores (léger queuing)
- Errors: 0 (pas d'erreurs CPU)
```

### SLI, SLO, SLA

Ces acronymes sont importants pour le monitoring professionnel :

**SLI** (Service Level Indicator) : Métrique mesurable
- Exemple : "Disponibilité du service"

**SLO** (Service Level Objective) : Objectif interne
- Exemple : "99.9% de disponibilité"

**SLA** (Service Level Agreement) : Contrat avec les clients
- Exemple : "Nous garantissons 99.5% de disponibilité, sinon remboursement"

**Relation** : SLI mesure → SLO objectif interne → SLA promesse client

## Prometheus : Le standard pour Kubernetes

### Pourquoi Prometheus ?

Pour le monitoring de Kubernetes et des applications cloud-native, **Prometheus** est devenu le standard de facto. Voici pourquoi :

1. **Conçu pour le cloud** : Comprend nativement les environnements dynamiques
2. **Open source** : Libre et gratuit, large communauté
3. **Projet CNCF** : Même fondation que Kubernetes
4. **Écosystème riche** : Des centaines d'exporters et d'intégrations
5. **PromQL** : Langage de requête puissant
6. **Intégration native** : Découverte automatique dans Kubernetes

### Prometheus dans votre parcours

Ce chapitre est **crucial** pour votre évolution :

**Niveau débutant** (où vous êtes) :
- Installation basique
- Compréhension des concepts
- Requêtes simples
- Dashboards de base

**Niveau intermédiaire** (où vous allez) :
- Création d'alertes
- Instrumentation d'applications
- Optimisation des requêtes
- Dashboards avancés

**Niveau avancé** (votre futur) :
- Architecture haute disponibilité
- Recording rules
- Stockage long-terme
- Métriques personnalisées complexes

## Structure de ce chapitre

Ce chapitre est organisé en **8 sections** qui vous guideront progressivement :

### Section 12.1 : Introduction à l'écosystème Prometheus
Vous découvrirez les composants de Prometheus et leur rôle.

### Section 12.2 : Architecture Prometheus
Vous comprendrez comment Prometheus fonctionne en profondeur.

### Section 12.3 : Installation de Prometheus
Vous installerez Prometheus dans MicroK8s en une commande.

### Section 12.4 : Configuration des targets et service discovery
Vous apprendrez comment Prometheus découvre automatiquement ce qu'il doit surveiller.

### Section 12.5 : PromQL : langage de requête Prometheus
Vous maîtriserez le langage pour interroger vos métriques.

### Section 12.6 : Exporters essentiels
Vous découvrirez les trois exporters fondamentaux : node-exporter, kube-state-metrics, et cAdvisor.

### Section 12.7 : Recording rules et optimisation
Vous apprendrez à optimiser Prometheus pour de meilleures performances.

### Section 12.8 : Haute disponibilité Prometheus
Vous explorerez comment rendre Prometheus hautement disponible.

## Ce que vous allez apprendre

À la fin de ce chapitre, vous serez capable de :

✅ **Comprendre** pourquoi le monitoring est essentiel
✅ **Installer** Prometheus dans MicroK8s
✅ **Naviguer** dans l'interface web de Prometheus
✅ **Écrire** des requêtes PromQL pour interroger vos métriques
✅ **Identifier** quelles métriques surveiller pour votre infrastructure
✅ **Configurer** Prometheus pour surveiller vos applications
✅ **Créer** des recording rules pour optimiser les performances
✅ **Comprendre** les concepts de haute disponibilité

## Prérequis pour ce chapitre

Avant de commencer, assurez-vous d'avoir :

✅ **MicroK8s installé et fonctionnel**
```bash
microk8s status
# Devrait afficher: microk8s is running
```

✅ **Au moins une application déployée** (pour avoir quelque chose à monitorer)
- Si vous n'avez rien, ce n'est pas grave, nous surveillerons Kubernetes lui-même

✅ **Ressources système suffisantes**
- CPU : 2+ cores (4 recommandés)
- RAM : 4+ Go (8 Go recommandés)
- Disque : 10+ Go libres

✅ **DNS activé** dans MicroK8s
```bash
microk8s enable dns
```

## Philosophie de l'apprentissage

### Progressif et pratique

Ce chapitre suit une approche **progressive** :
1. Concepts d'abord (pourquoi ?)
2. Installation ensuite (comment ?)
3. Pratique immédiate (essayer)
4. Approfondissement (maîtriser)

### Pas de panique !

Le monitoring peut sembler intimidant au début :
- Beaucoup de nouveaux concepts
- Un nouveau langage de requête (PromQL)
- Des centaines de métriques disponibles

**C'est normal !** Personne ne maîtrise Prometheus en un jour.

**Notre approche** :
- Commencer simple
- Apprendre par la pratique
- Progresser à votre rythme
- Revenir aux concepts quand nécessaire

### L'état d'esprit à adopter

> "Je n'ai pas besoin de tout comprendre immédiatement. Je vais apprendre les bases, pratiquer, et approfondir progressivement."

**Objectif réaliste pour ce chapitre** :
- Comprendre ce qu'est Prometheus ✅
- Savoir l'installer ✅
- Pouvoir exécuter des requêtes simples ✅
- Être capable de lire des dashboards ✅

Vous n'avez **pas** besoin de :
- Mémoriser toutes les fonctions PromQL ❌
- Connaître toutes les métriques par cœur ❌
- Créer des requêtes complexes immédiatement ❌

Ces compétences viendront avec le temps et la pratique.

## Un dernier mot avant de commencer

Le monitoring transforme votre façon de gérer l'infrastructure. Vous passerez de **réactif** ("Oh non, quelque chose est cassé !") à **proactif** ("Je vois un problème qui arrive, je vais le résoudre avant qu'il ne devienne critique").

C'est une compétence qui :
- Vous rendra **plus efficace** dans votre travail
- Vous donnera de la **confiance** dans vos systèmes
- Vous permettra de **dormir tranquille**
- Est **très recherchée** sur le marché du travail

**Prêt à ouvrir les yeux sur votre infrastructure ?**

C'est parti ! Direction la section 12.1 où nous découvrirons l'écosystème Prometheus dans son ensemble.

---

**Note importante** : Prenez votre temps avec ce chapitre. Le monitoring est un sujet vaste. Mieux vaut bien comprendre les bases que de survoler rapidement des concepts avancés. Vous pouvez toujours revenir aux sections précédentes si quelque chose n'est pas clair.

**Bonne formation !** 🚀📊

⏭️ [Introduction à l'écosystème Prometheus](/12-monitoring-avec-prometheus/01-introduction-a-lecosysteme-prometheus.md)
