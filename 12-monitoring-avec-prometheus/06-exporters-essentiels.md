🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 12.6 Exporters essentiels (node, kube-state, cAdvisor)

## Introduction

Dans les sections précédentes, nous avons appris à utiliser PromQL pour interroger des métriques. Mais d'où viennent toutes ces métriques ? Comment Prometheus sait-il combien de CPU utilise un pod, combien de mémoire consomme un node, ou combien de pods sont en cours d'exécution ?

C'est le rôle des **exporters** : des programmes spécialisés qui collectent des informations depuis différentes sources et les exposent dans un format que Prometheus peut comprendre.

Lorsque vous avez installé Prometheus avec MicroK8s, trois exporters essentiels ont été automatiquement déployés :

1. **Node Exporter** : Métriques du système d'exploitation et du matériel
2. **kube-state-metrics** : État des objets Kubernetes
3. **cAdvisor** : Métriques de performance des conteneurs

Ces trois exporters forment la **triade fondamentale** du monitoring Kubernetes. Ensemble, ils fournissent une vue complète de votre cluster.

## Comprendre les exporters

### Qu'est-ce qu'un exporter ?

Un **exporter** est un programme qui :
1. Collecte des informations depuis une source (système, base de données, API)
2. Les transforme au format Prometheus
3. Les expose sur un endpoint HTTP (généralement `/metrics`)
4. Attend que Prometheus vienne les récupérer (scraping)

### Analogie simple

