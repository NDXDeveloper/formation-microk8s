🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 14. Alerting et Notifications

## Introduction au Chapitre

Bienvenue dans le chapitre 14 de cette formation MicroK8s, consacré à l'**alerting et aux notifications**. Après avoir découvert comment collecter et visualiser vos métriques avec Prometheus et Grafana, vous allez maintenant apprendre à transformer ces données en **actions concrètes**.

L'alerting est le mécanisme qui surveille votre infrastructure en continu et vous avertit automatiquement lorsque quelque chose nécessite votre attention. C'est la différence entre :
- Découvrir un problème quand vos utilisateurs se plaignent (trop tard)
- Être alerté avant même que vos utilisateurs ne remarquent un problème (proactif)

Ce chapitre vous guidera depuis les concepts fondamentaux jusqu'aux pratiques avancées, en vous donnant tous les outils pour créer un système d'alerting efficace, fiable et non-intrusif.

## Pourquoi l'Alerting est Crucial

### Le Problème Sans Alerting

Imaginez votre cluster Kubernetes comme une usine avec des centaines de machines. Sans système d'alerte :

```
Scénario 1 : Nuit calme
23h00 : Un pod commence à consommer trop de mémoire
23h30 : Le pod crash (OOMKill)
00h00 : Le service commence à renvoyer des erreurs
01h00 : Les utilisateurs remarquent le problème
02h00 : Quelqu'un vous envoie un email
08h00 : Vous découvrez l'email au réveil
08h30 : Vous commencez à investiguer
09h00 : Problème résolu

Durée d'interruption : 10 heures
Impact : Utilisateurs très mécontents, perte de revenus
```

### Avec un Système d'Alerting

```
Scénario 2 : Nuit surveillée
23h00 : Un pod commence à consommer trop de mémoire
23h10 : Alerte "HighMemoryUsage" se déclenche
23h11 : Notification reçue sur votre téléphone
23h15 : Vous consultez le dashboard
23h20 : Vous redémarrez le pod
23h25 : Service rétabli

Durée d'interruption : 25 minutes
Impact : Minimal, la plupart des utilisateurs n'ont rien remarqué
```

**Différence** : 10 heures vs 25 minutes. C'est tout l'intérêt de l'alerting.

## Les Bénéfices d'un Bon Système d'Alerting

### 1. Détection Rapide des Problèmes

**Temps de détection réduit** de plusieurs heures à quelques minutes, voire secondes.

Un bon système d'alerting détecte les anomalies avant qu'elles ne deviennent des incidents majeurs :
- Espace disque qui se remplit → Alerte à 80% au lieu d'attendre 100%
- Taux d'erreur qui augmente → Alerte dès +10% au lieu d'attendre la panne totale
- Certificat SSL → Alerte 30 jours avant expiration au lieu de découvrir le jour J

### 2. Réactivité et Disponibilité

**Amélioration de la disponibilité** de votre service grâce à des interventions rapides.

Plus vous détectez tôt, plus vous pouvez agir vite :
- **MTTR** (Mean Time To Repair - Temps moyen de réparation) : Réduit de 80%
- **MTTD** (Mean Time To Detect - Temps moyen de détection) : Réduit de 90%
- **Disponibilité globale** : Amélioration mesurable (ex: passage de 99.5% à 99.9%)

### 3. Tranquillité d'Esprit

**Vous pouvez dormir tranquille** en sachant que vous serez alerté en cas de problème.

L'alternative sans alerting :
- Vérifier manuellement les dashboards toutes les heures
- Stress constant : "Est-ce que tout va bien ?"
- Impossible de prendre des vacances sereinement

Avec l'alerting :
- Le système surveille pour vous 24/7
- Vous êtes notifié uniquement si nécessaire
- Vous pouvez vraiment déconnecter

### 4. Focus sur ce qui Compte

**Concentration sur les problèmes importants** plutôt que sur la surveillance constante.

Sans alerting, vous passez votre temps à :
- Regarder des dashboards en espérant voir un problème
- Vérifier compulsivement l'état de vos services
- Chercher des anomalies dans les métriques

Avec alerting, vous pouvez :
- Vous concentrer sur le développement de nouvelles fonctionnalités
- Travailler sur l'amélioration de l'infrastructure
- Intervenir uniquement quand c'est vraiment nécessaire

### 5. Amélioration Continue

