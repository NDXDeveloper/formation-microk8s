🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 14.6 Silences et Inhibitions

## Introduction

Deux mécanismes essentiels d'Alertmanager permettent de gérer intelligemment vos alertes : les **silences** et les **inhibitions**. Bien que leurs objectifs soient similaires (réduire le bruit), ils fonctionnent de manière très différente.

**Silences** : Désactivation **temporaire et manuelle** de certaines alertes
- Vous décidez de ne pas être notifié pour certaines alertes pendant une période donnée
- Typiquement utilisé pendant les maintenances planifiées
- Action manuelle et intentionnelle

**Inhibitions** : Suppression **automatique** d'alertes redondantes
- Le système supprime automatiquement certaines alertes quand d'autres sont actives
- Typiquement utilisé pour éviter les alertes en cascade
- Configuration permanente basée sur des règles

Ces deux mécanismes sont complémentaires et essentiels pour maintenir un système d'alerting sain et non-intrusif.

## Les Silences : Désactivation Temporaire

Un **silence** est comme un "mode ne pas déranger" pour vos alertes. Vous définissez des critères et une période pendant laquelle les alertes correspondantes ne généreront pas de notifications.

### Pourquoi Utiliser les Silences ?

**Maintenance planifiée** : Vous redémarrez un serveur, vous savez qu'il sera down
- Sans silence : 50 alertes "ServiceDown", notifications constantes
- Avec silence : Aucune notification pendant la maintenance

**Investigation en cours** : Vous travaillez déjà sur un problème
- Sans silence : Rappels toutes les heures alors que vous savez déjà
- Avec silence : Vous travaillez tranquillement sans être dérangé

**Déploiement** : Vous déployez une nouvelle version
- Sans silence : Alertes de pods qui redémarrent
- Avec silence : Déploiement serein

**Tests de charge** : Vous effectuez des tests qui vont déclencher des alertes
- Sans silence : Fausses alarmes pendant les tests
- Avec silence : Tests sans pollution du système d'alerting

### Anatomie d'un Silence

Un silence est composé de :

**Matchers** : Les critères pour identifier les alertes à silencer
```yaml
alertname = "HighCPU"
namespace = "production"
```

**Période** : Début et fin du silence
```yaml
startsAt: "2025-01-15T20:00:00Z"
endsAt: "2025-01-15T22:00:00Z"
```

**Créateur** : Qui a créé le silence
```yaml
createdBy: "john@example.com"
```

**Commentaire** : Pourquoi ce silence existe
```yaml
comment: "Maintenance planifiée du serveur database-01"
```

### Créer un Silence via l'Interface Web

C'est la méthode la plus simple pour les débutants.

**Étapes** :

1. **Exposer Alertmanager** :
```bash
microk8s kubectl port-forward -n monitoring svc/alertmanager-operated 9093:9093
```

2. **Ouvrir l'interface** : `http://localhost:9093`

3. **Aller dans l'onglet "Silences"**

4. **Cliquer sur "New Silence"**

5. **Remplir le formulaire** :
   - **Matchers** : Cliquez sur "+" pour ajouter des critères
     - Name: `alertname`
     - Value: `HighCPU`
     - IsRegex: coché ou non selon le besoin

   - **Start** : Date et heure de début

   - **End** : Date et heure de fin (ou Duration)

   - **Creator** : Votre nom/email

   - **Comment** : Raison du silence (OBLIGATOIRE)
     - Exemple : "Maintenance database - ticket #1234"

6. **Cliquer sur "Create"**

### Créer un Silence via l'API

Pour automatiser ou scripter la création de silences :

**Créer un silence pour 2 heures** :

```bash
curl -X POST http://localhost:9093/api/v2/silences \
  -H 'Content-Type: application/json' \
  -d '{
    "matchers": [
      {
        "name": "alertname",
        "value": "HighCPU",
        "isRegex": false,
        "isEqual": true
      },
      {
        "name": "instance",
        "value": "server-01",
        "isRegex": false,
        "isEqual": true
      }
    ],
    "startsAt": "2025-01-15T20:00:00.000Z",
    "endsAt": "2025-01-15T22:00:00.000Z",
    "createdBy": "automation@example.com",
    "comment": "Maintenance planifiée - ticket #1234"
  }'
```

**Réponse** :
```json
{
  "silenceID": "abc123-def456-ghi789"
}
```

Conservez cet ID pour supprimer le silence plus tard si nécessaire.

