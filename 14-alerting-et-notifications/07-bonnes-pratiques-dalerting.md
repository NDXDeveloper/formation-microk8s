🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 14.7 Bonnes Pratiques d'Alerting

## Introduction

Après avoir exploré les concepts, la configuration et les outils d'alerting, il est temps de consolider tout ce savoir en un ensemble de **bonnes pratiques** éprouvées. Cette section est un guide pour créer et maintenir un système d'alerting qui soit :

- **Efficace** : Détecte les problèmes réels rapidement
- **Fiable** : Ne rate pas les incidents critiques
- **Non-intrusif** : Ne submerge pas les équipes
- **Actionnable** : Chaque alerte nécessite et permet une action
- **Évolutif** : S'adapte à la croissance de votre infrastructure

Les bonnes pratiques présentées ici sont le fruit de l'expérience collective de la communauté DevOps/SRE et s'appliquent aussi bien aux petites équipes qu'aux grandes organisations.

## Philosophie Générale

### Le Principe Fondamental : "Alertez sur les Symptômes, pas les Causes"

**Mauvais exemple** : Alerter parce qu'un pod redémarre
- Cause technique : Pod restart
- Impact utilisateur : Possiblement aucun (restart normal)

**Bon exemple** : Alerter parce que le taux d'erreur API augmente
- Symptôme : Taux d'erreur 5xx élevé
- Impact utilisateur : Requêtes qui échouent

**Principe** :
```
Alertes basées sur l'impact utilisateur > Alertes basées sur l'état interne
```

Les utilisateurs se soucient que le service fonctionne, pas des détails de son fonctionnement interne.

### La Règle d'Or : Une Alerte = Une Action

Chaque alerte qui se déclenche devrait :

1. **Nécessiter une action humaine** : Quelque chose doit être fait
2. **Permettre une action** : On sait quoi faire
3. **Être urgente** : Nécessite une intervention maintenant (ou bientôt)

**Test simple** : Si vous recevez une alerte et que votre première réaction est "je verrai ça demain", alors ce n'est pas une alerte - c'est un rapport.

### Les Quatre Questions Magiques

Avant de créer une alerte, posez-vous ces questions :

**1. Y a-t-il un impact utilisateur réel ou imminent ?**
- ✓ Oui → C'est probablement une bonne alerte
- ✗ Non → Peut-être un dashboard ou une métrique suffit

**2. Quelqu'un doit-il être réveillé à 3h du matin pour ça ?**
- ✓ Oui → Alerte critical, notification immédiate
- ✗ Non → Alerte warning, notification pendant les heures de travail

**3. Peut-on automatiser la résolution ?**
- ✓ Oui → Automatisez plutôt que d'alerter
- ✗ Non → L'alerte est justifiée

**4. Y a-t-il assez d'informations pour agir ?**
- ✓ Oui → Incluez ces infos dans les annotations
- ✗ Non → Complétez l'alerte avec un runbook

## Création de Bonnes Règles d'Alerte

### 1. Nommage Cohérent

**Convention recommandée** :
```
[Condition][Component][Métrique]

Exemples :
- HighPodMemoryUsage
- LowNodeDiskSpace
- CriticalDatabaseConnectionFailure
- SlowAPIResponseTime
```

**Bonnes pratiques** :
- CamelCase (pas de tirets, underscores)
- Commence par la condition (High, Low, Critical, Slow, etc.)
- Inclut le composant (Pod, Node, Database, API)
- Descriptif sans être verbeux

**✓ Bons noms** :
- `HighMemoryUsage`
- `ServiceDown`
- `CertificateExpiringSoon`

**✗ Mauvais noms** :
- `alert1`
- `problem`
- `check_this`
- `URGENT!!!`

### 2. Seuils Appropriés

**Méthode pour définir les seuils** :

**Étape 1 : Observer**
- Collectez des métriques pendant 1-2 semaines
- Identifiez les valeurs normales
- Notez les pics et creux

**Étape 2 : Analyser**
```
Valeur normale moyenne : 40%
Pic maximum observé : 75%
Pic d'utilisation acceptable : 80%
```

