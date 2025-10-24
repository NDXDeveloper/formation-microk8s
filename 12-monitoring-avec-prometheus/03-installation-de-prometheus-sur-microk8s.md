🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 12.3 Installation de Prometheus (microk8s enable prometheus)

## Introduction

L'un des grands avantages de MicroK8s est la simplicité d'installation de Prometheus. Là où d'autres distributions Kubernetes nécessitent des configurations complexes avec des fichiers YAML de plusieurs centaines de lignes, MicroK8s vous permet d'installer un stack de monitoring complet avec une seule commande.

Dans cette section, nous allons installer Prometheus et vérifier que tout fonctionne correctement.

## Prérequis

Avant d'installer Prometheus, assurez-vous que :

### 1. MicroK8s est installé et fonctionne

Vérifiez l'état de votre cluster :

```bash
microk8s status
```

Vous devriez voir :

```
microk8s is running
high-availability: no
  datastore master nodes: 127.0.0.1:19001
  datastore standby nodes: none
```

Si MicroK8s n'est pas démarré :

```bash
microk8s start
```

### 2. Ressources système suffisantes

Prometheus et son écosystème consomment des ressources. Voici les recommandations minimales :

- **CPU** : 2 cores minimum (4 cores recommandés)
- **RAM** : 4 Go minimum (8 Go recommandés)
- **Disque** : 10 Go d'espace libre minimum (20 Go recommandés)

Pour vérifier vos ressources disponibles :

```bash
# Sur Linux
free -h          # Mémoire
df -h            # Espace disque
nproc            # Nombre de CPUs
```

### 3. DNS activé

Prometheus nécessite CoreDNS pour la résolution de noms dans le cluster.

Vérifiez si DNS est activé :

```bash
microk8s status | grep dns
```

Si DNS n'est pas activé, activez-le :

```bash
microk8s enable dns
```

Attendez quelques secondes que CoreDNS soit prêt :

```bash
microk8s kubectl get pods -n kube-system -l k8s-app=kube-dns
```

Vous devriez voir :

```
NAME                       READY   STATUS    RESTARTS   AGE
coredns-64c6478b6c-xxxxx   1/1     Running   0          1m
```

## Installation de Prometheus

### La commande magique

C'est le moment que vous attendiez ! Pour installer Prometheus, exécutez simplement :

```bash
microk8s enable prometheus
```

**C'est tout !** Cette simple commande va :

1. Créer un namespace `monitoring`
2. Déployer Prometheus Operator
3. Déployer le serveur Prometheus
4. Déployer Alertmanager
5. Déployer les exporters (node-exporter, kube-state-metrics)
6. Configurer le service discovery automatique
7. Créer les services pour accéder aux interfaces web
8. Appliquer des règles d'alerte par défaut

### Processus d'installation

Lors de l'exécution, vous verrez une sortie similaire à :

```
Infer repository core for addon prometheus
Enabling Prometheus
namespace/monitoring created
customresourcedefinition.apiextensions.k8s.io/alertmanagerconfigs.monitoring.coreos.com created
customresourcedefinition.apiextensions.k8s.io/alertmanagers.monitoring.coreos.com created
customresourcedefinition.apiextensions.k8s.io/podmonitors.monitoring.coreos.com created
customresourcedefinition.apiextensions.k8s.io/probes.monitoring.coreos.com created
customresourcedefinition.apiextensions.k8s.io/prometheuses.monitoring.coreos.com created
customresourcedefinition.apiextensions.k8s.io/prometheusrules.monitoring.coreos.com created
customresourcedefinition.apiextensions.k8s.io/servicemonitors.monitoring.coreos.com created
customresourcedefinition.apiextensions.k8s.io/thanosrulers.monitoring.coreos.com created
clusterrole.rbac.authorization.k8s.io/prometheus-operator created
clusterrolebinding.rbac.authorization.k8s.io/prometheus-operator created
deployment.apps/prometheus-operator created
service/prometheus-operator created
...
Prometheus is enabled
```

