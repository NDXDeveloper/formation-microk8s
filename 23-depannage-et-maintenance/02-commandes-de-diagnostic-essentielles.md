🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 23.2 Commandes de Diagnostic Essentielles

## Introduction

Dans cette section, nous allons explorer en détail les commandes qui vous permettront de diagnostiquer efficacement les problèmes dans votre cluster MicroK8s. Chaque commande sera expliquée avec des exemples concrets et des astuces pour interpréter les résultats.

> **Pour les débutants** : Ne vous sentez pas obligés de mémoriser toutes ces commandes. Gardez cette section comme référence et revenez-y au besoin. Avec la pratique, les commandes les plus utiles deviendront naturelles.

## Anatomie d'une Commande kubectl

Avant de plonger dans les commandes spécifiques, comprenons la structure générale :

```bash
microk8s kubectl <verbe> <ressource> <nom> [options]
```

- **verbe** : l'action à effectuer (get, describe, logs, etc.)
- **ressource** : le type de ressource (pod, service, deployment, etc.)
- **nom** : le nom spécifique de la ressource (optionnel pour certaines commandes)
- **options** : drapeaux pour affiner la commande (-n pour namespace, -o pour format de sortie, etc.)

**Astuce** : Si vous avez configuré l'alias `kubectl` pour `microk8s kubectl`, vous pouvez simplement taper `kubectl` au lieu de `microk8s kubectl`. Pour le reste de cette section, nous utiliserons la forme complète `microk8s kubectl`.

## Catégorie 1 : Commandes d'État Général

### 1.1 microk8s status - État du Cluster

**Description** : Vérifier si MicroK8s fonctionne correctement.

**Commande** :
```bash
microk8s status
```

**Sortie typique** :
```
microk8s is running
high-availability: no
  datastore master nodes: 127.0.0.1:19001
  datastore standby nodes: none
addons:
  enabled:
    dns                  # (core) CoreDNS
    ha-cluster           # (core) Configure high availability on the current node
    helm                 # (core) Helm - the package manager for Kubernetes
    helm3                # (core) Helm 3 - the package manager for Kubernetes
  disabled:
    cert-manager         # (core) Cloud native certificate management
    dashboard            # (core) The Kubernetes dashboard
    ...
```

**Interprétation** :
- **"microk8s is running"** : Tout va bien, le cluster est opérationnel
- **"microk8s is not running"** : Le service est arrêté, vous devez le démarrer avec `microk8s start`
- **Section addons** : Liste les addons activés et désactivés

**Cas d'utilisation** : Première vérification à faire en cas de problème.

### 1.2 kubectl cluster-info - Informations du Cluster

**Description** : Obtenir les informations de base sur le cluster.

**Commande** :
```bash
microk8s kubectl cluster-info
```

**Sortie typique** :
```
Kubernetes control plane is running at https://127.0.0.1:16443
CoreDNS is running at https://127.0.0.1:16443/api/v1/namespaces/kube-system/services/kube-dns:dns/proxy

To further debug and diagnose cluster problems, use 'kubectl cluster-info dump'.
```

**Interprétation** :
- Confirme que l'API server est accessible
- Montre l'URL du control plane
- Liste les services système en cours d'exécution

**Cas d'utilisation** : Vérifier que vous pouvez communiquer avec le cluster.

### 1.3 kubectl get nodes - État des Nœuds

**Description** : Voir l'état de tous les nœuds du cluster.

**Commande** :
```bash
microk8s kubectl get nodes
```

**Sortie typique** :
```
NAME        STATUS   ROLES    AGE   VERSION
ubuntu-pc   Ready    <none>   5d    v1.28.3
```

**Options utiles** :
```bash
# Avec plus de détails
microk8s kubectl get nodes -o wide

# Voir les labels des nœuds
microk8s kubectl get nodes --show-labels
```

**Interprétation des statuts** :
- **Ready** : Le nœud fonctionne correctement
- **NotReady** : Le nœud a un problème (souvent réseau ou kubelet)
- **SchedulingDisabled** : Le nœud est cordonné (ne reçoit pas de nouveaux pods)

**Cas d'utilisation** : Vérifier que tous les nœuds sont disponibles.

### 1.4 kubectl version - Versions Kubernetes

**Description** : Afficher les versions du client et du serveur.

