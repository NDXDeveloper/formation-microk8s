🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 10.1 Concepts d'Ingress

## Introduction

Imaginez que vous ayez plusieurs applications web hébergées dans votre cluster Kubernetes : un blog, une API, un tableau de bord d'administration, etc. Comment permettre aux utilisateurs externes d'accéder à ces différentes applications de manière simple et élégante ? C'est exactement le problème que résout **Ingress**.

Dans ce chapitre, nous allons découvrir ce qu'est un Ingress et pourquoi il est devenu un composant essentiel dans l'écosystème Kubernetes.

## Le Problème : Exposer Plusieurs Applications

### Rappel sur les Services

Vous avez déjà appris que les Services Kubernetes permettent d'exposer vos applications. Il existe trois types principaux :

1. **ClusterIP** : accès interne uniquement (par défaut)
2. **NodePort** : exposition sur un port spécifique de chaque nœud (30000-32767)
3. **LoadBalancer** : attribution d'une IP externe (nécessite MetalLB dans MicroK8s)

### Les Limites des Services Traditionnels

Prenons un exemple concret. Vous avez trois applications dans votre cluster :
- Un site web sur `mon-site.com`
- Une API sur `api.mon-site.com`
- Un blog sur `blog.mon-site.com`

**Avec les Services de type NodePort**, vous devriez :
- Exposer chaque application sur un port différent (ex: 30001, 30002, 30003)
- Les utilisateurs devraient accéder avec des URLs compliquées : `mon-site.com:30001`, `mon-site.com:30002`, etc.
- Ce n'est pas pratique et peu professionnel

**Avec les Services de type LoadBalancer**, vous devriez :
- Créer une IP externe pour chaque application
- Gérer plusieurs adresses IP différentes
- Cela peut coûter cher et devenir complexe à gérer

## Qu'est-ce qu'un Ingress ?

Un **Ingress** est une ressource Kubernetes qui gère l'accès externe aux services de votre cluster, typiquement en HTTP et HTTPS. C'est comme un **réceptionniste intelligent** ou un **routeur HTTP** qui dirige le trafic entrant vers la bonne application en fonction de règles que vous définissez.

### Analogie Simple

Pensez à un Ingress comme à la **réception d'un immeuble de bureaux** :

- **Sans Ingress** : Chaque entreprise (application) aurait besoin de sa propre entrée avec une adresse différente. Les visiteurs devraient connaître l'entrée exacte pour chaque entreprise.

- **Avec Ingress** : Il y a une seule entrée principale (une seule adresse IP). Le réceptionniste (Ingress) regarde le nom de l'entreprise que vous cherchez (le nom de domaine dans l'URL) et vous dirige vers le bon étage/bureau (le bon Service).

### Schéma Conceptuel

```
Internet
   ↓
[Ingress] (IP unique : 192.168.1.100)
   ↓
   ├─→ mon-site.com      → Service "frontend"    → Pods de l'application web
   ├─→ api.mon-site.com  → Service "api"         → Pods de l'API
   └─→ blog.mon-site.com → Service "blog"        → Pods du blog
```

## Les Avantages de l'Ingress

### 1. Une Seule Adresse IP

Au lieu d'avoir une IP différente pour chaque application, vous n'avez besoin que d'**une seule IP externe**. C'est l'Ingress qui se charge de router le trafic vers la bonne application.

### 2. Routage Intelligent

L'Ingress peut router le trafic de plusieurs façons :

**Par nom d'hôte (host-based routing)** :
```
blog.mon-site.com    → Service blog
api.mon-site.com     → Service api
admin.mon-site.com   → Service admin
```

**Par chemin (path-based routing)** :
```
mon-site.com/blog    → Service blog
mon-site.com/api     → Service api
mon-site.com/admin   → Service admin
```

### 3. Gestion SSL/TLS Centralisée

Au lieu de configurer les certificats HTTPS sur chaque application, vous pouvez gérer tous vos certificats SSL/TLS au niveau de l'Ingress. C'est beaucoup plus simple et plus sûr.

