🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 23.5 Problèmes Réseau Courants

## Introduction

Le réseau est souvent la source la plus fréquente et la plus frustrante de problèmes dans Kubernetes. Un pod qui ne peut pas communiquer avec un autre, un service inaccessible, un DNS qui ne résout pas les noms... Ces problèmes peuvent sembler mystérieux au début, mais avec une compréhension de base et les bons outils, vous pouvez les diagnostiquer efficacement.

> **Pour les débutants** : Le réseau dans Kubernetes peut sembler complexe, mais vous n'avez pas besoin de tout comprendre pour déboguer la plupart des problèmes. Cette section vous donnera les connaissances essentielles et les commandes pratiques pour résoudre les 95% des problèmes réseau les plus courants.

## Comprendre le Réseau Kubernetes (Simplifié)

Avant de plonger dans les problèmes, comprenons rapidement comment fonctionne le réseau dans Kubernetes.

### Les Trois Niveaux de Communication

Dans Kubernetes, il existe trois niveaux de communication réseau :

#### 1. Communication Conteneur-à-Conteneur (dans un même pod)

**Concept** : Les conteneurs d'un même pod partagent le même espace réseau.

**Conséquence** :
- Ils peuvent communiquer via `localhost`
- Ils partagent la même adresse IP
- Ils doivent utiliser des ports différents

**Exemple** :
```yaml
# Un pod avec deux conteneurs
apiVersion: v1
kind: Pod
metadata:
  name: mon-pod
spec:
  containers:
  - name: app
    image: mon-app:1.0
    ports:
    - containerPort: 8080
  - name: sidecar
    image: mon-sidecar:1.0
    ports:
    - containerPort: 9090
```

Dans ce pod, `app` peut accéder à `sidecar` via `localhost:9090` et vice-versa.

#### 2. Communication Pod-à-Pod

**Concept** : Chaque pod a sa propre adresse IP unique dans le cluster.

**Conséquence** :
- Les pods peuvent communiquer directement entre eux via leurs IPs
- Mais les IPs des pods changent quand ils redémarrent
- C'est pourquoi on utilise des Services

**Exemple** :
```bash
# Pod A (IP: 10.1.1.5) peut contacter Pod B (IP: 10.1.1.8) directement
curl http://10.1.1.8:8080
```

#### 3. Communication Service-à-Service

**Concept** : Les Services fournissent une adresse stable (nom DNS et IP virtuelle) devant un groupe de pods.

**Conséquence** :
- Vous utilisez le nom du service au lieu des IPs de pods
- Le service route automatiquement vers les pods disponibles
- C'est la méthode recommandée pour la communication

**Exemple** :
```bash
# Au lieu de curl http://10.1.1.8:8080
# Vous faites curl http://mon-service:8080
```

### Le Rôle de CoreDNS

**CoreDNS** est le serveur DNS interne de Kubernetes. Il permet de :
- Résoudre les noms de services en adresses IP
- Utiliser des noms simples dans le même namespace : `mon-service`
- Utiliser des noms complets : `mon-service.mon-namespace.svc.cluster.local`

**Activation dans MicroK8s** :
```bash
microk8s enable dns
```

**Vérification** :
```bash
microk8s kubectl get pods -n kube-system | grep coredns
```

### Le CNI (Container Network Interface)

Le CNI est le plugin réseau qui gère la communication entre les pods. MicroK8s utilise par défaut **Calico**.

**Vérification** :
```bash
microk8s kubectl get pods -n kube-system | grep calico
```

## Problème 1 : Résolution DNS qui ne Fonctionne Pas

C'est LE problème réseau le plus courant dans Kubernetes.

### Symptômes

- Erreurs dans les logs : `unable to resolve host`, `no such host`, `DNS resolution failed`
- Votre application ne peut pas se connecter à d'autres services par leur nom
- `curl http://mon-service` échoue avec une erreur de résolution

### Diagnostic

#### Étape 1 : Vérifier que CoreDNS est en cours d'exécution

```bash
# Vérifier les pods CoreDNS
microk8s kubectl get pods -n kube-system -l k8s-app=kube-dns

# Sortie attendue :
# NAME                       READY   STATUS    RESTARTS   AGE
# coredns-7f9c69c78c-xxxxx   1/1     Running   0          5d
```

**Si CoreDNS n'est pas Running** :
```bash
# Voir les détails
microk8s kubectl describe pod -n kube-system -l k8s-app=kube-dns

# Voir les logs
microk8s kubectl logs -n kube-system -l k8s-app=kube-dns
```

**Si CoreDNS n'existe pas** :
```bash
# L'activer
microk8s enable dns

# Attendre qu'il démarre
microk8s kubectl wait --for=condition=ready pod -n kube-system -l k8s-app=kube-dns --timeout=60s
```

