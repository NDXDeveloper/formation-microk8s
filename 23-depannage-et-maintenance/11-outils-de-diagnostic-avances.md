🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 23.11 Outils de Diagnostic Avancés

## Introduction

Au-delà des commandes kubectl de base, il existe tout un écosystème d'outils qui facilitent le diagnostic, le debugging et le monitoring des clusters Kubernetes. Ces outils peuvent vous faire gagner un temps précieux lors du dépannage de problèmes complexes. Cette section vous présente les outils les plus utiles et comment les utiliser efficacement.

> **Pour les débutants** : Si kubectl de base est comme un tournevis, les outils avancés sont comme une boîte à outils complète. Chaque outil a sa spécialité : certains excellent pour voir les logs, d'autres pour inspecter le réseau, d'autres encore pour surveiller les performances. Apprendre à utiliser ces outils vous transforme d'un bricoleur occasionnel en un véritable professionnel du dépannage Kubernetes.

## Kubectl : Options Avancées

### Output Formats Avancés

Au-delà du format par défaut, kubectl supporte plusieurs formats de sortie très utiles.

#### Format YAML/JSON Complet

```bash
# YAML complet (toute la configuration)
microk8s kubectl get pod mon-pod -o yaml

# JSON complet
microk8s kubectl get pod mon-pod -o json
```

**Utilité** : Voir TOUTE la configuration, y compris les valeurs par défaut ajoutées par Kubernetes.

#### JSONPath pour Extraction Ciblée

**Extraire des champs spécifiques** :

```bash
# Obtenir uniquement l'IP d'un pod
microk8s kubectl get pod mon-pod -o jsonpath='{.status.podIP}'

# Obtenir le node d'un pod
microk8s kubectl get pod mon-pod -o jsonpath='{.spec.nodeName}'

# Image utilisée
microk8s kubectl get pod mon-pod -o jsonpath='{.spec.containers[0].image}'

# Status de tous les conteneurs
microk8s kubectl get pod mon-pod -o jsonpath='{.status.containerStatuses[*].state}'
```

**Exemples pratiques** :

```bash
# Liste de tous les pods avec leur IP et nœud
microk8s kubectl get pods -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.status.podIP}{"\t"}{.spec.nodeName}{"\n"}{end}'

# Toutes les images utilisées dans le cluster
microk8s kubectl get pods --all-namespaces -o jsonpath='{.items[*].spec.containers[*].image}' | tr -s '[[:space:]]' '\n' | sort | uniq

# Trouver les pods qui utilisent le plus de CPU requests
microk8s kubectl get pods --all-namespaces -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.spec.containers[*].resources.requests.cpu}{"\n"}{end}' | sort -k2 -rh
```

#### Custom Columns

**Créer des tableaux personnalisés** :

```bash
# Colonnes personnalisées
microk8s kubectl get pods -o custom-columns=NAME:.metadata.name,STATUS:.status.phase,NODE:.spec.nodeName,IP:.status.podIP

# Sortie :
# NAME      STATUS    NODE        IP
# mon-pod   Running   microk8s    10.1.1.5
```

**Exemples avancés** :

```bash
# Pods avec resources requests
microk8s kubectl get pods -o custom-columns=NAME:.metadata.name,CPU_REQ:.spec.containers[0].resources.requests.cpu,MEM_REQ:.spec.containers[0].resources.requests.memory

# Services avec leurs types et IPs
microk8s kubectl get svc -o custom-columns=NAME:.metadata.name,TYPE:.spec.type,CLUSTER-IP:.spec.clusterIP,EXTERNAL-IP:.status.loadBalancer.ingress[0].ip
```

#### Wide Output

```bash
# Format wide (plus d'informations)
microk8s kubectl get pods -o wide

# Sortie :
# NAME      READY   STATUS    RESTARTS   AGE   IP           NODE
# mon-pod   1/1     Running   0          5m    10.1.1.5     microk8s
```

### Watch Mode (Surveillance en Temps Réel)

**Observer les changements en direct** :

```bash
# Watch sur les pods
microk8s kubectl get pods --watch

# Watch avec output personnalisé
microk8s kubectl get pods -o wide --watch

# Watch sur les événements
microk8s kubectl get events --watch --sort-by='.lastTimestamp'
```

**Raccourci** : `-w` pour `--watch`

```bash
microk8s kubectl get pods -w
```

### Describe Avancé

```bash
# Describe avec plus de contexte
microk8s kubectl describe pod mon-pod

# Describe de toutes les ressources d'un type
microk8s kubectl describe pods

# Describe par label
microk8s kubectl describe pods -l app=nginx
```

### Commandes Exec Avancées

#### Shell Interactif

