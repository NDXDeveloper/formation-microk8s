🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 13.7 Organisation des dashboards

## Introduction

À mesure que votre utilisation de Grafana s'intensifie, vous accumulerez de nombreux dashboards. Sans organisation appropriée, retrouver le bon dashboard au bon moment devient rapidement problématique. Ce chapitre vous apprend à structurer, organiser et naviguer efficacement dans votre collection de dashboards.

### Pourquoi l'organisation est cruciale

**Les symptômes d'une mauvaise organisation :**

- Vous ne retrouvez plus vos dashboards
- Vous créez des doublons sans le savoir
- Vous perdez du temps à chercher l'information
- Les nouveaux utilisateurs sont perdus
- La maintenance devient cauchemardesque

**Les bénéfices d'une bonne organisation :**

- **Accessibilité** : Trouvez instantanément ce que vous cherchez
- **Efficacité** : Moins de temps perdu en navigation
- **Collaboration** : L'équipe sait où chercher
- **Maintenance** : Mise à jour et évolution facilitées
- **Scalabilité** : Supporte la croissance du nombre de dashboards

### Analogie

Imaginez votre collection de dashboards comme une bibliothèque :

```
Mauvaise organisation = Livres entassés en vrac au sol
Bonne organisation = Bibliothèque avec sections, étagères, classement
```

## Principes d'organisation

### La règle des 3 clics

Un utilisateur devrait pouvoir atteindre n'importe quelle information en **maximum 3 clics** :

```
Clic 1 : Home → Catégorie principale
Clic 2 : Catégorie → Dashboard spécifique
Clic 3 : Dashboard → Information détaillée
```

### La pyramide d'information

Organisez vos dashboards du général au spécifique :

```
┌─────────────────────────────────┐
│   NIVEAU 1 : Vue d'ensemble     │ ← Dashboards exécutifs
│   (1-2 dashboards)              │
├─────────────────────────────────┤
│   NIVEAU 2 : Vues fonctionnelles│ ← Dashboards par domaine
│   (5-10 dashboards)             │
├─────────────────────────────────┤
│   NIVEAU 3 : Détails techniques │ ← Dashboards spécifiques
│   (20-50 dashboards)            │
└─────────────────────────────────┘
```

### Le principe MECE

**Mutually Exclusive, Collectively Exhaustive** (Mutuellement exclusif, collectivement exhaustif)

- **Mutuellement exclusif** : Chaque dashboard a un objectif unique, pas de duplication
- **Collectivement exhaustif** : Tous les aspects sont couverts, pas de trous

## Structure de dossiers (Folders)

### Créer des dossiers

1. **Accédez à la gestion des dashboards :**
   - Menu latéral → **Dashboards** → **Browse**

2. **Créez un nouveau dossier :**
   - Cliquez sur **New folder**
   - **Name** : Nom du dossier
   - **Create**

### Stratégies de structuration

#### Stratégie 1 : Par couche technique

Organisation basée sur les couches de l'infrastructure.

```
📁 Grafana Dashboards
├── 📁 01-Overview
│   ├── Dashboard : Global Overview
│   └── Dashboard : Executive Summary
├── 📁 02-Infrastructure
│   ├── Dashboard : Cluster Status
│   ├── Dashboard : Nodes Monitoring
│   └── Dashboard : Network Performance
├── 📁 03-Platform
│   ├── Dashboard : Kubernetes Control Plane
│   ├── Dashboard : etcd Metrics
│   └── Dashboard : API Server
├── 📁 04-Workloads
│   ├── Dashboard : Deployments
│   ├── Dashboard : StatefulSets
│   └── Dashboard : DaemonSets
├── 📁 05-Applications
│   ├── Dashboard : Frontend Apps
│   ├── Dashboard : Backend Services
│   └── Dashboard : Databases
└── 📁 06-Troubleshooting
    ├── Dashboard : Error Investigation
    ├── Dashboard : Performance Debug
    └── Dashboard : Resource Analysis
```

**Avantages :**
- Logique technique claire
- Facile pour les opérateurs
- Suit l'architecture du système

**Inconvénients :**
- Peut être complexe pour les non-techniques
- Nécessite de connaître l'architecture

