🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 14.4 Routing et Grouping

## Introduction

Le **routing** (routage) et le **grouping** (regroupement) sont deux mécanismes fondamentaux d'Alertmanager qui déterminent **comment** et **quand** vos alertes sont envoyées. Bien maîtrisés, ils transforment un flux chaotique d'alertes individuelles en notifications organisées et pertinentes.

Imaginez un système sans routing ni grouping : vous recevriez une notification séparée pour chaque alerte, potentiellement des centaines par jour, sans distinction de priorité. Avec une bonne configuration, vous recevez à la place des notifications groupées, intelligemment routées vers les bonnes personnes, au bon moment.

## Le Routing : Diriger les Alertes

Le routing est le système qui décide **où envoyer** chaque alerte en fonction de ses caractéristiques (labels).

### Analogie : Le Tri Postal

Pensez au routing comme un centre de tri postal :
- Chaque alerte est une lettre
- Les labels sont l'adresse
- Les routes sont les règles de tri
- Les receivers sont les destinations finales

Une lettre adressée à Paris va à Paris, une lettre urgente prend un canal prioritaire - c'est exactement ce que fait le routing avec vos alertes.

### Concepts Fondamentaux

**Route** : Une règle qui dit "si l'alerte correspond à ces critères, envoie-la à ce receiver"

**Match** : Les critères pour qu'une alerte corresponde à une route

**Receiver** : La destination finale (email, Slack, PagerDuty, etc.)

**Default Route** : La route par défaut utilisée si aucune autre route ne correspond

**Continue** : Permet à une alerte d'être traitée par plusieurs routes

## Structure d'une Configuration de Routing

### Configuration Minimale

Voici la structure la plus simple possible :

```yaml
route:
  receiver: 'default-receiver'

receivers:
  - name: 'default-receiver'
    email_configs:
      - to: 'admin@example.com'
```

**Explication** :
- Toutes les alertes vont vers `default-receiver`
- `default-receiver` envoie un email à admin@example.com
- Aucun tri, aucune logique

### Configuration avec Routes Multiples

```yaml
route:
  receiver: 'default-receiver'
  routes:
    - match:
        severity: 'critical'
      receiver: 'pagerduty'

    - match:
        severity: 'warning'
      receiver: 'slack'

receivers:
  - name: 'default-receiver'
    email_configs:
      - to: 'team@example.com'

  - name: 'pagerduty'
    pagerduty_configs:
      - service_key: 'xxx'

  - name: 'slack'
    slack_configs:
      - channel: '#alerts'
```

**Explication** :
- Route 1 : Les alertes `critical` vont vers PagerDuty
- Route 2 : Les alertes `warning` vont vers Slack
- Autres : Vont vers l'email par défaut

### L'Arbre de Routage

Le routing fonctionne comme un **arbre de décision**. Alertmanager évalue les routes dans l'ordre et utilise la première qui correspond.

```
┌─────────────────┐
│  Alerte arrive  │
└────────┬────────┘
         │
         ▼
┌────────────────────┐
│ Route par défaut   │ ──► Receiver par défaut
└────────┬───────────┘
         │
         ├──► Route 1: severity=critical ? ──► PagerDuty
         │
         ├──► Route 2: severity=warning ?  ──► Slack
         │
         └──► Route 3: team=frontend ?     ──► Email frontend
```

## Match : Filtrer les Alertes

Le `match` détermine si une alerte correspond à une route. Il y a deux types de match.

### Match Exact

Compare la valeur d'un label avec une chaîne exacte :

```yaml
match:
  severity: 'critical'
  environment: 'production'
```

**Se déclenche si** :
- Le label `severity` vaut exactement `critical`
- **ET** le label `environment` vaut exactement `production`

**Ne se déclenche PAS si** :
- `severity: 'Critical'` (casse différente)
- `severity: 'critical-high'` (valeur différente)
- `environment: 'prod'` (valeur différente)

### Match avec Regex (match_re)

