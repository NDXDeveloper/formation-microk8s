🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 14.2 Prometheus Alertmanager

## Introduction

Maintenant que vous comprenez les concepts fondamentaux de l'alerting, il est temps de découvrir **Prometheus Alertmanager**, le composant qui transforme vos alertes en notifications concrètes. Si Prometheus est le détective qui identifie les problèmes, Alertmanager est le messager qui vous informe de la manière la plus appropriée.

Alertmanager est bien plus qu'un simple service d'envoi de notifications. C'est un système sophistiqué capable de :
- Regrouper les alertes similaires
- Éviter les notifications en double
- Router les alertes vers les bonnes personnes
- Gérer les périodes de silence
- Escalader les alertes non traitées

## Architecture d'Alertmanager

### Le Flux de Données

Voici comment les alertes circulent dans votre infrastructure :

```
[Prometheus] → Évalue les règles d'alerte
     ↓
[Alertmanager] → Reçoit les alertes
     ↓
Grouping → Regroupe les alertes similaires
     ↓
Throttling → Évite les notifications répétées
     ↓
Silencing → Applique les silences définis
     ↓
Routing → Dirige vers les bons canaux
     ↓
[Notifications] → Email, Slack, PagerDuty, etc.
```

### Les Composants Internes

**Receiver** : La configuration d'un canal de notification (email, Slack, webhook, etc.)

**Route** : Les règles qui déterminent quel receiver utiliser pour quelle alerte

**Inhibition** : Les règles qui suppriment certaines alertes en fonction d'autres

**Silence** : Les périodes pendant lesquelles certaines alertes sont temporairement désactivées

**Grouping** : Le mécanisme qui regroupe plusieurs alertes en une seule notification

## Installation sur MicroK8s

La bonne nouvelle : si vous avez déjà activé l'addon Prometheus sur MicroK8s, Alertmanager est probablement déjà installé !

### Vérifier si Alertmanager est Présent

```bash
microk8s kubectl get pods -n monitoring
```

Vous devriez voir un pod nommé `alertmanager-...` s'il est déjà déployé.

### Installation avec l'Addon Prometheus

Si vous n'avez pas encore activé Prometheus :

```bash
microk8s enable prometheus
```

Cette commande installe :
- Prometheus Server
- Prometheus Alertmanager
- Grafana
- Node Exporter
- Kube-state-metrics

### Accéder à l'Interface Web d'Alertmanager

Alertmanager dispose d'une interface web pour visualiser les alertes actives et les silences.

**Vérifier le service** :
```bash
microk8s kubectl get svc -n monitoring | grep alertmanager
```

**Exposer temporairement l'interface** :
```bash
microk8s kubectl port-forward -n monitoring svc/alertmanager-operated 9093:9093
```

Puis ouvrez votre navigateur sur : `http://localhost:9093`

### Exploration de l'Interface Web

L'interface d'Alertmanager comporte plusieurs sections :

**Alerts** : Affiche toutes les alertes actives (firing), regroupées selon votre configuration

**Silences** : Gère les silences actifs et permet d'en créer de nouveaux

**Status** : Affiche la configuration en cours et l'état du cluster Alertmanager

## Configuration de Base

La configuration d'Alertmanager se fait via un fichier YAML. Dans MicroK8s, ce fichier est généralement stocké dans un ConfigMap.

### Structure du Fichier de Configuration

```yaml
global:
  # Configuration globale (serveurs SMTP, URLs API, etc.)

route:
  # Arbre de routage des alertes

receivers:
  # Définition des canaux de notification

inhibit_rules:
  # Règles de suppression d'alertes redondantes
```

### Localiser la Configuration dans MicroK8s

```bash
microk8s kubectl get configmap -n monitoring | grep alertmanager
```

Pour voir la configuration actuelle :
```bash
microk8s kubectl get configmap alertmanager-config -n monitoring -o yaml
```

## Les Receivers : Canaux de Notification

Un **receiver** définit comment et où envoyer les notifications. Alertmanager supporte de nombreux types de receivers.

### Types de Receivers Disponibles

**Email** : Notifications par email via SMTP