### Créer un Silence via amtool

L'outil CLI officiel d'Alertmanager :

**Installation** :
```bash
# Via Go
go install github.com/prometheus/alertmanager/cmd/amtool@latest

# Ou télécharger depuis GitHub releases
```

**Créer un silence** :
```bash
amtool silence add \
  alertname=HighCPU \
  instance=server-01 \
  --duration=2h \
  --author=john@example.com \
  --comment="Maintenance planifiée"
```

**Lister les silences actifs** :
```bash
amtool silence query
```

**Supprimer un silence** :
```bash
amtool silence expire abc123-def456-ghi789
```

### Matchers : Cibler les Alertes

Les **matchers** définissent quelles alertes seront silencées. C'est comme un filtre.

#### Match Exact

Silencer une alerte spécifique :

```json
{
  "name": "alertname",
  "value": "HighCPU",
  "isRegex": false,
  "isEqual": true
}
```

**Effet** : Silencer uniquement les alertes nommées exactement "HighCPU"

#### Match avec Regex

Silencer plusieurs alertes avec un pattern :

```json
{
  "name": "alertname",
  "value": "High.*",
  "isRegex": true,
  "isEqual": true
}
```

**Effet** : Silencer toutes les alertes commençant par "High" (HighCPU, HighMemory, HighDisk, etc.)

#### Multiple Matchers (ET logique)

Combiner plusieurs critères :

```json
{
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
    },
    {
      "name": "cluster",
      "value": "eu-west",
      "isRegex": false
    }
  ]
}
```

**Effet** : Silencer uniquement les alertes HighCPU dans le namespace production du cluster eu-west

#### Matcher Négatif (NOT)

Silencer tout sauf certaines alertes :

```json
{
  "name": "severity",
  "value": "critical",
  "isRegex": false,
  "isEqual": false  // NOT equal
}
```

**Effet** : Silencer toutes les alertes sauf celles de sévérité critical

#### Exemples de Matchers Courants

**Silencer un serveur spécifique** :
```json
{"name": "instance", "value": "server-01"}
```

**Silencer un namespace** :
```json
{"name": "namespace", "value": "staging"}
```

**Silencer un cluster entier** :
```json
{"name": "cluster", "value": "dev-cluster"}
```

**Silencer toutes les alertes de type "Pod"** :
```json
{"name": "alertname", "value": "Pod.*", "isRegex": true}
```

**Silencer un niveau de sévérité** :
```json
{"name": "severity", "value": "warning"}
```

### Durée des Silences

#### Silence Immédiat

Commence maintenant :

```bash
curl -X POST http://localhost:9093/api/v2/silences \
  -d '{
    "matchers": [...],
    "startsAt": "'$(date -u +"%Y-%m-%dT%H:%M:%S.000Z")'",
    "endsAt": "'$(date -u -d "+2 hours" +"%Y-%m-%dT%H:%M:%S.000Z")'",
    "comment": "Maintenance en cours"
  }'
```

#### Silence Planifié

Commence dans le futur :

```json
{
  "startsAt": "2025-01-20T02:00:00.000Z",
  "endsAt": "2025-01-20T06:00:00.000Z",
  "comment": "Maintenance planifiée dimanche 2h-6h"
}
```

#### Silence Long

Pour des périodes prolongées :

```json
{
  "startsAt": "2025-01-15T00:00:00.000Z",
  "endsAt": "2025-02-01T00:00:00.000Z",
  "comment": "Migration infrastructure - 2 semaines"
}
```

**⚠️ Attention** : Les silences longs sont risqués - vous pourriez manquer de vrais problèmes.

### Expirer un Silence Manuellement

Si votre maintenance se termine plus tôt que prévu :

**Via l'interface web** :
1. Onglet "Silences"
2. Trouver votre silence
3. Cliquer sur "Expire"

**Via l'API** :
```bash
curl -X DELETE http://localhost:9093/api/v2/silence/SILENCE_ID
```

**Via amtool** :
```bash
amtool silence expire SILENCE_ID
```

### Visualiser les Silences Actifs

**Interface web** :
- Onglet "Silences"
- Voir tous les silences actifs, en attente et expirés

**API** :
```bash
curl http://localhost:9093/api/v2/silences
```

**amtool** :
```bash
amtool silence query
```

**Filtrer les silences** :
```bash
amtool silence query alertname=HighCPU
```

### Bonnes Pratiques pour les Silences

#### ✓ Toujours Ajouter un Commentaire Détaillé

