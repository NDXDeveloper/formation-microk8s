🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 20. Ordonnancement Avancé

## Introduction au Chapitre

L'**ordonnancement** (scheduling) est l'un des aspects les plus puissants et les plus sophistiqués de Kubernetes. C'est le processus par lequel Kubernetes décide **sur quel nœud** un Pod doit être exécuté. Bien que ce processus se fasse automatiquement et de manière transparente, comprendre et maîtriser les mécanismes d'ordonnancement vous permet de :

- **Optimiser les performances** de vos applications
- **Garantir la haute disponibilité** de vos services
- **Contrôler les coûts** en utilisant efficacement vos ressources
- **Isoler les workloads** pour la sécurité et la conformité
- **Créer des architectures sophistiquées** adaptées à vos besoins spécifiques

Dans ce chapitre, nous allons explorer en profondeur tous les mécanismes qui vous permettent d'influencer et de contrôler comment vos Pods sont placés dans votre cluster.

---

## Qu'est-ce que l'Ordonnancement ?

### Le Concept de Base

Lorsque vous créez un Pod dans Kubernetes (directement ou via un Deployment, StatefulSet, etc.), voici ce qui se passe :

```
1. Vous créez un Pod
        ↓
2. Le Pod est créé dans l'état "Pending" (en attente)
        ↓
3. Le Scheduler examine le Pod
        ↓
4. Le Scheduler évalue tous les nœuds disponibles
        ↓
5. Le Scheduler choisit le meilleur nœud
        ↓
6. Le Pod est assigné à ce nœud
        ↓
7. Le Kubelet du nœud démarre le Pod
        ↓
8. Le Pod passe à l'état "Running"
```

### Pourquoi l'Ordonnancement est Important ?

#### 1. **Performance et Latence**

Placer une application web près de son cache Redis peut réduire la latence de millisecondes à microsecondes.

```
❌ Mauvais placement :
Nœud 1: [Application Web]
Nœud 2: [Cache Redis]
└─ Latence réseau entre nœuds

✅ Bon placement :
Nœud 1: [Application Web] + [Cache Redis]
└─ Communication locale ultra-rapide
```

#### 2. **Haute Disponibilité**

Distribuer les réplicas d'une application sur différents nœuds garantit que votre service reste disponible même en cas de panne.

```
❌ Pas de contrôle :
Nœud 1: [App Replica 1, App Replica 2, App Replica 3]
└─ Si ce nœud tombe, tout le service est down

✅ Distribution contrôlée :
Nœud 1: [App Replica 1]
Nœud 2: [App Replica 2]
Nœud 3: [App Replica 3]
└─ Le service reste disponible même si un nœud tombe
```

#### 3. **Optimisation des Ressources**

Certaines applications nécessitent du matériel spécifique (GPU, SSD, beaucoup de mémoire). L'ordonnancement permet de placer ces applications sur les bons nœuds.

```
Nœud GPU: [Applications de Machine Learning]
Nœud SSD: [Bases de données]
Nœud Standard: [Applications web classiques]
```

#### 4. **Isolation et Sécurité**

Séparer les environnements de production et de développement, ou isoler les données sensibles.

```
Nœuds Production: [Apps Production uniquement]
Nœuds Dev/Test: [Apps Dev/Test uniquement]
└─ Isolation complète
```

#### 5. **Gestion des Coûts**

Dans un environnement cloud, utiliser des instances spot (moins chères mais peuvent être interrompues) pour les workloads non-critiques.

```
Instances Premium: [Applications critiques]
Instances Spot: [Jobs batch, traitements non urgents]
└─ Économies importantes
```

---

## L'Ordonnancement par Défaut

Par défaut, si vous ne spécifiez rien, le scheduler de Kubernetes fait un excellent travail :

```yaml
# Pod simple sans contraintes d'ordonnancement
apiVersion: v1
kind: Pod
metadata:
  name: mon-pod
spec:
  containers:
  - name: nginx
    image: nginx
```

