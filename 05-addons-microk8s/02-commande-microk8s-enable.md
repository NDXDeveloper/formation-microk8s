🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 5.2 Commande microk8s enable : activer des fonctionnalités en un clic

## Introduction

Maintenant que vous comprenez la philosophie des addons MicroK8s, il est temps de découvrir l'outil qui va vous permettre de les utiliser : la commande **`microk8s enable`**. Cette commande est véritablement magique dans sa simplicité, et c'est elle qui transforme MicroK8s en un outil si puissant et accessible.

## La commande la plus importante de MicroK8s

Si vous ne deviez retenir qu'une seule commande MicroK8s (en dehors de `kubectl`), ce serait celle-ci :

```bash
microk8s enable <nom-addon>
```

Cette commande unique vous permet d'activer n'importe quelle fonctionnalité disponible dans MicroK8s. C'est votre baguette magique pour enrichir votre cluster.

## Anatomie de la commande

Décortiquons la commande pour bien comprendre chaque élément :

```bash
microk8s enable dashboard
│         │      │
│         │      └─ Nom de l'addon à activer
│         └─ Action : activer/enable
└─ Programme MicroK8s
```

### Les éléments de base

- **`microk8s`** : C'est le programme principal, le point d'entrée pour toutes les commandes spécifiques à MicroK8s
- **`enable`** : C'est l'action que vous voulez effectuer (activer un addon)
- **`<nom-addon>`** : C'est le nom de l'addon que vous souhaitez activer (par exemple : `dashboard`, `dns`, `registry`, etc.)

## Utilisation basique

### Activer un addon simple

La forme la plus simple de la commande est :

```bash
microk8s enable dns
```

