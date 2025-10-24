🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 12.1 Introduction à l'écosystème Prometheus

## Pourquoi le monitoring est essentiel ?

Imaginez que vous avez déployé plusieurs applications dans votre cluster Kubernetes. Tout fonctionne bien... en apparence. Mais comment savoir réellement ce qui se passe à l'intérieur ?

- Votre application consomme-t-elle trop de mémoire ?
- Les requêtes HTTP sont-elles lentes ?
- Un pod redémarre-t-il constamment sans que vous le remarquiez ?
- Votre disque est-il en train de se remplir ?

Sans système de monitoring, vous naviguez à l'aveugle. C'est là qu'intervient **Prometheus**.

## Qu'est-ce que Prometheus ?

Prometheus est un système de monitoring et d'alerting open-source devenu le standard de facto pour surveiller les applications cloud-natives et Kubernetes. Créé par SoundCloud en 2012, il est maintenant un projet de la Cloud Native Computing Foundation (CNCF), tout comme Kubernetes.

### Pourquoi Prometheus est-il si populaire ?

1. **Conçu pour le cloud** : Prometheus comprend nativement les environnements dynamiques comme Kubernetes où les conteneurs apparaissent et disparaissent constamment.

2. **Modèle de collecte actif** : Contrairement à d'autres solutions où les applications envoient leurs données (push), Prometheus vient régulièrement "interroger" (pull) les applications pour récupérer leurs métriques.

3. **Langage de requête puissant** : PromQL (Prometheus Query Language) permet d'analyser et d'agréger les données facilement.

4. **Intégration native avec Kubernetes** : Prometheus découvre automatiquement les services et pods dans votre cluster.

5. **Écosystème riche** : Des centaines d'exporters et d'intégrations sont disponibles.

## L'écosystème Prometheus : Vue d'ensemble

Prometheus n'est pas un outil isolé, c'est tout un écosystème de composants qui travaillent ensemble :

### 1. **Prometheus Server** (Le cœur du système)

C'est le composant principal qui :
- Collecte et stocke les métriques dans une base de données temporelle (time-series database)
- Évalue les règles d'alerte
- Fournit une interface web basique pour visualiser les données
- Expose une API pour que d'autres outils (comme Grafana) puissent requêter les données

**Analogie** : Pensez au serveur Prometheus comme un journaliste qui fait régulièrement le tour de tous les services pour leur demander "Comment vas-tu ? Quels sont tes chiffres du moment ?" et note tout dans son carnet.

### 2. **Exporters** (Les collecteurs de données)

Les exporters sont de petits programmes qui exposent les métriques dans un format que Prometheus peut comprendre. Il en existe pour presque tout :

- **Node Exporter** : Collecte les métriques du système d'exploitation (CPU, RAM, disque, réseau)
- **kube-state-metrics** : Fournit des informations sur l'état des objets Kubernetes (pods, deployments, services)
- **cAdvisor** : Intégré dans Kubelet, il donne des métriques sur les conteneurs
- **Application exporters** : Pour PostgreSQL, MySQL, Redis, NGINX, etc.

**Analogie** : Les exporters sont comme des traducteurs qui convertissent les informations de différentes sources dans un langage que Prometheus comprend.

### 3. **Pushgateway** (Optionnel)

Pour les jobs éphémères (batch, scripts) qui ne vivent pas assez longtemps pour que Prometheus les interroge, le Pushgateway permet de "pousser" les métriques.

**Note pour débutants** : Vous n'aurez probablement pas besoin du Pushgateway au début. C'est un composant optionnel pour des cas d'usage spécifiques.

### 4. **Alertmanager**

