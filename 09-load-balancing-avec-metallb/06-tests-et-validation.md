🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 9.6 Tests et Validation

## Introduction

Déployer un Service LoadBalancer n'est que la moitié du travail. Pour garantir que tout fonctionne correctement, il est essentiel de tester et valider le déploiement. Cette section vous guidera à travers les différentes méthodes et outils pour vérifier que vos Services LoadBalancer fonctionnent comme prévu.

Les tests ne servent pas seulement à détecter les problèmes, ils vous donnent aussi la confiance que votre infrastructure est prête pour la production et qu'elle réagira correctement en cas de panne.

## Niveaux de Test

Les tests d'un Service LoadBalancer se font à plusieurs niveaux :

```
┌─────────────────────────────────────┐
│  Niveau 1 : Infrastructure          │
│  - MetalLB fonctionne               │
│  - IP assignées correctement        │
└─────────────────────────────────────┘
           ↓
┌─────────────────────────────────────┐
│  Niveau 2 : Connectivité            │
│  - IP accessible sur le réseau      │
│  - Ports ouverts                    │
└─────────────────────────────────────┘
           ↓
┌─────────────────────────────────────┐
│  Niveau 3 : Application             │
│  - L'application répond             │
│  - Contenu correct                  │
└─────────────────────────────────────┘
           ↓
┌─────────────────────────────────────┐
│  Niveau 4 : Load Balancing          │
│  - Trafic distribué                 │
│  - Failover fonctionnel             │
└─────────────────────────────────────┘
           ↓
┌─────────────────────────────────────┐
│  Niveau 5 : Performance             │
│  - Temps de réponse acceptable      │
│  - Capacité suffisante              │
└─────────────────────────────────────┘
```

Nous allons explorer chaque niveau en détail.

## Niveau 1 : Tests d'Infrastructure

### Vérifier que MetalLB Fonctionne

**Étape 1 : État de MetalLB**

```bash
# Vérifier que MetalLB est activé
microk8s status | grep metallb
```

Vous devriez voir :
```
metallb: enabled
```

**Étape 2 : Pods MetalLB**

```bash
# Vérifier que les pods MetalLB tournent
microk8s kubectl get pods -n metallb-system
```

Sortie attendue :
```
NAME                          READY   STATUS    RESTARTS   AGE
controller-xxxxx              1/1     Running   0          5d
speaker-xxxxx                 1/1     Running   0          5d
```

**Interprétation** :
- **READY 1/1** : Le pod fonctionne correctement
- **STATUS Running** : Le pod est actif
- **RESTARTS 0** : Pas de crash récent (un nombre élevé indique un problème)

**Étape 3 : Configuration du Pool**

```bash
# Voir le pool d'adresses configuré
microk8s kubectl get ipaddresspool -n metallb-system -o yaml
```

Vérifiez que :
- Les adresses sont correctes
- La plage est suffisante
- La syntaxe est valide

**Étape 4 : Logs MetalLB**

En cas de doute, consultez les logs :

```bash
# Logs du controller
microk8s kubectl logs -n metallb-system -l component=controller --tail=50

# Logs des speakers
microk8s kubectl logs -n metallb-system -l component=speaker --tail=50
```

**Messages normaux** :
```
"msg":"service has IP","service":"default/mon-service","ip":"192.168.1.200"
"msg":"announcing service","service":"default/mon-service","protocol":"layer2"
```

**Messages d'erreur** :
```
"msg":"no available IPs","service":"default/mon-service"
"msg":"invalid IP address format"
```

### Vérifier l'Assignation d'IP

**Liste de tous les Services LoadBalancer** :

```bash
# Voir tous les Services LoadBalancer et leurs IP
microk8s kubectl get svc --all-namespaces -o wide | grep LoadBalancer
```

Exemple de sortie :
```
NAMESPACE   NAME         TYPE           EXTERNAL-IP     PORT(S)         AGE
default     nginx-lb     LoadBalancer   192.168.1.200   80:30123/TCP    2d
default     api-lb       LoadBalancer   192.168.1.201   443:30456/TCP   1d
```

