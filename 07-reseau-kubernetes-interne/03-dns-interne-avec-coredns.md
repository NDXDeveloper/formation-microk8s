🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 7.3 DNS interne avec CoreDNS

## Introduction

Imaginez devoir mémoriser tous les numéros de téléphone de vos contacts. Épuisant, non ? C'est exactement pourquoi nous utilisons des noms : "Papa", "Marie", "Bureau". Le DNS (Domain Name System) fait la même chose pour les réseaux : il traduit des noms faciles à retenir en adresses IP.

Dans Kubernetes, le DNS est encore plus crucial car les adresses IP des Pods changent constamment. Sans DNS, vos applications devraient constamment mettre à jour leurs configurations. Avec DNS, elles utilisent simplement des noms stables, et le système s'occupe du reste.

**CoreDNS** est le serveur DNS qui fournit ce service dans Kubernetes. Dans ce chapitre, nous allons explorer en profondeur comment il fonctionne et pourquoi il est indispensable.

## Rappel : Qu'est-ce que le DNS ?

### Le DNS sur Internet

Quand vous tapez `www.google.com` dans votre navigateur :

```
1. Votre ordinateur demande : "Quelle est l'IP de www.google.com ?"
2. Le serveur DNS répond : "C'est 142.250.185.78"
3. Votre navigateur se connecte à cette IP
```

Sans DNS, vous devriez taper `http://142.250.185.78` - peu pratique et difficile à mémoriser !

### Le DNS dans Kubernetes

Le principe est identique, mais adapté au monde des conteneurs :

```
Application demande : "Quelle est l'IP du service 'database' ?"
CoreDNS répond : "C'est 10.152.183.25"
Application se connecte à cette IP
```

### Pourquoi le DNS est crucial dans Kubernetes ?

**Raison 1 : Les IPs des Pods sont éphémères**

```
Lundi 9h : Pod "api" redémarre → nouvelle IP 10.1.1.5
Lundi 10h : Pod "api" redémarre → nouvelle IP 10.1.1.18
Lundi 11h : Pod "api" redémarre → nouvelle IP 10.1.1.23
```

Si votre application frontend utilisait l'IP directement, elle devrait se reconfigurer à chaque redémarrage. Avec le nom DNS `api`, elle fonctionne toujours, peu importe l'IP.

**Raison 2 : Abstraction des Services**

Un Service peut avoir plusieurs Pods backend :

```
Service "api" (nom DNS stable)
    ↓
IP du Service : 10.152.183.25
    ↓
Redirige vers :
    - Pod 1 : 10.1.1.5
    - Pod 2 : 10.1.1.8
    - Pod 3 : 10.1.1.12
```

Votre application appelle simplement `api`, sans savoir combien de Pods il y a derrière.

**Raison 3 : Portabilité du code**

Le même code fonctionne dans différents environnements :

```python
# Votre code (identique partout)
db_host = "database"

# En développement : database → 10.152.183.10
# En staging : database → 10.152.183.20
# En production : database → 10.152.183.30
```

Pas besoin de changer le code, juste la configuration Kubernetes.

## CoreDNS : le serveur DNS de Kubernetes

### Qu'est-ce que CoreDNS ?

**CoreDNS** est un serveur DNS moderne, flexible et extensible, écrit en Go. Il est devenu le serveur DNS par défaut de Kubernetes depuis la version 1.13.

**Caractéristiques principales :**

- **Léger et performant** : Consomme peu de ressources
- **Modulaire** : Fonctionnalités ajoutées via des plugins
- **Natif Kubernetes** : Comprend les ressources Kubernetes (Services, Pods, etc.)
- **Cloud-native** : Conçu pour les environnements conteneurisés

### CoreDNS dans MicroK8s

Dans MicroK8s, CoreDNS est installé via l'addon `dns` :

```bash
# Vérifier si DNS est activé
microk8s status

# Activer DNS si nécessaire
microk8s enable dns
```

Une fois activé, CoreDNS tourne comme un Deployment dans le namespace `kube-system`.

### Architecture de CoreDNS dans le cluster