Compare la valeur d'un label avec une expression régulière :

```yaml
match_re:
  severity: '(critical|warning)'
  service: 'api.*'
```

**Se déclenche si** :
- Le label `severity` vaut `critical` OU `warning`
- **ET** le label `service` commence par `api` (api-v1, api-backend, etc.)

### Exemples de Patterns Regex Utiles

**Commence par** :
```yaml
match_re:
  service: '^frontend.*'
# Match: frontend, frontend-api, frontend-web
```

**Termine par** :
```yaml
match_re:
  pod: '.*-production$'
# Match: app-production, web-production
```

**Contient** :
```yaml
match_re:
  namespace: '.*-prod-.*'
# Match: team-prod-eu, service-prod-us
```

**Liste de valeurs** :
```yaml
match_re:
  team: '(backend|data|infra)'
# Match: backend, data, ou infra
```

**Exclure des valeurs** :
```yaml
match_re:
  environment: '(?!dev|test).*'
# Match: tout sauf dev et test
```

## Hiérarchie de Routes

Les routes peuvent être **imbriquées** pour créer une logique complexe.

### Exemple : Routage Multi-niveaux

```yaml
route:
  receiver: 'default'
  group_by: ['alertname', 'cluster']

  routes:
    # Niveau 1 : Par environnement
    - match:
        environment: 'production'
      receiver: 'prod-default'

      routes:
        # Niveau 2 : Par sévérité en production
        - match:
            severity: 'critical'
          receiver: 'prod-pagerduty'

        - match:
            severity: 'warning'
          receiver: 'prod-slack'

    # Staging : moins urgent
    - match:
        environment: 'staging'
      receiver: 'staging-slack'
      group_wait: 5m
      repeat_interval: 12h
```

**Logique** :
1. Si `environment=production` :
   - Et `severity=critical` → PagerDuty (urgent)
   - Et `severity=warning` → Slack production
   - Sinon → Email production par défaut
2. Si `environment=staging` → Slack staging (moins urgent)
3. Sinon → Email par défaut

### Visualisation de l'Arbre

```
Route racine (default)
│
├─► production
│   ├─► critical → PagerDuty
│   ├─► warning  → Slack prod
│   └─► autres   → Email prod
│
├─► staging → Slack staging
│
└─► autres → Email default
```

## Le Paramètre Continue

Par défaut, Alertmanager s'arrête à la première route qui correspond. Le paramètre `continue: true` permet de continuer l'évaluation.

### Sans Continue (comportement par défaut)

```yaml
routes:
  - match:
      severity: 'critical'
    receiver: 'pagerduty'
    # continue: false (par défaut)

  - match:
      team: 'backend'
    receiver: 'slack-backend'
```

**Résultat** : Une alerte `critical` du `team=backend` va **uniquement** vers PagerDuty.

### Avec Continue

```yaml
routes:
  - match:
      severity: 'critical'
    receiver: 'pagerduty'
    continue: true  # Continue l'évaluation

  - match:
      team: 'backend'
    receiver: 'slack-backend'
```

**Résultat** : Une alerte `critical` du `team=backend` va vers **PagerDuty ET Slack**.

### Cas d'Usage pour Continue

**Notifications multiples** : Envoyer les alertes critiques à la fois à l'astreinte ET au canal Slack de l'équipe

**Logging centralisé** : Toutes les alertes vont vers un système de logging en plus de leur destination normale

**Escalade** : Alertes importantes notifiées à plusieurs niveaux

### Exemple Pratique

```yaml
route:
  receiver: 'default'

  routes:
    # Toutes les alertes critiques vont en astreinte
    - match:
        severity: 'critical'
      receiver: 'oncall-pagerduty'
      continue: true

    # ET aussi vers l'équipe concernée
    - match:
        team: 'backend'
      receiver: 'slack-backend'

    - match:
        team: 'frontend'
      receiver: 'slack-frontend'

    # ET toutes les alertes sont loggées
    - match: {}  # Match toutes les alertes
      receiver: 'webhook-logging'
```

