🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 13.4 Création de dashboards personnalisés

## Introduction

Maintenant que vous avez exploré les dashboards pré-configurés, il est temps de créer vos propres dashboards adaptés à vos besoins spécifiques. La création de dashboards personnalisés vous permet de visualiser exactement les métriques qui comptent pour vous, dans le format qui vous convient.

### Pourquoi créer des dashboards personnalisés ?

**Avantages :**

- **Personnalisation totale** : Affichez uniquement ce qui vous intéresse
- **Contexte métier** : Intégrez des métriques spécifiques à vos applications
- **Simplicité** : Évitez la surcharge d'informations des dashboards génériques
- **Apprentissage** : Meilleure compréhension de vos systèmes
- **Flexibilité** : Adaptez au fur et à mesure de vos besoins

### Ce que vous apprendrez

Dans ce chapitre, vous découvrirez comment :

- Créer un dashboard vide
- Ajouter différents types de panneaux
- Écrire des requêtes PromQL simples
- Configurer les visualisations
- Organiser efficacement votre dashboard
- Personnaliser l'apparence
- Sauvegarder et gérer vos dashboards

## Concepts de base

### Structure d'un dashboard Grafana

Un dashboard Grafana est composé de plusieurs éléments :

```
┌─────────────────────────────────────────────────────┐
│  DASHBOARD : Mon Dashboard Kubernetes               │
│  [Variables] [Intervalle de temps] [Rafraîchir]     │
├─────────────────────────────────────────────────────┤
│  ROW 1 : Métriques Générales                        │
│  ┌──────────────┐  ┌──────────────┐  ┌────────────┐ │
│  │   Panneau 1  │  │   Panneau 2  │  │  Panneau 3 │ │
│  │   (Graph)    │  │   (Gauge)    │  │  (Stat)    │ │
│  └──────────────┘  └──────────────┘  └────────────┘ │
│  ROW 2 : Métriques Détaillées                       │
│  ┌────────────────────────────────────────────────┐ │
│  │   Panneau 4 (Table)                            │ │
│  └────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────┘
```

**Éléments principaux :**

- **Variables** : Filtres dynamiques (namespace, pod, etc.)
- **Rows** : Lignes pour organiser les panneaux
- **Panneaux (Panels)** : Visualisations individuelles
- **Requêtes** : Code PromQL pour récupérer les données

### Types de panneaux disponibles

Grafana offre de nombreux types de visualisations :

| Type | Usage | Exemple |
|------|-------|---------|
| **Time series** | Évolution dans le temps | CPU usage au fil du temps |
| **Stat** | Valeur unique | Nombre de pods actifs |
| **Gauge** | Jauge avec seuils | % d'utilisation mémoire |
| **Bar chart** | Comparaisons | Consommation par namespace |
| **Table** | Données tabulaires | Liste des pods avec métriques |
| **Pie chart** | Répartitions | Distribution des ressources |
| **Text** | Documentation | Instructions, notes |
| **Heatmap** | Densité de données | Latence distribution |

## Création de votre premier dashboard

### Étape 1 : Créer un dashboard vide

1. **Connectez-vous à Grafana** (`http://localhost:3000`)

2. **Créez un nouveau dashboard :**
   - Cliquez sur l'icône **+** dans le menu latéral
   - Sélectionnez **Dashboard**
   - Ou allez dans **Dashboards → New Dashboard**

3. **Interface de création :**
   Vous voyez maintenant un dashboard vide avec un bouton **Add visualization**.

### Étape 2 : Ajouter votre premier panneau

1. **Cliquez sur Add visualization**

2. **Sélectionnez la source de données :**
   - Choisissez **Prometheus**
   - Cette étape définit d'où viendront les données

3. **Vous êtes maintenant dans l'éditeur de panneau** avec :
   - Zone de requête en bas
   - Aperçu de la visualisation en haut
   - Options de configuration à droite

### Étape 3 : Écrire votre première requête

Commençons par une requête simple : afficher le nombre de pods en cours d'exécution.

