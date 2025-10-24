🔝 Retour au [Sommaire](/SOMMAIRE.md)

# Glossaire des Termes Kubernetes

## Introduction

Ce glossaire explique les termes essentiels de Kubernetes de manière accessible aux débutants. Chaque terme est accompagné d'une définition claire et, quand c'est pertinent, d'une analogie du monde réel.

**Comment utiliser ce glossaire :**
- Les termes sont classés par ordre alphabétique
- Les mots en **gras** sont définis ailleurs dans le glossaire
- Les 🎯 indiquent les termes les plus importants à connaître

---

## A

### Admission Controller
Composant qui intercepte les requêtes à l'**API Server** avant la persistance des objets. Il peut valider, modifier ou rejeter les requêtes.

**Analogie :** C'est comme un garde de sécurité qui vérifie que tout ce qui entre dans un bâtiment respecte les règles.

**Exemple :** Un Admission Controller peut s'assurer que tous les **Pods** ont des limites de ressources définies.

---

### Affinity (Affinité) 🎯
Règles qui influencent sur quel **nœud** un **Pod** sera programmé. Il existe deux types :

1. **Node Affinity** : Préférence pour certains nœuds
2. **Pod Affinity** : Préférence pour être près d'autres pods
3. **Pod Anti-Affinity** : Préférence pour être loin d'autres pods

**Analogie :** C'est comme choisir de s'asseoir près de ses amis (affinity) ou loin de personnes bruyantes (anti-affinity) dans un restaurant.

---

### Annotation 🎯
Paire clé-valeur attachée à un objet Kubernetes pour stocker des métadonnées non-identifiantes. Contrairement aux **labels**, les annotations ne sont pas utilisées pour sélectionner des objets.

**Utilisation :** Documentation, configuration d'outils externes, informations de build.

**Exemple :**
```yaml
metadata:
  annotations:
    description: "Application de production"
    maintainer: "team@example.com"
    last-updated: "2024-10-24"
```

---

### API Server 🎯
Composant central de Kubernetes qui expose l'API REST. C'est le point d'entrée pour toutes les commandes administratives et opérations sur le **cluster**.

**Analogie :** C'est comme le standard téléphonique d'une entreprise - toutes les communications passent par là.

**Rôle :** Authentification, autorisation, validation, persistance dans **etcd**.

---

### API Version
Version de l'API Kubernetes utilisée pour définir une ressource.

**Exemples courants :**
- `v1` : API core (Pod, Service, ConfigMap)
- `apps/v1` : Applications (Deployment, StatefulSet)
- `batch/v1` : Jobs et CronJobs
- `networking.k8s.io/v1` : Ingress

---

### Auto-scaling
Ajustement automatique du nombre de **Pods** ou de la taille des ressources en fonction de la charge.

**Types :**
- **HPA** (Horizontal Pod Autoscaler) : Ajuste le nombre de pods
- **VPA** (Vertical Pod Autoscaler) : Ajuste les ressources par pod
- **Cluster Autoscaler** : Ajuste le nombre de nœuds

---

## B

### Backend
Dans le contexte d'un **Ingress** ou d'un **Service**, le backend désigne la destination du trafic - généralement un ensemble de **Pods**.

**Exemple :** Un Ingress route `example.com/api` vers le backend `api-service`.

---

### Blue-Green Deployment
Stratégie de déploiement où deux environnements identiques (bleu et vert) existent. L'un sert le trafic en production pendant que l'autre est mis à jour, puis on bascule le trafic.

**Avantage :** Rollback instantané en cas de problème.

---

## C

### Canary Deployment
Stratégie de déploiement où une nouvelle version est déployée progressivement sur un petit pourcentage d'utilisateurs avant un déploiement complet.

**Analogie :** Comme tester un nouveau plat sur quelques clients avant de le mettre à la carte pour tout le monde.

---

### Cloud Native
Approche de construction et d'exécution d'applications qui exploite les avantages du cloud computing. Kubernetes est une plateforme cloud native.

**Caractéristiques :** Microservices, conteneurs, orchestration, automation.

---

### Cluster 🎯
Ensemble de machines (physiques ou virtuelles) qui exécutent des applications conteneurisées orchestrées par Kubernetes.

**Composition :**
- **Control Plane** : Gère le cluster
- **Worker Nodes** : Exécutent les applications

**Analogie :** Un cluster est comme une équipe de travail où certains membres gèrent (control plane) et d'autres exécutent (workers).

---

### ClusterIP 🎯
Type de **Service** qui expose l'application uniquement à l'intérieur du **cluster** via une IP interne.

**Usage :** Communication entre microservices internes.

**Exemple :** Un service de base de données accessible uniquement par les autres applications du cluster.

---

### CNI (Container Network Interface)
Plugin qui permet aux **conteneurs** de communiquer sur le réseau. Kubernetes utilise des plugins CNI pour la mise en réseau.

**Exemples :** Calico, Flannel, Weave.

---

### ConfigMap 🎯
Objet Kubernetes qui stocke des données de configuration non sensibles sous forme de paires clé-valeur.

**Usage :** Externaliser la configuration des applications.

**Analogie :** C'est comme un fichier de configuration que votre application peut lire.

**Exemple :**
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
data:
  database_url: "postgres://db:5432"
  log_level: "info"