#### Stratégie 2 : Par environnement

Organisation basée sur les environnements.

```
📁 Grafana Dashboards
├── 📁 Production
│   ├── 📁 Kubernetes
│   ├── 📁 Applications
│   └── 📁 Infrastructure
├── 📁 Staging
│   ├── 📁 Kubernetes
│   └── 📁 Applications
├── 📁 Development
│   ├── 📁 Kubernetes
│   └── 📁 Applications
└── 📁 Shared
    ├── Dashboard : Templates
    └── Dashboard : Global Metrics
```

**Avantages :**
- Séparation claire des environnements
- Évite les confusions
- Facile à gérer les permissions

**Inconvénients :**
- Duplication potentielle de dashboards
- Navigation répétitive

#### Stratégie 3 : Par équipe/département

Organisation basée sur les équipes.

```
📁 Grafana Dashboards
├── 📁 Platform-Team
│   ├── Dashboard : Kubernetes Cluster
│   ├── Dashboard : CI/CD Pipeline
│   └── Dashboard : Infrastructure
├── 📁 Development-Team
│   ├── Dashboard : Application Performance
│   ├── Dashboard : API Metrics
│   └── Dashboard : Error Tracking
├── 📁 Security-Team
│   ├── Dashboard : Security Events
│   ├── Dashboard : Compliance
│   └── Dashboard : Vulnerabilities
├── 📁 Business-Team
│   ├── Dashboard : Business KPIs
│   ├── Dashboard : Revenue Metrics
│   └── Dashboard : User Analytics
└── 📁 Shared
    └── Dashboard : Company Overview
```

**Avantages :**
- Chaque équipe trouve rapidement ses dashboards
- Permissions faciles à gérer
- Autonomie des équipes

**Inconvénients :**
- Information en silos
- Vues transverses difficiles

#### Stratégie 4 : Hybride (Recommandée)

Combinaison des approches précédentes.

```
📁 Grafana Dashboards
├── 📁 00-Executive
│   ├── Dashboard : Company Dashboard
│   └── Dashboard : SLA Overview
├── 📁 01-Platform
│   ├── 📁 Production
│   │   ├── Dashboard : K8s Prod Overview
│   │   ├── Dashboard : Nodes Prod
│   │   └── Dashboard : Workloads Prod
│   └── 📁 Staging
│       └── Dashboard : K8s Staging Overview
├── 📁 02-Applications
│   ├── 📁 Frontend
│   │   ├── Dashboard : Web App Metrics
│   │   └── Dashboard : Mobile App Metrics
│   └── 📁 Backend
│       ├── Dashboard : API Performance
│       └── Dashboard : Microservices
├── 📁 03-Infrastructure
│   ├── Dashboard : Network
│   ├── Dashboard : Storage
│   └── Dashboard : Security
└── 📁 99-Troubleshooting
    ├── Dashboard : Debug Tools
    └── Dashboard : Investigation
```

**Avantages :**
- Flexible et complet
- S'adapte à différents besoins
- Évolutif

### Conventions de nommage des dossiers

**Bonnes pratiques :**

```
✅ Bon : 01-Infrastructure, 02-Applications, 03-Security
✅ Bon : Kubernetes-Production, Kubernetes-Staging
✅ Bon : Team-Platform, Team-Development

❌ Mauvais : folder1, folder2, folder3
❌ Mauvais : Stuff, Things, Misc
❌ Mauvais : temp, test, old
```

**Règles :**
- Utilisez des préfixes numériques pour forcer l'ordre (01-, 02-, etc.)
- Noms descriptifs et explicites
- Cohérents dans toute l'organisation
- Évitez les dossiers temporaires ("temp", "test")

### Déplacer des dashboards entre dossiers

1. **Ouvrez le dashboard**
2. **Dashboard settings** (⚙️) → **General**
3. **Folder** : Sélectionnez le nouveau dossier
4. **Save dashboard**

**Ou par glisser-déposer :**
1. **Dashboards** → **Browse**
2. Glissez le dashboard vers le dossier souhaité

## Organisation interne des dashboards

### Structuration avec des Rows

Les rows permettent d'organiser les panneaux en sections logiques.

