🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 13.6 Variables et templating dans Grafana

## Introduction

Les variables (ou template variables) sont l'une des fonctionnalités les plus puissantes de Grafana. Elles permettent de créer des dashboards dynamiques et réutilisables qui s'adaptent aux besoins de l'utilisateur en temps réel.

### Qu'est-ce qu'une variable ?

Une variable est un **espace réservé** dans votre dashboard que vous pouvez modifier dynamiquement. Imaginez un dashboard comme un formulaire où vous pouvez choisir ce que vous voulez voir.

**Analogie simple :**
```
Dashboard sans variables = Téléviseur avec une seule chaîne
Dashboard avec variables = Téléviseur avec télécommande pour changer de chaîne
```

### Pourquoi utiliser des variables ?

**Avantages principaux :**

- **Flexibilité** : Un seul dashboard pour plusieurs contextes (dev, staging, production)
- **Réutilisabilité** : Même dashboard pour tous les namespaces, pods, nodes
- **Efficacité** : Évite de créer 10 dashboards différents pour 10 namespaces
- **Interactivité** : L'utilisateur choisit ce qu'il veut voir
- **Maintenance** : Modifier un dashboard au lieu de dix

### Exemple concret

**Sans variables :**
```
Dashboard 1 : Production Namespace
Dashboard 2 : Staging Namespace
Dashboard 3 : Development Namespace
→ 3 dashboards à maintenir
```

**Avec variables :**
```
Dashboard unique avec variable $namespace
L'utilisateur sélectionne : Production | Staging | Development
→ 1 seul dashboard à maintenir !
```

## Accéder aux variables

### Où trouver les variables

1. **Ouvrez votre dashboard**
2. Cliquez sur l'icône **engrenage** (⚙️) en haut à droite
3. Sélectionnez **Variables**
4. Cliquez sur **New variable** ou **Add variable**

### Interface de configuration

L'interface de configuration d'une variable contient plusieurs sections :

- **General** : Nom, type, label
- **Query options** : Source des données
- **Selection options** : Comportement de sélection
- **Value groups/tags** : Organisation des valeurs
- **Preview of values** : Aperçu des valeurs disponibles

## Types de variables

Grafana propose plusieurs types de variables, chacun avec son utilité.

### 1. Query (Requête)

**Description :** Récupère les valeurs depuis une source de données (Prometheus).

**Usage :** Le plus utilisé pour Kubernetes (namespaces, pods, nodes).

**Exemple :**
```
Récupère tous les namespaces depuis Prometheus
```

### 2. Custom (Personnalisée)

**Description :** Liste manuelle de valeurs fixes.

**Usage :** Environnements, régions, ou toute liste prédéfinie.

**Exemple :**
```
production, staging, development
```

### 3. Constant (Constante)

**Description :** Valeur unique non modifiable.

**Usage :** Paramètres globaux, cluster name, région fixe.

**Exemple :**
```
cluster_name = "prod-k8s-01"
```

### 4. Interval (Intervalle)

**Description :** Intervalles de temps prédéfinis.

**Usage :** Changer la résolution des graphiques (1m, 5m, 15m).

**Exemple :**
```
1m, 5m, 15m, 30m, 1h
```

### 5. Data source (Source de données)

**Description :** Liste des sources de données disponibles.

**Usage :** Dashboards multi-clusters avec plusieurs Prometheus.

**Exemple :**
```
Prometheus-Cluster-A, Prometheus-Cluster-B
```

### 6. Text box (Zone de texte)

**Description :** Champ de saisie libre.

**Usage :** Recherche personnalisée, expressions regex.

**Exemple :**
```
Nom de pod à rechercher
```

### 7. Ad hoc filters (Filtres à la volée)

**Description :** Filtres dynamiques créés par l'utilisateur.

**Usage :** Exploration libre des données.

**Exemple :**
```
Ajouter des filtres comme: pod=nginx, namespace=prod
```

## Créer votre première variable : Namespace

Créons ensemble une variable pour sélectionner un namespace Kubernetes.

### Étape 1 : Configuration générale

**Name :**
```
namespace
```

**Important :** Pas d'espaces, pas de caractères spéciaux. Ce nom sera utilisé comme `$namespace` dans les requêtes.

**Label :**
```
Namespace
```

C'est le texte affiché à l'utilisateur dans le menu déroulant.

**Type :**
```
Query
```