```

---

### Container 🎯
Unité logicielle standard qui package le code et toutes ses dépendances pour que l'application s'exécute de manière fiable d'un environnement à un autre.

**Technologie :** Docker, containerd, CRI-O.

**Analogie :** C'est comme une boîte de déménagement - tout ce dont vous avez besoin est à l'intérieur, prêt à être utilisé n'importe où.

---

### Container Runtime
Logiciel responsable de l'exécution des conteneurs.

**Exemples :** Docker, containerd, CRI-O.

**Note :** MicroK8s utilise containerd par défaut.

---

### Context (Contexte)
Configuration qui définit quel **cluster** Kubernetes utiliser et avec quelles credentials.

**Utilité :** Basculer facilement entre plusieurs clusters (dev, staging, production).

**Commandes :**
```bash
kubectl config get-contexts
kubectl config use-context production
```

---

### Control Plane 🎯
Ensemble des composants qui gèrent le **cluster** Kubernetes.

**Composants :**
- **API Server** : Point d'entrée
- **Scheduler** : Place les pods sur les nœuds
- **Controller Manager** : Maintient l'état désiré
- **etcd** : Base de données du cluster

**Analogie :** C'est comme le cerveau et le système nerveux qui contrôlent le corps.

---

### Controller 🎯
Boucle de contrôle qui surveille l'état du **cluster** et effectue des changements pour atteindre l'état désiré.

**Exemples :**
- **Deployment Controller** : Gère les Deployments
- **ReplicaSet Controller** : Maintient le nombre de réplicas
- **Job Controller** : Gère les Jobs

**Principe :** Observe → Compare → Agit (pour corriger les écarts).

---

### CrashLoopBackOff
État d'un **Pod** qui démarre, crash, redémarre, crash à nouveau, etc.

**Cause fréquente :** Erreur dans l'application, configuration incorrecte, dépendance manquante.

**Diagnostic :** `kubectl logs pod-name --previous`

---

### CRI (Container Runtime Interface)
Interface standard qui permet à Kubernetes de communiquer avec différents **container runtimes**.

---

### CronJob 🎯
Ressource Kubernetes qui exécute des **Jobs** selon un planning défini (comme cron sous Linux).

**Usage :** Tâches planifiées (backups, nettoyage, rapports).

**Exemple :**
```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: backup
spec:
  schedule: "0 2 * * *"  # Tous les jours à 2h
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: backup
            image: backup-tool:latest