**Commande** :
```bash
microk8s kubectl version --short
```

**Sortie typique** :
```
Client Version: v1.28.3
Kustomize Version: v5.0.1
Server Version: v1.28.3
```

**Cas d'utilisation** : Vérifier la version de Kubernetes, utile pour compatibilité.

## Catégorie 2 : Commandes de Listage des Ressources

### 2.1 kubectl get - Lister les Ressources

**Description** : La commande la plus utilisée pour lister n'importe quelle ressource.

#### Syntaxe de base :
```bash
microk8s kubectl get <type-ressource>
```

#### Exemples courants :

**Lister les pods** :
```bash
# Dans le namespace par défaut
microk8s kubectl get pods

# Dans un namespace spécifique
microk8s kubectl get pods -n kube-system

# Dans tous les namespaces
microk8s kubectl get pods --all-namespaces
# ou la version courte
microk8s kubectl get pods -A
```

**Lister les services** :
```bash
microk8s kubectl get services
# ou la version courte
microk8s kubectl get svc
```

**Lister les deployments** :
```bash
microk8s kubectl get deployments
# ou
microk8s kubectl get deploy
```

**Lister les namespaces** :
```bash
microk8s kubectl get namespaces
# ou
microk8s kubectl get ns
```

#### Options puissantes :

**Format wide (plus de colonnes)** :
```bash
microk8s kubectl get pods -o wide
```
Affiche des informations supplémentaires comme l'IP du pod, le nœud, etc.

**Format YAML complet** :
```bash
microk8s kubectl get pod mon-pod -o yaml
```
Affiche la définition complète de la ressource, utile pour déboguer ou exporter.

**Format JSON** :
```bash
microk8s kubectl get pod mon-pod -o json
```

**Format personnalisé avec jsonpath** :
```bash
# Obtenir uniquement l'IP d'un pod
microk8s kubectl get pod mon-pod -o jsonpath='{.status.podIP}'

# Lister tous les pods avec leur nœud
microk8s kubectl get pods -o custom-columns=NAME:.metadata.name,NODE:.spec.nodeName
```

**Surveillance en temps réel** :
```bash
microk8s kubectl get pods --watch
# ou
microk8s kubectl get pods -w
```
Affiche les changements en temps réel. Très utile pendant un déploiement.

**Filtrer par label** :
```bash
microk8s kubectl get pods -l app=nginx
microk8s kubectl get pods -l environment=production,tier=frontend
```

#### Raccourcis pratiques :

Kubernetes accepte des noms courts pour les ressources courantes :
- `po` = pods
- `svc` = services
- `deploy` = deployments
- `rs` = replicasets
- `ns` = namespaces
- `cm` = configmaps
- `pv` = persistentvolumes
- `pvc` = persistentvolumeclaims

**Exemple** :
```bash
microk8s kubectl get po,svc,deploy
```
Liste les pods, services et deployments en une seule commande.

### 2.2 kubectl get all - Vue d'Ensemble

**Description** : Lister plusieurs types de ressources courantes simultanément.

**Commande** :
```bash
microk8s kubectl get all -n mon-namespace
```

**Ce qui est inclus** :
- Pods
- Services
- Deployments
- ReplicaSets
- StatefulSets
- DaemonSets
- Jobs

**Attention** : Malgré son nom, `get all` ne liste PAS vraiment TOUTES les ressources (pas les ConfigMaps, Secrets, PV, PVC, etc.).

**Cas d'utilisation** : Vue rapide de l'état d'un namespace.

## Catégorie 3 : Commandes de Description Détaillée

### 3.1 kubectl describe - Détails Complets

**Description** : Obtenir une description détaillée d'une ressource, incluant les événements.

**Syntaxe** :
```bash
microk8s kubectl describe <type-ressource> <nom> -n <namespace>
```

#### Exemples :

**Décrire un pod** :
```bash
microk8s kubectl describe pod mon-pod -n default
```

**Décrire un service** :
```bash
microk8s kubectl describe service mon-service
```

**Décrire un nœud** :
```bash
microk8s kubectl describe node ubuntu-pc
```

#### Sections importantes dans la sortie :