**Patience** : L'installation peut prendre de 2 à 5 minutes, car MicroK8s doit télécharger plusieurs images Docker.

## Vérification de l'installation

### Étape 1 : Vérifier que l'addon est activé

```bash
microk8s status
```

Vous devriez voir `prometheus` dans la liste des addons activés :

```
...
enabled:
    dns
    prometheus        # ← Prometheus est actif
...
```

### Étape 2 : Vérifier le namespace monitoring

Tous les composants Prometheus sont déployés dans le namespace `monitoring` :

```bash
microk8s kubectl get namespaces
```

Vous devriez voir :

```
NAME              STATUS   AGE
default           Active   10d
kube-system       Active   10d
kube-public       Active   10d
kube-node-lease   Active   10d
monitoring        Active   2m    # ← Nouveau namespace
```

### Étape 3 : Vérifier les pods

Vérifions que tous les pods Prometheus sont en cours d'exécution :

```bash
microk8s kubectl get pods -n monitoring
```

Résultat attendu (peut varier légèrement selon la version) :

```
NAME                                   READY   STATUS    RESTARTS   AGE
prometheus-operator-6d9b4b9b8c-xxxxx   1/1     Running   0          3m
prometheus-k8s-0                       2/2     Running   0          2m
alertmanager-main-0                    2/2     Running   0          2m
node-exporter-xxxxx                    1/1     Running   0          3m
kube-state-metrics-xxxxxxxxxx-xxxxx    1/1     Running   0          3m
```

**Détails des pods** :

- **prometheus-operator** : Gère la configuration de Prometheus
- **prometheus-k8s-0** : Le serveur Prometheus principal (StatefulSet avec 1 réplica par défaut)
- **alertmanager-main-0** : Gestionnaire d'alertes
- **node-exporter** : DaemonSet qui s'exécute sur chaque node (1 pod par node)
- **kube-state-metrics** : Expose l'état des objets Kubernetes

**États possibles** :

- `Running` : ✅ Tout va bien
- `ContainerCreating` : ⏳ En cours de démarrage, attendez
- `Pending` : ⏳ Attend d'être planifié, attendez
- `ImagePullBackOff` : ❌ Problème de téléchargement d'image (voir section dépannage)
- `CrashLoopBackOff` : ❌ Le pod redémarre continuellement (voir section dépannage)

**Si tous les pods ne sont pas prêts immédiatement**, c'est normal ! Attendez quelques minutes. Vous pouvez surveiller l'évolution avec :

```bash
microk8s kubectl get pods -n monitoring -w
```

(Appuyez sur `Ctrl+C` pour arrêter la surveillance)

### Étape 4 : Vérifier les services

Les services exposent les interfaces web de Prometheus et Alertmanager :

```bash
microk8s kubectl get services -n monitoring
```

Vous devriez voir :

```
NAME                    TYPE        CLUSTER-IP       PORT(S)             AGE
prometheus-operator     ClusterIP   10.152.183.xxx   8080/TCP            5m
prometheus-k8s          ClusterIP   10.152.183.xxx   9090/TCP,8080/TCP   5m
alertmanager-main       ClusterIP   10.152.183.xxx   9093/TCP,8080/TCP   5m
node-exporter           ClusterIP   None             9100/TCP            5m
kube-state-metrics      ClusterIP   None             8080/TCP,8081/TCP   5m
```

**Services clés** :

- **prometheus-k8s** (port 9090) : Interface web Prometheus
- **alertmanager-main** (port 9093) : Interface web Alertmanager
- **node-exporter** (port 9100) : Métriques du node
- **kube-state-metrics** (port 8080) : Métriques Kubernetes

### Étape 5 : Vérifier les ServiceMonitors

Prometheus utilise des objets `ServiceMonitor` pour découvrir automatiquement quoi surveiller :

