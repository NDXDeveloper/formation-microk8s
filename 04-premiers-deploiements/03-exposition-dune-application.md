🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 4.3 Exposition d'une Application

## Introduction

Dans la section précédente, nous avons déployé une application Nginx avec succès. Cependant, cette application n'est accessible qu'à l'intérieur du cluster Kubernetes. Pour qu'elle soit utilisable, nous devons l'**exposer** au monde extérieur (ou au moins à notre réseau local).

Dans cette section, nous allons découvrir le concept de **Service** dans Kubernetes, l'objet qui permet d'exposer vos applications de manière stable et fiable.

## Le Problème : Les Pods sont Éphémères

Avant de comprendre les Services, il faut comprendre le problème qu'ils résolvent.

### Pods et adresses IP volatiles

Chaque Pod dans Kubernetes reçoit sa propre adresse IP. Cependant :

❌ **Les IPs des Pods changent** : Quand un Pod est recréé (crash, mise à jour, scaling), il obtient une nouvelle IP
❌ **Impossible à mémoriser** : Avec plusieurs répliques, il faudrait connaître toutes les IPs
❌ **Pas de load balancing** : Comment répartir le trafic entre plusieurs répliques ?

**Exemple concret :**

```bash
# Voir les IPs des Pods
microk8s kubectl get pods -o wide

# Sortie :
# NAME                                READY   STATUS    RESTARTS   AGE   IP
# nginx-deployment-5d59d67564-7k9xm   1/1     Running   0          5m    10.1.1.5
# nginx-deployment-5d59d67564-h2p4l   1/1     Running   0          5m    10.1.1.6
# nginx-deployment-5d59d67564-tz8wq   1/1     Running   0          5m    10.1.1.7
```

Si vous supprimez un de ces Pods, le nouveau aura une IP différente !

### La Solution : Les Services Kubernetes

Un **Service** est un objet Kubernetes qui :

✅ Fournit une **adresse IP stable** (qui ne change jamais)
✅ Expose un **nom DNS** facile à retenir
✅ Fait du **load balancing** automatique entre les Pods
✅ Surveille la santé des Pods et route uniquement vers les Pods sains

**Analogie :** Un Service est comme un standard téléphonique d'entreprise. Vous appelez toujours le même numéro (IP stable), et le standard vous connecte à un employé disponible (Pod), peu importe qui c'est ou s'il y a eu des changements dans le personnel.

## Types de Services

Kubernetes propose quatre types de Services, chacun adapté à un cas d'usage spécifique :

| Type | Utilisation | Accessibilité |
|------|-------------|---------------|
| **ClusterIP** | Communication interne | Uniquement dans le cluster |
| **NodePort** | Accès simple depuis l'extérieur | Via IP du nœud + port spécifique |
| **LoadBalancer** | Production avec load balancer externe | Via une IP publique dédiée |
| **ExternalName** | Alias vers un service externe | Redirection DNS |

Nous allons explorer les trois premiers types en détail.

## Type 1 : ClusterIP (Par Défaut)

### Qu'est-ce que ClusterIP ?

**ClusterIP** est le type de Service par défaut. Il expose votre application uniquement **à l'intérieur du cluster** Kubernetes.

**Cas d'usage :**
- Communication entre microservices
- Backend non accessible depuis l'extérieur
- Base de données interne
- API interne

### Créer un Service ClusterIP

Créez un fichier `nginx-service-clusterip.yaml` :

```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-service
  labels:
    app: nginx
spec:
  type: ClusterIP
  selector:
    app: nginx
  ports:
  - protocol: TCP
    port: 80
    targetPort: 80
```

**Décortiquons ce manifeste :**

#### selector
```yaml
selector:
  app: nginx
```

Le **selector** est crucial ! Il indique au Service quels Pods cibler. Le Service routera le trafic vers tous les Pods ayant le label `app: nginx`.

**⚠️ Important :** Ce label doit correspondre aux labels de votre Deployment !

#### ports
```yaml
ports:
- protocol: TCP
  port: 80          # Port du Service
  targetPort: 80    # Port du conteneur dans le Pod
```