**Pour un pod** :
```
Name:             mon-pod
Namespace:        default
Priority:         0
Service Account:  default
Node:             ubuntu-pc/192.168.1.100
Start Time:       Mon, 21 Oct 2025 10:30:00 +0200
Labels:           app=nginx
Status:           Running
IP:               10.1.1.5

Containers:
  nginx:
    Container ID:   containerd://abc123...
    Image:          nginx:1.21
    Port:           80/TCP
    State:          Running
      Started:      Mon, 21 Oct 2025 10:30:05 +0200
    Ready:          True
    Restart Count:  0
    Environment:    <none>
    Mounts:
      /var/run/secrets/kubernetes.io/serviceaccount from kube-api-access-xxxxx (ro)

Conditions:
  Type              Status
  Initialized       True
  Ready             True
  ContainersReady   True
  PodScheduled      True

Events:
  Type    Reason     Age   From               Message
  ----    ------     ----  ----               -------
  Normal  Scheduled  2m    default-scheduler  Successfully assigned default/mon-pod to ubuntu-pc
  Normal  Pulling    2m    kubelet            Pulling image "nginx:1.21"
  Normal  Pulled     2m    kubelet            Successfully pulled image "nginx:1.21"
  Normal  Created    2m    kubelet            Created container nginx
  Normal  Started    2m    kubelet            Started container nginx
```

**Sections clés à examiner** :

1. **Status** : État actuel du pod
2. **Containers/State** : État de chaque conteneur
3. **Ready** : Le pod est-il prêt à recevoir du trafic ?
4. **Restart Count** : Nombre de redémarrages (élevé = problème)
5. **Conditions** : État des conditions de santé
6. **Events** : Historique chronologique (LA section la plus utile !)

**Pour un nœud** :
```bash
microk8s kubectl describe node ubuntu-pc
```

Vous verrez :
- **Conditions** : État du nœud (Ready, DiskPressure, MemoryPressure, etc.)
- **Capacity** : Ressources totales du nœud
- **Allocatable** : Ressources disponibles pour les pods
- **Non-terminated Pods** : Liste des pods sur ce nœud avec leur consommation

**Cas d'utilisation** : Comprendre pourquoi une ressource ne fonctionne pas correctement.

### 3.2 kubectl explain - Documentation des Ressources

**Description** : Obtenir la documentation d'une ressource ou d'un champ spécifique.

**Commande** :
```bash
# Documentation générale d'une ressource
microk8s kubectl explain pod

# Documentation d'un champ spécifique
microk8s kubectl explain pod.spec.containers

# Documentation avec tous les champs imbriqués
microk8s kubectl explain pod.spec.containers --recursive
```

**Sortie typique** :
```
KIND:     Pod
VERSION:  v1

DESCRIPTION:
     Pod is a collection of containers that can run on a host. This resource is
     created by clients and scheduled onto hosts.

FIELDS:
   apiVersion	<string>
     APIVersion defines the versioned schema of this representation of an
     object.
   ...
```

**Cas d'utilisation** : Comprendre comment structurer un fichier YAML sans chercher sur internet.

## Catégorie 4 : Commandes de Logs

### 4.1 kubectl logs - Consulter les Logs

**Description** : Afficher les logs d'un conteneur dans un pod.

#### Syntaxe de base :
```bash
microk8s kubectl logs <nom-pod> -n <namespace>
```

#### Options essentielles :

**Logs en temps réel (follow)** :
```bash
microk8s kubectl logs -f mon-pod
```
Comme `tail -f`, affiche les nouveaux logs au fur et à mesure.

**Logs du conteneur précédent (après crash)** :
```bash
microk8s kubectl logs mon-pod --previous
```
Très utile quand un pod crash : vous pouvez voir les logs juste avant le crash.

**Logs d'un conteneur spécifique** (si plusieurs conteneurs dans le pod) :
```bash
microk8s kubectl logs mon-pod -c nom-conteneur
```

**Limiter le nombre de lignes** :
```bash
# Dernières 100 lignes
microk8s kubectl logs mon-pod --tail=100

# Dernières 50 lignes en temps réel
microk8s kubectl logs mon-pod --tail=50 -f
```

**Logs depuis un certain temps** :
```bash
# Logs des 5 dernières minutes
microk8s kubectl logs mon-pod --since=5m

# Logs des 2 dernières heures
microk8s kubectl logs mon-pod --since=2h
```

**Logs avec timestamps** :
```bash
microk8s kubectl logs mon-pod --timestamps
```