**Slack** : Messages dans des canaux Slack

**PagerDuty** : Intégration avec le système d'astreinte PagerDuty

**Webhook** : Envoi vers une URL HTTP personnalisée

**OpsGenie, VictorOps, WeChat, Telegram** : Autres intégrations populaires

### Exemple : Receiver Email

```yaml
receivers:
  - name: 'email-admin'
    email_configs:
      - to: 'admin@example.com'
        from: 'alertmanager@example.com'
        smarthost: 'smtp.gmail.com:587'
        auth_username: 'alertmanager@example.com'
        auth_password: 'votre-mot-de-passe'
        require_tls: true
```

**Explication** :
- `name` : Identifiant unique du receiver
- `to` : Adresse email du destinataire
- `from` : Adresse email de l'expéditeur
- `smarthost` : Serveur SMTP et port
- `auth_username/auth_password` : Identifiants SMTP
- `require_tls` : Utilisation de TLS (recommandé)

### Exemple : Receiver Slack

```yaml
receivers:
  - name: 'slack-alerts'
    slack_configs:
      - api_url: 'https://hooks.slack.com/services/XXXXX/YYYYY/ZZZZZ'
        channel: '#alerts'
        title: 'Alerte Kubernetes'
        text: '{{ range .Alerts }}{{ .Annotations.description }}{{ end }}'
```

**Explication** :
- `api_url` : Webhook URL fournie par Slack (Incoming Webhooks)
- `channel` : Canal Slack de destination
- `title` : Titre du message
- `text` : Corps du message (utilise le templating Go)

### Exemple : Receiver Webhook

```yaml
receivers:
  - name: 'webhook-custom'
    webhook_configs:
      - url: 'http://mon-service:8080/alerts'
        send_resolved: true
```

**Explication** :
- `url` : URL de votre endpoint qui recevra les alertes au format JSON
- `send_resolved` : Envoie aussi une notification quand l'alerte est résolue

## Le Routage des Alertes

Le **routing** est le mécanisme qui détermine quel receiver utiliser pour chaque alerte. C'est comme un système d'aiguillage ferroviaire pour vos notifications.

### Structure d'une Route

Une route est définie par :
- **match** ou **match_re** : Conditions pour qu'une alerte corresponde à cette route
- **receiver** : Le receiver à utiliser si l'alerte correspond
- **continue** : Si `true`, continue à évaluer les routes suivantes
- **routes** : Routes enfants (sous-routes)

### Exemple Simple : Routage par Sévérité

```yaml
route:
  receiver: 'default-receiver'
  group_by: ['alertname', 'cluster']
  group_wait: 10s
  group_interval: 5m
  repeat_interval: 3h

  routes:
    - match:
        severity: 'critical'
      receiver: 'pagerduty-critical'

    - match:
        severity: 'warning'
      receiver: 'slack-warnings'

    - match:
        severity: 'info'
      receiver: 'email-info'
```

**Explication** :
- Route par défaut : Utilise `default-receiver` pour toutes les alertes
- Route 1 : Les alertes critiques vont vers PagerDuty
- Route 2 : Les warnings vont vers Slack
- Route 3 : Les infos vont par email

### Exemple Avancé : Routage Multi-critères

```yaml
route:
  receiver: 'default'

  routes:
    # Alertes critiques de production
    - match:
        severity: 'critical'
        environment: 'production'
      receiver: 'pagerduty-oncall'
      continue: false

    # Alertes de l'équipe backend
    - match_re:
        service: '(api|database|cache)'
      receiver: 'slack-backend-team'
      continue: true

    # Alertes de l'équipe frontend
    - match_re:
        service: '(webapp|mobile-app)'
      receiver: 'slack-frontend-team'
```

**Explication** :
- Les alertes critiques en production vont directement à l'astreinte
- `continue: false` empêche l'évaluation des routes suivantes
- Les routes backend et frontend utilisent des regex pour matcher plusieurs services
- `continue: true` permet d'envoyer une alerte à plusieurs receivers

## Le Grouping : Regroupement d'Alertes

