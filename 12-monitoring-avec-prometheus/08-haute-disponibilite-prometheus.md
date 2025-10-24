🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 12.8 Haute disponibilité Prometheus

## Introduction

Imaginez que vous ayez passé des semaines à configurer Prometheus, créé des dizaines de dashboards Grafana, et configuré des alertes critiques pour votre infrastructure. Un jour, le serveur qui héberge Prometheus tombe en panne. Résultat :
- ❌ Plus de dashboards
- ❌ Plus d'alertes
- ❌ Plus de monitoring
- ❌ Vous êtes aveugle sur votre infrastructure

C'est exactement le problème que résout la **Haute Disponibilité** (High Availability ou HA).

### Qu'est-ce que la Haute Disponibilité ?

La **haute disponibilité** est la capacité d'un système à rester opérationnel même en cas de panne d'un ou plusieurs composants. C'est l'élimination des **points uniques de défaillance** (Single Point of Failure ou SPOF).

**Analogie** :

Imaginez un hôpital avec un seul médecin. Si ce médecin tombe malade, plus personne ne peut être soigné (SPOF). Un hôpital avec 5 médecins a une haute disponibilité : si un médecin est absent, les autres continuent à travailler.

### Pourquoi s'en préoccuper dans un lab ?

**Bonne question !** Pour un lab personnel sur MicroK8s, la haute disponibilité n'est généralement **pas nécessaire**. Si Prometheus tombe, vous pouvez simplement le redémarrer.

**Cependant**, comprendre la HA est important car :
1. C'est essentiel en **production** (entreprise)
2. Cela vous prépare pour des **certifications** Kubernetes (CKA, CKAD)
3. C'est une **compétence recherchée** par les employeurs
4. Vous pourriez vouloir **expérimenter** dans votre lab
5. Comprendre l'architecture HA approfondit votre connaissance de Prometheus

**Dans cette section**, nous couvrirons :
- Les concepts généraux de HA
- Comment implémenter HA avec Prometheus
- Comment l'activer dans MicroK8s (si vous le souhaitez)
- Les solutions de stockage long-terme

## Les défis de la Haute Disponibilité avec Prometheus

### Le problème fondamental

Prometheus a été conçu comme un système **simple et autonome** :
- Il stocke ses données **localement** (sur son propre disque)
- Il **ne partage pas** ses données entre instances
- Il n'y a **pas de réplication native**

**Conséquence** : Si vous avez 2 instances Prometheus qui scrapent les mêmes targets, vous obtenez :
- 2 bases de données **indépendantes**
- 2 copies **légèrement différentes** des mêmes métriques
- Pas de synchronisation entre elles

### Différences entre les instances

Même si deux instances Prometheus scrapent exactement les mêmes targets, leurs données diffèrent légèrement :

```
Instance 1 scrape à 10:00:00 → valeur: 100
Instance 2 scrape à 10:00:01 → valeur: 101

Instance 1 scrape à 10:00:30 → valeur: 150
Instance 2 scrape à 10:00:31 → valeur: 151
```

Les timestamps et les valeurs ne sont jamais parfaitement identiques.

### Implications pour les requêtes

Quand vous interrogez un Prometheus HA, vous devez :
- Interroger **toutes les instances**
- **Dédupliquer** les résultats
- Gérer les **séries temporelles manquantes** (si une instance était down)

## Architecture Haute Disponibilité : Concepts

### Stratégie 1 : Prometheus en paires (Réplication simple)

C'est l'approche la plus simple pour la HA.

```
        ┌────────────────────────────────┐
        │    CLUSTER KUBERNETES          │
        │                                │
        │   ┌──────┐    ┌──────┐         │
        │   │ App1 │    │ App2 │         │
        │   └───┬──┘    └───┬──┘         │
        │       │           │            │
        │       ↓           ↓            │
        │   ┌───────────────────┐        │
        │   │                   │        │
        │   ↓                   ↓        │
        │ ┌────────────┐  ┌────────────┐ │
        │ │Prometheus 1│  │Prometheus 2│ │
        │ │            │  │            │ │
        │ │ Data Store │  │ Data Store │ │
        │ └────────────┘  └────────────┘ │
        └────────────────────────────────┘
              │                │
              ↓                ↓
        ┌──────────┐    ┌──────────┐
        │Grafana   │    │AlertMgr  │
        │(query    │    │(cluster) │
        │both)     │    │          │
        └──────────┘    └──────────┘
```

**Fonctionnement** :

