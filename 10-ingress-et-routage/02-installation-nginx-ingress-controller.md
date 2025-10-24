🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 10.2 Installation NGINX Ingress Controller (microk8s enable ingress)

## Introduction

Maintenant que vous comprenez ce qu'est un Ingress et pourquoi il est utile, il est temps de l'installer dans votre cluster MicroK8s. La bonne nouvelle ? Avec MicroK8s, c'est d'une simplicité déconcertante ! Une seule commande suffit.

Dans ce chapitre, nous allons voir comment activer NGINX Ingress Controller, comprendre ce qui se passe en coulisses, et vérifier que tout fonctionne correctement.

## Pourquoi NGINX Ingress Controller ?

NGINX Ingress Controller est l'implémentation d'Ingress la plus populaire dans l'écosystème Kubernetes. Voici pourquoi :

✅ **Mature et fiable** : utilisé en production par des milliers d'entreprises
✅ **Performant** : capable de gérer des millions de requêtes
✅ **Riche en fonctionnalités** : supporte toutes les fonctionnalités avancées
✅ **Bien documenté** : documentation exhaustive et communauté active
✅ **Gratuit et open-source** : maintenu par la communauté Kubernetes

Dans MicroK8s, NGINX Ingress Controller est fourni comme addon officiel, ce qui signifie qu'il est :
- Pré-configuré et prêt à l'emploi
- Maintenu et testé par l'équipe MicroK8s
- Compatible avec les autres addons MicroK8s

## Prérequis

Avant d'activer l'Ingress Controller, assurez-vous que :

### 1. MicroK8s est Installé et Fonctionnel

Vérifiez l'état de votre cluster :

```bash
microk8s status
```

Vous devriez voir quelque chose comme :
```
microk8s is running
high-availability: no
  datastore master nodes: 127.0.0.1:19001
  datastore standby nodes: none
```

### 2. DNS est Activé (Fortement Recommandé)

L'Ingress Controller a besoin de DNS pour fonctionner correctement. Si ce n'est pas déjà fait, activez-le :

```bash
microk8s enable dns
```

Le DNS permet aux pods de résoudre les noms de services internes, ce qui est essentiel pour le bon fonctionnement de l'Ingress.

### 3. Vous Avez les Droits Administrateur

Certaines opérations nécessitent des droits sudo. Assurez-vous d'avoir les permissions nécessaires.

## Installation : La Magie d'une Seule Commande

### Activation de l'Addon Ingress

Pour installer NGINX Ingress Controller, exécutez simplement :

```bash
microk8s enable ingress
```

C'est tout ! Vous devriez voir une sortie similaire à :

```
Infer repository core for addon ingress
Enabling Ingress
ingressclass.networking.k8s.io/public created
ingressclass.networking.k8s.io/nginx created
namespace/ingress created
serviceaccount/nginx-ingress-microk8s-serviceaccount created
clusterrole.rbac.authorization.k8s.io/nginx-ingress-microk8s-clusterrole created
role.rbac.authorization.k8s.io/nginx-ingress-microk8s-role created
clusterrolebinding.rbac.authorization.k8s.io/nginx-ingress-microk8s created
rolebinding.rbac.authorization.k8s.io/nginx-ingress-microk8s created
configmap/nginx-load-balancer-microk8s-conf created
configmap/nginx-ingress-tcp-microk8s-conf created
configmap/nginx-ingress-udp-microk8s-conf created
daemonset.apps/nginx-ingress-microk8s-controller created
Ingress is enabled
```

### Que S'est-il Passé ?

Lors de l'exécution de cette commande, MicroK8s a automatiquement :

1. **Créé un namespace dédié** : `ingress`
   - C'est un espace isolé pour les composants de l'Ingress Controller

2. **Déployé le contrôleur NGINX** : sous forme de DaemonSet
   - Un DaemonSet garantit qu'un pod Ingress s'exécute sur chaque nœud
   - Dans un cluster mono-nœud, il y aura un seul pod

3. **Configuré les permissions RBAC** : ServiceAccount, ClusterRole, RoleBinding
   - Donne au contrôleur les permissions nécessaires pour lire les ressources Ingress

4. **Créé les IngressClass** : `public` et `nginx`
   - Permet de définir quel contrôleur gère quelles ressources Ingress

5. **Créé des ConfigMaps** : pour la configuration NGINX
   - Stocke les paramètres de configuration du serveur NGINX

## Vérification de l'Installation

### 1. Vérifier le Statut de l'Addon

