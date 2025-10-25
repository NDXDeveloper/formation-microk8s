🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 18.3 LimitRanges

## Introduction

Nous avons maintenant couvert deux concepts importants :
- **18.1 - Requests et Limits** : contrôle des ressources au niveau des conteneurs
- **18.2 - Resource Quotas** : contrôle du total des ressources dans un namespace

Mais il manque une pièce au puzzle : comment **empêcher qu'un seul Pod monopolise tout le quota** d'un namespace ? C'est exactement le rôle des **LimitRanges**.

Les LimitRanges définissent des **contraintes par défaut** et des **valeurs min/max** pour les ressources au niveau individuel (Pod, conteneur, PVC).

## La trilogie complète de gestion des ressources

Voici comment ces trois concepts travaillent ensemble :

```
┌──────────────────────────────────────────────────────────┐
│                     NAMESPACE                            │
│                                                          │
│  ResourceQuota: Max total = 10 CPU, 20Gi RAM             │
│  ┌────────────────────────────────────────────────────┐  │
│  │ LimitRange: Par Pod max = 2 CPU, 4Gi RAM           │  │
│  │                                                    │  │
│  │  ┌──────────────────┐  ┌──────────────────┐        │  │
│  │  │  Pod 1           │  │  Pod 2           │        │  │
│  │  │  ──────          │  │  ──────          │        │  │
│  │  │  Requests/Limits │  │  Requests/Limits │        │  │
│  │  │  500m / 1 CPU    │  │  800m / 1.5 CPU  │        │  │
│  │  │  1Gi / 2Gi       │  │  2Gi / 3Gi       │        │  │
│  │  └──────────────────┘  └──────────────────┘        │  │
│  └────────────────────────────────────────────────────┘  │
└──────────────────────────────────────────────────────────┘
```

**Niveau 1** - Requests/Limits : Ressources de chaque conteneur
**Niveau 2** - LimitRange : Contraintes par objet (Pod, conteneur, PVC)
**Niveau 3** - ResourceQuota : Budget total du namespace

## Pourquoi les LimitRanges sont-ils nécessaires ?

### Le problème sans LimitRange

Imaginez un namespace avec un Resource Quota de **10 CPU** :

❌ **Sans LimitRange** :
```yaml
# Quelqu'un crée ce Pod
apiVersion: v1
kind: Pod
metadata:
  name: pod-gourmand
spec:
  containers:
  - name: app
    image: my-app
    resources:
      requests:
        cpu: "9"        # Monopolise 90% du quota !
        memory: "18Gi"
```

**Conséquences** :
- Ce Pod unique consomme presque tout le quota
- Les autres équipes ne peuvent plus déployer
- Une seule erreur de configuration bloque tout le namespace

### La solution avec LimitRange

✅ **Avec LimitRange** :
```yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: limit-per-pod
spec:
  limits:
  - max:
      cpu: "2"          # Maximum par Pod
      memory: "4Gi"
    type: Pod
```

**Résultat** :
- Le Pod gourmand serait **refusé** car il dépasse le max
- Impossible de monopoliser le quota accidentellement
- Protection collective du namespace

## Analogie pour comprendre

Pensez à une bibliothèque publique :

**Sans LimitRange** :
- Une personne pourrait emprunter 50 livres
- Les autres n'auraient plus rien à lire
- Même si la bibliothèque a 200 livres (quota total)

**Avec LimitRange** :
- Maximum 5 livres par personne (limit)
- Tout le monde peut emprunter équitablement
- La bibliothèque reste accessible à tous

Dans Kubernetes :
- **Bibliothèque** = Namespace
- **Total des livres** = ResourceQuota
- **Limite par personne** = LimitRange
- **Livres empruntés** = Requests/Limits des Pods

## Types de LimitRange

Un LimitRange peut contrôler trois types d'objets :

### 1. Container (Conteneur)

Contraintes appliquées à **chaque conteneur individuellement**.

```yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: container-limits
  namespace: dev
spec:
  limits:
  - type: Container
    max:
      cpu: "2"
      memory: "2Gi"
    min:
      cpu: "100m"
      memory: "128Mi"
    default:
      cpu: "500m"
      memory: "512Mi"
    defaultRequest:
      cpu: "250m"
      memory: "256Mi"
```

**Champs disponibles** :
- `max` : Valeur maximum autorisée
- `min` : Valeur minimum requise
- `default` : Valeur par défaut pour les **limits** si non spécifiées
- `defaultRequest` : Valeur par défaut pour les **requests** si non spécifiées

### 2. Pod

Contraintes appliquées au **total de tous les conteneurs d'un Pod**.

```yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: pod-limits
  namespace: dev
spec:
  limits:
  - type: Pod
    max:
      cpu: "4"
      memory: "8Gi"
    min:
      cpu: "200m"
      memory: "256Mi"
```

**Important** : C'est la **somme** des ressources de tous les conteneurs du Pod qui est vérifiée.

### 3. PersistentVolumeClaim (PVC)

Contraintes pour les demandes de stockage persistant.

```yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: storage-limits
  namespace: dev
spec:
  limits:
  - type: PersistentVolumeClaim
    max:
      storage: "10Gi"
    min:
      storage: "1Gi"
```

## Configuration complète

### Exemple 1 : LimitRange de base pour conteneurs

Configuration simple pour un environnement de développement :

```yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: dev-limits
  namespace: development
spec:
  limits:
  - type: Container
    max:
      cpu: "1"
      memory: "1Gi"
    min:
      cpu: "50m"
      memory: "64Mi"
    default:
      cpu: "500m"
      memory: "512Mi"
    defaultRequest:
      cpu: "100m"
      memory: "128Mi"
```

**Ce que cela signifie** :
- Aucun conteneur ne peut demander plus de 1 CPU / 1Gi
- Aucun conteneur ne peut demander moins de 50m CPU / 64Mi
- Si pas spécifié, limits = 500m CPU / 512Mi
- Si pas spécifié, requests = 100m CPU / 128Mi

**Appliquer** :
```bash
kubectl apply -f dev-limits.yaml
```

### Exemple 2 : LimitRange complet (Container + Pod)

Protection à deux niveaux :

```yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: multi-level-limits
  namespace: production
spec:
  limits:
  # Niveau conteneur
  - type: Container
    max:
      cpu: "2"
      memory: "4Gi"
    min:
      cpu: "100m"
      memory: "128Mi"
    default:
      cpu: "500m"
      memory: "512Mi"
    defaultRequest:
      cpu: "250m"
      memory: "256Mi"
  # Niveau Pod (somme de tous les conteneurs)
  - type: Pod
    max:
      cpu: "4"
      memory: "8Gi"
    min:
      cpu: "200m"
      memory: "256Mi"
```

**Validation** :
- Un conteneur seul : max 2 CPU / 4Gi
- Un Pod entier : max 4 CPU / 8Gi (permet 2 conteneurs de 2 CPU chacun)

### Exemple 3 : Environnement avec stockage contrôlé

Inclure des limites sur les PVC :

```yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: complete-limits
  namespace: data-team
spec:
  limits:
  # Conteneurs
  - type: Container
    max:
      cpu: "2"
      memory: "4Gi"
    defaultRequest:
      cpu: "500m"
      memory: "512Mi"

  # Pods
  - type: Pod
    max:
      cpu: "6"
      memory: "12Gi"

  # Stockage persistant
  - type: PersistentVolumeClaim
    max:
      storage: "50Gi"
    min:
      storage: "1Gi"
```

### Exemple 4 : Ratio limits/requests

Contrôler le ratio entre requests et limits :

```yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: ratio-control
  namespace: my-namespace
spec:
  limits:
  - type: Container
    maxLimitRequestRatio:
      cpu: "4"        # Limits CPU max = 4x requests
      memory: "2"     # Limits memory max = 2x requests
```

