🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 15.6 Synthetic Monitoring avec Blackbox Exporter

## Introduction au Synthetic Monitoring

Jusqu'à présent, nous avons vu trois piliers de l'observabilité :
- **Métriques** : Mesures internes de vos applications
- **Logs** : Événements textuels de vos applications
- **Traces** : Parcours des requêtes dans vos applications

Mais il manque une dimension importante : **le point de vue utilisateur**.

### Le Problème du Monitoring "De l'Intérieur"

**Scénario** : Votre monitoring vous dit que tout va bien.

```
✅ Tous les pods sont UP
✅ CPU et mémoire normaux
✅ Aucune erreur dans les logs
✅ Latence API : 50ms

Mais... vos utilisateurs vous disent : "Le site ne marche pas !" 😱
```

**Que s'est-il passé ?**

Possibilités :
- Le certificat SSL a expiré
- Le DNS ne résout plus
- Le firewall bloque le trafic externe
- L'Ingress est mal configuré
- Un service externe (API tierce) est down

Tous ces problèmes sont **invisibles** pour votre monitoring interne car :
- Les pods fonctionnent normalement
- Les métriques internes sont bonnes
- Mais **l'utilisateur ne peut pas accéder au service**

### La Solution : Synthetic Monitoring

Le **synthetic monitoring** (monitoring synthétique) consiste à **simuler des utilisateurs** qui testent vos services de l'extérieur.

**Analogie** :
- **Monitoring interne** = Vous vérifiez que votre voiture démarre dans le garage
- **Synthetic monitoring** = Vous conduisez réellement la voiture pour vérifier qu'elle roule

**Principe** :
```
Robot/Sonde (externe) → Essaie d'accéder → Votre application
                          ↓
                    ✅ Succès ?
                    ❌ Échec ?
                    ⏱️ Combien de temps ?
```

Le robot simule un véritable utilisateur et mesure :
- **Disponibilité** : Le service répond-il ?
- **Latence** : Combien de temps pour obtenir une réponse ?
- **Validité** : La réponse est-elle correcte ?
- **Certificats** : Sont-ils valides et pas bientôt expirés ?

### Types de Monitoring

**Real User Monitoring (RUM)** :
- Mesure les expériences des **vrais utilisateurs**
- Via JavaScript dans le navigateur
- Données réelles mais volume élevé

**Synthetic Monitoring** :
- Simulations **automatisées** et **régulières**
- Proactif : Détecte les problèmes avant les utilisateurs
- Contrôlable : Vous choisissez quoi tester

**Complémentarité** :
```
Synthetic → Détecte "Le site est down"
RUM → Mesure "Les utilisateurs mettent 3s à charger la page"
```

## Qu'est-ce que Blackbox Exporter ?

**Blackbox Exporter** est un outil de l'écosystème Prometheus qui permet de faire du synthetic monitoring.

### Concept "Blackbox"

**Blackbox** signifie "boîte noire" : on teste **de l'extérieur** sans connaître l'intérieur.

```
┌─────────────────────────────────────┐
│      Votre Application              │
│         (Blackbox)                  │
│    On ne sait pas ce qu'il          │
│    y a dedans                       │
└─────────────────────────────────────┘
         ↑
         │ HTTP GET
         │ "Es-tu vivant ?"
         │
    ┌─────────────┐
    │  Blackbox   │
    │  Exporter   │
    └─────────────┘
```

Contrairement aux exporters "whitebox" (node_exporter, application metrics) qui connaissent l'interne, Blackbox teste comme un utilisateur externe.

### Protocoles Supportés

Blackbox Exporter peut tester plusieurs types de services :

**HTTP/HTTPS** :
- Vérifier qu'un site web répond
- Valider le certificat SSL/TLS
- Vérifier le contenu de la réponse
- Mesurer la latence

**TCP** :
- Tester une connexion TCP (ex: base de données, serveur mail)
- Vérifier qu'un port est ouvert

**ICMP (Ping)** :
- Tester la connectivité réseau
- Mesurer le temps de réponse

**DNS** :
- Vérifier la résolution DNS
- Valider les enregistrements

**GRPC** :
- Tester les services gRPC

### Architecture

```
┌────────────────────────────────────────────────┐
│            Prometheus                          │
│  ┌──────────────────────────────────────┐      │
│  │ Config: Scrape Blackbox Exporter     │      │
│  │ Targets:                             │      │
│  │  - https://myapp.example.com         │      │
│  │  - https://api.example.com           │      │
│  └──────────────────────────────────────┘      │
│                                                │
│  "Scrape /probe?target=myapp.example.com"      │
└────────────────┬───────────────────────────────┘
                 ↓
        ┌─────────────────┐
        │    Blackbox     │
        │    Exporter     │
        └────────┬────────┘
                 │
                 │ "Je vais tester pour toi"
                 ↓
        ┌─────────────────┐
        │  https://myapp  │ ← Votre application
        │  .example.com   │
        └─────────────────┘
```

