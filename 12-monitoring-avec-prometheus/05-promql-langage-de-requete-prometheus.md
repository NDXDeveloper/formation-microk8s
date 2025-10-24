🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 12.5 PromQL : langage de requête Prometheus

## Introduction

Maintenant que Prometheus collecte des métriques de votre cluster, comment les exploiter ? Comment répondre à des questions comme :
- "Quelle est l'utilisation CPU moyenne de mon application ?"
- "Combien de requêtes HTTP reçoit mon API par seconde ?"
- "Quels pods consomment le plus de mémoire ?"

C'est là qu'intervient **PromQL** (Prometheus Query Language), le langage de requête de Prometheus.

### Analogie avec SQL

Si vous connaissez SQL pour interroger des bases de données :
- **SQL** interroge des tables avec des lignes et des colonnes
- **PromQL** interroge des séries temporelles avec des valeurs et des timestamps

Mais PromQL est spécialement conçu pour les données temporelles, avec des fonctions pour calculer des taux, des moyennes sur des périodes, etc.

### Pourquoi apprendre PromQL ?

1. **Comprendre vos métriques** : Extraire les informations pertinentes
2. **Créer des dashboards** : Les graphiques Grafana utilisent PromQL
3. **Définir des alertes** : Les règles d'alerte sont écrites en PromQL
4. **Diagnostiquer des problèmes** : Analyser les métriques pendant un incident
5. **Optimiser les performances** : Identifier les goulots d'étranglement

## Les fondamentaux

### Qu'est-ce qu'une métrique ?

Une **métrique** est une mesure numérique d'une ressource ou d'un comportement, identifiée par un nom et des labels.

**Exemple** :

```
http_requests_total{method="GET", status="200", instance="app-1"} 1547
```

Décomposons cela :
- **http_requests_total** : Nom de la métrique
- **{...}** : Labels (clés-valeurs qui identifient la métrique)
- **1547** : Valeur actuelle

### Qu'est-ce qu'une série temporelle ?

Une **série temporelle** (time series) est une séquence de valeurs d'une métrique dans le temps.

**Exemple** :

```
http_requests_total{method="GET", status="200"} 1547 @10:00:00
http_requests_total{method="GET", status="200"} 1552 @10:00:30
http_requests_total{method="GET", status="200"} 1560 @10:01:00
```

Chaque combinaison unique de métrique + labels = une série temporelle distincte.

### Les 4 types de métriques Prometheus

Comprendre les types de métriques est crucial pour bien utiliser PromQL.

#### 1. Counter (Compteur)

Un **Counter** ne fait qu'augmenter (ou se réinitialise à 0 lors d'un redémarrage).

**Exemples** :
- Nombre total de requêtes HTTP
- Nombre total d'erreurs
- Octets transférés
- Tâches complétées

**Caractéristiques** :
- ✅ Ne fait qu'augmenter
- ✅ Peut se réinitialiser (ex: redémarrage du pod)
- ❌ Ne diminue jamais

**Convention de nommage** : Se termine par `_total`

```
http_requests_total
error_count_total
bytes_sent_total
```

**Utilisation en PromQL** :

Un counter seul n'est pas très utile. Vous voulez généralement connaître le **taux** de changement :

```promql
# Taux de requêtes par seconde sur les 5 dernières minutes
rate(http_requests_total[5m])
```

#### 2. Gauge (Jauge)

Un **Gauge** peut augmenter ou diminuer.

**Exemples** :
- Utilisation CPU (%)
- Utilisation mémoire (bytes)
- Nombre de connexions actives
- Température
- Taille d'une file d'attente

**Caractéristiques** :
- ✅ Peut augmenter
- ✅ Peut diminuer
- ✅ Représente une valeur instantanée

**Utilisation en PromQL** :

Un gauge peut être utilisé directement :

```promql
# Utilisation mémoire actuelle
container_memory_usage_bytes

# Moyenne sur 5 minutes
avg_over_time(container_memory_usage_bytes[5m])
```

#### 3. Histogram (Histogramme)

Un **Histogram** échantillonne des observations et les compte dans des buckets configurables.

