🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 18.1 Requests et Limits

## Introduction

Dans Kubernetes, chaque conteneur dans un Pod consomme des ressources système : principalement du **CPU** et de la **mémoire (RAM)**. Sans contrôle, un conteneur pourrait monopoliser toutes les ressources disponibles sur un nœud, impactant négativement les autres applications.

Les **Requests** et **Limits** sont les mécanismes que Kubernetes met à votre disposition pour gérer intelligemment ces ressources et garantir la stabilité de votre cluster.

## Pourquoi est-ce important ?

Imaginez un immeuble d'appartements où chaque locataire (conteneur) partage les mêmes ressources communes (eau, électricité). Sans règles :
- Un locataire pourrait utiliser toute l'eau disponible
- Les autres locataires seraient privés de ressources
- L'immeuble entier pourrait devenir invivable

Dans Kubernetes, c'est exactement le même principe. Les Requests et Limits permettent de :

✅ **Garantir des ressources minimales** à chaque application
✅ **Éviter qu'une application monopolise tout**
✅ **Améliorer la stabilité globale** du cluster
✅ **Optimiser l'utilisation des ressources** disponibles
✅ **Permettre au scheduler de prendre de meilleures décisions** de placement

## Les deux concepts clés

### 1. Requests (Demandes)

Les **Requests** représentent la **quantité minimale de ressources garantie** à votre conteneur.

**Analogie** : C'est comme réserver une place dans un restaurant. Vous êtes certain d'avoir au moins cette place, même si le restaurant est plein.

**Ce qu'il faut savoir :**
- Le scheduler Kubernetes utilise les requests pour décider sur quel nœud placer votre Pod
- Si un nœud n'a pas suffisamment de ressources disponibles pour satisfaire les requests, le Pod ne sera pas placé sur ce nœud
- Les requests sont **garanties** : votre conteneur aura toujours au minimum ces ressources
- Votre conteneur peut utiliser **plus** que ses requests si des ressources sont disponibles

### 2. Limits (Limites)

Les **Limits** représentent la **quantité maximale de ressources** qu'un conteneur peut utiliser.

**Analogie** : C'est comme un compteur d'eau dans un appartement. Vous ne pouvez pas dépasser un certain débit, même si vous le souhaitez.

**Ce qu'il faut savoir :**
- Les limits empêchent un conteneur de consommer plus que prévu
- Si un conteneur tente de dépasser sa limit, Kubernetes prendra des mesures correctives
- Les limits protègent les autres conteneurs du cluster

## Ressources concernées

Kubernetes gère deux types principaux de ressources :

### CPU (Processeur)

**Unités :**
- Exprimé en "cores" (cœurs)
- `1` = 1 core CPU complet
- `0.5` ou `500m` = un demi-core (m = millicore, 1000m = 1 core)
- `100m` = 10% d'un core

**Comportement :**
- Le CPU est une ressource **compressible**
- Si un conteneur atteint sa limit CPU, il sera simplement **ralenti** (throttled)
- Il ne sera pas tué, juste moins performant

**Exemple de valeurs courantes :**
```yaml
cpu: "100m"   # Application légère (petit service web)
cpu: "500m"   # Application moyenne
cpu: "2"      # Application intensive (traitement de données)
```

### Memory (Mémoire)

**Unités :**
- Exprimé en bytes avec des suffixes
- `128Mi` = 128 Mebibytes (Mi = 1024² bytes)
- `1Gi` = 1 Gibibyte (Gi = 1024³ bytes)
- `256M` = 256 Megabytes (M = 1000² bytes)

**Note** : Préférez toujours les unités binaires (Mi, Gi) qui sont plus précises dans le contexte Kubernetes.

**Comportement :**
- La mémoire est une ressource **non-compressible**
- Si un conteneur dépasse sa limit mémoire, il sera **tué** (OOMKilled - Out Of Memory Killed)
- Le Pod sera ensuite redémarré automatiquement

**Exemple de valeurs courantes :**
```yaml
memory: "128Mi"  # Application très légère
memory: "512Mi"  # Application moyenne
memory: "2Gi"    # Application gourmande (base de données)
```

## Comment ça fonctionne ?

### Schéma conceptuel