```

---

### CSI (Container Storage Interface)
Interface standard qui permet aux plugins de stockage de fonctionner avec Kubernetes.

---

## D

### DaemonSet 🎯
Ressource qui s'assure qu'une copie d'un **Pod** s'exécute sur tous (ou certains) **nœuds** du cluster.

**Usage :** Agents de monitoring, collecteurs de logs, pilotes réseau.

**Analogie :** C'est comme avoir un gardien de sécurité dans chaque bâtiment d'un campus.

**Exemple :** Un collecteur de logs qui doit être présent sur chaque serveur.

---

### Declarative Configuration 🎯
Approche où vous décrivez l'état désiré du système, et Kubernetes se charge de l'atteindre.

**Opposé :** Configuration impérative (vous donnez des commandes explicites).

**Exemple :** Fichiers YAML décrivant ce que vous voulez, pas comment l'obtenir.

---

### Deployment 🎯
Ressource de haut niveau qui gère les **ReplicaSets** et fournit des mises à jour déclaratives pour les **Pods**.

**Fonctionnalités :**
- Rolling updates
- Rollback
- Scaling
- Historique des versions

**Usage :** Déployer et gérer des applications stateless.

**Analogie :** C'est comme un chef d'équipe qui s'assure que le bon nombre de travailleurs (pods) fait le travail, et gère les remplacements si nécessaire.

---

### Docker
Plateforme pour développer, livrer et exécuter des applications dans des **conteneurs**.

**Note :** Kubernetes peut utiliser Docker, mais utilise souvent containerd directement.

---

### Downtime
Période pendant laquelle un service n'est pas disponible.

**Objectif Kubernetes :** Minimiser ou éliminer le downtime avec les rolling updates et la haute disponibilité.

---

### Drain
Action de vider un **nœud** de tous ses **Pods** de manière contrôlée, généralement avant maintenance.

**Commande :** `kubectl drain node-name`

---

### Dry Run
Mode de simulation qui montre ce qui se passerait sans réellement exécuter l'action.

**Commande :** `kubectl apply -f deployment.yaml --dry-run=client`

**Utilité :** Tester avant d'appliquer.

---

## E

### Egress
Trafic réseau sortant d'un **Pod** ou d'un cluster.

**Opposé :** **Ingress** (trafic entrant).

**Exemple :** Un pod qui appelle une API externe.

---

### EmptyDir
Type de **volume** temporaire créé avec un **Pod** et supprimé quand le Pod est détruit.

**Usage :** Stockage temporaire, cache, partage entre conteneurs d'un même Pod.

---

### Endpoint 🎯
Adresse IP et port d'un **Pod** qui fait partie d'un **Service**.

**Commande :** `kubectl get endpoints service-name`

**Important :** Si un Service n'a pas d'endpoints, c'est souvent que le **selector** ne correspond à aucun Pod.

---

### etcd 🎯
Base de données clé-valeur distribuée qui stocke toutes les données du **cluster** Kubernetes.

**Contenu :** Configuration, état de tous les objets, secrets.

**Analogie :** C'est la mémoire permanente du cluster.

**Criticité :** Très critique - sauvegardez régulièrement !

---

### Event (Événement) 🎯
Enregistrement d'un changement ou d'une action dans le cluster.

**Commande :** `kubectl get events --sort-by='.lastTimestamp'`

**Utilité :** Premier endroit à consulter lors du debugging.

**Durée de vie :** Les événements sont généralement conservés 1 heure.

---

### Eviction (Éviction)
Action de retirer un **Pod** d'un **nœud**, généralement à cause d'un manque de ressources.

**Causes :** Mémoire insuffisante, disque plein, pression sur le nœud.

---

## F

### Finalizer
Mécanisme qui empêche la suppression d'un objet jusqu'à ce que certaines conditions soient remplies.

**Usage :** Nettoyage de ressources associées avant suppression.

---

### Flannel
Plugin **CNI** pour la mise en réseau des **Pods** dans Kubernetes.

**Alternative à :** Calico, Weave.

---

## G

### Grace Period
Temps accordé à un **Pod** pour se terminer proprement avant d'être forcé à s'arrêter.

**Défaut :** 30 secondes.

**Processus :**
1. SIGTERM envoyé au conteneur
2. Attente du grace period
3. SIGKILL si toujours actif

---

## H

### Health Check
Mécanisme pour vérifier qu'un **Pod** fonctionne correctement. Voir **Liveness Probe** et **Readiness Probe**.

---

### Helm
Gestionnaire de packages pour Kubernetes. Permet de définir, installer et gérer des applications complexes.

**Concept :** Charts (packages Helm) contenant des templates de ressources.

**Analogie :** C'est comme apt ou yum pour Linux, mais pour Kubernetes.

---

### HostPath
Type de **volume** qui monte un chemin du système de fichiers du **nœud** hôte dans un **Pod**.

**⚠️ Attention :** Risque de sécurité - à utiliser avec précaution.

**Usage :** Accès à des fichiers du nœud (logs, Docker socket).

---

### HPA (Horizontal Pod Autoscaler) 🎯
Contrôleur qui ajuste automatiquement le nombre de **Pods** dans un **Deployment** basé sur l'utilisation des ressources.

**Métriques :** CPU, mémoire, métriques personnalisées.

**Exemple :** Passer de 2 à 10 pods quand le CPU dépasse 70%.

---

## I

### Image 🎯
Template en lecture seule contenant tout le nécessaire pour exécuter une application (**conteneur**).

**Format :** `registry/repository:tag`

**Exemple :** `nginx:1.21`, `myapp:v2.0`

**Registries :** Docker Hub, GitHub Container Registry, registries privés.

---

### ImagePullBackOff
État d'erreur quand Kubernetes ne peut pas télécharger une **image** de conteneur.

**Causes courantes :**
- Image n'existe pas
- Nom/tag incorrect
- Registry privé sans credentials
- Problème réseau

---

### Immutable
Qui ne peut pas être modifié après création. Principe important dans Kubernetes.

**Exemple :** Les **Pods** sont généralement traités comme immutables - on les remplace plutôt que de les modifier.

---

### Imperative Configuration
Approche où vous donnez des commandes explicites décrivant comment atteindre un état.

**Exemple :** `kubectl run nginx --image=nginx`

**Opposé :** **Declarative Configuration** (recommandée).

---

### Ingress 🎯
Ressource qui gère l'accès HTTP/HTTPS externe au **cluster**.

**Fonctionnalités :**
- Routage basé sur l'hôte
- Routage basé sur le chemin
- Terminaison TLS/SSL
- Load balancing

**Analogie :** C'est comme le réceptionniste d'un hôtel qui dirige les visiteurs vers les bonnes chambres.

**Exemple :**
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: my-ingress
spec:
  rules:
  - host: example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: frontend
            port:
              number: 80
```

---

### Ingress Controller 🎯
Composant qui implémente les règles définies dans les ressources **Ingress**.

**Exemples :** NGINX Ingress Controller, Traefik, HAProxy.

**Note :** L'Ingress resource seule ne fait rien - il faut un Ingress Controller.

---

### Init Container
Conteneur spécialisé qui s'exécute avant les conteneurs principaux d'un **Pod**.

**Usage :**
- Initialisation
- Attente de dépendances
- Configuration préalable

**Caractéristique :** Doit se terminer avec succès avant que les conteneurs principaux démarrent.

---

## J

### Job 🎯
Ressource qui crée un ou plusieurs **Pods** et s'assure qu'un nombre spécifié se termine avec succès.

**Usage :** Tâches ponctuelles (migrations, calculs batch).

**Différence avec Deployment :** Job se termine, Deployment reste actif.

**Exemple :**
```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: data-migration
spec:
  template:
    spec:
      containers:
      - name: migrate
        image: migration-tool:latest
      restartPolicy: OnFailure
```

---

## K

### k8s
Abréviation de Kubernetes (K + 8 lettres + s).

---

### kube-proxy
Composant réseau qui s'exécute sur chaque **nœud** et gère les règles réseau pour les **Services**.

**Rôle :** Permet la communication réseau vers les Pods via les Services.

---

### Kubeconfig
Fichier de configuration qui contient les informations pour se connecter à un ou plusieurs **clusters** Kubernetes.

**Emplacement par défaut :** `~/.kube/config`

**Contenu :** Clusters, utilisateurs, contextes.

---

### kubectl 🎯
Outil en ligne de commande pour interagir avec les **clusters** Kubernetes.

**Prononciation :** "koube-see-tee-el" ou "koube-control"

**Commandes courantes :**
```bash
kubectl get pods
kubectl describe pod nginx
kubectl logs myapp
kubectl apply -f deployment.yaml
```

---

### Kubelet 🎯
Agent qui s'exécute sur chaque **nœud** et s'assure que les **conteneurs** fonctionnent dans les **Pods**.