```yaml
comment: "Maintenance DB - Ticket #1234 - John Doe - Migration schema"
```

**Bon commentaire contient** :
- Raison du silence
- Référence ticket/issue
- Qui est responsable
- Contexte additionnel

#### ✓ Utiliser des Durées Raisonnables

- **Maintenance courte** : 1-2 heures
- **Maintenance longue** : 4-6 heures maximum
- **Éviter** : silences de plusieurs jours

Si vous avez besoin de plus, créez plusieurs silences successifs ou revoyez votre approche.

#### ✓ Être Spécifique

**Mauvais** (trop large) :
```json
{"name": "alertname", "value": ".*", "isRegex": true}
// Silencer TOUTES les alertes - DANGEREUX !
```

**Bon** (spécifique) :
```json
[
  {"name": "alertname", "value": "DatabaseConnectionIssue"},
  {"name": "instance", "value": "db-01"}
]
// Silencer uniquement cette alerte sur ce serveur
```

#### ✓ Nettoyer les Silences Expirés

L'interface conserve l'historique. Nettoyez périodiquement pour maintenir la clarté.

#### ✗ Ne Jamais Silencer les Alertes Critiques de Manière Permanente

Si une alerte critique se déclenche constamment :
- ✗ Ne la silencez pas indéfiniment
- ✓ Corrigez le problème sous-jacent
- ✓ Ajustez le seuil de l'alerte si inapproprié
- ✓ Documentez pourquoi elle se déclenche

## Les Inhibitions : Suppression Automatique

Les **inhibitions** suppriment automatiquement certaines alertes lorsque d'autres alertes plus importantes sont actives. C'est un mécanisme permanent configuré dans Alertmanager.

### Pourquoi Utiliser les Inhibitions ?

**Éviter les alertes en cascade** : Quand un problème en cause d'autres

**Scénario sans inhibition** :
1. Serveur database tombe en panne
2. 50 alertes se déclenchent :
   - DatabaseDown
   - APIResponseSlow (car attend la DB)
   - WebsiteUnavailable (car pas de DB)
   - CacheInvalid (car pas de DB)
   - BackgroundJobsFailing (car pas de DB)
   - etc.

**Résultat** : Vous êtes submergé de notifications pour des effets secondaires alors que le vrai problème est simple : la DB est down.

**Scénario avec inhibition** :
1. Serveur database tombe en panne
2. Alerte DatabaseDown se déclenche
3. Toutes les autres alertes liées sont automatiquement inhibées
4. Vous recevez 1 seule notification : DatabaseDown

### Anatomie d'une Règle d'Inhibition

Une règle d'inhibition dans Alertmanager :

```yaml
inhibit_rules:
  - source_match:
      # L'alerte "source" qui, si active, inhibe les autres
      alertname: 'NodeDown'
      severity: 'critical'

    target_match:
      # Les alertes "cibles" qui seront inhibées
      severity: 'warning'

    target_match_re:
      # Avec regex pour matcher plusieurs alertes
      alertname: '(PodNotReady|ServiceUnavailable)'

    equal:
      # Les labels qui doivent être identiques entre source et cible
      - 'instance'
      - 'cluster'
```

**Composants** :

**source_match** : L'alerte "inhibitrice" (plus importante)
- Si cette alerte est active (firing), elle peut inhiber d'autres alertes

**target_match** : Les alertes qui seront inhibées (moins importantes)
- Match exact sur les labels

**target_match_re** : Les alertes qui seront inhibées (avec regex)
- Match avec expressions régulières

**equal** : Labels qui doivent être identiques
- Pour garantir que les alertes sont liées (même serveur, même cluster, etc.)

### Principe de Fonctionnement

```
[Source Alert: NodeDown, instance=server-01] → Active (firing)
                    ↓
         Vérifie les règles d'inhibition
                    ↓
    Les labels 'instance' sont égaux ?
                    ↓
              Oui : server-01
                    ↓
[Target Alert: PodNotReady, instance=server-01] → INHIBÉE (pas de notification)
```

### Exemples d'Inhibitions Courantes

#### 1. Nœud Down Inhibe les Alertes de Pods

Quand un nœud Kubernetes tombe, tous ses pods sont affectés.

```yaml
inhibit_rules:
  - source_match:
      alertname: 'NodeDown'
    target_match_re:
      alertname: '(PodNotReady|PodCrashLooping|ContainerRestarting)'
    equal:
      - 'node'
```

