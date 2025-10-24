🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 13.5 Panneaux et visualisations

## Introduction

Les panneaux (panels) sont les briques de base de vos dashboards Grafana. Chaque panneau est une visualisation qui affiche des données d'une manière spécifique. Choisir le bon type de visualisation est essentiel pour communiquer efficacement l'information.

### Pourquoi la visualisation est importante ?

**Un bon graphique vaut mille chiffres :**

- **Compréhension rapide** : Identifiez les tendances et anomalies en un coup d'œil
- **Communication efficace** : Partagez l'information de manière claire
- **Prise de décision** : Les visuels facilitent l'analyse et les décisions
- **Détection de problèmes** : Les alertes visuelles attirent immédiatement l'attention

### Les 3 questions à se poser

Avant de choisir un type de visualisation, demandez-vous :

1. **Quel type de données ?** (valeur unique, série temporelle, comparaison, distribution)
2. **Quel message ?** (évolution, état actuel, comparaison, proportion)
3. **Pour qui ?** (technique, managérial, opérationnel)

## Vue d'ensemble des types de panneaux

Grafana offre de nombreux types de visualisations. Voici les plus utilisés :

| Type | Icône | Usage principal | Exemple |
|------|-------|----------------|---------|
| **Time series** | 📈 | Évolution dans le temps | CPU au fil des heures |
| **Stat** | 🔢 | Valeur unique actuelle | Nombre de pods |
| **Gauge** | 🌡️ | Jauge avec seuils | % d'utilisation |
| **Bar chart** | 📊 | Comparaisons | Usage par namespace |
| **Table** | 📋 | Données tabulaires | Liste de pods |
| **Pie chart** | 🥧 | Proportions | Répartition ressources |
| **Heatmap** | 🔥 | Densité/Distribution | Latence distribution |
| **Graph (legacy)** | 📉 | Ancien graphique | (remplacé par Time series) |
| **State timeline** | ⏱️ | États au fil du temps | Status pods |
| **Status history** | 📅 | Historique d'états | Disponibilité |
| **Bar gauge** | 📶 | Jauge en barres | Comparaison avec seuils |
| **Text** | 📝 | Texte/Markdown | Documentation |
| **News** | 📰 | Flux RSS | Actualités |
| **Logs** | 📜 | Logs applicatifs | Logs des pods |

## Time Series (Graphiques temporels)

### Description

Le Time series est le type de visualisation le plus utilisé. Il affiche l'évolution d'une ou plusieurs métriques au fil du temps.

**Quand l'utiliser :**

- Suivre des tendances (CPU, mémoire, requêtes)
- Identifier des patterns (pics, cycles)
- Comparer plusieurs métriques simultanément
- Analyser des périodes temporelles

### Configuration de base

#### Onglet Panel options

**Title** : Nom du panneau
```
Utilisation CPU du Cluster
```

**Description** : Explication détaillée (visible au survol du `i`)
```
Affiche l'utilisation totale du CPU pour tous les conteneurs du cluster
```

**Transparent background** : Rend le fond transparent

#### Onglet Query

**Exemple de requête simple :**
```promql
sum(rate(container_cpu_usage_seconds_total{container!=""}[5m]))
```

**Multiple queries :**
Vous pouvez ajouter plusieurs requêtes pour comparer :
- Query A : CPU usage
- Query B : CPU limits
- Query C : CPU requests

#### Onglet Graph styles

**Style :**
- **Lines** : Lignes continues (par défaut)
- **Bars** : Barres verticales
- **Points** : Points individuels

**Line interpolation :**
- **Linear** : Lignes droites entre points
- **Smooth** : Lignes courbes lissées
- **Step before/after** : Marches d'escalier

**Line width** : Épaisseur (0-10 pixels)
```
2px (standard)
4px (pour les métriques importantes)
```

**Fill opacity** : Remplissage sous la courbe (0-100%)
```
0% : Ligne seule
20% : Légère transparence (recommandé)
100% : Remplissage opaque
```

**Gradient mode :**
- **None** : Couleur unie
- **Opacity** : Dégradé de transparence
- **Hue** : Dégradé de couleur