**Exemples** :
- Durée des requêtes HTTP
- Taille des réponses
- Temps de traitement

**Utilisation** : Calculer des percentiles (p50, p95, p99)

**Format** :

Un histogram génère plusieurs séries temporelles :

```
http_request_duration_seconds_bucket{le="0.1"} 1000    # <= 0.1s
http_request_duration_seconds_bucket{le="0.5"} 1450    # <= 0.5s
http_request_duration_seconds_bucket{le="1.0"} 1480    # <= 1.0s
http_request_duration_seconds_bucket{le="+Inf"} 1500   # Total
http_request_duration_seconds_sum 450                  # Somme
http_request_duration_seconds_count 1500               # Nombre
```

**Utilisation en PromQL** :

```promql
# Calcul du percentile 95
histogram_quantile(0.95, rate(http_request_duration_seconds_bucket[5m]))
```

#### 4. Summary (Résumé)

Similaire à l'histogram mais calcule les quantiles côté client.

**Moins utilisé** que l'histogram dans Kubernetes.

### Instant Vector vs Range Vector

C'est un concept clé de PromQL.

#### Instant Vector

Un **instant vector** est un ensemble de séries temporelles avec **une seule valeur par série** à un instant donné.

**Exemple** :

```promql
http_requests_total
```

Résultat :

```
http_requests_total{method="GET", instance="app-1"} 1547
http_requests_total{method="POST", instance="app-1"} 234
http_requests_total{method="GET", instance="app-2"} 1892
```

C'est comme une photo instantanée de toutes les séries.

#### Range Vector

Un **range vector** est un ensemble de séries temporelles avec **plusieurs valeurs** sur une période de temps.

**Exemple** :

```promql
http_requests_total[5m]
```

Résultat : Pour chaque série, toutes les valeurs des 5 dernières minutes.

```
http_requests_total{method="GET"}
  1500 @10:00:00
  1510 @10:00:30
  1520 @10:01:00
  ...
  1547 @10:05:00
```

**Utilisation** : Les range vectors sont utilisés avec des fonctions qui ont besoin de l'historique :

```promql
rate(http_requests_total[5m])      # Calcule le taux sur 5 minutes
avg_over_time(cpu_usage[1h])       # Moyenne sur 1 heure
```

**Durées possibles** :
- `ms` : millisecondes
- `s` : secondes
- `m` : minutes
- `h` : heures
- `d` : jours
- `w` : semaines
- `y` : années

Exemples : `30s`, `5m`, `1h`, `7d`

## Sélecteurs de séries temporelles

### Sélecteur simple

Sélectionner toutes les séries d'une métrique :

```promql
http_requests_total
```

### Sélection par label (égalité)

Sélectionner les séries avec un label spécifique :

```promql
http_requests_total{method="GET"}
```

Plusieurs labels :

```promql
http_requests_total{method="GET", status="200"}
```

### Opérateurs de matching

#### Égalité `=`

```promql
http_requests_total{method="GET"}
```

#### Inégalité `!=`

```promql
http_requests_total{method!="GET"}  # Tout sauf GET
```

#### Regex `=~`

Correspondance avec une expression régulière :

```promql
http_requests_total{method=~"GET|POST"}  # GET ou POST
http_requests_total{status=~"5.."}       # Tous les codes 5xx
http_requests_total{pod=~"app-.*"}       # Pods commençant par "app-"
```

#### Regex négative `!~`

Ne correspond pas à l'expression régulière :

```promql
http_requests_total{status!~"2.."}  # Tout sauf 2xx
```

### Exemples pratiques de sélection

```promql
# Toutes les requêtes réussies (200, 201, 204)
http_requests_total{status=~"20."}

# Toutes les erreurs (4xx et 5xx)
http_requests_total{status=~"[45].."}

# Application spécifique dans le namespace production
http_requests_total{namespace="production", app="api"}

# Tous les pods sauf ceux du système
kube_pod_info{namespace!~"kube-.*"}
```

## Opérateurs arithmétiques

PromQL supporte les opérateurs mathématiques de base.

### Addition `+`

