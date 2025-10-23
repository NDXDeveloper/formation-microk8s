🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 4.6 Commandes kubectl Essentielles

## Introduction

`kubectl` (prononcé "koube-cé-té-el" ou "koube-control") est l'outil en ligne de commande pour interagir avec Kubernetes. C'est votre interface principale pour gérer votre cluster, déployer des applications, et diagnostiquer des problèmes.

Cette section est un **guide de référence** des commandes kubectl les plus importantes. Gardez-la à portée de main, elle vous sera utile au quotidien !

**Note pour MicroK8s :** Toutes les commandes présentées utilisent `microk8s kubectl`. Si vous avez un alias configuré (`alias kubectl='microk8s kubectl'`), vous pouvez simplement utiliser `kubectl`.

## Syntaxe Générale de kubectl

```bash
kubectl [commande] [TYPE] [NOM] [flags]
```

**Composants :**
- **commande** : L'action à effectuer (`get`, `create`, `delete`, etc.)
- **TYPE** : Le type de ressource (`pod`, `deployment`, `service`, etc.)
- **NOM** : Le nom de la ressource (optionnel)
- **flags** : Options additionnelles (`-n`, `-o`, `--all-namespaces`, etc.)

**Exemples :**
```bash
# Structure complète
microk8s kubectl get pods nginx-pod -n production -o yaml

# Commande : get
# Type     : pods
# Nom      : nginx-pod
# Flags    : -n production (namespace), -o yaml (format de sortie)
```

## Catégories de Commandes

Les commandes kubectl se divisent en plusieurs catégories :

| Catégorie | Exemples | Usage |
|-----------|----------|-------|
| **Gestion de base** | `get`, `describe`, `create`, `delete` | CRUD des ressources |
| **Déploiement** | `apply`, `rollout`, `scale` | Déployer et gérer |
| **Inspection** | `logs`, `exec`, `port-forward` | Déboguer |
| **Contexte** | `config`, `cluster-info` | Configuration |
| **Avancé** | `edit`, `patch`, `replace` | Modifications |

## 1. Commandes de Base (CRUD)

### get - Lister les ressources

La commande la plus utilisée pour visualiser l'état des ressources.

```bash
# Lister les Pods
microk8s kubectl get pods

# Lister les Deployments
microk8s kubectl get deployments

# Lister les Services
microk8s kubectl get services
# ou forme courte :
microk8s kubectl get svc

# Lister plusieurs types en une commande
microk8s kubectl get pods,services,deployments

# Tous les Pods de tous les namespaces
microk8s kubectl get pods --all-namespaces
# ou forme courte :
microk8s kubectl get pods -A

# Dans un namespace spécifique
microk8s kubectl get pods -n production

# Avec plus d'informations (IP, Node, etc.)
microk8s kubectl get pods -o wide

# Afficher les labels
microk8s kubectl get pods --show-labels

# Filtrer par label
microk8s kubectl get pods -l app=nginx
microk8s kubectl get pods -l app=nginx,environment=production
```

**Formats de sortie :**

```bash
# YAML complet
microk8s kubectl get pod nginx-pod -o yaml

# JSON complet
microk8s kubectl get pod nginx-pod -o json

# Nom uniquement
microk8s kubectl get pods -o name

# Format personnalisé (JSONPath)
microk8s kubectl get pods -o jsonpath='{.items[*].metadata.name}'

# Tableau personnalisé
microk8s kubectl get pods -o custom-columns=NAME:.metadata.name,STATUS:.status.phase,IP:.status.podIP
```

**Raccourcis de types de ressources :**

| Type complet | Raccourci | Type complet | Raccourci |
|--------------|-----------|--------------|-----------|
| `pods` | `po` | `services` | `svc` |
| `deployments` | `deploy` | `replicasets` | `rs` |
| `namespaces` | `ns` | `nodes` | `no` |
| `configmaps` | `cm` | `secrets` | `secret` |
| `persistentvolumes` | `pv` | `persistentvolumeclaims` | `pvc` |
| `ingresses` | `ing` | `daemonsets` | `ds` |