**Points à vérifier** :
- ✅ **EXTERNAL-IP** affiche une IP (pas `<pending>`)
- ✅ L'IP est dans le pool MetalLB configuré
- ✅ L'IP n'est pas en conflit avec une autre ressource

**Pour un Service spécifique** :

```bash
# Voir les détails d'un Service
microk8s kubectl describe svc nginx-lb
```

Cherchez la section :
```
LoadBalancer Ingress:     192.168.1.200
```

### Vérifier les Endpoints

Les endpoints sont les Pods vers lesquels le Service route le trafic.

```bash
# Voir les endpoints d'un Service
microk8s kubectl get endpoints nginx-lb
```

Sortie attendue :
```
NAME       ENDPOINTS                                    AGE
nginx-lb   10.1.1.5:80,10.1.1.6:80,10.1.1.7:80         2d
```

**Interprétation** :
- Chaque endpoint correspond à un Pod
- Si vide, vérifiez que :
  - Les Pods existent : `kubectl get pods -l app=nginx`
  - Les labels correspondent
  - Les Pods sont "ready"

**Détails des endpoints** :

```bash
microk8s kubectl describe endpoints nginx-lb
```

Vous verrez les adresses IP des Pods et leurs ports.

## Niveau 2 : Tests de Connectivité

### Test Ping

Le test le plus basique : est-ce que l'IP répond ?

```bash
# Depuis votre machine ou une autre machine du réseau
ping 192.168.1.200
```

**Résultat attendu** :
```
PING 192.168.1.200 (192.168.1.200) 56(84) bytes of data.
64 bytes from 192.168.1.200: icmp_seq=1 ttl=64 time=0.234 ms
64 bytes from 192.168.1.200: icmp_seq=2 ttl=64 time=0.198 ms
```

**Si le ping échoue** :
- Vérifiez que vous êtes sur le même réseau
- Vérifiez le firewall
- Vérifiez que MetalLB annonce bien l'IP

**Note** : Certaines configurations peuvent bloquer le ping (ICMP) mais autoriser les connexions TCP. C'est normal.

### Test de Port avec Telnet

Vérifiez qu'un port spécifique est ouvert :

```bash
# Test du port 80
telnet 192.168.1.200 80
```

**Si ouvert** :
```
Trying 192.168.1.200...
Connected to 192.168.1.200.
Escape character is '^]'.
```

**Si fermé** :
```
Trying 192.168.1.200...
telnet: Unable to connect to remote host: Connection refused
```

### Test de Port avec Netcat (nc)

Alternative plus flexible que telnet :

```bash
# Test de connexion TCP
nc -zv 192.168.1.200 80
```

**Sortie si ouvert** :
```
Connection to 192.168.1.200 80 port [tcp/http] succeeded!
```

**Test UDP** (si votre Service utilise UDP) :

```bash
nc -zuv 192.168.1.200 53
```

### Test de Port avec Nmap

Pour un scan plus complet :

```bash
# Scanner les ports d'une IP
nmap -p 80,443 192.168.1.200
```

Sortie :
```
PORT    STATE SERVICE
80/tcp  open  http
443/tcp open  https
```

**Scan rapide de tous les ports exposés** :

```bash
nmap 192.168.1.200
```

## Niveau 3 : Tests d'Application

### Tests HTTP avec curl

Le test le plus courant pour les applications web.

**Test basique** :

```bash
# Requête HTTP GET simple
curl http://192.168.1.200
```

**Test avec informations détaillées** :

```bash
# Voir les headers de réponse
curl -I http://192.168.1.200
```

Sortie :
```
HTTP/1.1 200 OK
Server: nginx/1.25.0
Date: Thu, 23 Oct 2025 10:30:00 GMT
Content-Type: text/html
Content-Length: 612
```

**Test avec timeout** :

```bash
# Timeout de 5 secondes
curl --connect-timeout 5 http://192.168.1.200
```

**Test HTTPS (avec certificat auto-signé)** :

```bash
# Ignorer la validation du certificat (-k)
curl -k https://192.168.1.201
```

**Test avec authentification** :

