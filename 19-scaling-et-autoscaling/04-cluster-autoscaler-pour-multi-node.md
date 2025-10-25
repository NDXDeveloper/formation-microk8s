🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 19.4 Cluster Autoscaler (pour multi-node)

## Introduction au Cluster Autoscaler

Le **Cluster Autoscaler (CA)** est un composant Kubernetes qui ajuste automatiquement la taille de votre cluster en ajoutant ou retirant des **nœuds** (nodes) en fonction des besoins. C'est le troisième type d'autoscaling que nous découvrons, et il complète le HPA et le VPA.

### Les trois types d'autoscaling : Récapitulatif

Avant d'entrer dans les détails, récapitulons les trois niveaux d'autoscaling :

**1. Horizontal Pod Autoscaler (HPA) :**
- **Niveau :** Pods
- **Action :** Ajoute ou retire des pods
- **Exemple :** 3 pods → 10 pods (sur les mêmes nœuds)

**2. Vertical Pod Autoscaler (VPA) :**
- **Niveau :** Ressources des pods
- **Action :** Augmente ou diminue CPU/mémoire par pod
- **Exemple :** Pod avec 100m CPU → Pod avec 500m CPU

**3. Cluster Autoscaler (CA) :**
- **Niveau :** Nœuds du cluster
- **Action :** Ajoute ou retire des machines (nœuds)
- **Exemple :** Cluster de 3 nœuds → Cluster de 6 nœuds

### Analogie de l'immeuble

Imaginez un immeuble de bureaux :

**HPA (Horizontal Pod Autoscaler) :**
- Ajouter plus d'employés dans les bureaux existants
- "On embauche 5 personnes de plus"

**VPA (Vertical Pod Autoscaler) :**
- Donner un plus grand bureau à chaque employé
- "Chaque employé passe d'un bureau de 10m² à 20m²"

**CA (Cluster Autoscaler) :**
- Louer un étage supplémentaire dans l'immeuble
- "On ajoute tout un nouvel étage avec plus de bureaux"

Le Cluster Autoscaler agit au niveau le plus haut : l'infrastructure elle-même !

## Pourquoi utiliser le Cluster Autoscaler ?

### Le problème : Capacité limitée du cluster

Imaginez cette situation :

```
État initial :
- Cluster avec 3 nœuds
- Chaque nœud : 4 CPU, 8 Gi RAM
- Capacité totale : 12 CPU, 24 Gi RAM

Une application nécessite :
- 20 répliques
- Chaque pod : 1 CPU, 2 Gi RAM
- Besoin total : 20 CPU, 40 Gi RAM

Problème : Pas assez de ressources !
```

Le HPA voudrait créer 20 pods, mais le cluster n'a que 12 CPU disponibles. Les pods restent en état `Pending` (en attente).

### La solution : Cluster Autoscaler

Le Cluster Autoscaler détecte que des pods sont en attente faute de ressources, et :
1. Ajoute automatiquement de nouveaux nœuds au cluster
2. Les pods en attente sont alors programmés sur ces nouveaux nœuds
3. Quand la charge diminue, retire les nœuds inutilisés

**Avantages :**

- ✅ Capacité illimitée (dans les limites de votre infrastructure)
- ✅ Optimisation automatique des coûts (retire les nœuds inutiles)
- ✅ Pas de limite artificielle de ressources
- ✅ Fonctionne en synergie avec le HPA
- ✅ Transparence pour les applications

## Comment fonctionne le Cluster Autoscaler ?

Le Cluster Autoscaler suit un cycle de décision en deux phases :

### Phase 1 : Scale Up (Ajout de nœuds)

Le CA surveille en permanence l'état du cluster :

```
1. Détection de pods en Pending
   ↓
2. Analyse : Pourquoi sont-ils en Pending ?
   - Pas assez de CPU ?
   - Pas assez de mémoire ?
   - Problème de scheduling ?
   ↓
3. Simulation : Si on ajoute un nœud, est-ce que ça aide ?
   ↓
4. Si oui : Demander un nouveau nœud au cloud provider
   ↓
5. Attendre que le nœud soit prêt (3-5 minutes)
   ↓
6. Scheduler place les pods sur le nouveau nœud
```

**Déclencheurs de scale up :**
- Pods en état `Pending` depuis plus de 30 secondes (configurable)
- Raison : Ressources insuffisantes (CPU, mémoire)
- Le nœud à ajouter pourrait résoudre le problème