**Exemples avec raccourcis :**
```bash
microk8s kubectl get po        # Pods
microk8s kubectl get deploy    # Deployments
microk8s kubectl get svc       # Services
microk8s kubectl get ns        # Namespaces
```

### describe - Détails complets d'une ressource

Affiche des informations détaillées incluant les événements récents.

```bash
# Détails d'un Pod
microk8s kubectl describe pod nginx-pod

# Détails d'un Deployment
microk8s kubectl describe deployment nginx-deployment

# Détails d'un Service
microk8s kubectl describe service nginx-service

# Détails d'un Node
microk8s kubectl describe node

# Dans un namespace spécifique
microk8s kubectl describe pod nginx-pod -n production
```

**Ce que montre `describe` :**
- Métadonnées (nom, namespace, labels, annotations)
- Spécifications (images, ports, volumes)
- État actuel (statut, conditions)
- **Événements** (très utile pour le débogage !)
- Ressources (requests, limits)

**Astuce :** La section **Events** en bas du `describe` est souvent la clé pour résoudre les problèmes.

### create - Créer une ressource

Crée une ressource de manière impérative.

```bash
# Créer un namespace
microk8s kubectl create namespace production

# Créer un Deployment
microk8s kubectl create deployment nginx --image=nginx:1.21

# Créer un Service
microk8s kubectl create service clusterip nginx --tcp=80:80

# Créer un ConfigMap depuis un literal
microk8s kubectl create configmap app-config \
  --from-literal=key1=value1 \
  --from-literal=key2=value2

# Créer un Secret
microk8s kubectl create secret generic db-secret \
  --from-literal=username=admin \
  --from-literal=password=secret123

# Créer depuis un fichier YAML
microk8s kubectl create -f deployment.yaml

# Créer depuis plusieurs fichiers
microk8s kubectl create -f ./configs/

# Créer depuis une URL
microk8s kubectl create -f https://example.com/deployment.yaml
```

### apply - Créer ou mettre à jour (déclaratif)

**Recommandé** pour la gestion déclarative (GitOps).

```bash
# Créer ou mettre à jour depuis un fichier
microk8s kubectl apply -f deployment.yaml

# Appliquer un répertoire entier
microk8s kubectl apply -f ./manifests/

# Appliquer récursivement
microk8s kubectl apply -f ./configs/ -R

# Appliquer depuis une URL
microk8s kubectl apply -f https://example.com/app.yaml

# Dry-run (tester sans appliquer)
microk8s kubectl apply -f deployment.yaml --dry-run=client

# Dry-run avec validation serveur
microk8s kubectl apply -f deployment.yaml --dry-run=server
```

**Différence `create` vs `apply` :**

| `create` | `apply` |
|----------|---------|
| Échoue si la ressource existe | Crée ou met à jour |
| Impératif | Déclaratif |
| Usage ponctuel | Usage GitOps |

### delete - Supprimer des ressources

```bash
# Supprimer un Pod
microk8s kubectl delete pod nginx-pod

# Supprimer un Deployment
microk8s kubectl delete deployment nginx-deployment

# Supprimer plusieurs ressources
microk8s kubectl delete pod pod1 pod2 pod3

# Supprimer depuis un fichier
microk8s kubectl delete -f deployment.yaml

# Supprimer par label
microk8s kubectl delete pods -l app=nginx

# Supprimer tous les Pods d'un namespace
microk8s kubectl delete pods --all -n development

# Supprimer un namespace et tout son contenu
microk8s kubectl delete namespace development

# Forcer la suppression immédiate (attention !)
microk8s kubectl delete pod nginx-pod --force --grace-period=0
```

**⚠️ Attention :** `delete` est irréversible ! Utilisez `--dry-run` pour tester.

### edit - Éditer une ressource en direct