**Logique** :
- Si NodeDown est actif sur "node-01"
- Alors inhiber toutes les alertes de pods sur "node-01"
- Car la cause root est le nœud, pas les pods

#### 2. Alerte Critique Inhibe les Warnings Similaires

Si vous avez déjà une alerte critique, pas besoin du warning équivalent.

```yaml
inhibit_rules:
  - source_match:
      alertname: 'CriticalDiskSpace'
      severity: 'critical'
    target_match:
      alertname: 'LowDiskSpace'
      severity: 'warning'
    equal:
      - 'instance'
      - 'mountpoint'
```

**Logique** :
- Si CriticalDiskSpace est actif (< 5% libre)
- Alors inhiber LowDiskSpace (< 15% libre)
- Car le critique inclut déjà le warning

#### 3. Cluster Down Inhibe Tout

Si un cluster entier est inaccessible :

```yaml
inhibit_rules:
  - source_match:
      alertname: 'ClusterDown'
    target_match_re:
      alertname: '.*'
    equal:
      - 'cluster'
```

**Logique** :
- Si ClusterDown est actif
- Inhiber TOUTES les autres alertes du même cluster
- Car elles sont toutes des conséquences

#### 4. Maintenance Mode Automatique

Si un label "maintenance" existe :

```yaml
inhibit_rules:
  - source_match:
      maintenance: 'true'
    target_match:
      severity: 'warning'
    equal:
      - 'instance'
```

**Note** : Nécessite que vos alertes aient un label `maintenance` ajouté par Prometheus.

#### 5. Service Principal Down Inhibe les Services Dépendants

```yaml
inhibit_rules:
  - source_match:
      alertname: 'DatabaseDown'
      service: 'postgres'
    target_match_re:
      alertname: '(APIDown|WebsiteDown|BackendDown)'
    equal:
      - 'environment'
```

**Logique** :
- Si la base de données est down en production
- Inhiber les alertes API/Website/Backend en production
- Car ils dépendent tous de la DB

#### 6. Alertes Réseau

```yaml
inhibit_rules:
  - source_match:
      alertname: 'NetworkPartition'
    target_match_re:
      alertname: '(NodeUnreachable|ServiceTimeout|ProbeFailure)'
    equal:
      - 'datacenter'
```

#### 7. Sévérité en Cascade

```yaml
inhibit_rules:
  # Critical inhibe Warning
  - source_match:
      severity: 'critical'
    target_match:
      severity: 'warning'
    equal:
      - 'alertname'
      - 'instance'

  # Warning inhibe Info
  - source_match:
      severity: 'warning'
    target_match:
      severity: 'info'
    equal:
      - 'alertname'
      - 'instance'
```

**Logique** :
- Si même alerte sur même instance
- La sévérité supérieure masque les inférieures

### Configuration Complète d'Inhibitions

```yaml
inhibit_rules:
  # 1. Nœud down → inhibe alertes pods/containers
  - source_match:
      alertname: 'NodeDown'
      severity: 'critical'
    target_match_re:
      alertname: '(Pod.*|Container.*|Kubelet.*)'
    equal:
      - 'node'
      - 'cluster'

  # 2. Cluster down → inhibe tout
  - source_match:
      alertname: 'ClusterUnreachable'
    target_match_re:
      alertname: '.*'
    equal:
      - 'cluster'

  # 3. Database down → inhibe services dépendants
  - source_match:
      alertname: 'DatabaseConnectionFailure'
      component: 'database'
    target_match_re:
      alertname: '(APILatencyHigh|BackendError|CacheFailure)'
    equal:
      - 'environment'
      - 'datacenter'

  # 4. Critical masque Warning (même alerte)
  - source_match:
      severity: 'critical'
    target_match:
      severity: 'warning'
    equal:
      - 'alertname'
      - 'instance'

  # 5. Warning masque Info (même alerte)
  - source_match:
      severity: 'warning'
    target_match:
      severity: 'info'
    equal:
      - 'alertname'
      - 'instance'

  # 6. Alerte haute priorité masque basse priorité
  - source_match:
      priority: 'P1'
    target_match_re:
      priority: '(P2|P3|P4)'
    equal:
      - 'alertname'
      - 'service'
```

### Appliquer les Inhibitions

Les inhibitions sont configurées dans le fichier principal d'Alertmanager.

**Éditer la configuration** :

```bash
# Sur MicroK8s
microk8s kubectl edit configmap alertmanager-config -n monitoring
```