**Étape 3 : Définir**
- **Warning** : 80% (au-dessus du pic normal, marge confortable)
- **Critical** : 95% (proche de la limite, action urgente)

**Étape 4 : Ajuster**
- Observez les alertes pendant 1 semaine
- Trop d'alertes ? Augmentez les seuils
- Pas assez ? Diminuez-les

### 3. Durée "for" Appropriée

Le paramètre `for` évite les fausses alertes dues à des pics temporaires.

**Règle générale** :

| Sévérité | Durée recommandée | Raison |
|----------|-------------------|--------|
| Critical | 1-5 minutes | Besoin de réaction immédiate |
| Warning | 5-15 minutes | Problème qui persiste |
| Info | 30-60 minutes | Tendance à long terme |

**Exemples** :

```yaml
# Service complètement down → Réaction immédiate
- alert: ServiceDown
  expr: up == 0
  for: 1m

# Utilisation mémoire élevée → Laisser le temps au GC
- alert: HighMemory
  expr: memory_usage > 80
  for: 10m

# Tendance à long terme → Observer plus longtemps
- alert: DiskFillRate
  expr: predict_linear(disk_free[1h], 24*3600) < 0
  for: 1h
```

### 4. Labels et Annotations Complets

**Labels essentiels** :
```yaml
labels:
  severity: critical|warning|info      # OBLIGATOIRE
  component: pod|node|service|network  # Recommandé
  team: backend|frontend|infra         # Si multi-équipes
  environment: production|staging|dev  # Si multi-environnements
```

**Annotations essentielles** :
```yaml
annotations:
  summary: "Description courte (1 ligne)"
  description: "Description détaillée avec contexte et valeurs"
  runbook_url: "https://wiki.example.com/runbooks/alert-name"
  dashboard_url: "https://grafana.example.com/d/xxx"
  impact: "Impact sur les utilisateurs ou le business"
  action: "Premières actions à entreprendre"
```

**Exemple complet** :
```yaml
- alert: HighPodMemoryUsage
  expr: |
    (container_memory_working_set_bytes / container_spec_memory_limit_bytes) > 0.9
  for: 10m
  labels:
    severity: warning
    component: pod
    team: backend
  annotations:
    summary: "Pod {{ $labels.pod }} utilise {{ $value | humanizePercentage }} de sa mémoire"
    description: |
      Le pod {{ $labels.pod }} dans le namespace {{ $labels.namespace }}
      utilise {{ $value | humanizePercentage }} de sa limite mémoire
      depuis 10 minutes.

      Risque d'OOMKill si l'utilisation continue d'augmenter.
    runbook_url: "https://wiki.example.com/runbooks/high-memory"
    dashboard_url: "https://grafana.example.com/d/pods?var-pod={{ $labels.pod }}"
    impact: "Possible dégradation des performances, risque de crash"
    action: |
      1. Vérifier les logs du pod pour des fuites mémoire
      2. Vérifier si c'est lié à une charge inhabituelle
      3. Envisager de scaler ou d'augmenter les limites mémoire
```

### 5. Alertes Multi-niveaux

Pour les problèmes qui évoluent, créez plusieurs niveaux d'alerte :

```yaml
# Niveau 1 : Avertissement précoce
- alert: HighDiskUsage
  expr: disk_used_percent > 80
  for: 30m
  labels:
    severity: warning
  annotations:
    summary: "Disque {{ $labels.device }} à {{ $value }}%"
    action: "Surveiller et planifier un nettoyage"

# Niveau 2 : Situation sérieuse
- alert: VeryHighDiskUsage
  expr: disk_used_percent > 90
  for: 10m
  labels:
    severity: warning
  annotations:
    summary: "Disque {{ $labels.device }} à {{ $value }}%"
    action: "Action nécessaire dans les prochaines heures"

# Niveau 3 : Critique
- alert: CriticalDiskUsage
  expr: disk_used_percent > 95
  for: 5m
  labels:
    severity: critical
  annotations:
    summary: "CRITIQUE : Disque {{ $labels.device }} à {{ $value }}%"
    action: "ACTION IMMEDIATE - Risque de panne imminente"
```