Imaginez un traducteur dans une réunion internationale :
- **La source** : Une personne qui parle japonais (le système d'exploitation)
- **L'exporter** : Le traducteur qui convertit en anglais (format Prometheus)
- **Prometheus** : La personne anglophone qui écoute (scraping)

Sans le traducteur, Prometheus ne pourrait pas comprendre ce que dit le système.

### Format des métriques Prometheus

Les exporters exposent des métriques dans un format texte simple :

```
# HELP node_cpu_seconds_total Seconds the CPUs spent in each mode.
# TYPE node_cpu_seconds_total counter
node_cpu_seconds_total{cpu="0",mode="idle"} 123456.78
node_cpu_seconds_total{cpu="0",mode="user"} 12345.67
node_cpu_seconds_total{cpu="1",mode="idle"} 123450.12

# HELP node_memory_MemTotal_bytes Total memory in bytes.
# TYPE node_memory_MemTotal_bytes gauge
node_memory_MemTotal_bytes 8589934592
```

**Éléments** :
- `# HELP` : Description de la métrique
- `# TYPE` : Type (counter, gauge, histogram, summary)
- Ligne de métrique : `nom{labels} valeur`

## Node Exporter : Les métriques système

### Qu'est-ce que Node Exporter ?

**Node Exporter** collecte les métriques au niveau du **système d'exploitation** et du **matériel** de chaque node (serveur) de votre cluster Kubernetes.

**Analogie** : C'est comme un médecin qui prend les signes vitaux d'un patient (température, pression, pouls). Node Exporter prend les "signes vitaux" de chaque serveur.

### Déploiement dans MicroK8s

Node Exporter est déployé comme un **DaemonSet**, ce qui signifie qu'il y a exactement **un pod par node**.

Vérifiez son état :

```bash
microk8s kubectl get daemonset node-exporter -n monitoring
```

Résultat attendu :

```
NAME            DESIRED   CURRENT   READY   UP-TO-DATE   AVAILABLE   NODE SELECTOR
node-exporter   1         1         1       1            1           <none>
```

**DESIRED = nombre de nodes** : Si vous avez 3 nodes, vous aurez 3 pods node-exporter.

### Catégories de métriques Node Exporter

Node Exporter collecte des centaines de métriques regroupées en catégories :

#### 1. CPU (Processeur)

**Métrique principale** : `node_cpu_seconds_total`

C'est un **counter** qui compte le temps CPU cumulé passé dans différents modes.

**Modes CPU** :
- **idle** : Inactif (le CPU ne fait rien)
- **user** : Temps passé à exécuter des applications utilisateur
- **system** : Temps passé dans le noyau Linux (kernel)
- **iowait** : Temps passé à attendre des I/O (disque, réseau)
- **irq** : Temps passé à gérer les interruptions matérielles
- **nice** : Temps passé sur des processus basse priorité
- **steal** : Temps "volé" (en environnement virtualisé)

**Requêtes PromQL utiles** :

```promql
# Utilisation CPU en % (moyenne de tous les cores)
100 - (avg(rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100)

# Utilisation CPU par core
100 - (rate(node_cpu_seconds_total{mode="idle"}[5m]) * 100)

# Temps d'attente I/O (peut indiquer un problème de disque)
rate(node_cpu_seconds_total{mode="iowait"}[5m]) * 100

# CPU utilisé par les applications
rate(node_cpu_seconds_total{mode="user"}[5m]) * 100
```

#### 2. Mémoire (RAM)

Node Exporter expose de nombreuses métriques mémoire issues de `/proc/meminfo`.

**Métriques principales** :

```
node_memory_MemTotal_bytes       # Total de RAM
node_memory_MemFree_bytes        # RAM libre
node_memory_MemAvailable_bytes   # RAM disponible (mieux que MemFree)
node_memory_Buffers_bytes        # RAM utilisée pour les buffers
node_memory_Cached_bytes         # RAM utilisée pour le cache
node_memory_SwapTotal_bytes      # Total swap
node_memory_SwapFree_bytes       # Swap libre
```

**Différence importante** :
- **MemFree** : RAM complètement inutilisée (souvent faible, c'est normal)
- **MemAvailable** : RAM effectivement disponible pour de nouveaux processus (inclut cache libérable)

**Requêtes PromQL utiles** :

```promql
# Mémoire utilisée en bytes
node_memory_MemTotal_bytes - node_memory_MemAvailable_bytes

# Mémoire utilisée en %
((node_memory_MemTotal_bytes - node_memory_MemAvailable_bytes) /
 node_memory_MemTotal_bytes) * 100

# Mémoire disponible en GB
node_memory_MemAvailable_bytes / 1024 / 1024 / 1024

# Swap utilisé (alerte si > 0, c'est souvent mauvais signe)
node_memory_SwapTotal_bytes - node_memory_SwapFree_bytes

# Utilisation du cache (normal qu'elle soit élevée)
node_memory_Cached_bytes / 1024 / 1024 / 1024
```

#### 3. Disque (Stockage)

**Métriques filesystem** :

```
node_filesystem_size_bytes       # Taille totale
node_filesystem_free_bytes       # Espace libre
node_filesystem_avail_bytes      # Espace disponible (peut être < free)
node_filesystem_files             # Nombre total d'inodes
node_filesystem_files_free        # Inodes libres
```

**Labels importants** :
- `mountpoint` : Point de montage (ex: `/`, `/home`, `/mnt/data`)
- `fstype` : Type de système de fichiers (ext4, xfs, etc.)

**Requêtes PromQL utiles** :

```promql
# Espace disque utilisé en %
(node_filesystem_size_bytes - node_filesystem_avail_bytes) /
node_filesystem_size_bytes * 100

# Espace libre en GB
node_filesystem_avail_bytes / 1024 / 1024 / 1024

# Filesystems à plus de 80% d'utilisation
((node_filesystem_size_bytes - node_filesystem_avail_bytes) /
 node_filesystem_size_bytes) > 0.8

# Utilisation par mountpoint
((node_filesystem_size_bytes - node_filesystem_avail_bytes) /
 node_filesystem_size_bytes * 100) and
 node_filesystem_mountpoint != ""
```

**Métriques I/O disque** :

```
node_disk_read_bytes_total       # Octets lus (counter)
node_disk_written_bytes_total    # Octets écrits (counter)
node_disk_reads_completed_total  # Nombre de lectures
node_disk_writes_completed_total # Nombre d'écritures
node_disk_io_time_seconds_total  # Temps passé en I/O
```

**Requêtes PromQL utiles** :

```promql
# Taux de lecture en MB/s
rate(node_disk_read_bytes_total[5m]) / 1024 / 1024

# Taux d'écriture en MB/s
rate(node_disk_written_bytes_total[5m]) / 1024 / 1024

# I/O total (lecture + écriture)
(rate(node_disk_read_bytes_total[5m]) +
 rate(node_disk_written_bytes_total[5m])) / 1024 / 1024

# IOPS (opérations I/O par seconde)
rate(node_disk_reads_completed_total[5m]) +
rate(node_disk_writes_completed_total[5m])
```

#### 4. Réseau

**Métriques réseau** :

```
node_network_receive_bytes_total    # Octets reçus (counter)
node_network_transmit_bytes_total   # Octets transmis (counter)
node_network_receive_packets_total  # Paquets reçus
node_network_transmit_packets_total # Paquets transmis
node_network_receive_errs_total     # Erreurs en réception
node_network_transmit_errs_total    # Erreurs en transmission
node_network_receive_drop_total     # Paquets droppés en réception
node_network_transmit_drop_total    # Paquets droppés en transmission
```

**Label important** :
- `device` : Interface réseau (eth0, ens160, wlan0, lo, etc.)

**Requêtes PromQL utiles** :

```promql
# Trafic entrant en MB/s
rate(node_network_receive_bytes_total[5m]) / 1024 / 1024

# Trafic sortant en MB/s
rate(node_network_transmit_bytes_total[5m]) / 1024 / 1024

# Bande passante totale (entrée + sortie)
(rate(node_network_receive_bytes_total[5m]) +
 rate(node_network_transmit_bytes_total[5m])) / 1024 / 1024

# Erreurs réseau (mauvais signe si > 0)
rate(node_network_receive_errs_total[5m]) +
rate(node_network_transmit_errs_total[5m])

# Paquets droppés
rate(node_network_receive_drop_total[5m]) +
rate(node_network_transmit_drop_total[5m])
```

#### 5. Load Average (Charge système)

**Métriques** :

```
node_load1   # Load average sur 1 minute
node_load5   # Load average sur 5 minutes
node_load15  # Load average sur 15 minutes
```

**Qu'est-ce que le load average ?**

Le load average représente le **nombre moyen de processus en attente d'exécution** (dans la file d'attente du CPU).

**Interprétation** :
- **Load < nombre de CPUs** : ✅ Système pas surchargé
- **Load = nombre de CPUs** : ⚠️ Système à pleine charge
- **Load > nombre de CPUs** : ❌ Système surchargé (processus en attente)

**Exemple** :
- Serveur avec 4 CPUs et load de 2.0 → OK, seulement 50% utilisé
- Serveur avec 4 CPUs et load de 8.0 → Problème, 2x trop de charge

**Requêtes PromQL utiles** :

```promql
# Load average 1 minute
node_load1

# Load par CPU (pour normaliser)
node_load1 / count(node_cpu_seconds_total{mode="idle"}) without (cpu, mode)

# Alerte si load > 80% de la capacité CPU
node_load1 / count(node_cpu_seconds_total{mode="idle"}) without (cpu, mode) > 0.8
```

#### 6. Uptime et boot time

```
node_boot_time_seconds  # Timestamp du dernier démarrage
node_time_seconds       # Heure actuelle du système
```

**Requêtes PromQL utiles** :

```promql
# Uptime en secondes
time() - node_boot_time_seconds

# Uptime en heures
(time() - node_boot_time_seconds) / 3600

# Uptime en jours
(time() - node_boot_time_seconds) / 86400

# Nodes redémarrés récemment (moins de 1h)
(time() - node_boot_time_seconds) < 3600
```

### Accéder directement aux métriques Node Exporter

Vous pouvez consulter les métriques exposées par Node Exporter :

```bash
# Port-forward vers node-exporter
microk8s kubectl port-forward -n monitoring daemonset/node-exporter 9100:9100

# Dans un autre terminal, récupérez les métriques
curl http://localhost:9100/metrics
```

Vous verrez des centaines de lignes de métriques au format Prometheus.

## kube-state-metrics : L'état de Kubernetes

### Qu'est-ce que kube-state-metrics ?

**kube-state-metrics** est un service qui interroge l'**API Kubernetes** et expose l'**état des objets** comme des métriques Prometheus.

**Différence clé** :
- **Node Exporter** : Métriques du système d'exploitation (niveau infrastructure)
- **kube-state-metrics** : Métriques Kubernetes (niveau orchestration)

**Analogie** : Si Kubernetes était une entreprise :
- **Node Exporter** : Surveille l'état des bureaux (électricité, climatisation, etc.)
- **kube-state-metrics** : Surveille l'organisation (combien d'employés, qui est en congé, quels projets sont actifs, etc.)