Le **grouping** évite que vous receviez 50 notifications individuelles quand 50 pods d'une même application ont un problème. Au lieu de cela, vous recevez une seule notification groupée.

### Paramètres de Grouping

**group_by** : Liste des labels utilisés pour regrouper les alertes
```yaml
group_by: ['alertname', 'namespace', 'cluster']
```
Les alertes ayant les mêmes valeurs pour ces labels seront regroupées.

**group_wait** : Temps d'attente avant d'envoyer la première notification d'un nouveau groupe
```yaml
group_wait: 10s
```
Attend 10 secondes pour rassembler les alertes similaires avant d'envoyer.

**group_interval** : Temps d'attente avant d'envoyer des mises à jour pour un groupe existant
```yaml
group_interval: 5m
```
Si de nouvelles alertes arrivent dans un groupe existant, attend 5 minutes avant de notifier.

**repeat_interval** : Intervalle entre les répétitions de notification
```yaml
repeat_interval: 4h
```
Renvoie une notification toutes les 4 heures tant que l'alerte est active.

### Exemple Pratique de Grouping

Imaginons ces alertes qui se déclenchent simultanément :
- `HighCPU` sur le pod `api-1` dans `production`
- `HighCPU` sur le pod `api-2` dans `production`
- `HighCPU` sur le pod `api-3` dans `production`

Avec `group_by: ['alertname', 'cluster']`, vous recevrez **une seule notification** qui liste les 3 pods concernés, plutôt que 3 notifications séparées.

## Les Inhibitions : Supprimer les Alertes Redondantes

Les **inhibitions** permettent de supprimer certaines alertes lorsque d'autres alertes plus importantes sont actives. C'est comme une règle de priorité.

### Cas d'Usage Typique

Si un serveur entier est down, pas besoin d'être alerté sur tous les services qui tournent dessus - vous le savez déjà.

### Structure d'une Règle d'Inhibition

```yaml
inhibit_rules:
  - source_match:
      severity: 'critical'
      alertname: 'NodeDown'
    target_match:
      severity: 'warning'
    target_match_re:
      alertname: '(ServiceUnavailable|HighLatency)'
    equal: ['node']
```

**Explication** :
- **source_match** : L'alerte "source" qui, si active, inhibe les autres
- **target_match** : Les alertes "cibles" qui seront inhibées
- **equal** : Les labels qui doivent être identiques entre source et cible
- Ici : Si `NodeDown` est actif sur un nœud, les alertes `ServiceUnavailable` et `HighLatency` sur ce même nœud sont inhibées

### Exemple Pratique

```yaml
inhibit_rules:
  # Si le cluster est down, ne pas alerter sur les pods
  - source_match:
      alertname: 'ClusterDown'
    target_match_re:
      alertname: 'Pod.*'
    equal: ['cluster']

  # Si une alerte critique est active, masquer les warnings similaires
  - source_match:
      severity: 'critical'
    target_match:
      severity: 'warning'
    equal: ['alertname', 'namespace']
```

## Les Silences : Désactiver Temporairement des Alertes

Un **silence** permet de désactiver temporairement certaines alertes, typiquement pendant :
- Une maintenance planifiée
- Un déploiement
- Une investigation en cours

### Créer un Silence via l'Interface Web

1. Accédez à l'interface web d'Alertmanager : `http://localhost:9093`
2. Cliquez sur l'onglet **"Silences"**
3. Cliquez sur **"New Silence"**
4. Configurez les matchers (ex: `alertname=HighCPU`)
5. Définissez la durée (ex: 2 heures)
6. Ajoutez un commentaire expliquant pourquoi
7. Cliquez sur **"Create"**

### Créer un Silence via l'API

```bash
curl -X POST http://localhost:9093/api/v2/silences \
  -H 'Content-Type: application/json' \
  -d '{
    "matchers": [
      {
        "name": "alertname",
        "value": "HighCPU",
        "isRegex": false
      },
      {
        "name": "namespace",
        "value": "production",
        "isRegex": false
      }
    ],
    "startsAt": "2025-01-01T10:00:00.000Z",
    "endsAt": "2025-01-01T12:00:00.000Z",
    "createdBy": "admin",
    "comment": "Maintenance planifiée"
  }'
```