Ouvre la ressource dans un éditeur pour modification directe.

```bash
# Éditer un Deployment
microk8s kubectl edit deployment nginx-deployment

# Éditer un Service
microk8s kubectl edit service nginx-service

# Éditer avec un éditeur spécifique
EDITOR=nano microk8s kubectl edit deployment nginx-deployment
```

**Note :** Les modifications prennent effet immédiatement après sauvegarde.

## 2. Commandes de Déploiement

### apply - Déployer ou mettre à jour

Déjà couverte ci-dessus, mais voici des cas d'usage avancés :

```bash
# Appliquer avec annotation de changement
microk8s kubectl apply -f deployment.yaml \
  --record  # Enregistre la commande dans l'historique

# Forcer la mise à jour (remplace)
microk8s kubectl apply -f deployment.yaml --force

# Appliquer seulement les objets d'un certain type
microk8s kubectl apply -f . --prune -l app=myapp
```

### rollout - Gérer les déploiements

Contrôle les mises à jour progressives (rolling updates).

```bash
# Voir le statut d'un déploiement
microk8s kubectl rollout status deployment/nginx-deployment

# Voir l'historique des déploiements
microk8s kubectl rollout history deployment/nginx-deployment

# Voir les détails d'une révision spécifique
microk8s kubectl rollout history deployment/nginx-deployment --revision=2

# Annuler le dernier déploiement (rollback)
microk8s kubectl rollout undo deployment/nginx-deployment

# Revenir à une révision spécifique
microk8s kubectl rollout undo deployment/nginx-deployment --to-revision=3

# Mettre en pause un déploiement
microk8s kubectl rollout pause deployment/nginx-deployment

# Reprendre un déploiement
microk8s kubectl rollout resume deployment/nginx-deployment

# Redémarrer un déploiement (tous les Pods)
microk8s kubectl rollout restart deployment/nginx-deployment
```

**Cas d'usage du restart :**
- Après modification d'un ConfigMap ou Secret
- Pour appliquer des mises à jour du Node
- Pour résoudre des problèmes temporaires

### scale - Changer le nombre de répliques

```bash
# Augmenter à 5 répliques
microk8s kubectl scale deployment nginx-deployment --replicas=5

# Réduire à 1 réplique
microk8s kubectl scale deployment nginx-deployment --replicas=1

# Scaler si la condition est remplie
microk8s kubectl scale deployment nginx-deployment --replicas=3 --current-replicas=2

# Scaler un StatefulSet
microk8s kubectl scale statefulset web --replicas=3

# Scaler un ReplicaSet
microk8s kubectl scale rs nginx-rs --replicas=4
```

### set - Modifier des propriétés

```bash
# Changer l'image d'un conteneur
microk8s kubectl set image deployment/nginx-deployment nginx=nginx:1.22

# Changer les ressources
microk8s kubectl set resources deployment nginx-deployment \
  -c=nginx --limits=cpu=500m,memory=512Mi

# Changer les variables d'environnement
microk8s kubectl set env deployment/nginx-deployment \
  ENV=production LOG_LEVEL=info

# Supprimer une variable d'environnement
microk8s kubectl set env deployment/nginx-deployment ENV-

# Changer le ServiceAccount
microk8s kubectl set serviceaccount deployment nginx-deployment myserviceaccount
```

## 3. Commandes d'Inspection et Débogage

### logs - Voir les logs

Affiche les logs des conteneurs.

```bash
# Logs d'un Pod
microk8s kubectl logs nginx-pod

# Logs d'un conteneur spécifique (si plusieurs dans le Pod)
microk8s kubectl logs nginx-pod -c nginx-container

# Suivre les logs en temps réel
microk8s kubectl logs -f nginx-pod

# Logs des 100 dernières lignes
microk8s kubectl logs --tail=100 nginx-pod

# Logs depuis les 1h dernières
microk8s kubectl logs --since=1h nginx-pod

# Logs du conteneur précédent (avant crash)
microk8s kubectl logs nginx-pod --previous

# Logs de tous les Pods d'un Deployment
microk8s kubectl logs -l app=nginx

# Logs avec timestamps
microk8s kubectl logs nginx-pod --timestamps
```