### Déploiement dans MicroK8s

kube-state-metrics est déployé comme un **Deployment** (généralement 1 réplica).

Vérifiez son état :

```bash
microk8s kubectl get deployment kube-state-metrics -n monitoring
```

### Catégories de métriques kube-state-metrics

#### 1. Pods

**Métriques principales** :

```
kube_pod_info                           # Informations générales sur les pods
kube_pod_status_phase                   # Phase du pod (Running, Pending, Failed, etc.)
kube_pod_status_ready                   # Statut ready du pod
kube_pod_container_status_ready         # Statut ready de chaque conteneur
kube_pod_container_status_restarts_total # Nombre de restarts
kube_pod_container_status_waiting       # Conteneur en attente
kube_pod_container_status_terminated    # Conteneur terminé
```

**Labels importants** :
- `namespace` : Namespace du pod
- `pod` : Nom du pod
- `container` : Nom du conteneur
- `phase` : Phase actuelle (Running, Pending, Failed, Succeeded, Unknown)

**Requêtes PromQL utiles** :

```promql
# Nombre total de pods
count(kube_pod_info)

# Pods par namespace
count(kube_pod_info) by (namespace)

# Pods running
count(kube_pod_status_phase{phase="Running"})

# Pods en erreur (Failed ou Unknown)
count(kube_pod_status_phase{phase=~"Failed|Unknown"})

# Pods avec restarts élevés (> 5)
kube_pod_container_status_restarts_total > 5

# Top 10 des pods avec le plus de restarts
topk(10, kube_pod_container_status_restarts_total)

# Pods not ready
kube_pod_status_ready{condition="false"}
```

