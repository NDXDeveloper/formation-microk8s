🔝 Retour au [Sommaire](/SOMMAIRE.md)

# Commandes kubectl Essentielles

## Introduction à kubectl

### Qu'est-ce que kubectl ?

**kubectl** (prononcé "koube-see-tee-el" ou "koube-control") est l'outil en ligne de commande officiel pour interagir avec Kubernetes. C'est votre interface principale pour gérer votre cluster, déployer des applications, inspecter les ressources et diagnostiquer les problèmes.

**Analogie :** Si Kubernetes est un système d'exploitation pour conteneurs, kubectl est comme votre terminal : l'outil qui vous permet de tout contrôler.

### kubectl et MicroK8s

Avec MicroK8s, vous avez deux options pour utiliser kubectl :

#### Option 1 : Via le préfixe MicroK8s (par défaut)

```bash
microk8s kubectl get pods
```

**Avantages :**
- Fonctionne immédiatement après installation
- Pas de conflit avec d'autres installations Kubernetes

#### Option 2 : Créer un alias

```bash
# Ajouter dans ~/.bashrc ou ~/.zshrc
alias kubectl='microk8s kubectl'

# Recharger la configuration
source ~/.bashrc

# Utilisation simplifiée
kubectl get pods
```

**Avantages :**
- Commandes plus courtes
- Compatible avec la documentation standard Kubernetes
- Habitudes transférables à d'autres environnements

#### Option 3 : Installer kubectl séparément

```bash
# Exporter la config MicroK8s
microk8s config > ~/.kube/config

# Installer kubectl standard
sudo snap install kubectl --classic

# Utiliser kubectl directement
kubectl get pods
```

**Dans ce guide :** Nous utiliserons `kubectl` dans les exemples. Si vous utilisez MicroK8s sans alias, ajoutez simplement `microk8s` devant chaque commande.

---

## Structure d'une Commande kubectl

### Syntaxe Générale

```bash
kubectl [commande] [type-ressource] [nom-ressource] [options]
```

**Exemples :**

```bash
kubectl get pods
│       │   │
│       │   └── Type de ressource
│       └────── Commande (action)
└────────────── Outil

kubectl delete pod nginx
│       │      │   │
│       │      │   └── Nom de la ressource
│       │      └────── Type de ressource
│       └────────────── Commande
└──────────────────────── Outil

kubectl get pods -n production
│       │   │    │  │
│       │   │    │  └── Nom du namespace
│       │   │    └────── Option namespace
│       │   └──────────── Type de ressource
│       └───────────────── Commande
└────────────────────────── Outil
```

### Les Trois Éléments Clés

1. **Commande** : L'action à effectuer (get, create, delete, etc.)
2. **Ressource** : Le type d'objet Kubernetes (pod, service, deployment, etc.)
3. **Options** : Modificateurs du comportement (namespace, format, labels, etc.)

---

## Commandes Fondamentales

### get - Lister des Ressources

La commande la plus utilisée pour voir l'état de vos ressources.

**Syntaxe :**
```bash
kubectl get [ressource] [nom] [options]
```

**Exemples de base :**

```bash
# Lister tous les pods
kubectl get pods

# Lister tous les pods dans tous les namespaces
kubectl get pods --all-namespaces
# ou
kubectl get pods -A

# Lister les pods d'un namespace spécifique
kubectl get pods -n production

# Lister un pod spécifique
kubectl get pod nginx

# Lister plusieurs types de ressources
kubectl get pods,services,deployments
```

**Formats de sortie :**

```bash
# Format large (plus d'informations)
kubectl get pods -o wide

# Résultat :
# NAME    READY   STATUS    RESTARTS   AGE   IP           NODE
# nginx   1/1     Running   0          5m    10.1.128.5   node1

# Format YAML complet
kubectl get pod nginx -o yaml

# Format JSON complet
kubectl get pod nginx -o json

# Afficher uniquement les noms
kubectl get pods -o name

# Format personnalisé avec colonnes
kubectl get pods -o custom-columns=NAME:.metadata.name,STATUS:.status.phase

# Afficher les labels
kubectl get pods --show-labels
```

**Exemples avec ressources communes :**

```bash
# Pods
kubectl get pods
kubectl get po              # Raccourci

# Services
kubectl get services
kubectl get svc             # Raccourci

# Déploiements
kubectl get deployments
kubectl get deploy          # Raccourci

# Namespaces
kubectl get namespaces
kubectl get ns              # Raccourci

# Nœuds
kubectl get nodes
kubectl get no              # Raccourci

# Tout dans un namespace
kubectl get all -n production
```

### describe - Détails Complets

Affiche des informations détaillées sur une ressource, incluant les événements récents.

**Syntaxe :**
```bash
kubectl describe [ressource] [nom]
```

**Exemples :**

```bash
# Décrire un pod
kubectl describe pod nginx

# Résultat typique :
# Name:         nginx
# Namespace:    default
# Node:         hostname/192.168.1.100
# Start Time:   Mon, 23 Oct 2023 10:30:00 +0000
# Labels:       app=nginx
# Status:       Running
# IP:           10.1.128.5
# Containers:
#   nginx:
#     Image:        nginx:latest
#     Port:         80/TCP
#     State:        Running
#     Ready:        True
# Events:
#   Type    Reason     Age   From               Message
#   ----    ------     ----  ----               -------
#   Normal  Scheduled  5m    default-scheduler  Successfully assigned
#   Normal  Pulled     5m    kubelet            Container image pulled
#   Normal  Created    5m    kubelet            Created container
#   Normal  Started    5m    kubelet            Started container

# Décrire un service
kubectl describe service nginx

# Décrire un déploiement
kubectl describe deployment nginx

# Décrire un nœud
kubectl describe node hostname

# Décrire dans un namespace spécifique
kubectl describe pod nginx -n production
```

**Quand utiliser describe :**
- Debugger un pod qui ne démarre pas
- Voir pourquoi un service ne fonctionne pas
- Examiner les événements récents
- Comprendre la configuration complète

### create - Créer une Ressource

Crée une nouvelle ressource de manière impérative.

**Syntaxe :**
```bash
kubectl create [type] [nom] [options]
```

**Exemples :**

