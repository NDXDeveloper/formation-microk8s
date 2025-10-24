🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 13.2 Connexion Grafana-Prometheus

## Introduction

La connexion entre Grafana et Prometheus est le cœur de votre système de monitoring. Prometheus collecte les métriques, les stocke et les rend disponibles via une API. Grafana, de son côté, interroge cette API pour récupérer les données et les afficher sous forme de graphiques et tableaux de bord.

### Comprendre la relation Grafana-Prometheus

Imaginez cette analogie simple :

- **Prometheus** = Une bibliothèque qui stocke des livres (les métriques)
- **Grafana** = Un lecteur qui vient chercher des livres spécifiques pour vous les présenter de manière agréable

Grafana ne stocke pas de métriques, il se contente de les afficher. C'est Prometheus qui fait le travail de collecte et de stockage.

### Prérequis

Avant de configurer la connexion, assurez-vous que :

1. **Grafana est installé et accessible** (voir chapitre 13.1)
2. **Prometheus est installé et opérationnel**
3. **Les deux services sont dans le même cluster Kubernetes** (cas standard)

## Architecture de la connexion

### Communication dans Kubernetes

Dans un cluster Kubernetes, les services communiquent entre eux via le réseau interne du cluster. Voici comment cela fonctionne :

```
┌─────────────────────────────────────────────────────┐
│                  Cluster Kubernetes                 │
│                                                     │
│  ┌──────────────┐              ┌─────────────────┐  │
│  │   Grafana    │──── HTTP ───→│   Prometheus    │  │
│  │   (Pod)      │   Requêtes   │     (Pod)       │  │
│  │   Port 3000  │   PromQL     │   Port 9090     │  │
│  └──────────────┘              └─────────────────┘  │
│         ↑                              ↑            │
│         │                              │            │
│  ┌──────────────┐              ┌─────────────────┐  │
│  │   Service    │              │    Service      │  │
│  │   grafana    │              │ prometheus-k8s  │  │
│  └──────────────┘              └─────────────────┘  │
│                                                     │
└─────────────────────────────────────────────────────┘
```

**Explications pour débutants :**

- Les Pods sont les conteneurs qui exécutent Grafana et Prometheus
- Les Services sont des "adresses" stables qui permettent de joindre ces Pods
- La communication se fait via HTTP sur le réseau interne de Kubernetes
- Grafana envoie des requêtes en langage PromQL pour récupérer les métriques

## Configuration de la source de données Prometheus

### Accès à la configuration

1. **Connectez-vous à Grafana** (par défaut : `http://localhost:3000`)
2. Dans le menu latéral, cliquez sur l'icône **engrenage** (⚙️)
3. Sélectionnez **Data Sources**

Si vous avez déjà configuré Prometheus dans le chapitre 13.1, vous devriez le voir dans la liste. Sinon, cliquez sur **Add data source**.

### Paramètres essentiels de connexion

#### 1. Informations générales

**Name (Nom)**
```
Prometheus
```

Le nom que vous donnez à cette source de données. Vous pouvez le personnaliser, mais "Prometheus" est un choix logique et clair.

**Type**
```
Prometheus
```

Le type de source de données. Sélectionnez "Prometheus" dans la liste.

#### 2. Configuration HTTP

**URL**

C'est le paramètre le plus important. L'URL indique à Grafana où trouver Prometheus.

**Format standard pour MicroK8s :**
```
http://prometheus-k8s.prometheus.svc.cluster.local:9090
```

**Décomposition de cette URL :**

- `http://` : Protocole de communication (non sécurisé, suffisant pour un réseau interne)
- `prometheus-k8s` : Nom du service Prometheus
- `prometheus` : Namespace où Prometheus est déployé
- `svc.cluster.local` : Suffixe DNS interne de Kubernetes
- `9090` : Port par défaut de Prometheus

**Comment trouver le nom correct du service ?**

```bash
microk8s kubectl get svc -n prometheus
```

Résultat exemple :
```
NAME                    TYPE        CLUSTER-IP       PORT(S)    AGE
prometheus-k8s          ClusterIP   10.152.183.100   9090/TCP   1h
```

Le nom à utiliser est dans la colonne `NAME`, ici `prometheus-k8s`.

**Variantes possibles selon votre installation :**

Si Prometheus est dans un autre namespace :
```
http://prometheus-k8s.<votre-namespace>.svc.cluster.local:9090
```

Si le service a un nom différent (par exemple `prometheus-service`) :
```
http://prometheus-service.prometheus.svc.cluster.local:9090
```

**Access**

Sélectionnez : **Server (default)**