**Avantages** :
- Détection précoce (warning à 80%)
- Escalade automatique si la situation empire
- Actions adaptées à chaque niveau

### 6. Alertes Basées sur les SLOs

Au lieu d'alerter sur des métriques brutes, alertez sur vos objectifs de service.

**SLO (Service Level Objective)** : Notre API doit répondre en < 500ms dans 99% des cas

**Alerte basée sur le SLO** :
```yaml
- alert: SLOViolationAPILatency
  expr: |
    (
      histogram_quantile(0.99,
        rate(http_request_duration_seconds_bucket[5m])
      ) > 0.5
    )
  for: 10m
  labels:
    severity: warning
  annotations:
    summary: "SLO de latence API violé"
    description: |
      Le 99ème percentile de latence est de {{ $value }}s,
      au-dessus de notre objectif de 0.5s.

      Impact : Dégradation de l'expérience utilisateur.
```

**Avantages** :
- Aligné sur les besoins business
- Plus significatif que "CPU à 75%"
- Focus sur l'impact utilisateur

## Organisation et Gestion

### 1. Structure de Fichiers

Organisez vos règles d'alerte par thématique :

```
/prometheus/rules/
├── node-alerts.yml           # Alertes nœuds Kubernetes
├── pod-alerts.yml            # Alertes pods
├── service-alerts.yml        # Alertes services applicatifs
├── database-alerts.yml       # Alertes bases de données
├── network-alerts.yml        # Alertes réseau
├── storage-alerts.yml        # Alertes stockage
├── security-alerts.yml       # Alertes sécurité
└── business-alerts.yml       # Alertes métier/SLO
```

**Dans chaque fichier** :
```yaml
groups:
  - name: node_alerts
    interval: 30s
    rules:
      - alert: ...
      - alert: ...
```

### 2. Versioning et Documentation

**Git pour les règles d'alerte** :
```bash
/monitoring-config/
├── .git/
├── README.md
├── CHANGELOG.md
├── prometheus/
│   └── rules/
│       ├── node-alerts.yml
│       └── pod-alerts.yml
└── alertmanager/
    ├── config.yml
    └── templates/
```

**Changelog exemple** :
```markdown
## [1.3.0] - 2025-01-20

### Added
- Alerte DatabaseConnectionPoolExhausted
- Alerte CertificateExpiringSoon (30 jours avant expiration)

### Changed
- HighPodMemory : seuil augmenté de 80% à 85%
  Raison : Trop de fausses alertes avec 80%
- ServiceDown : durée réduite de 5m à 2m
  Raison : Besoin de réaction plus rapide

### Removed
- Alerte ObsoleteServiceCheck (service décommissionné)
```

### 3. Hiérarchie des Alertes

Organisez vos alertes en pyramide :

```
                     [Critical - Urgent]
                    /                  \
                 Critical             Critical
              (Infrastructure)      (Application)
                /        \           /          \
           Warning     Warning    Warning     Warning
          (Infra)     (App)      (Perf)      (Resource)
            |           |          |            |
         [Info - Tendances - Capacité - Audit]
```

**Règles** :
- **Critiques** : < 5% de toutes les alertes
- **Warnings** : 15-20% des alertes
- **Infos** : Le reste

Si vous avez plus de 10% d'alertes critiques, vous avez probablement trop d'alertes critiques.

### 4. Documentation : Les Runbooks

**Chaque alerte doit avoir un runbook** qui explique :

1. **Quoi** : Que signifie cette alerte ?
2. **Pourquoi** : Pourquoi c'est important ?
3. **Impact** : Quel est l'impact utilisateur ?
4. **Diagnostic** : Comment investiguer ?
5. **Résolution** : Comment résoudre ?
6. **Prévention** : Comment éviter à l'avenir ?

