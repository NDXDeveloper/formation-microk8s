🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 7.5 Test de connectivité interne

## Introduction

Dans les chapitres précédents, nous avons appris comment fonctionne le réseau Kubernetes en théorie. Maintenant, passons à la pratique : comment vérifier que tout fonctionne réellement ? Comment diagnostiquer quand quelque chose ne va pas ?

Ce chapitre est votre boîte à outils réseau. Vous apprendrez à utiliser les commandes et outils essentiels pour tester, valider et déboguer la connectivité dans votre cluster Kubernetes.

**Philosophie du debugging :** Quand un problème survient, ne paniquez pas et ne devinez pas. Testez méthodiquement, une chose à la fois, du plus simple au plus complexe.

## Pourquoi tester la connectivité ?

### Scénarios courants nécessitant des tests

**1. Après le déploiement d'une nouvelle application**
- "Mon application est déployée, mais elle ne peut pas contacter la base de données"

**2. Après un changement de configuration**
- "J'ai modifié mon Service, mais ça ne fonctionne toujours pas"

**3. Problèmes intermittents**
- "Parfois ça marche, parfois ça échoue"

**4. Migration ou upgrade**
- "Après la mise à jour, plus rien ne communique"

**5. Validation proactive**
- "Je veux m'assurer que tout fonctionne avant de déployer en production"

### L'approche méthodique : du bas vers le haut

Quand vous testez, suivez toujours cet ordre (du plus simple au plus complexe) :

```
1. Le Pod existe-t-il et est-il en "Running" ?
   ↓
2. Le Pod a-t-il une IP ?
   ↓
3. Le Pod peut-il pinguer d'autres Pods ?
   ↓
4. Le DNS fonctionne-t-il ?
   ↓
5. Le Service existe-t-il et a-t-il des Endpoints ?
   ↓
6. Le Service est-il accessible ?
   ↓
7. L'application dans le Pod répond-elle ?
```

## Les outils essentiels

Avant de plonger dans les tests, familiarisons-nous avec les outils que nous allons utiliser.

### 1. kubectl : votre couteau suisse

`kubectl` est l'outil principal pour interagir avec Kubernetes. Dans MicroK8s, c'est `microk8s kubectl`.

**Commandes essentielles :**

```bash
# Voir les Pods
microk8s kubectl get pods

# Voir les Services
microk8s kubectl get svc

# Voir les Endpoints
microk8s kubectl get endpoints

# Détails d'une ressource
microk8s kubectl describe pod <nom-pod>

# Logs d'un Pod
microk8s kubectl logs <nom-pod>

# Exécuter une commande dans un Pod
microk8s kubectl exec <nom-pod> -- <commande>
```

**Astuce :** Créez un alias pour simplifier :
```bash
# Ajouter dans ~/.bashrc ou ~/.zshrc
alias k='microk8s kubectl'

# Maintenant vous pouvez faire :
k get pods
```

### 2. Pods de debug : votre laboratoire réseau

Pour tester le réseau, vous avez besoin d'un Pod avec des outils réseau. Il existe plusieurs images pratiques :

#### busybox : léger et rapide

```bash
# Créer un Pod busybox
microk8s kubectl run busybox --image=busybox --restart=Never -- sleep 3600

# Outils disponibles : ping, wget, nslookup, nc, traceroute
```

**Avantages :**
- Très léger (1-2 MB)
- Démarre rapidement
- Outils de base inclus

**Inconvénients :**
- Pas d'outils avancés (pas de curl, dig limité)

#### nicolaka/netshoot : l'arsenal complet

```bash
# Créer un Pod netshoot
microk8s kubectl run netshoot --image=nicolaka/netshoot --restart=Never -- sleep 3600

# Outils disponibles : curl, dig, nslookup, ping, traceroute, iperf3, tcpdump, et bien plus
```

**Avantages :**
- Tous les outils réseau imaginables
- Idéal pour debugging avancé

**Inconvénients :**
- Image plus lourde (~300 MB)
- Temps de démarrage plus long

#### alpine : un bon compromis

```bash
# Créer un Pod alpine avec des outils
microk8s kubectl run alpine --image=alpine:latest --restart=Never -- sleep 3600

# Installer des outils supplémentaires si besoin
microk8s kubectl exec alpine -- apk add --no-cache curl bind-tools
```

**Recommandation :** Commencez avec **busybox** pour les tests simples, passez à **netshoot** pour les cas complexes.

### 3. Ephemeral Containers : debug sans redémarrage

Kubernetes 1.23+ permet d'ajouter un conteneur temporaire dans un Pod existant pour le déboguer sans le redémarrer.

```bash
# Ajouter un conteneur de debug à un Pod existant
microk8s kubectl debug <nom-pod> -it --image=busybox --target=<nom-conteneur>

# Vous êtes maintenant dans le Pod avec des outils de debug
```