- **port** : Le port sur lequel le Service écoute (ce que les clients utilisent)
- **targetPort** : Le port sur lequel votre application écoute dans le Pod
- **protocol** : TCP ou UDP

**Schéma conceptuel :**

```
Client → Service (port 80) → Pod (targetPort 80)
```

### Déployer le Service

```bash
# Créer le Service
microk8s kubectl apply -f nginx-service-clusterip.yaml

# Sortie :
# service/nginx-service created
```

### Vérifier le Service

```bash
# Lister les Services
microk8s kubectl get services
# ou la forme courte :
microk8s kubectl get svc

# Sortie :
# NAME            TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)   AGE
# kubernetes      ClusterIP   10.152.183.1    <none>        443/TCP   5d
# nginx-service   ClusterIP   10.152.183.45   <none>        80/TCP    30s
```

**Comprendre la sortie :**
- `CLUSTER-IP` : L'IP interne stable du Service (10.152.183.45)
- `EXTERNAL-IP` : `<none>` car ClusterIP est interne uniquement
- `PORT(S)` : 80/TCP - le port exposé

### Tester le Service

Depuis un Pod dans le cluster, vous pouvez accéder au Service :

```bash
# Créer un Pod temporaire pour tester
microk8s kubectl run test-pod --image=busybox --rm -it --restart=Never -- sh

# Dans le Pod, tester avec wget
wget -qO- nginx-service
# ou avec curl si disponible
wget -qO- http://nginx-service:80
```

Vous devriez voir le HTML de la page d'accueil Nginx !

**Magie du DNS :** Le nom `nginx-service` est automatiquement résolu par le DNS interne de Kubernetes vers l'IP du Service.

### Accéder depuis votre machine (port-forward)

Pour tester depuis votre machine locale :

```bash
# Créer un tunnel
microk8s kubectl port-forward service/nginx-service 8080:80

# Accéder dans le navigateur : http://localhost:8080
```

**Note :** Le port-forward est pratique pour le développement mais pas pour la production.

## Type 2 : NodePort

### Qu'est-ce que NodePort ?

**NodePort** expose votre Service sur un port spécifique de **chaque nœud** du cluster. C'est le moyen le plus simple d'accéder à votre application depuis l'extérieur.

**Comment ça marche :**
1. Kubernetes ouvre un port (entre 30000-32767) sur tous les nœuds
2. Le trafic vers `<IP-du-nœud>:<NodePort>` est routé vers votre Service
3. Le Service fait du load balancing vers les Pods

**Cas d'usage :**
- Labs et environnements de développement
- Accès rapide depuis l'extérieur
- Clusters sans load balancer externe

### Créer un Service NodePort

Créez un fichier `nginx-service-nodeport.yaml` :

```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-nodeport
  labels:
    app: nginx
spec:
  type: NodePort
  selector:
    app: nginx
  ports:
  - protocol: TCP
    port: 80
    targetPort: 80
    nodePort: 30080
```

**Nouveaux éléments :**

```yaml
type: NodePort
nodePort: 30080
```

- `type: NodePort` : Change le type de Service
- `nodePort: 30080` : Port spécifique à utiliser (optionnel, sinon Kubernetes en assigne un automatiquement)

**⚠️ Plage valide :** Les NodePorts doivent être entre 30000 et 32767.

### Déployer le Service NodePort

```bash
# Créer le Service
microk8s kubectl apply -f nginx-service-nodeport.yaml

# Vérifier
microk8s kubectl get svc nginx-nodeport

# Sortie :
# NAME             TYPE       CLUSTER-IP       EXTERNAL-IP   PORT(S)        AGE
# nginx-nodeport   NodePort   10.152.183.100   <none>        80:30080/TCP   10s
```

**Comprendre `PORT(S)` :**
- `80:30080/TCP` signifie :
  - Port du Service : 80
  - NodePort (port externe) : 30080

### Accéder au Service

Vous pouvez maintenant accéder à votre application via l'IP de votre machine !

```bash
# Obtenir l'IP de votre nœud
hostname -I
# ou
ip addr show

# Supposons que votre IP est 192.168.1.100
# Accédez via : http://192.168.1.100:30080
```