#### 2. Deployments

**Métriques principales** :

```
kube_deployment_status_replicas              # Nombre de replicas souhaité
kube_deployment_status_replicas_available    # Replicas disponibles
kube_deployment_status_replicas_unavailable  # Replicas indisponibles
kube_deployment_status_replicas_updated      # Replicas à jour
kube_deployment_spec_replicas                # Replicas spécifiés dans le spec
```

**Requêtes PromQL utiles** :

```promql
# Deployments avec des replicas manquants
kube_deployment_status_replicas_available < kube_deployment_spec_replicas

# Ratio de disponibilité
kube_deployment_status_replicas_available / kube_deployment_spec_replicas

# Deployments non à jour (rolling update en cours)
kube_deployment_status_replicas_updated < kube_deployment_spec_replicas

# Liste des deployments incomplets
kube_deployment_status_replicas{namespace="production"} !=
kube_deployment_status_replicas_available{namespace="production"}
```

#### 3. Nodes

**Métriques principales** :

```
kube_node_info                    # Informations sur les nodes
kube_node_status_condition        # Conditions des nodes (Ready, MemoryPressure, etc.)
kube_node_status_allocatable      # Ressources allocables (CPU, mémoire)
kube_node_status_capacity         # Capacité totale
kube_node_spec_unschedulable      # Node marqué comme non planifiable
```

**Conditions importantes** :
- **Ready** : Node prêt à accepter des pods
- **MemoryPressure** : Mémoire faible
- **DiskPressure** : Disque faible
- **PIDPressure** : Trop de processus
- **NetworkUnavailable** : Réseau indisponible

**Requêtes PromQL utiles** :

```promql
# Nombre de nodes ready
count(kube_node_status_condition{condition="Ready", status="true"})

# Nodes not ready
kube_node_status_condition{condition="Ready", status="false"}

# Nodes avec pression mémoire
kube_node_status_condition{condition="MemoryPressure", status="true"}

# Capacité CPU totale du cluster
sum(kube_node_status_capacity{resource="cpu"})

# Capacité mémoire totale du cluster (en GB)
sum(kube_node_status_capacity{resource="memory"}) / 1024 / 1024 / 1024
```

#### 4. Services

**Métriques principales** :

```
kube_service_info            # Informations sur les services
kube_service_spec_type       # Type de service (ClusterIP, NodePort, LoadBalancer)
kube_service_status_load_balancer_ingress  # Ingress du LoadBalancer
```

**Requêtes PromQL utiles** :

```promql
# Nombre total de services
count(kube_service_info)

# Services par type
count(kube_service_info) by (type)

# Services LoadBalancer
kube_service_spec_type{type="LoadBalancer"}
```