**Validation** :
- ✅ Requests: 500m, Limits: 2000m (ratio 4:1) → OK
- ❌ Requests: 500m, Limits: 3000m (ratio 6:1) → REFUSÉ
- ✅ Requests: 1Gi, Limits: 2Gi (ratio 2:1) → OK
- ❌ Requests: 1Gi, Limits: 3Gi (ratio 3:1) → REFUSÉ

## Comment ça fonctionne en pratique

### Scénario 1 : Pod sans resources définies

**Sans LimitRange** :
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-pod
spec:
  containers:
  - name: nginx
    image: nginx
    # Pas de resources - problème si ResourceQuota existe !
```

Ce Pod serait **refusé** s'il y a un ResourceQuota.

**Avec LimitRange** :
```yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: defaults
spec:
  limits:
  - type: Container
    default:
      cpu: "500m"
      memory: "512Mi"
    defaultRequest:
      cpu: "250m"
      memory: "256Mi"
```

Le même Pod serait **accepté** avec les valeurs par défaut appliquées automatiquement :
```yaml
# Pod réel créé par Kubernetes
apiVersion: v1
kind: Pod
metadata:
  name: my-pod
spec:
  containers:
  - name: nginx
    image: nginx
    resources:
      requests:
        cpu: "250m"
        memory: "256Mi"
      limits:
        cpu: "500m"
        memory: "512Mi"
```

C'est **automatique** ! Vous n'avez rien à faire.

### Scénario 2 : Pod avec des valeurs trop élevées

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: big-pod
  namespace: dev
spec:
  containers:
  - name: app
    image: my-app
    resources:
      requests:
        cpu: "5"          # Trop !
        memory: "10Gi"    # Trop !
```

**Avec LimitRange** :
```yaml
spec:
  limits:
  - type: Container
    max:
      cpu: "2"
      memory: "4Gi"
```

**Résultat** :
```
Error from server (Forbidden): error when creating "big-pod.yaml":
pods "big-pod" is forbidden: maximum cpu usage per Container is 2, but request is 5
```

Le Pod est **rejeté** immédiatement.

### Scénario 3 : Pod avec des valeurs trop basses

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: tiny-pod
spec:
  containers:
  - name: app
    image: my-app
    resources:
      requests:
        cpu: "10m"       # Trop petit
        memory: "32Mi"   # Trop petit
```

**Avec LimitRange** :
```yaml
spec:
  limits:
  - type: Container
    min:
      cpu: "100m"
      memory: "128Mi"
```

**Résultat** :
```
Error from server (Forbidden): error when creating "tiny-pod.yaml":
pods "tiny-pod" is forbidden: minimum memory usage per Container is 128Mi, but request is 32Mi
```

Le Pod est **rejeté** car trop petit.

### Scénario 4 : Pod multi-conteneurs

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: multi-container-pod
spec:
  containers:
  - name: app
    image: my-app
    resources:
      requests:
        cpu: "1"
        memory: "2Gi"
  - name: sidecar
    image: nginx
    resources:
      requests:
        cpu: "500m"
        memory: "1Gi"
  # Total Pod = 1.5 CPU, 3Gi RAM
```

**Avec LimitRange** :
```yaml
spec:
  limits:
  - type: Container
    max:
      cpu: "2"
      memory: "4Gi"
  - type: Pod
    max:
      cpu: "4"
      memory: "8Gi"
```

**Validation** :
- ✅ Chaque conteneur < 2 CPU / 4Gi → OK
- ✅ Total Pod (1.5 CPU / 3Gi) < 4 CPU / 8Gi → OK
- Pod **accepté**

## Combinaison avec Resource Quotas

Les LimitRanges et Resource Quotas travaillent ensemble :

```yaml
# 1. Resource Quota : budget total du namespace
apiVersion: v1
kind: ResourceQuota
metadata:
  name: team-quota
  namespace: my-team
spec:
  hard:
    requests.cpu: "20"
    requests.memory: "40Gi"
    limits.cpu: "40"
    limits.memory: "80Gi"
---
# 2. LimitRange : contraintes individuelles
apiVersion: v1
kind: LimitRange
metadata:
  name: team-limits
  namespace: my-team
spec:
  limits:
  # Niveau conteneur
  - type: Container
    max:
      cpu: "2"
      memory: "4Gi"
    defaultRequest:
      cpu: "500m"
      memory: "512Mi"
  # Niveau Pod
  - type: Pod
    max:
      cpu: "4"
      memory: "8Gi"
```