**Template de runbook** :
```markdown
# Runbook : HighPodMemoryUsage

## Description
Cette alerte se déclenche quand un pod utilise plus de 85% de sa limite mémoire
pendant plus de 10 minutes.

## Impact
- Risque d'OOMKill (Out Of Memory) du pod
- Possible dégradation des performances
- Potentiel crash de l'application

## Diagnostic
1. Vérifier l'utilisation mémoire actuelle :
   ```bash
   kubectl top pod <pod-name> -n <namespace>
   ```

2. Vérifier l'historique dans Grafana :
   [Dashboard Pod Memory](https://grafana.example.com/d/pods)

3. Examiner les logs pour des fuites mémoire :
   ```bash
   kubectl logs <pod-name> -n <namespace> | grep -i "memory\|oom"
   ```

4. Vérifier si c'est corrélé à une charge inhabituelle :
   - Trafic plus élevé ?
   - Nouveau déploiement ?

## Résolution immédiate
Si le pod est en danger imminent (> 95%) :

1. Redémarrer le pod :
   ```bash
   kubectl delete pod <pod-name> -n <namespace>
   ```

2. Scaler horizontalement temporairement :
   ```bash
   kubectl scale deployment <deployment> --replicas=5 -n <namespace>
   ```

## Résolution à moyen terme
1. Si fuite mémoire détectée :
   - Créer un ticket pour l'équipe dev
   - Investiguer avec un profiler mémoire

2. Si limite trop basse :
   - Augmenter la limite mémoire dans le manifeste
   - Déployer avec la nouvelle limite

3. Si charge légitime :
   - Optimiser le code
   - Ou scaler davantage

## Prévention
- Activer les profilers mémoire en dev/staging
- Tests de charge réguliers
- Revue des limites mémoire tous les trimestres

## Contacts
- Équipe propriétaire : Backend Team
- Slack : #backend-support
- Email : backend-team@example.com

## Historique
- 2025-01-15 : Seuil ajusté de 80% à 85%
- 2024-12-01 : Alerte créée
```

## Routage et Notifications

### 1. Stratégie de Routage par Sévérité

**Configuration recommandée** :

```yaml
route:
  receiver: 'default'
  group_by: ['alertname', 'cluster', 'namespace']

  routes:
    # Critical en production : Réveillez-moi !
    - match:
        severity: 'critical'
        environment: 'production'
      receiver: 'pagerduty-oncall'
      group_wait: 10s
      group_interval: 5m
      repeat_interval: 5m
      continue: false

    # Warning en production : Notification Slack
    - match:
        severity: 'warning'
        environment: 'production'
      receiver: 'slack-production'
      group_wait: 1m
      group_interval: 10m
      repeat_interval: 4h

    # Info : Email quotidien
    - match:
        severity: 'info'
      receiver: 'email-daily-digest'
      group_wait: 1h
      group_interval: 24h
      repeat_interval: 24h
```

### 2. Channels Adaptés aux Niveaux

| Sévérité | Canal Principal | Canal Secondaire | Fréquence Répétition |
|----------|-----------------|------------------|----------------------|
| Critical | PagerDuty / Appel téléphonique | Slack + Email | 5 minutes |
| Warning | Slack | Email | 4 heures |
| Info | Email | Slack (optionnel) | 24 heures |

### 3. Éviter la Fatigue d'Alerte

**Symptômes de fatigue** :
- Notifications ignorées
- Temps de réponse augmenté
- Moral en baisse
- Turnover dans l'équipe

**Solutions** :

**1. Audit régulier (mensuel)** :
```sql
-- Alertes qui se déclenchent le plus souvent
SELECT alertname, COUNT(*) as frequency
FROM alerts_history
WHERE created_at > NOW() - INTERVAL '30 days'
GROUP BY alertname
ORDER BY frequency DESC
LIMIT 20;
```

**2. Règle du 80/20** :
- Si 20% des alertes génèrent 80% des notifications
- Concentrez-vous sur ces 20%
- Ajustez les seuils ou supprimez-les