#### 5. PersistentVolumes et PersistentVolumeClaims

**Métriques principales** :

```
kube_persistentvolume_status_phase              # Phase du PV (Available, Bound, Released, Failed)
kube_persistentvolumeclaim_status_phase         # Phase du PVC
kube_persistentvolume_capacity_bytes            # Capacité du PV
kube_persistentvolumeclaim_resource_requests_storage_bytes  # Stockage demandé
```

**Requêtes PromQL utiles** :

```promql
# PVCs non bound
kube_persistentvolumeclaim_status_phase{phase!="Bound"}

# Capacité totale de stockage disponible
sum(kube_persistentvolume_capacity_bytes) / 1024 / 1024 / 1024

# PVs par phase
count(kube_persistentvolume_status_phase) by (phase)
```

#### 6. Jobs et CronJobs

**Métriques principales** :

```
kube_job_status_succeeded        # Job réussi
kube_job_status_failed           # Job échoué
kube_job_complete                # Job terminé
kube_cronjob_next_schedule_time  # Prochaine exécution du CronJob
```

**Requêtes PromQL utiles** :

```promql
# Jobs échoués
kube_job_status_failed > 0

# Jobs actifs
kube_job_status_active

# Temps avant la prochaine exécution d'un CronJob (en minutes)
(kube_cronjob_next_schedule_time - time()) / 60
```

#### 7. Namespaces

**Métriques principales** :

```
kube_namespace_status_phase      # Phase du namespace (Active, Terminating)
kube_namespace_labels            # Labels du namespace
```

**Requêtes PromQL utiles** :

```promql
# Nombre de namespaces
count(kube_namespace_status_phase)

# Namespaces en cours de suppression
kube_namespace_status_phase{phase="Terminating"}
```

#### 8. Resource Quotas et LimitRanges

**Métriques principales** :

```
kube_resourcequota                    # Quotas de ressources
kube_resourcequota_used               # Utilisation des quotas
kube_limitrange                       # LimitRanges configurés
```

**Requêtes PromQL utiles** :

```promql
# Utilisation des quotas en %
(kube_resourcequota_used / kube_resourcequota) * 100

# Namespaces proches de leurs limites de quota
(kube_resourcequota_used / kube_resourcequota) > 0.9
```

### Accéder directement aux métriques kube-state-metrics

```bash
# Port-forward vers kube-state-metrics
microk8s kubectl port-forward -n monitoring deployment/kube-state-metrics 8080:8080

# Dans un autre terminal
curl http://localhost:8080/metrics
```

## cAdvisor : Métriques de performance des conteneurs

### Qu'est-ce que cAdvisor ?

**cAdvisor** (Container Advisor) collecte les **métriques de performance** des conteneurs : utilisation CPU, mémoire, réseau, I/O.

**Particularité** : cAdvisor est **intégré dans Kubelet**, le composant Kubernetes qui s'exécute sur chaque node. Vous n'avez donc pas besoin de le déployer séparément.

**Différences avec les autres exporters** :

| Exporter | Focus | Niveau |
|----------|-------|--------|
| Node Exporter | Métriques système/matériel | Infrastructure (OS) |
| kube-state-metrics | État des objets K8s | Orchestration (K8s API) |
| cAdvisor | Performance des conteneurs | Runtime (conteneurs) |

**Analogie** : Si votre cluster était une ville :
- **Node Exporter** : Surveille l'état des routes, de l'électricité, de l'eau
- **kube-state-metrics** : Surveille le cadastre (quels bâtiments, où, dans quel état)
- **cAdvisor** : Surveille l'activité dans chaque bâtiment (consommation électrique, occupation)

### Métriques cAdvisor

#### 1. CPU des conteneurs

**Métrique principale** : `container_cpu_usage_seconds_total`

Type : **Counter** (temps CPU cumulé en secondes)

**Labels importants** :
- `namespace` : Namespace Kubernetes
- `pod` : Nom du pod
- `container` : Nom du conteneur
- `image` : Image du conteneur

**Requêtes PromQL utiles** :