```bash
# Shell bash
microk8s kubectl exec -it mon-pod -- bash

# Si bash n'existe pas, essayer sh
microk8s kubectl exec -it mon-pod -- sh

# Spécifier le conteneur (si pod multi-conteneurs)
microk8s kubectl exec -it mon-pod -c mon-conteneur -- bash
```

#### Commandes Non-Interactives

```bash
# Exécuter une commande simple
microk8s kubectl exec mon-pod -- ls -la /app

# Avec pipe
microk8s kubectl exec mon-pod -- cat /etc/hosts | grep kubernetes

# Variables d'environnement
microk8s kubectl exec mon-pod -- env

# Processus en cours
microk8s kubectl exec mon-pod -- ps aux
```

### Logs Avancés

#### Options de Base

```bash
# Logs en temps réel
microk8s kubectl logs -f mon-pod

# Dernières N lignes
microk8s kubectl logs --tail=100 mon-pod

# Logs avec timestamps
microk8s kubectl logs --timestamps mon-pod

# Logs depuis une durée
microk8s kubectl logs --since=1h mon-pod
microk8s kubectl logs --since=5m mon-pod
```

#### Logs de Conteneur Précédent

**Après un crash/restart** :

```bash
# Logs du conteneur précédent (avant restart)
microk8s kubectl logs mon-pod --previous

# Utile pour voir pourquoi un pod a crashé
```

#### Logs Multi-Conteneurs

```bash
# Logs d'un conteneur spécifique
microk8s kubectl logs mon-pod -c conteneur-1

# Logs de tous les conteneurs
microk8s kubectl logs mon-pod --all-containers=true

# Follow tous les conteneurs
microk8s kubectl logs -f mon-pod --all-containers=true
```

#### Logs par Label

```bash
# Logs de tous les pods avec un label
microk8s kubectl logs -l app=nginx --tail=50

# Attention : ceci affiche les logs séquentiellement, pas en temps réel combiné
```

## Kubectl Debug : Conteneurs Éphémères

Kubernetes 1.23+ supporte les conteneurs éphémères pour le debugging.

### Concept

**Conteneur éphémère** : Un conteneur temporaire ajouté à un pod existant pour le debugging.

**Cas d'usage** :
- Pod sans shell (image distroless)
- Pod en CrashLoopBackOff
- Debugging sans modifier l'image

### Utilisation

#### Créer un Conteneur de Debug

```bash
# Attacher un conteneur debug avec une image complète
microk8s kubectl debug mon-pod -it --image=ubuntu

# Avec une image spécialisée
microk8s kubectl debug mon-pod -it --image=nicolaka/netshoot

# Partager le namespace réseau du conteneur cible
microk8s kubectl debug mon-pod -it --image=ubuntu --target=mon-conteneur
```

**Dans le conteneur debug** :

```bash
# Vous avez accès à tous les outils de l'image ubuntu
apt update && apt install -y curl netcat

# Tester la connectivité
curl http://localhost:8080

# Voir les processus (si --target utilisé)
ps aux
```

#### Debug d'un Pod qui Crashe

```bash
# Créer une copie du pod avec debugging
microk8s kubectl debug mon-pod --copy-to=mon-pod-debug --container=debug -- sh

# Le pod copié ne démarre pas automatiquement les conteneurs
# Vous pouvez investiguer
```

#### Debug d'un Node

```bash
# Créer un pod debug sur un nœud
microk8s kubectl debug node/microk8s -it --image=ubuntu

# Vous avez accès au système de fichiers du nœud dans /host
ls /host/var/snap/microk8s
```

## Port Forwarding et Proxy

### Port Forward

**Rediriger un port local vers un pod** :

```bash
# Forwarder le port 8080 du pod vers le port local 8080
microk8s kubectl port-forward pod/mon-pod 8080:8080

# Port local différent
microk8s kubectl port-forward pod/mon-pod 9000:8080

# Accès depuis d'autres machines (bind 0.0.0.0)
microk8s kubectl port-forward --address 0.0.0.0 pod/mon-pod 8080:8080
```

**Utilisation avec Services** :

```bash
# Forward vers un service
microk8s kubectl port-forward service/mon-service 8080:80

# Forward vers un deployment
microk8s kubectl port-forward deployment/mon-app 8080:8080
```

**Cas d'usage** :
- Accéder à une application non exposée
- Tester sans créer d'Ingress
- Debug d'une base de données

**Exemple - Base de données** :

```bash
# Forwarder PostgreSQL
microk8s kubectl port-forward pod/postgres-0 5432:5432

# Dans un autre terminal, se connecter
psql -h localhost -p 5432 -U postgres
```

### Kubectl Proxy

**Créer un proxy vers l'API server** :

