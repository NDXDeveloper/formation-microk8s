🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 18.2 Resource Quotas

## Introduction

Dans le chapitre précédent (18.1), nous avons appris à gérer les ressources **au niveau des Pods individuels** avec les Requests et Limits. Maintenant, nous allons monter d'un niveau pour gérer les ressources **au niveau d'un namespace entier** avec les **Resource Quotas**.

Les Resource Quotas sont comme un **budget global** que vous allouez à un namespace. C'est un mécanisme de gouvernance qui permet de contrôler la consommation totale de ressources dans un environnement partagé.

## Pourquoi les Resource Quotas sont-ils importants ?

### Le problème sans Resource Quotas

Imaginez un cluster Kubernetes partagé par plusieurs équipes :

❌ **Sans Resource Quotas** :
- L'équipe A déploie 100 Pods et monopolise tout le cluster
- L'équipe B ne peut plus déployer ses applications
- L'équipe C crée un Pod avec des limits énormes qui bloque les autres
- Aucune prévisibilité ni équité dans l'utilisation des ressources

### La solution avec Resource Quotas

✅ **Avec Resource Quotas** :
- Chaque équipe a son namespace avec un budget défini
- L'équipe A ne peut pas dépasser son quota
- Les équipes B et C ont leurs ressources garanties
- Gestion prévisible et équitable du cluster

## Analogie pour comprendre

Pensez à un immeuble avec plusieurs appartements :

**Sans quotas** :
- Un locataire pourrait utiliser toute l'eau de l'immeuble
- Les autres seraient privés d'eau
- Facturation imprévisible

**Avec quotas** :
- Chaque appartement a un compteur d'eau individuel
- Limite mensuelle définie par appartement
- Si un locataire dépasse, il ne peut plus consommer
- Les autres restent protégés

Dans Kubernetes :
- **Immeuble** = Cluster
- **Appartements** = Namespaces
- **Compteurs** = Resource Quotas
- **Eau** = CPU, RAM, nombre de Pods, etc.

## Différence avec Requests et Limits

Il est important de comprendre la différence entre ces trois concepts :

| Concept | Niveau | Objectif |
|---------|--------|----------|
| **Requests** | Pod/Conteneur | Garantir des ressources minimales |
| **Limits** | Pod/Conteneur | Limiter la consommation maximale |
| **Resource Quotas** | Namespace | Limiter la consommation totale du namespace |

**Exemple concret** :

```
Namespace "dev-team-a" avec quota total :
├─ Quota : 4 CPU, 8Gi RAM maximum
│
├─ Pod 1
│  └─ Requests: 1 CPU, 2Gi RAM
│
├─ Pod 2
│  └─ Requests: 1 CPU, 2Gi RAM
│
├─ Pod 3
│  └─ Requests: 2 CPU, 4Gi RAM
│
└─ Tentative Pod 4 ❌
   └─ Requests: 1 CPU, 2Gi RAM
   └─ REFUSÉ : dépasserait le quota (5 CPU > 4 CPU)
```

## Types de ressources contrôlables

Les Resource Quotas peuvent limiter plusieurs types de ressources :

### 1. Ressources de calcul

**CPU et Mémoire** :
- `requests.cpu` : Total des CPU requests dans le namespace
- `requests.memory` : Total de la RAM requests dans le namespace
- `limits.cpu` : Total des CPU limits dans le namespace
- `limits.memory` : Total de la RAM limits dans le namespace

### 2. Nombre d'objets

**Comptage d'objets Kubernetes** :
- `pods` : Nombre total de Pods
- `services` : Nombre de Services
- `persistentvolumeclaims` : Nombre de PVCs
- `configmaps` : Nombre de ConfigMaps
- `secrets` : Nombre de Secrets
- `replicationcontrollers` : Nombre de ReplicationControllers
- `deployments.apps` : Nombre de Deployments
- `statefulsets.apps` : Nombre de StatefulSets

### 3. Stockage

**Volume persistants** :
- `requests.storage` : Total de stockage demandé
- `persistentvolumeclaims` : Nombre de PVCs

### 4. Ressources étendues

**Ressources spécifiques** :
- `requests.nvidia.com/gpu` : Nombre de GPUs
- Services de type particulier (LoadBalancer, NodePort)