```promql
# Utilisation CPU par conteneur (en cores)
rate(container_cpu_usage_seconds_total[5m])

# En pourcentage (supposant 1 core)
rate(container_cpu_usage_seconds_total[5m]) * 100

# Top 10 des conteneurs utilisant le plus de CPU
topk(10, rate(container_cpu_usage_seconds_total{container!=""}[5m]))

# Moyenne CPU par namespace
avg(rate(container_cpu_usage_seconds_total{container!=""}[5m])) by (namespace)

# CPU utilisé vs limite
rate(container_cpu_usage_seconds_total[5m]) /
on(namespace,pod,container) container_spec_cpu_quota * 100000
```

#### 2. Mémoire des conteneurs

**Métriques principales** :

```
container_memory_usage_bytes          # Utilisation mémoire actuelle
container_memory_working_set_bytes    # Mémoire de travail (plus précis)
container_memory_rss                  # Resident Set Size
container_memory_cache                # Mémoire cache
container_memory_swap                 # Swap utilisé
```

**Différence importante** :
- `container_memory_usage_bytes` : Inclut le cache
- `container_memory_working_set_bytes` : Mémoire réellement utilisée (recommandé)

**Requêtes PromQL utiles** :

```promql
# Utilisation mémoire actuelle
container_memory_working_set_bytes

# En megabytes
container_memory_working_set_bytes / 1024 / 1024

# Top 10 des conteneurs utilisant le plus de mémoire
topk(10, container_memory_working_set_bytes{container!=""})

# Utilisation vs limite (en %)
(container_memory_working_set_bytes / container_spec_memory_limit_bytes) * 100

# Conteneurs dépassant 80% de leur limite
(container_memory_working_set_bytes / container_spec_memory_limit_bytes) > 0.8

# Mémoire totale utilisée par namespace
sum(container_memory_working_set_bytes{container!=""}) by (namespace) / 1024 / 1024 / 1024
```

#### 3. Réseau des conteneurs

**Métriques principales** :

```
container_network_receive_bytes_total     # Octets reçus (counter)
container_network_transmit_bytes_total    # Octets transmis (counter)
container_network_receive_packets_total   # Paquets reçus
container_network_transmit_packets_total  # Paquets transmis
container_network_receive_errors_total    # Erreurs en réception
container_network_transmit_errors_total   # Erreurs en transmission
```

**Requêtes PromQL utiles** :

```promql
# Trafic entrant en MB/s
rate(container_network_receive_bytes_total[5m]) / 1024 / 1024

# Trafic sortant en MB/s
rate(container_network_transmit_bytes_total[5m]) / 1024 / 1024

# Bande passante totale par pod
sum(rate(container_network_receive_bytes_total[5m]) +
    rate(container_network_transmit_bytes_total[5m])) by (pod) / 1024 / 1024

# Erreurs réseau
rate(container_network_receive_errors_total[5m]) +
rate(container_network_transmit_errors_total[5m])
```

#### 4. I/O disque des conteneurs

**Métriques principales** :

```
container_fs_reads_bytes_total   # Octets lus
container_fs_writes_bytes_total  # Octets écrits
container_fs_usage_bytes         # Espace disque utilisé
container_fs_limit_bytes         # Limite d'espace disque
```

**Requêtes PromQL utiles** :

```promql
# Taux de lecture en MB/s
rate(container_fs_reads_bytes_total[5m]) / 1024 / 1024

# Taux d'écriture en MB/s
rate(container_fs_writes_bytes_total[5m]) / 1024 / 1024

# Utilisation disque en %
(container_fs_usage_bytes / container_fs_limit_bytes) * 100

# Conteneurs avec utilisation disque > 80%
(container_fs_usage_bytes / container_fs_limit_bytes) > 0.8
```

#### 5. Ressources requests et limits

**Métriques principales** :

```
container_spec_cpu_quota          # CPU quota (limite)
container_spec_cpu_shares         # CPU shares (requests)
container_spec_memory_limit_bytes # Limite mémoire
container_spec_memory_reservation_bytes # Requests mémoire
```

**Requêtes PromQL utiles** :

```promql
# Ratio utilisation / limite CPU
rate(container_cpu_usage_seconds_total[5m]) /
(container_spec_cpu_quota / 100000)

# Ratio utilisation / limite mémoire
container_memory_working_set_bytes / container_spec_memory_limit_bytes

# Conteneurs sans limites (dangereux)
container_memory_working_set_bytes and
on(namespace,pod,container) container_spec_memory_limit_bytes == 0
```