**Depuis un autre ordinateur sur le même réseau :**
- Ouvrez un navigateur
- Allez à `http://192.168.1.100:30080`
- Vous verrez la page Nginx !

### Laisser Kubernetes choisir le port

Si vous ne spécifiez pas `nodePort`, Kubernetes en assigne un automatiquement :

```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-nodeport-auto
spec:
  type: NodePort
  selector:
    app: nginx
  ports:
  - protocol: TCP
    port: 80
    targetPort: 80
    # Pas de nodePort spécifié
```

Kubernetes choisira un port disponible dans la plage 30000-32767.

```bash
# Voir le port assigné
microk8s kubectl get svc nginx-nodeport-auto

# Sortie exemple :
# NAME                  TYPE       CLUSTER-IP       PORT(S)        AGE
# nginx-nodeport-auto   NodePort   10.152.183.101   80:31245/TCP   5s
```

Le port assigné ici est 31245.

### Avantages et Inconvénients

**✅ Avantages :**
- Simple à configurer
- Pas besoin d'infrastructure supplémentaire
- Idéal pour les labs et le développement

**❌ Inconvénients :**
- Nécessite de connaître l'IP du nœud
- Ports peu conviviaux (30000-32767)
- Un seul Service par port
- Pas idéal pour la production

## Type 3 : LoadBalancer

### Qu'est-ce qu'un LoadBalancer ?

**LoadBalancer** est le type de Service le plus "professionnel". Il provisionne automatiquement un **load balancer externe** qui distribue le trafic vers vos Pods.

**Comment ça marche :**
1. Vous créez un Service de type LoadBalancer
2. Kubernetes demande à votre plateforme cloud (ou MetalLB pour MicroK8s) de créer un load balancer
3. Une IP externe est assignée
4. Le trafic vers cette IP est distribué entre vos Pods

**Cas d'usage :**
- Applications de production
- Besoin d'une IP publique propre
- Load balancing professionnel

### Prérequis pour MicroK8s : MetalLB

Sur les clouds publics (AWS, GCP, Azure), le LoadBalancer est fourni automatiquement. Pour MicroK8s, nous devons activer **MetalLB**, un addon qui émule un load balancer.

```bash
# Activer MetalLB
microk8s enable metallb

# MicroK8s vous demandera une plage d'IPs
# Exemple pour un réseau local 192.168.1.0/24 :
# Entrez : 192.168.1.200-192.168.1.250
```

**⚠️ Important :** Choisissez des IPs libres sur votre réseau local, pas déjà utilisées par d'autres machines.

**Qu'est-ce que MetalLB ?**
- Fournit des IPs externes pour les Services LoadBalancer
- Annonce ces IPs sur le réseau local via ARP
- Distribue le trafic vers les Pods

Nous approfondirons MetalLB dans le chapitre 9.

### Créer un Service LoadBalancer

Créez un fichier `nginx-service-loadbalancer.yaml` :

```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-loadbalancer
  labels:
    app: nginx
spec:
  type: LoadBalancer
  selector:
    app: nginx
  ports:
  - protocol: TCP
    port: 80
    targetPort: 80
```

**Changement clé :**
```yaml
type: LoadBalancer
```

C'est tout ! Pas besoin de spécifier de NodePort.

### Déployer le Service LoadBalancer

```bash
# Créer le Service
microk8s kubectl apply -f nginx-service-loadbalancer.yaml

# Vérifier (peut prendre quelques secondes)
microk8s kubectl get svc nginx-loadbalancer

# Sortie :
# NAME                 TYPE           CLUSTER-IP       EXTERNAL-IP     PORT(S)        AGE
# nginx-loadbalancer   LoadBalancer   10.152.183.102   192.168.1.200   80:31678/TCP   1m
```

**Comprendre la sortie :**
- `EXTERNAL-IP` : L'IP externe assignée par MetalLB (192.168.1.200)
- Si vous voyez `<pending>`, attendez quelques secondes
- Si cela reste `<pending>`, vérifiez que MetalLB est bien activé

### Accéder au Service

C'est très simple maintenant :