**Rôle :**
- Reçoit les **PodSpecs** de l'**API Server**
- Garantit que les conteneurs décrits sont en cours d'exécution et sains

**Analogie :** C'est le superviseur local sur chaque machine.

---

### Kubernetes 🎯
Système open-source d'orchestration de conteneurs pour automatiser le déploiement, la mise à l'échelle et la gestion d'applications conteneurisées.

**Origine :** Développé par Google, basé sur Borg.

**Nom :** Grec pour "timonier" ou "pilote".

**Logo :** Roue de navire à 7 rayons (référence à "Seven of Nine" de Star Trek).

---

## L

### Label 🎯
Paire clé-valeur attachée aux objets Kubernetes pour les identifier et les organiser.

**Usage :** Sélection, filtrage, groupement.

**Exemple :**
```yaml
metadata:
  labels:
    app: frontend
    environment: production
    version: v2.0
```

**Commandes :**
```bash
kubectl get pods -l app=frontend
kubectl get pods -l environment=production,version=v2.0
```

---

### Limit 🎯
Quantité maximale de ressources (CPU, mémoire) qu'un **conteneur** peut utiliser.

**Exemple :**
```yaml
resources:
  limits:
    memory: "512Mi"
    cpu: "500m"
```

**Important :** Si dépassé, le conteneur peut être throttled (CPU) ou killed (mémoire).

---

### LimitRange
Politique qui contraint les **requests** et **limits** de ressources dans un **namespace**.

**Usage :** Empêcher qu'un seul Pod consomme toutes les ressources.

---

### Liveness Probe 🎯
Vérification qui détermine si un **conteneur** est vivant. Si la probe échoue, Kubernetes redémarre le conteneur.

**Types :**
- HTTP GET
- TCP Socket
- Exec Command

**Exemple :**
```yaml
livenessProbe:
  httpGet:
    path: /health
    port: 8080
  initialDelaySeconds: 30
  periodSeconds: 10
```

**Usage :** Détecter les deadlocks ou applications gelées.

---

### LoadBalancer 🎯
Type de **Service** qui expose l'application à l'extérieur du **cluster** via un load balancer externe.

**Note :** Nécessite généralement un provider cloud (AWS, GCP, Azure).

**Alternative locale :** MetalLB pour MicroK8s.

---

## M

### Manifest 🎯
Fichier (généralement YAML ou JSON) qui décrit un objet Kubernetes.

**Exemple :** Un fichier `deployment.yaml` contenant la définition d'un Deployment.

---

### Master Node
Ancien terme pour les nœuds exécutant le **Control Plane**.

**Terme moderne préféré :** Control Plane Node.

---

### Metadata 🎯
Section d'un **manifest** contenant les informations identifiantes d'un objet.

**Contenu typique :**
- `name` : Nom de l'objet (obligatoire)
- `namespace` : Namespace (optionnel)
- `labels` : Labels
- `annotations` : Annotations

---

### Metrics Server
Addon qui collecte les métriques de ressources (CPU, mémoire) des **Pods** et **nœuds**.

**Nécessaire pour :** `kubectl top`, **HPA**.

**Activation MicroK8s :** `microk8s enable metrics-server`

---

### MicroK8s 🎯
Distribution légère de Kubernetes développée par Canonical, conçue pour être facile à installer et à utiliser.

**Avantages :**
- Installation simple
- Faible empreinte ressource
- Addons préconfigurés
- Idéal pour dev/test

---

### Microservice
Architecture où une application est composée de plusieurs petits services indépendants.

**Kubernetes** est idéal pour orchestrer des microservices.

---

### Multi-tenancy
Capacité d'un cluster à héberger en toute sécurité des applications de différents utilisateurs/équipes.

**Isolation via :** Namespaces, RBAC, Network Policies, Resource Quotas.

---

## N

### Namespace 🎯
Mécanisme pour isoler des groupes de ressources dans un même **cluster**.

**Usage :** Environnements (dev, staging, prod), équipes, projets.

**Namespaces par défaut :**
- `default` : Ressources sans namespace spécifié
- `kube-system` : Composants système Kubernetes
- `kube-public` : Ressources publiques accessibles à tous
- `kube-node-lease` : Informations de heartbeat des nœuds

**Commandes :**
```bash
kubectl get namespaces
kubectl create namespace production
kubectl get pods -n production
```

**Analogie :** C'est comme des dossiers pour organiser vos fichiers.

---

### Network Policy 🎯
Règles définissant comment les **Pods** peuvent communiquer entre eux et avec d'autres endpoints réseau.

**Usage :** Sécurité, isolation réseau entre applications.

**Nécessite :** Un plugin **CNI** qui supporte les Network Policies (Calico).

**Exemple :**
```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-frontend
spec:
  podSelector:
    matchLabels:
      app: backend
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: frontend
    ports:
    - protocol: TCP
      port: 8080
```

---

### Node 🎯
Machine (physique ou virtuelle) qui exécute des applications conteneurisées.

**Types :**
- **Control Plane Node** : Gère le cluster
- **Worker Node** : Exécute les applications

**Commandes :**
```bash
kubectl get nodes
kubectl describe node node-name
kubectl top nodes
```

---

### NodePort 🎯
Type de **Service** qui expose l'application sur un port statique de chaque **nœud**.

**Plage de ports :** 30000-32767

**Usage :** Accès externe simple sans load balancer.

**Exemple :**
```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-service
spec:
  type: NodePort
  selector:
    app: myapp
  ports:
  - port: 80
    targetPort: 8080
    nodePort: 30080
```