Le scheduler va :
1. **Filtrer** les nœuds qui ne peuvent pas héberger le Pod (pas assez de ressources, taints, etc.)
2. **Noter** les nœuds restants selon plusieurs critères
3. **Choisir** le nœud avec le meilleur score

**Critères par défaut** :
- Équilibrage de la charge entre nœuds
- Ressources disponibles (CPU, mémoire)
- Présence de l'image Docker (pour éviter de télécharger)
- Répartition des Pods d'un même Service

Pour la plupart des applications simples, **cela suffit amplement**.

---

## Quand Faut-il Intervenir ?

Vous devriez envisager de contrôler l'ordonnancement dans ces situations :

### ✅ Vous Devriez Intervenir Si :

1. **Haute disponibilité critique** : Vos réplicas doivent être sur des nœuds différents
2. **Matériel spécifique** : Vous avez besoin de GPU, SSD, ou haute mémoire
3. **Performance importante** : Certains Pods doivent être proches les uns des autres
4. **Isolation requise** : Production vs Dev, ou multi-tenant
5. **Coûts à optimiser** : Utilisation d'instances diverses (premium, standard, spot)
6. **Conformité** : Données qui doivent rester dans certaines zones géographiques

### ❌ Vous N'Avez Pas Besoin d'Intervenir Si :

1. **Application simple** : Une application web standard sans besoins spécifiques
2. **Cluster homogène** : Tous vos nœuds sont identiques
3. **Pas d'exigences strictes** : Les performances actuelles vous conviennent
4. **Environnement de test** : Vous expérimentez simplement avec Kubernetes

---

## Les Mécanismes d'Ordonnancement Disponibles

Kubernetes offre plusieurs outils pour contrôler l'ordonnancement, du plus simple au plus sophistiqué :

### 1. **Node Selectors** (Le Plus Simple)

**Concept** : "Ce Pod doit aller sur un nœud avec ce label"

**Syntaxe** :
```yaml
nodeSelector:
  disktype: ssd
```

**Quand l'utiliser** : Besoins simples, un seul critère

---

### 2. **Node Affinity** (Plus Puissant)

**Concept** : "Ce Pod doit/préfère aller sur des nœuds qui correspondent à ces règles"

**Syntaxe** :
```yaml
affinity:
  nodeAffinity:
    requiredDuringSchedulingIgnoredDuringExecution:
      nodeSelectorTerms:
      - matchExpressions:
        - key: disktype
          operator: In
          values:
          - ssd
          - nvme
```

**Quand l'utiliser** : Règles complexes (OR, NOT), préférences

---

### 3. **Pod Affinity** (Relations Entre Pods)

**Concept** : "Ce Pod doit/préfère être proche d'autres Pods"

**Syntaxe** :
```yaml
affinity:
  podAffinity:
    requiredDuringSchedulingIgnoredDuringExecution:
    - labelSelector:
        matchLabels:
          app: cache
      topologyKey: kubernetes.io/hostname
```

**Quand l'utiliser** : Colocalisation pour performance

---

### 4. **Pod Anti-Affinity** (Séparation de Pods)

**Concept** : "Ce Pod doit/préfère être éloigné d'autres Pods"

**Syntaxe** :
```yaml
affinity:
  podAntiAffinity:
    requiredDuringSchedulingIgnoredDuringExecution:
    - labelSelector:
        matchLabels:
          app: mon-app
      topologyKey: kubernetes.io/hostname
```

**Quand l'utiliser** : Haute disponibilité, isolation

---

### 5. **Taints et Tolerations** (Restrictions d'Accès)

**Concept** : "Ce nœud est réservé/spécial, seuls certains Pods peuvent y accéder"

**Syntaxe** :
```yaml
# Sur le nœud (taint)
kubectl taint nodes node1 gpu=true:NoSchedule

# Sur le Pod (toleration)
tolerations:
- key: "gpu"
  operator: "Equal"
  value: "true"
  effect: "NoSchedule"
```

**Quand l'utiliser** : Réserver des nœuds, maintenance, isolation stricte

---

### 6. **Topology Spread Constraints** (Distribution Uniforme)