```bash
# Depuis n'importe où sur votre réseau local
curl http://192.168.1.200
# ou ouvrez http://192.168.1.200 dans un navigateur
```

**Avantage :** Pas besoin de spécifier un port ! Le Service est accessible sur le port standard 80.

### Multiples ports

Un Service peut exposer plusieurs ports :

```yaml
apiVersion: v1
kind: Service
metadata:
  name: web-service
spec:
  type: LoadBalancer
  selector:
    app: webapp
  ports:
  - name: http
    protocol: TCP
    port: 80
    targetPort: 8080
  - name: https
    protocol: TCP
    port: 443
    targetPort: 8443
  - name: metrics
    protocol: TCP
    port: 9090
    targetPort: 9090
```

**Note :** Quand vous avez plusieurs ports, il est recommandé de leur donner des noms (`http`, `https`, `metrics`).

## Comparaison des Types de Services

| Critère | ClusterIP | NodePort | LoadBalancer |
|---------|-----------|----------|--------------|
| **Accessibilité** | Interne uniquement | Externe via NodePort | Externe via IP dédiée |
| **IP externe** | ❌ Non | ❌ Non | ✅ Oui |
| **Port** | Port standard | 30000-32767 | Port standard |
| **Load balancing** | ✅ Oui | ✅ Oui | ✅ Oui |
| **Use case** | Microservices internes | Dev/Test | Production |
| **Complexité** | Simple | Simple | Nécessite MetalLB/Cloud |
| **Coût** | Gratuit | Gratuit | Gratuit avec MetalLB |

**Recommandations :**
- **Dev/Test local** : NodePort ou port-forward
- **Production** : LoadBalancer (avec MetalLB ou cloud provider)
- **Communication interne** : ClusterIP

## DNS Interne et Discovery

### Résolution DNS automatique

Kubernetes fournit un **DNS interne** (CoreDNS) qui permet aux Pods de se trouver facilement.

**Format du nom DNS :**
```
<service-name>.<namespace>.svc.cluster.local
```

**Exemples :**

```bash
# Depuis un Pod dans le même namespace
curl http://nginx-service

# Depuis un Pod dans un autre namespace
curl http://nginx-service.default.svc.cluster.local

# Format court
curl http://nginx-service.default
```

### Variables d'environnement

Kubernetes injecte automatiquement des variables d'environnement pour chaque Service :

```bash
# Lister les variables dans un Pod
microk8s kubectl exec <pod-name> -- env | grep NGINX_SERVICE

# Sortie exemple :
# NGINX_SERVICE_SERVICE_HOST=10.152.183.45
# NGINX_SERVICE_SERVICE_PORT=80
```

**Format :**
- `<SERVICE_NAME>_SERVICE_HOST` : IP du Service
- `<SERVICE_NAME>_SERVICE_PORT` : Port du Service

**Note :** Le nom est en majuscules avec underscores au lieu de tirets.

## Load Balancing et Session Affinity

### Load Balancing par défaut

Par défaut, Kubernetes distribue le trafic de manière **round-robin** entre tous les Pods sains.

**Exemple avec 3 répliques :**
1. Requête 1 → Pod A
2. Requête 2 → Pod B
3. Requête 3 → Pod C
4. Requête 4 → Pod A (on recommence)

### Session Affinity (Sticky Sessions)

Parfois, vous voulez que les requêtes d'un même client soient toujours routées vers le même Pod :

```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-sticky
spec:
  type: LoadBalancer
  selector:
    app: nginx
  sessionAffinity: ClientIP
  sessionAffinityConfig:
    clientIP:
      timeoutSeconds: 3600
  ports:
  - protocol: TCP
    port: 80
    targetPort: 80
```

**Nouveaux champs :**
- `sessionAffinity: ClientIP` : Route les requêtes de la même IP client vers le même Pod
- `timeoutSeconds: 3600` : Durée de validité (1 heure ici)

**Use cases :**
- Applications avec sessions utilisateur
- Caching local dans les Pods
- WebSockets

## Health Checks et Endpoints

### Comment le Service sait quels Pods sont prêts ?