### Accéder aux métriques cAdvisor

cAdvisor est accessible via Kubelet :

```bash
# cAdvisor écoute sur le port 10250 de Kubelet
# Les métriques sont disponibles via l'endpoint /metrics/cadvisor
```

Dans MicroK8s, Prometheus est automatiquement configuré pour scraper cAdvisor.

## Comparaison des trois exporters

### Tableau récapitulatif

| Aspect | Node Exporter | kube-state-metrics | cAdvisor |
|--------|---------------|-------------------|----------|
| **Focus** | Système/Matériel | État K8s | Performance conteneurs |
| **Source** | /proc, /sys | API Kubernetes | Cgroups |
| **Déploiement** | DaemonSet | Deployment | Intégré (Kubelet) |
| **Type métriques** | Gauge, Counter | Gauge principalement | Counter, Gauge |
| **Exemples** | CPU host, RAM, disque | Pods ready, Deployments | CPU pod, RAM pod |

### Quelle métrique utiliser ?

**Pour surveiller le node (machine physique/VM)** :
→ **Node Exporter**
```promql
# CPU du node
node_cpu_seconds_total
# RAM du node
node_memory_MemAvailable_bytes
```

**Pour surveiller l'état des objets Kubernetes** :
→ **kube-state-metrics**
```promql
# Nombre de pods
kube_pod_status_phase
# État des deployments
kube_deployment_status_replicas_available
```

**Pour surveiller la performance des conteneurs/pods** :
→ **cAdvisor**
```promql
# CPU du pod
container_cpu_usage_seconds_total
# RAM du pod
container_memory_working_set_bytes
```

### Exemple de requête combinée

Souvent, vous combinerez des métriques de différents exporters :

```promql
# Ratio : CPU pods / CPU total du node
sum(rate(container_cpu_usage_seconds_total{container!=""}[5m])) /
sum(rate(node_cpu_seconds_total{mode!="idle"}[5m]))

# Mémoire utilisée par les pods vs capacité du node
sum(container_memory_working_set_bytes{container!=""}) /
node_memory_MemTotal_bytes
```

## Vérifier que les exporters fonctionnent

### Via l'interface Prometheus

1. Accédez à Prometheus : `http://localhost:9090`
2. Allez dans **Status > Targets**
3. Vérifiez que les targets sont **UP** :

```
monitoring/node-exporter/0        ✅ UP
monitoring/kube-state-metrics/0   ✅ UP
monitoring/kubelet/0              ✅ UP (cAdvisor)
```

### Via des requêtes PromQL

**Test Node Exporter** :

```promql
up{job="node-exporter"}
```

Devrait retourner `1` si actif.

**Test kube-state-metrics** :

```promql
up{job="kube-state-metrics"}
```

**Test cAdvisor** :

```promql
up{job="kubelet"}
```

### Via kubectl

**Vérifier les pods** :

```bash
microk8s kubectl get pods -n monitoring
```

Tous les pods doivent être en état `Running`.

**Logs des exporters** :

```bash
# Node Exporter
microk8s kubectl logs -n monitoring daemonset/node-exporter

# kube-state-metrics
microk8s kubectl logs -n monitoring deployment/kube-state-metrics
```

## Bonnes pratiques

### 1. Ne filtrez pas les métriques trop tôt

Au début, laissez tous les exporters collecter toutes leurs métriques. Vous pourrez filtrer plus tard si nécessaire pour réduire la cardinalité.

### 2. Utilisez les bons agrégateurs

```promql
# ✅ Bon : sum pour additionner
sum(container_memory_usage_bytes)

# ✅ Bon : avg pour moyenner
avg(node_cpu_seconds_total)

# ❌ Mauvais : sum sur un gauge peut donner des résultats absurdes
sum(node_load1)  # Additionner des load averages n'a pas de sens
```

### 3. Attention aux labels vides

cAdvisor expose des métriques pour les conteneurs système avec `container=""`. Filtrez-les souvent :

```promql
# Sans filtre : inclut les conteneurs système
container_memory_usage_bytes

# ✅ Filtré : uniquement vos conteneurs
container_memory_usage_bytes{container!=""}

# Encore mieux : exclure aussi POD
container_memory_usage_bytes{container!="",container!="POD"}
```

