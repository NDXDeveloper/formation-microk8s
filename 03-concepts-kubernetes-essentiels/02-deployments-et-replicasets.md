🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 3.2 Deployments et ReplicaSets

## Introduction

Dans la section précédente, nous avons découvert les Pods, l'unité de base de Kubernetes. Maintenant, nous allons apprendre comment gérer ces Pods de manière professionnelle et automatisée grâce aux **Deployments** et aux **ReplicaSets**.

Ces deux concepts sont essentiels pour déployer des applications en production de manière fiable et scalable.

## Le problème avec les Pods seuls

### Scénario problématique

Imaginez que vous créez un Pod directement pour votre application web :

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: mon-app-web
spec:
  containers:
  - name: nginx
    image: nginx:1.25
```

**Que se passe-t-il si :**
- Le Pod plante ? → Il ne redémarre pas automatiquement
- Le nœud tombe en panne ? → Le Pod est perdu définitivement
- Vous voulez 5 copies de votre application ? → Vous devez créer 5 Pods manuellement
- Vous voulez mettre à jour l'image ? → Vous devez supprimer et recréer chaque Pod

**Conclusion** : Gérer des Pods directement n'est pas pratique et pas fiable.

### La solution Kubernetes

Kubernetes propose des **contrôleurs** qui gèrent automatiquement vos Pods :
- **ReplicaSet** : maintient un nombre fixe de Pods identiques en fonctionnement
- **Deployment** : gère les ReplicaSets et ajoute des fonctionnalités avancées (mises à jour, rollback)

## Qu'est-ce qu'un ReplicaSet ?

### Définition simple

Un **ReplicaSet** est un contrôleur qui garantit qu'un nombre spécifique de Pods identiques fonctionnent en permanence.

### Analogie : le gardien de Pods

Imaginez un **gardien** dont le travail est de s'assurer qu'il y a toujours exactement 3 vigiles devant un bâtiment :
- Si un vigile part (Pod qui meurt) → le gardien en embauche un nouveau
- Si un vigile en trop arrive → le gardien le renvoie
- Le gardien vérifie constamment qu'il y a bien 3 vigiles

Le **ReplicaSet** fait exactement la même chose avec les Pods.

### Fonctionnement

Un ReplicaSet surveille en permanence :
- **Nombre désiré** : combien de Pods doivent fonctionner (vous le définissez)
- **Nombre actuel** : combien de Pods fonctionnent réellement

**Actions automatiques** :
- Si `actuel < désiré` → crée de nouveaux Pods
- Si `actuel > désiré` → supprime des Pods
- Si un Pod plante → en crée un nouveau immédiatement

### Représentation visuelle

```
┌────────────────────────────────────────────┐
│          ReplicaSet (replicas: 3)          │
│                                            │
│  Surveillance continue :                   │
│  ✓ Nombre désiré: 3 Pods                   │
│  ✓ Nombre actuel: 3 Pods                   │
│                                            │
│  Gère automatiquement :                    │
│  ┌──────┐    ┌──────┐    ┌──────┐          │
│  │ Pod1 │    │ Pod2 │    │ Pod3 │          │
│  └──────┘    └──────┘    └──────┘          │
│                                            │
│  Si Pod2 meurt → crée Pod4 automatiquement │
└────────────────────────────────────────────┘
```

### Manifeste YAML d'un ReplicaSet

```yaml
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: mon-replicaset
  labels:
    app: nginx
spec:
  replicas: 3                      # Nombre de Pods désirés
  selector:                        # Comment identifier les Pods à gérer
    matchLabels:
      app: nginx
  template:                        # Modèle pour créer les Pods
    metadata:
      labels:
        app: nginx                 # Labels des Pods créés
    spec:
      containers:
      - name: nginx
        image: nginx:1.25
        ports:
        - containerPort: 80
