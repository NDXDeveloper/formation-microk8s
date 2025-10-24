🔝 Retour au [Sommaire](/SOMMAIRE.md)

# Commandes MicroK8s Essentielles

## Introduction

MicroK8s fournit un ensemble de commandes simples et intuitives pour gérer votre cluster Kubernetes. Toutes les commandes commencent par `microk8s` suivi d'une sous-commande.

### Structure Générale

```bash
microk8s [commande] [options]
```

**Exemple :**
```bash
microk8s status
microk8s enable dns
microk8s kubectl get pods
```

### Pourquoi `microk8s kubectl` ?

MicroK8s encapsule `kubectl` pour éviter les conflits avec d'autres installations Kubernetes. Vous pouvez :

1. **Utiliser avec le préfixe** : `microk8s kubectl get pods`
2. **Créer un alias** : `alias kubectl='microk8s kubectl'`
3. **Exporter la config** : Pour utiliser `kubectl` directement

---

## Commandes de Gestion du Cluster

### microk8s status

Affiche l'état actuel du cluster MicroK8s.

**Syntaxe :**
```bash
microk8s status [options]
```

**Options :**
- `--wait-ready` : Attendre que le cluster soit prêt
- `--timeout SECONDS` : Timeout pour wait-ready (défaut: 30s)
- `--format yaml|json` : Format de sortie

**Exemples :**

```bash
# Statut simple
microk8s status

# Résultat typique :
# microk8s is running
# high-availability: no
# addons:
#   enabled:
#     dns                  # (core) CoreDNS
#     ha-cluster           # (core) Configure high availability on the current node
#   disabled:
#     dashboard            # (core) The Kubernetes dashboard
#     ingress              # (core) Ingress controller for external access
```

```bash
# Attendre que le cluster soit prêt
microk8s status --wait-ready

# Avec timeout de 60 secondes
microk8s status --wait-ready --timeout 60

# Format YAML
microk8s status --format yaml

# Format JSON pour scripts
microk8s status --format json
```

**Interprétation du résultat :**

- **running** : Le cluster fonctionne normalement ✓
- **stopped** : Le cluster est arrêté
- **not installed** : MicroK8s n'est pas installé
- **addons enabled** : Liste des addons actifs
- **addons disabled** : Liste des addons disponibles mais inactifs

### microk8s start

Démarre le cluster MicroK8s s'il est arrêté.

**Syntaxe :**
```bash
microk8s start
```

**Exemple :**

```bash
microk8s start
# Started
```

**Cas d'usage :**
- Après un redémarrage de la machine
- Après avoir arrêté manuellement le cluster
- Économiser des ressources quand le cluster n'est pas utilisé

**Temps de démarrage :** Généralement 10-30 secondes selon votre machine.

### microk8s stop

Arrête le cluster MicroK8s proprement.

**Syntaxe :**
```bash
microk8s stop
```

**Exemple :**

```bash
microk8s stop
# Stopped
```

**⚠️ Important :**
- Les pods en cours d'exécution seront terminés
- Les données dans les volumes persistants sont préservées
- La configuration du cluster est sauvegardée
- Les addons restent activés (ils redémarreront avec `microk8s start`)

**Cas d'usage :**
- Maintenance du système
- Économie de ressources (RAM/CPU)
- Avant une mise à jour système

### microk8s reset

Réinitialise complètement le cluster MicroK8s.

**Syntaxe :**
```bash
microk8s reset [--destroy-storage]
```

**Options :**
- `--destroy-storage` : Supprime également les volumes persistants

**Exemple :**

```bash
# Reset standard (préserve les volumes)
microk8s reset

# Reset complet (supprime tout)
microk8s reset --destroy-storage
```

**⚠️ ATTENTION :**
Cette commande est **destructive** :
- Supprime tous les pods et déploiements
- Supprime toutes les configurations
- Supprime tous les secrets et ConfigMaps
- Désactive tous les addons
- Avec `--destroy-storage` : supprime toutes les données

**Cas d'usage :**
- Repartir de zéro après des tests
- Nettoyer un cluster corrompu
- Préparer un nouvel environnement

**Alternative plus douce :**
```bash
# Supprimer tous les déploiements sans réinitialiser
microk8s kubectl delete all --all --all-namespaces
```

### microk8s version

Affiche la version de MicroK8s et de Kubernetes.

**Syntaxe :**
```bash
microk8s version
```

**Exemple :**

```bash
microk8s version

# Résultat :
# MicroK8s v1.28.3 revision 5891
# Kubernetes v1.28.3
```

**Informations affichées :**
- Version de MicroK8s (distribution)
- Numéro de révision (build)
- Version de Kubernetes (upstream)

**Cas d'usage :**
- Vérifier la version avant une mise à jour
- Documenter l'environnement
- Vérifier la compatibilité avec des applications

---

## Commandes de Gestion des Addons

### microk8s enable

Active un addon MicroK8s.

**Syntaxe :**
```bash
microk8s enable [addon] [options]
```

**Exemples courants :**

