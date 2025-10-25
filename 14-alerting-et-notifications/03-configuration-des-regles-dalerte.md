🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 14.3 Configuration des Règles d'Alerte

## Introduction

Maintenant que vous comprenez les concepts d'alerting et le fonctionnement d'Alertmanager, il est temps d'apprendre à créer vos propres **règles d'alerte** dans Prometheus. C'est ici que vous définissez les conditions qui déclencheront des alertes dans votre infrastructure Kubernetes.

Les règles d'alerte sont le cerveau de votre système de monitoring : elles analysent constamment vos métriques et détectent les situations anormales. Bien configurées, elles vous permettent de détecter les problèmes avant qu'ils n'affectent vos utilisateurs.

## Anatomie d'une Règle d'Alerte

Une règle d'alerte dans Prometheus est définie en YAML et comprend plusieurs éléments essentiels.

### Structure de Base

```yaml
groups:
  - name: nom_du_groupe
    interval: 30s
    rules:
      - alert: NomDeLAlerte
        expr: expression_promql
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "Résumé court"
          description: "Description détaillée"
```

### Décortiquons Chaque Élément

**groups** : Les règles sont organisées en groupes. Un groupe peut contenir plusieurs règles liées.

**name** : Le nom du groupe (ex: `kubernetes_pods`, `node_alerts`)

**interval** : Fréquence d'évaluation des règles dans ce groupe (par défaut: 1m)

**alert** : Le nom unique de votre alerte (ex: `HighPodMemory`, `NodeDown`)

**expr** : L'expression PromQL qui définit la condition de déclenchement

**for** : Durée pendant laquelle la condition doit être vraie avant de déclencher l'alerte

**labels** : Étiquettes ajoutées à l'alerte (utilisées pour le routage dans Alertmanager)

**annotations** : Informations descriptives pour les humains (non utilisées pour le routage)

## Expressions PromQL pour les Alertes

Le cœur d'une règle d'alerte est son **expression PromQL**. Cette expression doit retourner une valeur qui, si elle est présente/non-nulle, déclenche l'alerte.

### Principe de Base

Une expression qui retourne **quelque chose** = problème détecté
Une expression qui ne retourne **rien** = tout va bien

### Exemple Simple

```yaml
expr: up{job="my-service"} == 0
```

Cette expression signifie : "Le service n'est pas accessible (up == 0)"

### Opérateurs de Comparaison

- `==` : Égal à
- `!=` : Différent de
- `>` : Supérieur à
- `<` : Inférieur à
- `>=` : Supérieur ou égal à
- `<=` : Inférieur ou égal à

### Exemple avec Seuil

```yaml
expr: node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes * 100 < 10
```

Cette expression calcule le pourcentage de mémoire disponible et alerte si c'est moins de 10%.

## Exemples de Règles d'Alerte Essentielles

Voici des règles d'alerte courantes et utiles pour un cluster Kubernetes.

### 1. Service Indisponible

```yaml
groups:
  - name: service_availability
    interval: 30s
    rules:
      - alert: ServiceDown
        expr: up{job="mon-service"} == 0
        for: 2m
        labels:
          severity: critical
          team: backend
        annotations:
          summary: "Le service {{ $labels.job }} est indisponible"
          description: "Le service {{ $labels.job }} sur l'instance {{ $labels.instance }} est down depuis plus de 2 minutes."
          runbook_url: "https://wiki.example.com/runbooks/service-down"
```

**Explication** :
- Déclenche quand un service ne répond plus (métrique `up` = 0)
- Attend 2 minutes avant d'alerter (évite les faux positifs)
- Sévérité critique car impact direct sur les utilisateurs
- Les `{{ $labels.xxx }}` sont remplacés par les vraies valeurs

### 2. Utilisation Mémoire Élevée des Pods