```bash
microk8s kubectl get servicemonitors -n monitoring
```

Vous devriez voir :

```
NAME                      AGE
prometheus-operator       5m
prometheus-k8s            5m
alertmanager-main         5m
node-exporter             5m
kube-state-metrics        5m
kubelet                   5m
```

Ces ServiceMonitors indiquent à Prometheus de surveiller automatiquement tous ces composants.

## Accès à l'interface web Prometheus

### Méthode 1 : Port-forward (Recommandé pour débuter)

Le **port-forward** redirige un port de votre machine locale vers un service dans le cluster. C'est la méthode la plus simple pour accéder à Prometheus :

```bash
microk8s kubectl port-forward -n monitoring service/prometheus-k8s 9090:9090
```

Vous verrez :

```
Forwarding from 127.0.0.1:9090 -> 9090
Forwarding from [::1]:9090 -> 9090
```

**Important** : Cette commande doit rester active. Ouvrez un **nouveau terminal** pour continuer à travailler.

Maintenant, ouvrez votre navigateur web et accédez à :

```
http://localhost:9090
```

🎉 **Vous devriez voir l'interface web de Prometheus !**

Pour arrêter le port-forward, revenez au terminal et appuyez sur `Ctrl+C`.

### Méthode 2 : NodePort (Accès depuis le réseau local)

Si vous souhaitez accéder à Prometheus depuis d'autres machines de votre réseau local, modifiez le service pour utiliser un NodePort :

```bash
microk8s kubectl patch service prometheus-k8s -n monitoring -p '{"spec":{"type":"NodePort"}}'
```

Récupérez le port assigné :

```bash
microk8s kubectl get service prometheus-k8s -n monitoring
```

Vous verrez quelque chose comme :

```
NAME             TYPE       CLUSTER-IP       PORT(S)          AGE
prometheus-k8s   NodePort   10.152.183.xxx   9090:30090/TCP   10m
```

Le port `30090` (peut varier) est maintenant accessible depuis votre réseau local.

Accédez à Prometheus via :

```
http://<IP_DE_VOTRE_MACHINE>:30090
```

Pour trouver l'IP de votre machine :

```bash
# Sur Linux
hostname -I | awk '{print $1}'

# Ou
ip addr show | grep "inet " | grep -v 127.0.0.1
```

### Méthode 3 : Ingress (Pour une exposition plus professionnelle)

Cette méthode sera couverte dans les sections avancées (chapitre 10 sur les Ingress). Elle permet d'accéder à Prometheus via un nom de domaine avec HTTPS.

## Première visite de l'interface Prometheus

### Page d'accueil

Lorsque vous accédez à `http://localhost:9090`, vous arrivez sur la page d'accueil de Prometheus avec plusieurs onglets :

#### 1. **Graph** (Graphiques)

C'est l'onglet principal où vous pouvez exécuter des requêtes PromQL et visualiser les résultats.

**Essayez votre première requête** :

Dans le champ de recherche, tapez :

```promql
up
```

Cliquez sur **Execute** (ou appuyez sur `Shift+Enter`).

**Que signifie cette requête ?**

`up` est une métrique spéciale créée par Prometheus qui indique si une target est accessible :
- `1` = Target accessible (up)
- `0` = Target inaccessible (down)

Vous verrez une liste de toutes les targets surveillées avec leur état.

**Passez en mode Graph** :

Cliquez sur l'onglet **Graph** (à côté de Table) pour voir une visualisation temporelle.

#### 2. **Alerts** (Alertes)

Cet onglet montre toutes les règles d'alerte configurées et leur état actuel :

- **Inactive** : Condition non remplie, tout va bien
- **Pending** : Condition remplie, mais pas encore assez longtemps
- **Firing** : Alerte active

Par défaut, MicroK8s configure plusieurs alertes de base.

#### 3. **Status** (État)

Cet onglet contient plusieurs sous-menus utiles :

**Status > Targets** :