**Avantage :** Pas besoin de recréer le Pod ou de modifier son image.

## Tester la connectivité de base

### Test 1 : Vérifier l'état des Pods

**Objectif :** S'assurer que les Pods existent et sont en état de fonctionnement.

```bash
# Lister tous les Pods
microk8s kubectl get pods

# Lister les Pods avec plus d'informations (IP, Node)
microk8s kubectl get pods -o wide

# Lister les Pods d'un namespace spécifique
microk8s kubectl get pods -n kube-system
```

**Interpréter la sortie :**

```
NAME          READY   STATUS    RESTARTS   AGE   IP          NODE
frontend-1    1/1     Running   0          5m    10.1.1.5    node1
backend-1     1/1     Running   2          3m    10.1.1.8    node1
database-1    0/1     Pending   0          1m    <none>      <none>
```

**Colonne STATUS - Ce qu'elle signifie :**

- **Running** ✅ : Le Pod fonctionne normalement
- **Pending** ⚠️ : Kubernetes essaie de démarrer le Pod (attente de ressources)
- **CrashLoopBackOff** ❌ : Le Pod démarre puis plante en boucle
- **Error** ❌ : Le Pod a rencontré une erreur
- **ImagePullBackOff** ❌ : Impossible de télécharger l'image
- **Completed** ✅ : Pod terminé (normal pour un Job)

**Colonne READY - Format : X/Y**

- **X** : Nombre de conteneurs prêts (Readiness Probe OK)
- **Y** : Nombre total de conteneurs dans le Pod
- **1/1** ✅ : Tout va bien
- **0/1** ❌ : Le conteneur n'est pas prêt

**Si un Pod n'est pas "Running" ou "Ready" :**

```bash
# Voir les détails et événements
microk8s kubectl describe pod <nom-pod>

# Regarder les logs
microk8s kubectl logs <nom-pod>

# Si plusieurs conteneurs, spécifier lequel
microk8s kubectl logs <nom-pod> -c <nom-conteneur>
```

### Test 2 : Vérifier l'attribution des IPs

**Objectif :** Confirmer que chaque Pod a bien reçu une adresse IP.

```bash
# Voir les IPs des Pods
microk8s kubectl get pods -o wide

# Voir l'IP d'un Pod spécifique
microk8s kubectl get pod <nom-pod> -o jsonpath='{.status.podIP}'
```

**Exemple de sortie :**

```
NAME        READY   STATUS    RESTARTS   AGE   IP           NODE
frontend    1/1     Running   0          5m    10.1.1.5     node1
backend     1/1     Running   0          5m    10.1.1.8     node1
```

**Si un Pod n'a pas d'IP :**

C'est un problème grave, probablement lié au CNI (Calico).

```bash
# Vérifier que Calico fonctionne
microk8s kubectl get pods -n kube-system -l k8s-app=calico-node

# Tous les Pods calico-node doivent être "Running"
```

### Test 3 : Ping entre Pods

**Objectif :** Vérifier la connectivité réseau de base au niveau IP.

```bash
# Créer un Pod de test
microk8s kubectl run busybox --image=busybox --restart=Never -- sleep 3600

# Obtenir l'IP d'un Pod cible
TARGET_IP=$(microk8s kubectl get pod backend -o jsonpath='{.status.podIP}')

# Depuis busybox, pinguer le Pod cible
microk8s kubectl exec busybox -- ping -c 3 $TARGET_IP
```

**Sortie attendue :**

```
PING 10.1.1.8 (10.1.1.8): 56 data bytes
64 bytes from 10.1.1.8: seq=0 ttl=63 time=0.123 ms
64 bytes from 10.1.1.8: seq=1 ttl=63 time=0.098 ms
64 bytes from 10.1.1.8: seq=2 ttl=63 time=0.105 ms

--- 10.1.1.8 ping statistics ---
3 packets transmitted, 3 packets received, 0% packet loss
```

**Interprétation :**

- ✅ **0% packet loss** : Connectivité réseau OK
- ✅ **time < 1ms** : Communication sur le même Node (très rapide)
- ✅ **time 1-10ms** : Communication entre Nodes (normal)
- ❌ **100% packet loss** : Pas de connectivité (problème réseau)
- ⚠️ **50% packet loss** : Connectivité instable

**Si le ping échoue :**

```bash
# Vérifier les routes réseau sur le Node
# (nécessite un accès SSH au Node)
ip route show

# Vérifier les règles iptables
sudo iptables -L -n -v

# Vérifier les logs de Calico
microk8s kubectl logs -n kube-system -l k8s-app=calico-node
```

### Test 4 : Connectivité sur un port spécifique

Ping teste seulement ICMP (niveau réseau). Pour tester une application (HTTP, base de données), vous devez tester le port TCP/UDP.

#### Avec netcat (nc)