```promql
# Ajouter 100 à chaque valeur
http_requests_total + 100
```

### Soustraction `-`

```promql
# Calculer l'espace disque libre
node_filesystem_size_bytes - node_filesystem_avail_bytes
```

### Multiplication `*`

```promql
# Convertir de bytes en megabytes
container_memory_usage_bytes / 1024 / 1024
```

### Division `/`

```promql
# Pourcentage d'utilisation mémoire
(container_memory_usage_bytes / container_memory_limit_bytes) * 100
```

### Modulo `%`

```promql
node_cpu_seconds_total % 60  # Reste de la division par 60
```

### Puissance `^`

```promql
http_requests_total ^ 2  # Élévation au carré
```

## Opérateurs de comparaison

Ces opérateurs retournent des séries temporelles qui correspondent à la condition.

### Égalité `==`

```promql
kube_pod_status_phase{phase="Running"} == 1
```

### Inégalité `!=`

```promql
up != 1  # Toutes les targets down
```

### Comparaisons `>`, `<`, `>=`, `<=`

```promql
# Pods utilisant plus de 80% de mémoire
(container_memory_usage_bytes / container_memory_limit_bytes) > 0.8

# CPU inférieur à 10%
rate(container_cpu_usage_seconds_total[5m]) < 0.1
```

## Fonctions essentielles

### rate() - Taux de changement par seconde

**Utilisation** : Calculer le taux pour les **counters**.

**Syntaxe** : `rate(counter[durée])`

**Exemple** :

```promql
# Requêtes par seconde (moyenne sur 5 minutes)
rate(http_requests_total[5m])
```

**Comment ça marche ?**

1. Prend les valeurs sur 5 minutes
2. Calcule la différence entre la dernière et la première valeur
3. Divise par le temps écoulé

**Gestion des redémarrages** : `rate()` détecte automatiquement les resets (counters qui retombent à 0).

**Conseil** : Utilisez toujours `rate()` avec les counters, jamais directement la valeur.

### irate() - Taux instantané

**Utilisation** : Taux basé sur les 2 derniers points seulement.

**Différence avec rate()** :
- `rate()` : Moyenne sur toute la période (plus lisse)
- `irate()` : Plus réactif mais plus volatile

**Exemple** :

```promql
# Taux instantané de requêtes
irate(http_requests_total[5m])
```

**Quand utiliser** :
- `rate()` pour les alertes et graphiques (plus stable)
- `irate()` pour les pics brefs et l'analyse en temps réel

### increase() - Augmentation totale

**Utilisation** : Augmentation totale d'un counter sur une période.

**Syntaxe** : `increase(counter[durée])`

**Exemple** :

```promql
# Nombre total de requêtes dans la dernière heure
increase(http_requests_total[1h])
```

**Relation avec rate()** :

```promql
increase(http_requests_total[5m]) == rate(http_requests_total[5m]) * 300
```

### sum() - Somme

**Utilisation** : Additionner des valeurs.

**Exemple** :

```promql
# Total de requêtes sur toutes les instances
sum(http_requests_total)
```

**Avec grouping (by)** :

```promql
# Total de requêtes par méthode HTTP
sum(http_requests_total) by (method)
```

Résultat :

```
{method="GET"} 5000
{method="POST"} 1200
{method="DELETE"} 50
```

**Sans certains labels (without)** :

```promql
# Somme en ignorant le label instance
sum(http_requests_total) without (instance)
```

### avg() - Moyenne

```promql
# Utilisation CPU moyenne de tous les conteneurs
avg(rate(container_cpu_usage_seconds_total[5m]))

# Moyenne par namespace
avg(rate(container_cpu_usage_seconds_total[5m])) by (namespace)
```

### min() et max()

```promql
# Utilisation mémoire minimale
min(container_memory_usage_bytes)

# Utilisation mémoire maximale par pod
max(container_memory_usage_bytes) by (pod)
```

### count() - Compter

```promql
# Nombre de pods running
count(kube_pod_status_phase{phase="Running"})

# Nombre de pods par namespace
count(kube_pod_info) by (namespace)
```