```

### Explication détaillée

- **replicas: 3** : Le ReplicaSet maintiendra toujours 3 Pods en fonctionnement

- **selector.matchLabels** : Définit comment le ReplicaSet identifie "ses" Pods
  - Ici, il gère tous les Pods avec le label `app: nginx`
  - C'est crucial pour que le ReplicaSet sache quels Pods surveiller

- **template** : C'est le modèle utilisé pour créer de nouveaux Pods
  - **metadata** : Les labels que chaque Pod créé aura
  - **spec** : La spécification du Pod (conteneurs, images, etc.)

### Important : vous ne créez presque jamais de ReplicaSets directement

Bien que vous puissiez créer des ReplicaSets manuellement, **vous ne le ferez presque jamais** en pratique. Pourquoi ? Parce que les **Deployments** sont bien plus puissants et créent automatiquement des ReplicaSets pour vous.

## Qu'est-ce qu'un Deployment ?

### Définition simple

Un **Deployment** est un contrôleur de niveau supérieur qui gère des ReplicaSets et fournit des fonctionnalités avancées pour déployer et mettre à jour vos applications.

### Pourquoi utiliser un Deployment plutôt qu'un ReplicaSet ?

Les Deployments offrent des fonctionnalités essentielles que les ReplicaSets n'ont pas :

1. **Mises à jour progressives** (rolling updates)
   - Mise à jour de votre application sans interruption de service
   - Remplace progressivement les anciens Pods par des nouveaux

2. **Rollback automatique**
   - Retour à une version précédente en cas de problème
   - Historique des versions

3. **Gestion déclarative**
   - Vous décrivez l'état désiré, Kubernetes fait le reste
   - Simplifie grandement les opérations

4. **Pause et reprise**
   - Possibilité de suspendre temporairement le déploiement

### Analogie : le chef d'orchestre

Si le **ReplicaSet** est le gardien qui maintient le bon nombre de Pods, le **Deployment** est le **chef d'orchestre** qui :
- Décide quand et comment changer de version
- Coordonne la transition en douceur
- Garde une trace de l'historique
- Peut revenir en arrière si nécessaire

### Architecture : Deployment → ReplicaSet → Pods

Voici comment ces trois composants interagissent :

```
┌─────────────────────────────────────────────────┐
│              DEPLOYMENT                         │
│  - Gère les mises à jour                        │
│  - Historique des versions                      │
│  - Stratégie de déploiement                     │
└────────────┬────────────────────────────────────┘
             │ crée et gère
             v
┌─────────────────────────────────────────────────┐
│           REPLICASET (version 1)                │
│  - Maintient 3 Pods                             │
│  - Image: nginx:1.25                            │
└────────────┬────────────────────────────────────┘
             │ crée et surveille
             v
      ┌──────┐    ┌──────┐    ┌──────┐
      │ Pod1 │    │ Pod2 │    │ Pod3 │
      └──────┘    └──────┘    └──────┘
```

**Lors d'une mise à jour vers nginx:1.26 :**

```
┌─────────────────────────────────────────────────┐
│              DEPLOYMENT                         │
│  Ordonne la mise à jour                         │
└───────┬─────────────────────────┬───────────────┘
        │                         │
        v                         v
┌───────────────────┐    ┌───────────────────┐
│ ReplicaSet v1     │    │ ReplicaSet v2     │
│ nginx:1.25        │    │ nginx:1.26        │
│ 3→2→1→0 Pods      │    │ 0→1→2→3 Pods      │
└───────────────────┘    └───────────────────┘
     Progressivement          Progressivement
     diminué                  augmenté
```

### Manifeste YAML d'un Deployment

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mon-deployment-nginx
  labels:
    app: nginx
spec:
  replicas: 3                      # Nombre de Pods désirés
  selector:                        # Sélecteur de Pods
    matchLabels:
      app: nginx
  template:                        # Modèle de Pod
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.25
        ports:
        - containerPort: 80
```

**Remarque** : La structure est presque identique à celle d'un ReplicaSet ! La différence réside dans les fonctionnalités avancées qu'offre le Deployment.

## Création et gestion d'un Deployment

### Créer un Deployment

Pour créer ce Deployment dans votre cluster MicroK8s :

```bash
microk8s kubectl apply -f mon-deployment.yaml
```

Ou directement via la ligne de commande :

```bash
microk8s kubectl create deployment nginx-app --image=nginx:1.25 --replicas=3
```

### Vérifier le statut du Deployment

```bash
microk8s kubectl get deployments
```

Sortie exemple :
```
NAME                  READY   UP-TO-DATE   AVAILABLE   AGE
mon-deployment-nginx  3/3     3            3           2m
```

**Explication** :
- **READY** : 3/3 signifie que 3 Pods sur 3 sont prêts
- **UP-TO-DATE** : Nombre de Pods à la dernière version
- **AVAILABLE** : Nombre de Pods disponibles pour recevoir du trafic
- **AGE** : Temps écoulé depuis la création

