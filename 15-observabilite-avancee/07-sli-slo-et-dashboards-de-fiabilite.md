🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 15.7 SLI/SLO et Dashboards de Fiabilité

## Introduction

Vous avez maintenant une observabilité complète de votre cluster MicroK8s :
- ✅ Métriques avec Prometheus
- ✅ Logs avec Loki
- ✅ Traces avec Jaeger
- ✅ Synthetic monitoring avec Blackbox

Mais une question reste : **Comment mesurer si votre service est "bon" ?**

Les **SLI** et **SLO** répondent à cette question en définissant des **objectifs mesurables** pour la fiabilité de vos services.

### Le Problème Sans SLI/SLO

**Conversation typique** :

```
Manager : "Le site est-il fiable ?"
DevOps : "Euh... on a 99% d'uptime ?"
Manager : "C'est bien ou pas ?"
DevOps : "Ça dépend... de quoi ?"
Manager : "Est-ce qu'on peut déployer cette nouvelle fonctionnalité ?"
DevOps : "Je ne sais pas si on peut se permettre plus de risques..."
```

**Sans objectifs clairs** :
- ❌ Pas de référentiel commun
- ❌ Discussions subjectives ("c'est lent" vs "non, c'est rapide")
- ❌ Impossible de prendre des décisions basées sur les données
- ❌ Burn-out des équipes (tout est "urgent")

**Avec SLI/SLO** :
- ✅ Objectifs mesurables et partagés
- ✅ Discussions basées sur les faits
- ✅ Décisions éclairées (déployer ou pas ?)
- ✅ Équilibre entre innovation et stabilité

## Les Trois Concepts : SLI, SLO, SLA

### SLI (Service Level Indicator)

**Définition** : Un **SLI** est une **métrique** qui mesure un aspect spécifique de votre service.

**En français simple** : "Comment je mesure si mon service fonctionne bien ?"

**Exemples** :
- Pourcentage de requêtes réussies
- Temps de réponse (latence)
- Disponibilité du service
- Débit (throughput)
- Durabilité des données

**Caractéristiques d'un bon SLI** :
- ✅ Mesurable objectivement (métrique)
- ✅ Important pour l'utilisateur
- ✅ Directement contrôlable par vous

**Exemple concret** :

```
SLI: "Pourcentage de requêtes HTTP qui retournent un status code 2xx en moins de 1 seconde"

Mesure sur la dernière heure:
- Total requêtes: 10 000
- Requêtes réussies (<1s): 9 950
- SLI = 9950/10000 = 99.5%
```

### SLO (Service Level Objective)

**Définition** : Un **SLO** est un **objectif** pour votre SLI.

**En français simple** : "Quel niveau je veux atteindre pour cette métrique ?"

**Format typique** : `[SLI] [comparateur] [seuil]` sur `[période]`

**Exemples** :
- 99% des requêtes doivent réussir (sur 30 jours)
- 95% des requêtes doivent être servies en moins de 500ms (sur 7 jours)
- Le service doit être disponible 99.9% du temps (sur 1 mois)

**Exemple concret** :

```
SLO: "99.5% des requêtes HTTP doivent réussir en moins de 1 seconde sur une fenêtre glissante de 30 jours"

État actuel:
- SLI actuel: 99.7%
- SLO cible: 99.5%
- Marge: +0.2%
✅ SLO respecté !
```

### SLA (Service Level Agreement)

**Définition** : Un **SLA** est un **contrat** (souvent légal) qui définit les conséquences si un SLO n'est pas atteint.

**En français simple** : "Qu'est-ce qui se passe si je rate mon objectif ?"

**Exemples** :
- Remboursement de 10% si uptime < 99.9%
- Crédit de 25% si latence > 500ms pour 5% des requêtes
- Pénalités contractuelles

**Différence clé** :

```
SLI: "Comment je mesure"        → Métrique technique
SLO: "Quel niveau je vise"      → Objectif interne
SLA: "Engagement contractuel"   → Promesse client avec conséquences
```

**Relation** :

```
        SLA (99.0%)  ← Promesse client (contrat)
           ↑
    SLO interne (99.5%)  ← Objectif interne (plus strict)
           ↑
        SLI (99.7%)  ← Performance réelle (mesure)
```

**Pour un lab MicroK8s** : Vous utilisez principalement **SLI** et **SLO**. Les SLA sont pour les environnements production avec clients externes.

## Pourquoi les SLI/SLO Sont Importants

### 1. Langage Commun

**Sans SLO** :
```
Dev: "Le service est rapide"
Ops: "Non, il est lent"
Product: "Les utilisateurs se plaignent"
```