### topk() et bottomk()

**Utilisation** : Obtenir les N séries avec les valeurs les plus hautes/basses.

```promql
# Top 5 des pods consommant le plus de mémoire
topk(5, container_memory_usage_bytes)

# Top 3 des namespaces avec le plus de pods
topk(3, count(kube_pod_info) by (namespace))

# Bottom 5 des nodes avec le moins d'espace disque
bottomk(5, node_filesystem_free_bytes)
```

### abs() - Valeur absolue

```promql
abs(delta(http_requests_total[5m]))
```

### round() - Arrondir

```promql
# Arrondir à l'entier le plus proche
round(container_memory_usage_bytes / 1024 / 1024)

# Arrondir au dixième
round(cpu_usage, 0.1)
```

### sort() et sort_desc()

```promql
# Trier par ordre croissant
sort(container_memory_usage_bytes)

# Trier par ordre décroissant
sort_desc(container_memory_usage_bytes)
```

## Fonctions temporelles

### avg_over_time() - Moyenne sur une période

```promql
# Moyenne d'utilisation CPU sur la dernière heure
avg_over_time(rate(container_cpu_usage_seconds_total[5m])[1h:])
```

**Attention** : Syntaxe particulière avec le `:` après la durée.

### max_over_time() et min_over_time()

```promql
# Pic d'utilisation mémoire dans les dernières 24h
max_over_time(container_memory_usage_bytes[24h])

# Minimum de trafic dans la dernière heure
min_over_time(rate(http_requests_total[5m])[1h:])
```

### sum_over_time() et count_over_time()

```promql
# Somme des valeurs sur 1h
sum_over_time(metric[1h])

# Nombre de points de données sur 1h
count_over_time(metric[1h])
```

### delta() - Différence

**Utilisation** : Différence entre la première et dernière valeur (pour gauges).

```promql
# Changement d'utilisation mémoire en 5 minutes
delta(container_memory_usage_bytes[5m])
```

### deriv() - Dérivée

**Utilisation** : Taux de changement par seconde (pour gauges).

```promql
# Vitesse de croissance de l'utilisation disque
deriv(node_filesystem_avail_bytes[5m])
```

### predict_linear() - Prédiction linéaire

**Utilisation** : Prédire une valeur future basée sur une tendance linéaire.

```promql
# Prédire l'espace disque dans 4 heures
predict_linear(node_filesystem_avail_bytes[1h], 4 * 3600)
```

**Exemple d'alerte** : Alerter si le disque sera plein dans 4 heures.

```promql
predict_linear(node_filesystem_avail_bytes[1h], 4 * 3600) < 0
```

## Opérateurs logiques

### AND

Intersection de deux ensembles de séries :

```promql
# Pods running ET dans le namespace production
kube_pod_status_phase{phase="Running"} and kube_pod_info{namespace="production"}
```

### OR

Union de deux ensembles :

```promql
# Erreurs 4xx ou 5xx
http_requests_total{status=~"4.."} or http_requests_total{status=~"5.."}
```

### UNLESS

Séries du premier ensemble qui ne sont pas dans le second :

```promql
# Toutes les requêtes sauf les succès
http_requests_total unless http_requests_total{status="200"}
```

## Agrégations avancées

### group() - Grouper sans calculer

```promql
# Regrouper par namespace (retourne 1 pour chaque groupe)
group(kube_pod_info) by (namespace)
```

### stddev() et stdvar() - Écart-type et variance

```promql
# Écart-type de l'utilisation CPU
stddev(rate(container_cpu_usage_seconds_total[5m]))
```

### quantile() - Quantile

```promql
# Médiane (50e percentile)
quantile(0.5, container_memory_usage_bytes)

# 95e percentile
quantile(0.95, http_request_duration_seconds)
```

## Requêtes pratiques courantes

### Utilisation CPU

```promql
# Taux d'utilisation CPU par pod (0-1)
rate(container_cpu_usage_seconds_total[5m])

# En pourcentage (0-100)
rate(container_cpu_usage_seconds_total[5m]) * 100

# Top 10 des pods utilisant le plus de CPU
topk(10, rate(container_cpu_usage_seconds_total[5m]))

# Moyenne CPU par namespace
avg(rate(container_cpu_usage_seconds_total[5m])) by (namespace)
```