```yaml
- alert: HighPodMemoryUsage
  expr: |
    (
      container_memory_working_set_bytes{container!=""}
      /
      container_spec_memory_limit_bytes{container!=""} * 100
    ) > 80
  for: 10m
  labels:
    severity: warning
    component: pod
  annotations:
    summary: "Pod {{ $labels.pod }} utilise {{ $value | humanize }}% de sa mémoire"
    description: |
      Le pod {{ $labels.pod }} dans le namespace {{ $labels.namespace }}
      utilise {{ $value | humanize }}% de sa limite mémoire depuis 10 minutes.
      Cela peut causer un OOMKill (Out Of Memory).
```

**Explication** :
- Calcule le pourcentage d'utilisation mémoire
- Alerte si > 80% pendant 10 minutes
- Utilise `humanize` pour formater les valeurs
- Description multi-lignes avec contexte

### 3. Pod en Crash Loop

```yaml
- alert: PodCrashLooping
  expr: rate(kube_pod_container_status_restarts_total[15m]) > 0
  for: 5m
  labels:
    severity: warning
  annotations:
    summary: "Pod {{ $labels.pod }} redémarre fréquemment"
    description: |
      Le pod {{ $labels.pod }} dans {{ $labels.namespace }}
      a redémarré {{ $value | humanize }} fois/seconde sur les 15 dernières minutes.
      Cela indique un problème applicatif.
```

**Explication** :
- Utilise `rate()` pour calculer le taux de redémarrage
- Détecte les pods qui redémarrent en boucle
- `rate()` sur 15m pour lisser les variations

### 4. Utilisation CPU Élevée du Nœud

```yaml
- alert: HighNodeCPU
  expr: |
    (
      100 - (avg by(instance) (rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100)
    ) > 85
  for: 15m
  labels:
    severity: warning
    component: node
  annotations:
    summary: "Utilisation CPU élevée sur {{ $labels.instance }}"
    description: |
      Le nœud {{ $labels.instance }} a une utilisation CPU de {{ $value | humanize }}%
      depuis 15 minutes.
```

**Explication** :
- Calcule l'utilisation CPU (100 - idle = utilisé)
- Alerte si > 85% pendant 15 minutes
- `avg by(instance)` agrège tous les CPU du nœud

### 5. Espace Disque Faible

```yaml
- alert: LowDiskSpace
  expr: |
    (
      node_filesystem_avail_bytes{fstype!~"tmpfs|fuse.*"}
      /
      node_filesystem_size_bytes{fstype!~"tmpfs|fuse.*"} * 100
    ) < 15
  for: 10m
  labels:
    severity: warning
  annotations:
    summary: "Espace disque faible sur {{ $labels.instance }}"
    description: |
      Le système de fichiers {{ $labels.mountpoint }} sur {{ $labels.instance }}
      n'a plus que {{ $value | humanize }}% d'espace disponible.
```

**Explication** :
- Calcule le pourcentage d'espace disque disponible
- Exclut tmpfs et fuse (systèmes de fichiers temporaires)
- Alerte si < 15% d'espace disponible

### 6. Espace Disque Critique

```yaml
- alert: CriticalDiskSpace
  expr: |
    (
      node_filesystem_avail_bytes{fstype!~"tmpfs|fuse.*"}
      /
      node_filesystem_size_bytes{fstype!~"tmpfs|fuse.*"} * 100
    ) < 5
  for: 5m
  labels:
    severity: critical
  annotations:
    summary: "Espace disque CRITIQUE sur {{ $labels.instance }}"
    description: |
      URGENT: Le système de fichiers {{ $labels.mountpoint }} sur {{ $labels.instance }}
      n'a plus que {{ $value | humanize }}% d'espace disponible.
      Action immédiate requise pour éviter une panne.
    runbook_url: "https://wiki.example.com/runbooks/disk-space-critical"
```

**Explication** :
- Même métrique que LowDiskSpace mais seuil plus bas
- Sévérité critique car risque de panne imminent
- Durée plus courte (5m vs 10m) pour réagir vite

### 7. Taux d'Erreur HTTP Élevé

