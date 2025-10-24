🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 13.3 Dashboards Kubernetes pré-configurés

## Introduction

Les dashboards pré-configurés sont des tableaux de bord créés et partagés par la communauté Grafana. Au lieu de créer vos propres dashboards à partir de zéro (ce qui peut être long et complexe), vous pouvez importer des dashboards déjà prêts et optimisés pour Kubernetes.

### Pourquoi utiliser des dashboards pré-configurés ?

**Avantages :**

- **Gain de temps** : Des dashboards professionnels en quelques clics
- **Expertise incluse** : Créés par des experts qui connaissent les bonnes métriques à surveiller
- **Meilleures pratiques** : Intègrent les standards de l'industrie
- **Point de départ** : Base solide que vous pouvez personnaliser ensuite
- **Apprentissage** : Excellente façon d'apprendre PromQL et les bonnes pratiques

**Ce que vous pouvez surveiller avec des dashboards Kubernetes :**

- État général du cluster
- Utilisation CPU et mémoire des nœuds
- Performances des Pods
- État des Deployments et StatefulSets
- Métriques réseau
- Utilisation du stockage
- État des conteneurs
- Et bien plus encore...

## La bibliothèque de dashboards Grafana

### Grafana.com/dashboards

La communauté Grafana partage des milliers de dashboards sur le site officiel : [grafana.com/grafana/dashboards](https://grafana.com/grafana/dashboards)

**Caractéristiques de la bibliothèque :**

- Plus de 10 000 dashboards disponibles
- Système de notation et avis
- Recherche par catégorie et tags
- Dashboards spécifiques à Kubernetes, Docker, Prometheus, etc.
- Mises à jour régulières par la communauté

### Comprendre les identifiants de dashboard

Chaque dashboard a un **ID unique** (numéro) qui permet de l'importer facilement dans Grafana.

**Exemple :**
```
Dashboard : Kubernetes Cluster Monitoring
ID : 315
URL : https://grafana.com/grafana/dashboards/315
```

## Dashboards Kubernetes recommandés

Voici une sélection des dashboards les plus populaires et utiles pour surveiller un cluster Kubernetes avec MicroK8s.

### 1. Kubernetes Cluster Monitoring (Dashboard ID: 315)

**Créé par :** Instrumentisto

**Ce qu'il affiche :**
- Vue d'ensemble du cluster
- Utilisation CPU et mémoire par namespace
- Nombre de Pods en cours d'exécution
- Utilisation réseau
- Performance des conteneurs

**Idéal pour :** Vue d'ensemble rapide de la santé du cluster

**Niveau :** Débutant à Intermédiaire

### 2. Kubernetes Cluster (Prometheus) (Dashboard ID: 6417)

**Créé par :** Grafana Labs

**Ce qu'il affiche :**
- Métriques détaillées du cluster
- État des nœuds
- Utilisation des ressources par Pod
- Taux d'erreur et latence
- Métriques de l'API Kubernetes

**Idéal pour :** Monitoring approfondi du cluster

**Niveau :** Intermédiaire

### 3. Kubernetes Pod Resources (Dashboard ID: 6781)

**Créé par :** Dotdc

**Ce qu'il affiche :**
- Consommation CPU et mémoire par Pod
- Requests et Limits
- Comparaison entre namespaces
- Identification des Pods gourmands en ressources

**Idéal pour :** Optimisation des ressources

**Niveau :** Débutant à Intermédiaire

### 4. Kubernetes Node Exporter Full (Dashboard ID: 1860)

**Créé par :** Idealo

**Ce qu'il affiche :**
- Métriques système des nœuds (CPU, mémoire, disque, réseau)
- Charge système
- Utilisation des inodes
- Performance I/O

**Idéal pour :** Surveillance infrastructure des nœuds

**Niveau :** Intermédiaire à Avancé

### 5. Kubernetes Deployment Statefulset Daemonset Metrics (Dashboard ID: 8588)

**Créé par :** Banzai Cloud

**Ce qu'il affiche :**
- État des Deployments
- StatefulSets
- DaemonSets
- Replicas disponibles vs désirées
- Historique des rollouts

**Idéal pour :** Suivi des déploiements

**Niveau :** Intermédiaire

### 6. Kubernetes Persistent Volumes (Dashboard ID: 13646)

**Créé par :** Various contributors

**Ce qu'il affiche :**
- Utilisation des volumes persistants
- Capacité disponible
- Taux de remplissage
- PVC par namespace

**Idéal pour :** Gestion du stockage

**Niveau :** Intermédiaire

### 7. Kubernetes API Server (Dashboard ID: 12006)

**Créé par :** Various contributors

**Ce qu'il affiche :**
- Performance de l'API Kubernetes
- Latence des requêtes
- Taux d'erreur
- Charge de l'API server

**Idéal pour :** Diagnostic de performance avancé

**Niveau :** Avancé

## Comment importer un dashboard

### Méthode 1 : Import via l'ID (Recommandée)

C'est la méthode la plus simple et la plus rapide.

**Étapes :**

1. **Connectez-vous à Grafana** (`http://localhost:3000`)

2. **Accédez au menu d'import :**
   - Cliquez sur l'icône **+** dans le menu latéral gauche
   - Sélectionnez **Import**
   - Ou allez directement dans **Dashboards → Import**

3. **Entrez l'ID du dashboard :**
   - Dans le champ **Import via grafana.com**, tapez l'ID
   - Par exemple : `315` pour le dashboard "Kubernetes Cluster Monitoring"
   - Cliquez sur **Load**

4. **Configuration du dashboard :**
   - **Name** : Le nom est pré-rempli, vous pouvez le modifier
   - **Folder** : Choisissez un dossier (ou créez-en un, exemple : "Kubernetes")
   - **Unique identifier (UID)** : Laissez Grafana générer automatiquement
   - **Prometheus** : Sélectionnez votre source de données Prometheus

5. **Importez :**
   - Cliquez sur **Import**
   - Le dashboard s'affiche immédiatement avec vos données !

### Méthode 2 : Import via JSON

Si vous avez téléchargé le fichier JSON d'un dashboard ou si vous voulez partager un dashboard personnalisé.

**Étapes :**

1. **Obtenez le fichier JSON :**
   - Téléchargez-le depuis grafana.com
   - Ou copiez le JSON depuis un autre Grafana

2. **Accédez à l'import :**
   - **Dashboards → Import**

3. **Uploadez le JSON :**
   - Cliquez sur **Upload JSON file**
   - Sélectionnez votre fichier `.json`
   - Ou collez le JSON directement dans la zone de texte

4. **Configurez et importez** (même processus que la méthode 1)

### Méthode 3 : Import via URL

Si vous avez l'URL complète du dashboard.

**Étapes :**

1. **Accédez à l'import :**
   - **Dashboards → Import**

2. **Entrez l'URL :**
   ```
   https://grafana.com/grafana/dashboards/315
   ```

3. **Load et configurez** (même processus)

## Configuration post-import

Une fois un dashboard importé, vous devrez peut-être effectuer quelques ajustements.

### Sélection de la source de données

Si vous avez plusieurs sources Prometheus, sélectionnez la bonne :

1. En haut du dashboard, vous verrez un menu déroulant **datasource**
2. Sélectionnez votre source Prometheus
3. Le dashboard se met à jour automatiquement

### Ajustement des variables

Certains dashboards utilisent des **variables** pour filtrer les données.

**Variables communes :**

- **cluster** : Nom du cluster (si vous en avez plusieurs)
- **namespace** : Filtrer par namespace Kubernetes
- **node** : Sélectionner un nœud spécifique
- **pod** : Sélectionner un pod spécifique

**Comment utiliser les variables :**

1. En haut du dashboard, vous verrez des menus déroulants
2. Sélectionnez les valeurs souhaitées
3. Le dashboard filtre automatiquement les données

**Exemple :**
```
Cluster: [default]  Namespace: [kube-system]  Node: [node-01]
```

### Intervalles de temps

Grafana permet de sélectionner la période de temps à afficher :

1. **En haut à droite**, cliquez sur l'icône d'horloge
2. Sélectionnez une période :
   - **Last 5 minutes** : Données très récentes
   - **Last 1 hour** : Vue horaire
   - **Last 24 hours** : Journée complète
   - **Last 7 days** : Vue hebdomadaire
3. Ou définissez une plage personnalisée avec **Absolute time range**

### Rafraîchissement automatique

Pour voir les données en temps réel :

1. **En haut à droite**, cliquez sur l'icône de rafraîchissement
2. Sélectionnez l'intervalle :
   - **5s** : Très fréquent (pour du monitoring actif)
   - **30s** : Équilibre performance/fraîcheur
   - **1m** : Standard
   - **Off** : Pas de rafraîchissement automatique

## Organisation de vos dashboards

### Création de dossiers

Pour organiser vos dashboards importés, créez des dossiers thématiques :

1. **Accédez à :** Dashboards → Browse
2. Cliquez sur **New folder**
3. Nommez votre dossier (exemples ci-dessous)

**Structure recommandée pour Kubernetes :**

```
📁 Kubernetes - Overview
   ├─ Cluster Monitoring (ID: 315)
   └─ Cluster Status (ID: 6417)

📁 Kubernetes - Workloads
   ├─ Pod Resources (ID: 6781)
   ├─ Deployments (ID: 8588)
   └─ StatefulSets & DaemonSets

📁 Kubernetes - Infrastructure
   ├─ Node Metrics (ID: 1860)
   ├─ Persistent Volumes (ID: 13646)
   └─ Network Performance

📁 Kubernetes - Advanced
   ├─ API Server (ID: 12006)
   └─ etcd Monitoring
```

### Déplacer un dashboard vers un dossier

1. Ouvrez le dashboard
2. Cliquez sur l'icône **engrenage** (⚙️) en haut à droite
3. Sélectionnez **Settings**
4. Dans **General**, changez le **Folder**
5. Cliquez sur **Save dashboard**

### Dashboard comme page d'accueil

Pour définir un dashboard comme page d'accueil de Grafana :

1. Ouvrez le dashboard souhaité
2. Cliquez sur l'icône **étoile** (⭐) pour le marquer comme favori
3. Allez dans **Configuration → Preferences**
4. **Home Dashboard** : Sélectionnez votre dashboard
5. **Save**

Désormais, ce dashboard s'affichera automatiquement à la connexion.

## Problèmes courants et solutions

### Problème : "No data" ou panneaux vides

**Causes possibles :**

1. **Prometheus ne collecte pas ces métriques**

   **Solution :** Vérifiez que les exporters nécessaires sont actifs.

   Exemple : Pour le Node Exporter Dashboard (1860), vous devez avoir node-exporter installé :
   ```bash
   microk8s kubectl get pods -n prometheus | grep node-exporter
   ```

2. **Labels ou selectors incorrects**

   **Solution :** Éditez le panneau et ajustez la requête PromQL pour correspondre à vos labels.

3. **Intervalle de temps inapproprié**

   **Solution :** Changez la plage de temps. Peut-être que les données n'existent que pour une période spécifique.

### Problème : Dashboard très lent

**Causes possibles :**

1. **Trop de données à afficher**

   **Solution :** Réduisez l'intervalle de temps ou augmentez l'intervalle de scrape.

2. **Requêtes PromQL non optimisées**

   **Solution :** Éditez les panneaux et optimisez les requêtes (ajout de filtres, agrégations).

3. **Trop de panneaux sur un seul dashboard**

   **Solution :** Divisez le dashboard en plusieurs dashboards plus petits.

### Problème : Erreurs "Template variables could not be initialized"

**Cause :** Les variables du dashboard ne peuvent pas être chargées.

**Solutions :**

1. **Vérifiez la source de données** est correctement configurée

2. **Éditez les variables du dashboard :**
   - Cliquez sur **Settings** (⚙️) → **Variables**
   - Pour chaque variable, vérifiez que la requête est valide

3. **Réinitialisez les variables :**
   - Supprimez et recréez les variables si nécessaire

### Problème : Panneaux avec des noms de métriques différents

**Cause :** Différentes versions de Prometheus/Kubernetes utilisent des noms de métriques différents.

**Solution :**

1. Identifiez les métriques disponibles dans votre Prometheus :
   - Allez dans Explore
   - Tapez le début du nom de la métrique
   - Grafana suggérera les métriques disponibles

2. Éditez le panneau et remplacez la métrique par celle qui existe dans votre environnement

**Exemple :**

Dashboard cherche : `container_memory_usage_bytes`
Votre Prometheus a : `container_memory_working_set_bytes`

→ Remplacez dans la requête PromQL

## Personnalisation basique des dashboards importés

### Modification d'un panneau

1. **Passez la souris** sur le titre du panneau
2. Cliquez sur le **menu déroulant** (▼)
3. Sélectionnez **Edit**

**Dans l'éditeur, vous pouvez :**

- **Modifier la requête PromQL**
- **Changer le type de visualisation** (graph, gauge, stat, table)
- **Ajuster les seuils** (thresholds) pour les alertes visuelles
- **Personnaliser les couleurs**
- **Modifier les axes** (min/max, unités)

4. **Sauvegardez** avec **Apply** puis sauvegardez le dashboard

### Ajout d'un nouveau panneau

1. En haut à droite du dashboard, cliquez sur **Add panel**
2. Sélectionnez **Add a new panel**
3. Configurez votre panneau :
   - Choisissez la source de données (Prometheus)
   - Écrivez votre requête PromQL
   - Sélectionnez le type de visualisation
4. **Apply** pour ajouter le panneau au dashboard

### Duplication d'un dashboard

Pour créer une version personnalisée sans modifier l'original :

1. Ouvrez le dashboard
2. Cliquez sur **Settings** (⚙️) en haut à droite
3. Sélectionnez **Save As...**
4. Donnez un nouveau nom : "Mon Dashboard Kubernetes Custom"
5. **Save**

Vous avez maintenant une copie que vous pouvez modifier librement.

### Suppression de panneaux inutiles

1. Passez la souris sur le panneau à supprimer
2. Cliquez sur le menu déroulant (▼)
3. Sélectionnez **Remove**
4. **Confirmez**
5. **Sauvegardez le dashboard**

## Mise à jour des dashboards

Les dashboards de la communauté sont régulièrement mis à jour avec des améliorations et corrections de bugs.

### Vérifier les mises à jour disponibles

1. Visitez la page du dashboard sur grafana.com
2. Comparez le numéro de version avec celui de votre dashboard
3. Les notes de version indiquent les changements

### Mettre à jour un dashboard

**Attention :** La mise à jour écrasera vos modifications personnelles.

**Méthode :**

1. **Si vous avez personnalisé le dashboard :**
   - Exportez d'abord votre version personnalisée
   - Ou dupliquez-le avant la mise à jour

2. **Supprimez l'ancien dashboard :**
   - Ouvrez le dashboard
   - **Settings** → **General** → **Delete dashboard**

3. **Ré-importez la nouvelle version** avec l'ID du dashboard

**Alternative (conserver les modifications) :**

1. Exportez votre dashboard actuel (JSON)
2. Téléchargez la nouvelle version (JSON)
3. Comparez et fusionnez manuellement les modifications
4. Ré-importez le JSON fusionné

## Export et partage de dashboards

### Exporter un dashboard

Pour sauvegarder ou partager votre dashboard :

1. Ouvrez le dashboard
2. Cliquez sur **Share** (icône de partage) en haut
3. Sélectionnez **Export**
4. **Save to file** : Télécharge le JSON
5. Ou **View JSON** : Copiez le code JSON

### Partager via snapshot

Pour partager temporairement un dashboard avec quelqu'un qui n'a pas accès à votre Grafana :

1. Cliquez sur **Share**
2. Onglet **Snapshot**
3. Définissez l'expiration (1 heure, 1 jour, 1 semaine, jamais)
4. **Publish to snapshot.raintank.io** (service public Grafana)
5. Copiez et partagez le lien généré

**Note :** Les snapshots sont publics, ne les utilisez pas pour des données sensibles.

## Dashboards spécifiques pour MicroK8s

### Dashboard optimisé pour single-node

Pour un cluster MicroK8s à nœud unique, certains panneaux ne sont pas pertinents (comparaisons multi-nœuds). Créez un dashboard simplifié :

**Panneaux recommandés :**

- État global du cluster
- Utilisation CPU/RAM du nœud
- Nombre de Pods par namespace
- Top 10 Pods par CPU
- Top 10 Pods par RAM
- Utilisation du stockage persistant
- Trafic réseau

### Dashboard pour environnement de développement

Focus sur les métriques utiles pour le développement :

- Logs récents
- Redémarrages de conteneurs
- Temps de déploiement
- État des builds CI/CD
- Métriques applicatives custom

## Bonnes pratiques

### Ne surchargez pas vos dashboards

**Règle d'or :** Un dashboard = un objectif

- Dashboard "Vue d'ensemble" : Métriques essentielles uniquement
- Dashboard "Deep Dive" : Analyses détaillées par composant
- Dashboard "Troubleshooting" : Métriques de diagnostic

### Utilisez des noms descriptifs

Renommez vos dashboards importés pour plus de clarté :

❌ Mauvais : "Dashboard 315"
✅ Bon : "K8s - Vue d'ensemble du cluster"

### Organisez avec des lignes (rows)

Les dashboards peuvent être organisés en lignes repliables :

1. Éditez le dashboard
2. **Add panel** → **Add a new row**
3. Nommez la ligne (exemple : "Métriques CPU")
4. Glissez-déposez les panneaux dans la ligne
5. La ligne peut être repliée pour gagner de l'espace

### Documentez vos dashboards

Ajoutez des panneaux texte pour expliquer :

1. **Add panel** → **Visualization** : Sélectionnez **Text**
2. Écrivez en Markdown :
   ```markdown
   # Vue d'ensemble du cluster

   Ce dashboard affiche les métriques essentielles du cluster Kubernetes.

   ## Alertes
   - Rouge : Action immédiate requise
   - Orange : Attention nécessaire
   - Vert : Tout va bien
   ```

### Définissez des seuils visuels

Utilisez des couleurs pour identifier rapidement les problèmes :

- **Vert** : Normal (< 70% d'utilisation)
- **Orange** : Attention (70-85%)
- **Rouge** : Critique (> 85%)

## Dashboards avancés et spécialisés

### Pour la sécurité

- **Falco Dashboard** : Détection d'anomalies de sécurité
- **Kubernetes Security** : Audit des configurations de sécurité

### Pour le réseau

- **Kubernetes Network Policies** : Visualisation des politiques réseau
- **Ingress Controller** : Métriques NGINX/Traefik

### Pour les applications

- **Application Performance Monitoring (APM)** : Si vous instrumentez vos apps
- **JVM Metrics** : Pour applications Java
- **Go Metrics** : Pour applications Go

### Pour le stockage

- **Ceph Dashboard** : Si vous utilisez Ceph
- **NFS Monitoring** : Pour stockage NFS

## Collection de dashboards recommandée pour débuter

Pour un lab MicroK8s, commencez avec ces 5 dashboards essentiels :

1. **Kubernetes Cluster Monitoring (315)**
   - Vue d'ensemble générale

2. **Node Exporter Full (1860)**
   - Métriques système détaillées

3. **Kubernetes Pod Resources (6781)**
   - Surveillance des Pods

4. **Kubernetes Deployments (8588)**
   - Suivi des déploiements

5. **Kubernetes Persistent Volumes (13646)**
   - Gestion du stockage

Cette collection couvre 90% des besoins de monitoring d'un cluster Kubernetes.

## Prochaines étapes

Maintenant que vous maîtrisez l'import et l'utilisation de dashboards pré-configurés, vous pouvez :

- **Chapitre 13.4 :** Créer vos propres dashboards personnalisés
- **Chapitre 13.5 :** Maîtriser les panneaux et types de visualisations
- **Chapitre 13.6 :** Utiliser les variables et le templating
- **Chapitre 14 :** Configurer des alertes sur vos dashboards

## Récapitulatif

Dans ce chapitre, vous avez appris à :

✅ Comprendre l'intérêt des dashboards pré-configurés
✅ Trouver des dashboards sur grafana.com
✅ Identifier les dashboards Kubernetes les plus utiles
✅ Importer des dashboards via ID, JSON ou URL
✅ Configurer et ajuster les dashboards importés
✅ Organiser vos dashboards avec des dossiers
✅ Résoudre les problèmes courants (no data, lenteur)
✅ Personnaliser basiquement les dashboards
✅ Exporter et partager vos dashboards
✅ Appliquer les bonnes pratiques d'organisation

Vous disposez maintenant d'un ensemble complet de dashboards professionnels pour surveiller efficacement votre cluster Kubernetes !

---

**Astuce finale :** Prenez le temps d'explorer chaque dashboard importé. Cliquez sur les panneaux, regardez les requêtes PromQL utilisées, comprenez ce qui est mesuré. C'est la meilleure façon d'apprendre et de devenir autonome dans la création de vos propres dashboards.

⏭️ [Création de dashboards personnalisés](/13-visualisation-avec-grafana/04-creation-de-dashboards-personnalises.md)