```bash
# Démarrer le proxy
microk8s kubectl proxy

# Par défaut : http://localhost:8001

# Spécifier le port
microk8s kubectl proxy --port=9000
```

**Accéder aux ressources via le proxy** :

```bash
# API version
curl http://localhost:8001/api/v1

# Pods du namespace default
curl http://localhost:8001/api/v1/namespaces/default/pods

# Logs d'un pod
curl http://localhost:8001/api/v1/namespaces/default/pods/mon-pod/log

# Métriques d'un pod
curl http://localhost:8001/apis/metrics.k8s.io/v1beta1/namespaces/default/pods/mon-pod
```

**Cas d'usage** :
- Accéder à l'API Kubernetes depuis des scripts
- Explorer l'API sans authentification complexe
- Dashboard Kubernetes

## Outils Tiers Essentiels

### K9s : Interface TUI Interactive

**K9s** est un outil terminal interactif pour gérer Kubernetes.

#### Installation

```bash
# Via snap
sudo snap install k9s

# Via wget (dernière version)
wget https://github.com/derailed/k9s/releases/download/v0.32.4/k9s_Linux_amd64.tar.gz
tar -xzf k9s_Linux_amd64.tar.gz
sudo mv k9s /usr/local/bin/

# Vérifier
k9s version
```

#### Configuration pour MicroK8s

```bash
# K9s utilise le kubeconfig par défaut
# Configurer pour MicroK8s
microk8s config > ~/.kube/config

# Ou lancer avec kubeconfig spécifique
k9s --kubeconfig ~/.kube/config
```

#### Utilisation

**Lancer K9s** :

```bash
k9s
```

**Navigation** :
- `:pods` - Voir les pods
- `:deployments` - Voir les deployments
- `:services` - Voir les services
- `:nodes` - Voir les nœuds
- `:<ressource>` - Aller à n'importe quelle ressource

**Raccourcis clavier** :
- `?` - Aide
- `/` - Filtrer
- `l` - Logs
- `d` - Describe
- `e` - Edit
- `Ctrl+d` - Delete
- `s` - Shell
- `:q` - Quitter

**Fonctionnalités** :
- Vue en temps réel
- Filtrage facile
- Logs en couleur
- Edit YAML intégré
- Multiple namespaces
- Recherche rapide

**Avantages** :
- Interface intuitive
- Pas besoin de taper de longues commandes
- Vue d'ensemble rapide
- Idéal pour l'exploration

### Stern : Logs Multiples en Temps Réel

**Stern** permet de suivre les logs de plusieurs pods simultanément.

#### Installation

```bash
# Via wget
wget https://github.com/stern/stern/releases/download/v1.28.0/stern_1.28.0_linux_amd64.tar.gz
tar -xzf stern_1.28.0_linux_amd64.tar.gz
sudo mv stern /usr/local/bin/

# Vérifier
stern --version
```

#### Utilisation

**Syntaxe de base** :

```bash
# Logs de tous les pods commençant par "nginx"
stern nginx

# Dans un namespace spécifique
stern nginx -n production

# Tous les namespaces
stern nginx --all-namespaces

# Pattern plus complexe
stern "^nginx-.*" --regex
```

**Options utiles** :

```bash
# Avec timestamps
stern nginx --timestamps

# Depuis une durée
stern nginx --since 15m

# Exclure certains conteneurs
stern nginx --exclude-container init

# Par label
stern -l app=nginx

# Contexte (lignes avant/après)
stern nginx --context 5
```

**Cas d'usage** :
- Suivre les logs d'un deployment complet (tous les replicas)
- Debugging d'applications distribuées
- Surveillance en temps réel

**Exemple** :

```bash
# Suivre tous les pods nginx en production
stern nginx -n production --timestamps

# Sortie :
# + nginx-abc123 › nginx
# nginx-abc123 nginx 2024-10-26 10:00:01 [INFO] Request received
# + nginx-def456 › nginx
# nginx-def456 nginx 2024-10-26 10:00:02 [INFO] Request received
```

### Kubectx et Kubens : Changement de Contexte Rapide

**Kubectx** : Changer de contexte Kubernetes facilement
**Kubens** : Changer de namespace par défaut

#### Installation

```bash
# Via git
sudo git clone https://github.com/ahmetb/kubectx /opt/kubectx
sudo ln -s /opt/kubectx/kubectx /usr/local/bin/kubectx
sudo ln -s /opt/kubectx/kubens /usr/local/bin/kubens

# Vérifier
kubectx --version
kubens --version
```

#### Utilisation de Kubectx

```bash
# Lister les contextes
kubectx

# Changer de contexte
kubectx microk8s

# Revenir au contexte précédent
kubectx -

# Renommer un contexte
kubectx prod=microk8s
```