1. **Deux instances Prometheus** scrapent exactement les mêmes targets
2. Chaque instance a sa **propre base de données**
3. **Grafana** interroge les deux instances
4. **Alertmanager** reçoit les alertes des deux instances (et déduplique)

**Avantages** :
- ✅ Simple à mettre en place
- ✅ Si une instance tombe, l'autre continue
- ✅ Pas de dépendance externe

**Inconvénients** :
- ❌ Consomme 2x les ressources (2x CPU, 2x RAM, 2x disque)
- ❌ Les données ne sont pas identiques (légères variations)
- ❌ Grafana doit gérer la déduplication
- ❌ Pas de stockage long-terme

### Stratégie 2 : Prometheus avec stockage externe (Remote Storage)

Au lieu de stocker localement, Prometheus envoie ses métriques vers un **stockage externe** centralisé.

```
     ┌──────────────────────────────────────┐
     │      CLUSTER KUBERNETES              │
     │                                      │
     │  ┌────────────┐      ┌────────────┐  │
     │  │Prometheus 1│      │Prometheus 2│  │
     │  │            │      │            │  │
     │  │(15 jours)  │      │(15 jours)  │  │
     │  └─────┬──────┘      └─────┬──────┘  │
     │        │                   │         │
     └────────┼───────────────────┼─────────┘
              │                   │
              └──────┬────────────┘
                     ↓
            ┌──────────────────┐
            │ REMOTE STORAGE   │
            │                  │
            │ (Thanos, Cortex, │
            │  VictoriaMetrics)│
            │                  │
            │  (plusieurs      │
            │   années)        │
            └──────────────────┘
                     ↑
                     │
              ┌──────┴──────┐
              │   Grafana   │
              │   (query)   │
              └─────────────┘
```

**Fonctionnement** :

1. Chaque Prometheus stocke **localement** les données récentes (ex: 15 jours)
2. Chaque Prometheus **envoie** aussi ses métriques au stockage externe
3. Le stockage externe **conserve** l'historique long-terme (plusieurs années)
4. Grafana peut interroger soit Prometheus directement (données récentes) soit le stockage externe (historique)

**Avantages** :
- ✅ Stockage long-terme (années)
- ✅ Requêtes sur l'historique complet
- ✅ Moins de stockage local nécessaire
- ✅ Déduplication gérée par le stockage externe