**Description (optionnel) :**
```
Sélectionnez le namespace Kubernetes à surveiller
```

**Show on dashboard :**
```
✓ (coché)
```

La variable sera visible en haut du dashboard.

### Étape 2 : Configuration de la requête

**Data source :**
```
Prometheus
```

Sélectionnez votre source de données Prometheus.

**Refresh :**
```
On dashboard load
```

Quand rafraîchir les valeurs de la variable :
- **Never** : Jamais (valeurs statiques)
- **On dashboard load** : Au chargement du dashboard (recommandé)
- **On time range change** : Quand l'intervalle de temps change

**Query :**
```promql
label_values(kube_namespace_labels, namespace)
```

**Explication de cette requête :**
- `label_values()` : Fonction pour récupérer les valeurs d'un label
- `kube_namespace_labels` : Métrique qui contient tous les namespaces
- `namespace` : Le label qu'on veut récupérer

**Alternatives de requêtes :**

Si la première ne fonctionne pas, essayez :
```promql
label_values(kube_pod_info, namespace)
```

Ou pour les namespaces avec des pods actifs uniquement :
```promql
label_values(kube_pod_status_phase{phase="Running"}, namespace)
```

**Regex (optionnel) :**

Pour filtrer les résultats, par exemple exclure `kube-system` :
```
/^(?!kube-system$).*/
```

Ou pour ne garder que certains namespaces :
```
/^(production|staging|development)$/
```

**Sort :**
```
Alphabetical (asc)
```

Options :
- **Disabled** : Ordre naturel
- **Alphabetical (asc)** : A-Z (recommandé)
- **Alphabetical (desc)** : Z-A
- **Numerical (asc)** : 1-10
- **Alphabetical (case-insensitive)** : Ignore majuscules/minuscules

### Étape 3 : Options de sélection

**Multi-value :**
```
✓ (coché)
```

Permet de sélectionner plusieurs namespaces à la fois.

**Include All option :**
```
✓ (coché)
```

Ajoute une option "All" pour sélectionner tous les namespaces.

**Custom all value (optionnel) :**
```
.*
```

Expression regex pour "All". Laissez vide si vous voulez que Grafana gère automatiquement.

### Étape 4 : Aperçu et sauvegarde

**Preview of values :**

Vous devriez voir la liste de vos namespaces :
```
All
default
kube-system
production
staging
```

**Cliquez sur :**
```
Update
```

Votre variable est créée ! Elle apparaît maintenant en haut du dashboard.

## Utiliser les variables dans les requêtes

### Syntaxe de base

Pour utiliser une variable dans une requête PromQL :

```promql
votre_metrique{namespace="$namespace"}
```

Le `$namespace` sera remplacé par la valeur sélectionnée.

### Exemples d'utilisation

#### Compter les pods dans le namespace sélectionné

```promql
count(kube_pod_info{namespace="$namespace"})
```

#### CPU usage pour un namespace

```promql
sum(rate(container_cpu_usage_seconds_total{namespace="$namespace"}[5m]))
```

#### Memory usage avec plusieurs filtres

```promql
sum(container_memory_usage_bytes{namespace="$namespace", container!=""}) / 1024 / 1024 / 1024
```

### Multi-value : Syntaxe pour plusieurs valeurs

Si l'utilisateur sélectionne plusieurs namespaces, utilisez l'opérateur `=~` :

```promql
kube_pod_info{namespace=~"$namespace"}
```

**Explication :**
- `=` : Égalité exacte (une seule valeur)
- `=~` : Regex match (permet plusieurs valeurs)

**Exemple :**
Si l'utilisateur sélectionne "production" et "staging", la requête devient :
```promql
kube_pod_info{namespace=~"production|staging"}
```

### Variable "All"

Quand l'utilisateur sélectionne "All", utilisez toujours `=~` :

```promql
kube_pod_info{namespace=~"$namespace"}
```

Si "Custom all value" est défini sur `.*`, cela devient :
```promql
kube_pod_info{namespace=~".*"}
```

Ce qui correspond à tous les namespaces.

## Variables communes pour Kubernetes

### Variable : Pod

**Configuration :**

```
Name: pod
Type: Query
Data source: Prometheus
Query: label_values(kube_pod_info{namespace="$namespace"}, pod)
```

**Note :** Cette variable dépend de `$namespace` (dépendance en chaîne).