```
┌─────────────────────────────────────────────┐
│           Ressources du Nœud                │
│                                             │
│  ┌───────────────────────────────────────┐  │
│  │   Request    │  Usage réel │  Limit   │  │
│  │   (garanti)  │             │  (max)   │  │
│  ├──────────────┼─────────────┼──────────┤  │
│  │      │       │      │      │    │     │  │
│  │      │       │      ▼      │    │     │  │
│  │      ▼       │   Flexible  │    ▼     │  │
│  │   Minimum    │   peut      │ Maximum  │  │
│  │   garanti    │   augmenter │ autorisé │  │
│  └──────────────┴─────────────┴──────────┘  │
└─────────────────────────────────────────────┘
```

### Cas de figure

**Cas 1 : Usage normal**
- Votre conteneur utilise entre ses requests et limits
- Tout va bien, le conteneur fonctionne normalement

**Cas 2 : Sous-utilisation**
- Votre conteneur utilise moins que ses requests
- Les ressources inutilisées peuvent être utilisées par d'autres conteneurs
- Pas de problème, mais vous avez peut-être surestimé les requests

**Cas 3 : Sur-utilisation (CPU)**
- Votre conteneur veut utiliser plus que sa limit CPU
- Kubernetes limite (throttle) l'utilisation du CPU
- Le conteneur ralentit mais continue de fonctionner

**Cas 4 : Sur-utilisation (Memory)**
- Votre conteneur veut utiliser plus que sa limit mémoire
- Kubernetes tue le conteneur (OOMKilled)
- Le Pod redémarre automatiquement

## Configuration dans les manifestes

### Syntaxe de base

Voici comment définir les requests et limits dans un manifeste YAML :

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: mon-application
spec:
  containers:
  - name: mon-conteneur
    image: nginx:latest
    resources:
      requests:
        memory: "64Mi"
        cpu: "250m"
      limits:
        memory: "128Mi"
        cpu: "500m"
```

### Exemple complet avec Deployment

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api-backend
  labels:
    app: api-backend
spec:
  replicas: 3
  selector:
    matchLabels:
      app: api-backend
  template:
    metadata:
      labels:
        app: api-backend
    spec:
      containers:
      - name: api
        image: mon-api:v1.0
        ports:
        - containerPort: 8080
        resources:
          requests:
            memory: "256Mi"
            cpu: "200m"
          limits:
            memory: "512Mi"
            cpu: "500m"
```

### Exemple avec plusieurs conteneurs

Un Pod peut contenir plusieurs conteneurs. Chacun doit avoir ses propres resources :

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: application-avec-sidecar
spec:
  containers:
  # Conteneur principal
  - name: app-principale
    image: mon-app:latest
    resources:
      requests:
        memory: "512Mi"
        cpu: "500m"
      limits:
        memory: "1Gi"
        cpu: "1000m"

  # Conteneur sidecar (proxy)
  - name: proxy-sidecar
    image: envoy:latest
    resources:
      requests:
        memory: "128Mi"
        cpu: "100m"
      limits:
        memory: "256Mi"
        cpu: "200m"
```

**Note importante** : Les requests et limits du Pod sont la **somme** des requests et limits de tous ses conteneurs.

## Que se passe-t-il sans Requests et Limits ?

### Sans Requests

❌ **Problèmes potentiels :**
- Le scheduler ne peut pas prendre de décisions éclairées
- Risque de surcharge d'un nœud (trop de Pods sur un nœud sous-dimensionné)
- Pas de garantie de ressources pour vos applications
- Performances imprévisibles

### Sans Limits

❌ **Problèmes potentiels :**
- Un conteneur peut monopoliser toutes les ressources d'un nœud
- Impact négatif sur les autres applications
- Risque de déstabilisation du nœud entier
- Difficile de prévoir les coûts d'infrastructure

### Sans aucune configuration

C'est le **pire des scénarios** :
- Kubernetes place les Pods un peu au hasard
- Aucune protection contre la surconsommation
- Instabilité globale du cluster
- Difficulté extrême à diagnostiquer les problèmes de performance

## Bonnes pratiques

### 1. Toujours définir les Requests

✅ **Recommandation** : Définissez **toujours** des requests, même approximatives.

```yaml
resources:
  requests:
    memory: "256Mi"
    cpu: "200m"
```

Même si vous n'êtes pas sûr des valeurs exactes, c'est mieux que rien.

### 2. Définir des Limits pour la mémoire

✅ **Recommandation** : Définissez **toujours** une limit mémoire pour éviter les fuites mémoire incontrôlées.

```yaml
resources:
  requests:
    memory: "256Mi"
  limits:
    memory: "512Mi"  # Important !