### Bonnes Pratiques pour les Silences

✓ Toujours ajouter un commentaire expliquant la raison

✓ Définir une durée appropriée (pas trop longue)

✓ Utiliser des matchers spécifiques (éviter de silencer trop largement)

✓ Nettoyer les silences une fois la maintenance terminée

✗ Ne jamais créer de silence permanent pour masquer un problème récurrent

## Templates de Notifications

Alertmanager utilise le système de templating de Go pour personnaliser le contenu des notifications.

### Variables Disponibles

Dans vos templates, vous pouvez utiliser :

- `.Alerts` : Liste de toutes les alertes dans le groupe
- `.GroupLabels` : Les labels utilisés pour grouper
- `.CommonLabels` : Les labels communs à toutes les alertes du groupe
- `.CommonAnnotations` : Les annotations communes
- `.ExternalURL` : L'URL d'Alertmanager
- `.Status` : État du groupe (firing/resolved)

### Exemple de Template pour Slack

```yaml
receivers:
  - name: 'slack-custom'
    slack_configs:
      - api_url: 'https://hooks.slack.com/services/XXX'
        channel: '#alerts'
        title: '{{ .Status | toUpper }} : {{ .GroupLabels.alertname }}'
        text: |
          {{ range .Alerts }}
          *Alerte:* {{ .Labels.alertname }}
          *Sévérité:* {{ .Labels.severity }}
          *Description:* {{ .Annotations.description }}
          *Pod:* {{ .Labels.pod }}
          *Namespace:* {{ .Labels.namespace }}
          {{ end }}
        color: '{{ if eq .Status "firing" }}danger{{ else }}good{{ end }}'
```

**Résultat** : Messages Slack avec des couleurs (rouge pour firing, vert pour resolved) et des informations structurées.

### Exemple de Template pour Email

```yaml
receivers:
  - name: 'email-detailed'
    email_configs:
      - to: 'oncall@example.com'
        headers:
          Subject: '[{{ .Status }}] {{ .GroupLabels.alertname }}'
        html: |
          <!DOCTYPE html>
          <html>
            <body>
              <h2>Alertes Kubernetes</h2>
              <p><strong>Statut:</strong> {{ .Status }}</p>
              <table border="1">
                <tr>
                  <th>Alerte</th>
                  <th>Sévérité</th>
                  <th>Description</th>
                </tr>
                {{ range .Alerts }}
                <tr>
                  <td>{{ .Labels.alertname }}</td>
                  <td>{{ .Labels.severity }}</td>
                  <td>{{ .Annotations.description }}</td>
                </tr>
                {{ end }}
              </table>
            </body>
          </html>
```

## Configuration Complète : Exemple Réaliste

Voici un exemple de configuration Alertmanager complet et réaliste pour un environnement de production :

```yaml
global:
  smtp_smarthost: 'smtp.gmail.com:587'
  smtp_from: 'alertmanager@example.com'
  smtp_auth_username: 'alertmanager@example.com'
  smtp_auth_password: 'votre-mot-de-passe'
  slack_api_url: 'https://hooks.slack.com/services/XXX/YYY/ZZZ'

# Route principale
route:
  receiver: 'default-email'
  group_by: ['alertname', 'cluster', 'namespace']
  group_wait: 10s
  group_interval: 10m
  repeat_interval: 12h

  routes:
    # Alertes critiques en production
    - match:
        severity: 'critical'
        environment: 'production'
      receiver: 'pagerduty-oncall'
      group_wait: 10s
      repeat_interval: 5m
      continue: false

    # Warnings en production
    - match:
        severity: 'warning'
        environment: 'production'
      receiver: 'slack-prod-warnings'
      group_wait: 30s
      repeat_interval: 3h

    # Alertes de staging
    - match:
        environment: 'staging'
      receiver: 'slack-staging'
      group_wait: 5m
      repeat_interval: 24h

    # Infos (pas urgent)
    - match:
        severity: 'info'
      receiver: 'email-info'
      group_wait: 10m
      repeat_interval: 24h

# Définition des receivers
receivers:
  - name: 'default-email'
    email_configs:
      - to: 'team@example.com'

  - name: 'pagerduty-oncall'
    pagerduty_configs:
      - service_key: 'votre-service-key'

  - name: 'slack-prod-warnings'
    slack_configs:
      - channel: '#prod-alerts'
        title: 'Warning Production'
        text: '{{ range .Alerts }}{{ .Annotations.description }}{{ end }}'

  - name: 'slack-staging'
    slack_configs:
      - channel: '#staging-alerts'

  - name: 'email-info'
    email_configs:
      - to: 'info@example.com'

# Règles d'inhibition
inhibit_rules:
  # Si un nœud est down, ne pas alerter sur les pods
  - source_match:
      alertname: 'NodeDown'
    target_match_re:
      alertname: '(PodCrashLooping|HighMemory|HighCPU)'
    equal: ['node']

  # Si critique, masquer les warnings similaires
  - source_match:
      severity: 'critical'
    target_match:
      severity: 'warning'
    equal: ['alertname', 'namespace']
```