#### Utilisation de Kubens

```bash
# Lister les namespaces
kubens

# Changer de namespace par défaut
kubens production

# Revenir au namespace précédent
kubens -

# Toutes les commandes kubectl utiliseront maintenant ce namespace
microk8s kubectl get pods  # pods de "production"
```

**Avantages** :
- Gain de temps énorme
- Évite les erreurs de namespace
- Navigation rapide entre environnements

### Kube-Capacity : Vue d'Ensemble des Ressources

**Voir l'utilisation et la capacité du cluster** :

#### Installation

```bash
wget https://github.com/robscott/kube-capacity/releases/download/v0.7.4/kube-capacity_0.7.4_Linux_x86_64.tar.gz
tar -xzf kube-capacity_0.7.4_Linux_x86_64.tar.gz
sudo mv kube-capacity /usr/local/bin/
```

#### Utilisation

```bash
# Vue d'ensemble
kube-capacity

# Par namespace
kube-capacity --namespace production

# Avec utilisation réelle
kube-capacity --util

# Format pod
kube-capacity --pod-count

# Sortir en JSON
kube-capacity -o json
```

**Sortie exemple** :

```
NODE         CPU REQUESTS   CPU LIMITS   MEMORY REQUESTS   MEMORY LIMITS
microk8s     2000m (50%)    4000m (100%) 4Gi (50%)        8Gi (100%)

NAMESPACE    POD            CPU REQUESTS   CPU LIMITS   MEMORY REQUESTS   MEMORY LIMITS
default      nginx-abc      100m           500m         128Mi             512Mi
production   api-def        500m           2000m        1Gi               4Gi
```

### Krew : Plugin Manager pour Kubectl

**Krew** permet d'installer des plugins kubectl facilement.

#### Installation

```bash
# Installation de Krew
(
  set -x; cd "$(mktemp -d)" &&
  OS="$(uname | tr '[:upper:]' '[:lower:]')" &&
  ARCH="$(uname -m | sed -e 's/x86_64/amd64/' -e 's/\(arm\)\(64\)\?.*/\1\2/' -e 's/aarch64$/arm64/')" &&
  KREW="krew-${OS}_${ARCH}" &&
  curl -fsSLO "https://github.com/kubernetes-sigs/krew/releases/latest/download/${KREW}.tar.gz" &&
  tar zxvf "${KREW}.tar.gz" &&
  ./"${KREW}" install krew
)

# Ajouter au PATH
export PATH="${KREW_ROOT:-$HOME/.krew}/bin:$PATH"

# Ajouter à ~/.bashrc pour permanence
echo 'export PATH="${KREW_ROOT:-$HOME/.krew}/bin:$PATH"' >> ~/.bashrc
```

#### Utilisation

```bash
# Lister les plugins disponibles
kubectl krew search

# Installer un plugin
kubectl krew install neat
kubectl krew install tree
kubectl krew install whoami

# Mettre à jour
kubectl krew upgrade

# Utiliser un plugin
kubectl neat get pod mon-pod -o yaml
```

**Plugins utiles** :

- **neat** : Nettoie le YAML (enlève les champs générés)
- **tree** : Vue arborescente des ressources
- **whoami** : Voir votre identité Kubernetes
- **view-allocations** : Vue des allocations de ressources
- **sniff** : Capturer le trafic réseau
- **get-all** : Vraiment TOUTES les ressources

## Outils d'Analyse de Configuration

### Kubectl-Neat : Nettoyage de YAML

**Enlève les champs ajoutés par Kubernetes** :

```bash
# Installer via krew
kubectl krew install neat

# Utilisation
kubectl get pod mon-pod -o yaml | kubectl neat

# Ou directement
kubectl neat get pod mon-pod -o yaml
```

**Avant** :

```yaml
apiVersion: v1
kind: Pod
metadata:
  creationTimestamp: "2024-10-26T10:00:00Z"
  managedFields:
    - apiVersion: v1
      # ... beaucoup de lignes générées ...
  resourceVersion: "12345"
  uid: abc-123-def
spec:
  # ...
```

**Après** :

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: mon-pod
spec:
  # ... seulement ce que vous avez défini
```

### Kubectl-Tree : Vue Hiérarchique

**Voir les relations entre ressources** :

```bash
# Installer via krew
kubectl krew install tree

# Utilisation
kubectl tree deployment mon-app