**Apprentissage basé sur les données** : chaque alerte est une opportunité d'améliorer votre système.

Les alertes vous enseignent :
- Quels sont vos points faibles
- Quels seuils ajuster
- Quelles améliorations architecturales apporter
- Comment optimiser vos ressources

## Les Composants du Système d'Alerting

Votre système d'alerting repose sur plusieurs composants qui travaillent ensemble :

### 1. Prometheus : Le Détective

```
┌─────────────────────────────────────────────┐
│            PROMETHEUS                       │
│                                             │
│  Collecte des métriques                     │
│         ↓                                   │
│  Évalue les règles d'alerte                 │
│         ↓                                   │
│  Détecte les anomalies                      │
│         ↓                                   │
│  Déclenche les alertes                      │
└─────────────────────────────────────────────┘
                    ↓
         Envoie vers Alertmanager
```

**Rôle** : Surveiller en permanence vos métriques et détecter quand quelque chose sort de l'ordinaire.

**Exemple** : Prometheus vérifie toutes les minutes si l'utilisation mémoire de vos pods dépasse 85%. Si c'est le cas pendant 10 minutes, il déclenche une alerte.

### 2. Alertmanager : Le Coordinateur

```
┌─────────────────────────────────────────────┐
│          ALERTMANAGER                       │
│                                             │
│  Reçoit les alertes                         │
│         ↓                                   │
│  Regroupe les alertes similaires            │
│         ↓                                   │
│  Applique les silences                      │
│         ↓                                   │
│  Route vers les bons destinataires          │
│         ↓                                   │
│  Envoie les notifications                   │
└─────────────────────────────────────────────┘
                    ↓
        ┌───────────┼───────────┐
        ↓           ↓           ↓
     Email       Slack     PagerDuty
```

**Rôle** : Gérer intelligemment les alertes (grouper, router, notifier) pour éviter le spam et assurer que les bonnes personnes soient informées.

**Exemple** : Si 10 pods d'une même application ont un problème, Alertmanager regroupe ces 10 alertes en une seule notification au lieu de vous envoyer 10 messages séparés.

### 3. Canaux de Notification : Les Messagers

Les moyens par lesquels vous êtes informé :

**Email** 📧
- Universel, tout le monde a un email
- Bon pour les alertes non-urgentes
- Peut contenir beaucoup d'informations

**Slack/Teams** 💬
- Instantané, toute l'équipe voit
- Bon pour les alertes moyennes
- Facilite la collaboration

**PagerDuty/OpsGenie** 📱
- Appels téléphoniques, SMS
- Pour les alertes critiques
- Gestion d'astreinte intégrée

**Webhooks** 🔗
- Intégrations personnalisées
- Automatisation possible
- Flexibilité maximale

### 4. Runbooks : Les Guides

Documentation qui explique comment répondre à chaque alerte :

```
Alerte reçue
    ↓
Ouvrir le runbook
    ↓
Suivre les étapes
    ↓
Problème résolu
```

**Rôle** : Transformer une alerte stressante en processus maîtrisé.

**Exemple** : Quand vous recevez "HighPodMemory", le runbook vous dit exactement quelles commandes exécuter et dans quel ordre.

## Philosophie de l'Alerting

Avant de plonger dans les détails techniques, il est important de comprendre la philosophie d'un bon système d'alerting.

### Principe 1 : Moins, c'est Plus

**Mauvaise approche** : Alerter sur tout
- 200 alertes par jour
- Fatigue d'alerte (alert fatigue)
- Les vraies urgences noyées dans le bruit
- Équipe démotivée, alertes ignorées

**Bonne approche** : Alerter sur ce qui compte
- 5-10 alertes par semaine
- Chaque alerte est importante
- Temps de réaction rapide
- Équipe attentive et réactive

### Principe 2 : Une Alerte = Une Action

Chaque alerte doit nécessiter une action humaine. Sinon, ce n'est pas une alerte, c'est une métrique.

**Mauvais exemple** :
```yaml
Alerte : CPUUtilization
Quand : CPU > 20%
Action : ... aucune, c'est normal
```

**Bon exemple** :
```yaml
Alerte : CriticalCPUUtilization
Quand : CPU > 90% pendant 15 minutes
Action : Scaler l'application ou investiguer
```

### Principe 3 : Alerter sur l'Impact Utilisateur