**Flux** :
1. Prometheus dit à Blackbox : "Teste https://myapp.example.com"
2. Blackbox effectue la requête HTTP
3. Blackbox retourne les métriques à Prometheus :
   - `probe_success{} 1` (succès)
   - `probe_duration_seconds{} 0.234` (temps de réponse)
   - `probe_http_status_code{} 200`
   - `probe_ssl_earliest_cert_expiry{} 1735689600` (expiration SSL)

## Concepts Clés

### Probes (Sondes)

Une **probe** est un test que Blackbox effectue.

**Composants d'une probe** :
- **Target** : Ce qu'on teste (URL, IP, domaine)
- **Module** : Comment on teste (http, tcp, icmp, dns)
- **Configuration** : Paramètres du test

**Exemple** :
```
Probe:
  Target: https://myapp.example.com/health
  Module: http_2xx
  Config:
    - Méthode: GET
    - Timeout: 5s
    - Valid status codes: 200-299
```

### Modules

Un **module** définit **comment** effectuer le test.

Blackbox Exporter vient avec des modules pré-configurés :

**http_2xx** : Teste une URL HTTP et attend un status 2xx (200-299)
**http_post_2xx** : Teste avec méthode POST
**tcp_connect** : Teste la connexion TCP
**icmp** : Ping
**dns** : Résolution DNS

Vous pouvez aussi créer des modules personnalisés.

### Métriques Générées

Pour chaque probe, Blackbox génère des métriques :

**Métriques de base** :
```
probe_success{instance="https://myapp.example.com"} 1
# 1 = succès, 0 = échec

probe_duration_seconds{instance="https://myapp.example.com"} 0.234
# Temps pris pour la probe
```

**Métriques HTTP spécifiques** :
```
probe_http_status_code{} 200
probe_http_content_length{} 1024
probe_http_redirects{} 2
probe_http_ssl{} 1
probe_ssl_earliest_cert_expiry{} 1735689600
# Timestamp Unix de l'expiration du certificat
```

**Métriques DNS** :
```
probe_dns_lookup_time_seconds{} 0.012
probe_ip_protocol{} 4  # IPv4
```

## Déploiement dans MicroK8s

### 1. Déployer Blackbox Exporter

**ConfigMap pour la Configuration**

Fichier `blackbox-exporter-config.yaml` :

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: blackbox-exporter-config
  namespace: monitoring
data:
  blackbox.yml: |
    modules:
      # Module HTTP basique - vérifie status 2xx
      http_2xx:
        prober: http
        timeout: 5s
        http:
          valid_http_versions: ["HTTP/1.1", "HTTP/2.0"]
          valid_status_codes: []  # Par défaut 2xx
          method: GET
          fail_if_ssl: false
          fail_if_not_ssl: false
          tls_config:
            insecure_skip_verify: false

      # Module HTTP POST
      http_post_2xx:
        prober: http
        timeout: 5s
        http:
          method: POST
          headers:
            Content-Type: application/json
          body: '{"test": true}'

      # Module HTTP avec validation de contenu
      http_2xx_with_body:
        prober: http
        timeout: 5s
        http:
          valid_status_codes: [200]
          method: GET
          fail_if_body_not_matches_regexp:
            - ".*healthy.*"

      # Module ICMP (Ping)
      icmp:
        prober: icmp
        timeout: 5s
        icmp:
          preferred_ip_protocol: "ip4"

      # Module TCP
      tcp_connect:
        prober: tcp
        timeout: 5s
        tcp:
          preferred_ip_protocol: "ip4"

      # Module DNS
      dns_example:
        prober: dns
        timeout: 5s
        dns:
          query_name: "example.com"
          query_type: "A"
```

**Service Blackbox Exporter**

Fichier `blackbox-exporter-service.yaml` :

```yaml
apiVersion: v1
kind: Service
metadata:
  name: blackbox-exporter
  namespace: monitoring
  labels:
    app: blackbox-exporter
spec:
  selector:
    app: blackbox-exporter
  ports:
  - name: http
    port: 9115
    targetPort: 9115
  type: ClusterIP