**Show points :**
- **Auto** : Points sur les valeurs clés
- **Always** : Tous les points
- **Never** : Aucun point

**Point size** : Taille des points (1-10)

**Stack series :**
- **Off** : Séries superposées
- **Normal** : Empilées
- **100%** : Pourcentage empilé

#### Onglet Axis

**Placement :**
- **Auto** : Grafana choisit automatiquement
- **Left** : Axe Y à gauche
- **Right** : Axe Y à droite
- **Hidden** : Cache l'axe

**Scale :**
- **Linear** : Échelle linéaire (par défaut)
- **Logarithmic** : Échelle logarithmique (pour grandes variations)

**Soft min/max** : Valeurs min/max suggérées
**Hard min/max** : Valeurs min/max forcées

```
Min: 0 (pour CPU et mémoire, commencez à 0)
Max: Auto (laissez Grafana décider)
```

**Label** : Nom de l'axe
```
CPU (cores)
Memory (GiB)
```

#### Onglet Legend

**Visibility :**
- **Show** : Affiche la légende
- **Hide** : Cache la légende

**Mode :**
- **List** : Liste simple des séries
- **Table** : Tableau avec statistiques

**Placement :**
- **Bottom** : En bas (recommandé, économise l'espace horizontal)
- **Right** : À droite
- **Top** : En haut

**Values to show (mode Table) :**
- **Last** : Dernière valeur
- **Min** : Valeur minimale
- **Max** : Valeur maximale
- **Mean** : Moyenne
- **Total** : Somme totale
- **First** : Première valeur

**Width** : Largeur de la légende (en mode Right)

**Legend exemplars** : Affiche les exemplars (lien vers traces)

#### Onglet Tooltip

**Tooltip mode :**
- **Single** : Affiche une seule série au survol
- **All** : Affiche toutes les séries
- **Hidden** : Désactive le tooltip

**Sort order :**
- **None** : Ordre d'origine
- **Ascending** : Croissant
- **Descending** : Décroissant

### Exemples pratiques

#### Graphique CPU avec seuils visuels

**Requête :**
```promql
sum(rate(container_cpu_usage_seconds_total{namespace="production"}[5m]))
```

**Configuration :**
- Style : Lines
- Fill opacity : 20%
- Show points : Never
- Thresholds :
  - Base (vert) : < 2 cores
  - 2 (orange) : 2-4 cores
  - 4 (rouge) : > 4 cores

#### Comparaison multi-métriques

**Queries :**
- A : `sum(container_memory_usage_bytes) / 1024^3` (Usage)
- B : `sum(container_memory_limits_bytes) / 1024^3` (Limits)

**Configuration :**
- Deux séries de couleurs différentes
- Légende en mode Table avec Last, Min, Max

#### Graphique empilé (stacked)

**Requête :**
```promql
sum(rate(container_cpu_usage_seconds_total[5m])) by (namespace)
```

**Configuration :**
- Stack series : Normal
- Fill opacity : 50%
- Permet de voir la contribution de chaque namespace

## Stat (Statistiques)

### Description

Le panneau Stat affiche une seule valeur numérique, souvent avec un indicateur de tendance ou d'état. Idéal pour les métriques clés (KPIs).

**Quand l'utiliser :**

- Afficher une métrique importante (nombre de pods, uptime)
- Montrer l'état actuel du système
- Créer un tableau de bord exécutif
- Indicateurs de performance clés (KPIs)

### Configuration

#### Calculation

Définit quelle valeur afficher parmi les résultats de la requête :

- **Last** : Dernière valeur (par défaut, pour le temps réel)
- **First** : Première valeur
- **Min** : Valeur minimale
- **Max** : Valeur maximale
- **Mean** : Moyenne
- **Sum** : Somme de toutes les valeurs
- **Count** : Nombre de points de données

#### Stat styles

**Graph mode :**
- **None** : Chiffre seul
- **Area** : Mini graphique en arrière-plan (sparkline)
  ```
  Idéal pour montrer la tendance
  ```

**Text mode :**
- **Auto** : Calcule automatiquement
- **Value** : Affiche uniquement la valeur
- **Value and name** : Valeur + nom de la métrique
- **Name** : Nom seul

**Color mode :**
- **None** : Pas de couleur basée sur la valeur
- **Value** : Le chiffre change de couleur
- **Background** : Le fond change de couleur (recommandé)
- **Background gradient** : Dégradé de couleur de fond

**Text size :**
- **Auto** : Taille automatique
- Ou spécifiez une taille (%) : `150%` pour plus grand

#### Orientation

- **Auto** : Grafana décide
- **Horizontal** : Pour plusieurs stats côte à côte
- **Vertical** : Empilées verticalement

### Exemples pratiques

#### Stat simple avec seuils

**Requête :**
```promql
count(kube_pod_status_phase{phase="Running"})
```

**Configuration :**
- Calculation : Last
- Title : "Pods actifs"
- Unit : none
- Thresholds :
  - Base (vert) : 0-50
  - 50 (orange) : 50-80
  - 80 (rouge) : > 80
- Color mode : Background

#### Stat avec sparkline

**Requête :**
```promql
sum(rate(http_requests_total[5m]))
```

**Configuration :**
- Graph mode : Area
- Color mode : Value
- Shows trending over time

#### Multiple stats

Créez 4 panneaux Stat côte à côte pour :
- Pods Running
- Pods Pending
- Pods Failed
- Total Pods

## Gauge (Jauge)

### Description

Le Gauge affiche une valeur sous forme de jauge circulaire ou horizontale avec des seuils de couleur. Parfait pour les pourcentages et valeurs bornées.

**Quand l'utiliser :**

- Afficher des pourcentages (utilisation CPU/RAM)
- Montrer la progression vers un objectif
- Visualiser des valeurs avec min/max connus
- Alertes visuelles immédiates

### Configuration

#### Gauge options

**Show threshold labels** : Affiche les valeurs des seuils sur la jauge

**Show threshold markers** : Affiche des marqueurs visuels aux seuils

**Neutral** : Valeur neutre (point de départ de la jauge)

#### Orientation

- **Auto** : Grafana choisit
- **Horizontal** : Jauge horizontale
- **Vertical** : Jauge verticale

### Exemples pratiques

#### Jauge d'utilisation CPU

**Requête :**
```promql
100 - (avg(rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100)
```

**Configuration :**
- Title : "CPU Usage %"
- Unit : Percent (0-100)
- Min : 0
- Max : 100
- Thresholds :
  - Base (vert) : 0-70%
  - 70 (orange) : 70-85%
  - 85 (rouge) : > 85%
- Show threshold labels : Oui

#### Jauge d'espace disque

**Requête :**
```promql
100 - ((node_filesystem_avail_bytes{mountpoint="/"} / node_filesystem_size_bytes{mountpoint="/"}) * 100)
```

**Configuration :**
- Title : "Disk Usage"
- Unit : percent (0-100)
- Thresholds : 0=vert, 80=orange, 90=rouge

## Bar Chart (Graphique en barres)

### Description

Le Bar chart affiche des barres horizontales ou verticales pour comparer des valeurs entre différentes catégories.

**Quand l'utiliser :**

- Comparer des valeurs entre catégories (namespaces, pods)
- Afficher un classement (top 10)
- Comparer l'avant/après
- Montrer des distributions

### Configuration

#### Bar chart options

**Orientation :**
- **Auto** : Grafana choisit
- **Horizontal** : Barres horizontales (recommandé pour noms longs)
- **Vertical** : Barres verticales

**X-Axis / Y-Axis :**
- **Label** : Nom de l'axe
- **Mode** : Categorical ou Time

**Bar width** : Largeur des barres (0-1)
```
0.5 : Barres moyennes avec espacement
0.9 : Barres larges avec peu d'espacement
```

**Group width** : Espace entre groupes de barres

**Show values** : Affiche les valeurs sur les barres
- **Auto** : Affiche si l'espace le permet
- **Always** : Toujours afficher
- **Never** : Jamais afficher

**Stacking :**
- **None** : Barres côte à côte
- **Normal** : Empilées
- **Percent** : 100% empilées

### Exemples pratiques

#### Top 10 pods par CPU

**Requête :**
```promql
topk(10, sum(rate(container_cpu_usage_seconds_total{container!=""}[5m])) by (pod))
```

**Configuration :**
- Orientation : Horizontal
- Show values : Always
- Unit : cores
- Sort : Descending

#### Comparaison par namespace

**Requête :**
```promql
sum(kube_pod_info) by (namespace)
```

**Configuration :**
- Orientation : Vertical
- Grouped by namespace
- Color : From thresholds or by series

## Table (Tableau)

### Description

Le panneau Table affiche les données sous forme tabulaire avec colonnes et lignes. Idéal pour les listes détaillées.

**Quand l'utiliser :**

- Lister des ressources (pods, deployments)
- Afficher des détails multiples
- Créer des rapports
- Combiner plusieurs métriques

### Configuration

#### Table options

**Show header** : Affiche l'en-tête du tableau

**Cell display mode** :
- **Auto** : Grafana choisit
- **Color text** : Texte coloré
- **Color background** : Fond coloré
- **Gradient gauge** : Jauge dégradée dans la cellule
- **LCD gauge** : Jauge LCD
- **JSON view** : Vue JSON

**Column width** : Largeur des colonnes
- **Auto** : Automatique
- Ou spécifiez en pixels

**Column alignment** :
- **Auto** : Automatique
- **Left** : Aligné à gauche
- **Center** : Centré
- **Right** : Aligné à droite (nombres)

**Cell type** :
- **Auto** : Détection automatique
- **String** : Texte
- **Number** : Nombre
- **Timestamp** : Horodatage

### Transformations utiles pour tables

#### Organize fields

Réorganisez et renommez les colonnes :

1. **Transform → Add transformation → Organize fields**
2. Glissez-déposez pour réorganiser
3. Renommez les colonnes pour plus de clarté
4. Cachez les colonnes inutiles

**Exemple :**
```
Time → Horodatage
pod → Nom du Pod
Value → CPU Usage
```

#### Join by field

Fusionnez plusieurs requêtes dans une table :

1. Créez plusieurs queries (A, B, C)
2. **Transform → Join by field**
3. Sélectionnez le champ commun (ex: pod)

**Exemple :**
- Query A : CPU par pod
- Query B : Memory par pod
- Join on : pod
- Résultat : Table avec pod, CPU, Memory

### Exemples pratiques

#### Liste des pods avec métriques

**Query A - CPU :**
```promql
sum(rate(container_cpu_usage_seconds_total{container!=""}[5m])) by (pod, namespace)
```

**Query B - Memory :**
```promql
sum(container_memory_usage_bytes{container!=""}) by (pod, namespace) / 1024 / 1024
```

**Configuration :**
- Transform : Join by field (pod)
- Columns : Namespace, Pod, CPU, Memory
- Cell display : Gradient gauge pour métriques
- Sort : By CPU descending

#### Table d'état des deployments

**Query A - Replicas available :**
```promql
kube_deployment_status_replicas_available
```

**Query B - Replicas desired :**
```promql
kube_deployment_spec_replicas
```

**Configuration :**
- Columns : Deployment, Available, Desired, Status
- Calculated column : Status = (Available / Desired) * 100
- Color : Vert si 100%, Rouge si < 100%

## Pie Chart (Graphique circulaire)

### Description

Le Pie chart affiche des proportions sous forme de parts de gâteau. Utile pour visualiser la répartition.

**Quand l'utiliser :**

- Afficher des proportions (% par namespace)
- Visualiser la composition (types de pods)
- Montrer la distribution des ressources
- Maximum 5-7 catégories (sinon illisible)

**Quand NE PAS l'utiliser :**

- Plus de 7-8 catégories
- Valeurs très proches (difficile à distinguer)
- Évolution dans le temps (utilisez Time series)

### Configuration

#### Pie chart options

**Pie chart type :**
- **Pie** : Cercle complet
- **Donut** : Cercle avec trou au centre (recommandé)

**Labels :**
- **Name** : Nom de la catégorie
- **Percent** : Pourcentage
- **Value** : Valeur numérique
- **Name and percent** : Combinaison (recommandé)

**Legend :**
- **Values** : Affiche les valeurs dans la légende
- **Percent** : Affiche les pourcentages

**Tooltip :**
- **Single** : Une seule part au survol
- **All** : Toutes les parts

### Exemples pratiques

#### Répartition des pods par namespace

**Requête :**
```promql
sum(kube_pod_info) by (namespace)
```

**Configuration :**
- Type : Donut
- Labels : Name and percent
- Legend : Right, show percent
- Display labels with minimum : 5% (cache petites parts)

#### Distribution de l'utilisation mémoire

**Requête :**
```promql
sum(container_memory_usage_bytes{container!=""}) by (namespace) / 1024 / 1024 / 1024
```

**Configuration :**
- Unit : GiB
- Type : Pie
- Color scheme : Classic palette

## Heatmap (Carte de chaleur)

### Description

La Heatmap affiche la densité et la distribution de valeurs avec des couleurs. Excellent pour visualiser les patterns de latence ou de distribution.

**Quand l'utiliser :**

- Distribution de latence (P50, P90, P99)
- Patterns temporels (charge par heure du jour)
- Densité de données
- Anomalies de performance

### Configuration

#### Heatmap options

**Calculate from data** : Grafana calcule automatiquement les buckets

**Y-Axis :**
- **Unit** : Unité des valeurs (ms, seconds)
- **Decimals** : Nombre de décimales
- **Scale** : Linear ou Log

**Color scheme :**
- **Interpolate** : Dégradé lisse
- **Scheme** : Palette de couleurs
  - **Greens** : Vert clair à foncé
  - **Blues** : Bleu clair à foncé
  - **Reds** : Rouge clair à foncé
  - **Spectral** : Arc-en-ciel

**Color space :**
- **RGB** : Interpolation standard
- **HCL** : Interpolation perceptuelle (meilleur pour l'œil humain)

### Exemples pratiques

#### Heatmap de latence HTTP

**Requête :**
```promql
rate(http_request_duration_seconds_bucket[5m])
```

**Configuration :**
- Y-Axis unit : seconds
- Color scheme : Reds (interpolate)
- Les zones rouges = haute latence
- Les zones vertes = basse latence

#### Distribution CPU par heure

**Requête :**
```promql
sum(rate(container_cpu_usage_seconds_total[5m])) by (hour)
```

**Configuration :**
- Permet d'identifier les heures de pointe
- Color scheme : Blues

## State Timeline (Timeline d'états)

### Description

Le State timeline affiche l'évolution des états au fil du temps. Parfait pour visualiser les changements de statut.

**Quand l'utiliser :**

- Suivi de l'état des pods (Running, Pending, Failed)
- Historique de disponibilité
- Transitions d'états
- Uptime/Downtime

### Configuration

#### State timeline options

**Value mappings** :
Mappez les valeurs numériques à des états :

```
0 → Down (Rouge)
1 → Up (Vert)
2 → Unknown (Gris)
```

**Line width** : Épaisseur des lignes d'état

**Fill opacity** : Opacité du remplissage

### Exemples pratiques

#### État des pods au fil du temps

**Requête :**
```promql
kube_pod_status_phase
```

**Value mappings :**
```
phase="Pending" → 0 (Orange)
phase="Running" → 1 (Vert)
phase="Failed" → 2 (Rouge)
phase="Unknown" → 3 (Gris)
```

**Configuration :**
- Permet de voir quand les pods ont redémarré
- Identifie les périodes d'instabilité

## Bar Gauge (Jauge en barres)

### Description

Le Bar gauge combine les barres et les jauges avec des seuils de couleur. Utile pour comparer plusieurs valeurs avec alertes visuelles.

**Quand l'utiliser :**

- Comparer plusieurs métriques avec seuils
- Afficher l'état de plusieurs ressources
- Dashboard de surveillance multi-composants
- Alternative visuelle au tableau

### Configuration

#### Bar gauge options

**Orientation :**
- **Auto** : Automatique
- **Horizontal** : Barres horizontales (recommandé)
- **Vertical** : Barres verticales

**Display mode :**
- **Gradient** : Dégradé de couleur
- **LCD** : Style LCD
- **Basic** : Couleurs plates

**Show unfilled area** : Affiche la partie non remplie en gris

### Exemples pratiques

#### Utilisation par namespace

**Requête :**
```promql
sum(rate(container_cpu_usage_seconds_total[5m])) by (namespace)
```

**Configuration :**
- Orientation : Horizontal
- Display mode : Gradient
- Thresholds : 0=vert, 2=orange, 4=rouge
- Montre visuellement quel namespace consomme le plus

## Text (Texte et Markdown)

### Description

Le panneau Text permet d'afficher du texte statique, documentation, ou contenu formaté en Markdown.

**Quand l'utiliser :**

- Documentation du dashboard
- Instructions d'utilisation
- Explication des métriques
- Liens vers ressources externes
- En-têtes et titres de sections

### Configuration

#### Text options

**Mode :**
- **Markdown** : Syntaxe Markdown (recommandé)
- **HTML** : Code HTML
- **Code** : Code avec coloration syntaxique

**Content** : Le contenu à afficher

### Exemples pratiques

#### En-tête de dashboard

```markdown
# Dashboard Kubernetes - Production

Ce dashboard affiche les métriques essentielles du cluster de production.

## Alertes
- 🔴 **Rouge** : Action immédiate requise
- 🟠 **Orange** : Attention nécessaire dans les prochaines heures
- 🟢 **Vert** : Tout fonctionne normalement

## Contact
En cas de problème : ops-team@example.com
```

#### Instructions

```markdown
## Comment utiliser ce dashboard

1. **Sélectionnez le namespace** dans le menu déroulant en haut
2. **Ajustez l'intervalle de temps** avec le sélecteur de date
3. **Cliquez sur les panneaux** pour plus de détails

### Liens utiles
- [Documentation Kubernetes](https://kubernetes.io)
- [Runbook Operations](https://wiki.example.com/runbook)
- [Escalade](https://wiki.example.com/escalation)
```

#### Variables et liens dynamiques

```markdown
## Namespace sélectionné : $namespace

[Voir les logs](https://kibana.example.com/app/discover#/?q=namespace:$namespace)

[Prometheus query](http://prometheus:9090/graph?g0.expr=kube_pod_info{namespace="$namespace"})
```

## Logs (Journaux)

### Description

Le panneau Logs affiche les logs des applications en temps réel. Nécessite une source de données compatible (Loki).

**Quand l'utiliser :**

- Debugging en temps réel
- Surveillance des erreurs applicatives
- Corrélation logs-métriques
- Investigation d'incidents

### Configuration

#### Logs options

**Show time** : Affiche l'horodatage

**Show labels** : Affiche les labels Kubernetes

**Order** :
- **Newest first** : Plus récents en haut (par défaut)
- **Oldest first** : Plus anciens en haut

**Wrap lines** : Retour à la ligne automatique

**Prettify JSON** : Formate le JSON de manière lisible

**Deduplication** : Élimine les logs dupliqués

### Exemples pratiques

#### Logs d'un pod spécifique

**Requête (Loki) :**
```logql
{namespace="$namespace", pod="$pod"}
```

**Configuration :**
- Order : Newest first
- Show time : Yes
- Wrap lines : Yes
- Utile en combinaison avec des métriques de pod

## Options communes à tous les panneaux

### Standard options

Ces options sont disponibles pour tous les types de panneaux.

#### Unit

Définit l'unité de mesure :

**Catégories principales :**

- **Misc** : none, short, percent (0-100), percent (0.0-1.0)
- **Data** : bytes, kilobytes, megabytes, gigabytes
- **Data rate** : Bps, KBps, MBps, GBps
- **Time** : nanoseconds, microseconds, milliseconds, seconds, minutes, hours
- **Throughput** : ops/sec, reads/sec, writes/sec
- **Temperature** : celsius, fahrenheit, kelvin
- **Pressure** : mbar, bar, psi, pascal

**Exemples courants pour Kubernetes :**
```
CPU : cores, millicores, percent (0-100)
Memory : bytes, megabytes (MB), gibibytes (GiB)
Network : bytes/sec, megabytes/sec
Latency : milliseconds (ms), seconds (s)
Requests : reqps (requests per second), ops/sec
```

#### Decimals

Nombre de chiffres après la virgule :
```
0 : Nombres entiers (3, 42, 156)
2 : Deux décimales (3.14, 42.00)
Auto : Grafana choisit automatiquement
```

#### Min / Max

Valeurs minimales et maximales pour l'échelle :
```
Min: 0 (pour commencer à zéro)
Max: auto (laisse Grafana calculer)
Max: 100 (pour les pourcentages)
```

#### Display name

Renomme la série pour un affichage plus clair :
```
Expression regex : .*pod="([^"]+).*
Renommer en : Pod $1
```

### Thresholds (Seuils)

Les seuils définissent les changements de couleur selon la valeur.

#### Configuration

1. **Thresholds mode** :
   - **Absolute** : Valeurs exactes (50, 100, 200)
   - **Percentage** : Pourcentages (50%, 75%, 90%)

2. **Add threshold** : Ajouter un nouveau seuil

3. **Base** : Couleur par défaut (verte généralement)

**Exemple pour CPU (cores) :**
```
Base : Vert (< 2 cores) - Normal
2 : Orange (2-4 cores) - Attention
4 : Rouge (> 4 cores) - Critique
```

**Exemple pour mémoire (%) :**
```
Base : Vert (< 70%) - Normal
70 : Orange (70-85%) - Attention
85 : Rouge (> 85%) - Critique
```

### Value mappings

Transforme les valeurs numériques en texte ou autres valeurs.

#### Types de mappings

**Value** : Mappe une valeur exacte
```
1 → "Running"
0 → "Stopped"
```

**Range** : Mappe une plage de valeurs
```
0-50 → "Low"
51-80 → "Medium"
81-100 → "High"
```

**Regex** : Mappe avec expression régulière
```
^prod.* → "Production"
^dev.* → "Development"
```

**Special** : Mappe des valeurs spéciales
```
null → "No data"
NaN → "Invalid"
0 → "Zero"
```

### Data links

Créez des liens vers d'autres dashboards ou URLs externes.

#### Configuration

1. **Title** : Texte du lien
   ```
   Voir les détails du pod
   ```

2. **URL** : Destination du lien
   ```
   https://grafana.example.com/d/pod-details?var-pod=${__field.labels.pod}
   ```

3. **Open in new tab** : Ouvre dans un nouvel onglet

#### Variables disponibles

```
${__field.labels.pod} : Label pod de la série
${__field.labels.namespace} : Label namespace
${__from} : Début de l'intervalle de temps
${__to} : Fin de l'intervalle de temps
${__value.raw} : Valeur brute
${namespace} : Variable dashboard $namespace
```

## Bonnes pratiques de visualisation

### Choisir le bon type de panneau

**Règle générale :**

| Besoin | Type recommandé |
|--------|----------------|
| Tendance temporelle | Time series |
| Valeur actuelle | Stat |
| Pourcentage avec seuils | Gauge |
| Comparaison | Bar chart |
| Détails multiples | Table |
| Proportions | Pie chart (max 7 catégories) |
| Distribution | Heatmap |
| États temporels | State timeline |
| Documentation | Text |

### Cohérence visuelle

**Couleurs :**
- Utilisez la même palette dans tout le dashboard
- Vert = Bon, Orange = Attention, Rouge = Critique
- Évitez trop de couleurs différentes

**Tailles :**
- Panneaux importants : Plus grands
- Métriques secondaires : Plus petits
- Gardez des proportions harmonieuses

**Organisation :**
- Haut du dashboard = Métriques les plus importantes
- Regroupez les panneaux liés
- Utilisez des rows pour séparer les sections

### Accessibilité

**Daltonisme :**
- Évitez rouge/vert seuls
- Utilisez des formes en plus des couleurs
- Testez avec un simulateur de daltonisme

**Contrastes :**
- Texte lisible sur tous les fonds
- Évitez les couleurs trop claires

**Taille de texte :**
- Police suffisamment grande (min 10pt)
- Titres clairs et distincts

### Performance

**Optimisez les requêtes :**
- Ajoutez des filtres (namespace, pod)
- Limitez les séries avec `topk()`
- Utilisez des agrégations (`sum`, `avg`)

**Nombre de panneaux :**
- Maximum 15-20 panneaux par dashboard
- Utilisez des rows repliables si plus
- Divisez en plusieurs dashboards si nécessaire

**Refresh rate :**
- 30s ou 1m pour monitoring général
- 5s uniquement pour debugging actif
- Évitez auto-refresh sur dashboards lourds

### Communication efficace

**Principe de la hiérarchie visuelle :**

1. **Métriques critiques** : Grandes, en haut, couleurs vives
2. **Métriques importantes** : Taille moyenne, visibles
3. **Détails** : Plus petits, en bas, ou dans rows repliables

**Nommage clair :**
```
❌ Mauvais : "CPU"
✅ Bon : "CPU Usage - Production Cluster"

❌ Mauvais : "Memory"
✅ Bon : "Memory Usage (GiB) - Last 24h"
```

**Contexte :**
- Ajoutez des panneaux Text pour expliquer
- Incluez des unités dans les titres
- Précisez la période si pertinent

## Templates et collections

### Template de panneau Stat

```json
{
  "type": "stat",
  "title": "Metric Name",
  "targets": [
    {
      "expr": "your_promql_query"
    }
  ],
  "options": {
    "reduceOptions": {
      "values": false,
      "calcs": ["lastNotNull"]
    },
    "graphMode": "area",
    "colorMode": "background"
  },
  "fieldConfig": {
    "defaults": {
      "unit": "short",
      "thresholds": {
        "mode": "absolute",
        "steps": [
          {"value": null, "color": "green"},
          {"value": 80, "color": "orange"},
          {"value": 90, "color": "red"}
        ]
      }
    }
  }
}
```

### Template de panneau Time series

```json
{
  "type": "timeseries",
  "title": "Metric Over Time",
  "targets": [
    {
      "expr": "your_promql_query"
    }
  ],
  "fieldConfig": {
    "defaults": {
      "custom": {
        "drawStyle": "line",
        "lineInterpolation": "linear",
        "fillOpacity": 20
      }
    }
  }
}
```

## Prochaines étapes

Maintenant que vous maîtrisez les panneaux et visualisations :

- **Chapitre 13.6 :** Variables et templating avancés
- **Chapitre 13.7 :** Organisation des dashboards
- **Chapitre 14 :** Alerting et notifications
- **Chapitre 15 :** Observabilité avancée

## Récapitulatif

Dans ce chapitre, vous avez appris à :

✅ Comprendre les différents types de panneaux Grafana
✅ Configurer des Time series pour suivre les tendances
✅ Utiliser des Stats et Gauges pour les valeurs clés
✅ Créer des Bar charts pour les comparaisons
✅ Construire des Tables pour les données détaillées
✅ Afficher des proportions avec des Pie charts
✅ Visualiser les distributions avec des Heatmaps
✅ Configurer les options communes (units, thresholds, mappings)
✅ Appliquer les bonnes pratiques de visualisation
✅ Optimiser la performance des dashboards

Vous disposez maintenant de tous les outils pour créer des visualisations efficaces et professionnelles adaptées à vos besoins spécifiques de monitoring Kubernetes !

---

**Conseil final :** La visualisation est un art autant qu'une science. Expérimentez avec différents types de panneaux pour le même ensemble de données. Demandez-vous toujours : "Ce graphique communique-t-il l'information de la manière la plus claire et rapide possible ?" Si la réponse est non, essayez un autre type de visualisation !

⏭️ [Variables et templating dans Grafana](/13-visualisation-avec-grafana/06-variables-et-templating-dans-grafana.md)
