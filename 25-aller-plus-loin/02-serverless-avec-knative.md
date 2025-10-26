🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 25.2 Serverless avec Knative

## Introduction : Qu'est-ce que le Serverless ?

Le terme "serverless" (sans serveur) est un peu trompeur : il y a toujours des serveurs, mais **vous n'avez plus à vous en soucier**. C'est une approche où vous vous concentrez uniquement sur votre code, et l'infrastructure s'occupe automatiquement de tout le reste.

### Le problème traditionnel

Imaginons que vous avez développé une API de traitement d'images. Avec une approche traditionnelle :

1. Vous devez **provisionner des serveurs** (ou des Pods dans Kubernetes)
2. Votre application **tourne en permanence**, même sans requêtes
3. Vous payez pour ces ressources **24h/24, 7j/7**
4. Si vous avez un pic de trafic, vous devez **scaler manuellement** (ou configurer l'autoscaling)
5. Si vous n'avez plus de trafic, vos Pods **continuent de consommer des ressources**

### La promesse du Serverless

Avec le serverless :

1. Votre code **ne s'exécute que lorsqu'il est sollicité** (event-driven)
2. Quand il n'y a pas de requêtes, **il ne consomme aucune ressource**
3. L'infrastructure **scale automatiquement** de 0 à N instances
4. Vous ne payez que pour le **temps d'exécution réel**
5. Démarrage très rapide (**scale to zero** en quelques secondes)

```
Approche traditionnelle :
[3 Pods actifs] → Consomment toujours des ressources, même sans trafic

Approche serverless :
Pas de trafic    : [0 Pod]  → Aucune ressource consommée ✅
Requête arrive   : [1 Pod]  → Démarre automatiquement en <1s
10 requêtes      : [10 Pods] → Scale automatiquement
Fin du trafic    : [0 Pod]  → Retour à zéro après inactivité
```

## Knative : Le Serverless sur Kubernetes

**Knative** (prononcé "kay-native") est une plateforme open-source qui apporte les capacités serverless à Kubernetes. Développé initialement par Google, elle est maintenant un projet de la Cloud Native Computing Foundation (CNCF).

### Pourquoi Knative ?

Au lieu d'utiliser des services serverless propriétaires (AWS Lambda, Azure Functions, Google Cloud Functions), Knative vous permet de :

- **Rester sur Kubernetes** : Pas de migration vers une autre plateforme
- **Éviter le vendor lock-in** : Votre code fonctionne partout où Kubernetes tourne
- **Utiliser vos compétences existantes** : Même concepts que Kubernetes
- **Avoir le contrôle** : Vous gérez votre infrastructure
- **Économiser** : Scale to zero = pas de gaspillage de ressources

### Les deux faces de Knative

Knative se compose de deux modules principaux (qu'on peut installer séparément) :

#### 1. Knative Serving

C'est le cœur du serverless sur Kubernetes. Il permet de :
- Déployer des applications qui **scale automatiquement**
- Gérer le **scale-to-zero** (réduction à zéro instance)
- Router le trafic intelligemment
- Gérer les révisions et le **déploiement progressif**

**Cas d'usage typiques** :
- APIs REST qui ont du trafic variable
- Webhooks
- Applications web avec traffic fluctuant
- Traitements déclenchés par des événements

#### 2. Knative Eventing

Gestion des événements pour construire des architectures event-driven :
- **Sources d'événements** : Kafka, Google Cloud Pub/Sub, GitHub, etc.
- **Brokers** : Distribution des événements
- **Triggers** : Déclenchement de fonctions sur événements
- **Channels** : Canaux de communication entre composants

**Cas d'usage typiques** :
- Traitement de flux de données
- Réaction à des événements externes (nouveau fichier, webhook GitHub)
- Architectures de microservices réactifs

## Concepts clés de Knative Serving

### 1. Service Knative (KService)

Un Service Knative est une abstraction de haut niveau qui encapsule tout ce dont vous avez besoin :

```yaml
apiVersion: serving.knative.dev/v1
kind: Service
metadata:
  name: hello-world
spec:
  template:
    spec:
      containers:
      - image: gcr.io/knative-samples/helloworld-go
        env:
        - name: TARGET
          value: "Monde"
```

Avec ce simple manifeste, Knative crée automatiquement :
- Une **Route** : Point d'entrée HTTP(S) pour votre service
- Une **Configuration** : Définit l'état désiré de votre application
- Une ou plusieurs **Révisions** : Snapshots immuables de votre code
- Un **Deployment** : Gestion des Pods sous-jacents

### 2. Révisions : Versioning automatique

Chaque fois que vous mettez à jour votre Service Knative, une nouvelle **Révision** est créée automatiquement :

```
Service "mon-api"
├── Révision v1 (100% du trafic) → Déployée le 01/10/2025
├── Révision v2 (0% du trafic)   → Déployée le 15/10/2025
└── Révision v3 (0% du trafic)   → Déployée le 25/10/2025 (actuelle)
```

Vous pouvez :
- **Garder plusieurs révisions actives** simultanément
- **Router le trafic** entre différentes révisions (canary, blue-green)
- **Revenir en arrière** instantanément vers une révision précédente
- **Tester une nouvelle version** avant de basculer le trafic

### 3. Autoscaling intelligent

Knative Serving inclut un autoscaler sophistiqué appelé **KPA (Knative Pod Autoscaler)** :

#### Scale-to-Zero
Quand il n'y a pas de requêtes pendant un certain temps (par défaut 60 secondes), Knative réduit le nombre de Pods à zéro :

```
Minute 0-5  : [3 Pods] → Traitement de requêtes
Minute 5-6  : [2 Pods] → Trafic diminue
Minute 6-7  : [1 Pod]  → Peu de requêtes
Minute 7+   : [0 Pod]  → Aucune requête = Scale to zero ✅
```

#### Cold Start et Activation

Quand une requête arrive sur un service à zéro instance :

```
1. Requête HTTP arrive
2. Activator Knative intercepte la requête
3. Un Pod démarre rapidement (<3 secondes généralement)
4. Requête transmise au Pod
5. Réponse envoyée au client
```

Ce délai de démarrage s'appelle un **"cold start"**. Knative l'optimise au maximum.

#### Scale-Up automatique

Knative surveille plusieurs métriques pour décider du scaling :

- **Concurrency** : Nombre de requêtes simultanées par Pod
- **RPS** (Requests Per Second) : Requêtes par seconde
- **CPU/Memory** : Utilisation des ressources

```yaml
apiVersion: serving.knative.dev/v1
kind: Service
metadata:
  name: mon-api
spec:
  template:
    metadata:
      annotations:
        # Cible : 10 requêtes simultanées par Pod max
        autoscaling.knative.dev/target: "10"
        # Scale to zero après 30s d'inactivité
        autoscaling.knative.dev/scale-to-zero-pod-retention-period: "30s"
        # Minimum 0 Pods (scale to zero activé)
        autoscaling.knative.dev/min-scale: "0"
        # Maximum 50 Pods
        autoscaling.knative.dev/max-scale: "50"
    spec:
      containers:
      - image: mon-image:v1
```

### 4. Traffic Splitting : Déploiement progressif

Une fonctionnalité puissante de Knative est le **traffic splitting** (répartition du trafic) entre révisions :

#### Canary Deployment
Déployer progressivement une nouvelle version :

```yaml
apiVersion: serving.knative.dev/v1
kind: Service
metadata:
  name: mon-service
spec:
  traffic:
  - revisionName: mon-service-v1
    percent: 90       # 90% du trafic sur v1
  - revisionName: mon-service-v2
    percent: 10       # 10% du trafic sur v2 (canary)
```

#### Blue-Green Deployment
Basculement instantané entre deux versions :

```yaml
spec:
  traffic:
  - revisionName: mon-service-blue
    percent: 100      # Tout le trafic sur blue
  - revisionName: mon-service-green
    percent: 0        # Green prêt mais pas de trafic
    tag: staging      # Accessible via URL distincte
```

La révision "green" est accessible via une URL spéciale (https://staging-mon-service...) pour tester.

## Knative Eventing : Architecture événementielle

### Qu'est-ce qu'une architecture événementielle ?

Au lieu d'appeler directement des services (approche synchrone), les composants **émettent et consomment des événements** (approche asynchrone) :

```
Approche traditionnelle (synchrone) :
Service A → appelle directement → Service B

Approche événementielle (asynchrone) :
Service A → émet événement → [Broker] → déclenche → Service B
```

**Avantages** :
- **Découplage** : Les services ne se connaissent pas directement
- **Résilience** : Si B est down, l'événement est conservé
- **Scalabilité** : Plusieurs consommateurs peuvent traiter en parallèle
- **Flexibilité** : Ajouter de nouveaux consommateurs sans modifier A

### Composants de Knative Eventing

#### 1. Event Sources (Sources d'événements)

Les sources génèrent des événements depuis des systèmes externes :

- **KafkaSource** : Événements depuis Apache Kafka
- **GitHub Source** : Webhooks GitHub (push, pull request, issue)
- **CronJobSource** : Événements planifiés (comme un cron)
- **PingSource** : Événements réguliers simples
- **ContainerSource** : Événements depuis n'importe quel conteneur

Exemple conceptuel :
```yaml
apiVersion: sources.knative.dev/v1
kind: PingSource
metadata:
  name: heartbeat
spec:
  schedule: "*/5 * * * *"  # Toutes les 5 minutes
  data: '{"message": "Hello every 5 minutes!"}'
  sink:
    ref:
      apiVersion: serving.knative.dev/v1
      kind: Service
      name: mon-service-handler
```

#### 2. Brokers

Un **Broker** est un hub central qui reçoit tous les événements et les distribue :

```
[Source 1] ──┐
[Source 2] ──┼──→ [Broker] ──┬──→ [Service A]
[Source 3] ──┘               ├──→ [Service B]
                             └──→ [Service C]
```

#### 3. Triggers

Les **Triggers** définissent quels événements vont vers quels services :

```yaml
apiVersion: eventing.knative.dev/v1
kind: Trigger
metadata:
  name: mon-trigger-github
spec:
  broker: default
  filter:
    attributes:
      type: dev.knative.github.push  # Ne réagir qu'aux "push"
      source: github.com/mon-repo    # Depuis ce repo uniquement
  subscriber:
    ref:
      apiVersion: serving.knative.dev/v1
      kind: Service
      name: build-on-push
```

#### 4. Channels et Subscriptions

Alternative aux Brokers, les **Channels** permettent une communication point-à-point ou pub/sub :

```yaml
apiVersion: messaging.knative.dev/v1
kind: Channel
metadata:
  name: ma-chaine
spec:
  channelTemplate:
    apiVersion: messaging.knative.dev/v1
    kind: InMemoryChannel
```

## Cas d'usage pratiques

### 1. API web avec trafic variable

**Scénario** : Vous avez une API qui reçoit beaucoup de requêtes pendant la journée, mais très peu la nuit.

**Avec Knative** :
- La nuit : 0 Pod → Aucune ressource consommée
- Le matin : Scale automatique progressif
- Pic de midi : 20 Pods
- Après-midi : 15 Pods
- Soirée : 5 Pods
- Nuit : Retour à 0 Pod

**Économie** : Au lieu de maintenir 20 Pods 24h/24, vous ne payez que pour l'utilisation réelle.

### 2. Traitement d'images à la demande

**Scénario** : Les utilisateurs uploadent des images que vous devez redimensionner.

```
1. Utilisateur uploade une image → [Stockage S3]
2. S3 émet un événement → [Knative EventSource]
3. Événement routé → [Broker Knative]
4. Trigger déclenche → [Service de traitement d'images]
5. Service scale automatiquement selon le nombre d'images
6. Après traitement → Retour à 0 Pod
```

### 3. Webhooks et intégrations

**Scénario** : Recevoir des webhooks de services externes (Stripe, GitHub, etc.)

**Avec Knative** :
- Service à 0 Pod en attente
- Webhook arrive → Démarre un Pod en <1s
- Traite le webhook
- Retourne à 0 Pod après 30s d'inactivité

**Avantage** : Pas besoin de maintenir un service actif en permanence pour des événements sporadiques.

### 4. Tâches planifiées (Cron Jobs améliorés)

**Scénario** : Exécuter des rapports quotidiens, des backups, du nettoyage.

```yaml
apiVersion: sources.knative.dev/v1
kind: CronJobSource
metadata:
  name: backup-quotidien
spec:
  schedule: "0 2 * * *"  # Tous les jours à 2h du matin
  data: '{"type": "daily-backup"}'
  sink:
    ref:
      apiVersion: serving.knative.dev/v1
      kind: Service
      name: backup-service
```

Le service de backup :
- N'existe pas (0 Pod) sauf à 2h du matin
- Démarre automatiquement quand l'événement arrive
- Effectue le backup
- S'arrête après exécution

### 5. Architecture de microservices événementielle

**Scénario** : E-commerce avec plusieurs services découplés.

```
Événement "OrderPlaced" émis par le service Order
    ↓
[Broker Knative]
    ↓
    ├──→ [Trigger 1] → Service Inventory (déduire le stock)
    ├──→ [Trigger 2] → Service Payment (traiter le paiement)
    ├──→ [Trigger 3] → Service Notification (email au client)
    └──→ [Trigger 4] → Service Analytics (statistiques)
```

Chaque service :
- Scale indépendamment
- Peut être développé dans une techno différente
- Démarre seulement quand nécessaire
- Ne connaît pas les autres services

## Installation de Knative sur MicroK8s

### Prérequis

Knative nécessite :
- **Kubernetes 1.28+** (vérifiez avec `microk8s version`)
- Au moins **4 GB de RAM** disponibles
- **Un Ingress controller** (NGINX recommandé)

### Installation pas à pas

#### Étape 1 : Préparer le cluster

```bash
# Activer les addons nécessaires
microk8s enable dns
microk8s enable ingress
microk8s enable storage

# Créer un alias kubectl pour simplifier
alias kubectl='microk8s kubectl'
```

#### Étape 2 : Installer les CRDs Knative

```bash
# Installer les Custom Resource Definitions de Knative Serving
kubectl apply -f https://github.com/knative/serving/releases/download/knative-v1.12.0/serving-crds.yaml

# Installer le core de Knative Serving
kubectl apply -f https://github.com/knative/serving/releases/download/knative-v1.12.0/serving-core.yaml
```

#### Étape 3 : Installer un réseau pour Knative

Knative nécessite une couche réseau. Plusieurs options existent, **Kourier** est la plus simple :

```bash
# Installer Kourier (networking layer)
kubectl apply -f https://github.com/knative/net-kourier/releases/download/knative-v1.12.0/kourier.yaml

# Configurer Knative pour utiliser Kourier
kubectl patch configmap/config-network \
  --namespace knative-serving \
  --type merge \
  --patch '{"data":{"ingress-class":"kourier.ingress.networking.knative.dev"}}'
```

#### Étape 4 : Configurer le DNS (pour un lab)

Pour un environnement de lab, utilisez **Magic DNS** (sslip.io) :

```bash
kubectl apply -f https://github.com/knative/serving/releases/download/knative-v1.12.0/serving-default-domain.yaml
```

Vos services seront accessibles via des URLs comme :
`http://mon-service.default.127.0.0.1.sslip.io`

#### Étape 5 : Vérifier l'installation

```bash
# Vérifier que tous les pods Knative sont Running
kubectl get pods -n knative-serving

# Vous devriez voir :
# - activator
# - autoscaler
# - controller
# - webhook
# - net-kourier-controller (si vous utilisez Kourier)
```

#### Étape 6 (Optionnel) : Installer Knative Eventing

Si vous voulez utiliser la partie événementielle :

```bash
# Installer les CRDs Eventing
kubectl apply -f https://github.com/knative/eventing/releases/download/knative-v1.12.0/eventing-crds.yaml

# Installer le core Eventing
kubectl apply -f https://github.com/knative/eventing/releases/download/knative-v1.12.0/eventing-core.yaml

# Installer un système de messaging (InMemoryChannel pour débuter)
kubectl apply -f https://github.com/knative/eventing/releases/download/knative-v1.12.0/in-memory-channel.yaml

# Installer un broker par défaut
kubectl apply -f https://github.com/knative/eventing/releases/download/knative-v1.12.0/mt-channel-broker.yaml
```

### Déployer votre premier service Knative

```bash
# Créer un fichier hello.yaml
cat <<EOF > hello.yaml
apiVersion: serving.knative.dev/v1
kind: Service
metadata:
  name: hello
spec:
  template:
    spec:
      containers:
      - image: gcr.io/knative-samples/helloworld-go
        env:
        - name: TARGET
          value: "Knative sur MicroK8s"
EOF

# Déployer
kubectl apply -f hello.yaml

# Attendre que le service soit prêt
kubectl get ksvc hello

# Récupérer l'URL du service
kubectl get ksvc hello -o jsonpath='{.status.url}'

# Tester le service
curl $(kubectl get ksvc hello -o jsonpath='{.status.url}')
```

Vous devriez voir : `Hello Knative sur MicroK8s!`

Observer le scale-to-zero :
```bash
# Voir les Pods
kubectl get pods

# Attendre 60-90 secondes sans envoyer de requêtes
# Puis vérifier à nouveau
kubectl get pods
# Les Pods du service hello ont disparu ! (scale to zero)

# Envoyer une nouvelle requête
curl $(kubectl get ksvc hello -o jsonpath='{.status.url}')

# Vérifier immédiatement les Pods
kubectl get pods
# Un nouveau Pod vient d'apparaître (cold start)
```

## Knative CLI : kn

Knative fournit une CLI dédiée **kn** qui simplifie grandement les opérations :

### Installation de kn

```bash
# Télécharger la dernière version
curl -L https://github.com/knative/client/releases/download/knative-v1.12.0/kn-linux-amd64 -o kn

# Rendre exécutable
chmod +x kn

# Déplacer dans le PATH
sudo mv kn /usr/local/bin/

# Vérifier
kn version
```

### Commandes essentielles

```bash
# Lister tous les services Knative
kn service list

# Décrire un service
kn service describe hello

# Créer un service en une commande
kn service create mon-api \
  --image gcr.io/knative-samples/helloworld-go \
  --env TARGET="Mon API" \
  --scale-min 0 \
  --scale-max 10

# Mettre à jour un service
kn service update mon-api --env TARGET="API v2"

# Router le trafic (canary deployment)
kn service update mon-api \
  --traffic hello-00001=50 \
  --traffic hello-00002=50

# Voir les révisions
kn revision list

# Supprimer un service
kn service delete mon-api

# Voir les logs en temps réel
kn service logs mon-api -f
```

La CLI **kn** est beaucoup plus intuitive que de manipuler des fichiers YAML !

## Performance et optimisation

### Optimiser le Cold Start

Le **cold start** (temps de démarrage d'un Pod) peut être réduit :

#### 1. Utiliser des images légères

```dockerfile
# Mauvais : Image lourde (1 GB)
FROM ubuntu:latest

# Bon : Image alpine légère (5 MB)
FROM alpine:latest

# Excellent : Image distroless (2 MB)
FROM gcr.io/distroless/static
```

#### 2. Préconfigurer le scale-min

Pour des services critiques, éviter le scale-to-zero :

```yaml
metadata:
  annotations:
    autoscaling.knative.dev/min-scale: "1"  # Toujours 1 Pod minimum
```

#### 3. Utiliser des readiness probes optimisées

```yaml
spec:
  containers:
  - image: mon-image
    readinessProbe:
      httpGet:
        path: /health
        port: 8080
      initialDelaySeconds: 1  # Démarrage très rapide
      periodSeconds: 1
```

### Optimiser la concurrence

Ajuster le nombre de requêtes simultanées par Pod :

```yaml
metadata:
  annotations:
    # Faible concurrence pour CPU-intensive
    autoscaling.knative.dev/target: "5"

    # Haute concurrence pour I/O-bound
    autoscaling.knative.dev/target: "100"
```

### Limiter les ressources

Définir des limites pour éviter le gaspillage :

```yaml
spec:
  containers:
  - image: mon-image
    resources:
      requests:
        memory: "64Mi"
        cpu: "100m"
      limits:
        memory: "128Mi"
        cpu: "200m"
```

## Monitoring et observabilité

### Métriques Knative

Knative expose automatiquement des métriques Prometheus :

- **Request Count** : Nombre total de requêtes
- **Request Latency** : Latence des requêtes (p50, p95, p99)
- **Request Errors** : Taux d'erreur
- **Pod Count** : Nombre de Pods actifs
- **Cold Start Duration** : Durée des cold starts

### Intégration avec Prometheus et Grafana

Si vous avez Prometheus installé :

```bash
# Activer le monitoring sur MicroK8s
microk8s enable prometheus

# Les métriques Knative sont automatiquement scrapées
```

Dashboards Grafana recommandés :
- **Knative Serving - Revision HTTP Requests** : Métriques par révision
- **Knative Serving - Scaling** : Visualisation du scaling

### Logs

```bash
# Logs d'un service Knative
kubectl logs -l serving.knative.dev/service=mon-service -c user-container

# Avec kn CLI (plus simple)
kn service logs mon-service -f

# Logs de l'autoscaler
kubectl logs -n knative-serving -l app=autoscaler
```

## Limitations et considérations

### ❌ Quand Knative n'est PAS adapté

1. **Applications stateful** : Bases de données, cache, sessions utilisateur
2. **Long-running processes** : Traitement de données qui prend des heures
3. **WebSockets persistantes** : Connexions longue durée
4. **Très haute performance** : Le cold start ajoute toujours un peu de latence

### ⚠️ Points de vigilance

1. **Cold Start** : Toujours quelques secondes de latence à la première requête
2. **Complexité** : Nécessite de bien comprendre l'architecture événementielle
3. **Ressources** : Knative lui-même consomme ~300-500 Mo de RAM
4. **Debugging** : Plus complexe qu'un Deployment classique

### ✅ Quand Knative est idéal

1. **APIs sporadiques** : Webhooks, intégrations, traitements ponctuels
2. **Microservices événementiels** : Architecture découplée
3. **Économie de ressources** : Lab personnel, environnement de dev/test
4. **Prototypage rapide** : Tester rapidement des idées
5. **Background jobs** : Traitement d'images, génération de rapports, etc.

## Comparaison avec d'autres solutions

| Critère | Knative | AWS Lambda | OpenFaaS | Kubeless |
|---------|---------|------------|----------|----------|
| **Plateforme** | Kubernetes | AWS | Kubernetes | Kubernetes |
| **Vendor Lock-in** | ❌ Aucun | ⚠️ Élevé | ❌ Aucun | ❌ Aucun |
| **Maturité** | Mature | Très mature | Mature | Archivé |
| **Communauté** | Grande | Énorme | Moyenne | Petite |
| **Complexité** | Moyenne | Faible | Faible | Moyenne |
| **Scale to zero** | ✅ Oui | ✅ Oui | ✅ Oui | ✅ Oui |
| **Cold start** | ~1-3s | ~100-500ms | ~1-2s | ~1-3s |
| **Eventing natif** | ✅ Oui | ✅ Oui | ❌ Non | ⚠️ Limité |
| **Multi-cloud** | ✅ Oui | ❌ Non | ✅ Oui | ✅ Oui |
| **Coût** | Gratuit | Pay-per-use | Gratuit | Gratuit |

## Bonnes pratiques

### 1. Design des services

```yaml
# ✅ BON : Service stateless, traitement rapide
apiVersion: serving.knative.dev/v1
kind: Service
metadata:
  name: image-resizer
spec:
  template:
    spec:
      containers:
      - image: mon-resizer:v1
        env:
        - name: MAX_IMAGE_SIZE
          value: "10MB"

# ❌ MAUVAIS : Service stateful
# N'utilisez pas Knative pour ça
apiVersion: apps/v1
kind: StatefulSet  # Utilisez un StatefulSet classique
```

### 2. Configuration du scaling

```yaml
metadata:
  annotations:
    # Définir clairement les limites
    autoscaling.knative.dev/min-scale: "0"
    autoscaling.knative.dev/max-scale: "20"
    # Ajuster selon votre charge
    autoscaling.knative.dev/target: "10"
    # Temps avant scale-to-zero
    autoscaling.knative.dev/scale-to-zero-pod-retention-period: "60s"
```

### 3. Gestion des révisions

```yaml
spec:
  template:
    metadata:
      # Donner des noms explicites aux révisions
      name: mon-api-v2-canary
    spec:
      containers:
      - image: mon-api:v2
  traffic:
  - revisionName: mon-api-v1
    percent: 90
  - revisionName: mon-api-v2-canary
    percent: 10
    tag: canary  # URL dédiée pour tester
```

### 4. Monitoring et alerting

```yaml
# Ajouter des labels pour Prometheus
metadata:
  labels:
    team: backend
    service: api
    environment: production
```

### 5. Sécurité

```yaml
spec:
  template:
    spec:
      # Limiter les privilèges
      securityContext:
        runAsNonRoot: true
        runAsUser: 1000
      containers:
      - image: mon-image
        securityContext:
          allowPrivilegeEscalation: false
          readOnlyRootFilesystem: true
```

## Conclusion

Knative apporte les capacités serverless à Kubernetes, permettant de créer des applications qui **scalent automatiquement de 0 à N** en fonction de la demande. C'est particulièrement pertinent pour un lab MicroK8s où les ressources sont limitées.

### Points clés à retenir

1. **Serverless ≠ Sans serveur** : C'est une abstraction qui cache la gestion des serveurs
2. **Scale-to-zero** : La killer feature qui économise les ressources
3. **Knative Serving** : Pour les applications HTTP qui scalent automatiquement
4. **Knative Eventing** : Pour les architectures événementielles découplées
5. **Révisions** : Versioning automatique et traffic splitting natif
6. **Cold Start** : Il y a toujours un petit délai au démarrage (1-3s)
7. **Pas pour tout** : Réservez Knative aux cas d'usage appropriés

### Recommandations pour MicroK8s

**Pour apprendre** :
- Installez Knative Serving uniquement pour commencer
- Utilisez la CLI **kn** pour la simplicité
- Déployez quelques services simples pour comprendre le scale-to-zero
- Observez les métriques avec Prometheus/Grafana

**Pour un usage productif** :
- Identifiez les services qui ont un trafic variable
- Commencez par 1-2 services non-critiques
- Configurez des min-scale pour les services critiques (éviter cold start)
- Monitorer attentivement les cold starts et ajuster

**Cas d'usage parfaits pour un lab** :
- Webhooks (GitHub, Discord, Slack)
- APIs de développement
- Tâches planifiées (rapports, backups)
- Environnements de test

### Prochaines étapes

Après avoir exploré Knative, vous pourriez vous intéresser à :
- **25.3 Operators et Custom Resources** : Étendre Kubernetes
- **17.5 ArgoCD pour GitOps** : Déploiement déclaratif et automatisé
- **25.1 Service Mesh** : Combiner Knative avec Istio/Linkerd pour plus de contrôle

---

**Ressources complémentaires** :

- Documentation officielle Knative : https://knative.dev/docs/
- Knative Cookbook : https://knative.dev/docs/samples/
- Knative CLI (kn) : https://knative.dev/docs/client/
- Awesome Knative : Liste de ressources communautaires
- Cloud Native Computing Foundation : Projets serverless

⏭️ [Operators et Custom Resources](/25-aller-plus-loin/03-operators-et-custom-resources.md)