#### Créer une row

1. **Éditez le dashboard**
2. **Add** → **Row**
3. **Title** : Nom de la section
4. **Add**

#### Structure type d'un dashboard bien organisé

```
┌─────────────────────────────────────────────┐
│ Dashboard : Kubernetes Production Cluster   │
├─────────────────────────────────────────────┤
│ 📄 ROW : Documentation (repliée)            │
│   └─ Panneau Text : Guide d'utilisation     │
├─────────────────────────────────────────────┤
│ 📊 ROW : Vue d'ensemble (dépliée)           │
│   ├─ [Stat] Pods   ├─ [Stat] Nodes          │
│   ├─ [Gauge] CPU   └─ [Gauge] Memory        │
├─────────────────────────────────────────────┤
│ 📈 ROW : Tendances CPU/Memory (dépliée)     │
│   ├─ [Time series] CPU Usage                │
│   └─ [Time series] Memory Usage             │
├─────────────────────────────────────────────┤
│ 🔍 ROW : Analyse détaillée (repliée)        │
│   ├─ [Table] Pods Details                   │
│   ├─ [Bar chart] Top Consumers              │
│   └─ [Heatmap] Latency Distribution         │
├─────────────────────────────────────────────┤
│ ⚠️ ROW : Alertes et erreurs (repliée)       │
│   ├─ [Table] Failed Pods                    │
│   └─ [Time series] Restart Rate             │
└─────────────────────────────────────────────┘
```

#### Ordre recommandé des rows

1. **Documentation** (optionnelle, repliée par défaut)
2. **Vue d'ensemble** (KPIs principaux, toujours visible)
3. **Métriques principales** (graphiques temporels importants)
4. **Analyses détaillées** (repliées par défaut)
5. **Troubleshooting** (repliées par défaut)

### Disposition des panneaux

#### Grille de base

Grafana utilise une grille de **24 colonnes** :

```
Full width        : [========================] (24)
Half width        : [===========] [===========] (12+12)
Third width       : [=======][=======][=======] (8+8+8)
Quarter width     : [=====][=====][=====][=====] (6+6+6+6)
```

#### Layouts recommandés

**Layout "KPI" (pour Stats et Gauges) :**
```
┌──────┬──────┬──────┬──────┐
│      │      │      │      │
│  6   │  6   │  6   │  6   │ ← 4 panneaux de 6 colonnes
│      │      │      │      │
└──────┴──────┴──────┴──────┘
```

**Layout "Dashboard" (mixte) :**
```
┌────────────┬────────────┐
│            │            │
│     12     │     12     │ ← 2 Stats larges
│            │            │
├────────────┴────────────┤
│                         │
│          24             │ ← 1 Time series pleine largeur
│                         │
├───────────┬─────────────┤
│           │             │
│    12     │     12      │ ← 2 graphiques moyens
│           │             │
└───────────┴─────────────┘
```

**Layout "Détails" (pour Tables) :**
```
┌─────────────────────────┐
│                         │
│          24             │ ← Table pleine largeur
│                         │
└─────────────────────────┘
```

### Alignement et espacement

**Bonnes pratiques :**

1. **Alignez les panneaux** :
   - Utilisez la grille pour un alignement parfait
   - Les panneaux de même type doivent avoir la même taille

2. **Espacement cohérent** :
   - Pas d'espaces vides inutiles
   - Marges régulières entre les panneaux

3. **Hiérarchie visuelle** :
   - Panneaux importants plus grands
   - Panneaux secondaires plus petits
   - Du haut vers le bas = du plus important au moins important

## Navigation entre dashboards

### Dashboard home (page d'accueil)

#### Définir un dashboard comme home

1. **Ouvrez le dashboard souhaité**
2. **Cliquez sur l'étoile** ⭐ pour le marquer en favori
3. **User preferences** → **Home Dashboard**
4. Sélectionnez votre dashboard
5. **Save**

Désormais, ce dashboard s'affiche automatiquement à la connexion.

#### Créer un dashboard "Hub"

Un dashboard hub sert de point d'entrée central vers tous les autres dashboards.

**Structure d'un dashboard Hub :**