# Sortie :
# NAMESPACE  NAME                           READY  REASON  AGE
# default    Deployment/mon-app             -              5m
# default    └─ReplicaSet/mon-app-abc123    -              5m
# default      └─Pod/mon-app-abc123-xyz     True           5m
```

### Kubeval : Validation de YAML

**Valider vos manifests avant de les appliquer** :

#### Installation

```bash
wget https://github.com/instrumenta/kubeval/releases/download/v0.16.1/kubeval-linux-amd64.tar.gz
tar -xzf kubeval-linux-amd64.tar.gz
sudo mv kubeval /usr/local/bin/
```

#### Utilisation

```bash
# Valider un fichier
kubeval deployment.yaml

# Valider plusieurs fichiers
kubeval *.yaml

# Valider un répertoire
kubeval manifests/

# Avec version Kubernetes spécifique
kubeval --kubernetes-version 1.28.0 deployment.yaml
```

**Sortie** :

```
PASS - deployment.yaml contains a valid Deployment (nginx)
WARN - service.yaml contains an invalid Service (nginx)
---> spec.type: Invalid value: "LoadBalancerr" (typo!)
```

## Outils de Monitoring et Métriques

### Metrics Server : Métriques de Base

**Déjà couvert dans les sections précédentes, rappel** :

```bash
# Activer
microk8s enable metrics-server

# Utiliser
microk8s kubectl top nodes
microk8s kubectl top pods --all-namespaces
```

### Prometheus et Grafana

**Monitoring avancé avec addon MicroK8s** :

```bash
# Activer
microk8s enable prometheus

# Accéder à Grafana
microk8s kubectl port-forward -n prometheus service/prometheus-grafana 3000:80

# Ouvrir : http://localhost:3000
# Login : admin / prom-operator
```

**Dashboards pré-installés** :
- Utilisation des nœuds
- Performance des pods
- État du cluster
- Utilisation réseau
- Stockage

**Requêtes Prometheus utiles** :

```promql
# Utilisation CPU par pod
rate(container_cpu_usage_seconds_total[5m])

# Utilisation mémoire par namespace
sum(container_memory_usage_bytes) by (namespace)

# Nombre de restarts de pods
kube_pod_container_status_restarts_total

# Latence des requêtes HTTP
histogram_quantile(0.95, rate(http_request_duration_seconds_bucket[5m]))
```

### Kube-State-Metrics

**Déjà inclus avec Prometheus** :

Expose les métriques sur l'état des ressources Kubernetes :
- Nombre de pods par état
- Capacité des nodes
- État des deployments
- Status des PVC

```bash
# Voir les métriques
microk8s kubectl port-forward -n prometheus svc/prometheus-kube-state-metrics 8080:8080

# Accéder
curl http://localhost:8080/metrics
```

## Outils de Debug Réseau

### Netshoot : Swiss Army Knife Réseau

**Image Docker avec tous les outils réseau** :

```bash
# Lancer un pod de debug réseau
microk8s kubectl run netshoot --rm -it --image=nicolaka/netshoot -- bash

# Dans le pod, vous avez accès à :
# - curl, wget
# - ping, traceroute, mtr
# - nslookup, dig, host
# - netstat, ss, lsof
# - tcpdump, iperf
# - etc.
```

**Exemples d'utilisation** :

```bash
# DNS lookup
nslookup kubernetes.default

# Tester la connectivité vers un service
curl http://mon-service.default.svc.cluster.local

# Voir les connexions
netstat -tuln

# Capturer le trafic
tcpdump -i any port 80

# Test de bande passante
iperf3 -s  # Sur un pod
iperf3 -c <ip-pod> # Sur un autre
```

### Kubectl Sniff : Capture de Trafic

**Capturer le trafic réseau d'un pod** :

```bash
# Installer via krew
kubectl krew install sniff

# Capturer le trafic d'un pod
kubectl sniff mon-pod

# Avec filtre
kubectl sniff mon-pod -f "port 80"

# Sauvegarder dans un fichier
kubectl sniff mon-pod -o capture.pcap

# Analyser avec Wireshark
wireshark capture.pcap
```

### Service Mesh Debugging (Istio, Linkerd)

Si vous utilisez un service mesh :

**Istio** :

```bash
# Installer istioctl
curl -L https://istio.io/downloadIstio | sh -
sudo mv istio-*/bin/istioctl /usr/local/bin/

# Analyser la configuration
istioctl analyze

# Voir la config d'un pod
istioctl proxy-config cluster mon-pod

# Logs du proxy Envoy
kubectl logs mon-pod -c istio-proxy
```

## Outils d'Analyse de Sécurité

### Kubesec : Analyse de Sécurité de Manifests

```bash
# Installation
wget https://github.com/controlplaneio/kubesec/releases/download/v2.13.0/kubesec_linux_amd64.tar.gz
tar -xzf kubesec_linux_amd64.tar.gz
sudo mv kubesec /usr/local/bin/

