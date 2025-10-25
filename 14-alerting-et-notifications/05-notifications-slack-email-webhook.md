🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 14.5 Notifications (Slack, email, webhook)

## Introduction

Les **receivers** (récepteurs) dans Alertmanager sont les canaux par lesquels vous recevez vos notifications. Chaque receiver représente une destination spécifique : un canal Slack, une adresse email, un webhook HTTP, ou une intégration avec un service tiers.

Dans cette section, nous allons explorer en détail comment configurer les trois types de notifications les plus courants :
- **Email** : Le classique, fiable et universel
- **Slack** : Communication d'équipe en temps réel
- **Webhook** : Intégration personnalisée avec vos propres systèmes

Chaque type a ses avantages et ses cas d'usage. Souvent, vous utiliserez plusieurs types en parallèle pour différents niveaux d'alertes.

## Vue d'Ensemble des Receivers

### Receivers Disponibles

Alertmanager supporte nativement de nombreux types de receivers :

**Communication** :
- Email (SMTP)
- Slack
- Microsoft Teams
- Discord
- Telegram

**Incident Management** :
- PagerDuty
- OpsGenie
- VictorOps

**Générique** :
- Webhook (HTTP/HTTPS)

**Autres** :
- WeChat
- Pushover
- SNS (AWS)

### Anatomie d'un Receiver

Un receiver est défini dans la section `receivers` de la configuration Alertmanager :

```yaml
receivers:
  - name: 'identifiant-unique'
    type_configs:
      - paramètre1: valeur1
        paramètre2: valeur2
```

**name** : Identifiant unique du receiver (utilisé dans le routing)

**type_configs** : Configuration spécifique au type (email_configs, slack_configs, etc.)

### Configuration de Base

```yaml
receivers:
  - name: 'team-email'
    email_configs:
      - to: 'team@example.com'

  - name: 'team-slack'
    slack_configs:
      - channel: '#alerts'
        api_url: 'webhook-url'

  - name: 'custom-webhook'
    webhook_configs:
      - url: 'http://mon-service/alerts'
```

## Notifications par Email

L'email est le canal de notification le plus universel. Tout le monde a une adresse email, et les emails peuvent contenir des informations riches et détaillées.

### Prérequis : Configuration SMTP Globale

Avant de configurer les receivers email, vous devez définir les paramètres SMTP globaux :

```yaml
global:
  smtp_smarthost: 'smtp.gmail.com:587'
  smtp_from: 'alertmanager@example.com'
  smtp_auth_username: 'alertmanager@example.com'
  smtp_auth_password: 'votre-mot-de-passe-application'
  smtp_require_tls: true
```

**Explication** :
- `smtp_smarthost` : Serveur SMTP et port (587 pour TLS, 465 pour SSL)
- `smtp_from` : Adresse email de l'expéditeur
- `smtp_auth_username` : Identifiant SMTP (souvent identique à smtp_from)
- `smtp_auth_password` : Mot de passe SMTP
- `smtp_require_tls` : Utiliser TLS (recommandé pour la sécurité)

### Configuration avec Gmail

Gmail nécessite un "mot de passe d'application" spécifique :