```bash
# Basic auth
curl -u username:password http://192.168.1.200/api

# Bearer token
curl -H "Authorization: Bearer TOKEN" http://192.168.1.200/api
```

**Mesurer le temps de réponse** :

```bash
curl -o /dev/null -s -w "Temps total: %{time_total}s\n" http://192.168.1.200
```

### Tests HTTP avec wget

Alternative à curl :

```bash
# Télécharger la page
wget http://192.168.1.200

# Voir juste les headers
wget --spider -S http://192.168.1.200
```

### Tests d'API REST

**GET Request** :

```bash
curl http://192.168.1.200/api/users
```

**POST Request** :

```bash
curl -X POST \
  -H "Content-Type: application/json" \
  -d '{"name":"John","email":"john@example.com"}' \
  http://192.168.1.200/api/users
```

**PUT Request** :

```bash
curl -X PUT \
  -H "Content-Type: application/json" \
  -d '{"name":"John Updated"}' \
  http://192.168.1.200/api/users/1
```

**DELETE Request** :

```bash
curl -X DELETE http://192.168.1.200/api/users/1
```

### Test de Base de Données

Pour un Service exposant une base de données :

**PostgreSQL** :

```bash
# Test de connexion
psql -h 192.168.1.210 -U postgres -c "SELECT version();"
```

**MySQL/MariaDB** :

```bash
mysql -h 192.168.1.211 -u root -p -e "SELECT 1;"
```

**Redis** :

```bash
redis-cli -h 192.168.1.212 ping
```

**MongoDB** :

```bash
mongo --host 192.168.1.213 --eval "db.version()"
```

### Test depuis un Navigateur

Le test le plus intuitif pour les applications web :

1. Ouvrez votre navigateur
2. Accédez à `http://192.168.1.200`
3. Vérifiez que la page se charge correctement

**Console développeur** (F12) :
- Onglet **Network** : Voir les requêtes HTTP
- Onglet **Console** : Voir les erreurs JavaScript
- Temps de chargement des ressources

## Niveau 4 : Tests de Load Balancing

### Vérifier la Distribution du Trafic

**Méthode 1 : Via les logs des Pods**

Ouvrez plusieurs terminaux et suivez les logs de chaque Pod :

```bash
# Terminal 1
microk8s kubectl logs -f nginx-deployment-xxxxx-aaaaa

# Terminal 2
microk8s kubectl logs -f nginx-deployment-xxxxx-bbbbb

# Terminal 3
microk8s kubectl logs -f nginx-deployment-xxxxx-ccccc
```

Faites plusieurs requêtes :

```bash
# Dans un autre terminal
for i in {1..10}; do
  curl http://192.168.1.200
  sleep 1
done
```

Vous devriez voir les logs apparaître dans différents terminaux, prouvant que le trafic est distribué.

**Méthode 2 : Application affichant le hostname**

Déployez une application qui affiche le nom du Pod :

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: hostname-demo
spec:
  replicas: 3
  selector:
    matchLabels:
      app: hostname-demo
  template:
    metadata:
      labels:
        app: hostname-demo
    spec:
      containers:
      - name: hostname
        image: registry.k8s.io/serve_hostname
        ports:
        - containerPort: 9376
---
apiVersion: v1
kind: Service
metadata:
  name: hostname-demo-lb
spec:
  type: LoadBalancer
  selector:
    app: hostname-demo
  ports:
  - port: 80
    targetPort: 9376
```

Testez :

```bash
# Faire 10 requêtes et voir les différents hostnames
for i in {1..10}; do
  curl http://192.168.1.200
done
```

Sortie attendue (différents noms de Pods) :
```
hostname-demo-xxxxx-aaaaa
hostname-demo-xxxxx-bbbbb
hostname-demo-xxxxx-aaaaa
hostname-demo-xxxxx-ccccc
hostname-demo-xxxxx-bbbbb
...
```

**Méthode 3 : Vérifier les connexions actives**

```bash
# Voir les connexions sur les Pods
microk8s kubectl exec -it <nom-pod> -- netstat -an | grep ESTABLISHED
```

### Tester la Session Affinity

Si vous avez configuré `sessionAffinity: ClientIP` :

```bash
# Faire plusieurs requêtes depuis la même machine
for i in {1..10}; do
  curl http://192.168.1.200