```bash
# Créer un namespace
kubectl create namespace production

# Créer un déploiement
kubectl create deployment nginx --image=nginx

# Créer un déploiement avec plusieurs replicas
kubectl create deployment nginx --image=nginx --replicas=3

# Créer un service
kubectl create service clusterip nginx --tcp=80:80

# Créer un secret depuis des valeurs littérales
kubectl create secret generic db-secret \
  --from-literal=username=admin \
  --from-literal=password=secret123

# Créer un secret depuis un fichier
kubectl create secret generic tls-secret \
  --from-file=tls.crt=./cert.crt \
  --from-file=tls.key=./cert.key

# Créer un ConfigMap depuis des valeurs
kubectl create configmap app-config \
  --from-literal=api_url=https://api.example.com \
  --from-literal=log_level=info

# Créer un ConfigMap depuis un fichier
kubectl create configmap app-config --from-file=config.json

# Créer une ServiceAccount
kubectl create serviceaccount my-service-account

# Créer un Job
kubectl create job test-job --image=busybox -- echo "Hello World"
```

**Note :** `create` échoue si la ressource existe déjà. Utilisez `apply` pour une approche idempotente.

### apply - Appliquer une Configuration

Applique une configuration depuis un fichier. Crée la ressource si elle n'existe pas, la met à jour sinon.

**Syntaxe :**
```bash
kubectl apply -f [fichier-ou-url]
```

**Exemples :**

```bash
# Appliquer un fichier YAML
kubectl apply -f deployment.yaml

# Appliquer tous les fichiers d'un dossier
kubectl apply -f ./manifests/

# Appliquer récursivement tous les sous-dossiers
kubectl apply -f ./manifests/ -R

# Appliquer depuis une URL
kubectl apply -f https://raw.githubusercontent.com/example/app/main/deploy.yaml

# Appliquer avec dry-run (simuler sans appliquer)
kubectl apply -f deployment.yaml --dry-run=client

# Appliquer et voir les changements
kubectl apply -f deployment.yaml --dry-run=server

# Appliquer depuis stdin
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: nginx
spec:
  containers:
  - name: nginx
    image: nginx
EOF
```

**Différence create vs apply :**

| Commande | Comportement | Cas d'usage |
|----------|--------------|-------------|
| `create` | Échoue si existe | Création initiale impérative |
| `apply` | Crée ou met à jour | Gestion déclarative, GitOps |

**Exemple pratique :**

```bash
# Première fois : create et apply ont le même effet
kubectl apply -f app.yaml     # ✓ Crée la ressource

# Deuxième fois : comportement différent
kubectl create -f app.yaml    # ✗ Error: already exists
kubectl apply -f app.yaml     # ✓ Met à jour si changements
```

### delete - Supprimer une Ressource

Supprime une ou plusieurs ressources.

**Syntaxe :**
```bash
kubectl delete [type] [nom] [options]
```

**Exemples :**

```bash
# Supprimer un pod
kubectl delete pod nginx

# Supprimer plusieurs pods
kubectl delete pod nginx redis mysql

# Supprimer un déploiement
kubectl delete deployment nginx

# Supprimer par fichier
kubectl delete -f deployment.yaml

# Supprimer tous les fichiers d'un dossier
kubectl delete -f ./manifests/

# Supprimer par label
kubectl delete pods -l app=nginx

# Supprimer tous les pods dans un namespace
kubectl delete pods --all -n staging

# Supprimer tout dans un namespace (⚠️ dangereux)
kubectl delete all --all -n staging

# Forcer la suppression immédiate
kubectl delete pod nginx --force --grace-period=0

# Supprimer un namespace (supprime tout dedans)
kubectl delete namespace staging
```

**Options utiles :**

| Option | Description |
|--------|-------------|
| `--grace-period=SECONDS` | Temps d'attente avant terminaison forcée |
| `--force` | Force la suppression immédiate |
| `--all` | Supprime toutes les ressources du type |
| `-l, --selector` | Filtre par label |
| `--field-selector` | Filtre par champ |

**Suppressions conditionnelles :**

```bash
# Supprimer les pods Failed
kubectl delete pods --field-selector=status.phase==Failed

# Supprimer les pods Succeeded
kubectl delete pods --field-selector=status.phase==Succeeded

# Supprimer les pods d'un nœud spécifique
kubectl delete pods --field-selector=spec.nodeName==node-1
```

### edit - Éditer une Ressource

Ouvre la ressource dans un éditeur pour modification directe.

**Syntaxe :**
```bash
kubectl edit [type] [nom]
```

**Exemples :**

```bash
# Éditer un déploiement
kubectl edit deployment nginx

# Éditer un service
kubectl edit service nginx

# Éditer avec un éditeur spécifique
EDITOR=nano kubectl edit deployment nginx
EDITOR=vim kubectl edit deployment nginx

# Éditer dans un namespace
kubectl edit deployment nginx -n production
```

**Workflow typique :**

1. La commande ouvre votre éditeur par défaut
2. Vous modifiez le YAML
3. Vous sauvegardez et fermez l'éditeur
4. kubectl applique les changements

**⚠️ Attention :**
- Les modifications sont appliquées immédiatement
- Certains champs sont immuables (ne peuvent pas être modifiés)
- Préférez `kubectl apply` pour un meilleur contrôle de version

---

## Gestion des Pods

### logs - Voir les Logs

Affiche les logs d'un conteneur dans un pod.

**Syntaxe :**
```bash
kubectl logs [pod] [options]
```

**Exemples :**

```bash
# Logs d'un pod
kubectl logs nginx

# Logs en temps réel (follow)
kubectl logs nginx -f

# Logs avec timestamp
kubectl logs nginx --timestamps

# Dernières N lignes
kubectl logs nginx --tail=100

# Logs depuis X temps
kubectl logs nginx --since=1h
kubectl logs nginx --since=10m

# Logs du conteneur précédent (si pod crashé et redémarré)
kubectl logs nginx --previous

# Logs d'un conteneur spécifique (si multiple conteneurs dans le pod)
kubectl logs nginx -c nginx-container

# Logs de tous les pods d'un déploiement
kubectl logs deployment/nginx

# Logs de tous les pods avec un label
kubectl logs -l app=nginx

# Logs de tous les conteneurs dans un pod
kubectl logs nginx --all-containers=true

# Sauvegarder les logs dans un fichier
kubectl logs nginx > nginx-logs.txt
```

**Cas d'usage courants :**

