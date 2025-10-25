🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 16.8 Audit Logging

## Introduction

L'audit logging dans Kubernetes enregistre toutes les actions effectuées dans le cluster : qui a fait quoi, quand, et quel a été le résultat. C'est un élément crucial de la sécurité qui permet de détecter les comportements anormaux, enquêter sur les incidents et garantir la conformité.

**Analogie simple :** Imaginez une banque sans caméras de surveillance ni registre des entrées. En cas de problème, impossible de savoir ce qui s'est passé. L'audit logging, c'est comme avoir un système de vidéosurveillance et un registre détaillé de toutes les actions dans votre cluster Kubernetes.

## Pourquoi l'Audit est Crucial ?

### Scénarios Nécessitant l'Audit

#### 1. Incident de Sécurité

```
🔴 Alerte : Pod supprimé en production !
         │
         ▼
Questions sans audit :
  • Qui a supprimé le pod ?
  • Quand exactement ?
  • Était-ce intentionnel ou une erreur ?
  • D'où venait la requête ?
  • Quels autres pods ont été affectés ?
         │
         ▼
❌ Impossible de répondre → Investigation bloquée
```

**Avec audit :**
```
✅ Logs d'audit montrent :
  • User: alice@company.com
  • Action: DELETE pod/api-server
  • Timestamp: 2024-01-15 14:23:45 UTC
  • Source IP: 192.168.1.100
  • User-Agent: kubectl/1.28
  • Result: Success
         │
         ▼
Investigation possible → Actions correctives
```

#### 2. Conformité Réglementaire

De nombreuses réglementations exigent des logs d'audit :

- **SOC 2** : Traçabilité des accès
- **PCI-DSS** : Audit des systèmes de paiement
- **HIPAA** : Audit des systèmes de santé
- **GDPR** : Traçabilité des accès aux données personnelles
- **ISO 27001** : Gestion de la sécurité de l'information

#### 3. Détection d'Anomalies

```
Comportement Normal :
  Bob crée 2-3 pods par jour pendant les heures de bureau

Anomalie Détectée :
  Bob crée 50 pods à 3h du matin
         │
         ▼
🚨 Alerte : Compte compromis ou activité malveillante ?
```

#### 4. Déboguer des Problèmes

```
Problème : Déploiement échoue mystérieusement

Audit logs révèlent :
  • 14:00 : Deployment créé par CI/CD
  • 14:01 : ServiceAccount modifié par admin
  • 14:02 : Deployment échoue (permissions insuffisantes)
         │
         ▼
Cause identifiée : Modification du SA entre-temps
```

### Ce Que l'Audit Enregistre

```
┌─────────────────────────────────────────┐
│         Événement d'Audit               │
│                                         │
│  • QUI ? (Utilisateur, ServiceAccount)  │
│  • QUOI ? (Action: GET, CREATE, DELETE) │
│  • QUAND ? (Timestamp précis)           │
│  • OÙ ? (Source IP, User-Agent)         │
│  • SUR QUOI ? (Ressource affectée)      │
│  • RÉSULTAT ? (Success, Failure, Error) │
│  • DÉTAILS ? (Request body, Response)   │
└─────────────────────────────────────────┘
```

## Architecture de l'Audit

### Pipeline d'Audit

```
┌──────────────────────────────────────────────────────┐
│                                                      │
│  1. Requête vers API Server                          │
│     (kubectl, dashboard, autre service)              │
│                                                      │
└────────────────┬─────────────────────────────────────┘
                 │
                 ▼
┌──────────────────────────────────────────────────────┐
│  2. API Server                                       │
│     • Authentification                               │
│     • Autorisation (RBAC)                            │
│     • Admission Controllers                          │
│     • Traitement                                     │
└────────────────┬─────────────────────────────────────┘
                 │
                 ▼
┌──────────────────────────────────────────────────────┐
│  3. Audit Backend                                    │
│     • Log (fichier local)                            │
│     • Webhook (service externe)                      │
│     • Dynamic Backend                                │
└────────────────┬─────────────────────────────────────┘
                 │
                 ▼
┌──────────────────────────────────────────────────────┐
│  4. Stockage et Analyse                              │
│     • Elasticsearch                                  │
│     • Splunk                                         │
│     • CloudWatch / Cloud Logging                     │
│     • SIEM (Security Information Event Management)   │
└──────────────────────────────────────────────────────┘
```

