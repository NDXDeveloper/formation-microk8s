🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 23.1 Méthodologie de Dépannage

## Introduction

Le dépannage dans Kubernetes peut sembler intimidant au début, mais avec une méthodologie structurée, vous pouvez résoudre la plupart des problèmes efficacement. Cette section vous présente une approche systématique pour diagnostiquer et résoudre les problèmes dans votre cluster MicroK8s.

> **Pour les débutants** : Ne vous inquiétez pas si certains termes vous semblent complexes. Le dépannage est une compétence qui s'acquiert avec la pratique. Cette méthodologie vous servira de guide tout au long de votre apprentissage.

## Pourquoi une Méthodologie ?

Sans approche structurée, le dépannage peut devenir :
- **Chaotique** : vous sautez d'une commande à l'autre sans logique
- **Inefficace** : vous perdez du temps sur des pistes erronées
- **Frustrant** : vous ne comprenez pas vraiment ce qui s'est passé

Une méthodologie claire vous permet de :
- Gagner du temps en allant à l'essentiel
- Comprendre la cause réelle du problème
- Apprendre de chaque situation
- Documenter vos solutions pour le futur

## Les 6 Étapes de la Méthodologie de Dépannage

### Étape 1 : Définir le Problème

**Objectif** : Comprendre précisément ce qui ne fonctionne pas.

Avant de chercher une solution, il faut savoir exactement ce qui est cassé. Posez-vous ces questions :

- **Quel est le symptôme observable ?**
  - "Mon application ne répond pas"
  - "Je ne peux pas accéder à mon service"
  - "Mon pod redémarre en boucle"

- **Quand le problème est-il apparu ?**
  - Après un déploiement ?
  - Après une mise à jour ?
  - De manière aléatoire ?

- **Le problème est-il constant ou intermittent ?**
  - Cela aide à identifier si c'est un problème de ressources, de réseau, etc.

- **Qu'avez-vous modifié récemment ?**
  - Un nouveau déploiement ?
  - Une modification de configuration ?
  - Un changement de ressources ?

**Conseil** : Écrivez une description claire du problème en une phrase. Par exemple : "Mon pod nginx redémarre toutes les 30 secondes depuis que j'ai déployé la version 2.0 ce matin."

### Étape 2 : Collecter les Informations

**Objectif** : Rassembler les données nécessaires pour analyser le problème.

C'est l'étape de l'enquête. Vous allez collecter des indices sur l'état de votre système.

#### 2.1 Vérifier l'état global du cluster

Commencez toujours par une vue d'ensemble :

```bash
# État général de MicroK8s
microk8s status

# État des nœuds
microk8s kubectl get nodes

# État des pods dans tous les namespaces
microk8s kubectl get pods --all-namespaces
```

**Ce que vous cherchez** :
- Le cluster est-il en cours d'exécution ?
- Y a-t-il des erreurs évidentes ?
- D'autres pods sont-ils affectés ?

#### 2.2 Identifier les ressources concernées

Localisez précisément les ressources impliquées :

```bash
# Lister les pods dans votre namespace
microk8s kubectl get pods -n votre-namespace

# Lister les services
microk8s kubectl get services -n votre-namespace

# Lister les déploiements
microk8s kubectl get deployments -n votre-namespace
```

**Ce que vous cherchez** :
- Quel est le statut des ressources ? (Running, CrashLoopBackOff, Pending, etc.)
- Combien de réplicas sont prêts ?
- Y a-t-il des redémarrages fréquents ?

#### 2.3 Examiner les détails de la ressource problématique

Utilisez `describe` pour obtenir des informations détaillées :

```bash
# Détails d'un pod spécifique
microk8s kubectl describe pod nom-du-pod -n votre-namespace
```

**Ce que vous cherchez** dans la sortie :
- **Status** : État actuel du pod
- **Events** : Historique des événements (la section la plus importante !)
- **Conditions** : État des conditions de santé
- **Container States** : État de chaque conteneur