**Avec SLO** :
```
Tous: "Notre SLO est 95% des requêtes < 500ms"
Monitoring: "On est à 92%"
Tous: "On a un problème, qu'est-ce qu'on fait ?"
```

### 2. Priorisation

**Sans SLO** :
- Toutes les alertes semblent critiques
- Impossible de prioriser
- Burn-out de l'équipe

**Avec SLO** :
```
Alerte 1: SLO violation (95% → 92%)  → PRIORITÉ 1
Alerte 2: CPU à 60% (seuil: 80%)     → Pas urgent
Alerte 3: 1 pod down (sur 10)        → Impact sur SLO ? Non → Pas urgent
```

### 3. Error Budget (Budget d'Erreur)

**Concept révolutionnaire** introduit par Google SRE.

**Calcul** :

```
SLO: 99.5% de disponibilité
Donc acceptable: 100% - 99.5% = 0.5% d'indisponibilité

Sur 30 jours:
30 jours × 24h × 60min × 60s = 2,592,000 secondes
Budget d'erreur = 2,592,000 × 0.5% = 12,960 secondes
                = 216 minutes
                = 3.6 heures

Vous pouvez être down 3.6 heures par mois sans violer le SLO !
```

**Utilisation du budget** :

```
Début du mois: Budget = 3.6h
Incident 1 (30 min): Budget restant = 3.1h
Déploiement risqué (10 min downtime): Budget = 3.0h
Incident 2 (1h): Budget = 2.0h

Question: "Peut-on déployer cette feature risquée ?"
Réponse: "Oui, on a encore 2h de budget. Mais attention !"
```

**Si le budget est épuisé** :
- ❌ STOP les déploiements (sauf critiques)
- ✅ FOCUS sur la stabilité
- ✅ Post-mortem des incidents
- ✅ Correction des problèmes

**Avantage** : Équilibre entre **innovation** (déployer) et **stabilité** (ne pas casser).

### 4. Décisions Basées sur les Données

**Question** : "Doit-on ajouter plus de réplicas ?"

**Sans SLO** :
```
"Je pense que oui... le service semble lent parfois"
```

**Avec SLO** :
```
SLO latence: 95% < 500ms
SLI actuel: 93% < 500ms
❌ SLO violé → OUI, il faut scaler
```

**Question** : "Doit-on optimiser ce code qui prend 100ms ?"

**Sans SLO** :
```
"100ms c'est lent, optimisons !"
Temps perdu: 2 semaines
Impact utilisateur: Négligeable
```

**Avec SLO** :
```
SLO latence: 95% < 500ms
Ce code représente 0.1% des requêtes
Impact sur SLO: 0.1% × 100ms = négligeable
Décision: PAS PRIORITAIRE, focus ailleurs
```

## Choisir les Bons SLI

### Les Quatre Golden Signals

Google SRE recommande de surveiller 4 métriques clés :

#### 1. Latence (Latency)

**Définition** : Temps pour traiter une requête.

**Métriques** :
- Latence moyenne
- P50 (médiane)
- P95 (95ème percentile)
- P99 (99ème percentile)

**Pourquoi les percentiles ?**

```
100 requêtes:
- 95 requêtes: 100ms
- 4 requêtes: 200ms
- 1 requête: 5000ms (timeout)

Moyenne: 147ms (masque le problème)
P95: 200ms (capte les vraies expériences)
P99: 5000ms (révèle les pires cas)
```

**SLI exemple** :
```
"P95 de la latence des requêtes HTTP"
```

**SLO exemple** :
```
"P95 < 500ms sur 30 jours"
```

#### 2. Trafic (Traffic)

**Définition** : Demande sur votre système.

**Métriques** :
- Requêtes par seconde
- Utilisateurs actifs
- Bande passante

**SLI exemple** :
```
"Nombre de requêtes par seconde"
```

**SLO exemple** :
```
"Le système doit supporter au moins 1000 req/s"
```

**Note** : Le trafic est souvent un input, pas un SLO direct.

#### 3. Erreurs (Errors)

**Définition** : Taux de requêtes qui échouent.

**Types d'erreurs** :
- Explicites : Status 5xx, exceptions
- Implicites : Status 2xx mais données incorrectes
- Partielle : Timeouts, dégradation

**SLI exemple** :
```
"Pourcentage de requêtes qui retournent 5xx ou timeout"
```

**SLO exemple** :
```
"Taux d'erreur < 0.1% sur 30 jours"
ou
"99.9% des requêtes doivent réussir sur 30 jours"
```

#### 4. Saturation (Saturation)

**Définition** : "Plénitude" du système.

**Métriques** :
- CPU
- Mémoire
- Espace disque
- Connexions DB
- Queue depth