**3. Métriques de qualité** :
```promql
# Ratio alertes actionnées / alertes totales
(alerts_acknowledged / alerts_total) * 100

# Temps moyen avant acquittement
avg(alert_acknowledge_time_seconds)

# Alertes résolues sans action (faux positifs)
count(alerts{resolved="true", action_taken="false"})
```

**4. Objectif** :
- Taux de fausses alertes < 5%
- Temps d'acquittement < 5 minutes pour critical
- 90%+ des alertes nécessitent une action

## Maintenance et Amélioration Continue

### 1. Revue Post-Incident

Après chaque incident, posez ces questions :

**Sur la détection** :
- ☐ Avons-nous été alertés ?
- ☐ Quand ? Était-ce assez tôt ?
- ☐ L'alerte était-elle claire ?
- ☐ Manque-t-il des alertes ?

**Sur la réponse** :
- ☐ Le runbook était-il utile ?
- ☐ Avions-nous les bonnes informations ?
- ☐ Les bonnes personnes ont-elles été notifiées ?

**Sur l'amélioration** :
- ☐ Nouvelles alertes à créer ?
- ☐ Alertes existantes à ajuster ?
- ☐ Runbooks à améliorer ?

**Template de rapport** :
```markdown
## Post-Mortem : [Nom de l'Incident]

Date : 2025-01-20
Durée : 2h15m
Impact : 15% des utilisateurs affectés

### Chronologie
- 14:00 : Début du problème
- 14:05 : Première alerte (HighAPILatency)
- 14:10 : Équipe notifiée
- 16:15 : Problème résolu

### Alerting
✓ Ce qui a bien fonctionné :
- Alerte HighAPILatency détectée rapidement
- Notifications envoyées correctement

✗ Ce qui n'a pas fonctionné :
- Pas d'alerte sur la cause root (DatabaseConnectionPoolExhausted)
- Trop d'alertes en cascade (bruit)

### Actions Alerting
1. [ ] Créer alerte DatabaseConnectionPoolExhausted (seuil: 90%)
2. [ ] Ajouter inhibition : DatabaseDown masque APIDown
3. [ ] Mettre à jour runbook HighAPILatency avec section "Check database"
```

### 2. Audit Trimestriel

Tous les 3 mois, faites un audit complet :

**Checklist d'audit** :

**Alertes actives** :
- ☐ Toutes les alertes ont-elles un runbook ?
- ☐ Y a-t-il des alertes qui ne se déclenchent jamais ? (> 6 mois)
- ☐ Y a-t-il des alertes trop bruyantes ? (> 10/jour)
- ☐ Les seuils sont-ils toujours appropriés ?

**Documentation** :
- ☐ Runbooks à jour ?
- ☐ Contacts à jour ?
- ☐ Procédures d'escalade à jour ?

**Configuration** :
- ☐ Routing optimal ?
- ☐ Grouping efficace ?
- ☐ Inhibitions pertinentes ?

**Métriques** :
- ☐ Temps de détection acceptable ?
- ☐ Temps de réponse acceptable ?
- ☐ Taux de fausses alertes acceptable ?

### 3. Tests Réguliers

**Tests mensuels** :

1. **Test des canaux de notification** :
```bash
# Envoyer une alerte de test
amtool alert add \
  alertname="TestMonthlyNotifications" \
  severity="info" \
  --annotation=summary="Test mensuel - Ignorer" \
  --end=5m
```

Vérifier que vous recevez bien la notification sur tous les canaux.

2. **Test de la procédure d'astreinte** :
- Simuler une alerte critical
- Vérifier que la bonne personne est notifiée
- Vérifier l'escalade si pas de réponse

3. **Test des runbooks** :
- Choisir 2-3 runbooks au hasard
- Les suivre étape par étape
- Noter ce qui est obsolète ou manquant
- Mettre à jour

### 4. Formation Continue

**Pour les nouveaux membres** :

**Semaine 1 : Lecture**
- Documentation du système d'alerting
- Principaux runbooks
- Procédures d'escalade

**Semaine 2 : Observation**
- Observer les alertes qui arrivent
- Comprendre le routing
- Voir comment l'équipe répond

