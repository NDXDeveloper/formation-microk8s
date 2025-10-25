🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 19.1 Scaling Manuel

## Introduction au Scaling

Le **scaling** (mise à l'échelle en français) consiste à augmenter ou diminuer le nombre de répliques de votre application qui s'exécutent simultanément dans votre cluster Kubernetes. C'est l'une des fonctionnalités les plus puissantes de Kubernetes.

### Pourquoi scaler une application ?

Imaginez que vous avez une boutique en ligne. En temps normal, un seul serveur suffit. Mais pendant les soldes ou le Black Friday, le trafic explose et un seul serveur ne peut plus gérer toutes les demandes. C'est là qu'intervient le scaling :

- **Augmenter la capacité** : Plus de pods = plus de capacité à traiter les requêtes
- **Améliorer la disponibilité** : Si un pod tombe en panne, les autres continuent de fonctionner
- **Gérer les pics de charge** : Adapter les ressources en fonction du besoin
- **Optimiser les coûts** : Réduire le nombre de pods quand la charge diminue

## Concepts de Base

### Qu'est-ce qu'un pod ?

Un **pod** est la plus petite unité déployable dans Kubernetes. C'est un conteneur (ou groupe de conteneurs) qui exécute votre application.

### Qu'est-ce qu'un Deployment ?

Un **Deployment** gère plusieurs répliques identiques de votre application. C'est lui que nous allons scaler. Quand vous demandez 5 répliques, Kubernetes crée et maintient 5 pods identiques.

### Répliques vs Pods

- Une **réplique** est une copie de votre application
- Un **pod** est l'instance concrète qui exécute cette réplique
- Le **Deployment** gère automatiquement les pods pour maintenir le nombre de répliques souhaité

## Le Scaling Manuel avec kubectl

Le scaling manuel signifie que **vous** décidez du nombre de répliques, manuellement, via une commande. C'est vous qui contrôlez quand augmenter ou diminuer ce nombre.

### Commande de base

La commande principale pour scaler manuellement est :

```bash
kubectl scale deployment <nom-du-deployment> --replicas=<nombre>
```

**Exemple concret :**

```bash
kubectl scale deployment nginx-web --replicas=5
```

Cette commande demande à Kubernetes de maintenir exactement 5 répliques du déploiement `nginx-web`.

### Vérifier l'état actuel

Avant de scaler, il est utile de voir combien de répliques vous avez actuellement :

```bash
kubectl get deployment nginx-web
```

Sortie typique :

```
NAME        READY   UP-TO-DATE   AVAILABLE   AGE
nginx-web   1/1     1            1           10m
```

Ici, `1/1` signifie : 1 réplique disponible sur 1 demandée.

### Scaler vers le haut (Scale Up)

Pour augmenter le nombre de répliques (par exemple, passer de 1 à 5) :

```bash
kubectl scale deployment nginx-web --replicas=5
```

Kubernetes va alors :
1. Créer 4 nouveaux pods
2. Les démarrer progressivement
3. Vérifier qu'ils sont en bonne santé
4. Les ajouter au service de load balancing

Vérification après scaling :

```bash
kubectl get deployment nginx-web
```

Sortie :

```
NAME        READY   UP-TO-DATE   AVAILABLE   AGE
nginx-web   5/5     5            5           12m
```

Vous pouvez aussi voir les pods individuels :

```bash
kubectl get pods -l app=nginx-web
```

Sortie :

```
NAME                         READY   STATUS    RESTARTS   AGE
nginx-web-7d8c9f5b6d-abcde   1/1     Running   0          2m
nginx-web-7d8c9f5b6d-fghij   1/1     Running   0          2m
nginx-web-7d8c9f5b6d-klmno   1/1     Running   0          2m
nginx-web-7d8c9f5b6d-pqrst   1/1     Running   0          12m
nginx-web-7d8c9f5b6d-uvwxy   1/1     Running   0          2m
```

### Scaler vers le bas (Scale Down)

Pour réduire le nombre de répliques (par exemple, revenir à 2) :

```bash
kubectl scale deployment nginx-web --replicas=2
```

Kubernetes va alors :
1. Sélectionner 3 pods à terminer
2. Arrêter progressivement ces pods
3. Ne conserver que 2 pods actifs

**Important :** Kubernetes choisit intelligemment quels pods arrêter (généralement les plus récents ou ceux sur des nœuds moins chargés).

### Scaler à zéro

Vous pouvez même scaler à zéro pour arrêter complètement une application sans la supprimer :

```bash
kubectl scale deployment nginx-web --replicas=0
```

Cela arrête tous les pods mais conserve la configuration du Deployment. Pour redémarrer, il suffit de scaler à nouveau :

```bash
kubectl scale deployment nginx-web --replicas=3
```

## Scaler avec des fichiers YAML

Au lieu d'utiliser la commande `kubectl scale`, vous pouvez modifier directement le fichier YAML de votre Deployment.

**Fichier original :**

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-web
spec:
  replicas: 1    # Nombre initial de répliques
  selector:
    matchLabels:
      app: nginx-web
  template:
    metadata:
      labels:
        app: nginx-web
    spec:
      containers:
      - name: nginx
        image: nginx:latest
        ports:
        - containerPort: 80
```

**Pour scaler à 5 répliques, modifiez la ligne :**

```yaml
spec:
  replicas: 5    # Changé de 1 à 5
```

Puis appliquez la modification :

```bash
kubectl apply -f nginx-deployment.yaml
```

Cette méthode est préférable pour :
- Garder une trace des modifications dans Git
- Reproduire la même configuration ailleurs
- Documenter votre infrastructure

## Surveillance du Scaling

### Observer le scaling en temps réel

Pour voir le scaling se produire en direct :

```bash
kubectl get pods -l app=nginx-web --watch
```

L'option `--watch` actualise l'affichage en temps réel. Vous verrez les nouveaux pods apparaître avec le statut `ContainerCreating`, puis `Running`.

### Vérifier les événements

Pour voir l'historique des événements liés au scaling :

```bash
kubectl describe deployment nginx-web
```

Dans la section "Events", vous verrez des lignes comme :

```
Events:
  Type    Reason             Age   From                   Message
  ----    ------             ----  ----                   -------
  Normal  ScalingReplicaSet  2m    deployment-controller  Scaled up replica set nginx-web-7d8c9f5b6d to 5
```

### Vérifier la distribution des pods

Si vous avez un cluster multi-nœuds, vérifiez sur quels nœuds les pods sont distribués :

```bash
kubectl get pods -l app=nginx-web -o wide
```

La colonne `NODE` vous montre la distribution.

## Cas d'Usage du Scaling Manuel

### 1. Pic de trafic prévu

Vous savez que votre application va recevoir beaucoup de trafic (promotion, lancement de produit) :

```bash
# Avant l'événement
kubectl scale deployment boutique-en-ligne --replicas=20

# Après l'événement
kubectl scale deployment boutique-en-ligne --replicas=3
```

### 2. Maintenance ou debugging

Pour isoler un problème, réduisez à 1 réplique :

```bash
kubectl scale deployment app-problematique --replicas=1
```

Cela facilite le débogage en évitant d'avoir plusieurs pods qui génèrent des logs en même temps.

### 3. Tests de charge

Pour tester comment votre application se comporte avec plusieurs instances :

```bash
kubectl scale deployment mon-api --replicas=10
```

### 4. Économie de ressources

La nuit ou le week-end, si votre application interne n'est pas utilisée :

```bash
kubectl scale deployment app-interne --replicas=0
```

Le lundi matin :

```bash
kubectl scale deployment app-interne --replicas=5
```

## Limitations du Scaling Manuel

Bien que le scaling manuel soit simple et efficace, il présente des limitations :

### 1. Intervention humaine requise

Vous devez être là pour exécuter la commande. Si un pic de trafic arrive à 3h du matin, votre application peut tomber en panne avant que vous ne réagissiez.

### 2. Pas de réactivité automatique

Le scaling manuel ne s'adapte pas automatiquement à la charge. Vous devez surveiller constamment et ajuster manuellement.

### 3. Risque de sur-provisionnement ou sous-provisionnement

- **Sur-provisionnement** : Trop de répliques = gaspillage de ressources
- **Sous-provisionnement** : Pas assez de répliques = performances dégradées

### 4. Difficulté de prédiction

Il est difficile de prédire avec précision le nombre de répliques nécessaires pour une charge donnée.

## Bonnes Pratiques

### 1. Commencez petit

Ne scalez pas immédiatement à 100 répliques. Augmentez progressivement :
- 1 → 3 répliques
- Observez
- 3 → 5 répliques
- Observez
- Etc.

### 2. Surveillez les ressources

Avant de scaler, vérifiez que votre cluster a suffisamment de ressources :

```bash
kubectl top nodes    # CPU et mémoire par nœud
kubectl top pods     # CPU et mémoire par pod
```

**Note :** Cette commande nécessite le Metrics Server (nous le verrons dans la section 19.5).

### 3. Définissez des requests et limits

Assurez-vous que vos pods ont des `requests` et `limits` de ressources définis :

```yaml
resources:
  requests:
    memory: "128Mi"
    cpu: "100m"
  limits:
    memory: "256Mi"
    cpu: "500m"
```

Cela permet à Kubernetes de mieux gérer le placement des pods.

### 4. Utilisez des nombres impairs pour la haute disponibilité

Pour des applications critiques, utilisez 3, 5, 7 répliques plutôt que 2, 4, 6. Cela aide à prévenir les situations de "split-brain" et améliore la résilience.

### 5. Documentez vos décisions

Notez pourquoi vous avez choisi un certain nombre de répliques :

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: frontend
  annotations:
    scaling-notes: "3 répliques suffisent pour la charge normale. Scaler à 10 pendant les promotions."
spec:
  replicas: 3
  # ...
```

### 6. Testez le scaling en environnement de développement

Avant de scaler en production, testez dans votre lab MicroK8s :
- Vérifiez que l'application fonctionne avec plusieurs répliques
- Observez la consommation de ressources
- Testez le scale up et scale down

### 7. Préparez-vous pour l'autoscaling

Le scaling manuel est un bon point de départ, mais envisagez l'autoscaling (HPA) pour automatiser ce processus. Nous verrons cela dans les sections suivantes.

## Différence entre Scaling et Mise à Jour

Il est important de ne pas confondre :

- **Scaling** : Changer le nombre de répliques (même version de l'application)
- **Mise à jour (Rolling Update)** : Déployer une nouvelle version de l'application

Exemples :

```bash
# Scaling : changer le nombre de répliques
kubectl scale deployment nginx-web --replicas=5

# Mise à jour : changer l'image (nouvelle version)
kubectl set image deployment/nginx-web nginx=nginx:1.21
```

## Commandes Récapitulatives

Voici un résumé des commandes les plus utiles :

```bash
# Scaler un deployment
kubectl scale deployment <nom> --replicas=<nombre>

# Voir l'état des deployments
kubectl get deployments

# Voir l'état détaillé d'un deployment
kubectl describe deployment <nom>

# Voir les pods d'un deployment
kubectl get pods -l app=<nom-app>

# Observer le scaling en temps réel
kubectl get pods --watch

# Voir les événements récents
kubectl get events --sort-by=.metadata.creationTimestamp

# Scaler plusieurs deployments
kubectl scale deployment app1 app2 app3 --replicas=3
```

## Résumé

Le **scaling manuel** est la méthode la plus simple pour ajuster le nombre de répliques de votre application dans Kubernetes. C'est un excellent point de départ pour comprendre comment Kubernetes gère la scalabilité.

**Points clés à retenir :**

1. Le scaling change le nombre de pods d'un Deployment
2. La commande de base est `kubectl scale deployment <nom> --replicas=<nombre>`
3. Kubernetes gère automatiquement la création et la suppression des pods
4. Le scaling manuel nécessite une intervention humaine
5. C'est utile pour les pics de charge prévisibles et les tests
6. Pour une réactivité automatique, l'autoscaling (HPA) est préférable

Dans les prochaines sections, nous découvrirons comment automatiser ce processus avec le Horizontal Pod Autoscaler (HPA) et le Vertical Pod Autoscaler (VPA), qui ajusteront automatiquement vos répliques en fonction de la charge réelle de votre application.

---

**Prochaine section :** 19.2 Horizontal Pod Autoscaler (HPA) - Automatisez le scaling en fonction des métriques !

⏭️ [Horizontal Pod Autoscaler (HPA)](/19-scaling-et-autoscaling/02-horizontal-pod-autoscaler-hpa.md)
