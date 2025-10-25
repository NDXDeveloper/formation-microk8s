🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 19.7 Tests de charge

## Introduction aux tests de charge

Les **tests de charge** (load testing) consistent à simuler un grand nombre d'utilisateurs ou de requêtes pour vérifier comment votre application et votre autoscaling réagissent sous la pression. C'est une étape **essentielle** pour valider que vos configurations HPA, VPA, et métriques fonctionnent correctement.

### Pourquoi tester la charge ?

Imaginez que vous avez configuré un HPA magnifique :

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: mon-api-hpa
spec:
  minReplicas: 2
  maxReplicas: 20
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 50
```

**Questions importantes :**

- ✅ Le HPA scale-t-il vraiment quand la charge augmente ?
- ✅ Scale-t-il assez vite ?
- ✅ Scale-t-il au bon moment (pas trop tôt, pas trop tard) ?
- ✅ L'application reste-t-elle performante pendant le scaling ?
- ✅ Le scale down fonctionne-t-il correctement après le pic ?
- ✅ Les seuils sont-ils bien calibrés ?

**Sans tests de charge, vous ne le saurez jamais !**

### Analogie simple

**Sans tests de charge :**
- C'est comme installer un système anti-incendie dans une maison mais ne jamais vérifier qu'il fonctionne
- Vous pensez être protégé, mais vous n'avez aucune garantie

**Avec tests de charge :**
- Vous simulez un incendie (de manière contrôlée)
- Vous vérifiez que les détecteurs se déclenchent
- Vous vérifiez que l'eau arrive bien
- Vous identifiez les problèmes avant qu'ils ne soient critiques

Les tests de charge sont votre "simulation d'incendie" pour l'autoscaling !

## Types de tests de charge

Il existe plusieurs types de tests selon vos objectifs :

### 1. Test de montée en charge (Ramp-up test)

Augmentation progressive de la charge pour observer le comportement.

```
Charge
  ↑
  |                    ████████████
  |              ██████
  |        ██████
  |  ██████
  |██
  └─────────────────────────────────> Temps
    0    2    4    6    8   10 min
```

**Objectif :** Voir comment le HPA réagit à une augmentation progressive.

### 2. Test de pic brutal (Spike test)

Augmentation soudaine de la charge.

```
Charge
  ↑
  |  ████████████████████
  |██                    ████
  |█                        ████
  └─────────────────────────────────> Temps
    0    2    4    6    8   10 min
```

**Objectif :** Voir si le système peut gérer un pic soudain (flash sale, événement viral).

### 3. Test de charge soutenue (Soak test)

Charge constante pendant une longue période.

```
Charge
  ↑
  |  ████████████████████████████████
  |██                                ██
  |█                                  █
  └─────────────────────────────────> Temps
    0    1h   2h   3h   4h   5h
```

**Objectif :** Détecter les fuites mémoire, les dégradations progressives.

### 4. Test de stress (Stress test)

Augmentation jusqu'à la rupture pour trouver les limites.

```
Charge
  ↑
  |                          ████ CRASH!
  |                    ██████
  |              ██████
  |        ██████
  |  ██████
  |██
  └─────────────────────────────────> Temps
    0    2    4    6    8   10 min
```

**Objectif :** Trouver le point de rupture, valider que le système ne tombe pas brutalement.

## Outils de tests de charge

### 1. Apache Bench (ab) - Simple et rapide

**Installation :**

```bash
# Ubuntu/Debian
sudo apt install apache2-utils

# macOS
# Déjà installé par défaut

# MicroK8s (dans un pod)
kubectl run ab --image=httpd:alpine --rm -it -- sh
apk add apache2-utils
```

**Utilisation basique :**

```bash
ab -n 10000 -c 100 http://mon-api.example.com/
```

**Paramètres :**
- `-n 10000` : Nombre total de requêtes (10,000)
- `-c 100` : Nombre de requêtes concurrentes (100 en parallèle)

**Sortie :**

```
Benchmarking mon-api.example.com (be patient)
Completed 1000 requests
Completed 2000 requests
...
Completed 10000 requests
Finished 10000 requests

Server Software:        nginx/1.21
Server Hostname:        mon-api.example.com
Server Port:            80