**Astuce pour débutants** : La section "Events" en bas de la commande `describe` est souvent la plus utile. Elle vous indique ce qui s'est passé chronologiquement.

#### 2.4 Consulter les logs

Les logs sont vos meilleurs amis pour comprendre ce qui se passe à l'intérieur de votre application :

```bash
# Logs d'un pod
microk8s kubectl logs nom-du-pod -n votre-namespace

# Logs du conteneur précédent (si le pod a redémarré)
microk8s kubectl logs nom-du-pod -n votre-namespace --previous

# Suivre les logs en temps réel
microk8s kubectl logs -f nom-du-pod -n votre-namespace
```

**Ce que vous cherchez** :
- Messages d'erreur
- Exceptions ou stack traces
- Avertissements répétés
- Derniers messages avant un crash

### Étape 3 : Analyser les Données

**Objectif** : Comprendre la cause racine du problème à partir des informations collectées.

Maintenant que vous avez les données, il faut les interpréter. Voici les problèmes les plus courants et comment les identifier :

#### 3.1 Problèmes de démarrage de pod

**Symptômes** :
- Pod en état `Pending`
- Pod en état `ImagePullBackOff`
- Pod en état `CrashLoopBackOff`

**Analyse** :

**Cas 1 : Pod en `Pending`**
- **Cause probable** : Ressources insuffisantes ou problème de scheduling
- **Vérifier** : Les events du pod mentionnent-ils "Insufficient CPU/memory" ?
- **Vérifier** : Y a-t-il des contraintes de placement (nodeSelector, taints) ?

**Cas 2 : Pod en `ImagePullBackOff`**
- **Cause probable** : L'image Docker n'est pas accessible
- **Vérifier** : Le nom de l'image est-il correct ?
- **Vérifier** : Le registry est-il accessible ?
- **Vérifier** : Y a-t-il un secret pour s'authentifier au registry ?

**Cas 3 : Pod en `CrashLoopBackOff`**
- **Cause probable** : L'application crash au démarrage
- **Vérifier** : Les logs du conteneur montrent-ils une erreur ?
- **Vérifier** : La configuration (ConfigMap, Secrets) est-elle correcte ?
- **Vérifier** : Les liveness/readiness probes sont-elles bien configurées ?

#### 3.2 Problèmes de connectivité

**Symptômes** :
- Impossible d'accéder à un service
- Timeouts réseau
- Erreurs de résolution DNS

**Analyse** :

**Vérifier la configuration du service** :
```bash
microk8s kubectl get service nom-du-service -n votre-namespace
```
- Le sélecteur du service correspond-il aux labels du pod ?
- Le port exposé est-il le bon ?

**Vérifier la résolution DNS** :
```bash
# Créer un pod temporaire pour tester
microk8s kubectl run test-dns --image=busybox --rm -it -- nslookup nom-du-service
```

**Vérifier les Network Policies** :
- Y a-t-il des Network Policies qui bloquent le trafic ?

#### 3.3 Problèmes de performance

**Symptômes** :
- Application lente
- Pods évictés (expulsés)
- Out Of Memory (OOM)

**Analyse** :

**Vérifier l'utilisation des ressources** :
```bash
# Utilisation CPU/Mémoire des pods
microk8s kubectl top pods -n votre-namespace

# Utilisation des nœuds
microk8s kubectl top nodes
```

**Vérifier les limites de ressources** :
- Les requests/limits sont-ils définis ?
- Sont-ils adaptés à votre application ?

#### 3.4 Problèmes de configuration

**Symptômes** :
- Application avec comportement incorrect
- Erreurs de configuration dans les logs

**Analyse** :

**Vérifier les ConfigMaps et Secrets** :
```bash
# Lister les ConfigMaps
microk8s kubectl get configmaps -n votre-namespace

# Voir le contenu
microk8s kubectl describe configmap nom-configmap -n votre-namespace
```