Affiche toutes les targets que Prometheus surveille avec leur état :
- **UP** (vert) : Target accessible
- **DOWN** (rouge) : Target inaccessible
- **UNKNOWN** : État inconnu

Vous devriez voir plusieurs groupes :
- `monitoring/prometheus-k8s/0` (Prometheus lui-même)
- `monitoring/alertmanager-main/0` (Alertmanager)
- `monitoring/node-exporter/0` (Node exporter)
- `monitoring/kube-state-metrics/0` (Kube state metrics)

**Status > Service Discovery** :

Montre tous les services découverts automatiquement par Prometheus dans Kubernetes. C'est fascinant de voir comment Prometheus trouve automatiquement tout ce qu'il doit surveiller !

**Status > Configuration** :

Affiche la configuration complète de Prometheus en YAML. Très utile pour le débogage.

**Status > Rules** :

Liste toutes les règles d'alerte et de recording configurées.

**Status > Command-Line Flags** :

Montre tous les paramètres avec lesquels Prometheus a été démarré.

### Comprendre ce que vous voyez

Dans **Status > Targets**, vous devriez voir toutes vos targets en état **UP** (vert). Si tout est vert, **félicitations** ! Votre installation Prometheus fonctionne parfaitement.

**Exemple de targets** :

```
monitoring/alertmanager-main/0 (1/1 up)
├── http://alertmanager-main-0.alertmanager-operated.monitoring.svc:9093/metrics

monitoring/kube-state-metrics/0 (1/1 up)
├── http://10.1.x.x:8080/metrics

monitoring/node-exporter/0 (1/1 up)
├── http://10.1.x.x:9100/metrics

monitoring/prometheus-k8s/0 (1/1 up)
├── http://prometheus-k8s-0.prometheus-operated.monitoring.svc:9090/metrics
```

Chaque target expose des métriques sur un endpoint `/metrics`, et Prometheus les scrape régulièrement (par défaut toutes les 30 secondes).

## Accès à Alertmanager

De la même manière que pour Prometheus, vous pouvez accéder à l'interface Alertmanager :

### Port-forward vers Alertmanager

```bash
microk8s kubectl port-forward -n monitoring service/alertmanager-main 9093:9093
```

Accédez à :

```
http://localhost:9093
```

**Interface Alertmanager** :

Sur la page d'accueil, vous verrez :
- **Alerts** : Liste des alertes actives (vide si aucune alerte)
- **Silences** : Alertes temporairement désactivées
- **Status** : État général du cluster Alertmanager

Pour le moment, il ne devrait y avoir aucune alerte active si votre cluster fonctionne normalement.

## Vérification du scraping des métriques

Pour vérifier que Prometheus collecte bien les métriques, exécutez quelques requêtes simples :

### Requête 1 : Vérifier la disponibilité des targets

```promql
up
```

Toutes les valeurs devraient être à `1`.

### Requête 2 : Utilisation CPU des nodes

```promql
node_cpu_seconds_total
```

Vous devriez voir de nombreuses séries temporelles avec les statistiques CPU de votre(vos) node(s).

### Requête 3 : Nombre de pods en cours d'exécution

```promql
kube_pod_status_phase{phase="Running"}
```

Cette requête montre tous les pods en cours d'exécution dans votre cluster.

### Requête 4 : Utilisation mémoire des conteneurs

```promql
container_memory_usage_bytes
```

Affiche l'utilisation mémoire de tous les conteneurs.

**Si ces requêtes retournent des résultats**, c'est que Prometheus collecte bien les métriques ! 🎉

## Composants déployés : Vue détaillée

Récapitulons ce qui a été installé :

### Custom Resource Definitions (CRDs)

Prometheus Operator ajoute de nouveaux types d'objets Kubernetes :

```bash
microk8s kubectl get crds | grep monitoring.coreos.com
```