### Utilisation mémoire

```promql
# Utilisation mémoire actuelle
container_memory_usage_bytes

# En megabytes
container_memory_usage_bytes / 1024 / 1024

# Pourcentage d'utilisation
(container_memory_usage_bytes / container_memory_limit_bytes) * 100

# Pods dépassant 80% de leur limite
(container_memory_usage_bytes / container_memory_limit_bytes) > 0.8
```

### Trafic réseau

```promql
# Octets reçus par seconde
rate(container_network_receive_bytes_total[5m])

# En megabytes par seconde
rate(container_network_receive_bytes_total[5m]) / 1024 / 1024

# Trafic total (reçu + envoyé)
rate(container_network_receive_bytes_total[5m]) +
rate(container_network_transmit_bytes_total[5m])
```

### Utilisation disque

```promql
# Espace disque utilisé (en bytes)
node_filesystem_size_bytes - node_filesystem_avail_bytes

# Pourcentage d'utilisation
(node_filesystem_size_bytes - node_filesystem_avail_bytes) /
node_filesystem_size_bytes * 100

# Espace libre en GB
node_filesystem_avail_bytes / 1024 / 1024 / 1024

# Filesystems à plus de 80%
((node_filesystem_size_bytes - node_filesystem_avail_bytes) /
node_filesystem_size_bytes) > 0.8
```

### Métriques HTTP/API

```promql
# Requêtes par seconde
rate(http_requests_total[5m])

# Taux d'erreur (4xx + 5xx)
rate(http_requests_total{status=~"[45].."}[5m])

# Taux de succès
rate(http_requests_total{status=~"2.."}[5m])

# Ratio erreurs/total
rate(http_requests_total{status=~"[45].."}[5m]) /
rate(http_requests_total[5m])

# Pourcentage d'erreurs
(rate(http_requests_total{status=~"[45].."}[5m]) /
rate(http_requests_total[5m])) * 100
```

### Latence

```promql
# Latence moyenne (en secondes)
rate(http_request_duration_seconds_sum[5m]) /
rate(http_request_duration_seconds_count[5m])

# En millisecondes
(rate(http_request_duration_seconds_sum[5m]) /
rate(http_request_duration_seconds_count[5m])) * 1000

# Percentile 95
histogram_quantile(0.95,
  rate(http_request_duration_seconds_bucket[5m]))

# Percentile 99
histogram_quantile(0.99,
  rate(http_request_duration_seconds_bucket[5m]))
```

### État des pods

```promql
# Nombre de pods running
count(kube_pod_status_phase{phase="Running"})

# Pods par état
count(kube_pod_status_phase) by (phase)

# Pods en erreur (Failed, CrashLoopBackOff)
kube_pod_status_phase{phase=~"Failed|Unknown"}

# Nombre de restarts par pod
kube_pod_container_status_restarts_total

# Pods avec plus de 5 restarts
kube_pod_container_status_restarts_total > 5
```

### Targets Prometheus

```promql
# Nombre de targets up
count(up == 1)

# Nombre de targets down
count(up == 0)

# Targets down (liste)
up == 0

# Taux de disponibilité
count(up == 1) / count(up) * 100
```

## Combinaisons et requêtes complexes

### Disponibilité (SLA)

```promql
# Pourcentage de requêtes réussies (SLA)
(sum(rate(http_requests_total{status=~"2.."}[5m])) /
sum(rate(http_requests_total[5m]))) * 100

# Disponibilité > 99.9% ?
(sum(rate(http_requests_total{status=~"2.."}[5m])) /
sum(rate(http_requests_total[5m]))) > 0.999
```

### Saturation des ressources

```promql
# Pods proches de leur limite mémoire (> 90%)
(container_memory_usage_bytes / container_memory_limit_bytes) > 0.9

# Nodes avec CPU élevé
avg(rate(node_cpu_seconds_total{mode!="idle"}[5m])) by (instance) > 0.8
```