```bash
# Débugger une application qui crash
kubectl logs problematic-pod --previous

# Surveiller en temps réel
kubectl logs -f app-pod

# Analyser les erreurs récentes
kubectl logs app-pod --tail=50 | grep -i error

# Exporter pour analyse
kubectl logs app-pod --since=24h > app-errors-24h.log
```

### exec - Exécuter des Commandes

Exécute une commande dans un conteneur d'un pod.

**Syntaxe :**
```bash
kubectl exec [pod] [options] -- [commande]
```

**Exemples :**

```bash
# Exécuter une commande simple
kubectl exec nginx -- ls /app

# Se connecter au pod (shell interactif)
kubectl exec -it nginx -- /bin/bash

# Si bash n'est pas disponible, utiliser sh
kubectl exec -it nginx -- /bin/sh

# Exécuter une commande avec arguments
kubectl exec nginx -- cat /etc/nginx/nginx.conf

# Dans un conteneur spécifique
kubectl exec -it nginx -c nginx-container -- /bin/bash

# Exécuter plusieurs commandes
kubectl exec nginx -- sh -c "ls -la && pwd && whoami"

# Variables d'environnement
kubectl exec nginx -- env

# Vérifier la connectivité réseau
kubectl exec nginx -- curl http://backend-service:8080

# Test DNS
kubectl exec nginx -- nslookup backend-service

# Voir les processus
kubectl exec nginx -- ps aux
```

**Options importantes :**

| Option | Description |
|--------|-------------|
| `-i, --stdin` | Passe stdin au conteneur |
| `-t, --tty` | Alloue un pseudo-TTY |
| `-c, --container` | Conteneur cible si multiple |

**Cas d'usage debugging :**

```bash
# Vérifier la configuration
kubectl exec app -- cat /etc/config/app.conf

# Tester la connectivité DB
kubectl exec app -- nc -zv database-service 5432

# Vérifier l'espace disque
kubectl exec app -- df -h

# Voir les logs d'application internes
kubectl exec app -- tail -f /var/log/app/error.log

# Installer un outil temporairement (si autorisé)
kubectl exec -it app -- sh
# Dans le pod:
apt-get update && apt-get install -y curl
curl http://api-service/health
```

### port-forward - Redirection de Ports

Redirige un port local vers un pod.

**Syntaxe :**
```bash
kubectl port-forward [pod|service/nom] [port-local]:[port-pod]
```

**Exemples :**

```bash
# Redirection simple
kubectl port-forward pod/nginx 8080:80
# Accès via http://localhost:8080

# Redirection vers un service
kubectl port-forward service/nginx 8080:80

# Redirection vers un déploiement
kubectl port-forward deployment/nginx 8080:80

# Plusieurs ports
kubectl port-forward pod/app 8080:80 9090:9090

# Écouter sur toutes les interfaces (pas seulement localhost)
kubectl port-forward --address 0.0.0.0 pod/nginx 8080:80

# Écouter sur IP spécifique
kubectl port-forward --address 192.168.1.100 pod/nginx 8080:80

# Dans un namespace
kubectl port-forward -n production pod/nginx 8080:80
```

**Cas d'usage :**

```bash
# Accéder à une base de données
kubectl port-forward service/postgres 5432:5432
psql -h localhost -p 5432 -U user database

# Accéder au dashboard
kubectl port-forward -n kube-system service/kubernetes-dashboard 8443:443
# https://localhost:8443

# Débugger une API
kubectl port-forward service/api 8080:8080
curl http://localhost:8080/health

# Accéder à Prometheus
kubectl port-forward -n monitoring service/prometheus 9090:9090
# http://localhost:9090
```

### cp - Copier des Fichiers

Copie des fichiers entre votre machine locale et un pod.

**Syntaxe :**
```bash
kubectl cp [source] [destination]
```

**Exemples :**

```bash
# Copier un fichier local vers un pod
kubectl cp ./config.json nginx:/etc/app/config.json

# Copier un dossier vers un pod
kubectl cp ./configs nginx:/etc/app/

# Copier depuis un pod vers local
kubectl cp nginx:/var/log/app.log ./app.log

# Copier un dossier depuis un pod
kubectl cp nginx:/var/log/app ./logs/

# Spécifier le conteneur
kubectl cp ./file.txt nginx:/app/file.txt -c nginx-container

# Avec namespace
kubectl cp ./config.json production/nginx:/etc/app/config.json

# Préserver les permissions
kubectl cp ./script.sh nginx:/app/script.sh --no-preserve=false
```

**⚠️ Notes importantes :**
- Le pod doit avoir `tar` installé (présent par défaut dans la plupart des images)
- Les fichiers sont copiés sans préservation des permissions par défaut
- Pas de progression affichée pour les gros fichiers

**Cas d'usage :**

```bash
# Déployer une configuration rapidement
kubectl cp ./new-config.yaml app:/etc/app/config.yaml
kubectl exec app -- kill -HUP 1  # Recharger la config

# Récupérer des logs ou dumps
kubectl cp app:/tmp/debug.log ./debug-$(date +%Y%m%d).log

# Déployer un hotfix
kubectl cp ./patched-file.js app:/app/lib/file.js
kubectl exec app -- pm2 reload all
```

### attach - Se Connecter à un Processus

Attache à un processus en cours dans un conteneur (similaire à docker attach).

**Syntaxe :**
```bash
kubectl attach [pod] [options]
```

**Exemples :**

```bash
# Attacher au processus principal
kubectl attach nginx

# Mode interactif
kubectl attach -it nginx

# Attacher à un conteneur spécifique
kubectl attach nginx -c nginx-container

# Obtenir stdin du conteneur
kubectl attach -i nginx
```

**Différence exec vs attach :**

| Commande | Description | Usage |
|----------|-------------|-------|
| `exec` | Lance une nouvelle commande | Shell interactif, commandes ponctuelles |
| `attach` | Se connecte au processus principal | Voir la sortie du processus, envoyer des signaux |

---

## Gestion des Deployments

### scale - Modifier le Nombre de Replicas

Change le nombre de réplicas d'un déploiement, ReplicaSet ou StatefulSet.

**Syntaxe :**
```bash
kubectl scale [type]/[nom] --replicas=[nombre]
```

**Exemples :**