**Ajouter/modifier la section inhibit_rules** :

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: alertmanager-config
  namespace: monitoring
data:
  alertmanager.yml: |
    global:
      resolve_timeout: 5m

    route:
      receiver: 'default'
      # ... routing config

    inhibit_rules:
      - source_match:
          alertname: 'NodeDown'
        target_match_re:
          alertname: 'Pod.*'
        equal:
          - 'node'

    receivers:
      # ... receivers config
```

**Recharger la configuration** :

```bash
# Méthode 1 : Rechargement à chaud (si supporté)
microk8s kubectl exec -n monitoring alertmanager-0 -- \
  curl -X POST http://localhost:9093/-/reload

# Méthode 2 : Redémarrer le pod
microk8s kubectl delete pod -n monitoring alertmanager-0
```

### Tester les Inhibitions

#### 1. Créer des Alertes de Test

**Alerte source (inhibitrice)** :
```bash
curl -X POST http://localhost:9093/api/v2/alerts \
  -H 'Content-Type: application/json' \
  -d '[
    {
      "labels": {
        "alertname": "NodeDown",
        "node": "worker-01",
        "severity": "critical"
      },
      "annotations": {
        "summary": "Node worker-01 is down"
      }
    }
  ]'
```

**Alerte cible (devrait être inhibée)** :
```bash
curl -X POST http://localhost:9093/api/v2/alerts \
  -H 'Content-Type: application/json' \
  -d '[
    {
      "labels": {
        "alertname": "PodNotReady",
        "node": "worker-01",
        "pod": "app-123",
        "severity": "warning"
      },
      "annotations": {
        "summary": "Pod not ready"
      }
    }
  ]'
```

#### 2. Vérifier l'Inhibition

**Interface web** :
1. Aller sur `http://localhost:9093`
2. Onglet "Alerts"
3. L'alerte PodNotReady devrait apparaître mais avec une indication d'inhibition

**API** :
```bash
curl http://localhost:9093/api/v2/alerts
```

Cherchez `"status": "suppressed"` dans la réponse.

### Visualiser les Inhibitions Actives

**API Status** :
```bash
curl http://localhost:9093/api/v2/status
```

Affiche la configuration actuelle incluant les inhibit_rules.

**Logs** :
```bash
microk8s kubectl logs -n monitoring alertmanager-0 | grep inhibit
```

Recherchez les lignes mentionnant les inhibitions appliquées.

### Différence entre Silences et Inhibitions

| Aspect | Silences | Inhibitions |
|--------|----------|-------------|
| **Type** | Manuel | Automatique |
| **Durée** | Temporaire | Permanent |
| **Configuration** | Via UI/API à chaque fois | Règles dans config YAML |
| **Cas d'usage** | Maintenances, investigations | Alertes en cascade |
| **Décision** | Humaine | Système |
| **Flexibilité** | Très flexible, ad-hoc | Règles fixes |
| **Visibilité** | Visible dans interface | Transparent pour l'utilisateur |

### Bonnes Pratiques pour les Inhibitions

#### ✓ Créer une Hiérarchie Claire

Organisez vos inhibitions par niveaux :

```
Niveau 1: Infrastructure
  ├─ DatacenterDown
  └─ ClusterDown
      ├─ Niveau 2: Nœuds
      │   ├─ NodeDown
      │   └─ NodeUnreachable
      │       └─ Niveau 3: Applications
      │           ├─ PodNotReady
      │           ├─ ServiceDown
      │           └─ ContainerCrashing
```

#### ✓ Utiliser le Label 'equal' Judicieusement

**Trop large** (problématique) :
```yaml
equal: []  # Pas de filtre - inhibe tout partout
```

**Trop spécifique** (inefficace) :
```yaml
equal:
  - 'node'
  - 'pod'
  - 'container'
  - 'namespace'
  - 'deployment'
```

**Juste bien** :
```yaml
equal:
  - 'node'
  - 'cluster'
```

#### ✓ Tester Avant de Déployer en Production

1. Déployer dans environnement de test
2. Simuler des alertes
3. Vérifier que les inhibitions fonctionnent
4. Ajuster si nécessaire
5. Déployer en production

#### ✓ Documenter Vos Inhibitions