```

**Deployment Blackbox Exporter**

Fichier `blackbox-exporter-deployment.yaml` :

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: blackbox-exporter
  namespace: monitoring
  labels:
    app: blackbox-exporter
spec:
  replicas: 1
  selector:
    matchLabels:
      app: blackbox-exporter
  template:
    metadata:
      labels:
        app: blackbox-exporter
    spec:
      containers:
      - name: blackbox-exporter
        image: prom/blackbox-exporter:v0.24.0
        args:
        - --config.file=/config/blackbox.yml
        ports:
        - name: http
          containerPort: 9115
        resources:
          requests:
            memory: "64Mi"
            cpu: "100m"
          limits:
            memory: "128Mi"
            cpu: "200m"
        volumeMounts:
        - name: config
          mountPath: /config
        livenessProbe:
          httpGet:
            path: /health
            port: 9115
          initialDelaySeconds: 10
        readinessProbe:
          httpGet:
            path: /health
            port: 9115
          initialDelaySeconds: 5
      volumes:
      - name: config
        configMap:
          name: blackbox-exporter-config
```

**Déployer** :

```bash
# Créer le namespace si nécessaire
microk8s kubectl create namespace monitoring

# Déployer Blackbox Exporter
microk8s kubectl apply -f blackbox-exporter-config.yaml
microk8s kubectl apply -f blackbox-exporter-service.yaml
microk8s kubectl apply -f blackbox-exporter-deployment.yaml

# Vérifier
microk8s kubectl get pods -n monitoring -l app=blackbox-exporter
microk8s kubectl logs -n monitoring -l app=blackbox-exporter
```

### 2. Configurer Prometheus pour Scraper Blackbox

Maintenant, configurez Prometheus pour utiliser Blackbox Exporter.

**ServiceMonitor** (si vous utilisez Prometheus Operator) :

Fichier `blackbox-servicemonitor.yaml` :

```yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: blackbox-exporter
  namespace: monitoring
spec:
  selector:
    matchLabels:
      app: blackbox-exporter
  endpoints:
  - port: http
    interval: 30s
```

**Ou configuration directe Prometheus** :

Ajoutez à votre ConfigMap Prometheus :

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: prometheus-config
  namespace: monitoring
data:
  prometheus.yml: |
    global:
      scrape_interval: 15s

    scrape_configs:

    # ... vos configs existantes ...

    # Blackbox Exporter - HTTP Probes
    - job_name: 'blackbox-http'
      metrics_path: /probe
      params:
        module: [http_2xx]  # Module à utiliser
      static_configs:
        - targets:
          # Liste des URLs à tester
          - https://myapp.example.com
          - https://myapp.example.com/health
          - https://api.example.com
      relabel_configs:
        # Le target devient un label
        - source_labels: [__address__]
          target_label: __param_target
        # Blackbox exporter est la vraie cible à scraper
        - source_labels: [__param_target]
          target_label: instance
        - target_label: __address__
          replacement: blackbox-exporter.monitoring.svc.cluster.local:9115

    # Blackbox Exporter - TCP Probes
    - job_name: 'blackbox-tcp'
      metrics_path: /probe
      params:
        module: [tcp_connect]
      static_configs:
        - targets:
          - database.example.com:5432
          - redis.example.com:6379
      relabel_configs:
        - source_labels: [__address__]
          target_label: __param_target
        - source_labels: [__param_target]
          target_label: instance
        - target_label: __address__
          replacement: blackbox-exporter.monitoring.svc.cluster.local:9115

    # Blackbox Exporter - ICMP Probes
    - job_name: 'blackbox-icmp'
      metrics_path: /probe
      params:
        module: [icmp]
      static_configs:
        - targets:
          - 8.8.8.8  # Google DNS
          - 1.1.1.1  # Cloudflare DNS
      relabel_configs:
        - source_labels: [__address__]
          target_label: __param_target
        - source_labels: [__param_target]
          target_label: instance
        - target_label: __address__
          replacement: blackbox-exporter.monitoring.svc.cluster.local:9115
```

**Points importants** :

Les `relabel_configs` sont essentiels :
1. `__address__` (votre target) devient `__param_target` (paramètre de l'URL)
2. L'instance label garde la trace du target original
3. `__address__` est remplacé par l'adresse de Blackbox Exporter

**Résultat** :
```
URL scrapée par Prometheus:
http://blackbox-exporter:9115/probe?target=https://myapp.example.com&module=http_2xx
```

Appliquez la nouvelle configuration :

```bash
microk8s kubectl apply -f prometheus-config.yaml

# Rechargez Prometheus (ou redémarrez)
microk8s kubectl rollout restart statefulset prometheus -n monitoring
```

### 3. Vérifier les Probes

**Interface Prometheus** :

1. Accédez à Prometheus : `http://localhost:9090`
2. **Status** → **Targets**
3. Vous devriez voir vos jobs `blackbox-http`, `blackbox-tcp`, `blackbox-icmp`
4. Status devrait être **UP** pour les targets accessibles