```yaml
- alert: HighHTTPErrorRate
  expr: |
    (
      sum by(job, namespace) (rate(http_requests_total{status=~"5.."}[5m]))
      /
      sum by(job, namespace) (rate(http_requests_total[5m]))
    ) * 100 > 5
  for: 5m
  labels:
    severity: critical
  annotations:
    summary: "Taux d'erreur HTTP élevé pour {{ $labels.job }}"
    description: |
      Le service {{ $labels.job }} dans {{ $labels.namespace }}
      a un taux d'erreur 5xx de {{ $value | humanize }}% sur les 5 dernières minutes.
```

**Explication** :
- Calcule le ratio erreurs 5xx / total des requêtes
- Status `5..` = toutes les erreurs 5xx (500, 502, 503, etc.)
- Alerte si > 5% d'erreurs

### 8. Latence API Élevée

```yaml
- alert: HighAPILatency
  expr: |
    histogram_quantile(0.95,
      sum by(job, le) (rate(http_request_duration_seconds_bucket[5m]))
    ) > 1
  for: 10m
  labels:
    severity: warning
  annotations:
    summary: "Latence API élevée pour {{ $labels.job }}"
    description: |
      Le 95ème percentile de latence pour {{ $labels.job }}
      est de {{ $value | humanize }}s sur les 5 dernières minutes.
      Les utilisateurs peuvent ressentir de la lenteur.
```

**Explication** :
- Utilise `histogram_quantile` pour calculer le p95
- Le p95 représente la latence pour 95% des requêtes
- Alerte si p95 > 1 seconde

### 9. Certificat SSL Proche de l'Expiration

```yaml
- alert: CertificateExpiringSoon
  expr: |
    (probe_ssl_earliest_cert_expiry - time()) / 86400 < 7
  for: 1h
  labels:
    severity: warning
  annotations:
    summary: "Certificat SSL expire bientôt pour {{ $labels.instance }}"
    description: |
      Le certificat SSL de {{ $labels.instance }} expire dans
      {{ $value | humanize }} jours.
      Renouvellement nécessaire.
```

**Explication** :
- Calcule le nombre de jours avant expiration
- 86400 = nombre de secondes dans un jour
- Alerte 7 jours avant l'expiration

### 10. PersistentVolume Presque Plein

```yaml
- alert: PersistentVolumeAlmostFull
  expr: |
    (
      kubelet_volume_stats_available_bytes
      /
      kubelet_volume_stats_capacity_bytes * 100
    ) < 10
  for: 5m
  labels:
    severity: warning
  annotations:
    summary: "PersistentVolume {{ $labels.persistentvolumeclaim }} presque plein"
    description: |
      Le PVC {{ $labels.persistentvolumeclaim }} dans {{ $labels.namespace }}
      n'a plus que {{ $value | humanize }}% d'espace disponible.
```

**Explication** :
- Surveille l'espace disponible dans les PersistentVolumes
- Crucial pour les bases de données et applications stateful
- Alerte précoce pour éviter les problèmes

## Utilisation du Paramètre "for"

Le paramètre `for` est crucial pour éviter les **fausses alertes** (false positives).

### Sans "for" - Alerte Immédiate

```yaml
- alert: ExempleImmédiat
  expr: métrique > seuil
  # Pas de "for"
```
Déclenche **immédiatement** dès que la condition est vraie.

**Quand l'utiliser** :
- Alertes critiques nécessitant une réaction immédiate
- Situations où un pic momentané est déjà problématique

### Avec "for" - Alerte Retardée

```yaml
- alert: ExempleRetardé
  expr: métrique > seuil
  for: 10m
```
Déclenche **seulement** si la condition reste vraie pendant 10 minutes.

**Quand l'utiliser** :
- La plupart des alertes (80-90% des cas)
- Évite les alertes sur des pics temporaires normaux
- Détecte les problèmes persistants

### Choisir la Bonne Durée

**30s - 2m** : Problèmes critiques nécessitant une action immédiate
- Service complètement down
- Perte de données imminente

**5m - 10m** : Problèmes sérieux mais pas immédiatement critiques
- Utilisation ressources élevée
- Latence augmentée
- Taux d'erreur élevé

**15m - 30m** : Tendances préoccupantes
- Utilisation mémoire croissante
- Espace disque qui diminue
- Performance qui se dégrade