**Astuces pour les logs :**

```bash
# Chercher dans les logs
microk8s kubectl logs nginx-pod | grep ERROR

# Sauvegarder les logs dans un fichier
microk8s kubectl logs nginx-pod > nginx-logs.txt

# Suivre les logs de plusieurs Pods
microk8s kubectl logs -f -l app=nginx --all-containers=true
```

### exec - Exécuter des commandes dans un conteneur

Accède à un shell dans un conteneur en cours d'exécution.

```bash
# Ouvrir un shell interactif
microk8s kubectl exec -it nginx-pod -- /bin/sh
microk8s kubectl exec -it nginx-pod -- /bin/bash

# Exécuter une commande unique
microk8s kubectl exec nginx-pod -- ls /etc
microk8s kubectl exec nginx-pod -- cat /etc/nginx/nginx.conf

# Dans un conteneur spécifique (si plusieurs)
microk8s kubectl exec -it nginx-pod -c nginx-container -- /bin/sh

# Exécuter avec des variables d'environnement
microk8s kubectl exec nginx-pod -- env
```

**Commandes utiles dans un Pod :**

```bash
# Tester la connectivité réseau
microk8s kubectl exec nginx-pod -- ping google.com
microk8s kubectl exec nginx-pod -- curl http://service-name

# Voir les processus
microk8s kubectl exec nginx-pod -- ps aux

# Vérifier les ports écoutés
microk8s kubectl exec nginx-pod -- netstat -tlnp

# Tester une requête HTTP locale
microk8s kubectl exec nginx-pod -- curl localhost:80
```

### port-forward - Créer un tunnel vers un Pod/Service

Redirige un port local vers un Pod ou Service.

```bash
# Port-forward vers un Pod
microk8s kubectl port-forward nginx-pod 8080:80

# Port-forward vers un Service
microk8s kubectl port-forward service/nginx-service 8080:80

# Port-forward vers un Deployment
microk8s kubectl port-forward deployment/nginx-deployment 8080:80

# Utiliser un port local aléatoire
microk8s kubectl port-forward nginx-pod :80

# Écouter sur toutes les interfaces (pas seulement localhost)
microk8s kubectl port-forward --address 0.0.0.0 nginx-pod 8080:80

# Dans un namespace spécifique
microk8s kubectl port-forward -n production service/api 8080:80
```

**Usage :** Accédez ensuite à `http://localhost:8080` dans votre navigateur.

**Note :** Le port-forward reste actif jusqu'à ce que vous l'arrêtiez (Ctrl+C).

### cp - Copier des fichiers

Copie des fichiers entre votre machine et un Pod.

```bash
# Copier un fichier local vers un Pod
microk8s kubectl cp ./local-file.txt nginx-pod:/tmp/remote-file.txt

# Copier un fichier d'un Pod vers local
microk8s kubectl cp nginx-pod:/etc/nginx/nginx.conf ./nginx.conf

# Copier un répertoire
microk8s kubectl cp ./local-dir nginx-pod:/tmp/remote-dir

# Dans un conteneur spécifique
microk8s kubectl cp local-file.txt nginx-pod:/tmp/file.txt -c nginx-container

# Dans un namespace spécifique
microk8s kubectl cp local-file.txt production/nginx-pod:/tmp/file.txt
```

**⚠️ Limitation :** Nécessite que `tar` soit disponible dans le conteneur.

### top - Utilisation des ressources

Affiche l'utilisation CPU et RAM (nécessite Metrics Server).