**SLI exemple** :
```
"Pourcentage d'utilisation CPU moyenne"
```

**SLO exemple** :
```
"CPU < 80% sur 95% du temps (30 jours)"
```

### Choisir VOS SLI

**Étapes** :

1. **Identifiez ce qui compte pour l'utilisateur**
   - Qu'est-ce qui rend votre service "bon" ?
   - Qu'est-ce qui frustre vos utilisateurs ?

2. **Rendez-le mesurable**
   - Quelle métrique capture ça ?
   - Pouvez-vous la mesurer avec vos outils ?

3. **Gardez ça simple**
   - 3-5 SLI maximum par service
   - Trop de SLI = dilution de l'attention

**Exemples par type de service** :

**API REST** :
- Disponibilité : % de requêtes réussies
- Latence : P95 du temps de réponse
- Erreurs : % de status 5xx

**Application Web** :
- Disponibilité : % de pages chargées avec succès
- Latence : P95 du temps de chargement
- Completeness : % de pages avec toutes les ressources chargées

**Service de Base de Données** :
- Disponibilité : % de connexions réussies
- Latence : P95 du temps de requête
- Durabilité : % de données sans corruption

**Service de Traitement Batch** :
- Débit : Jobs traités par heure
- Durée : P95 du temps de traitement
- Success rate : % de jobs réussis

## Implémenter les SLI dans Prometheus

### Exemple 1 : SLI de Disponibilité

**Objectif** : Mesurer le pourcentage de requêtes HTTP réussies.

**Métriques Prometheus nécessaires** :

```promql
# Total des requêtes
http_requests_total{job="myapp"}

# Requêtes réussies (status 2xx-3xx)
http_requests_total{job="myapp", status=~"[23].."}
```

**Calcul du SLI** :

```promql
# SLI = requêtes réussies / total requêtes
sum(rate(http_requests_total{job="myapp", status=~"[23].."}[5m]))
/
sum(rate(http_requests_total{job="myapp"}[5m]))
```

**Résultat** : Valeur entre 0 et 1 (ex: 0.995 = 99.5%)

**Pour l'afficher en pourcentage** :

```promql
(
  sum(rate(http_requests_total{job="myapp", status=~"[23].."}[5m]))
  /
  sum(rate(http_requests_total{job="myapp"}[5m]))
) * 100
```

### Exemple 2 : SLI de Latence

**Objectif** : P95 du temps de réponse.

**Métrique Prometheus** (histogram) :

```promql
http_request_duration_seconds_bucket{job="myapp"}
```

**Calcul du SLI** :

```promql
# P95 de la latence
histogram_quantile(
  0.95,
  sum(rate(http_request_duration_seconds_bucket{job="myapp"}[5m])) by (le)
)
```

**Résultat** : Latence en secondes (ex: 0.234)

### Exemple 3 : SLI d'Erreurs

**Objectif** : Pourcentage de requêtes en erreur.

**Calcul du SLI** :

```promql
# SLI = requêtes en erreur / total requêtes
sum(rate(http_requests_total{job="myapp", status=~"5.."}[5m]))
/
sum(rate(http_requests_total{job="myapp"}[5m]))
* 100
```

**Résultat** : Pourcentage d'erreurs (ex: 0.5%)

### Exemple 4 : Synthetic Monitoring (Blackbox)

**Objectif** : Disponibilité du service vue de l'extérieur.

**Métrique Prometheus** :

```promql
probe_success{job="blackbox-http"}
```

**Calcul du SLI** :

```promql
# Disponibilité moyenne
avg_over_time(probe_success{instance="https://myapp.com"}[30d]) * 100
```

**Résultat** : Pourcentage de disponibilité (ex: 99.7%)

## Recording Rules pour les SLI

Pour éviter de recalculer les SLI à chaque fois, utilisez des **recording rules**.

**Fichier `prometheus-sli-rules.yaml`** :

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: prometheus-sli-rules
  namespace: monitoring