```bash
# Tester si un port est ouvert
microk8s kubectl exec busybox -- nc -zv 10.1.1.8 8080

# -z : mode scan (pas de données envoyées)
# -v : verbose
```

**Sortie attendue :**

```
10.1.1.8 (10.1.1.8:8080) open
```

**Si connexion refuse :**

```
10.1.1.8 (10.1.1.8:8080): Connection refused
```

Cela signifie que le Pod est accessible réseau, mais aucune application n'écoute sur le port 8080.

#### Avec telnet (netshoot)

```bash
# Créer un Pod avec telnet
microk8s kubectl run netshoot --image=nicolaka/netshoot --restart=Never -- sleep 3600

# Tester la connexion
microk8s kubectl exec netshoot -- telnet 10.1.1.8 8080
```

**Sortie attendue :**

```
Trying 10.1.1.8...
Connected to 10.1.1.8.
Escape character is '^]'.
```

Si vous voyez "Connected", le port est accessible.

#### Avec curl (HTTP/HTTPS)

Pour les services HTTP :

```bash
# Tester une requête HTTP simple
microk8s kubectl exec busybox -- wget -O- http://10.1.1.8:8080

# Ou avec curl (dans netshoot)
microk8s kubectl exec netshoot -- curl -v http://10.1.1.8:8080
```

**Sortie attendue :**

```
HTTP/1.1 200 OK
Content-Type: text/html
...
```

## Tester le DNS

Le DNS est critique pour la découverte de services. Testons-le en détail.

### Test 5 : Résolution DNS de base

**Objectif :** Vérifier que CoreDNS fonctionne et résout les noms.

```bash
# Créer un Pod de test
microk8s kubectl run busybox --image=busybox --restart=Never -- sleep 3600

# Tester la résolution du Service Kubernetes (devrait toujours fonctionner)
microk8s kubectl exec busybox -- nslookup kubernetes

# Tester la résolution d'un nom externe
microk8s kubectl exec busybox -- nslookup google.com
```

**Sortie attendue pour kubernetes :**

```
Server:    10.152.183.10
Address 1: 10.152.183.10 kube-dns.kube-system.svc.cluster.local

Name:      kubernetes
Address 1: 10.152.183.1 kubernetes.default.svc.cluster.local
```

**Interprétation :**

- **Server: 10.152.183.10** : IP de CoreDNS (correct)
- **Address 1: 10.152.183.1** : IP du Service kubernetes (correct)

**Sortie attendue pour google.com :**

```
Server:    10.152.183.10
Address 1: 10.152.183.10 kube-dns.kube-system.svc.cluster.local

Name:      google.com
Address 1: 142.250.185.78
```

Si vous voyez une IP pour google.com, le forwarding DNS externe fonctionne.

**Si nslookup échoue complètement :**

```bash
# Vérifier que CoreDNS tourne
microk8s kubectl get pods -n kube-system -l k8s-app=kube-dns

# Vérifier le Service kube-dns
microk8s kubectl get svc -n kube-system kube-dns

# Vérifier /etc/resolv.conf du Pod
microk8s kubectl exec busybox -- cat /etc/resolv.conf
```

Le resolv.conf devrait contenir :

```
nameserver 10.152.183.10
search default.svc.cluster.local svc.cluster.local cluster.local
options ndots:5
```

### Test 6 : Résolution des Services

**Objectif :** Tester que vos Services sont résolvables par DNS.

```bash
# Supposons que vous avez un Service "backend" dans le namespace "default"

# Test avec nom court (depuis le même namespace)
microk8s kubectl exec busybox -- nslookup backend

# Test avec namespace
microk8s kubectl exec busybox -- nslookup backend.default

# Test avec FQDN complet
microk8s kubectl exec busybox -- nslookup backend.default.svc.cluster.local
```

**Sortie attendue :**

```
Server:    10.152.183.10
Address 1: 10.152.183.10 kube-dns.kube-system.svc.cluster.local

Name:      backend.default.svc.cluster.local
Address 1: 10.152.183.25 backend.default.svc.cluster.local
```

**L'IP retournée (10.152.183.25) est le ClusterIP du Service.**

**Si le Service ne résout pas :**

```bash
# Vérifier que le Service existe
microk8s kubectl get svc backend

# Si le Service n'existe pas, le créer
# Si le Service existe, attendre 30 secondes (expiration du cache CoreDNS)
```

### Test 7 : Résolution avec dig (plus détaillé)

`dig` donne plus d'informations que `nslookup`.

```bash
# Utiliser netshoot qui inclut dig
microk8s kubectl run netshoot --image=nicolaka/netshoot --restart=Never -- sleep 3600

# Requête DNS de type A (IPv4)
microk8s kubectl exec netshoot -- dig backend.default.svc.cluster.local

# Requête DNS de type SRV (découverte de port)
microk8s kubectl exec netshoot -- dig SRV _http._tcp.backend.default.svc.cluster.local
```