## Configuration de base

### Créer un Resource Quota simple

Voici un exemple de quota basique qui limite les ressources de calcul :

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: compute-quota
  namespace: dev-team
spec:
  hard:
    requests.cpu: "4"           # Max 4 CPU en requests
    requests.memory: "8Gi"      # Max 8 Gi RAM en requests
    limits.cpu: "8"             # Max 8 CPU en limits
    limits.memory: "16Gi"       # Max 16 Gi RAM en limits
```

**Appliquer le quota** :

```bash
kubectl apply -f compute-quota.yaml
```

### Vérifier le quota

```bash
kubectl get resourcequota -n dev-team
```

Résultat :
```
NAME            AGE   REQUEST                                      LIMIT
compute-quota   5m    requests.cpu: 0/4, requests.memory: 0/8Gi   limits.cpu: 0/8, limits.memory: 0/16Gi
```

La notation `0/4` signifie : **0 utilisé sur 4 disponibles**.

### Description détaillée

```bash
kubectl describe resourcequota compute-quota -n dev-team
```

Résultat :
```
Name:            compute-quota
Namespace:       dev-team
Resource         Used  Hard
--------         ----  ----
limits.cpu       0     8
limits.memory    0     16Gi
requests.cpu     0     4
requests.memory  0     8Gi
```

## Exemples pratiques

### Exemple 1 : Quota pour environnement de développement

Environnement dev avec ressources limitées :

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: dev-quota
  namespace: development
spec:
  hard:
    # Ressources de calcul
    requests.cpu: "2"
    requests.memory: "4Gi"
    limits.cpu: "4"
    limits.memory: "8Gi"

    # Nombre d'objets
    pods: "10"
    services: "5"
    persistentvolumeclaims: "3"

    # Stockage
    requests.storage: "20Gi"
```

**Caractéristiques** :
- Ressources modestes (dev seulement)
- Limitation du nombre de Pods à 10
- 20Gi de stockage total

### Exemple 2 : Quota pour environnement de production

Environnement prod avec ressources généreuses :

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: prod-quota
  namespace: production
spec:
  hard:
    # Ressources de calcul
    requests.cpu: "20"
    requests.memory: "40Gi"
    limits.cpu: "40"
    limits.memory: "80Gi"

    # Nombre d'objets
    pods: "50"
    services: "20"
    persistentvolumeclaims: "15"

    # Stockage
    requests.storage: "200Gi"
```

**Caractéristiques** :
- Ressources importantes (production critique)
- Plus de Pods autorisés
- Stockage conséquent

### Exemple 3 : Quota avec comptage d'objets uniquement

Limiter uniquement le nombre d'objets sans contrainte sur les ressources :

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: object-count-quota
  namespace: testing
spec:
  hard:
    pods: "20"
    services: "10"
    configmaps: "15"
    secrets: "20"
    persistentvolumeclaims: "5"
    deployments.apps: "10"
    statefulsets.apps: "3"
```

**Cas d'usage** : Éviter qu'une équipe ne crée trop d'objets qui encombrent le cluster.

### Exemple 4 : Quota pour services exposés

Limiter les services exposés (coûteux en production cloud) :

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: services-quota
  namespace: web-apps
spec:
  hard:
    services.loadbalancers: "3"    # Max 3 LoadBalancers
    services.nodeports: "5"        # Max 5 NodePort
```

**Cas d'usage** : En production cloud, les LoadBalancers coûtent cher. Ce quota évite les dépenses excessives.

### Exemple 5 : Quota avec priorité différente

Combiner plusieurs quotas avec scopes différents :

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: priority-high-quota
  namespace: critical-apps
spec:
  hard:
    requests.cpu: "10"
    requests.memory: "20Gi"
  scopeSelector:
    matchExpressions:
    - operator: In
      scopeName: PriorityClass
      values: ["high"]
---
apiVersion: v1
kind: ResourceQuota
metadata:
  name: priority-low-quota
  namespace: critical-apps
spec:
  hard:
    requests.cpu: "2"
    requests.memory: "4Gi"
  scopeSelector:
    matchExpressions:
    - operator: In
      scopeName: PriorityClass
      values: ["low"]
```