## Appliquer une Nouvelle Configuration

### Méthode 1 : Modifier le ConfigMap

```bash
# Éditer le ConfigMap
microk8s kubectl edit configmap alertmanager-config -n monitoring

# Recharger la configuration sans redémarrer
microk8s kubectl exec -n monitoring alertmanager-0 -- \
  curl -X POST http://localhost:9093/-/reload
```

### Méthode 2 : Créer un Nouveau ConfigMap

```bash
# Créer le ConfigMap depuis un fichier
microk8s kubectl create configmap alertmanager-config \
  --from-file=alertmanager.yml=./mon-config.yml \
  -n monitoring \
  --dry-run=client -o yaml | microk8s kubectl apply -f -

# Redémarrer les pods Alertmanager
microk8s kubectl rollout restart statefulset alertmanager -n monitoring
```

## Vérification et Débogage

### Vérifier que la Configuration est Valide

```bash
# Vérifier les logs d'Alertmanager
microk8s kubectl logs -n monitoring alertmanager-0 --tail=50

# Vérifier le statut via l'API
curl http://localhost:9093/api/v2/status
```

### Tester une Alerte Manuellement

Vous pouvez envoyer une alerte de test à Alertmanager :

```bash
curl -X POST http://localhost:9093/api/v2/alerts \
  -H 'Content-Type: application/json' \
  -d '[
    {
      "labels": {
        "alertname": "TestAlert",
        "severity": "warning",
        "instance": "test"
      },
      "annotations": {
        "summary": "Ceci est un test",
        "description": "Alerte de test pour vérifier la configuration"
      }
    }
  ]'
```

### Problèmes Courants et Solutions

**Problème** : Les notifications ne sont pas envoyées

**Solutions** :
- Vérifiez les logs d'Alertmanager
- Vérifiez que le receiver est correctement configuré
- Testez la connectivité réseau (SMTP, webhooks)
- Vérifiez les credentials (mot de passe SMTP, tokens API)

**Problème** : Trop de notifications (spam)

**Solutions** :
- Augmentez `group_interval` et `repeat_interval`
- Utilisez le grouping plus agressivement
- Ajoutez des inhibitions pour supprimer les alertes redondantes
- Revoyez vos seuils d'alerte dans Prometheus

**Problème** : Les alertes ne sont pas routées correctement

**Solutions** :
- Vérifiez que les labels de vos alertes correspondent aux matchers
- Vérifiez l'ordre de vos routes (première route qui match gagne)
- Utilisez `continue: true` si vous voulez plusieurs notifications
- Testez vos regex avec des outils en ligne

## Haute Disponibilité d'Alertmanager

Pour un environnement critique, vous pouvez déployer plusieurs instances d'Alertmanager en cluster.

### Principe du Clustering

Plusieurs instances d'Alertmanager se synchronisent entre elles pour :
- Partager l'état des silences
- Déduplication des notifications
- Tolérance aux pannes

### Configuration de Base du Cluster

```yaml
alertmanager:
  replicas: 3
  cluster:
    peers:
      - alertmanager-0.alertmanager:9094
      - alertmanager-1.alertmanager:9094
      - alertmanager-2.alertmanager:9094
```