#### Étape 2 : Tester la résolution DNS depuis un pod

Créez un pod de test pour vérifier le DNS :

```bash
# Créer un pod temporaire avec des outils réseau
microk8s kubectl run test-dns --image=busybox:1.28 --rm -it --restart=Never -- sh
```

Une fois dans le pod, testez :

```bash
# Tester la résolution d'un service Kubernetes interne
nslookup kubernetes.default

# Sortie attendue :
# Server:    10.152.183.10
# Address 1: 10.152.183.10 kube-dns.kube-system.svc.cluster.local
#
# Name:      kubernetes.default
# Address 1: 10.152.183.1 kubernetes.default.svc.cluster.local

# Tester la résolution d'un nom externe
nslookup google.com

# Tester votre service (si vous en avez un)
nslookup mon-service
```

**Interprétation** :

- ✅ **Si `kubernetes.default` fonctionne** : Le DNS interne fonctionne
- ✅ **Si `google.com` fonctionne** : La résolution externe fonctionne
- ✅ **Si votre service fonctionne** : Tout est OK
- ❌ **Si rien ne fonctionne** : Problème de configuration DNS (voir ci-dessous)
- ❌ **Si seulement l'externe fonctionne** : Problème avec CoreDNS

#### Étape 3 : Vérifier la configuration DNS du pod

```bash
# Vérifier le fichier resolv.conf d'un pod
microk8s kubectl exec mon-pod -- cat /etc/resolv.conf

# Sortie attendue :
# nameserver 10.152.183.10
# search default.svc.cluster.local svc.cluster.local cluster.local
# options ndots:5
```

**Vérifications** :
- Le `nameserver` devrait pointer vers l'IP du service CoreDNS (généralement 10.152.183.10)
- La section `search` devrait contenir les domaines Kubernetes

**Si le nameserver est incorrect** :
```bash
# Vérifier l'IP du service DNS
microk8s kubectl get service -n kube-system kube-dns

# Devrait afficher quelque chose comme :
# NAME       TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)         AGE
# kube-dns   ClusterIP   10.152.183.10   <none>        53/UDP,53/TCP   5d
```

### Solutions Courantes

#### Solution 1 : Redémarrer CoreDNS

Souvent, un simple redémarrage résout le problème :

```bash
# Supprimer les pods CoreDNS (ils redémarreront automatiquement)
microk8s kubectl delete pod -n kube-system -l k8s-app=kube-dns

# Attendre qu'ils redémarrent
microk8s kubectl wait --for=condition=ready pod -n kube-system -l k8s-app=kube-dns --timeout=60s
```

#### Solution 2 : Utiliser des Noms Complets (FQDN)

Si les noms courts ne fonctionnent pas, utilisez le nom complet :

**Au lieu de** :
```bash
curl http://mon-service
```

**Utilisez** :
```bash
curl http://mon-service.mon-namespace.svc.cluster.local
```

**Format FQDN** : `<nom-service>.<namespace>.svc.cluster.local`

#### Solution 3 : Vérifier la Configuration du Service

Le service existe-t-il vraiment ?

```bash
# Lister les services
microk8s kubectl get services

# Vérifier un service spécifique
microk8s kubectl describe service mon-service
```

#### Solution 4 : Problème de dnsPolicy

Vérifiez la politique DNS du pod :

```bash
microk8s kubectl get pod mon-pod -o jsonpath='{.spec.dnsPolicy}'
```

**Valeur attendue** : `ClusterFirst` (politique par défaut)

Si la valeur est différente, modifiez votre deployment :

```yaml
spec:
  dnsPolicy: ClusterFirst  # Ajouter cette ligne
  containers:
  - name: mon-app
    image: mon-image
```

## Problème 2 : Impossible de Contacter un Service

### Symptômes

- `connection refused` lors d'une tentative de connexion à un service
- `timeout` lors d'une tentative de connexion
- L'application ne peut pas atteindre un autre service

### Diagnostic

#### Étape 1 : Vérifier que le Service Existe

```bash
# Lister les services
microk8s kubectl get services -n mon-namespace

# Détails du service
microk8s kubectl describe service mon-service -n mon-namespace
```

**Ce qu'il faut vérifier** :
- Le service existe-t-il ?
- Dans le bon namespace ?
- A-t-il une CLUSTER-IP ?
- Les ports sont-ils corrects ?

#### Étape 2 : Vérifier les Endpoints

Un service sans endpoints ne routera nulle part !

```bash
# Voir les endpoints du service
microk8s kubectl get endpoints mon-service -n mon-namespace

# Ou dans la description du service
microk8s kubectl describe service mon-service -n mon-namespace
```