**Scénario** :
- Quota total : 20 CPU requests
- Max par Pod : 4 CPU
- Minimum de Pods possibles : 5 Pods (5 × 4 = 20)
- Maximum de Pods possibles : 40 Pods (40 × 500m = 20)

**Protection à deux niveaux** :
1. LimitRange empêche un Pod de monopoliser le quota
2. ResourceQuota empêche le namespace de dépasser ses limites

## Valeurs par défaut intelligentes

Les LimitRanges permettent d'avoir des **valeurs par défaut sensées** sans forcer les développeurs à toujours spécifier les resources.

### Pattern : Defaults pour développeurs paresseux

```yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: dev-defaults
  namespace: development
spec:
  limits:
  - type: Container
    # Valeurs par défaut raisonnables
    default:
      cpu: "500m"
      memory: "512Mi"
    defaultRequest:
      cpu: "250m"
      memory: "256Mi"
    # Mais possibilité d'aller plus haut si besoin
    max:
      cpu: "2"
      memory: "4Gi"
```

**Avantages** :
- Les développeurs peuvent déployer rapidement sans se soucier des resources
- Les valeurs par défaut sont raisonnables
- Protection contre les abus avec les max
- Possibilité d'optimiser manuellement si besoin

### Pattern : Strict en production

```yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: prod-limits
  namespace: production
spec:
  limits:
  - type: Container
    # Pas de defaults - force la spécification manuelle
    max:
      cpu: "4"
      memory: "8Gi"
    min:
      cpu: "100m"
      memory: "128Mi"
```

**Philosophie** :
- En production, les équipes **doivent** spécifier explicitement les resources
- Pas de valeurs par défaut = configuration consciente et réfléchie
- Évite les déploiements accidentels mal dimensionnés

## Exemples par type d'environnement

### Environnement Development

```yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: development-limits
  namespace: development
spec:
  limits:
  - type: Container
    max:
      cpu: "1"
      memory: "2Gi"
    min:
      cpu: "50m"
      memory: "64Mi"
    default:
      cpu: "200m"
      memory: "256Mi"
    defaultRequest:
      cpu: "100m"
      memory: "128Mi"
  - type: Pod
    max:
      cpu: "2"
      memory: "4Gi"
  - type: PersistentVolumeClaim
    max:
      storage: "5Gi"
    min:
      storage: "1Gi"
```

**Caractéristiques** :
- Limites basses (économie de ressources)
- Defaults généreux pour faciliter le travail
- Stockage limité

### Environnement Staging

```yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: staging-limits
  namespace: staging
spec:
  limits:
  - type: Container
    max:
      cpu: "2"
      memory: "4Gi"
    min:
      cpu: "100m"
      memory: "128Mi"
    default:
      cpu: "500m"
      memory: "512Mi"
    defaultRequest:
      cpu: "250m"
      memory: "256Mi"
  - type: Pod
    max:
      cpu: "4"
      memory: "8Gi"
  - type: PersistentVolumeClaim
    max:
      storage: "20Gi"
```

**Caractéristiques** :
- Limites moyennes (proche de la prod)
- Defaults raisonnables
- Plus de stockage que dev

### Environnement Production

```yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: production-limits
  namespace: production
spec:
  limits:
  - type: Container
    max:
      cpu: "4"
      memory: "8Gi"
    min:
      cpu: "100m"
      memory: "256Mi"
    # Pas de defaults en prod !
    maxLimitRequestRatio:
      cpu: "2"         # Limite le ratio
      memory: "2"
  - type: Pod
    max:
      cpu: "8"
      memory: "16Gi"
  - type: PersistentVolumeClaim
    max:
      storage: "100Gi"
    min:
      storage: "5Gi"
```