### Étapes du Cycle d'Audit

Chaque requête passe par plusieurs étapes, toutes enregistrables :

1. **RequestReceived** : Requête reçue, avant traitement
2. **ResponseStarted** : Headers de réponse envoyés (streaming)
3. **ResponseComplete** : Réponse complète envoyée
4. **Panic** : Erreur inattendue pendant le traitement

## Niveaux d'Audit

Kubernetes définit quatre niveaux d'audit, du moins au plus détaillé :

### 1. None

**Description :** Aucun log d'audit.

```yaml
- level: None
```

**Usage :** Événements non importants à ignorer.

### 2. Metadata

**Description :** Enregistre les métadonnées de la requête (qui, quoi, quand) sans le contenu.

```yaml
- level: Metadata
```

**Contenu enregistré :**
```json
{
  "kind": "Event",
  "apiVersion": "audit.k8s.io/v1",
  "level": "Metadata",
  "stage": "ResponseComplete",
  "requestURI": "/api/v1/namespaces/default/pods",
  "verb": "create",
  "user": {
    "username": "alice@company.com",
    "groups": ["developers"]
  },
  "sourceIPs": ["192.168.1.100"],
  "userAgent": "kubectl/1.28",
  "objectRef": {
    "resource": "pods",
    "namespace": "default",
    "name": "nginx-pod",
    "apiVersion": "v1"
  },
  "responseStatus": {
    "code": 201
  },
  "requestReceivedTimestamp": "2024-01-15T14:23:45.123456Z",
  "stageTimestamp": "2024-01-15T14:23:45.234567Z"
}
```

**Avantages :**
- ✅ Faible volume de logs
- ✅ Performances acceptables
- ✅ Suffisant pour la plupart des audits

**Limites :**
- ❌ Pas de détails sur le contenu de la requête
- ❌ Pas de body de la requête/réponse

### 3. Request

**Description :** Enregistre les métadonnées ET le body de la requête.

```yaml
- level: Request
```

**Contenu supplémentaire :**
```json
{
  "level": "Request",
  "requestObject": {
    "apiVersion": "v1",
    "kind": "Pod",
    "metadata": {
      "name": "nginx-pod",
      "namespace": "default"
    },
    "spec": {
      "containers": [
        {
          "name": "nginx",
          "image": "nginx:latest"
        }
      ]
    }
  }
}
```

**Usage :** Audit détaillé sans surcharger avec les réponses.

### 4. RequestResponse

**Description :** Enregistre tout : métadonnées, requête ET réponse.

```yaml
- level: RequestResponse
```

**Contenu supplémentaire :**
```json
{
  "level": "RequestResponse",
  "requestObject": { ... },
  "responseObject": {
    "apiVersion": "v1",
    "kind": "Pod",
    "metadata": {
      "name": "nginx-pod",
      "namespace": "default",
      "uid": "abc-123-xyz",
      "creationTimestamp": "2024-01-15T14:23:45Z"
    },
    "status": {
      "phase": "Pending"
    }
  }
}
```

**Avantages :**
- ✅ Audit complet
- ✅ Investigation détaillée possible

**Limites :**
- ⚠️ Volume de logs très élevé
- ⚠️ Impact sur les performances
- ⚠️ Peut contenir des données sensibles

### Comparaison des Niveaux

| Niveau | Métadonnées | Body Requête | Body Réponse | Volume | Performance |
|--------|-------------|--------------|--------------|--------|-------------|
| None | ❌ | ❌ | ❌ | 0 | ⭐⭐⭐ |
| Metadata | ✅ | ❌ | ❌ | Faible | ⭐⭐⭐ |
| Request | ✅ | ✅ | ❌ | Moyen | ⭐⭐ |
| RequestResponse | ✅ | ✅ | ✅ | Élevé | ⭐ |

**Recommandation :** Commencer avec **Metadata** pour la plupart des ressources, **Request** pour les ressources sensibles.

## Configuration de l'Audit Policy

### Structure d'une Politique d'Audit