**Résultat** : Une alerte `critical` de l'équipe `backend` sera envoyée à :
1. PagerDuty (astreinte)
2. Slack backend (équipe)
3. Webhook de logging

## Le Grouping : Regrouper les Alertes

Le grouping évite que vous receviez 50 notifications individuelles quand 50 pods ont le même problème. Au lieu de cela, vous recevez une seule notification qui les regroupe.

### Le Problème sans Grouping

Imaginez ce scénario :
- Vous déployez une nouvelle version d'une application
- 10 pods redémarrent simultanément
- Chaque pod génère 3 alertes différentes

**Sans grouping** : 30 notifications arrivent en 1 minute. Votre téléphone ne s'arrête pas de sonner.

**Avec grouping** : 1 notification groupée vous informe que "10 pods ont des problèmes de mémoire".

### Les Paramètres de Grouping

Il y a 4 paramètres qui contrôlent le grouping :

**group_by** : Sur quels labels regrouper
**group_wait** : Temps d'attente avant la première notification
**group_interval** : Temps entre les mises à jour d'un groupe
**repeat_interval** : Temps entre les répétitions

### group_by : Critères de Regroupement

Le paramètre `group_by` définit les labels utilisés pour créer les groupes.

```yaml
group_by: ['alertname', 'namespace']
```

**Signification** : Les alertes ayant les mêmes valeurs pour `alertname` ET `namespace` seront regroupées.

#### Exemple Concret

Vous avez ces alertes :

| Alert | alertname | namespace | pod |
|-------|-----------|-----------|-----|
| A1 | HighMemory | production | app-1 |
| A2 | HighMemory | production | app-2 |
| A3 | HighMemory | production | app-3 |
| A4 | HighMemory | staging | app-1 |
| A5 | HighCPU | production | api-1 |

**Avec `group_by: ['alertname', 'namespace']`** :

- **Groupe 1** : A1, A2, A3 (HighMemory + production)
- **Groupe 2** : A4 (HighMemory + staging)
- **Groupe 3** : A5 (HighCPU + production)

Vous recevrez **3 notifications** au lieu de 5.

**Avec `group_by: ['alertname']`** :

- **Groupe 1** : A1, A2, A3, A4 (tous HighMemory)
- **Groupe 2** : A5 (HighCPU)

Vous recevrez **2 notifications** au lieu de 5.

**Avec `group_by: ['...']`** (groupe spécial) :

- **Groupe 1** : Toutes les alertes regroupées

Vous recevrez **1 seule notification** pour toutes les alertes.

### group_wait : Temps d'Attente Initial

Quand une **nouvelle alerte** arrive et crée un nouveau groupe, Alertmanager attend un certain temps avant d'envoyer la notification. Cela permet de collecter d'autres alertes similaires qui pourraient arriver juste après.

```yaml
group_wait: 30s
```

**Timeline** :
```
T+0s    : Alerte 1 arrive → Nouveau groupe créé
T+5s    : Alerte 2 arrive → Ajoutée au groupe
T+10s   : Alerte 3 arrive → Ajoutée au groupe
T+30s   : Notification envoyée avec les 3 alertes
```

**Quand utiliser** :
- **court (10s-30s)** : Alertes critiques nécessitant une réaction rapide
- **moyen (1m-2m)** : Alertes importantes mais non urgentes
- **long (5m-10m)** : Alertes d'information

### group_interval : Intervalle de Mise à Jour

Pour un groupe **existant**, si de nouvelles alertes arrivent, Alertmanager attend `group_interval` avant d'envoyer une mise à jour.

```yaml
group_interval: 5m
```