**Cas d'usage** : Allouer plus de ressources aux Pods haute priorité.

## Comportement avec les Pods

### Règle importante : Requests obligatoires

⚠️ **Point crucial** : Quand un namespace a un Resource Quota sur le CPU ou la mémoire, **tous les Pods doivent définir des requests et limits**.

**Sans quota** - Ce Pod fonctionne :
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: mon-pod
spec:
  containers:
  - name: nginx
    image: nginx
    # Pas de resources défini - OK
```

**Avec quota** - Ce même Pod sera **REFUSÉ** :
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: mon-pod
  namespace: dev-team  # Namespace avec quota
spec:
  containers:
  - name: nginx
    image: nginx
    # Pas de resources défini - ERREUR !
```

**Message d'erreur** :
```
Error from server (Forbidden): error when creating "pod.yaml":
pods "mon-pod" is forbidden: failed quota: compute-quota:
must specify limits.cpu,limits.memory,requests.cpu,requests.memory
```

### Solution : Toujours définir les resources

**Pod correct avec quota** :
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: mon-pod
  namespace: dev-team
spec:
  containers:
  - name: nginx
    image: nginx
    resources:
      requests:
        memory: "128Mi"
        cpu: "100m"
      limits:
        memory: "256Mi"
        cpu: "200m"
```

Ce Pod sera accepté si les resources totales du namespace ne dépassent pas le quota.

## Scénarios de dépassement de quota

### Scénario 1 : Dépassement lors de la création

```bash
# Créer un Deployment qui dépasserait le quota
kubectl apply -f big-deployment.yaml -n dev-team
```

**Résultat** :
```
Error from server (Forbidden): error when creating "big-deployment.yaml":
deployments.apps "big-app" is forbidden: exceeded quota: compute-quota,
requested: requests.cpu=5, used: requests.cpu=3, limited: requests.cpu=4
```

Le Deployment est **rejeté** car il demanderait 5 CPU alors que le quota est de 4 CPU.

### Scénario 2 : Dépassement lors du scaling

```bash
# Tenter de scaler un Deployment existant
kubectl scale deployment mon-app --replicas=10 -n dev-team
```

Si cette opération dépasserait le quota :
- Le ReplicaSet tentera de créer les nouveaux Pods
- Les Pods seront **refusés** s'ils dépassent le quota
- Certains Pods resteront en état `Pending`

**Vérification** :
```bash
kubectl get pods -n dev-team
```

Résultat :
```
NAME                      READY   STATUS    RESTARTS   AGE
mon-app-7d8f9c-abc12      1/1     Running   0          5m
mon-app-7d8f9c-def34      1/1     Running   0          5m
mon-app-7d8f9c-ghi56      1/1     Running   0          5m
```

Seulement 3 Pods au lieu de 10 car le quota est atteint.

### Scénario 3 : Vérifier l'utilisation actuelle

```bash
kubectl describe resourcequota compute-quota -n dev-team
```

```
Name:            compute-quota
Namespace:       dev-team
Resource         Used    Hard
--------         ----    ----
limits.cpu       600m    8
limits.memory    768Mi   16Gi
requests.cpu     3900m   4      ⚠️ Proche du quota !
requests.memory  7.5Gi   8Gi    ⚠️ Proche du quota !
pods             8       10
```

L'équipe est proche de la limite - il est temps d'optimiser ou de demander une augmentation.

## Multiples quotas dans un namespace

Vous pouvez avoir **plusieurs Resource Quotas** dans le même namespace pour organiser différents aspects :

```yaml
# Quota 1 : Ressources de calcul
apiVersion: v1
kind: ResourceQuota
metadata:
  name: compute-quota
  namespace: my-namespace
spec:
  hard:
    requests.cpu: "4"
    requests.memory: "8Gi"
---
# Quota 2 : Objets Kubernetes
apiVersion: v1
kind: ResourceQuota
metadata:
  name: object-quota
  namespace: my-namespace
spec:
  hard:
    pods: "20"
    services: "10"
---
# Quota 3 : Stockage
apiVersion: v1
kind: ResourceQuota
metadata:
  name: storage-quota
  namespace: my-namespace