```yaml
inhibit_rules:
  # Inhibition 1: Nœud down masque les alertes de pods
  # Raison: Quand un nœud est down, tous ses pods sont affectés
  # Propriétaire: Équipe SRE
  # Date: 2025-01-15
  - source_match:
      alertname: 'NodeDown'
    target_match_re:
      alertname: 'Pod.*'
    equal:
      - 'node'
```

#### ✓ Monitorer l'Efficacité

Créez des métriques pour suivre :
- Nombre d'alertes inhibées
- Ratio alertes inhibées / alertes totales
- Alertes jamais inhibées (peut-être inutiles ?)

```promql
# Alertes actuellement inhibées
count(ALERTS{alertstate="suppressed"})

# Ratio d'inhibition
count(ALERTS{alertstate="suppressed"}) / count(ALERTS)
```

#### ✗ Ne Pas Sur-utiliser les Inhibitions

**Symptôme** : Trop d'alertes inhibées en permanence

**Cause** : Mauvaise configuration d'alertes ou inhibitions trop agressives

**Solution** : Revoir vos règles d'alerte plutôt que de tout inhiber

## Scénarios Pratiques Combinés

### Scénario 1 : Maintenance Planifiée d'un Nœud

**Objectif** : Redémarrer le nœud worker-02 pour des mises à jour

**Approche combinée** :

1. **Inhibition permanente** (déjà configurée) :
```yaml
inhibit_rules:
  - source_match:
      alertname: 'NodeDown'
    target_match_re:
      alertname: 'Pod.*'
    equal: ['node']
```
Gère automatiquement les alertes de pods quand le nœud redémarre.

2. **Silence temporaire** (créé manuellement) :
```bash
amtool silence add \
  alertname=NodeDown \
  node=worker-02 \
  --duration=1h \
  --comment="Maintenance worker-02 - Mises à jour système"
```
Évite l'alerte NodeDown elle-même pendant la maintenance.

**Résultat** :
- Pas d'alerte NodeDown (silence)
- Pas d'alertes de pods (inhibition automatique)
- Maintenance sereine

### Scénario 2 : Investigation d'un Problème Récurrent

**Objectif** : Une alerte HighMemory se déclenche régulièrement, vous enquêtez

**Approche** :

1. **Silence pendant investigation** :
```bash
amtool silence add \
  alertname=HighMemory \
  pod=problematic-app-xyz \
  namespace=production \
  --duration=4h \
  --comment="Investigation fuite mémoire - Ticket #5678 - Alice"
```

2. **Ne PAS créer d'inhibition** :
- L'inhibition serait permanente
- Le problème doit être résolu, pas masqué

**Résultat** :
- Vous travaillez sans être dérangé par les rappels
- Après résolution, le silence expire naturellement

### Scénario 3 : Déploiement Canary

**Objectif** : Déployer progressivement une nouvelle version

**Approche** :

```bash
# Silencer les alertes de l'ancienne version en cours de suppression
amtool silence add \
  namespace=production \
  version=v1.2.3 \
  --duration=30m \
  --comment="Déploiement canary v1.2.4 - Suppression v1.2.3"

# Silencer temporairement les alertes de démarrage de la nouvelle version
amtool silence add \
  namespace=production \
  version=v1.2.4 \
  --duration=15m \
  --comment="Déploiement canary v1.2.4 - Warmup period"
```

### Scénario 4 : Incident en Cascade

**Situation** :
1. Database primaire tombe
2. 100 alertes se déclenchent en cascade