```bash
# Utilisation des Nodes
microk8s kubectl top nodes

# Utilisation des Pods
microk8s kubectl top pods

# Dans un namespace spécifique
microk8s kubectl top pods -n production

# Tous les namespaces
microk8s kubectl top pods -A

# Avec les conteneurs
microk8s kubectl top pods --containers

# Trier par CPU
microk8s kubectl top pods --sort-by=cpu

# Trier par mémoire
microk8s kubectl top pods --sort-by=memory
```

**Note :** Si vous obtenez une erreur, activez Metrics Server :
```bash
microk8s enable metrics-server
```

## 4. Commandes de Contexte et Configuration

### config - Gérer la configuration kubectl

```bash
# Voir la configuration actuelle
microk8s kubectl config view

# Voir le contexte actuel
microk8s kubectl config current-context

# Lister tous les contextes
microk8s kubectl config get-contexts

# Changer de contexte
microk8s kubectl config use-context minikube

# Définir le namespace par défaut
microk8s kubectl config set-context --current --namespace=production
```

**Note pour MicroK8s :** La configuration est gérée automatiquement, ces commandes sont surtout utiles si vous gérez plusieurs clusters.

### cluster-info - Informations sur le cluster

```bash
# Informations de base
microk8s kubectl cluster-info

# Informations détaillées
microk8s kubectl cluster-info dump

# Sauvegarder les infos dans un fichier
microk8s kubectl cluster-info dump > cluster-info.txt
```

### version - Version de kubectl et du cluster

```bash
# Version client et serveur
microk8s kubectl version

# Version courte
microk8s kubectl version --short

# Seulement la version client
microk8s kubectl version --client
```

### api-resources - Lister les types de ressources

```bash
# Toutes les ressources disponibles
microk8s kubectl api-resources

# Avec détails (shortnames, API group)
microk8s kubectl api-resources -o wide

# Filtrer par API group
microk8s kubectl api-resources --api-group=apps

# Ressources dans le namespace
microk8s kubectl api-resources --namespaced=true

# Ressources hors namespace
microk8s kubectl api-resources --namespaced=false
```

### explain - Documentation des ressources

```bash
# Documentation d'une ressource
microk8s kubectl explain pods

# Documentation d'un champ spécifique
microk8s kubectl explain pods.spec.containers

# Documentation récursive
microk8s kubectl explain pods.spec --recursive

# Avec plus de détails
microk8s kubectl explain deployment.spec.strategy
```

**Astuce :** Utilisez `explain` pour découvrir les champs disponibles sans consulter la documentation en ligne.

## 5. Commandes Avancées

### patch - Modifier partiellement une ressource

```bash
# Patch JSON
microk8s kubectl patch deployment nginx-deployment \
  -p '{"spec":{"replicas":5}}'

# Patch YAML
microk8s kubectl patch deployment nginx-deployment \
  --patch-file patch.yaml

# Patch pour ajouter un label
microk8s kubectl patch pod nginx-pod \
  -p '{"metadata":{"labels":{"env":"production"}}}'

# Patch pour changer l'image
microk8s kubectl patch deployment nginx-deployment \
  -p '{"spec":{"template":{"spec":{"containers":[{"name":"nginx","image":"nginx:1.22"}]}}}}'
```

### replace - Remplacer une ressource

```bash
# Remplacer depuis un fichier
microk8s kubectl replace -f deployment.yaml

# Forcer le remplacement
microk8s kubectl replace -f deployment.yaml --force

# Remplacer avec confirmation
microk8s kubectl replace -f deployment.yaml --validate=true
```

### label - Gérer les labels

```bash
# Ajouter un label
microk8s kubectl label pods nginx-pod env=production

# Modifier un label existant
microk8s kubectl label pods nginx-pod env=staging --overwrite

# Supprimer un label
microk8s kubectl label pods nginx-pod env-

# Ajouter un label à plusieurs ressources
microk8s kubectl label pods -l app=nginx tier=frontend
```

### annotate - Gérer les annotations