```
alertmanagerconfigs.monitoring.coreos.com
alertmanagers.monitoring.coreos.com
podmonitors.monitoring.coreos.com
probes.monitoring.coreos.com
prometheuses.monitoring.coreos.com
prometheusrules.monitoring.coreos.com
servicemonitors.monitoring.coreos.com
thanosrulers.monitoring.coreos.com
```

Ces CRDs permettent de configurer Prometheus de manière déclarative, "à la Kubernetes".

### StatefulSets

```bash
microk8s kubectl get statefulsets -n monitoring
```

```
NAME                READY   AGE
prometheus-k8s      1/1     10m
alertmanager-main   1/1     10m
```

**Pourquoi StatefulSet ?**

Les StatefulSets sont utilisés pour les applications stateful (avec état), comme les bases de données. Prometheus a besoin de conserver ses données de métriques, donc il utilise un StatefulSet avec un volume persistant.

### DaemonSets

```bash
microk8s kubectl get daemonsets -n monitoring
```

```
NAME            DESIRED   CURRENT   READY   AGE
node-exporter   1         1         1       10m
```

**Pourquoi DaemonSet ?**

Node-exporter doit s'exécuter sur **chaque node** du cluster pour collecter les métriques système de chaque machine. Le DaemonSet garantit qu'il y a exactement un pod node-exporter par node.

### ConfigMaps

```bash
microk8s kubectl get configmaps -n monitoring
```

Les ConfigMaps contiennent :
- Les règles d'alerte Prometheus
- La configuration d'Alertmanager
- Les dashboards Grafana (si Grafana est installé)

### PersistentVolumeClaims (PVC)

```bash
microk8s kubectl get pvc -n monitoring
```

```
NAME                                 STATUS   VOLUME                                     CAPACITY
prometheus-k8s-db-prometheus-k8s-0   Bound    pvc-xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx   10Gi
alertmanager-main-db-alertmanager... Bound    pvc-xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx   2Gi
```

Ces volumes persistent les données de Prometheus et Alertmanager, même si les pods sont redémarrés.

## Configuration par défaut

### Rétention des données

Par défaut, Prometheus conserve les données pendant **15 jours** et utilise un maximum de **10 Go** d'espace disque.

Vous pouvez voir cette configuration en inspectant le StatefulSet :

```bash
microk8s kubectl get statefulset prometheus-k8s -n monitoring -o yaml | grep -A5 retention
```

### Intervalle de scraping

Par défaut, Prometheus scrape les targets toutes les **30 secondes**.

### Règles d'alerte par défaut

MicroK8s configure automatiquement des alertes de base pour :
- Targets down
- Utilisation excessive des ressources
- Problèmes avec les composants Kubernetes

Vous pouvez les voir dans :

```bash
microk8s kubectl get prometheusrules -n monitoring
```

## Dépannage

### Problème 1 : Les pods restent en Pending

**Symptôme** :

```bash
microk8s kubectl get pods -n monitoring
```

```
NAME                 READY   STATUS    RESTARTS   AGE
prometheus-k8s-0     0/2     Pending   0          5m
```

**Causes possibles** :

1. **Ressources insuffisantes** : Pas assez de CPU ou RAM disponible
2. **Pas de node disponible** : Dans un cluster multi-node, aucun node ne peut accueillir le pod
3. **Volume non disponible** : Le stockage persistant ne peut pas être créé

**Solution** :

Vérifiez les événements du pod :

```bash
microk8s kubectl describe pod prometheus-k8s-0 -n monitoring
```

Regardez la section `Events` à la fin pour voir la raison exacte.

**Si c'est un problème de stockage**, assurez-vous que l'addon storage est activé :

```bash
microk8s enable storage
```

### Problème 2 : ImagePullBackOff

**Symptôme** :

```
NAME                 READY   STATUS             RESTARTS   AGE
prometheus-k8s-0     0/2     ImagePullBackOff   0          3m
```

**Cause** : Impossible de télécharger l'image Docker

**Solutions** :

