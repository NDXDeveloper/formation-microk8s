🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 5.4 DNS (CoreDNS - microk8s enable dns)

## Introduction

Le **DNS** (Domain Name System) est probablement l'addon le plus fondamental de tous. Sans DNS, vos applications dans Kubernetes ne peuvent pas communiquer entre elles de manière fiable. Si le Dashboard est les yeux de votre cluster, le DNS en est le système nerveux qui permet à toutes les parties de se trouver et de communiquer.

## Qu'est-ce que le DNS ?

### Dans le monde réel

Le DNS est comme l'annuaire téléphonique d'Internet. Quand vous tapez `google.com` dans votre navigateur, votre ordinateur demande au DNS : "Quelle est l'adresse IP de google.com ?" Le DNS répond : "C'est 142.250.185.46". Votre ordinateur peut alors se connecter à Google.

Sans DNS, vous devriez mémoriser des adresses IP pour chaque site web. Imaginez devoir taper `142.250.185.46` au lieu de `google.com` !

### Dans Kubernetes

Le même principe s'applique à l'intérieur de votre cluster Kubernetes. Vos applications (pods et services) ont besoin de communiquer entre elles, mais :

- **Avec DNS** : Une application peut appeler une autre en utilisant un nom simple comme `my-database` ou `api-service`
- **Sans DNS** : Il faudrait connaître et utiliser l'adresse IP exacte de chaque pod, qui change à chaque redémarrage

### Analogie simple

Imaginez un grand hôtel :

- **Sans DNS** : Vous devez mémoriser le numéro de chambre exact de chaque personne. "Jean est en chambre 3547, Marie en 2189, Paul en 4032..." Et quand quelqu'un change de chambre, vous devez mettre à jour toutes vos notes.

- **Avec DNS** : Vous appelez simplement la réception et demandez "Passez-moi Jean, s'il vous plaît". La réception sait où le trouver et vous met en contact. Si Jean change de chambre, la réception met à jour ses registres, mais vous continuez à demander "Jean" sans vous soucier du numéro.

Le DNS dans Kubernetes joue le rôle de cette réception : il maintient un annuaire à jour de tous les services et permet aux applications de se trouver par leur nom.

## Pourquoi le DNS est-il si important ?

### 1. Abstraction des adresses IP

Dans Kubernetes, les pods sont **éphémères** : ils peuvent être détruits et recréés à tout moment. À chaque fois, ils obtiennent une nouvelle adresse IP.

Sans DNS :
- Votre application frontend doit connaître l'IP exacte du backend
- Si le backend redémarre → nouvelle IP → votre frontend ne peut plus le joindre
- Vous devez constamment mettre à jour les configurations

Avec DNS :
- Votre frontend appelle simplement `backend-service`
- Peu importe l'IP réelle, le DNS fait toujours le lien
- Tout fonctionne automatiquement, même après des redémarrages

### 2. Découverte de services

Le DNS permet la **découverte automatique de services**. Dès qu'un nouveau service est créé dans Kubernetes, le DNS l'enregistre automatiquement. Les autres applications peuvent immédiatement le trouver.

### 3. Communication inter-namespace

Le DNS permet également aux applications dans différents namespaces de communiquer de manière structurée et sécurisée.

### 4. Standard de l'industrie

Utiliser des noms DNS est une pratique standard. Vos applications peuvent fonctionner de la même manière localement, en développement, et en production, simplement en changeant les noms DNS.

## CoreDNS : L'implémentation DNS de Kubernetes

Kubernetes utilise **CoreDNS** comme serveur DNS par défaut. CoreDNS est :

- **Léger** : Consomme peu de ressources
- **Rapide** : Répond très rapidement aux requêtes
- **Flexible** : Peut être configuré de nombreuses manières
- **Fiable** : Utilisé en production par des milliers d'entreprises
- **Cloud-native** : Conçu spécifiquement pour les environnements conteneurisés

CoreDNS s'exécute lui-même comme un pod dans votre cluster, généralement dans le namespace `kube-system`.

## Activation du DNS

L'activation du DNS avec MicroK8s est extrêmement simple.

### Commande d'activation

```bash
microk8s enable dns
```

C'est tout ! En une seule commande, vous activez un serveur DNS complet pour votre cluster.

### Ce qui se passe lors de l'activation

Quand vous activez le DNS, MicroK8s :