## Labels et Annotations : Bonnes Pratiques

### Labels : Pour le Routage

Les labels sont utilisés par Alertmanager pour router les alertes. Choisissez-les avec soin.

**Labels recommandés** :

```yaml
labels:
  severity: critical|warning|info
  team: backend|frontend|infra|data
  component: pod|node|network|storage
  environment: production|staging|dev
  service: nom-du-service
```

**Bonnes pratiques** :
- Utilisez des valeurs standardisées (pas de texte libre)
- Minimum : toujours avoir `severity`
- Maximum : 5-7 labels par alerte
- Cohérence : mêmes labels à travers toutes les alertes

### Annotations : Pour les Humains

Les annotations contiennent des informations descriptives. Elles peuvent contenir du texte libre et des templates.

**Annotations recommandées** :

```yaml
annotations:
  summary: "Résumé court (< 100 caractères)"
  description: "Description détaillée avec contexte et valeurs"
  impact: "Impact sur les utilisateurs ou le business"
  action: "Actions recommandées pour résoudre"
  runbook_url: "https://wiki.example.com/runbooks/alerte"
  dashboard_url: "https://grafana.example.com/d/xxx"
```

### Templating dans les Annotations

Vous pouvez utiliser des templates Go pour insérer des valeurs dynamiques :

**Variables disponibles** :
- `{{ $labels.nom_label }}` : Valeur d'un label
- `{{ $value }}` : Valeur numérique de l'alerte
- `{{ $value | humanize }}` : Valeur formatée lisiblement
- `{{ $value | humanizePercentage }}` : Formatage en pourcentage
- `{{ $value | humanizeDuration }}` : Formatage en durée

**Exemple complet** :

```yaml
annotations:
  summary: "CPU élevé sur {{ $labels.instance }}"
  description: |
    Le nœud {{ $labels.instance }} ({{ $labels.node }})
    a une utilisation CPU de {{ $value | humanizePercentage }}
    depuis {{ $activeAt | humanizeDuration }}.

    Valeur actuelle: {{ $value | humanize }}%
    Seuil configuré: 85%

    Dashboard: https://grafana.example.com/d/nodes?var-node={{ $labels.node }}
  action: |
    1. Vérifier les pods consommant le plus de CPU
    2. Vérifier s'il y a un pic de trafic inhabituel
    3. Envisager de scaler horizontalement si nécessaire
```

## Organisation des Fichiers de Règles

### Structure Recommandée

Organisez vos règles en plusieurs fichiers thématiques :

```
/etc/prometheus/rules/
├── node-alerts.yml          # Alertes liées aux nœuds
├── pod-alerts.yml           # Alertes liées aux pods
├── service-alerts.yml       # Alertes liées aux services
├── storage-alerts.yml       # Alertes liées au stockage
├── network-alerts.yml       # Alertes réseau
└── business-alerts.yml      # Alertes métier/applicatives
```

### Exemple : node-alerts.yml

```yaml
groups:
  - name: node_alerts
    interval: 30s
    rules:
      - alert: NodeDown
        expr: up{job="node-exporter"} == 0
        for: 2m
        labels:
          severity: critical
          component: node
        annotations:
          summary: "Nœud {{ $labels.instance }} inaccessible"
          description: "Le nœud {{ $labels.instance }} ne répond plus depuis 2 minutes."

      - alert: HighNodeCPU
        expr: (100 - (avg by(instance) (rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100)) > 85
        for: 15m
        labels:
          severity: warning
          component: node
        annotations:
          summary: "CPU élevé sur {{ $labels.instance }}"
          description: "Utilisation CPU de {{ $value | humanize }}% depuis 15 minutes."

      - alert: HighNodeMemory
        expr: (node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes * 100) < 10
        for: 5m
        labels:
          severity: warning
          component: node
        annotations:
          summary: "Mémoire faible sur {{ $labels.instance }}"
          description: "Seulement {{ $value | humanize }}% de mémoire disponible."
```

### Exemple : pod-alerts.yml