### 4. Fonctionnalités Avancées

L'Ingress peut également fournir :
- **Redirections HTTP vers HTTPS** automatiques
- **Réécriture d'URL** (URL rewriting)
- **Authentification basique** pour protéger certaines routes
- **Limitation de débit** (rate limiting)
- **Équilibrage de charge** entre plusieurs pods

## Architecture de l'Ingress

### Les Deux Composants Clés

Pour comprendre Ingress, il faut connaître ces deux éléments :

#### 1. La Ressource Ingress

C'est un **fichier de configuration YAML** que vous créez pour définir vos règles de routage. C'est comme un plan ou un ensemble d'instructions.

Exemple simplifié :
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: mon-ingress
spec:
  rules:
  - host: mon-site.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: frontend
            port:
              number: 80
```

Cette configuration dit : "Quand quelqu'un visite `mon-site.com`, dirige-le vers le Service `frontend` sur le port 80".

#### 2. L'Ingress Controller

La ressource Ingress seule ne fait rien. Vous avez besoin d'un **Ingress Controller** pour interpréter et appliquer ces règles. C'est le composant qui fait le travail réel.

**Analogie** :
- La ressource Ingress = le plan d'architecte
- L'Ingress Controller = les ouvriers qui construisent selon le plan

### Les Différents Ingress Controllers

Il existe plusieurs implémentations d'Ingress Controller. Les plus populaires sont :

1. **NGINX Ingress Controller** (le plus utilisé, inclus dans MicroK8s)
2. **Traefik**
3. **HAProxy**
4. **Kong**
5. **Contour**

Dans MicroK8s, nous utiliserons **NGINX Ingress Controller** car il est :
- Mature et stable
- Bien documenté
- Facile à activer avec `microk8s enable ingress`
- Très performant

## Comment Fonctionne l'Ingress ?

Voici le flux complet d'une requête HTTP avec Ingress :

```
1. Utilisateur tape "blog.mon-site.com" dans son navigateur
   ↓
2. Le DNS résout vers l'adresse IP de l'Ingress (ex: 192.168.1.100)
   ↓
3. La requête HTTP arrive sur l'Ingress Controller
   ↓
4. L'Ingress Controller examine l'en-tête "Host: blog.mon-site.com"
   ↓
5. Il consulte les règles définies dans les ressources Ingress
   ↓
6. Il trouve que "blog.mon-site.com" doit être routé vers le Service "blog"
   ↓
7. Il envoie la requête au Service "blog"
   ↓
8. Le Service "blog" transmet la requête à un Pod disponible
   ↓
9. Le Pod traite la requête et renvoie la réponse
   ↓