---

### Node Selector 🎯
Mécanisme simple pour contraindre les **Pods** à s'exécuter sur certains **nœuds**.

**Exemple :**
```yaml
spec:
  nodeSelector:
    disktype: ssd
```

**Alternative plus flexible :** **Affinity**.

---

## O

### OOMKilled
État d'un conteneur tué par le système d'exploitation car il a dépassé sa limite de mémoire (Out Of Memory).

**Exit Code :** 137

**Solution :** Augmenter les **limits** de mémoire ou optimiser l'application.

---

### Orchestration
Automatisation de la configuration, coordination et gestion d'applications et services.

**Kubernetes** est un orchestrateur de conteneurs.

---

## P

### PDB (Pod Disruption Budget)
Limite le nombre de **Pods** d'une application qui peuvent être indisponibles simultanément lors d'interruptions volontaires.

**Usage :** Maintenir la disponibilité pendant les maintenances.

---

### Pending
État d'un **Pod** qui attend d'être programmé sur un **nœud**.

**Causes courantes :**
- Ressources insuffisantes
- **PVC** non disponible
- Contraintes de scheduling non satisfaites

---

### Persistent Volume (PV) 🎯
Morceau de stockage dans le **cluster** provisionné par un administrateur ou dynamiquement via une **StorageClass**.

**Cycle de vie :** Indépendant des Pods.

**Exemple :**
```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv-data
spec:
  capacity:
    storage: 10Gi
  accessModes:
  - ReadWriteOnce
  hostPath:
    path: /mnt/data
```

---

### Persistent Volume Claim (PVC) 🎯
Demande de stockage par un utilisateur. Comme un "bon" pour obtenir un **PV**.

**Analogie :** Un PV est un parking, un PVC est votre ticket de parking.

**Exemple :**
```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: data-pvc
spec:
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 5Gi
```

---

### Pod 🎯
Plus petite unité déployable dans Kubernetes. Un Pod contient un ou plusieurs **conteneurs** qui partagent le réseau et le stockage.

**Caractéristiques :**
- Un Pod = une instance de l'application
- Les conteneurs d'un Pod partagent localhost
- Éphémère par nature

**Analogie :** Un Pod est comme un petit appartement - plusieurs personnes (conteneurs) peuvent y vivre et partager les ressources.

**Exemple :**
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx
spec:
  containers:
  - name: nginx
    image: nginx:1.21
    ports:
    - containerPort: 80
```

---

### PodSpec
Section d'un **manifest** qui définit le conteneur(s) et les volumes d'un **Pod**.

---

### Port-Forward
Redirection d'un port local vers un port de **Pod** pour tests/debugging.

**Commande :** `kubectl port-forward pod/nginx 8080:80`

**Usage :** Accéder temporairement à un Pod sans exposer de Service.

---

### Probe 🎯
Vérification périodique effectuée par le **kubelet** sur un conteneur.

**Types :**
- **Liveness Probe** : Redémarre si échoue
- **Readiness Probe** : Retire du Service si échoue
- **Startup Probe** : Pour applications à démarrage lent

---

### PVC
Voir **Persistent Volume Claim**.

---

## Q

### QoS (Quality of Service)
Classe assignée à un **Pod** basée sur ses **requests** et **limits**.

**Classes (par ordre de priorité) :**
1. **Guaranteed** : Requests = Limits
2. **Burstable** : Requests < Limits
3. **BestEffort** : Pas de requests ni limits

**Impact :** En cas de pression sur les ressources, les BestEffort sont tués en premier.

---

### Quota
Voir **ResourceQuota**.

---

## R

### RBAC (Role-Based Access Control) 🎯
Système de gestion des permissions dans Kubernetes.

**Composants :**
- **Role** / **ClusterRole** : Définit les permissions
- **RoleBinding** / **ClusterRoleBinding** : Attribue les permissions

**Exemple :**
```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: pod-reader
rules:
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["get", "list"]
```

---

### Readiness Probe 🎯
Vérification qui détermine si un **conteneur** est prêt à recevoir du trafic. Si la probe échoue, le Pod est retiré des **endpoints** du **Service**.

**Exemple :**
```yaml
readinessProbe:
  httpGet:
    path: /ready
    port: 8080
  initialDelaySeconds: 5
  periodSeconds: 5
```

**Différence avec Liveness :** Readiness retire du service (temporairement), Liveness redémarre (définitivement).

---

### Reconciliation Loop
Boucle continue d'un **controller** qui compare l'état actuel à l'état désiré et effectue des actions pour les aligner.

**Principe fondamental de Kubernetes.**

---

### Registry
Service qui stocke et distribue des **images** de conteneurs.

**Exemples :**
- Docker Hub (public)
- GitHub Container Registry
- Google Container Registry
- Registry privé

**MicroK8s :** Peut activer son propre registry local.

---

### Replica 🎯
Copie identique d'un **Pod**.

**Usage :** Haute disponibilité, répartition de charge.

**Exemple :** Un Deployment avec `replicas: 3` crée 3 Pods identiques.

---

### ReplicaSet 🎯
Ressource qui maintient un nombre stable de **Pods** répliques en cours d'exécution.

**Note :** Généralement géré par un **Deployment**, rarement créé directement.

**Rôle :** Si un Pod meurt, le ReplicaSet en crée un nouveau.

---

### Request 🎯
Quantité minimum de ressources (CPU, mémoire) qu'un **conteneur** a besoin.

**Exemple :**
```yaml
resources:
  requests:
    memory: "256Mi"
    cpu: "250m"