```bash
# DNS (essentiel pour la résolution de noms)
microk8s enable dns

# Dashboard Kubernetes
microk8s enable dashboard

# Ingress Controller
microk8s enable ingress

# Stockage persistant
microk8s enable hostpath-storage

# Registry privé
microk8s enable registry

# Prometheus pour monitoring
microk8s enable prometheus

# MetalLB pour load balancing
microk8s enable metallb:10.64.140.43-10.64.140.49

# Cert-Manager pour certificats SSL
microk8s enable cert-manager
```

**Addons avec configuration :**

Certains addons nécessitent des paramètres :

```bash
# MetalLB avec plage d'IPs
microk8s enable metallb:192.168.1.240-192.168.1.250

# GPU support avec runtime spécifique
microk8s enable gpu

# Helm3
microk8s enable helm3

# Multiple addons en une commande
microk8s enable dns dashboard ingress
```

**Temps d'activation :**
- Addons simples (dns) : 10-30 secondes
- Addons complexes (prometheus) : 1-3 minutes

**Vérification :**
```bash
# Vérifier que l'addon est activé
microk8s status

# Vérifier que les pods de l'addon sont Running
microk8s kubectl get pods -n [namespace-addon]
```

### microk8s disable

Désactive un addon MicroK8s.

**Syntaxe :**
```bash
microk8s disable [addon]
```

**Exemples :**

```bash
# Désactiver le dashboard
microk8s disable dashboard

# Désactiver l'ingress
microk8s disable ingress

# Désactiver plusieurs addons
microk8s disable prometheus grafana
```