**Sortie attendue** :
```
Name:         mon-service
Namespace:    default
Endpoints:    10.1.1.5:8080,10.1.1.8:8080,10.1.1.12:8080
```

**Si Endpoints est vide ou dit `<none>`** :
- Aucun pod ne correspond au sélecteur du service
- Ou les pods correspondants ne sont pas en état `Ready`

**Vérification des sélecteurs** :

```bash
# Voir le sélecteur du service
microk8s kubectl get service mon-service -o jsonpath='{.spec.selector}'

# Exemple de sortie : {"app":"nginx"}

# Vérifier si des pods ont ce label
microk8s kubectl get pods -l app=nginx
```

**Solution** : Corriger les labels du pod ou du service pour qu'ils correspondent.

#### Étape 3 : Tester la Connectivité Directe

Testez depuis un autre pod :

```bash
# Créer un pod de test avec des outils réseau
microk8s kubectl run netshoot --rm -it --image=nicolaka/netshoot -- bash
```

Une fois dans le pod :

```bash
# Tester avec curl
curl http://mon-service:8080

# Tester avec telnet
telnet mon-service 8080

# Tester avec netcat
nc -zv mon-service 8080

# Ping (fonctionne rarement car ICMP n'est souvent pas routé)
ping mon-service
```

**Interprétation** :

- **Connection refused** : Le service est joignable mais aucun pod n'écoute sur ce port
- **Timeout** : Le service n'est pas joignable (problème réseau ou firewall)
- **No route to host** : Problème de routage réseau
- **Name resolution failed** : Problème DNS (voir section précédente)

#### Étape 4 : Vérifier que les Pods Écoutent sur le Bon Port

```bash
# Trouver un pod du service
POD_NAME=$(microk8s kubectl get pods -l app=nginx -o jsonpath='{.items[0].metadata.name}')

# Vérifier les ports ouverts
microk8s kubectl exec $POD_NAME -- netstat -tulpn

# Ou avec ss
microk8s kubectl exec $POD_NAME -- ss -tulpn
```

**Vérifiez** : Le port que vous essayez d'atteindre est-il bien ouvert ?

### Solutions Courantes

#### Solution 1 : Corriger le Sélecteur du Service

Si le service n'a pas d'endpoints, vérifiez que les labels correspondent :

**Service** :
```yaml
apiVersion: v1
kind: Service
metadata:
  name: mon-service
spec:
  selector:
    app: nginx  # Ce label doit correspondre aux pods
  ports:
  - port: 80
    targetPort: 8080
```

**Pod/Deployment** :
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
spec:
  template:
    metadata:
      labels:
        app: nginx  # Doit correspondre au sélecteur du service
    spec:
      containers:
      - name: nginx
        image: nginx:1.21
```

#### Solution 2 : Corriger les Ports

Vérifiez que les ports sont correctement configurés :

```yaml
spec:
  ports:
  - port: 80           # Port exposé par le service
    targetPort: 8080   # Port sur lequel le conteneur écoute
    protocol: TCP
```

- **port** : Le port sur lequel le service écoute
- **targetPort** : Le port sur lequel le conteneur écoute réellement

**Vérification rapide** :
```bash
# Voir la configuration du service
microk8s kubectl get service mon-service -o yaml
```

#### Solution 3 : Vérifier les Network Policies

Les Network Policies peuvent bloquer le trafic :

```bash
# Lister les Network Policies
microk8s kubectl get networkpolicies -n mon-namespace

# Détails d'une policy
microk8s kubectl describe networkpolicy ma-policy -n mon-namespace
```

**Si une policy existe** : Vérifiez qu'elle autorise le trafic que vous essayez d'effectuer (voir section Network Policies plus loin).

#### Solution 4 : Redémarrer les Pods

Parfois, un simple redémarrage résout le problème :

```bash
# Redémarrer le deployment
microk8s kubectl rollout restart deployment mon-deployment -n mon-namespace
```

## Problème 3 : Connectivité Pod-à-Pod Directe

### Symptômes

- Impossible de contacter directement un pod via son IP
- Timeouts lors de communications entre pods
- Erreurs de réseau aléatoires

### Diagnostic

#### Étape 1 : Vérifier que les Pods Sont sur le Même Cluster

```bash
# Lister tous les pods avec leurs IPs
microk8s kubectl get pods -o wide --all-namespaces
```

**Vérifications** :
- Les pods sont-ils tous sur des nœuds du cluster ?
- Ont-ils tous des adresses IP ?
- Les IPs sont-elles dans le bon range (généralement 10.x.x.x) ?

#### Étape 2 : Tester la Connectivité Directe

```bash
# Obtenir l'IP d'un pod cible
TARGET_IP=$(microk8s kubectl get pod target-pod -o jsonpath='{.status.podIP}')