**Concept** : "Distribuer mes Pods uniformément à travers les nœuds/zones"

**Syntaxe** :
```yaml
topologySpreadConstraints:
- maxSkew: 1
  topologyKey: topology.kubernetes.io/zone
  whenUnsatisfiable: DoNotSchedule
  labelSelector:
    matchLabels:
      app: mon-app
```

**Quand l'utiliser** : Haute disponibilité, équilibrage de charge

---

## Comparaison Rapide des Mécanismes

| Mécanisme | Complexité | Flexibilité | Usage Principal |
|-----------|------------|-------------|-----------------|
| **Node Selector** | ⭐ Très simple | ⭐⭐ Limitée | Placement basique |
| **Node Affinity** | ⭐⭐ Moyenne | ⭐⭐⭐⭐ Élevée | Règles avancées sur nœuds |
| **Pod Affinity** | ⭐⭐⭐ Moyenne-Élevée | ⭐⭐⭐⭐ Élevée | Colocalisation |
| **Pod Anti-Affinity** | ⭐⭐⭐ Moyenne-Élevée | ⭐⭐⭐⭐ Élevée | Séparation, HA |
| **Taints/Tolerations** | ⭐⭐ Moyenne | ⭐⭐⭐ Moyenne | Réservation, isolation |
| **Topology Spread** | ⭐⭐⭐ Moyenne-Élevée | ⭐⭐⭐⭐⭐ Très élevée | Distribution uniforme |

---

## Direction des Contraintes

Il est important de comprendre la "direction" de chaque mécanisme :

### Pod → Nœud (Attraction)

Ces mécanismes permettent aux **Pods d'attirer** vers certains nœuds :

- **Node Selector** : "Je veux aller sur un nœud avec ce label"
- **Node Affinity** : "Je veux/préfère aller sur des nœuds qui correspondent"

### Pod ↔ Pod (Relations)

Ces mécanismes définissent des **relations entre Pods** :

- **Pod Affinity** : "Je veux être proche de ces Pods"
- **Pod Anti-Affinity** : "Je veux être loin de ces Pods"
- **Topology Spread** : "Je veux être distribué uniformément avec mes pairs"

### Nœud → Pod (Répulsion)

Ce mécanisme permet aux **nœuds de repousser** certains Pods :

- **Taints/Tolerations** : "Ce nœud repousse tous les Pods, sauf ceux qui tolèrent"

---

## Combinaison des Mécanismes

La vraie puissance vient de la **combinaison** de ces mécanismes. Vous pouvez utiliser plusieurs contraintes sur un même Pod :

```yaml
spec:
  # Node Affinity : Doit être sur un nœud SSD
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
        - matchExpressions:
          - key: disktype
            operator: In
            values:
            - ssd
    # Pod Affinity : Préfère être avec Redis
    podAffinity:
      preferredDuringSchedulingIgnoredDuringExecution:
      - weight: 100
        podAffinityTerm:
          labelSelector:
            matchLabels:
              app: redis
          topologyKey: kubernetes.io/hostname
    # Pod Anti-Affinity : Réplicas sur nœuds différents
    podAntiAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
      - labelSelector:
          matchLabels:
            app: mon-app
        topologyKey: kubernetes.io/hostname
  # Topology Spread : Distribution par zone
  topologySpreadConstraints:
  - maxSkew: 1
    topologyKey: topology.kubernetes.io/zone
    whenUnsatisfiable: DoNotSchedule
    labelSelector:
      matchLabels:
        app: mon-app
  # Tolerations : Accès à des nœuds spéciaux
  tolerations:
  - key: "dedicated"
    operator: "Equal"
    value: "database"
    effect: "NoSchedule"
```

---

## Architecture de ce Chapitre

Ce chapitre est organisé de manière progressive, du plus simple au plus complexe :

### **20.1 Scheduler Kubernetes**
Comprendre le cerveau de l'ordonnancement : comment fonctionne le scheduler, son processus de décision, et pourquoi il fait certains choix.

### **20.2 Node Selectors**
Le mécanisme le plus simple pour diriger vos Pods vers des nœuds spécifiques en utilisant des labels.