Alertmanager reçoit les alertes depuis Prometheus et s'occupe de :
- Regrouper les alertes similaires
- Éliminer les doublons
- Router les alertes vers les bons destinataires (email, Slack, PagerDuty, etc.)
- Gérer les silences (désactivation temporaire d'alertes)

**Analogie** : C'est comme un standard téléphonique intelligent qui reçoit tous les appels d'urgence, évite de vous appeler 10 fois pour le même problème, et sait à qui transférer chaque type d'urgence.

### 5. **Grafana** (Visualisation)

Bien que Grafana ne fasse pas officiellement partie de Prometheus, ils sont presque toujours utilisés ensemble. Grafana crée de magnifiques dashboards à partir des données Prometheus.

**Analogie** : Si Prometheus est le journaliste qui collecte les faits, Grafana est le designer qui transforme ces faits en infographies compréhensibles.

## Comment fonctionne Prometheus ? (Simplifié)

Voici le flux de base :

```
1. Vos applications et systèmes → exposent des métriques sur /metrics
                                  ↓
2. Prometheus Server          → scrape (interroge) régulièrement ces endpoints
                                  ↓
3. Base de données Prometheus → stocke les métriques avec timestamps
                                  ↓
4. Règles d'alerte            → évaluées périodiquement
                                  ↓
5. Alertmanager               → gère et route les alertes
                                  ↓
6. Notifications              → email, Slack, etc.
```

En parallèle :
```
Grafana → requête Prometheus → affiche de beaux graphiques
```

## Les concepts clés à retenir

### Métrique (Metric)
Une mesure numérique d'une ressource ou d'un comportement. Exemples :
- Nombre de requêtes HTTP
- Utilisation CPU en pourcentage
- Température d'un serveur
- Durée d'exécution d'une requête

### Série temporelle (Time Series)
Une métrique avec ses valeurs dans le temps. Chaque série est identifiée par :
- Un **nom de métrique** (ex: `http_requests_total`)
- Des **labels** (ex: `{method="GET", status="200"}`)

### Scraping
L'action de Prometheus d'aller chercher les métriques. Par défaut, Prometheus scrape toutes les 15 secondes.

### Target
Un endpoint (URL) que Prometheus scrape. Par exemple : `http://mon-app:8080/metrics`

## Pourquoi cet écosystème pour votre lab MicroK8s ?

Dans un environnement de lab personnel avec MicroK8s, Prometheus vous apporte :

1. **Visibilité totale** : Vous savez exactement ce qui se passe dans votre cluster
2. **Apprentissage** : Vous comprenez comment vos applications consomment les ressources
3. **Détection précoce** : Vous identifiez les problèmes avant qu'ils ne deviennent critiques
4. **Expérimentation** : Vous pouvez tester l'impact de vos configurations
5. **Professionnalisation** : Vous apprenez les outils utilisés en production dans l'industrie

## L'intégration MicroK8s : La simplicité incarnée

La grande force de MicroK8s, c'est qu'installer Prometheus est aussi simple que :

```bash
microk8s enable prometheus
```

Cette commande unique déploie :
- Le serveur Prometheus
- Alertmanager
- Les exporters essentiels (node-exporter, kube-state-metrics)
- Une configuration par défaut fonctionnelle

**Vous n'avez pas besoin de comprendre tous les détails immédiatement**. MicroK8s s'occupe de la complexité pour vous permettre de vous concentrer sur l'apprentissage progressif.

## Ce que vous allez apprendre dans les prochaines sections

Dans les sections suivantes de ce chapitre, nous allons :

1. **Section 12.2** : Comprendre l'architecture détaillée de Prometheus
2. **Section 12.3** : Installer et configurer Prometheus avec MicroK8s
3. **Section 12.4** : Découvrir comment Prometheus trouve automatiquement vos services
4. **Section 12.5** : Apprendre le langage PromQL pour interroger vos métriques
5. **Section 12.6** : Explorer les exporters essentiels
6. **Section 12.7** : Optimiser Prometheus avec les recording rules
7. **Section 12.8** : Mettre en place la haute disponibilité

## Résumé

Prometheus est **l'outil standard de monitoring pour Kubernetes**. Son écosystème complet vous permet de :
- **Collecter** des métriques depuis vos applications et infrastructure
- **Stocker** ces données de manière efficace
- **Interroger** et analyser vos métriques avec PromQL
- **Alerter** quand quelque chose ne va pas
- **Visualiser** avec Grafana pour une meilleure compréhension

Avec MicroK8s, démarrer avec Prometheus est extrêmement simple, ce qui en fait un excellent choix pour votre lab d'apprentissage.

Dans la prochaine section, nous plongerons dans l'architecture technique de Prometheus pour comprendre comment tous ces composants s'articulent ensemble.

---

**Point clé à retenir** : Ne vous sentez pas submergé par tous ces composants. Comme pour Kubernetes lui-même, vous allez les découvrir progressivement. L'important est de comprendre que Prometheus est une solution complète et intégrée pour surveiller vos applications cloud-native.

⏭️ [Architecture Prometheus](/12-monitoring-avec-prometheus/02-architecture-prometheus.md)