```markdown
# 🏠 Kubernetes Monitoring Hub

## 🎯 Dashboards principaux

- [Vue d'ensemble cluster](link)
- [Monitoring des nodes](link)
- [Workloads status](link)

## 📊 Par environnement

- [Production](link)
- [Staging](link)
- [Development](link)

## 🔧 Troubleshooting

- [Investigation erreurs](link)
- [Performance debugging](link)

## 📚 Documentation

- [Guide d'utilisation](external-link)
- [Runbook](external-link)
```

### Panel links

Créez des liens depuis les panneaux vers d'autres dashboards.

#### Configuration

1. **Éditez le panneau**
2. **Panel options** → **Panel links**
3. **Add link**
4. **Configuration :**
   - **Title** : "Voir les détails"
   - **Type** : Dashboard
   - **Dashboard** : Sélectionnez le dashboard cible
   - **Include current time range** : ✓
   - **Include current template variable values** : ✓
   - **Open in new tab** : Selon préférence

#### Exemple de drill-down

**Dashboard 1 : Vue d'ensemble**
```
Panneau : CPU Usage par Namespace
Link : → Dashboard "Namespace Details" avec var-namespace=${namespace}
```

**Dashboard 2 : Namespace Details**
```
Panneau : Pods dans le namespace
Link : → Dashboard "Pod Details" avec var-pod=${pod}
```

**Dashboard 3 : Pod Details**
```
Panneau : Détails complets du pod
Link : → Logs externes (Kibana)
```

### Data links

Les data links permettent de créer des liens contextuels basés sur les données.

#### Configuration

1. **Panel options** → **Data links**
2. **Add link**
3. **Configuration :**
   - **Title** : `Détails du pod ${__field.labels.pod}`
   - **URL** : `/d/pod-details?var-pod=${__field.labels.pod}&var-namespace=${__field.labels.namespace}`

#### Variables disponibles

```
${__field.labels.XXX}    : Labels de la série de données
${__value.raw}           : Valeur brute
${__value.numeric}       : Valeur numérique
${__value.text}          : Valeur texte
${__from}                : Timestamp début
${__to}                  : Timestamp fin
${namespace}             : Variable dashboard
```

### Breadcrumbs (fil d'Ariane)

Ajoutez des breadcrumbs en haut de vos dashboards pour faciliter la navigation.

**Panneau Text en haut :**

```markdown
🏠 [Home](/) > 📁 [Kubernetes](/dashboards/f/kubernetes) > 📊 Cluster Overview
```

Ou avec variables :

```markdown
🏠 [Home](/) > 🌍 ${cluster} > 📦 ${namespace} > 🚀 ${pod}
```

## Tags et recherche

### Utiliser les tags

Les tags permettent de catégoriser et retrouver facilement les dashboards.

#### Ajouter des tags

1. **Dashboard settings** → **General**
2. **Tags** : Ajoutez vos tags
3. **Save**

#### Stratégie de tags

**Tags hiérarchiques :**

```
Niveau 1 (Type) : kubernetes, application, infrastructure, security
Niveau 2 (Env)  : production, staging, development
Niveau 3 (Team) : platform, development, security, business
Niveau 4 (Autre): critical, detailed, executive, troubleshooting
```

**Exemples de combinaisons :**

```
Dashboard "K8s Production Overview"
Tags: kubernetes, production, platform, critical

Dashboard "API Performance Staging"
Tags: application, staging, development, detailed

Dashboard "Security Audit"
Tags: security, production, security, critical
```

#### Rechercher par tags

1. **Dashboards** → **Browse**
2. **Filter by tag** : Sélectionnez un ou plusieurs tags
3. Les dashboards correspondants s'affichent

**Recherche combinée :**
```
Tags: kubernetes + production
→ Tous les dashboards K8s de production
```

### Fonction de recherche

#### Search box

1. **Appuyez sur `/`** (raccourci clavier)
2. Ou cliquez sur **Search** dans le menu
3. Tapez votre recherche

**Types de recherche :**

```
kubernetes                    → Recherche dans les noms
tag:production               → Recherche par tag
folder:Infrastructure        → Recherche par dossier
starred:true                 → Dashboards favoris uniquement
```