#### Exemples pratiques :

**Déboguer un pod qui crash** :
```bash
# 1. Voir les logs actuels
microk8s kubectl logs mon-pod

# 2. Si le pod a redémarré, voir les logs du crash précédent
microk8s kubectl logs mon-pod --previous

# 3. Surveiller en temps réel après un redémarrage
microk8s kubectl logs mon-pod -f
```

**Chercher une erreur dans les logs** :
```bash
# Avec grep
microk8s kubectl logs mon-pod | grep -i error

# Avec grep et contexte (3 lignes avant/après)
microk8s kubectl logs mon-pod | grep -i error -C 3
```

**Sauvegarder les logs dans un fichier** :
```bash
microk8s kubectl logs mon-pod > logs-mon-pod.txt
```

**Logs de tous les pods d'un deployment** :
```bash
# Nécessite l'utilisation de labels
microk8s kubectl logs -l app=mon-app --all-containers=true
```

**Cas d'utilisation** : Comprendre ce qui se passe à l'intérieur de votre application.

## Catégorie 5 : Commandes d'Exécution

### 5.1 kubectl exec - Exécuter des Commandes dans un Pod

**Description** : Exécuter une commande à l'intérieur d'un conteneur en cours d'exécution.

#### Syntaxe de base :
```bash
microk8s kubectl exec <nom-pod> -- <commande>
```

#### Exemples courants :

**Exécuter une commande simple** :
```bash
# Voir les processus en cours dans le pod
microk8s kubectl exec mon-pod -- ps aux

# Voir le contenu d'un fichier
microk8s kubectl exec mon-pod -- cat /etc/nginx/nginx.conf

# Tester la connectivité réseau
microk8s kubectl exec mon-pod -- ping -c 3 google.com
```

**Mode interactif (shell)** :
```bash
microk8s kubectl exec -it mon-pod -- /bin/bash
```

Options importantes :
- `-i` : mode interactif (stdin ouvert)
- `-t` : alloue un pseudo-TTY (terminal)

Si bash n'est pas disponible, essayez `/bin/sh` :
```bash
microk8s kubectl exec -it mon-pod -- /bin/sh
```

**Spécifier un conteneur** (si plusieurs conteneurs dans le pod) :
```bash
microk8s kubectl exec -it mon-pod -c nom-conteneur -- /bin/bash
```

#### Cas d'usage pratiques :

**Déboguer une application** :
```bash
# Entrer dans le pod
microk8s kubectl exec -it mon-pod -- /bin/bash

# Puis à l'intérieur du pod :
cd /app
ls -la
cat config.json
env | grep DATABASE
curl localhost:8080/health
```

**Tester la connectivité réseau** :
```bash
# Tester la résolution DNS
microk8s kubectl exec mon-pod -- nslookup kubernetes.default

# Tester l'accès à un service
microk8s kubectl exec mon-pod -- curl -v http://mon-service:80

# Tester l'accès à une base de données
microk8s kubectl exec mon-pod -- nc -zv postgres-service 5432
```

**Vérifier les variables d'environnement** :
```bash
microk8s kubectl exec mon-pod -- env
```

**Vérifier les volumes montés** :
```bash
microk8s kubectl exec mon-pod -- ls -la /mnt/data
microk8s kubectl exec mon-pod -- df -h
```

**Attention** : Les modifications que vous faites avec `exec` ne sont PAS persistantes. Si le pod redémarre, elles disparaissent. Pour des changements permanents, modifiez l'image ou la configuration.

### 5.2 kubectl run - Créer un Pod Temporaire de Test

**Description** : Créer rapidement un pod pour tester quelque chose.

**Commande** :
```bash
# Pod temporaire qui se supprime automatiquement
microk8s kubectl run test-pod --image=busybox --rm -it -- sh
```

Options :
- `--rm` : supprime le pod après utilisation
- `-it` : mode interactif avec terminal
- `--` : tout ce qui suit est la commande à exécuter

**Exemples d'utilisation** :

**Tester la résolution DNS** :
```bash
microk8s kubectl run test-dns --image=busybox --rm -it -- nslookup kubernetes.default
```

**Tester la connectivité à un service** :
```bash
microk8s kubectl run test-net --image=nicolaka/netshoot --rm -it -- curl http://mon-service:80
```