spec:
  hard:
    requests.storage: "50Gi"
    persistentvolumeclaims: "10"
```

**Tous les quotas s'appliquent simultanément** - un Pod doit respecter tous les quotas pour être créé.

## LimitRange : Le complément des Resource Quotas

Les Resource Quotas contrôlent le **total**, mais vous voudrez peut-être aussi contrôler les **valeurs individuelles**. C'est le rôle des **LimitRanges** (voir section 18.3).

**Combinaison ResourceQuota + LimitRange** :

```yaml
# ResourceQuota : limite TOTALE du namespace
apiVersion: v1
kind: ResourceQuota
metadata:
  name: namespace-quota
  namespace: team-a
spec:
  hard:
    requests.cpu: "10"
    requests.memory: "20Gi"
---
# LimitRange : limites PAR POD
apiVersion: v1
kind: LimitRange
metadata:
  name: pod-limits
  namespace: team-a
spec:
  limits:
  - max:
      cpu: "2"           # Un Pod ne peut pas demander plus de 2 CPU
      memory: "4Gi"      # Un Pod ne peut pas demander plus de 4Gi
    type: Pod
```

**Résultat** :
- Un Pod individuel ne peut pas dépasser 2 CPU / 4Gi
- Le total de tous les Pods ne peut pas dépasser 10 CPU / 20Gi

Cela évite qu'un seul Pod monopolise tout le quota.

## Bonnes pratiques

### 1. Définir des quotas pour chaque namespace

✅ **Recommandation** : Chaque namespace devrait avoir un Resource Quota, sauf le namespace `kube-system`.

**Organisation recommandée** :

```
Cluster
├─ kube-system (pas de quota - système)
├─ development (quota léger)
├─ staging (quota moyen)
├─ production (quota généreux)
└─ monitoring (quota dédié)
```

### 2. Commencer avec des quotas généreux

✅ **Approche progressive** :

1. **Phase 1** : Définir des quotas très généreux (ne devraient jamais être atteints)
2. **Phase 2** : Monitor l'utilisation réelle pendant 2-4 semaines
3. **Phase 3** : Ajuster les quotas à +30% au-dessus du pic observé
4. **Phase 4** : Affiner régulièrement

**Exemple** :
```yaml
# Phase 1 - Quota initial généreux
spec:
  hard:
    requests.cpu: "50"    # Très large
    requests.memory: "100Gi"
```

Après observation, réduire à l'usage réel + marge :
```yaml
# Phase 3 - Quota ajusté après monitoring
spec:
  hard:
    requests.cpu: "10"     # Usage max observé: 7 CPU
    requests.memory: "20Gi" # Usage max observé: 15Gi
```

### 3. Documenter les quotas

✅ **Documentation dans les annotations** :

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: dev-quota
  namespace: development
  annotations:
    contact: "team-platform@company.com"
    purpose: "Environnement de développement - quota léger"
    last-review: "2025-10-01"
    review-frequency: "quarterly"
spec:
  hard:
    requests.cpu: "4"
    requests.memory: "8Gi"
```

### 4. Monitoring des quotas

✅ **Alertes recommandées** :

```yaml
# Exemple d'alerte Prometheus (pseudo-code)
alert: QuotaAlmostFull
expr: |
  (kube_resourcequota{type="used"} / kube_resourcequota{type="hard"}) > 0.80
annotations:
  summary: "Quota {{ $labels.resource }} at {{ $value }}% in namespace {{ $labels.namespace }}"
```

### 5. Prévoir un processus d'augmentation

✅ **Établir un workflow** :

1. L'équipe détecte que son quota est proche de la limite
2. Elle soumet une demande d'augmentation avec justification
3. L'équipe plateforme analyse l'usage réel
4. Décision : optimisation ou augmentation du quota

**Template de demande** :
```
Namespace: production
Quota actuel: 10 CPU, 20Gi RAM
Quota demandé: 15 CPU, 30Gi RAM
Justification: Nouveau microservice client-api avec 5 replicas
Usage actuel: 9.2 CPU (92%), 18Gi RAM (90%)
Deadline: 2025-11-01
```

### 6. Combiner avec LimitRange

✅ **Protection à deux niveaux** :