```yaml
groups:
  - name: pod_alerts
    interval: 1m
    rules:
      - alert: PodNotReady
        expr: kube_pod_status_phase{phase!~"Running|Succeeded"} > 0
        for: 5m
        labels:
          severity: warning
          component: pod
        annotations:
          summary: "Pod {{ $labels.pod }} n'est pas Ready"
          description: |
            Le pod {{ $labels.pod }} dans {{ $labels.namespace }}
            est en phase {{ $labels.phase }} depuis 5 minutes.

      - alert: PodCrashLooping
        expr: rate(kube_pod_container_status_restarts_total[15m]) > 0
        for: 5m
        labels:
          severity: warning
          component: pod
        annotations:
          summary: "Pod {{ $labels.pod }} crash loop"
          description: "Le pod redémarre fréquemment."

      - alert: HighPodMemory
        expr: |
          (
            container_memory_working_set_bytes{container!=""}
            /
            container_spec_memory_limit_bytes{container!=""} * 100
          ) > 90
        for: 10m
        labels:
          severity: critical
          component: pod
        annotations:
          summary: "Mémoire critique pour {{ $labels.pod }}"
          description: "Utilisation de {{ $value | humanize }}% - risque d'OOMKill."
```

## Déploiement des Règles sur MicroK8s

### Méthode 1 : Via ConfigMap

C'est la méthode recommandée sur Kubernetes/MicroK8s.

**Créer le ConfigMap** :

```bash
# Créer un fichier avec vos règles
cat > my-alerts.yml <<EOF
groups:
  - name: my_custom_alerts
    interval: 30s
    rules:
      - alert: ExampleAlert
        expr: up == 0
        for: 1m
        labels:
          severity: critical
        annotations:
          summary: "Service down"
EOF

# Créer le ConfigMap
microk8s kubectl create configmap prometheus-additional-rules \
  --from-file=my-alerts.yml \
  -n monitoring
```

**Configurer Prometheus pour charger ces règles** :

Vous devez modifier la configuration de Prometheus pour qu'il charge ce ConfigMap. Cela dépend de votre installation, mais généralement :

```bash
# Éditer le StatefulSet ou Deployment de Prometheus
microk8s kubectl edit statefulset prometheus-k8s -n monitoring
```

Ajoutez un volume et un volumeMount pour votre ConfigMap.

### Méthode 2 : Via PrometheusRule (Operator)

Si vous utilisez le Prometheus Operator, vous pouvez créer un objet `PrometheusRule` :

```yaml
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: my-custom-alerts
  namespace: monitoring
  labels:
    prometheus: kube-prometheus
spec:
  groups:
    - name: my_custom_alerts
      interval: 30s
      rules:
        - alert: ExampleAlert
          expr: up{job="my-service"} == 0
          for: 2m
          labels:
            severity: critical
          annotations:
            summary: "Service my-service is down"
            description: "Check the service immediately"
```

Appliquer la règle :

```bash
microk8s kubectl apply -f my-prometheus-rule.yml
```

Le Prometheus Operator détectera automatiquement cette nouvelle règle et la chargera.

### Vérifier que les Règles sont Chargées

**Via l'interface Prometheus** :

1. Exposer Prometheus :
```bash
microk8s kubectl port-forward -n monitoring svc/prometheus-k8s 9090:9090
```

2. Ouvrir `http://localhost:9090`

3. Aller dans **Status** → **Rules**

Vous devriez voir toutes vos règles listées.

**Via kubectl** :

```bash
# Voir les PrometheusRules
microk8s kubectl get prometheusrules -n monitoring

# Détail d'une règle
microk8s kubectl describe prometheusrule my-custom-alerts -n monitoring
```

## Validation et Test des Règles

### Validation de la Syntaxe

Avant de déployer, validez vos fichiers de règles :

**Outil promtool** :

```bash
# Installer promtool (si pas déjà présent)
# Sur le nœud où Prometheus tourne, ou téléchargez-le

# Valider le fichier
promtool check rules my-alerts.yml
```

Si tout va bien, vous verrez :
```
Checking my-alerts.yml
  SUCCESS: 3 rules found
```

### Test des Expressions PromQL