Confirmez que l'addon est bien activé :

```bash
microk8s status
```

Dans la liste des addons, vous devriez voir :
```
ingress              # (core) NGINX Ingress Controller
```

Ou de manière plus détaillée :

```bash
microk8s status --format yaml | grep ingress
```

### 2. Vérifier le Namespace Ingress

Listez les namespaces pour confirmer la création du namespace `ingress` :

```bash
microk8s kubectl get namespaces
```

Vous devriez voir :
```
NAME              STATUS   AGE
default           Active   10d
kube-system       Active   10d
kube-public       Active   10d
kube-node-lease   Active   10d
ingress           Active   2m
```

### 3. Vérifier les Pods de l'Ingress Controller

Vérifiez que le pod du contrôleur NGINX s'exécute correctement :

```bash
microk8s kubectl get pods -n ingress
```

Sortie attendue :
```
NAME                                      READY   STATUS    RESTARTS   AGE
nginx-ingress-microk8s-controller-xxxxx   1/1     Running   0          3m
```

**Points à vérifier** :
- `READY` doit être `1/1` (le pod est prêt)
- `STATUS` doit être `Running` (le pod fonctionne)
- `RESTARTS` devrait être `0` ou un nombre faible (pas de redémarrages fréquents)

### 4. Vérifier les Logs du Contrôleur

Pour voir ce que fait le contrôleur, consultez ses logs :

```bash
microk8s kubectl logs -n ingress -l app.kubernetes.io/name=ingress-nginx
```

Vous devriez voir des messages indiquant que NGINX a démarré avec succès :
```
-------------------------------------------------------------------------------
NGINX Ingress controller
  Release:       v1.x.x
  Build:         xxxxx
  Repository:    https://github.com/kubernetes/ingress-nginx
-------------------------------------------------------------------------------

W1024 10:15:30.123456       7 client_config.go:617] Neither --kubeconfig nor --master was specified.  Using the inClusterConfig.
I1024 10:15:30.234567       7 main.go:205] Creating API client for https://10.152.183.1:443
I1024 10:15:30.345678       7 main.go:249] Running in Kubernetes cluster version v1.28 (v1.28.x) - git (clean) commit xxxxx
I1024 10:15:31.456789       7 nginx.go:271] Starting NGINX Ingress controller
```

### 5. Vérifier le DaemonSet

Le contrôleur est déployé comme un DaemonSet. Vérifiez-le :

```bash
microk8s kubectl get daemonsets -n ingress
```

Sortie :
```
NAME                                DESIRED   CURRENT   READY   UP-TO-DATE   AVAILABLE   NODE SELECTOR   AGE
nginx-ingress-microk8s-controller   1         1         1       1            1           <none>          5m
```

**Explications** :
- `DESIRED` : nombre de pods souhaité (1 par nœud)
- `CURRENT` : nombre de pods actuellement en cours d'exécution
- `READY` : nombre de pods prêts à recevoir du trafic
- Ces trois valeurs doivent être égales

### 6. Vérifier les Services

Listez les services dans le namespace ingress :

```bash
microk8s kubectl get services -n ingress
```

Selon la configuration, vous pourriez voir :
```
NAME                       TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)   AGE
ingress-nginx-controller   ClusterIP   10.152.183.xx   <none>        80/TCP    5m
```

Note : Le type de service peut varier selon la configuration MicroK8s.

### 7. Vérifier les IngressClass

Les IngressClass définissent quel contrôleur gère vos ressources Ingress :

```bash
microk8s kubectl get ingressclass
```

Sortie attendue :
```
NAME     CONTROLLER                     PARAMETERS   AGE
nginx    k8s.io/ingress-nginx           <none>       5m
public   k8s.io/ingress-nginx           <none>       5m
```

L'IngressClass `nginx` est généralement celle par défaut. Vous pouvez vérifier laquelle est la classe par défaut :

```bash
microk8s kubectl get ingressclass -o yaml
```

Cherchez l'annotation `ingressclass.kubernetes.io/is-default-class: "true"`.

## Comprendre l'Architecture Installée

Voici comment les composants sont organisés après l'installation :