**Inhibitions (configurées à l'avance)** :
```yaml
inhibit_rules:
  - source_match:
      alertname: 'DatabaseDown'
      role: 'primary'
    target_match_re:
      alertname: '(API.*|Backend.*|Frontend.*|Cache.*)'
    equal:
      - 'environment'
```

**Résultat** :
- 1 seule alerte : DatabaseDown
- Toutes les alertes dépendantes inhibées automatiquement
- Focus sur la vraie cause

## Outils de Gestion

### amtool : L'Outil en Ligne de Commande

**Installation** :
```bash
go install github.com/prometheus/alertmanager/cmd/amtool@latest
```

**Configuration** :
```bash
# Créer ~/.config/amtool/config.yml
alertmanager.url: http://localhost:9093
```

**Commandes utiles** :

```bash
# Silences
amtool silence query                    # Liste tous les silences
amtool silence query alertname=HighCPU  # Filtrer les silences
amtool silence add alertname=Test --duration=1h --comment="Test"
amtool silence expire SILENCE_ID        # Supprimer un silence

# Alertes
amtool alert query                      # Toutes les alertes actives
amtool alert query severity=critical    # Filtrer par label

# Configuration
amtool config routes                    # Afficher les routes
amtool config routes test \             # Tester le routing
  severity=critical team=backend

# Check de santé
amtool check-config alertmanager.yml    # Valider une config
```

### Scripts d'Automatisation

**Script de silence automatique pour maintenance** :

```bash
#!/bin/bash
# maintenance-silence.sh

INSTANCE=$1
DURATION=$2
TICKET=$3

if [ -z "$INSTANCE" ] || [ -z "$DURATION" ] || [ -z "$TICKET" ]; then
  echo "Usage: $0 <instance> <duration> <ticket>"
  echo "Example: $0 server-01 2h TICKET-1234"
  exit 1
fi

amtool silence add \
  instance="$INSTANCE" \
  --duration="$DURATION" \
  --author="$(whoami)@example.com" \
  --comment="Maintenance $INSTANCE - Ticket $TICKET"

echo "Silence créé pour $INSTANCE pendant $DURATION"
```

**Utilisation** :
```bash
./maintenance-silence.sh server-01 2h TICKET-1234
```

### API Python pour Gérer les Silences

```python
import requests
from datetime import datetime, timedelta
import json

ALERTMANAGER_URL = "http://localhost:9093"

def create_silence(matchers, duration_hours, comment, created_by):
    """Créer un silence dans Alertmanager"""

    now = datetime.utcnow()
    start = now.isoformat() + "Z"
    end = (now + timedelta(hours=duration_hours)).isoformat() + "Z"

    silence = {
        "matchers": matchers,
        "startsAt": start,
        "endsAt": end,
        "createdBy": created_by,
        "comment": comment
    }

    response = requests.post(
        f"{ALERTMANAGER_URL}/api/v2/silences",
        json=silence
    )

    if response.status_code == 200:
        silence_id = response.json()["silenceID"]
        print(f"✓ Silence créé: {silence_id}")
        return silence_id
    else:
        print(f"✗ Erreur: {response.text}")
        return None

def delete_silence(silence_id):
    """Supprimer un silence"""
    response = requests.delete(
        f"{ALERTMANAGER_URL}/api/v2/silence/{silence_id}"
    )

    if response.status_code == 200:
        print(f"✓ Silence {silence_id} supprimé")
        return True
    else:
        print(f"✗ Erreur: {response.text}")
        return False

def list_silences():
    """Lister tous les silences actifs"""
    response = requests.get(f"{ALERTMANAGER_URL}/api/v2/silences")

    if response.status_code == 200:
        silences = response.json()
        print(f"Silences actifs: {len(silences)}")
        for s in silences:
            print(f"  - {s['id']}: {s['comment']}")
        return silences
    else:
        print(f"✗ Erreur: {response.text}")
        return []

# Exemple d'utilisation
if __name__ == "__main__":
    # Créer un silence pour maintenance
    matchers = [
        {"name": "alertname", "value": "HighCPU", "isRegex": False},
        {"name": "instance", "value": "server-01", "isRegex": False}
    ]

    silence_id = create_silence(
        matchers=matchers,
        duration_hours=2,
        comment="Maintenance server-01 - Ticket #1234",
        created_by="automation@example.com"
    )

    # Lister les silences
    list_silences()
```

## Métriques et Monitoring

### Métriques Importantes

**Alertmanager expose des métriques sur les silences et inhibitions** :

```promql
# Nombre de silences actifs
alertmanager_silences{state="active"}

# Nombre d'alertes inhibées
count(ALERTS{alertstate="suppressed"})

# Alertes supprimées par les silences
count(ALERTS{silenced="true"})

# Ratio d'alertes inhibées
count(ALERTS{alertstate="suppressed"}) / count(ALERTS) * 100
```

### Alertes sur les Silences

Créez des alertes pour monitorer l'utilisation des silences :

```yaml
groups:
  - name: alertmanager_health
    rules:
      - alert: TooManySilences
        expr: alertmanager_silences{state="active"} > 10
        for: 30m
        labels:
          severity: warning
        annotations:
          summary: "Trop de silences actifs"
          description: "{{ $value }} silences actifs - vérifier si normal"

      - alert: LongRunningSilence
        expr: |
          time() - alertmanager_silence_start_timestamp > 86400
        for: 1h
        labels:
          severity: warning
        annotations:
          summary: "Silence actif depuis > 24h"
          description: "Silence {{ $labels.id }} actif depuis longtemps"

      - alert: HighInhibitionRate
        expr: |
          count(ALERTS{alertstate="suppressed"})
          /
          count(ALERTS) * 100 > 50
        for: 15m
        labels:
          severity: info
        annotations:
          summary: "Taux d'inhibition élevé"
          description: "{{ $value }}% des alertes sont inhibées"
```

## Troubleshooting

### Problème : Les Silences ne Fonctionnent Pas

**Symptômes** :
- Silence créé mais alertes toujours notifiées
- Matchers ne correspondent pas

**Vérifications** :

1. **Vérifier les matchers** :
```bash
# Voir les labels de l'alerte
curl http://localhost:9093/api/v2/alerts | jq '.[] | .labels'

# Comparer avec les matchers du silence
amtool silence query
```

2. **Vérifier l'orthographe** :
```yaml
# ✗ Mauvais
{"name": "alertName", "value": "HighCPU"}

# ✓ Bon
{"name": "alertname", "value": "HighCPU"}
```

3. **Vérifier la période** :
```bash
# Le silence est-il actif maintenant ?
amtool silence query
```

4. **Logs Alertmanager** :
```bash
microk8s kubectl logs -n monitoring alertmanager-0 | grep -i silence
```

### Problème : Les Inhibitions ne S'Appliquent Pas

**Symptômes** :
- Alertes en cascade toujours notifiées
- Inhibition configurée mais inefficace

**Vérifications** :

1. **Vérifier les labels 'equal'** :
```bash
# Les alertes source et cible ont-elles les mêmes valeurs pour les labels 'equal' ?
curl http://localhost:9093/api/v2/alerts | \
  jq '.[] | select(.labels.alertname == "NodeDown" or .labels.alertname == "PodNotReady") | .labels'
```

2. **Vérifier la syntaxe** :
```yaml
# ✗ Mauvais (target_match avec regex sans _re)
target_match:
  alertname: 'Pod.*'

# ✓ Bon
target_match_re:
  alertname: 'Pod.*'
```

3. **Tester la configuration** :
```bash
# Valider la syntaxe
amtool check-config alertmanager.yml

# Voir les inhibitions actives
curl http://localhost:9093/api/v2/status | jq '.config.inhibit_rules'
```

4. **Logs détaillés** :
```bash
microk8s kubectl logs -n monitoring alertmanager-0 | grep -i inhibit
```

### Problème : Trop d'Alertes Inhibées

**Symptômes** :
- 80%+ des alertes inhibées
- Alertes importantes manquées

**Diagnostic** :

```promql
# Vérifier le ratio
count(ALERTS{alertstate="suppressed"}) / count(ALERTS) * 100
```

**Solutions** :

1. Revoir vos règles d'inhibition (trop agressives ?)
2. Revoir vos règles d'alerte (trop d'alertes redondantes ?)
3. Améliorer la hiérarchie des alertes

