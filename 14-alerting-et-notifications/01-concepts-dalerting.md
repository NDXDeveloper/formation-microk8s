🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 14.1 Concepts d'Alerting

## Introduction

L'alerting (ou système d'alerte) est un composant essentiel de tout système de monitoring. Imaginez que vous surveillez votre infrastructure Kubernetes 24h/24 : il serait impossible de rester devant vos dashboards en permanence. C'est là qu'intervient l'alerting : il surveille pour vous et vous avertit uniquement lorsque quelque chose nécessite votre attention.

Dans l'écosystème Kubernetes avec Prometheus, l'alerting permet de détecter automatiquement les problèmes, les anomalies ou les situations critiques, et de vous notifier via différents canaux (email, Slack, SMS, etc.).

## Pourquoi l'Alerting est Crucial

Un système d'alerting efficace vous permet de :

- **Réagir rapidement** : Être informé des problèmes avant que vos utilisateurs ne les remarquent
- **Prévenir les incidents majeurs** : Détecter les signes précurseurs d'une panne
- **Optimiser votre temps** : Se concentrer sur les problèmes réels plutôt que de surveiller constamment
- **Améliorer la fiabilité** : Maintenir vos services disponibles et performants
- **Dormir tranquille** : Savoir que vous serez alerté en cas de problème, même la nuit

## Les Composants de l'Alerting dans Prometheus

### 1. Prometheus Server

Le serveur Prometheus collecte les métriques et **évalue les règles d'alerte**. Il vérifie régulièrement (par défaut toutes les minutes) si les conditions définies dans vos règles d'alerte sont remplies.

**Rôle** : Détection des conditions problématiques

### 2. Alertmanager

Alertmanager est un composant séparé qui **gère les alertes** une fois qu'elles sont déclenchées par Prometheus. C'est lui qui décide comment, quand et à qui envoyer les notifications.

**Rôle** : Gestion et routage des notifications

## Anatomie d'une Alerte

Une alerte est composée de plusieurs éléments :

### Le Nom (Alert Name)

Un identifiant unique et descriptif de l'alerte. Par exemple : `HighCPUUsage`, `PodCrashLooping`, `DiskSpaceRunningOut`.

**Bonne pratique** : Utilisez des noms clairs qui décrivent immédiatement le problème.

### La Condition (Expression PromQL)

La règle qui définit quand l'alerte doit se déclencher. Elle utilise le langage PromQL pour interroger les métriques.

Exemple simple :
```
container_cpu_usage_seconds_total > 0.8
```
Cette expression vérifie si l'utilisation CPU d'un conteneur dépasse 80%.

### La Durée (For)

Le temps pendant lequel la condition doit rester vraie avant de déclencher l'alerte. Cela évite les "fausses alertes" causées par des pics temporaires.

Exemple :
```
for: 5m
```
L'alerte ne se déclenche que si la condition reste vraie pendant 5 minutes consécutives.

### Les Labels

Des étiquettes qui permettent de catégoriser et de router les alertes. Les labels courants incluent :

- `severity` : L'importance de l'alerte (critical, warning, info)
- `team` : L'équipe responsable
- `component` : Le composant concerné
- `environment` : L'environnement (production, staging, dev)

### Les Annotations

Des informations supplémentaires destinées aux humains, comme :

- `summary` : Un résumé court du problème
- `description` : Une description détaillée avec des valeurs concrètes
- `runbook_url` : Un lien vers la documentation de résolution

## Les États d'une Alerte

Une alerte passe par différents états dans son cycle de vie :

### 1. Inactive

L'alerte est définie mais sa condition n'est pas remplie. Tout va bien.

### 2. Pending

La condition est remplie, mais la durée définie dans `for` n'est pas encore écoulée. Prometheus surveille la situation.

### 3. Firing

La condition est remplie depuis suffisamment longtemps. L'alerte est déclenchée et envoyée à Alertmanager.

### 4. Resolved

La condition n'est plus remplie. Le problème est résolu. Une notification de résolution peut être envoyée.

## Niveaux de Sévérité

Il est important de classifier vos alertes par niveau de gravité :

### Critical (Critique)

**Quand l'utiliser** : Problème majeur nécessitant une action immédiate, impact sur les utilisateurs.

**Exemples** :
- Service principal complètement indisponible
- Perte de données imminente
- Espace disque critique (>95%)

**Action** : Notification immédiate, intervention urgente, peut réveiller l'équipe d'astreinte.

### Warning (Avertissement)

**Quand l'utiliser** : Situation anormale qui nécessite de l'attention mais pas d'urgence immédiate.

**Exemples** :
- Performance dégradée
- Espace disque élevé (>80%)
- Augmentation du taux d'erreur

**Action** : Notification pendant les heures de travail, investigation nécessaire.

### Info (Information)

**Quand l'utiliser** : Information utile mais qui ne nécessite pas forcément d'action.

**Exemples** :
- Nouveau déploiement effectué
- Scaling automatique déclenché
- Certificat qui expire dans 30 jours

**Action** : Notification passive, pour information.

## Bonnes Pratiques Fondamentales

### 1. Alertez sur les Symptômes, pas les Causes

**Mauvais exemple** : Alerter parce qu'un pod redémarre.
**Bon exemple** : Alerter parce que le taux d'erreur API augmente.

Les utilisateurs se soucient du service qui fonctionne ou non, pas des détails techniques internes.

### 2. Évitez les Alertes Bruyantes (Alert Fatigue)

Si vous recevez trop d'alertes non critiques, vous risquez d'ignorer les vraies urgences. Chaque alerte doit être :
- **Actionnable** : Elle doit nécessiter une action de votre part
- **Pertinente** : Elle doit indiquer un problème réel
- **Non redondante** : Ne pas dupliquer d'autres alertes