**Requêtes PromQL** :

```promql
# Voir toutes les probes
probe_success

# Probes qui échouent
probe_success == 0

# Durée des probes
probe_duration_seconds

# Status HTTP
probe_http_status_code
```

## Exemples de Probes

### 1. Vérifier la Disponibilité d'un Site Web

**Objectif** : Tester que votre site web répond.

**Configuration** :
```yaml
- job_name: 'website-availability'
  metrics_path: /probe
  params:
    module: [http_2xx]
  static_configs:
    - targets:
      - https://myapp.example.com
```

**Métriques générées** :
```
probe_success{instance="https://myapp.example.com"} 1
probe_duration_seconds{instance="https://myapp.example.com"} 0.234
probe_http_status_code{instance="https://myapp.example.com"} 200
```

**Alerte** :
```yaml
- alert: WebsiteDown
  expr: probe_success{job="website-availability"} == 0
  for: 2m
  labels:
    severity: critical
  annotations:
    summary: "Website {{ $labels.instance }} is down"
```

### 2. Surveiller l'Expiration des Certificats SSL

**Objectif** : Être alerté avant l'expiration d'un certificat.

**Configuration** : Même que ci-dessus (module http_2xx inclut SSL)

**Métriques** :
```
probe_ssl_earliest_cert_expiry{instance="https://myapp.example.com"} 1735689600
# Timestamp Unix de l'expiration
```

**Alerte** :
```yaml
- alert: SSLCertExpiringSoon
  expr: (probe_ssl_earliest_cert_expiry - time()) / 86400 < 30
  for: 1h
  labels:
    severity: warning
  annotations:
    summary: "SSL certificate for {{ $labels.instance }} expires in less than 30 days"
    description: "Certificate expires in {{ $value | humanizeDuration }}"
```

**Explication** :
- `probe_ssl_earliest_cert_expiry` : Timestamp d'expiration
- `time()` : Timestamp actuel
- Différence en secondes / 86400 = Jours restants
- Alerte si < 30 jours

### 3. Vérifier un Endpoint API

**Objectif** : Tester que votre API répond correctement.

**Module personnalisé** :
```yaml
http_api_health:
  prober: http
  timeout: 5s
  http:
    method: GET
    valid_status_codes: [200]
    headers:
      Authorization: "Bearer test-token"
    fail_if_body_not_matches_regexp:
      - '"status":"healthy"'
```

**Configuration Prometheus** :
```yaml
- job_name: 'api-health'
  metrics_path: /probe
  params:
    module: [http_api_health]
  static_configs:
    - targets:
      - https://api.example.com/health
```

**Ce qui est vérifié** :
- ✅ API répond
- ✅ Status 200
- ✅ Corps contient `"status":"healthy"`

### 4. Tester une Base de Données (TCP)

**Objectif** : Vérifier que PostgreSQL est joignable.

**Configuration** :
```yaml
- job_name: 'database-connectivity'
  metrics_path: /probe
  params:
    module: [tcp_connect]
  static_configs:
    - targets:
      - postgres.example.com:5432
      - redis.example.com:6379
```

**Métriques** :
```
probe_success{instance="postgres.example.com:5432"} 1
probe_duration_seconds{instance="postgres.example.com:5432"} 0.012
```

**Note** : Ceci teste seulement la **connectivité TCP**, pas l'authentification ou les requêtes SQL.

### 5. Surveiller la Résolution DNS

**Objectif** : Vérifier que votre domaine se résout correctement.

**Module personnalisé** :
```yaml
dns_query:
  prober: dns
  timeout: 5s
  dns:
    query_name: "myapp.example.com"
    query_type: "A"
    validate_answer_rrs:
      fail_if_not_matches_regexp:
        - "192\\.168\\..*"  # Doit résoudre vers 192.168.x.x
```

**Configuration** :
```yaml
- job_name: 'dns-resolution'
  metrics_path: /probe
  params:
    module: [dns_query]
  static_configs:
    - targets:
      - 8.8.8.8  # DNS server à utiliser
```

### 6. Test de Performance (Latence)

**Objectif** : Surveiller la latence de réponse.

**Configuration** : Standard http_2xx

**Alerte sur latence élevée** :
```yaml
- alert: HighLatency
  expr: probe_duration_seconds{job="website-availability"} > 2
  for: 5m
  labels:
    severity: warning
  annotations:
    summary: "High latency for {{ $labels.instance }}"
    description: "Response time is {{ $value }}s (threshold: 2s)"
```

### 7. Vérifier les Redirections