**Timeline** :
```
T+0s     : Groupe créé (Alerte 1, 2, 3)
T+30s    : Première notification envoyée
T+2m     : Alerte 4 arrive → Ajoutée au groupe
T+5m30s  : Notification de mise à jour envoyée (maintenant 4 alertes)
T+7m     : Alerte 5 arrive → Ajoutée au groupe
T+10m30s : Notification de mise à jour envoyée (maintenant 5 alertes)
```

**Quand utiliser** :
- **court (2m-5m)** : Situations qui évoluent rapidement
- **moyen (10m-15m)** : Situations normales
- **long (30m-1h)** : Situations stables

### repeat_interval : Répétition de Notification

Tant que les alertes d'un groupe sont actives, Alertmanager renvoie une notification à intervalles réguliers.

```yaml
repeat_interval: 4h
```

**Timeline** :
```
T+0      : Première notification
T+4h     : Rappel (alertes toujours actives)
T+8h     : Rappel
T+12h    : Rappel
```

**Quand utiliser** :
- **court (30m-1h)** : Alertes critiques qui doivent être résolues rapidement
- **moyen (4h-12h)** : Alertes importantes mais tolérables temporairement
- **long (24h)** : Alertes d'information ou tendances à long terme

### Configuration Complète d'un Groupe

```yaml
route:
  receiver: 'default'
  group_by: ['alertname', 'cluster', 'namespace']
  group_wait: 30s
  group_interval: 5m
  repeat_interval: 4h
```

**Comportement** :
1. Les alertes sont regroupées par `alertname`, `cluster` et `namespace`
2. Quand un nouveau groupe se forme, on attend 30s avant d'envoyer
3. Les mises à jour sont envoyées toutes les 5 minutes max
4. Les rappels sont envoyés toutes les 4 heures

## Exemples de Configurations Réalistes

### Configuration 1 : Startup Simple

Pour une petite équipe qui débute :

```yaml
route:
  receiver: 'team-slack'
  group_by: ['alertname']
  group_wait: 30s
  group_interval: 5m
  repeat_interval: 12h

receivers:
  - name: 'team-slack'
    slack_configs:
      - api_url: 'webhook-url'
        channel: '#alerts'
```

**Caractéristiques** :
- Tout va sur Slack
- Grouping simple par nom d'alerte
- Rappels toutes les 12h

**Adapté pour** : Petites équipes, environnement de développement

### Configuration 2 : Production Sérieuse

Pour un environnement de production avec une équipe d'astreinte :

```yaml
route:
  receiver: 'default'
  group_by: ['alertname', 'cluster', 'namespace']
  group_wait: 10s
  group_interval: 5m
  repeat_interval: 3h

  routes:
    # Critiques en production : astreinte immédiate
    - match:
        severity: 'critical'
        environment: 'production'
      receiver: 'pagerduty-oncall'
      group_wait: 10s
      repeat_interval: 5m
      continue: false

    # Warnings en production : Slack
    - match:
        severity: 'warning'
        environment: 'production'
      receiver: 'slack-prod-warnings'
      group_wait: 1m
      group_interval: 10m
      repeat_interval: 4h

    # Staging et dev : email quotidien
    - match_re:
        environment: '(staging|dev)'
      receiver: 'email-dev-team'
      group_wait: 10m
      group_interval: 1h
      repeat_interval: 24h

receivers:
  - name: 'default'
    email_configs:
      - to: 'team@example.com'

  - name: 'pagerduty-oncall'
    pagerduty_configs:
      - service_key: 'xxx'

  - name: 'slack-prod-warnings'
    slack_configs:
      - channel: '#prod-alerts'

  - name: 'email-dev-team'
    email_configs:
      - to: 'dev-team@example.com'
```

**Caractéristiques** :
- Séparation claire par environnement et sévérité
- Production critique → PagerDuty rapide
- Production warning → Slack modéré
- Non-production → Email lent

**Adapté pour** : Équipes matures avec astreinte

### Configuration 3 : Multi-équipes

Pour une organisation avec plusieurs équipes :