**Explication :**
- **Server** : Grafana contacte Prometheus depuis le serveur (recommandé dans un cluster)
- **Browser** : Votre navigateur contacte directement Prometheus (rarement utilisé)

Dans un cluster Kubernetes, utilisez toujours "Server" car Grafana et Prometheus communiquent via le réseau interne du cluster.

#### 3. Configuration Auth (Authentification)

Dans un environnement de lab avec MicroK8s, Prometheus n'a généralement pas d'authentification activée.

**Paramètres recommandés :**

- **Basic Auth** : Désactivé
- **TLS Client Auth** : Désactivé
- **Skip TLS Verify** : Désactivé (ou activé si vous utilisez HTTPS avec des certificats auto-signés)

Si votre Prometheus nécessite une authentification :

1. Activez **Basic Auth**
2. Entrez les identifiants :
   - **User** : nom d'utilisateur Prometheus
   - **Password** : mot de passe Prometheus

#### 4. Paramètres Prometheus spécifiques

**HTTP Method**

Choisissez : **POST**

**Explication :**
- **GET** : Pour des requêtes simples
- **POST** : Recommandé car il permet des requêtes plus complexes et longues

**Scrape interval**

Valeur recommandée : `15s`

C'est l'intervalle auquel Prometheus collecte les métriques. Grafana utilise cette valeur pour optimiser l'affichage des graphiques.

**Query timeout**

Valeur recommandée : `60s`

Le temps maximum qu'une requête peut prendre avant d'être annulée. Augmentez cette valeur si vous avez des requêtes complexes qui prennent du temps.

**Default editor**

Choisissez : **Builder** (pour les débutants) ou **Code** (pour les utilisateurs avancés)

- **Builder** : Interface graphique pour construire des requêtes
- **Code** : Écriture directe en PromQL

### Test de la connexion

Une fois tous les paramètres configurés :

1. Cliquez sur **Save & Test** en bas de la page

2. Résultats possibles :

   **✅ Succès :**
   ```
   Data source is working
   ```
   Félicitations ! La connexion fonctionne parfaitement.

   **❌ Échec :**
   ```
   HTTP Error Bad Gateway
   ```
   ou
   ```
   Connection refused
   ```
   La connexion a échoué. Voir la section dépannage ci-dessous.

## Vérification de la connexion

### Méthode 1 : Interface Explore

L'interface Explore de Grafana permet de tester des requêtes Prometheus en temps réel.

1. Dans le menu latéral, cliquez sur l'icône **Explore** (boussole)

2. En haut, sélectionnez **Prometheus** comme source de données

3. Dans le champ de requête, tapez une requête simple :
   ```promql
   up
   ```

4. Cliquez sur **Run Query** (ou appuyez sur Shift + Entrée)

**Résultat attendu :**

Vous devriez voir un graphique montrant l'état de tous les targets Prometheus :
- Valeur `1` = Target actif (up)
- Valeur `0` = Target inactif (down)

### Méthode 2 : Requêtes de test additionnelles

Testez d'autres requêtes pour vérifier que les métriques sont bien collectées :

**Nombre de Pods en cours d'exécution :**
```promql
kube_pod_status_phase{phase="Running"}
```

**Utilisation CPU du cluster :**
```promql
sum(rate(container_cpu_usage_seconds_total[5m]))
```

**Utilisation mémoire :**
```promql
sum(container_memory_usage_bytes) / 1024 / 1024 / 1024
```

Si ces requêtes retournent des données, votre connexion fonctionne parfaitement !

### Méthode 3 : Via les logs Grafana

Vérifiez les logs du pod Grafana pour voir s'il y a des erreurs de connexion :

```bash
microk8s kubectl logs -n prometheus deployment/grafana
```

Recherchez des lignes comme :
```
logger=context error="failed to query" ...
```

L'absence d'erreurs est un bon signe.

## Configuration avancée de la connexion

### Custom HTTP Headers

Dans certains cas, vous devrez ajouter des en-têtes HTTP personnalisés pour l'authentification ou le routage.

**Exemple : Ajout d'un header personnalisé**

Dans la section **Custom HTTP Headers** :

1. Cliquez sur **Add header**
2. **Header** : `X-Custom-Auth`
3. **Value** : `votre-token-secret`

### Configuration de proxy HTTP

Si Grafana doit passer par un proxy pour atteindre Prometheus :

1. Dans la configuration de la source de données, section **HTTP**
2. **HTTP Proxy** : `http://proxy.example.com:8080`

### Exemplars (Échantillons de traces)

Les exemplars permettent de lier les métriques avec des traces distribuées.