```
┌─────────────────────────────────────────────────────────┐
│                    Cluster Kubernetes                   │
│                                                         │
│  ┌───────────────────────────────────────────────────┐  │
│  │         Namespace: kube-system                    │  │
│  │                                                   │  │
│  │  ┌──────────────────────────────────────────────┐ │  │
│  │  │   Deployment: coredns                        │ │  │
│  │  │                                              │ │  │
│  │  │   ┌──────────────┐     ┌──────────────┐      │ │  │
│  │  │   │ CoreDNS Pod 1│     │ CoreDNS Pod 2│      │ │  │
│  │  │   │ 10.1.1.50    │     │ 10.1.1.51    │      │ │  │
│  │  │   └──────────────┘     └──────────────┘      │ │  │
│  │  │                                              │ │  │
│  │  └──────────────────────────────────────────────┘ │  │
│  │                        │                          │  │
│  │                        ▼                          │  │
│  │  ┌──────────────────────────────────────────────┐ │  │
│  │  │   Service: kube-dns                          │ │  │
│  │  │   ClusterIP: 10.152.183.10                   │ │  │
│  │  │   Port: 53 (DNS standard)                    │ │  │
│  │  └──────────────────────────────────────────────┘ │  │
│  └───────────────────────────────────────────────────┘  │
│                        ▲                                │
│                        │ Requêtes DNS                   │
│                        │                                │
│  ┌─────────────────────┴───────────────────────────┐    │
│  │              Tous les Pods du cluster           │    │
│  │                                                 │    │
│  │  /etc/resolv.conf configuré pour utiliser       │    │
│  │  10.152.183.10 comme serveur DNS                │    │
│  └─────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────┘
```

**Points importants :**

1. **CoreDNS tourne dans des Pods** (généralement 2 pour la redondance)
2. **Exposé via un Service** nommé `kube-dns` avec une IP ClusterIP fixe
3. **Tous les Pods** sont automatiquement configurés pour utiliser ce Service DNS

## Format des noms DNS dans Kubernetes

Kubernetes utilise une convention de nommage stricte pour le DNS. Comprendre ce format est essentiel.

### Anatomie d'un nom DNS complet (FQDN)

Un nom DNS complet dans Kubernetes ressemble à ceci :

```
<nom-service>.<namespace>.svc.cluster.local
```

**Décomposition :**

1. **nom-service** : Le nom de votre Service Kubernetes
2. **namespace** : Le namespace où se trouve le Service
3. **svc** : Indique que c'est un Service (mot-clé fixe)
4. **cluster.local** : Le domaine du cluster (configurable mais rarement modifié)

### Exemples concrets

**Exemple 1 : Service "database" dans le namespace "production"**

```
Nom court (dans le même namespace) : database
Nom avec namespace : database.production
Nom complet (FQDN) : database.production.svc.cluster.local
```

**Exemple 2 : Service "api" dans le namespace "default"**

```
Nom court : api
Nom avec namespace : api.default
Nom complet : api.default.svc.cluster.local
```

### Noms courts vs noms complets

Kubernetes supporte plusieurs formes de noms DNS grâce au **search path** (chemin de recherche).

#### Configuration DNS dans un Pod

Chaque Pod a un fichier `/etc/resolv.conf` qui configure son comportement DNS :

```bash
# Contenu typique de /etc/resolv.conf dans un Pod
nameserver 10.152.183.10
search default.svc.cluster.local svc.cluster.local cluster.local
options ndots:5
```

**Explications :**

- **nameserver 10.152.183.10** : Adresse IP du Service CoreDNS
- **search** : Liste des suffixes à ajouter automatiquement
- **ndots:5** : Si le nom a moins de 5 points, essayer les suffixes search

#### Comment fonctionne la résolution avec search path ?

Quand votre application demande "database", voici ce qui se passe :

```
1. Essai : database.default.svc.cluster.local
   ↓
   Trouvé ? → OUI → Renvoyer l'IP

2. Si non trouvé, essai : database.svc.cluster.local
   ↓
   Trouvé ? → Vérifier

3. Si non trouvé, essai : database.cluster.local
   ↓
   Trouvé ? → Vérifier

4. Si non trouvé, essai : database
   ↓
   Trouvé ? → Vérifier

5. Si rien trouvé → Erreur "name not resolved"
```

**Pratique : Noms selon le contexte**

```
┌───────────────────────────────────────────────────────────┐
│ Depuis un Pod dans le namespace "default"                 │
├───────────────────────────────────────────────────────────┤
│                                                           │
│ Pour contacter un Service dans le même namespace :        │
│   → database                                              │
│                                                           │
│ Pour contacter un Service dans un autre namespace :       │
│   → database.production                                   │
│   → database.production.svc.cluster.local                 │
│                                                           │
│ Pour contacter un Service quelconque (forme la plus sûre):│
│   → database.production.svc.cluster.local                 │
│                                                           │
└───────────────────────────────────────────────────────────┘
```

**Bonnes pratiques :**

- **Même namespace** : Utilisez le nom court (`database`)
- **Namespace différent** : Utilisez au moins `service.namespace`
- **Code partagé/bibliothèque** : Utilisez le FQDN complet pour éviter toute ambiguïté