1. **Vérifiez votre connexion internet**
2. **Attendez un peu** : Parfois c'est temporaire
3. **Vérifiez les logs** :

```bash
microk8s kubectl describe pod prometheus-k8s-0 -n monitoring
```

### Problème 3 : CrashLoopBackOff

**Symptôme** :

```
NAME                 READY   STATUS              RESTARTS   AGE
prometheus-k8s-0     1/2     CrashLoopBackOff    5          10m
```

**Cause** : Le conteneur démarre puis crash immédiatement

**Solution** :

Consultez les logs :

```bash
microk8s kubectl logs prometheus-k8s-0 -n monitoring -c prometheus
```

(Le flag `-c prometheus` spécifie le conteneur, car le pod en contient plusieurs)

### Problème 4 : Impossible d'accéder à l'interface web

**Symptôme** : `http://localhost:9090` ne répond pas

**Solutions** :

1. **Vérifiez que le port-forward est actif** : La commande doit être en cours d'exécution
2. **Vérifiez que le pod Prometheus est Running** :

```bash
microk8s kubectl get pods -n monitoring -l app.kubernetes.io/name=prometheus
```

3. **Essayez un autre port** si 9090 est déjà utilisé :

```bash
microk8s kubectl port-forward -n monitoring service/prometheus-k8s 9091:9090
```

Puis accédez à `http://localhost:9091`

### Problème 5 : Aucune métrique visible

**Symptôme** : Les requêtes PromQL ne retournent rien

**Solution** :

Vérifiez l'état des targets :

```bash
# Via l'interface web
http://localhost:9090/targets

# Ou via kubectl
microk8s kubectl get servicemonitors -n monitoring
```

Si toutes les targets sont DOWN, vérifiez :

1. **DNS fonctionne** :

```bash
microk8s kubectl run test-dns --image=busybox --rm -it -- nslookup prometheus-k8s.monitoring.svc.cluster.local
```

2. **Les services existent** :

```bash
microk8s kubectl get svc -n monitoring
```

## Commandes utiles pour l'administration

### Redémarrer Prometheus

```bash
microk8s kubectl rollout restart statefulset/prometheus-k8s -n monitoring
```

### Voir les logs de Prometheus

```bash
# Logs en direct
microk8s kubectl logs -f prometheus-k8s-0 -n monitoring -c prometheus

# Logs des 100 dernières lignes
microk8s kubectl logs --tail=100 prometheus-k8s-0 -n monitoring -c prometheus
```

### Voir l'utilisation des ressources

```bash
microk8s kubectl top pods -n monitoring
```

(Nécessite que metrics-server soit activé : `microk8s enable metrics-server`)

### Exporter la configuration Prometheus

```bash
microk8s kubectl get prometheus prometheus-k8s -n monitoring -o yaml > prometheus-config.yaml
```

## Résumé

Dans cette section, vous avez :

✅ **Installé Prometheus** avec une seule commande (`microk8s enable prometheus`)
✅ **Vérifié** que tous les composants sont déployés et fonctionnent
✅ **Accédé** à l'interface web de Prometheus
✅ **Exploré** l'interface et exécuté vos premières requêtes
✅ **Compris** ce qui a été déployé dans votre cluster
✅ **Appris** à dépanner les problèmes courants

Prometheus est maintenant opérationnel dans votre cluster MicroK8s ! Dans la prochaine section (12.4), nous explorerons en détail **comment Prometheus découvre automatiquement les services** et comment configurer le scraping pour vos propres applications.

---

**Point important** : Gardez à l'esprit que vous venez d'installer un système de monitoring professionnel en production-ready avec une seule commande. C'est la magie de MicroK8s ! Dans un environnement Kubernetes standard, cette installation prendrait beaucoup plus de temps et de configuration.

⏭️ [Configuration des targets et service discovery](/12-monitoring-avec-prometheus/04-configuration-des-targets-et-service-discovery.md)