Avant de créer une alerte, testez votre expression PromQL dans l'interface Prometheus :

1. Ouvrir Prometheus UI : `http://localhost:9090`
2. Aller dans l'onglet **Graph**
3. Entrer votre expression PromQL
4. Cliquer sur **Execute**

Vérifiez que :
- L'expression retourne des résultats quand le problème existe
- L'expression ne retourne rien quand tout va bien
- Les labels sont corrects

### Simuler une Alerte

Pour tester qu'une alerte se déclenche correctement :

**1. Provoquer volontairement la condition** :

Par exemple, pour tester une alerte de CPU :
```bash
# Déployer un pod qui consomme beaucoup de CPU
kubectl run cpu-stress --image=progrium/stress \
  -- --cpu 2 --timeout 60s
```

**2. Vérifier dans Prometheus UI** :

- Aller dans **Alerts**
- Chercher votre alerte
- Vérifier qu'elle passe à l'état **Pending** puis **Firing**

**3. Vérifier dans Alertmanager** :

- Ouvrir Alertmanager UI : `http://localhost:9093`
- Vérifier que l'alerte apparaît
- Vérifier qu'une notification est envoyée

## Techniques PromQL Avancées pour les Alertes

### Agrégations

**sum** : Additionner des valeurs
```yaml
expr: sum by(namespace) (kube_pod_container_resource_requests_memory_bytes)
```

**avg** : Moyenne
```yaml
expr: avg by(instance) (node_load1)
```

**max** : Maximum
```yaml
expr: max by(pod) (container_cpu_usage_seconds_total)
```

**count** : Compter
```yaml
expr: count(up{job="my-service"} == 0) > 3
# Alerte si plus de 3 instances sont down
```

### Fonctions Temporelles

**rate()** : Taux par seconde
```yaml
expr: rate(http_requests_total[5m]) > 1000
# Plus de 1000 requêtes/seconde sur 5 minutes
```

**irate()** : Taux instantané (plus sensible)
```yaml
expr: irate(node_network_receive_bytes_total[5m]) > 100000000
# Trafic réseau > 100MB/s
```

**increase()** : Augmentation totale
```yaml
expr: increase(http_requests_total[1h]) > 10000
# Plus de 10000 requêtes dans la dernière heure
```

### Opérations Mathématiques

**Pourcentage** :
```yaml
expr: (métrique_utilisée / métrique_totale) * 100 > 80
```

**Prédiction** :
```yaml
expr: predict_linear(node_filesystem_free_bytes[1h], 4*3600) < 0
# Prédit que le disque sera plein dans 4 heures
```

### Comparaisons Temporelles

**Comparer avec le passé** :
```yaml
expr: |
  rate(http_requests_total[5m])
  /
  rate(http_requests_total[5m] offset 1d)
  > 2
# Trafic 2x supérieur à la même heure hier
```

### Conditions Multiples (AND, OR)

**AND** (les deux conditions doivent être vraies) :
```yaml
expr: |
  (
    rate(http_requests_total{status="500"}[5m]) > 10
  )
  and
  (
    rate(http_requests_total[5m]) > 100
  )
# Erreurs 500 ET trafic élevé
```

**OR** (au moins une condition vraie) :
```yaml
expr: |
  (node_filesystem_avail_bytes / node_filesystem_size_bytes * 100 < 10)
  or
  (predict_linear(node_filesystem_avail_bytes[1h], 4*3600) < 0)
# Disque faible OU sera bientôt plein
```

**UNLESS** (sauf si) :
```yaml
expr: |
  up{job="my-service"} == 0
  unless
  on(instance) kube_pod_status_phase{phase="Pending"}
# Service down sauf s'il est en démarrage
```

## Bonnes Pratiques de Configuration

### 1. Nommage des Alertes

**Bonne pratique** :
- Utilisez le CamelCase : `HighPodMemory`, `NodeDown`
- Commencez par la condition : `High...`, `Low...`, `Critical...`
- Soyez descriptif mais concis
- Évitez les abréviations obscures