1. **Déploie CoreDNS** comme un Deployment dans le namespace `kube-system`
2. **Crée un Service** nommé `kube-dns` pour exposer CoreDNS
3. **Configure** tous les pods du cluster pour utiliser ce DNS
4. **Génère** une configuration par défaut adaptée à votre environnement

**Sortie attendue :**
```
Infer repository core for addon dns
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

### Temps d'activation

L'activation du DNS prend généralement **10 à 30 secondes**. C'est très rapide car CoreDNS est léger.

### Vérification de l'activation

Pour vérifier que le DNS est bien activé :

```bash
microk8s status
```

Vous devriez voir :
```
enabled:
  dns                  # (core) CoreDNS
```

### Vérifier que CoreDNS fonctionne

Pour s'assurer que les pods CoreDNS sont bien en cours d'exécution :

```bash
microk8s kubectl get pods -n kube-system -l k8s-app=kube-dns
```

Sortie attendue :
```
NAME                       READY   STATUS    RESTARTS   AGE
coredns-xxxxxxxxxx-xxxxx   1/1     Running   0          2m
```

Le pod doit être en état `Running` avec `1/1` dans la colonne `READY`.

### Vérifier le service DNS

```bash
microk8s kubectl get service -n kube-system kube-dns
```

Sortie attendue :
```
NAME       TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)         AGE
kube-dns   ClusterIP   10.152.183.10   <none>        53/UDP,53/TCP   2m
```

Le service `kube-dns` doit avoir une CLUSTER-IP (généralement dans la plage 10.152.x.x pour MicroK8s).

## Comment fonctionne le DNS dans Kubernetes

### Structure des noms DNS

Dans Kubernetes, les noms DNS suivent une structure hiérarchique :

```
<service-name>.<namespace>.svc.cluster.local
```

Décomposons :
- **service-name** : Le nom de votre service
- **namespace** : Le namespace où se trouve le service
- **svc** : Indique qu'il s'agit d'un service
- **cluster.local** : Le domaine par défaut du cluster

**Exemple :**
Si vous avez un service nommé `database` dans le namespace `production`, son nom DNS complet est :
```
database.production.svc.cluster.local
```

### Formes courtes

La bonne nouvelle : vous n'avez pas toujours besoin d'utiliser le nom complet !

**Dans le même namespace :**
Si votre application et le service sont dans le même namespace, vous pouvez simplement utiliser :
```
database
```

**Dans un namespace différent :**
Si le service est dans un autre namespace, vous devez au minimum spécifier :
```
database.production
```

**Forme complète :**
Vous pouvez toujours utiliser la forme complète pour être explicite :
```
database.production.svc.cluster.local
```

### Exemple concret

Imaginons cette architecture :
- Un service `frontend` dans le namespace `default`
- Un service `api` dans le namespace `default`
- Un service `database` dans le namespace `production`

**Le frontend peut appeler l'API avec :**
- `api` (même namespace)
- `api.default` (forme explicite)
- `api.default.svc.cluster.local` (forme complète)

**Le frontend peut appeler la database avec :**
- `database.production` (namespace différent, minimum requis)
- `database.production.svc.cluster.local` (forme complète)

## Résolution DNS pour les Pods

Chaque pod dans Kubernetes est automatiquement configuré pour utiliser CoreDNS. Kubernetes injecte automatiquement ces informations dans le pod :

### Fichier /etc/resolv.conf dans un pod

Si vous inspectez le fichier `/etc/resolv.conf` d'un pod, vous verrez quelque chose comme :

```
nameserver 10.152.183.10
search default.svc.cluster.local svc.cluster.local cluster.local
options ndots:5
```

**Explication :**
- **nameserver** : L'adresse IP du service CoreDNS (kube-dns)
- **search** : Les domaines de recherche (permet d'utiliser les noms courts)
- **options ndots:5** : Configuration de recherche DNS

Tout cela est configuré automatiquement, vous n'avez rien à faire !

## Tester le DNS

### Méthode 1 : Déployer un pod de test

Pour tester que le DNS fonctionne, déployons un pod simple avec des outils réseau :

```bash
microk8s kubectl run test-dns --image=busybox:1.28 --rm -it --restart=Never -- sh
```

Cette commande :
- Crée un pod temporaire nommé `test-dns`
- Utilise l'image `busybox` (qui contient des outils réseau)
- Ouvre un shell interactif
- Supprime le pod quand vous sortez

Une fois dans le shell du pod, vous pouvez tester le DNS :

```bash
# Tester la résolution du service kube-dns
nslookup kubernetes.default