**Le CA ne scale PAS up si :**
- Les pods sont Pending pour d'autres raisons (image non disponible, secrets manquants, etc.)
- Le nombre maximum de nœuds est atteint
- Le budget est épuisé

### Phase 2 : Scale Down (Retrait de nœuds)

Le CA analyse régulièrement si des nœuds sont sous-utilisés :

```
1. Identifier les nœuds sous-utilisés
   - Utilisation < 50% (par défaut)
   - Pendant 10 minutes (par défaut)
   ↓
2. Simulation : Peut-on déplacer tous les pods ailleurs ?
   ↓
3. Vérifier les contraintes :
   - PodDisruptionBudgets respectés ?
   - Pas de pods avec local storage ?
   - Pas de DaemonSets (sauf exceptions) ?
   ↓
4. Si OK : Marquer le nœud pour suppression
   ↓
5. Évacuer (drain) les pods vers d'autres nœuds
   ↓
6. Attendre que tous les pods soient relocalisés
   ↓
7. Supprimer le nœud
```

**Conditions pour scale down :**
- Nœud utilisé à moins de 50% de sa capacité
- Tous les pods peuvent être déplacés ailleurs
- Pas de pods "non évacuables" (système, DaemonSets, local storage)
- PodDisruptionBudgets respectés
- Le nœud est éligible depuis 10+ minutes

**Le CA ne scale PAS down si :**
- Cela violerait les PodDisruptionBudgets
- Des pods ne peuvent pas être déplacés
- Le nombre minimum de nœuds serait atteint
- Le nœud a été ajouté il y a moins de 10 minutes

## Prérequis pour utiliser le Cluster Autoscaler

### 1. Cluster multi-node

**Important :** Le Cluster Autoscaler n'a de sens que sur un cluster avec plusieurs nœuds. Sur un cluster single-node, il ne peut rien faire !

**Pour MicroK8s :**
Vous devez avoir un cluster avec au moins 2-3 nœuds. Voir la section 21 du tutoriel pour créer un cluster multi-node.

```bash
# Vérifier le nombre de nœuds
kubectl get nodes
```

Sortie attendue (exemple) :
```
NAME          STATUS   ROLES    AGE   VERSION
node-master   Ready    <none>   5d    v1.28.3
node-worker1  Ready    <none>   5d    v1.28.3
node-worker2  Ready    <none>   5d    v1.28.3
```

### 2. Infrastructure compatible

Le Cluster Autoscaler fonctionne **seulement** avec des infrastructures qui supportent l'ajout/retrait automatique de machines :

**☁️ Cloud providers supportés :**
- AWS (Auto Scaling Groups)
- Google Cloud Platform (Managed Instance Groups)
- Azure (Virtual Machine Scale Sets)
- DigitalOcean
- Et autres...

**⚠️ Limitations pour MicroK8s on-premise :**

Si votre cluster MicroK8s tourne sur des machines physiques ou virtuelles que vous gérez vous-même (homelab, datacenter local), le Cluster Autoscaler **ne peut pas** automatiquement créer de nouvelles machines.

**Solutions pour on-premise :**
1. **Infrastructure as Code** : Utiliser Terraform, Ansible pour automatiser la création de VMs
2. **Bare Metal Autoscaling** : Solutions comme Cluster API pour bare metal
3. **Hybrid approach** : Pool de machines "dormantes" que le CA peut activer
4. **Limitation acceptée** : Pas de vrai autoscaling, gestion manuelle du nombre de nœuds

Pour un **homelab typique**, le Cluster Autoscaler n'est généralement **pas pratique**. Cette section est surtout pertinente si vous déployez MicroK8s dans le cloud ou avec une infrastructure automatisable.

### 3. Permissions nécessaires

Le Cluster Autoscaler doit pouvoir interagir avec votre infrastructure :

**Dans le cloud :**
- Permissions IAM pour créer/détruire des instances
- Accès à l'API du cloud provider
- Credentials configurés

**Configuration typique (AWS exemple) :**
Le CA a besoin de :
- Lire les Auto Scaling Groups
- Modifier la taille des Auto Scaling Groups
- Lire les tags des instances

## Installation du Cluster Autoscaler

### Méthode 1 : Installation via Helm (Recommandé)

**Étape 1 : Installer Helm si nécessaire**