# Depuis un autre pod, tester la connectivité
microk8s kubectl run test --rm -it --image=busybox -- ping -c 3 $TARGET_IP
```

#### Étape 3 : Vérifier le Plugin CNI (Calico)

```bash
# Vérifier que Calico est en cours d'exécution
microk8s kubectl get pods -n kube-system -l k8s-app=calico-node

# Logs de Calico
microk8s kubectl logs -n kube-system -l k8s-app=calico-node --tail=100
```

### Solutions Courantes

#### Solution 1 : Redémarrer Calico

```bash
# Redémarrer tous les pods Calico
microk8s kubectl delete pod -n kube-system -l k8s-app=calico-node

# Attendre qu'ils redémarrent
microk8s kubectl wait --for=condition=ready pod -n kube-system -l k8s-app=calico-node --timeout=120s
```

#### Solution 2 : Vérifier les Routes Réseau

Sur le nœud MicroK8s :

```bash
# Voir les routes
ip route show

# Voir les interfaces
ip addr show
```

**Note** : C'est plus avancé et rarement nécessaire pour des problèmes simples.

#### Solution 3 : Utiliser des Services

Au lieu de la communication pod-à-pod directe (fragile car les IPs changent), utilisez toujours des Services :

❌ **Mauvais** :
```python
# Connexion directe à l'IP du pod
response = requests.get("http://10.1.1.5:8080/api")
```

✅ **Bon** :
```python
# Connexion via le service
response = requests.get("http://mon-service:8080/api")
```

## Problème 4 : Ingress qui ne Fonctionne Pas

### Symptômes

- Impossible d'accéder à l'application depuis l'extérieur du cluster
- Erreur 404, 502, 503 depuis le navigateur
- Timeout lors de l'accès à l'URL externe

### Diagnostic

#### Étape 1 : Vérifier que l'Ingress Controller est Installé

```bash
# Vérifier que l'addon ingress est activé
microk8s status | grep ingress

# Vérifier les pods du controller
microk8s kubectl get pods -n ingress

# Logs du controller
microk8s kubectl logs -n ingress -l app.kubernetes.io/name=ingress-nginx
```

**Si ingress n'est pas activé** :
```bash
microk8s enable ingress
```

#### Étape 2 : Vérifier la Ressource Ingress

```bash
# Lister les ingress
microk8s kubectl get ingress -n mon-namespace

# Détails
microk8s kubectl describe ingress mon-ingress -n mon-namespace
```

**Ce qu'il faut vérifier** :
- La ressource Ingress existe-t-elle ?
- A-t-elle une ADDRESS assignée ?
- Les règles pointent-elles vers les bons services ?

**Exemple de sortie** :
```
Name:             mon-ingress
Namespace:        default
Address:          192.168.1.100
Rules:
  Host              Path  Backends
  ----              ----  --------
  mon-app.local     /     mon-service:80 (10.1.1.5:8080,10.1.1.8:8080)
```

#### Étape 3 : Vérifier le Service Backend

```bash
# Le service référencé par l'ingress existe-t-il ?
microk8s kubectl get service mon-service -n mon-namespace

# A-t-il des endpoints ?
microk8s kubectl get endpoints mon-service -n mon-namespace
```

#### Étape 4 : Tester Depuis l'Intérieur du Cluster

```bash
# Créer un pod de test
microk8s kubectl run test --rm -it --image=curlimages/curl -- sh

# Tester l'accès direct au service
curl http://mon-service:80

# Tester via l'ingress (utiliser l'IP de l'ingress controller)
curl -H "Host: mon-app.local" http://192.168.1.100/
```

#### Étape 5 : Vérifier la Configuration DNS Externe

Si vous utilisez un nom de domaine externe :

```bash
# Depuis votre machine (pas dans le cluster)
nslookup mon-app.example.com

# Vérifier que l'IP correspond à celle de votre cluster
```

### Solutions Courantes

#### Solution 1 : Corriger la Configuration Ingress

**Exemple de configuration correcte** :

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: mon-ingress
  namespace: default
spec:
  ingressClassName: nginx  # Important pour MicroK8s
  rules:
  - host: mon-app.local
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: mon-service  # Doit exister
            port:
              number: 80       # Port du service, pas du pod
```

**Points critiques** :
- `ingressClassName: nginx` est nécessaire
- `service.name` doit correspondre à un service existant
- `service.port.number` est le port du SERVICE, pas du conteneur

#### Solution 2 : Ajouter le Host dans /etc/hosts

Pour tester en local sans DNS :