**Objectif** : S'assurer que les redirections HTTP fonctionnent.

**Module personnalisé** :
```yaml
http_redirect:
  prober: http
  timeout: 5s
  http:
    method: GET
    valid_status_codes: [301, 302]
    fail_if_ssl: false
    fail_if_not_ssl: false
```

**Ou** vérifier qu'un redirect aboutit au bon endroit :
```yaml
http_2xx_with_redirect:
  prober: http
  http:
    valid_status_codes: [200]
    method: GET
    fail_if_not_ssl: false
```

Teste `http://example.com` → Doit rediriger vers `https://example.com` et retourner 200.

## Patterns de Configuration Avancés

### 1. Tester Plusieurs Endpoints d'une Application

**Fichier de targets externe** :

Créez un fichier `blackbox-targets.yaml` :

```yaml
- targets:
  - https://myapp.example.com/
  - https://myapp.example.com/health
  - https://myapp.example.com/api/status
  - https://myapp.example.com/api/users
  labels:
    env: production
    service: myapp
```

**Configuration Prometheus** :
```yaml
- job_name: 'myapp-endpoints'
  metrics_path: /probe
  params:
    module: [http_2xx]
  file_sd_configs:
    - files:
      - /etc/prometheus/blackbox-targets.yaml
  relabel_configs:
    - source_labels: [__address__]
      target_label: __param_target
    - source_labels: [__param_target]
      target_label: instance
    - target_label: __address__
      replacement: blackbox-exporter.monitoring.svc.cluster.local:9115
```

**Avantage** : Ajouter/retirer des targets sans recharger Prometheus.

### 2. Tester depuis Plusieurs Localisations

Si vous avez un cluster multi-nœuds :

```yaml
- job_name: 'external-from-node1'
  metrics_path: /probe
  params:
    module: [http_2xx]
  static_configs:
    - targets:
      - https://external-api.example.com
      labels:
        probe_location: node1
  relabel_configs:
    # ... standard relabels ...
    - target_label: __address__
      replacement: blackbox-exporter-node1.monitoring.svc:9115

- job_name: 'external-from-node2'
  metrics_path: /probe
  params:
    module: [http_2xx]
  static_configs:
    - targets:
      - https://external-api.example.com
      labels:
        probe_location: node2
  relabel_configs:
    - target_label: __address__
      replacement: blackbox-exporter-node2.monitoring.svc:9115
```

**Utilité** : Comparer les latences depuis différents nœuds/régions.

### 3. Modules avec Authentification

**Basic Auth** :
```yaml
http_basic_auth:
  prober: http
  http:
    method: GET
    basic_auth:
      username: "monitoring"
      password: "secret"
```

**Bearer Token** :
```yaml
http_bearer_token:
  prober: http
  http:
    method: GET
    headers:
      Authorization: "Bearer your-token-here"
```

**Note de sécurité** : Ne mettez pas de secrets en dur dans la config. Utilisez Kubernetes Secrets.

### 4. Vérification de Contenu Complexe

**Vérifier du JSON** :
```yaml
http_json_api:
  prober: http
  http:
    method: GET
    valid_status_codes: [200]
    fail_if_body_not_matches_regexp:
      - '"status":\s*"ok"'
      - '"version":\s*"v\d+\.\d+\.\d+"'
```

**Vérifier plusieurs conditions** :
```yaml
http_complex_check:
  prober: http
  http:
    method: GET
    valid_status_codes: [200]
    fail_if_body_not_matches_regexp:
      - "healthy"
    fail_if_body_matches_regexp:
      - "error"
      - "fatal"
    fail_if_header_not_matches:
      - header: Content-Type
        regexp: "application/json"
```

## Visualisation dans Grafana

### Dashboard Blackbox Exporter

**Dashboard ID Grafana** : 7587 (dashboard communautaire)

**Import** :
1. Grafana → **Dashboards** → **Import**
2. ID : **7587**
3. Datasource : Votre Prometheus
4. **Import**

**Panneaux typiques** :

**1. Disponibilité (Success Rate)** :
```promql
avg(probe_success) * 100
```

**2. Latence moyenne** :
```promql
avg(probe_duration_seconds) * 1000  # en millisecondes
```

**3. Status des probes** (table) :
```promql
probe_success
```

**4. Expiration SSL** :
```promql
(probe_ssl_earliest_cert_expiry - time()) / 86400
```

**5. Timeline de disponibilité** :
```promql
probe_success{instance="https://myapp.example.com"}
```

### Créer Votre Dashboard Personnalisé

**Panel 1 : État de Tous les Endpoints**

Type : **Stat**

Requête :
```promql
sum(probe_success{job="website-availability"})
```