#### Favoris (Starred)

Marquez vos dashboards fréquemment utilisés :

1. **Ouvrez le dashboard**
2. **Cliquez sur l'étoile** ⭐ en haut
3. Le dashboard apparaît dans **Starred** du menu principal

**Raccourci :** Les dashboards favoris sont accessibles rapidement depuis le menu latéral.

## Playlists

Les playlists permettent de faire défiler automatiquement une série de dashboards.

### Créer une playlist

1. **Menu latéral** → **Playlists**
2. **New playlist**
3. **Configuration :**
   - **Name** : "Production Monitoring Loop"
   - **Interval** : 30s (temps d'affichage par dashboard)

4. **Add dashboards :**
   - Recherchez et ajoutez les dashboards souhaités
   - Ordonnez-les par glisser-déposer

5. **Save**

### Cas d'usage des playlists

**Affichage mural (wall display) :**
```
Playlist "NOC Screen"
1. Global Overview (30s)
2. Production Cluster (30s)
3. Critical Alerts (20s)
4. Top Consumers (20s)
Loop: Infinite
```

**Présentation executive :**
```
Playlist "Weekly Review"
1. Executive Summary (60s)
2. SLA Dashboard (45s)
3. Business Metrics (45s)
4. Incidents Report (30s)
Loop: Once
```

**Monitoring room :**
```
Playlist "Operations Center"
1. Cluster Status (20s)
2. Application Health (20s)
3. Infrastructure (20s)
4. Security Events (20s)
Loop: Infinite
```

### Lancer une playlist

1. **Playlists** → Sélectionnez votre playlist
2. **Start playlist**
3. **Mode :**
   - **Normal** : Navigation standard
   - **TV mode** : Plein écran, cache les menus
   - **Kiosk mode** : Plein écran, cache tout

**Raccourcis en mode TV/Kiosk :**
```
Esc : Quitter
d   : Dashboard suivant
s   : Mettre en pause
```

## Permissions et partage

### Niveaux de permissions

Grafana propose 3 niveaux de permissions :

1. **Viewer** : Consultation uniquement
2. **Editor** : Consultation + Modification
3. **Admin** : Contrôle total

### Permissions par dossier

Les permissions se définissent au niveau du dossier et s'appliquent à tous les dashboards qu'il contient.

#### Configurer les permissions

1. **Dashboards** → **Browse**
2. **Folder** → Cliquez sur l'icône de dossier
3. **Permissions**
4. **Add Permission** :
   - **User/Team/Role** : Sélectionnez
   - **Permission** : Viewer/Editor/Admin
5. **Save**

#### Stratégie de permissions

**Exemple pour une organisation :**

```
📁 Executive
   Permissions:
   - Executives: Viewer
   - Platform Team: Admin

📁 Platform-Team
   Permissions:
   - Platform Team: Editor
   - Development Team: Viewer
   - Everyone: No access

📁 Public
   Permissions:
   - Everyone: Viewer
   - Platform Team: Editor
```

### Partage de dashboards

#### Snapshot

Pour partager temporairement un dashboard (sans accès Grafana requis) :

1. **Share dashboard** (icône partage)
2. **Snapshot**
3. **Configuration :**
   - **Snapshot name** : Nom descriptif
   - **Expire** : 1 hour / 1 day / 1 week / Never
   - **Timeout** : Temps de chargement des données
4. **Publish to snapshots.raintank.io** (public)
5. **Copiez et partagez l'URL**

⚠️ **Attention :** Les snapshots sont publics, ne partagez pas de données sensibles.

#### Export JSON

Pour partager le dashboard lui-même (pas les données) :

1. **Dashboard settings** → **JSON Model**
2. **Copy to clipboard** ou **Save to file**
3. Partagez le fichier JSON

Le destinataire peut l'importer dans son Grafana.

#### URL avec variables

Partagez des URLs avec des variables pré-remplies :

```
https://grafana.example.com/d/dashboard-id?var-namespace=production&var-pod=nginx&from=now-1h&to=now
```

**Paramètres d'URL :**
```
var-XXX=value     : Définit une variable
from=timestamp    : Début intervalle temps
to=timestamp      : Fin intervalle temps
refresh=5s        : Auto-refresh
orgId=1           : Organisation (multi-tenant)
```

## Conventions de nommage des dashboards

### Format recommandé

```
[Catégorie] Nom descriptif - Détail (Environnement)
```

**Exemples :**

```
✅ [K8s] Cluster Overview - Production
✅ [K8s] Namespace Details - ${namespace}
✅ [App] API Performance - Staging
✅ [Infra] Network Monitoring
✅ [Security] Audit Dashboard - All Environments

❌ Dashboard 1
❌ New Dashboard
❌ test
❌ Copy of Copy of Production
```

### Préfixes par type

Utilisez des préfixes cohérents :

```
[K8s]      : Dashboards Kubernetes
[App]      : Dashboards applicatifs
[Infra]    : Dashboards infrastructure
[Security] : Dashboards sécurité
[Business] : Dashboards métier
[Debug]    : Dashboards troubleshooting
```

### Suffixes informatifs

```
- Overview         : Vue d'ensemble
- Details          : Vue détaillée
- Troubleshooting  : Diagnostic
- (Production)     : Environnement spécifique
- (${namespace})   : Utilise des variables
```

## Maintenance et nettoyage

### Auditer vos dashboards

#### Identifier les dashboards inutilisés

1. **Dashboards** → **Browse**
2. **Sort by** : Last viewed
3. Les dashboards jamais ou rarement consultés apparaissent en bas

**Action :** Archivez ou supprimez les dashboards non utilisés depuis 3+ mois.

#### Trouver les doublons

Recherchez des dashboards avec des noms similaires :

```
Production Cluster
Production Cluster 2
Production Cluster Copy
Production Cluster NEW
```

**Action :** Fusionnez ou supprimez les doublons.

### Versionning et sauvegarde

#### Export régulier

Automatisez l'export de vos dashboards importants :

```bash
# Script de backup (exemple)
for dashboard in dashboard1 dashboard2 dashboard3; do
  curl -H "Authorization: Bearer ${GRAFANA_API_KEY}" \
    "http://grafana:3000/api/dashboards/uid/${dashboard}" \
    > "backup/${dashboard}.json"
done
```

#### Git pour versionning

Stockez vos dashboards dans Git :

```
dashboards/
├── kubernetes/
│   ├── cluster-overview.json
│   ├── node-details.json
│   └── pod-details.json
├── applications/
│   ├── api-performance.json
│   └── frontend-metrics.json
└── infrastructure/
    ├── network.json
    └── storage.json
```

**Avantages :**
- Historique complet des changements
- Review des modifications (Pull Requests)
- Rollback facile en cas d'erreur
- Collaboration efficace

### Politique de rétention

Définissez des règles claires :

```
Dashboard types:
- Production : Conserver indéfiniment
- Staging : Conserver 6 mois
- Dev/Test : Conserver 1 mois
- Experimental : Conserver 2 semaines

Après expiration:
- Archive (export JSON) avant suppression
- Notification avant suppression
```

## Bonnes pratiques d'organisation

### Principe KISS (Keep It Simple, Stupid)

**Simplicité > Complexité**

```
✅ Structure simple:
   📁 Production
   📁 Staging
   📁 Development

❌ Structure complexe:
   📁 Prod-Cluster-1-Region-EU-West
      📁 Namespace-Production-App-1
         📁 Version-1.2.3
            📁 Microservice-API-Gateway
```

### Cohérence

**Appliquez les mêmes conventions partout :**

- Même structure de dossiers
- Mêmes conventions de nommage
- Mêmes tags
- Même organisation interne des dashboards

### Documentation

**Chaque dashboard devrait avoir :**

1. **Description** : Objectif du dashboard
2. **Panneau Text en haut** : Guide d'utilisation
3. **Tags appropriés** : Pour la découverte
4. **Liens** : Vers documentation externe

**Template de documentation :**

```markdown
# 📊 Dashboard: Kubernetes Cluster Overview

## 🎯 Objectif
Vue d'ensemble de la santé du cluster Kubernetes de production.

## 👥 Audience
Équipe Platform, Opérations, Management

## 📋 Métriques affichées
- Nombre de pods/nodes actifs
- Utilisation CPU et mémoire
- Performance réseau
- Alertes actives

## 🔗 Dashboards liés
- [Détails des nodes](link)
- [Workloads](link)
- [Troubleshooting](link)

## 📞 Contact
En cas de question : platform-team@example.com
```

### Feedback et amélioration continue

**Collectez les retours :**

1. Ajoutez un lien feedback dans vos dashboards
2. Organisez des revues régulières
3. Analysez les patterns d'utilisation
4. Itérez sur l'organisation

**Questions à se poser régulièrement :**

- Les utilisateurs trouvent-ils facilement ce qu'ils cherchent ?
- Y a-t-il des dashboards rarement utilisés ?
- Les noms sont-ils clairs et cohérents ?
- La structure de dossiers est-elle toujours pertinente ?

## Exemples d'organisation complète

### Exemple 1 : Startup tech

**Contexte :** Petite équipe, un seul cluster, focus sur simplicité.

```
📁 Grafana
├── 📁 Overview
│   └── Dashboard : Company Dashboard
├── 📁 Kubernetes
│   ├── Dashboard : Cluster Status
│   ├── Dashboard : Applications
│   └── Dashboard : Troubleshooting
├── 📁 Applications
│   ├── Dashboard : API Metrics
│   ├── Dashboard : Frontend Metrics
│   └── Dashboard : Background Jobs
└── 📁 Business
    ├── Dashboard : KPIs
    └── Dashboard : User Metrics
```

### Exemple 2 : Entreprise mid-size

**Contexte :** Plusieurs équipes, environnements multiples.

```
📁 Grafana
├── 📁 00-Executive
│   ├── Dashboard : Executive Summary
│   └── Dashboard : SLA Dashboard
├── 📁 01-Production
│   ├── 📁 Kubernetes
│   │   ├── Dashboard : Cluster Overview
│   │   ├── Dashboard : Nodes
│   │   ├── Dashboard : Workloads
│   │   └── Dashboard : Network
│   ├── 📁 Applications
│   │   ├── Dashboard : Web App
│   │   ├── Dashboard : API Gateway
│   │   └── Dashboard : Microservices
│   └── 📁 Infrastructure
│       ├── Dashboard : Databases
│       └── Dashboard : Cache Systems
├── 📁 02-Staging
│   └── [Structure similaire à Production]
├── 📁 03-Development
│   └── [Structure simplifiée]
├── 📁 Team-Platform
│   ├── Dashboard : CI/CD Pipeline
│   └── Dashboard : Infrastructure Cost
├── 📁 Team-Security
│   ├── Dashboard : Security Events
│   └── Dashboard : Compliance
└── 📁 Troubleshooting
    ├── Dashboard : Error Investigation
    └── Dashboard : Performance Debug
```

### Exemple 3 : Grande entreprise

**Contexte :** Multi-cluster, multi-région, équipes nombreuses.

```
📁 Grafana
├── 📁 00-Global
│   ├── Dashboard : Global Overview
│   ├── Dashboard : Multi-Cluster Status
│   └── Dashboard : SLA Tracking
├── 📁 01-EU-West
│   ├── 📁 Production
│   │   ├── 📁 Cluster-A
│   │   └── 📁 Cluster-B
│   └── 📁 Staging
├── 📁 02-US-East
│   ├── 📁 Production
│   └── 📁 Staging
├── 📁 Teams
│   ├── 📁 Platform
│   ├── 📁 Development
│   ├── 📁 Security
│   └── 📁 Business
├── 📁 Applications
│   ├── 📁 E-commerce
│   ├── 📁 Analytics
│   └── 📁 Internal-Tools
└── 📁 Shared
    ├── Dashboard : Templates
    └── Dashboard : Documentation
```

## Migration et réorganisation

### Planifier une réorganisation

Si votre organisation actuelle est chaotique, voici comment migrer :

#### Phase 1 : Audit (Semaine 1)

1. **Listez tous vos dashboards**
2. **Identifiez l'utilisation** (dernière consultation)
3. **Classifiez** par type, équipe, importance
4. **Identifiez les doublons**

#### Phase 2 : Design (Semaine 2)

1. **Définissez la nouvelle structure** de dossiers
2. **Établissez les conventions** de nommage
3. **Définissez les tags**
4. **Créez un plan de migration**

#### Phase 3 : Migration (Semaine 3-4)

1. **Créez les nouveaux dossiers**
2. **Migrez par vagues** (par priorité)
3. **Communiquez** à l'équipe
4. **Mettez à jour** les favoris et liens

#### Phase 4 : Nettoyage (Semaine 5)

1. **Archivez** les anciens dashboards
2. **Supprimez** les doublons
3. **Documentez** la nouvelle organisation
4. **Formez** l'équipe

### Communication pendant la migration

**Email d'annonce :**

```
Objet: Migration Grafana - Nouvelle organisation des dashboards

Bonjour à tous,

Nous réorganisons nos dashboards Grafana pour améliorer la navigation et la maintenance.

Changements principaux:
- Nouvelle structure de dossiers par environnement et équipe
- Conventions de nommage standardisées
- Tags pour faciliter la recherche

Actions de votre côté:
- Mettez à jour vos favoris
- Consultez le guide de migration: [lien]
- Signalez les problèmes à: platform-team@example.com

Calendrier:
- Semaine du 1er: Création des dossiers
- Semaine du 8: Migration des dashboards
- Semaine du 15: Nettoyage final

Merci de votre compréhension!
```

## Outils et automation

### API Grafana

Utilisez l'API pour automatiser la gestion :

```bash
# Lister tous les dashboards
curl -H "Authorization: Bearer ${API_KEY}" \
  http://grafana:3000/api/search

# Créer un dossier
curl -X POST -H "Authorization: Bearer ${API_KEY}" \
  -H "Content-Type: application/json" \
  -d '{"title":"New Folder"}' \
  http://grafana:3000/api/folders

# Déplacer un dashboard
curl -X POST -H "Authorization: Bearer ${API_KEY}" \
  -H "Content-Type: application/json" \
  -d '{"dashboard":{...},"folderId":2}' \
  http://grafana:3000/api/dashboards/db
```

### Grafana CLI

Pour les opérations en ligne de commande :

```bash
# Backup de tous les dashboards
grafana-cli admin data-migration --export

# Import de dashboards
grafana-cli admin data-migration --import
```

### Terraform pour Infrastructure as Code

Gérez vos dashboards comme du code :

```hcl
resource "grafana_folder" "kubernetes" {
  title = "Kubernetes"
}

resource "grafana_dashboard" "cluster_overview" {
  folder      = grafana_folder.kubernetes.id
  config_json = file("dashboards/cluster-overview.json")
}
```

## Prochaines étapes

Maintenant que vous maîtrisez l'organisation des dashboards :

- **Chapitre 14 :** Alerting et notifications
- **Chapitre 15 :** Observabilité avancée
- **Chapitre 16 :** Sécurité Kubernetes

## Récapitulatif

Dans ce chapitre, vous avez appris à :

✅ Comprendre l'importance de l'organisation des dashboards
✅ Créer et structurer des dossiers efficacement
✅ Choisir une stratégie d'organisation adaptée
✅ Organiser l'intérieur des dashboards avec des rows
✅ Créer une navigation fluide entre dashboards
✅ Utiliser les tags et la recherche efficacement
✅ Créer et utiliser des playlists
✅ Gérer les permissions et le partage
✅ Appliquer des conventions de nommage cohérentes
✅ Maintenir et nettoyer votre bibliothèque de dashboards
✅ Planifier et exécuter une réorganisation

Une organisation claire et cohérente de vos dashboards transforme Grafana d'une collection chaotique de graphiques en un système de monitoring professionnel, efficace et agréable à utiliser !

---

**Conseil final :** L'organisation parfaite n'existe pas - elle évolue avec votre équipe et vos besoins. Commencez avec une structure simple et faites-la évoluer progressivement. L'important est la cohérence : une fois que vous avez défini des conventions, tenez-vous y ! Et n'oubliez pas : un dashboard bien organisé est un dashboard qui sera effectivement utilisé.

⏭️ [Alerting et Notifications](/14-alerting-et-notifications/README.md)