**⚠️ Attention :**
- Supprime les ressources de l'addon
- Les données peuvent être perdues (selon l'addon)
- Les applications dépendantes peuvent cesser de fonctionner

**Exemple avec impact :**
```bash
# Désactiver DNS = les pods ne pourront plus résoudre les noms
microk8s disable dns
# ⚠️ Vos applications ne fonctionneront plus correctement !
```

**Cas d'usage :**
- Libérer des ressources
- Changer de configuration d'addon
- Nettoyage d'environnement de test

### microk8s enable/disable (liste complète)

**Addons Core (toujours disponibles) :**

| Addon | Description | Commande |
|-------|-------------|----------|
| **dns** | CoreDNS pour résolution de noms | `microk8s enable dns` |
| **dashboard** | Interface web Kubernetes | `microk8s enable dashboard` |
| **storage** | Stockage persistant (hostpath) | `microk8s enable storage` |
| **ingress** | NGINX Ingress Controller | `microk8s enable ingress` |
| **registry** | Registry Docker privé | `microk8s enable registry` |
| **metallb** | Load Balancer pour bare metal | `microk8s enable metallb:IPs` |
| **cert-manager** | Gestion automatique certificats | `microk8s enable cert-manager` |
| **prometheus** | Monitoring et métriques | `microk8s enable prometheus` |
| **metrics-server** | Métriques pour HPA | `microk8s enable metrics-server` |
| **helm3** | Gestionnaire de packages | `microk8s enable helm3` |

**Addons Communautaires :**

| Addon | Description | Commande |
|-------|-------------|----------|
| **gpu** | Support NVIDIA GPU | `microk8s enable gpu` |
| **istio** | Service mesh | `microk8s enable istio` |
| **knative** | Serverless | `microk8s enable knative` |
| **linkerd** | Service mesh léger | `microk8s enable linkerd` |
| **argocd** | GitOps CD | `microk8s enable argocd` |
| **portainer** | UI de gestion Docker/K8s | `microk8s enable portainer` |

**Voir la liste complète :**
```bash
microk8s status
```

---

## Commandes Kubectl via MicroK8s

MicroK8s encapsule `kubectl`. Toutes les commandes `kubectl` standard fonctionnent avec le préfixe `microk8s kubectl`.

### Gestion des Pods

```bash
# Lister tous les pods
microk8s kubectl get pods

# Lister dans tous les namespaces
microk8s kubectl get pods --all-namespaces
# ou
microk8s kubectl get pods -A

# Détails d'un pod
microk8s kubectl describe pod [nom-pod]

# Logs d'un pod
microk8s kubectl logs [nom-pod]

# Logs en temps réel (follow)
microk8s kubectl logs -f [nom-pod]

# Logs d'un conteneur spécifique (si plusieurs dans le pod)
microk8s kubectl logs [nom-pod] -c [nom-conteneur]

# Se connecter à un pod
microk8s kubectl exec -it [nom-pod] -- /bin/bash
# ou si bash n'est pas disponible
microk8s kubectl exec -it [nom-pod] -- /bin/sh

# Exécuter une commande dans un pod
microk8s kubectl exec [nom-pod] -- ls /app

# Supprimer un pod
microk8s kubectl delete pod [nom-pod]

# Afficher les pods avec plus d'infos (IPs, nœuds)
microk8s kubectl get pods -o wide

# Surveiller les pods en temps réel
microk8s kubectl get pods --watch
```

### Gestion des Deployments

```bash
# Lister les déploiements
microk8s kubectl get deployments

# Créer un déploiement
microk8s kubectl create deployment nginx --image=nginx

# Détails d'un déploiement
microk8s kubectl describe deployment [nom]

# Scaler un déploiement
microk8s kubectl scale deployment [nom] --replicas=3

# Mettre à jour l'image d'un déploiement
microk8s kubectl set image deployment/[nom] [conteneur]=[nouvelle-image]

# Voir l'historique des déploiements
microk8s kubectl rollout history deployment/[nom]

# Rollback vers version précédente
microk8s kubectl rollout undo deployment/[nom]

# Supprimer un déploiement
microk8s kubectl delete deployment [nom]
```

### Gestion des Services

```bash
# Lister les services
microk8s kubectl get services
# ou
microk8s kubectl get svc

# Détails d'un service
microk8s kubectl describe service [nom]

# Créer un service
microk8s kubectl expose deployment [nom] --type=ClusterIP --port=80

# Supprimer un service
microk8s kubectl delete service [nom]

# Voir les endpoints (pods derrière le service)
microk8s kubectl get endpoints [nom-service]
```

### Gestion des Namespaces

```bash
# Lister les namespaces
microk8s kubectl get namespaces
# ou
microk8s kubectl get ns

# Créer un namespace
microk8s kubectl create namespace [nom]

# Supprimer un namespace (⚠️ supprime tout dedans)
microk8s kubectl delete namespace [nom]

# Changer le namespace par défaut
microk8s kubectl config set-context --current --namespace=[nom]
```

### Application de Manifestes YAML

```bash
# Appliquer un fichier YAML
microk8s kubectl apply -f [fichier.yaml]

# Appliquer tous les YAMLs d'un dossier
microk8s kubectl apply -f [dossier]/

# Appliquer depuis une URL
microk8s kubectl apply -f https://example.com/manifest.yaml

# Supprimer les ressources d'un fichier
microk8s kubectl delete -f [fichier.yaml]

# Dry-run (simuler sans appliquer)
microk8s kubectl apply -f [fichier.yaml] --dry-run=client

# Afficher les différences avant application
microk8s kubectl diff -f [fichier.yaml]
```

### Affichage des Ressources

```bash
# Format de sortie large (plus d'infos)
microk8s kubectl get pods -o wide

# Format YAML
microk8s kubectl get pod [nom] -o yaml

# Format JSON
microk8s kubectl get pod [nom] -o json

# Extraction de champ spécifique (avec jsonpath)
microk8s kubectl get pods -o jsonpath='{.items[*].metadata.name}'

# Afficher les labels
microk8s kubectl get pods --show-labels

# Filtrer par label
microk8s kubectl get pods -l app=nginx

# Trier par date de création
microk8s kubectl get pods --sort-by=.metadata.creationTimestamp
```

---

## Commandes de Diagnostic et Troubleshooting

### microk8s inspect

Génère un rapport de diagnostic complet du cluster.

**Syntaxe :**
```bash
microk8s inspect [options]
```

**Options :**
- `--help` : Affiche l'aide

**Exemple :**

```bash
microk8s inspect

# Crée un fichier d'inspection
# Inspecting services
# Inspecting AppArmor
# Inspecting dmesg
# Inspecting snap services
#
# Building the report tarball
# Report tarball is at /var/snap/microk8s/common/inspection-report-20231024_143022.tar.gz
```

**Contenu du rapport :**
- Logs de tous les services système
- État des pods système (kube-system)
- Configuration réseau
- Snapshots des ressources
- Logs d'erreurs système

**Cas d'usage :**
- Debugging de problèmes complexes
- Support technique (partager le rapport)
- Documentation d'incidents

**Analyser le rapport :**
```bash
# Extraire le rapport
tar -xzf inspection-report-*.tar.gz

# Explorer le contenu
cd inspection-report-*/
ls -la
```

### microk8s kubectl top

Affiche l'utilisation des ressources (CPU, RAM).

**Prérequis :** L'addon `metrics-server` doit être activé.

```bash
# Activer metrics-server
microk8s enable metrics-server

# Attendre 1-2 minutes pour collecte des métriques
```

**Exemples :**

```bash
# Utilisation des nœuds
microk8s kubectl top nodes

# Résultat :
# NAME       CPU(cores)   CPU%   MEMORY(bytes)   MEMORY%
# hostname   523m         26%    2048Mi          51%

# Utilisation des pods
microk8s kubectl top pods

# Dans tous les namespaces
microk8s kubectl top pods -A

# Trier par CPU
microk8s kubectl top pods --sort-by=cpu

# Trier par mémoire
microk8s kubectl top pods --sort-by=memory

# Pour un namespace spécifique
microk8s kubectl top pods -n kube-system
```

**Interprétation :**
- **CPU** : en millicores (m) - 1000m = 1 CPU core
- **Memory** : en Mi (Mebibytes) - 1024Mi = 1Gi

### Commandes de Debug

```bash
# Événements du cluster (très utile pour debug)
microk8s kubectl get events

# Événements triés par date
microk8s kubectl get events --sort-by='.lastTimestamp'

# Événements dans tous les namespaces
microk8s kubectl get events -A

# Voir les événements d'une ressource spécifique
microk8s kubectl describe pod [nom] | grep -A 10 Events

# Logs système de MicroK8s
sudo journalctl -u snap.microk8s.daemon-kubelite

# Logs en temps réel
sudo journalctl -u snap.microk8s.daemon-kubelite -f

# Logs d'un service spécifique
sudo journalctl -u snap.microk8s.daemon-containerd

# Vérifier l'état des services MicroK8s
systemctl status snap.microk8s.daemon-*

# Redémarrer un service spécifique
sudo systemctl restart snap.microk8s.daemon-kubelite
```

---

## Commandes de Configuration

### microk8s config

Exporte la configuration kubectl pour utilisation externe.

**Syntaxe :**
```bash
microk8s config
```

**Exemple :**

```bash
# Voir la configuration
microk8s config

# Sauvegarder dans un fichier
microk8s config > ~/.kube/microk8s-config

# Utiliser avec kubectl standard
export KUBECONFIG=~/.kube/microk8s-config
kubectl get pods

# Fusionner avec config kubectl existante
microk8s config > ~/.kube/config-microk8s
export KUBECONFIG=~/.kube/config:~/.kube/config-microk8s
kubectl config view --flatten > ~/.kube/config
```

**Cas d'usage :**
- Utiliser `kubectl` sans le préfixe `microk8s`
- Intégration avec outils externes (Lens, k9s)
- Gestion multi-clusters

### Alias kubectl

Pour éviter de taper `microk8s kubectl` à chaque fois :

**Option 1 : Alias temporaire**
```bash
alias kubectl='microk8s kubectl'

# Utilisation
kubectl get pods
```

**Option 2 : Alias permanent**
```bash
# Ajouter dans ~/.bashrc ou ~/.zshrc
echo "alias kubectl='microk8s kubectl'" >> ~/.bashrc
source ~/.bashrc

# Ou pour zsh
echo "alias kubectl='microk8s kubectl'" >> ~/.zshrc
source ~/.zshrc
```

**Option 3 : Fonction avec complétion**
```bash
# Ajouter dans ~/.bashrc
kubectl() {
  microk8s kubectl "$@"
}

# Complétion automatique
source <(microk8s kubectl completion bash)
```

### microk8s add-node (Multi-node)

Génère une commande pour ajouter un nœud au cluster.

**Syntaxe :**
```bash
microk8s add-node [--token TOKEN] [--token-ttl SECONDS]
```

**Exemple sur le nœud maître :**

```bash
microk8s add-node

# Résultat :
# From the node you wish to join to this cluster, run the following:
# microk8s join 192.168.1.101:25000/abc123def456/uuid
#
# Use the '--worker' flag to join as a worker only node.
#
# If the node you are adding is not reachable through the default interface,
# you can use the following command:
# microk8s join 192.168.1.101:25000/abc123def456/uuid --refresh-interval 10
```

**Sur le nouveau nœud :**

```bash
# Rejoindre en tant que nœud complet (control plane)
microk8s join 192.168.1.101:25000/abc123def456/uuid

# Rejoindre en tant que worker uniquement
microk8s join 192.168.1.101:25000/abc123def456/uuid --worker
```

**Options :**
- `--worker` : Rejoindre comme worker (pas de control plane)
- `--token TOKEN` : Spécifier un token personnalisé
- `--token-ttl SECONDS` : Durée de validité du token (défaut: 1h)

**Vérification :**
```bash
# Sur le nœud maître, vérifier les nœuds
microk8s kubectl get nodes
```

### microk8s remove-node

Retire un nœud du cluster.

**Syntaxe :**
```bash
microk8s remove-node [nom-du-nœud]
```

**Exemple :**

```bash
# Lister les nœuds
microk8s kubectl get nodes

# Retirer un nœud
microk8s remove-node worker-node-2

# Sur le nœud retiré, nettoyer
microk8s leave
```

---

## Commandes Réseau et Connectivité

### microk8s kubectl port-forward

Rediriger un port local vers un pod.

**Syntaxe :**
```bash
microk8s kubectl port-forward [pod|service]/[nom] [port-local]:[port-pod]
```

**Exemples :**

```bash
# Rediriger port 8080 local vers port 80 du pod
microk8s kubectl port-forward pod/nginx 8080:80

# Accès via http://localhost:8080

# Port-forward vers un service
microk8s kubectl port-forward service/backend 8080:8080

# Écouter sur toutes les interfaces (pas seulement localhost)
microk8s kubectl port-forward --address 0.0.0.0 pod/nginx 8080:80

# Multiple ports
microk8s kubectl port-forward pod/app 8080:80 9090:9090
```

**Cas d'usage :**
- Accéder à un service depuis votre machine locale
- Débugger une application dans le cluster
- Accéder au dashboard Kubernetes

### microk8s kubectl proxy

Crée un proxy vers l'API Kubernetes.

**Syntaxe :**
```bash
microk8s kubectl proxy [--port=PORT]
```

**Exemple :**

```bash
# Démarrer le proxy (port 8001 par défaut)
microk8s kubectl proxy

# Accéder à l'API via http://localhost:8001

# Spécifier un port
microk8s kubectl proxy --port=8080

# Accéder au dashboard (si activé)
# http://localhost:8001/api/v1/namespaces/kube-system/services/https:kubernetes-dashboard:/proxy/
```

---

## Commandes de Gestion des Ressources

### Créer des Ressources

```bash
# Créer un namespace
microk8s kubectl create namespace [nom]

# Créer un déploiement
microk8s kubectl create deployment [nom] --image=[image]

# Créer un service
microk8s kubectl create service clusterip [nom] --tcp=80:80

# Créer un secret depuis un fichier
microk8s kubectl create secret generic [nom] --from-file=[fichier]

# Créer un secret depuis des valeurs littérales
microk8s kubectl create secret generic db-secret \
  --from-literal=username=admin \
  --from-literal=password=secret123

# Créer un ConfigMap
microk8s kubectl create configmap [nom] --from-literal=key=value

# Créer depuis stdin
cat <<EOF | microk8s kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: test
spec:
  containers:
  - name: nginx
    image: nginx
EOF
```

### Éditer des Ressources

```bash
# Éditer un déploiement (ouvre dans l'éditeur)
microk8s kubectl edit deployment [nom]

# Éditer un service
microk8s kubectl edit service [nom]

# Éditer avec un éditeur spécifique
EDITOR=nano microk8s kubectl edit deployment [nom]

# Patcher une ressource (modification ciblée)
microk8s kubectl patch deployment [nom] -p '{"spec":{"replicas":5}}'

# Mettre à jour un label
microk8s kubectl label pod [nom] environment=production

# Retirer un label
microk8s kubectl label pod [nom] environment-
```

### Copier des Fichiers

```bash
# Copier un fichier vers un pod
microk8s kubectl cp [fichier-local] [pod]:[chemin-pod]

# Exemple
microk8s kubectl cp config.json nginx:/etc/app/config.json

# Copier depuis un pod vers local
microk8s kubectl cp [pod]:[chemin-pod] [fichier-local]

# Exemple
microk8s kubectl cp nginx:/var/log/app.log ./app.log

# Spécifier le conteneur (si plusieurs dans le pod)
microk8s kubectl cp [fichier] [pod]:[chemin] -c [conteneur]
```

---

## Commandes de Nettoyage

### Supprimer des Ressources

```bash
# Supprimer un pod
microk8s kubectl delete pod [nom]

# Supprimer un déploiement
microk8s kubectl delete deployment [nom]

# Supprimer par fichier
microk8s kubectl delete -f [fichier.yaml]

# Supprimer par label
microk8s kubectl delete pods -l app=nginx

# Supprimer tous les pods d'un namespace
microk8s kubectl delete pods --all -n [namespace]

# Supprimer tout dans un namespace (⚠️ dangereux)
microk8s kubectl delete all --all -n [namespace]

# Forcer la suppression (si bloqué)
microk8s kubectl delete pod [nom] --force --grace-period=0

# Supprimer un namespace et tout son contenu
microk8s kubectl delete namespace [nom]
```

### Nettoyer les Ressources Inutilisées

```bash
# Supprimer les pods en état Completed
microk8s kubectl delete pods --field-selector=status.phase==Succeeded --all-namespaces

# Supprimer les pods en état Failed
microk8s kubectl delete pods --field-selector=status.phase==Failed --all-namespaces

# Supprimer les pods Evicted
microk8s kubectl get pods -A | grep Evicted | awk '{print $2, $1}' | xargs -n2 microk8s kubectl delete pod -n

# Nettoyer les images Docker inutilisées
microk8s ctr images ls | tail -n +2 | awk '{print $1}' | xargs -I {} microk8s ctr images rm {}
```

---

## Exemples de Workflows Complets

### Workflow 1 : Déployer une Application Simple

```bash
# 1. Créer un namespace
microk8s kubectl create namespace myapp

# 2. Créer un déploiement
microk8s kubectl create deployment nginx --image=nginx -n myapp

# 3. Vérifier que le pod démarre
microk8s kubectl get pods -n myapp --watch

# 4. Exposer le déploiement
microk8s kubectl expose deployment nginx --port=80 --type=ClusterIP -n myapp

# 5. Vérifier le service
microk8s kubectl get svc -n myapp

# 6. Tester depuis un pod temporaire
microk8s kubectl run test --image=busybox --rm -it -n myapp -- wget -O- http://nginx
```

### Workflow 2 : Débugger une Application

```bash
# 1. Lister les pods problématiques
microk8s kubectl get pods -A | grep -v Running

# 2. Voir les détails du pod
microk8s kubectl describe pod [nom-pod] -n [namespace]

# 3. Vérifier les logs
microk8s kubectl logs [nom-pod] -n [namespace]

# 4. Logs précédents (si le pod a crashé)
microk8s kubectl logs [nom-pod] -n [namespace] --previous

# 5. Se connecter au pod pour investigation
microk8s kubectl exec -it [nom-pod] -n [namespace] -- /bin/sh

# 6. Vérifier les événements récents
microk8s kubectl get events -n [namespace] --sort-by='.lastTimestamp'

# 7. Vérifier les ressources
microk8s kubectl top pods -n [namespace]
```

### Workflow 3 : Mise à Jour d'Application

```bash
# 1. Vérifier l'état actuel
microk8s kubectl get deployment [nom] -n [namespace]

# 2. Mettre à jour l'image
microk8s kubectl set image deployment/[nom] [conteneur]=[nouvelle-image:tag] -n [namespace]

# 3. Suivre le rollout
microk8s kubectl rollout status deployment/[nom] -n [namespace]

# 4. Si problème, rollback
microk8s kubectl rollout undo deployment/[nom] -n [namespace]

# 5. Vérifier l'historique
microk8s kubectl rollout history deployment/[nom] -n [namespace]
```

### Workflow 4 : Backup de la Configuration

```bash
# 1. Exporter tous les déploiements
microk8s kubectl get deployments --all-namespaces -o yaml > deployments-backup.yaml

# 2. Exporter tous les services
microk8s kubectl get services --all-namespaces -o yaml > services-backup.yaml

# 3. Exporter tous les configmaps
microk8s kubectl get configmaps --all-namespaces -o yaml > configmaps-backup.yaml

# 4. Exporter tous les secrets
microk8s kubectl get secrets --all-namespaces -o yaml > secrets-backup.yaml

# 5. Créer un rapport d'inspection
microk8s inspect

# 6. Tout compresser
tar -czf kubernetes-backup-$(date +%Y%m%d).tar.gz *.yaml inspection-report-*.tar.gz
```

---

## Tableau Récapitulatif des Commandes Essentielles

### Gestion du Cluster

| Commande | Description | Exemple |
|----------|-------------|---------|
| `microk8s status` | État du cluster | `microk8s status` |
| `microk8s start` | Démarrer le cluster | `microk8s start` |
| `microk8s stop` | Arrêter le cluster | `microk8s stop` |
| `microk8s reset` | Réinitialiser | `microk8s reset` |
| `microk8s version` | Version | `microk8s version` |
| `microk8s inspect` | Diagnostic | `microk8s inspect` |

### Addons

| Commande | Description | Exemple |
|----------|-------------|---------|
| `microk8s enable [addon]` | Activer addon | `microk8s enable dns` |
| `microk8s disable [addon]` | Désactiver addon | `microk8s disable dashboard` |
| `microk8s status` | Liste addons | `microk8s status` |

### Pods

| Commande | Description | Exemple |
|----------|-------------|---------|
| `kubectl get pods` | Lister pods | `microk8s kubectl get pods` |
| `kubectl describe pod` | Détails pod | `microk8s kubectl describe pod nginx` |
| `kubectl logs` | Voir logs | `microk8s kubectl logs nginx -f` |
| `kubectl exec` | Se connecter | `microk8s kubectl exec -it nginx -- bash` |
| `kubectl delete pod` | Supprimer | `microk8s kubectl delete pod nginx` |

### Déploiements

| Commande | Description | Exemple |
|----------|-------------|---------|
| `kubectl get deployments` | Lister | `microk8s kubectl get deployments` |
| `kubectl create deployment` | Créer | `microk8s kubectl create deployment nginx --image=nginx` |
| `kubectl scale` | Scaler | `microk8s kubectl scale deployment nginx --replicas=3` |
| `kubectl rollout` | Gérer rollout | `microk8s kubectl rollout status deployment/nginx` |

### Services

| Commande | Description | Exemple |
|----------|-------------|---------|
| `kubectl get services` | Lister | `microk8s kubectl get services` |
| `kubectl expose` | Exposer | `microk8s kubectl expose deployment nginx --port=80` |
| `kubectl describe service` | Détails | `microk8s kubectl describe service nginx` |

### Configuration

| Commande | Description | Exemple |
|----------|-------------|---------|
| `kubectl apply -f` | Appliquer YAML | `microk8s kubectl apply -f app.yaml` |
| `kubectl create` | Créer ressource | `microk8s kubectl create namespace prod` |
| `kubectl edit` | Éditer ressource | `microk8s kubectl edit deployment nginx` |
| `kubectl delete -f` | Supprimer par fichier | `microk8s kubectl delete -f app.yaml` |

---

## Astuces et Raccourcis

### Raccourcis kubectl

```bash
# Raccourcis de ressources
po    = pods
svc   = services
deploy = deployments
ns    = namespaces
cm    = configmaps
no    = nodes

# Exemples
microk8s kubectl get po
microk8s kubectl get svc
microk8s kubectl get deploy
```

### Options Globales Utiles

```bash
# Tous les namespaces
-A
--all-namespaces

# Namespace spécifique
-n [namespace]

# Format de sortie
-o wide          # Plus d'infos
-o yaml          # YAML complet
-o json          # JSON complet
-o name          # Juste les noms

# Watch (surveillance temps réel)
--watch
-w

# Labels
--show-labels
-l key=value     # Filtrer par label

# Dry-run (simulation)
--dry-run=client

# Help
--help
-h
```

### Scripts Bash Utiles

**Script 1 : Vérification santé complète**

```bash
#!/bin/bash
# health-check.sh

echo "=== MicroK8s Status ==="
microk8s status

echo -e "\n=== Nodes ==="
microk8s kubectl get nodes

echo -e "\n=== Pods Not Running ==="
microk8s kubectl get pods -A | grep -v Running | grep -v Completed

echo -e "\n=== Services ==="
microk8s kubectl get svc -A

echo -e "\n=== Resource Usage ==="
microk8s kubectl top nodes 2>/dev/null || echo "metrics-server not enabled"

echo -e "\n=== Recent Events ==="
microk8s kubectl get events -A --sort-by='.lastTimestamp' | tail -10
```

**Script 2 : Nettoyage pods Failed/Completed**

```bash
#!/bin/bash
# cleanup.sh

echo "Cleaning up Failed pods..."
microk8s kubectl delete pods --field-selector=status.phase==Failed -A

echo "Cleaning up Completed pods..."
microk8s kubectl delete pods --field-selector=status.phase==Succeeded -A

echo "Done!"
```

**Script 3 : Backup complet**

```bash
#!/bin/bash
# backup.sh

BACKUP_DIR="k8s-backup-$(date +%Y%m%d-%H%M%S)"
mkdir -p $BACKUP_DIR

echo "Backing up resources to $BACKUP_DIR..."

microk8s kubectl get all -A -o yaml > $BACKUP_DIR/all-resources.yaml
microk8s kubectl get configmaps -A -o yaml > $BACKUP_DIR/configmaps.yaml
microk8s kubectl get secrets -A -o yaml > $BACKUP_DIR/secrets.yaml
microk8s kubectl get pvc -A -o yaml > $BACKUP_DIR/pvc.yaml
microk8s kubectl get ingress -A -o yaml > $BACKUP_DIR/ingress.yaml
microk8s config > $BACKUP_DIR/kubeconfig

echo "Creating inspection report..."
microk8s inspect
mv inspection-report-*.tar.gz $BACKUP_DIR/

tar -czf $BACKUP_DIR.tar.gz $BACKUP_DIR
echo "Backup completed: $BACKUP_DIR.tar.gz"
```

---

## Complétion Automatique

### Activer la Complétion Bash

```bash
# Pour la session en cours
source <(microk8s kubectl completion bash)

# Permanent (ajouter à ~/.bashrc)
echo 'source <(microk8s kubectl completion bash)' >> ~/.bashrc

# Si vous utilisez l'alias kubectl
echo 'alias kubectl="microk8s kubectl"' >> ~/.bashrc
echo 'complete -F __start_kubectl kubectl' >> ~/.bashrc
```

### Activer la Complétion Zsh

```bash
# Pour la session en cours
source <(microk8s kubectl completion zsh)

# Permanent (ajouter à ~/.zshrc)
echo 'source <(microk8s kubectl completion zsh)' >> ~/.zshrc

# Si vous utilisez l'alias kubectl
echo 'alias kubectl="microk8s kubectl"' >> ~/.zshrc
echo 'compdef __start_kubectl kubectl' >> ~/.zshrc
```

---

## Commandes Avancées

### Gestion des Contextes

```bash
# Voir les contextes disponibles
microk8s kubectl config get-contexts

# Changer de contexte
microk8s kubectl config use-context [nom]

# Voir le contexte actuel
microk8s kubectl config current-context

# Renommer un contexte
microk8s kubectl config rename-context [ancien] [nouveau]
```

### Annotations et Labels

```bash
# Ajouter un label
microk8s kubectl label pod [nom] env=production

# Modifier un label existant
microk8s kubectl label pod [nom] env=staging --overwrite

# Supprimer un label
microk8s kubectl label pod [nom] env-

# Ajouter une annotation
microk8s kubectl annotate pod [nom] description="Application principale"

# Voir les labels et annotations
microk8s kubectl describe pod [nom]
```

### Gestion des Taints et Tolerations

```bash
# Ajouter un taint à un nœud
microk8s kubectl taint nodes [nom] key=value:NoSchedule

# Retirer un taint
microk8s kubectl taint nodes [nom] key:NoSchedule-

# Voir les taints
microk8s kubectl describe node [nom] | grep Taints
```

### Drain et Cordon

```bash
# Cordon (empêcher nouveaux pods)
microk8s kubectl cordon [nom-nœud]

# Uncordon (autoriser nouveaux pods)
microk8s kubectl uncordon [nom-nœud]

# Drain (évacuer tous les pods)
microk8s kubectl drain [nom-nœud] --ignore-daemonsets --delete-emptydir-data

# Drain avec période de grâce
microk8s kubectl drain [nom-nœud] --grace-period=300 --ignore-daemonsets
```

---

## Commandes de Performance

### Benchmarking

```bash
# Mesurer le temps de démarrage des pods
time microk8s kubectl run test --image=nginx --restart=Never

# Stress test avec multiple replicas
microk8s kubectl create deployment stress --image=nginx --replicas=50

# Mesurer le temps de scale
time microk8s kubectl scale deployment stress --replicas=100
```

### Profiling

```bash
# Afficher les ressources triées
microk8s kubectl top pods --sort-by=memory -A
microk8s kubectl top pods --sort-by=cpu -A

# Voir les limites de ressources
microk8s kubectl describe nodes | grep -A 5 "Allocated resources"
```

---

## Bonnes Pratiques

### 1. Toujours Spécifier le Namespace

```bash
# ✗ Mauvais (utilise default)
microk8s kubectl get pods

# ✓ Bon
microk8s kubectl get pods -n production
```

### 2. Utiliser des Labels Cohérents

```bash
# ✓ Bon système de labels
app: backend
tier: api
environment: production
version: v2.1
team: platform
```

### 3. Dry-run Avant Application

```bash
# Toujours tester avant d'appliquer
microk8s kubectl apply -f manifest.yaml --dry-run=client
microk8s kubectl diff -f manifest.yaml
```

### 4. Utiliser des Manifestes YAML

```bash
# ✗ Éviter les commandes imperatives en production
microk8s kubectl create deployment nginx --image=nginx

# ✓ Préférer les manifestes
microk8s kubectl apply -f deployment.yaml
```

### 5. Versionner les Configurations

```bash
# Stocker dans Git
git add manifests/
git commit -m "Update deployment configuration"
git push
```

### 6. Nettoyer Régulièrement

```bash
# Script de nettoyage hebdomadaire
microk8s kubectl delete pods --field-selector=status.phase==Failed -A
microk8s kubectl delete pods --field-selector=status.phase==Succeeded -A
```

---

## Résolution de Problèmes Courants

### Problème : Commande bloquée

```bash
# Si une commande ne répond pas
Ctrl+C  # Interrompre

# Vérifier l'état du cluster
microk8s status

# Redémarrer si nécessaire
microk8s stop
microk8s start
```

### Problème : Permission denied

```bash
# Ajouter votre utilisateur au groupe microk8s
sudo usermod -a -G microk8s $USER
sudo chown -f -R $USER ~/.kube

# Recharger les groupes
newgrp microk8s

# Ou se reconnecter
```

### Problème : Addon ne démarre pas

```bash
# Voir les logs détaillés
microk8s inspect

# Désactiver et réactiver
microk8s disable [addon]
microk8s enable [addon]

# Vérifier les pods système
microk8s kubectl get pods -n kube-system
```

---

## Checklist Quotidienne

```bash
# 1. Vérifier l'état du cluster
microk8s status

# 2. Vérifier la santé des nœuds
microk8s kubectl get nodes

# 3. Vérifier les pods problématiques
microk8s kubectl get pods -A | grep -v Running | grep -v Completed

# 4. Vérifier les événements récents
microk8s kubectl get events -A --sort-by='.lastTimestamp' | tail -20

# 5. Vérifier l'utilisation des ressources
microk8s kubectl top nodes
microk8s kubectl top pods -A --sort-by=memory | head -10

# 6. Nettoyer si nécessaire
microk8s kubectl delete pods --field-selector=status.phase==Failed -A
```

---

## Ressources et Documentation

### Documentation Officielle

- **MicroK8s** : https://microk8s.io/docs
- **Kubectl** : https://kubernetes.io/docs/reference/kubectl/
- **Kubernetes** : https://kubernetes.io/docs/home/

### Cheat Sheets

- **Kubectl Cheat Sheet** : https://kubernetes.io/docs/reference/kubectl/cheatsheet/
- **MicroK8s Commands** : https://microk8s.io/docs/command-reference

### Outils Complémentaires

- **k9s** : Terminal UI pour Kubernetes
- **Lens** : IDE Kubernetes
- **kubectx/kubens** : Changement rapide de contexte/namespace

---

## Conclusion

Vous disposez maintenant d'une référence complète des commandes MicroK8s essentielles. Voici un récapitulatif des commandes les plus utilisées au quotidien :

### Top 10 des Commandes Quotidiennes

1. `microk8s status` - Vérifier l'état
2. `microk8s kubectl get pods -A` - Voir tous les pods
3. `microk8s kubectl logs [pod] -f` - Suivre les logs
4. `microk8s kubectl describe pod [nom]` - Débugger
5. `microk8s kubectl apply -f [fichier]` - Déployer
6. `microk8s kubectl get svc` - Voir les services
7. `microk8s kubectl exec -it [pod] -- bash` - Se connecter
8. `microk8s enable [addon]` - Activer fonctionnalités
9. `microk8s kubectl top pods` - Surveiller ressources
10. `microk8s inspect` - Diagnostic complet

**Pro Tip :** Créez un alias `kubectl` et utilisez la complétion automatique pour gagner du temps !

```bash
alias kubectl='microk8s kubectl'
source <(microk8s kubectl completion bash)
```

Avec ces commandes maîtrisées, vous êtes prêt à gérer efficacement votre cluster MicroK8s ! 🚀

⏭️ [Commandes kubectl essentielles](/annexes/annexe-d-reference-rapide/commandes-kubectl-essentielles.md)