Concurrency Level:      100
Time taken for tests:   12.456 seconds
Complete requests:      10000
Failed requests:        0
Total transferred:      2450000 bytes
HTML transferred:       1250000 bytes
Requests per second:    802.84 [#/sec] (mean)
Time per request:       124.558 [ms] (mean)
Time per request:       1.246 [ms] (mean, across all concurrent requests)
Transfer rate:          192.05 [Kbytes/sec] received

Connection Times (ms)
              min  mean[+/-sd] median   max
Connect:        1    5   2.1      4      15
Processing:    15  118  45.2    110     450
Waiting:       12  115  44.8    108     445
Total:         20  123  45.8    115     455

Percentage of the requests served within a certain time (ms)
  50%    115
  66%    130
  75%    145
  80%    155
  90%    180
  95%    220
  98%    280
  99%    350
 100%    455 (longest request)
```

**Avantages :**
- ✅ Très simple à utiliser
- ✅ Installé sur la plupart des systèmes
- ✅ Résultats faciles à lire

**Inconvénients :**
- ❌ Limité aux requêtes HTTP GET simples
- ❌ Pas de scénarios complexes
- ❌ Pas de graphiques

### 2. Hey - Ab moderne

**Installation :**

```bash
# Via Go
go install github.com/rakyll/hey@latest

# Ou télécharger le binaire
wget https://hey-release.s3.us-east-2.amazonaws.com/hey_linux_amd64
chmod +x hey_linux_amd64
sudo mv hey_linux_amd64 /usr/local/bin/hey
```

**Utilisation :**

```bash
# Test de 5 minutes avec 50 workers
hey -z 5m -c 50 http://mon-api.example.com/
```

**Paramètres utiles :**
- `-z 5m` : Duration (5 minutes)
- `-c 50` : Concurrency (50 workers)
- `-q 100` : Rate limit (100 requêtes/sec max)
- `-m POST` : Méthode HTTP
- `-H "Authorization: Bearer token"` : Header custom

**Exemple avec POST et body :**

```bash
hey -z 2m -c 30 -m POST \
  -H "Content-Type: application/json" \
  -d '{"name":"test","value":123}' \
  http://mon-api.example.com/api/data
```

**Sortie :**

```
Summary:
  Total:        300.0534 secs
  Slowest:      1.2456 secs
  Fastest:      0.0234 secs
  Average:      0.1567 secs
  Requests/sec: 159.97

  Total data:   2400000 bytes
  Size/request: 50 bytes

Response time histogram:
  0.023 [1]     |
  0.145 [12543] |■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■
  0.267 [23456] |■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■
  0.389 [8754]  |■■■■■■■■■■■■■■■■■■■■■■■■■■■
  0.511 [3214]  |■■■■■■■■■■
  0.633 [876]   |■■■
  0.755 [234]   |■
  0.877 [87]    |
  0.999 [32]    |
  1.121 [12]    |
  1.246 [1]     |

Latency distribution:
  10% in 0.0856 secs
  25% in 0.1234 secs
  50% in 0.1567 secs
  75% in 0.2134 secs
  90% in 0.3456 secs
  95% in 0.4567 secs
  99% in 0.7890 secs

Status code distribution:
  [200] 48000 responses
```

**Avantages :**
- ✅ Plus moderne et flexible que ab
- ✅ Supporte POST, PUT, headers custom
- ✅ Histogrammes et percentiles
- ✅ Rate limiting intégré

**Inconvénients :**
- ❌ Pas de scénarios complexes
- ❌ Installation supplémentaire nécessaire

### 3. K6 - Tests avancés

**Installation :**

```bash
# Ubuntu/Debian
sudo gpg -k
sudo gpg --no-default-keyring --keyring /usr/share/keyrings/k6-archive-keyring.gpg --keyserver hkp://keyserver.ubuntu.com:80 --recv-keys C5AD17C747E3415A3642D57D77C6C491D6AC1D69
echo "deb [signed-by=/usr/share/keyrings/k6-archive-keyring.gpg] https://dl.k6.io/deb stable main" | sudo tee /etc/apt/sources.list.d/k6.list
sudo apt-get update
sudo apt-get install k6

# macOS
brew install k6

# Docker
docker pull grafana/k6
```

**Script de test (script.js) :**

```javascript
import http from 'k6/http';
import { check, sleep } from 'k6';

export let options = {
  stages: [
    { duration: '2m', target: 10 },   // Ramp up to 10 users
    { duration: '5m', target: 50 },   // Stay at 50 users
    { duration: '2m', target: 100 },  // Ramp up to 100 users
    { duration: '5m', target: 100 },  // Stay at 100 users
    { duration: '2m', target: 0 },    // Ramp down to 0 users
  ],
  thresholds: {
    'http_req_duration': ['p(95)<500'], // 95% des requêtes < 500ms
    'http_req_failed': ['rate<0.01'],   // Moins de 1% d'erreurs
  },
};

export default function () {
  let response = http.get('http://mon-api.example.com/');

  check(response, {
    'status is 200': (r) => r.status === 200,
    'response time < 500ms': (r) => r.timings.duration < 500,
  });

  sleep(1); // Pause entre les requêtes
}
```

**Exécution :**

```bash
k6 run script.js
```

**Sortie :**

```
          /\      |‾‾| /‾‾/   /‾‾/
     /\  /  \     |  |/  /   /  /
    /  \/    \    |     (   /   ‾‾\
   /          \   |  |\  \ |  (‾)  |
  / __________ \  |__| \__\ \_____/ .io

  execution: local
     script: script.js
     output: -

  scenarios: (100.00%) 1 scenario, 100 max VUs, 18m0s max duration
           * default: Up to 100 looping VUs for 16m0s over 5 stages

running (16m00.2s), 000/100 VUs, 45832 complete and 0 interrupted iterations
default ✓ [======================================] 000/100 VUs  16m0s

     ✓ status is 200
     ✓ response time < 500ms

     checks.........................: 100.00% ✓ 91664      ✗ 0
     data_received..................: 229 MB  238 kB/s
     data_sent......................: 4.1 MB  4.3 kB/s
     http_req_blocked...............: avg=1.2ms    min=1µs     med=3µs      max=234ms   p(90)=5µs      p(95)=7µs
     http_req_connecting............: avg=567µs    min=0s      med=0s       max=123ms   p(90)=0s       p(95)=0s
     http_req_duration..............: avg=123.45ms min=23.12ms med=98.76ms  max=987ms   p(90)=234.56ms p(95)=345.67ms
     http_req_failed................: 0.00%   ✓ 0          ✗ 45832
     http_req_receiving.............: avg=234µs    min=12µs    med=156µs    max=23ms    p(90)=456µs    p(95)=678µs
     http_req_sending...............: avg=45µs     min=4µs     med=23µs     max=5ms     p(90)=67µs     p(95)=89µs
     http_req_tls_handshaking.......: avg=0s       min=0s      med=0s       max=0s      p(90)=0s       p(95)=0s
     http_req_waiting...............: avg=123.17ms min=23.08ms med=98.54ms  max=986ms   p(90)=234.12ms p(95)=345.23ms
     http_reqs......................: 45832   47.8/s
     iteration_duration.............: avg=1.12s    min=1.02s   med=1.09s    max=1.98s   p(90)=1.23s    p(95)=1.34s
     iterations.....................: 45832   47.8/s
     vus............................: 1       min=1        max=100
     vus_max........................: 100     min=100      max=100
```

**Avantages :**
- ✅ Scénarios complexes (ramping, stages)
- ✅ Scripting JavaScript complet
- ✅ Thresholds et checks intégrés
- ✅ Métriques détaillées
- ✅ Peut envoyer les résultats à Grafana Cloud

**Inconvénients :**
- ❌ Plus complexe à apprendre
- ❌ Nécessite l'écriture de scripts

### 4. Locust - Tests avec interface web

**Installation :**

```bash
pip install locust
```

**Script de test (locustfile.py) :**

```python
from locust import HttpUser, task, between

class MyUser(HttpUser):
    wait_time = between(1, 3)  # Attente entre 1 et 3 secondes

    @task(3)  # Poids : 3x plus fréquent
    def get_homepage(self):
        self.client.get("/")

    @task(1)  # Poids : 1x
    def get_api_data(self):
        self.client.get("/api/data")

    @task(2)  # Poids : 2x
    def post_data(self):
        self.client.post("/api/submit", json={
            "name": "test",
            "value": 123
        })

    def on_start(self):
        # Exécuté au démarrage (login, etc.)
        self.client.post("/login", json={
            "username": "test",
            "password": "test123"
        })
```

**Lancer Locust :**

```bash
locust -f locustfile.py --host=http://mon-api.example.com
```

Puis ouvrez http://localhost:8089 dans votre navigateur.

**Interface web :**
- Définir le nombre d'utilisateurs (ex: 100)
- Définir le taux d'apparition (ex: 10 users/sec)
- Démarrer le test
- Voir les graphiques en temps réel !

**Avantages :**
- ✅ Interface web intuitive
- ✅ Graphiques en temps réel
- ✅ Scénarios Python flexibles
- ✅ Distribution sur plusieurs machines possible

**Inconvénients :**
- ❌ Nécessite Python
- ❌ Plus lourd que hey/ab

### 5. Tests depuis Kubernetes (Busybox/Curl)

Pour des tests rapides depuis le cluster :

```bash
# Pod busybox interactif
kubectl run -i --tty load-generator --rm --image=busybox --restart=Never -- /bin/sh

# Dans le pod
while true; do
  wget -q -O- http://mon-service.default.svc.cluster.local/
done
```

Ou un Job Kubernetes :

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: load-test
spec:
  parallelism: 10  # 10 pods en parallèle
  completions: 100 # 100 exécutions au total
  template:
    spec:
      containers:
      - name: curl
        image: curlimages/curl
        command:
        - sh
        - -c
        - |
          for i in $(seq 1 100); do
            curl -s http://mon-service.default.svc.cluster.local/ > /dev/null
            sleep 0.1
          done
      restartPolicy: Never
```

## Méthodologie de test

### Phase 1 : Préparation

**1. Définir les objectifs**

Qu'est-ce que vous voulez tester ?
- ✅ Le HPA scale-t-il correctement ?
- ✅ À quel seuil de charge scale-t-il ?
- ✅ La latence reste-t-elle acceptable ?
- ✅ Quelle est la capacité maximale ?

**2. Identifier les endpoints à tester**

```
Endpoints critiques :
- GET  /                      (homepage)
- GET  /api/products          (liste)
- GET  /api/products/:id      (détail)
- POST /api/orders            (création)
- GET  /api/orders/:id        (statut)
```

**3. Préparer l'environnement**

```bash
# Vérifier l'état initial
kubectl get hpa
kubectl get pods
kubectl top nodes
kubectl top pods

# S'assurer que Metrics Server fonctionne
kubectl top nodes
```

**4. Préparer les outils de monitoring**

Ouvrez plusieurs terminaux :

**Terminal 1 : Observer le HPA**
```bash
watch -n 2 kubectl get hpa
```

**Terminal 2 : Observer les pods**
```bash
watch -n 2 kubectl get pods
```

**Terminal 3 : Observer les métriques**
```bash
watch -n 5 kubectl top pods
```

**Terminal 4 : Lancer le test de charge**
```bash
hey -z 10m -c 50 http://mon-api.example.com/
```

### Phase 2 : Test de base (Baseline)

Commencez avec une charge faible pour établir une référence.

```bash
# Charge faible : 10 requêtes/sec pendant 2 minutes
hey -z 2m -q 10 http://mon-api.example.com/
```

**Observer :**
- Latence moyenne
- Taux d'erreurs
- Utilisation CPU/mémoire
- Nombre de pods (doit rester stable)

**Résultat attendu :** Tout est stable, pas de scaling.

### Phase 3 : Test de montée en charge

Augmentez progressivement la charge.

```bash
# Étape 1 : 50 requêtes/sec
hey -z 3m -q 50 http://mon-api.example.com/

# Étape 2 : 100 requêtes/sec
hey -z 3m -q 100 http://mon-api.example.com/

# Étape 3 : 200 requêtes/sec
hey -z 3m -q 200 http://mon-api.example.com/
```

**Observer :**

**Dans le terminal HPA :**
```
NAME      REFERENCE        TARGETS    MINPODS   MAXPODS   REPLICAS   AGE
my-hpa    Deployment/app   25%/50%    2         10        2          5m

# 2 minutes plus tard...
NAME      REFERENCE        TARGETS    MINPODS   MAXPODS   REPLICAS   AGE
my-hpa    Deployment/app   65%/50%    2         10        4          7m

# Encore 2 minutes...
NAME      REFERENCE        TARGETS    MINPODS   MAXPODS   REPLICAS   AGE
my-hpa    Deployment/app   48%/50%    2         10        6          9m
```

**Questions à se poser :**
- ✅ À quel moment le HPA a-t-il commencé à scaler ?
- ✅ Le scaling est-il suffisamment rapide ?
- ✅ L'utilisation CPU est-elle revenue proche de la cible ?
- ✅ La latence est-elle restée acceptable ?

### Phase 4 : Test de pic

Simulez un pic soudain de trafic.

```bash
# Pic brutal : 500 requêtes/sec pendant 5 minutes
hey -z 5m -q 500 http://mon-api.example.com/
```

**Observer :**
- Le HPA réagit-il assez vite ?
- Y a-t-il des erreurs pendant le pic ?
- Combien de temps pour stabiliser ?

**Événements typiques :**

```
0:00 - Début du test, 2 pods, CPU 20%
0:30 - CPU monte à 85%, HPA détecte
1:00 - HPA crée 2 nouveaux pods (total: 4)
1:30 - Pods démarrent, CPU à 60%
2:00 - HPA crée encore 2 pods (total: 6)
2:30 - CPU stable à 45%
5:00 - Fin du test
7:00 - HPA commence à scale down
12:00 - Retour à 2 pods
```

### Phase 5 : Test de descente

Arrêtez la charge et observez le scale down.

```bash
# Arrêter le test
# Attendre et observer
watch -n 10 kubectl get hpa
```

**Points d'attention :**
- Le scale down prend **plus de temps** que le scale up (par design)
- Par défaut : 5 minutes avant de scale down
- C'est normal et souhaitable (évite les oscillations)

### Phase 6 : Test de stress (optionnel)

Trouvez les limites du système.

```bash
# Augmentation progressive jusqu'à la rupture
hey -z 1m -q 100 http://mon-api.example.com/   # 100 req/s
hey -z 1m -q 200 http://mon-api.example.com/   # 200 req/s
hey -z 1m -q 500 http://mon-api.example.com/   # 500 req/s
hey -z 1m -q 1000 http://mon-api.example.com/  # 1000 req/s
hey -z 1m -q 2000 http://mon-api.example.com/  # 2000 req/s
```

**Continuez jusqu'à observer :**
- Taux d'erreurs qui augmente significativement (>1%)
- Latence qui explose (>5 secondes)
- HPA qui atteint maxReplicas mais ne peut plus suivre
- Cluster qui manque de ressources

**Résultat :** Vous connaissez maintenant la capacité maximale de votre système !

## Interpréter les résultats

### Métriques clés à surveiller

**1. Latence (Response Time)**

```
p50 (médiane):  125ms   ← 50% des requêtes
p95:            340ms   ← 95% des requêtes
p99:            850ms   ← 99% des requêtes
```

**Bon :** p95 < 500ms, p99 < 1000ms
**Problème :** p95 > 1000ms, p99 > 5000ms

**2. Taux d'erreurs (Error Rate)**

```
Total requests: 10000
Failed:         15
Error rate:     0.15%
```

**Bon :** < 0.1%
**Acceptable :** < 1%
**Problème :** > 5%

**3. Throughput (Débit)**

```
Requests per second: 247.5
```

**Comparez avec vos objectifs :**
- Objectif : 500 req/s → **Problème** (seulement 247)
- Objectif : 200 req/s → **Bon** (dépasse l'objectif)

**4. Utilisation des ressources**

```bash
kubectl top pods
```

```
NAME                CPU      MEMORY
app-xxxxx-abcd      450m     256Mi    # 90% du request (500m)
app-xxxxx-efgh      480m     280Mi    # 96% du request
app-xxxxx-ijkl      420m     240Mi    # 84% du request
```

**Bon :** 50-70% d'utilisation (marge pour les pics)
**Problème :** >90% (risque de throttling/OOM)

### Signes que tout fonctionne bien

✅ **Le HPA scale up quand nécessaire**
```
Charge augmente → CPU > 50% → HPA ajoute des pods → CPU redescend vers 50%
```

✅ **La latence reste stable**
```
p95 avant scaling:  450ms
p95 après scaling:  480ms   (acceptable, <20% augmentation)
```

✅ **Pas (ou très peu) d'erreurs**
```
Error rate: 0.05%   (excellent)
```

✅ **Le scale down fonctionne**
```
Charge diminue → Après 5-10min → HPA retire des pods
```

### Signes de problèmes

❌ **HPA ne scale pas du tout**

```
TARGETS: 85%/50%   REPLICAS: 2/2   (reste à 2 malgré 85% CPU)
```

**Causes possibles :**
- Metrics Server ne fonctionne pas
- Pas de requests définis dans les pods
- HPA mal configuré

❌ **HPA scale mais trop lentement**

```
0:00 - CPU 80%
3:00 - CPU 85%, toujours 2 pods
6:00 - Enfin, scaling à 4 pods
```

**Solutions :**
- Réduire le seuil (de 50% à 40%)
- Augmenter minReplicas pour avoir plus de marge

❌ **Latence explose pendant le scaling**

```
p95: 450ms → 3500ms pendant 2 minutes → 500ms
```

**Causes :**
- Nouveaux pods trop lents à démarrer
- Probes mal configurées
- Image trop lourde

**Solutions :**
- Optimiser le startup (readinessProbe)
- Utiliser des images plus légères
- Pré-scaler avant les pics connus

❌ **Oscillations (flapping)**

```
10:00 - 2 pods
10:03 - 5 pods
10:08 - 2 pods
10:11 - 5 pods
```

**Causes :**
- Seuil trop proche de l'utilisation réelle
- Fenêtre de stabilisation trop courte

**Solutions :**
- Augmenter la fenêtre de stabilisation pour scale down
- Ajuster les seuils avec plus de marge

❌ **Cluster saturé**

```
HPA recommande 20 pods mais reste à 10
Events: FailedScheduling - Insufficient cpu
```

**Cause :** Pas assez de ressources dans le cluster

**Solutions :**
- Ajouter des nœuds (scale up le cluster)
- Optimiser les requests des pods
- Utiliser Cluster Autoscaler (section 19.4)

## Scénarios de test réalistes

### Scénario 1 : E-commerce - Vente flash

Simuler une vente flash avec pic soudain :

```bash
# Trafic normal (2 min)
hey -z 2m -q 50 https://boutique.example.com/

# Vente flash annoncée ! (pic brutal - 5 min)
hey -z 5m -q 500 https://boutique.example.com/products/promo

# Après la vente (retour à la normale - 3 min)
hey -z 3m -q 80 https://boutique.example.com/
```

**Objectifs :**
- Latence < 1s même pendant le pic
- 0 erreurs
- HPA scale rapidement (< 2 minutes)

### Scénario 2 : API - Montée progressive journée type

Simuler une journée typique :

```bash
# Nuit (charge faible - 5 min)
hey -z 5m -q 10 https://api.example.com/

# Matin (montée - 5 min)
hey -z 5m -q 100 https://api.example.com/

# Midi (pic - 5 min)
hey -z 5m -q 300 https://api.example.com/

# Après-midi (charge moyenne - 5 min)
hey -z 5m -q 150 https://api.example.com/

# Soir (descente - 5 min)
hey -z 5m -q 50 https://api.example.com/
```

**Objectifs :**
- Utilisation CPU stable autour de 50-60%
- HPA adapte le nombre de pods progressivement
- Coûts optimisés (scale down le soir)

### Scénario 3 : Webhook - Rafales irrégulières

Simuler des webhooks qui arrivent par rafales :

```bash
# Script K6 avec pattern irrégulier
cat <<EOF > webhook-test.js
import http from 'k6/http';
import { sleep } from 'k6';

export let options = {
  stages: [
    { duration: '1m', target: 10 },
    { duration: '30s', target: 100 },  // Rafale 1
    { duration: '2m', target: 10 },
    { duration: '30s', target: 150 },  // Rafale 2
    { duration: '2m', target: 10 },
    { duration: '30s', target: 200 },  // Rafale 3
    { duration: '1m', target: 0 },
  ],
};

export default function () {
  http.post('https://webhook.example.com/events', JSON.stringify({
    event: 'order.created',
    data: { order_id: Math.floor(Math.random() * 10000) }
  }), {
    headers: { 'Content-Type': 'application/json' },
  });
  sleep(Math.random() * 2);
}
EOF

k6 run webhook-test.js
```

**Objectifs :**
- HPA réagit aux rafales
- Aucun webhook perdu
- Scale down entre les rafales

## Bonnes pratiques

### 1. Tester en environnement de dev/staging d'abord

**Ne testez jamais directement en production !**

```bash
# Dev/Staging
hey -z 10m -c 100 https://staging.example.com/

# Analyser les résultats
# Ajuster la configuration
# Répéter jusqu'à satisfaction

# Seulement ensuite : Production
```

### 2. Commencer petit, augmenter progressivement

```bash
# Mauvais : 10 000 utilisateurs d'un coup
hey -z 1m -c 10000 https://api.example.com/

# Bon : Progression
hey -z 2m -c 10 https://api.example.com/     # 10 users
hey -z 2m -c 50 https://api.example.com/     # 50 users
hey -z 2m -c 100 https://api.example.com/    # 100 users
hey -z 2m -c 500 https://api.example.com/    # 500 users
```

### 3. Tester différents endpoints

Ne testez pas uniquement `/` :

```bash
# Test sur plusieurs endpoints
hey -z 5m -c 50 https://api.example.com/ &
hey -z 5m -c 30 https://api.example.com/api/products &
hey -z 5m -c 20 https://api.example.com/api/orders &
wait
```

### 4. Inclure des données réalistes

```bash
# Mauvais : toujours les mêmes données
hey -z 5m -d '{"id":1}' https://api.example.com/

# Bon : données variées (avec script K6/Locust)
for i in $(seq 1 100); do
  curl -X POST https://api.example.com/api/orders \
    -H "Content-Type: application/json" \
    -d "{\"product_id\": $((RANDOM % 1000)), \"quantity\": $((RANDOM % 10))}"
  sleep 0.1
done
```

### 5. Mesurer avant et après les changements

```bash
# Baseline
hey -z 5m -c 100 https://api.example.com/ > baseline.txt

# Faire des modifications (ajuster HPA, optimiser code, etc.)

# Re-test
hey -z 5m -c 100 https://api.example.com/ > after-changes.txt

# Comparer
diff baseline.txt after-changes.txt
```

### 6. Automatiser les tests

Intégrez les tests de charge dans votre CI/CD :

```yaml
# .gitlab-ci.yml
load-test:
  stage: test
  script:
    - hey -z 2m -c 50 https://staging.example.com/ > results.txt
    - |
      if grep -q "Failed requests:.*[1-9]" results.txt; then
        echo "Load test failed: errors detected"
        exit 1
      fi
    - |
      AVG_LATENCY=$(grep "Average:" results.txt | awk '{print $2}')
      if (( $(echo "$AVG_LATENCY > 500" | bc -l) )); then
        echo "Load test failed: average latency > 500ms"
        exit 1
      fi
  artifacts:
    paths:
      - results.txt
```

### 7. Documenter les résultats

Conservez un historique :

```bash
# Structure de dossiers
load-tests/
├── 2025-10-25-baseline/
│   ├── results.txt
│   ├── hpa-state.txt
│   └── notes.md
├── 2025-11-01-after-optimization/
│   ├── results.txt
│   ├── hpa-state.txt
│   └── notes.md
└── README.md
```

**notes.md exemple :**

```markdown
# Load Test - 2025-10-25

## Configuration
- HPA: min=2, max=10, target=50% CPU
- Instance: 2 CPU, 4GB RAM per pod
- Metrics Server: enabled

## Résultats
- Baseline: 150 req/s, p95=450ms
- With 100 concurrent users:
  - Throughput: 245 req/s
  - p95 latency: 680ms
  - Error rate: 0.02%
  - HPA scaled from 2 to 6 pods in 3 minutes

## Observations
- HPA réagit bien
- Latence acceptable
- Scale down après 8 minutes

## Actions
- ✅ Prêt pour production
- Considérer minReplicas=3 pour plus de marge
```

### 8. Surveiller le cluster pendant les tests

Pendant les tests, vérifiez aussi :

```bash
# Utilisation des nœuds
kubectl top nodes

# Événements du cluster
kubectl get events --sort-by=.metadata.creationTimestamp | tail -20

# Logs des pods (erreurs ?)
kubectl logs -l app=mon-api --tail=50

# État du réseau
kubectl get svc
kubectl describe svc mon-api
```

## Outils de monitoring pendant les tests

### Grafana dashboards

Si vous avez Prometheus + Grafana (section 12-13), créez un dashboard pour les tests de charge :

**Métriques à afficher :**
- Nombre de pods (gauge)
- Requêtes par seconde (graph)
- Latence p50, p95, p99 (graph)
- Taux d'erreurs (graph)
- Utilisation CPU par pod (heatmap)
- Utilisation mémoire par pod (heatmap)

### Terminal multiplexé

Utilisez `tmux` ou `screen` pour voir tout en même temps :

```
┌─────────────────────┬─────────────────────┐
│ watch kubectl get   │ watch kubectl top   │
│ hpa                 │ pods                │
├─────────────────────┼─────────────────────┤
│ watch kubectl get   │ hey -z 10m -c 100   │
│ pods                │ https://api.com     │
└─────────────────────┴─────────────────────┘
```

Configuration tmux :

```bash
tmux new-session \; \
  split-window -h \; \
  split-window -v \; \
  select-pane -t 0 \; \
  split-window -v \; \
  select-pane -t 0 \; \
  send-keys 'watch -n 2 kubectl get hpa' C-m \; \
  select-pane -t 1 \; \
  send-keys 'watch -n 2 kubectl get pods' C-m \; \
  select-pane -t 2 \; \
  send-keys 'watch -n 5 kubectl top pods' C-m \; \
  select-pane -t 3
```

## Checklist de test de charge

Avant de considérer que votre autoscaling est prêt pour la production :

### Tests de base
- [ ] Test de charge faible (baseline établi)
- [ ] Test de montée progressive
- [ ] Test de pic soudain
- [ ] Test de descente (scale down)
- [ ] Test de durée (soak test 1h+)

### Validations
- [ ] HPA scale up correctement
- [ ] HPA scale down correctement
- [ ] Latence reste acceptable (p95 < objectif)
- [ ] Taux d'erreurs < 0.1%
- [ ] Pas d'oscillations (flapping)
- [ ] Capacité maximale identifiée
- [ ] Temps de réponse du scaling < 3 minutes

### Métriques documentées
- [ ] Throughput maximal connu
- [ ] Latence à différentes charges mesurée
- [ ] Configuration HPA optimale documentée
- [ ] Ressources par pod calibrées (requests/limits)

### Outils configurés
- [ ] Monitoring en place (Prometheus/Grafana)
- [ ] Alertes configurées
- [ ] Tests automatisés dans CI/CD
- [ ] Runbook de réponse aux incidents

## Résumé

Les **tests de charge** sont essentiels pour valider que votre autoscaling fonctionne comme prévu.

**Points clés :**

1. Testez **avant** la production, jamais directement en prod
2. Commencez avec des charges faibles, augmentez progressivement
3. Utilisez les bons outils : hey/ab pour le simple, K6/Locust pour l'avancé
4. Observez simultanément : HPA, pods, métriques, latence, erreurs
5. Différents types de tests : baseline, ramp-up, spike, soak, stress
6. Interprétez les résultats : latence (p95, p99), erreurs, throughput, scaling
7. Documentez tout : configurations, résultats, observations, actions
8. Automatisez les tests dans votre CI/CD

**Outils recommandés selon le contexte :**

| Besoin | Outil recommandé |
|--------|------------------|
| Test rapide simple | Apache Bench (ab) |
| Tests HTTP modernes | Hey |
| Scénarios complexes | K6 |
| Interface web + Python | Locust |
| Tests internes au cluster | Busybox/curl |

**Progression recommandée :**

1. **Débutant** : Utilisez `hey` avec des commandes simples
2. **Intermédiaire** : Créez des scripts K6 avec stages
3. **Avancé** : Automatisez dans CI/CD, tests continus

**Citation importante :**

> "Le meilleur moment pour tester votre autoscaling est AVANT le Black Friday, pas PENDANT !"

Les tests de charge vous permettent de dormir tranquille en sachant que votre système peut gérer la charge. C'est un investissement de temps qui rapporte énormément en fiabilité et confiance !

---

**Félicitations !** Vous avez maintenant une compréhension complète du scaling et autoscaling dans Kubernetes, du manuel (19.1) aux tests de validation (19.7). Vous êtes prêt à gérer des applications scalables en production !

⏭️ [Ordonnancement Avancé](/20-ordonnancement-avance/README.md)