```yaml
apiVersion: audit.k8s.io/v1
kind: Policy
# Don't generate audit events for all requests in RequestReceived stage.
omitStages:
  - "RequestReceived"
rules:
  # Log pod changes at RequestResponse level
  - level: RequestResponse
    resources:
    - group: ""
      resources: ["pods"]

  # Log "pods/log", "pods/status" at Metadata level
  - level: Metadata
    resources:
    - group: ""
      resources: ["pods/log", "pods/status"]

  # Don't log requests to a configmap called "controller-leader"
  - level: None
    resources:
    - group: ""
      resources: ["configmaps"]
      resourceNames: ["controller-leader"]

  # Log all other resources in core and apps at Metadata level
  - level: Metadata
    resources:
    - group: ""  # core API group
    - group: "apps"

  # Log secrets changes at Metadata level (don't include body)
  - level: Metadata
    resources:
    - group: ""
      resources: ["secrets"]

  # Default: log everything else at Request level
  - level: Request
```

### Règles de Priorité

Les règles sont évaluées **dans l'ordre** : la première règle qui correspond s'applique.

```yaml
rules:
  # Règle 1 : Plus spécifique (appliquée en premier si match)
  - level: None
    users: ["system:serviceaccount:kube-system:generic-garbage-collector"]

  # Règle 2 : Moins spécifique
  - level: Metadata
    resources:
    - group: ""

  # Règle 3 : Catch-all (appliquée en dernier)
  - level: Request
```

**⚠️ Important :** Mettez les règles les plus spécifiques en premier.

### Filtrer par Utilisateur

```yaml
# Ignorer les requêtes du kubelet
- level: None
  users: ["kubelet"]

# Ignorer les health checks
- level: None
  users: ["system:anonymous"]
  verbs: ["get"]
  resources:
  - group: ""
    resources: ["healthz*", "livez*", "readyz*"]

# Auditer les admin strictement
- level: RequestResponse
  users: ["admin", "root"]
```

### Filtrer par Verbe

```yaml
# Auditer toutes les modifications
- level: Request
  verbs: ["create", "update", "patch", "delete"]
  resources:
  - group: ""
  - group: "apps"

# Ignorer les lectures (GET, LIST, WATCH)
- level: None
  verbs: ["get", "list", "watch"]
```

### Filtrer par Namespace

```yaml
# Audit strict en production
- level: RequestResponse
  namespaces: ["production"]
  resources:
  - group: ""
  - group: "apps"

# Moins strict en dev
- level: Metadata
  namespaces: ["dev", "staging"]

# Ignorer kube-system (trop verbeux)
- level: None
  namespaces: ["kube-system"]
```

### Filtrer par Ressource

```yaml
# Secrets : Metadata uniquement (ne pas logger les valeurs)
- level: Metadata
  resources:
  - group: ""
    resources: ["secrets"]

# ConfigMaps : Request level
- level: Request
  resources:
  - group: ""
    resources: ["configmaps"]

# Pods : RequestResponse
- level: RequestResponse
  resources:
  - group: ""
    resources: ["pods"]
```

## Exemples de Politiques

### Politique de Production (Recommandée)

```yaml
apiVersion: audit.k8s.io/v1
kind: Policy
omitStages:
  - "RequestReceived"
rules:
  # 1. Ignorer les health checks et system components
  - level: None
    users: ["system:kube-proxy", "system:kube-scheduler", "system:kube-controller-manager"]

  - level: None
    userGroups: ["system:nodes"]

  - level: None
    verbs: ["get"]
    resources:
    - group: ""
      resources: ["healthz*", "livez*", "readyz*", "version"]

  # 2. Secrets : Metadata uniquement (sécurité)
  - level: Metadata
    resources:
    - group: ""
      resources: ["secrets", "configmaps"]

  # 3. Changements critiques : RequestResponse
  - level: RequestResponse
    verbs: ["create", "update", "patch", "delete"]
    resources:
    - group: ""
      resources: ["pods", "services", "persistentvolumes", "persistentvolumeclaims"]
    - group: "apps"
      resources: ["deployments", "daemonsets", "statefulsets", "replicasets"]
    - group: "rbac.authorization.k8s.io"
      resources: ["roles", "rolebindings", "clusterroles", "clusterrolebindings"]

  # 4. Namespace production : audit strict
  - level: Request
    namespaces: ["production"]
    verbs: ["create", "update", "patch", "delete"]

  # 5. Tout le reste : Metadata
  - level: Metadata
    resources:
    - group: ""
    - group: "apps"
    - group: "batch"

  # 6. Catch-all : Request
  - level: Request
```