Avec 3 instances, si une tombe en panne, les deux autres continuent de fonctionner.

## Bonnes Pratiques de Production

### 1. Stratégie de Routage Multi-niveaux

Organisez vos routes par :
- Environnement (prod > staging > dev)
- Sévérité (critical > warning > info)
- Équipe (backend, frontend, infra)

### 2. Notifications Graduelles

- **Critical** : Notification immédiate (PagerDuty, appel)
- **Warning** : Notification pendant les heures de travail (Slack, email)
- **Info** : Notification passive (email quotidien groupé)

### 3. Documentation et Runbooks

Chaque alerte devrait avoir :
- Un lien vers un runbook dans les annotations
- Une description claire du problème
- Les étapes de résolution suggérées

### 4. Monitoring d'Alertmanager Lui-même

Créez des alertes pour surveiller Alertmanager :
- Alertmanager down
- Configuration invalide
- Échec d'envoi de notifications

### 5. Tests Réguliers

- Testez vos canaux de notification mensuellement
- Simulez des alertes de chaque niveau de sévérité
- Vérifiez le routage et l'escalade
- Pratiquez votre procédure d'astreinte

## Intégrations Populaires

### PagerDuty

Service d'astreinte professionnel avec escalade automatique.

```yaml
receivers:
  - name: 'pagerduty'
    pagerduty_configs:
      - service_key: 'votre-integration-key'
        description: '{{ .GroupLabels.alertname }}'
```

### Slack

Communication d'équipe avec notifications riches.

```yaml
receivers:
  - name: 'slack'
    slack_configs:
      - api_url: 'webhook-url'
        channel: '#alerts'
        username: 'Alertmanager'
        icon_emoji: ':fire:'
```

### Microsoft Teams

```yaml
receivers:
  - name: 'teams'
    webhook_configs:
      - url: 'teams-webhook-url'
        send_resolved: true
```

### OpsGenie

Plateforme d'incident management.

```yaml
receivers:
  - name: 'opsgenie'
    opsgenie_configs:
      - api_key: 'votre-api-key'
        teams: 'backend-team'
```

## Métriques d'Alertmanager

Alertmanager expose ses propres métriques que Prometheus peut scraper :

- `alertmanager_notifications_total` : Nombre total de notifications envoyées
- `alertmanager_notifications_failed_total` : Notifications échouées
- `alertmanager_alerts` : Nombre d'alertes actives
- `alertmanager_silences` : Nombre de silences actifs

Ces métriques vous permettent de surveiller la santé de votre système d'alerting.

## Conclusion

Prometheus Alertmanager est un composant puissant et flexible qui transforme vos métriques en actions concrètes. Les concepts clés à retenir :

- **Receivers** : Où envoyer les notifications
- **Routing** : Comment diriger les alertes
- **Grouping** : Regrouper pour éviter le spam
- **Inhibitions** : Supprimer les alertes redondantes
- **Silences** : Désactiver temporairement pendant les maintenances

Avec une configuration bien pensée, Alertmanager devient votre allié pour maintenir vos services en bonne santé tout en préservant votre tranquillité d'esprit.

## Points Clés à Retenir

✓ Alertmanager gère les notifications envoyées par Prometheus

✓ Les receivers définissent les canaux de notification (email, Slack, etc.)

✓ Le routing dirige les alertes vers les bons destinataires

✓ Le grouping évite le spam en regroupant les alertes similaires

✓ Les inhibitions suppriment les alertes redondantes

✓ Les silences désactivent temporairement certaines alertes

✓ Les templates permettent de personnaliser les messages

✓ Testez régulièrement votre configuration d'alerting

✓ Documentez vos alertes avec des runbooks

✓ Surveillez Alertmanager lui-même avec des métriques

---

**Prochaine section** : 14.3 Configuration des règles d'alerte - Nous apprendrons à créer des règles d'alerte dans Prometheus.

⏭️ [Configuration des règles d'alerte](/14-alerting-et-notifications/03-configuration-des-regles-dalerte.md)
