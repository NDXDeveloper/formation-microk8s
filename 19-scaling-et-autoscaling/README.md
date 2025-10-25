🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 19. Scaling et Autoscaling

## Introduction

Bienvenue dans le chapitre **Scaling et Autoscaling** ! C'est l'une des fonctionnalités les plus puissantes de Kubernetes et l'une des principales raisons pour lesquelles les entreprises adoptent cette technologie.

Dans ce chapitre, vous allez découvrir comment faire en sorte que vos applications s'adaptent automatiquement à la charge qu'elles reçoivent, un peu comme si elles avaient une "intelligence" qui leur permet de grandir quand nécessaire et de rétrécir quand la charge diminue.

## Qu'est-ce que le scaling ?

Le **scaling** (mise à l'échelle en français) est la capacité d'ajuster les ressources allouées à votre application en fonction de ses besoins. C'est un concept fondamental dans le monde du cloud computing et des applications modernes.

### Analogie du restaurant

Imaginez que vous gérez un restaurant :

**Situation 1 : Mardi midi (faible affluence)**
- 10 clients dans le restaurant
- 2 serveurs suffisent largement
- La cuisine tourne au ralenti
- Tout le monde est content

**Situation 2 : Samedi soir (forte affluence)**
- 80 clients dans le restaurant
- 2 serveurs ne suffisent plus !
- Les clients attendent 30 minutes pour être servis
- La cuisine est débordée
- Tout le monde est frustré

**Solution traditionnelle (sans scaling) :**
Embaucher 10 serveurs en permanence "au cas où" → Coûteux et inefficace les jours calmes

**Solution moderne (avec scaling) :**
- Mardi midi : 2 serveurs travaillent (coût optimisé)
- Samedi soir : 8 serveurs travaillent (service optimal)
- Les serveurs supplémentaires n'arrivent que quand nécessaire

C'est exactement ce que fait le scaling dans Kubernetes : adapter les ressources en temps réel selon la demande !

## Pourquoi le scaling est-il important ?

### 1. Économies de coûts

**Sans scaling :**
```
Ressources provisionnées : ████████████████████ (100%)
Ressources utilisées :     ████░░░░░░░░░░░░░░░░ (20%)
                           ↑
                    Gaspillage : 80% !
```

**Avec scaling :**
```
Charge faible :  ████░░░░░░░░░░░░░░░░ (20% utilisé, 20% provisionné)
Charge élevée :  ████████████░░░░░░░░ (60% utilisé, 60% provisionné)
```

Vous ne payez que ce que vous utilisez réellement !

### 2. Meilleures performances

Quand votre application reçoit soudainement beaucoup de trafic :
- **Sans scaling** : Les serveurs saturent, tout ralentit, certains utilisateurs reçoivent des erreurs
- **Avec scaling** : De nouveaux serveurs s'ajoutent automatiquement, les performances restent bonnes

### 3. Haute disponibilité

Si un pod tombe en panne :
- **Sans scaling** : Si vous aviez 1 seul pod, votre application est down
- **Avec scaling** : Vous avez plusieurs répliques, une défaillance n'impacte pas le service

### 4. Adaptabilité aux événements

Votre application doit gérer :
- Les pics de trafic (promotions, lancements, événements)
- Les variations journalières (plus de charge en journée, moins la nuit)
- Les variations saisonnières (Noël, soldes, vacances)
- Les événements imprévus (contenu viral, mention dans les médias)

Le scaling vous permet de gérer tout cela automatiquement !

## Les différents types de scaling

Dans ce chapitre, nous allons explorer **trois dimensions** du scaling :

### 1. Scaling Horizontal (Horizontal Scaling)

**Principe :** Ajouter ou retirer des **pods** (instances de votre application)

```
Avant :
┌─────┐ ┌─────┐
│ Pod │ │ Pod │
│  1  │ │  2  │
└─────┘ └─────┘

Après (scale up) :
┌─────┐ ┌─────┐ ┌─────┐ ┌─────┐ ┌─────┐
│ Pod │ │ Pod │ │ Pod │ │ Pod │ │ Pod │
│  1  │ │  2  │ │  3  │ │  4  │ │  5  │
└─────┘ └─────┘ └─────┘ └─────┘ └─────┘
```

**Avantages :**
- ✅ Très flexible
- ✅ Peut scaler indéfiniment (dans les limites de votre cluster)
- ✅ Résilience accrue (plusieurs copies)

**Inconvénients :**
- ❌ Nécessite que votre application soit stateless ou bien conçue

### 2. Scaling Vertical (Vertical Scaling)

**Principe :** Augmenter ou diminuer les **ressources** (CPU, mémoire) de chaque pod

```
Avant :
┌──────────────┐
│     Pod      │
│  CPU: 1 core │
│  RAM: 512Mi  │
└──────────────┘

Après (scale up) :
┌───────────────┐
│     Pod       │
│  CPU: 4 cores │
│  RAM: 2Gi     │
└───────────────┘
```

**Avantages :**
- ✅ Plus simple pour certaines applications
- ✅ Pas besoin de gérer plusieurs instances

**Inconvénients :**
- ❌ Limité par la taille des machines
- ❌ Nécessite généralement un redémarrage du pod

### 3. Scaling du Cluster (Cluster Scaling)

**Principe :** Ajouter ou retirer des **nœuds** (machines) dans le cluster

```
Avant :
┌──────────────────┐ ┌──────────────────┐
│      Nœud 1      │ │      Nœud 2      │
│  ┌───┐ ┌───┐     │ │  ┌───┐ ┌───┐     │
│  │Pod│ │Pod│     │ │  │Pod│ │Pod│     │
│  └───┘ └───┘     │ │  └───┘ └───┘     │
└──────────────────┘ └──────────────────┘

Après (scale up) :
┌──────────────────┐ ┌──────────────────┐ ┌──────────────────┐
│      Nœud 1      │ │      Nœud 2      │ │      Nœud 3      │
│  ┌───┐ ┌───┐     │ │  ┌───┐ ┌───┐     │ │  ┌───┐ ┌───┐     │
│  │Pod│ │Pod│     │ │  │Pod│ │Pod│     │ │  │Pod│ │Pod│     │
│  └───┘ └───┘     │ │  └───┘ └───┘     │ │  └───┘ └───┘     │
└──────────────────┘ └──────────────────┘ └──────────────────┘
```

**Avantages :**
- ✅ Augmente la capacité totale du cluster
- ✅ Permet au scaling horizontal de continuer

**Inconvénients :**
- ❌ Plus lent (provisionner une machine prend plusieurs minutes)
- ❌ Nécessite une infrastructure compatible (cloud généralement)

## Manuel vs Automatique

Il existe deux façons de gérer le scaling :

### Scaling Manuel

**Vous** décidez quand et comment scaler.

```bash
# Vous exécutez cette commande
kubectl scale deployment mon-app --replicas=10
```

**Avantages :**
- ✅ Simple à comprendre
- ✅ Contrôle total
- ✅ Bon pour les pics prévisibles

**Inconvénients :**
- ❌ Nécessite une intervention humaine
- ❌ Pas de réactivité 24/7
- ❌ Peut oublier de scale down

**Cas d'usage :**
- Pic de trafic planifié (lancement produit, promotion)
- Tests de charge
- Situations d'urgence

### Autoscaling (Scaling Automatique)

Kubernetes décide automatiquement quand scaler basé sur des métriques.

```bash
# Vous configurez une règle
kubectl autoscale deployment mon-app --cpu-percent=50 --min=2 --max=20

# Kubernetes gère le reste automatiquement !
```

**Avantages :**
- ✅ Réactivité 24/7
- ✅ Pas d'intervention nécessaire
- ✅ Optimisation automatique des coûts
- ✅ Gère les pics imprévus

**Inconvénients :**
- ❌ Plus complexe à configurer
- ❌ Nécessite des métriques fiables
- ❌ Peut être imprévisible si mal configuré

**Cas d'usage :**
- Applications en production
- Charge variable et imprévisible
- Besoin de disponibilité 24/7

## Les outils Kubernetes pour le scaling

Kubernetes fournit plusieurs outils pour gérer le scaling :

### HPA (Horizontal Pod Autoscaler)

```
┌─────────────────────────────────────────┐
│   Horizontal Pod Autoscaler (HPA)       │
│                                         │
│   Surveille : CPU, mémoire, métriques   │
│   Ajuste   : Nombre de pods             │
│   Décision : Automatique                │
└─────────────────────────────────────────┘
                    ↓
         Ajoute/retire des pods
```

**C'est l'outil le plus utilisé et le plus important !**

Nous verrons le HPA en détail dans la section 19.2.

### VPA (Vertical Pod Autoscaler)

```
┌─────────────────────────────────────────┐
│   Vertical Pod Autoscaler (VPA)         │
│                                         │
│   Surveille : Utilisation réelle        │
│   Ajuste   : CPU/RAM par pod            │
│   Décision : Recommandations ou auto    │
└─────────────────────────────────────────┘
                    ↓
     Modifie les ressources des pods
```

Nous verrons le VPA dans la section 19.3.

### Cluster Autoscaler

```
┌─────────────────────────────────────────┐
│      Cluster Autoscaler (CA)            │
│                                         │
│   Surveille : Pods en attente           │
│   Ajuste   : Nombre de nœuds            │
│   Décision : Automatique                │
└─────────────────────────────────────────┘
                    ↓
       Ajoute/retire des nœuds
```

Nous verrons le Cluster Autoscaler dans la section 19.4.

## La fondation : Metrics Server

Pour que l'autoscaling fonctionne, Kubernetes a besoin de **métriques** :

```
┌──────────────────────────────────────────────────┐
│             Metrics Server                       │
│  "Combien de CPU/RAM chaque pod utilise ?"       │
└──────────────────────────────────────────────────┘
                     ↓
         Fournit les données aux autoscalers
                     ↓
┌──────────────────────────────────────────────────┐
│  HPA / VPA / kubectl top                         │
│  Prennent des décisions basées sur ces données   │
└──────────────────────────────────────────────────┘
```

Le Metrics Server est **indispensable** pour tout autoscaling !

Nous verrons comment l'activer et l'utiliser dans la section 19.5.

## Comment fonctionne l'autoscaling ?

Voici le cycle de décision d'un autoscaler (exemple avec le HPA) :

```
1. Metrics Server collecte les métriques
   (toutes les 60 secondes)
          ↓
2. HPA interroge Metrics Server
   "Quelle est l'utilisation CPU moyenne ?"
          ↓
3. HPA compare avec la cible configurée
   Exemple : Actuel=70%, Cible=50%
          ↓
4. HPA calcule le nombre de répliques nécessaires
   Formule : répliques = actuel × (métrique_actuelle / métrique_cible)
          ↓
5. HPA ajuste le Deployment
   "Passe de 3 à 5 répliques"
          ↓
6. Kubernetes crée les nouveaux pods
          ↓
7. Load balancer répartit la charge
          ↓
8. Utilisation CPU diminue vers 50%
          ↓
9. HPA observe pendant quelques minutes
          ↓
10. Si stable, HPA considère que c'est bon
          ↓
    Retour à l'étape 1 (boucle continue)
```

Ce cycle se répète **en permanence**, 24h/24, 7j/7 !

## Cas d'usage réels

### Exemple 1 : Site e-commerce

**Sans autoscaling :**
- Black Friday arrive
- Le site reçoit 100x plus de trafic
- Les serveurs saturent
- Le site plante
- Perte de revenus énorme 💸

**Avec autoscaling :**
- Black Friday arrive
- Le trafic augmente progressivement
- HPA détecte la hausse de CPU
- HPA ajoute automatiquement des pods (2→5→10→20)
- Le site reste rapide
- Clients satisfaits ✅
- Après le Black Friday, scale down automatique (économies)

### Exemple 2 : API publique

**Sans autoscaling :**
- Application mentionnée sur Hacker News
- 10 000 visiteurs en 10 minutes
- API submergée
- Erreurs 503 pour tout le monde
- Mauvaise réputation

**Avec autoscaling :**
- Trafic soudain détecté
- HPA scale rapidement (2→15 pods)
- API reste disponible et rapide
- Bonne première impression ✅

### Exemple 3 : Traitement batch

**Sans autoscaling :**
- 10 000 jobs à traiter pendant la nuit
- 2 workers fixes
- Traitement prend 12 heures
- Jobs pas terminés le matin ❌

**Avec autoscaling :**
- Jobs s'accumulent dans la queue
- HPA scale basé sur la longueur de la queue
- 20 workers sont créés
- Traitement terminé en 2 heures ✅
- Workers supprimés après, économie de ressources

## Ce que vous allez apprendre

Dans ce chapitre, nous allons couvrir :

### 19.1 - Scaling Manuel
- Comment scaler manuellement avec kubectl
- Quand utiliser le scaling manuel
- Avantages et limitations

### 19.2 - Horizontal Pod Autoscaler (HPA)
- Configuration du HPA
- Métriques basées sur CPU et mémoire
- Comment le HPA prend ses décisions
- Bonnes pratiques

### 19.3 - Vertical Pod Autoscaler (VPA)
- Installation du VPA
- Modes de fonctionnement (Off, Initial, Auto)
- Recommandations de ressources
- Quand utiliser VPA vs HPA

### 19.4 - Cluster Autoscaler
- Scaling du cluster lui-même
- Configuration pour multi-node
- Limitations pour homelab
- Intégration cloud

### 19.5 - Metrics Server
- Installation et configuration
- Comment fonctionne la collecte de métriques
- Commandes kubectl top
- Dépannage

### 19.6 - Custom Metrics
- Utiliser des métriques personnalisées
- Prometheus Adapter
- Scaler sur des métriques métier
- Exemples pratiques (requêtes/sec, longueur de queue)

### 19.7 - Tests de charge
- Outils de load testing (hey, ab, k6, locust)
- Méthodologie de test
- Interpréter les résultats
- Valider votre autoscaling

## Prérequis pour ce chapitre

Avant de commencer, assurez-vous d'avoir :

✅ **MicroK8s installé et fonctionnel**
```bash
microk8s status
```

✅ **Au moins une application déployée**
```bash
kubectl get deployments
```

✅ **Compréhension des Pods et Deployments** (Chapitre 3 et 4)

✅ **Notions de ressources (CPU/RAM)** (Chapitre 18)

### Addons recommandés

Activez ces addons avant de commencer :

```bash
# Obligatoire pour l'autoscaling
microk8s enable metrics-server

# Recommandé pour le monitoring
microk8s enable prometheus

# Optionnel mais utile
microk8s enable dashboard
```

Vérifiez que Metrics Server fonctionne :

```bash
kubectl top nodes
```

Si cette commande fonctionne, vous êtes prêt !

## Progression recommandée

Nous vous recommandons de suivre les sections **dans l'ordre** :

```
1. Scaling manuel (19.1)
   ↓ Comprendre les bases

2. HPA (19.2)
   ↓ Autoscaling basique mais puissant

3. Metrics Server (19.5)
   ↓ Fondation technique

4. VPA (19.3)
   ↓ Optimisation des ressources

5. Custom Metrics (19.6)
   ↓ Autoscaling avancé

6. Tests de charge (19.7)
   ↓ Validation

7. Cluster Autoscaler (19.4)
   ↓ Si multi-node et cloud
```

**Note :** Si vous êtes sur un homelab avec un seul nœud, vous pouvez sauter la section 19.4 (Cluster Autoscaler) qui est principalement pertinente pour le cloud.

## Philosophie du scaling

Avant de plonger dans les détails techniques, gardez à l'esprit ces principes :

### 1. Commencez simple

```
Phase 1 : Scaling manuel
         ↓
Phase 2 : HPA avec CPU uniquement
         ↓
Phase 3 : HPA avec CPU + mémoire
         ↓
Phase 4 : Custom metrics
```

Ne sautez pas directement au niveau avancé !

### 2. Mesurez, ne devinez pas

**Mauvais :**
"Je pense que 50% CPU est un bon seuil"

**Bon :**
"J'ai testé avec différentes charges, et 50% CPU maintient une latence <200ms"

### 3. Testez avant la production

Validez toujours votre configuration de scaling en environnement de test avec des tests de charge réels.

### 4. Surveillez

L'autoscaling n'est pas "configure et oublie". Surveillez régulièrement :
- Le scaling fonctionne-t-il comme prévu ?
- Les seuils sont-ils bien calibrés ?
- Y a-t-il des oscillations (flapping) ?

### 5. Documentez vos décisions

Notez pourquoi vous avez choisi certains seuils ou configurations. Dans 6 mois, vous (ou votre équipe) devrez comprendre les choix faits.

## Concepts clés à comprendre

Avant de continuer, assurez-vous de bien comprendre ces concepts :

### Répliques
Copies identiques de votre application qui tournent en parallèle.

### Requests et Limits
```yaml
resources:
  requests:     # Ressources garanties
    cpu: 100m
    memory: 128Mi
  limits:       # Maximum autorisé
    cpu: 500m
    memory: 512Mi
```

**Crucial pour le HPA !** Sans requests, le HPA ne peut pas calculer les pourcentages.

### Métriques
Mesures de l'utilisation des ressources (CPU, mémoire, requêtes/sec, etc.)

### Seuil (Target/Threshold)
Valeur à laquelle l'autoscaler réagit. Exemple : "Maintenir CPU à 50%"

### Scale Up vs Scale Down
- **Scale Up** : Augmenter les ressources (rapide, prioritaire)
- **Scale Down** : Diminuer les ressources (lent, prudent)

## Terminologie

Voici les termes que vous rencontrerez souvent :

| Terme | Signification |
|-------|---------------|
| **Scaling** | Ajuster les ressources |
| **Autoscaling** | Scaling automatique |
| **Scale Up** | Augmenter les ressources |
| **Scale Down** | Diminuer les ressources |
| **Scale Out** | Ajouter des instances (horizontal) |
| **Scale In** | Retirer des instances (horizontal) |
| **Répliques** | Nombre de copies de pods |
| **Target** | Valeur cible pour les métriques |
| **Cooldown** | Période d'attente avant un nouveau scaling |
| **Flapping** | Oscillations (scale up/down répété) |
| **Throttling** | Limitation de CPU quand limite atteinte |
| **OOM (Out Of Memory)** | Pod tué car mémoire épuisée |

## Avertissements et bonnes pratiques

### ⚠️ Le scaling n'est pas magique

Le scaling améliore la capacité, mais ne résout pas :
- ❌ Le code mal optimisé
- ❌ Les fuites mémoire
- ❌ Les goulots d'étranglement de base de données
- ❌ Les problèmes d'architecture

**Règle d'or :** Optimisez d'abord votre code, scalez ensuite !

### ⚠️ Testez en environnement de dev

Ne testez **jamais** l'autoscaling directement en production. Utilisez un environnement de staging.

### ⚠️ Surveillez les coûts

Dans le cloud, le scaling automatique peut faire exploser les coûts si mal configuré. Définissez toujours des limites maximales (maxReplicas).

### ⚠️ Prévoyez du temps

Le scaling n'est pas instantané :
- Nouveaux pods : 30 secondes à 2 minutes
- Nouveaux nœuds : 3 à 5 minutes
- Décisions HPA : 15 secondes à 3 minutes

Planifiez en conséquence pour les événements prévisibles.

## À vous de jouer !

Vous êtes maintenant prêt à découvrir le monde du scaling et de l'autoscaling dans Kubernetes. Ce chapitre est dense mais passionnant, et les connaissances que vous allez acquérir sont parmi les plus valorisées dans le monde de l'infrastructure moderne.

**Conseil pour débutants :**
Ne vous précipitez pas. Prenez le temps de comprendre chaque concept avant de passer au suivant. Le scaling manuel peut sembler "basique" mais c'est la fondation pour tout comprendre ensuite.

**Conseil pour utilisateurs avancés :**
Même si vous connaissez déjà le HPA, nous vous encourageons à lire toutes les sections. Vous y trouverez des bonnes pratiques, des pièges à éviter, et des astuces qui amélioreront vos configurations existantes.

---

**Prêt ?** Commençons par la section 19.1 - Scaling manuel, pour bien comprendre les bases avant de nous lancer dans l'autoscaling !

**➡️ Prochaine section : Scaling manuel**

⏭️ [Scaling manuel](/19-scaling-et-autoscaling/01-scaling-manuel.md)