Le Service ne route le trafic que vers les Pods **"Ready"**. Kubernetes utilise les **readiness probes** pour déterminer si un Pod est prêt.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.21-alpine
        ports:
        - containerPort: 80
        readinessProbe:
          httpGet:
            path: /
            port: 80
          initialDelaySeconds: 5
          periodSeconds: 5
```

**Avec cette configuration :**
- Kubernetes teste `GET http://pod-ip/` toutes les 5 secondes
- Si le code HTTP est 200-399, le Pod est "Ready"
- Sinon, le Pod est retiré temporairement du Service

### Voir les Endpoints

Les **Endpoints** représentent les IPs des Pods sélectionnés par le Service :

```bash
# Voir les Endpoints
microk8s kubectl get endpoints nginx-service

# Sortie :
# NAME            ENDPOINTS                                      AGE
# nginx-service   10.1.1.5:80,10.1.1.6:80,10.1.1.7:80           5m
```

Chaque IP correspond à un Pod "Ready".

**Vérifier en détail :**
```bash
microk8s kubectl describe endpoints nginx-service
```

## Exposer via kubectl expose

Au lieu de créer un fichier YAML, vous pouvez utiliser `kubectl expose` :

```bash
# Exposer un Deployment
microk8s kubectl expose deployment nginx-deployment \
  --type=LoadBalancer \
  --port=80 \
  --target-port=80 \
  --name=nginx-exposed

# Exposer un Pod
microk8s kubectl expose pod nginx-pod \
  --type=NodePort \
  --port=80 \
  --name=nginx-pod-service
```

**Avantages :**
- Rapide pour tester
- Moins de fichiers à gérer

**Inconvénients :**
- Pas de versioning (pas dans Git)
- Moins de contrôle
- Difficile à reproduire

**💡 Recommandation :** Utilisez des fichiers YAML pour la production, `kubectl expose` pour les tests rapides.

## Commandes Essentielles

### Gestion des Services

```bash
# Lister les Services
microk8s kubectl get services
microk8s kubectl get svc

# Détails d'un Service
microk8s kubectl describe service nginx-service

# Voir les Endpoints
microk8s kubectl get endpoints

# Supprimer un Service
microk8s kubectl delete service nginx-service

# Créer depuis un fichier
microk8s kubectl apply -f service.yaml
```

### Debugging

```bash
# Tester avec port-forward
microk8s kubectl port-forward service/nginx-service 8080:80

# Voir les événements
microk8s kubectl get events --sort-by='.lastTimestamp'

# Vérifier les labels des Pods
microk8s kubectl get pods --show-labels

# Tester depuis un Pod temporaire
microk8s kubectl run test --image=busybox --rm -it -- wget -qO- http://nginx-service
```

## Manifeste Complet : Deployment + Service

Voici un exemple complet combinant Deployment et Service dans un seul fichier :

```yaml
---
# Deployment
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-app
  labels:
    app: nginx
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.21-alpine
        ports:
        - containerPort: 80
        resources:
          requests:
            memory: "64Mi"
            cpu: "100m"
          limits:
            memory: "128Mi"
            cpu: "200m"
        readinessProbe:
          httpGet:
            path: /
            port: 80
          initialDelaySeconds: 5
          periodSeconds: 5
        livenessProbe:
          httpGet:
            path: /
            port: 80
          initialDelaySeconds: 30
          periodSeconds: 10

---
# Service
apiVersion: v1
kind: Service
metadata:
  name: nginx-app-service
  labels:
    app: nginx
spec:
  type: LoadBalancer
  selector:
    app: nginx
  ports:
  - protocol: TCP
    port: 80
    targetPort: 80
    name: http
```

**Déployer le tout :**
```bash
microk8s kubectl apply -f nginx-complete.yaml

# Vérifier
microk8s kubectl get deployments,services,pods
```

## Bonnes Pratiques

### 1. Toujours nommer vos ports

```yaml
ports:
- name: http
  protocol: TCP
  port: 80
  targetPort: 80
- name: https
  protocol: TCP
  port: 443
  targetPort: 443
```

Les noms facilitent la compréhension et la documentation.