# Analyse
kubesec scan deployment.yaml

# Sortie :
# [
#   {
#     "object": "Deployment/nginx",
#     "valid": true,
#     "message": "Passed with a score of 3 points",
#     "score": 3,
#     "scoring": {
#       "passed": [
#         {
#           "selector": "containers[] .securityContext .readOnlyRootFilesystem == true",
#           "reason": "Immutable root filesystem"
#         }
#       ],
#       "advise": [
#         {
#           "selector": "containers[] .securityContext .runAsNonRoot == true",
#           "reason": "Force running as non-root user"
#         }
#       ]
#     }
#   }
# ]
```

### Trivy : Scan de Vulnérabilités

**Scanner les images pour des vulnérabilités** :

```bash
# Installation
sudo snap install trivy

# Scanner une image
trivy image nginx:latest

# Scanner toutes les images du cluster
kubectl get pods --all-namespaces -o jsonpath="{.items[*].spec.containers[*].image}" | tr -s '[[:space:]]' '\n' | sort | uniq | xargs -I {} trivy image {}

# Scanner un fichier Kubernetes
trivy config deployment.yaml
```

### Polaris : Audit de Best Practices

```bash
# Installation
kubectl apply -f https://github.com/FairwindsOps/polaris/releases/latest/download/dashboard.yaml

# Accéder au dashboard
kubectl port-forward -n polaris service/polaris-dashboard 8080:80

# Ouvrir : http://localhost:8080
```

**Vérifications** :
- Sécurité (runAsNonRoot, capabilities, etc.)
- Efficacité (resources requests/limits)
- Fiabilité (probes, replicas)

## Outils de Performance

### Kube-Bench : Vérification CIS Benchmark

**Vérifier si votre cluster respecte les bonnes pratiques CIS** :

```bash
# Exécuter kube-bench
kubectl apply -f https://raw.githubusercontent.com/aquasecurity/kube-bench/main/job.yaml

# Voir les résultats
kubectl logs job/kube-bench

# Nettoyer
kubectl delete job kube-bench
```

### Kubectl-Profiling

**Profiler les performances** :

```bash
# Installer via krew
kubectl krew install profefe

# Profiler un pod
kubectl profefe pod mon-pod
```

### Load Testing avec K6

**Tester la performance d'une application** :

```bash
# Script k6
cat > loadtest.js <<EOF
import http from 'k6/http';
import { check } from 'k6';

export let options = {
  vus: 10,
  duration: '30s',
};

export default function() {
  let res = http.get('http://mon-service.default.svc.cluster.local');
  check(res, {
    'status is 200': (r) => r.status === 200,
  });
}
EOF

# Exécuter dans un pod
kubectl run k6 --rm -it --image=loadimpact/k6 -- run - <loadtest.js
```

## Scripts et Outils Personnalisés

### Script d'Analyse Complète

```bash
#!/bin/bash
# cluster-health-check.sh

echo "╔══════════════════════════════════════════════════════════╗"
echo "║         ANALYSE COMPLÈTE DU CLUSTER                      ║"
echo "╚══════════════════════════════════════════════════════════╝"

# 1. Version et État
echo ""
echo "=== VERSION ET ÉTAT ==="
kubectl version --short
kubectl cluster-info

# 2. Nœuds
echo ""
echo "=== NŒUDS ==="
kubectl get nodes -o wide
kubectl top nodes 2>/dev/null || echo "Metrics server non disponible"

# 3. Namespaces
echo ""
echo "=== NAMESPACES ==="
kubectl get namespaces

# 4. Pods Problématiques
echo ""
echo "=== PODS AVEC PROBLÈMES ==="
kubectl get pods --all-namespaces | grep -v "Running\|Completed" | grep -v NAME || echo "Aucun problème"

# 5. Utilisation Ressources
echo ""
echo "=== TOP PODS (CPU) ==="
kubectl top pods --all-namespaces --sort-by=cpu 2>/dev/null | head -6

echo ""
echo "=== TOP PODS (MEMORY) ==="
kubectl top pods --all-namespaces --sort-by=memory 2>/dev/null | head -6

# 6. PVC
echo ""
echo "=== PERSISTENTVOLUMECLAIMS ==="
kubectl get pvc --all-namespaces

# 7. Services
echo ""
echo "=== SERVICES ==="
kubectl get svc --all-namespaces

# 8. Événements Récents
echo ""
echo "=== DERNIERS ÉVÉNEMENTS ==="
kubectl get events --all-namespaces --sort-by='.lastTimestamp' | tail -10