**Semaine 3 : Pratique supervisée**
- Répondre aux alertes info avec supervision
- Créer/modifier des silences
- Utiliser les runbooks

**Semaine 4 : Autonomie**
- Gérer les alertes warning seul
- Être en backup pour les critical

**Pour toute l'équipe** :

**Game days trimestriels** :
- Simuler des incidents
- Tester les alertes et runbooks
- Identifier les gaps
- S'entraîner sans stress

## Anti-patterns à Éviter

### ✗ 1. L'Alerte "Au Cas Où"

**Symptôme** :
```yaml
- alert: JustInCaseAlert
  expr: some_metric > threshold
  annotations:
    summary: "Peut-être un problème"
```

**Problème** : Alerte créée "au cas où" sans réel besoin

**Solution** : Chaque alerte doit avoir un objectif clair et actionnable

### ✗ 2. Le Seuil Magique Sans Justification

**Symptôme** :
```yaml
expr: cpu_usage > 42  # Pourquoi 42 ? Mystère...
```

**Problème** : Seuil choisi arbitrairement

**Solution** : Basez les seuils sur des observations réelles et documentez le choix

### ✗ 3. L'Alerte Descriptive au Lieu d'Actionnable

**Mauvais** :
```yaml
annotations:
  summary: "Le CPU est élevé"
  description: "Le CPU est élevé sur ce serveur"
```

**Bon** :
```yaml
annotations:
  summary: "CPU élevé sur {{ $labels.instance }}"
  description: |
    CPU à {{ $value }}% sur {{ $labels.instance }}.
    Impact : Possible dégradation des performances.
    Action :
    1. Vérifier les processus : kubectl top pods
    2. Vérifier si charge légitime ou problème
    3. Voir runbook : https://wiki.example.com/cpu-high
```

### ✗ 4. Trop d'Alertes Critiques

**Symptôme** : 30% de vos alertes sont "critical"

**Problème** : Si tout est critique, rien n'est critique

**Solution** : Réservez "critical" aux situations nécessitant un réveil nocturne

**Règle** : Si vous ne voulez pas être réveillé à 3h pour ça, ce n'est pas "critical"

### ✗ 5. Alertes Sans Runbook

**Symptôme** :
```yaml
annotations:
  summary: "Problème détecté"
  # Pas de runbook, pas d'action suggérée
```

**Problème** : L'équipe ne sait pas quoi faire

**Solution** : Chaque alerte = runbook obligatoire

### ✗ 6. Copier-Coller Sans Adaptation

**Symptôme** : Copier des alertes d'Internet sans les adapter à votre contexte

**Problème** : Seuils inappropriés, labels incorrects, inapplicable

**Solution** : Comprenez l'alerte, adaptez-la, testez-la

### ✗ 7. L'Alerte Permanente Silencée

**Symptôme** :
```bash
# Silence créé il y a 6 mois, renouvelé tous les mois
amtool silence query | grep "6 months ago"
```

**Problème** : Masquer un problème au lieu de le résoudre

**Solution** : Si une alerte est toujours silencée, soit corrigez le problème, soit supprimez l'alerte

### ✗ 8. Négliger les Inhibitions

**Symptôme** : Recevoir 50 alertes quand un serveur tombe

**Problème** : Pas d'inhibitions configurées

**Solution** : Configurez des inhibitions pour éviter les alertes en cascade

### ✗ 9. Ignorer les Métriques de l'Alerting

**Symptôme** : Ne jamais regarder les statistiques d'alerting

**Problème** : Impossible d'améliorer sans mesurer

**Solution** : Suivez les métriques clés (taux de fausses alertes, temps de réponse, etc.)

### ✗ 10. Une Seule Personne Comprend les Alertes

**Symptôme** : "Demandez à Bob, lui seul sait comment ça marche"

**Problème** : Bus factor = 1

**Solution** : Documentation, formation, rotation des responsabilités

## Checklist de Déploiement

Avant de déployer une nouvelle alerte en production, vérifiez :