## Conclusion

Les silences et inhibitions sont deux outils complémentaires essentiels pour gérer intelligemment vos alertes :

**Silences** : Pour les situations temporaires et planifiées
- Maintenances
- Investigations
- Déploiements
- Tests

**Inhibitions** : Pour les situations récurrentes et prévisibles
- Alertes en cascade
- Hiérarchie d'infrastructure
- Dépendances entre services

Bien utilisés, ils transforment un système d'alerting bruyant en un assistant intelligent qui ne vous dérange que quand c'est vraiment nécessaire.

## Points Clés à Retenir

✓ Silences = désactivation temporaire et manuelle des alertes

✓ Inhibitions = suppression automatique d'alertes redondantes

✓ Toujours documenter les silences avec un commentaire détaillé

✓ Utiliser des durées de silence raisonnables (quelques heures max)

✓ Configurer les inhibitions pour éviter les alertes en cascade

✓ Le paramètre 'equal' dans les inhibitions garantit que les alertes sont liées

✓ Tester les inhibitions avant de les déployer en production

✓ Monitorer l'utilisation des silences et l'efficacité des inhibitions

✓ Ne jamais masquer définitivement un problème avec des silences

✓ Les inhibitions sont permanentes, les silences sont temporaires

---

**Prochaine section** : 14.7 Bonnes pratiques d'alerting - Nous consoliderons tout ce que nous avons appris avec des recommandations complètes pour un système d'alerting optimal.

⏭️ [Bonnes pratiques d'alerting](/14-alerting-et-notifications/07-bonnes-pratiques-dalerting.md)