**Utilisation :**
```promql
container_cpu_usage_seconds_total{namespace="$namespace", pod=~"$pod"}
```

### Variable : Node

**Configuration :**

```
Name: node
Type: Query
Query: label_values(kube_node_info, node)
```

**Utilisation :**
```promql
node_cpu_seconds_total{node=~"$node"}
```

### Variable : Container

**Configuration :**

```
Name: container
Type: Query
Query: label_values(container_cpu_usage_seconds_total{namespace="$namespace", pod=~"$pod"}, container)
```

**Utilisation :**
```promql
container_memory_usage_bytes{namespace="$namespace", pod=~"$pod", container="$container"}
```

### Variable : Deployment

**Configuration :**

```
Name: deployment
Type: Query
Query: label_values(kube_deployment_labels{namespace="$namespace"}, deployment)
```

**Utilisation :**
```promql
kube_deployment_status_replicas{namespace="$namespace", deployment="$deployment"}
```

### Variable : Cluster (pour multi-cluster)

**Configuration :**

```
Name: cluster
Type: Custom
Values: cluster-a,cluster-b,cluster-c
```

**Ou avec Query (si métrique disponible) :**
```
Query: label_values(kube_node_info, cluster)
```

**Utilisation :**
```promql
kube_pod_info{cluster="$cluster", namespace="$namespace"}
```

### Variable : Environnement

**Configuration :**

```
Name: environment
Type: Custom
Values: production,staging,development
```

**Utilisation :**
```promql
kube_pod_labels{label_environment="$environment"}
```

### Variable : Interval

**Configuration :**

```
Name: interval
Type: Interval
Values: 1m,5m,15m,30m,1h,2h
Auto: Yes
```

**Utilisation :**
```promql
rate(container_cpu_usage_seconds_total[$interval])
```

Permet à l'utilisateur de changer la résolution du graphique dynamiquement.

## Variables en chaîne (Chained variables)

Les variables peuvent dépendre les unes des autres, créant une hiérarchie logique.

### Exemple : Cluster → Namespace → Pod

**Variable 1 : Cluster**
```
Name: cluster
Type: Query
Query: label_values(kube_node_info, cluster)
```

**Variable 2 : Namespace**
```
Name: namespace
Type: Query
Query: label_values(kube_pod_info{cluster="$cluster"}, namespace)
```

**Variable 3 : Pod**
```
Name: pod
Type: Query
Query: label_values(kube_pod_info{cluster="$cluster", namespace="$namespace"}, pod)
```

**Comportement :**
1. L'utilisateur sélectionne un cluster
2. La liste des namespaces se met à jour automatiquement
3. La liste des pods se met à jour selon le namespace sélectionné

### Ordre des variables

**Important :** L'ordre des variables dans la liste détermine l'ordre d'affichage ET l'ordre d'évaluation.

**Glissez-déposez** les variables pour les réorganiser :
```
1. cluster    ↑↓
2. namespace  ↑↓
3. pod        ↑↓
```

## Variables avancées

### Variable avec filtres regex

Pour ne montrer que certaines valeurs :

**Exemple : Exclure les namespaces système**

```
Name: namespace
Query: label_values(kube_namespace_labels, namespace)
Regex: /^(?!kube-.*|default).*$/
```