```bash
# Sur votre machine locale (pas dans le cluster)
# Ajouter une ligne dans /etc/hosts (Linux/Mac) ou C:\Windows\System32\drivers\etc\hosts (Windows)
192.168.1.100 mon-app.local

# Tester
curl http://mon-app.local
```

#### Solution 3 : Vérifier le Pare-feu

Assurez-vous que le port 80/443 est ouvert sur votre machine hôte :

```bash
# Vérifier les ports ouverts
sudo ufw status

# Ouvrir le port 80 si nécessaire
sudo ufw allow 80/tcp

# Ouvrir le port 443 si nécessaire
sudo ufw allow 443/tcp
```

#### Solution 4 : Redirection de Port (Port Forwarding)

Si l'ingress controller n'est pas accessible directement :

```bash
# Créer un tunnel vers l'ingress controller
microk8s kubectl port-forward -n ingress service/ingress-nginx-controller 8080:80

# Dans un autre terminal, tester
curl -H "Host: mon-app.local" http://localhost:8080
```

## Problème 5 : Network Policies qui Bloquent le Trafic

### Symptômes

- Certains pods ne peuvent pas communiquer entre eux
- Connexions qui fonctionnaient avant ne fonctionnent plus
- Problèmes apparaissant après l'ajout d'une Network Policy

### Diagnostic

#### Étape 1 : Lister les Network Policies

```bash
# Dans le namespace concerné
microk8s kubectl get networkpolicies -n mon-namespace

# Détails d'une policy
microk8s kubectl describe networkpolicy ma-policy -n mon-namespace
```

#### Étape 2 : Comprendre la Policy

Une Network Policy définit :
- **podSelector** : À quels pods elle s'applique
- **policyTypes** : Ingress (entrant) et/ou Egress (sortant)
- **ingress** : Qui peut contacter ces pods
- **egress** : Qui ces pods peuvent contacter

**Exemple** :
```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-frontend
spec:
  podSelector:
    matchLabels:
      role: backend  # S'applique aux pods avec ce label
  policyTypes:
  - Ingress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          role: frontend  # Autorise seulement les pods frontend
    ports:
    - protocol: TCP
      port: 8080
```

**Interprétation** : Les pods `role=backend` n'acceptent des connexions QUE depuis les pods `role=frontend` sur le port 8080.

#### Étape 3 : Vérifier les Labels des Pods

```bash
# Voir les labels d'un pod
microk8s kubectl get pod mon-pod --show-labels

# Lister les pods avec un label spécifique
microk8s kubectl get pods -l role=frontend
```

### Solutions Courantes

#### Solution 1 : Policy Trop Restrictive

**Problème** : La policy bloque tout sauf ce qui est explicitement autorisé.

**Solution temporaire** (pour tester) :
```bash
# Supprimer temporairement la policy
microk8s kubectl delete networkpolicy ma-policy -n mon-namespace

# Tester si ça fonctionne
# Si oui, le problème vient de la policy
```

**Solution permanente** : Modifier la policy pour autoriser le trafic nécessaire.

#### Solution 2 : Autoriser le Trafic DNS

Une erreur courante : oublier d'autoriser le trafic DNS sortant.

**Ajoutez à votre policy** :
```yaml
spec:
  policyTypes:
  - Egress
  egress:
  # Autoriser le DNS
  - to:
    - namespaceSelector:
        matchLabels:
          name: kube-system
    ports:
    - protocol: UDP
      port: 53
  # Autoriser le reste de votre trafic...
```

#### Solution 3 : Policy par Défaut "Allow All"

Si vous voulez autoriser tout le trafic (utile pour déboguer) :

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-all
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  - Egress
  ingress:
  - {}
  egress:
  - {}
```

**Attention** : N'utilisez ceci qu'en développement ou pour tester !

## Problème 6 : Latence Réseau Élevée

### Symptômes

- Les requêtes sont lentes
- Timeouts fréquents
- Performance dégradée

### Diagnostic

#### Étape 1 : Mesurer la Latence

```bash
# Depuis un pod, mesurer le temps de réponse
microk8s kubectl exec mon-pod -- time curl http://mon-service

# Ou utiliser un pod de test
microk8s kubectl run test --rm -it --image=nicolaka/netshoot -- bash

# Dans le pod :
curl -w "@curl-format.txt" -o /dev/null -s http://mon-service
```

**Fichier curl-format.txt** :
```
time_namelookup:  %{time_namelookup}\n
time_connect:     %{time_connect}\n
time_total:       %{time_total}\n
```

#### Étape 2 : Identifier la Source de Latence

```bash
# Tester la latence DNS
time nslookup mon-service

# Tester la latence réseau (ping entre pods)
# Obtenir l'IP du pod cible
TARGET_IP=$(microk8s kubectl get pod target-pod -o jsonpath='{.status.podIP}')