10. La réponse remonte via l'Ingress jusqu'à l'utilisateur
```

## Comparaison : Services vs Ingress

Pour clarifier, voici un tableau comparatif :

| Critère | Service (LoadBalancer/NodePort) | Ingress |
|---------|----------------------------------|---------|
| **Nombre d'IPs externes** | Une par service | Une pour tous |
| **Routage par nom de domaine** | Non | Oui |
| **Routage par chemin** | Non | Oui |
| **Gestion SSL/TLS** | Sur chaque service | Centralisée |
| **Coût** | Plus élevé (plusieurs IPs) | Économique (une IP) |
| **Complexité** | Simple | Moyenne |
| **Cas d'usage** | Application unique | Applications multiples |

## Quand Utiliser un Ingress ?

### ✅ Utilisez un Ingress quand :

- Vous avez **plusieurs applications web** à exposer
- Vous voulez utiliser des **noms de domaine** différents (blog.example.com, api.example.com)
- Vous voulez router par **chemins** (/blog, /api, /admin)
- Vous voulez **centraliser la gestion SSL/TLS**
- Vous voulez des **fonctionnalités avancées** (redirections, authentification, etc.)

### ❌ N'utilisez PAS un Ingress quand :

- Vous avez une **seule application simple** à exposer (un Service LoadBalancer suffit)
- Vous n'avez pas besoin de **routage HTTP/HTTPS** (ex: base de données)
- Votre protocole n'est **pas HTTP/HTTPS** (ex: TCP pur, UDP)

## Cas d'Usage Concrets

### Exemple 1 : Plateforme Web Complète

Vous développez une plateforme avec :
- Un site vitrine : `www.monprojet.com`
- Une application web : `app.monprojet.com`
- Une API REST : `api.monprojet.com`
- Un espace administrateur : `admin.monprojet.com`

**Avec Ingress** : Une seule IP externe, quatre noms de domaine, routage intelligent automatique.

### Exemple 2 : Environnement de Développement

Vous testez plusieurs versions de votre application :
- Version stable : `monapp.com/stable`
- Version beta : `monapp.com/beta`
- Version de développement : `monapp.com/dev`

**Avec Ingress** : Une seule URL de base, routage par chemin vers différents Services/Déploiements.

### Exemple 3 : Microservices

Votre architecture microservices comprend :
- Frontend : `boutique.com`
- Service de catalogue : `boutique.com/catalogue`
- Service de panier : `boutique.com/panier`
- Service de paiement : `boutique.com/paiement`

**Avec Ingress** : Une façade unifiée pour tous vos microservices.

## Ingress dans MicroK8s

Dans MicroK8s, l'activation de l'Ingress est extrêmement simple grâce au système d'addons :

```bash
microk8s enable ingress
```

Cette commande installe automatiquement le **NGINX Ingress Controller** dans votre cluster. En quelques secondes, vous êtes prêt à créer vos règles de routage.

## Points Clés à Retenir

🔑 **Ingress = Routeur HTTP intelligent** pour gérer l'accès externe à vos applications

🔑 **Une seule IP externe** pour exposer plusieurs applications avec des noms de domaine différents

🔑 **Deux composants** : la ressource Ingress (configuration) et l'Ingress Controller (implémentation)

🔑 **Routage flexible** : par nom d'hôte (host-based) ou par chemin (path-based)

🔑 **Gestion SSL/TLS centralisée** pour tous vos certificats HTTPS

🔑 **NGINX Ingress Controller** est le plus populaire et s'active facilement dans MicroK8s

🔑 **Utilisation recommandée** : dès que vous avez plusieurs applications web à exposer

## Vocabulaire Important

- **Ingress** : Ressource Kubernetes qui définit les règles de routage HTTP/HTTPS
- **Ingress Controller** : Composant qui implémente ces règles (NGINX, Traefik, etc.)
- **Host-based routing** : Routage basé sur le nom de domaine (ex: blog.example.com)
- **Path-based routing** : Routage basé sur le chemin d'URL (ex: example.com/blog)
- **Backend** : Le Service Kubernetes vers lequel le trafic est dirigé
- **TLS Termination** : Déchiffrement HTTPS au niveau de l'Ingress

## Prochaines Étapes

Maintenant que vous comprenez les concepts d'Ingress, vous êtes prêt à :
- Installer NGINX Ingress Controller dans MicroK8s (section 10.2)
- Créer vos premières règles de routage (sections 10.8 et 10.9)
- Configurer des certificats SSL/TLS automatiques (chapitre 11)

L'Ingress est un outil puissant qui va transformer la manière dont vous exposez vos applications. Avec MicroK8s, sa mise en œuvre est particulièrement simple et rapide !

---

**📚 Résumé du chapitre** : L'Ingress est une ressource Kubernetes qui permet de gérer l'accès HTTP/HTTPS à vos applications avec une seule adresse IP. Il offre un routage intelligent par nom de domaine ou par chemin, et centralise la gestion des certificats SSL. C'est l'outil idéal pour exposer plusieurs applications web de manière professionnelle et économique.

⏭️ [Installation NGINX Ingress Controller (microk8s enable ingress)](/10-ingress-et-routage/02-installation-nginx-ingress-controller.md)
