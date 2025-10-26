🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 20.1 Scheduler Kubernetes

## Introduction

Le **Scheduler Kubernetes** (ordonnanceur) est l'un des composants les plus critiques du control plane de Kubernetes. Son rôle principal est de décider sur quel nœud (node) un Pod nouvellement créé doit être exécuté. Cette décision est prise en fonction de nombreux critères et contraintes, et elle a un impact direct sur les performances, la fiabilité et l'efficacité de votre cluster.

Dans cette section, nous allons explorer en profondeur le fonctionnement du scheduler, comprendre ses mécanismes et apprendre comment il prend ses décisions.

---

## Qu'est-ce que le Scheduler ?

Le **kube-scheduler** est le composant du control plane responsable de l'ordonnancement des Pods. Lorsqu'un nouveau Pod est créé (par exemple via un Deployment, un Job ou directement), il commence dans un état "Pending" (en attente) sans être assigné à un nœud spécifique.

Le scheduler surveille en permanence les Pods non assignés et, pour chacun d'eux :
1. **Identifie les nœuds candidats** qui pourraient héberger le Pod
2. **Évalue et filtre** ces nœuds selon différents critères
3. **Score et classe** les nœuds éligibles
4. **Sélectionne le nœud optimal** et assigne le Pod à ce nœud

Une fois qu'un Pod est assigné à un nœud, le **kubelet** de ce nœud prend le relais pour lancer effectivement le Pod.

---

## Pourquoi le Scheduler est-il Important ?

Le scheduler joue un rôle crucial pour plusieurs raisons :

### 1. **Optimisation des Ressources**
Il répartit intelligemment les Pods sur les nœuds disponibles pour éviter la surcharge de certains nœuds et le sous-emploi d'autres.

### 2. **Respect des Contraintes**
Il s'assure que les Pods sont placés uniquement sur des nœuds qui satisfont toutes leurs exigences (ressources CPU/mémoire, labels spécifiques, etc.).

### 3. **Haute Disponibilité**
En distribuant les Pods d'une même application sur différents nœuds, il améliore la résilience du système.

### 4. **Performance**
Un bon placement des Pods peut améliorer les performances en tenant compte de la localité des données, de la latence réseau, etc.

---

## Le Processus d'Ordonnancement : Étape par Étape

Comprendre comment le scheduler fonctionne vous aidera à mieux configurer vos Pods et à résoudre les problèmes d'ordonnancement.

### Étape 1 : Surveillance des Pods Non Assignés

Le scheduler surveille continuellement l'API Server pour détecter les nouveaux Pods qui n'ont pas encore été assignés à un nœud (Pods avec `spec.nodeName` vide).

### Étape 2 : Phase de Filtrage (Predicate Phase)

Pour chaque Pod en attente, le scheduler commence par **filtrer** les nœuds disponibles pour ne garder que ceux qui sont **éligibles**. Un nœud est éligible s'il passe tous les "predicates" (tests de faisabilité).

#### Exemples de Predicates Courants

1. **PodFitsResources**
   - Vérifie que le nœud dispose de suffisamment de CPU et de mémoire pour satisfaire les `requests` du Pod

2. **PodFitsHostPorts**
   - Vérifie qu'aucun autre Pod n'utilise déjà les ports host requis

3. **NodeSelector**
   - Vérifie que le nœud possède tous les labels spécifiés dans le `nodeSelector` du Pod

4. **MatchNodeAffinity**
   - Évalue les règles d'affinité avec les nœuds si elles sont définies

5. **TaintToleration**
   - Vérifie que le Pod tolère tous les taints présents sur le nœud

6. **PodToleratesNodeTaints**
   - S'assure que le Pod peut être planifié sur un nœud avec des taints

7. **CheckVolumeBinding**
   - Vérifie que les volumes demandés peuvent être provisionnés et attachés sur ce nœud

8. **NoDiskConflict**
   - S'assure qu'il n'y a pas de conflit avec les volumes existants

### Étape 3 : Phase de Scoring (Priority Phase)

Une fois que le scheduler a filtré les nœuds éligibles, il passe à la phase de **scoring** (notation). Chaque nœud restant reçoit un score basé sur plusieurs fonctions de priorité.

#### Exemples de Fonctions de Scoring

1. **LeastRequestedPriority**
   - Favorise les nœuds ayant le plus de ressources disponibles (CPU/mémoire)
   - Encourage une répartition équilibrée de la charge