## Noms DNS pour les Pods

Les Services ont des noms DNS, mais qu'en est-il des Pods individuels ?

### Format du nom DNS d'un Pod

Par défaut, les Pods **ne reçoivent PAS** de nom DNS automatique. Mais si vous en avez besoin, le format serait :

```
<ip-du-pod-avec-tirets>.<namespace>.pod.cluster.local
```

**Exemple :**

```
Pod IP : 10.1.1.5
Namespace : production
Nom DNS : 10-1-1-5.production.pod.cluster.local
```

**Notez** que l'IP utilise des tirets `-` au lieu de points `.`

### Pourquoi les Pods n'ont généralement pas de DNS ?

**Raison :** Les Pods sont éphémères et leurs IPs changent constamment. Donner un nom DNS à chaque Pod n'a pas de sens car ce nom changerait tout le temps.

**Solution :** Utilisez les **Services** qui ont des noms DNS stables et redirigent vers les Pods.

### Exception : StatefulSets

Les **StatefulSets** sont une exception. Chaque Pod d'un StatefulSet reçoit un nom DNS stable :

```
<nom-pod>.<nom-service-headless>.<namespace>.svc.cluster.local
```

**Exemple avec StatefulSet "mongodb" :**

```
Pod 0 : mongodb-0.mongodb.production.svc.cluster.local
Pod 1 : mongodb-1.mongodb.production.svc.cluster.local
Pod 2 : mongodb-2.mongodb.production.svc.cluster.local
```