### 3. Utilisez des Seuils Appropriés

Des seuils trop bas génèrent des fausses alertes. Des seuils trop hauts vous font rater des problèmes. Ajustez-les en fonction de :
- Vos observations historiques
- Vos SLA (Service Level Agreements)
- Le comportement normal de vos services

### 4. Documentez Vos Alertes

Chaque alerte devrait avoir :
- Une description claire du problème
- Un lien vers un runbook expliquant comment la résoudre
- Des informations de contexte (métriques, logs)

### 5. Testez Vos Alertes

Vérifiez régulièrement que vos alertes fonctionnent :
- Simulez des conditions d'alerte
- Vérifiez que les notifications arrivent
- Testez les différents canaux de notification

## Le Principe des SLI/SLO

### SLI (Service Level Indicator)

Une métrique qui mesure un aspect de votre service. Par exemple :
- Taux de disponibilité : 99.9%
- Temps de réponse : 200ms en moyenne
- Taux d'erreur : 0.1%

### SLO (Service Level Objective)

L'objectif que vous vous fixez pour un SLI. Par exemple :
- "Notre API doit avoir un temps de réponse < 500ms dans 95% des cas"
- "Notre service doit être disponible à 99.5%"

**Alerting basé sur les SLO** : Plutôt que d'alerter sur des métriques techniques, alertez lorsque vous risquez de ne pas atteindre vos SLO. C'est une approche plus orientée utilisateur.

## Stratégies d'Alerting Communes

### Alerting par Seuil

La méthode la plus simple : déclencher une alerte quand une métrique dépasse une valeur fixe.

**Exemple** : Alerte si l'utilisation CPU > 80% pendant 5 minutes.

**Avantages** : Simple à comprendre et à configurer.
**Inconvénients** : Ne s'adapte pas aux variations normales du trafic.

### Alerting par Anomalie

Détecter les comportements inhabituels en comparant avec des patterns historiques.

**Exemple** : Alerte si le nombre de requêtes est 3x supérieur à la moyenne des 7 derniers jours.

**Avantages** : S'adapte aux variations naturelles.
**Inconvénients** : Plus complexe à mettre en place.

### Alerting par Taux de Changement

Détecter les changements soudains plutôt que les valeurs absolues.

**Exemple** : Alerte si le taux d'erreur augmente de 50% en 10 minutes.

**Avantages** : Détecte rapidement les régressions.
**Inconvénients** : Peut être sensible aux variations normales.

## La Chaîne de Responsabilité

Un bon système d'alerting définit clairement :

1. **Qui** doit être notifié
2. **Quand** (en fonction de la sévérité et de l'heure)
3. **Comment** (email, SMS, appel, Slack)
4. **Que faire** (runbook, procédure d'escalade)

### Escalade

Si une alerte critique n'est pas acquittée dans un délai donné, elle peut être automatiquement escaladée vers un niveau supérieur (ex: du développeur au manager, puis à l'équipe d'astreinte).

## Vocabulaire Essentiel

**Alert Rule** : La définition d'une condition qui déclenche une alerte.

**Firing** : État d'une alerte dont la condition est remplie.

**Grouping** : Regroupement de plusieurs alertes similaires en une seule notification.

**Throttling** : Limitation du nombre de notifications pour éviter le spam.

**Silencing** : Désactivation temporaire d'une alerte (pendant une maintenance par exemple).

**Inhibition** : Suppression automatique d'alertes redondantes (ex: si le serveur est down, ne pas alerter sur les services qui tournent dessus).

**Routing** : Direction des alertes vers les bons destinataires selon leurs labels.

## Exemple Conceptuel : Une Alerte Complète

Imaginons une alerte qui détecte un problème de mémoire :

**Nom** : `HighMemoryUsage`

**Condition** : Utilisation mémoire > 85% pendant 10 minutes

**Sévérité** : `warning`

**Description** : "Le pod {{ $labels.pod }} utilise {{ $value }}% de sa mémoire allouée"

**Runbook** : Lien vers la procédure de vérification des fuites mémoire

**Routing** : Notification vers le canal Slack #alerts-infra

**Grouping** : Regroupé avec d'autres alertes du même namespace

**Silence** : Peut être silencé manuellement pendant une investigation

## Conclusion

L'alerting est l'art de transformer vos métriques en notifications actionnables. Un bon système d'alerting vous aide à :

- Détecter les problèmes rapidement
- Réagir de manière appropriée
- Maintenir vos services en bonne santé
- Éviter le stress inutile lié aux fausses alertes

Dans les sections suivantes, nous verrons comment mettre en pratique ces concepts avec Prometheus Alertmanager dans votre cluster MicroK8s.

## Points Clés à Retenir

✓ L'alerting surveille vos services à votre place et vous avertit des problèmes

✓ Prometheus évalue les règles, Alertmanager gère les notifications

✓ Chaque alerte a un nom, une condition, une durée, des labels et des annotations

✓ Les alertes passent par les états : Inactive → Pending → Firing → Resolved

✓ Classez vos alertes par sévérité : Critical, Warning, Info

✓ Alertez sur les symptômes (impact utilisateur) plutôt que les causes techniques

✓ Évitez les alertes bruyantes : chaque alerte doit être actionnable

✓ Documentez vos alertes avec des runbooks

✓ Testez régulièrement votre système d'alerting

---

**Prochaine section** : 14.2 Prometheus Alertmanager - Nous verrons comment installer et configurer Alertmanager pour gérer vos alertes.

⏭️ [Prometheus Alertmanager](/14-alerting-et-notifications/02-prometheus-alertmanager.md)