# 9. Certificats
echo ""
echo "=== EXPIRATION CERTIFICATS ==="
for cert in /var/snap/microk8s/current/certs/*.crt; do
    echo "$(basename $cert): $(openssl x509 -in $cert -noout -enddate 2>/dev/null | cut -d'=' -f2)"
done

# 10. Espace Disque
echo ""
echo "=== ESPACE DISQUE ==="
df -h /var/snap/microk8s | tail -1

echo ""
echo "═══════════════════════════════════════════════════════════"
echo "Analyse terminée - $(date)"
```

### Script de Collecte de Logs

```bash
#!/bin/bash
# collect-logs.sh

NAMESPACE=${1:-default}
OUTPUT_DIR="logs-$(date +%Y%m%d-%H%M%S)"

echo "Collecte des logs du namespace: $NAMESPACE"
mkdir -p "$OUTPUT_DIR"

# Logs de tous les pods
for pod in $(kubectl get pods -n $NAMESPACE -o name); do
    pod_name=$(basename $pod)
    echo "Collecte logs de $pod_name..."

    # Logs actuels
    kubectl logs -n $NAMESPACE $pod_name --all-containers > "$OUTPUT_DIR/${pod_name}-current.log" 2>&1

    # Logs précédents (si restart)
    kubectl logs -n $NAMESPACE $pod_name --all-containers --previous > "$OUTPUT_DIR/${pod_name}-previous.log" 2>&1

    # Describe
    kubectl describe -n $NAMESPACE $pod > "$OUTPUT_DIR/${pod_name}-describe.txt" 2>&1
done

# Événements
kubectl get events -n $NAMESPACE --sort-by='.lastTimestamp' > "$OUTPUT_DIR/events.txt" 2>&1

# Archive
tar -czf "$OUTPUT_DIR.tar.gz" "$OUTPUT_DIR"
rm -rf "$OUTPUT_DIR"

echo "Logs collectés dans : $OUTPUT_DIR.tar.gz"
```

### Script de Test de Connectivité

```bash
#!/bin/bash
# connectivity-test.sh

echo "=== TEST DE CONNECTIVITÉ ==="

# Pod de test
POD_NAME="connectivity-test-$$"

echo "Création du pod de test..."
kubectl run $POD_NAME --image=nicolaka/netshoot --restart=Never -- sleep 3600

echo "Attente du pod..."
kubectl wait --for=condition=ready pod/$POD_NAME --timeout=60s

# Tests DNS
echo ""
echo "=== TEST DNS ==="
kubectl exec $POD_NAME -- nslookup kubernetes.default
kubectl exec $POD_NAME -- nslookup google.com

# Tests Connectivité Interne
echo ""
echo "=== TEST CONNECTIVITÉ INTERNE ==="
API_IP=$(kubectl get svc kubernetes -o jsonpath='{.spec.clusterIP}')
kubectl exec $POD_NAME -- curl -sk https://$API_IP/healthz

# Tests Connectivité Externe
echo ""
echo "=== TEST CONNECTIVITÉ EXTERNE ==="
kubectl exec $POD_NAME -- curl -s -o /dev/null -w "HTTP Code: %{http_code}\n" https://www.google.com

# Nettoyage
echo ""
echo "Nettoyage..."
kubectl delete pod $POD_NAME

echo ""
echo "Tests terminés!"
```

## Bonnes Pratiques de Diagnostic

### 1. Approche Systématique

**Suivre un workflow** :

```
1. IDENTIFIER
   ├── Quel est le symptôme ?
   ├── Depuis quand ?
   └── Quelle est la portée ?

2. ISOLER
   ├── Nœud, Pod, ou Réseau ?
   ├── Tous les pods ou un seul ?
   └── Reproductible ?

3. COLLECTER
   ├── Logs (kubectl logs)
   ├── État (kubectl describe)
   ├── Métriques (kubectl top)
   └── Événements (kubectl get events)

4. ANALYSER
   ├── Corréler les informations
   ├── Chercher des patterns
   └── Tester des hypothèses

5. RÉSOUDRE
   ├── Appliquer le fix
   ├── Vérifier
   └── Documenter
```

### 2. Utiliser les Bons Outils

**Choisir l'outil selon le problème** :

| Problème | Outil Recommandé |
|----------|------------------|
| Vue d'ensemble | k9s, kubectl get all |
| Logs multiples | stern |
| Debug réseau | netshoot, kubectl sniff |
| Performance | kubectl top, prometheus |
| Configuration | kubectl neat, kubeval |
| Sécurité | trivy, kubesec, polaris |
| Navigation | kubectx, kubens |

### 3. Automatiser les Vérifications

**Scripts de monitoring** :

```bash
# Créer des scripts pour tâches répétitives
# - Health check quotidien
# - Collecte de logs
# - Tests de connectivité
# - Validation de configuration
```

### 4. Garder des Traces

**Logging structuré** :

```bash
# Toujours logger vos investigations
echo "[$(date)] Début investigation: Pod en CrashLoop" >> investigations.log
kubectl describe pod mon-pod >> investigations.log
kubectl logs mon-pod --previous >> investigations.log
echo "[$(date)] Cause trouvée: OOMKilled" >> investigations.log
```

### 5. Construire une Boîte à Outils

**Installer et configurer** :

```bash
# Liste minimale recommandée
- kubectl (évidemment)
- k9s (interface interactive)
- stern (logs multiples)
- kubectx/kubens (navigation)
- jq (parsing JSON)
- yq (parsing YAML)
```

### 6. Tester dans un Environnement Sûr

**Toujours tester les commandes dans dev/staging** :

```bash
# JAMAIS directement en prod
kubectl delete pod --all  # 💀 DANGER

# Toujours avec dry-run en prod
kubectl delete pod mon-pod --dry-run=client
```

### 7. Documenter les Solutions

**Créer un runbook** :

```markdown
# Runbook: Pod CrashLoopBackOff

## Symptômes
- Pod redémarre constamment
- Status: CrashLoopBackOff

## Diagnostic
1. `kubectl describe pod <name>`
2. `kubectl logs <name> --previous`
3. Vérifier les ressources (CPU/Mem)

## Solutions Courantes
1. OOMKilled → Augmenter memory limit
2. Erreur application → Corriger le code
3. ConfigMap manquant → Créer/vérifier ConfigMap

## Commandes
```bash
# Voir la cause
kubectl describe pod <name> | grep -A 10 "Last State"

# Logs du crash
kubectl logs <name> --previous
```

## Checklist d'Outils

### Essentiels (Installation Obligatoire)

- [ ] kubectl (inclus avec MicroK8s)
- [ ] k9s (interface interactive)
- [ ] stern (logs multiples)
- [ ] jq (parsing JSON)

### Recommandés (Très Utiles)

- [ ] kubectx/kubens (navigation)
- [ ] kubectl neat (nettoyage YAML)
- [ ] netshoot (debug réseau)
- [ ] kubeval (validation)

### Avancés (Selon Besoins)

- [ ] krew (plugin manager)
- [ ] trivy (sécurité)
- [ ] istioctl (si service mesh)
- [ ] helm (si utilisé)

### Optionnels (Nice to Have)

- [ ] kube-capacity
- [ ] polaris
- [ ] kubectl tree
- [ ] kubectl sniff

## Résumé

### Commandes Kubectl Avancées

```bash
# Output formats
kubectl get pods -o yaml
kubectl get pods -o jsonpath='{.items[*].metadata.name}'
kubectl get pods -o custom-columns=NAME:.metadata.name,IP:.status.podIP

# Watch
kubectl get pods --watch

# Logs avancés
kubectl logs -f pod --all-containers --previous --timestamps

# Debug
kubectl debug pod -it --image=ubuntu
kubectl debug node/node-name -it

# Port forward
kubectl port-forward pod/name 8080:8080
kubectl port-forward svc/name 8080:80

# Exec
kubectl exec -it pod -- bash
kubectl exec pod -- command
```

### Outils Tiers Essentiels

| Outil | Usage Principal | Installation |
|-------|-----------------|--------------|
| k9s | Interface interactive | `snap install k9s` |
| stern | Logs multiples | Binary GitHub |
| kubectx | Changer contexte | Script GitHub |
| kubens | Changer namespace | Script GitHub |
| netshoot | Debug réseau | Image Docker |
| trivy | Scan sécurité | `snap install trivy` |

### Workflow de Debug Recommandé

```
1. Vue d'ensemble (k9s ou kubectl get all)
2. Identifier le problème (describe, logs)
3. Isoler (exec, debug, port-forward)
4. Collecter infos (logs, metrics, events)
5. Analyser (corrélation, patterns)
6. Tester solution (dry-run, staging)
7. Appliquer (production)
8. Documenter (runbook, logs)
```

### Points Clés

1. **Commencer simple** - kubectl describe et logs résolvent 80% des problèmes
2. **Outils interactifs** - k9s pour exploration, stern pour logs
3. **Network debugging** - netshoot est votre ami
4. **Automatiser** - Scripts pour tâches répétitives
5. **Documenter** - Runbooks pour problèmes courants
6. **Tester avant prod** - Toujours dry-run
7. **Boîte à outils** - Installer les essentiels dès le début
8. **Approche méthodique** - Suivre un workflow, ne pas improviser

---


⏭️ [Cas d'Usage Pratiques](/24-cas-dusage-pratiques/README.md)