```bash
# Sur MicroK8s
microk8s enable helm3

# Créer un alias
alias helm='microk8s helm3'
```

**Étape 2 : Ajouter le repository Helm**

```bash
helm repo add autoscaler https://kubernetes.github.io/autoscaler
helm repo update
```

**Étape 3 : Créer un fichier de configuration**

Créez `cluster-autoscaler-values.yaml` :

```yaml
cloudProvider: aws    # Ou gce, azure, etc.

awsRegion: eu-west-1  # Votre région AWS

autoDiscovery:
  clusterName: mon-cluster-microk8s
  enabled: true

rbac:
  create: true
  serviceAccount:
    name: cluster-autoscaler
    annotations:
      eks.amazonaws.com/role-arn: arn:aws:iam::123456789:role/cluster-autoscaler

extraArgs:
  scale-down-enabled: true
  scale-down-delay-after-add: 10m
  scale-down-unneeded-time: 10m
  skip-nodes-with-local-storage: false
  balance-similar-node-groups: true

resources:
  requests:
    cpu: 100m
    memory: 300Mi
  limits:
    cpu: 100m
    memory: 300Mi
```

**Étape 4 : Installer le Cluster Autoscaler**

```bash
helm install cluster-autoscaler autoscaler/cluster-autoscaler \
  --namespace kube-system \
  --values cluster-autoscaler-values.yaml
```

**Étape 5 : Vérifier l'installation**

```bash
kubectl get pods -n kube-system | grep cluster-autoscaler
```

Sortie attendue :
```
cluster-autoscaler-xxxxxxxxx-xxxxx   1/1     Running   0          2m
```

### Méthode 2 : Installation via manifeste YAML

Si vous ne pouvez pas utiliser Helm, vous pouvez déployer via YAML :

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: cluster-autoscaler
  namespace: kube-system
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: cluster-autoscaler
rules:
- apiGroups: [""]
  resources: ["events", "endpoints"]
  verbs: ["create", "patch"]
- apiGroups: [""]
  resources: ["pods/eviction"]
  verbs: ["create"]
- apiGroups: [""]
  resources: ["pods/status"]
  verbs: ["update"]
- apiGroups: [""]
  resources: ["endpoints"]
  resourceNames: ["cluster-autoscaler"]
  verbs: ["get", "update"]
- apiGroups: [""]
  resources: ["nodes"]
  verbs: ["watch", "list", "get", "update"]
- apiGroups: [""]
  resources: ["namespaces", "pods", "services", "replicationcontrollers", "persistentvolumeclaims", "persistentvolumes"]
  verbs: ["watch", "list", "get"]
- apiGroups: ["extensions"]
  resources: ["replicasets", "daemonsets"]
  verbs: ["watch", "list", "get"]
- apiGroups: ["policy"]
  resources: ["poddisruptionbudgets"]
  verbs: ["watch", "list"]
- apiGroups: ["apps"]
  resources: ["statefulsets", "replicasets", "daemonsets"]
  verbs: ["watch", "list", "get"]
- apiGroups: ["storage.k8s.io"]
  resources: ["storageclasses", "csinodes", "csidrivers", "csistoragecapacities"]
  verbs: ["watch", "list", "get"]
- apiGroups: ["batch"]
  resources: ["jobs", "cronjobs"]
  verbs: ["watch", "list", "get"]
- apiGroups: ["coordination.k8s.io"]
  resources: ["leases"]
  verbs: ["create"]