**Vérifier les variables d'environnement** :
```bash
# Exécuter une commande dans le pod pour voir les variables
microk8s kubectl exec nom-du-pod -n votre-namespace -- env
```

### Étape 4 : Formuler une Hypothèse

**Objectif** : Proposer une explication logique basée sur les données collectées.

À ce stade, vous devriez avoir suffisamment d'informations pour formuler une hypothèse sur la cause du problème.

**Comment formuler une bonne hypothèse** :

1. **Basez-vous sur les faits**, pas sur des suppositions
2. **Soyez spécifique** : "Le pod crash parce que la variable d'environnement DATABASE_URL est manquante" plutôt que "Il y a un problème de configuration"
3. **Priorisez** : Si vous avez plusieurs hypothèses, commencez par la plus probable

**Exemple de raisonnement** :

```
Symptôme observé :
- Pod en CrashLoopBackOff

Données collectées :
- Les logs montrent "Error: Cannot connect to database"
- La variable d'environnement DATABASE_URL n'apparaît pas dans 'env'
- Le Secret 'db-credentials' existe mais n'est pas monté dans le Deployment

Hypothèse :
Le pod crash parce que le Secret contenant les credentials de la base
de données n'est pas correctement référencé dans le Deployment.
```

### Étape 5 : Tester la Solution

**Objectif** : Vérifier votre hypothèse en appliquant une correction.

Maintenant que vous avez une hypothèse, testez-la en appliquant une solution ciblée.

#### 5.1 Appliquer la correction de manière contrôlée

**Bonnes pratiques** :

1. **Faites une sauvegarde** si vous modifiez une configuration importante :
```bash
microk8s kubectl get deployment nom-deployment -n votre-namespace -o yaml > backup-deployment.yaml
```

2. **Modifiez une seule chose à la fois** : Si vous changez plusieurs paramètres simultanément, vous ne saurez pas lequel a résolu le problème.

3. **Utilisez `kubectl edit` ou `kubectl apply`** selon le cas :
```bash
# Édition interactive
microk8s kubectl edit deployment nom-deployment -n votre-namespace

# Ou appliquer un fichier modifié
microk8s kubectl apply -f deployment-corrige.yaml
```

#### 5.2 Observer les résultats

Après avoir appliqué la correction, surveillez attentivement :

```bash
# Surveiller le déploiement de la nouvelle version
microk8s kubectl rollout status deployment/nom-deployment -n votre-namespace

# Voir les événements en temps réel
microk8s kubectl get events -n votre-namespace --watch

# Surveiller les pods
microk8s kubectl get pods -n votre-namespace --watch
```

**Questions à se poser** :
- Le pod démarre-t-il correctement ?
- Y a-t-il encore des erreurs dans les logs ?
- L'application fonctionne-t-elle comme prévu ?

#### 5.3 Valider la résolution complète

Ne vous arrêtez pas au premier signe d'amélioration. Vérifiez que le problème est complètement résolu :

- **Testez la fonctionnalité** : L'application fait-elle ce qu'elle doit faire ?
- **Attendez quelques minutes** : Le problème réapparaît-il ?
- **Vérifiez les métriques** : CPU, mémoire, réseau sont-ils normaux ?

### Étape 6 : Documenter la Solution

**Objectif** : Capitaliser sur votre expérience pour éviter de reproduire les mêmes erreurs.

C'est l'étape la plus souvent négligée, mais c'est l'une des plus importantes !

#### 6.1 Que documenter ?

Créez un document (fichier texte, note, wiki) avec :

1. **Le problème rencontré**
   - Description claire du symptôme
   - Date et contexte

2. **La cause identifiée**
   - Explication de la cause racine
   - Pourquoi c'est arrivé

3. **La solution appliquée**
   - Commandes exactes utilisées
   - Modifications de configuration