**Explication du regex :**
- `^` : Début de la chaîne
- `(?!...)` : Negative lookahead (n'inclut pas si correspond)
- `kube-.*` : Commence par "kube-"
- `|` : OU
- `default` : Exactement "default"
- `.*` : N'importe quel autre caractère
- `$` : Fin de la chaîne

**Résultat :** Exclut `kube-system`, `kube-public`, `default`, etc.

### Variable avec format personnalisé

**Format options :**

Contrôle comment la variable est interpolée dans les requêtes.

**Glob (par défaut) :**
```
Format: ${namespace:glob}
Résultat: production|staging|development
Usage: namespace=~"${namespace:glob}"
```

**Regex :**
```
Format: ${namespace:regex}
Résultat: (production|staging|development)
Usage: namespace=~"${namespace:regex}"
```

**Pipe :**
```
Format: ${namespace:pipe}
Résultat: production|staging|development
```

**Distributed (pour Graphite) :**
```
Format: ${namespace:distributed}
```

**CSV :**
```
Format: ${namespace:csv}
Résultat: production,staging,development
```

**JSON :**
```
Format: ${namespace:json}
Résultat: ["production","staging","development"]
```

### Variable calculée

Créez une variable basée sur une autre variable.

**Exemple : Suffix pour environnement**

**Variable 1 : Environment**
```
Name: env
Type: Custom
Values: prod,staging,dev
```

**Variable 2 : Cluster Name**
```
Name: cluster_name
Type: Custom
Values: k8s-$env-01
```

### Variable avec valeur par défaut

Définissez une valeur par défaut quand le dashboard se charge.

**Dans l'URL :**
```
http://grafana.example.com/d/dashboard-id?var-namespace=production
```

**Dans la configuration de la variable :**

Malheureusement, Grafana ne permet pas de définir une valeur par défaut directement dans l'interface pour les variables Query. Mais vous pouvez :

1. **Utiliser l'ordre :** La première valeur est sélectionnée par défaut
2. **Utiliser Selection options → Default :** Pour variables Custom
3. **Utiliser l'URL** avec `var-namespace=production`

## Utiliser les variables dans les titres et texte

### Dans les titres de panneaux

Les variables fonctionnent aussi dans les titres !

**Exemple :**
```
CPU Usage - Namespace: $namespace
```

**Avec plusieurs variables :**
```
$cluster / $namespace / $pod - CPU Usage
```

**Résultat affiché :**
```
prod-k8s / production / nginx-deployment-abc123 - CPU Usage
```

### Dans les panneaux Text

**Markdown avec variables :**

```markdown
# Dashboard pour ${cluster}

## Namespace sélectionné : **${namespace}**

### Statistiques

- Cluster : $cluster
- Namespace : $namespace
- Pod : $pod
- Interval : $interval

[Voir les logs](https://kibana.example.com/app/discover#/?q=namespace:${namespace})
```

### Dans les descriptions

Les descriptions de panneaux peuvent aussi utiliser des variables :

```
Ce panneau affiche l'utilisation CPU pour le namespace ${namespace}
dans le cluster ${cluster}.
```

## Variables dans les Data Links

Créez des liens dynamiques vers d'autres dashboards ou outils externes.

### Lien vers un autre dashboard

**Configuration du data link :**

```
Title: Détails du pod
URL: /d/pod-details?var-namespace=${namespace}&var-pod=${__field.labels.pod}
```

**Variables disponibles dans les data links :**

- `${namespace}` : Variable du dashboard
- `${__field.labels.pod}` : Label de la série de données
- `${__from}` : Timestamp de début
- `${__to}` : Timestamp de fin
- `${__value.raw}` : Valeur brute du point

### Lien vers Prometheus

```
URL: http://prometheus:9090/graph?g0.expr=kube_pod_info{namespace="${namespace}",pod="${pod}"}
```

### Lien vers Kibana

```
URL: https://kibana.example.com/app/discover#/?_a=(query:(query_string:(query:'kubernetes.namespace:${namespace} AND kubernetes.pod.name:${pod}')))
```

## Répétition de panneaux avec variables

Créez automatiquement un panneau pour chaque valeur d'une variable.

### Configuration

1. **Créez votre panneau normalement**
2. **Éditez le panneau**
3. **Panel options → Repeat options**
4. **Repeat by variable :** Sélectionnez votre variable (ex: `namespace`)
5. **Max per row :** Nombre de panneaux par ligne (4-6 recommandé)

### Exemple pratique

**Panneau unique :**
```
Title: CPU Usage - $namespace
Query: sum(rate(container_cpu_usage_seconds_total{namespace="$namespace"}[5m]))
Repeat by: namespace
```

**Résultat :**
Si vous avez 3 namespaces (prod, staging, dev), Grafana créera automatiquement 3 panneaux :
```
[CPU Usage - prod]  [CPU Usage - staging]  [CPU Usage - dev]
```

### Répétition de rows

Vous pouvez aussi répéter des rows entières :

1. **Créez une row**
2. **Row settings → Repeat for :** Sélectionnez une variable
3. Tous les panneaux de cette row seront répétés

## Bonnes pratiques pour les variables

### Nommage des variables

**Conventions recommandées :**

```
✅ Bon : namespace, pod, node, cluster
❌ Mauvais : ns, p, n, c

✅ Bon : environment, region
❌ Mauvais : env, reg

✅ Bon : time_interval
❌ Mauvais : ti
```

**Règles :**
- Noms descriptifs en minuscules
- Underscores pour séparer les mots
- Pas de caractères spéciaux
- Cohérent dans tous vos dashboards

### Organisation des variables

**Ordre logique :**

```
1. Cluster / Data source (le plus global)
2. Namespace
3. Deployment / StatefulSet
4. Pod
5. Container (le plus spécifique)
6. Time interval (à la fin)
```

**Visibilité :**

Ne montrez que les variables que l'utilisateur doit modifier :

```
✓ Affichée : namespace, pod (contrôle utilisateur)
✗ Cachée : cluster_name (si toujours le même)
✗ Cachée : datasource (si unique)
```

### Performance

**Évitez les requêtes lourdes :**

❌ Mauvais :
```promql
label_values(container_cpu_usage_seconds_total, pod)
```
Retourne potentiellement des milliers de pods.

✅ Bon :
```promql
label_values(kube_pod_info{namespace="$namespace"}, pod)
```
Filtré par namespace d'abord.

**Refresh strategy :**

- **On dashboard load** : Pour des valeurs qui changent rarement
- **Never** : Pour des valeurs statiques (environnements)
- Évitez **On time range change** si non nécessaire (charge supplémentaire)

### Multi-value vs Single-value

**Utilisez Multi-value quand :**
- Comparaison utile (plusieurs namespaces, plusieurs pods)
- L'utilisateur peut vouloir voir plusieurs éléments

**Utilisez Single-value quand :**
- Un seul choix logique à la fois
- Dashboards focused sur un élément spécifique

### Include All option

**Activez "All" quand :**
- Vue d'ensemble nécessaire
- Métriques agrégées pertinentes

**Désactivez "All" quand :**
- Trop de données simultanées
- Performance impactée
- Visualisation illisible avec toutes les valeurs

## Cas d'usage avancés

### Variable dynamique basée sur une métrique

Montrez uniquement les pods qui consomment plus de X% de CPU :

```promql
label_values(
  (rate(container_cpu_usage_seconds_total[5m]) > 0.5),
  pod
)
```

### Variable avec labels Kubernetes

Récupérez les valeurs depuis des labels custom :

```promql
label_values(kube_pod_labels{label_app="myapp"}, label_version)
```

Permet de filtrer par version d'application.

### Variable conditionnelle

Créez une variable qui change selon une autre variable.

**Variable environment :**
```
prod, staging, dev
```

**Variable namespace (dépend de environment) :**

Utilisez un regex pour mapper :
```
Query: label_values(kube_namespace_labels, namespace)
Regex: /${environment}.*/
```

Si environment = "prod", montre seulement les namespaces commençant par "prod".

### Variable pour alerting threshold

```
Name: cpu_alert_threshold
Type: Custom
Values: 0.5,0.7,0.8,0.9
Default: 0.7
```

**Utilisation dans une requête d'alerte :**
```promql
avg(rate(container_cpu_usage_seconds_total[5m])) > $cpu_alert_threshold
```

## Debugging des variables

### La variable ne s'affiche pas

**Vérifications :**

1. **Show on dashboard** est coché ?
2. La variable a-t-elle des valeurs ? (Preview of values)
3. La requête retourne-t-elle des résultats ?

**Test de la requête :**
1. Allez dans **Explore**
2. Testez votre requête `label_values(...)`
3. Vérifiez qu'elle retourne des résultats

### Valeurs vides ou incorrectes

**Causes possibles :**

1. **Métrique inexistante**
   - Vérifiez dans Prometheus que la métrique existe

2. **Labels incorrects**
   - Utilisez l'auto-complétion dans Explore

3. **Regex trop restrictif**
   - Testez votre regex sur regex101.com

4. **Refresh non configuré**
   - Définissez "On dashboard load"

### Variable dépendante ne se met pas à jour

**Solution :**

1. Vérifiez l'ordre des variables (la dépendance doit être avant)
2. Vérifiez que la requête utilise bien `$variable_precedente`
3. Testez la requête manuellement dans Explore

**Exemple :**

Si `pod` ne se met pas à jour quand vous changez `namespace` :

```
Vérifiez que la requête pod contient : namespace="$namespace"
```

### Performance lente

**Si les variables sont lentes à charger :**

1. **Simplifiez les requêtes**
   - Ajoutez des filtres
   - Limitez avec `topk(100, ...)`

2. **Changez le refresh**
   - "Never" si valeurs statiques
   - "On dashboard load" au lieu de "On time range change"

3. **Cachez les résultats Prometheus**
   - Augmentez le cache de Prometheus

## Templates de variables

### Template pour un dashboard Kubernetes basique

```yaml
Variables:
  1. namespace:
     Type: Query
     Query: label_values(kube_namespace_labels, namespace)
     Multi: Yes
     Include All: Yes

  2. pod:
     Type: Query
     Query: label_values(kube_pod_info{namespace="$namespace"}, pod)
     Multi: Yes
     Include All: Yes

  3. interval:
     Type: Interval
     Values: 1m,5m,15m,30m,1h
     Auto: Yes
```

### Template pour monitoring applicatif

```yaml
Variables:
  1. environment:
     Type: Custom
     Values: production,staging,development

  2. cluster:
     Type: Query
     Query: label_values(kube_node_info{environment="$environment"}, cluster)

  3. application:
     Type: Query
     Query: label_values(kube_pod_labels{cluster="$cluster"}, label_app)

  4. version:
     Type: Query
     Query: label_values(kube_pod_labels{label_app="$application"}, label_version)
```

### Template pour troubleshooting

```yaml
Variables:
  1. namespace:
     Type: Query
     Query: label_values(kube_pod_info, namespace)
     Default: All

  2. pod_status:
     Type: Custom
     Values: Running,Pending,Failed,Unknown
     Multi: Yes
     Include All: Yes

  3. search_pod:
     Type: Text box
     Default: .*
```

## Export et import de variables

### Exporter un dashboard avec variables

1. **Dashboard settings** → **JSON Model**
2. Copiez le JSON complet
3. Les variables sont dans la section `templating`

**Structure JSON des variables :**

```json
{
  "templating": {
    "list": [
      {
        "name": "namespace",
        "type": "query",
        "datasource": "Prometheus",
        "query": "label_values(kube_namespace_labels, namespace)",
        "multi": true,
        "includeAll": true,
        "refresh": 1
      }
    ]
  }
}
```

### Réutiliser les variables dans un autre dashboard

1. Copiez la section `templating` du JSON
2. Collez-la dans le nouveau dashboard
3. Adaptez si nécessaire

## Intégration avec d'autres fonctionnalités

### Variables et alertes

Les variables peuvent être utilisées dans les règles d'alerte :

```
Alert condition:
WHEN avg() OF query(A, 5m, now) IS ABOVE $cpu_alert_threshold
```

### Variables et annotations

Créez des annotations dynamiques :

```
Query: kube_pod_start_time{namespace="$namespace"}
Text: Pod started: {{ pod }}
```

### Variables et transformations

Utilisez des variables dans les transformations de données :

```
Filter data by values:
Field: namespace
Match: $namespace
```

## Prochaines étapes

Maintenant que vous maîtrisez les variables et le templating :

- **Chapitre 13.7 :** Organisation avancée des dashboards
- **Chapitre 14 :** Alerting et notifications
- **Chapitre 15 :** Observabilité avancée avec corrélation

## Récapitulatif

Dans ce chapitre, vous avez appris à :

✅ Comprendre le concept de variables et leur utilité
✅ Créer différents types de variables (Query, Custom, Interval)
✅ Configurer des variables pour Kubernetes (namespace, pod, node)
✅ Utiliser les variables dans les requêtes PromQL
✅ Créer des variables en chaîne (dépendances)
✅ Utiliser les variables dans les titres et textes
✅ Répéter automatiquement des panneaux avec les variables
✅ Appliquer les bonnes pratiques de nommage et performance
✅ Déboguer les problèmes courants de variables
✅ Créer des dashboards dynamiques et réutilisables

Les variables sont un outil extrêmement puissant qui transforme vos dashboards statiques en interfaces interactives et flexibles. Avec la maîtrise des variables, vous pouvez créer des dashboards universels qui s'adaptent à tous les contextes !

---

**Conseil final :** Commencez avec 2-3 variables simples (namespace, pod), puis ajoutez progressivement de la complexité. Les variables rendent vos dashboards plus puissants, mais trop de variables peuvent aussi les rendre confus. Trouvez le bon équilibre entre flexibilité et simplicité d'utilisation. Une règle d'or : si vous avez plus de 7-8 variables visibles, votre dashboard est probablement trop complexe !

⏭️ [Organisation des dashboards](/13-visualisation-avec-grafana/07-organisation-des-dashboards.md)