### Politique Minimaliste (Lab)

```yaml
apiVersion: audit.k8s.io/v1
kind: Policy
rules:
  # Ignorer presque tout sauf les modifications
  - level: None
    verbs: ["get", "list", "watch"]

  # Logger uniquement les modifications importantes
  - level: Metadata
    verbs: ["create", "update", "patch", "delete"]
```

### Politique de Conformité (Maximum Security)

```yaml
apiVersion: audit.k8s.io/v1
kind: Policy
omitStages:
  - "RequestReceived"
rules:
  # Audit complet des accès administrateurs
  - level: RequestResponse
    userGroups: ["system:masters", "cluster-admins"]

  # Audit complet des ressources sensibles
  - level: RequestResponse
    resources:
    - group: ""
      resources: ["secrets", "serviceaccounts"]
    - group: "rbac.authorization.k8s.io"

  # Audit complet des modifications en production
  - level: RequestResponse
    namespaces: ["production"]
    verbs: ["create", "update", "patch", "delete"]

  # Audit Metadata pour le reste
  - level: Metadata
```

## Backends d'Audit

### 1. Log Backend (Fichier Local)

Écrit les logs dans un fichier sur le nœud.

**Configuration de l'API Server :**

```yaml
# /etc/kubernetes/manifests/kube-apiserver.yaml
spec:
  containers:
  - command:
    - kube-apiserver
    - --audit-policy-file=/etc/kubernetes/audit-policy.yaml
    - --audit-log-path=/var/log/kubernetes/audit.log
    - --audit-log-maxage=30     # Garder 30 jours
    - --audit-log-maxbackup=10  # Garder 10 fichiers
    - --audit-log-maxsize=100   # Rotation à 100MB
    volumeMounts:
    - name: audit-policy
      mountPath: /etc/kubernetes/audit-policy.yaml
      readOnly: true
    - name: audit-logs
      mountPath: /var/log/kubernetes
  volumes:
  - name: audit-policy
    hostPath:
      path: /etc/kubernetes/audit-policy.yaml
      type: File
  - name: audit-logs
    hostPath:
      path: /var/log/kubernetes
      type: DirectoryOrCreate
```

**Avantages :**
- ✅ Simple
- ✅ Pas de dépendance externe
- ✅ Performances correctes

**Limites :**
- ⚠️ Stockage local limité
- ⚠️ Difficile à centraliser
- ⚠️ Peut être perdu si le nœud meurt

### 2. Webhook Backend

Envoie les événements d'audit à un service externe via HTTP.

**Configuration :**

```yaml
# /etc/kubernetes/audit-policy.yaml (même que précédemment)

# /etc/kubernetes/audit-webhook-config.yaml
apiVersion: v1
kind: Config
clusters:
- name: audit-webhook
  cluster:
    server: https://audit-collector.company.com/api/v1/audit
    certificate-authority: /etc/kubernetes/pki/audit-ca.crt
contexts:
- context:
    cluster: audit-webhook
    user: audit-user
  name: audit-webhook
current-context: audit-webhook
users:
- name: audit-user
  user:
    client-certificate: /etc/kubernetes/pki/audit-client.crt
    client-key: /etc/kubernetes/pki/audit-client.key
```

**Configuration de l'API Server :**

```yaml
spec:
  containers:
  - command:
    - kube-apiserver
    - --audit-policy-file=/etc/kubernetes/audit-policy.yaml
    - --audit-webhook-config-file=/etc/kubernetes/audit-webhook-config.yaml
    - --audit-webhook-mode=batch  # batch, blocking, ou blocking-strict
    - --audit-webhook-batch-max-size=100
    - --audit-webhook-batch-max-wait=30s
```

**Modes :**
- `batch` : Envoie par lots, ne bloque pas si échec (recommandé)
- `blocking` : Bloque si échec, retente
- `blocking-strict` : Bloque si échec, pas de retente

**Avantages :**
- ✅ Centralisation facile
- ✅ Intégration avec SIEM
- ✅ Scalabilité