```yaml
route:
  receiver: 'default'
  group_by: ['alertname', 'team']
  group_wait: 30s
  group_interval: 5m
  repeat_interval: 4h

  routes:
    # Équipe Backend
    - match:
        team: 'backend'
      receiver: 'slack-backend'

      routes:
        - match:
            severity: 'critical'
          receiver: 'pagerduty-backend'

    # Équipe Frontend
    - match:
        team: 'frontend'
      receiver: 'slack-frontend'

      routes:
        - match:
            severity: 'critical'
          receiver: 'pagerduty-frontend'

    # Équipe Data
    - match:
        team: 'data'
      receiver: 'slack-data'

      routes:
        - match:
            severity: 'critical'
          receiver: 'pagerduty-data'

    # Infrastructure (tout ce qui n'a pas de team)
    - match_re:
        alertname: '(Node|Disk|Network).*'
      receiver: 'slack-infra'

      routes:
        - match:
            severity: 'critical'
          receiver: 'pagerduty-infra'

receivers:
  - name: 'default'
    email_configs:
      - to: 'all-teams@example.com'

  - name: 'slack-backend'
    slack_configs:
      - channel: '#backend-alerts'

  - name: 'pagerduty-backend'
    pagerduty_configs:
      - service_key: 'backend-key'

  # ... autres receivers
```

**Caractéristiques** :
- Routage par équipe
- Chaque équipe a son Slack et son PagerDuty
- Hiérarchie : team → severity
- Infrastructure gérée séparément

**Adapté pour** : Grandes organisations avec équipes autonomes

### Configuration 4 : Heures de Travail

Pour différencier les heures ouvrables des heures non-ouvrables :

```yaml
route:
  receiver: 'default'
  group_by: ['alertname']

  routes:
    # Critiques : toujours PagerDuty
    - match:
        severity: 'critical'
      receiver: 'pagerduty-24-7'
      repeat_interval: 5m

    # Warnings : heures de travail → Slack, nuit → email
    - match:
        severity: 'warning'
      receiver: 'slack-business-hours'
      group_wait: 2m
      repeat_interval: 4h

      # Note : Le time-based routing nécessite une configuration externe
      # ou des labels ajoutés par Prometheus

receivers:
  - name: 'default'
    email_configs:
      - to: 'team@example.com'

  - name: 'pagerduty-24-7'
    pagerduty_configs:
      - service_key: 'xxx'

  - name: 'slack-business-hours'
    slack_configs:
      - channel: '#alerts'
```

**Note** : Alertmanager ne supporte pas nativement le routing basé sur l'heure. Vous devez :
- Utiliser des outils externes (scripts, Prometheus recording rules)
- Ajouter des labels temporels à vos alertes
- Utiliser des solutions tierces comme Grafana OnCall

## Stratégies de Grouping Avancées

### Grouping par Criticité

Différents niveaux d'urgence = différents paramètres de grouping :

```yaml
route:
  receiver: 'default'

  routes:
    # Critical : notification rapide, peu de grouping
    - match:
        severity: 'critical'
      receiver: 'pagerduty'
      group_by: ['alertname']
      group_wait: 10s
      group_interval: 2m
      repeat_interval: 5m

    # Warning : plus de grouping, moins urgent
    - match:
        severity: 'warning'
      receiver: 'slack'
      group_by: ['alertname', 'namespace', 'cluster']
      group_wait: 2m
      group_interval: 10m
      repeat_interval: 4h

    # Info : grouping maximal, très lent
    - match:
        severity: 'info'
      receiver: 'email'
      group_by: ['alertname', 'namespace', 'cluster', 'team']
      group_wait: 10m
      group_interval: 1h
      repeat_interval: 24h
```

**Logique** :
- Plus c'est critique, moins on groupe
- Plus c'est critique, plus c'est rapide
- Plus c'est critique, plus on rappelle souvent

### Grouping Dynamique par Label