- apiGroups: ["coordination.k8s.io"]
  resourceNames: ["cluster-autoscaler"]
  resources: ["leases"]
  verbs: ["get", "update"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: cluster-autoscaler
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-autoscaler
subjects:
- kind: ServiceAccount
  name: cluster-autoscaler
  namespace: kube-system
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: cluster-autoscaler
  namespace: kube-system
spec:
  replicas: 1
  selector:
    matchLabels:
      app: cluster-autoscaler
  template:
    metadata:
      labels:
        app: cluster-autoscaler
    spec:
      serviceAccountName: cluster-autoscaler
      containers:
      - name: cluster-autoscaler
        image: registry.k8s.io/autoscaling/cluster-autoscaler:v1.28.2
        command:
        - ./cluster-autoscaler
        - --v=4
        - --stderrthreshold=info
        - --cloud-provider=aws
        - --skip-nodes-with-local-storage=false
        - --expander=least-waste
        - --node-group-auto-discovery=asg:tag=k8s.io/cluster-autoscaler/enabled,k8s.io/cluster-autoscaler/mon-cluster
        - --balance-similar-node-groups
        - --skip-nodes-with-system-pods=false
        resources:
          requests:
            cpu: 100m
            memory: 300Mi
          limits:
            cpu: 100m
            memory: 300Mi
        volumeMounts:
        - name: ssl-certs
          mountPath: /etc/ssl/certs/ca-certificates.crt
          readOnly: true
      volumes:
      - name: ssl-certs
        hostPath:
          path: /etc/ssl/certs/ca-certificates.crt
```

**Note importante :** Cette configuration est pour AWS. Adaptez les paramètres `--cloud-provider` et `--node-group-auto-discovery` selon votre infrastructure.

## Configuration du Cluster Autoscaler

### Paramètres essentiels

**1. Cloud Provider**

```bash
--cloud-provider=aws    # ou gce, azure, etc.
```

Indique au CA comment interagir avec votre infrastructure.

**2. Node Groups Auto-Discovery**

**Pour AWS :**
```bash
--node-group-auto-discovery=asg:tag=k8s.io/cluster-autoscaler/enabled,k8s.io/cluster-autoscaler/mon-cluster
```

Le CA découvre automatiquement les Auto Scaling Groups tagués correctement.

**Pour GCP :**
```bash
--node-group-auto-discovery=mig:name=mig-prefix-[0-9]+
```

**3. Limites de scaling**

```bash
--nodes=1:10:node-group-name    # Min:Max:Name
```

Définit le nombre minimum et maximum de nœuds par groupe.

**Exemple complet :**
```bash
--nodes=2:10:worker-nodes-a
--nodes=1:5:worker-nodes-b
```

Cela signifie :
- Groupe "worker-nodes-a" : Entre 2 et 10 nœuds
- Groupe "worker-nodes-b" : Entre 1 et 5 nœuds

### Paramètres de comportement

**Scale Up :**

```bash
--max-node-provision-time=15m        # Temps max pour provisionner un nœud
--max-total-unready-percentage=33    # Max de nœuds non-ready tolérés
--ok-total-unready-count=3           # Nombre absolu de nœuds non-ready OK
```

**Scale Down :**

```bash
--scale-down-enabled=true                     # Activer le scale down
--scale-down-delay-after-add=10m              # Attendre 10min après un scale up
--scale-down-delay-after-delete=10s           # Attendre 10s après un scale down
--scale-down-delay-after-failure=3m           # Attendre 3min après un échec
--scale-down-unneeded-time=10m                # Nœud sous-utilisé pendant 10min
--scale-down-utilization-threshold=0.5        # Seuil d'utilisation (50%)
```

**Expander (Stratégie de choix de nœud) :**

```bash
--expander=least-waste    # Stratégie de sélection
```

Stratégies disponibles :
- **random** : Choisir aléatoirement
- **most-pods** : Groupe qui peut héberger le plus de pods
- **least-waste** : Groupe qui minimise le gaspillage de ressources (recommandé)
- **price** : Le moins cher (nécessite configuration prix)
- **priority** : Basé sur des priorités configurées

### Configuration via ConfigMap

Vous pouvez aussi utiliser un ConfigMap pour certains paramètres :

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: cluster-autoscaler-status
  namespace: kube-system
data:
  status: |
    Cluster Autoscaler Status:
    - Nodes: 3
    - Min: 2
    - Max: 10
    - Last scale up: 2025-10-25 10:30:00
    - Last scale down: 2025-10-25 09:15:00
```

## Interaction avec le HPA

Le Cluster Autoscaler et le HPA fonctionnent **ensemble** de manière complémentaire :

### Scénario typique : Pic de trafic

**Étape 1 : Le trafic augmente**
```
État initial :
- 3 nœuds (chacun 4 CPU)
- Application : 2 pods (200m CPU chacun)
- HPA configuré : min=2, max=20
```

**Étape 2 : HPA détecte la charge élevée**
```
- Utilisation CPU > 70%
- HPA décide de scaler à 15 pods
- HPA crée des nouveaux pods
```

**Étape 3 : Certains pods restent Pending**
```
- 10 pods Running (ressources suffisantes sur les 3 nœuds)
- 5 pods Pending (pas assez de CPU disponible)
```

**Étape 4 : Cluster Autoscaler intervient**
```
- CA détecte les 5 pods Pending
- CA calcule qu'il faut 2 nœuds supplémentaires
- CA demande 2 nouveaux nœuds à AWS
```

**Étape 5 : Nœuds provisionnés**
```
- Après 3-5 minutes, les 2 nouveaux nœuds sont prêts
- Total : 5 nœuds maintenant
- Les 5 pods Pending sont programmés sur les nouveaux nœuds
- Tous les pods sont Running !
```

**Étape 6 : Le trafic diminue (plus tard)**
```
- HPA réduit à 3 pods (charge faible)
- Les 5 nœuds sont maintenant sous-utilisés
```

**Étape 7 : Cluster Autoscaler nettoie**
```
- Après 10 minutes, CA détecte 2 nœuds quasi-vides
- CA évacue les quelques pods restants
- CA supprime les 2 nœuds inutiles
- Retour à 3 nœuds
```

### Flux décisionnel combiné

```
Augmentation de charge
    ↓
HPA ajoute des pods
    ↓
Pods en Pending ?
    ↓ Oui
Cluster Autoscaler ajoute des nœuds
    ↓
Pods programmés sur nouveaux nœuds
    ↓
[Temps passe, charge diminue]
    ↓
HPA retire des pods
    ↓
Nœuds sous-utilisés ?
    ↓ Oui
Cluster Autoscaler retire des nœuds
```

**C'est une danse coordonnée !**

## Annoter les pods pour contrôler le comportement

Vous pouvez utiliser des annotations pour contrôler le comportement du Cluster Autoscaler avec certains pods :

### Empêcher l'éviction d'un pod

Pour les pods critiques que vous ne voulez **jamais** voir évacués :

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: critical-app
  annotations:
    cluster-autoscaler.kubernetes.io/safe-to-evict: "false"
spec:
  containers:
  - name: app
    image: mon-app:latest
```

**Cas d'usage :**
- Pods avec state local important
- Applications avec startup très long
- Pods de monitoring critiques

### Autoriser l'éviction malgré certaines conditions

Certains pods sont normalement "non-évacuables" mais vous pouvez forcer :

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: tolerable-app
  annotations:
    cluster-autoscaler.kubernetes.io/safe-to-evict: "true"
spec:
  containers:
  - name: app
    image: mon-app:latest
```

## Monitoring du Cluster Autoscaler

### Logs du Cluster Autoscaler

```bash
kubectl logs -n kube-system deployment/cluster-autoscaler -f
```

**Messages typiques :**

**Scale up :**
```
Scale-up: group worker-nodes-a size set to 5
Pod default/nginx-web-xxxxx is unschedulable
```

**Scale down :**
```
Scale-down: node node-worker-3 is unneeded
Scale-down: removing node node-worker-3
```

**Échec :**
```
Failed to increase node group size: request failed
```

### Métriques Prometheus

Le Cluster Autoscaler expose des métriques pour Prometheus :

```yaml
# Métriques utiles
cluster_autoscaler_nodes_count              # Nombre de nœuds
cluster_autoscaler_unschedulable_pods_count # Pods non schedulables
cluster_autoscaler_scaled_up_nodes_total    # Total de scale ups
cluster_autoscaler_scaled_down_nodes_total  # Total de scale downs
cluster_autoscaler_failed_scale_ups_total   # Échecs de scale up
```

### Commandes utiles

```bash
# Voir l'état des nœuds
kubectl get nodes

# Voir les pods en Pending
kubectl get pods --all-namespaces --field-selector=status.phase=Pending

# Voir les événements du cluster
kubectl get events --sort-by=.metadata.creationTimestamp

# Détails d'un nœud
kubectl describe node <node-name>

# Voir l'utilisation des ressources
kubectl top nodes
```

### Dashboard Kubernetes

Le Dashboard Kubernetes (si activé) montre visuellement :
- Le nombre de nœuds
- Les pods en Pending
- L'utilisation des ressources par nœud

```bash
microk8s enable dashboard
```

## Cas d'usage du Cluster Autoscaler

### 1. Applications avec trafic variable dans le temps

**Exemple :** Site e-commerce avec pics pendant les soldes

```yaml
# HPA pour les pods
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: boutique-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: boutique
  minReplicas: 5
  maxReplicas: 100
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 60

---
# Cluster Autoscaler configuré pour 2-20 nœuds
# Permet de gérer jusqu'à 100 pods si nécessaire
```

### 2. Batch processing avec charges imprévisibles

**Exemple :** Traitement d'images uploadées par les utilisateurs

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: image-processor
spec:
  parallelism: 50    # Jusqu'à 50 pods en parallèle
  completions: 1000  # 1000 images à traiter
  template:
    spec:
      containers:
      - name: processor
        image: image-processor:latest
        resources:
          requests:
            cpu: "500m"
            memory: "1Gi"
```

Si le cluster n'a pas assez de ressources pour 50 pods, le CA ajoutera des nœuds.

### 3. Environnements de dev/test coûteux

**Stratégie :** Scale down agressif la nuit et les week-ends

```bash
# Configuration CA pour dev
--scale-down-delay-after-add=5m         # Scale down rapide
--scale-down-unneeded-time=5m           # Nœud inutile après 5min
--scale-down-utilization-threshold=0.3  # Seuil bas (30%)
```

Combiné avec un script pour scaler le HPA à 0 la nuit :

```bash
# Cron job pour scaler down à 22h
0 22 * * * kubectl scale deployment --all --replicas=0

# Cron job pour scaler up à 8h
0 8 * * * kubectl scale deployment --all --replicas=3
```

### 4. Multi-tenant avec isolation

**Exemple :** Plateforme SaaS avec clients ayant des nœuds dédiés

```yaml
# Client A : Nœuds dédiés
nodeSelector:
  customer: client-a

# Configuration CA
--nodes=1:5:client-a-nodes
--nodes=1:5:client-b-nodes
```

Chaque client a son propre pool de nœuds qui scale indépendamment.

## Limitations et pièges à éviter

### 1. Temps de provisionnement

**Problème :** Les nœuds prennent du temps à démarrer (3-5 minutes en cloud, plus en bare metal).

**Impact :** Pendant ce temps, les pods restent Pending.

**Mitigation :**
- Gardez toujours une marge de capacité (min replicas + buffer)
- Utilisez des readiness probes rapides
- Considérez des stratégies de pre-scaling pour des événements prévus

### 2. Coûts imprévus

**Problème :** Le CA peut ajouter beaucoup de nœuds si mal configuré, augmentant drastiquement les coûts.

**Mitigation :**
- Définissez toujours `maxReplicas` pour le HPA
- Définissez toujours `max nodes` pour chaque node group
- Configurez des alertes sur le nombre de nœuds et les coûts
- Testez en environnement de dev d'abord !

### 3. Pods non évacuables

**Problème :** Le CA ne peut pas scale down si des pods ne peuvent pas être déplacés.

**Exemples de pods non évacuables :**
- Pods avec `local storage` (volumes hostPath, emptyDir)
- Pods sans controller (pods "nus" non gérés par Deployment/StatefulSet)
- Pods avec annotation `safe-to-evict: false`
- DaemonSets (par défaut)

**Mitigation :**
- Utilisez des PersistentVolumes au lieu de local storage
- Toujours créer des pods via des Deployments
- Utilisez l'annotation `safe-to-evict` avec parcimonie

### 4. Scale down trop agressif

**Problème :** Le CA retire des nœuds trop vite, puis doit les recréer peu après (thrashing).

**Mitigation :**
```bash
--scale-down-delay-after-add=10m       # Attendre 10min après scale up
--scale-down-unneeded-time=10m         # Observer pendant 10min
--scale-down-utilization-threshold=0.5 # Pas trop bas
```

### 5. Conflits avec mises à jour de cluster

**Problème :** Pendant une mise à jour de cluster (upgrade K8s), le CA peut interférer.

**Solution :**
Désactivez temporairement le CA pendant les maintenances :

```bash
kubectl scale deployment cluster-autoscaler -n kube-system --replicas=0

# Faire la maintenance...

kubectl scale deployment cluster-autoscaler -n kube-system --replicas=1
```

### 6. Limitations avec MicroK8s on-premise

**Rappel important :**

Sur un homelab avec MicroK8s sur des machines physiques ou des VMs locales, le Cluster Autoscaler ne peut pas créer automatiquement de nouvelles machines !

**Alternatives pour homelab :**
1. Utiliser le HPA uniquement (scaling horizontal des pods)
2. Dimensionner le cluster avec suffisamment de nœuds fixes
3. Ajouter manuellement des nœuds quand nécessaire
4. Utiliser le VPA pour optimiser les ressources par pod

## Bonnes pratiques

### 1. Commencez avec des limites conservatrices

```bash
--nodes=2:5:worker-nodes    # Commencez petit
```

Augmentez progressivement après observation.

### 2. Utilisez des PodDisruptionBudgets

Protégez vos applications critiques :

```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: mon-app-pdb
spec:
  minAvailable: 2
  selector:
    matchLabels:
      app: mon-app
```

### 3. Taggez correctement vos node groups

Pour AWS :

```
Tag: k8s.io/cluster-autoscaler/enabled = true
Tag: k8s.io/cluster-autoscaler/mon-cluster = owned
```

Sans ces tags, le CA ne peut pas découvrir les groupes.

### 4. Surveillez les métriques

Configurez des alertes sur :
- Nombre de nœuds
- Pods en Pending prolongé
- Échecs de scale up/down
- Coûts mensuels

### 5. Testez les scénarios de scale down

Vérifiez régulièrement que le scale down fonctionne :

```bash
# Créer une charge temporaire
kubectl run stress --image=stress -- stress --cpu 4 --timeout 60s

# Observer le scale up
kubectl get nodes -w

# Arrêter la charge
kubectl delete pod stress

# Observer le scale down (après 10min)
kubectl get nodes -w
```

### 6. Documentez votre configuration

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: ca-config-doc
  namespace: kube-system
data:
  notes: |
    Cluster Autoscaler Configuration
    ---------------------------------
    Provider: AWS
    Region: eu-west-1
    Min nodes: 2
    Max nodes: 10
    Scale down delay: 10m
    Last updated: 2025-10-25
    Contact: devops@example.com
```

### 7. Combinez avec le HPA intelligemment

```yaml
# HPA : Réactivité rapide (pods)
minReplicas: 3
maxReplicas: 50

# CA : Capacité élastique (nœuds)
--nodes=2:10:worker-nodes
```

Assurez-vous que `maxReplicas * resources_per_pod ≤ max_nodes * resources_per_node`

## Résumé

Le **Cluster Autoscaler** est un outil puissant pour gérer automatiquement la capacité de votre cluster Kubernetes.

**Points clés :**

1. Le CA ajoute ou retire des **nœuds** (machines) du cluster
2. Complémentaire au HPA (qui gère les pods) et VPA (qui gère les ressources)
3. Nécessite une infrastructure compatible (cloud ou automatisable)
4. **Pas pratique pour un homelab MicroK8s typique** (machines physiques)
5. Fonctionne en synergie avec le HPA pour un scaling complet
6. Les nœuds prennent du temps à provisionner (3-5 minutes minimum)
7. Le scale down est plus prudent que le scale up (par design)

**Les trois niveaux de scaling récapitulés :**

| Niveau | Outil | Ce qui change | Vitesse | Scope |
|--------|-------|---------------|---------|-------|
| **Pods** | HPA | Nombre de pods | Rapide (secondes) | Application |
| **Ressources** | VPA | CPU/RAM par pod | Lent (redémarrage) | Pod |
| **Nœuds** | CA | Nombre de machines | Très lent (minutes) | Cluster |

**Quand utiliser le Cluster Autoscaler ?**

✅ Cluster dans le cloud (AWS, GCP, Azure)
✅ Charges très variables nécessitant capacité élastique
✅ Applications avec HPA atteignant souvent les limites
✅ Optimisation automatique des coûts
✅ Infrastructure automatisable (Terraform, Cluster API)

❌ Homelab avec machines physiques fixes
❌ Cluster single-node
❌ Infrastructure non-automatisable
❌ Besoin de réactivité instantanée (utilisez HPA seul)
❌ Contraintes de budget strictes sans surveillance

**Pour un lab MicroK8s typique :**

Si vous êtes sur des machines physiques/VMs locales, concentrez-vous sur :
- **HPA** pour le scaling horizontal des pods
- **VPA** pour l'optimisation des ressources
- **Dimensionnement manuel** du cluster avec suffisamment de nœuds

Le Cluster Autoscaler est fantastique dans le cloud, mais moins pertinent pour un environnement on-premise traditionnel !

---

**Prochaine section :** 19.5 Metrics Server - La fondation de tous les autoscalers !

⏭️ [Metrics Server](/19-scaling-et-autoscaling/05-metrics-server.md)