2. **BalancedResourceAllocation**
   - Préfère les nœuds où les ressources CPU et mémoire sont utilisées de manière équilibrée
   - Évite qu'un nœud soit surchargé en CPU mais sous-utilisé en mémoire (ou vice versa)

3. **NodeAffinityPriority**
   - Augmente le score des nœuds qui correspondent mieux aux préférences d'affinité

4. **InterPodAffinityPriority**
   - Favorise ou défavorise les nœuds en fonction de la présence d'autres Pods

5. **ImageLocalityPriority**
   - Favorise les nœuds qui ont déjà téléchargé l'image Docker du Pod
   - Réduit le temps de démarrage

6. **TaintTolerationPriority**
   - Donne un score en fonction de la tolérance aux taints

### Étape 4 : Sélection du Nœud

Le scheduler sélectionne le nœud ayant obtenu le **score le plus élevé**. En cas d'égalité, il choisit de manière pseudo-aléatoire parmi les nœuds ayant le meilleur score.

### Étape 5 : Binding

Une fois le nœud sélectionné, le scheduler effectue une opération appelée **binding** :
- Il met à jour le Pod dans l'API Server en définissant le champ `spec.nodeName` avec le nom du nœud sélectionné
- Le kubelet du nœud désigné détecte ce nouveau Pod et commence à le lancer

---

## Ressources et Requests : Impact sur l'Ordonnancement

Les spécifications de ressources dans vos Pods influencent directement les décisions du scheduler.

### Requests vs Limits

```yaml
resources:
  requests:
    cpu: "250m"        # Le scheduler utilise cette valeur
    memory: "512Mi"    # Le scheduler utilise cette valeur
  limits:
    cpu: "500m"        # Utilisé par le kubelet, pas le scheduler
    memory: "1Gi"      # Utilisé par le kubelet, pas le scheduler
```

**Point Important** : Le scheduler utilise uniquement les **requests** pour ses décisions, pas les **limits**.

- Les **requests** représentent les ressources garanties dont le Pod a besoin
- Le scheduler s'assure qu'un nœud a suffisamment de ressources disponibles pour honorer toutes les requests des Pods qui y tournent

### Conséquences d'une Mauvaise Configuration

#### Requests trop Faibles
- Le scheduler peut placer trop de Pods sur un même nœud
- Risque de contention des ressources et de dégradation des performances

#### Requests trop Élevées
- Sous-utilisation des nœuds
- Pods qui ne peuvent pas être ordonnancés même si les ressources réelles sont disponibles

#### Pas de Requests Définies
- Le scheduler considère que le Pod demande 0 ressource
- Tous les nœuds sont potentiellement éligibles, même ceux déjà surchargés

---

## Que se Passe-t-il Quand un Pod ne Peut Pas Être Ordonnancé ?

Si le scheduler ne trouve aucun nœud éligible pour un Pod, le Pod reste dans l'état **Pending**. Vous pouvez investiguer avec :

```bash
kubectl get pods
```

Résultat typique :
```
NAME                    READY   STATUS    RESTARTS   AGE
mon-app-xyz123-abc      0/1     Pending   0          2m
```

Pour comprendre pourquoi :

```bash
kubectl describe pod mon-app-xyz123-abc
```

Dans la section **Events**, vous verrez des messages explicatifs :

```
Events:
  Type     Reason            Age   Message
  ----     ------            ----  -------
  Warning  FailedScheduling  1m    0/3 nodes are available:
                                   3 Insufficient cpu.
```

### Raisons Courantes de Non-Ordonnancement

1. **Ressources Insuffisantes**
   - Aucun nœud n'a assez de CPU ou mémoire disponible
   - Solution : Ajouter des nœuds ou réduire les requests

2. **Taints Non Tolérés**
   - Les nœuds ont des taints que le Pod ne tolère pas
   - Solution : Ajouter des tolerations au Pod ou retirer les taints

3. **NodeSelector Non Satisfait**
   - Aucun nœud ne possède les labels requis
   - Solution : Labelliser des nœuds ou modifier le nodeSelector

4. **Affinités Non Respectées**
   - Les règles d'affinité/anti-affinité ne peuvent être satisfaites
   - Solution : Revoir les règles ou ajouter des nœuds

5. **Volumes Non Disponibles**
   - Les volumes demandés ne peuvent pas être provisionnés
   - Solution : Vérifier la configuration du stockage