Affichage :
- Green si valeur = nombre de targets
- Red si valeur < nombre de targets

**Panel 2 : Uptime Percentage (30 jours)**

Type : **Gauge**

Requête :
```promql
avg_over_time(probe_success{instance="https://myapp.example.com"}[30d]) * 100
```

Seuils :
- Red : < 95%
- Yellow : 95-99%
- Green : > 99%

**Panel 3 : Latence par Endpoint**

Type : **Graph**

Requêtes :
```promql
probe_duration_seconds{job="website-availability"}
```

Légende : `{{ instance }}`

**Panel 4 : Certificats Expirant Bientôt**

Type : **Table**

Requête :
```promql
(probe_ssl_earliest_cert_expiry - time()) / 86400 < 60
```

Colonnes :
- Instance
- Jours restants
- Date d'expiration

### Variables de Dashboard

Créez des variables pour filtrer dynamiquement :

**Variable `job`** :
```promql
label_values(probe_success, job)
```

**Variable `instance`** :
```promql
label_values(probe_success{job="$job"}, instance)
```

Utilisez dans les requêtes :
```promql
probe_success{job="$job", instance="$instance"}
```

## Alerting avec Blackbox

### Alertes Essentielles

**1. Service Down**

```yaml
groups:
- name: blackbox
  interval: 30s
  rules:
  - alert: EndpointDown
    expr: probe_success == 0
    for: 2m
    labels:
      severity: critical
    annotations:
      summary: "Endpoint {{ $labels.instance }} is down"
      description: "{{ $labels.instance }} has been down for more than 2 minutes."
```

**2. Latence Élevée**

```yaml
- alert: HighLatency
  expr: probe_duration_seconds > 5
  for: 5m
  labels:
    severity: warning
  annotations:
    summary: "High latency on {{ $labels.instance }}"
    description: "Response time is {{ $value }}s (threshold: 5s)"
```

**3. Certificat SSL Expirant**

```yaml
- alert: SSLCertExpiring
  expr: (probe_ssl_earliest_cert_expiry - time()) / 86400 < 30
  labels:
    severity: warning
  annotations:
    summary: "SSL certificate expiring soon for {{ $labels.instance }}"
    description: "Certificate expires in {{ $value | humanizeDuration }}"

- alert: SSLCertExpired
  expr: probe_ssl_earliest_cert_expiry < time()
  labels:
    severity: critical
  annotations:
    summary: "SSL certificate EXPIRED for {{ $labels.instance }}"
```

**4. Status Code Anormal**

```yaml
- alert: UnexpectedHTTPStatus
  expr: probe_http_status_code < 200 or probe_http_status_code >= 400
  for: 2m
  labels:
    severity: warning
  annotations:
    summary: "Unexpected HTTP status for {{ $labels.instance }}"
    description: "HTTP status code is {{ $value }}"
```

**5. SLA Violation**

```yaml
- alert: SLAViolation
  expr: avg_over_time(probe_success[1h]) < 0.99
  for: 5m
  labels:
    severity: warning
  annotations:
    summary: "SLA violation for {{ $labels.instance }}"
    description: "Uptime over last hour: {{ $value | humanizePercentage }}"
```

### Notification Intelligente

**Grouper les alertes** :

```yaml
route:
  group_by: ['alertname', 'job']
  group_wait: 30s
  group_interval: 5m
  repeat_interval: 4h
  receiver: 'team-X'

  routes:
  # Alertes critiques → Pager
  - match:
      severity: critical
    receiver: pagerduty

  # Alertes warning → Slack
  - match:
      severity: warning
    receiver: slack
```

## Cas d'Usage Pratiques

### 1. Monitoring d'un Lab Personnel

**Ce que vous testez** :
```yaml
- https://grafana.mylab.local
- https://prometheus.mylab.local
- https://myapp.mylab.local
- https://api.mylab.local/health
```

**Bénéfices** :
- Savoir si vos services sont accessibles de l'extérieur
- Détecter les problèmes de certificats
- Mesurer les performances

### 2. Monitoring d'APIs Externes

**Scénario** : Votre app dépend d'APIs tierces (Stripe, SendGrid, etc.).

**Configuration** :
```yaml
- job_name: 'external-apis'
  metrics_path: /probe
  params:
    module: [http_2xx]
  static_configs:
    - targets:
      - https://api.stripe.com/healthcheck
      - https://api.sendgrid.com/v3/healthcheck
      labels:
        type: external_dependency
```

**Alerte** :
```yaml
- alert: ExternalAPIDown
  expr: probe_success{type="external_dependency"} == 0
  for: 5m
  annotations:
    summary: "External API {{ $labels.instance }} is unreachable"
```