```

### 3. Limits CPU : à utiliser avec précaution

⚠️ **Attention** : Les limits CPU peuvent causer des problèmes de performance subtils.

**Option 1 - Conservative** : Définir des limits CPU
```yaml
resources:
  requests:
    cpu: "200m"
  limits:
    cpu: "500m"
```

**Option 2 - Permissive** : Ne pas définir de limits CPU (seulement requests)
```yaml
resources:
  requests:
    cpu: "200m"
  # Pas de limit CPU - le conteneur peut burst si nécessaire
```

Cette deuxième approche permet à votre application d'utiliser plus de CPU quand disponible.

### 4. Ratio Requests/Limits raisonnable

✅ **Recommandation** : Gardez un ratio de 1:2 ou 1:3 maximum entre requests et limits.

**Bon exemple** :
```yaml
resources:
  requests:
    memory: "256Mi"
  limits:
    memory: "512Mi"  # Ratio 1:2
```

**Mauvais exemple** :
```yaml
resources:
  requests:
    memory: "64Mi"
  limits:
    memory: "2Gi"  # Ratio 1:32 - trop élevé !
```

### 5. Ajuster selon l'environnement

Les besoins en ressources varient selon l'environnement :

**Développement** :
```yaml
resources:
  requests:
    memory: "128Mi"
    cpu: "100m"
  limits:
    memory: "256Mi"
    cpu: "200m"
```

**Production** :
```yaml
resources:
  requests:
    memory: "512Mi"
    cpu: "500m"
  limits:
    memory: "1Gi"
    cpu: "1000m"
```

### 6. Monitorer et ajuster

✅ **Cycle d'amélioration continue** :

1. **Démarrez** avec des valeurs raisonnables estimées
2. **Monitorez** l'utilisation réelle avec Prometheus/Grafana
3. **Analysez** les métriques sur plusieurs jours/semaines
4. **Ajustez** les valeurs en fonction des observations
5. **Répétez** régulièrement

## Comment déterminer les bonnes valeurs ?

### Méthode 1 : Démarrer avec des valeurs conservatrices

Pour une application que vous ne connaissez pas encore :

```yaml
resources:
  requests:
    memory: "256Mi"
    cpu: "200m"
  limits:
    memory: "512Mi"
    cpu: "500m"
```

### Méthode 2 : Tester en local d'abord

1. Lancez votre application sans limits
2. Utilisez `kubectl top pod` pour observer la consommation
3. Ajoutez une marge de sécurité (ex: +30%)
4. Configurez les requests et limits

```bash
# Observer la consommation
kubectl top pod mon-pod

# Exemple de sortie :
# NAME      CPU(cores)   MEMORY(bytes)
# mon-pod   150m         280Mi
```

Si votre pod utilise 150m CPU et 280Mi RAM :
- Requests : `200m` CPU, `300Mi` RAM (légèrement au-dessus)
- Limits : `400m` CPU, `600Mi` RAM (double pour la marge)

### Méthode 3 : Utiliser le monitoring

Une fois Prometheus installé (voir chapitre 12), vous pouvez analyser :

**Métriques CPU** :
- `container_cpu_usage_seconds_total` : utilisation réelle
- Observez le percentile 95 sur plusieurs jours

**Métriques mémoire** :
- `container_memory_working_set_bytes` : mémoire réellement utilisée
- Observez le maximum atteint

## Patterns courants

### Application web simple (frontend)

```yaml
resources:
  requests:
    memory: "128Mi"
    cpu: "100m"
  limits:
    memory: "256Mi"
    cpu: "200m"
```

### API backend

```yaml
resources:
  requests:
    memory: "512Mi"
    cpu: "250m"
  limits:
    memory: "1Gi"
    cpu: "500m"
```

### Base de données (PostgreSQL, MySQL)

```yaml
resources:
  requests:
    memory: "1Gi"
    cpu: "500m"
  limits:
    memory: "2Gi"
    cpu: "1000m"
```

### Worker de traitement batch

```yaml
resources:
  requests:
    memory: "512Mi"
    cpu: "500m"
  limits:
    memory: "2Gi"
    cpu: "2000m"  # Peut avoir besoin de bursts CPU
```

### Proxy/Sidecar (Envoy, nginx)

```yaml
resources:
  requests:
    memory: "64Mi"
    cpu: "50m"
  limits:
    memory: "128Mi"
    cpu: "100m"