# Tester un service externe
nslookup google.com
```

**Sortie attendue pour kubernetes.default :**
```
Server:    10.152.183.10
Address 1: 10.152.183.10 kube-dns.kube-system.svc.cluster.local

Name:      kubernetes.default
Address 1: 10.152.183.1 kubernetes.default.svc.cluster.local
```

Si vous voyez cette sortie, votre DNS fonctionne parfaitement !

Pour sortir du pod :
```bash
exit
```

### Méthode 2 : Utiliser un pod existant

Si vous avez déjà des pods qui tournent, vous pouvez y accéder et tester le DNS :

```bash
# Lister vos pods
microk8s kubectl get pods

# Accéder à un pod
microk8s kubectl exec -it <nom-du-pod> -- sh

# Puis tester le DNS (si les outils sont disponibles)
nslookup kubernetes.default
```

### Méthode 3 : Créer un service et le tester

Créons un service simple et testons sa résolution DNS :

**1. Créer un deployment et un service**

Créez un fichier `nginx-test.yaml` :
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-test
spec:
  replicas: 1
  selector:
    matchLabels:
      app: nginx-test
  template:
    metadata:
      labels:
        app: nginx-test
    spec:
      containers:
      - name: nginx
        image: nginx:alpine
        ports:
        - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: nginx-test
spec:
  selector:
    app: nginx-test
  ports:
  - port: 80
    targetPort: 80
```

Appliquez-le :
```bash
microk8s kubectl apply -f nginx-test.yaml
```

**2. Tester la résolution DNS**

```bash
microk8s kubectl run test --image=busybox:1.28 --rm -it --restart=Never -- sh
```

Dans le pod :
```bash
# Résoudre le nom du service
nslookup nginx-test

# Essayer de se connecter
wget -O- http://nginx-test
```

Vous devriez voir la page d'accueil de nginx, prouvant que :
1. Le DNS a résolu `nginx-test` en adresse IP
2. La connexion a fonctionné

## Comprendre les enregistrements DNS créés

Quand vous créez un Service dans Kubernetes, CoreDNS crée automatiquement plusieurs enregistrements :

### Enregistrement A (Address)

Un enregistrement A qui pointe le nom du service vers son ClusterIP :

```
my-service.default.svc.cluster.local  →  10.152.183.50
```

### Enregistrement SRV (Service)

Des enregistrements SRV pour chaque port nommé :

```
_http._tcp.my-service.default.svc.cluster.local
```

Ces enregistrements SRV sont utiles pour la découverte de services avancée, mais la plupart du temps, vous utiliserez simplement les enregistrements A.

### Résolution inverse

CoreDNS supporte aussi la résolution inverse (PTR records), permettant de trouver le nom à partir de l'IP.

## DNS externe et résolution Internet

Par défaut, CoreDNS dans MicroK8s est configuré pour :

1. **Résoudre les services internes** du cluster
2. **Transférer les requêtes externes** vers le DNS de votre système hôte

Cela signifie que vos pods peuvent :
- Communiquer entre eux via DNS interne (ex: `api-service`)
- Accéder à Internet via DNS externe (ex: `google.com`, `github.com`)

### Configuration du forwarding

CoreDNS utilise votre DNS système (généralement configuré dans `/etc/resolv.conf` de votre hôte) pour résoudre les noms externes.

Si vous utilisez Google DNS (8.8.8.8) ou Cloudflare (1.1.1.1) sur votre machine hôte, vos pods les utiliseront automatiquement pour les requêtes externes.

## Configuration avancée de CoreDNS

CoreDNS est hautement configurable via une ConfigMap. Voyons sa configuration par défaut :

```bash
microk8s kubectl get configmap coredns -n kube-system -o yaml
```

Vous verrez une configuration qui ressemble à :

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: coredns
  namespace: kube-system
data:
  Corefile: |
    .:53 {
        errors
        health {
          lameduck 5s
        }
        ready
        kubernetes cluster.local in-addr.arpa ip6.arpa {
          pods insecure
          fallthrough in-addr.arpa ip6.arpa
          ttl 30
        }
        prometheus :9153
        forward . /etc/resolv.conf
        cache 30
        loop
        reload
        loadbalance
    }