```

**Usage :** Le **scheduler** utilise les requests pour décider où placer le Pod.

---

### Resource
Objet géré par Kubernetes (Pod, Service, Deployment, etc.).

**Types de ressources :**
- **Workloads** : Pod, Deployment, StatefulSet, Job
- **Services** : Service, Ingress
- **Config** : ConfigMap, Secret
- **Storage** : PV, PVC

---

### ResourceQuota 🎯
Contrainte qui limite la consommation totale de ressources dans un **namespace**.

**Usage :** Empêcher qu'un namespace consomme toutes les ressources du cluster.

**Exemple :**
```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: compute-quota
  namespace: production
spec:
  hard:
    requests.cpu: "10"
    requests.memory: 20Gi
    limits.cpu: "20"
    limits.memory: 40Gi
    pods: "100"
```

---

### Rolling Update 🎯
Stratégie de mise à jour progressive où les anciens **Pods** sont remplacés graduellement par de nouveaux.

**Avantage :** Zéro downtime.

**Configuration :**
```yaml
strategy:
  type: RollingUpdate
  rollingUpdate:
    maxSurge: 1
    maxUnavailable: 0
```

---

### Rollback 🎯
Action de revenir à une version précédente d'un **Deployment**.

**Commande :** `kubectl rollout undo deployment/app`

**Utile quand :** Nouvelle version défectueuse.

---

### Rollout 🎯
Processus de mise à jour d'un **Deployment**.

**Commandes :**
```bash
kubectl rollout status deployment/app
kubectl rollout history deployment/app
kubectl rollout restart deployment/app
```

---

## S

### Scale (Scaling) 🎯
Ajuster le nombre de **réplicas** d'une application.

**Types :**
- **Horizontal** : Ajouter/retirer des Pods
- **Vertical** : Augmenter/diminuer les ressources par Pod

**Commande :** `kubectl scale deployment/app --replicas=5`

---

### Scheduler 🎯
Composant du **Control Plane** qui assigne les **Pods** aux **nœuds**.

**Critères :**
- Ressources disponibles
- Contraintes (affinity, taints/tolerations)
- Politique de répartition

**Analogie :** C'est comme un gestionnaire d'hôtel qui attribue les chambres aux clients.

---

### Secret 🎯
Objet Kubernetes qui stocke des données sensibles (mots de passe, tokens, clés SSH).

**Différence avec ConfigMap :** Les données sont encodées en base64 et traitées avec plus de précautions de sécurité.

**Types :**
- `Opaque` : Secret générique
- `kubernetes.io/tls` : Certificats TLS
- `kubernetes.io/dockerconfigjson` : Credentials Docker

**Exemple :**
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: db-secret
type: Opaque
stringData:
  username: admin
  password: secretpassword123
```

---

### Selector 🎯
Critère de sélection basé sur les **labels** pour identifier un ensemble d'objets.

**Usage :** Services sélectionnent les Pods, Deployments sélectionnent les ReplicaSets.

**Types :**
- **Equality-based** : `app=nginx`
- **Set-based** : `environment in (prod, staging)`

**Exemple :**
```yaml
selector:
  matchLabels:
    app: myapp
    tier: backend
```

---

### Service 🎯
Abstraction qui définit un ensemble logique de **Pods** et une politique pour y accéder.

**Types :**
- **ClusterIP** : Accès interne uniquement
- **NodePort** : Expose sur un port de chaque nœud
- **LoadBalancer** : Expose via un load balancer externe
- **ExternalName** : Alias DNS vers un service externe

**Rôle :** Fournit une IP/DNS stable même si les Pods changent.

**Analogie :** C'est comme un numéro de téléphone fixe pour une entreprise - même si les employés changent, le numéro reste le même.

---

### Service Account 🎯
Identité pour les processus qui s'exécutent dans un **Pod**.

**Usage :** Authentification des Pods auprès de l'**API Server**.

**Par défaut :** Chaque namespace a un Service Account `default`.

---

### Service Discovery
Mécanisme permettant aux applications de trouver et communiquer avec des **Services**.

**Méthodes Kubernetes :**
- DNS (recommandé)
- Variables d'environnement

---

### Service Mesh
Couche d'infrastructure dédiée pour gérer la communication service-à-service.

**Exemples :** Istio, Linkerd.

**Fonctionnalités :** Observabilité, sécurité, traffic management.

---

### Sidecar
**Conteneur** supplémentaire dans un **Pod** qui étend ou améliore le conteneur principal.

**Usages courants :**
- Collecte de logs
- Proxy (service mesh)
- Surveillance

**Exemple :** Container Fluentd à côté d'une application web pour collecter les logs.

---

### Spec 🎯
Section d'un **manifest** qui définit l'état désiré de la ressource.

**Exemple :**
```yaml
spec:
  replicas: 3
  containers:
  - name: nginx
    image: nginx:1.21
```

---

### StatefulSet 🎯
Ressource pour gérer des applications stateful (avec état).

**Caractéristiques :**
- Identités réseau stables et uniques
- Stockage persistant stable
- Ordre de déploiement et de scaling garanti

**Usage :** Bases de données, systèmes distribués.

**Différence avec Deployment :** Les Pods ont des noms prévisibles (app-0, app-1, app-2) au lieu de noms aléatoires.

---