data:
  sli-rules.yml: |
    groups:
    - name: sli_availability
      interval: 30s
      rules:
      # SLI: Availability
      - record: sli:http_requests:availability:rate5m
        expr: |
          sum(rate(http_requests_total{status=~"[23].."}[5m])) by (job)
          /
          sum(rate(http_requests_total[5m])) by (job)

      # SLI sur 30 jours
      - record: sli:http_requests:availability:rate30d
        expr: |
          sum(rate(http_requests_total{status=~"[23].."}[30d])) by (job)
          /
          sum(rate(http_requests_total[30d])) by (job)

    - name: sli_latency
      interval: 30s
      rules:
      # SLI: P95 Latency
      - record: sli:http_request_duration:p95:rate5m
        expr: |
          histogram_quantile(
            0.95,
            sum(rate(http_request_duration_seconds_bucket[5m])) by (le, job)
          )

      # SLI: P99 Latency
      - record: sli:http_request_duration:p99:rate5m
        expr: |
          histogram_quantile(
            0.99,
            sum(rate(http_request_duration_seconds_bucket[5m])) by (le, job)
          )

    - name: sli_errors
      interval: 30s
      rules:
      # SLI: Error rate
      - record: sli:http_requests:error_rate:rate5m
        expr: |
          sum(rate(http_requests_total{status=~"5.."}[5m])) by (job)
          /
          sum(rate(http_requests_total[5m])) by (job)

    - name: sli_synthetic
      interval: 1m
      rules:
      # SLI: Synthetic availability
      - record: sli:probe:availability:rate30d
        expr: |
          avg_over_time(probe_success[30d])
```

**Appliquer** :

```bash
kubectl apply -f prometheus-sli-rules.yaml

# Ajouter à la config Prometheus
# rule_files:
#   - /etc/prometheus/rules/sli-rules.yml

# Recharger Prometheus
kubectl rollout restart statefulset prometheus -n monitoring
```

**Avantage** : Requêtes plus rapides, nommage cohérent.

## Définir les SLO

### Méthodologie

**1. Commencer Simple**

Ne définissez pas 20 SLO le premier jour. Commencez par 1-3 SLO critiques.

**2. Basé sur les Données Historiques**

```
Regardez vos 30 derniers jours:
- P95 latence: 234ms
- Availability: 99.7%
- Error rate: 0.2%

Ne mettez PAS un SLO de "P95 < 100ms" si vous êtes à 234ms !
Soyez réaliste.
```

**3. Pensez au Coût**

Plus le SLO est strict, plus c'est coûteux :

```
SLO 99%   = 7.2h downtime/mois   → Facile et peu coûteux
SLO 99.9% = 43 min downtime/mois → Coût moyen
SLO 99.99% = 4.3 min downtime/mois → Très coûteux !
```

**Pour un lab** : 95-99% est raisonnable.
**Pour production critique** : 99.5-99.9%.

**4. Itérez**

```
Mois 1: SLO 95% → Atteint facilement → Trop facile
Mois 2: SLO 98% → Atteint avec efforts → Bon équilibre
Mois 3: SLO 99% → Raté plusieurs fois → Trop dur

Retour à 98% → C'est votre SLO idéal
```

### Exemples de SLO Réalistes

**Service Web (Lab Personnel)** :

```yaml
SLO:
  - name: "Disponibilité"
    sli: "sli:http_requests:availability:rate30d"
    target: 0.95  # 95%
    window: 30d

  - name: "Latence P95"
    sli: "sli:http_request_duration:p95:rate5m"
    target: 1.0  # 1 seconde
    window: 30d

  - name: "Taux d'erreur"
    sli: "sli:http_requests:error_rate:rate5m"
    target: 0.01  # < 1%
    window: 30d
```

**API de Production** :

```yaml
SLO:
  - name: "Disponibilité"
    target: 0.999  # 99.9%
    window: 30d

  - name: "Latence P95"
    target: 0.5  # 500ms
    window: 30d

  - name: "Latence P99"
    target: 1.0  # 1 seconde
    window: 30d
```

## Alertes basées sur les SLO

### Approche Traditionnelle (Symptôme)

```yaml
# ❌ Alerte sur un symptôme
- alert: HighCPU
  expr: cpu_usage > 80
  for: 5m
```

**Problème** : Est-ce que 80% CPU impacte réellement l'utilisateur ?

### Approche SLO (Impact)

```yaml
# ✅ Alerte sur l'impact SLO
- alert: SLO_AvailabilityBreach
  expr: sli:http_requests:availability:rate5m < 0.95
  for: 5m
  labels:
    severity: warning
  annotations:
    summary: "SLO Availability breached"
    description: "Availability is {{ $value | humanizePercentage }}, below target of 95%"