done
```

**Résultat attendu** : Vous devriez toujours obtenir le même Pod.

**Test depuis une IP différente** :

Depuis une autre machine sur le réseau, faites la même chose. Vous devriez possiblement obtenir un Pod différent, mais toujours le même pour cette IP.

## Niveau 5 : Tests de Haute Disponibilité

### Test de Failover (Basculement)

Testez que le Service continue de fonctionner même si un Pod tombe.

**Scénario 1 : Arrêt d'un Pod**

```bash
# Terminal 1 : Surveillance continue
watch -n 1 'curl -s http://192.168.1.200 | head -n 1'

# Terminal 2 : Supprimer un Pod
microk8s kubectl delete pod nginx-deployment-xxxxx-aaaaa
```

**Résultat attendu** :
- Le trafic continue de fonctionner
- Kubernetes crée automatiquement un nouveau Pod
- Après quelques secondes, le nouveau Pod est intégré au Service

**Scénario 2 : Crash simulé**

```bash
# Faire crasher l'application dans un Pod
microk8s kubectl exec nginx-deployment-xxxxx-aaaaa -- kill 1
```

Le Pod redémarre automatiquement (si configuré), et le Service cesse temporairement de lui envoyer du trafic.

### Test de Scalabilité

**Scaler l'application** :

```bash
# Augmenter le nombre de replicas
microk8s kubectl scale deployment nginx-deployment --replicas=5

# Vérifier que les nouveaux Pods reçoivent du trafic
watch microk8s kubectl get endpoints nginx-lb
```

**Test de charge pendant le scaling** :

```bash
# Terminal 1 : Générer du trafic continu
while true; do curl -s http://192.168.1.200 > /dev/null; done

# Terminal 2 : Scaler
microk8s kubectl scale deployment nginx-deployment --replicas=10
```

Le Service devrait continuer de fonctionner sans interruption.

### Test de Résilience du Réseau

**Simuler une latence** (nécessite `tc` - traffic control) :

```bash
# Sur le nœud Kubernetes
sudo tc qdisc add dev eth0 root netem delay 100ms

# Tester l'impact
time curl http://192.168.1.200

# Retirer la latence
sudo tc qdisc del dev eth0 root
```

## Tests de Performance

### Tests de Charge avec Apache Bench (ab)

Apache Bench est un outil simple pour tester la performance.

**Installation** :

```bash
# Ubuntu/Debian
sudo apt-get install apache2-utils