**Étapes pour créer un mot de passe d'application** :
1. Aller sur votre compte Google
2. Sécurité → Validation en 2 étapes (l'activer si pas déjà fait)
3. Mots de passe des applications
4. Créer un nouveau mot de passe pour "Alertmanager"
5. Utiliser ce mot de passe dans la configuration

```yaml
global:
  smtp_smarthost: 'smtp.gmail.com:587'
  smtp_from: 'votre-email@gmail.com'
  smtp_auth_username: 'votre-email@gmail.com'
  smtp_auth_password: 'xxxx xxxx xxxx xxxx'  # Mot de passe d'application
  smtp_require_tls: true
```

### Configuration avec d'Autres Fournisseurs

**Office 365 / Outlook** :
```yaml
global:
  smtp_smarthost: 'smtp.office365.com:587'
  smtp_from: 'alertmanager@votredomaine.com'
  smtp_auth_username: 'alertmanager@votredomaine.com'
  smtp_auth_password: 'votre-mot-de-passe'
  smtp_require_tls: true
```

**SendGrid** :
```yaml
global:
  smtp_smarthost: 'smtp.sendgrid.net:587'
  smtp_from: 'alertmanager@votredomaine.com'
  smtp_auth_username: 'apikey'
  smtp_auth_password: 'votre-api-key-sendgrid'
  smtp_require_tls: true
```

**Serveur SMTP Local** :
```yaml
global:
  smtp_smarthost: 'localhost:25'
  smtp_from: 'alertmanager@example.com'
  # Pas d'authentification nécessaire pour un serveur local
```

### Receiver Email Simple

Configuration minimale d'un receiver email :

```yaml
receivers:
  - name: 'team-email'
    email_configs:
      - to: 'team@example.com'
```

**Utilisation** :
```yaml
route:
  receiver: 'team-email'
```

### Receiver Email avec Plusieurs Destinataires

```yaml
receivers:
  - name: 'multiple-emails'
    email_configs:
      - to: 'admin@example.com, team@example.com, oncall@example.com'
```

Ou avec plusieurs configurations :

```yaml
receivers:
  - name: 'multiple-emails'
    email_configs:
      - to: 'admin@example.com'
      - to: 'team@example.com'
      - to: 'oncall@example.com'
```

### Personnalisation des Emails

Vous pouvez personnaliser l'objet, le corps et les headers :

```yaml
receivers:
  - name: 'custom-email'
    email_configs:
      - to: 'team@example.com'
        send_resolved: true
        headers:
          Subject: '[{{ .Status | toUpper }}] {{ .GroupLabels.alertname }}'
          From: 'Alertmanager Kubernetes <alertmanager@example.com>'
        html: |
          <!DOCTYPE html>
          <html>
            <head>
              <style>
                body { font-family: Arial, sans-serif; }
                .critical { background-color: #ff4444; color: white; }
                .warning { background-color: #ffaa00; color: white; }
                table { border-collapse: collapse; width: 100%; }
                th, td { border: 1px solid #ddd; padding: 8px; text-align: left; }
                th { background-color: #4CAF50; color: white; }
              </style>
            </head>
            <body>
              <h2>Alertes Kubernetes - {{ .Status | toUpper }}</h2>
              <p><strong>Nombre d'alertes:</strong> {{ .Alerts | len }}</p>

              <table>
                <tr>
                  <th>Alerte</th>
                  <th>Sévérité</th>
                  <th>Description</th>
                  <th>Début</th>
                </tr>
                {{ range .Alerts }}
                <tr class="{{ .Labels.severity }}">
                  <td>{{ .Labels.alertname }}</td>
                  <td>{{ .Labels.severity }}</td>
                  <td>{{ .Annotations.description }}</td>
                  <td>{{ .StartsAt.Format "2006-01-02 15:04:05" }}</td>
                </tr>
                {{ end }}
              </table>

              <p><a href="{{ .ExternalURL }}">Voir dans Alertmanager</a></p>
            </body>
          </html>
```

**Explication** :
- `send_resolved` : Envoie aussi un email quand l'alerte est résolue
- `headers` : Personnalise les en-têtes de l'email (Subject, From, etc.)
- `html` : Corps de l'email au format HTML
- Templates Go pour insérer des valeurs dynamiques

### Email en Texte Brut

Si vous préférez le texte brut au HTML :

```yaml
receivers:
  - name: 'text-email'
    email_configs:
      - to: 'team@example.com'
        headers:
          Subject: '[{{ .Status }}] {{ .GroupLabels.alertname }}'
        text: |
          Alerte Kubernetes

          Statut: {{ .Status | toUpper }}
          Nombre d'alertes: {{ .Alerts | len }}

          Détails:
          {{ range .Alerts }}
          - Alerte: {{ .Labels.alertname }}
            Sévérité: {{ .Labels.severity }}
            Description: {{ .Annotations.description }}
            Début: {{ .StartsAt }}
          {{ end }}

          Lien: {{ .ExternalURL }}
```

### Configuration Complète Email

```yaml
global:
  smtp_smarthost: 'smtp.gmail.com:587'
  smtp_from: 'alertmanager@example.com'
  smtp_auth_username: 'alertmanager@example.com'
  smtp_auth_password: 'xxxx-xxxx-xxxx-xxxx'
  smtp_require_tls: true

receivers:
  - name: 'production-critical'
    email_configs:
      - to: 'oncall@example.com'
        send_resolved: true
        headers:
          Subject: '[CRITICAL] {{ .GroupLabels.alertname }} - PRODUCTION'
          Priority: 'urgent'
        html: |
          <h1 style="color: red;">ALERTE CRITIQUE PRODUCTION</h1>
          <p>Action immédiate requise</p>
          {{ range .Alerts }}
          <div style="border: 2px solid red; padding: 10px; margin: 10px;">
            <h3>{{ .Labels.alertname }}</h3>
            <p>{{ .Annotations.description }}</p>
            <p><strong>Runbook:</strong> <a href="{{ .Annotations.runbook_url }}">{{ .Annotations.runbook_url }}</a></p>
          </div>
          {{ end }}
```

## Notifications Slack

Slack est devenu le standard pour la communication d'équipe. Les notifications Slack sont instantanées, visibles par toute l'équipe, et peuvent être enrichies avec des couleurs et des boutons.

### Prérequis : Créer un Webhook Slack

Avant de configurer Alertmanager, vous devez créer un Webhook Incoming dans Slack :

**Étapes** :
1. Aller sur https://api.slack.com/apps
2. Créer une nouvelle app ou utiliser une existante
3. Activer **Incoming Webhooks**
4. Cliquer sur **Add New Webhook to Workspace**
5. Choisir le canal de destination
6. Copier l'URL du webhook (format : `https://hooks.slack.com/services/XXX/YYY/ZZZ`)

### Configuration Slack Simple

```yaml
receivers:
  - name: 'slack-alerts'
    slack_configs:
      - api_url: 'https://hooks.slack.com/services/T00000000/B00000000/XXXXXXXXXXXXXXXXXXXX'
        channel: '#alerts'
```

**Note** : Vous pouvez aussi définir l'URL globalement :

```yaml
global:
  slack_api_url: 'https://hooks.slack.com/services/T00000000/B00000000/XXXXXXXXXXXXXXXXXXXX'

receivers:
  - name: 'slack-alerts'
    slack_configs:
      - channel: '#alerts'
```

### Personnalisation des Messages Slack

```yaml
receivers:
  - name: 'slack-custom'
    slack_configs:
      - api_url: 'webhook-url'
        channel: '#alerts'
        send_resolved: true
        username: 'Alertmanager'
        icon_emoji: ':fire:'
        title: '{{ .Status | toUpper }}: {{ .GroupLabels.alertname }}'
        text: |
          {{ range .Alerts }}
          *Alerte:* {{ .Labels.alertname }}
          *Sévérité:* {{ .Labels.severity }}
          *Description:* {{ .Annotations.description }}
          *Namespace:* {{ .Labels.namespace }}
          *Pod:* {{ .Labels.pod }}
          {{ end }}
        color: '{{ if eq .Status "firing" }}danger{{ else }}good{{ end }}'
```

**Explication** :
- `send_resolved` : Envoie aussi un message quand l'alerte est résolue
- `username` : Nom affiché dans Slack
- `icon_emoji` : Emoji à afficher (`:fire:`, `:warning:`, `:white_check_mark:`)
- `title` : Titre du message
- `text` : Corps du message (supporte le markdown Slack)
- `color` : Couleur de la barre latérale (`danger`=rouge, `warning`=orange, `good`=vert)

### Messages Slack avec Formatting Riche

Slack supporte un formatting markdown spécifique :

```yaml
receivers:
  - name: 'slack-rich'
    slack_configs:
      - channel: '#alerts'
        title: '{{ .Status | toUpper }}: {{ .GroupLabels.alertname }}'
        text: |
          *Résumé:* {{ len .Alerts }} alerte(s) en cours
          *Cluster:* {{ .GroupLabels.cluster }}

          {{ range .Alerts }}
          ---
          *{{ .Labels.alertname }}* ({{ .Labels.severity }})
          • *Description:* {{ .Annotations.description }}
          • *Namespace:* `{{ .Labels.namespace }}`
          • *Pod:* `{{ .Labels.pod }}`
          • *Depuis:* {{ .StartsAt.Format "15:04:05" }}
          {{ if .Annotations.runbook_url }}
          • *Runbook:* <{{ .Annotations.runbook_url }}|Voir la documentation>
          {{ end }}
          {{ end }}

          <{{ .ExternalURL }}|Voir dans Alertmanager>
        color: '{{ if eq .Status "firing" }}danger{{ else }}good{{ end }}'
```

**Formatting Slack** :
- `*texte*` : Gras
- `_texte_` : Italique
- `` `code` `` : Code inline
- `<url|texte>` : Lien cliquable

### Slack avec Plusieurs Canaux

Envoyez les alertes vers différents canaux selon le contexte :

```yaml
receivers:
  - name: 'slack-critical'
    slack_configs:
      - channel: '#critical-alerts'
        title: '🚨 ALERTE CRITIQUE'
        color: 'danger'

  - name: 'slack-warnings'
    slack_configs:
      - channel: '#warnings'
        title: '⚠️ Warning'
        color: 'warning'

  - name: 'slack-info'
    slack_configs:
      - channel: '#info-alerts'
        title: 'ℹ️ Info'
        color: 'good'
```

### Slack avec Messages Threads

Pour regrouper les mises à jour d'une même alerte :

```yaml
receivers:
  - name: 'slack-threaded'
    slack_configs:
      - channel: '#alerts'
        title: '{{ .GroupLabels.alertname }}'
        # Les mises à jour du même groupe seront en thread
```

**Note** : Le threading est automatique - les mises à jour d'un même groupe d'alertes apparaissent comme réponses au message initial.

### Configuration Slack Complète

```yaml
receivers:
  - name: 'slack-production'
    slack_configs:
      - api_url: 'https://hooks.slack.com/services/XXX/YYY/ZZZ'
        channel: '#prod-alerts'
        send_resolved: true
        username: 'Alertmanager Production'
        icon_emoji: '{{ if eq .Status "firing" }}:fire:{{ else }}:white_check_mark:{{ end }}'
        title: |
          {{ if eq .Status "firing" }}🚨{{ else }}✅{{ end }}
          {{ .Status | toUpper }}: {{ .GroupLabels.alertname }}
        title_link: '{{ .ExternalURL }}'
        text: |
          *Cluster:* {{ .GroupLabels.cluster | toUpper }}
          *Environnement:* {{ .CommonLabels.environment }}
          *Nombre d'alertes:* {{ .Alerts | len }}

          {{ if eq .Status "firing" }}
          {{ range .Alerts }}
          ---
          *🔴 {{ .Labels.alertname }}* | Sévérité: *{{ .Labels.severity }}*
          {{ if .Annotations.summary }}📝 {{ .Annotations.summary }}{{ end }}
          {{ if .Annotations.description }}
          📋 {{ .Annotations.description }}
          {{ end }}

          *Détails:*
          • Namespace: `{{ .Labels.namespace }}`
          {{ if .Labels.pod }}• Pod: `{{ .Labels.pod }}`{{ end }}
          {{ if .Labels.instance }}• Instance: `{{ .Labels.instance }}`{{ end }}
          • Début: {{ .StartsAt.Format "2006-01-02 15:04:05" }}

          {{ if .Annotations.runbook_url }}
          📖 <{{ .Annotations.runbook_url }}|Consulter le runbook>
          {{ end }}
          {{ if .Annotations.dashboard_url }}
          📊 <{{ .Annotations.dashboard_url }}|Voir le dashboard>
          {{ end }}
          {{ end }}
          {{ else }}
          ✅ *Toutes les alertes ont été résolues*
          {{ end }}
        color: '{{ if eq .Status "firing" }}danger{{ else }}good{{ end }}'
        footer: 'Alertmanager Kubernetes'
```

## Notifications par Webhook

Les webhooks permettent d'envoyer les alertes vers vos propres systèmes ou services personnalisés via HTTP.

### Configuration Webhook Simple

```yaml
receivers:
  - name: 'custom-webhook'
    webhook_configs:
      - url: 'http://mon-service.example.com:8080/alerts'
        send_resolved: true
```

**Explication** :
- `url` : URL de votre endpoint qui recevra les alertes
- `send_resolved` : Envoie aussi les notifications de résolution

### Format du Payload Webhook

Alertmanager envoie un POST JSON avec cette structure :

```json
{
  "version": "4",
  "groupKey": "{}:{alertname=\"TestAlert\"}",
  "truncatedAlerts": 0,
  "status": "firing",
  "receiver": "custom-webhook",
  "groupLabels": {
    "alertname": "TestAlert"
  },
  "commonLabels": {
    "alertname": "TestAlert",
    "severity": "critical"
  },
  "commonAnnotations": {
    "description": "Test alert"
  },
  "externalURL": "http://alertmanager:9093",
  "alerts": [
    {
      "status": "firing",
      "labels": {
        "alertname": "TestAlert",
        "severity": "critical",
        "instance": "server1"
      },
      "annotations": {
        "description": "Test alert",
        "summary": "This is a test"
      },
      "startsAt": "2025-01-15T10:00:00Z",
      "endsAt": "0001-01-01T00:00:00Z",
      "generatorURL": "http://prometheus:9090/graph?g0.expr=..."
    }
  ]
}
```

### Webhook avec Headers Personnalisés

Pour ajouter des headers HTTP (authentification, etc.) :

```yaml
receivers:
  - name: 'authenticated-webhook'
    webhook_configs:
      - url: 'https://api.example.com/alerts'
        send_resolved: true
        http_config:
          bearer_token: 'votre-token-secret'
```

Ou avec Basic Auth :

```yaml
receivers:
  - name: 'basic-auth-webhook'
    webhook_configs:
      - url: 'https://api.example.com/alerts'
        http_config:
          basic_auth:
            username: 'alertmanager'
            password: 'secret-password'
```

Ou avec headers personnalisés :

```yaml
receivers:
  - name: 'custom-headers-webhook'
    webhook_configs:
      - url: 'https://api.example.com/alerts'
        http_config:
          headers:
            X-API-Key: 'your-api-key'
            X-Custom-Header: 'custom-value'
```

### Webhook avec TLS

Pour des connexions HTTPS sécurisées :

```yaml
receivers:
  - name: 'secure-webhook'
    webhook_configs:
      - url: 'https://secure-api.example.com/alerts'
        http_config:
          tls_config:
            ca_file: /etc/alertmanager/ca.crt
            cert_file: /etc/alertmanager/client.crt
            key_file: /etc/alertmanager/client.key
            insecure_skip_verify: false
```

### Exemple de Serveur Webhook en Python

Voici un exemple simple de serveur qui reçoit les webhooks :

```python
from flask import Flask, request, jsonify
import json
from datetime import datetime

app = Flask(__name__)

@app.route('/alerts', methods=['POST'])
def receive_alerts():
    data = request.get_json()

    print(f"Received webhook at {datetime.now()}")
    print(f"Status: {data['status']}")
    print(f"Number of alerts: {len(data['alerts'])}")

    for alert in data['alerts']:
        print(f"\nAlert: {alert['labels']['alertname']}")
        print(f"  Severity: {alert['labels'].get('severity', 'unknown')}")
        print(f"  Description: {alert['annotations'].get('description', 'N/A')}")
        print(f"  Status: {alert['status']}")

    # Ici vous pouvez traiter les alertes comme vous voulez:
    # - Les enregistrer en base de données
    # - Les envoyer vers d'autres systèmes
    # - Créer des tickets automatiquement
    # - Déclencher des actions de remediation

    return jsonify({"status": "success"}), 200

if __name__ == '__main__':
    app.run(host='0.0.0.0', port=8080)
```

### Exemple de Serveur Webhook en Go

```go
package main

import (
    "encoding/json"
    "fmt"
    "log"
    "net/http"
    "time"
)

type WebhookData struct {
    Version  string  `json:"version"`
    Status   string  `json:"status"`
    Alerts   []Alert `json:"alerts"`
}

type Alert struct {
    Status      string            `json:"status"`
    Labels      map[string]string `json:"labels"`
    Annotations map[string]string `json:"annotations"`
    StartsAt    time.Time         `json:"startsAt"`
}

func alertHandler(w http.ResponseWriter, r *http.Request) {
    var data WebhookData

    if err := json.NewDecoder(r.Body).Decode(&data); err != nil {
        http.Error(w, err.Error(), http.StatusBadRequest)
        return
    }

    log.Printf("Received %d alerts with status: %s\n", len(data.Alerts), data.Status)

    for _, alert := range data.Alerts {
        log.Printf("Alert: %s | Severity: %s | Description: %s\n",
            alert.Labels["alertname"],
            alert.Labels["severity"],
            alert.Annotations["description"])
    }

    w.WriteHeader(http.StatusOK)
    fmt.Fprintf(w, "OK")
}

func main() {
    http.HandleFunc("/alerts", alertHandler)
    log.Println("Webhook server listening on :8080")
    log.Fatal(http.ListenAndServe(":8080", nil))
}
```

### Webhook vers des Services Tiers

**Discord** :
```yaml
receivers:
  - name: 'discord'
    webhook_configs:
      - url: 'https://discord.com/api/webhooks/YOUR_WEBHOOK_ID/YOUR_WEBHOOK_TOKEN'
        send_resolved: true
```

**Microsoft Teams** (nécessite un format spécial) :
```yaml
receivers:
  - name: 'teams'
    webhook_configs:
      - url: 'https://outlook.office.com/webhook/YOUR_WEBHOOK_URL'
```

**Mattermost** :
```yaml
receivers:
  - name: 'mattermost'
    webhook_configs:
      - url: 'https://mattermost.example.com/hooks/YOUR_WEBHOOK_ID'
```

## Intégrations avec Services d'Incident Management

### PagerDuty

PagerDuty est un service d'astreinte et de gestion d'incidents populaire.

```yaml
global:
  pagerduty_url: 'https://events.pagerduty.com/v2/enqueue'

receivers:
  - name: 'pagerduty-critical'
    pagerduty_configs:
      - service_key: 'votre-integration-key'
        routing_key: 'votre-routing-key'
        description: '{{ .GroupLabels.alertname }}'
        severity: '{{ if eq .GroupLabels.severity "critical" }}critical{{ else }}warning{{ end }}'
        details:
          firing: '{{ .Alerts.Firing | len }}'
          resolved: '{{ .Alerts.Resolved | len }}'
          num_alerts: '{{ .Alerts | len }}'
        links:
          - href: '{{ .ExternalURL }}'
            text: 'View in Alertmanager'
```

**Obtenir les clés** :
1. Dans PagerDuty, aller dans **Services**
2. Créer ou sélectionner un service
3. Onglet **Integrations**
4. Ajouter une intégration **Prometheus** ou **Generic Webhook**
5. Copier l'**Integration Key**

### OpsGenie

OpsGenie est un autre service de gestion d'incidents.

```yaml
receivers:
  - name: 'opsgenie'
    opsgenie_configs:
      - api_key: 'votre-api-key'
        api_url: 'https://api.opsgenie.com/'
        message: '{{ .GroupLabels.alertname }}'
        description: '{{ range .Alerts }}{{ .Annotations.description }}{{ end }}'
        priority: '{{ if eq .GroupLabels.severity "critical" }}P1{{ else }}P3{{ end }}'
        teams: 'backend-team,infra-team'
        tags: 'kubernetes,{{ .GroupLabels.cluster }}'
```

### VictorOps (Splunk On-Call)

```yaml
receivers:
  - name: 'victorops'
    victorops_configs:
      - api_key: 'votre-api-key'
        routing_key: 'your-routing-key'
        message_type: '{{ if eq .Status "firing" }}CRITICAL{{ else }}RECOVERY{{ end }}'
        entity_display_name: '{{ .GroupLabels.alertname }}'
        state_message: '{{ range .Alerts }}{{ .Annotations.description }}{{ end }}'
```

## Templates Avancés

### Variables Disponibles dans les Templates

**Variables de base** :
- `{{ .Status }}` : État du groupe ("firing" ou "resolved")
- `{{ .Alerts }}` : Liste de toutes les alertes
- `{{ .Alerts.Firing }}` : Alertes en cours
- `{{ .Alerts.Resolved }}` : Alertes résolues
- `{{ .GroupLabels }}` : Labels de grouping
- `{{ .CommonLabels }}` : Labels communs à toutes les alertes
- `{{ .CommonAnnotations }}` : Annotations communes
- `{{ .ExternalURL }}` : URL d'Alertmanager

**Pour chaque alerte** :
- `{{ .Status }}` : État de l'alerte
- `{{ .Labels.xxx }}` : Accès aux labels
- `{{ .Annotations.xxx }}` : Accès aux annotations
- `{{ .StartsAt }}` : Date de début
- `{{ .EndsAt }}` : Date de fin (si résolue)
- `{{ .GeneratorURL }}` : URL vers Prometheus

### Fonctions de Template Utiles

**Formatage de texte** :
```yaml
{{ .Status | toUpper }}        # FIRING
{{ .Status | toLower }}        # firing
{{ .Status | title }}          # Firing
```

**Conditions** :
```yaml
{{ if eq .Status "firing" }}
  Alerte en cours!
{{ else }}
  Alerte résolue
{{ end }}
```

**Boucles** :
```yaml
{{ range .Alerts }}
  - {{ .Labels.alertname }}
{{ end }}
```

**Comptage** :
```yaml
{{ .Alerts | len }}            # Nombre d'alertes
{{ .Alerts.Firing | len }}     # Nombre d'alertes actives
```

**Formatage de nombres** :
```yaml
{{ $value | humanize }}        # 1234567 → 1.234567M
{{ $value | humanizePercentage }} # 0.85 → 85%
{{ $value | humanizeDuration }}   # 3665 → 1h1m5s
```

**Formatage de dates** :
```yaml
{{ .StartsAt.Format "2006-01-02 15:04:05" }}
{{ .StartsAt.Format "15:04" }}
```

### Template Réutilisable

Vous pouvez définir des templates réutilisables :

```yaml
templates:
  - '/etc/alertmanager/templates/*.tmpl'

receivers:
  - name: 'slack-with-template'
    slack_configs:
      - channel: '#alerts'
        title: '{{ template "slack.title" . }}'
        text: '{{ template "slack.text" . }}'
```

**Fichier /etc/alertmanager/templates/slack.tmpl** :

```go
{{ define "slack.title" }}
[{{ .Status | toUpper }}{{ if eq .Status "firing" }}:{{ .Alerts.Firing | len }}{{ end }}]
{{ .GroupLabels.alertname }}
{{ end }}

{{ define "slack.text" }}
{{ if eq .Status "firing" }}
🔴 *Alertes actives*
{{ range .Alerts.Firing }}
• *{{ .Labels.alertname }}* ({{ .Labels.severity }})
  {{ .Annotations.description }}
  Depuis: {{ .StartsAt.Format "15:04:05" }}
{{ end }}
{{ else }}
✅ *Alertes résolues*
{{ range .Alerts.Resolved }}
• {{ .Labels.alertname }}
  Résolue à: {{ .EndsAt.Format "15:04:05" }}
{{ end }}
{{ end }}
{{ end }}
```

## Combinaisons de Receivers

### Receiver avec Plusieurs Canaux

Un receiver peut utiliser plusieurs types de notifications :

```yaml
receivers:
  - name: 'multi-channel'
    email_configs:
      - to: 'team@example.com'
    slack_configs:
      - channel: '#alerts'
    webhook_configs:
      - url: 'http://logger:8080/alerts'
```

Toutes ces notifications seront envoyées pour chaque alerte routée vers ce receiver.

### Escalade Progressive

Utilisez plusieurs receivers avec différents délais :

```yaml
route:
  receiver: 'first-notification'
  routes:
    - match:
        severity: 'critical'
      receiver: 'immediate-pagerduty'
      repeat_interval: 5m

      routes:
        # Si pas d'acquittement après 15 minutes, escalader
        - match: {}
          receiver: 'escalated-oncall-manager'
          group_wait: 15m
```

**Note** : L'escalade automatique nécessite généralement des outils externes ou des intégrations PagerDuty/OpsGenie qui gèrent cela nativement.

### Notification Conditionnelle

Envoyez vers différents canaux selon les conditions :

```yaml
receivers:
  - name: 'conditional'
    slack_configs:
      # Slack pour tous
      - channel: '#alerts'

    email_configs:
      # Email uniquement pour critical
      - to: 'oncall@example.com'
        send_resolved: true
```

Ou utilisez le routing pour plus de contrôle :

```yaml
route:
  routes:
    - match:
        severity: 'critical'
      receiver: 'critical-all-channels'

    - match:
        severity: 'warning'
      receiver: 'warning-slack-only'

receivers:
  - name: 'critical-all-channels'
    pagerduty_configs:
      - service_key: 'xxx'
    slack_configs:
      - channel: '#alerts'
    email_configs:
      - to: 'oncall@example.com'

  - name: 'warning-slack-only'
    slack_configs:
      - channel: '#alerts'
```

## Troubleshooting

### Problème : Les Emails ne Sont Pas Envoyés

**Diagnostic** :

1. Vérifier les logs d'Alertmanager :
```bash
microk8s kubectl logs -n monitoring alertmanager-0 | grep -i email
```

2. Erreurs communes :
```
level=error msg="Notify failed" err="dial tcp: lookup smtp.gmail.com: no such host"
→ Problème DNS ou connectivité réseau

level=error msg="Notify failed" err="535 5.7.8 Username and Password not accepted"
→ Identifiants SMTP incorrects

level=error msg="Notify failed" err="tls: first record does not look like a TLS handshake"
→ Mauvaise configuration TLS (vérifier le port)
```

**Solutions** :

✓ Vérifier les identifiants SMTP
✓ Utiliser un mot de passe d'application (Gmail)
✓ Vérifier le port (587 pour TLS, 465 pour SSL, 25 pour non-sécurisé)
✓ Tester la connectivité : `telnet smtp.gmail.com 587`
✓ Vérifier les pare-feu

### Problème : Les Messages Slack ne Sont Pas Envoyés

**Diagnostic** :

```bash
microk8s kubectl logs -n monitoring alertmanager-0 | grep -i slack
```

Erreurs communes :
```
level=error msg="Notify failed" receiver=slack err="webhook error: HTTP 404"
→ URL du webhook invalide

level=error msg="Notify failed" receiver=slack err="invalid_payload"
→ Problème dans le template du message
```

**Solutions** :

✓ Vérifier que l'URL du webhook est correcte
✓ Vérifier que le canal existe (avec le # : `#alerts`)
✓ Tester le webhook manuellement :
```bash
curl -X POST 'votre-webhook-url' \
  -H 'Content-Type: application/json' \
  -d '{"text": "Test message"}'
```
✓ Vérifier que l'app Slack a les permissions nécessaires

### Problème : Les Webhooks Échouent

**Diagnostic** :

```bash
microk8s kubectl logs -n monitoring alertmanager-0 | grep webhook
```

Erreurs communes :
```
level=error msg="Notify failed" receiver=webhook err="connection refused"
→ Service webhook inaccessible

level=error msg="Notify failed" receiver=webhook err="context deadline exceeded"
→ Timeout - le service est trop lent

level=error msg="Notify failed" receiver=webhook err="HTTP 401"
→ Problème d'authentification
```

**Solutions** :

✓ Vérifier que le service webhook est accessible depuis le pod Alertmanager
✓ Tester la connectivité :
```bash
microk8s kubectl exec -n monitoring alertmanager-0 -- wget -O- http://mon-service:8080/health
```
✓ Vérifier les credentials et headers d'authentification
✓ Augmenter le timeout si nécessaire
✓ Vérifier les logs du service webhook

### Problème : Trop de Notifications

**Symptômes** :
- Spam de notifications
- Notifications répétées trop souvent
- Canal Slack inondé

**Solutions** :

✓ Augmenter `group_interval` et `repeat_interval`
```yaml
group_interval: 10m
repeat_interval: 4h
```

✓ Améliorer le grouping
```yaml
group_by: ['alertname', 'namespace', 'cluster']
```

✓ Ajouter des inhibitions pour supprimer les alertes redondantes

✓ Revoir les seuils de vos alertes dans Prometheus

### Tester les Notifications

**Envoyer une alerte de test** :

```bash
curl -X POST http://localhost:9093/api/v2/alerts \
  -H 'Content-Type: application/json' \
  -d '[
    {
      "labels": {
        "alertname": "TestNotification",
        "severity": "warning",
        "instance": "test-server",
        "team": "platform"
      },
      "annotations": {
        "summary": "Ceci est un test",
        "description": "Test de notification Alertmanager"
      },
      "startsAt": "2025-01-15T10:00:00Z",
      "endsAt": "0001-01-01T00:00:00Z"
    }
  ]'
```

Vérifiez que vous recevez la notification sur tous vos canaux configurés.

## Bonnes Pratiques

### 1. Sécuriser les Credentials

Ne stockez jamais les mots de passe en clair dans vos fichiers de configuration. Utilisez des Secrets Kubernetes :

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: alertmanager-secrets
  namespace: monitoring
type: Opaque
stringData:
  smtp-password: "votre-mot-de-passe"
  slack-webhook: "https://hooks.slack.com/services/XXX/YYY/ZZZ"
  pagerduty-key: "votre-integration-key"
```

Référencez-les dans votre ConfigMap :

```yaml
global:
  smtp_auth_password: ${SMTP_PASSWORD}
```

### 2. Tester Régulièrement

Configurez des alertes de test automatiques pour vérifier que vos notifications fonctionnent :

```yaml
# Dans Prometheus
- alert: WeeklyNotificationTest
  expr: day_of_week() == 1 and hour() == 9
  for: 0m
  labels:
    severity: info
  annotations:
    summary: "Test hebdomadaire des notifications"
    description: "Ceci est un test automatique"
```

### 3. Documenter les Canaux

Maintenez une documentation claire de vos receivers :

```yaml
receivers:
  # Production Critical - PagerDuty 24/7
  # Propriétaire: Équipe SRE
  # Contact: sre@example.com
  # Rotation: https://pagerduty.com/schedules/xxx
  - name: 'prod-critical-pagerduty'
    pagerduty_configs:
      - service_key: 'xxx'
```

### 4. Utilisez des Templates Réutilisables

Créez des templates partagés pour maintenir la cohérence :

```
/alertmanager/
├── config.yml
└── templates/
    ├── email.tmpl
    ├── slack.tmpl
    └── common.tmpl
```

### 5. Monitorer les Notifications Elles-mêmes

Créez des alertes pour surveiller votre système de notifications :

```yaml
- alert: NotificationFailureRate
  expr: |
    rate(alertmanager_notifications_failed_total[5m])
    /
    rate(alertmanager_notifications_total[5m]) > 0.1
  for: 10m
  labels:
    severity: warning
  annotations:
    summary: "Taux d'échec des notifications élevé"
```

### 6. Adaptez le Canal à la Sévérité

- **Critical** : PagerDuty/OpsGenie + Slack + Email
- **Warning** : Slack + Email
- **Info** : Email uniquement (digest quotidien)

### 7. Respectez les Limites de Rate

Slack, email et autres services ont des limites de débit. Configurez vos grouping/repeat_interval en conséquence pour éviter d'être bloqué.

## Conclusion

Les notifications sont le pont entre vos alertes et les actions humaines. En configurant correctement vos receivers, vous assurez que :

- Les bonnes personnes sont notifiées
- Au bon moment
- Avec les bonnes informations
- Via les canaux appropriés

Une configuration bien pensée des notifications transforme le monitoring passif en système proactif qui protège vos services tout en respectant la tranquillité de vos équipes.

## Points Clés à Retenir

✓ Email : universel et riche, nécessite configuration SMTP

✓ Slack : instantané et collaboratif, utilise des webhooks

✓ Webhook : flexible pour intégrations personnalisées

✓ Sécurisez vos credentials avec des Secrets Kubernetes

✓ Personnalisez les messages avec des templates Go

✓ Combinez plusieurs canaux pour les alertes critiques

✓ Testez régulièrement vos notifications

✓ Adaptez le canal à la sévérité de l'alerte

✓ Surveillez les échecs de notification

✓ Documentez vos receivers et leurs propriétaires

---

**Prochaine section** : 14.6 Silences et inhibitions - Nous verrons comment gérer intelligemment les alertes pendant les maintenances et éviter les notifications redondantes.

⏭️ [Silences et inhibitions](/14-alerting-et-notifications/06-silences-et-inhibitions.md)