### Voir les ReplicaSets créés

```bash
microk8s kubectl get replicasets
```

Sortie exemple :
```
NAME                            DESIRED   CURRENT   READY   AGE
mon-deployment-nginx-7f5d8c9b   3         3         3       2m
```

Vous voyez que le Deployment a automatiquement créé un ReplicaSet avec un nom généré.

### Voir les Pods créés

```bash
microk8s kubectl get pods
```

Sortie exemple :
```
NAME                                  READY   STATUS    RESTARTS   AGE
mon-deployment-nginx-7f5d8c9b-8kx2m   1/1     Running   0          2m
mon-deployment-nginx-7f5d8c9b-m4p9n   1/1     Running   0          2m
mon-deployment-nginx-7f5d8c9b-zt7q4   1/1     Running   0          2m
```

**Remarque** : Le nom de chaque Pod contient :
1. Le nom du Deployment
2. Le hash du ReplicaSet
3. Un identifiant unique aléatoire

## Stratégies de déploiement

Les Deployments supportent différentes stratégies pour mettre à jour vos applications.

### 1. RollingUpdate (par défaut)

**Description** : Remplace progressivement les anciens Pods par des nouveaux.

**Avantages** :
- Aucune interruption de service
- Possibilité de rollback rapide
- Détection des problèmes progressivement

**Configuration** :

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mon-deployment
spec:
  replicas: 10
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 2        # Max de Pods indisponibles pendant la mise à jour
      maxSurge: 2              # Max de Pods supplémentaires créés temporairement
  template:
    # ... spécification du Pod
```

**Explication des paramètres** :

- **maxUnavailable: 2**
  - Maximum 2 Pods peuvent être indisponibles pendant la mise à jour
  - Garantit qu'au moins 8 Pods (10 - 2) restent disponibles
  - Peut être un nombre absolu (2) ou un pourcentage (20%)

- **maxSurge: 2**
  - Maximum 2 Pods supplémentaires peuvent être créés temporairement
  - Pendant la mise à jour, il peut y avoir jusqu'à 12 Pods (10 + 2)
  - Accélère la mise à jour mais utilise plus de ressources

**Processus de mise à jour** :

```
État initial: 10 Pods v1
┌──┐┌──┐┌──┐┌──┐┌──┐┌──┐┌──┐┌──┐┌──┐┌──┐

Étape 1: Créer 2 nouveaux Pods v2 (maxSurge: 2)
┌──┐┌──┐┌──┐┌──┐┌──┐┌──┐┌──┐┌──┐┌──┐┌──┐[v2][v2]

Étape 2: Supprimer 2 anciens Pods v1 (maxUnavailable: 2)
┌──┐┌──┐┌──┐┌──┐┌──┐┌──┐┌──┐┌──┐[v2][v2]

Étape 3: Répéter jusqu'à ce que tous soient v2
[v2][v2][v2][v2][v2][v2][v2][v2][v2][v2]
```

### 2. Recreate

**Description** : Supprime tous les anciens Pods avant de créer les nouveaux.

**Avantages** :
- Simple et rapide
- Pas de versions mixtes pendant la mise à jour

**Inconvénients** :
- **Interruption de service** (downtime)
- Tous les Pods sont arrêtés en même temps

**Configuration** :

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mon-deployment
spec:
  replicas: 3
  strategy:
    type: Recreate              # Stratégie Recreate
  template:
    # ... spécification du Pod
```

**Quand l'utiliser** :
- Applications qui ne supportent pas plusieurs versions simultanées
- Environnements de développement/test où le downtime est acceptable
- Migrations de base de données nécessitant l'arrêt complet

**Processus de mise à jour** :

```
État initial: 3 Pods v1
┌───┐┌───┐┌───┐

Étape 1: Supprimer TOUS les Pods
(vide - downtime)

Étape 2: Créer les nouveaux Pods v2
[v2][v2][v2]
```

## Mise à jour d'un Deployment

### Mettre à jour l'image d'un conteneur

Imaginons que vous voulez mettre à jour nginx de la version 1.25 à 1.26 :

**Méthode 1 : Modifier le fichier YAML**

```yaml
# mon-deployment.yaml
spec:
  template:
    spec:
      containers:
      - name: nginx
        image: nginx:1.26    # Changé de 1.25 à 1.26
```

Puis appliquer les changements :