# Ping depuis un autre pod
microk8s kubectl exec source-pod -- ping -c 10 $TARGET_IP
```

#### Étape 3 : Vérifier les Ressources

```bash
# Utilisation CPU/Mémoire des pods
microk8s kubectl top pods -n mon-namespace

# Utilisation du nœud
microk8s kubectl top nodes
```

### Solutions Courantes

#### Solution 1 : Optimiser les Ressources

Si un pod est surchargé :

```yaml
spec:
  containers:
  - name: mon-app
    resources:
      requests:
        cpu: 100m
        memory: 128Mi
      limits:
        cpu: 500m
        memory: 512Mi
```

#### Solution 2 : Augmenter les Timeouts

Dans votre application, augmentez les timeouts de connexion :

```python
# Exemple Python
import requests

response = requests.get(
    "http://mon-service",
    timeout=30  # 30 secondes au lieu de 5 par défaut
)
```

#### Solution 3 : Utiliser des Readiness Probes

Pour éviter d'envoyer du trafic à des pods non prêts :

```yaml
spec:
  containers:
  - name: mon-app
    readinessProbe:
      httpGet:
        path: /health
        port: 8080
      initialDelaySeconds: 5
      periodSeconds: 5
```

## Outils de Diagnostic Réseau

### 1. netshoot : Le Couteau Suisse

**Installation d'un pod netshoot** :
```bash
microk8s kubectl run netshoot --rm -it --image=nicolaka/netshoot -- bash
```

**Outils disponibles** :
- `curl` : Tester des endpoints HTTP
- `dig`, `nslookup` : Tester DNS
- `ping` : Tester la connectivité ICMP
- `telnet`, `nc` (netcat) : Tester des ports TCP/UDP
- `traceroute` : Tracer le chemin réseau
- `tcpdump` : Capturer le trafic réseau
- `nmap` : Scanner des ports

**Exemples d'utilisation** :

```bash
# Tester un service HTTP
curl -v http://mon-service:80

# Tester DNS avec dig (plus détaillé que nslookup)
dig mon-service.default.svc.cluster.local

# Tester un port TCP
nc -zv mon-service 8080

# Scanner les ports ouverts
nmap mon-service

# Tracer le chemin réseau
traceroute mon-service
```

### 2. kubectl debug

Déboguer directement un pod existant :

```bash
# Créer une copie d'un pod avec des outils
microk8s kubectl debug mon-pod -it --image=nicolaka/netshoot -- bash
```

### 3. tcpdump : Capturer le Trafic

```bash
# Dans un pod netshoot
tcpdump -i any -w capture.pcap

# Ou capturer seulement un port spécifique
tcpdump -i any port 8080 -w capture.pcap

# Copier le fichier vers votre machine pour l'analyser
microk8s kubectl cp netshoot:/capture.pcap ./capture.pcap

# Analyser avec Wireshark sur votre machine
wireshark capture.pcap
```

### 4. curl avec Options Détaillées

```bash
# Voir tous les détails de la connexion
curl -v http://mon-service

# Voir le temps de chaque étape
curl -w "@curl-format.txt" -o /dev/null -s http://mon-service

# Suivre les redirections
curl -L http://mon-service

# Spécifier un timeout
curl --connect-timeout 5 --max-time 10 http://mon-service
```

### 5. netcat (nc) : Tester des Ports

```bash
# Vérifier si un port est ouvert
nc -zv mon-service 8080

# Créer un serveur de test
nc -l -p 9999

# Se connecter à ce serveur depuis un autre pod
nc mon-service 9999
```

## Méthodologie de Dépannage Réseau

Voici une approche systématique pour tout problème réseau :

### 1. Définir le Problème

- Quel est exactement le symptôme ?
- Quelle connexion échoue ? (source → destination)
- Est-ce intermittent ou constant ?

### 2. Vérifier les Bases

```bash
# Les pods sont-ils Running ?
microk8s kubectl get pods -A

# CoreDNS fonctionne-t-il ?
microk8s kubectl get pods -n kube-system -l k8s-app=kube-dns

# Les services existent-ils ?
microk8s kubectl get services

# Ont-ils des endpoints ?
microk8s kubectl get endpoints
```

### 3. Tester par Couche

#### Couche 1 : DNS

```bash
microk8s kubectl run test --rm -it --image=busybox -- nslookup mon-service
```

Si ça échoue → Problème DNS (voir section DNS)

#### Couche 2 : Connectivité Service

```bash
microk8s kubectl run test --rm -it --image=curlimages/curl -- curl http://mon-service
```

Si ça échoue → Problème de service ou endpoints

#### Couche 3 : Connectivité Pod

```bash
# Obtenir l'IP d'un pod
POD_IP=$(microk8s kubectl get pod target-pod -o jsonpath='{.status.podIP}')