```bash
# Ajouter une annotation
microk8s kubectl annotate pods nginx-pod description="Production web server"

# Modifier une annotation
microk8s kubectl annotate pods nginx-pod description="Updated description" --overwrite

# Supprimer une annotation
microk8s kubectl annotate pods nginx-pod description-
```

### wait - Attendre une condition

```bash
# Attendre qu'un Pod soit prêt
microk8s kubectl wait --for=condition=ready pod nginx-pod

# Attendre que tous les Pods soient prêts
microk8s kubectl wait --for=condition=ready pod -l app=nginx

# Avec timeout
microk8s kubectl wait --for=condition=ready pod nginx-pod --timeout=60s

# Attendre la suppression
microk8s kubectl wait --for=delete pod nginx-pod --timeout=60s
```

## 6. Commandes Utiles au Quotidien

### Obtenir des informations rapidement

```bash
# Statut global du cluster
microk8s kubectl get all

# Statut global avec plus de détails
microk8s kubectl get all -o wide

# Tous les objets dans un namespace
microk8s kubectl get all -n production

# Événements récents (très utile pour déboguer)
microk8s kubectl get events

# Événements triés par date
microk8s kubectl get events --sort-by='.lastTimestamp'

# Événements d'un Pod spécifique
microk8s kubectl get events --field-selector involvedObject.name=nginx-pod

# Compter les ressources
microk8s kubectl get pods --no-headers | wc -l
```

### Recherche et filtrage

```bash
# Filtrer par label (une seule valeur)
microk8s kubectl get pods -l app=nginx

# Filtrer par plusieurs labels
microk8s kubectl get pods -l app=nginx,env=production

# Filtrer par label avec opérateurs
microk8s kubectl get pods -l 'app in (nginx,apache)'
microk8s kubectl get pods -l 'env!=production'

# Filtrer par field
microk8s kubectl get pods --field-selector status.phase=Running
microk8s kubectl get pods --field-selector metadata.namespace=default

# Combiner field et label selectors
microk8s kubectl get pods -l app=nginx --field-selector status.phase=Running
```

### Formats de sortie utiles

```bash
# Noms uniquement (facile à piper)
microk8s kubectl get pods -o name

# Format large avec plus d'infos
microk8s kubectl get pods -o wide

# YAML complet
microk8s kubectl get pod nginx-pod -o yaml

# JSON complet
microk8s kubectl get pod nginx-pod -o json

# JSONPath personnalisé
microk8s kubectl get pods -o jsonpath='{.items[*].metadata.name}'
microk8s kubectl get pods -o jsonpath='{.items[*].status.podIP}'

# Colonnes personnalisées
microk8s kubectl get pods -o custom-columns=NAME:.metadata.name,STATUS:.status.phase,IP:.status.podIP,NODE:.spec.nodeName
```

### Nettoyer les ressources

```bash
# Supprimer les Pods terminés
microk8s kubectl delete pods --field-selector status.phase=Succeeded

# Supprimer les Pods en erreur
microk8s kubectl delete pods --field-selector status.phase=Failed

# Supprimer tous les Pods Evicted
microk8s kubectl get pods -A --field-selector status.phase=Failed | grep Evicted | awk '{print $2}' | xargs kubectl delete pod

# Nettoyer un namespace entier
microk8s kubectl delete all --all -n development
```

## 7. Options Globales Importantes

Ces flags fonctionnent avec la plupart des commandes :

```bash
# Namespace spécifique
-n, --namespace=production

# Tous les namespaces
-A, --all-namespaces

# Format de sortie
-o yaml
-o json
-o wide
-o name

# Afficher les labels
--show-labels

# Dry-run (simulation)
--dry-run=client
--dry-run=server

# Aide
-h, --help

# Verbose (débogage)
-v=6  # Niveau de verbosité (0-10)
```

## 8. Alias et Raccourcis

### Configurer des alias

Ajoutez ces lignes à votre `~/.bashrc` ou `~/.zshrc` :