---

## Le Scheduler et MicroK8s

Dans MicroK8s, le scheduler fonctionne exactement de la même manière que dans Kubernetes standard. Il est automatiquement déployé et configuré lors de l'installation de MicroK8s.

### Vérifier l'État du Scheduler

```bash
microk8s kubectl get pods -n kube-system | grep scheduler
```

Vous devriez voir quelque chose comme :
```
kube-scheduler-node1    1/1     Running   0          5d
```

### Logs du Scheduler

Si vous avez besoin de déboguer des problèmes d'ordonnancement, vous pouvez consulter les logs du scheduler :

```bash
microk8s kubectl logs -n kube-system kube-scheduler-<nom-du-pod>
```

---

## Scheduler par Défaut vs Schedulers Personnalisés

### Scheduler par Défaut

Kubernetes utilise un scheduler par défaut qui convient à la plupart des cas d'usage. Il est déjà optimisé et bien testé.

### Schedulers Personnalisés

Pour des besoins très spécifiques, vous pouvez :
- Créer votre propre scheduler personnalisé
- Configurer un Pod pour utiliser un scheduler spécifique via `spec.schedulerName`

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: mon-pod
spec:
  schedulerName: mon-scheduler-custom
  containers:
  - name: nginx
    image: nginx
```

**Note** : Dans un contexte d'apprentissage avec MicroK8s, le scheduler par défaut est largement suffisant.

---

## Points Clés à Retenir

1. **Le scheduler est automatique** : Vous n'avez généralement rien à configurer, il fonctionne en arrière-plan

2. **Les requests sont cruciales** : Toujours définir des requests réalistes pour CPU et mémoire

3. **Phase de filtrage puis scoring** : Le scheduler élimine d'abord les nœuds non éligibles, puis choisit le meilleur parmi ceux qui restent

4. **Pending = problème d'ordonnancement** : Utilisez `kubectl describe pod` pour comprendre pourquoi

5. **Influence via les contraintes** : Vous pouvez guider les décisions du scheduler via nodeSelector, affinities, taints/tolerations (que nous verrons dans les sections suivantes)

---

## Bonnes Pratiques

### 1. Toujours Définir des Requests
Ne laissez jamais vos Pods sans `requests`. Cela permet au scheduler de prendre des décisions éclairées.

### 2. Requests Réalistes
- Basez-vous sur des mesures réelles de consommation
- N'exagérez pas : cela gaspille des ressources
- N'minimisez pas : cela cause des problèmes de performance

### 3. Monitorer les Pods Pending
- Surveillez régulièrement les Pods qui ne peuvent pas être ordonnancés
- Analysez les causes et agissez rapidement

### 4. Comprendre la Capacité du Cluster
- Connaissez les ressources totales de votre cluster
- Planifiez l'ajout de nœuds avant d'atteindre la saturation

### 5. Utiliser les Contraintes avec Parcimonie
- Trop de contraintes (affinités, nodeSelectors) peuvent compliquer l'ordonnancement
- N'ajoutez des contraintes que quand c'est vraiment nécessaire

---

## Conclusion

Le scheduler Kubernetes est un composant sophistiqué mais transparent pour l'utilisateur dans la plupart des cas. Il prend automatiquement les meilleures décisions possibles en fonction des ressources disponibles et des contraintes définies.

En comprenant son fonctionnement, vous êtes mieux équipé pour :
- Configurer vos Pods de manière optimale
- Diagnostiquer les problèmes d'ordonnancement
- Utiliser les mécanismes avancés comme les affinités et les taints (sections suivantes)

Dans les prochaines sections, nous découvrirons comment influencer les décisions du scheduler grâce à des mécanismes comme les **Node Selectors**, les **Affinités** et les **Taints/Tolerations**.

---

## Pour Aller Plus Loin

- **Documentation officielle** : [Kubernetes Scheduler](https://kubernetes.io/docs/concepts/scheduling-eviction/kube-scheduler/)
- **Scheduler Policies** : Configuration avancée du comportement du scheduler
- **Scheduler Profiles** : Différents profils de scheduling pour différents besoins
- **Scheduling Framework** : Architecture extensible pour créer des plugins personnalisés

---

**Prochain chapitre** : 20.2 Node Selectors - Comment diriger vos Pods vers des nœuds spécifiques

⏭️ [Node Selectors](/20-ordonnancement-avance/02-node-selectors.md)