```yaml
route:
  receiver: 'default'
  group_by: ['alertname', 'cluster']

  routes:
    # Alertes pod : grouper par namespace aussi
    - match_re:
        alertname: 'Pod.*'
      receiver: 'slack-pods'
      group_by: ['alertname', 'cluster', 'namespace']

    # Alertes node : grouper par datacenter
    - match_re:
        alertname: 'Node.*'
      receiver: 'slack-nodes'
      group_by: ['alertname', 'datacenter']

    # Alertes réseau : grouper au maximum
    - match_re:
        alertname: 'Network.*'
      receiver: 'slack-network'
      group_by: ['alertname']
```

**Adaptation** : Le grouping s'adapte au type d'alerte

### Grouping Minimal (group_by: [...])

Le groupe spécial `[...]` groupe **toutes les alertes ensemble** :

```yaml
route:
  receiver: 'digest-email'
  group_by: ['...']
  group_wait: 1h
  group_interval: 24h
  repeat_interval: 24h

receivers:
  - name: 'digest-email'
    email_configs:
      - to: 'daily-digest@example.com'
        headers:
          Subject: 'Résumé quotidien des alertes'
```

**Usage** : Rapports quotidiens, digests, environnements non-critiques

## Patterns Anti-patterns et Bonnes Pratiques

### ✓ Bonnes Pratiques

**1. Routes spécifiques avant routes générales**

```yaml
routes:
  # ✓ Spécifique d'abord
  - match:
      severity: 'critical'
      team: 'backend'
    receiver: 'backend-pagerduty'

  # Général après
  - match:
      severity: 'critical'
    receiver: 'general-pagerduty'
```

**2. Utiliser continue pour notifications multiples**

```yaml
routes:
  - match:
      severity: 'critical'
    receiver: 'pagerduty'
    continue: true  # ✓ Continue pour aussi notifier l'équipe

  - match:
      team: 'backend'
    receiver: 'slack-backend'
```

**3. Adapter le grouping à la sévérité**

```yaml
# ✓ Critical : grouping minimal, rapide
- match:
    severity: 'critical'
  group_by: ['alertname']
  group_wait: 10s

# ✓ Warning : grouping plus large, plus lent
- match:
    severity: 'warning'
  group_by: ['alertname', 'namespace', 'cluster']
  group_wait: 2m
```

**4. Documenter les routes complexes**

```yaml
routes:
  # Route 1: Alertes critiques production
  # Envoi immédiat vers PagerDuty pour l'astreinte
  # Répétition toutes les 5 minutes jusqu'à résolution
  - match:
      severity: 'critical'
      environment: 'production'
    receiver: 'pagerduty-oncall'
    group_wait: 10s
    repeat_interval: 5m
```

### ✗ Anti-patterns à Éviter

**1. Grouping trop agressif sur les alertes critiques**

```yaml
# ✗ Mauvais : toutes les alertes critiques dans un seul groupe
- match:
    severity: 'critical'
  group_by: ['...']
  group_wait: 10m
```

**Problème** : Vous perdez le détail et réagissez lentement

**Solution** :
```yaml
# ✓ Bon : grouping spécifique
- match:
    severity: 'critical'
  group_by: ['alertname', 'cluster']
  group_wait: 10s
```

**2. Répétitions trop fréquentes**

```yaml
# ✗ Mauvais : spam toutes les minutes
- match:
    severity: 'warning'
  repeat_interval: 1m
```

**Problème** : Fatigue d'alerte, notifications ignorées

**Solution** :
```yaml
# ✓ Bon : intervalle raisonnable
- match:
    severity: 'warning'
  repeat_interval: 4h
```

**3. Routes trop génériques avant les spécifiques**

```yaml
# ✗ Mauvais : ordre incorrect
routes:
  - match: {}  # Matche tout
    receiver: 'default'

  - match:
      severity: 'critical'
    receiver: 'pagerduty'  # Ne sera jamais atteint !
```

**Problème** : Les routes spécifiques ne sont jamais atteintes