```
┌─────────────────────────────────────────────┐
│         Cluster MicroK8s                    │
│                                             │
│  ┌────────────────────────────────────┐     │
│  │  Namespace: ingress                │     │
│  │                                    │     │
│  │  ┌──────────────────────────────┐  │     │
│  │  │  DaemonSet:                  │  │     │
│  │  │  nginx-ingress-controller    │  │     │
│  │  │                              │  │     │
│  │  │  ┌────────────────────────┐  │  │     │
│  │  │  │  Pod: nginx-ingress    │  │  │     │
│  │  │  │  - Port 80 (HTTP)      │  │  │     │
│  │  │  │  - Port 443 (HTTPS)    │  │  │     │
│  │  │  │  - Lit les Ingress     │  │  │     │
│  │  │  │  - Configure NGINX     │  │  │     │
│  │  │  └────────────────────────┘  │  │     │
│  │  └──────────────────────────────┘  │     │
│  │                                    │     │
│  │  ConfigMaps:                       │     │
│  │  - nginx-load-balancer-conf        │     │
│  │  - nginx-ingress-tcp-conf          │     │
│  │  - nginx-ingress-udp-conf          │     │
│  └────────────────────────────────────┘     │
│                                             │
│  IngressClass: nginx, public                │
│                                             │
└─────────────────────────────────────────────┘
```

### Flux de Fonctionnement