**Utilité** : Savoir si le problème vient de vous ou du fournisseur externe.

### 3. Monitoring de DNS

**Scénario** : Vous avez des doutes sur votre DNS.

**Configuration** :
```yaml
dns_google:
  prober: dns
  dns:
    query_name: "myapp.example.com"
    query_type: "A"

- job_name: 'dns-check'
  metrics_path: /probe
  params:
    module: [dns_google]
  static_configs:
    - targets:
      - 8.8.8.8   # Google DNS
      - 1.1.1.1   # Cloudflare DNS
      labels:
        record: myapp.example.com
```

**Métriques** :
```promql
probe_dns_lookup_time_seconds
probe_success{job="dns-check"}
```

### 4. Monitoring de Connectivité Réseau

**Scénario** : Vérifier la connectivité réseau de base.

**Configuration** :
```yaml
- job_name: 'network-connectivity'
  metrics_path: /probe
  params:
    module: [icmp]
  static_configs:
    - targets:
      - 8.8.8.8           # Google DNS
      - 1.1.1.1           # Cloudflare DNS
      - your-router-ip    # Votre routeur
      - external-host-ip  # Serveur externe
```

**Métriques** :
```promql
probe_icmp_duration_seconds
probe_success{job="network-connectivity"}
```

**Utilité** : Diagnostiquer les problèmes réseau.

### 5. Monitoring Multi-Environnement

**Scénario** : Vous avez dev, staging, production.

**Configuration** :
```yaml
- job_name: 'app-dev'
  params:
    module: [http_2xx]
  static_configs:
    - targets:
      - https://dev.myapp.com
      labels:
        env: dev

- job_name: 'app-staging'
  params:
    module: [http_2xx]
  static_configs:
    - targets:
      - https://staging.myapp.com
      labels:
        env: staging

- job_name: 'app-prod'
  params:
    module: [http_2xx]
  static_configs:
    - targets:
      - https://myapp.com
      labels:
        env: production
```

**Dashboard avec variable `env`** :
```promql
probe_success{env="$env"}
```

## Bonnes Pratiques

### 1. Fréquence de Scraping

**Recommandations** :

```yaml
# Services critiques
scrape_interval: 15s

# Services standards
scrape_interval: 30s

# Services non-critiques
scrape_interval: 60s
```

**Compromis** :
- Plus fréquent = Détection rapide mais plus de charge
- Moins fréquent = Moins de charge mais détection plus lente

### 2. Timeouts Appropriés

```yaml
# Pour sites web
timeout: 5s

# Pour APIs rapides
timeout: 2s

# Pour services lents (ex: génération de rapport)
timeout: 30s
```

**Règle** : Timeout = 2-3x la latence normale attendue

### 3. Méthode HEAD vs GET

**GET** : Télécharge tout le contenu
**HEAD** : Télécharge seulement les headers

```yaml
http_head:
  prober: http
  http:
    method: HEAD
    valid_status_codes: [200]
```

**Utilisez HEAD** quand vous voulez juste vérifier la disponibilité sans charger le contenu.

### 4. Ne Pas Surcharger les Cibles

**Évitez** :
```yaml
scrape_interval: 1s  # ❌ Trop fréquent !
```

C'est du **monitoring**, pas un **stress test**.

### 5. Tester ce Que les Utilisateurs Voient

**Testez le hostname public** :
```
✅ https://myapp.com
❌ http://myapp-service.default.svc.cluster.local
```

Le test interne ne détecte pas :
- Problèmes DNS externes
- Problèmes d'Ingress
- Certificats SSL invalides pour le domaine public

### 6. Labels Cohérents

Utilisez les mêmes labels dans :
- Blackbox targets
- Application metrics
- Logs

```yaml
targets:
  - https://api.example.com
    labels:
      app: api
      env: production
      team: backend
```

Facilite la corrélation !

### 7. Documentation des Probes

Dans votre ConfigMap, documentez chaque module :

```yaml
# Module: http_api_authenticated
# Purpose: Test authenticated API endpoints
# Expected: HTTP 200 with valid JSON response
# Contact: team-api@example.com
http_api_authenticated:
  prober: http
  http:
    method: GET
    # ... config ...
```

## Limitations et Alternatives

### Limitations de Blackbox Exporter

**Ce qu'il NE fait PAS** :
- ❌ Tests complexes multi-étapes (login → navigate → submit)
- ❌ Exécution de JavaScript
- ❌ Tests de rendu visuel
- ❌ Tests de performance côté client

**Ce qu'il FAIT bien** :
- ✅ Tests simples HTTP/TCP/ICMP/DNS
- ✅ Vérification de disponibilité
- ✅ Mesure de latence réseau
- ✅ Validation de certificats

