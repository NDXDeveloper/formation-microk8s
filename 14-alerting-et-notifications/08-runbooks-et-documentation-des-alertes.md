🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 14.8 Runbooks et Documentation des Alertes

## Introduction

Un **runbook** (aussi appelé playbook ou procédure opérationnelle) est un document qui explique **comment répondre à une alerte spécifique**. C'est votre guide de dépannage pas à pas, votre mode d'emploi en cas de problème.

Imaginez que vous êtes d'astreinte à 3h du matin. Votre téléphone sonne : alerte critique. Vous êtes encore à moitié endormi. Un bon runbook est ce qui vous permet de résoudre le problème efficacement, même dans cet état.

**Sans runbook** :
```
🚨 Alerte : HighPodMemoryUsage
🤔 Qu'est-ce que ça veut dire ?
🤔 Qu'est-ce que je dois faire ?
🤔 Est-ce urgent ?
😰 Panique...
```

**Avec runbook** :
```
🚨 Alerte : HighPodMemoryUsage
📖 Ouvre le runbook
✅ Étape 1 : Vérifier l'utilisation actuelle → OK
✅ Étape 2 : Examiner les logs → Fuite mémoire détectée
✅ Étape 3 : Redémarrer le pod → Problème résolu
😌 Retour au lit
```

Dans cette section, nous allons explorer comment créer, organiser et maintenir des runbooks efficaces pour votre système d'alerting.

## Pourquoi les Runbooks sont Essentiels

### 1. Réduction du Stress

**Sans runbook** : Chaque alerte est une source de stress et d'incertitude

**Avec runbook** : Processus clair, pas de panique, confiance

### 2. Temps de Résolution

**Sans runbook** : 30-60 minutes pour investiguer et trouver la solution

**Avec runbook** : 5-15 minutes en suivant les étapes

### 3. Cohérence

**Sans runbook** : Chaque personne résout à sa manière, résultats variables

**Avec runbook** : Même processus pour tous, qualité constante

### 4. Formation

**Sans runbook** : Nouvelles recrues dépendantes des seniors

**Avec runbook** : Autonomie rapide, courbe d'apprentissage réduite

### 5. Amélioration Continue

**Sans runbook** : Connaissances dans les têtes, difficiles à améliorer

**Avec runbook** : Documentation vivante qui s'améliore à chaque incident

### 6. Réduction du Bus Factor

**Bus Factor** : Si une personne clé se fait renverser par un bus, peut-on continuer ?

**Sans runbook** : Bus factor = 1 (seul Bob sait comment réparer)