```bash
# Alias de base
alias k='microk8s kubectl'
alias kgp='microk8s kubectl get pods'
alias kgs='microk8s kubectl get services'
alias kgd='microk8s kubectl get deployments'
alias kga='microk8s kubectl get all'
alias kdp='microk8s kubectl describe pod'
alias kdd='microk8s kubectl describe deployment'
alias kds='microk8s kubectl describe service'
alias klf='microk8s kubectl logs -f'
alias kex='microk8s kubectl exec -it'
alias kaf='microk8s kubectl apply -f'
alias kdelf='microk8s kubectl delete -f'

# Alias avancés
alias kgpa='microk8s kubectl get pods -A'
alias kgpw='microk8s kubectl get pods -o wide'
alias kgpn='microk8s kubectl get pods -o name'
alias kge='microk8s kubectl get events --sort-by=.lastTimestamp'
alias ktn='microk8s kubectl top nodes'
alias ktp='microk8s kubectl top pods'
```

Rechargez la configuration :
```bash
source ~/.bashrc
```

Maintenant vous pouvez utiliser :
```bash
k get pods              # Au lieu de microk8s kubectl get pods
kgp                     # Au lieu de microk8s kubectl get pods
klf nginx-pod           # Au lieu de microk8s kubectl logs -f nginx-pod
```

### Auto-complétion

Activez l'auto-complétion pour kubectl (si vous utilisez l'alias `kubectl`) :

```bash
# Pour Bash
echo 'source <(kubectl completion bash)' >> ~/.bashrc
echo 'complete -F __start_kubectl k' >> ~/.bashrc

# Pour Zsh
echo 'source <(kubectl completion zsh)' >> ~/.zshrc
echo 'complete -F __start_kubectl k' >> ~/.zshrc
```

## 9. Commandes de Debug Avancées

### Créer un Pod de debug

```bash
# Pod temporaire pour tester
microk8s kubectl run test-pod --image=busybox --rm -it --restart=Never -- sh

# Pod avec des outils réseau
microk8s kubectl run netshoot --image=nicolaka/netshoot --rm -it --restart=Never -- bash

# Pod dans le même namespace réseau qu'un autre Pod
microk8s kubectl debug nginx-pod -it --image=busybox
```

### Vérifier la connectivité

```bash
# DNS interne
microk8s kubectl run dns-test --image=busybox --rm -it --restart=Never -- nslookup kubernetes.default

# Tester un Service
microk8s kubectl run curl-test --image=curlimages/curl --rm -it --restart=Never -- curl http://nginx-service

# Tester depuis un Pod existant
microk8s kubectl exec -it nginx-pod -- wget -qO- http://other-service
```

### Inspecter les Endpoints

```bash
# Voir les Endpoints d'un Service
microk8s kubectl get endpoints nginx-service

# Détails des Endpoints
microk8s kubectl describe endpoints nginx-service

# Vérifier que les Pods sont dans les Endpoints
microk8s kubectl get pods -l app=nginx -o wide
microk8s kubectl get endpoints nginx-service
```

## 10. Génération de Manifestes

kubectl peut générer des manifestes YAML sans créer les ressources :

```bash
# Deployment
microk8s kubectl create deployment nginx --image=nginx --dry-run=client -o yaml

# Service
microk8s kubectl create service clusterip nginx --tcp=80:80 --dry-run=client -o yaml

# ConfigMap
microk8s kubectl create configmap app-config --from-literal=key=value --dry-run=client -o yaml

# Job
microk8s kubectl create job test --image=busybox --dry-run=client -o yaml

# CronJob
microk8s kubectl create cronjob test --image=busybox --schedule="*/5 * * * *" --dry-run=client -o yaml

# Sauvegarder dans un fichier
microk8s kubectl create deployment nginx --image=nginx --dry-run=client -o yaml > nginx-deployment.yaml
```

## 11. Commandes de Production

### Backup d'une ressource