```bash
# Scaler un déploiement à 3 replicas
kubectl scale deployment/nginx --replicas=3

# Scaler à 0 (arrêter toutes les instances)
kubectl scale deployment/nginx --replicas=0

# Scaler un ReplicaSet
kubectl scale replicaset/nginx-rs --replicas=5

# Scaler un StatefulSet
kubectl scale statefulset/mongodb --replicas=3

# Scaler plusieurs déploiements
kubectl scale deployment nginx redis --replicas=2

# Scaler conditionnellement (seulement si actuellement à N replicas)
kubectl scale deployment/nginx --replicas=5 --current-replicas=3

# Dans un namespace
kubectl scale deployment/nginx --replicas=3 -n production
```

**Vérification :**

```bash
# Voir le nombre actuel de replicas
kubectl get deployment nginx

# Voir les pods créés
kubectl get pods -l app=nginx
```

### rollout - Gérer les Déploiements

Gère les mises à jour progressives (rolling updates) des déploiements.

**Commandes rollout :**

```bash
# Voir le statut d'un rollout
kubectl rollout status deployment/nginx

# Voir l'historique des rollouts
kubectl rollout history deployment/nginx

# Voir les détails d'une révision spécifique
kubectl rollout history deployment/nginx --revision=2

# Mettre en pause un rollout
kubectl rollout pause deployment/nginx

# Reprendre un rollout
kubectl rollout resume deployment/nginx

# Annuler le dernier rollout (rollback)
kubectl rollout undo deployment/nginx

# Revenir à une révision spécifique
kubectl rollout undo deployment/nginx --to-revision=2

# Redémarrer un déploiement (force un nouveau rollout)
kubectl rollout restart deployment/nginx
```

**Workflow de mise à jour :**

```bash
# 1. Mettre à jour l'image
kubectl set image deployment/nginx nginx=nginx:1.21

# 2. Surveiller le rollout
kubectl rollout status deployment/nginx

# Résultat :
# Waiting for deployment "nginx" rollout to finish: 1 out of 3 new replicas...
# Waiting for deployment "nginx" rollout to finish: 2 out of 3 new replicas...
# deployment "nginx" successfully rolled out

# 3. Si problème, rollback
kubectl rollout undo deployment/nginx

# 4. Vérifier l'historique
kubectl rollout history deployment/nginx
```

### set - Modifier des Configurations

Met à jour des champs spécifiques d'une ressource.

**Exemples :**

```bash
# Mettre à jour l'image d'un conteneur
kubectl set image deployment/nginx nginx=nginx:1.21

# Mettre à jour plusieurs conteneurs
kubectl set image deployment/app \
  frontend=frontend:v2 \
  backend=backend:v2

# Mettre à jour les ressources (requests/limits)
kubectl set resources deployment/nginx \
  --limits=cpu=200m,memory=512Mi \
  --requests=cpu=100m,memory=256Mi

# Mettre à jour les variables d'environnement
kubectl set env deployment/nginx API_URL=https://api.example.com

# Mettre à jour depuis un ConfigMap
kubectl set env deployment/nginx --from=configmap/app-config

# Mettre à jour depuis un Secret
kubectl set env deployment/nginx --from=secret/db-secret

# Retirer une variable d'environnement
kubectl set env deployment/nginx API_URL-

# Mettre à jour la ServiceAccount
kubectl set serviceaccount deployment/nginx my-service-account

# Mettre à jour le selector (⚠️ dangereux)
kubectl set selector service/nginx app=new-nginx
```

---

## Gestion des Services et Réseau

### expose - Exposer une Ressource

Crée un service pour exposer un pod, déploiement ou ReplicaSet.

**Syntaxe :**
```bash
kubectl expose [type]/[nom] [options]
```

**Exemples :**

```bash
# Exposer un déploiement avec ClusterIP (par défaut)
kubectl expose deployment/nginx --port=80

# Exposer avec un nom de service spécifique
kubectl expose deployment/nginx --name=nginx-service --port=80

# Exposer avec NodePort
kubectl expose deployment/nginx --type=NodePort --port=80

# Exposer avec LoadBalancer
kubectl expose deployment/nginx --type=LoadBalancer --port=80

# Exposer sur un port cible différent
kubectl expose deployment/nginx --port=80 --target-port=8080

# Exposer un pod directement
kubectl expose pod/nginx --port=80 --name=nginx-pod-service

# Exposer avec un protocole spécifique
kubectl expose deployment/nginx --port=53 --protocol=UDP --name=dns-service

# Exposer avec un sélecteur personnalisé
kubectl expose deployment/nginx --selector=app=nginx,version=v1 --port=80
```

**Types de services :**

| Type | Description | Cas d'usage |
|------|-------------|-------------|
| **ClusterIP** | IP interne uniquement | Communication interne |
| **NodePort** | Port sur chaque nœud | Accès externe simple |
| **LoadBalancer** | Load balancer externe | Production avec cloud provider |
| **ExternalName** | Alias CNAME | Référence à un service externe |

### proxy - Proxy vers l'API Kubernetes

Crée un proxy local vers le serveur API Kubernetes.

**Syntaxe :**
```bash
kubectl proxy [options]
```

**Exemples :**

```bash
# Démarrer le proxy (port 8001 par défaut)
kubectl proxy

# Accès à l'API : http://localhost:8001

# Spécifier un port
kubectl proxy --port=8080

# Accepter les connexions de toutes les interfaces
kubectl proxy --address='0.0.0.0' --accept-hosts='.*'

# Avec un préfixe d'API spécifique
kubectl proxy --api-prefix=/k8s/
```

**Utilisation :**

```bash
# Démarrer le proxy
kubectl proxy &

# Accéder à l'API Kubernetes
curl http://localhost:8001/api/v1/namespaces/default/pods

# Accéder au dashboard (si installé)
# http://localhost:8001/api/v1/namespaces/kube-system/services/https:kubernetes-dashboard:/proxy/

# Accéder aux métriques
curl http://localhost:8001/apis/metrics.k8s.io/v1beta1/nodes
```

---

## Gestion des Configurations

### ConfigMaps et Secrets

**Créer un ConfigMap :**