```

## Comprendre les QoS Classes

Kubernetes classe automatiquement vos Pods en 3 catégories de **Quality of Service** (QoS) selon comment vous définissez les requests et limits :

### 1. Guaranteed (Garantie maximale)

**Conditions** :
- Tous les conteneurs ont des requests ET limits
- Requests = Limits pour CPU et mémoire

```yaml
resources:
  requests:
    memory: "256Mi"
    cpu: "500m"
  limits:
    memory: "256Mi"  # Égal à requests
    cpu: "500m"      # Égal à requests
```

**Avantages** :
- Priorité la plus élevée
- Dernier à être évincé en cas de pression sur les ressources
- Performances prévisibles

**Quand l'utiliser** : Applications critiques en production

### 2. Burstable (Flexible)

**Conditions** :
- Au moins un conteneur a des requests ou limits
- Requests ≠ Limits

```yaml
resources:
  requests:
    memory: "128Mi"
    cpu: "250m"
  limits:
    memory: "512Mi"  # Différent de requests
    cpu: "1000m"     # Différent de requests
```

**Avantages** :
- Bon compromis flexibilité/stabilité
- Peut utiliser plus que les requests si disponible

**Quand l'utiliser** : Cas général, la majorité des applications

### 3. BestEffort (Meilleur effort)

**Conditions** :
- Aucun requests ni limits définis

```yaml
# Pas de section resources du tout
```

**Caractéristiques** :
- Priorité la plus basse
- Premier à être évincé en cas de pression
- Aucune garantie de ressources

**Quand l'utiliser** : Jamais en production ! Uniquement pour des tests rapides.

## Vérifier la configuration

### Voir les resources d'un Pod

```bash
kubectl describe pod mon-pod
```

Cherchez la section `Containers` qui affiche :
```
Containers:
  mon-conteneur:
    Limits:
      cpu:     500m
      memory:  512Mi
    Requests:
      cpu:        250m
      memory:     256Mi
```

### Voir la QoS Class

```bash
kubectl get pod mon-pod -o jsonpath='{.status.qosClass}'
```

Résultat possible : `Guaranteed`, `Burstable`, ou `BestEffort`

### Voir l'utilisation actuelle

```bash
kubectl top pod mon-pod
```

Affiche la consommation en temps réel :
```
NAME      CPU(cores)   MEMORY(bytes)
mon-pod   234m         384Mi
```

## Signes que vos valeurs sont incorrectes

### Requests trop basses

**Symptômes** :
- Pods fréquemment évincés (evicted)
- Performances dégradées
- Messages `Insufficient memory` dans les événements

**Solution** : Augmentez les requests

### Requests trop hautes

**Symptômes** :
- Pods en état `Pending` (ne trouvent pas de nœud)
- Faible densité de Pods par nœud
- Sous-utilisation des ressources du cluster

**Solution** : Réduisez les requests

### Limits trop basses (mémoire)

**Symptômes** :
- Pods tués fréquemment (OOMKilled)
- Redémarrages constants
- Messages `OOMKilled` dans `kubectl describe pod`

**Solution** : Augmentez les limits mémoire

### Limits trop basses (CPU)

**Symptômes** :
- Application lente
- Timeouts fréquents
- Métriques montrant un CPU throttling élevé

**Solution** : Augmentez les limits CPU ou retirez-les

## Points clés à retenir

🔑 **Les essentiels** :

1. **Requests** = ressources garanties, utilisées par le scheduler
2. **Limits** = ressources maximum, protection contre la surconsommation
3. **Toujours définir des requests** pour des décisions de scheduling optimales
4. **Toujours définir une limit mémoire** pour éviter les OOMKilled
5. **CPU est compressible** (ralentissement), **mémoire ne l'est pas** (kill)
6. **Monitorer et ajuster** régulièrement selon l'usage réel
7. **QoS Guaranteed** pour les applications critiques
8. **QoS Burstable** pour le cas général

## Conclusion

Les Requests et Limits sont fondamentaux pour un cluster Kubernetes stable et performant. Même si cela peut sembler complexe au début, commencez simplement avec des valeurs raisonnables et affinez au fil du temps grâce au monitoring.

**Prochaine étape** : Une fois que vous maîtrisez les Requests et Limits, vous pouvez aller plus loin avec les Resource Quotas (18.2) et LimitRanges (18.3) pour gérer les ressources au niveau namespace.

N'oubliez pas : il vaut mieux des valeurs approximatives que pas de valeurs du tout !

---

⏭️ [Resource Quotas](/18-gestion-des-ressources/02-resource-quotas.md)