**Caractéristiques** :
- Limites élevées (applications critiques)
- Pas de defaults (spécification explicite)
- Ratio contrôlé (évite les sur-allocations)
- Stockage généreux

## Commandes utiles

### Lister les LimitRanges

```bash
# Tous les LimitRanges du cluster
kubectl get limitrange --all-namespaces

# LimitRanges d'un namespace
kubectl get limitrange -n dev

# Alias court
kubectl get limits -n dev
```

### Voir les détails

```bash
kubectl describe limitrange dev-limits -n dev
```

Résultat :
```
Name:       dev-limits
Namespace:  dev
Type        Resource  Min   Max  Default Request  Default Limit
----        --------  ---   ---  ---------------  -------------
Container   cpu       50m   1    100m             200m
Container   memory    64Mi  2Gi  128Mi            256Mi
Pod         cpu       -     2    -                -
Pod         memory    -     4Gi  -                -
```

### Vérifier l'impact sur un Pod

```bash
# Créer un Pod et voir les resources appliquées
kubectl apply -f my-pod.yaml -n dev

# Vérifier les resources
kubectl get pod my-pod -n dev -o jsonpath='{.spec.containers[0].resources}'
```

### Modifier un LimitRange

```bash
# Éditer directement
kubectl edit limitrange dev-limits -n dev

# Ou réappliquer le fichier modifié
kubectl apply -f dev-limits-updated.yaml
```

### Supprimer un LimitRange

```bash
kubectl delete limitrange dev-limits -n dev
```

⚠️ **Attention** : Supprimer un LimitRange n'affecte pas les Pods existants, mais les nouveaux Pods n'auront plus les contraintes.

## Bonnes pratiques

### 1. Toujours avoir un LimitRange par namespace

✅ **Recommandation** : Chaque namespace devrait avoir un LimitRange, sauf `kube-system`.

```yaml
# Template minimal
apiVersion: v1
kind: LimitRange
metadata:
  name: namespace-defaults
  namespace: <YOUR-NAMESPACE>
spec:
  limits:
  - type: Container
    default:
      cpu: "500m"
      memory: "512Mi"
    defaultRequest:
      cpu: "250m"
      memory: "256Mi"
```

### 2. Defaults en dev, pas en prod

✅ **Pattern recommandé** :

**Development** : Avec defaults (facilite le travail)
```yaml
default:
  cpu: "500m"
  memory: "512Mi"
```

**Production** : Sans defaults (force la réflexion)
```yaml
# Pas de section default/defaultRequest
max:
  cpu: "4"
  memory: "8Gi"
```

### 3. Ratio raisonnable entre min et max

✅ **Bon ratio** :
```yaml
min:
  cpu: "100m"
  memory: "128Mi"
max:
  cpu: "2"        # 20x le min
  memory: "4Gi"   # 32x le min
```

❌ **Mauvais ratio** :
```yaml
min:
  cpu: "10m"
  memory: "16Mi"
max:
  cpu: "16"       # 1600x le min - trop large !
  memory: "64Gi"  # 4096x le min - trop large !
```

Un ratio trop large rend les contraintes inefficaces.

### 4. Limites Pod cohérentes avec limites Container

✅ **Configuration cohérente** :
```yaml
- type: Container
  max:
    cpu: "2"
    memory: "4Gi"
- type: Pod
  max:
    cpu: "6"        # Permet 3 conteneurs de 2 CPU
    memory: "12Gi"  # Permet 3 conteneurs de 4Gi
```

❌ **Configuration incohérente** :
```yaml
- type: Container
  max:
    cpu: "4"
    memory: "8Gi"
- type: Pod
  max:
    cpu: "2"        # Impossible ! Un conteneur seul dépasse
    memory: "4Gi"   # Impossible !
```

### 5. Contrôler le ratio limits/requests

✅ **Empêcher les sur-allocations** :
```yaml
- type: Container
  maxLimitRequestRatio:
    cpu: "2"      # Limits max = 2x requests
    memory: "2"
```

Cela évite des configurations comme :
- Requests: 100m, Limits: 8000m (ratio 80:1) ❌
- Encourage des configurations réalistes