```bash
# Sauvegarder un Deployment
microk8s kubectl get deployment nginx-deployment -o yaml > nginx-deployment-backup.yaml

# Sauvegarder tous les Deployments
microk8s kubectl get deployments -o yaml > all-deployments-backup.yaml

# Sauvegarder tout un namespace
microk8s kubectl get all -n production -o yaml > production-backup.yaml
```

### Comparer des versions

```bash
# Avant modification
microk8s kubectl get deployment nginx-deployment -o yaml > before.yaml

# Après modification
microk8s kubectl get deployment nginx-deployment -o yaml > after.yaml

# Comparer
diff before.yaml after.yaml
```

### Validation

```bash
# Valider un fichier sans l'appliquer
microk8s kubectl apply -f deployment.yaml --dry-run=client

# Valider côté serveur (plus complet)
microk8s kubectl apply -f deployment.yaml --dry-run=server

# Valider la syntaxe YAML
microk8s kubectl apply -f deployment.yaml --validate=true
```

## 12. Cheat Sheet Rapide

### Commandes les plus utilisées

```bash
# Voir l'état
microk8s kubectl get pods
microk8s kubectl get all
microk8s kubectl describe pod <name>
microk8s kubectl logs <pod-name>
microk8s kubectl get events --sort-by=.lastTimestamp

# Créer/Modifier
microk8s kubectl apply -f <file>
microk8s kubectl create <resource>
microk8s kubectl edit <resource> <name>

# Déploiement
microk8s kubectl rollout status deployment/<name>
microk8s kubectl rollout undo deployment/<name>
microk8s kubectl scale deployment/<name> --replicas=<n>

# Debug
microk8s kubectl logs -f <pod-name>
microk8s kubectl exec -it <pod-name> -- /bin/sh
microk8s kubectl port-forward <pod-name> 8080:80
microk8s kubectl top pods

# Nettoyage
microk8s kubectl delete <resource> <name>
microk8s kubectl delete -f <file>
microk8s kubectl delete pods -l app=<label>
```

### Raccourcis de types

```bash
po = pods
deploy = deployments
svc = services
ns = namespaces
cm = configmaps
pv = persistentvolumes
pvc = persistentvolumeclaims
ing = ingresses
```

### Options courantes

```bash
-n <namespace>          # Namespace spécifique
-A                      # Tous les namespaces
-o yaml                 # Format YAML
-o wide                 # Plus d'infos
--show-labels           # Afficher labels
-l app=nginx            # Filtrer par label
--dry-run=client        # Simulation
-f                      # Suivre (logs)
-it                     # Interactif + TTY
```

## Récapitulatif

Dans cette section, vous avez découvert :

✅ La **syntaxe générale** de kubectl
✅ Les **commandes de base** : get, describe, create, apply, delete
✅ Les **commandes de déploiement** : rollout, scale, set
✅ Les **commandes de debug** : logs, exec, port-forward, top
✅ Les **commandes de configuration** : config, cluster-info
✅ Les **commandes avancées** : patch, replace, label, annotate
✅ Les **alias et raccourcis** pour gagner du temps
✅ Les **techniques de debugging** et de production
✅ Un **cheat sheet** de référence rapide

**Points clés :**

- `kubectl` est votre interface principale avec Kubernetes
- La plupart des commandes supportent les **raccourcis** (po, deploy, svc)
- Les **flags globaux** (-n, -o, -l) fonctionnent avec presque toutes les commandes
- `describe` et `get events` sont vos meilleurs amis pour le **debugging**
- Utilisez `--dry-run` pour **tester sans risque**
- Configurez des **alias** pour être plus productif

**Conseil :** Gardez ce document à portée de main comme référence. Avec la pratique, ces commandes deviendront naturelles !

---

**Prochaine section :** 4.7 Inspection et débogage de base

⏭️ [Inspection et débogage de base](/04-premiers-deploiements/07-inspection-et-debogage-de-base.md)