### **20.3 Node Affinity et Anti-Affinity**
Version évoluée des Node Selectors avec des règles sophistiquées (OR, NOT, préférences).

### **20.4 Pod Affinity et Anti-Affinity**
Contrôler le placement des Pods **les uns par rapport aux autres** : colocalisation ou séparation.

### **20.5 Taints et Tolerations**
Marquer des nœuds comme "spéciaux" et contrôler qui peut y accéder : réservation et isolation.

### **20.6 Topology Spread Constraints**
Le mécanisme moderne pour distribuer uniformément vos Pods à travers votre infrastructure.

### **20.7 Cas d'Usage Pratiques**
Exemples concrets et complets qui combinent tous les mécanismes pour créer des architectures sophistiquées.

---

## Prérequis pour ce Chapitre

Pour tirer le meilleur parti de ce chapitre, vous devriez avoir :

### Connaissances Requises

✅ **Concepts de base Kubernetes** :
- Pods, Deployments, Services
- Namespaces
- Labels et Selectors

✅ **Commandes kubectl essentielles** :
- `kubectl get`, `kubectl describe`
- `kubectl apply`, `kubectl delete`
- `kubectl logs`

✅ **Architecture Kubernetes de base** :
- Control plane vs Worker nodes
- Kubelet, Scheduler, API Server

### Environnement

✅ **Cluster MicroK8s fonctionnel**
- De préférence avec plusieurs nœuds (pour tester la distribution)
- Si vous n'avez qu'un seul nœud, vous pourrez quand même apprendre les concepts

✅ **Accès kubectl configuré**

### Pour les Sections Avancées (optionnel)

Pour expérimenter pleinement avec les mécanismes avancés, idéalement :

- **Cluster multi-node** (3+ nœuds) pour tester la distribution
- **Labels de topologie** configurés (zones, régions)
- **Plusieurs types de nœuds** (si possible) pour tester les affinités

Mais ne vous inquiétez pas : même avec un cluster simple, vous pourrez comprendre tous les concepts !

---

## Approche Pédagogique de ce Chapitre

### 1. **Théorie Avant Pratique**

Chaque section commence par expliquer **pourquoi** et **comment** fonctionne le mécanisme, avec des analogies et des schémas.

### 2. **Exemples Progressifs**

Nous partons d'exemples simples et construisons progressivement vers des configurations complexes.

### 3. **Cas d'Usage Réels**

Chaque mécanisme est illustré avec des cas d'usage concrets que vous rencontrerez dans la vraie vie.

### 4. **Bonnes Pratiques**

Nous partageons les bonnes pratiques et les pièges à éviter, basés sur l'expérience.

### 5. **Comparaisons**

Nous comparons les différents mécanismes pour vous aider à choisir le bon outil pour chaque situation.

---

## Vocabulaire Important

Avant de commencer, voici les termes clés que vous rencontrerez :

- **Scheduler** : Le composant qui décide où placer les Pods
- **Node** : Une machine (physique ou virtuelle) dans le cluster
- **Label** : Une paire clé-valeur attachée à un objet (nœud, Pod, etc.)
- **Selector** : Un filtre basé sur les labels
- **Affinity** : Attirance (rapprochement)
- **Anti-Affinity** : Répulsion (éloignement)
- **Taint** : Une "souillure" sur un nœud qui repousse les Pods
- **Toleration** : Une tolérance qui permet à un Pod d'aller sur un nœud avec taint
- **Topology** : La structure de votre infrastructure (nœuds, zones, régions)
- **Skew** : L'écart entre différents domaines de topologie
- **Pending** : État d'un Pod qui attend d'être ordonnancé
- **Eviction** : Expulsion d'un Pod d'un nœud

---

## Principes Fondamentaux

Avant de plonger dans les détails, voici les principes fondamentaux à garder en tête :

### 1. **L'Ordonnancement est une Décision Ponctuelle**