```bash
microk8s kubectl apply -f mon-deployment.yaml
```

**Méthode 2 : Via la ligne de commande**

```bash
microk8s kubectl set image deployment/mon-deployment-nginx nginx=nginx:1.26
```

### Suivre la progression de la mise à jour

```bash
microk8s kubectl rollout status deployment/mon-deployment-nginx
```

Sortie exemple :
```
Waiting for deployment "mon-deployment-nginx" rollout to finish: 1 out of 3 new replicas have been updated...
Waiting for deployment "mon-deployment-nginx" rollout to finish: 2 out of 3 new replicas have been updated...
Waiting for deployment "mon-deployment-nginx" rollout to finish: 1 old replicas are pending termination...
deployment "mon-deployment-nginx" successfully rolled out
```

### Voir l'historique des révisions

```bash
microk8s kubectl rollout history deployment/mon-deployment-nginx
```

Sortie exemple :
```
REVISION  CHANGE-CAUSE
1         <none>
2         Image updated to nginx:1.26
```

## Rollback : retour en arrière

Si la nouvelle version pose problème, vous pouvez facilement revenir à la version précédente.

### Rollback vers la version précédente

```bash
microk8s kubectl rollout undo deployment/mon-deployment-nginx
```

Kubernetes va automatiquement :
1. Récupérer la configuration de la révision précédente
2. Recréer le ReplicaSet correspondant
3. Effectuer une mise à jour progressive vers l'ancienne version

### Rollback vers une révision spécifique

Si vous voulez revenir à une révision particulière :

```bash
# Revenir à la révision 1
microk8s kubectl rollout undo deployment/mon-deployment-nginx --to-revision=1
```

### Voir les détails d'une révision

```bash
microk8s kubectl rollout history deployment/mon-deployment-nginx --revision=2
```

## Scaling : ajuster le nombre de Pods

Le scaling consiste à augmenter ou diminuer le nombre de Pods en fonction de vos besoins.

### Scaling manuel

**Méthode 1 : Modifier le fichier YAML**

```yaml
spec:
  replicas: 5    # Changé de 3 à 5
```

Puis appliquer :

```bash
microk8s kubectl apply -f mon-deployment.yaml
```

**Méthode 2 : Via la ligne de commande**

```bash
# Passer de 3 à 5 Pods
microk8s kubectl scale deployment/mon-deployment-nginx --replicas=5
```

Kubernetes va immédiatement :
- Créer 2 nouveaux Pods
- Les distribuer sur les nœuds disponibles
- Commencer à les exécuter

### Vérifier le scaling

```bash
microk8s kubectl get deployment mon-deployment-nginx
```

Sortie :
```
NAME                   READY   UP-TO-DATE   AVAILABLE   AGE
mon-deployment-nginx   5/5     5            5           10m
```

### Scaling descendant

Pour réduire le nombre de Pods :

```bash
# Passer de 5 à 2 Pods
microk8s kubectl scale deployment/mon-deployment-nginx --replicas=2
```

Kubernetes va :
- Sélectionner 3 Pods à supprimer (selon des critères de priorité)
- Les arrêter gracieusement
- Ne garder que 2 Pods en fonctionnement

## Labels et Selectors : le lien vital

Les labels et les selectors sont essentiels pour que les Deployments et ReplicaSets fonctionnent correctement.

### Comment ça marche

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mon-deployment
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx              # ← Le Deployment cherche les Pods avec ce label
      tier: frontend
  template:
    metadata:
      labels:
        app: nginx            # ← Les Pods créés auront ces labels
        tier: frontend
    spec:
      containers:
      - name: nginx
        image: nginx:1.25
```

**Règle d'or** : Les labels dans `template.metadata.labels` DOIVENT correspondre aux labels dans `selector.matchLabels`.

### Pourquoi c'est important

Le selector permet au Deployment de :
- Identifier les Pods qu'il doit gérer
- Compter combien de Pods existent actuellement
- Décider s'il faut créer ou supprimer des Pods

### Selectors avancés

Vous pouvez utiliser des selectors plus complexes :

```yaml
selector:
  matchLabels:              # ET logique : tous doivent correspondre
    app: nginx
  matchExpressions:         # Expressions plus avancées
  - key: environment
    operator: In            # Doit être "prod" OU "staging"
    values:
    - prod
    - staging
  - key: version
    operator: NotIn         # Ne doit PAS être "legacy"
    values:
    - legacy