# Tester
microk8s kubectl run test --rm -it --image=curlimages/curl -- curl http://$POD_IP:8080
```

Si ça échoue → Problème réseau au niveau pod

### 4. Vérifier les Logs

```bash
# Logs de l'application
microk8s kubectl logs mon-pod

# Logs CoreDNS
microk8s kubectl logs -n kube-system -l k8s-app=kube-dns

# Logs Calico
microk8s kubectl logs -n kube-system -l k8s-app=calico-node

# Logs Ingress (si applicable)
microk8s kubectl logs -n ingress -l app.kubernetes.io/name=ingress-nginx
```

### 5. Vérifier les Configurations

```bash
# Configuration du service
microk8s kubectl get service mon-service -o yaml

# Configuration de l'ingress
microk8s kubectl get ingress mon-ingress -o yaml

# Network Policies
microk8s kubectl get networkpolicies
```

### 6. Isoler le Problème

- Ça fonctionne depuis l'intérieur du cluster mais pas de l'extérieur ? → Problème d'Ingress ou d'exposition
- Ça fonctionne avec l'IP mais pas avec le nom ? → Problème DNS
- Ça fonctionne parfois mais pas toujours ? → Problème d'endpoints ou de readiness
- Ça ne fonctionne jamais ? → Problème de configuration ou network policy

## Checklist de Dépannage Réseau

Utilisez cette checklist pour diagnostiquer méthodiquement :

### Problème DNS
- [ ] CoreDNS est-il Running ? (`kubectl get pods -n kube-system -l k8s-app=kube-dns`)
- [ ] Le service kube-dns existe-t-il ? (`kubectl get svc -n kube-system kube-dns`)
- [ ] La résolution fonctionne-t-elle dans un pod de test ? (`nslookup kubernetes.default`)
- [ ] Le /etc/resolv.conf du pod est-il correct ?

### Problème de Service
- [ ] Le service existe-t-il ? (`kubectl get svc`)
- [ ] A-t-il des endpoints ? (`kubectl get endpoints`)
- [ ] Les labels du service correspondent-ils aux pods ? (`kubectl describe svc`)
- [ ] Les ports sont-ils corrects ? (port vs targetPort)
- [ ] Le service est-il accessible depuis un pod ? (`curl http://service`)

### Problème Pod-à-Pod
- [ ] Les pods ont-ils des IPs ? (`kubectl get pods -o wide`)
- [ ] Calico est-il Running ? (`kubectl get pods -n kube-system -l k8s-app=calico-node`)
- [ ] La connectivité directe fonctionne-t-elle ? (`ping pod-ip`)

### Problème Ingress
- [ ] L'Ingress Controller est-il Running ? (`kubectl get pods -n ingress`)
- [ ] La ressource Ingress existe-t-elle ? (`kubectl get ingress`)
- [ ] A-t-elle une ADDRESS ? (`kubectl describe ingress`)
- [ ] Le service backend a-t-il des endpoints ?
- [ ] Le DNS externe est-il configuré correctement ?

### Problème Network Policy
- [ ] Y a-t-il des Network Policies ? (`kubectl get networkpolicies`)
- [ ] Bloquent-elles le trafic nécessaire ?
- [ ] Les labels correspondent-ils ?
- [ ] Le trafic DNS est-il autorisé ?

## Scénarios de Dépannage Complets

### Scénario 1 : "Mon service ne répond pas"

**Étapes** :

```bash
# 1. Vérifier que le service existe
microk8s kubectl get service mon-service
# Si absent → créer le service

# 2. Vérifier les endpoints
microk8s kubectl get endpoints mon-service
# Si vide → vérifier les labels et l'état des pods

# 3. Vérifier les pods
microk8s kubectl get pods -l app=mon-app
# Si pas Running → diagnostiquer pourquoi (logs, describe)

# 4. Tester depuis un pod
microk8s kubectl run test --rm -it --image=curlimages/curl -- curl http://mon-service
# Si échec → problème de configuration

# 5. Vérifier les logs
microk8s kubectl logs -l app=mon-app
# Chercher des erreurs
```

### Scénario 2 : "DNS ne fonctionne pas"

**Étapes** :

```bash
# 1. Vérifier CoreDNS
microk8s kubectl get pods -n kube-system -l k8s-app=kube-dns
# Si pas Running → redémarrer ou réactiver l'addon

# 2. Tester la résolution
microk8s kubectl run test --rm -it --image=busybox -- nslookup kubernetes.default
# Si échec → problème CoreDNS

# 3. Vérifier les logs CoreDNS
microk8s kubectl logs -n kube-system -l k8s-app=kube-dns
# Chercher des erreurs

# 4. Redémarrer CoreDNS
microk8s kubectl delete pod -n kube-system -l k8s-app=kube-dns

# 5. Re-tester
```