### Phase 1 : Conception

- [ ] L'alerte détecte un problème réel qui impacte les utilisateurs
- [ ] L'alerte nécessite une action humaine
- [ ] Le seuil est basé sur des observations réelles
- [ ] La durée "for" évite les fausses alertes
- [ ] Le nom est clair et suit la convention
- [ ] Les labels sont appropriés (severity, team, etc.)

### Phase 2 : Documentation

- [ ] Summary clair et concis
- [ ] Description détaillée avec contexte
- [ ] Runbook créé et complet
- [ ] Dashboard lié (si pertinent)
- [ ] Impact utilisateur documenté
- [ ] Actions suggérées documentées

### Phase 3 : Tests

- [ ] Expression PromQL testée dans Prometheus UI
- [ ] Alerte testée en dev/staging
- [ ] Notification reçue sur tous les canaux configurés
- [ ] Runbook suivi et validé
- [ ] Faux positifs identifiés et corrigés

### Phase 4 : Intégration

- [ ] Routage configuré correctement
- [ ] Grouping approprié
- [ ] Inhibitions ajoutées si nécessaire
- [ ] Équipe notifiée du déploiement
- [ ] Ajouté au versioning (Git)
- [ ] Changelog mis à jour

### Phase 5 : Post-Déploiement

- [ ] Observé pendant 1 semaine
- [ ] Aucune fausse alerte
- [ ] Déclenchements légitimes uniquement
- [ ] Feedback de l'équipe collecté
- [ ] Ajustements faits si nécessaire

## Métriques de Succès

Mesurez la qualité de votre système d'alerting avec ces KPIs :

### 1. Time to Detect (TTD)

Temps entre le début du problème et la première alerte.

**Objectif** : < 1 minute pour critical

**Mesure** :
```promql
avg(alert_start_time - incident_start_time)
```

### 2. Time to Acknowledge (TTA)

Temps entre l'alerte et l'acquittement par un humain.

**Objectif** :
- Critical : < 5 minutes
- Warning : < 30 minutes

**Mesure** :
```promql
avg(alert_acknowledge_time - alert_start_time)
```

### 3. Taux de Fausses Alertes

Pourcentage d'alertes qui se résolvent sans action.

**Objectif** : < 5%

**Mesure** :
```promql
(count(alerts{action_taken="false"}) / count(alerts)) * 100
```

### 4. Taux de Couverture

Pourcentage d'incidents détectés par les alertes.

**Objectif** : > 95%

**Mesure** :
```
(incidents_detected_by_alerts / total_incidents) * 100
```

### 5. Alert Fatigue Index

Nombre moyen d'alertes par jour par personne d'astreinte.

**Objectif** : < 10 alertes/jour

**Mesure** :
```promql
sum(rate(alerts_total[24h])) / count(oncall_people)
```

## Culture d'Équipe

### 1. Blameless Post-Mortems

Quand une alerte rate un incident :
- ✗ Ne blâmez pas la personne qui a configuré l'alerte
- ✓ Cherchez comment améliorer le système
- ✓ Documentez ce qui a été appris
- ✓ Implémentez les améliorations

### 2. Ownership Partagé

- Chaque équipe possède ses alertes
- Revues croisées entre équipes
- Base de connaissance partagée
- Rotation des responsabilités

### 3. Amélioration Continue

**Culture du feedback** :
- Après chaque alerte, se demander "Était-elle utile ?"
- Proposer des améliorations
- Implémenter les suggestions
- Partager les apprentissages

**Rétrospectives régulières** :
```markdown
## Rétrospective Alerting - Janvier 2025

### 😊 Ce qui va bien
- Temps de détection en baisse (TTD : 45s en moyenne)
- Pas de fausses alertes cette semaine
- Nouveaux runbooks bien accueillis

### 😐 Ce qui peut s'améliorer
- Trop d'alertes warning en staging (bruit)
- Runbook DatabaseDown incomplet
- Confusion sur la procédure d'escalade

### 💡 Actions
1. [ ] Revoir les seuils en staging (+20%)
2. [ ] Alice complète le runbook DatabaseDown
3. [ ] Bob documente la procédure d'escalade
4. [ ] Prochain game day : 15 février
```