```

## Gestion du cycle de vie des Pods

### Terminaison gracieuse

Quand Kubernetes doit supprimer un Pod (lors d'une mise à jour ou d'un scaling), il suit un processus de terminaison gracieuse :

1. **Retrait du Service** : Le Pod est retiré des endpoints des Services
2. **SIGTERM** : Signal de terminaison envoyé au conteneur
3. **Grace period** : Attente de 30 secondes par défaut pour que l'application se termine proprement
4. **SIGKILL** : Si le conteneur ne s'est pas arrêté, il est tué de force

### Configuration de la période de grâce

```yaml
spec:
  template:
    spec:
      terminationGracePeriodSeconds: 60    # 60 secondes au lieu de 30
      containers:
      - name: nginx
        image: nginx:1.25
```

### Hooks de cycle de vie

Vous pouvez exécuter des actions à des moments précis du cycle de vie :

```yaml
spec:
  template:
    spec:
      containers:
      - name: nginx
        image: nginx:1.25
        lifecycle:
          preStop:                    # Avant l'arrêt du conteneur
            exec:
              command:
              - /bin/sh
              - -c
              - nginx -s quit          # Arrêt gracieux de nginx
```

## Pause et reprise d'un Deployment

### Mettre en pause

Vous pouvez suspendre un Deployment pour effectuer plusieurs modifications sans déclencher de mises à jour multiples :

```bash
microk8s kubectl rollout pause deployment/mon-deployment-nginx
```

Ensuite, vous pouvez faire plusieurs changements :

```bash
microk8s kubectl set image deployment/mon-deployment-nginx nginx=nginx:1.26
microk8s kubectl set resources deployment/mon-deployment-nginx -c=nginx --limits=cpu=200m,memory=512Mi
```

### Reprendre

Une fois toutes les modifications faites, reprenez le déploiement :

```bash
microk8s kubectl rollout resume deployment/mon-deployment-nginx
```

Kubernetes effectuera alors UNE SEULE mise à jour intégrant tous les changements.

## Informations détaillées sur un Deployment

### Commande describe

Pour obtenir des informations détaillées :

```bash
microk8s kubectl describe deployment mon-deployment-nginx
```

Vous verrez :
- La configuration complète
- L'état actuel vs désiré
- Les événements récents
- Les conditions
- Le ReplicaSet associé

### Voir les événements

```bash
microk8s kubectl get events --sort-by=.metadata.creationTimestamp
```

Affiche tous les événements récents du cluster, utile pour le débogage.

## Bonnes pratiques

### 1. Toujours utiliser des Deployments

**Plutôt que** :
- Créer des Pods directement
- Créer des ReplicaSets directement

**Utilisez** :
- Des Deployments pour tout

**Pourquoi** : Les Deployments offrent bien plus de fonctionnalités et de flexibilité.

### 2. Utiliser des labels significatifs

```yaml
metadata:
  labels:
    app: nginx                    # Nom de l'application
    version: v1.25                # Version
    environment: production       # Environnement
    team: frontend                # Équipe responsable
    component: webserver          # Composant
```

### 3. Spécifier des limites de ressources

```yaml
spec:
  template:
    spec:
      containers:
      - name: nginx
        image: nginx:1.25
        resources:
          requests:               # Ressources demandées (minimum)
            memory: "128Mi"
            cpu: "100m"
          limits:                 # Ressources maximales
            memory: "256Mi"
            cpu: "200m"
```

### 4. Définir des readiness et liveness probes

```yaml
spec:
  template:
    spec:
      containers:
      - name: nginx
        image: nginx:1.25
        livenessProbe:            # Vérifie si le conteneur est vivant
          httpGet:
            path: /
            port: 80
          initialDelaySeconds: 10
          periodSeconds: 5
        readinessProbe:           # Vérifie si le conteneur est prêt
          httpGet:
            path: /
            port: 80
          initialDelaySeconds: 5
          periodSeconds: 3
```

### 5. Utiliser RollingUpdate avec précaution

```yaml
strategy:
  type: RollingUpdate
  rollingUpdate:
    maxUnavailable: 1           # Conservateur : maximum 1 Pod indisponible
    maxSurge: 1                 # Économe en ressources
```

### 6. Conserver l'historique des révisions

```yaml
spec:
  revisionHistoryLimit: 10      # Garde les 10 dernières révisions