Alertez sur ce que les utilisateurs ressentent, pas sur les détails techniques internes.

**Moins bon** : "Un pod redémarre"
- Technique, interne
- Peut-être normal (rolling update)
- Pas forcément un problème

**Meilleur** : "Taux d'erreur API > 5%"
- Impact utilisateur direct
- Nécessite une action
- Problème réel

### Principe 4 : Progressive et Préventive

Créez des alertes à plusieurs niveaux pour détecter les problèmes tôt.

```
┌─────────────────────────────────────────┐
│  80% Disque : INFO                      │  ← Première alerte
│  Temps restant : 7 jours                │     (surveillance)
└─────────────────────────────────────────┘
              ↓ Si non résolu
┌─────────────────────────────────────────┐
│  90% Disque : WARNING                   │  ← Deuxième alerte
│  Temps restant : 2 jours                │     (action nécessaire)
└─────────────────────────────────────────┘
              ↓ Si non résolu
┌─────────────────────────────────────────┐
│  95% Disque : CRITICAL                  │  ← Troisième alerte
│  Temps restant : 6 heures               │     (urgent)
└─────────────────────────────────────────┘
```

**Avantage** : Vous avez plusieurs chances de réagir avant que ça devienne critique.

## Le Parcours de ce Chapitre

Ce chapitre est organisé de manière progressive, du plus simple au plus avancé.

### 14.1 Concepts d'Alerting
**Ce que vous apprendrez** :
- Qu'est-ce qu'une alerte ?
- Les états d'une alerte (pending, firing, resolved)
- Les niveaux de sévérité (critical, warning, info)
- Vocabulaire essentiel de l'alerting

**Durée estimée** : 30 minutes de lecture

### 14.2 Prometheus Alertmanager
**Ce que vous apprendrez** :
- Architecture d'Alertmanager
- Installation sur MicroK8s
- Configuration de base
- Interface web et utilisation

**Durée estimée** : 1 heure

### 14.3 Configuration des Règles d'Alerte
**Ce que vous apprendrez** :
- Créer des règles d'alerte dans Prometheus
- Écrire des expressions PromQL pour les alertes
- Exemples de règles courantes (CPU, mémoire, disque, etc.)
- Tester et valider vos règles

**Durée estimée** : 2 heures

### 14.4 Routing et Grouping
**Ce que vous apprendrez** :
- Router les alertes vers les bonnes personnes
- Regrouper les alertes pour éviter le spam
- Gérer les répétitions et les timings
- Stratégies de routage avancées

**Durée estimée** : 1h30

### 14.5 Notifications (Slack, email, webhook)
**Ce que vous apprendrez** :
- Configurer les notifications par email
- Intégrer avec Slack
- Utiliser les webhooks pour des intégrations personnalisées
- Services d'incident management (PagerDuty, OpsGenie)

**Durée estimée** : 1h30

### 14.6 Silences et Inhibitions
**Ce que vous apprendrez** :
- Désactiver temporairement des alertes (silences)
- Supprimer automatiquement les alertes redondantes (inhibitions)
- Gérer les maintenances planifiées
- Éviter les alertes en cascade

**Durée estimée** : 1 heure

### 14.7 Bonnes Pratiques d'Alerting
**Ce que vous apprendrez** :
- Comment créer de bonnes alertes
- Anti-patterns à éviter
- Organiser et maintenir vos règles
- Mesurer la qualité de votre alerting
- Culture d'équipe autour de l'alerting

**Durée estimée** : 1h30

### 14.8 Runbooks et Documentation
**Ce que vous apprendrez** :
- Créer des runbooks efficaces
- Templates et organisation
- Maintenir la documentation à jour
- Automatisation des runbooks

**Durée estimée** : 1 heure

**Durée totale du chapitre** : 10-12 heures (réparties sur plusieurs sessions)

## Prérequis

Avant de commencer ce chapitre, vous devriez :

### Connaissances Requises

✓ **Chapitre 12 terminé** : Monitoring avec Prometheus
- Comprendre comment Prometheus collecte les métriques
- Savoir écrire des requêtes PromQL basiques
- Connaître les types de métriques (counter, gauge, histogram)

✓ **Chapitre 13 terminé** : Visualisation avec Grafana
- Savoir créer des dashboards
- Interpréter des graphiques de métriques
- Comprendre les alertes visuelles