### 4. Reconnaissance

- Célébrez les bonnes détections
- Remerciez ceux qui améliorent les alertes
- Partagez les success stories
- Valorisez la qualité sur la quantité

## Outils et Ressources

### Outils Recommandés

**Alerting** :
- Prometheus + Alertmanager (ce que nous utilisons)
- Grafana pour la visualisation
- amtool pour l'administration CLI

**Documentation** :
- Wiki interne (Confluence, Notion, etc.)
- Git pour versioning
- Markdown pour les runbooks

**Communication** :
- Slack/Teams pour notifications non-critiques
- PagerDuty/OpsGenie pour astreintes

**Analyse** :
- Grafana dashboards pour métriques d'alerting
- Scripts custom pour analyses avancées

### Templates et Exemples

**Template de règle d'alerte** :
```yaml
- alert: [NomDeLAlerte]
  expr: [Expression PromQL]
  for: [Durée]
  labels:
    severity: [critical|warning|info]
    component: [pod|node|service|...]
    team: [nom-equipe]
  annotations:
    summary: "[Description courte avec variables {{ $labels.xxx }}]"
    description: |
      [Description détaillée]
      Valeur actuelle : {{ $value | humanize }}
      Impact : [Impact utilisateur]
    runbook_url: "[URL du runbook]"
    dashboard_url: "[URL du dashboard]"
    action: |
      [Actions à entreprendre]
```

### Ressources Externes

**Livres** :
- "Site Reliability Engineering" (Google)
- "The Site Reliability Workbook" (Google)
- "Practical Monitoring" (Mike Julian)

**Articles de référence** :
- "My Philosophy on Alerting" (Rob Ewaschuk)
- "Alerting on SLOs" (Google SRE)
- "Alert Fatigue" (PagerDuty)

**Communautés** :
- Prometheus Users mailing list
- CNCF Slack (#prometheus)
- r/kubernetes, r/devops

## Conclusion

Un système d'alerting efficace est un équilibre délicat entre :
- Détecter rapidement les problèmes
- Ne pas submerger les équipes
- Fournir les bonnes informations
- Permettre l'action

Les bonnes pratiques présentées ici sont un guide, pas des règles absolues. Adaptez-les à votre contexte, votre organisation, votre culture. L'important est de :

1. **Commencer simple** : Quelques alertes bien configurées
2. **Mesurer** : Suivez vos métriques de succès
3. **Itérer** : Améliorez continuellement
4. **Apprendre** : Chaque incident est une opportunité
5. **Partager** : La connaissance bénéficie à tous

N'oubliez pas : un bon système d'alerting vous réveille quand c'est nécessaire, et vous laisse dormir quand tout va bien.

## Points Clés à Retenir

✓ Alertez sur l'impact utilisateur, pas sur l'état technique

✓ Une alerte = une action nécessaire et possible

✓ Posez-vous les 4 questions magiques avant chaque alerte

✓ Documentez TOUT : runbooks, seuils, décisions

✓ Testez avant de déployer en production

✓ Auditez régulièrement (mensuellement)

✓ Mesurez la qualité avec des KPIs concrets

✓ Moins de 5% d'alertes critical

✓ Moins de 5% de fausses alertes

✓ Améliorez continuellement basé sur les incidents

✓ Partagez la responsabilité dans l'équipe

✓ Célébrez les succès, apprenez des échecs

---

**Fin du chapitre 14 : Alerting et Notifications**

Vous avez maintenant toutes les clés pour construire un système d'alerting robuste, efficace et humain. La prochaine grande étape de votre parcours Kubernetes sera d'approfondir l'observabilité avancée (logs, traces) ou d'explorer la sécurité de votre cluster.

Bon monitoring ! 🎯

⏭️ [Runbooks et documentation des alertes](/14-alerting-et-notifications/08-runbooks-et-documentation-des-alertes.md)