```bash
# Depuis des valeurs littérales
kubectl create configmap app-config \
  --from-literal=database_url=postgres://db:5432 \
  --from-literal=log_level=debug

# Depuis un fichier
kubectl create configmap app-config --from-file=config.json

# Depuis un dossier (tous les fichiers)
kubectl create configmap app-config --from-file=./config/

# Depuis un fichier .env
kubectl create configmap app-config --from-env-file=.env

# Voir un ConfigMap
kubectl get configmap app-config -o yaml
```

**Créer un Secret :**

```bash
# Secret générique depuis littéraux
kubectl create secret generic db-secret \
  --from-literal=username=admin \
  --from-literal=password=secret123

# Secret TLS depuis certificats
kubectl create secret tls tls-secret \
  --cert=path/to/cert.crt \
  --key=path/to/cert.key

# Secret Docker registry
kubectl create secret docker-registry regcred \
  --docker-server=registry.example.com \
  --docker-username=user \
  --docker-password=password \
  --docker-email=user@example.com

# Voir un Secret (encodé en base64)
kubectl get secret db-secret -o yaml

# Décoder un Secret
kubectl get secret db-secret -o jsonpath='{.data.password}' | base64 -d
```

**Utiliser dans un Pod :**

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: app
spec:
  containers:
  - name: app
    image: myapp
    # Variables d'environnement depuis ConfigMap
    envFrom:
    - configMapRef:
        name: app-config
    # Variables depuis Secret
    env:
    - name: DB_PASSWORD
      valueFrom:
        secretKeyRef:
          name: db-secret
          key: password
    # Monter ConfigMap comme volume
    volumeMounts:
    - name: config
      mountPath: /etc/config
  volumes:
  - name: config
    configMap:
      name: app-config
```

---

## Filtrage et Sélection

### Par Label

Les labels permettent de filtrer et organiser les ressources.

**Exemples :**

```bash
# Lister avec les labels affichés
kubectl get pods --show-labels

# Filtrer par label exact
kubectl get pods -l app=nginx

# Filtrer par label avec opérateur
kubectl get pods -l 'environment in (production,staging)'

# Multiple labels (AND)
kubectl get pods -l app=nginx,environment=production

# Label existe (peu importe la valeur)
kubectl get pods -l environment

# Label n'existe pas
kubectl get pods -l '!environment'

# Label différent de valeur
kubectl get pods -l app!=nginx

# Opérateurs avancés
kubectl get pods -l 'version notin (v1,v2)'
```

**Ajouter/Modifier des labels :**

```bash
# Ajouter un label
kubectl label pod nginx environment=production

# Modifier un label existant
kubectl label pod nginx environment=staging --overwrite

# Supprimer un label
kubectl label pod nginx environment-

# Labelliser plusieurs ressources
kubectl label pods --all environment=production
```

### Par Champ (Field Selector)

Filtrer par champs de l'objet Kubernetes.

**Exemples :**

```bash
# Par status
kubectl get pods --field-selector=status.phase=Running
kubectl get pods --field-selector=status.phase=Failed

# Par nom
kubectl get pods --field-selector=metadata.name=nginx

# Par namespace
kubectl get pods --field-selector=metadata.namespace=production

# Par nœud
kubectl get pods --field-selector=spec.nodeName=node-1

# Combiner plusieurs conditions (AND)
kubectl get pods --field-selector=status.phase=Running,spec.nodeName=node-1

# Exclure (NOT supporté via !=)
kubectl get pods --field-selector=status.phase!=Running
```

### Tri

Trier les résultats par champ.

**Exemples :**

```bash
# Trier par date de création
kubectl get pods --sort-by=.metadata.creationTimestamp

# Trier par nom
kubectl get pods --sort-by=.metadata.name

# Trier par restart count
kubectl get pods --sort-by=.status.containerStatuses[0].restartCount

# Trier par utilisation de CPU (nécessite metrics-server)
kubectl top pods --sort-by=cpu

# Trier par utilisation mémoire
kubectl top pods --sort-by=memory
```

---

## Formats de Sortie Avancés

### JSONPath

Extraire des champs spécifiques au format JSON.

**Exemples :**

```bash
# Extraire juste les noms
kubectl get pods -o jsonpath='{.items[*].metadata.name}'

# Extraire noms et IPs
kubectl get pods -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.status.podIP}{"\n"}{end}'

# Extraire les images utilisées
kubectl get pods -o jsonpath='{.items[*].spec.containers[*].image}'

# Format tableau
kubectl get pods -o jsonpath='{.items[*].metadata.name}' | tr ' ' '\n'

# Extraire une valeur spécifique
kubectl get pod nginx -o jsonpath='{.status.phase}'

# Avec conditions
kubectl get pods -o jsonpath='{.items[?(@.status.phase=="Running")].metadata.name}'
```

### Custom Columns

Créer des colonnes personnalisées.

**Exemples :**

```bash
# Colonnes personnalisées simples
kubectl get pods -o custom-columns=NAME:.metadata.name,STATUS:.status.phase

# Plusieurs colonnes
kubectl get pods -o custom-columns=\
NAME:.metadata.name,\
STATUS:.status.phase,\
IP:.status.podIP,\
NODE:.spec.nodeName

# Avec headers personnalisés
kubectl get pods -o custom-columns=\
POD:.metadata.name,\
STATE:.status.phase,\
RESTARTS:.status.containerStatuses[0].restartCount

# Sans headers
kubectl get pods --no-headers -o custom-columns=NAME:.metadata.name,STATUS:.status.phase
```

### Go Template

Utiliser les templates Go pour un formatage avancé.

**Exemples :**

```bash
# Template simple
kubectl get pods -o go-template='{{range .items}}{{.metadata.name}}{{"\n"}}{{end}}'

# Avec conditions
kubectl get pods -o go-template='{{range .items}}{{if eq .status.phase "Running"}}{{.metadata.name}}{{"\n"}}{{end}}{{end}}'

# Formatage complexe
kubectl get pods -o go-template='{{range .items}}Pod: {{.metadata.name}} - Status: {{.status.phase}}{{"\n"}}{{end}}'
```

---

## Commandes de Diagnostic

### Événements

Les événements fournissent des informations sur ce qui se passe dans le cluster.

**Exemples :**

```bash
# Tous les événements
kubectl get events

# Événements dans tous les namespaces
kubectl get events -A

# Événements triés par date
kubectl get events --sort-by='.lastTimestamp'

# Événements récents (dernière heure)
kubectl get events --sort-by='.lastTimestamp' | tail -20