**Limites :**
- ⚠️ Dépendance à un service externe
- ⚠️ Risque de perte en mode batch
- ⚠️ Configuration plus complexe

### 3. Dynamic Backend (Avancé)

Permet de configurer plusieurs backends dynamiquement.

```yaml
apiVersion: v1
kind: AuditSink
metadata:
  name: mysink
spec:
  policy:
    level: Metadata
    stages:
    - ResponseComplete
  webhook:
    throttle:
      qps: 10
      burst: 15
    clientConfig:
      url: "https://audit.example.com/audit"
```

## Analyser les Logs d'Audit

### Structure d'un Événement

```json
{
  "kind": "Event",
  "apiVersion": "audit.k8s.io/v1",
  "level": "Metadata",
  "auditID": "abc-123-xyz",
  "stage": "ResponseComplete",
  "requestURI": "/api/v1/namespaces/production/pods/nginx",
  "verb": "delete",
  "user": {
    "username": "alice@company.com",
    "uid": "user-123",
    "groups": ["developers", "system:authenticated"]
  },
  "sourceIPs": ["192.168.1.100"],
  "userAgent": "kubectl/1.28.0 (linux/amd64) kubernetes/abc123",
  "objectRef": {
    "resource": "pods",
    "namespace": "production",
    "name": "nginx",
    "apiVersion": "v1"
  },
  "responseStatus": {
    "metadata": {},
    "code": 200
  },
  "requestReceivedTimestamp": "2024-01-15T14:23:45.123456Z",
  "stageTimestamp": "2024-01-15T14:23:45.234567Z",
  "annotations": {
    "authorization.k8s.io/decision": "allow",
    "authorization.k8s.io/reason": "RBAC: allowed by RoleBinding \"developer/default\""
  }
}
```

### Requêtes Utiles avec jq

```bash
# Tous les DELETE
cat audit.log | jq 'select(.verb=="delete")'

# Toutes les actions d'un utilisateur
cat audit.log | jq 'select(.user.username=="alice@company.com")'

# Toutes les erreurs (status code >= 400)
cat audit.log | jq 'select(.responseStatus.code >= 400)'

# Actions sur les secrets
cat audit.log | jq 'select(.objectRef.resource=="secrets")'

# Actions dans le namespace production
cat audit.log | jq 'select(.objectRef.namespace=="production")'

# Top 10 utilisateurs les plus actifs
cat audit.log | jq -r '.user.username' | sort | uniq -c | sort -rn | head -10

# Suppressions en production
cat audit.log | jq 'select(.verb=="delete" and .objectRef.namespace=="production")'

# Échecs d'authentification
cat audit.log | jq 'select(.responseStatus.code==401 or .responseStatus.code==403)'
```

### Détection d'Anomalies

#### Activité Hors Horaires

```bash
# Activités entre 22h et 6h
cat audit.log | jq 'select(
  .stageTimestamp |
  strptime("%Y-%m-%dT%H:%M:%S.%fZ") |
  .hour >= 22 or .hour <= 6
)'
```

#### Suppressions Massives

```bash
# Plus de 10 suppressions en 5 minutes
cat audit.log | \
  jq 'select(.verb=="delete")' | \
  jq -r '.stageTimestamp' | \
  uniq -c | \
  awk '$1 > 10'
```

#### Accès depuis IPs Inhabituelles

```bash
# IPs uniques par utilisateur
cat audit.log | \
  jq -r '[.user.username, .sourceIPs[]] | @tsv' | \
  sort | uniq
```

## Intégration avec des Outils

### Elasticsearch + Kibana

#### 1. Déployer EFK Stack

```bash
# Elasticsearch
kubectl apply -f https://raw.githubusercontent.com/elastic/cloud-on-k8s/main/config/samples/elasticsearch/elasticsearch.yaml

# Filebeat pour collecter les logs
kubectl apply -f filebeat-config.yaml

# Kibana pour visualisation
kubectl apply -f https://raw.githubusercontent.com/elastic/cloud-on-k8s/main/config/samples/kibana/kibana.yaml
```

#### 2. Configuration Filebeat

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: filebeat-config
  namespace: kube-system