```

### Comprendre la configuration (pour curieux)

- **.:53** : Écoute sur le port 53 (port DNS standard)
- **errors** : Log les erreurs
- **health** : Endpoint de santé
- **kubernetes** : Plugin pour la résolution des noms Kubernetes
- **prometheus** : Expose des métriques
- **forward** : Transfère les requêtes externes au DNS du système
- **cache 30** : Cache les réponses pendant 30 secondes
- **loop** : Détecte les boucles DNS
- **reload** : Recharge automatiquement si la config change
- **loadbalance** : Répartit la charge entre les endpoints

**Note pour débutants :** Vous n'avez généralement pas besoin de modifier cette configuration. Elle fonctionne très bien par défaut. Ne touchez à cela que si vous avez des besoins spécifiques avancés.

## Cas d'usage pratiques

### 1. Application web avec base de données

**Architecture :**
- Frontend (Nginx) → appelle `api-service`
- API (Node.js) → appelle `database-service`
- Database (PostgreSQL)

**Configuration de l'API (variables d'environnement) :**
```yaml
env:
- name: DATABASE_HOST
  value: "database-service"  # Simple nom du service
- name: DATABASE_PORT
  value: "5432"
```

L'API peut se connecter à la base de données avec `database-service:5432`. Le DNS résout automatiquement vers la bonne IP.

### 2. Microservices

Dans une architecture microservices avec plusieurs services :
- `user-service`
- `order-service`
- `payment-service`
- `notification-service`

Chaque service peut appeler les autres par leur nom :
- `http://user-service/api/users`
- `http://order-service/api/orders`
- etc.

Le DNS gère toute la complexité de trouver les bonnes adresses IP.

### 3. Communication cross-namespace

**Production et Staging séparés :**
- Services de production dans namespace `prod`
- Services de staging dans namespace `staging`

Le service `monitoring` dans namespace `monitoring` peut surveiller les deux environnements :
- `http://api.prod:8080/health`
- `http://api.staging:8080/health`

## Dépannage DNS

### Le DNS ne se résout pas

**Symptôme :** Les pods ne peuvent pas se trouver par leur nom

**Diagnostics :**

1. **Vérifier que CoreDNS tourne :**
```bash
microk8s kubectl get pods -n kube-system -l k8s-app=kube-dns
```

2. **Vérifier les logs de CoreDNS :**
```bash
microk8s kubectl logs -n kube-system -l k8s-app=kube-dns
```

3. **Tester depuis un pod :**
```bash
microk8s kubectl run test-dns --image=busybox:1.28 --rm -it --restart=Never -- nslookup kubernetes.default
```

**Solutions courantes :**
- Redémarrer CoreDNS : `microk8s kubectl rollout restart -n kube-system deployment/coredns`
- Réactiver l'addon : `microk8s disable dns && microk8s enable dns`

### CoreDNS en CrashLoopBackOff

**Symptôme :** Le pod CoreDNS redémarre continuellement

**Causes possibles :**
1. Conflit de port 53 sur l'hôte
2. Problème de configuration
3. Ressources insuffisantes

**Solutions :**
1. Vérifier qu'aucun autre DNS ne tourne sur l'hôte :
```bash
sudo netstat -tulpn | grep :53
```

2. Consulter les logs :
```bash
microk8s kubectl logs -n kube-system -l k8s-app=kube-dns --previous
```

3. Réinitialiser :
```bash
microk8s disable dns
microk8s enable dns
```

### Résolution lente

**Symptôme :** Les requêtes DNS prennent plusieurs secondes

**Causes possibles :**
1. Cache DNS insuffisant
2. Problème de forwarding vers DNS externe
3. Trop de requêtes

**Solutions :**
1. Augmenter le cache dans la ConfigMap CoreDNS (passer de 30 à 300 secondes)
2. Utiliser un DNS externe plus rapide (Google 8.8.8.8, Cloudflare 1.1.1.1)
3. Scaler CoreDNS :
```bash
microk8s kubectl scale deployment coredns -n kube-system --replicas=2
```

### Services non trouvés

**Symptôme :** Un service spécifique n'est pas résolu

**Vérifications :**
1. Le service existe-t-il vraiment ?
```bash
microk8s kubectl get service <nom-service>
```

2. Est-il dans le bon namespace ?
```bash
microk8s kubectl get service <nom-service> -n <namespace>
```

3. A-t-il une ClusterIP ?
```bash
microk8s kubectl describe service <nom-service>
```

Si le service est de type `ExternalName` ou n'a pas de ClusterIP, sa résolution DNS fonctionne différemment.

## Bonnes pratiques