**Sortie dig (extrait) :**

```
;; ANSWER SECTION:
backend.default.svc.cluster.local. 30 IN A 10.152.183.25

;; Query time: 2 msec
;; SERVER: 10.152.183.10#53(10.152.183.10)
```

**Informations importantes :**

- **30 IN A** : TTL de 30 secondes, type A (IPv4)
- **Query time: 2 msec** : Temps de réponse (rapide = OK)
- **SERVER: 10.152.183.10** : Serveur DNS utilisé (CoreDNS)

### Test 8 : Résolution de Headless Services

Pour les Headless Services (clusterIP: None), DNS retourne toutes les IPs des Pods.

```bash
# Supposons un Headless Service "database-headless"
microk8s kubectl exec netshoot -- nslookup database-headless
```

**Sortie attendue :**

```
Server:    10.152.183.10
Address 1: 10.152.183.10 kube-dns.kube-system.svc.cluster.local

Name:      database-headless.default.svc.cluster.local
Address 1: 10.1.1.10
Address 2: 10.1.1.11
Address 3: 10.1.1.12
```

Vous voyez les 3 IPs des Pods backend, pas une ClusterIP.

## Tester les Services

### Test 9 : Existence et configuration du Service

```bash
# Lister les Services
microk8s kubectl get svc

# Détails d'un Service
microk8s kubectl describe svc backend
```

**Exemple de describe :**

```
Name:              backend
Namespace:         default
Labels:            app=backend
Annotations:       <none>
Selector:          app=backend
Type:              ClusterIP
IP Family Policy:  SingleStack
IP Families:       IPv4
IP:                10.152.183.25
IPs:               10.152.183.25
Port:              http  80/TCP
TargetPort:        8080/TCP
Endpoints:         10.1.1.10:8080,10.1.1.11:8080,10.1.1.12:8080
Session Affinity:  None
Events:            <none>
```

**Points à vérifier :**

- ✅ **Selector: app=backend** : Correspond aux labels de vos Pods ?
- ✅ **IP: 10.152.183.25** : Le Service a une ClusterIP
- ✅ **TargetPort: 8080** : Correspond au port de votre application ?
- ✅ **Endpoints: 10.1.1.10:8080...** : Des Pods sont listés ?

**Si Endpoints est vide :**

```bash
# Vérifier les Endpoints directement
microk8s kubectl get endpoints backend

# Vérifier que des Pods existent avec le bon label
microk8s kubectl get pods -l app=backend
```

Si aucun Pod n'a le label `app=backend`, le Service ne peut pas les trouver.

### Test 10 : Accès au Service depuis un Pod

**Objectif :** Vérifier qu'un Pod peut contacter le Service.

```bash
# Depuis le Pod busybox, appeler le Service par son nom DNS
microk8s kubectl exec busybox -- wget -O- http://backend:80

# Ou avec son FQDN
microk8s kubectl exec busybox -- wget -O- http://backend.default.svc.cluster.local:80

# Ou avec l'IP du Service directement
microk8s kubectl exec busybox -- wget -O- http://10.152.183.25:80
```

**Si cela fonctionne :**

```
Connecting to backend:80 (10.152.183.25:80)
writing to stdout
-                    100% |********************************|  1024  0:00:00 ETA
written to stdout
<!DOCTYPE html>
<html>...
```

Félicitations ! La communication via Service fonctionne.

**Si wget échoue :**

```
wget: can't connect to remote host (10.152.183.25): Connection refused
```

**Diagnostic :**