✓ **Compétences Kubernetes de base**
- Utiliser kubectl
- Comprendre les pods, deployments, services
- Naviguer dans les namespaces

### Installation MicroK8s

✓ **Addon Prometheus activé** :
```bash
microk8s enable prometheus
```

Cet addon installe automatiquement :
- Prometheus Server
- Prometheus Alertmanager
- Grafana
- Node Exporter
- Kube-state-metrics

### Vérification Rapide

Avant de continuer, vérifiez que tout fonctionne :

```bash
# Vérifier les pods du monitoring
microk8s kubectl get pods -n monitoring

# Devrait afficher :
# - prometheus-xxx
# - alertmanager-xxx
# - grafana-xxx
# - node-exporter-xxx
```

Si tous les pods sont "Running", vous êtes prêt à commencer !

## Ce que Vous Saurez Faire à la Fin

À l'issue de ce chapitre, vous serez capable de :

### Compétences Techniques

✅ **Créer des alertes** adaptées à votre infrastructure
- Règles pour les pods, nœuds, services
- Alertes sur les métriques applicatives
- Alertes basées sur les SLO (Service Level Objectives)

✅ **Configurer Alertmanager** de A à Z
- Routage des alertes
- Grouping et timings
- Intégrations multiples (email, Slack, PagerDuty)

✅ **Gérer les notifications** intelligemment
- Éviter le spam d'alertes
- Silences pour les maintenances
- Inhibitions pour les alertes redondantes

✅ **Documenter et maintenir** votre système d'alerting
- Créer des runbooks efficaces
- Organiser vos règles d'alerte
- Améliorer continuellement

### Compétences Opérationnelles

✅ **Diagnostiquer** rapidement les problèmes
- Comprendre ce qui déclenche une alerte
- Suivre une procédure de résolution
- Escalader si nécessaire

✅ **Optimiser** votre temps et celui de votre équipe
- Réduire le temps de détection des problèmes
- Réagir plus rapidement aux incidents
- Dormir tranquille en sachant que vous serez alerté

✅ **Construire** un système d'alerting professionnel
- Comparable aux entreprises tech leaders
- Scalable et maintenable
- Adapté à la croissance

## Votre Lab Personnel

Ce chapitre est particulièrement adapté à un lab personnel car :

### Coût Minimal

**Ressources nécessaires** :
- Prometheus : ~500MB RAM
- Alertmanager : ~100MB RAM
- Total : Peut tourner sur un petit serveur

**Pas de coûts externes** :
- Pas besoin d'abonnement PagerDuty pour apprendre
- Email et Slack gratuits
- Webhooks locaux pour tester

### Apprentissage Pratique

Vous pouvez :
- Créer des alertes "factices" pour tester
- Simuler des problèmes sans risque
- Expérimenter avec les configurations
- Casser et réparer sans conséquences

**Exemple de test** :
```bash
# Créer un pod qui consomme beaucoup de CPU pour tester l'alerte
kubectl run cpu-stress --image=progrium/stress -- --cpu 2
```

### Transférable au Travail

Les compétences acquises sont **directement applicables** :
- Mêmes outils (Prometheus, Alertmanager)
- Mêmes concepts (routing, grouping, silences)
- Mêmes bonnes pratiques
- Expérience valorisable sur un CV

## État d'Esprit pour ce Chapitre

### Patience et Itération

L'alerting est un **processus itératif**. Vous n'aurez pas le système parfait du premier coup :

**Première version** (Jour 1) :
- Quelques alertes basiques
- Notifications par email
- C'est déjà bien !