# Événements d'un namespace
kubectl get events -n production

# Événements d'une ressource spécifique
kubectl get events --field-selector involvedObject.name=nginx

# Événements de type Warning
kubectl get events --field-selector type=Warning

# Surveiller les événements en temps réel
kubectl get events --watch
```

### Ressources (top)

Voir l'utilisation CPU et mémoire (nécessite metrics-server).

**Exemples :**

```bash
# Utilisation des nœuds
kubectl top nodes

# Résultat :
# NAME       CPU(cores)   CPU%   MEMORY(bytes)   MEMORY%
# node-1     523m         26%    2048Mi          51%

# Utilisation des pods
kubectl top pods

# Dans tous les namespaces
kubectl top pods -A

# Trier par CPU
kubectl top pods --sort-by=cpu

# Trier par mémoire
kubectl top pods --sort-by=memory

# Top 10 des pods par mémoire
kubectl top pods -A --sort-by=memory | head -11

# Utilisation d'un pod spécifique
kubectl top pod nginx

# Avec conteneurs détaillés
kubectl top pod nginx --containers
```

### Cluster Info

Informations sur le cluster.

**Exemples :**

```bash
# Informations de base
kubectl cluster-info

# Résultat :
# Kubernetes control plane is running at https://127.0.0.1:16443
# CoreDNS is running at https://127.0.0.1:16443/api/v1/namespaces/kube-system/services/kube-dns:dns/proxy

# Informations détaillées (dump complet)
kubectl cluster-info dump

# Sauvegarder le dump
kubectl cluster-info dump > cluster-dump.txt

# Dump d'un namespace spécifique
kubectl cluster-info dump --namespaces=production

# Version du serveur
kubectl version

# Version courte
kubectl version --short
```

### Debugging de Pods

**Créer un pod de debug :**

```bash
# Pod temporaire pour debug
kubectl run debug --image=nicolaka/netshoot --rm -it --restart=Never

# Pod busybox pour tests rapides
kubectl run busybox --image=busybox:1.28 --rm -it --restart=Never -- sh

# Pod curl pour tests API
kubectl run curl --image=curlimages/curl --rm -it --restart=Never -- sh

# Pod avec outils réseau complets
kubectl run netshoot --image=nicolaka/netshoot --rm -it --restart=Never -- bash
```

**Debug d'un pod existant :**

```bash
# Créer une copie pour debug (sans modifier l'original)
kubectl debug pod/nginx -it --image=ubuntu --share-processes --copy-to=nginx-debug

# Debug avec image différente
kubectl debug pod/nginx -it --image=busybox:1.28 --copy-to=nginx-debug

# Debug en tant que root
kubectl debug pod/nginx -it --image=busybox --target=nginx --profile=general

# Voir tous les processus du pod
kubectl debug pod/nginx -it --image=busybox --share-processes --copy-to=nginx-debug -- sh
```

---

## Contexts et Namespaces

### Gestion des Contexts

Les contexts permettent de gérer plusieurs clusters.

**Exemples :**

```bash
# Voir tous les contexts
kubectl config get-contexts

# Voir le context actuel
kubectl config current-context

# Changer de context
kubectl config use-context microk8s

# Renommer un context
kubectl config rename-context old-name new-name

# Supprimer un context
kubectl config delete-context context-name

# Définir le namespace par défaut pour un context
kubectl config set-context --current --namespace=production
```

### Gestion des Namespaces

```bash
# Lister tous les namespaces
kubectl get namespaces
kubectl get ns

# Créer un namespace
kubectl create namespace production

# Créer depuis YAML
kubectl apply -f - <<EOF
apiVersion: v1
kind: Namespace
metadata:
  name: production
  labels:
    environment: production
EOF

# Supprimer un namespace (⚠️ supprime tout dedans)
kubectl delete namespace staging

# Voir les ressources dans tous les namespaces
kubectl get all -A

# Changer le namespace par défaut
kubectl config set-context --current --namespace=production

# Vérifier le namespace actuel
kubectl config view --minify --output 'jsonpath={..namespace}'

# Labelliser un namespace
kubectl label namespace production environment=prod
```

---

## Commandes Utilitaires

### explain - Documentation

Affiche la documentation d'une ressource Kubernetes.

**Exemples :**

```bash
# Documentation d'une ressource
kubectl explain pod

# Documentation d'un champ spécifique
kubectl explain pod.spec

# Documentation d'un sous-champ
kubectl explain pod.spec.containers

# Documentation complète (récursif)
kubectl explain pod --recursive

# Documentation d'un DeploymentSpec
kubectl explain deployment.spec

# Format de sortie
kubectl explain pod --output=plaintext-openapiv2
```

### api-resources - Liste des Ressources

Liste toutes les ressources disponibles dans le cluster.

**Exemples :**

```bash
# Toutes les ressources
kubectl api-resources

# Résultat :
# NAME                    SHORTNAMES   APIVERSION        NAMESPACED   KIND
# pods                    po           v1                true         Pod
# services                svc          v1                true         Service
# deployments             deploy       apps/v1           true         Deployment

# Ressources avec verbes (actions possibles)
kubectl api-resources --verbs=list,get

# Ressources namespacées uniquement
kubectl api-resources --namespaced=true

# Ressources globales (non-namespacées)
kubectl api-resources --namespaced=false

# Filtrer par API group
kubectl api-resources --api-group=apps

# Format large
kubectl api-resources -o wide
```

### api-versions - Versions d'API

Liste les versions d'API disponibles.

**Exemples :**

```bash
# Toutes les versions
kubectl api-versions

# Résultat :
# admissionregistration.k8s.io/v1
# apiextensions.k8s.io/v1
# apps/v1
# authentication.k8s.io/v1
# ...
```

### diff - Voir les Différences

Compare la configuration actuelle avec un fichier.

**Exemples :**

```bash
# Voir les différences avant apply
kubectl diff -f deployment.yaml

# Différences sur un dossier
kubectl diff -f ./manifests/

# Voir ce qui changerait
kubectl diff -f updated-config.yaml

# Format compact
kubectl diff -f deployment.yaml --compact
```

### wait - Attendre une Condition

Attend qu'une ressource atteigne une condition spécifique.

**Exemples :**

```bash
# Attendre qu'un pod soit prêt
kubectl wait --for=condition=ready pod/nginx