**Solution** :
```yaml
# ✓ Bon : spécifique d'abord
routes:
  - match:
      severity: 'critical'
    receiver: 'pagerduty'

  - match: {}
    receiver: 'default'
```

**4. Oublier le paramètre continue**

```yaml
# ✗ Mauvais : l'équipe ne sera pas notifiée
routes:
  - match:
      severity: 'critical'
    receiver: 'pagerduty'
    # Pas de continue

  - match:
      team: 'backend'
    receiver: 'slack-backend'
```

**Solution** :
```yaml
# ✓ Bon : avec continue
routes:
  - match:
      severity: 'critical'
    receiver: 'pagerduty'
    continue: true  # Continue l'évaluation

  - match:
      team: 'backend'
    receiver: 'slack-backend'
```

**5. Group_wait trop long pour les alertes critiques**

```yaml
# ✗ Mauvais : attend 10 minutes avant de notifier
- match:
    severity: 'critical'
  group_wait: 10m
```

**Problème** : Retard de 10 minutes pour les alertes urgentes

**Solution** :
```yaml
# ✓ Bon : notification rapide
- match:
    severity: 'critical'
  group_wait: 10s
```

## Debugging du Routing

### Vérifier Quelle Route Sera Utilisée

Alertmanager fournit une API pour tester vos routes sans envoyer de vraies alertes.

**Simuler une alerte** :

```bash
curl -X POST http://localhost:9093/api/v2/alerts \
  -H 'Content-Type: application/json' \
  -d '[
    {
      "labels": {
        "alertname": "TestAlert",
        "severity": "critical",
        "team": "backend",
        "environment": "production"
      },
      "annotations": {
        "summary": "Test de routing"
      }
    }
  ]'
```

**Vérifier dans l'interface** :
1. Ouvrir `http://localhost:9093`
2. Aller dans l'onglet **Alerts**
3. Voir comment l'alerte a été routée

### Activer le Mode Debug

Pour voir les décisions de routing en détail :

```bash
# Voir les logs d'Alertmanager
microk8s kubectl logs -n monitoring alertmanager-0 --tail=100 -f
```

Les logs montrent :
- Quelle route a matché
- Quel receiver a été utilisé
- Les groupes créés
- Les notifications envoyées

### Utiliser amtool

`amtool` est l'outil CLI officiel pour Alertmanager :

```bash
# Installer amtool
go install github.com/prometheus/alertmanager/cmd/amtool@latest

# Vérifier la configuration
amtool config routes --config.file=alertmanager.yml

# Tester une alerte
amtool alert add alertname=test severity=critical team=backend
```

## Exemples de Scénarios Réels

### Scénario 1 : Déploiement qui Échoue

**Situation** :
- Vous déployez une nouvelle version
- 20 pods crashent en même temps
- Chaque pod génère 3 alertes : CrashLoop, HighMemory, HighRestarts

**Sans grouping** : 60 notifications

**Avec grouping** :

```yaml
group_by: ['alertname', 'deployment']
group_wait: 30s
```

**Résultat** :
- 1 notification pour CrashLoop (20 pods)
- 1 notification pour HighMemory (20 pods)
- 1 notification pour HighRestarts (20 pods)

Total : **3 notifications** au lieu de 60

### Scénario 2 : Nœud qui Tombe

**Situation** :
- Un nœud Kubernetes tombe en panne
- Déclenche 50 alertes : NodeDown, 30 pods NotReady, 15 services Unavailable

**Configuration intelligente** :

```yaml
inhibit_rules:
  - source_match:
      alertname: 'NodeDown'
    target_match_re:
      alertname: '(PodNotReady|ServiceUnavailable)'
    equal: ['node']
```

**Résultat** :
- 1 notification : "NodeDown"
- Les 45 autres alertes sont inhibées car la cause est connue

### Scénario 3 : Pic de Trafic

**Situation** :
- Pic de trafic inattendu
- Latence augmente progressivement
- 10 alertes HighLatency sur différents services en 5 minutes