```

**Avantage** : Vous êtes alerté seulement quand ça impacte vraiment les utilisateurs.

### Multi-Window Multi-Burn-Rate

**Concept avancé** de Google SRE.

**Problème** :
- Alerte immédiate sur violation → Trop de faux positifs
- Alerte après 1 heure → Trop tard

**Solution** : Alertes à plusieurs niveaux de sévérité.

```yaml
groups:
- name: slo_alerts
  rules:

  # CRITICAL: Burn rapide, fenêtre courte
  # Consomme le budget en 2 jours au rythme actuel
  - alert: SLO_ErrorBudget_CriticalBurn
    expr: |
      (
        sli:http_requests:availability:rate1h < (1 - 0.95)  # Target SLO
        and
        sli:http_requests:availability:rate5m < (1 - 0.95)
      )
    for: 2m
    labels:
      severity: critical
    annotations:
      summary: "Critical: SLO error budget burning fast"

  # WARNING: Burn moyen, fenêtre moyenne
  # Consomme le budget en 7 jours au rythme actuel
  - alert: SLO_ErrorBudget_HighBurn
    expr: |
      (
        sli:http_requests:availability:rate6h < (1 - 0.95)
        and
        sli:http_requests:availability:rate30m < (1 - 0.95)
      )
    for: 15m
    labels:
      severity: warning
    annotations:
      summary: "Warning: SLO error budget burning"

  # INFO: Burn lent mais constant
  - alert: SLO_ErrorBudget_SlowBurn
    expr: |
      sli:http_requests:availability:rate30d < (1 - 0.95)
    for: 1h
    labels:
      severity: info
    annotations:
      summary: "Info: SLO error budget exhausted"