### 2. Utiliser des labels cohérents

```yaml
# Dans le Deployment
metadata:
  labels:
    app: nginx
    version: "1.21"
    environment: production

# Dans le Service
selector:
  app: nginx
  environment: production
```

### 3. Définir des readiness probes

```yaml
readinessProbe:
  httpGet:
    path: /health
    port: 80
  initialDelaySeconds: 5
  periodSeconds: 5
```

Assurez-vous que le Service ne route que vers des Pods sains.

### 4. Choisir le bon type de Service

- **Développement local** : ClusterIP + port-forward ou NodePort
- **Production** : LoadBalancer avec MetalLB
- **Communication interne** : ClusterIP

### 5. Documenter avec des annotations

```yaml
metadata:
  name: nginx-service
  annotations:
    description: "Service principal pour l'application web Nginx"
    contact: "devops-team@example.com"
    prometheus.io/scrape: "true"
    prometheus.io/port: "9090"
```

### 6. Limiter l'exposition

N'exposez que ce qui doit l'être :
- Bases de données → ClusterIP uniquement
- APIs internes → ClusterIP
- Frontends web → LoadBalancer

## Résolution de Problèmes

### Le Service n'a pas d'Endpoints

**Symptôme :**
```bash
microk8s kubectl get endpoints nginx-service
# NAME            ENDPOINTS   AGE
# nginx-service   <none>      2m
```

**Causes possibles :**
1. Les labels du selector ne correspondent pas
2. Aucun Pod n'est "Ready"
3. Les Pods n'existent pas

**Solution :**
```bash
# Vérifier les labels des Pods
microk8s kubectl get pods --show-labels

# Vérifier le selector du Service
microk8s kubectl describe service nginx-service

# Vérifier si les Pods sont Ready
microk8s kubectl get pods
```

### LoadBalancer reste en "Pending"

**Symptôme :**
```bash
# EXTERNAL-IP reste <pending>
microk8s kubectl get svc
```

**Causes :**
- MetalLB n'est pas activé
- Pas d'IPs disponibles dans le pool MetalLB

**Solution :**
```bash
# Activer MetalLB
microk8s enable metallb

# Vérifier les logs
microk8s kubectl logs -n metallb-system -l app=metallb
```

### Impossible d'accéder au Service NodePort

**Vérifications :**
1. Le firewall bloque-t-il le port ?
```bash
sudo ufw status
sudo ufw allow 30080/tcp
```

2. Utilisez-vous la bonne IP ?
```bash
hostname -I
```

3. Le Service est-il bien de type NodePort ?
```bash
microk8s kubectl get svc
```

### Connexion refusée

**Vérifications :**
1. Le Pod est-il Ready ?
```bash
microk8s kubectl get pods
```

2. Le conteneur écoute-t-il sur le bon port ?
```bash
microk8s kubectl exec <pod-name> -- netstat -tlnp
```

3. Le targetPort est-il correct ?
```bash
microk8s kubectl describe service nginx-service
```

## Récapitulatif

Dans cette section, vous avez appris à :

✅ Comprendre le rôle des Services dans Kubernetes
✅ Créer des Services de type **ClusterIP**, **NodePort** et **LoadBalancer**
✅ Choisir le bon type de Service selon le cas d'usage
✅ Utiliser le DNS interne pour la découverte de services
✅ Configurer le load balancing et la session affinity
✅ Déboguer les problèmes d'exposition courants

**Points clés :**

- Les **Services** fournissent une IP stable et du load balancing
- **ClusterIP** : Communication interne uniquement
- **NodePort** : Accès simple depuis l'extérieur (ports 30000-32767)
- **LoadBalancer** : Solution professionnelle avec IP externe dédiée
- Les **selectors** et **labels** doivent correspondre entre Service et Pods
- **MetalLB** émule un load balancer pour MicroK8s

Votre application est maintenant déployée et accessible ! Les prochaines sections couvriront la configuration avec des variables d'environnement et des données sensibles.

---

**Prochaine section :** 4.4 Variables d'environnement

⏭️ [Variables d'environnement](/04-premiers-deploiements/04-variables-denvironnement.md)