**Exemples** :
- ✓ `HighPodCPUUsage`
- ✓ `CertificateExpiringSoon`
- ✓ `DatabaseConnectionPoolExhausted`
- ✗ `alert1`
- ✗ `prob_cpu`

### 2. Définir les Bonnes Durées "for"

**Règle générale** :
- Critique + immédiat : `for: 1m` ou pas de `for`
- Critique + tolérable : `for: 5m`
- Warning : `for: 10m` à `15m`
- Info : `for: 30m` à `1h`

### 3. Calibrer les Seuils

**Méthode** :
1. Observez vos métriques pendant 1-2 semaines
2. Identifiez les valeurs normales
3. Définissez des seuils avec une marge
4. Ajustez après les premières alertes

**Exemple** :
- CPU normal : 20-40%
- Pic occasionnel : 60-70%
- Seuil warning : 80%
- Seuil critical : 95%

### 4. Alertes Multicouches

Créez plusieurs niveaux d'alerte pour le même problème :

```yaml
# Niveau 1 : Avertissement précoce
- alert: HighDiskUsage
  expr: disk_used_percent > 75
  for: 30m
  labels:
    severity: warning

# Niveau 2 : Critique
- alert: CriticalDiskUsage
  expr: disk_used_percent > 90
  for: 5m
  labels:
    severity: critical
```

### 5. Documentation Obligatoire

Chaque alerte devrait avoir au minimum :
- `summary` : Ce qui se passe
- `description` : Détails et contexte
- `runbook_url` : Lien vers la procédure de résolution

### 6. Éviter les Alertes Bruyantes

**Checklist** :
- ☐ L'alerte indique-t-elle un problème réel ?
- ☐ L'alerte nécessite-t-elle une action humaine ?
- ☐ L'action à prendre est-elle documentée ?
- ☐ Le seuil est-il approprié ?
- ☐ La durée `for` évite-t-elle les faux positifs ?

Si vous répondez "non" à l'une de ces questions, revoyez votre alerte.

### 7. Tester Avant de Déployer en Production

**Process recommandé** :
1. Tester en environnement de dev/staging
2. Observer pendant quelques jours
3. Ajuster les seuils si nécessaire
4. Déployer en production
5. Continuer à affiner

## Exemples de Règles pour Cas Spécifiques

### Application E-commerce

```yaml
groups:
  - name: ecommerce_alerts
    rules:
      - alert: CheckoutAPIHighLatency
        expr: histogram_quantile(0.95, rate(http_request_duration_seconds_bucket{endpoint="/checkout"}[5m])) > 2
        for: 5m
        labels:
          severity: critical
          team: checkout
        annotations:
          summary: "Latence checkout élevée"
          description: "Le checkout met {{ $value }}s (p95)"

      - alert: PaymentFailureRateHigh
        expr: |
          (
            sum(rate(payment_attempts_total{status="failed"}[5m]))
            /
            sum(rate(payment_attempts_total[5m]))
          ) * 100 > 5
        for: 10m
        labels:
          severity: critical
          team: payments
        annotations:
          summary: "Taux d'échec de paiement élevé"
          description: "{{ $value }}% des paiements échouent"
```

### Base de Données

```yaml
groups:
  - name: database_alerts
    rules:
      - alert: DatabaseConnectionPoolNearLimit
        expr: (mysql_global_status_threads_connected / mysql_global_variables_max_connections) * 100 > 80
        for: 5m
        labels:
          severity: warning
          team: data
        annotations:
          summary: "Pool de connexions DB saturé"
          description: "{{ $value }}% des connexions utilisées"

      - alert: DatabaseSlowQueries
        expr: rate(mysql_global_status_slow_queries[5m]) > 10
        for: 10m
        labels:
          severity: warning
          team: data
        annotations:
          summary: "Requêtes SQL lentes détectées"
          description: "{{ $value }} requêtes lentes/seconde"
```

### File d'Attente (Queue)