```

Par défaut, Kubernetes conserve les 10 dernières révisions. Ajustez selon vos besoins.

### 7. Ajouter des annotations pour la traçabilité

```yaml
metadata:
  annotations:
    kubernetes.io/change-cause: "Update nginx to 1.26 for security patch"
    deployment.kubernetes.io/revision: "2"
    contact: "team-frontend@example.com"
```

### 8. Tester avant de déployer en production

Utilisez plusieurs environnements :
- **dev** : tests rapides
- **staging** : tests complets dans un environnement similaire à la production
- **production** : déploiement final

## Dépannage courant

### Le Deployment ne crée pas de Pods

**Vérifications** :

```bash
# Vérifier l'état du Deployment
microk8s kubectl describe deployment mon-deployment-nginx

# Vérifier les événements
microk8s kubectl get events

# Vérifier les ReplicaSets
microk8s kubectl get replicasets
microk8s kubectl describe replicaset <nom-du-replicaset>
```

**Causes courantes** :
- Problème d'image (image inexistante ou inaccessible)
- Ressources insuffisantes sur le cluster
- Problèmes de configuration (labels ne correspondant pas)

### Les Pods sont en état CrashLoopBackOff

**Signification** : Le conteneur démarre puis plante immédiatement, Kubernetes essaie de le redémarrer en boucle.

**Vérifications** :

```bash
# Voir les logs du Pod
microk8s kubectl logs <nom-du-pod>

# Voir les logs du conteneur précédent (après un crash)
microk8s kubectl logs <nom-du-pod> --previous
```

**Causes courantes** :
- Erreur dans le code de l'application
- Configuration manquante (variables d'environnement, ConfigMaps)
- Problème de permissions

### La mise à jour est bloquée

**Vérifications** :

```bash
# Voir le statut du rollout
microk8s kubectl rollout status deployment/mon-deployment-nginx

# Voir les Pods
microk8s kubectl get pods

# Voir les événements
microk8s kubectl describe deployment mon-deployment-nginx
```

**Causes courantes** :
- Readiness probe qui échoue sur les nouveaux Pods
- Ressources insuffisantes pour créer les nouveaux Pods
- Problème avec la nouvelle version de l'image

**Solution** : Rollback

```bash
microk8s kubectl rollout undo deployment/mon-deployment-nginx
```

## Différence entre Deployment et ReplicaSet : tableau récapitulatif

| Caractéristique | ReplicaSet | Deployment |
|----------------|------------|------------|
| Maintient un nombre fixe de Pods | ✓ | ✓ |
| Remplace automatiquement les Pods défaillants | ✓ | ✓ |
| Mises à jour progressives | ✗ | ✓ |
| Rollback | ✗ | ✓ |
| Historique des versions | ✗ | ✓ |
| Stratégies de déploiement | ✗ | ✓ |
| Pause/Reprise | ✗ | ✓ |
| **Utilisation recommandée** | Jamais directement | Toujours |

## Résumé des points clés

- Un **ReplicaSet** maintient un nombre constant de Pods identiques en fonctionnement
- Un **Deployment** est un contrôleur de plus haut niveau qui gère les ReplicaSets
- L'architecture est : **Deployment → ReplicaSet → Pods**
- Les Deployments permettent des **mises à jour progressives** sans interruption de service
- Vous pouvez faire un **rollback** facilement vers une version précédente
- Le **scaling** (augmenter/diminuer le nombre de Pods) est simple et rapide
- Les **labels** et **selectors** sont essentiels pour lier Deployments, ReplicaSets et Pods
- **Stratégies de déploiement** : RollingUpdate (par défaut) et Recreate
- **Toujours utiliser des Deployments** plutôt que des Pods ou ReplicaSets seuls

## Prochaines étapes

Maintenant que vous savez gérer vos Pods avec des Deployments, vous êtes prêt à découvrir :
- **Les Services** : comment exposer vos applications et permettre la communication entre Pods
- **Les Namespaces** : comment organiser et isoler vos ressources
- **Les ConfigMaps et Secrets** : comment gérer la configuration et les données sensibles

Les Deployments sont l'un des concepts les plus importants de Kubernetes. Maîtriser leur utilisation vous permettra de déployer des applications robustes et faciles à maintenir !

⏭️ [Services (ClusterIP, NodePort, LoadBalancer)](/03-concepts-kubernetes-essentiels/03-services-clusterip-nodeport-loadbalancer.md)