Quand vous exécutez cette commande :
1. MicroK8s vérifie que l'addon existe
2. Il télécharge les composants nécessaires (si ce n'est pas déjà fait)
3. Il configure automatiquement l'addon
4. Il démarre les services requis
5. Il vous confirme que l'addon est actif

**Sortie typique :**
```
Infer repository core for addon dns
Addon core/dns is already enabled
```

Ou si l'addon n'était pas encore activé :
```
Infer repository core for addon dns
Enabling DNS
Applying manifest
...
DNS is enabled
```

### Ce qui se passe en coulisses

Lorsque vous activez un addon, MicroK8s effectue automatiquement plusieurs opérations complexes :

1. **Téléchargement** : Si nécessaire, télécharge les images Docker et les fichiers de configuration
2. **Configuration** : Génère les fichiers de configuration adaptés à votre environnement
3. **Déploiement** : Crée les ressources Kubernetes nécessaires (pods, services, deployments, etc.)
4. **Vérification** : S'assure que tout fonctionne correctement
5. **Intégration** : Configure l'intégration avec les autres composants du cluster

Tout cela se fait **automatiquement**, sans que vous ayez à vous en préoccuper. C'est la magie de MicroK8s !

## Activer plusieurs addons en une commande

Vous pouvez activer plusieurs addons d'un coup en les listant les uns après les autres :

```bash
microk8s enable dns dashboard storage
```

Cette commande activera successivement :
1. L'addon DNS
2. L'addon Dashboard
3. L'addon Storage

MicroK8s les active dans l'ordre où vous les avez spécifiés. C'est particulièrement utile quand vous configurez un nouveau cluster et que vous voulez activer plusieurs fonctionnalités de base en une seule fois.

### Ordre d'activation

Dans certains cas, l'ordre peut être important. Par exemple, il est recommandé d'activer le DNS avant d'autres addons qui pourraient en dépendre :

```bash
# Bon ordre : DNS d'abord, puis les autres
microk8s enable dns ingress cert-manager
```

Cependant, MicroK8s gère généralement bien les dépendances, donc ne vous inquiétez pas trop de l'ordre dans la plupart des cas.

## Addons avec configuration

Certains addons nécessitent ou acceptent des paramètres de configuration lors de l'activation. La syntaxe générale est :

```bash
microk8s enable <nom-addon>:<paramètre>
```

### Exemple avec MetalLB

MetalLB est un load balancer qui nécessite de spécifier une plage d'adresses IP :

```bash
microk8s enable metallb:192.168.1.100-192.168.1.110
```

Ici, vous indiquez que MetalLB pourra utiliser les adresses IP de 192.168.1.100 à 192.168.1.110 pour vos services.

**Explication pour débutants :** Imaginez MetalLB comme un parking qui a besoin de savoir quelles places de stationnement il peut attribuer aux voitures (vos services). Vous lui donnez une plage de places disponibles.

### Exemple avec le Registry

Le registry (registre d'images) peut être configuré avec une taille de stockage :

```bash
microk8s enable registry:size=20Gi
```

Cela crée un registry avec 20 Go d'espace de stockage pour vos images Docker.

### Paramètres multiples

Certains addons acceptent plusieurs paramètres, séparés par des virgules ou des espaces (selon l'addon) :

```bash
microk8s enable rbac
```

Pour connaître les paramètres disponibles pour un addon spécifique, vous pouvez consulter la documentation ou utiliser :

```bash
microk8s enable <nom-addon> --help
```

Bien que cette fonctionnalité ne soit pas disponible pour tous les addons.

## Vérifier l'état d'un addon

Avant ou après avoir activé un addon, vous voudrez probablement vérifier son état. Utilisez la commande `status` :

```bash
microk8s status
```

**Sortie typique :**
```
microk8s is running
high-availability: no
  datastore master nodes: 127.0.0.1:19001
  datastore standby nodes: none
addons:
  enabled:
    dashboard            # (core) The Kubernetes dashboard
    dns                  # (core) CoreDNS
    ha-cluster           # (core) Configure high availability on the current node
    helm                 # (core) Helm - the package manager for Kubernetes
  disabled:
    cert-manager         # (core) Cloud native certificate management
    community            # (community) The community addons repository
    gpu                  # (core) Automatic enablement of Nvidia CUDA
    host-access          # (core) Allow Pods connecting to Host services smoothly
    hostpath-storage     # (core) Storage class; allocates storage from host directory
```

Cette sortie vous montre :
- **enabled** : Les addons qui sont actuellement actifs
- **disabled** : Les addons disponibles mais non activés

### Vérification détaillée

Pour avoir plus d'informations sur l'état du cluster et des addons, utilisez :

```bash
microk8s inspect
```

Cette commande génère un rapport détaillé incluant :
- L'état de tous les composants Kubernetes
- Les logs récents
- Les problèmes potentiels
- Des suggestions de résolution

C'est votre outil de diagnostic principal en cas de problème.

## Temps d'activation

Le temps nécessaire pour activer un addon varie selon plusieurs facteurs :

### Addons rapides (quelques secondes)
- **DNS** : ~5-10 secondes
- **Dashboard** : ~10-20 secondes
- **RBAC** : ~2-5 secondes

### Addons moyens (quelques minutes)
- **Storage** : ~30 secondes
- **Registry** : ~1-2 minutes
- **Ingress** : ~1-2 minutes

### Addons longs (plusieurs minutes)
- **Prometheus** : ~3-5 minutes (beaucoup de composants)
- **Cert-Manager** : ~2-3 minutes
- **Istio** : ~5-10 minutes (très complexe)

**Note** : Le premier téléchargement des images peut prendre plus de temps selon votre connexion Internet. Les activations suivantes sont plus rapides car les images sont déjà en cache.

### Comprendre ce qui prend du temps

Pendant l'activation, MicroK8s doit :
1. **Télécharger** les images Docker (si première fois) - peut être long selon votre connexion
2. **Créer** les ressources Kubernetes - rapide
3. **Démarrer** les pods - rapide
4. **Attendre** que les pods soient prêts (health checks) - variable

Pour les addons complexes comme Prometheus, plusieurs pods doivent démarrer et être prêts, ce qui explique le temps d'attente.

## Messages courants et leur signification

### Succès
```
Enabling DNS
Applying manifest
serviceaccount/coredns created
configmap/coredns created
deployment.apps/coredns created
service/kube-dns created
clusterrole.rbac.authorization.k8s.io/coredns created
clusterrolebinding.rbac.authorization.k8s.io/coredns created
DNS is enabled
```

Ce message détaillé vous montre toutes les ressources Kubernetes créées. Tout s'est bien passé !

### Addon déjà activé
```
Addon core/dns is already enabled
```

Pas d'inquiétude, cela signifie simplement que l'addon était déjà actif. MicroK8s ne le réactive pas inutilement.

### Erreur de nom
```
Infer repository core for addon dashbaord
Error: Addon dashbaord not found
```

Vous avez fait une faute de frappe dans le nom de l'addon. Vérifiez l'orthographe !

### Erreur de dépendance
```
Enabling Cert-Manager requires DNS to be enabled
Please enable DNS first: microk8s enable dns
```

Certains addons dépendent d'autres. MicroK8s vous indique clairement quoi faire.

## Patienter pendant l'activation

Quand vous activez un addon, la commande peut sembler "figée" pendant quelques instants. C'est normal ! MicroK8s est en train de :
- Télécharger les images
- Créer les ressources
- Attendre que tout soit prêt

### Que faire pendant l'attente ?

**Option 1 : Patienter**
Simplement attendre que la commande se termine. C'est l'option la plus simple.

**Option 2 : Surveiller dans un autre terminal**
Ouvrez un second terminal et observez ce qui se passe :

```bash
# Voir les pods en cours de création
microk8s kubectl get pods --all-namespaces -w

# Le -w (watch) permet de voir les changements en temps réel
```

Vous verrez les pods passer de l'état `Pending` à `ContainerCreating` puis à `Running`. C'est fascinant de voir le cluster prendre vie !

## Bonnes pratiques d'activation

### 1. Activer les addons de base en premier

Commencez toujours par les addons fondamentaux :

```bash
microk8s enable dns dashboard storage
```

Ces trois addons constituent une base solide pour travailler.

### 2. Activer un addon à la fois (au début)

Quand vous débutez, activez les addons un par un pour :
- Comprendre l'effet de chaque addon
- Identifier plus facilement les problèmes
- Apprendre progressivement

Une fois à l'aise, vous pourrez en activer plusieurs simultanément.

### 3. Vérifier l'état après activation

Après chaque activation, prenez l'habitude de vérifier :

```bash
microk8s status
```

Cela vous confirme que l'addon est bien activé et vous familiarise avec les commandes de diagnostic.

### 4. Lire les messages de sortie

Les messages affichés lors de l'activation contiennent souvent des informations utiles :
- URL pour accéder à un service
- Tokens d'authentification
- Prochaines étapes suggérées

Ne les ignorez pas, ils sont là pour vous aider !

## Cas particuliers et astuces

### Réactiver un addon

Si un addon ne fonctionne pas correctement, vous pouvez le désactiver puis le réactiver :

```bash
microk8s disable dashboard
microk8s enable dashboard
```

Cela réinitialise complètement l'addon. C'est souvent une solution rapide aux problèmes mineurs.

### Activation avec sudo

Sur certains systèmes, vous pourriez avoir besoin d'utiliser `sudo` :

```bash
sudo microk8s enable dns
```

Cependant, si votre utilisateur a été ajouté au groupe `microk8s` lors de l'installation, ce n'est normalement pas nécessaire.

### Addons communautaires

En plus des addons officiels (core), MicroK8s propose des addons communautaires. Pour les utiliser, activez d'abord le dépôt communautaire :

```bash
microk8s enable community
```

Ensuite, vous aurez accès à des addons supplémentaires développés par la communauté.

## Visualiser ce qu'un addon fait réellement

Si vous êtes curieux de voir concrètement ce qu'un addon déploie dans votre cluster, vous pouvez inspecter les ressources créées :

```bash
# Lister tous les pods (après avoir activé le dashboard, par exemple)
microk8s kubectl get pods --all-namespaces

# Voir les services créés
microk8s kubectl get services --all-namespaces

# Voir les deployments
microk8s kubectl get deployments --all-namespaces
```

Vous verrez apparaître de nouvelles ressources correspondant à l'addon que vous venez d'activer. C'est une excellente façon d'apprendre comment Kubernetes fonctionne !

## Commandes complémentaires utiles

### Lister tous les addons disponibles

```bash
microk8s status
```

Affiche tous les addons disponibles avec leur état (enabled/disabled).

### Obtenir de l'aide

```bash
microk8s enable --help
```

Affiche l'aide générale sur la commande enable.

### Version de MicroK8s

```bash
microk8s version
```

Utile pour savoir quelle version de MicroK8s (et donc de Kubernetes) vous utilisez. Les addons disponibles peuvent varier selon la version.

## Dépannage des activations

### L'activation ne se termine jamais

Si une commande `enable` semble bloquée :

1. **Patientez** : Pour les gros addons comme Prometheus, cela peut prendre 5-10 minutes
2. **Vérifiez votre connexion Internet** : Le téléchargement d'images peut être long
3. **Ouvrez un autre terminal** et vérifiez l'état :
   ```bash
   microk8s kubectl get pods --all-namespaces
   ```
4. **Consultez les logs** :
   ```bash
   microk8s inspect
   ```

### Message d'erreur après activation

Si vous recevez un message d'erreur :

1. **Lisez attentivement le message** : Il contient souvent la solution
2. **Vérifiez les dépendances** : Certains addons nécessitent que d'autres soient actifs
3. **Consultez les logs** avec `microk8s inspect`
4. **Recherchez l'erreur** : Une simple recherche Google avec le message d'erreur aide souvent

### L'addon est activé mais ne fonctionne pas

1. **Vérifiez l'état des pods** :
   ```bash
   microk8s kubectl get pods --all-namespaces
   ```
   Tous les pods doivent être en état `Running`

2. **Consultez les logs du pod problématique** :
   ```bash
   microk8s kubectl logs <nom-du-pod> -n <namespace>
   ```

3. **Réactivez l'addon** :
   ```bash
   microk8s disable <addon>
   microk8s enable <addon>
   ```

## Exemple complet : Activer le Dashboard

Voici un exemple complet et commenté de l'activation du Dashboard Kubernetes :

```bash
# 1. Vérifier l'état actuel
$ microk8s status
# Le dashboard devrait apparaître comme "disabled"

# 2. Activer le dashboard
$ microk8s enable dashboard
Infer repository core for addon dashboard
Enabling Kubernetes Dashboard
Infer repository core for addon metrics-server
Enabling Metrics-Server
serviceaccount/metrics-server created
clusterrole.rbac.authorization.k8s.io/system:aggregated-metrics-reader created
...
Metrics-Server is enabled
Applying manifest
...
Kubernetes Dashboard is enabled

# 3. Vérifier que c'est activé
$ microk8s status
# Le dashboard devrait maintenant apparaître comme "enabled"

# 4. Voir les ressources créées
$ microk8s kubectl get pods -n kube-system | grep dashboard
kubernetes-dashboard-XXXXX   1/1     Running   0          1m

# 5. Accéder au dashboard (nous verrons cela en détail dans la section 5.3)
$ microk8s dashboard-proxy
```

Notez que l'activation du dashboard a également activé automatiquement `metrics-server`, car c'est une dépendance. MicroK8s gère cela pour vous !

## Récapitulatif

La commande `microk8s enable` est votre porte d'entrée vers toutes les fonctionnalités de MicroK8s. Retenez :

✅ **Syntaxe de base** : `microk8s enable <nom-addon>`
✅ **Plusieurs addons** : `microk8s enable addon1 addon2 addon3`
✅ **Avec configuration** : `microk8s enable addon:paramètre`
✅ **Vérifier l'état** : `microk8s status`
✅ **Diagnostic** : `microk8s inspect`

## Ce que vous avez appris

Dans cette section, vous avez découvert :
- Comment activer des addons en une seule commande
- Comment activer plusieurs addons simultanément
- Comment passer des paramètres aux addons
- Comment vérifier l'état des addons
- Les bonnes pratiques d'activation
- Comment diagnostiquer les problèmes d'activation

## Prochaine étape

Maintenant que vous maîtrisez la commande `microk8s enable`, vous êtes prêt à découvrir les addons essentiels en détail. Dans les sections suivantes, nous allons explorer quatre addons fondamentaux que vous utiliserez constamment :

1. **Dashboard** (section 5.3) - L'interface graphique de votre cluster
2. **DNS** (section 5.4) - La communication entre vos services
3. **Registry** (section 5.5) - Votre registre d'images privé
4. **Storage** (section 5.6) - Le stockage persistant de vos données

Chaque section vous montrera non seulement comment activer l'addon, mais aussi comment l'utiliser concrètement dans vos projets.

Prêt à activer votre premier addon ? Passons au Dashboard !

---

**Note pratique** : Gardez toujours un terminal ouvert avec `microk8s status` à portée de main. C'est votre tableau de bord textuel pour savoir rapidement quels addons sont actifs dans votre cluster.

⏭️ [Dashboard Kubernetes (microk8s enable dashboard)](/05-addons-microk8s/03-dashboard-kubernetes.md)
