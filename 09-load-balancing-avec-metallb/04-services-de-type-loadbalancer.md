🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 9.4 Services de Type LoadBalancer

## Introduction

Maintenant que MetalLB est installé et configuré avec un pool d'adresses IP, nous pouvons créer des **Services de type LoadBalancer**. Ces Services permettent d'exposer vos applications avec une IP externe unique et stable, accessible depuis l'extérieur du cluster.

Dans cette section, nous allons voir comment créer, configurer et gérer ces Services pour rendre vos applications accessibles de manière professionnelle.

## Rappel : Les Types de Services Kubernetes

Kubernetes propose plusieurs types de Services pour exposer vos applications. Récapitulons rapidement :

### 1. ClusterIP (Par défaut)

**Portée** : Interne au cluster uniquement

```
Application → Service ClusterIP → Pods
(accessible uniquement depuis l'intérieur du cluster)
```

**Utilisation** : Communication entre microservices, APIs internes

### 2. NodePort

**Portée** : Accessible depuis l'extérieur via un port sur chaque nœud

```
Utilisateur → NodeIP:NodePort → Service → Pods
```

**Limitation** :
- Nécessite de connaître l'IP des nœuds
- Utilise des ports hauts (30000-32767)
- Pas pratique pour la production

### 3. LoadBalancer

**Portée** : Accessible depuis l'extérieur via une IP externe unique

```
Utilisateur → IP externe (MetalLB) → Service → Pods
```

**Avantages** :
- IP dédiée et stable pour chaque service
- Port standard (80, 443, etc.)
- Expérience similaire aux environnements cloud
- C'est ce que nous allons utiliser !

## Qu'est-ce qu'un Service LoadBalancer ?

Un Service de type LoadBalancer est un Service Kubernetes qui :

1. **Demande une IP externe** à MetalLB (depuis le pool configuré)
2. **Reçoit une IP dédiée** qui est annoncée sur votre réseau
3. **Distribue le trafic** entrant vers tous les Pods correspondants
4. **Effectue un health check** pour n'envoyer le trafic qu'aux Pods sains

C'est l'équivalent Kubernetes des load balancers cloud (AWS ELB, GCP Load Balancer, Azure Load Balancer), mais géré par MetalLB dans votre environnement local.

## Anatomie d'un Service LoadBalancer

Voici la structure de base d'un Service LoadBalancer :

```yaml
apiVersion: v1
kind: Service
metadata:
  name: mon-application
  namespace: default
spec:
  type: LoadBalancer
  selector:
    app: mon-application
  ports:
  - name: http
    port: 80
    targetPort: 8080
    protocol: TCP
```

Décortiquons chaque élément :

### Section `metadata`

```yaml
metadata:
  name: mon-application        # Nom du Service
  namespace: default           # Namespace (default si omis)
```

- **name** : Identifiant unique du Service dans le namespace
- **namespace** : Compartiment logique (optionnel, "default" par défaut)

### Section `spec.type`

```yaml
spec:
  type: LoadBalancer
```

Le type `LoadBalancer` indique à Kubernetes (et à MetalLB) que ce Service a besoin d'une IP externe.

### Section `spec.selector`

```yaml
spec:
  selector:
    app: mon-application
```

Le **selector** définit quels Pods ce Service cible. Il utilise les labels des Pods :
- Tous les Pods avec le label `app: mon-application` recevront du trafic
- Si vous ajoutez ou supprimez des Pods avec ce label, le Service s'ajuste automatiquement

### Section `spec.ports`

```yaml
spec:
  ports:
  - name: http              # Nom du port (optionnel mais recommandé)
    port: 80                # Port exposé par le Service
    targetPort: 8080        # Port sur lequel le Pod écoute
    protocol: TCP           # TCP ou UDP
```

**Explication des ports** :

- **port** : Le port sur lequel le Service écoute (celui que les clients utilisent)
- **targetPort** : Le port sur lequel votre application dans le Pod écoute réellement
- **protocol** : TCP (par défaut) ou UDP

**Exemple concret** :