**Tester l'accès à une base de données** :
```bash
microk8s kubectl run test-db --image=postgres:15 --rm -it -- psql -h postgres-service -U myuser -d mydb
```

**Cas d'utilisation** : Créer un pod "jetable" pour déboguer la connectivité réseau ou DNS.

## Catégorie 6 : Commandes de Surveillance

### 6.1 kubectl top - Utilisation des Ressources

**Description** : Afficher l'utilisation CPU et mémoire en temps réel.

**Prérequis** : Le Metrics Server doit être activé :
```bash
microk8s enable metrics-server
```

**Commandes** :

**Utilisation par pod** :
```bash
microk8s kubectl top pods -n mon-namespace
```

Sortie typique :
```
NAME           CPU(cores)   MEMORY(bytes)
mon-pod        10m          64Mi
autre-pod      50m          128Mi
```

**Utilisation par nœud** :
```bash
microk8s kubectl top nodes
```

Sortie typique :
```
NAME        CPU(cores)   CPU%   MEMORY(bytes)   MEMORY%
ubuntu-pc   500m         25%    2048Mi          50%
```

**Options utiles** :
```bash
# Trier par utilisation CPU
microk8s kubectl top pods --sort-by=cpu

# Trier par utilisation mémoire
microk8s kubectl top pods --sort-by=memory

# Tous les namespaces
microk8s kubectl top pods -A

# Afficher les conteneurs individuellement
microk8s kubectl top pods --containers
```

**Cas d'utilisation** : Identifier rapidement les pods qui consomment trop de ressources.

### 6.2 kubectl get events - Surveiller les Événements

**Description** : Voir les événements du cluster (créations, erreurs, warnings).

**Commande** :
```bash
# Événements du namespace default
microk8s kubectl get events

# Événements d'un namespace spécifique
microk8s kubectl get events -n kube-system

# Tous les événements
microk8s kubectl get events -A

# Surveiller en temps réel
microk8s kubectl get events --watch
```

**Trier les événements** :
```bash
# Trier par date (plus récents en premier)
microk8s kubectl get events --sort-by='.lastTimestamp'

# Filtrer par type
microk8s kubectl get events --field-selector type=Warning
```

**Sortie typique** :
```
LAST SEEN   TYPE      REASON              OBJECT           MESSAGE
2m          Normal    Scheduled           pod/mon-pod      Successfully assigned default/mon-pod to ubuntu-pc
2m          Normal    Pulling             pod/mon-pod      Pulling image "nginx:1.21"
2m          Normal    Pulled              pod/mon-pod      Successfully pulled image
2m          Normal    Created             pod/mon-pod      Created container nginx
1m          Warning   BackOff             pod/autre-pod    Back-off restarting failed container
```

**Types d'événements** :
- **Normal** : Opérations standard (création, démarrage, etc.)
- **Warning** : Problèmes non critiques mais importants
- **Error** : Erreurs graves

**Cas d'utilisation** : Voir rapidement ce qui s'est passé récemment dans le cluster.

## Catégorie 7 : Commandes de Modification

### 7.1 kubectl edit - Éditer une Ressource

**Description** : Éditer une ressource directement avec votre éditeur de texte.

**Commande** :
```bash
microk8s kubectl edit deployment mon-deployment -n mon-namespace
```

**Ce qui se passe** :
1. La commande ouvre la définition YAML dans votre éditeur (vi/vim par défaut)
2. Vous modifiez le contenu
3. En sauvegardant et fermant, les modifications sont appliquées

**Changer l'éditeur** :
```bash
# Temporairement
EDITOR=nano microk8s kubectl edit deployment mon-deployment

# Ou définir la variable d'environnement
export EDITOR=nano
microk8s kubectl edit deployment mon-deployment
```

**Cas d'utilisation** : Modifications rapides en développement. Pour la production, préférez `kubectl apply -f` avec un fichier versionné.

### 7.2 kubectl scale - Changer le Nombre de Réplicas

**Description** : Augmenter ou diminuer le nombre de réplicas d'un deployment.

**Commande** :
```bash
# Passer à 5 réplicas
microk8s kubectl scale deployment mon-deployment --replicas=5

# Réduire à 1 réplica
microk8s kubectl scale deployment mon-deployment --replicas=1

# Échelle à 0 (arrêter tous les pods sans supprimer le deployment)
microk8s kubectl scale deployment mon-deployment --replicas=0
```