```yaml
# Niveau 1 : ResourceQuota (total namespace)
apiVersion: v1
kind: ResourceQuota
metadata:
  name: total-quota
  namespace: my-team
spec:
  hard:
    requests.cpu: "10"
    requests.memory: "20Gi"
---
# Niveau 2 : LimitRange (par Pod)
apiVersion: v1
kind: LimitRange
metadata:
  name: pod-limits
  namespace: my-team
spec:
  limits:
  - max:
      cpu: "2"
      memory: "4Gi"
    min:
      cpu: "100m"
      memory: "128Mi"
    type: Pod
```

Cela assure :
- Pas de Pod trop petit (min)
- Pas de Pod trop gros (max)
- Total contrôlé (quota)

### 7. Environnements différents = quotas différents

✅ **Adaptation par environnement** :

```yaml
# Développement - quota minimal
requests.cpu: "2"
requests.memory: "4Gi"
pods: "10"
---
# Staging - quota moyen
requests.cpu: "8"
requests.memory: "16Gi"
pods: "25"
---
# Production - quota généreux
requests.cpu: "30"
requests.memory: "60Gi"
pods: "100"
```

## Commandes utiles

### Lister les quotas

```bash
# Tous les quotas du cluster
kubectl get resourcequota --all-namespaces

# Quotas d'un namespace spécifique
kubectl get resourcequota -n dev-team

# Format détaillé
kubectl get resourcequota -n dev-team -o wide
```

### Voir les détails d'un quota

```bash
kubectl describe resourcequota compute-quota -n dev-team
```

### Surveiller l'utilisation

```bash
# Voir l'utilisation en temps réel
kubectl top pods -n dev-team

# Script pour voir le % d'utilisation
kubectl get resourcequota compute-quota -n dev-team -o json | \
  jq '.status.used, .status.hard'
```

### Modifier un quota existant

```bash
# Éditer directement
kubectl edit resourcequota compute-quota -n dev-team

# Ou appliquer un fichier modifié
kubectl apply -f compute-quota-updated.yaml
```

### Supprimer un quota

```bash
kubectl delete resourcequota compute-quota -n dev-team
```

⚠️ **Attention** : Supprimer un quota n'affecte pas les Pods existants, mais permet la création de nouveaux Pods sans limite.

## Diagnostic et dépannage

### Problème 1 : Pod refusé à cause du quota

**Symptôme** :
```bash
kubectl apply -f my-pod.yaml -n dev-team
# Error: exceeded quota
```

**Diagnostic** :
```bash
# 1. Vérifier le quota
kubectl describe resourcequota -n dev-team

# 2. Voir les ressources demandées par le Pod
kubectl describe -f my-pod.yaml | grep -A5 "Requests:"

# 3. Calculer si ça rentre
# Used + New Requests <= Hard ?
```

**Solutions** :
- Réduire les requests du nouveau Pod
- Supprimer des Pods existants pour libérer de l'espace
- Augmenter le quota
- Optimiser les Pods existants (réduire leurs requests)

### Problème 2 : Impossible de scaler

**Symptôme** :
```bash
kubectl scale deployment my-app --replicas=10 -n dev-team
# Certains Pods ne démarrent pas
```

**Diagnostic** :
```bash
# Voir les événements
kubectl get events -n dev-team --sort-by='.lastTimestamp'

# Vérifier les ReplicaSets
kubectl describe replicaset -n dev-team
```

**Solutions** :
- Réduire le nombre de replicas
- Optimiser les resources des Pods
- Augmenter le quota

### Problème 3 : Pod sans resources refusé

**Symptôme** :
```
Error: must specify limits.cpu,limits.memory,requests.cpu,requests.memory
```

**Cause** : Le namespace a un quota mais le Pod ne définit pas de resources.

**Solution** : Ajouter la section resources au Pod :
```yaml
resources:
  requests:
    memory: "128Mi"
    cpu: "100m"
  limits:
    memory: "256Mi"
    cpu: "200m"
```

### Problème 4 : Quota atteint mais Pods en Running

**Observation bizarre** :
- Le quota montre `used = hard` (100% utilisé)
- Mais on ne voit que quelques Pods running
- Où sont les ressources ?