# Attendre avec timeout
kubectl wait --for=condition=ready pod/nginx --timeout=60s

# Attendre la suppression
kubectl wait --for=delete pod/nginx --timeout=30s

# Attendre plusieurs pods
kubectl wait --for=condition=ready pod -l app=nginx

# Attendre qu'un déploiement soit disponible
kubectl wait --for=condition=available deployment/nginx --timeout=120s
```

---

## Raccourcis et Astuces

### Raccourcis de Ressources

| Ressource Complète | Raccourci | Exemple |
|-------------------|-----------|---------|
| pods | po | `kubectl get po` |
| services | svc | `kubectl get svc` |
| deployments | deploy | `kubectl get deploy` |
| replicasets | rs | `kubectl get rs` |
| namespaces | ns | `kubectl get ns` |
| nodes | no | `kubectl get no` |
| persistentvolumes | pv | `kubectl get pv` |
| persistentvolumeclaims | pvc | `kubectl get pvc` |
| configmaps | cm | `kubectl get cm` |
| serviceaccounts | sa | `kubectl get sa` |
| ingresses | ing | `kubectl get ing` |
| networkpolicies | netpol | `kubectl get netpol` |

### Options Communes

| Option | Raccourci | Description |
|--------|-----------|-------------|
| `--namespace` | `-n` | Spécifier le namespace |
| `--all-namespaces` | `-A` | Tous les namespaces |
| `--selector` | `-l` | Filtrer par label |
| `--output` | `-o` | Format de sortie |
| `--watch` | `-w` | Surveiller les changements |
| `--recursive` | `-R` | Récursif (pour dossiers) |
| `--force` | - | Forcer l'action |
| `--dry-run` | - | Simuler sans appliquer |

### Commandes Composées Utiles

```bash
# Redémarrer tous les pods d'un déploiement
kubectl rollout restart deployment/nginx

# Supprimer tous les pods Failed
kubectl delete pods --field-selector=status.phase==Failed -A

# Copier un secret d'un namespace à un autre
kubectl get secret my-secret -n source-ns -o yaml | \
  sed 's/namespace: source-ns/namespace: target-ns/' | \
  kubectl apply -f -

# Lister toutes les images utilisées
kubectl get pods -A -o jsonpath='{.items[*].spec.containers[*].image}' | \
  tr ' ' '\n' | sort -u

# Trouver les pods qui redémarrent beaucoup
kubectl get pods -A --sort-by='.status.containerStatuses[0].restartCount'

# Exporter toutes les ressources d'un namespace
kubectl get all -n production -o yaml > production-backup.yaml

# Vérifier quels pods utilisent le plus de mémoire
kubectl top pods -A --sort-by=memory | head -10

# Lister les pods sur chaque nœud
kubectl get pods -A -o wide --sort-by=.spec.nodeName
```

---

## Exemples de Workflows

### Workflow 1 : Déploiement Complet d'une Application

```bash
# 1. Créer un namespace
kubectl create namespace myapp

# 2. Créer un ConfigMap
kubectl create configmap app-config \
  --from-literal=API_URL=https://api.example.com \
  -n myapp

# 3. Créer un Secret
kubectl create secret generic app-secret \
  --from-literal=API_KEY=secret123 \
  -n myapp

# 4. Appliquer le déploiement
kubectl apply -f - <<EOF
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
  namespace: myapp
spec:
  replicas: 3
  selector:
    matchLabels:
      app: myapp
  template:
    metadata:
      labels:
        app: myapp
    spec:
      containers:
      - name: myapp
        image: myapp:v1.0
        envFrom:
        - configMapRef:
            name: app-config
        - secretRef:
            name: app-secret
EOF

# 5. Exposer le service
kubectl expose deployment myapp --port=80 --type=LoadBalancer -n myapp

# 6. Vérifier le déploiement
kubectl get pods -n myapp
kubectl get svc -n myapp

# 7. Voir les logs
kubectl logs -f deployment/myapp -n myapp
```

### Workflow 2 : Mise à Jour avec Rollback

```bash
# 1. Voir la version actuelle
kubectl get deployment myapp -n myapp

# 2. Mettre à jour l'image
kubectl set image deployment/myapp myapp=myapp:v2.0 -n myapp

# 3. Surveiller le rollout
kubectl rollout status deployment/myapp -n myapp

# 4. Si problème détecté, rollback
kubectl rollout undo deployment/myapp -n myapp

# 5. Vérifier le statut
kubectl rollout status deployment/myapp -n myapp

# 6. Voir l'historique
kubectl rollout history deployment/myapp -n myapp
```

### Workflow 3 : Debugging d'une Application

```bash
# 1. Identifier le pod problématique
kubectl get pods -n myapp

# 2. Voir les détails
kubectl describe pod problematic-pod -n myapp

# 3. Vérifier les logs
kubectl logs problematic-pod -n myapp

# 4. Logs du conteneur précédent (si crash)
kubectl logs problematic-pod -n myapp --previous

# 5. Se connecter au pod
kubectl exec -it problematic-pod -n myapp -- sh

# 6. Vérifier les événements
kubectl get events -n myapp --sort-by='.lastTimestamp' | tail -20

# 7. Vérifier les ressources
kubectl top pod problematic-pod -n myapp

# 8. Port-forward pour tester localement
kubectl port-forward problematic-pod 8080:8080 -n myapp
```

### Workflow 4 : Nettoyage et Maintenance

```bash
# 1. Lister les ressources
kubectl get all -n myapp

# 2. Scaler à 0 (arrêt temporaire)
kubectl scale deployment/myapp --replicas=0 -n myapp

# 3. Supprimer le déploiement
kubectl delete deployment myapp -n myapp

# 4. Supprimer le service
kubectl delete service myapp -n myapp

# 5. Supprimer ConfigMap et Secret
kubectl delete configmap app-config -n myapp
kubectl delete secret app-secret -n myapp

# 6. Supprimer le namespace (tout d'un coup)
kubectl delete namespace myapp

# 7. Nettoyer les pods Failed globalement
kubectl delete pods --field-selector=status.phase==Failed -A
```

---

## Bonnes Pratiques

### 1. Toujours Spécifier le Namespace

```bash
# ✗ Mauvais (utilise 'default')
kubectl get pods