**Cas d'utilisation** : Ajuster rapidement la capacité sans modifier le fichier YAML.

### 7.3 kubectl delete - Supprimer des Ressources

**Description** : Supprimer une ou plusieurs ressources.

**Commandes** :
```bash
# Supprimer un pod spécifique
microk8s kubectl delete pod mon-pod

# Supprimer un deployment (et ses pods)
microk8s kubectl delete deployment mon-deployment

# Supprimer avec un fichier YAML
microk8s kubectl delete -f mon-fichier.yaml

# Supprimer toutes les ressources d'un label
microk8s kubectl delete pods -l app=nginx

# Supprimer toutes les ressources d'un namespace
microk8s kubectl delete all --all -n mon-namespace
```

**Options importantes** :
```bash
# Suppression immédiate (sans période de grâce)
microk8s kubectl delete pod mon-pod --grace-period=0 --force

# Suppression en cascade (par défaut)
microk8s kubectl delete deployment mon-deployment --cascade=foreground
```

**Attention** : `kubectl delete` est définitif ! Soyez prudent, surtout avec `--all`.

## Catégorie 8 : Commandes de Configuration

### 8.1 kubectl config - Gérer les Contextes

**Description** : Gérer les configurations kubectl (contextes, clusters, credentials).

**Commandes utiles** :

**Voir la configuration actuelle** :
```bash
microk8s kubectl config view
```

**Voir le contexte actuel** :
```bash
microk8s kubectl config current-context
```

**Lister tous les contextes** :
```bash
microk8s kubectl config get-contexts
```

**Changer de contexte** :
```bash
microk8s kubectl config use-context autre-contexte
```

**Définir le namespace par défaut** :
```bash
microk8s kubectl config set-context --current --namespace=mon-namespace
```

**Cas d'utilisation** : Gérer plusieurs clusters ou éviter de taper `-n namespace` à chaque fois.

### 8.2 kubectl api-resources - Lister les Types de Ressources

**Description** : Voir tous les types de ressources disponibles dans le cluster.

**Commande** :
```bash
microk8s kubectl api-resources
```

**Options utiles** :
```bash
# Afficher les noms courts
microk8s kubectl api-resources -o wide

# Filtrer par namespace (ressources qui ont un namespace)
microk8s kubectl api-resources --namespaced=true

# Filtrer par groupe d'API
microk8s kubectl api-resources --api-group=apps
```

**Cas d'utilisation** : Découvrir les noms courts des ressources ou vérifier si une ressource existe.

## Catégorie 9 : Commandes d'Inspection Réseau

### 9.1 kubectl port-forward - Accéder à un Pod Localement

**Description** : Créer un tunnel pour accéder à un port d'un pod depuis votre machine.

**Syntaxe** :
```bash
microk8s kubectl port-forward <nom-pod> <port-local>:<port-pod>
```

**Exemples** :
```bash
# Accéder au port 80 du pod sur le port 8080 local
microk8s kubectl port-forward mon-pod 8080:80

# Puis dans un autre terminal ou navigateur
curl localhost:8080
```

**Port-forward vers un service** :
```bash
microk8s kubectl port-forward service/mon-service 8080:80
```

**Port-forward vers un deployment** :
```bash
microk8s kubectl port-forward deployment/mon-deployment 8080:80
```

**Écouter sur toutes les interfaces** (attention à la sécurité) :
```bash
microk8s kubectl port-forward --address 0.0.0.0 mon-pod 8080:80
```

**Cas d'utilisation** : Accéder temporairement à un service non exposé pour le déboguer.

### 9.2 kubectl proxy - Accéder à l'API Kubernetes

**Description** : Créer un proxy vers l'API server Kubernetes.

**Commande** :
```bash
microk8s kubectl proxy --port=8001
```

Vous pouvez ensuite accéder à l'API via :
```bash
curl http://localhost:8001/api/v1/namespaces/default/pods
```

**Cas d'utilisation** : Interagir directement avec l'API Kubernetes pour du développement ou des tests.

## Catégorie 10 : Commandes Avancées de Dépannage

### 10.1 kubectl debug - Créer un Pod de Débogage

**Description** : Créer une copie d'un pod avec des outils de débogage (Kubernetes 1.18+).