**Activation :**
1. Activez **Enable Exemplars**
2. Configurez l'URL de votre système de tracing (Jaeger, Tempo)

**Pour débutants :** Cette fonctionnalité est avancée, vous pouvez l'ignorer pour l'instant.

### Cache de requêtes

Grafana peut mettre en cache les résultats de requêtes pour améliorer les performances.

**Paramètres de cache :**
- **Cache Timeout** : Durée de vie du cache (exemple : `300` secondes = 5 minutes)

## Connexions multiples Prometheus

### Pourquoi plusieurs sources Prometheus ?

Vous pourriez avoir plusieurs instances Prometheus :

- Un Prometheus pour le cluster Kubernetes
- Un autre pour les applications métier
- Un troisième pour l'infrastructure externe

### Configuration de multiples data sources

1. Répétez le processus de configuration pour chaque Prometheus
2. Donnez des noms distincts :
   - `Prometheus-K8s`
   - `Prometheus-Apps`
   - `Prometheus-Infra`

3. Dans vos dashboards, vous pourrez sélectionner la source appropriée

### Data source par défaut

Pour définir un Prometheus comme source par défaut :

1. Allez dans la configuration de la source de données
2. Cochez **Default**

Les nouveaux dashboards utiliseront automatiquement cette source.

## Dépannage de la connexion

### Erreur : "HTTP Error Bad Gateway"

**Cause :** Grafana ne peut pas joindre Prometheus.

**Solutions :**

1. **Vérifiez que Prometheus est en cours d'exécution :**
   ```bash
   microk8s kubectl get pods -n prometheus
   ```
   Le pod Prometheus doit avoir le statut `Running`.

2. **Vérifiez le nom du service :**
   ```bash
   microk8s kubectl get svc -n prometheus
   ```
   Assurez-vous que le nom et le port correspondent à votre configuration.

3. **Testez la connectivité réseau :**
   ```bash
   microk8s kubectl exec -n prometheus deployment/grafana -- curl http://prometheus-k8s.prometheus.svc.cluster.local:9090/api/v1/query?query=up
   ```

   Vous devriez obtenir une réponse JSON avec des métriques.

### Erreur : "Connection refused"

**Cause :** Le port ou l'adresse est incorrect.

**Solutions :**

1. Vérifiez le port du service Prometheus :
   ```bash
   microk8s kubectl get svc prometheus-k8s -n prometheus -o yaml
   ```

   Regardez la section `ports` :
   ```yaml
   ports:
   - name: http
     port: 9090
     protocol: TCP
     targetPort: 9090
   ```

2. Assurez-vous d'utiliser le bon port dans l'URL Grafana

### Erreur : "No such host"

**Cause :** Le nom DNS du service est incorrect.

**Solutions :**

1. Vérifiez le namespace de Prometheus :
   ```bash
   microk8s kubectl get pods --all-namespaces | grep prometheus
   ```

2. Ajustez l'URL en conséquence :
   ```
   http://prometheus-k8s.<namespace-correct>.svc.cluster.local:9090
   ```

### Erreur : "Query timeout"

**Cause :** Les requêtes prennent trop de temps.

**Solutions :**

1. Augmentez le **Query timeout** dans la configuration (par exemple, à `120s`)

2. Optimisez vos requêtes PromQL pour qu'elles soient moins gourmandes

3. Vérifiez les performances de Prometheus :
   ```bash
   microk8s kubectl top pod -n prometheus
   ```

### Problème : Données manquantes ou incomplètes

**Causes possibles :**

1. **Prometheus ne collecte pas toutes les métriques**

   Vérifiez les targets Prometheus :
   - Accédez à Prometheus : `kubectl port-forward -n prometheus svc/prometheus-k8s 9090:9090`
   - Ouvrez `http://localhost:9090/targets`
   - Vérifiez que tous les targets sont "UP"

2. **Problème de rétention des données**

   Par défaut, Prometheus conserve 15 jours de données. Les données plus anciennes sont supprimées.

3. **Intervalle de scrape incorrect**

   Vérifiez la configuration Prometheus pour l'intervalle de collecte.

### Logs de débogage

Pour activer plus de logs dans Grafana :

1. Modifiez le Deployment Grafana pour ajouter :
   ```yaml
   env:
   - name: GF_LOG_LEVEL
     value: "debug"
   ```

2. Redémarrez Grafana :
   ```bash
   microk8s kubectl rollout restart deployment/grafana -n prometheus
   ```

3. Consultez les logs détaillés :
   ```bash
   microk8s kubectl logs -n prometheus deployment/grafana --tail=100 -f
   ```