1. **Dans l'onglet Query**, section **Metrics browser** :

   Tapez ou sélectionnez :
   ```promql
   kube_pod_status_phase{phase="Running"}
   ```

2. **Comprenons cette requête :**
   - `kube_pod_status_phase` : Nom de la métrique (statut des pods)
   - `{phase="Running"}` : Filtre (uniquement les pods en cours d'exécution)

3. **Cliquez sur Run query** ou appuyez sur **Shift + Entrée**

4. **Résultat :**
   Vous devriez voir une ligne de temps montrant le nombre de pods actifs.

### Étape 4 : Choisir le type de visualisation

Par défaut, Grafana utilise un graphique temporel (Time series). Pour notre exemple, un **Stat** est plus approprié.

1. **À droite, dans le panneau de configuration**, cherchez **Panel options**

2. **Type de visualisation :**
   - En haut à droite, cliquez sur l'icône de visualisation
   - Sélectionnez **Stat**

3. **Le panneau se transforme** en un grand nombre affichant la valeur actuelle.

### Étape 5 : Personnaliser le panneau

#### Titre et description

1. **Panel options → Title** :
   ```
   Pods en cours d'exécution
   ```

2. **Description (optionnel)** :
   ```
   Nombre total de pods avec le statut Running dans le cluster
   ```

3. **Transparent background** :
   - Décochez si vous voulez un fond pour le panneau

#### Unités et formatage

1. **Standard options → Unit** :
   - Cherchez "short" ou "none"
   - Cela définit comment les nombres sont affichés

2. **Decimals** :
   - Mettez `0` pour des nombres entiers

3. **Color scheme** :
   - **Single color** : Une seule couleur
   - **From thresholds** : Couleur basée sur des seuils

### Étape 6 : Définir des seuils (Thresholds)

Les seuils permettent de changer la couleur selon la valeur.

1. **Thresholds → Add threshold** :

   Configuration exemple :
   - **Base** (défaut) : Vert = Tout va bien
   - **50** : Orange = Attention
   - **100** : Rouge = Trop de pods

2. **Mode** :
   - **Absolute** : Valeurs exactes
   - **Percentage** : Pourcentages

3. **Visualisation :**
   Le panneau change de couleur automatiquement selon la valeur !

### Étape 7 : Sauvegarder le panneau

1. **En haut à droite**, cliquez sur **Apply**
2. Vous revenez au dashboard avec votre premier panneau !

### Étape 8 : Sauvegarder le dashboard

1. **En haut à droite**, cliquez sur l'icône **Save** (💾)

2. **Remplissez les informations :**
   - **Dashboard name** : `Mon Premier Dashboard Kubernetes`
   - **Folder** : Sélectionnez ou créez un dossier
   - **Description** : (Optionnel) Description du dashboard

3. **Cliquez sur Save**

Félicitations ! Vous avez créé votre premier dashboard personnalisé !

## Panneaux essentiels pour Kubernetes

Voici une série de panneaux utiles que vous pouvez ajouter à votre dashboard.

### Panneau 1 : Utilisation CPU du cluster

**Type :** Time series (graphique)

**Requête PromQL :**
```promql
sum(rate(container_cpu_usage_seconds_total{container!=""}[5m]))
```

**Explication :**
- `container_cpu_usage_seconds_total` : Utilisation CPU des conteneurs
- `rate(...[5m])` : Taux de changement sur 5 minutes
- `sum(...)` : Somme pour tout le cluster

**Configuration :**
- **Title** : Utilisation CPU du Cluster
- **Unit** : cores
- **Legend** : Mode = List, Placement = Bottom

### Panneau 2 : Utilisation mémoire du cluster

**Type :** Time series

**Requête PromQL :**
```promql
sum(container_memory_usage_bytes{container!=""}) / 1024 / 1024 / 1024
```

**Explication :**
- `container_memory_usage_bytes` : Mémoire utilisée
- Division par 1024³ : Conversion en GiB (Gibibytes)
- `sum(...)` : Total pour le cluster

**Configuration :**
- **Title** : Utilisation Mémoire du Cluster
- **Unit** : gibibytes (GiB)
- **Min** : 0

### Panneau 3 : Pods par namespace

**Type :** Bar chart

**Requête PromQL :**
```promql
sum(kube_pod_status_phase{phase="Running"}) by (namespace)
```

**Explication :**
- `by (namespace)` : Groupe les résultats par namespace
- Compte les pods par namespace

**Configuration :**
- **Title** : Pods par Namespace
- **Orientation** : Horizontal
- **Show values** : Always

### Panneau 4 : État des Deployments

**Type :** Table

**Requêtes multiples :**

Requête A - Replicas disponibles :
```promql
kube_deployment_status_replicas_available
```

Requête B - Replicas désirés :
```promql
kube_deployment_spec_replicas
```

**Configuration :**
- **Title** : État des Deployments
- **Transform** : Merge (fusionner les résultats)
- **Table columns** : deployment, available, desired

### Panneau 5 : Top 5 Pods par CPU

**Type :** Bar chart

**Requête PromQL :**
```promql
topk(5, sum(rate(container_cpu_usage_seconds_total{container!=""}[5m])) by (pod))
```

**Explication :**
- `topk(5, ...)` : Garde uniquement les 5 valeurs les plus élevées
- `by (pod)` : Groupe par pod

**Configuration :**
- **Title** : Top 5 Pods - Consommation CPU
- **Unit** : cores
- **Orientation** : Horizontal

### Panneau 6 : Espace disque disponible

**Type :** Gauge

**Requête PromQL :**
```promql
(1 - (node_filesystem_avail_bytes{mountpoint="/"} / node_filesystem_size_bytes{mountpoint="/"})) * 100
```

**Explication :**
- Calcule le pourcentage d'espace disque utilisé sur la racine (/)
- `1 - (disponible / total)` = utilisé
- `* 100` : Conversion en pourcentage

**Configuration :**
- **Title** : Utilisation Disque
- **Unit** : percent (0-100)
- **Thresholds** : 0=vert, 70=orange, 85=rouge
- **Max** : 100

### Panneau 7 : Trafic réseau

**Type :** Time series

**Requêtes multiples :**

Requête A - Trafic entrant :
```promql
sum(rate(container_network_receive_bytes_total[5m])) / 1024 / 1024
```

Requête B - Trafic sortant :
```promql
sum(rate(container_network_transmit_bytes_total[5m])) / 1024 / 1024
```

**Configuration :**
- **Title** : Trafic Réseau
- **Unit** : MBps (megabytes/sec)
- **Legend** : Requête A = "Réception", Requête B = "Transmission"

## Organisation avancée du dashboard

### Utilisation des Rows (Lignes)

Les rows permettent d'organiser logiquement vos panneaux et de les replier.

#### Créer une row

1. **Cliquez sur Add** en haut du dashboard
2. Sélectionnez **Row**
3. **Configurez la row :**
   - **Title** : "Vue d'ensemble" ou "Métriques détaillées"
4. **Cliquez sur Add**

#### Organiser les panneaux dans les rows

1. **Glissez-déposez les panneaux** dans les rows
2. **Cliquez sur l'icône de la row** pour la replier/déplier
3. Vous pouvez avoir plusieurs rows pour organiser thématiquement :

```
ROW : Vue d'ensemble
  ├─ Pods en cours d'exécution
  ├─ CPU Cluster
  └─ Mémoire Cluster

ROW : Analyse par Namespace
  ├─ Pods par Namespace
  └─ CPU par Namespace

ROW : Top Consommateurs
  ├─ Top 5 Pods CPU
  └─ Top 5 Pods Mémoire
```

### Disposition des panneaux (Layout)

#### Système de grille

Grafana utilise une grille de **24 colonnes** :

- 1 panneau pleine largeur = 24 unités
- 2 panneaux côte à côte = 12 unités chacun
- 3 panneaux = 8 unités chacun
- 4 panneaux = 6 unités chacun

#### Redimensionner les panneaux

1. **Passez la souris** sur le coin inférieur droit du panneau
2. **Glissez pour redimensionner**
3. Les panneaux s'ajustent automatiquement

#### Déplacer les panneaux

1. **Cliquez et maintenez** sur le titre du panneau
2. **Glissez-déposez** à l'emplacement souhaité
3. Les autres panneaux se réorganisent automatiquement

### Répétition de panneaux (Repeat panels)

Pour créer automatiquement un panneau par namespace, pod, etc.

1. **Éditez le panneau**
2. **Panel options → Repeat options**
3. **Repeat by variable** : Sélectionnez une variable (par exemple, `namespace`)
4. **Direction** : Horizontal ou Vertical

Grafana créera automatiquement un panneau pour chaque valeur de la variable !

## Requêtes PromQL courantes pour débutants

### Métriques de base

**Nombre de nœuds dans le cluster :**
```promql
count(kube_node_info)
```

**Nombre total de pods :**
```promql
count(kube_pod_info)
```

**Pods par statut :**
```promql
sum(kube_pod_status_phase) by (phase)
```

### Métriques de ressources

**CPU requests vs limits par namespace :**
```promql
sum(kube_pod_container_resource_requests{resource="cpu"}) by (namespace)
sum(kube_pod_container_resource_limits{resource="cpu"}) by (namespace)
```

**Mémoire requests vs limits :**
```promql
sum(kube_pod_container_resource_requests{resource="memory"}) by (namespace) / 1024 / 1024 / 1024
sum(kube_pod_container_resource_limits{resource="memory"}) by (namespace) / 1024 / 1024 / 1024
```

### Métriques de disponibilité

**Pods redémarrant fréquemment :**
```promql
sum(rate(kube_pod_container_status_restarts_total[1h])) by (pod) > 0
```

**Pods en erreur :**
```promql
kube_pod_status_phase{phase="Failed"}
```

**Pods pending :**
```promql
kube_pod_status_phase{phase="Pending"}
```

### Métriques réseau

**Bande passante par pod :**
```promql
sum(rate(container_network_receive_bytes_total[5m])) by (pod) / 1024 / 1024
```

**Erreurs réseau :**
```promql
rate(container_network_receive_errors_total[5m])
```

### Métriques de stockage

**Utilisation des PersistentVolumes :**
```promql
(kubelet_volume_stats_used_bytes / kubelet_volume_stats_capacity_bytes) * 100
```

**Inodes disponibles :**
```promql
kubelet_volume_stats_inodes_free
```

## Fonctionnalités avancées des panneaux

### Transformations de données

Les transformations permettent de manipuler les données avant affichage.

1. **Dans l'éditeur de panneau**, cliquez sur l'onglet **Transform**
2. **Add transformation**

**Transformations utiles :**

#### Merge (Fusionner)

Combine plusieurs séries de données :
- Utile pour comparer replicas disponibles vs désirés

#### Filter by name (Filtrer par nom)

Exclut certaines séries de données :
- Exemple : Exclure les pods système

#### Organize fields (Organiser les champs)

Renomme ou cache des colonnes dans une table :
- Rend les tables plus lisibles

#### Calculate field (Calculer un champ)

Crée de nouveaux champs calculés :
- Exemple : Pourcentage = (utilisé / total) * 100

### Overrides (Remplacements)

Permet de personnaliser individuellement des séries de données.

1. **Panel options → Overrides**
2. **Add override**
3. **Match by name** ou **Match by regex**

**Exemples d'usage :**

- Donner une couleur spécifique à une série
- Afficher certaines séries sur l'axe Y de droite
- Cacher certaines séries de la légende

### Liens et drill-downs

Créez des liens vers d'autres dashboards ou URLs.

1. **Panel options → Panel links**
2. **Add link**
3. **Configuration :**
   - **Type** : Dashboard ou External link
   - **URL** : Lien vers dashboard ou site externe
   - **Title** : Texte du lien

**Exemple :**
Lien depuis "Vue d'ensemble" vers "Détails du namespace"

### Annotations

Les annotations marquent des événements sur les graphiques.

**Types d'annotations :**

- Déploiements effectués
- Incidents survenus
- Maintenance planifiée

**Configuration :**

1. **Dashboard settings** → **Annotations**
2. **Add annotation query**
3. Configurez la requête pour récupérer les événements

## Variables pour des dashboards dynamiques

Les variables permettent de filtrer dynamiquement les données affichées.

### Créer une variable

1. **Dashboard settings** (⚙️) → **Variables**
2. **Add variable**

### Types de variables courantes

#### Variable Namespace

**Configuration :**

- **Name** : `namespace`
- **Type** : Query
- **Data source** : Prometheus
- **Query** :
  ```promql
  label_values(kube_namespace_labels, namespace)
  ```
- **Refresh** : On dashboard load
- **Multi-value** : Activé (permet de sélectionner plusieurs namespaces)
- **Include All option** : Activé

#### Variable Pod

**Configuration :**

- **Name** : `pod`
- **Type** : Query
- **Query** :
  ```promql
  label_values(kube_pod_info{namespace="$namespace"}, pod)
  ```

Notice : Utilise `$namespace` pour dépendre de la première variable

#### Variable Node

**Configuration :**

- **Name** : `node`
- **Type** : Query
- **Query** :
  ```promql
  label_values(kube_node_info, node)
  ```

### Utiliser les variables dans les requêtes

Une fois créées, utilisez les variables dans vos requêtes PromQL :

```promql
kube_pod_status_phase{namespace="$namespace", pod=~"$pod"}
```

**Syntaxe :**
- `$namespace` : Valeur simple
- `$pod` : Valeur simple
- `=~"$pod"` : Regex (pour multi-select)

### Variables personnalisées (Custom)

Pour des valeurs fixes :

1. **Type** : Custom
2. **Values separated by comma** :
   ```
   production,staging,development
   ```

Utile pour filtrer par environnement.

### Variables d'intervalle

Créez une variable pour changer l'intervalle de scrape :

- **Name** : `interval`
- **Type** : Interval
- **Values** : `1m,5m,15m,30m,1h`

Puis dans vos requêtes :
```promql
rate(container_cpu_usage_seconds_total[$interval])
```

## Personnalisation de l'apparence

### Thèmes et couleurs

#### Thème du dashboard

1. **Dashboard settings** → **General**
2. **Theme** :
   - Default (utilise les préférences utilisateur)
   - Dark
   - Light

#### Schémas de couleurs

Dans chaque panneau :

1. **Standard options → Color scheme**
2. Options :
   - **Single color** : Couleur unique
   - **From thresholds** : Basé sur les seuils
   - **Gradient** : Dégradé de couleurs
   - **Classic palette** : Palette Grafana classique

### Styles de graphiques

Pour les Time series :

1. **Graph styles**
   - **Lines** : Lignes (par défaut)
   - **Bars** : Barres
   - **Points** : Points

2. **Fill opacity** : Transparence sous la ligne (0-100%)

3. **Line width** : Épaisseur de la ligne

4. **Point size** : Taille des points

### Légendes

Configuration de la légende :

1. **Legend → Mode** :
   - **List** : Liste simple
   - **Table** : Tableau avec statistiques

2. **Placement** :
   - **Bottom** : En bas (recommandé)
   - **Right** : À droite
   - **Top** : En haut

3. **Values** (pour mode Table) :
   - Last : Dernière valeur
   - Min : Valeur minimale
   - Max : Valeur maximale
   - Mean : Moyenne
   - Total : Somme

### Axes

Pour les Time series et Bar charts :

1. **Axis → Y-axis**
   - **Scale** : Linear ou Logarithmic
   - **Min/Max** : Valeurs min/max de l'axe
   - **Decimals** : Nombre de décimales

2. **Two Y axes** :
   - Activez pour avoir deux échelles différentes
   - Utile pour comparer CPU (cores) et mémoire (GB)

## Gestion des dashboards

### Copier un dashboard

1. Ouvrez le dashboard
2. **Settings** → **Save as...**
3. Nouveau nom : "Copie de..."
4. Save

### Partager un dashboard

#### Export en JSON

1. **Settings** → **JSON Model**
2. **Copy to clipboard** ou **Save to file**
3. Partagez le fichier JSON

#### Snapshot

1. **Share dashboard** (icône de partage)
2. **Snapshot**
3. **Publish to snapshot.raintank.io**
4. Copiez et partagez l'URL

⚠️ **Attention** : Les snapshots sont publics, ne partagez pas de données sensibles.

### Versionning

Grafana garde un historique des versions :

1. **Settings** → **Versions**
2. Voyez toutes les modifications
3. **Compare versions** pour voir les différences
4. **Restore** pour revenir à une ancienne version

### Permissions

Pour contrôler qui peut voir/éditer le dashboard :

1. **Settings** → **Permissions**
2. **Add Permission**
3. **Role** : Viewer, Editor, ou Admin
4. **Save**

## Bonnes pratiques de création

### Principe de la pyramide

Organisez vos dashboards du général au spécifique :

```
NIVEAU 1 (Haut) : Métriques globales
   ↓
NIVEAU 2 : Métriques par composant
   ↓
NIVEAU 3 : Métriques détaillées
```

### Règle des 7±2

- Maximum 5-9 panneaux par écran visible
- Si plus, utilisez des rows repliables
- Trop de panneaux = surcharge cognitive

### Cohérence visuelle

- **Même palette de couleurs** pour tous les panneaux
- **Taille uniforme** pour des panneaux similaires
- **Alignement** : Utilisez la grille pour aligner proprement

### Nommage clair

- **Titres explicites** : "CPU Usage" plutôt que "CPU"
- **Descriptions** : Ajoutez une description à chaque panneau
- **Unités** : Toujours spécifier les unités (%, MB, cores)

### Performance

- **Limitez le nombre de séries** : Utilisez `topk()` ou des filtres
- **Intervalles raisonnables** : 5m est un bon compromis
- **Évitez les requêtes sans filtre** : Toujours filtrer par namespace, pod, etc.

### Documentation intégrée

Ajoutez des panneaux Text pour :

- **En-tête** : Expliquer le but du dashboard
- **Instructions** : Comment interpréter les métriques
- **Alertes** : Définir les seuils critiques
- **Contacts** : Qui contacter en cas de problème

## Exemples de dashboards complets

### Dashboard 1 : Vue d'ensemble Kubernetes

**Objectif :** Santé générale du cluster en un coup d'œil

**Structure :**

```
┌─────────────────────────────────────────────┐
│  Kubernetes - Vue d'ensemble                │
├─────────────────────────────────────────────┤
│ [Namespace▼] [Node▼] [Dernières 24h]        │
├─────────────────────────────────────────────┤
│  📊 Métriques Globales                      │
│  ┌─────┐  ┌─────┐  ┌─────┐  ┌─────┐         │
│  │ 12  │  │ 45  │  │ 98% │  │ 65% │         │
│  │Nodes│  │Pods │  │ CPU │  │ RAM │         │
│  └─────┘  └─────┘  └─────┘  └─────┘         │
│                                             │
│  📈 Tendances                               │
│  ┌─────────────────────────────────────┐    │
│  │  CPU Usage (24h)                    │    │
│  └─────────────────────────────────────┘    │
│  ┌─────────────────────────────────────┐    │
│  │  Memory Usage (24h)                 │    │
│  └─────────────────────────────────────┘    │
└─────────────────────────────────────────────┘
```

### Dashboard 2 : Monitoring applicatif

**Objectif :** Surveiller une application spécifique

**Panneaux :**

1. État de l'application (Stat)
2. Nombre de requêtes HTTP (Time series)
3. Latence moyenne (Gauge)
4. Taux d'erreur 5xx (Time series)
5. Replicas actifs vs désirés (Stat)
6. Consommation ressources (Time series)

### Dashboard 3 : Troubleshooting

**Objectif :** Diagnostiquer les problèmes rapidement

**Panneaux :**

1. Pods en erreur (Table)
2. Pods redémarrant (Table)
3. Pods en pending (Table)
4. Logs récents (Text/Logs panel)
5. Top consommateurs CPU (Bar chart)
6. Top consommateurs RAM (Bar chart)
7. Saturation du réseau (Time series)

## Debugging et optimisation

### Dashboard lent à charger

**Causes :**

1. Trop de panneaux (>20)
2. Requêtes non optimisées
3. Intervalle de temps trop long
4. Trop de séries temporelles

**Solutions :**

1. **Réduisez le nombre de panneaux** ou utilisez des rows repliées
2. **Optimisez les requêtes :**
   - Ajoutez des filtres
   - Utilisez `topk()` pour limiter
3. **Réduisez l'intervalle de temps** par défaut (6h au lieu de 24h)
4. **Augmentez le scrape interval** dans les requêtes

### Pas de données affichées

**Vérifications :**

1. La requête PromQL est-elle correcte ?
   - Testez dans Explore
2. Les métriques existent-elles ?
   - Vérifiez dans Prometheus (`/targets`)
3. L'intervalle de temps contient-il des données ?
4. Les filtres (variables) ne sont-ils pas trop restrictifs ?

### Valeurs incorrectes

**Problèmes communs :**

1. **Unités** : Vérifiez la conversion (bytes → GB)
2. **Rate/Increase** : Utilisez `rate()` pour les counters
3. **Agrégations** : `sum()`, `avg()`, `max()` - choisissez le bon
4. **Intervalle** : `[5m]` est généralement bon, ajustez si nécessaire

## Intégration avec d'autres outils

### Liens vers Prometheus

Ajoutez un lien pour investiguer dans Prometheus :

```
http://prometheus:9090/graph?g0.expr=YOUR_QUERY&g0.tab=0
```

### Liens vers Kibana/Logs

Si vous avez une stack de logs (ELK/EFK) :

```
http://kibana:5601/app/discover#/?_a=(query:(query_string:(query:'kubernetes.pod:$pod')))
```

### Liens vers documentation

Ajoutez des liens externes vers :

- Runbooks (procédures de résolution)
- Documentation de l'application
- Pages d'incident

## Templates de dashboards

### Template minimaliste

```json
{
  "title": "Mon Dashboard",
  "panels": [
    {
      "title": "Metric 1",
      "type": "timeseries",
      "targets": [
        {
          "expr": "votre_requete_promql"
        }
      ]
    }
  ]
}
```

### Template avec variables

```json
{
  "templating": {
    "list": [
      {
        "name": "namespace",
        "type": "query",
        "query": "label_values(kube_namespace_labels, namespace)"
      }
    ]
  }
}
```

## Prochaines étapes

Maintenant que vous maîtrisez la création de dashboards personnalisés :

- **Chapitre 13.5 :** Panneaux et visualisations avancées
- **Chapitre 13.6 :** Variables et templating en profondeur
- **Chapitre 13.7 :** Organisation avancée des dashboards
- **Chapitre 14 :** Alerting et notifications

## Récapitulatif

Dans ce chapitre, vous avez appris à :

✅ Créer un dashboard vide et y ajouter des panneaux
✅ Écrire des requêtes PromQL pour récupérer des métriques
✅ Choisir le type de visualisation adapté
✅ Configurer les panneaux (titres, unités, seuils)
✅ Organiser les dashboards avec des rows
✅ Utiliser des variables pour des dashboards dynamiques
✅ Personnaliser l'apparence des visualisations
✅ Appliquer les bonnes pratiques de création
✅ Optimiser les performances des dashboards
✅ Gérer et partager vos créations

Vous êtes maintenant capable de créer des dashboards professionnels et personnalisés pour surveiller efficacement votre infrastructure Kubernetes !

---

**Conseil final :** Commencez simple avec 3-4 panneaux essentiels, puis ajoutez progressivement de la complexité. Un dashboard sobre et clair est toujours plus efficace qu'un dashboard surchargé. N'ayez pas peur d'expérimenter - Grafana sauvegarde automatiquement les versions, vous pouvez toujours revenir en arrière !

⏭️ [Panneaux et visualisations](/13-visualisation-avec-grafana/05-panneaux-et-visualisations.md)