**Commande** :
```bash
microk8s kubectl debug mon-pod -it --image=busybox --target=mon-conteneur
```

**Créer un pod éphémère de debug** :
```bash
microk8s kubectl debug mon-pod -it --image=ubuntu --share-processes --copy-to=mon-pod-debug
```

**Cas d'utilisation** : Déboguer un pod sans modifier l'image originale.

### 10.2 kubectl diff - Prévisualiser les Changements

**Description** : Voir les différences avant d'appliquer un changement.

**Commande** :
```bash
microk8s kubectl diff -f mon-fichier-modifie.yaml
```

**Cas d'utilisation** : Vérifier ce qui va changer avant un `kubectl apply`.

### 10.3 kubectl rollout - Gérer les Déploiements

**Description** : Contrôler les mises à jour et rollbacks des deployments.

**Commandes** :
```bash
# Voir le statut d'un rollout
microk8s kubectl rollout status deployment/mon-deployment

# Voir l'historique des déploiements
microk8s kubectl rollout history deployment/mon-deployment

# Revenir à la version précédente
microk8s kubectl rollout undo deployment/mon-deployment

# Revenir à une version spécifique
microk8s kubectl rollout undo deployment/mon-deployment --to-revision=2

# Mettre en pause un rollout
microk8s kubectl rollout pause deployment/mon-deployment

# Reprendre un rollout
microk8s kubectl rollout resume deployment/mon-deployment
```

**Cas d'utilisation** : Gérer les mises à jour d'application et revenir en arrière si problème.

## Catégorie 11 : Commandes MicroK8s Spécifiques

### 11.1 microk8s inspect - Diagnostic Complet

**Description** : Outil de diagnostic intégré à MicroK8s qui collecte toutes les informations importantes.

**Commande** :
```bash
microk8s inspect
```

**Ce qu'il fait** :
- Vérifie l'état du système
- Collecte les logs des services
- Vérifie la configuration réseau
- Teste la connectivité
- Génère un rapport complet dans `/var/snap/microk8s/current/inspection-report/`

**Options** :
```bash
# Générer un tarball pour partager
microk8s inspect --output /tmp/rapport.tar.gz
```

**Cas d'utilisation** : Premier diagnostic en cas de problème complexe ou pour partager avec le support.

### 11.2 microk8s kubectl - Commande Unified

**Description** : Accès direct à kubectl avec la configuration MicroK8s.

Toutes les commandes `kubectl` vues précédemment fonctionnent avec `microk8s kubectl`.

### 11.3 microk8s enable/disable - Gérer les Addons

**Description** : Activer ou désactiver des fonctionnalités.

**Commandes** :
```bash
# Lister les addons disponibles
microk8s status

# Activer un addon
microk8s enable dns

# Désactiver un addon
microk8s disable dns
```

## Workflows de Diagnostic Courants

Voici des séquences de commandes pour les situations courantes :

### Workflow 1 : Pod qui ne démarre pas

```bash
# 1. Vérifier l'état
microk8s kubectl get pods

# 2. Voir les détails et events
microk8s kubectl describe pod mon-pod

# 3. Vérifier les logs (si le pod a démarré au moins une fois)
microk8s kubectl logs mon-pod

# 4. Vérifier les logs du crash précédent
microk8s kubectl logs mon-pod --previous

# 5. Vérifier le deployment parent
microk8s kubectl describe deployment mon-deployment
```

### Workflow 2 : Problème de connectivité

```bash
# 1. Vérifier que le service existe
microk8s kubectl get service mon-service

# 2. Vérifier les endpoints
microk8s kubectl get endpoints mon-service

# 3. Décrire le service
microk8s kubectl describe service mon-service

# 4. Tester la résolution DNS
microk8s kubectl run test --image=busybox --rm -it -- nslookup mon-service

# 5. Tester la connectivité
microk8s kubectl run test --image=nicolaka/netshoot --rm -it -- curl http://mon-service:80
```

### Workflow 3 : Problème de performance

```bash
# 1. Vérifier l'utilisation des ressources des pods
microk8s kubectl top pods -A --sort-by=cpu

# 2. Vérifier l'utilisation des nœuds
microk8s kubectl top nodes

# 3. Vérifier les événements récents
microk8s kubectl get events --sort-by='.lastTimestamp' | head -20

# 4. Voir les pods évictés ou en erreur
microk8s kubectl get pods -A | grep -v Running

# 5. Vérifier les logs des pods problématiques
microk8s kubectl logs <pod-problematique>
```