data:
  filebeat.yml: |
    filebeat.inputs:
    - type: log
      paths:
        - /var/log/kubernetes/audit.log
      json.keys_under_root: true
      json.add_error_key: true

    output.elasticsearch:
      hosts: ['elasticsearch:9200']
      index: "k8s-audit-%{+yyyy.MM.dd}"

    setup.template.name: "k8s-audit"
    setup.template.pattern: "k8s-audit-*"
```

#### 3. Dashboards Kibana

Créer des visualisations :
- Timeline des événements d'audit
- Top utilisateurs
- Distribution des verbes (create, delete, etc.)
- Taux d'erreurs
- Carte des IPs sources

### Splunk

```yaml
# Splunk Universal Forwarder
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: splunk-forwarder
spec:
  selector:
    matchLabels:
      app: splunk-forwarder
  template:
    metadata:
      labels:
        app: splunk-forwarder
    spec:
      containers:
      - name: splunk
        image: splunk/universalforwarder:latest
        env:
        - name: SPLUNK_START_ARGS
          value: "--accept-license"
        - name: SPLUNK_FORWARD_SERVER
          value: "splunk-server:9997"
        volumeMounts:
        - name: audit-logs
          mountPath: /var/log/kubernetes
          readOnly: true
      volumes:
      - name: audit-logs
        hostPath:
          path: /var/log/kubernetes
```

### Falco (Sécurité Runtime)

Falco peut ingérer les audit logs pour détecter des comportements suspects :

```yaml
# falco-config.yaml
json_output: true
json_include_output_property: true

# Règles personnalisées
- rule: Unauthorized Pod Delete in Production
  desc: Detect unauthorized pod deletion in production
  condition: >
    k8s_audit and
    ka.verb="delete" and
    ka.target.resource="pods" and
    ka.target.namespace="production" and
    not ka.user.name in (authorized_users)
  output: >
    Unauthorized pod deletion in production
    (user=%ka.user.name namespace=%ka.target.namespace pod=%ka.target.name)
  priority: CRITICAL
  tags: [k8s, audit]
```

## Cas d'Usage Pratiques

### Cas 1 : Enquête sur une Suppression Accidentelle

**Problème :** Un deployment a été supprimé en production.

```bash
# Trouver l'événement de suppression
cat audit.log | jq 'select(
  .verb=="delete" and
  .objectRef.resource=="deployments" and
  .objectRef.namespace=="production"
)' | tail -1

# Résultat :
{
  "user": {"username": "bob@company.com"},
  "stageTimestamp": "2024-01-15T14:23:45Z",
  "objectRef": {
    "name": "api-server",
    "namespace": "production"
  },
  "sourceIPs": ["192.168.1.150"],
  "userAgent": "kubectl/1.28",
  "responseStatus": {"code": 200}
}
```

**Conclusion :** Bob a supprimé le deployment à 14h23 depuis l'IP 192.168.1.150.

### Cas 2 : Détection de Compromission

**Alerte :** Activité suspecte détectée.

```bash
# Vérifier les actions d'un utilisateur suspect
cat audit.log | jq 'select(.user.username=="alice@company.com")' | \
  jq -r '[.verb, .objectRef.resource, .objectRef.name] | @tsv'

# Résultat montre :
# delete  clusterrolebinding  cluster-admin-binding
# create  clusterrolebinding  backdoor-admin
# create  pod                 crypto-miner
```

**Analyse :**
1. Alice a supprimé un binding admin
2. Créé un nouveau binding suspect
3. Créé un pod de cryptominage

→ **Compte compromis ou attaque interne**

### Cas 3 : Audit de Conformité

**Besoin :** Rapport mensuel des accès aux secrets.

```bash
# Tous les accès aux secrets du mois
cat audit.log | jq 'select(
  .objectRef.resource=="secrets" and
  (.stageTimestamp | startswith("2024-01"))
)' | \
jq -r '[.user.username, .verb, .objectRef.name, .stageTimestamp] | @csv' > secrets-access-report.csv
```

**Générer un rapport :**
- Qui a accédé à quels secrets ?
- Combien d'accès par utilisateur ?
- Accès depuis quelles IPs ?

### Cas 4 : Performance Debugging

**Problème :** API Server lent.

```bash
# Requêtes les plus lentes (différence entre reception et completion)
cat audit.log | jq '
  select(.stage=="ResponseComplete") |
  {
    duration: (
      (.stageTimestamp | fromdateiso8601) -
      (.requestReceivedTimestamp | fromdateiso8601)
    ),
    verb: .verb,
    resource: .objectRef.resource,
    user: .user.username
  } |
  select(.duration > 1)  # Plus d'1 seconde