Une fois qu'un Pod est placé sur un nœud et qu'il démarre, il y reste. Le scheduler ne déplace **pas** les Pods déjà en cours d'exécution (sauf cas particuliers avec taints `NoExecute`).

### 2. **Les Contraintes sont Évaluées au Moment de l'Ordonnancement**

Les règles d'affinity, node affinity, etc. sont vérifiées uniquement quand le Pod est créé, pas après.

### 3. **Plusieurs Contraintes = ET Logique**

Si vous spécifiez plusieurs contraintes, elles doivent **toutes** être satisfaites simultanément.

### 4. **Le Scheduler Cherche le Meilleur Compromis**

Avec des contraintes "preferred" (souples), le scheduler essaie de les respecter au mieux, mais ne bloque pas le déploiement.

### 5. **Préférez la Simplicité**

N'ajoutez des contraintes que si vous en avez vraiment besoin. Plus c'est complexe, plus c'est difficile à déboguer.

---

## Philosophie de l'Ordonnancement

### Le Bon Équilibre

L'ordonnancement est un équilibre entre plusieurs objectifs parfois contradictoires :

```
Performance       ←→  Coût
    ↕                  ↕
Simplicité       ←→  Contrôle
    ↕                  ↕
Flexibilité      ←→  Garanties strictes
```

**Exemples de compromis** :

- **Performance vs Coût** : Utiliser des nœuds GPU (performants) coûte cher
- **Contrôle vs Simplicité** : Plus vous contrôlez, plus c'est complexe
- **Garanties vs Flexibilité** : Des contraintes strictes peuvent bloquer des déploiements

### Notre Recommandation

1. **Commencez simple** : Laissez le scheduler faire son travail
2. **Ajoutez des contraintes progressivement** : Uniquement quand nécessaire
3. **Mesurez l'impact** : Vérifiez que vos contraintes améliorent vraiment les choses
4. **Documentez** : Expliquez pourquoi vous avez fait certains choix

---

## Comment Utiliser ce Chapitre

### Pour les Débutants

Si vous découvrez l'ordonnancement :

1. Lisez **20.1 Scheduler** pour comprendre les bases
2. Commencez par **20.2 Node Selectors** (le plus simple)
3. Passez à **20.6 Topology Spread Constraints** (moderne et simple)
4. Explorez les autres sections selon vos besoins

### Pour les Utilisateurs Intermédiaires

Si vous connaissez déjà les bases :

1. Survolez **20.1** et **20.2**
2. Concentrez-vous sur **20.3 à 20.6** pour les mécanismes avancés
3. Étudiez **20.7** pour les architectures complètes

### Pour les Experts

Si vous maîtrisez déjà les concepts :

1. Allez directement à **20.7 Cas d'Usage Pratiques**
2. Utilisez les autres sections comme référence
3. Adaptez les exemples à vos besoins spécifiques

---

## À Retenir

1. **L'ordonnancement est automatique** par défaut et fonctionne bien pour la majorité des cas
2. **Vous pouvez l'influencer** avec différents mécanismes selon vos besoins
3. **Commencez simple** et complexifiez uniquement si nécessaire
4. **Testez vos contraintes** avant de les déployer en production
5. **Documentez vos choix** pour votre équipe et votre futur vous-même

---

## Prêt à Commencer ?

Maintenant que vous avez une vue d'ensemble de l'ordonnancement dans Kubernetes et de ce qui vous attend dans ce chapitre, vous êtes prêt à plonger dans les détails !

Dans la prochaine section (**20.1 Scheduler Kubernetes**), nous allons découvrir le fonctionnement interne du scheduler : comment il prend ses décisions, quels critères il utilise, et comment il choisit le meilleur nœud pour chaque Pod.

---

**Bonne lecture et bon apprentissage !** 🚀

L'ordonnancement est l'une des fonctionnalités les plus élégantes de Kubernetes. En la maîtrisant, vous pourrez créer des architectures sophistiquées, performantes et résilientes qui tirent pleinement parti de la puissance de l'orchestration de conteneurs.

⏭️ [Scheduler Kubernetes](/20-ordonnancement-avance/01-scheduler-kubernetes.md)