### Status
Section d'un objet Kubernetes qui contient l'état actuel observé (géré par Kubernetes, en lecture seule pour l'utilisateur).

---

### Storage Class 🎯
Définit les différentes classes de stockage disponibles dans un **cluster**.

**Usage :** Provisionnement dynamique de **PV**.

**Exemples :** fast (SSD), slow (HDD), replicated.

**Commande :** `kubectl get storageclass`

---

## T

### Taint
Propriété d'un **nœud** qui repousse certains **Pods**.

**Complémentaire de :** **Toleration** (permet à un Pod d'ignorer une taint).

**Usage :** Réserver des nœuds pour certaines charges de travail.

**Exemple :**
```bash
kubectl taint nodes node1 key=value:NoSchedule
```

---

### Toleration
Permet à un **Pod** d'être programmé sur un **nœud** avec une **taint** correspondante.

**Exemple :**
```yaml
spec:
  tolerations:
  - key: "key"
    operator: "Equal"
    value: "value"
    effect: "NoSchedule"
```

---

### TTL (Time To Live)
Durée de vie d'un objet avant suppression automatique.

**Usage :** Nettoyage automatique de **Jobs** terminés.

---

## U

### UID (Unique Identifier)
Identifiant unique généré par Kubernetes pour chaque objet.

**Format :** UUID

**Exemple :** `6f8c8944-91a5-45e7-a8aa-0e6e2a3b5c0d`

---

### Update Strategy
Stratégie définissant comment les **Pods** sont mis à jour.

**Types :**
- **RollingUpdate** : Progressif (défaut)
- **Recreate** : Tout arrêter puis recréer

---

## V

### Volume 🎯
Répertoire accessible aux **conteneurs** d'un **Pod**.

**Types courants :**
- **emptyDir** : Temporaire, vie du Pod
- **hostPath** : Chemin du nœud hôte
- **persistentVolumeClaim** : Stockage persistant
- **configMap** : Depuis un ConfigMap
- **secret** : Depuis un Secret

**Exemple :**
```yaml
spec:
  containers:
  - name: app
    volumeMounts:
    - name: data
      mountPath: /data
  volumes:
  - name: data
    persistentVolumeClaim:
      claimName: data-pvc
```

---

### VPA (Vertical Pod Autoscaler)
Ajuste automatiquement les **requests** et **limits** de ressources d'un **Pod**.

**Complémentaire de :** **HPA** (qui ajuste le nombre de Pods).

---

## W

### Watch
Mécanisme pour suivre les changements sur des ressources en temps réel.

**Commande :** `kubectl get pods --watch`

---

### Worker Node 🎯
**Nœud** qui exécute les applications conteneurisées (**Pods**).

**Composants :**
- **Kubelet** : Agent du nœud
- **Container Runtime** : Exécute les conteneurs
- **kube-proxy** : Gère le réseau

---

### Workload 🎯
Application ou service s'exécutant sur Kubernetes.

**Ressources de workload :**
- Pod
- Deployment
- StatefulSet
- DaemonSet
- Job
- CronJob

---

## Y

### YAML 🎯
Format de sérialisation de données lisible par l'homme, utilisé pour écrire les **manifests** Kubernetes.

**Caractéristiques :**
- Basé sur l'indentation (2 espaces)
- Pas de tabulations
- Sensible aux espaces

**Exemple :**
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx
spec:
  containers:
  - name: nginx
    image: nginx:1.21
```

---

## Z

### Zero Downtime Deployment
Déploiement d'une nouvelle version sans interruption de service.

**Méthodes Kubernetes :**
- Rolling Updates
- Blue-Green Deployment
- Canary Deployment

---

## Concepts Transversaux

### Architecture Kubernetes

```
┌─────────────────────────────────────────┐
│         CONTROL PLANE                   │
│  ┌──────────┐  ┌──────────┐  ┌──────┐   │
│  │API Server│  │Scheduler │  │ etcd │   │
│  └──────────┘  └──────────┘  └──────┘   │
│  ┌───────────────────────────────────┐  │
│  │    Controller Manager             │  │
│  └───────────────────────────────────┘  │
└─────────────────────────────────────────┘
                    ↓
┌─────────────────────────────────────────┐
│         WORKER NODES                    │
│  ┌──────────────────────────────────┐   │
│  │  Node 1                          │   │
│  │  ┌────────┐  ┌─────────────────┐ │   │
│  │  │Kubelet │  │Container Runtime│ │   │
│  │  └────────┘  └─────────────────┘ │   │
│  │  ┌────────────────────────────┐  │   │
│  │  │    Pods                    │  │   │
│  │  └────────────────────────────┘  │   │
│  └──────────────────────────────────┘   │
│  ┌──────────────────────────────────┐   │
│  │  Node 2 (similaire)              │   │
│  └──────────────────────────────────┘   │
└─────────────────────────────────────────┘
```

---

### Cycle de Vie d'un Pod

```
┌─────────────────────────────────────────┐
│  1. PENDING                             │
│     Pod créé, en attente d'assignation  │
└────────────────┬────────────────────────┘
                 │
                 ↓
┌─────────────────────────────────────────┐
│  2. CONTAINER CREATING                  │
│     Images en téléchargement            │
└────────────────┬────────────────────────┘
                 │
                 ↓
┌─────────────────────────────────────────┐
│  3. RUNNING                             │
│     Tous les conteneurs actifs          │
└────────────────┬────────────────────────┘
                 │
      ┌──────────┴──────────┐
      ↓                     ↓
┌─────────────┐       ┌───────────┐
│  SUCCEEDED  │       │  FAILED   │
│  (Job OK)   │       │  (Erreur) │
└─────────────┘       └───────────┘
```

---

### Hiérarchie des Ressources de Workload

```
Deployment
    │
    ├─→ ReplicaSet (version 1)
    │       ├─→ Pod 1
    │       ├─→ Pod 2
    │       └─→ Pod 3
    │
    └─→ ReplicaSet (version 2)
            ├─→ Pod 4
            └─→ Pod 5
```

---

### Flux de Requête HTTP

```
Internet
    │
    ↓
[Ingress Controller]
    │
    ↓
[Ingress Rules]
    │
    ↓
[Service]
    │
    ├─→ Pod 1
    ├─→ Pod 2
    └─→ Pod 3
```

---

## Mnémoniques et Astuces

### Les 3 "P" de Base
- **Pod** : Conteneur(s) qui tournent
- **PVC** : Demande de stockage
- **Port** : Exposition réseau

### Les 3 "S" Essentiels
- **Service** : Accès réseau aux Pods
- **Secret** : Données sensibles
- **Selector** : Choisir les Pods

### Les 3 Niveaux de Ressources
1. **Pod** : Un conteneur ou groupe de conteneurs
2. **ReplicaSet** : Maintient N Pods identiques
3. **Deployment** : Gère les ReplicaSets avec historique

### Aide-mémoire Probes
- **Liveness** = "Es-tu vivant?" → Redémarre si non
- **Readiness** = "Es-tu prêt?" → Retire du service si non
- **Startup** = "As-tu fini de démarrer?" → Pour apps lentes

### Unités de Ressources
- **CPU** : `1` = 1 core, `500m` = 0.5 core
- **Mémoire** : `Ki`, `Mi`, `Gi` (1Ki = 1024 bytes)

---

## Abréviations Courantes

| Abréviation | Signification |
|-------------|---------------|
| k8s | Kubernetes |
| PV | PersistentVolume |
| PVC | PersistentVolumeClaim |
| HPA | Horizontal Pod Autoscaler |
| VPA | Vertical Pod Autoscaler |
| RBAC | Role-Based Access Control |
| CNI | Container Network Interface |
| CRI | Container Runtime Interface |
| CSI | Container Storage Interface |
| SA | ServiceAccount |
| CM | ConfigMap |
| NS | Namespace |
| SVC | Service |
| DS | DaemonSet |
| RS | ReplicaSet |
| STS | StatefulSet |
| PDB | Pod Disruption Budget |

---

## Commandes kubectl par Ressource

| Ressource | Forme longue | Raccourci |
|-----------|--------------|-----------|
| pods | `kubectl get pods` | `kubectl get po` |
| services | `kubectl get services` | `kubectl get svc` |
| deployments | `kubectl get deployments` | `kubectl get deploy` |
| replicasets | `kubectl get replicasets` | `kubectl get rs` |
| statefulsets | `kubectl get statefulsets` | `kubectl get sts` |
| daemonsets | `kubectl get daemonsets` | `kubectl get ds` |
| namespaces | `kubectl get namespaces` | `kubectl get ns` |
| nodes | `kubectl get nodes` | `kubectl get no` |
| configmaps | `kubectl get configmaps` | `kubectl get cm` |
| persistentvolumes | `kubectl get persistentvolumes` | `kubectl get pv` |
| persistentvolumeclaims | `kubectl get persistentvolumeclaims` | `kubectl get pvc` |
| ingresses | `kubectl get ingresses` | `kubectl get ing` |

---

## Pour Aller Plus Loin

### Documentation Officielle
- **Kubernetes Docs** : https://kubernetes.io/docs/
- **Kubernetes Concepts** : https://kubernetes.io/docs/concepts/
- **API Reference** : https://kubernetes.io/docs/reference/kubernetes-api/

### Glossaire Officiel Kubernetes
- https://kubernetes.io/docs/reference/glossary/

### Ressources d'Apprentissage
- **Kubernetes Academy** : https://kube.academy/
- **Play with Kubernetes** : https://labs.play-with-k8s.com/
- **Katacoda Kubernetes** : Tutoriels interactifs

### Certifications
- **CKA** (Certified Kubernetes Administrator)
- **CKAD** (Certified Kubernetes Application Developer)
- **CKS** (Certified Kubernetes Security Specialist)

---

## Conclusion

Ce glossaire couvre les termes essentiels de Kubernetes. Voici les concepts absolument fondamentaux à maîtriser en premier :

### Top 10 des Termes à Connaître

1. **Pod** - L'unité de base
2. **Deployment** - Gère les applications
3. **Service** - Expose les applications
4. **Namespace** - Organise les ressources
5. **Label** - Identifie et sélectionne
6. **ConfigMap/Secret** - Configure les apps
7. **Ingress** - Routage HTTP/HTTPS
8. **Volume/PVC** - Stockage persistant
9. **kubectl** - L'outil en ligne de commande
10. **Node** - Machine exécutant les Pods

### Progression d'Apprentissage Suggérée

**Niveau 1 - Débutant** (commencez ici) :
- Pod, Container, Image
- Deployment, ReplicaSet
- Service (ClusterIP, NodePort)
- kubectl (get, describe, logs)
- Namespace, Label

**Niveau 2 - Intermédiaire** :
- ConfigMap, Secret
- Volume, PVC, PV
- Ingress, Ingress Controller
- Health Checks (Probes)
- Resources (Requests/Limits)

**Niveau 3 - Avancé** :
- StatefulSet, DaemonSet
- Network Policy
- RBAC
- HPA, VPA
- Custom Resources

Bonne découverte de Kubernetes ! 🚀

⏭️ Retour au [Sommaire](/SOMMAIRE.md)