**Inconvénients** :
- ❌ Plus complexe à configurer
- ❌ Dépendance externe (besoin d'un système de stockage supplémentaire)
- ❌ Coût (infrastructure supplémentaire)

### Stratégie 3 : Prometheus Operator avec StatefulSet

Avec Prometheus Operator (utilisé dans MicroK8s), Prometheus est déployé comme un **StatefulSet** avec des volumes persistants.

```
┌─────────────────────────────────┐
│    KUBERNETES CLUSTER           │
│                                 │
│  ┌────────────────────────────┐ │
│  │  Prometheus StatefulSet    │ │
│  │                            │ │
│  │  ┌──────────────────────┐  │ │
│  │  │ prometheus-k8s-0     │  │ │
│  │  │ ┌──────────────────┐ │  │ │
│  │  │ │  Prometheus      │ │  │ │
│  │  │ │                  │ │  │ │
│  │  │ └────────┬─────────┘ │  │ │
│  │  │          │           │  │ │
│  │  │  ┌───────▼─────────┐ │  │ │
│  │  │  │ PersistentVolume│ │  │ │
│  │  │  │   (15 jours)    │ │  │ │
│  │  │  └─────────────────┘ │  │ │
│  │  └──────────────────────┘  │ │
│  └────────────────────────────┘ │
└─────────────────────────────────┘
```

**Fonctionnement** :

1. Prometheus s'exécute dans un pod avec un **volume persistant**
2. Si le pod crash, Kubernetes le **redémarre automatiquement**
3. Les données sont **préservées** sur le volume persistant
4. Le nouveau pod retrouve toutes les données

**Avantages** :
- ✅ Redémarrage automatique
- ✅ Données préservées
- ✅ Simple (géré par Kubernetes)

**Inconvénients** :
- ❌ Si le **node** entier tombe, Prometheus est down jusqu'à ce que Kubernetes le reschedule
- ❌ Pas de vraie HA (un seul pod actif à la fois)

**Note** : C'est la configuration par défaut dans MicroK8s.

## Alertmanager en Haute Disponibilité

### Le problème des alertes dupliquées

Si vous avez 2 instances Prometheus qui envoient des alertes, vous recevrez **2 fois chaque alerte** :

```
Prometheus 1 → Alerte: "CPU High" → Slack: "CPU High"
Prometheus 2 → Alerte: "CPU High" → Slack: "CPU High"
```

Vous recevez 2 notifications identiques ! 😫

### La solution : Alertmanager en cluster

Alertmanager peut fonctionner en **mode cluster** pour dédupliquer automatiquement les alertes.

```
     ┌─────────────┐    ┌─────────────┐
     │Prometheus 1 │    │Prometheus 2 │
     └──────┬──────┘    └──────┬──────┘
            │                  │
            │ Alerte: CPU High │
            │                  │
            ↓                  ↓
     ┌─────────────┐    ┌─────────────┐
     │AlertManager1│◄──►│AlertManager2│
     │             │    │             │
     │  (cluster)  │    │  (cluster)  │
     └──────┬──────┘    └──────┬──────┘
            │                  │
            └────────┬─────────┘
                     │
                     ↓
              ┌────────────┐
              │   Slack    │
              │            │
              │ "CPU High" │ ← Une seule notification
              └────────────┘
```

**Fonctionnement** :

1. Les instances Alertmanager communiquent entre elles (gossip protocol)
2. Elles partagent les alertes qu'elles ont reçues
3. Elles **déduplicent** automatiquement
4. Une **seule** notification est envoyée

**Configuration** :

```yaml
apiVersion: monitoring.coreos.com/v1
kind: Alertmanager
metadata:
  name: main
  namespace: monitoring
spec:
  replicas: 3  # 3 instances en cluster
  # Les instances se découvrent automatiquement
```

## Configuration HA dans MicroK8s

### Configuration par défaut

Par défaut, MicroK8s déploie :
- **1 instance Prometheus** (prometheus-k8s-0)
- **1 instance Alertmanager** (alertmanager-main-0)

C'est suffisant pour un lab, mais pas HA.

### Activer la Haute Disponibilité

Pour activer la HA dans MicroK8s :

#### Étape 1 : Augmenter le nombre de replicas Prometheus

```bash
# Éditer la ressource Prometheus
microk8s kubectl edit prometheus prometheus-k8s -n monitoring
```

Modifiez la ligne `replicas` :

```yaml
spec:
  replicas: 2  # Changez de 1 à 2 (ou plus)
```

Sauvegardez et quittez. Kubernetes créera automatiquement un deuxième pod Prometheus.

Vérifiez :

```bash
microk8s kubectl get pods -n monitoring -l app.kubernetes.io/name=prometheus
```

Vous devriez voir :

```
NAME                READY   STATUS    RESTARTS   AGE
prometheus-k8s-0    2/2     Running   0          5m
prometheus-k8s-1    2/2     Running   0          1m  ← Nouveau pod
```

#### Étape 2 : Augmenter le nombre de replicas Alertmanager

```bash
# Éditer la ressource Alertmanager
microk8s kubectl edit alertmanager alertmanager-main -n monitoring
```

Modifiez :

```yaml
spec:
  replicas: 3  # 3 pour former un cluster (nombre impair recommandé)
```

Vérifiez :

```bash
microk8s kubectl get pods -n monitoring -l app.kubernetes.io/name=alertmanager
```

```
NAME                    READY   STATUS    RESTARTS   AGE
alertmanager-main-0     2/2     Running   0          5m
alertmanager-main-1     2/2     Running   0          2m
alertmanager-main-2     2/2     Running   0          2m
```

#### Étape 3 : Configurer Grafana pour interroger les deux Prometheus

Grafana doit être configuré pour interroger les deux instances Prometheus.

**Option 1 : Data Source avec load balancing** (Simple)

Utilisez le Service Kubernetes qui fait déjà du load balancing :

```yaml
# Le service prometheus-k8s fait automatiquement du load balancing
# entre les pods
http://prometheus-k8s.monitoring.svc:9090
```

**Option 2 : Deux Data Sources séparées** (Plus de contrôle)

Dans Grafana, ajoutez deux data sources :

```
Data Source 1:
  Name: Prometheus-1
  URL: http://prometheus-k8s-0.prometheus-operated.monitoring.svc:9090

Data Source 2:
  Name: Prometheus-2
  URL: http://prometheus-k8s-1.prometheus-operated.monitoring.svc:9090
```

Puis dans vos dashboards, interrogez les deux sources et déduplicez les résultats.

### Vérifier la configuration HA

#### Test 1 : Simuler une panne

```bash
# Supprimer un pod Prometheus
microk8s kubectl delete pod prometheus-k8s-0 -n monitoring

# Vérifier que l'autre prend le relais
microk8s kubectl get pods -n monitoring -l app.kubernetes.io/name=prometheus
```

Vous devriez voir :
- `prometheus-k8s-1` toujours Running
- `prometheus-k8s-0` en cours de redémarrage (ContainerCreating)

**Pendant ce temps**, Grafana devrait continuer à fonctionner (en interrogeant prometheus-k8s-1).

#### Test 2 : Vérifier la déduplication Alertmanager

```bash
# Voir les logs d'un Alertmanager
microk8s kubectl logs alertmanager-main-0 -n monitoring -c alertmanager | grep -i cluster
```

Vous devriez voir des messages indiquant que les instances communiquent :

```
level=info msg="Listening on :9094 (cluster)"
level=info msg="Cluster joined" peers=3
```

### Considérations de ressources

**Important** : La HA consomme plus de ressources.

**Avant HA** :
- 1 Prometheus : ~500MB RAM, 0.5 CPU
- 1 Alertmanager : ~100MB RAM, 0.1 CPU
- **Total** : ~600MB RAM, 0.6 CPU

**Après HA (2 Prometheus, 3 Alertmanager)** :
- 2 Prometheus : ~1GB RAM, 1 CPU
- 3 Alertmanager : ~300MB RAM, 0.3 CPU
- **Total** : ~1.3GB RAM, 1.3 CPU

**Assurez-vous** d'avoir suffisamment de ressources sur votre machine avant d'activer la HA !

## Stockage Long-Terme : Thanos

### Qu'est-ce que Thanos ?

**Thanos** est une solution open-source qui étend Prometheus pour offrir :
- **Stockage long-terme** (années) dans un stockage objet (S3, GCS, etc.)
- **Requêtes globales** sur plusieurs Prometheus
- **Déduplication** automatique
- **Downsampling** (réduction de la résolution des anciennes données)

### Architecture Thanos

```
┌────────────────────────────────────────────────────┐
│            KUBERNETES CLUSTER                      │
│                                                    │
│  ┌──────────────┐           ┌──────────────┐       │
│  │ Prometheus 1 │           │ Prometheus 2 │       │
│  │              │           │              │       │
│  │ + Sidecar    │           │ + Sidecar    │       │
│  └──────┬───────┘           └──────┬───────┘       │
│         │                          │               │
│         └────────┬─────────────────┘               │
│                  │                                 │
│                  ↓                                 │
│         ┌─────────────────┐                        │
│         │  Thanos Store   │                        │
│         │   Gateway       │                        │
│         └────────┬────────┘                        │
└──────────────────┼─────────────────────────────────┘
                   │
                   ↓
          ┌────────────────┐
          │  Object Storage│
          │   (S3, GCS)    │
          │                │
          │  (plusieurs    │
          │   années)      │
          └────────┬───────┘
                   │
                   ↓
            ┌──────────────┐
            │Thanos Query  │
            │              │
            │ (API unifiée)│
            └──────┬───────┘
                   │
                   ↓
            ┌──────────────┐
            │   Grafana    │
            └──────────────┘
```

**Composants Thanos** :

1. **Thanos Sidecar** : S'exécute à côté de Prometheus, envoie les données vers le stockage objet
2. **Thanos Store Gateway** : Permet de requêter les données historiques dans le stockage objet
3. **Thanos Query** : Interface unifiée qui interroge à la fois Prometheus (données récentes) et Store Gateway (historique)
4. **Thanos Compactor** : Compacte et downsampling les anciennes données
5. **Thanos Ruler** : Évalue les recording et alerting rules sur les données long-terme

### Avantages de Thanos

✅ **Stockage illimité** : Gardez des années de métriques dans S3/GCS (peu coûteux)
✅ **Vue globale** : Interrogez plusieurs Prometheus depuis une seule interface
✅ **Déduplication** : Gère automatiquement les métriques dupliquées
✅ **Performance** : Downsampling réduit la quantité de données pour les requêtes longues
✅ **Haute disponibilité** : Si un Prometheus tombe, les autres continuent

### Installation de Thanos (Concepts)

**Note** : L'installation complète de Thanos est avancée et sort du cadre d'un lab débutant. Voici les concepts.

#### Option 1 : Helm Chart

```bash
# Ajouter le repo Thanos
helm repo add bitnami https://charts.bitnami.com/bitnami

# Installer Thanos
helm install thanos bitnami/thanos \
  --namespace monitoring \
  --set objstoreConfig="..." \
  --set query.enabled=true \
  --set storegateway.enabled=true
```

#### Option 2 : kube-thanos

Le projet kube-thanos fournit des manifests Kubernetes prêts à l'emploi.

### Configuration minimale

Pour utiliser Thanos, vous avez besoin :

1. **Stockage objet** : Compte AWS S3, Google Cloud Storage, ou MinIO local
2. **Configuration Prometheus** avec sidecar Thanos
3. **Déploiement** des composants Thanos

**Exemple de configuration stockage (S3)** :

```yaml
type: S3
config:
  bucket: "my-thanos-bucket"
  endpoint: "s3.amazonaws.com"
  access_key: "ACCESS_KEY"
  secret_key: "SECRET_KEY"
```

## Alternatives à Thanos

### Cortex

**Cortex** est une alternative à Thanos développée par Grafana Labs (maintenant intégré dans Grafana Mimir).

**Différences** :
- Cortex est un service complet (pas de sidecar)
- Prometheus envoie directement les métriques via remote write
- Architecture différente mais objectifs similaires

### VictoriaMetrics

**VictoriaMetrics** est une base de données de séries temporelles compatible avec Prometheus.

**Avantages** :
- Très performant
- Utilise moins de ressources que Prometheus
- Stockage long-terme intégré
- Compatible avec l'API Prometheus

**Utilisation** :

Prometheus envoie ses métriques vers VictoriaMetrics via remote write, et Grafana interroge VictoriaMetrics.

### M3DB

**M3DB** est la solution de stockage de séries temporelles d'Uber.

Compatible avec Prometheus via remote write/read.

### Comparaison rapide

| Solution | Complexité | Stockage LT | Performance | Communauté |
|----------|-----------|-------------|-------------|-----------|
| Thanos | Moyenne | ✅ Excellent | ✅ Bon | ✅ Large |
| Cortex/Mimir | Élevée | ✅ Excellent | ✅ Très bon | ✅ Moyenne |
| VictoriaMetrics | Faible | ✅ Excellent | ✅ Excellent | ✅ Croissante |
| M3DB | Élevée | ✅ Excellent | ✅ Bon | ⚠️ Petite |

**Pour un lab MicroK8s** : VictoriaMetrics est probablement le plus simple à expérimenter.

## Bonnes pratiques pour la Haute Disponibilité

### 1. Utilisez un nombre impair de replicas Alertmanager

```yaml
# ✅ Bon : nombre impair (consensus plus facile)
replicas: 3  # ou 5, 7

# ❌ Évitez : nombre pair
replicas: 2  # ou 4, 6
```

### 2. Distribuez sur plusieurs nodes (si multi-node)

Si vous avez un cluster multi-node, utilisez les **anti-affinity rules** pour que les pods Prometheus ne s'exécutent pas sur le même node.

```yaml
spec:
  affinity:
    podAntiAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
      - labelSelector:
          matchExpressions:
          - key: app.kubernetes.io/name
            operator: In
            values:
            - prometheus
        topologyKey: kubernetes.io/hostname
```

### 3. Configurez des requests et limits appropriés

```yaml
resources:
  requests:
    memory: 2Gi
    cpu: 1000m
  limits:
    memory: 4Gi
    cpu: 2000m
```

Cela évite qu'un Prometheus ne consomme toutes les ressources du node.

### 4. Surveillez Prometheus lui-même

Créez des alertes sur Prometheus :

```yaml
- alert: PrometheusDown
  expr: up{job="prometheus"} == 0
  for: 5m
  annotations:
    summary: "Prometheus instance {{ $labels.instance }} is down"
```

### 5. Testez régulièrement votre HA

```bash
# Test mensuel : simulez des pannes
microk8s kubectl delete pod prometheus-k8s-0 -n monitoring

# Vérifiez que :
# - L'autre instance continue
# - Les alertes fonctionnent
# - Grafana affiche toujours les données
```

### 6. Documentez votre architecture

Gardez un schéma d'architecture à jour montrant :
- Combien d'instances Prometheus
- Où sont les données stockées
- Quelle est la rétention
- Comment les données sont sauvegardées

### 7. Planifiez la capacité

```promql
# Espace disque restant
predict_linear(node_filesystem_avail_bytes[1h], 7*24*3600) < 0

# Croissance des métriques
rate(prometheus_tsdb_head_series[1d])
```

Surveillez la croissance et ajustez les ressources avant d'avoir des problèmes.

## Quand implémenter la HA ?

### Vous N'AVEZ PAS besoin de HA si :

- 🏠 Lab personnel / apprentissage
- 👥 Équipe de développement (environnement dev/test)
- 📊 Monitoring non critique
- 💻 Ressources limitées
- 🎯 Focus sur l'apprentissage de Prometheus de base

**Dans ces cas**, une seule instance avec un volume persistant suffit.

### Vous AVEZ besoin de HA si :

- 🏢 Environnement de production
- 💰 Monitoring critique pour le business
- 📈 SLA stricts (99.9%+ de disponibilité)
- 👥 Grande équipe qui dépend du monitoring
- 🔔 Alertes critiques (on-call, PagerDuty)
- 🌍 Monitoring multi-région/multi-cluster

## Backup et Disaster Recovery

Même avec HA, vous devriez avoir une stratégie de backup.

### Option 1 : Snapshot des volumes persistants

Si vous utilisez un StorageClass qui supporte les snapshots :

```bash
# Créer un VolumeSnapshot
apiVersion: snapshot.storage.k8s.io/v1
kind: VolumeSnapshot
metadata:
  name: prometheus-snapshot
spec:
  volumeSnapshotClassName: csi-snapclass
  source:
    persistentVolumeClaimName: prometheus-k8s-db-prometheus-k8s-0
```

### Option 2 : Export avec Prometheus-backup-tool

Utilisez des outils comme `prometheus-backup-tool` pour exporter les données :

```bash
# Exemple conceptuel
prometheus-backup-tool \
  --prometheus-url=http://prometheus-k8s:9090 \
  --output=/backups/prometheus-$(date +%Y%m%d).tar.gz
```

### Option 3 : Réplication vers stockage externe

Avec Thanos ou VictoriaMetrics, vos données sont automatiquement répliquées vers un stockage objet durable (S3, GCS).

### Fréquence de backup recommandée

- **Production critique** : Quotidien
- **Production standard** : Hebdomadaire
- **Dev/Test** : Optionnel

## Monitoring de la HA

Pour vérifier que votre configuration HA fonctionne :

```promql
# Nombre d'instances Prometheus UP
count(up{job="prometheus"} == 1)
# Devrait être égal au nombre de replicas

# Nombre d'instances Alertmanager en cluster
sum(alertmanager_cluster_members)
# Devrait être égal au nombre de replicas Alertmanager

# Alertes non déduplicées (problème si > 0 pour la même alerte)
# Vérifiez dans l'UI Alertmanager
```

## Résumé

Dans cette section, vous avez appris :

✅ **Pourquoi la HA est importante** (élimination des SPOF)
✅ **Les défis spécifiques** de la HA avec Prometheus
✅ **Les différentes stratégies HA** : réplication, remote storage, StatefulSet
✅ **Alertmanager en cluster** pour dédupliquer les alertes
✅ **Comment activer la HA** dans MicroK8s
✅ **Thanos** et les solutions de stockage long-terme
✅ **Les alternatives** : Cortex, VictoriaMetrics, M3DB
✅ **Les bonnes pratiques** pour implémenter et maintenir la HA
✅ **Quand la HA est nécessaire** (et quand elle ne l'est pas)
✅ **Backup et disaster recovery** pour protéger vos données

**Points clés à retenir** :

1. La HA **n'est pas toujours nécessaire**, surtout dans un lab
2. Prometheus HA = **plusieurs instances indépendantes** qui scrapent les mêmes targets
3. **Alertmanager en cluster** déduplique automatiquement les alertes
4. **Thanos/VictoriaMetrics** pour le stockage long-terme et la vue globale
5. La HA a un **coût** : ressources, complexité, maintenance
6. **Testez** régulièrement votre configuration HA
7. **Documentez** votre architecture et vos procédures

**Pour votre lab MicroK8s** : La configuration par défaut (1 Prometheus + 1 Alertmanager) est suffisante. Mais maintenant vous comprenez comment évoluer vers une architecture HA si nécessaire !

Dans le prochain chapitre (13), nous découvrirons **Grafana**, l'outil de visualisation qui transformera vos métriques Prometheus en magnifiques dashboards interactifs.

---

**Conseil** : Si vous voulez expérimenter la HA dans votre lab, commencez simplement avec 2 instances Prometheus et observez comment elles fonctionnent ensemble. Vous pouvez toujours revenir à 1 instance si les ressources sont limitées. L'important est de comprendre les concepts !

⏭️ [Visualisation avec Grafana](/13-visualisation-avec-grafana/README.md)