### 4. Comprenez requests vs limits

```promql
# Utilisation vs requests (ce que vous avez demandé)
container_memory_working_set_bytes /
container_spec_memory_reservation_bytes

# Utilisation vs limits (maximum autorisé)
container_memory_working_set_bytes /
container_spec_memory_limit_bytes
```

### 5. Utilisez les métriques appropriées

Pour la **mémoire des conteneurs**, utilisez `container_memory_working_set_bytes` plutôt que `container_memory_usage_bytes` car c'est ce que Kubernetes utilise pour les décisions d'OOM (Out of Memory).

### 6. Documentez vos requêtes

Gardez un document avec vos requêtes favorites annotées :

```promql
# Top 10 pods par utilisation CPU
# Utile pour identifier les pods gourmands
topk(10, rate(container_cpu_usage_seconds_total{container!=""}[5m]))

# Alerte si un pod dépasse 90% de sa limite mémoire
# Aide à prévenir les OOMKilled
(container_memory_working_set_bytes / container_spec_memory_limit_bytes) > 0.9
```

## Dépannage

### Problème : Métriques Node Exporter manquantes

**Vérifications** :

1. Le DaemonSet est-il running ?

```bash
microk8s kubectl get pods -n monitoring -l app.kubernetes.io/name=node-exporter
```

2. Le pod peut-il accéder à /proc et /sys ?

Node Exporter a besoin de volumes hostPath pour accéder au système.

### Problème : Métriques kube-state-metrics manquantes

**Vérifications** :

1. Le deployment est-il running ?

```bash
microk8s kubectl get deployment kube-state-metrics -n monitoring
```

2. A-t-il les permissions RBAC ?

kube-state-metrics a besoin de permissions pour lire l'API Kubernetes.

```bash
microk8s kubectl get clusterrole kube-state-metrics
```

### Problème : Métriques cAdvisor manquantes

**Vérifications** :

1. Kubelet est-il accessible ?

```promql
up{job="kubelet"}
```

2. Les certificats sont-ils valides ?

Prometheus a besoin des bons certificats pour accéder à Kubelet.

## Ressources supplémentaires

### Documentation officielle

- **Node Exporter** : https://github.com/prometheus/node_exporter
- **kube-state-metrics** : https://github.com/kubernetes/kube-state-metrics
- **cAdvisor** : https://github.com/google/cadvisor

### Dashboards Grafana recommandés

- **Node Exporter Full** : ID 1860
- **Kubernetes / Compute Resources / Cluster** : ID 7249
- **Kubernetes / Compute Resources / Pod** : ID 6417

(Nous verrons comment importer ces dashboards dans le chapitre Grafana)

## Résumé

Dans cette section, vous avez appris :

✅ **Le concept d'exporters** et leur rôle dans Prometheus
✅ **Node Exporter** pour les métriques système (CPU, RAM, disque, réseau)
✅ **kube-state-metrics** pour l'état des objets Kubernetes (pods, deployments, nodes)
✅ **cAdvisor** pour les métriques de performance des conteneurs
✅ **Les différences** entre les trois exporters et quand utiliser chacun
✅ **Des requêtes PromQL pratiques** pour chaque exporter
✅ **Comment vérifier** que les exporters fonctionnent correctement
✅ **Les bonnes pratiques** pour interroger les métriques

**Points clés à retenir** :

1. Ces trois exporters forment la **base du monitoring Kubernetes**
2. **Node Exporter** = niveau infrastructure (OS/hardware)
3. **kube-state-metrics** = niveau orchestration (objets K8s)
4. **cAdvisor** = niveau application (conteneurs)
5. Combinez-les pour avoir une **vue complète** de votre cluster

Dans la prochaine section (12.7), nous découvrirons les **Recording Rules**, un mécanisme pour optimiser Prometheus en pré-calculant des requêtes complexes.

---

**Conseil pratique** : Créez un dashboard Grafana simple avec quelques métriques de chaque exporter pour vous familiariser avec leurs différences. Cela vous aidera à mieux comprendre quelle métrique utiliser dans chaque situation !

⏭️ [Recording rules et optimisation](/12-monitoring-avec-prometheus/07-recording-rules-et-optimisation.md)