### Pour des Tests Plus Complexes

**Selenium/Puppeteer** :
- Tests de navigateur complets
- Scénarios utilisateurs complexes
- Capture d'écrans

**K6/Locust** :
- Tests de charge
- Scénarios de performance

**Synthetic Monitoring SaaS** :
- Pingdom, UptimeRobot, Datadog Synthetics
- Tests depuis plusieurs régions géographiques
- Plus de fonctionnalités mais coût

**Recommandation** : Blackbox pour monitoring basique, outils spécialisés pour tests complexes.

## Dépannage

### Problème : Probe Toujours en Échec

**Vérifications** :

1. **Test manuel depuis le pod Blackbox** :
```bash
kubectl exec -it -n monitoring blackbox-exporter-xxx -- /bin/sh
wget -O- https://myapp.example.com
```

2. **Vérifier les logs Blackbox** :
```bash
kubectl logs -n monitoring -l app=blackbox-exporter
```

3. **Tester l'endpoint /probe directement** :
```bash
kubectl port-forward -n monitoring svc/blackbox-exporter 9115:9115
curl "http://localhost:9115/probe?target=https://myapp.example.com&module=http_2xx"
```

4. **Causes courantes** :
   - URL invalide ou inaccessible
   - Certificat SSL invalide
   - Timeout trop court
   - Blackbox ne peut pas résoudre le DNS
   - Firewall bloque Blackbox

### Problème : Certificat SSL Invalide

Si vous avez des certificats auto-signés (lab) :

```yaml
http_2xx_insecure:
  prober: http
  http:
    tls_config:
      insecure_skip_verify: true  # ⚠️ Seulement pour lab !
```

**Attention** : Ne pas utiliser en production !

### Problème : DNS Ne Résout Pas

Blackbox utilise le DNS du cluster. Si problème :

1. Vérifier CoreDNS :
```bash
kubectl get pods -n kube-system -l k8s-app=kube-dns
```

2. Tester résolution :
```bash
kubectl run -it --rm debug --image=busybox --restart=Never -- nslookup myapp.example.com
```

## Ce Qu'il Faut Retenir

🎯 **Synthetic monitoring** :
- Teste vos services **du point de vue utilisateur**
- Proactif : Détecte les problèmes avant les utilisateurs
- Complémentaire au monitoring interne

🔍 **Blackbox Exporter** :
- Outil Prometheus pour synthetic monitoring
- Supporte HTTP, TCP, ICMP, DNS, GRPC
- Léger et facile à déployer

📊 **Métriques clés** :
- `probe_success` : Service accessible ?
- `probe_duration_seconds` : Temps de réponse
- `probe_http_status_code` : Code HTTP
- `probe_ssl_earliest_cert_expiry` : Expiration certificat

⚙️ **Configuration** :
- Modules définissent comment tester
- Prometheus scrape Blackbox avec relabel_configs
- Targets = Ce qu'on teste

🚨 **Alerting** :
- Service down
- Latence élevée
- Certificat expirant
- SLA violation

📈 **Visualisation** :
- Dashboards Grafana
- Uptime percentages
- Latence par endpoint
- État des certificats

✅ **Bonnes pratiques** :
- Fréquence adaptée (15-60s)
- Timeouts appropriés
- Tester ce que les utilisateurs voient
- Labels cohérents

🔗 **Complémentarité** :
- **Métriques internes** : Comment ça marche
- **Logs** : Ce qui s'est passé
- **Traces** : Parcours des requêtes
- **Synthetic monitoring** : Est-ce que ça marche du point de vue utilisateur

---

## Conclusion

Le synthetic monitoring avec Blackbox Exporter complète votre stack d'observabilité en ajoutant la **perspective utilisateur**. Vous ne surveillez plus seulement ce qui se passe à l'intérieur de vos applications, mais aussi si vos utilisateurs peuvent réellement y accéder.

**Stack d'observabilité complète** :

1. ✅ **Métriques** (Prometheus) → Santé interne
2. ✅ **Logs** (Loki) → Événements et erreurs
3. ✅ **Traces** (Jaeger) → Parcours des requêtes
4. ✅ **Synthetic Monitoring** (Blackbox) → Vue utilisateur

Avec ces quatre piliers, vous avez une **visibilité totale** sur votre système MicroK8s ! 🎉

**Prochaine section** :
- 15.7 : SLI/SLO et dashboards de fiabilité (définir et mesurer vos objectifs de service)

⏭️ [SLI/SLO et dashboards de fiabilité](/15-observabilite-avancee/07-sli-slo-et-dashboards-de-fiabilite.md)
