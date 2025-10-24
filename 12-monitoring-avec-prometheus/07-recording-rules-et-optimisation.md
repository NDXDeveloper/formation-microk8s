🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 12.7 Recording rules et optimisation

## Introduction

Imaginez que vous avez une requête PromQL complexe qui prend 5 secondes à s'exécuter. Cette requête est utilisée :
- Dans 10 dashboards Grafana différents
- Dans 5 règles d'alerte
- Actualisée toutes les 30 secondes

Résultat : Prometheus exécute cette requête coûteuse des dizaines de fois par minute ! Votre serveur Prometheus ralentit, les dashboards deviennent lents, et vous consommez beaucoup de ressources inutilement.

**Solution** : Les **Recording Rules**.

### Qu'est-ce qu'une Recording Rule ?

Une **recording rule** (règle d'enregistrement) est une requête PromQL qui :
1. S'exécute à intervalle régulier (ex: toutes les minutes)
2. Calcule un résultat
3. **Enregistre ce résultat comme une nouvelle métrique**
4. Cette nouvelle métrique peut ensuite être interrogée instantanément

**Analogie** :

Imaginez un restaurant :
- **Sans recording rule** : Chaque client commande une pizza. Le cuisinier prépare chaque pizza à la demande (5 minutes par pizza).
- **Avec recording rule** : Le cuisinier prépare des pizzas à l'avance toutes les 10 minutes. Quand un client commande, la pizza est déjà prête (instantané).

La recording rule "pré-calcule" et "pré-cuisine" les résultats pour qu'ils soient disponibles immédiatement.

## Pourquoi utiliser des Recording Rules ?

### 1. Performance améliorée

**Sans recording rule** :

```promql
# Requête complexe exécutée à chaque chargement du dashboard
sum(rate(container_cpu_usage_seconds_total[5m])) by (namespace) /
sum(rate(container_cpu_usage_seconds_total[5m]))
```

**Avec recording rule** :

```promql
# Requête simple qui utilise une métrique pré-calculée
namespace:container_cpu_usage:ratio
```

La deuxième requête est **beaucoup plus rapide** car le calcul complexe a déjà été fait.

### 2. Cohérence des résultats

Si la même requête complexe est utilisée dans 10 endroits différents, il y a un risque de légères variations (car elles sont évaluées à des moments légèrement différents). Avec une recording rule, tout le monde utilise la même métrique pré-calculée.

### 3. Simplification des requêtes

Les recording rules permettent de donner des **noms significatifs** à des calculs complexes :

```promql
# ❌ Difficile à comprendre
rate(http_requests_total{status=~"2.."}[5m]) / rate(http_requests_total[5m])

# ✅ Clair et explicite
http:requests:success_rate
```

### 4. Réduction de la charge

Moins de calculs répétitifs = moins de charge CPU sur Prometheus = plus de ressources disponibles pour d'autres tâches.

## Anatomie d'une Recording Rule

### Structure de base

```yaml
groups:
  - name: example_rules              # Nom du groupe de règles
    interval: 30s                    # Fréquence d'évaluation (optionnel, défaut: global.evaluation_interval)
    rules:
      - record: job:http_requests:rate5m    # Nom de la métrique créée
        expr: rate(http_requests_total[5m]) # Requête PromQL à exécuter
        labels:                             # Labels additionnels (optionnel)
          environment: production
```

**Décomposition** :

1. **groups** : Conteneur pour organiser vos règles
2. **name** : Nom du groupe (pour l'organisation)
3. **interval** : À quelle fréquence évaluer ces règles
4. **record** : Nom de la nouvelle métrique qui sera créée
5. **expr** : La requête PromQL à pré-calculer
6. **labels** : Labels supplémentaires à ajouter (optionnel)

### Convention de nommage

Prometheus recommande une convention de nommage stricte pour les recording rules :

```
niveau:métrique:opération
```

**Exemples** :

```
# Agrégation par job
job:http_requests_total:rate5m

# Agrégation par instance
instance:node_cpu:avg_rate5m

# Agrégation par namespace
namespace:container_memory_usage:sum

# Agrégation globale
cluster:cpu_usage:ratio
```

**Composants** :

1. **Niveau d'agrégation** : job, instance, namespace, node, cluster, etc.
2. **Métrique source** : La métrique de base
3. **Opération** : rate5m, sum, avg, ratio, etc.

### Pourquoi cette convention ?

Elle permet de **comprendre immédiatement** ce que fait la métrique :

```promql
namespace:container_cpu_usage:rate5m
```

Lecture : "Taux CPU par namespace, calculé sur 5 minutes"

## Exemples de Recording Rules

### Exemple 1 : Taux de requêtes HTTP

**Problème** : Calculer `rate(http_requests_total[5m])` est coûteux si fait des centaines de fois.

**Solution** :

```yaml
groups:
  - name: http_rules
    interval: 30s
    rules:
      # Taux de requêtes par job
      - record: job:http_requests_total:rate5m
        expr: sum(rate(http_requests_total[5m])) by (job)

      # Taux de requêtes par job et status
      - record: job:http_requests_total:rate5m
        expr: sum(rate(http_requests_total[5m])) by (job, status)
```

**Utilisation** :

```promql
# Au lieu de :
sum(rate(http_requests_total[5m])) by (job)

# Utilisez simplement :
job:http_requests_total:rate5m
```

### Exemple 2 : Utilisation CPU par namespace

**Problème** : Calculer l'utilisation CPU agrégée par namespace est une opération courante mais coûteuse.

**Solution** :

```yaml
groups:
  - name: cpu_rules
    interval: 1m
    rules:
      # CPU total par namespace
      - record: namespace:container_cpu_usage_seconds_total:sum_rate
        expr: |
          sum(rate(container_cpu_usage_seconds_total{container!=""}[5m]))
          by (namespace)

      # CPU moyen par pod dans chaque namespace
      - record: namespace:container_cpu_usage:avg
        expr: |
          avg(rate(container_cpu_usage_seconds_total{container!=""}[5m]))
          by (namespace)
```

**Utilisation** :

```promql
# CPU total par namespace
namespace:container_cpu_usage_seconds_total:sum_rate

# CPU moyen
namespace:container_cpu_usage:avg

# Top 5 des namespaces
topk(5, namespace:container_cpu_usage_seconds_total:sum_rate)
```

### Exemple 3 : Ratio de succès HTTP

**Problème** : Calculer le taux de succès nécessite deux requêtes et une division.

**Solution** :

```yaml
groups:
  - name: http_success_rules
    rules:
      # Requêtes réussies (2xx)
      - record: job:http_requests_success:rate5m
        expr: |
          sum(rate(http_requests_total{status=~"2.."}[5m])) by (job)

      # Toutes les requêtes
      - record: job:http_requests:rate5m
        expr: |
          sum(rate(http_requests_total[5m])) by (job)

      # Ratio de succès
      - record: job:http_requests:success_ratio
        expr: |
          job:http_requests_success:rate5m /
          job:http_requests:rate5m
```

**Utilisation** :

```promql
# Taux de succès en %
job:http_requests:success_ratio * 100

# Applications avec moins de 95% de succès
job:http_requests:success_ratio < 0.95
```

### Exemple 4 : Utilisation mémoire agrégée

**Solution** :

```yaml
groups:
  - name: memory_rules
    rules:
      # Mémoire par namespace
      - record: namespace:container_memory_usage:sum
        expr: |
          sum(container_memory_working_set_bytes{container!=""})
          by (namespace)

      # Mémoire totale du cluster
      - record: cluster:container_memory_usage:sum
        expr: |
          sum(container_memory_working_set_bytes{container!=""})

      # Ratio mémoire par namespace
      - record: namespace:container_memory_usage:ratio
        expr: |
          namespace:container_memory_usage:sum /
          cluster:container_memory_usage:sum
```

### Exemple 5 : Recording rules hiérarchiques

Les recording rules peuvent s'appuyer sur d'autres recording rules :

```yaml
groups:
  - name: hierarchical_rules
    rules:
      # Niveau 1 : Agrégation par pod
      - record: pod:container_cpu_usage:sum_rate
        expr: |
          sum(rate(container_cpu_usage_seconds_total{container!=""}[5m]))
          by (namespace, pod)

      # Niveau 2 : Agrégation par namespace (utilise niveau 1)
      - record: namespace:container_cpu_usage:sum_rate
        expr: |
          sum(pod:container_cpu_usage:sum_rate) by (namespace)

      # Niveau 3 : Total cluster (utilise niveau 2)
      - record: cluster:container_cpu_usage:sum_rate
        expr: |
          sum(namespace:container_cpu_usage:sum_rate)
```

Cette approche **hiérarchique** est très efficace car chaque niveau réutilise le précédent.

## Créer des Recording Rules dans MicroK8s

### Méthode 1 : Via un PrometheusRule (Recommandé)

Avec Prometheus Operator (utilisé par MicroK8s), vous créez un objet `PrometheusRule` :

```yaml
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: example-recording-rules
  namespace: monitoring
  labels:
    prometheus: kube-prometheus    # Important pour être détecté
    role: alert-rules
spec:
  groups:
  - name: example_rules
    interval: 30s
    rules:
    - record: job:http_requests_total:rate5m
      expr: sum(rate(http_requests_total[5m])) by (job)

    - record: namespace:container_cpu_usage:sum_rate
      expr: |
        sum(rate(container_cpu_usage_seconds_total{container!=""}[5m]))
        by (namespace)
```

**Appliquer la règle** :

```bash
microk8s kubectl apply -f recording-rules.yaml
```

### Méthode 2 : Éditer la ConfigMap (Avancé)

Vous pouvez aussi modifier directement la ConfigMap de Prometheus, mais c'est plus complexe et moins recommandé.

### Vérifier que les règles sont chargées

#### Via l'interface Prometheus

1. Accédez à Prometheus : `http://localhost:9090`
2. Allez dans **Status > Rules**
3. Vous devriez voir vos recording rules dans la liste

#### Via kubectl

```bash
# Lister tous les PrometheusRules
microk8s kubectl get prometheusrules -n monitoring

# Voir les détails d'une règle
microk8s kubectl describe prometheusrule example-recording-rules -n monitoring
```

#### Vérifier que la métrique existe

Après quelques minutes, votre nouvelle métrique devrait apparaître dans Prometheus :

```promql
# Dans l'interface Prometheus, exécutez :
job:http_requests_total:rate5m
```

Si la métrique apparaît, votre recording rule fonctionne ! 🎉

## Bonnes pratiques

### 1. Nommez vos règles de manière cohérente

```yaml
# ✅ Bon : suit la convention niveau:métrique:opération
- record: namespace:container_cpu_usage:sum_rate
  expr: sum(rate(container_cpu_usage_seconds_total[5m])) by (namespace)

# ❌ Mauvais : nom peu clair
- record: cpu_usage_thing
  expr: sum(rate(container_cpu_usage_seconds_total[5m])) by (namespace)
```

### 2. Choisissez le bon intervalle

```yaml
# Pour des métriques changent rapidement (métriques applicatives)
- name: fast_rules
  interval: 15s

# Pour des métriques qui changent lentement (agrégations système)
- name: slow_rules
  interval: 2m
```

**Compromis** :
- Intervalle court = données plus fraîches, mais plus de charge CPU
- Intervalle long = moins de charge, mais données moins à jour

**Règle générale** : Commencez avec 1 minute, ajustez ensuite.

### 3. Gardez les règles simples

```yaml
# ✅ Bon : simple et efficace
- record: namespace:http_requests:rate5m
  expr: sum(rate(http_requests_total[5m])) by (namespace)

# ❌ Mauvais : trop complexe pour une recording rule
- record: complex_metric
  expr: |
    (sum(rate(http_requests_total[5m])) by (namespace) /
     sum(rate(http_requests_total[5m])) by (namespace) offset 1h) *
    (avg(container_memory_usage_bytes) by (namespace) / 1024 / 1024)
```

Si une requête est trop complexe, décomposez-la en plusieurs recording rules.

### 4. Documentez vos règles

```yaml
groups:
  - name: application_rules
    # Ces règles pré-calculent les métriques pour le dashboard principal
    # Contact: team-platform@example.com
    rules:
      # Calcule le taux de requêtes HTTP par service
      # Utilisé dans le dashboard "Service Overview" et l'alerte "HighRequestRate"
      - record: service:http_requests:rate5m
        expr: sum(rate(http_requests_total[5m])) by (service)
```

### 5. Évitez la duplication

```yaml
# ❌ Mauvais : duplication
- record: namespace:cpu:sum
  expr: sum(rate(container_cpu_usage_seconds_total[5m])) by (namespace)

- record: namespace:cpu_usage:total
  expr: sum(rate(container_cpu_usage_seconds_total[5m])) by (namespace)

# ✅ Bon : une seule règle avec un nom clair
- record: namespace:container_cpu_usage:sum_rate
  expr: sum(rate(container_cpu_usage_seconds_total[5m])) by (namespace)
```

### 6. Organisez en groupes logiques

```yaml
groups:
  # Règles CPU
  - name: cpu_recording_rules
    interval: 1m
    rules:
      - record: namespace:container_cpu_usage:sum_rate
        expr: [...]
      - record: pod:container_cpu_usage:sum_rate
        expr: [...]

  # Règles mémoire
  - name: memory_recording_rules
    interval: 1m
    rules:
      - record: namespace:container_memory_usage:sum
        expr: [...]
      - record: pod:container_memory_usage:sum
        expr: [...]

  # Règles applicatives HTTP
  - name: http_recording_rules
    interval: 30s
    rules:
      - record: job:http_requests:rate5m
        expr: [...]
```

### 7. Testez vos règles avant de les déployer

```bash
# Testez la requête dans l'UI Prometheus d'abord
# Vérifiez qu'elle retourne les résultats attendus
# Vérifiez le temps d'exécution

# Puis créez la recording rule
# Attendez quelques minutes
# Vérifiez que la nouvelle métrique existe
```

### 8. Utilisez des labels additionnels avec parcimonie

```yaml
# ✅ Bon : les labels proviennent de la requête
- record: namespace:cpu:sum
  expr: sum(rate(container_cpu_usage_seconds_total[5m])) by (namespace)

# ⚠️ Utilisez avec parcimonie : labels additionnels
- record: namespace:cpu:sum
  expr: sum(rate(container_cpu_usage_seconds_total[5m])) by (namespace)
  labels:
    team: platform
    environment: production
```

Les labels additionnels sont utiles mais peuvent augmenter la cardinalité.

## Optimisation de Prometheus

Au-delà des recording rules, voici d'autres techniques pour optimiser Prometheus.

### 1. Ajuster la rétention des données

Par défaut, Prometheus conserve 15 jours de données. Si vous avez peu d'espace disque :

```yaml
# Dans la configuration Prometheus (via kubectl edit)
storage:
  tsdb:
    retention.time: 7d      # 7 jours au lieu de 15
    retention.size: 10GB    # Ou limiter par taille
```

**Compromis** :
- Moins de rétention = moins d'espace disque
- Mais vous perdez l'historique pour le dépannage

### 2. Ajuster les intervalles de scraping

```yaml
# Scraping plus fréquent (plus de données, plus de charge)
scrape_interval: 15s

# Scraping moins fréquent (moins de charge, moins précis)
scrape_interval: 60s
```

**Recommandation** : 30 secondes est un bon compromis pour la plupart des cas.

### 3. Filtrer les métriques non nécessaires

Si certaines métriques ne vous servent jamais, filtrez-les :

```yaml
# Dans un ServiceMonitor
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
spec:
  endpoints:
  - port: metrics
    metricRelabelings:
    # Supprimer les métriques Go internes
    - sourceLabels: [__name__]
      regex: 'go_.*'
      action: drop

    # Garder seulement certaines métriques
    - sourceLabels: [__name__]
      regex: 'http_.*|app_.*'
      action: keep
```

### 4. Réduire la cardinalité

La **cardinalité** = nombre de séries temporelles uniques.

**Problème** : Labels avec beaucoup de valeurs uniques :

```promql
# ❌ Mauvais : user_id a des milliers de valeurs
http_requests_total{user_id="12345", endpoint="/api/users"}

# ✅ Bon : utiliser des labels avec peu de valeurs
http_requests_total{endpoint="/api/users", method="GET"}
```

**Règle d'or** : Évitez les labels avec plus de 100 valeurs uniques.

### 5. Utiliser le remote storage (avancé)

Pour conserver un historique long sans surcharger Prometheus :

```
Prometheus (15 jours) → Remote Storage (1 an)
                        (Thanos, Cortex, etc.)
```

Cela permet de :
- Garder peu de données localement
- Archiver l'historique ailleurs
- Requêter l'historique quand nécessaire

**Note** : Configuration avancée, pas nécessaire pour un lab.

### 6. Monitoring de Prometheus lui-même

Prometheus expose ses propres métriques ! Surveillez :

```promql
# Utilisation mémoire de Prometheus
process_resident_memory_bytes

# Nombre de séries temporelles
prometheus_tsdb_symbol_table_size_bytes

# Durée d'un scrape
prometheus_target_interval_length_seconds

# Nombre de samples ingérés par seconde
rate(prometheus_tsdb_head_samples_appended_total[5m])
```

### 7. Activer la compression

```yaml
# Dans la configuration du scraping
scrape_configs:
  - job_name: 'my-app'
    scrape_interval: 30s
    compression: gzip    # Active la compression
```

Réduit la bande passante réseau.

## Recording Rules : Exemples complets pour MicroK8s

Voici un fichier de recording rules complet et prêt à l'emploi pour votre cluster MicroK8s :

```yaml
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: microk8s-recording-rules
  namespace: monitoring
  labels:
    prometheus: kube-prometheus
    role: alert-rules
spec:
  groups:

  # ===== Règles CPU =====
  - name: cpu_recording_rules
    interval: 1m
    rules:
    # CPU par namespace
    - record: namespace:container_cpu_usage_seconds_total:sum_rate
      expr: |
        sum(rate(container_cpu_usage_seconds_total{container!=""}[5m]))
        by (namespace)

    # CPU par pod
    - record: namespace_pod:container_cpu_usage_seconds_total:sum_rate
      expr: |
        sum(rate(container_cpu_usage_seconds_total{container!=""}[5m]))
        by (namespace, pod)

    # CPU total du cluster
    - record: cluster:container_cpu_usage_seconds_total:sum_rate
      expr: |
        sum(rate(container_cpu_usage_seconds_total{container!=""}[5m]))

    # Ratio CPU par namespace
    - record: namespace:container_cpu_usage:ratio
      expr: |
        namespace:container_cpu_usage_seconds_total:sum_rate /
        cluster:container_cpu_usage_seconds_total:sum_rate

  # ===== Règles Mémoire =====
  - name: memory_recording_rules
    interval: 1m
    rules:
    # Mémoire par namespace
    - record: namespace:container_memory_usage_bytes:sum
      expr: |
        sum(container_memory_working_set_bytes{container!=""})
        by (namespace)

    # Mémoire par pod
    - record: namespace_pod:container_memory_usage_bytes:sum
      expr: |
        sum(container_memory_working_set_bytes{container!=""})
        by (namespace, pod)

    # Mémoire totale du cluster
    - record: cluster:container_memory_usage_bytes:sum
      expr: |
        sum(container_memory_working_set_bytes{container!=""})

    # Ratio mémoire par namespace
    - record: namespace:container_memory_usage:ratio
      expr: |
        namespace:container_memory_usage_bytes:sum /
        cluster:container_memory_usage_bytes:sum

  # ===== Règles Réseau =====
  - name: network_recording_rules
    interval: 1m
    rules:
    # Trafic entrant par namespace (MB/s)
    - record: namespace:container_network_receive_bytes:rate
      expr: |
        sum(rate(container_network_receive_bytes_total{container!=""}[5m]))
        by (namespace) / 1024 / 1024

    # Trafic sortant par namespace (MB/s)
    - record: namespace:container_network_transmit_bytes:rate
      expr: |
        sum(rate(container_network_transmit_bytes_total{container!=""}[5m]))
        by (namespace) / 1024 / 1024

    # Trafic total par namespace
    - record: namespace:container_network_total_bytes:rate
      expr: |
        namespace:container_network_receive_bytes:rate +
        namespace:container_network_transmit_bytes:rate

  # ===== Règles Pods =====
  - name: pod_recording_rules
    interval: 1m
    rules:
    # Nombre de pods par namespace
    - record: namespace:kube_pod_info:count
      expr: |
        count(kube_pod_info) by (namespace)

    # Pods running par namespace
    - record: namespace:kube_pod_status_phase:count
      expr: |
        count(kube_pod_status_phase{phase="Running"}) by (namespace)

    # Pods not ready par namespace
    - record: namespace:kube_pod_status_ready:count_false
      expr: |
        count(kube_pod_status_ready{condition="false"}) by (namespace)

  # ===== Règles Node =====
  - name: node_recording_rules
    interval: 2m
    rules:
    # CPU disponible par node (%)
    - record: node:node_cpu_utilization:ratio
      expr: |
        100 - (avg(rate(node_cpu_seconds_total{mode="idle"}[5m]))
        by (instance) * 100)

    # Mémoire disponible par node (%)
    - record: node:node_memory_utilization:ratio
      expr: |
        (1 - (node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes)) * 100

    # Disque utilisé par node (%)
    - record: node:node_filesystem_utilization:ratio
      expr: |
        ((node_filesystem_size_bytes - node_filesystem_avail_bytes) /
        node_filesystem_size_bytes) * 100
```

**Pour appliquer ces règles** :

```bash
# Sauvegarder dans un fichier
cat > recording-rules.yaml << 'EOF'
[coller le contenu ci-dessus]
EOF

# Appliquer
microk8s kubectl apply -f recording-rules.yaml

# Vérifier
microk8s kubectl get prometheusrules -n monitoring

# Attendre 1-2 minutes, puis vérifier dans Prometheus
# Status > Rules
```

## Vérifier l'impact des Recording Rules

### Avant et après

**Mesurez la performance avant** :

1. Dans Prometheus, exécutez une requête complexe
2. Notez le temps d'exécution (affiché en bas de l'UI)

**Exemple** :

```promql
sum(rate(container_cpu_usage_seconds_total{container!=""}[5m])) by (namespace)
# Temps: 0.523s
```

**Créez la recording rule, attendez quelques minutes**

**Mesurez après** :

```promql
namespace:container_cpu_usage_seconds_total:sum_rate
# Temps: 0.012s
```

**Gain** : 40x plus rapide ! 🚀

### Vérifier l'utilisation des ressources

```promql
# Utilisation CPU de Prometheus lui-même
rate(process_cpu_seconds_total{job="prometheus"}[5m])

# Utilisation mémoire
process_resident_memory_bytes{job="prometheus"}

# Nombre de séries temporelles
prometheus_tsdb_head_series
```

Si après avoir ajouté des recording rules, ces métriques augmentent beaucoup, vous avez peut-être créé trop de séries ou des règles trop fréquentes.

## Quand NE PAS utiliser de Recording Rules

### 1. Requêtes déjà rapides

Si une requête s'exécute en moins de 100ms et n'est utilisée que rarement, une recording rule n'apporte rien.

### 2. Requêtes rarement utilisées

Pas besoin de pré-calculer une métrique qui n'est consultée qu'une fois par semaine.

### 3. Requêtes très spécifiques

Les recording rules sont pour les agrégations générales, pas pour des requêtes ad-hoc spécifiques.

```promql
# ❌ Pas besoin de recording rule
container_memory_usage_bytes{pod="my-specific-pod-abc123"}

# ✅ Bon candidat pour recording rule
sum(container_memory_usage_bytes) by (namespace)
```

### 4. Environnement avec peu de métriques

Si votre cluster est petit (peu de pods, peu de métriques), les recording rules apportent peu de bénéfices.

## Dépannage des Recording Rules

### Problème : La métrique n'apparaît pas

**Vérifications** :

1. **Le PrometheusRule est-il créé ?**

```bash
microk8s kubectl get prometheusrule -n monitoring
```

2. **A-t-il les bons labels ?**

```bash
microk8s kubectl get prometheusrule microk8s-recording-rules -n monitoring -o yaml | grep -A2 labels
```

Doit contenir :

```yaml
labels:
  prometheus: kube-prometheus
```

3. **La règle est-elle visible dans Prometheus ?**

UI Prometheus → Status → Rules

4. **Y a-t-il des erreurs ?**

```bash
# Logs de Prometheus
microk8s kubectl logs -n monitoring prometheus-k8s-0 -c prometheus | grep -i error
```

### Problème : Erreur de syntaxe dans la requête

Si votre requête PromQL a une erreur, elle apparaîtra dans les logs de Prometheus :

```bash
microk8s kubectl logs -n monitoring prometheus-k8s-0 -c prometheus | tail -50
```

Cherchez des messages comme :

```
level=error msg="Error evaluating recording rule" err="parse error at..."
```

### Problème : Résultats inattendus

1. **Testez la requête manuellement d'abord** dans l'UI Prometheus
2. **Vérifiez les labels** : Assurez-vous que les labels correspondent à vos attentes
3. **Vérifiez le type de données** : Counter vs Gauge

## Résumé

Dans cette section, vous avez appris :

✅ **Ce que sont les recording rules** et pourquoi elles sont importantes
✅ **Comment créer** des recording rules dans MicroK8s avec PrometheusRule
✅ **Les conventions de nommage** pour les recording rules
✅ **Des exemples pratiques** pour CPU, mémoire, réseau, pods
✅ **Les bonnes pratiques** d'organisation et d'optimisation
✅ **Comment optimiser Prometheus** au-delà des recording rules
✅ **Un ensemble complet** de recording rules prêtes à l'emploi
✅ **Comment vérifier** l'impact des recording rules
✅ **Le dépannage** des problèmes courants

**Points clés à retenir** :

1. Les recording rules **pré-calculent** des requêtes complexes
2. Elles **améliorent les performances** significativement
3. Utilisez la **convention de nommage** `niveau:métrique:opération`
4. **Organisez** vos règles en groupes logiques
5. **Testez** d'abord la requête avant de créer la règle
6. **Documentez** vos règles pour votre futur vous
7. Ne créez des recording rules que pour les **requêtes fréquentes et coûteuses**

Les recording rules sont un outil puissant qui transforme Prometheus d'un système réactif en un système proactif qui anticipe vos besoins de requêtes.

Dans la prochaine section (12.8), nous explorerons la **Haute Disponibilité de Prometheus**, pour rendre votre système de monitoring encore plus robuste.

---

**Conseil pratique** : Commencez avec les recording rules d'exemple fournies dans cette section. Observez leur impact pendant quelques jours, puis créez vos propres règles pour vos cas d'usage spécifiques. L'optimisation est un processus itératif !

⏭️ [Haute disponibilité Prometheus](/12-monitoring-avec-prometheus/08-haute-disponibilite-prometheus.md)