Votre application web écoute sur le port 8080 à l'intérieur du container, mais vous voulez que les utilisateurs y accèdent via le port 80 standard :

```
Client → 192.168.1.200:80 → Service (port: 80) → Pod (targetPort: 8080)
```

## Création de Votre Premier Service LoadBalancer

### Étape 1 : Déployer une Application

Avant de créer un Service, il faut une application à exposer. Créons un Deployment simple :

**Fichier : `nginx-deployment.yaml`**

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-demo
  labels:
    app: nginx-demo
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx-demo
  template:
    metadata:
      labels:
        app: nginx-demo
    spec:
      containers:
      - name: nginx
        image: nginx:latest
        ports:
        - containerPort: 80
```

Déployez-le :

```bash
microk8s kubectl apply -f nginx-deployment.yaml
```

Vérifiez que les Pods sont en cours d'exécution :

```bash
microk8s kubectl get pods -l app=nginx-demo
```

Vous devriez voir 3 Pods :

```
NAME                          READY   STATUS    RESTARTS   AGE
nginx-demo-xxxxx-aaaaa        1/1     Running   0          30s
nginx-demo-xxxxx-bbbbb        1/1     Running   0          30s
nginx-demo-xxxxx-ccccc        1/1     Running   0          30s
```

### Étape 2 : Créer le Service LoadBalancer

**Fichier : `nginx-service.yaml`**

```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-demo-lb
spec:
  type: LoadBalancer
  selector:
    app: nginx-demo
  ports:
  - name: http
    port: 80
    targetPort: 80
    protocol: TCP
```

Appliquez-le :

```bash
microk8s kubectl apply -f nginx-service.yaml
```

### Étape 3 : Vérifier l'Assignation de l'IP

Observez le Service :

```bash
microk8s kubectl get svc nginx-demo-lb
```

Initialement, vous pourriez voir :

```
NAME            TYPE           EXTERNAL-IP   PORT(S)        AGE
nginx-demo-lb   LoadBalancer   <pending>     80:30123/TCP   5s
```

L'état `<pending>` signifie que MetalLB est en train d'assigner une IP. Après quelques secondes :

```
NAME            TYPE           EXTERNAL-IP     PORT(S)        AGE
nginx-demo-lb   LoadBalancer   192.168.1.200   80:30123/TCP   20s
```

**🎉 Succès !** MetalLB a assigné l'IP `192.168.1.200` à votre Service.

### Étape 4 : Tester l'Accès

Depuis n'importe quelle machine sur votre réseau local :

```bash
curl http://192.168.1.200
```

Ou ouvrez dans votre navigateur : `http://192.168.1.200`

Vous devriez voir la page d'accueil de nginx !

## Comprendre le Flux de Trafic

Quand vous accédez à `http://192.168.1.200` :

```
1. Votre navigateur → 192.168.1.200:80
2. MetalLB (speaker) reçoit la requête
3. Service nginx-demo-lb distribue vers un des Pods
4. Pod nginx-demo-xxxxx-aaaaa répond
5. La réponse revient au navigateur
```

**Load balancing en action** :

Si vous faites plusieurs requêtes, le Service les distribue entre les 3 Pods nginx. Vous pouvez le vérifier en consultant les logs de chaque Pod :

```bash
# Voir les logs du premier Pod
microk8s kubectl logs -f deployment/nginx-demo --tail=10
```

## Ports Multiples

Un Service peut exposer plusieurs ports simultanément. C'est utile pour les applications qui ont plusieurs interfaces (HTTP, HTTPS, admin, etc.).

**Exemple : Service avec HTTP et HTTPS**

```yaml
apiVersion: v1
kind: Service
metadata:
  name: web-app-lb
spec:
  type: LoadBalancer
  selector:
    app: web-app
  ports:
  - name: http
    port: 80
    targetPort: 8080
    protocol: TCP
  - name: https
    port: 443
    targetPort: 8443
    protocol: TCP
```

Avec ce Service :
- `192.168.1.X:80` → Pod sur port 8080
- `192.168.1.X:443` → Pod sur port 8443