**Amélioration progressive** (Semaines 2-4) :
- Ajustement des seuils (trop ou pas assez d'alertes)
- Ajout de nouveaux canaux de notification
- Création de runbooks

**Maturité** (Mois 2-3) :
- Système stable et fiable
- Alertes pertinentes et actionnables
- Équipe confiante

### Apprendre de ses Erreurs

Vous allez probablement :
- Créer des alertes trop sensibles (spam)
- Ou trop peu sensibles (rate des problèmes)
- Oublier de documenter une alerte
- Mal configurer un canal de notification

**C'est normal et attendu !** Chaque erreur est une occasion d'apprendre et d'améliorer.

### Commencer Simple

Ne cherchez pas la perfection dès le début :

**❌ À éviter** :
- Créer 50 alertes le premier jour
- Configuration ultra-complexe d'Alertmanager
- Intégration avec 10 outils différents

**✅ Recommandé** :
- Commencer avec 3-5 alertes essentielles
- Configuration simple d'Alertmanager
- Un canal de notification (email ou Slack)
- Améliorer progressivement

## Ressources Complémentaires

Tout au long de ce chapitre, vous trouverez des liens vers :

**Documentation officielle** :
- Prometheus Alerting : https://prometheus.io/docs/alerting/
- Alertmanager : https://prometheus.io/docs/alerting/alertmanager/

**Communauté** :
- CNCF Slack #prometheus
- Forums et discussions
- GitHub issues pour les problèmes

**Exemples** :
- Bibliothèques de règles d'alerte
- Templates Alertmanager
- Runbooks open source

## Prêt à Commencer ?

L'alerting est un sujet vaste, mais ce chapitre vous guidera pas à pas. Vous commencerez par comprendre les concepts fondamentaux, puis vous apprendrez à configurer et optimiser chaque composant.

À la fin, vous disposerez d'un système d'alerting robuste qui vous permettra de :
- Détecter les problèmes avant vos utilisateurs
- Réagir rapidement et efficacement
- Dormir tranquille
- Vous concentrer sur ce qui compte vraiment

**Objectif final** : Un système d'alerting qui travaille pour vous, pas contre vous.

Prêt ? Alors passons à la première section : **14.1 Concepts d'Alerting** !

---

## Aperçu Visuel du Chapitre

```
CHAPITRE 14 : ALERTING ET NOTIFICATIONS
═══════════════════════════════════════════════════════════

┌─────────────────────────────────────────────────────────┐
│  14.1 Concepts d'Alerting                               │
│  ├─ Qu'est-ce qu'une alerte ?                           │
│  ├─ États et cycle de vie                               │
│  └─ Niveaux de sévérité                                 │
└─────────────────────────────────────────────────────────┘
                      ↓
┌─────────────────────────────────────────────────────────┐
│  14.2 Prometheus Alertmanager                           │
│  ├─ Architecture                                        │
│  ├─ Installation MicroK8s                               │
│  └─ Configuration de base                               │
└─────────────────────────────────────────────────────────┘
                      ↓
┌─────────────────────────────────────────────────────────┐
│  14.3 Configuration des Règles d'Alerte                 │
│  ├─ Écrire des règles                                   │
│  ├─ Expressions PromQL                                  │
│  └─ Exemples courants                                   │
└─────────────────────────────────────────────────────────┘
                      ↓
┌─────────────────────────────────────────────────────────┐
│  14.4 Routing et Grouping                               │
│  ├─ Router vers les bonnes personnes                    │
│  ├─ Regrouper pour éviter le spam                       │
│  └─ Gérer les timings                                   │
└─────────────────────────────────────────────────────────┘
                      ↓
┌─────────────────────────────────────────────────────────┐
│  14.5 Notifications                                     │
│  ├─ Email                                               │
│  ├─ Slack                                               │
│  └─ Webhooks et autres                                  │
└─────────────────────────────────────────────────────────┘
                      ↓
┌─────────────────────────────────────────────────────────┐
│  14.6 Silences et Inhibitions                           │
│  ├─ Désactiver temporairement                           │
│  └─ Supprimer les redondances                           │
└─────────────────────────────────────────────────────────┘
                      ↓
┌─────────────────────────────────────────────────────────┐
│  14.7 Bonnes Pratiques                                  │
│  ├─ Créer de bonnes alertes                             │
│  ├─ Anti-patterns à éviter                              │
│  └─ Amélioration continue                               │
└─────────────────────────────────────────────────────────┘
                      ↓
┌─────────────────────────────────────────────────────────┐
│  14.8 Runbooks et Documentation                         │
│  ├─ Créer des runbooks                                  │
│  ├─ Organiser la documentation                          │
│  └─ Maintenir à jour                                    │
└─────────────────────────────────────────────────────────┘
                      ↓
              🎯 SYSTÈME D'ALERTING
                 OPÉRATIONNEL
```

---

**Prochaine section** : [14.1 Concepts d'Alerting](14.1-concepts-alerting.md)

Bon apprentissage ! 🚀📊

⏭️ [Concepts d'alerting](/14-alerting-et-notifications/01-concepts-dalerting.md)