**Explication** : Les ressources sont comptées même pour :
- Pods en état `Pending`
- Pods en cours de création
- Pods qui viennent d'être supprimés (courte période)

**Solution** : Attendre quelques secondes que les Pods soient complètement supprimés.

## Cas d'usage avancés

### Quotas par équipe dans un cluster partagé

**Scénario** : Un cluster partagé par 3 équipes.

```yaml
# Équipe Frontend
apiVersion: v1
kind: ResourceQuota
metadata:
  name: frontend-quota
  namespace: team-frontend
spec:
  hard:
    requests.cpu: "8"
    requests.memory: "16Gi"
    pods: "30"
---
# Équipe Backend
apiVersion: v1
kind: ResourceQuota
metadata:
  name: backend-quota
  namespace: team-backend
spec:
  hard:
    requests.cpu: "12"
    requests.memory: "24Gi"
    pods: "40"
---
# Équipe Data
apiVersion: v1
kind: ResourceQuota
metadata:
  name: data-quota
  namespace: team-data
spec:
  hard:
    requests.cpu: "20"
    requests.memory: "50Gi"
    pods: "20"
    requests.storage: "500Gi"  # Beaucoup de stockage pour data
```

### Quotas pour environnements éphémères

**Scénario** : Review apps temporaires pour chaque Pull Request.

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: review-app-quota
  namespace: review-apps
spec:
  hard:
    # Ressources limitées (env temporaires)
    requests.cpu: "1"
    requests.memory: "2Gi"

    # Durée de vie des objets
    pods: "5"
    services: "2"

    # Pas de stockage persistant
    persistentvolumeclaims: "0"
```

### Quotas avec distinction QoS

**Scénario** : Favoriser les Pods Guaranteed.

```yaml
# Quota pour Pods Guaranteed
apiVersion: v1
kind: ResourceQuota
metadata:
  name: guaranteed-quota
  namespace: prod
spec:
  hard:
    requests.cpu: "15"
    requests.memory: "30Gi"
  scopes:
  - BestEffort: false  # Exclure BestEffort
---
# Quota pour Pods BestEffort
apiVersion: v1
kind: ResourceQuota
metadata:
  name: besteffort-quota
  namespace: prod
spec:
  hard:
    pods: "5"  # Très limité
  scopes:
  - BestEffort: true
```

## Points clés à retenir

🔑 **Les essentiels** :

1. **Resource Quotas** = budget total pour un namespace
2. **Protège** les autres namespaces d'un usage excessif
3. **Obligatoire** de définir requests/limits quand un quota existe
4. **Multiple quotas** possibles dans un namespace
5. **Combiner** avec LimitRange pour contrôle complet
6. **Monitor** régulièrement l'utilisation des quotas
7. **Adapter** les quotas selon l'environnement (dev/staging/prod)
8. **Documenter** les quotas et leurs justifications

## Différences avec d'autres concepts

| Concept | Niveau | Contrôle | Exemple |
|---------|--------|----------|---------|
| **Requests/Limits** | Conteneur | Ressources individuelles | Un Pod demande 500m CPU |
| **ResourceQuota** | Namespace | Total namespace | Max 10 CPU pour tout le namespace |
| **LimitRange** | Namespace | Valeurs min/max par objet | Un Pod ne peut pas dépasser 2 CPU |
| **PriorityClass** | Pod | Ordre d'éviction | Pods critiques protégés en premier |

Tous ces mécanismes travaillent ensemble pour une gestion optimale des ressources.

## Conclusion

Les Resource Quotas sont essentiels pour :
- **Gouverner** un cluster multi-équipes
- **Prévoir** les coûts et l'utilisation
- **Protéger** les environnements critiques
- **Éviter** la monopolisation des ressources

Commencez par des quotas généreux, monitorez l'usage réel, puis ajustez progressivement. Les quotas ne doivent pas être une contrainte gênante, mais un garde-fou qui assure la stabilité du cluster.

**Prochaine étape** : Dans la section suivante (18.3), nous verrons les **LimitRanges** qui complètent les Resource Quotas en contrôlant les valeurs individuelles des ressources.

---


⏭️ [LimitRanges](/18-gestion-des-ressources/03-limitranges.md)