## Annotations MetalLB

MetalLB supporte plusieurs annotations pour contrôler le comportement du load balancing.

### Choisir un Pool Spécifique

Par défaut, MetalLB utilise le premier pool disponible. Pour spécifier un pool :

```yaml
apiVersion: v1
kind: Service
metadata:
  name: mon-service
  annotations:
    metallb.universe.tf/address-pool: production-pool
spec:
  type: LoadBalancer
  # ... reste de la configuration
```

### Demander une IP Spécifique

Si vous voulez une IP particulière du pool :

```yaml
apiVersion: v1
kind: Service
metadata:
  name: mon-service-vip
spec:
  type: LoadBalancer
  loadBalancerIP: 192.168.1.205
  selector:
    app: mon-app
  ports:
  - port: 80
```

**⚠️ Important** :
- L'IP doit être dans un pool MetalLB
- L'IP ne doit pas être déjà utilisée
- Si l'IP n'est pas disponible, le Service reste en `<pending>`

### Contrôler l'Allocation d'IP

Pour empêcher MetalLB d'assigner automatiquement une IP (allocation manuelle uniquement) :

```yaml
metadata:
  annotations:
    metallb.universe.tf/allow-shared-ip: "false"
```

### Partager une IP entre Services (Avancé)

Dans certains cas, vous pouvez vouloir que plusieurs Services partagent la même IP (avec des ports différents) :

```yaml
metadata:
  annotations:
    metallb.universe.tf/allow-shared-ip: "shared-key-1"
```

Tous les Services avec la même clé de partage utiliseront la même IP.

**Exemple** :

```yaml
# Service 1 : HTTP
apiVersion: v1
kind: Service
metadata:
  name: web-http
  annotations:
    metallb.universe.tf/allow-shared-ip: "web-services"
spec:
  type: LoadBalancer
  selector:
    app: web
  ports:
  - port: 80
    targetPort: 8080
---
# Service 2 : HTTPS
apiVersion: v1
kind: Service
metadata:
  name: web-https
  annotations:
    metallb.universe.tf/allow-shared-ip: "web-services"
spec:
  type: LoadBalancer
  selector:
    app: web
  ports:
  - port: 443
    targetPort: 8443
```

Les deux Services partagent la même IP externe mais utilisent des ports différents.

## Gestion du Cycle de Vie

### Créer un Service

```bash
# Depuis un fichier YAML
microk8s kubectl apply -f mon-service.yaml

# Ou directement en ligne de commande (pour tests rapides)
microk8s kubectl expose deployment mon-app --type=LoadBalancer --port=80
```

### Lister les Services LoadBalancer

```bash
# Tous les Services LoadBalancer
microk8s kubectl get svc --all-namespaces | grep LoadBalancer

# Dans le namespace actuel
microk8s kubectl get svc -o wide
```

### Voir les Détails d'un Service

```bash
microk8s kubectl describe service nginx-demo-lb
```