### Croissance des métriques

```promql
# Taux de croissance des erreurs
deriv(rate(http_errors_total[5m])[10m:])

# Augmentation du trafic en %
((rate(http_requests_total[5m]) -
  rate(http_requests_total[5m] offset 1h)) /
  rate(http_requests_total[5m] offset 1h)) * 100
```

## L'opérateur offset

**Utilisation** : Décaler dans le temps pour comparer.

```promql
# Trafic actuel
rate(http_requests_total[5m])

# Trafic il y a 1 heure
rate(http_requests_total[5m] offset 1h)

# Différence avec il y a 1 heure
rate(http_requests_total[5m]) -
rate(http_requests_total[5m] offset 1h)

# Variation en % par rapport à hier
((rate(http_requests_total[5m]) -
  rate(http_requests_total[5m] offset 24h)) /
  rate(http_requests_total[5m] offset 24h)) * 100
```

## Subqueries (Sous-requêtes)

**Syntaxe** : `<instant_query>[<durée>:<résolution>]`

**Exemple** :

```promql
# Maximum de la moyenne CPU sur 2 heures
max_over_time(
  avg(rate(container_cpu_usage_seconds_total[5m]))[2h:1m]
)
```

Décomposition :
1. `avg(rate(...[5m]))` : Moyenne CPU
2. `[2h:1m]` : Évalue cette requête toutes les minutes pendant 2h
3. `max_over_time(...)` : Prend le maximum

## Bonnes pratiques

### 1. Choisir la bonne durée de range

**Trop courte** : Volatile, pics aléatoires
**Trop longue** : Manque de réactivité

**Règle générale** :
- Pour les alertes : 4-5x l'intervalle de scraping (ex: scrape 30s → range 2-5m)
- Pour les dashboards : Selon le zoom temporel

```promql
# Bon pour alerte (lisse les variations)
rate(http_requests_total[5m])

# Trop court (trop volatile)
rate(http_requests_total[30s])

# Trop long pour une alerte (trop lent à réagir)
rate(http_requests_total[1h])
```

### 2. Utiliser by() pour agréger efficacement

```promql
# ❌ Mauvais : trop de séries
rate(http_requests_total[5m])

# ✅ Bon : agrégé par endpoint
sum(rate(http_requests_total[5m])) by (endpoint)
```

### 3. Limiter la cardinalité

Évitez les labels avec beaucoup de valeurs uniques (user_id, session_id, timestamp).

```promql
# ❌ Mauvais : user_id a des milliers de valeurs
http_requests_total{user_id="12345"}

# ✅ Bon : Agrégé par endpoint
sum(http_requests_total) by (endpoint, method)
```

### 4. Nommer clairement les métriques

**Conventions Prometheus** :
- Nom descriptif : `http_requests_total` pas `requests`
- Unité à la fin : `_bytes`, `_seconds`, `_total`
- Utiliser des counters pour les cumuls : suffix `_total`

### 5. Tester dans l'UI Prometheus

Avant d'utiliser une requête dans Grafana ou une alerte :
1. Testez dans l'UI Prometheus (`http://localhost:9090`)
2. Vérifiez que les résultats sont cohérents
3. Testez avec différentes périodes de temps

### 6. Documenter les requêtes complexes

Ajoutez des commentaires dans Grafana :

```promql
# Calcul du taux d'erreur 5xx sur 5 minutes
# Utilisé pour l'alerte ErrorRateHigh
sum(rate(http_requests_total{status=~"5.."}[5m])) by (service)
```

## Débogage des requêtes PromQL

### Problème : Requête ne retourne rien

**Vérifications** :

1. **La métrique existe-t-elle ?**

```promql
{__name__=~".*request.*"}  # Chercher toutes les métriques contenant "request"
```

2. **Les labels sont-ils corrects ?**

```promql
# D'abord sans filtre
http_requests_total

# Puis avec filtre
http_requests_total{method="GET"}
```

3. **Y a-t-il des données récentes ?**

Vérifiez dans l'UI que la métrique a des valeurs récentes.