4. **Les leçons apprises**
   - Ce que vous feriez différemment
   - Comment prévenir ce problème à l'avenir

**Exemple de documentation** :

```markdown
## Problème : Pod CrashLoopBackOff sur l'application web

**Date** : 2025-10-26
**Namespace** : production
**Ressource** : deployment/webapp

### Symptôme
Le pod webapp redémarre en boucle toutes les 30 secondes.

### Cause
Le Secret contenant les credentials de base de données n'était pas
monté dans le Deployment. L'application crashait au démarrage en
ne trouvant pas la variable DATABASE_URL.

### Solution
Ajout de la référence au Secret dans le Deployment :
```yaml
envFrom:
  - secretRef:
      name: db-credentials
```

Commande appliquée :
```bash
microk8s kubectl apply -f webapp-deployment.yaml
```

### Prévention
- Toujours vérifier que les Secrets/ConfigMaps sont référencés
  avant de déployer
- Ajouter une vérification dans le pipeline CI/CD
```
```

#### 6.2 Créer des runbooks

Pour les problèmes récurrents, créez des "runbooks" - des procédures pas à pas :

```markdown
## Runbook : Redémarrage d'un pod bloqué

1. Identifier le pod problématique :
   `microk8s kubectl get pods -A | grep -v Running`

2. Collecter les logs :
   `microk8s kubectl logs <pod-name> -n <namespace> --previous`

3. Supprimer le pod pour forcer un redémarrage :
   `microk8s kubectl delete pod <pod-name> -n <namespace>`

4. Vérifier le redémarrage :
   `microk8s kubectl get pods -n <namespace> --watch`
```

## Outils pour Faciliter le Dépannage

### Outil 1 : microk8s inspect

MicroK8s dispose d'un outil de diagnostic intégré très puissant :

```bash
microk8s inspect
```

Cet outil :
- Collecte automatiquement les informations du cluster
- Génère un rapport complet
- Identifie les problèmes courants
- Crée une archive que vous pouvez partager pour obtenir de l'aide

**Quand l'utiliser** : Lorsque vous avez un problème complexe et que vous voulez une vue d'ensemble complète.

### Outil 2 : kubectl plugin stern (optionnel)

Pour suivre les logs de plusieurs pods simultanément :

```bash
# Installation
snap install stern