**Configuration** :

```yaml
group_by: ['alertname']
group_wait: 1m
group_interval: 5m
```

**Timeline** :
```
T+0   : 1ère alerte → Groupe créé
T+30s : 5 alertes de plus
T+1m  : Notification envoyée (6 alertes)
T+2m  : 4 alertes de plus
T+6m  : Mise à jour envoyée (10 alertes)
```

**Résultat** : 2 notifications au lieu de 10

## Métriques de Routing

Surveillez l'efficacité de votre routing avec ces métriques Alertmanager :

```promql
# Nombre de notifications par receiver
alertmanager_notifications_total{receiver="slack"}

# Taux d'échec des notifications
rate(alertmanager_notifications_failed_total[5m])

# Alertes par état
alertmanager_alerts{state="active"}

# Silences actifs
alertmanager_silences{state="active"}
```

Créez des alertes sur ces métriques :

```yaml
- alert: NotificationFailureHigh
  expr: |
    rate(alertmanager_notifications_failed_total[5m]) > 0.1
  for: 10m
  annotations:
    summary: "Échec de notifications Alertmanager"
    description: "{{ $value }} notifications/sec échouent"
```

## Migration et Évolution

### Tester avant de Déployer

1. **Créer un Alertmanager de test**
2. **Copier votre configuration de production**
3. **Modifier la configuration**
4. **Envoyer des alertes de test**
5. **Vérifier le comportement**
6. **Déployer en production**

### Rollout Progressif

Pour des changements majeurs :

```yaml
# Phase 1 : Envoyer à l'ancienne ET à la nouvelle destination
routes:
  - match:
      severity: 'critical'
    receiver: 'old-pagerduty'
    continue: true  # Continuer

  - match:
      severity: 'critical'
    receiver: 'new-pagerduty'  # Nouvelle destination en parallèle

# Phase 2 (après validation) : Supprimer l'ancienne
routes:
  - match:
      severity: 'critical'
    receiver: 'new-pagerduty'
```

### Versioning de Configuration

```bash
/alertmanager-config/
├── v1.0/
│   └── alertmanager.yml
├── v1.1/
│   └── alertmanager.yml
└── current -> v1.1/
```

Gardez un historique de vos configurations pour revenir en arrière si nécessaire.

## Conclusion

Le routing et le grouping sont les piliers d'un système d'alerting efficace. Bien configurés, ils transforment le chaos d'alertes individuelles en flux organisés de notifications pertinentes.

**Principes clés** :
- Le routing dirige les alertes vers les bonnes personnes
- Le grouping évite le spam en regroupant intelligemment
- Adaptez les paramètres à la sévérité des alertes
- Testez avant de déployer en production
- Itérez et améliorez continuellement

Avec une configuration bien pensée, vous maintenez vos équipes informées sans les submerger, et vous réagissez rapidement aux problèmes critiques tout en gérant efficacement les alertes moins urgentes.

## Points Clés à Retenir

✓ Le routing utilise match/match_re pour diriger les alertes vers les bons receivers

✓ Les routes sont évaluées dans l'ordre, la première qui matche gagne (sauf si continue: true)

✓ Le paramètre continue permet d'envoyer une alerte à plusieurs destinations

✓ group_by définit les critères de regroupement des alertes

✓ group_wait définit l'attente avant la première notification (collecte des alertes)

✓ group_interval définit l'intervalle entre les mises à jour d'un groupe

✓ repeat_interval définit la fréquence des rappels

✓ Adaptez les paramètres à la sévérité : critique = rapide, info = lent

✓ Routes spécifiques avant routes générales dans la configuration

✓ Testez votre configuration avant de déployer en production

---

**Prochaine section** : 14.5 Notifications (Slack, email, webhook) - Nous détaillerons la configuration des différents canaux de notification.

⏭️ [Notifications (Slack, email, webhook)](/14-alerting-et-notifications/05-notifications-slack-email-webhook.md)