# ✓ Bon
kubectl get pods -n production
```

### 2. Utiliser des Labels Cohérents

```bash
# ✓ Bon système de labels
kubectl label pod myapp \
  app=myapp \
  tier=backend \
  environment=production \
  version=v1.0
```

### 3. Dry-run Avant Application

```bash
# Toujours tester d'abord
kubectl apply -f config.yaml --dry-run=client
kubectl diff -f config.yaml
kubectl apply -f config.yaml
```

### 4. Utiliser YAML pour la Production

```bash
# ✗ Éviter en production
kubectl create deployment nginx --image=nginx

# ✓ Préférer
kubectl apply -f deployment.yaml
```

### 5. Versionner les Configurations

```bash
# Stocker dans Git
git add kubernetes/
git commit -m "Update deployment config"
git push
```

### 6. Utiliser des Alias

```bash
# Dans ~/.bashrc ou ~/.zshrc
alias k='kubectl'
alias kgp='kubectl get pods'
alias kgd='kubectl get deployments'
alias kgs='kubectl get services'
alias kd='kubectl describe'
alias kl='kubectl logs -f'
alias ke='kubectl exec -it'
```

### 7. Activer la Complétion

```bash
# Bash
source <(kubectl completion bash)
echo "source <(kubectl completion bash)" >> ~/.bashrc

# Zsh
source <(kubectl completion zsh)
echo "source <(kubectl completion zsh)" >> ~/.zshrc
```

---

## Cheat Sheet - Référence Rapide

### Gestion de Base

```bash
kubectl get [resource]              # Lister
kubectl describe [resource] [name]  # Détails
kubectl create [resource]           # Créer
kubectl apply -f [file]             # Appliquer
kubectl delete [resource] [name]    # Supprimer
kubectl edit [resource] [name]      # Éditer
```

### Pods

```bash
kubectl get pods                    # Lister
kubectl get pods -o wide            # Avec IPs
kubectl logs [pod] -f               # Logs temps réel
kubectl exec -it [pod] -- bash      # Shell interactif
kubectl port-forward [pod] 8080:80  # Port forward
kubectl cp [pod]:[path] [local]     # Copier fichier
```

### Deployments

```bash
kubectl get deployments             # Lister
kubectl scale deploy/[name] --replicas=3
kubectl set image deploy/[name] [container]=[image]
kubectl rollout status deploy/[name]
kubectl rollout undo deploy/[name]
kubectl rollout restart deploy/[name]
```

### Services

```bash
kubectl get services                # Lister
kubectl expose deploy/[name] --port=80
kubectl describe service [name]
```

### Debug

```bash
kubectl describe [resource] [name]
kubectl logs [pod] --previous
kubectl get events --sort-by='.lastTimestamp'
kubectl top nodes
kubectl top pods
```

### Namespaces

```bash
kubectl get namespaces
kubectl create namespace [name]
kubectl config set-context --current --namespace=[name]
kubectl get all -n [namespace]
```

---

## Résolution de Problèmes Courants

### Problème : "Error from server (NotFound)"

```bash
# Vérifier le nom de la ressource
kubectl get pods

# Vérifier le namespace
kubectl get pods -A | grep [nom]
```

### Problème : "Forbidden: User cannot..."

```bash
# Vérifier les permissions
kubectl auth can-i [verb] [resource]

# Exemple
kubectl auth can-i create deployments
```

### Problème : Pod en CrashLoopBackOff

```bash
# Voir les logs
kubectl logs [pod]
kubectl logs [pod] --previous

# Voir les événements
kubectl describe pod [pod]

# Vérifier les ressources
kubectl top pod [pod]
```

### Problème : Service ne répond pas

```bash
# Vérifier les endpoints
kubectl get endpoints [service]

# Vérifier les labels
kubectl get pods --show-labels
kubectl describe service [service]

# Tester depuis un pod
kubectl run test --image=busybox -it --rm -- wget -O- http://[service]
```

---

## Ressources Complémentaires

### Documentation Officielle

- **Kubectl Reference** : https://kubernetes.io/docs/reference/kubectl/
- **Kubectl Cheat Sheet** : https://kubernetes.io/docs/reference/kubectl/cheatsheet/
- **Kubectl Book** : https://kubectl.docs.kubernetes.io/

### Outils Complémentaires

- **k9s** : Terminal UI interactif pour Kubernetes
- **kubectx/kubens** : Changement rapide de context/namespace
- **stern** : Tail multi-pods amélioré
- **kube-ps1** : Afficher le context dans le prompt
- **Lens** : IDE Kubernetes avec interface graphique

### Plugins kubectl

```bash
# Installer krew (gestionnaire de plugins)
kubectl krew install [plugin]

# Plugins utiles
kubectl krew install ctx      # Changement de context
kubectl krew install ns       # Changement de namespace
kubectl krew install tree     # Vue arborescente des ressources
kubectl krew install neat     # Nettoyage des YAMLs
```

---

## Conclusion

Vous disposez maintenant d'une référence complète des commandes kubectl essentielles. Avec ces commandes, vous pouvez :

✅ **Gérer** vos applications Kubernetes
✅ **Déployer** de nouvelles versions
✅ **Debugger** les problèmes
✅ **Surveiller** les performances
✅ **Configurer** les ressources

**Top 10 des commandes à retenir :**

1. `kubectl get pods` - Voir les pods
2. `kubectl describe pod [nom]` - Débugger
3. `kubectl logs [pod] -f` - Suivre les logs
4. `kubectl apply -f [file]` - Déployer
5. `kubectl exec -it [pod] -- bash` - Shell interactif
6. `kubectl get svc` - Voir les services
7. `kubectl scale deploy/[nom] --replicas=N` - Scaler
8. `kubectl rollout undo deploy/[nom]` - Rollback
9. `kubectl port-forward [pod] 8080:80` - Test local
10. `kubectl get events --sort-by='.lastTimestamp'` - Événements

**Pro Tip** : Créez des alias et activez la complétion automatique pour une productivité maximale !

```bash
alias k='kubectl'
source <(kubectl completion bash)
complete -F __start_kubectl k
```

Avec ces commandes maîtrisées, vous êtes prêt à gérer efficacement n'importe quel cluster Kubernetes ! 🚀

⏭️ [Cheat sheet YAML](/annexes/annexe-d-reference-rapide/cheat-sheet-yaml.md)