1. Le Service a-t-il des Endpoints ? (test 9)
2. Les Pods backend sont-ils en "Running" et "Ready" ?
3. Le port est-il correct ? (targetPort = port de l'application)

### Test 11 : Load balancing du Service

**Objectif :** Vérifier que le Service distribue bien le trafic entre les Pods.

```bash
# Faire plusieurs requêtes et voir quel Pod répond
for i in {1..10}; do
  microk8s kubectl exec busybox -- wget -qO- http://backend:80/hostname
done
```

Si vous avez configuré votre application pour retourner son hostname, vous devriez voir différents noms de Pods :

```
backend-1
backend-2
backend-1
backend-3
backend-1
backend-2
...
```

**Si vous voyez toujours le même Pod :**

- Vérifiez `sessionAffinity` dans le Service (doit être `None` pour du load balancing standard)
- Vérifiez que plusieurs Pods backend existent

### Test 12 : Ports multiples

Si votre Service expose plusieurs ports :

```bash
# Tester le port HTTP (80)
microk8s kubectl exec busybox -- wget -O- http://backend:80

# Tester le port metrics (9090)
microk8s kubectl exec busybox -- wget -O- http://backend:9090/metrics
```

## Tester la connectivité externe

### Test 13 : Accès depuis le Pod vers Internet

**Objectif :** Vérifier que les Pods peuvent accéder à Internet.

```bash
# Tester la résolution DNS externe
microk8s kubectl exec busybox -- nslookup google.com

# Tester l'accès HTTP externe
microk8s kubectl exec busybox -- wget -O- https://www.google.com

# Tester avec curl
microk8s kubectl exec netshoot -- curl -I https://www.google.com
```

**Si cela échoue :**

1. **DNS ne résout pas** : Problème avec CoreDNS forwarding
   ```bash
   # Vérifier le Corefile
   microk8s kubectl get cm coredns -n kube-system -o yaml | grep forward
   ```

2. **DNS résout, mais connexion échoue** : Problème de routage ou firewall
   ```bash
   # Vérifier les règles iptables sur le Node
   # (nécessite accès SSH)
   sudo iptables -L -n -v -t nat
   ```

3. **Network Policies** : Peut-être qu'une policy bloque le trafic sortant
   ```bash
   microk8s kubectl get networkpolicies
   ```

### Test 14 : Accès depuis l'extérieur (NodePort/LoadBalancer)

Si vous avez exposé un Service en NodePort ou LoadBalancer :

```bash
# Obtenir le port NodePort
microk8s kubectl get svc backend

# Exemple de sortie :
# NAME      TYPE       CLUSTER-IP       EXTERNAL-IP   PORT(S)        AGE
# backend   NodePort   10.152.183.25    <none>        80:30080/TCP   5m
```

Le port **30080** est le NodePort.

**Tester depuis votre machine (hors du cluster) :**

```bash
# Obtenir l'IP du Node
NODE_IP=$(microk8s kubectl get nodes -o jsonpath='{.items[0].status.addresses[?(@.type=="InternalIP")].address}')

# Tester l'accès
curl http://$NODE_IP:30080
```

**Si cela ne fonctionne pas :**

1. **Firewall** : Vérifier que le port 30080 est ouvert
   ```bash
   sudo ufw status
   # Ouvrir le port si nécessaire
   sudo ufw allow 30080/tcp
   ```

2. **Le Service existe-t-il ?**
   ```bash
   microk8s kubectl get svc backend
   ```

## Analyse des logs

### Test 15 : Logs des Pods

Les logs sont votre fenêtre sur ce qui se passe dans l'application.

```bash
# Voir les logs d'un Pod
microk8s kubectl logs <nom-pod>

# Logs en temps réel (follow)
microk8s kubectl logs -f <nom-pod>

# Logs des X dernières lignes
microk8s kubectl logs --tail=50 <nom-pod>

# Logs d'un conteneur spécifique (si plusieurs conteneurs)
microk8s kubectl logs <nom-pod> -c <nom-conteneur>

# Logs du conteneur précédent (si le Pod a redémarré)
microk8s kubectl logs <nom-pod> --previous
```

**Que chercher dans les logs ?**

**Erreurs de connexion :**
```
Error: connect ECONNREFUSED 10.152.183.25:80
Error: getaddrinfo ENOTFOUND database
Connection timeout
Failed to connect to database
```

**Succès de connexion :**
```
Connected to database successfully
Listening on port 8080
Ready to accept connections
```

### Test 16 : Logs de CoreDNS

Si vous suspectez un problème DNS :

```bash
# Logs de CoreDNS
microk8s kubectl logs -n kube-system -l k8s-app=kube-dns

# Logs en temps réel
microk8s kubectl logs -n kube-system -l k8s-app=kube-dns -f
```

**Que chercher ?**

**Erreurs répétées :**
```
[ERROR] plugin/errors: 2 backend.default.svc.cluster.local. A: read udp timeout
```

**Requêtes DNS (si log activé) :**
```
[INFO] 10.1.1.5:45678 - "A IN backend.default.svc.cluster.local. udp 56 false 512" NOERROR qr,aa,rd 106 0.000s
```

### Test 17 : Logs de Calico

Si vous suspectez un problème réseau au niveau CNI :

```bash
# Logs de calico-node (un Pod par Node)
microk8s kubectl logs -n kube-system -l k8s-app=calico-node

# Logs d'un calico-node spécifique
microk8s kubectl logs -n kube-system <nom-pod-calico-node>
```

**Erreurs courantes :**

```
Failed to allocate IP
BGP connection failed
Failed to configure veth pair
```

## Events : l'historique Kubernetes

### Test 18 : Events du cluster

Les **Events** sont des messages que Kubernetes génère quand quelque chose se passe.

```bash
# Voir tous les events récents
microk8s kubectl get events

# Trier par timestamp (plus récent en premier)
microk8s kubectl get events --sort-by='.lastTimestamp'

# Events d'un namespace spécifique
microk8s kubectl get events -n kube-system

# Events concernant un Pod spécifique
microk8s kubectl describe pod <nom-pod> | grep -A 10 Events
```

**Exemples d'events utiles :**

**Normal (informationnel) :**
```
Normal  Scheduled  2m   Pod backend-1 successfully scheduled on node1
Normal  Pulling    2m   Pulling image "nginx:latest"
Normal  Pulled     1m   Successfully pulled image
Normal  Created    1m   Created container backend
Normal  Started    1m   Started container backend
```

**Warning (attention) :**
```
Warning  FailedScheduling  1m   0/1 nodes available: insufficient cpu
Warning  BackOff           1m   Back-off restarting failed container
Warning  Unhealthy         1m   Readiness probe failed: HTTP probe failed
```

**Events réseau :**
```
Warning  NetworkNotReady   1m   network is not ready: [runtime network not ready]
Normal   SandboxChanged    1m   Pod sandbox changed, it will be killed and re-created
```

## Outils de diagnostic avancés

### Test 19 : Inspecter les règles iptables

Les Services sont implémentés via iptables. Vous pouvez les inspecter (nécessite accès au Node).

```bash
# Sur le Node (via SSH ou localement si MicroK8s local)
sudo iptables -t nat -L -n -v | grep <ClusterIP-du-Service>

# Exemple pour 10.152.183.25
sudo iptables -t nat -L -n -v | grep 10.152.183.25
```

**Sortie typique :**

```
DNAT  tcp  --  *  *  0.0.0.0/0  10.152.183.25  tcp dpt:80 to:10.1.1.10:8080
DNAT  tcp  --  *  *  0.0.0.0/0  10.152.183.25  tcp dpt:80 to:10.1.1.11:8080
```

Cela montre que le trafic vers 10.152.183.25:80 est redirigé (DNAT) vers les Pods backend.

### Test 20 : Capturer le trafic réseau (tcpdump)

Pour debugging très avancé, capturez les paquets réseau.

```bash
# Utiliser netshoot qui inclut tcpdump
microk8s kubectl run netshoot --image=nicolaka/netshoot --restart=Never -- sleep 3600

# Capturer le trafic sur eth0
microk8s kubectl exec netshoot -- tcpdump -i eth0 -n

# Capturer seulement HTTP (port 80)
microk8s kubectl exec netshoot -- tcpdump -i eth0 -n port 80

# Capturer vers un fichier puis analyser
microk8s kubectl exec netshoot -- tcpdump -i eth0 -w /tmp/capture.pcap
microk8s kubectl cp netshoot:/tmp/capture.pcap ./capture.pcap
# Ouvrir avec Wireshark sur votre machine
```

**Que chercher ?**

- **SYN → SYN-ACK → ACK** : Handshake TCP réussi
- **SYN → RST** : Connexion refusée
- **SYN → (pas de réponse)** : Paquets perdus ou bloqués

### Test 21 : Tester la bande passante (iperf3)

Pour tester les performances réseau entre Pods.

```bash
# Créer un serveur iperf3
microk8s kubectl run iperf-server --image=networkstatic/iperf3 --port=5201 -- iperf3 -s

# Obtenir l'IP du serveur
SERVER_IP=$(microk8s kubectl get pod iperf-server -o jsonpath='{.status.podIP}')

# Créer un client iperf3
microk8s kubectl run iperf-client --image=networkstatic/iperf3 --restart=Never -- iperf3 -c $SERVER_IP
```

**Résultat typique :**

```
[  5] 0.00-10.00 sec  1.10 GBytes  945 Mbits/sec
```

Cela montre un débit de ~945 Mbits/sec entre les deux Pods.

### Test 22 : Tracer le chemin réseau (traceroute)

```bash
microk8s kubectl exec netshoot -- traceroute 10.1.1.8

# ou mtr pour un traceroute continu
microk8s kubectl exec netshoot -- mtr -n 10.1.1.8
```

**Sortie typique (même Node) :**

```
1. 10.1.1.8  0.123 ms
```

Un seul saut = communication locale (même Node).

**Sortie typique (autre Node) :**

```
1. 192.168.1.1   0.5 ms
2. 10.1.1.8      1.2 ms
```

Deux sauts = communication via routeur/switch.

## Méthodologie de debugging : checklist complète

Quand quelque chose ne fonctionne pas, suivez cette checklist étape par étape.

### Problème : "Mon application ne peut pas contacter le Service X"

**Étape 1 : Vérifier que le Service existe**

```bash
microk8s kubectl get svc <nom-service>
```

✅ Existe → Continuer
❌ N'existe pas → Créer le Service

**Étape 2 : Vérifier que le Service a des Endpoints**

```bash
microk8s kubectl get endpoints <nom-service>
```

✅ A des Endpoints → Continuer
❌ Pas d'Endpoints → Vérifier les Pods et les labels (voir Étape 3)

**Étape 3 : Vérifier les Pods backend**

```bash
# Vérifier que des Pods existent avec le bon label
microk8s kubectl get pods -l <selector-du-service>
```

✅ Des Pods existent et sont "Running" et "Ready" → Continuer
❌ Pas de Pods ou non Ready → Corriger les Pods d'abord

**Étape 4 : Tester la résolution DNS**

```bash
microk8s kubectl run test --image=busybox --restart=Never -- sleep 3600
microk8s kubectl exec test -- nslookup <nom-service>
```

✅ Résout vers une IP → Continuer
❌ Ne résout pas → Problème DNS (vérifier CoreDNS)

**Étape 5 : Tester la connexion au Service**

```bash
microk8s kubectl exec test -- wget -O- http://<nom-service>:<port>
```

✅ Fonctionne → Le problème est ailleurs (dans l'application appelante)
❌ Échoue → Continuer

**Étape 6 : Tester la connexion directe à un Pod**

```bash
# Obtenir l'IP d'un Pod backend
POD_IP=$(microk8s kubectl get pod <nom-pod-backend> -o jsonpath='{.status.podIP}')

# Tester la connexion directe
microk8s kubectl exec test -- wget -O- http://$POD_IP:<target-port>
```

✅ Fonctionne → Problème avec kube-proxy/iptables
❌ Échoue → Problème avec le Pod backend (port incorrect ? application ne répond pas ?)

**Étape 7 : Vérifier les logs du Pod backend**

```bash
microk8s kubectl logs <nom-pod-backend>
```

Chercher des erreurs, des messages de démarrage, etc.

**Étape 8 : Vérifier les Readiness/Liveness Probes**

```bash
microk8s kubectl describe pod <nom-pod-backend> | grep -A 5 Conditions
```

Si le Pod n'est pas "Ready", vérifier pourquoi la probe échoue.

**Étape 9 : Vérifier les Network Policies**

```bash
microk8s kubectl get networkpolicies
microk8s kubectl describe networkpolicy <nom-policy>
```

Une Network Policy peut bloquer le trafic.

**Étape 10 : Vérifier les events**

```bash
microk8s kubectl get events --sort-by='.lastTimestamp' | head -20
```

Rechercher des messages d'erreur récents.

## Outils de diagnostic intégrés à Kubernetes

### Test 23 : kubectl port-forward

Créer un tunnel depuis votre machine vers un Pod/Service.

```bash
# Forward le port 8080 du Pod vers votre localhost:8080
microk8s kubectl port-forward pod/<nom-pod> 8080:8080

# Forward le port 80 du Service vers votre localhost:8080
microk8s kubectl port-forward svc/<nom-service> 8080:80
```

Maintenant, depuis votre machine :

```bash
curl http://localhost:8080
```

**Utilité :** Tester rapidement un Service sans l'exposer via NodePort/LoadBalancer.

### Test 24 : kubectl proxy

Créer un proxy vers l'API Kubernetes.

```bash
microk8s kubectl proxy --port=8001
```

Maintenant vous pouvez accéder aux Pods via l'API :

```bash
# Accéder à un Pod via l'API
curl http://localhost:8001/api/v1/namespaces/default/pods/<nom-pod>/proxy/
```

### Test 25 : kubectl top (métriques)

Voir l'utilisation des ressources (nécessite metrics-server).

```bash
# CPU/Mémoire des Pods
microk8s kubectl top pods

# CPU/Mémoire des Nodes
microk8s kubectl top nodes
```

**Utilité :** Détecter si un Pod utilise trop de ressources et ralentit le réseau.

## Scripts et automatisation

### Script de santé réseau complet

Créez un script qui vérifie tous les aspects du réseau :

```bash
#!/bin/bash

echo "=== Vérification santé réseau Kubernetes ==="

# 1. CoreDNS
echo -e "\n1. Vérification CoreDNS..."
microk8s kubectl get pods -n kube-system -l k8s-app=kube-dns | grep -q "Running"
if [ $? -eq 0 ]; then
    echo "✅ CoreDNS: OK"
else
    echo "❌ CoreDNS: PROBLÈME"
fi

# 2. Calico
echo -e "\n2. Vérification Calico..."
microk8s kubectl get pods -n kube-system -l k8s-app=calico-node | grep -q "Running"
if [ $? -eq 0 ]; then
    echo "✅ Calico: OK"
else
    echo "❌ Calico: PROBLÈME"
fi

# 3. Test DNS
echo -e "\n3. Test résolution DNS..."
microk8s kubectl run test-dns --image=busybox --restart=Never --rm -it -- nslookup kubernetes > /dev/null 2>&1
if [ $? -eq 0 ]; then
    echo "✅ DNS: OK"
else
    echo "❌ DNS: PROBLÈME"
fi

# 4. Test connectivité Pod-to-Pod
echo -e "\n4. Test connectivité Pod-to-Pod..."
# (ajouter votre logique ici)

echo -e "\n=== Fin de la vérification ==="
```

## Outils tiers recommandés

### 1. K9s : tableau de bord terminal

**K9s** est un outil interactif en ligne de commande pour gérer Kubernetes.

```bash
# Installer k9s
snap install k9s

# Lancer k9s pour MicroK8s
k9s --kubeconfig ~/.kube/config
```

**Fonctionnalités :**
- Navigation interactive entre Pods, Services, etc.
- Logs en temps réel
- Port-forward en un clic
- Shell dans les Pods

### 2. Kubectx et Kubens

Changer rapidement de contexte et namespace.

```bash
# Installer
sudo apt install kubectx

# Changer de namespace
kubens production
```

### 3. Stern : logs multi-pods

Voir les logs de plusieurs Pods en même temps.

```bash
# Installer stern
brew install stern  # macOS
# ou télécharger depuis GitHub

# Logs de tous les Pods avec le label app=backend
stern -l app=backend
```

## Commandes de diagnostic rapides

### Référence rapide des commandes essentielles

```bash
# État général
microk8s kubectl get all
microk8s kubectl get events --sort-by='.lastTimestamp'

# Pods
microk8s kubectl get pods -o wide
microk8s kubectl describe pod <nom-pod>
microk8s kubectl logs <nom-pod>

# Services
microk8s kubectl get svc
microk8s kubectl get endpoints <nom-service>
microk8s kubectl describe svc <nom-service>

# DNS
microk8s kubectl exec <pod> -- nslookup <nom-service>
microk8s kubectl exec <pod> -- nslookup kubernetes

# Connectivité
microk8s kubectl exec <pod> -- ping <ip-ou-nom>
microk8s kubectl exec <pod> -- wget -O- http://<service>:<port>
microk8s kubectl exec <pod> -- nc -zv <ip> <port>

# Debugging
microk8s kubectl run test --image=busybox --restart=Never -- sleep 3600
microk8s kubectl run netshoot --image=nicolaka/netshoot --restart=Never -- sleep 3600
microk8s kubectl port-forward pod/<nom-pod> 8080:8080

# Composants système
microk8s kubectl get pods -n kube-system
microk8s kubectl logs -n kube-system -l k8s-app=kube-dns
microk8s kubectl logs -n kube-system -l k8s-app=calico-node
```

## Points clés à retenir

### ✅ Approche méthodique

Toujours tester du plus simple au plus complexe :
1. Les Pods existent ?
2. Ont des IPs ?
3. Communiquent au niveau IP ?
4. DNS fonctionne ?
5. Services configurés ?
6. Application répond ?

### ✅ Outils essentiels

- **kubectl** : Outil principal
- **busybox** : Pod de test léger
- **netshoot** : Arsenal complet pour debugging avancé
- **nslookup/dig** : Test DNS
- **ping/wget/curl** : Test connectivité

### ✅ Les trois piliers du diagnostic

1. **Logs** : Que disent les applications ?
2. **Events** : Que dit Kubernetes ?
3. **Tests réseau** : La connectivité fonctionne-t-elle réellement ?

### ✅ Documentation des tests

Documentez vos tests et résultats pour référence future.

### ✅ Automatisation

Créez des scripts pour automatiser les tests récurrents.

## Pourquoi tout cela est important ?

Tester et diagnostiquer efficacement le réseau vous permet de :

1. **Détecter rapidement les problèmes** avant qu'ils n'impactent les utilisateurs
2. **Réduire le temps de résolution** (MTTR - Mean Time To Resolution)
3. **Comprendre votre infrastructure** en profondeur
4. **Gagner en confiance** lors des déploiements
5. **Former votre équipe** avec des procédures claires

Un bon diagnostic réseau est la différence entre "ça ne marche pas, je ne sais pas pourquoi" et "le problème est X, la solution est Y".

## Prochaines étapes

Avec ces outils et techniques, vous êtes maintenant équipé pour :

- Déployer des applications avec confiance
- Diagnostiquer rapidement les problèmes réseau
- Valider les configurations avant la production
- Former d'autres personnes au debugging Kubernetes

Les prochains chapitres exploreront des sujets plus avancés :

- **8. Tâches planifiées et Batch** : Jobs et CronJobs
- **9. Load Balancing avec MetalLB** : Exposer des Services avec des IPs externes
- **10. Ingress et Routage** : Routage HTTP avancé

Le réseau est maintenant sous contrôle. En avant vers de nouvelles fonctionnalités !

---

**Note importante :** La pratique régulière est la clé. Plus vous utiliserez ces outils, plus vous deviendrez rapide et efficace dans le diagnostic. N'hésitez pas à "casser" volontairement des choses dans un environnement de test pour apprendre à les réparer !

⏭️ [Tâches Planifiées et Batch](/08-taches-planifiees-et-batch/README.md)