### Problème : Résultats incohérents

**Causes possibles** :

1. **Mauvaise fonction pour le type de métrique**

```promql
# ❌ rate() sur un gauge
rate(container_memory_usage_bytes[5m])  # Incorrect !

# ✅ avg_over_time() pour un gauge
avg_over_time(container_memory_usage_bytes[5m])
```

2. **Durée trop courte**

```promql
# Peut être trop volatile
rate(http_requests_total[30s])

# Plus stable
rate(http_requests_total[5m])
```

### Problème : Requête lente

**Solutions** :

1. **Réduire la cardinalité**

```promql
# ❌ Lent : toutes les séries
rate(http_requests_total[5m])

# ✅ Plus rapide : agrégé
sum(rate(http_requests_total[5m])) by (job)
```

2. **Utiliser des recording rules** (voir section 12.7)

3. **Limiter la période de temps**

Dans Grafana, ne chargez pas 30 jours de données si vous n'en avez besoin que de 24h.

## Exemples de requêtes pour des scénarios réels

### Surveillance d'une application web

```promql
# Requêtes par seconde
sum(rate(http_requests_total[5m]))

# Taux d'erreur
sum(rate(http_requests_total{status=~"5.."}[5m])) /
sum(rate(http_requests_total[5m])) * 100

# Latence p95
histogram_quantile(0.95,
  sum(rate(http_request_duration_seconds_bucket[5m])) by (le))

# Nombre d'instances actives
count(up{job="my-app"} == 1)
```

### Surveillance d'une base de données

```promql
# Connexions actives
pg_stat_activity_count

# Requêtes lentes (> 1s)
pg_stat_activity_max_tx_duration > 1

# Taille de la base
pg_database_size_bytes / 1024 / 1024 / 1024  # En GB

# Taux de hit du cache
rate(pg_stat_database_blks_hit[5m]) /
(rate(pg_stat_database_blks_hit[5m]) +
 rate(pg_stat_database_blks_read[5m])) * 100
```

### Surveillance système

```promql
# Load average
node_load1

# Utilisation CPU (100 - idle)
100 - (avg(rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100)

# Mémoire disponible en %
(node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes) * 100

# I/O disque (lectures + écritures en MB/s)
(rate(node_disk_read_bytes_total[5m]) +
 rate(node_disk_written_bytes_total[5m])) / 1024 / 1024
```

## Résumé

Dans cette section, vous avez appris :

✅ **Les bases de PromQL** : Métriques, séries temporelles, labels
✅ **Les types de métriques** : Counter, Gauge, Histogram, Summary
✅ **Les sélecteurs** : Comment filtrer et sélectionner des séries
✅ **Les opérateurs** : Arithmétiques, comparaison, logiques
✅ **Les fonctions essentielles** : rate(), sum(), avg(), topk(), etc.
✅ **Les agrégations** : Grouper et calculer sur plusieurs séries
✅ **Les requêtes pratiques** : Exemples pour CPU, mémoire, réseau, HTTP
✅ **Les bonnes pratiques** : Comment écrire des requêtes efficaces
✅ **Le débogage** : Comment résoudre les problèmes courants

**Points clés à retenir** :

1. **rate()** pour les counters, pas de valeur brute
2. Utilisez **by()** pour agréger intelligemment
3. Choisissez la **bonne durée** de range (généralement 5m)
4. Testez vos requêtes dans l'UI Prometheus avant de les utiliser
5. Commencez simple, puis complexifiez progressivement

PromQL est un langage puissant mais qui demande de la pratique. N'hésitez pas à expérimenter dans l'interface Prometheus !

Dans la prochaine section (12.6), nous explorerons les **exporters essentiels** qui fournissent toutes ces métriques que nous venons d'apprendre à interroger.

---

**Conseil d'apprentissage** : Créez un fichier avec vos requêtes PromQL favorites et documentez-les. C'est votre "cheat sheet" personnelle que vous enrichirez au fil du temps !

⏭️ [Exporters essentiels (node, kube-state, cadvisor)](/12-monitoring-avec-prometheus/06-exporters-essentiels.md)