### Scénario 3 : "Ingress retourne 404"

**Étapes** :

```bash
# 1. Vérifier l'Ingress Controller
microk8s kubectl get pods -n ingress
# Si pas Running → vérifier les logs

# 2. Vérifier la ressource Ingress
microk8s kubectl get ingress mon-ingress
microk8s kubectl describe ingress mon-ingress
# Vérifier les règles et backend

# 3. Tester le service backend
microk8s kubectl run test --rm -it --image=curlimages/curl -- curl http://mon-service
# Si ça fonctionne → problème dans la config Ingress

# 4. Vérifier les logs Ingress
microk8s kubectl logs -n ingress -l app.kubernetes.io/name=ingress-nginx | grep mon-ingress

# 5. Tester avec curl et Host header
curl -H "Host: mon-app.local" http://INGRESS-IP/
```

## Bonnes Pratiques Réseau

### 1. Toujours Utiliser des Services

❌ **Ne faites JAMAIS** :
```python
# Connexion directe aux IPs de pods
response = requests.get("http://10.1.1.5:8080")
```

✅ **Faites TOUJOURS** :
```python
# Connexion via service
response = requests.get("http://mon-service:8080")
```

**Pourquoi** : Les IPs de pods changent, pas les services.

### 2. Utiliser des FQDN en Production

✅ **En production** :
```python
# Nom complet (FQDN)
db_url = "postgres.database.svc.cluster.local"
```

**Format** : `<service>.<namespace>.svc.cluster.local`

### 3. Définir des Readiness Probes

```yaml
spec:
  containers:
  - name: mon-app
    readinessProbe:
      httpGet:
        path: /health
        port: 8080
      initialDelaySeconds: 5
      periodSeconds: 5
```

**Pourquoi** : Évite d'envoyer du trafic à des pods non prêts.

### 4. Configurer des Timeouts Appropriés

```python
# Timeouts adaptés
response = requests.get(
    url,
    timeout=(5, 30)  # (connect timeout, read timeout)
)
```

### 5. Activer le DNS dès le Début

```bash
# Toujours avoir DNS activé
microk8s enable dns
```

### 6. Surveiller les Network Policies

Si vous utilisez des Network Policies, documentez-les clairement et testez minutieusement.

### 7. Utiliser des Labels Cohérents

```yaml
# Convention de nommage claire
metadata:
  labels:
    app: mon-application
    component: backend
    version: v1.2.0
```

## Résumé

### Problèmes Courants et Solutions Rapides

| Problème | Solution Rapide |
|----------|----------------|
| DNS ne fonctionne pas | Redémarrer CoreDNS : `kubectl delete pod -n kube-system -l k8s-app=kube-dns` |
| Service sans endpoints | Vérifier les labels : `kubectl get pods -l <label>` et `kubectl describe svc` |
| Connection refused | Vérifier que le pod écoute sur le bon port : `kubectl exec pod -- netstat -tulpn` |
| Ingress 404 | Vérifier `ingressClassName: nginx` et que le service backend existe |
| Timeout | Vérifier Network Policies et que les pods sont Ready |
| Latence élevée | Vérifier ressources : `kubectl top pods` |

### Commandes Essentielles

```bash
# Tester DNS
kubectl run test --rm -it --image=busybox -- nslookup mon-service

# Tester connectivité HTTP
kubectl run test --rm -it --image=curlimages/curl -- curl http://mon-service

# Pod de diagnostic complet
kubectl run netshoot --rm -it --image=nicolaka/netshoot -- bash

# Vérifier endpoints
kubectl get endpoints mon-service

# Logs des composants réseau
kubectl logs -n kube-system -l k8s-app=kube-dns
kubectl logs -n kube-system -l k8s-app=calico-node
kubectl logs -n ingress -l app.kubernetes.io/name=ingress-nginx
```

### Points Clés à Retenir

1. **DNS d'abord** : 80% des problèmes réseau sont des problèmes DNS
2. **Endpoints = pods** : Pas d'endpoints = aucun pod ne correspond au service
3. **Labels doivent correspondre** : Entre service et pods
4. **Utilisez des Services** : Jamais de connexions directes aux IPs de pods
5. **Network Policies** : Peuvent bloquer silencieusement le trafic
6. **Outils de test** : netshoot est votre meilleur ami
7. **Logs partout** : Application, CoreDNS, Calico, Ingress Controller

**Prochaine étape** : Section 23.6 - Problèmes de certificats

---


⏭️ [Problèmes de certificats](/23-depannage-et-maintenance/06-problemes-de-certificats.md)