' | jq -s 'sort_by(.duration) | reverse | .[0:10]'
```

## Bonnes Pratiques

### 1. Commencer Simple

```yaml
# Phase 1 : Metadata pour tout (semaine 1-2)
- level: Metadata

# Phase 2 : Request pour modifications (semaine 3-4)
- level: Request
  verbs: ["create", "update", "patch", "delete"]

# Phase 3 : Politique détaillée (semaine 5+)
# (comme les exemples ci-dessus)
```

### 2. Rotation des Logs

```bash
# Configuration de rotation
--audit-log-maxage=30      # Garder 30 jours
--audit-log-maxbackup=10   # 10 fichiers max
--audit-log-maxsize=100    # 100MB par fichier
```

**Calcul du volume :**
```
Metadata : ~500 bytes/événement
Request : ~2KB/événement
RequestResponse : ~5KB/événement

Cluster actif : 1000 req/s
Metadata : 500B * 1000 * 3600 * 24 = ~43GB/jour
```

### 3. Sauvegarder les Logs

```bash
# Cron pour sauvegarder vers S3
0 1 * * * tar -czf /backup/audit-$(date +\%Y-\%m-\%d).tar.gz /var/log/kubernetes/audit.log* && \
          aws s3 cp /backup/audit-$(date +\%Y-\%m-\%d).tar.gz s3://my-bucket/audit-logs/
```

### 4. Monitoring des Logs d'Audit

```yaml
# Alerte Prometheus
- alert: AuditLogNotWritten
  expr: rate(apiserver_audit_event_total[5m]) == 0
  for: 10m
  annotations:
    summary: "Audit logs not being written"
```

### 5. Tester la Politique d'Audit

```bash
# Avant de déployer en production
kubectl create namespace audit-test

# Faire diverses actions
kubectl create deployment nginx --image=nginx -n audit-test
kubectl delete deployment nginx -n audit-test

# Vérifier les logs
grep "audit-test" /var/log/kubernetes/audit.log | jq .
```

### 6. Respecter la Vie Privée

```yaml
# Ne pas logger le contenu des secrets
- level: Metadata  # Pas Request ou RequestResponse
  resources:
  - group: ""
    resources: ["secrets"]

# Anonymiser les données sensibles si nécessaire
```

### 7. Alertes sur Actions Critiques

```yaml
# Règle d'alerte exemple
- alert: SecretDeleted
  expr: |
    k8s_audit_event{verb="delete", resource="secrets"} > 0
  annotations:
    summary: "Secret {{ $labels.name }} deleted by {{ $labels.user }}"
```

### 8. Documentation

```yaml
# Documenter votre politique
# /etc/kubernetes/audit-policy.yaml
# Politique d'audit pour cluster production
# Owner: security-team@company.com
# Last Updated: 2024-01-15
# Review Schedule: Quarterly

apiVersion: audit.k8s.io/v1
kind: Policy
rules: ...
```

## Audit dans MicroK8s

### Activer l'Audit

MicroK8s rend l'activation de l'audit relativement simple :

#### 1. Créer la Politique d'Audit

```bash
# Créer le fichier de politique
sudo mkdir -p /var/snap/microk8s/current/args/

sudo tee /var/snap/microk8s/current/args/audit-policy.yaml > /dev/null <<EOF
apiVersion: audit.k8s.io/v1
kind: Policy
rules:
- level: Metadata
  verbs: ["create", "update", "patch", "delete"]
- level: None
  verbs: ["get", "list", "watch"]
EOF
```

#### 2. Configurer l'API Server

```bash
# Éditer la configuration de l'API Server
sudo nano /var/snap/microk8s/current/args/kube-apiserver

# Ajouter ces lignes :
--audit-policy-file=/var/snap/microk8s/current/args/audit-policy.yaml
--audit-log-path=/var/log/kubernetes/audit.log
--audit-log-maxage=30
--audit-log-maxbackup=10
--audit-log-maxsize=100
```

#### 3. Redémarrer MicroK8s

```bash
# Redémarrer pour appliquer la configuration
microk8s stop
microk8s start