# Utilisation
stern nom-prefix -n votre-namespace
```

### Outil 3 : kubectl debug (Kubernetes 1.18+)

Pour déboguer un pod en créant une copie avec des outils de debug :

```bash
microk8s kubectl debug pod-problematique -it --image=busybox
```

## Checklist Rapide de Dépannage

Voici une checklist à suivre systématiquement :

- [ ] **Définir** : Ai-je décrit le problème en une phrase claire ?
- [ ] **Cluster** : MicroK8s est-il en cours d'exécution ? (`microk8s status`)
- [ ] **État** : Quel est l'état de mes ressources ? (`get pods/services/deployments`)
- [ ] **Détails** : Ai-je examiné les events ? (`describe`)
- [ ] **Logs** : Ai-je consulté les logs ? (`logs`)
- [ ] **Ressources** : Y a-t-il des problèmes de CPU/mémoire ? (`top`)
- [ ] **Réseau** : Les services et DNS fonctionnent-ils ?
- [ ] **Config** : Les ConfigMaps et Secrets sont-ils corrects ?
- [ ] **Hypothèse** : Ai-je formulé une cause probable basée sur les données ?
- [ ] **Test** : Ai-je testé une solution ciblée ?
- [ ] **Validation** : Le problème est-il complètement résolu ?
- [ ] **Documentation** : Ai-je documenté la solution ?

## Conseils Pratiques pour Débutants

### 1. Commencez simple

Ne cherchez pas immédiatement des causes complexes. Les problèmes les plus courants sont souvent simples :
- Faute de frappe dans un nom
- Port incorrect
- Image Docker inexistante
- Secret non créé

### 2. Utilisez les messages d'erreur

Les messages d'erreur Kubernetes sont souvent très explicites. Lisez-les attentivement au lieu de paniquer. Google est votre ami pour comprendre les messages que vous ne connaissez pas.

### 3. Comparez avec ce qui fonctionne

Si vous avez un déploiement qui fonctionne, comparez-le avec celui qui ne fonctionne pas. Utilisez `diff` pour comparer les fichiers YAML :

```bash
diff deployment-qui-marche.yaml deployment-probleme.yaml
```

### 4. Isolez le problème

Si tout est cassé, isolez un composant à la fois :
- Testez d'abord si le pod démarre seul
- Puis testez si le service fonctionne
- Ensuite testez l'Ingress
- Etc.

### 5. Prenez des notes

Pendant votre investigation, notez :
- Les commandes que vous exécutez
- Les observations que vous faites
- Les choses que vous avez essayées

Cela vous évitera de refaire les mêmes vérifications et vous aidera à comprendre votre progression.

### 6. Ne restez pas bloqué trop longtemps

Si après 30-45 minutes vous n'avancez pas :
- Faites une pause
- Expliquez le problème à quelqu'un (ou à un canard en plastique - c'est une vraie technique !)
- Cherchez de l'aide sur les forums ou Discord Kubernetes

### 7. Apprenez de chaque problème

Chaque problème résolu est une victoire et une leçon. Après avoir résolu un problème :
- Comprenez pourquoi c'est arrivé
- Réfléchissez à comment l'éviter à l'avenir
- Documentez la solution

## Erreurs Courantes à Éviter

### ❌ Modifier plusieurs choses en même temps

**Problème** : Vous ne saurez pas ce qui a résolu le problème.

**Solution** : Changez une chose à la fois, observez le résultat, puis passez à la suivante si nécessaire.

### ❌ Ignorer les warnings

**Problème** : Les warnings sont souvent des signes avant-coureurs de problèmes plus graves.

**Solution** : Prenez le temps de comprendre chaque warning même si tout semble fonctionner.

### ❌ Supprimer des pods sans comprendre pourquoi

**Problème** : Le pod va redémarrer, mais le problème sous-jacent persistera.

**Solution** : Comprenez d'abord la cause avant de supprimer.

### ❌ Ne pas consulter les logs assez tôt

**Problème** : Les logs contiennent souvent la réponse directe à votre question.

**Solution** : Consultez systématiquement les logs dès les premières étapes.

### ❌ Oublier le contexte de namespace

**Problème** : Vous cherchez des ressources dans le mauvais namespace.

**Solution** : Vérifiez toujours dans quel namespace vous travaillez ou utilisez `--all-namespaces`.

## Conclusion

Le dépannage est une compétence qui s'acquiert avec le temps et la pratique. Au début, vous utiliserez cette méthodologie de manière très consciente, étape par étape. Avec l'expérience, elle deviendra naturelle et vous pourrez diagnostiquer certains problèmes en quelques secondes.

**Points clés à retenir** :

1. **Soyez méthodique** : Suivez les étapes dans l'ordre
2. **Collectez les faits** : Ne devinez pas, observez
3. **Une chose à la fois** : Changez un paramètre, observez, puis continuez
4. **Documentez** : Votre futur vous remerciera
5. **Restez patient** : Chaque problème est une opportunité d'apprendre

Dans les sections suivantes, nous approfondirons les outils spécifiques de diagnostic comme `microk8s inspect`, `kubectl logs` avancé, et les problèmes courants par catégorie (réseau, stockage, certificats, etc.).

**Prochaine étape** : Section 23.2 - Commandes de diagnostic essentielles

---

*Cette section fait partie du tutoriel complet MicroK8s - Du Débutant à l'Avancé*

⏭️ [Commandes de diagnostic essentielles](/23-depannage-et-maintenance/02-commandes-de-diagnostic-essentielles.md)