1. Vous créez une ressource **Ingress** avec vos règles de routage
2. Le **contrôleur NGINX** détecte cette nouvelle ressource (via l'API Kubernetes)
3. Le contrôleur **génère automatiquement une configuration NGINX** basée sur vos règles
4. Le contrôleur **recharge NGINX** avec cette nouvelle configuration
5. NGINX commence à **router le trafic** selon vos règles

Tout cela se fait automatiquement sans intervention manuelle !

## Configuration Post-Installation

### Ports Exposés

Par défaut, l'Ingress Controller écoute sur :
- **Port 80** : pour le trafic HTTP
- **Port 443** : pour le trafic HTTPS

Ces ports sont exposés au niveau de l'hôte (le serveur qui exécute MicroK8s), ce qui signifie que vous pouvez y accéder directement.

### Accès Externe

Pour accéder à vos applications via l'Ingress depuis l'extérieur de votre machine MicroK8s, vous devrez :

1. **Configurer votre DNS** pour pointer vers l'IP de votre serveur MicroK8s
2. **Ouvrir les ports** 80 et 443 dans votre pare-feu
3. **Configurer le routage réseau** si nécessaire (NAT, port forwarding)

Nous verrons ces points en détail dans les sections suivantes du chapitre 10.

## Personnalisation de la Configuration

### ConfigMaps NGINX

Les ConfigMaps créés permettent de personnaliser le comportement de NGINX :

```bash
microk8s kubectl get configmap -n ingress
```

Vous pouvez éditer ces ConfigMaps pour ajuster la configuration NGINX. Par exemple :

```bash
microk8s kubectl edit configmap nginx-load-balancer-microk8s-conf -n ingress
```

Quelques options courantes :
- `proxy-connect-timeout` : timeout de connexion
- `proxy-read-timeout` : timeout de lecture
- `client-max-body-size` : taille maximale des requêtes
- `use-forwarded-headers` : utiliser les en-têtes X-Forwarded

Pour une liste complète des options : [NGINX Ingress Configuration](https://kubernetes.github.io/ingress-nginx/user-guide/nginx-configuration/configmap/)

### Exemple de Personnalisation

Pour augmenter la taille maximale des uploads :

```bash
microk8s kubectl edit configmap nginx-load-balancer-microk8s-conf -n ingress
```

Ajoutez sous `data:` :
```yaml
data:
  proxy-body-size: "50m"
  client-max-body-size: "50m"
```

Sauvegardez. Le contrôleur rechargera automatiquement NGINX avec cette nouvelle configuration.

## Désactivation de l'Ingress (Si Nécessaire)

Si vous devez désactiver l'Ingress Controller :

```bash
microk8s disable ingress
```

Cette commande :
- Supprime tous les composants de l'Ingress
- Supprime le namespace `ingress`
- N'affecte PAS vos ressources Ingress existantes (elles resteront mais ne seront plus traitées)

**Attention** : désactiver l'Ingress rendra toutes vos applications exposées via Ingress inaccessibles !

## Réactivation

Si vous avez désactivé l'Ingress et souhaitez le réactiver :

```bash
microk8s enable ingress
```

Vos ressources Ingress existantes seront automatiquement détectées et appliquées.

## Dépannage Commun

### Problème 1 : Le Pod ne Démarre Pas

**Symptôme** :
```bash
microk8s kubectl get pods -n ingress
NAME                                      READY   STATUS    RESTARTS   AGE
nginx-ingress-microk8s-controller-xxxxx   0/1     Pending   0          3m
```

**Solutions possibles** :
1. Vérifier les ressources disponibles :
   ```bash
   microk8s kubectl describe pod -n ingress nginx-ingress-microk8s-controller-xxxxx
   ```

2. Vérifier les événements du namespace :
   ```bash
   microk8s kubectl get events -n ingress
   ```

### Problème 2 : Le Pod Redémarre en Boucle

**Symptôme** :
```bash
NAME                                      READY   STATUS             RESTARTS   AGE
nginx-ingress-microk8s-controller-xxxxx   0/1     CrashLoopBackOff   5          3m
```

**Solutions** :
1. Consulter les logs :
   ```bash
   microk8s kubectl logs -n ingress nginx-ingress-microk8s-controller-xxxxx
   ```

2. Vérifier les permissions RBAC :
   ```bash
   microk8s kubectl get clusterrolebinding nginx-ingress-microk8s
   ```

### Problème 3 : DNS Non Activé

**Symptôme** : L'Ingress fonctionne mal ou les logs montrent des erreurs de résolution DNS.

**Solution** :
```bash
microk8s enable dns
```

Attendez que le pod DNS soit prêt :
```bash
microk8s kubectl get pods -n kube-system -l k8s-app=kube-dns
```

Puis redémarrez l'Ingress :
```bash
microk8s kubectl rollout restart daemonset nginx-ingress-microk8s-controller -n ingress
```

## Vérification Complète : Checklist

Avant de passer aux sections suivantes, assurez-vous que :

- [ ] L'addon `ingress` est marqué comme activé dans `microk8s status`
- [ ] Le namespace `ingress` existe
- [ ] Le pod NGINX Ingress Controller est en état `Running` avec `READY 1/1`
- [ ] Le DaemonSet a tous ses pods prêts (`DESIRED = CURRENT = READY`)
- [ ] Les logs ne montrent pas d'erreurs critiques
- [ ] Les IngressClass `nginx` et `public` existent
- [ ] L'addon `dns` est activé (fortement recommandé)

## Commandes de Référence Rapide

```bash
# Activer l'Ingress
microk8s enable ingress

# Vérifier le statut
microk8s status | grep ingress

# Lister les pods de l'Ingress
microk8s kubectl get pods -n ingress

# Voir les logs
microk8s kubectl logs -n ingress -l app.kubernetes.io/name=ingress-nginx

# Vérifier le DaemonSet
microk8s kubectl get daemonset -n ingress

# Lister les IngressClass
microk8s kubectl get ingressclass

# Désactiver l'Ingress (attention !)
microk8s disable ingress
```

## Points Clés à Retenir

🔑 **Installation ultra-simple** : une seule commande `microk8s enable ingress`

🔑 **NGINX Ingress Controller** déployé automatiquement dans le namespace `ingress`

🔑 **DaemonSet** : garantit un pod Ingress sur chaque nœud du cluster

🔑 **Ports 80 et 443** exposés par défaut pour HTTP et HTTPS

🔑 **Configuration personnalisable** via les ConfigMaps

🔑 **IngressClass** : définit quel contrôleur gère vos ressources Ingress

🔑 **DNS recommandé** : activez `dns` avant `ingress` pour éviter les problèmes

🔑 **Logs et diagnostics** : utilisez `kubectl logs` et `kubectl describe` en cas de problème

## Prochaines Étapes

Maintenant que NGINX Ingress Controller est installé et opérationnel, vous êtes prêt à :

1. **Configurer votre DNS** (section 10.4 et 10.5)
2. **Créer vos premières règles Ingress** (sections 10.8 et 10.9)
3. **Ajouter des certificats SSL/TLS** (chapitre 11)
4. **Exposer vos applications au monde** (section 10.6 et 10.7)

Félicitations ! Vous avez franchi une étape importante dans la maîtrise de Kubernetes. L'Ingress Controller est maintenant prêt à router vos applications de manière professionnelle.

---

**📚 Résumé du chapitre** : L'installation de NGINX Ingress Controller dans MicroK8s se fait en une seule commande : `microk8s enable ingress`. Cette commande déploie automatiquement tous les composants nécessaires dans un namespace dédié. Après vérification que le pod est en état Running, votre cluster est prêt à recevoir des règles de routage Ingress pour exposer vos applications.

⏭️ [Configuration NGINX Ingress Controller](/10-ingress-et-routage/03-configuration-nginx-ingress-controller.md)