### 6. Documenter les LimitRanges

✅ **Ajouter des annotations** :
```yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: dev-limits
  namespace: development
  annotations:
    description: "Limites pour l'environnement de développement"
    contact: "platform-team@company.com"
    last-review: "2025-10-01"
spec:
  limits: [...]
```

### 7. Tester avant d'appliquer

✅ **Processus de test** :

1. Créer le LimitRange dans un namespace de test
2. Déployer des Pods typiques
3. Vérifier qu'ils sont acceptés
4. Vérifier que les Pods abusifs sont rejetés
5. Appliquer dans les namespaces réels

### 8. Adapter aux workloads

✅ **Différencier selon le type d'application** :

**Namespace frontend** (apps légères) :
```yaml
max:
  cpu: "1"
  memory: "2Gi"
```

**Namespace backend** (apps moyennes) :
```yaml
max:
  cpu: "2"
  memory: "4Gi"
```

**Namespace data** (apps lourdes) :
```yaml
max:
  cpu: "8"
  memory: "16Gi"
```

## Diagnostic et dépannage

### Problème 1 : Pod refusé à cause du max

**Symptôme** :
```bash
kubectl apply -f my-pod.yaml -n dev
# Error: maximum cpu usage per Container is 2, but request is 4
```

**Diagnostic** :
```bash
# Voir les LimitRanges
kubectl describe limitrange -n dev

# Voir les resources du Pod
kubectl describe -f my-pod.yaml | grep -A10 "Containers:"
```

**Solutions** :
1. Réduire les resources du Pod
2. Augmenter le max du LimitRange (si justifié)
3. Utiliser un namespace différent avec des limites plus élevées

### Problème 2 : Pod refusé à cause du min

**Symptôme** :
```
Error: minimum memory usage per Container is 256Mi, but request is 128Mi
```

**Cause** : Les requests du Pod sont trop basses.

**Solutions** :
1. Augmenter les requests du Pod
2. Réduire le min du LimitRange (si justifié)

### Problème 3 : Pod multi-conteneurs refusé

**Symptôme** :
```
Error: maximum cpu usage per Pod is 4, but request is 5
```

**Cause** : La somme des conteneurs dépasse le max Pod.

**Exemple** :
```yaml
spec:
  containers:
  - name: app
    resources:
      requests:
        cpu: "3"     # 3 CPU
  - name: sidecar
    resources:
      requests:
        cpu: "2"     # 2 CPU
  # Total = 5 CPU > max Pod (4 CPU)
```

**Solutions** :
1. Réduire les resources de certains conteneurs
2. Séparer en plusieurs Pods
3. Augmenter le max Pod du LimitRange

### Problème 4 : Ratio refusé

**Symptôme** :
```
Error: cpu max limit to request ratio per Container is 2, but provided ratio is 4
```

**Cause** :
```yaml
resources:
  requests:
    cpu: "500m"
  limits:
    cpu: "2"      # Ratio 4:1 mais max autorisé = 2:1
```

**Solutions** :
1. Réduire les limits : `cpu: "1"` (ratio 2:1)
2. Augmenter les requests : `cpu: "1"` (ratio 2:1)
3. Modifier le maxLimitRequestRatio du LimitRange

### Problème 5 : PVC refusé

**Symptôme** :
```
Error: maximum storage usage per PersistentVolumeClaim is 10Gi, but request is 50Gi
```

**Diagnostic** :
```bash
kubectl describe limitrange -n dev | grep -A5 "PersistentVolumeClaim"
```

**Solutions** :
1. Réduire le storage demandé
2. Augmenter le max storage du LimitRange
3. Créer le PVC dans un autre namespace

## Cas d'usage avancés

### Cas 1 : Cluster multi-tenant

Chaque équipe a des besoins différents :