```

**Logique** :
- **Critical** : Problème immédiat et sévère → Action maintenant
- **Warning** : Problème qui s'installe → Action bientôt
- **Info** : Budget épuisé → Post-mortem, pas d'urgence

## Dashboards de Fiabilité

### Dashboard SLO Complet

Créez un dashboard Grafana dédié aux SLO.

**Structure recommandée** :

```
┌────────────────────────────────────────────────┐
│  Service: MyApp                  [30d ▼]       │
├────────────────────────────────────────────────┤
│                                                │
│  📊 SLO Summary                                │
│  ┌──────────────────────────────────────────┐  │
│  │ Disponibilité  99.7% ✅ Target: 95%      │  │
│  │ Latence P95    234ms ✅ Target: 500ms    │  │
│  │ Error Rate     0.2%  ✅ Target: 1%       │  │
│  └──────────────────────────────────────────┘  │
│                                                │
│  💰 Error Budget                               │
│  ┌──────────────────────────────────────────┐  │
│  │ Budget restant: 2.1h / 3.6h (58%) ✅     │  │
│  │ [████████████░░░░░░░]                    │  │
│  └──────────────────────────────────────────┘  │
│                                                │
│  📈 SLI Trends (30 days)                       │
│  ┌──────────────────────────────────────────┐  │
│  │  [Graphe de disponibilité]               │  │
│  │  [Graphe de latence]                     │  │
│  │  [Graphe d'erreurs]                      │  │
│  └──────────────────────────────────────────┘  │
│                                                │
│  🚨 Recent Incidents                           │
│  ┌──────────────────────────────────────────┐  │
│  │  Oct 20: 30min downtime (database)       │  │
│  │  Oct 15: 10min high latency (deploy)     │  │
│  └──────────────────────────────────────────┘  │
└────────────────────────────────────────────────┘
```

### Panneaux Grafana

#### 1. SLO Gauge (Jauge)

**Type** : Gauge

**Requête** :

```promql
sli:http_requests:availability:rate30d * 100
```

**Configuration** :
- Min : 0
- Max : 100
- Thresholds :
  - Red : 0-95
  - Yellow : 95-98
  - Green : 98-100
- Unité : percent (0-100)

**Display** :
- Show : Current value
- Title : "Disponibilité (30d)"

#### 2. SLO vs Target (Stat avec spark line)

**Type** : Stat

**Requête** :

```promql
sli:http_requests:availability:rate30d
```

**Second query** (Target) :

```promql
0.95
```

**Configuration** :
- Show : Both (value + graph)
- Thresholds :
  - Base (Red) : 0
  - Green : 0.95 (SLO target)
- Value :  Display : Percent (0.0-1.0)

#### 3. Error Budget Gauge

**Calcul du budget restant** :

```promql
# Budget total (en secondes sur 30 jours)
(1 - 0.95) * 30 * 24 * 60 * 60  # = 129,600 secondes

# Budget consommé (downtime constaté)
(1 - sli:http_requests:availability:rate30d) * 30 * 24 * 60 * 60

# Budget restant
(1 - 0.95) * 30 * 24 * 60 * 60
-
(1 - sli:http_requests:availability:rate30d) * 30 * 24 * 60 * 60
```

**Simplifié** :

```promql
# Budget restant en pourcentage
(sli:http_requests:availability:rate30d - 0.95) / (1 - 0.95) * 100
```

**Type** : Gauge

**Configuration** :
- Min : 0
- Max : 100
- Thresholds :
  - Red : 0-10 (Presque épuisé)
  - Yellow : 10-50
  - Green : 50-100
- Unité : percent

#### 4. SLI Timeline (30 jours)

**Type** : Time series

**Requêtes** :

```promql
# SLI actuel
sli:http_requests:availability:rate30d * 100

# Target SLO (ligne de référence)
95
```

**Configuration** :
- Legend : {{ job }} et "SLO Target"
- Y-axis : Percent (90-100)
- Thresholds :
  - Red : < 95
  - Green : >= 95

#### 5. Latence Multi-Percentile

**Type** : Time series

**Requêtes** :

```promql
# P50
sli:http_request_duration:p50:rate5m * 1000

# P95
sli:http_request_duration:p95:rate5m * 1000

# P99
sli:http_request_duration:p99:rate5m * 1000

# Target (ligne)
500
```

**Configuration** :
- Unité : ms
- Legend : P50, P95, P99, Target

#### 6. Error Budget Burn Rate

**Type** : Time series

**Requête** :

```promql
# Taux de consommation du budget
# 1.0 = consommation normale
# > 1.0 = consommation rapide (problème)
# < 1.0 = meilleur que prévu

(1 - sli:http_requests:availability:rate1h) / (1 - 0.95)
```

**Configuration** :
- Thresholds :
  - Red : > 10 (burn 10x plus rapide)
  - Yellow : > 5
  - Green : <= 1

#### 7. Table de Synthèse

**Type** : Table

**Requêtes** :

```promql
# Query A: Availability
sli:http_requests:availability:rate30d

# Query B: Latency P95
sli:http_request_duration:p95:rate5m

# Query C: Error Rate
sli:http_requests:error_rate:rate5m
```

**Transformations** :
- Join by job
- Organize fields

**Colonnes** :
- Service (job)
- Availability %
- Latency P95 (ms)
- Error Rate %
- Status (✅/⚠️/❌)

### Variables de Dashboard

```
$service : label_values(sli:http_requests:availability:rate30d, job)
$window : 1h, 6h, 24h, 7d, 30d
$slo_target : 0.90, 0.95, 0.99, 0.999
```

Utilisez dans les requêtes :

```promql
sli:http_requests:availability:rate30d{job="$service"}
```

## Calcul de l'Error Budget

### Formule de Base

```
Error Budget (temps) = Total Time × (1 - SLO)
```

**Exemple : SLO 99% sur 30 jours**

```
Total Time = 30 jours × 24h × 60min = 43,200 minutes
Error Budget = 43,200 × (1 - 0.99) = 432 minutes = 7.2 heures
```

Vous pouvez avoir **7.2 heures d'indisponibilité** par mois.

### Tracking du Budget

**Recording rule** :

```yaml
- name: error_budget
  interval: 1m
  rules:
  # Budget total (en secondes)
  - record: slo:error_budget:total_seconds
    expr: |
      (1 - 0.95) * 30 * 24 * 60 * 60  # 129,600 secondes pour SLO 95%

  # Budget consommé (downtime en secondes)
  - record: slo:error_budget:consumed_seconds
    expr: |
      (1 - sli:http_requests:availability:rate30d) * 30 * 24 * 60 * 60

  # Budget restant (en secondes)
  - record: slo:error_budget:remaining_seconds
    expr: |
      slo:error_budget:total_seconds - slo:error_budget:consumed_seconds

  # Budget restant (en pourcentage)
  - record: slo:error_budget:remaining_ratio
    expr: |
      slo:error_budget:remaining_seconds / slo:error_budget:total_seconds
```

**Affichage dans Grafana** :

```promql
# Heures restantes
slo:error_budget:remaining_seconds / 3600

# Pourcentage restant
slo:error_budget:remaining_ratio * 100
```

### Politique d'Error Budget

**Document à créer** : "Error Budget Policy"

**Exemple** :

```markdown
# Error Budget Policy - MyApp Service

## SLO
- Disponibilité: 95% sur 30 jours
- Error Budget: 3.6 heures/mois

## Politiques

### Budget > 50% ✅
- ✅ Déploiements autorisés sans restrictions
- ✅ Features risquées acceptées
- ✅ Expérimentations encouragées

### Budget 25-50% ⚠️
- ⚠️ Déploiements autorisés avec review
- ⚠️ Features risquées à évaluer
- ⚠️ Tests approfondis requis

### Budget < 25% 🚨
- ❌ FREEZE des déploiements non-critiques
- ✅ Seulement hotfixes de sécurité/stabilité
- 📋 Post-mortem obligatoire
- 🔧 Focus stabilité jusqu'à récupération

### Budget épuisé 🔥
- 🚫 ARRÊT TOTAL des déploiements
- 🏥 Mode "incident" activé
- 👥 War room quotidien
- 📊 Analyse root cause
- 🛠️ Correction des problèmes systémiques
```

## Rapport SLO Mensuel

### Template de Rapport

```markdown
# SLO Report - October 2025

## Executive Summary

- ✅ SLO Availability: **99.7%** (Target: 95%)
- ✅ SLO Latency P95: **234ms** (Target: 500ms)
- ⚠️ SLO Error Rate: **1.2%** (Target: 1%)

## Error Budget

- Budget total: 3.6 heures
- Budget consommé: 1.5 heures (42%)
- Budget restant: 2.1 heures (58%)

**Status: HEALTHY** ✅

## Incidents

### Oct 20 - Database Outage (30 min)
- **Impact**: 30 minutes downtime
- **Root Cause**: Primary DB node crash
- **Resolution**: Automatic failover to replica
- **Action Items**:
  - [ ] Implement health checks
  - [ ] Tune failover timeout

### Oct 15 - Deployment Rollout Issues (10 min)
- **Impact**: 10 minutes high latency
- **Root Cause**: Insufficient resources during rolling update
- **Resolution**: Increased replica count
- **Action Items**:
  - [ ] Update deployment strategy
  - [ ] Pre-scale before deploys

## Trends

- Availability improving (+0.2% vs last month)
- Latency stable
- Error rate slightly increased (investigation needed)

## Next Month

- Target: Maintain current SLO levels
- Focus: Reduce error rate below 1%
- Project: Implement caching to improve latency
```

### Automatisation du Rapport

**Script Python avec Prometheus API** :

```python
import requests
from datetime import datetime, timedelta

PROMETHEUS_URL = "http://prometheus:9090"

def query_prometheus(query):
    response = requests.get(f"{PROMETHEUS_URL}/api/v1/query", params={'query': query})
    return response.json()['data']['result'][0]['value'][1]

def generate_slo_report():
    # Query SLI metrics
    availability = float(query_prometheus('sli:http_requests:availability:rate30d'))
    latency_p95 = float(query_prometheus('sli:http_request_duration:p95:rate5m'))
    error_rate = float(query_prometheus('sli:http_requests:error_rate:rate5m'))

    # Error budget
    budget_total = 3.6  # hours
    budget_consumed = (1 - availability) * 30 * 24  # hours
    budget_remaining = budget_total - budget_consumed

    # Generate report
    report = f"""
# SLO Report - {datetime.now().strftime('%B %Y')}

## Metrics

- Availability: {availability * 100:.2f}% (Target: 95%)
- Latency P95: {latency_p95 * 1000:.0f}ms (Target: 500ms)
- Error Rate: {error_rate * 100:.2f}% (Target: 1%)

## Error Budget

- Consumed: {budget_consumed:.2f}h / {budget_total}h ({budget_consumed/budget_total*100:.0f}%)
- Remaining: {budget_remaining:.2f}h

## Status

{"✅ HEALTHY" if budget_remaining > budget_total * 0.5 else "⚠️ AT RISK" if budget_remaining > 0 else "🚨 EXHAUSTED"}
    """

    return report

if __name__ == "__main__":
    print(generate_slo_report())
```

## Bonnes Pratiques

### 1. Commencez Simple

```
❌ Premier jour: 20 SLO sur 10 services
✅ Premier mois: 3 SLO sur 1 service critique
```

### 2. Basé sur l'Expérience Utilisateur

```
❌ SLO: "CPU < 80%"  (métrique interne)
✅ SLO: "P95 latency < 500ms"  (expérience utilisateur)
```

### 3. Mesurez Ce Que Vous Pouvez Contrôler

```
❌ SLO sur API externe dont vous dépendez
✅ SLO sur votre propre service (avec fallbacks si API externe down)
```

### 4. Documentez les SLO

Créez un fichier `SLO.md` dans votre repo :

```markdown
# Service Level Objectives - MyApp

## SLO 1: Availability
- **Indicator**: % of HTTP requests returning 2xx/3xx
- **Target**: 95% over 30 days
- **Measurement**: `sli:http_requests:availability:rate30d`
- **Rationale**: Users must be able to access the service
- **Owner**: Team Backend

## SLO 2: Latency
- **Indicator**: P95 of HTTP request duration
- **Target**: < 500ms over 30 days
- **Measurement**: `sli:http_request_duration:p95:rate5m`
- **Rationale**: Users expect fast responses
- **Owner**: Team Backend
```

### 5. Revue Régulière

```
Mensuel:
- Revue des SLO atteints/ratés
- Ajustement des targets si nécessaire
- Post-mortem des incidents

Trimestriel:
- Réévaluation des SLI (sont-ils toujours pertinents ?)
- Ajustement de l'error budget policy
```

### 6. Culture, Pas Seulement des Métriques

Les SLO ne sont pas juste des chiffres :

```
✅ Utilisez les SLO pour prendre des décisions
✅ Partagez les dashboards SLO avec toute l'équipe
✅ Célébrez quand les SLO sont atteints
✅ Apprenez des violations de SLO (blameless post-mortems)
```

### 7. Évitez le Perfectionnisme

```
❌ SLO 99.999% "five nines"
   → Coût exorbitant
   → Équipe en burn-out
   → Pas de place pour l'innovation

✅ SLO 99% "two nines"
   → Coût raisonnable
   → Équipe équilibrée
   → Budget pour expérimenter
```

Google dit : "L'objectif n'est pas 100%, c'est l'équilibre."

## Cas Pratique : Lab Personnel

Voici une configuration réaliste pour un lab MicroK8s :

### SLO Définis

```yaml
services:
  - name: grafana
    slos:
      - name: availability
        target: 0.95  # 95%
        window: 30d

      - name: latency_p95
        target: 2.0   # 2 secondes (lab, pas critique)
        window: 30d

  - name: prometheus
    slos:
      - name: availability
        target: 0.95
        window: 30d
```

### Recording Rules

```yaml
groups:
- name: lab_sli
  interval: 1m
  rules:
  - record: sli:blackbox:availability:rate30d
    expr: |
      avg_over_time(probe_success{job="blackbox-http"}[30d])

  - record: sli:blackbox:latency:rate5m
    expr: |
      probe_duration_seconds{job="blackbox-http"}
```

### Dashboard Simple

Trois panneaux :
1. **Uptime** (Gauge) : `sli:blackbox:availability:rate30d * 100`
2. **Latency** (Graph) : `sli:blackbox:latency:rate5m * 1000`
3. **Error Budget** : `(sli:blackbox:availability:rate30d - 0.95) / 0.05 * 100`

## Ce Qu'il Faut Retenir

🎯 **SLI/SLO = Langage commun** :
- SLI : Comment je mesure (métrique)
- SLO : Quel niveau je vise (objectif)
- SLA : Engagement contractuel (promesse)

📊 **Les 4 Golden Signals** :
- Latency (temps de réponse)
- Traffic (charge)
- Errors (taux d'échec)
- Saturation (utilisation ressources)

💰 **Error Budget** :
- Budget = Temps d'indisponibilité acceptable
- Équilibre innovation vs stabilité
- Budget épuisé = Focus stabilité

🎨 **Dashboard de Fiabilité** :
- SLO summary (gauges)
- Trends (30 jours)
- Error budget (%)
- Récents incidents

🚨 **Alerting basé sur SLO** :
- Multi-window multi-burn-rate
- Critical / Warning / Info
- Impact utilisateur, pas symptômes internes

✅ **Bonnes pratiques** :
- Commencer simple (1-3 SLO)
- Basé sur l'expérience utilisateur
- Documentez et partagez
- Revue mensuelle
- Culture, pas juste des chiffres

📈 **Pour un lab MicroK8s** :
- SLO 95-98% est raisonnable
- Utilisez Blackbox pour synthetic monitoring
- Dashboard simple mais informatif
- Apprenez les concepts pour le futur

---

## Conclusion

Les **SLI/SLO** transforment votre monitoring de "réactif" (on répond aux alertes) à "proactif" (on mesure si on atteint nos objectifs).

**Avant SLI/SLO** :
```
"Le service marche ?"
"Euh... je crois ?"
*Check plein de dashboards*
"Je pense que oui ?"
```

**Avec SLI/SLO** :
```
"Le service marche ?"
"Oui, SLO à 99.7%, budget restant: 58%"
"Peut-on déployer ?"
"Oui, on a de la marge"
```

**Votre stack d'observabilité est maintenant complète** :

1. ✅ **Métriques** (Prometheus) → Quoi et combien
2. ✅ **Logs** (Loki) → Pourquoi et détails
3. ✅ **Traces** (Jaeger) → Où et comment (parcours)
4. ✅ **Synthetic** (Blackbox) → Vue utilisateur
5. ✅ **SLI/SLO** → Objectifs et fiabilité

Vous avez maintenant les outils et les pratiques pour gérer un cluster Kubernetes avec un **niveau d'observabilité professionnel**, même dans un lab personnel ! 🎉🚀

**Prochains chapitres** vous apprendront d'autres aspects avancés de Kubernetes, mais vous avez maintenant une base solide en observabilité qui vous servira tout au long de votre parcours Kubernetes.

Félicitations pour avoir complété cette section ! 🎊

⏭️ [Sécurité Kubernetes](/16-securite-kubernetes/README.md)