### Workflow 4 : Audit complet d'un namespace

```bash
# 1. Vue d'ensemble
microk8s kubectl get all -n mon-namespace

# 2. État détaillé de chaque ressource
microk8s kubectl describe all -n mon-namespace

# 3. Événements récents
microk8s kubectl get events -n mon-namespace --sort-by='.lastTimestamp'

# 4. Utilisation des ressources
microk8s kubectl top pods -n mon-namespace

# 5. ConfigMaps et Secrets
microk8s kubectl get configmaps,secrets -n mon-namespace
```

## Astuces et Bonnes Pratiques

### 1. Utiliser des Alias

Créer des alias pour les commandes fréquentes :

```bash
# Dans votre ~/.bashrc ou ~/.zshrc
alias k='microk8s kubectl'
alias kg='microk8s kubectl get'
alias kd='microk8s kubectl describe'
alias kl='microk8s kubectl logs'
alias kx='microk8s kubectl exec -it'

# Après avoir sourcé le fichier, vous pouvez faire :
k get pods
kl mon-pod
kx mon-pod -- bash
```

### 2. Autocomplétion

Activer l'autocomplétion kubectl :

```bash
# Pour bash
source <(microk8s kubectl completion bash)

# Pour zsh
source <(microk8s kubectl completion zsh)

# Pour l'ajouter de façon permanente dans bash
echo "source <(microk8s kubectl completion bash)" >> ~/.bashrc
```

### 3. Utiliser jq pour Parser le JSON

Installer jq pour traiter facilement les sorties JSON :

```bash
sudo apt install jq

# Exemple : obtenir les IPs de tous les pods
microk8s kubectl get pods -o json | jq '.items[].status.podIP'

# Obtenir les noms et statuts
microk8s kubectl get pods -o json | jq '.items[] | {name: .metadata.name, status: .status.phase}'
```

### 4. Combiner Plusieurs Commandes

```bash
# Supprimer tous les pods en erreur
microk8s kubectl get pods | grep Error | awk '{print $1}' | xargs microk8s kubectl delete pod

# Redémarrer tous les pods d'un deployment
microk8s kubectl rollout restart deployment/mon-deployment
```

### 5. Surveillance Continue

```bash
# Surveiller plusieurs ressources en même temps avec watch
watch -n 2 'microk8s kubectl get pods,svc,deploy'

# Ou utiliser kubectl directement
microk8s kubectl get pods --watch
```

## Commandes à Mémoriser en Priorité

Pour les débutants, concentrez-vous sur ces commandes essentielles :

| Commande | Utilité |
|----------|---------|
| `microk8s status` | État du cluster |
| `microk8s kubectl get pods` | Lister les pods |
| `microk8s kubectl describe pod <nom>` | Détails d'un pod |
| `microk8s kubectl logs <pod>` | Voir les logs |
| `microk8s kubectl exec -it <pod> -- bash` | Entrer dans un pod |
| `microk8s kubectl get events` | Voir les événements |
| `microk8s inspect` | Diagnostic complet |

## Résumé

Cette section a couvert les commandes essentielles de diagnostic regroupées en 11 catégories :

1. **État général** : status, cluster-info, get nodes
2. **Listage** : get, get all
3. **Description** : describe, explain
4. **Logs** : logs
5. **Exécution** : exec, run
6. **Surveillance** : top, events
7. **Modification** : edit, scale, delete
8. **Configuration** : config, api-resources
9. **Réseau** : port-forward, proxy
10. **Avancé** : debug, diff, rollout
11. **MicroK8s** : inspect, enable/disable

**Conseil pour progresser** : Ne tentez pas de tout mémoriser d'un coup. Utilisez cette section comme référence et pratiquez avec les commandes les plus courantes. Avec le temps, elles deviendront naturelles.

**Prochaine étape** : Section 23.3 - microk8s inspect : l'outil de diagnostic intégré

---


⏭️ [microk8s inspect : l'outil de diagnostic intégré](/23-depannage-et-maintenance/03-microk8s-inspect-loutil-de-diagnostic-integre.md)