```yaml
# Équipe Frontend
apiVersion: v1
kind: LimitRange
metadata:
  name: frontend-limits
  namespace: team-frontend
spec:
  limits:
  - type: Container
    max:
      cpu: "1"
      memory: "2Gi"
---
# Équipe Backend
apiVersion: v1
kind: LimitRange
metadata:
  name: backend-limits
  namespace: team-backend
spec:
  limits:
  - type: Container
    max:
      cpu: "2"
      memory: "4Gi"
---
# Équipe Data
apiVersion: v1
kind: LimitRange
metadata:
  name: data-limits
  namespace: team-data
spec:
  limits:
  - type: Container
    max:
      cpu: "8"
      memory: "16Gi"
```

### Cas 2 : Review apps temporaires

Applications éphémères avec limites strictes :

```yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: review-limits
  namespace: review-apps
spec:
  limits:
  - type: Container
    max:
      cpu: "500m"
      memory: "1Gi"
    default:
      cpu: "250m"
      memory: "512Mi"
    defaultRequest:
      cpu: "100m"
      memory: "256Mi"
  - type: Pod
    max:
      cpu: "1"
      memory: "2Gi"
  - type: PersistentVolumeClaim
    max:
      storage: "2Gi"  # Stockage limité
```

### Cas 3 : Environnement de formation

Pour des labs Kubernetes :

```yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: training-limits
  namespace: training
spec:
  limits:
  - type: Container
    max:
      cpu: "500m"
      memory: "512Mi"
    default:
      cpu: "100m"
      memory: "128Mi"
    defaultRequest:
      cpu: "50m"
      memory: "64Mi"
  - type: Pod
    max:
      cpu: "1"
      memory: "1Gi"
```

**Objectif** : Permettre à de nombreux étudiants de travailler simultanément.

## Points clés à retenir

🔑 **Les essentiels** :

1. **LimitRange** = contraintes min/max par objet individuel
2. **Trois niveaux** : Container, Pod, PersistentVolumeClaim
3. **Valeurs par défaut** automatiques si non spécifiées
4. **Complète** les Resource Quotas (total namespace)
5. **Empêche** qu'un objet monopolise le quota
6. **Un seul LimitRange** par namespace (peut avoir plusieurs sections)
7. **Défaults en dev**, pas en prod (bonne pratique)
8. **Cohérence** entre limites Container et Pod

## La hiérarchie complète

Voici comment tous les concepts s'imbriquent :

```
CLUSTER
│
├─ NAMESPACE A
│  ├─ ResourceQuota: 20 CPU total
│  ├─ LimitRange: max 4 CPU/Pod, defaults 500m
│  │
│  ├─ Pod 1 (uses defaults)
│  │  └─ Container: 500m CPU (default applied)
│  │
│  ├─ Pod 2 (explicit values)
│  │  └─ Container: 2 CPU (explicit)
│  │
│  └─ Pod 3 (rejected)
│     └─ Container: 5 CPU (exceeds LimitRange max)
│
└─ NAMESPACE B
   ├─ ResourceQuota: 10 CPU total
   ├─ LimitRange: max 2 CPU/Pod
   └─ ...
```

**Ordre de validation** :
1. LimitRange vérifie les valeurs individuelles
2. ResourceQuota vérifie le total du namespace
3. Les deux doivent être satisfaits pour accepter le Pod

## Conclusion

Les LimitRanges sont la **clé de voûte** d'une bonne gouvernance des ressources dans Kubernetes. Ils complètent parfaitement les Requests/Limits et les Resource Quotas pour offrir un système complet de contrôle.

**En résumé** :
- **Requests/Limits** (18.1) : Que demande chaque conteneur ?
- **Resource Quotas** (18.2) : Combien peut consommer tout le namespace ?
- **LimitRanges** (18.3) : Quelles sont les bornes acceptables ?

Ensemble, ces trois mécanismes assurent :
- ✅ Stabilité du cluster
- ✅ Équité entre les équipes
- ✅ Prévisibilité des coûts
- ✅ Protection contre les erreurs de configuration

**Prochaine étape** : Dans la section suivante (18.4), nous verrons les **QoS Classes** qui déterminent comment Kubernetes priorise les Pods en cas de pression sur les ressources.

---


⏭️ [QoS Classes](/18-gestion-des-ressources/04-qos-classes.md)