# Vérifier que l'audit fonctionne
sudo tail -f /var/log/kubernetes/audit.log
```

#### 4. Tester

```bash
# Faire une action
microk8s kubectl create deployment nginx --image=nginx

# Vérifier les logs
sudo cat /var/log/kubernetes/audit.log | jq 'select(.objectRef.name=="nginx")'
```

### Limitations dans MicroK8s

- Pas de webhook backend facilement configurable
- Stockage local uniquement par défaut
- Nécessite configuration manuelle

**Solution :** Utiliser Filebeat ou Fluentd pour envoyer vers un collecteur centralisé.

## Compliance et Réglementations

### SOC 2

**Exigences :**
- Logging de tous les accès
- Rétention 1 an minimum
- Audit trail immuable

**Configuration :**
```yaml
- level: Metadata  # Minimum
  verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]
```

### PCI-DSS

**Exigences :**
- Audit de tous les accès aux données de cartes
- Rétention 1 an, disponible immédiatement pendant 3 mois
- Revue quotidienne des logs

**Configuration :**
```yaml
# Audit strict pour namespaces avec données sensibles
- level: RequestResponse
  namespaces: ["payment", "pci-scope"]
```

### HIPAA

**Exigences :**
- Audit des accès aux données de santé protégées (PHI)
- Rétention 6 ans
- Alertes sur accès anormaux

**Configuration :**
```yaml
# Audit détaillé pour données de santé
- level: RequestResponse
  namespaces: ["healthcare-prod"]
  resources:
  - group: ""
    resources: ["secrets", "configmaps"]
```

## Checklist Audit Logging

Avant de mettre en production :

- [ ] Politique d'audit définie et testée
- [ ] Backend configuré (log file ou webhook)
- [ ] Rotation des logs configurée
- [ ] Sauvegarde des logs vers stockage externe
- [ ] Monitoring : alertes si logs s'arrêtent
- [ ] Centralisation vers SIEM ou Elasticsearch
- [ ] Dashboards créés pour visualisation
- [ ] Alertes sur actions critiques
- [ ] Documentation de la politique
- [ ] Procédure de revue régulière des logs
- [ ] Rétention conforme aux réglementations
- [ ] Accès aux logs restreint (RBAC)
- [ ] Tests de restauration de logs
- [ ] Formation de l'équipe

## Conclusion

L'audit logging est essentiel pour :

1. **Sécurité** : Détecter et enquêter sur les incidents
2. **Conformité** : Répondre aux exigences réglementaires
3. **Opérations** : Déboguer et optimiser
4. **Responsabilité** : Tracer les actions de chacun

**Formule Audit :**
```
Sécurité = Logs (Qui + Quoi + Quand) + Analyse + Alertes
```

**Points Clés à Retenir :**

- Activer l'audit dès le début
- Commencer avec **Metadata** pour la plupart des ressources
- Utiliser **Request** pour les modifications importantes
- **RequestResponse** uniquement pour les ressources critiques
- Centraliser les logs (Elasticsearch, Splunk, SIEM)
- Sauvegarder les logs vers stockage externe
- Configurer des alertes sur actions critiques
- Respecter les réglementations de rétention
- Tester régulièrement la politique d'audit
- Former l'équipe à l'analyse des logs

**Recommandations par Environnement :**

- **Lab/Dev** : Metadata minimal, 7 jours de rétention
- **Staging** : Metadata complet, 30 jours de rétention
- **Production** : Request pour modifications + Metadata, 1 an de rétention

**Prochaines Étapes :**

Dans les sections suivantes, nous explorerons :
- **16.9** : Bonnes pratiques de sécurité
- **16.10** : Checklist de sécurité complète

L'audit logging, combiné avec RBAC, Network Policies, Pod Security, Scan d'images et Secrets management, complète votre stratégie de sécurité en profondeur pour Kubernetes. C'est votre "boîte noire" qui vous permet de comprendre ce qui se passe dans votre cluster et de réagir rapidement en cas de problème.

⏭️ [Bonnes pratiques sécurité](/16-securite-kubernetes/09-bonnes-pratiques-securite.md)