### 1. Toujours utiliser des noms de services

**Faites :**
```yaml
env:
- name: API_URL
  value: "http://api-service:8080"
```

**Ne faites pas :**
```yaml
env:
- name: API_URL
  value: "http://10.152.183.45:8080"  # IP hardcodée
```

Les IPs peuvent changer, les noms de services sont stables.

### 2. Utiliser des noms courts dans le même namespace

Si votre frontend et votre API sont dans le même namespace, utilisez simplement :
```
api-service
```

Au lieu de :
```
api-service.default.svc.cluster.local
```

C'est plus lisible et tout aussi fonctionnel.

### 3. Spécifier le namespace pour les accès cross-namespace

Pour la clarté et éviter les erreurs, spécifiez toujours le namespace :
```
database.production
```

Au lieu de juste :
```
database  # Pourrait être ambigu
```

### 4. Nommer les services de manière descriptive

Utilisez des noms de services clairs et cohérents :
- ✅ `user-api`, `user-db`, `user-cache`
- ❌ `service1`, `service2`, `svc-abc`

### 5. Documenter les dépendances DNS

Dans vos README ou documentation, listez les services dont votre application dépend :

```
Dépendances DNS :
- api-service.default:8080
- cache.default:6379
- database.production:5432
```

Cela aide à comprendre l'architecture et à diagnostiquer les problèmes.

## DNS et sécurité

### Considérations de sécurité

1. **DNS poisoning** : Dans un cluster non sécurisé, un attaquant pourrait potentiellement manipuler les réponses DNS. Utilisez Network Policies pour isoler les namespaces sensibles.

2. **Information disclosure** : Les noms DNS peuvent révéler votre architecture. En production, limitez qui peut interroger le DNS.

3. **DNS tunneling** : Technique d'exfiltration de données via DNS. Surveillez les requêtes DNS anormales.

Pour un lab personnel ou environnement de développement, ces risques sont minimes. En production, implémentez RBAC et Network Policies appropriés.

## Récapitulatif

Le DNS (CoreDNS) est essentiel pour :

✅ **Permettre** aux services de se trouver par nom, pas par IP
✅ **Simplifier** la configuration des applications
✅ **Gérer automatiquement** les changements d'IP
✅ **Découvrir** les services dynamiquement
✅ **Résoudre** les noms Internet externes

**Commandes clés à retenir :**
```bash
# Activer DNS
microk8s enable dns

# Vérifier l'état
microk8s kubectl get pods -n kube-system -l k8s-app=kube-dns

# Tester le DNS
microk8s kubectl run test-dns --image=busybox:1.28 --rm -it --restart=Never -- nslookup kubernetes.default

# Voir les logs CoreDNS
microk8s kubectl logs -n kube-system -l k8s-app=kube-dns
```

**Structure des noms DNS :**
```
<service>.<namespace>.svc.cluster.local

Exemples :
- api (même namespace)
- api.default (avec namespace)
- api.default.svc.cluster.local (forme complète)
```

## Ce que vous avez appris

Dans cette section, vous avez découvert :
- Ce qu'est le DNS et pourquoi il est crucial dans Kubernetes
- Comment CoreDNS fonctionne dans MicroK8s
- Comment activer et vérifier le DNS
- La structure des noms DNS dans Kubernetes
- Comment tester la résolution DNS
- Les cas d'usage pratiques du DNS
- Comment diagnostiquer et résoudre les problèmes DNS
- Les bonnes pratiques d'utilisation du DNS

## Prochaine étape

Avec le Dashboard pour visualiser et le DNS pour communiquer, votre cluster prend forme ! La prochaine étape est d'activer le **Registry** (section 5.5), qui vous permettra d'héberger vos propres images Docker localement.

Le Registry est particulièrement utile pour :
- Ne pas dépendre de Docker Hub pour vos images personnelles
- Accélérer le déploiement (pas de téléchargement depuis Internet)
- Tester vos images avant de les pousser vers un registre public

Continuons !

---

**Astuce pratique :** Le DNS est tellement fondamental que vous oublierez rapidement qu'il est là. C'est le signe d'un bon outil : il fonctionne silencieusement en arrière-plan et vous n'y pensez que quand il y a un problème. Activez-le et appréciez la magie de pouvoir appeler vos services par leur nom !

⏭️ [Registry (registre d'images privé - microk8s enable registry)](/05-addons-microk8s/05-registry-registre-dimages-prive.md)