Vous verrez :
- L'IP externe assignée
- Les endpoints (Pods) ciblés
- Les événements (assignation d'IP, etc.)
- La configuration des ports

### Modifier un Service

```bash
microk8s kubectl edit service nginx-demo-lb
```

Ou modifiez le fichier YAML et réappliquez :

```bash
microk8s kubectl apply -f nginx-service.yaml
```

### Supprimer un Service

```bash
microk8s kubectl delete service nginx-demo-lb
```

**Important** : Quand vous supprimez un Service LoadBalancer :
- L'IP externe est libérée et retourne dans le pool MetalLB
- Les Pods ne sont PAS supprimés (sauf si vous supprimez aussi le Deployment)
- L'IP peut être réutilisée par un autre Service

## Différences avec NodePort

Quand vous créez un Service LoadBalancer, Kubernetes crée automatiquement aussi un NodePort. Vous l'avez peut-être remarqué :

```
NAME            TYPE           EXTERNAL-IP     PORT(S)        AGE
nginx-demo-lb   LoadBalancer   192.168.1.200   80:30123/TCP   1m
```

Le `80:30123` signifie :
- **80** : Port du LoadBalancer (IP externe)
- **30123** : NodePort automatiquement créé

Vous pouvez donc accéder au Service de deux façons :
1. Via l'IP LoadBalancer : `http://192.168.1.200:80` ✅ Recommandé
2. Via le NodePort : `http://<node-ip>:30123` (backup)

## SessionAffinity (Affinité de Session)

Par défaut, Kubernetes distribue les requêtes de manière round-robin entre les Pods. Si vous avez besoin que les requêtes d'un même client aillent toujours vers le même Pod (sticky sessions) :

```yaml
apiVersion: v1
kind: Service
metadata:
  name: sticky-service
spec:
  type: LoadBalancer
  sessionAffinity: ClientIP
  sessionAffinityConfig:
    clientIP:
      timeoutSeconds: 10800  # 3 heures
  selector:
    app: mon-app
  ports:
  - port: 80
```

**Comportement** :
- `sessionAffinity: ClientIP` : Les requêtes d'une même IP client vont au même Pod
- `timeoutSeconds: 10800` : L'affinité dure 3 heures (par défaut : 3 heures)

**Cas d'usage** : Applications web avec sessions côté serveur, WebSockets

## ExternalTrafficPolicy

Cette option contrôle comment le trafic est routé vers les Pods.

### Cluster (Par défaut)

```yaml
spec:
  externalTrafficPolicy: Cluster
```

**Comportement** :
- Le trafic peut être routé vers n'importe quel nœud
- Puis vers n'importe quel Pod
- L'IP source du client est masquée (SNAT)

**Avantage** : Distribution équitable
**Inconvénient** : Perte de l'IP client réelle

### Local

```yaml
spec:
  externalTrafficPolicy: Local
```

**Comportement** :
- Le trafic est routé uniquement vers les Pods sur le nœud qui reçoit le trafic
- L'IP source du client est préservée

**Avantage** : Préservation de l'IP client (important pour les logs, la géolocalisation)
**Inconvénient** : Distribution potentiellement inégale

**Exemple complet** :

```yaml
apiVersion: v1
kind: Service
metadata:
  name: service-ip-preservee
spec:
  type: LoadBalancer
  externalTrafficPolicy: Local
  selector:
    app: mon-app
  ports:
  - port: 80
    targetPort: 8080
```

## Health Checks et Readiness

Les Services LoadBalancer ne routent le trafic que vers les Pods "prêts" (ready). C'est géré par les **Readiness Probes** dans vos Pods.

**Exemple de Deployment avec Readiness Probe** :

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app-with-health
spec:
  replicas: 3
  selector:
    matchLabels:
      app: app-with-health
  template:
    metadata:
      labels:
        app: app-with-health
    spec:
      containers:
      - name: app
        image: mon-app:latest
        ports:
        - containerPort: 8080
        readinessProbe:
          httpGet:
            path: /health
            port: 8080
          initialDelaySeconds: 5
          periodSeconds: 10
```

**Fonctionnement** :
- Kubernetes interroge `/health` sur le port 8080 toutes les 10 secondes
- Si la réponse est un code 2xx ou 3xx, le Pod est "ready"
- Seuls les Pods "ready" reçoivent du trafic via le Service
- Si un Pod devient "not ready", le trafic est automatiquement redirigé vers les autres

## Exemples Pratiques

### Exemple 1 : Application Web Simple

```yaml
# Deployment
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-website
spec:
  replicas: 2
  selector:
    matchLabels:
      app: my-website
  template:
    metadata:
      labels:
        app: my-website
    spec:
      containers:
      - name: nginx
        image: nginx:alpine
        ports:
        - containerPort: 80
---
# Service LoadBalancer
apiVersion: v1
kind: Service
metadata:
  name: my-website-lb
spec:
  type: LoadBalancer
  selector:
    app: my-website
  ports:
  - name: http
    port: 80
    targetPort: 80
```

**Accès** : `http://<IP-assignée-par-MetalLB>`

### Exemple 2 : API Backend

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api-backend
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
        - containerPort: 3000
        env:
        - name: PORT
          value: "3000"
        readinessProbe:
          httpGet:
            path: /api/health
            port: 3000
          initialDelaySeconds: 10
          periodSeconds: 5
---
apiVersion: v1
kind: Service
metadata:
  name: api-backend-lb
spec:
  type: LoadBalancer
  selector:
    app: api-backend
  ports:
  - name: api
    port: 443
    targetPort: 3000
  externalTrafficPolicy: Local  # Préserver l'IP client
```

**Accès** : `https://<IP-assignée>:443`

### Exemple 3 : Base de Données (avec IP spécifique)

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: postgres
spec:
  serviceName: postgres
  replicas: 1
  selector:
    matchLabels:
      app: postgres
  template:
    metadata:
      labels:
        app: postgres
    spec:
      containers:
      - name: postgres
        image: postgres:15
        ports:
        - containerPort: 5432
        env:
        - name: POSTGRES_PASSWORD
          value: "motdepasse"
---
apiVersion: v1
kind: Service
metadata:
  name: postgres-lb
spec:
  type: LoadBalancer
  loadBalancerIP: 192.168.1.210  # IP fixe
  selector:
    app: postgres
  ports:
  - name: postgres
    port: 5432
    targetPort: 5432
```

**Accès** : `psql -h 192.168.1.210 -U postgres`

## Vérification et Tests

### Vérifier l'IP Assignée

```bash
microk8s kubectl get svc mon-service -o jsonpath='{.status.loadBalancer.ingress[0].ip}'
```

### Tester la Connectivité

```bash
# Test HTTP simple
curl http://<IP-externe>

# Test avec headers
curl -I http://<IP-externe>

# Test HTTPS (si applicable)
curl -k https://<IP-externe>

# Test de port spécifique
nc -zv <IP-externe> 80
```

### Voir les Endpoints

Les endpoints sont les Pods cibles du Service :

```bash
microk8s kubectl get endpoints mon-service
```

Vous verrez les IP internes des Pods et leurs ports :

```
NAME          ENDPOINTS                           AGE
mon-service   10.1.1.5:8080,10.1.1.6:8080,...     5m
```

### Tester le Load Balancing

Faites plusieurs requêtes et vérifiez que différents Pods répondent :

```bash
# Boucle de 10 requêtes
for i in {1..10}; do
  curl -s http://<IP-externe> | grep -i hostname
done
```

Si votre application affiche le hostname du Pod, vous verrez différents noms apparaître.

## Surveillance et Monitoring

### Événements Kubernetes

Consultez les événements liés au Service :

```bash
microk8s kubectl get events --field-selector involvedObject.name=mon-service
```

Vous verrez les événements comme :
- `EnsuredLoadBalancer` : IP assignée
- `UpdatedLoadBalancer` : Configuration mise à jour

### Logs MetalLB

En cas de problème, consultez les logs MetalLB :

```bash
# Logs du controller
microk8s kubectl logs -n metallb-system -l component=controller --tail=50

# Logs des speakers
microk8s kubectl logs -n metallb-system -l component=speaker --tail=50
```

### Métriques

Si vous avez Prometheus installé (chapitre 12), MetalLB expose des métriques :
- Nombre d'IP assignées
- Pool d'IP disponible
- Statut des annonces L2/BGP

## Résolution de Problèmes Courants

### Service Reste en `<pending>`

**Symptôme** : L'EXTERNAL-IP reste en `<pending>` indéfiniment.

**Causes possibles** :
1. **MetalLB n'est pas installé**
   ```bash
   microk8s status | grep metallb
   ```

2. **Plus d'IP disponibles dans le pool**
   ```bash
   microk8s kubectl get ipaddresspool -n metallb-system -o yaml
   ```

3. **Erreur dans la configuration MetalLB**
   ```bash
   microk8s kubectl logs -n metallb-system -l component=controller
   ```

### IP Assignée mais Non Accessible

**Vérifications** :

1. **Ping l'IP** :
   ```bash
   ping <IP-externe>
   ```

2. **Vérifier le firewall local** :
   ```bash
   sudo ufw status
   # Autoriser le trafic si nécessaire
   sudo ufw allow from 192.168.1.0/24
   ```

3. **Vérifier que les Pods sont running** :
   ```bash
   microk8s kubectl get pods -l app=<votre-app>
   ```

4. **Vérifier les endpoints** :
   ```bash
   microk8s kubectl get endpoints mon-service
   ```

### IP Assignée Incorrecte

Si MetalLB assigne une IP que vous ne voulez pas :

1. Supprimez le Service
2. Modifiez le pool pour exclure cette IP
3. Recréez le Service avec l'IP souhaitée via `loadBalancerIP`

### Conflit avec une IP Existante

Si l'IP assignée par MetalLB est déjà utilisée sur le réseau :

1. Identifiez l'équipement en conflit
2. Retirez cette IP du pool MetalLB
3. Supprimez et recréez le Service

## Bonnes Pratiques

### 1. Utiliser des Labels Clairs

```yaml
metadata:
  labels:
    app: mon-app
    environment: production
    tier: frontend
```

Cela facilite la gestion et le filtrage.

### 2. Documenter les IP Assignées

Maintenez un document listant :
- Service → IP externe
- But du service
- Ports exposés

### 3. Utiliser des Health Checks

Toujours définir des readiness probes pour que le Service ne route que vers des Pods sains.

### 4. Nommer les Ports

```yaml
ports:
- name: http    # Nom explicite
  port: 80
```

C'est plus lisible et utile pour le débogage.

### 5. Versionner les Configurations

Stockez vos manifestes YAML dans Git :

```bash
git add services/
git commit -m "Add LoadBalancer service for nginx-demo"
```

### 6. Tester Avant la Production

Créez d'abord le Service dans un namespace de test :

```yaml
metadata:
  namespace: test
```

### 7. Limiter l'Exposition

N'exposez en LoadBalancer que ce qui doit vraiment être accessible de l'extérieur. Utilisez ClusterIP pour le reste.

## Comparaison : LoadBalancer vs Ingress

Vous vous demandez peut-être quand utiliser un LoadBalancer vs un Ingress (que nous verrons au chapitre 10). Voici un guide :

| Critère | LoadBalancer | Ingress |
|---------|--------------|---------|
| **IP par service** | Une IP par Service | Une IP pour plusieurs Services |
| **Coût en IP** | Élevé (1 IP par service) | Faible (1 IP partagée) |
| **Niveau OSI** | L4 (TCP/UDP) | L7 (HTTP/HTTPS) |
| **Routage par URL** | Non | Oui (par domaine/chemin) |
| **SSL/TLS** | Au niveau de l'app | Centralisé |
| **Cas d'usage** | Protocoles non-HTTP, apps simples | Applications web, APIs, microservices |

**En pratique** : On utilise souvent les deux ensemble :
- LoadBalancer fournit l'IP à l'Ingress Controller
- Ingress Controller fait le routage intelligent vers les Services

## Résumé

Les Services de type LoadBalancer sont essentiels pour exposer vos applications avec des IP externes stables dans MicroK8s. Vous avez appris à :

✅ **Créer** des Services LoadBalancer
✅ **Configurer** les ports et les sélecteurs
✅ **Utiliser** les annotations MetalLB
✅ **Gérer** l'affinité de session et la politique de trafic
✅ **Tester** et vérifier l'accès
✅ **Déboguer** les problèmes courants
✅ **Appliquer** les bonnes pratiques

Avec MetalLB et les Services LoadBalancer, votre cluster MicroK8s se comporte comme une infrastructure cloud professionnelle, vous permettant d'exposer vos applications de manière simple, stable et fiable.

---

**Prochaine étape** : Intégration avec les Services (Section 9.5) ou Ingress et Routage (Chapitre 10)

⏭️ [Intégration avec les Services](/09-load-balancing-avec-metallb/05-integration-avec-les-services.md)