## Validation de la connexion

### Checklist de validation

Utilisez cette checklist pour vous assurer que tout fonctionne correctement :

- [ ] Le pod Prometheus est en statut `Running`
- [ ] Le service Prometheus est accessible dans le cluster
- [ ] La source de données Prometheus est configurée dans Grafana
- [ ] Le test de connexion réussit ("Data source is working")
- [ ] L'interface Explore retourne des résultats pour la requête `up`
- [ ] Les métriques Kubernetes sont visibles (pods, nodes, etc.)
- [ ] Aucune erreur dans les logs Grafana

### Test de performance

Pour tester la réactivité de la connexion :

1. Dans Explore, exécutez une requête complexe :
   ```promql
   sum(rate(container_cpu_usage_seconds_total{namespace="kube-system"}[5m])) by (pod)
   ```

2. Chronométrez le temps de réponse

3. Un temps de réponse < 2 secondes est excellent

## Bonnes pratiques de connexion

### Sécurité

1. **Utilisez HTTPS en production**

   Pour un lab, HTTP est suffisant, mais en production, sécurisez avec HTTPS :
   ```
   https://prometheus-k8s.prometheus.svc.cluster.local:9090
   ```

2. **Activez l'authentification**

   Configurez une authentification sur Prometheus pour éviter les accès non autorisés.

3. **Limitez les permissions**

   Utilisez RBAC Kubernetes pour restreindre l'accès aux métriques sensibles.

### Performance

1. **Ajustez le scrape interval**

   Un intervalle plus court (10s) = plus de précision mais plus de charge
   Un intervalle plus long (30s) = moins de charge mais moins de précision

2. **Utilisez le cache de Grafana**

   Configurez un cache pour éviter de surcharger Prometheus avec des requêtes identiques.

3. **Optimisez les requêtes PromQL**

   Évitez les requêtes sans filtre qui retournent trop de séries temporelles.

### Monitoring de la connexion

1. **Créez une alerte dans Grafana**

   Pour être notifié si la connexion Prometheus tombe.

2. **Surveillez les métriques Grafana**

   Grafana expose ses propres métriques sur `/metrics` :
   ```bash
   kubectl port-forward -n prometheus svc/grafana 3000:3000
   curl http://localhost:3000/metrics
   ```

3. **Vérifiez régulièrement les logs**

   Automatisez la surveillance des logs pour détecter les erreurs.

## Alternatives et extensions

### Prometheus Operator

Pour une configuration plus avancée, envisagez d'utiliser Prometheus Operator qui simplifie la gestion de Prometheus dans Kubernetes.

### Thanos

Thanos permet d'avoir une vue unifiée de plusieurs clusters Prometheus et offre une rétention de données illimitée.

### Cortex

Cortex est une alternative qui permet un stockage distribué et scalable des métriques Prometheus.

### Mimir

Grafana Mimir est la solution de Grafana Labs pour un Prometheus hautement scalable.

**Note pour débutants :** Ces solutions sont avancées. Concentrez-vous d'abord sur la connexion de base Grafana-Prometheus.

## Prochaines étapes

Maintenant que la connexion Grafana-Prometheus est établie et fonctionnelle, vous pouvez passer à :

- **Chapitre 13.3 :** Importer des dashboards Kubernetes pré-configurés
- **Chapitre 13.4 :** Créer vos propres dashboards personnalisés
- **Chapitre 13.5 :** Maîtriser les panneaux et visualisations

## Récapitulatif

Dans ce chapitre, vous avez appris à :

✅ Comprendre l'architecture de la connexion Grafana-Prometheus
✅ Configurer la source de données Prometheus dans Grafana
✅ Paramétrer correctement l'URL et les options de connexion
✅ Tester et valider la connexion
✅ Diagnostiquer et résoudre les problèmes de connexion
✅ Appliquer les bonnes pratiques de sécurité et performance
✅ Utiliser l'interface Explore pour tester des requêtes

Votre système de monitoring est maintenant pleinement opérationnel avec Grafana capable d'interroger et de visualiser toutes les métriques collectées par Prometheus !

---

**Conseil pour les débutants :** Si vous rencontrez des difficultés, commencez toujours par vérifier les trois éléments de base : (1) Prometheus est-il en cours d'exécution ? (2) Le nom du service est-il correct ? (3) Le port est-il le bon ? 90% des problèmes viennent de ces trois points.

⏭️ [Dashboards Kubernetes pré-configurés](/13-visualisation-avec-grafana/03-dashboards-kubernetes-pre-configures.md)