**Avec runbook** : Bus factor = ∞ (n'importe qui peut suivre le runbook)

## Anatomie d'un Runbook

Un bon runbook contient ces éléments essentiels :

### 1. En-tête Identifiant

```markdown
# Runbook : [Nom de l'Alerte]

**Propriétaire** : [Équipe responsable]
**Dernière mise à jour** : [Date]
**Version** : [Numéro de version]
**Contacts** : [Slack, Email, Téléphone]
```

### 2. Description Rapide

**Quoi** : En une phrase, que signifie cette alerte ?

```markdown
## Description Rapide

Cette alerte se déclenche quand un pod utilise plus de 85%
de sa limite mémoire pendant plus de 10 minutes.
```

### 3. Gravité et Urgence

```markdown
## Gravité

- **Sévérité** : Warning
- **Impact** : Performance dégradée, risque de crash
- **Urgence** : Modérée - Action nécessaire dans l'heure
- **Réveil nocturne** : Non (sauf si devient Critical)
```

### 4. Impact Utilisateur

```markdown
## Impact Utilisateur

**Impact actuel** :
- Possibles ralentissements de l'API
- Latence augmentée sur certaines requêtes

**Impact si non résolu** :
- Crash du pod (OOMKill)
- Indisponibilité temporaire du service
- Requêtes en échec pour les utilisateurs
```

### 5. Symptômes Associés

```markdown
## Symptômes Associés

Vous pourriez aussi observer :
- Latence API augmentée
- Logs avec warnings "Memory pressure"
- Métriques GC (Garbage Collection) élevées
- Messages "container memory limit exceeded"
```

### 6. Triage Initial

**Première évaluation rapide (30 secondes)** :

```markdown
## Triage Initial (30 secondes)

### Vérification Rapide

1. **Quelle est l'utilisation actuelle ?**
   ```bash
   kubectl top pod [POD_NAME] -n [NAMESPACE]
   ```
   - < 90% → Surveillance
   - 90-95% → Action nécessaire
   - > 95% → Action urgente

2. **Combien de pods sont affectés ?**
   ```bash
   kubectl top pods -n [NAMESPACE] | grep -v "0%"
   ```
   - 1 pod → Problème isolé
   - Plusieurs pods → Problème systémique

3. **Y a-t-il un pic de trafic ?**
   - Vérifier dashboard Grafana
   - Si oui → Charge légitime
   - Si non → Possible fuite mémoire
```

### 7. Diagnostic Approfondi

**Investigation détaillée si nécessaire** :

```markdown
## Diagnostic Détaillé

### Étape 1 : Examiner l'Historique

Ouvrir Grafana :
[Dashboard Pod Memory](https://grafana.example.com/d/pods?var-pod={{pod}})

Questions à se poser :
- La mémoire augmente-t-elle linéairement ? (fuite probable)
- Y a-t-il des pics réguliers ? (charge normale)
- Augmentation récente ? (depuis un déploiement ?)

### Étape 2 : Vérifier les Logs

```bash
# Logs récents
kubectl logs [POD_NAME] -n [NAMESPACE] --tail=100

# Rechercher des erreurs mémoire
kubectl logs [POD_NAME] -n [NAMESPACE] | grep -i "memory\|oom\|heap"

# Logs de tous les containers si plusieurs
kubectl logs [POD_NAME] -n [NAMESPACE] --all-containers=true
```

Indices à rechercher :
- "Out of memory" / "OOM"
- "Memory leak"
- "Heap exhausted"
- Stacktraces répétitives

### Étape 3 : Vérifier les Événements Kubernetes

```bash
kubectl get events -n [NAMESPACE] --sort-by='.lastTimestamp' | grep [POD_NAME]
```

Événements importants :
- "OOMKilled" : Pod tué par manque de mémoire
- "FailedScheduling" : Pas assez de mémoire sur les nœuds
- "BackOff" : Redémarrages répétés

### Étape 4 : Comparer avec d'Autres Pods

```bash
# Tous les pods du même deployment
kubectl top pods -n [NAMESPACE] -l app=[APP_LABEL]
```

Question : Ce pod est-il une anomalie ou tous les pods sont affectés ?
```

### 8. Solutions par Scénario

```markdown
## Solutions

### Scénario A : Charge Légitime Élevée

**Symptômes** :
- Pic de trafic visible dans Grafana
- Tous les pods affectés également
- Utilisation augmente puis se stabilise

**Action** :
1. Scaler horizontalement (ajouter des pods) :
   ```bash
   kubectl scale deployment [DEPLOYMENT] --replicas=[N+2] -n [NAMESPACE]
   ```

2. Vérifier que le scaling a fonctionné :
   ```bash
   kubectl get pods -n [NAMESPACE] -l app=[APP_LABEL]
   ```

3. Surveiller pendant 10 minutes
   - Si mémoire diminue → OK
   - Si mémoire continue d'augmenter → Scénario B

**Prévention** :
- Configurer Horizontal Pod Autoscaler (HPA)
- Revoir les limites de ressources si trop basses

---

### Scénario B : Fuite Mémoire Suspectée

**Symptômes** :
- Mémoire augmente linéairement
- Pas de corrélation avec le trafic
- Un seul pod ou quelques pods affectés

**Action Immédiate** :
1. Redémarrer le pod affecté :
   ```bash
   kubectl delete pod [POD_NAME] -n [NAMESPACE]
   ```
   Note : Le deployment va automatiquement recréer le pod

2. Vérifier que le nouveau pod démarre correctement :
   ```bash
   kubectl get pods -n [NAMESPACE] -w
   ```

3. Surveiller l'utilisation mémoire du nouveau pod

**Action à Court Terme** :
1. Créer un ticket pour l'équipe de développement :
   - Titre : "Fuite mémoire suspectée - [POD/SERVICE]"
   - Inclure : Graphiques Grafana, logs, événements
   - Priorité : Selon la fréquence de redémarrage nécessaire

2. Si redémarrages fréquents (> 1/jour) :
   - Augmenter temporairement la limite mémoire
   - Créer un CronJob de redémarrage automatique (solution temporaire)

**Investigation Développeur** :
- Profiler l'application (heap dump)
- Revoir le code pour fuites évidentes
- Vérifier les caches non limités
- Vérifier les connexions non fermées

---

### Scénario C : Limite Trop Basse

**Symptômes** :
- Tous les pods du deployment atteignent la limite
- Utilisation stable proche de la limite
- Application fonctionne normalement sinon

**Action** :
1. Vérifier l'utilisation "normale" sur 7 jours

2. Augmenter la limite mémoire de 20-30% :
   ```bash
   kubectl edit deployment [DEPLOYMENT] -n [NAMESPACE]
   ```

   Modifier :
   ```yaml
   resources:
     limits:
       memory: "1Gi"  # Était 800Mi
     requests:
       memory: "800Mi"  # Était 600Mi
   ```

3. Sauvegarder et attendre le rolling update

4. Ajuster le seuil d'alerte dans Prometheus :
   - Si limite était trop basse, l'alerte se déclenchera toujours
   - Recalculer : 85% de la nouvelle limite

**Prévention** :
- Revoir les limites tous les trimestres
- Baser les limites sur des observations réelles
```

### 9. Commandes Utiles

```markdown
## Commandes de Référence Rapide

### Informations Pod
```bash
# Utilisation CPU/Mémoire en temps réel
kubectl top pod [POD_NAME] -n [NAMESPACE]

# Description complète du pod
kubectl describe pod [POD_NAME] -n [NAMESPACE]

# Manifeste YAML du pod
kubectl get pod [POD_NAME] -n [NAMESPACE] -o yaml

# Logs
kubectl logs [POD_NAME] -n [NAMESPACE] --tail=100 -f
```

### Actions
```bash
# Redémarrer un pod
kubectl delete pod [POD_NAME] -n [NAMESPACE]

# Scaler un deployment
kubectl scale deployment [DEPLOYMENT] --replicas=5 -n [NAMESPACE]

# Éditer un deployment
kubectl edit deployment [DEPLOYMENT] -n [NAMESPACE]

# Rollback si problème après changement
kubectl rollout undo deployment [DEPLOYMENT] -n [NAMESPACE]
```

### Surveillance
```bash
# Surveiller les événements en temps réel
kubectl get events -n [NAMESPACE] -w

# Surveiller les pods en temps réel
kubectl get pods -n [NAMESPACE] -w

# Voir l'historique de déploiement
kubectl rollout history deployment [DEPLOYMENT] -n [NAMESPACE]
```
```

### 10. Escalade

```markdown
## Quand Escalader ?

### Escalade Immédiate (L2/L3)

Si :
- [ ] Solution standard ne fonctionne pas après 30 minutes
- [ ] Problème affecte > 50% des utilisateurs
- [ ] Service complètement down
- [ ] Impact business critique

**Contacter** :
- On-call Senior : +33 6 XX XX XX XX
- Slack : @oncall-senior dans #incidents
- Backup : manager-tech@example.com

### Escalade Planifiée (Ticket)

Si :
- [ ] Fuite mémoire confirmée (nécessite code fix)
- [ ] Problème récurrent (> 3 fois / semaine)
- [ ] Architecture à revoir

**Créer un ticket** :
- JIRA : Projet INFRA
- Priorité : selon l'impact
- Assigner : équipe propriétaire
```

### 11. Post-Résolution

```markdown
## Après Résolution

### 1. Vérification (15 minutes après)

- [ ] Service fonctionne normalement
- [ ] Métriques revenue à la normale
- [ ] Pas d'alertes supplémentaires
- [ ] Utilisateurs non impactés

### 2. Documentation

- [ ] Mettre à jour le ticket avec la résolution
- [ ] Noter le temps de résolution
- [ ] Noter ce qui a fonctionné / pas fonctionné

### 3. Communication

Notifier dans #incidents :
```
✅ Résolu : HighPodMemoryUsage sur payment-service
Cause : Fuite mémoire suite au déploiement v1.2.3
Action : Rollback vers v1.2.2
Durée : 25 minutes
Impact : Aucun (détecté avant impact utilisateur)
```

### 4. Suivi

Si fuite mémoire ou problème récurrent :
- [ ] Créer ticket pour l'équipe dev
- [ ] Planifier investigation approfondie
- [ ] Revoir si changement de configuration nécessaire
```

### 12. Améliorations du Runbook

```markdown
## Notes et Améliorations

### Historique des Incidents

| Date | Cause | Solution | Temps | Notes |
|------|-------|----------|-------|-------|
| 2025-01-20 | Fuite mémoire | Rollback | 25min | Ajouté check version dans triage |
| 2025-01-15 | Charge élevée | Scale x2 | 10min | Configuré HPA après |
| 2025-01-10 | Limite basse | +30% limite | 15min | Revu toutes les limites |

### Améliorations à Faire

- [ ] Ajouter script de diagnostic automatique
- [ ] Créer dashboard dédié "Memory Debugging"
- [ ] Automatiser le scaling pour ce service
- [ ] Améliorer les logs applicatifs

### Retours d'Équipe

*Ajoutez vos commentaires ici après avoir utilisé ce runbook*

- 2025-01-21, Alice : "Section diagnostic très claire, résolu en 10min"
- 2025-01-18, Bob : "Manquait commande pour voir tous les pods, ajoutée"
```

## Templates de Runbooks

### Template 1 : Runbook Basique (Problème Simple)

```markdown
# Runbook : [NomAlerte]

**Propriétaire** : [Équipe]
**Contacts** : [Slack/Email]
**Mise à jour** : [Date]

---

## 🚨 Description Rapide

[En une phrase : que signifie cette alerte ?]

**Sévérité** : [Critical/Warning/Info]
**Action requise dans** : [Immédiat / 1h / 24h]

---

## 🎯 Triage Rapide (30 secondes)

1. **Vérifier** : [Commande de vérification]
2. **Évaluer** : [Critère de gravité]
3. **Décider** : Action immédiate ou investigation ?

---

## 🔍 Diagnostic

### Vérifications Standard
```bash
[Commandes de diagnostic]
```

**À rechercher** :
- [Symptôme 1]
- [Symptôme 2]

---

## ✅ Résolution

### Solution Standard
1. [Étape 1]
2. [Étape 2]
3. [Étape 3]

### Vérification
```bash
[Commandes de vérification]
```

---

## 📞 Escalade

**Si échec après 30 min** : [Contact]

---

## 📝 Post-Résolution

- [ ] Service vérifié
- [ ] Documentation mise à jour
- [ ] Communication faite
```

### Template 2 : Runbook Avancé (Problème Complexe)

```markdown
# Runbook : [NomAlerte]

## Métadonnées

| Propriété | Valeur |
|-----------|--------|
| Équipe propriétaire | [Équipe] |
| Contacts | [Slack], [Email], [Phone] |
| Version | 2.1 |
| Dernière mise à jour | 2025-01-20 |
| Prochaine revue | 2025-04-20 |

---

## 📋 Table des Matières

1. [Description](#description)
2. [Impact](#impact)
3. [Triage](#triage)
4. [Diagnostic](#diagnostic)
5. [Solutions](#solutions)
6. [Escalade](#escalade)
7. [Prévention](#prévention)

---

## 📖 Description

### Vue d'Ensemble
[Description détaillée de l'alerte]

### Contexte Technique
[Architecture, composants impliqués]

### Métriques
- **Seuil Warning** : [Valeur]
- **Seuil Critical** : [Valeur]
- **Durée** : [Temps avant déclenchement]

---

## 💥 Impact

### Impact Utilisateur

**Si Warning** :
- [Impact si non résolu]

**Si Critical** :
- [Impact immédiat]

### Impact Business

- [Conséquences business]
- [SLA concernés]

---

## 🎯 Triage Initial

### Arbre de Décision

```
Vérifier [Métrique X]
├─ Si < [Seuil A] → Scénario A
├─ Si [Seuil A] < X < [Seuil B] → Scénario B
└─ Si > [Seuil B] → Scénario C (Escalade immédiate)
```

### Commandes de Triage
```bash
[Commandes rapides]
```

---

## 🔬 Diagnostic Approfondi

### Phase 1 : Collecte d'Informations
[Quelles données collecter]

### Phase 2 : Analyse
[Comment analyser les données]

### Phase 3 : Identification Cause Root
[Méthodologie]

---

## 🛠️ Solutions

### Solution 1 : [Nom]
**Quand l'utiliser** : [Conditions]

**Étapes** :
1. [Étape détaillée]
2. [Étape détaillée]

**Temps estimé** : [Durée]
**Risques** : [Risques éventuels]

### Solution 2 : [Nom]
[Même structure]

---

## 📞 Escalade

### Niveau 1 (Vous)
[Essayez les solutions standard]

### Niveau 2 (Senior On-call)
**Quand** : Échec après 30 min
**Contact** : [Détails]

### Niveau 3 (Management)
**Quand** : Impact critique > 1h
**Contact** : [Détails]

---

## 🛡️ Prévention

### Court Terme
- [Action 1]
- [Action 2]

### Long Terme
- [Amélioration architecture]
- [Monitoring amélioré]

---

## 📚 Ressources

- Dashboard : [URL]
- Logs : [URL]
- Documentation : [URL]
- Code source : [URL]

---

## 📝 Historique

| Date | Incident | Solution | Durée | Enseignement |
|------|----------|----------|-------|--------------|
| [Date] | [Desc] | [Action] | [Temps] | [Leçon] |
```

### Template 3 : Runbook pour Alerte Métier/SLO

```markdown
# Runbook : Violation SLO [Service]

## SLO Concerné

**SLO** : [Description du SLO]
- **Cible** : [Valeur, ex: 99.9% uptime]
- **Fenêtre** : [Période, ex: 30 jours glissants]
- **Budget d'erreur restant** : [À vérifier dans dashboard]

---

## Contexte Business

### Impact Client
[Qu'est-ce que ça signifie pour nos clients ?]

### Impact Financier
[Pénalités contractuelles ? Perte de revenus ?]

---

## Triage

### 1. Vérifier le Budget d'Erreur

```bash
# Ouvrir le dashboard SLO
[URL Dashboard]
```

Questions :
- Combien de budget reste-t-il ?
- À quelle vitesse le brûlons-nous ?
- Combien de temps avant épuisement ?

### 2. Identifier la Cause

**Causes possibles** :
- [ ] Incident en cours (vérifier #incidents)
- [ ] Dégradation progressive
- [ ] Pic de trafic inhabituel
- [ ] Déploiement récent

---

## Actions par Niveau de Gravité

### 🟢 Budget > 20% : Surveillance

**Action** :
- Surveiller l'évolution
- Investiguer la cause
- Pas d'action urgente

### 🟡 Budget 5-20% : Action Nécessaire

**Actions** :
1. Identifier et corriger la cause root
2. Communication à l'équipe
3. Planifier correction préventive

### 🔴 Budget < 5% : Urgence

**Actions immédiates** :
1. Escalade au management
2. All-hands pour résolution
3. Communication clients si nécessaire
4. Gel des déploiements

---

## Restauration du SLO

[Comment récupérer du budget d'erreur]

---

## Post-Mortem

Si violation du SLO :
- [ ] Post-mortem obligatoire
- [ ] Communication clients
- [ ] Plan d'action correctif
- [ ] Revue des SLO si nécessaire
```

## Organisation des Runbooks

### Structure de Répertoire

```
/runbooks/
├── README.md                    # Index des runbooks
├── _templates/                  # Templates réutilisables
│   ├── basic.md
│   ├── advanced.md
│   └── slo.md
├── infrastructure/
│   ├── high-cpu.md
│   ├── high-memory.md
│   ├── disk-full.md
│   ├── node-down.md
│   └── network-latency.md
├── applications/
│   ├── api-latency.md
│   ├── high-error-rate.md
│   ├── database-connection.md
│   └── cache-failure.md
├── security/
│   ├── unauthorized-access.md
│   ├── certificate-expiring.md
│   └── suspicious-activity.md
└── business/
    ├── slo-violation-api.md
    ├── payment-failure-rate.md
    └── user-signup-drop.md
```

### README.md Principal

```markdown
# Runbooks Index

## Par Sévérité

### 🔴 Critical
- [NodeDown](infrastructure/node-down.md) - Nœud Kubernetes inaccessible
- [DatabaseDown](applications/database-down.md) - Base de données indisponible
- [SLOViolationCritical](business/slo-violation-api.md) - SLO menacé

### 🟡 Warning
- [HighCPU](infrastructure/high-cpu.md) - Utilisation CPU élevée
- [HighMemory](infrastructure/high-memory.md) - Utilisation mémoire élevée
- [APILatency](applications/api-latency.md) - Latence API augmentée

### 🟢 Info
- [DiskSpaceWarning](infrastructure/disk-full.md) - Espace disque faible
- [CertificateExpiring](security/certificate-expiring.md) - Certificat expire bientôt

## Par Composant

### Infrastructure
- [Tous les runbooks infrastructure](infrastructure/)

### Applications
- [Tous les runbooks applications](applications/)

### Sécurité
- [Tous les runbooks sécurité](security/)

## Runbooks Récemment Mis à Jour

- 2025-01-20 : [HighMemory](infrastructure/high-memory.md) - Ajout scénario fuite mémoire
- 2025-01-15 : [APILatency](applications/api-latency.md) - Ajout diagnostic base de données
- 2025-01-10 : [NodeDown](infrastructure/node-down.md) - Procédure escalade mise à jour

## Guide de Contribution

[Comment créer ou modifier un runbook](CONTRIBUTING.md)
```

## Outils pour Runbooks

### 1. Markdown + Git

**Avantages** :
- Simple, léger, universel
- Versioning natif avec Git
- Diff facile à visualiser
- Fonctionne partout

**Structure** :
```bash
git clone git@github.com:company/runbooks.git
cd runbooks
vim infrastructure/new-runbook.md
git add infrastructure/new-runbook.md
git commit -m "Add runbook for XYZ"
git push
```

### 2. Wiki Interne (Confluence, Notion, etc.)

**Avantages** :
- Interface graphique friendly
- Collaboration en temps réel
- Recherche puissante
- Commentaires et discussions

**Organisation** :
```
Space: SRE Runbooks
├─ Page: Index
├─ Page: Infrastructure
│   ├─ Page: High CPU
│   ├─ Page: High Memory
│   └─ ...
├─ Page: Applications
└─ Page: Security
```

### 3. Solution Dédiée (PagerDuty Runbook Automation, etc.)

**Avantages** :
- Intégration avec système d'alerting
- Automatisation possible
- Analytics et metrics

### 4. Approche Hybride (Recommandé)

**Combine le meilleur des deux** :

1. **Source de vérité** : Git + Markdown
   - Versioning
   - Review process
   - CI/CD

2. **Consultation** : Wiki ou plateforme web
   - Interface friendly
   - Recherche
   - Liens depuis les alertes

3. **Pipeline** :
```
Git (runbooks/*.md)
    ↓
CI/CD (GitHub Actions)
    ↓
Wiki (pages générées automatiquement)
```

## Maintenir les Runbooks à Jour

### 1. Processus de Revue

**Revue Après Chaque Incident** :

```markdown
## Post-Incident Checklist

- [ ] Le runbook a-t-il été utile ?
- [ ] Manquait-il des informations ?
- [ ] Les commandes fonctionnent-elles ?
- [ ] La durée estimée est-elle correcte ?
- [ ] Nouvelles solutions découvertes ?

**Actions** :
- [ ] Mettre à jour le runbook
- [ ] Créer une PR avec les changements
- [ ] Ajouter l'incident à l'historique
```

**Revue Trimestrielle** :

```markdown
## Runbook Quarterly Review

Pour chaque runbook :

- [ ] Vérifié par 2 personnes différentes
- [ ] Commandes testées en staging
- [ ] Liens vérifiés (Grafana, etc.)
- [ ] Contacts à jour
- [ ] Historique récent analysé
- [ ] Améliorations identifiées

**Résultat** : ✅ À jour / 🔄 Nécessite mise à jour / ❌ Obsolète
```

### 2. Cycle de Vie

```
Création
   ↓
Validation (test en staging)
   ↓
Déploiement (merge to main)
   ↓
Utilisation active
   ↓
Amélioration continue (après incidents)
   ↓
Revue trimestrielle
   ↓
Mise à jour / Archive / Suppression
```

### 3. Métriques de Qualité

**KPIs pour les Runbooks** :

```yaml
Runbook Quality Metrics:
  - Utilisation: "Combien de fois consulté ?"
  - Efficacité: "Temps de résolution avec vs sans"
  - Satisfaction: "Note de 1 à 5 par les utilisateurs"
  - Fraîcheur: "Dernière mise à jour < 3 mois"
  - Complétude: "Toutes les sections remplies"
```

**Dashboard** :
```
📊 Runbook Health Dashboard

Total: 45 runbooks
✅ À jour (< 3 mois): 38 (84%)
🔄 À revoir (3-6 mois): 5 (11%)
❌ Obsolètes (> 6 mois): 2 (5%)

📈 Utilisation Last 30 Days
1. high-memory.md - 24 consultations
2. api-latency.md - 18 consultations
3. node-down.md - 12 consultations

⭐ Top Rated
1. high-memory.md - 4.8/5
2. database-down.md - 4.7/5
3. disk-full.md - 4.5/5
```

## Bonnes Pratiques

### 1. Écrire pour un Débutant à 3h du Matin

**Principe** : Si un junior d'astreinte, réveillé à 3h, peut suivre le runbook, alors il est bon.

**Implique** :
- Pas d'étapes implicites
- Toutes les commandes complètes (avec namespaces, etc.)
- Pas de jargon sans explication
- Ordre logique, pas de sauts

### 2. Une Action = Une Commande

**✗ Mauvais** :
```
Vérifier l'utilisation mémoire des pods
```

**✓ Bon** :
```
Vérifier l'utilisation mémoire des pods :
```bash
kubectl top pods -n production | sort -k3 -rn | head -10
```
```

### 3. Inclure les Sorties Attendues

**✓ Bon** :
```markdown
Vérifier que le pod est Running :
```bash
kubectl get pod my-pod -n production
```

Sortie attendue :
```
NAME     READY   STATUS    RESTARTS   AGE
my-pod   1/1     Running   0          2m
```

Si différent, voir [Section Troubleshooting](#troubleshooting)
```

### 4. Gestion des Erreurs

**Anticiper les problèmes** :

```markdown
## Commande : Redémarrer le Pod

```bash
kubectl delete pod my-pod -n production
```

**Erreurs possibles** :

### Erreur : "pod not found"
**Cause** : Le pod a déjà été supprimé ou n'existe pas
**Action** : Vérifier que le deployment existe et créera un nouveau pod

### Erreur : "permission denied"
**Cause** : Pas les droits suffisants
**Action** : Contacter l'admin ou utiliser le compte service approprié

### Erreur : "pod is terminating"
**Cause** : Le pod est en cours de suppression
**Action** : Attendre 30s et revérifier
```

### 5. Chemins Conditionnels Clairs

**Utiliser des arbres de décision** :

```markdown
## Diagnostic

1. Vérifier l'utilisation mémoire :
   ```bash
   kubectl top pod my-pod
   ```

   **Si < 85%** :
   → Fausse alerte, surveiller
   → Aller à [Section Surveillance](#surveillance)

   **Si 85-95%** :
   → Action nécessaire
   → Aller à [Solution Standard](#solution-standard)

   **Si > 95%** :
   → URGENT
   → Aller à [Solution Urgente](#solution-urgente)
```

### 6. Liens et Références

**Lier les ressources pertinentes** :

```markdown
## Ressources

- 📊 [Dashboard Grafana](https://grafana.example.com/d/xyz)
- 📖 [Documentation Architecture](https://wiki.example.com/arch/service-x)
- 💬 [Canal Slack](https://slack.com/app_redirect?channel=team-backend)
- 🎫 [Board JIRA](https://jira.example.com/browse/INFRA)
- 👤 [On-Call Schedule](https://pagerduty.com/schedules/xyz)
```

### 7. Contexte Visual

**Inclure des captures d'écran ou diagrammes quand utile** :

```markdown
## Architecture

```
┌──────────────┐
│   Frontend   │
└──────┬───────┘
       │
       ▼
┌──────────────┐     ┌──────────────┐
│   API GW     │────▶│   Database   │
└──────┬───────┘     └──────────────┘
       │
       ▼
┌──────────────┐
│   Cache      │
└──────────────┘
```

**Point de défaillance** : Si Database down, API GW timeout
```

### 8. Timing Réaliste

**Indiquer des durées réalistes** :

```markdown
## Temps Estimés

- **Triage initial** : 30 secondes
- **Diagnostic complet** : 5-10 minutes
- **Solution standard** : 5 minutes
- **Vérification** : 5 minutes
- **Documentation** : 5 minutes

**Total** : 20-30 minutes
```

### 9. Feedback Loop

**Encourager les retours** :

```markdown
---

## 💬 Votre Feedback

Ce runbook vous a-t-il été utile ?

👍 Oui - Excellent
👌 Oui - Mais peut être amélioré
👎 Non - Difficile à suivre

**Commentaires ou suggestions** :
[Créer une issue GitHub](https://github.com/company/runbooks/issues/new)
ou Slack: #runbooks-feedback
```

### 10. Versioning Clair

```markdown
---

## 📌 Historique des Versions

### v2.1 (2025-01-20)
- Ajouté scénario C (fuite mémoire)
- Mis à jour commandes kubectl
- Ajouté dashboard Grafana

### v2.0 (2024-12-15)
- Refonte complète du runbook
- Ajouté arbre de décision
- Nouvelles solutions

### v1.0 (2024-10-01)
- Version initiale
```

## Automatisation des Runbooks

### 1. Scripts Compagnons

Créer des scripts pour automatiser les parties répétitives :

```bash
#!/bin/bash
# scripts/diagnose-high-memory.sh

POD_NAME=$1
NAMESPACE=${2:-production}

echo "🔍 Diagnostic High Memory pour $POD_NAME"
echo "==========================================="

echo -e "\n📊 Utilisation actuelle:"
kubectl top pod $POD_NAME -n $NAMESPACE

echo -e "\n📝 Logs récents (erreurs mémoire):"
kubectl logs $POD_NAME -n $NAMESPACE --tail=50 | grep -i "memory\|oom"

echo -e "\n⚠️  Événements récents:"
kubectl get events -n $NAMESPACE --field-selector involvedObject.name=$POD_NAME

echo -e "\n📈 Dashboard Grafana:"
echo "https://grafana.example.com/d/pods?var-pod=$POD_NAME"

echo -e "\n✅ Diagnostic complet"
```

**Utilisation dans le runbook** :
```markdown
## Diagnostic Automatisé

Exécuter le script de diagnostic :
```bash
./scripts/diagnose-high-memory.sh [POD_NAME] [NAMESPACE]
```

Ce script collecte automatiquement toutes les informations nécessaires.
```

### 2. Runbook Automation Platforms

**Solutions** :
- PagerDuty Runbook Automation
- Rundeck
- Ansible Playbooks
- Custom scripts + ChatOps (Slack bots)

**Exemple avec Slack Bot** :
```
Vous: /runbook high-memory pod=api-123 namespace=prod
Bot: 🤖 Exécution du diagnostic...
     ✅ Utilisation: 92%
     ✅ Logs analysés: Fuite mémoire détectée
     ✅ Solution suggérée: Redémarrage du pod

     Souhaitez-vous que je redémarre le pod ? (oui/non)
Vous: oui
Bot: ✅ Pod redémarré. Nouveau pod: api-456
     📊 Dashboard: [lien]
```

## Conclusion

Les runbooks sont bien plus que de simples documents : ce sont des **multiplicateurs de force** pour votre équipe. Ils transforment le chaos des incidents en processus maîtrisés, réduisent le stress, accélèrent les résolutions et permettent l'amélioration continue.

**Un bon runbook** :
- Est écrit pour être compris à 3h du matin
- Contient des commandes complètes et testées
- Anticipe les erreurs et propose des solutions
- Est maintenu à jour après chaque incident
- S'améliore continuellement

**Investir dans les runbooks**, c'est investir dans :
- La qualité de vie de votre équipe
- La fiabilité de vos services
- La satisfaction de vos utilisateurs
- La résilience de votre organisation

Commencez petit : créez des runbooks pour vos 3-5 alertes les plus fréquentes. Améliorez-les après chaque utilisation. Partagez-les avec l'équipe. Et progressivement, construisez une bibliothèque de connaissances qui fera de votre équipe une force opérationnelle d'élite.

## Points Clés à Retenir

✓ Un runbook = un guide pas à pas pour répondre à une alerte

✓ Écrivez pour un débutant réveillé à 3h du matin

✓ Incluez toutes les commandes complètes et testées

✓ Anticipez les erreurs et les chemins conditionnels

✓ Mettez à jour après CHAQUE incident

✓ Organisez logiquement (triage → diagnostic → solution)

✓ Incluez des timings réalistes

✓ Liez aux ressources (dashboards, docs, contacts)

✓ Versionnez et maintenez à jour (revue trimestrielle)

✓ Collectez le feedback et améliorez continuellement

---

**Fin du chapitre 14 : Alerting et Notifications**

Avec ce chapitre complet, vous disposez maintenant de tous les outils et connaissances pour créer un système d'alerting robuste, bien documenté et efficace. Bonne chance dans la construction de votre infrastructure de monitoring ! 📚🚀

⏭️ [Observabilité Avancée](/15-observabilite-avancee/README.md)