# CentOS/RHEL
sudo yum install httpd-tools
```

**Test basique** :

```bash
# 100 requêtes, 10 connexions simultanées
ab -n 100 -c 10 http://192.168.1.200/
```

**Sortie** :
```
Requests per second:    523.45 [#/sec] (mean)
Time per request:       19.103 [ms] (mean)
Time per request:       1.910 [ms] (mean, across all concurrent requests)
Transfer rate:          156.78 [Kbytes/sec] received

Percentage of the requests served within a certain time (ms)
  50%     18
  66%     19
  75%     20
  80%     21
  90%     24
  95%     28
  98%     35
  99%     42
 100%     67 (longest request)
```

**Test plus intensif** :

```bash
# 10000 requêtes, 100 connexions simultanées
ab -n 10000 -c 100 http://192.168.1.200/
```

### Tests de Charge avec wrk

wrk est plus moderne et performant qu'ab.

**Installation** :

```bash
# Compilation depuis les sources
git clone https://github.com/wg/wrk.git
cd wrk
make
sudo cp wrk /usr/local/bin/
```

**Test basique** :

```bash
# 30 secondes, 10 threads, 100 connexions
wrk -t10 -c100 -d30s http://192.168.1.200/
```

**Sortie** :
```
Running 30s test @ http://192.168.1.200/
  10 threads and 100 connections
  Thread Stats   Avg      Stdev     Max   +/- Stdev
    Latency    12.45ms    8.12ms  89.23ms   74.12%
    Req/Sec   831.45    123.67     1.23k    68.45%
  249234 requests in 30.00s, 72.45MB read
Requests/sec:   8307.80
Transfer/sec:      2.41MB
```

**Test avec script Lua personnalisé** :

```lua
-- script.lua
wrk.method = "POST"
wrk.body   = '{"name":"test"}'
wrk.headers["Content-Type"] = "application/json"
```

```bash
wrk -t4 -c100 -d30s -s script.lua http://192.168.1.200/api/users
```

### Tests de Charge avec hey

hey est un outil moderne écrit en Go.

**Installation** :

```bash
# Télécharger le binaire
wget https://github.com/rakyll/hey/releases/download/v0.1.4/hey_linux_amd64
chmod +x hey_linux_amd64
sudo mv hey_linux_amd64 /usr/local/bin/hey
```

**Test basique** :

```bash
# 1000 requêtes, 50 connexions simultanées
hey -n 1000 -c 50 http://192.168.1.200/
```

**Sortie** :
```
Summary:
  Total:        2.3456 secs
  Slowest:      0.1234 secs
  Fastest:      0.0023 secs
  Average:      0.0145 secs
  Requests/sec: 426.34

Response time histogram:
  0.002 [1]     |
  0.015 [734]   |■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■
  0.027 [189]   |■■■■■■■■■■
  0.040 [58]    |■■■
  ...
```

### Analyse des Ressources sous Charge

**Pendant les tests de charge, surveillez** :

```bash
# Terminal 1 : Utilisation CPU/Mémoire des Pods
watch -n 1 'microk8s kubectl top pods -l app=nginx'

# Terminal 2 : Nombre de Pods ready
watch -n 1 'microk8s kubectl get pods -l app=nginx'

# Terminal 3 : Métriques du Service
watch -n 1 'microk8s kubectl get svc nginx-lb'
```

## Tests Automatisés

### Script de Test Complet

Créez un script shell pour automatiser vos tests :

```bash
#!/bin/bash
# test-loadbalancer.sh

SERVICE_NAME="nginx-lb"
EXTERNAL_IP=$(microk8s kubectl get svc $SERVICE_NAME -o jsonpath='{.status.loadBalancer.ingress[0].ip}')

echo "========================================="
echo "Tests du Service LoadBalancer: $SERVICE_NAME"
echo "IP Externe: $EXTERNAL_IP"
echo "========================================="

# Test 1 : IP assignée
echo -e "\n[Test 1] Vérification de l'IP..."
if [ -z "$EXTERNAL_IP" ]; then
    echo "❌ ÉCHEC : Aucune IP externe assignée"
    exit 1
else
    echo "✅ SUCCÈS : IP $EXTERNAL_IP assignée"
fi

# Test 2 : Ping
echo -e "\n[Test 2] Test de ping..."
if ping -c 3 $EXTERNAL_IP > /dev/null 2>&1; then
    echo "✅ SUCCÈS : IP répond au ping"
else
    echo "⚠️  ATTENTION : Ping échoué (peut être normal si ICMP bloqué)"
fi

# Test 3 : Port HTTP
echo -e "\n[Test 3] Test de connectivité HTTP..."
if nc -zv $EXTERNAL_IP 80 2>&1 | grep -q succeeded; then
    echo "✅ SUCCÈS : Port 80 ouvert"
else
    echo "❌ ÉCHEC : Port 80 inaccessible"
    exit 1
fi

# Test 4 : Réponse HTTP
echo -e "\n[Test 4] Test de réponse HTTP..."
HTTP_CODE=$(curl -s -o /dev/null -w "%{http_code}" http://$EXTERNAL_IP)
if [ "$HTTP_CODE" = "200" ]; then
    echo "✅ SUCCÈS : Application répond (HTTP $HTTP_CODE)"
else
    echo "❌ ÉCHEC : Code HTTP $HTTP_CODE"
    exit 1
fi

# Test 5 : Endpoints
echo -e "\n[Test 5] Vérification des endpoints..."
ENDPOINTS=$(microk8s kubectl get endpoints $SERVICE_NAME -o jsonpath='{.subsets[0].addresses[*].ip}')
ENDPOINT_COUNT=$(echo $ENDPOINTS | wc -w)
if [ $ENDPOINT_COUNT -gt 0 ]; then
    echo "✅ SUCCÈS : $ENDPOINT_COUNT endpoint(s) trouvé(s)"
    echo "   Endpoints: $ENDPOINTS"
else
    echo "❌ ÉCHEC : Aucun endpoint"
    exit 1
fi

# Test 6 : Load Balancing
echo -e "\n[Test 6] Test de distribution du trafic..."
echo "Envoi de 10 requêtes..."
for i in {1..10}; do
    curl -s http://$EXTERNAL_IP > /dev/null
done
echo "✅ SUCCÈS : 10 requêtes envoyées"

echo -e "\n========================================="
echo "Tous les tests ont réussi ! ✅"
echo "========================================="
```

**Utilisation** :

```bash
chmod +x test-loadbalancer.sh
./test-loadbalancer.sh
```

### Tests avec Kubernetes Jobs

Créez des Jobs Kubernetes pour tester depuis l'intérieur du cluster :

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: test-loadbalancer
spec:
  template:
    spec:
      containers:
      - name: curl
        image: curlimages/curl:latest
        command:
        - /bin/sh
        - -c
        - |
          echo "Test du Service nginx-lb..."
          for i in $(seq 1 10); do
            curl -s http://nginx-lb.default.svc.cluster.local
            echo "Requête $i envoyée"
            sleep 1
          done
          echo "Tests terminés avec succès"
      restartPolicy: Never
  backoffLimit: 4
```

Appliquez et vérifiez :

```bash
microk8s kubectl apply -f test-job.yaml
microk8s kubectl logs -f job/test-loadbalancer
```

## Monitoring Continu

### Surveillance avec Watch

Surveillez l'état en temps réel :

```bash
# État du Service
watch -n 2 'microk8s kubectl get svc nginx-lb'

# État des Pods
watch -n 2 'microk8s kubectl get pods -l app=nginx'

# Endpoints
watch -n 2 'microk8s kubectl get endpoints nginx-lb'
```

### Métriques Kubernetes

```bash
# Utilisation des ressources des Pods
microk8s kubectl top pods -l app=nginx

# Utilisation des nœuds
microk8s kubectl top nodes
```

### Logs en Temps Réel

```bash
# Logs agrégés de tous les Pods
microk8s kubectl logs -f -l app=nginx --all-containers=true

# Logs avec timestamps
microk8s kubectl logs -f -l app=nginx --timestamps=true
```

## Checklist de Validation

Avant de déclarer un Service LoadBalancer prêt pour la production, vérifiez :

### Infrastructure
- ✅ MetalLB est actif et sans erreur
- ✅ IP externe assignée depuis le pool
- ✅ Pas de conflits d'IP sur le réseau
- ✅ Configuration du pool correcte

### Connectivité
- ✅ IP accessible en ping (ou ICMP désactivé intentionnellement)
- ✅ Ports ouverts et accessibles
- ✅ Accessible depuis différentes machines du réseau
- ✅ Firewall configuré correctement

### Application
- ✅ Application répond aux requêtes
- ✅ Réponses HTTP correctes (200, 301, etc.)
- ✅ Contenu servi est correct
- ✅ Authentification fonctionne (si applicable)
- ✅ API endpoints fonctionnels (si applicable)

### Load Balancing
- ✅ Trafic distribué entre les Pods
- ✅ Session affinity fonctionne (si configurée)
- ✅ Health checks opérationnels
- ✅ Endpoints à jour

### Haute Disponibilité
- ✅ Service reste fonctionnel si un Pod tombe
- ✅ Nouveaux Pods intégrés automatiquement
- ✅ Failover fonctionne (< 5 secondes)
- ✅ Scaling up/down sans interruption

### Performance
- ✅ Temps de réponse acceptable (< 200ms pour web)
- ✅ Supporte la charge attendue
- ✅ Pas de dégradation sous charge
- ✅ Ressources suffisantes (CPU/RAM)

### Sécurité
- ✅ Seuls les ports nécessaires exposés
- ✅ Certificats SSL valides (si HTTPS)
- ✅ Authentification configurée (si nécessaire)
- ✅ Logs activés pour l'audit

### Documentation
- ✅ IP externe documentée
- ✅ Ports documentés
- ✅ Dépendances listées
- ✅ Procédure de rollback définie

## Résolution de Problèmes lors des Tests

### Le Service ne Répond Pas

**Diagnostic** :

```bash
# 1. Vérifier l'IP
microk8s kubectl get svc mon-service -o wide

# 2. Vérifier les endpoints
microk8s kubectl get endpoints mon-service

# 3. Vérifier les Pods
microk8s kubectl get pods -l app=mon-app

# 4. Tester depuis un Pod du cluster
microk8s kubectl run test-pod --rm -it --image=curlimages/curl -- sh
# Dans le pod: curl http://mon-service
```

### Performances Dégradées

**Vérifications** :

```bash
# Utilisation des ressources
microk8s kubectl top pods -l app=mon-app

# Limites atteintes ?
microk8s kubectl describe pod <pod-name> | grep -A 5 Limits

# Logs d'erreur
microk8s kubectl logs -l app=mon-app --tail=100 | grep -i error
```

### Trafic Non Distribué

**Causes possibles** :

1. **SessionAffinity activée** : Vérifiez `sessionAffinity` dans le Service
2. **Un seul Pod healthy** : Vérifiez les readiness probes
3. **Problème de réseau** : Vérifiez les Network Policies

```bash
# Vérifier la configuration
microk8s kubectl get svc mon-service -o yaml | grep sessionAffinity

# Vérifier les Pods ready
microk8s kubectl get pods -l app=mon-app -o wide
```

## Outils de Test Avancés

### K6 (Grafana k6)

Outil moderne pour les tests de charge avec scripting JavaScript.

```javascript
// test.js
import http from 'k6/http';
import { check, sleep } from 'k6';

export let options = {
  vus: 10,  // 10 utilisateurs virtuels
  duration: '30s',
};

export default function() {
  let res = http.get('http://192.168.1.200');
  check(res, {
    'status is 200': (r) => r.status === 200,
    'response time < 500ms': (r) => r.timings.duration < 500,
  });
  sleep(1);
}
```

```bash
k6 run test.js
```

### Vegeta

Outil de test de charge en ligne de commande.

```bash
# Installation
go install github.com/tsenart/vegeta@latest

# Test : 100 requêtes/seconde pendant 30 secondes
echo "GET http://192.168.1.200" | vegeta attack -rate=100 -duration=30s | vegeta report
```

### Locust

Framework Python pour tests de charge distribués.

```python
# locustfile.py
from locust import HttpUser, task, between

class WebsiteUser(HttpUser):
    wait_time = between(1, 3)
    host = "http://192.168.1.200"

    @task
    def index(self):
        self.client.get("/")

    @task(3)
    def api(self):
        self.client.get("/api/users")
```

```bash
locust -f locustfile.py --host=http://192.168.1.200
```

## Résumé

Les tests et la validation sont essentiels pour garantir que vos Services LoadBalancer fonctionnent correctement. Vous avez appris à :

✅ **Tester l'infrastructure** : MetalLB, IP, configuration
✅ **Tester la connectivité** : Ping, ports, réseau
✅ **Tester l'application** : HTTP, API, bases de données
✅ **Tester le load balancing** : Distribution, failover
✅ **Tester la performance** : Tests de charge, métriques
✅ **Automatiser les tests** : Scripts, Jobs Kubernetes
✅ **Surveiller en continu** : Monitoring, logs, alertes
✅ **Valider avec une checklist** : Prêt pour la production
✅ **Résoudre les problèmes** : Diagnostic et correction

Avec ces compétences en test et validation, vous pouvez déployer des Services LoadBalancer en toute confiance, en sachant qu'ils fonctionneront de manière fiable et performante en production.

---

**Fin du Chapitre 9 : Load Balancing avec MetalLB**

**Prochaine étape** : Ingress et Routage (Chapitre 10)

⏭️ [Ingress et Routage](/10-ingress-et-routage/README.md)