```yaml
groups:
  - name: queue_alerts
    rules:
      - alert: QueueBacklogGrowing
        expr: rate(queue_depth[10m]) > 0
        for: 30m
        labels:
          severity: warning
        annotations:
          summary: "File d'attente {{ $labels.queue }} en croissance"
          description: "La profondeur de la queue augmente de {{ $value }} msgs/s"

      - alert: QueueProcessingStalled
        expr: rate(queue_messages_processed[5m]) == 0 and queue_depth > 0
        for: 10m
        labels:
          severity: critical
        annotations:
          summary: "Traitement de la queue {{ $labels.queue }} bloqué"
          description: "Aucun message traité alors que {{ $value }} messages en attente"
```

## Maintenance et Évolution des Règles

### Audit Régulier

**Tous les 3 mois** :
- Réviser les alertes qui se déclenchent fréquemment
- Supprimer les alertes qui ne se déclenchent jamais
- Ajuster les seuils selon les nouvelles observations
- Mettre à jour la documentation

### Versioning

Gardez vos règles d'alerte dans un système de contrôle de version (Git) :

```bash
/monitoring-config/
├── README.md
├── alerts/
│   ├── v1.0/
│   │   ├── node-alerts.yml
│   │   └── pod-alerts.yml
│   └── v1.1/
│       ├── node-alerts.yml
│       └── pod-alerts.yml
└── CHANGELOG.md
```

### Changelog

Documentez chaque changement :

```markdown
## [1.1.0] - 2025-01-15
### Added
- Alerte DatabaseConnectionPoolNearLimit
- Alerte QueueBacklogGrowing

### Changed
- HighPodMemory : seuil augmenté de 80% à 85%
- PodCrashLooping : durée réduite de 10m à 5m

### Removed
- Alerte ObsoleteService (service supprimé)
```

## Métriques sur les Alertes

Créez des alertes pour surveiller votre système d'alerting lui-même :

```yaml
groups:
  - name: meta_alerts
    rules:
      - alert: TooManyAlerts
        expr: count(ALERTS{alertstate="firing"}) > 20
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "Trop d'alertes actives"
          description: "{{ $value }} alertes en cours - possible problème systémique"

      - alert: AlertmanagerDown
        expr: up{job="alertmanager"} == 0
        for: 2m
        labels:
          severity: critical
        annotations:
          summary: "Alertmanager inaccessible"
          description: "Le système d'alerting ne fonctionne plus !"

      - alert: AlertNeverFiring
        expr: time() - max(ALERTS_FOR_STATE{alertstate="firing"}) by(alertname) > 86400 * 30
        labels:
          severity: info
        annotations:
          summary: "L'alerte {{ $labels.alertname }} ne s'est jamais déclenchée"
          description: "Peut-être inutile ou mal configurée ?"
```

## Conclusion

La configuration des règles d'alerte est un art qui s'affine avec l'expérience. Les points essentiels :

- **Commencez simple** : Quelques alertes essentielles bien configurées valent mieux que des dizaines d'alertes bruyantes
- **Itérez** : Ajustez vos règles en fonction des retours
- **Documentez** : Chaque alerte doit avoir une procédure de résolution
- **Testez** : Validez vos alertes avant de les déployer en production
- **Maintenez** : Révisez régulièrement vos règles

Avec des règles d'alerte bien pensées, vous transformez votre monitoring passif en système proactif qui protège vos services.

## Points Clés à Retenir

✓ Une règle d'alerte = nom + expression PromQL + durée + labels + annotations

✓ Le paramètre `for` évite les fausses alertes en attendant que le problème persiste

✓ Les labels servent au routage, les annotations à la documentation

✓ Organisez vos règles en groupes thématiques dans des fichiers séparés

✓ Testez vos expressions PromQL avant de créer les alertes

✓ Calibrez vos seuils en observant vos métriques normales

✓ Créez des alertes multicouches (warning → critical)

✓ Documentez chaque alerte avec un runbook

✓ Versionez vos règles et auditez-les régulièrement

✓ Surveillez votre système d'alerting lui-même

---

**Prochaine section** : 14.4 Routing et grouping - Nous approfondirons les stratégies de routage et de regroupement dans Alertmanager.

⏭️ [Routing et grouping](/14-alerting-et-notifications/04-routing-et-grouping.md)