Ces noms restent stables même si les Pods redémarrent (tant qu'ils conservent le même nom).

## Résolution DNS : le parcours d'une requête

Suivons une requête DNS depuis une application jusqu'à la réponse.

### Scénario : Frontend appelle "api"

```
┌────────────────────────────────────────────────────────┐
│                                                        │
│  1. Application dans Pod Frontend                      │
│     ┌─────────────────────┐                            │
│     │  Code Python :      │                            │
│     │  api_url = "api"    │                            │
│     │  requests.get(...)  │                            │
│     └──────────┬──────────┘                            │
│                │                                       │
│                │ Requête DNS : "Quelle IP pour 'api'?" │
│                ▼                                       │
│     ┌─────────────────────┐                            │
│     │ /etc/resolv.conf    │                            │
│     │ nameserver:         │                            │
│     │   10.152.183.10     │                            │
│     └──────────┬──────────┘                            │
└────────────────┼───────────────────────────────────────┘
                 │
                 │ 2. Envoi de la requête DNS
                 ▼
┌────────────────────────────────────────────────────────┐
│         Service CoreDNS (10.152.183.10)                │
│                                                        │
│  3. CoreDNS reçoit : "api"                             │
│     ┌──────────────────────────┐                       │
│     │ Search path analysis :   │                       │
│     │ - Essai: api.default...  │                       │
│     │ - Trouvé !               │                       │
│     └──────────┬───────────────┘                       │
│                │                                       │
│                │ 4. Interroge l'API Kubernetes         │
│                ▼                                       │
│     ┌──────────────────────────┐                       │
│     │ Kubernetes API Server    │                       │
│     │ "Service 'api' existe ?" │                       │
│     │ "IP: 10.152.183.15"      │                       │
│     └──────────┬───────────────┘                       │
│                │                                       │
│                │ 5. CoreDNS prépare la réponse         │
│                ▼                                       │
│     ┌──────────────────────────┐                       │
│     │ Cache la réponse (TTL)   │                       │
│     │ Réponse: 10.152.183.15   │                       │
│     └──────────┬───────────────┘                       │
└────────────────┼───────────────────────────────────────┘
                 │
                 │ 6. Réponse DNS
                 ▼
┌────────────────────────────────────────────────────────┐
│  7. Application reçoit l'IP                            │
│     ┌─────────────────────────┐                        │
│     │ "api" = 10.152.183.15   │                        │
│     └──────────┬──────────────┘                      │
│                │                                       │
│                │ 8. Connexion HTTP vers 10.152.183.15  │
│                ▼                                       │
│     ┌─────────────────────────┐                        │
│     │   Service "api"         │                        │
│     │   10.152.183.15         │                        │
│     └─────────────────────────┘                        │
└────────────────────────────────────────────────────────┘
```

### Étapes détaillées

**Étape 1-2 : L'application fait une requête DNS**

L'application veut contacter "api". Le système d'exploitation du Pod consulte `/etc/resolv.conf` et envoie une requête DNS à CoreDNS (10.152.183.10).

**Étape 3 : CoreDNS analyse la requête**

CoreDNS reçoit "api" et applique le search path :
- Essaie d'abord `api.default.svc.cluster.local` (si Pod dans namespace default)
- Si trouvé, continue

**Étape 4 : Consultation de l'API Kubernetes**

CoreDNS ne stocke pas les IPs en dur. Il interroge l'API Kubernetes en temps réel :
```
"Y a-t-il un Service nommé 'api' dans le namespace 'default' ?"
API répond : "Oui, son ClusterIP est 10.152.183.15"
```

**Étape 5-6 : Mise en cache et réponse**

CoreDNS met la réponse en cache (pour éviter de surcharger l'API) et renvoie l'IP au Pod demandeur.

**Étape 7-8 : L'application se connecte**

Maintenant que l'application connaît l'IP (10.152.183.15), elle peut établir une connexion TCP/HTTP vers le Service.

## Le Corefile : configuration de CoreDNS

CoreDNS utilise un fichier de configuration appelé **Corefile**. Ce fichier définit quels plugins utiliser et comment ils doivent fonctionner.

### Où se trouve le Corefile ?

Dans MicroK8s, la configuration CoreDNS est stockée dans une **ConfigMap** :

```bash
# Voir la ConfigMap CoreDNS
microk8s kubectl get configmap coredns -n kube-system -o yaml
```

### Anatomie du Corefile

Voici un Corefile typique de MicroK8s (simplifié) :

```
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

Décomposons cette configuration :

### 1. Bloc de serveur `.:53`

```
.:53 {
    ...
}
```

**Signification :**
- **`.`** : Domaine racine (toutes les requêtes DNS)
- **53** : Port DNS standard

Ce bloc capture toutes les requêtes DNS arrivant sur le port 53.

### 2. Plugin `errors`

```
errors
```

**Rôle :** Journalise les erreurs dans les logs CoreDNS.

**Utilité :** Débogage et monitoring.

### 3. Plugin `health`

```
health {
    lameduck 5s
}
```

**Rôle :** Expose un endpoint de santé HTTP pour Kubernetes.

**Utilité :**
- Kubernetes vérifie que CoreDNS est vivant
- `lameduck 5s` : Période de grâce avant de marquer le Pod comme non sain

**Endpoint :** `http://<pod-ip>:8080/health`

### 4. Plugin `ready`

```
ready
```

**Rôle :** Expose un endpoint "ready" pour indiquer que CoreDNS est prêt à servir des requêtes.

**Endpoint :** `http://<pod-ip>:8181/ready`

### 5. Plugin `kubernetes` (le cœur !)

```
kubernetes cluster.local in-addr.arpa ip6.arpa {
    pods insecure
    fallthrough in-addr.arpa ip6.arpa
    ttl 30
}
```

**Rôle :** Plugin principal qui permet à CoreDNS de résoudre les Services et Pods Kubernetes.

**Paramètres détaillés :**

- **`cluster.local`** : Domaine du cluster
- **`in-addr.arpa ip6.arpa`** : Zones pour la résolution inverse (IP → nom)
- **`pods insecure`** : Active la résolution DNS des Pods (mode non sécurisé, sans vérification)
- **`fallthrough`** : Si pas trouvé, passe au plugin suivant
- **`ttl 30`** : Durée de vie du cache DNS (30 secondes)

**Ce plugin est la magie qui permet de résoudre `database` → `10.152.183.25` !**

### 6. Plugin `prometheus`

```
prometheus :9153
```

**Rôle :** Expose des métriques Prometheus pour le monitoring.

**Endpoint :** `http://<pod-ip>:9153/metrics`

**Métriques collectées :**
- Nombre de requêtes DNS
- Temps de réponse
- Erreurs
- Cache hits/misses

### 7. Plugin `forward`

```
forward . /etc/resolv.conf
```

**Rôle :** Transfère les requêtes non résolues vers un serveur DNS externe.

**Fonctionnement :**

Si CoreDNS ne peut pas résoudre un nom (exemple : `www.google.com`), il transfère la requête aux serveurs DNS configurés dans `/etc/resolv.conf` du Pod CoreDNS (généralement les DNS de votre réseau ou FAI).

**Schéma :**

```
Pod demande : www.google.com
    ↓
CoreDNS : "Je ne connais pas ce nom (pas dans Kubernetes)"
    ↓
Forward vers DNS externe (8.8.8.8, 1.1.1.1, etc.)
    ↓
DNS externe répond : 142.250.185.78
    ↓
CoreDNS transmet la réponse au Pod
```

### 8. Plugin `cache`

```
cache 30
```

**Rôle :** Met en cache les réponses DNS pendant 30 secondes.

**Avantages :**
- **Performance** : Évite de requêter l'API Kubernetes à chaque fois
- **Réduction de charge** : Moins de requêtes vers l'API Server
- **Latence réduite** : Réponses instantanées depuis le cache

**Comportement :**

```
Requête 1 (t=0s) : "api" → Interroge API → Cache la réponse
Requête 2 (t=5s) : "api" → Répond depuis le cache (rapide)
Requête 3 (t=35s) : "api" → Cache expiré → Interroge API → Re-cache
```

### 9. Plugin `loop`

```
loop
```

**Rôle :** Détecte les boucles de résolution DNS.

**Problème évité :**

```
CoreDNS A forward vers → CoreDNS B forward vers → CoreDNS A (boucle !)
```

Si une boucle est détectée, CoreDNS s'arrête pour éviter une charge infinie.

### 10. Plugin `reload`

```
reload
```

**Rôle :** Recharge automatiquement le Corefile quand la ConfigMap change.

**Utilité :** Pas besoin de redémarrer les Pods CoreDNS après une modification de configuration.

### 11. Plugin `loadbalance`

```
loadbalance
```

**Rôle :** Si un nom DNS a plusieurs IPs (plusieurs Pods backend), randomise l'ordre des réponses.

**Effet :** Distribution de charge basique au niveau DNS.

## Types de requêtes DNS supportées

CoreDNS dans Kubernetes supporte plusieurs types de requêtes DNS.

### 1. Requêtes A (IPv4)

```
Query : api.default.svc.cluster.local
Type : A
Réponse : 10.152.183.15 (adresse IPv4)
```

**Usage :** C'est le type le plus courant, utilisé pour obtenir l'IPv4 d'un Service.

### 2. Requêtes AAAA (IPv6)

```
Query : api.default.svc.cluster.local
Type : AAAA
Réponse : fd00::1234 (adresse IPv6)
```

**Usage :** Si votre cluster utilise IPv6.

### 3. Requêtes SRV (Service records)

```
Query : _http._tcp.api.default.svc.cluster.local
Type : SRV
Réponse : Priority, Weight, Port, Target
```

**Usage :** Découvrir le port d'un service dynamiquement.

**Exemple de réponse :**

```
Priority: 0
Weight: 100
Port: 8080
Target: api.default.svc.cluster.local
```

### 4. Requêtes PTR (Reverse DNS)

```
Query : 15.183.152.10.in-addr.arpa
Type : PTR
Réponse : api.default.svc.cluster.local
```

**Usage :** Résolution inverse (IP → nom). Peu utilisé dans Kubernetes.

## Services sans IP (Headless Services)

Les Services Kubernetes peuvent être **Headless**, c'est-à-dire sans ClusterIP.

### Qu'est-ce qu'un Headless Service ?

Un Service avec `clusterIP: None` dans sa définition :

```yaml
apiVersion: v1
kind: Service
metadata:
  name: database
spec:
  clusterIP: None  # ← Headless !
  selector:
    app: postgres
  ports:
  - port: 5432
```

### Comportement DNS d'un Headless Service

Au lieu de renvoyer une seule IP (celle du Service), CoreDNS renvoie **toutes les IPs des Pods backend**.

**Exemple :**

```
Service Headless "database" avec 3 Pods :
- Pod 1 : 10.1.1.5
- Pod 2 : 10.1.1.8
- Pod 3 : 10.1.1.12

Requête DNS : database.default.svc.cluster.local
Réponse : 10.1.1.5, 10.1.1.8, 10.1.1.12 (les 3 IPs)
```

### Pourquoi utiliser un Headless Service ?

**Cas d'usage :**

1. **Connexion directe aux Pods** : L'application veut choisir elle-même quel Pod contacter
2. **StatefulSets** : Chaque Pod d'un StatefulSet a besoin d'être adressable individuellement
3. **Protocoles spéciaux** : Certains protocoles (ex: databases clustering) nécessitent de connaître tous les membres

**Exemple : Cluster Cassandra**

```
Headless Service : cassandra
DNS query : cassandra.default.svc.cluster.local
Réponse : [10.1.1.5, 10.1.1.8, 10.1.1.12]

L'application Cassandra peut :
- Se connecter à tous les nœuds
- Former un cluster distribué
- Communiquer directement entre pairs
```

## Résolution DNS externe

Que se passe-t-il quand un Pod veut accéder à un nom externe (ex: `www.google.com`) ?

### Séquence de résolution

```
1. Pod demande : www.google.com
   ↓
2. Envoyé à CoreDNS (10.152.183.10)
   ↓
3. CoreDNS vérifie : est-ce un nom Kubernetes ?
   → NON (pas de format .svc.cluster.local)
   ↓
4. Plugin "forward" : transfère vers DNS externe
   → Lit /etc/resolv.conf du Pod CoreDNS
   → Envoie vers 8.8.8.8 (Google DNS) ou DNS de votre réseau
   ↓
5. DNS externe répond : 142.250.185.78
   ↓
6. CoreDNS transmet au Pod
   ↓
7. Pod peut maintenant se connecter à 142.250.185.78
```

### Configuration des serveurs DNS externes

Les serveurs DNS externes utilisés par CoreDNS proviennent de `/etc/resolv.conf` du Pod CoreDNS, qui lui-même hérite de la configuration du Node.

**Pour MicroK8s sur Ubuntu :**

Le Node utilise généralement :
- DNS du réseau local (votre box/routeur)
- Ou des DNS publics (8.8.8.8, 1.1.1.1, etc.)

## Cache DNS : performance et freshness

Le cache DNS de CoreDNS est un équilibre entre performance et fraîcheur des données.

### Comment fonctionne le cache ?

**TTL (Time To Live) : 30 secondes par défaut**

```
t=0s : Requête "api" → API Kubernetes → Cache : api=10.152.183.15 (expire à t=30s)
t=5s : Requête "api" → Cache HIT → Répond depuis cache (rapide)
t=15s : Requête "api" → Cache HIT → Répond depuis cache
t=30s : Cache expire
t=31s : Requête "api" → Cache MISS → API Kubernetes → Re-cache
```

### Avantages du cache

**Performance :**
- Réponse en quelques microsecondes au lieu de millisecondes
- Réduit la latence des applications

**Réduction de charge :**
- Moins de requêtes vers l'API Kubernetes
- Scalabilité accrue

### Inconvénients du cache

**Staleness (données périmées) :**

Si un Service change d'IP, le cache peut servir l'ancienne IP pendant 30 secondes maximum.

**Scénario :**

```
t=0s : Service "api" a l'IP 10.152.183.15
      Pod demande "api" → Réponse 10.152.183.15 → Mise en cache

t=10s : Vous supprimez et recréez le Service "api" → Nouvelle IP : 10.152.183.20

t=15s : Pod demande "api" → Cache retourne encore 10.152.183.15 (périmé)
        Pod ne peut pas se connecter (connexion échoue)

t=30s : Cache expire

t=31s : Pod demande "api" → Cache MISS → Nouvelle requête → 10.152.183.20 (correct)
```

**En pratique :** 30 secondes est un bon compromis. Les Services changent rarement d'IP, donc l'impact est minime.

### Purger le cache

Il n'existe pas de commande pour purger manuellement le cache CoreDNS. Solutions :

1. **Attendre** : Le cache expire naturellement (30s)
2. **Redémarrer CoreDNS** :
   ```bash
   microk8s kubectl rollout restart deployment coredns -n kube-system
   ```

## Debugging DNS : résoudre les problèmes

Les problèmes DNS sont fréquents. Voici comment les diagnostiquer.

### Symptôme 1 : "Name not resolved" / "Could not resolve host"

**Message d'erreur :**
```
curl: (6) Could not resolve host: api
```

**Diagnostic :**

**Étape 1 : Vérifier que CoreDNS tourne**

```bash
# Vérifier les Pods CoreDNS
microk8s kubectl get pods -n kube-system -l k8s-app=kube-dns

# Attendu : 2 Pods en "Running"
```

**Étape 2 : Vérifier que le Service kube-dns existe**

```bash
microk8s kubectl get svc -n kube-system kube-dns

# Attendu : Un service avec une ClusterIP
```

**Étape 3 : Vérifier le /etc/resolv.conf du Pod**

```bash
# Créer un Pod de test
microk8s kubectl run test-dns --image=busybox --restart=Never -- sleep 3600

# Vérifier resolv.conf
microk8s kubectl exec test-dns -- cat /etc/resolv.conf

# Attendu :
# nameserver 10.152.183.10
# search default.svc.cluster.local svc.cluster.local cluster.local
```

Si `nameserver` ne pointe pas vers l'IP du service kube-dns, il y a un problème de configuration.

**Étape 4 : Tester la résolution manuellement**

```bash
# Depuis un Pod, tester la résolution DNS
microk8s kubectl exec test-dns -- nslookup kubernetes

# Devrait résoudre vers l'IP du service kubernetes (généralement 10.152.183.1)
```

### Symptôme 2 : Résolution lente

**Diagnostic :**

**Cause possible 1 : ndots trop élevé**

Le paramètre `ndots:5` dans `/etc/resolv.conf` force Kubernetes à essayer tous les suffixes search avant le nom exact.

**Exemple :**
```
Query : www.google.com (2 dots, < 5)

Essais :
1. www.google.com.default.svc.cluster.local (fail)
2. www.google.com.svc.cluster.local (fail)
3. www.google.com.cluster.local (fail)
4. www.google.com (success)

Total : 4 requêtes au lieu de 1 !
```

**Solution :** Utilisez le point final pour court-circuiter le search path :

```python
# Au lieu de
host = "www.google.com"

# Utilisez
host = "www.google.com."  # ← Point final
```

Le point final indique "c'est un FQDN complet, n'ajoute pas de suffixes".

**Cause possible 2 : CoreDNS surchargé**

```bash
# Vérifier les métriques CoreDNS
microk8s kubectl port-forward -n kube-system svc/kube-dns 9153:9153

# Visiter http://localhost:9153/metrics
# Chercher : coredns_dns_request_duration_seconds
```

Si les requêtes prennent >100ms en moyenne, CoreDNS est peut-être surchargé.

**Solution :** Augmenter le nombre de replicas :

```bash
microk8s kubectl scale deployment coredns -n kube-system --replicas=3
```

### Symptôme 3 : Résolution fonctionne parfois, échoue parfois

**Diagnostic :**

**Cause probable : Race condition lors de la création de Services**

Quand vous créez un Service, il peut y avoir un délai (quelques secondes) avant que CoreDNS ne le connaisse.

**Solution :** Ajouter des retry dans votre code :

```python
import time
import requests

def call_api_with_retry(max_retries=5):
    for i in range(max_retries):
        try:
            response = requests.get("http://api:8080")
            return response
        except requests.exceptions.ConnectionError:
            time.sleep(2)
            continue
    raise Exception("Could not connect to API")
```

### Symptôme 4 : Résolution externe ne fonctionne pas

**Exemple :** `curl www.google.com` échoue depuis les Pods.

**Diagnostic :**

**Étape 1 : Vérifier le plugin forward**

```bash
microk8s kubectl get cm coredns -n kube-system -o yaml | grep forward

# Devrait afficher : forward . /etc/resolv.conf
```

**Étape 2 : Vérifier le /etc/resolv.conf de CoreDNS**

```bash
# Trouver un Pod CoreDNS
COREDNS_POD=$(microk8s kubectl get pod -n kube-system -l k8s-app=kube-dns -o name | head -1)

# Vérifier son resolv.conf
microk8s kubectl exec -n kube-system $COREDNS_POD -- cat /etc/resolv.conf

# Devrait contenir des nameservers valides (ex: 8.8.8.8, 1.1.1.1, ou DNS de votre réseau)
```

**Étape 3 : Tester depuis CoreDNS**

```bash
microk8s kubectl exec -n kube-system $COREDNS_POD -- nslookup www.google.com

# Devrait résoudre
```

Si ça fonctionne depuis CoreDNS mais pas depuis les Pods, le problème est ailleurs (réseau, Network Policies).

## Personnalisation de CoreDNS

Vous pouvez personnaliser CoreDNS pour des besoins spécifiques.

### Exemple 1 : Ajouter des entrées DNS personnalisées

Vous voulez que `myapp.local` résolve vers `192.168.1.100` dans tous les Pods.

**Solution : Plugin hosts**

Modifiez la ConfigMap CoreDNS :

```bash
microk8s kubectl edit cm coredns -n kube-system
```

Ajoutez le plugin `hosts` :

```
.:53 {
    # ... plugins existants ...

    hosts {
        192.168.1.100 myapp.local
        fallthrough
    }

    # ... reste de la config ...
}
```

Maintenant, `myapp.local` résoudra vers `192.168.1.100` depuis n'importe quel Pod.

### Exemple 2 : Changer le TTL du cache

Par défaut 30s, vous pouvez augmenter pour plus de performance ou réduire pour plus de fraîcheur.

```
cache 60  # Cache pendant 60 secondes au lieu de 30
```

### Exemple 3 : Logging détaillé pour debug

Ajoutez le plugin `log` :

```
.:53 {
    log  # ← Active le logging de toutes les requêtes
    errors
    # ... reste ...
}
```

**Attention :** Cela génère beaucoup de logs. À utiliser temporairement pour le debug uniquement.

## Métriques et monitoring de CoreDNS

CoreDNS expose des métriques Prometheus riches.

### Métriques clés

**coredns_dns_requests_total**
- Nombre total de requêtes DNS
- Labels : type (A, AAAA, SRV...), zone

**coredns_dns_request_duration_seconds**
- Temps de réponse des requêtes
- Permet de détecter la latence

**coredns_dns_responses_total**
- Réponses par code (NOERROR, NXDOMAIN, SERVFAIL)
- Permet de détecter les erreurs

**coredns_cache_hits_total / coredns_cache_misses_total**
- Efficacité du cache
- Ratio hits/misses

**coredns_forward_requests_total**
- Requêtes transférées vers DNS externes

### Dashboard Grafana

Si vous avez Prometheus et Grafana installés, il existe des dashboards pré-faits pour CoreDNS :

- Dashboard ID : 5926 (CoreDNS)
- Dashboard ID : 7279 (Kubernetes CoreDNS)

## Haute disponibilité de CoreDNS

Par défaut, MicroK8s déploie 2 replicas de CoreDNS pour la redondance.

### Pourquoi 2 replicas ?

**Redondance :**
- Si un Pod CoreDNS tombe, l'autre continue de servir
- Pas de Single Point of Failure (SPOF)

**Performance :**
- Les requêtes sont distribuées entre les 2 Pods
- Load balancing automatique via le Service

### Service kube-dns

Le Service `kube-dns` utilise un load balancer interne :

```yaml
apiVersion: v1
kind: Service
metadata:
  name: kube-dns
  namespace: kube-system
spec:
  clusterIP: 10.152.183.10
  selector:
    k8s-app: kube-dns
  ports:
  - port: 53
    protocol: UDP
  - port: 53
    protocol: TCP
```

**Fonctionnement :**

Quand un Pod envoie une requête DNS vers `10.152.183.10:53`, le Service la redirige vers un des Pods CoreDNS disponibles (round-robin).

### Augmenter le nombre de replicas

Pour de gros clusters ou charges élevées :

```bash
microk8s kubectl scale deployment coredns -n kube-system --replicas=3
```

**Quand augmenter ?**
- Cluster > 50 Nodes
- Beaucoup de création/suppression de Pods (churn élevé)
- Latence DNS > 50ms en moyenne

## Points clés à retenir

### ✅ CoreDNS : le DNS de Kubernetes

- **Serveur DNS** moderne et extensible
- **Installé via addon** dans MicroK8s (`microk8s enable dns`)
- **Tourne dans kube-system** avec 2 replicas par défaut

### ✅ Format des noms DNS

```
<service>.<namespace>.svc.cluster.local
```

- **Nom court** : Fonctionne dans le même namespace
- **Avec namespace** : Fonctionne cross-namespace
- **FQDN complet** : Toujours fonctionnel, non ambigu

### ✅ Résolution DNS

1. Pod envoie requête → CoreDNS
2. CoreDNS interroge API Kubernetes
3. CoreDNS met en cache (30s)
4. CoreDNS répond au Pod

### ✅ Corefile : configuration CoreDNS

- **Plugin kubernetes** : Résolution des Services/Pods Kubernetes
- **Plugin forward** : Transfère vers DNS externes
- **Plugin cache** : Améliore les performances

### ✅ Headless Services

- `clusterIP: None`
- DNS retourne toutes les IPs des Pods
- Utilisé pour connexions directes aux Pods

### ✅ Debugging

- Vérifier que CoreDNS tourne
- Inspecter `/etc/resolv.conf` des Pods
- Utiliser `nslookup` pour tester
- Surveiller les métriques Prometheus

## Pourquoi tout cela est important ?

Le DNS est **fondamental** dans Kubernetes :

1. **Découverte de services** : Les applications trouvent les dépendances par nom
2. **Découplage** : Le code ne dépend pas des IPs (qui changent)
3. **Portabilité** : Le même code fonctionne dans tous les environnements
4. **Simplicité** : Pas besoin de service discovery complexe

Sans DNS, vous devriez :
- Coder les IPs en dur (cauchemar de maintenance)
- Implémenter votre propre service discovery
- Reconfigurer à chaque changement

Avec DNS, tout fonctionne automatiquement et élégamment !

## Prochaines étapes

Maintenant que vous maîtrisez le DNS avec CoreDNS, les prochains chapitres exploreront :

- **7.4 Communication Pod-to-Pod et Service-to-Pod** : Exemples concrets de communication
- **7.5 Test de connectivité interne** : Outils et commandes pour diagnostiquer le réseau

Le DNS est la pierre angulaire de la communication dans Kubernetes. Avec ces connaissances, vous comprenez comment vos applications se trouvent et communiquent entre elles !

---

**Note :** CoreDNS est hautement configurable. Ce chapitre couvre les bases essentielles. Pour des cas d'usage avancés (multi-cluster, DNS externe custom, etc.), consultez la documentation officielle de CoreDNS.

⏭️ [Communication Pod-to-Pod et Service-to-Pod](/07-reseau-kubernetes-interne/04-communication-pod-to-pod-et-service-to-pod.md)
