🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 16.4 Network Policies

## Introduction

Les Network Policies sont des règles de pare-feu pour Kubernetes. Elles contrôlent le trafic réseau entre les pods et vers l'extérieur. Sans Network Policies, tous les pods peuvent communiquer librement entre eux, ce qui peut poser des problèmes de sécurité.

**Analogie simple :** Imaginez un immeuble de bureaux où tous les employés peuvent entrer dans tous les bureaux sans restriction. Les Network Policies sont comme des badges d'accès qui définissent qui peut entrer dans quel bureau et par quelle porte.

## Le Problème : Réseau Ouvert par Défaut

### Comportement par Défaut de Kubernetes

Par défaut, dans Kubernetes :

```
┌─────────────────────────────────────────┐
│           Cluster Kubernetes            │
│                                         │
│  ┌─────────┐      ┌─────────┐           │
│  │ Pod A   │◄────►│ Pod B   │           │
│  │ (web)   │      │ (api)   │           │
│  └─────────┘      └─────────┘           │
│       ▲                ▲                │
│       │                │                │
│       ▼                ▼                │
│  ┌─────────┐      ┌─────────┐           │
│  │ Pod C   │◄────►│ Pod D   │           │
│  │ (db)    │      │ (cache) │           │
│  └─────────┘      └─────────┘           │
│                                         │
│  Tous les pods peuvent communiquer      │
│  avec tous les autres pods              │
└─────────────────────────────────────────┘
```

**Problèmes :**
- Un pod compromis peut attaquer tous les autres pods
- Pas de segmentation réseau
- Difficile de suivre les flux de communication légitimes
- Violation du principe de moindre privilège

### Avec Network Policies

```
┌─────────────────────────────────────────┐
│           Cluster Kubernetes            │
│                                         │
│  ┌─────────┐      ┌─────────┐           │
│  │ Pod A   │─────►│ Pod B   │           │
│  │ (web)   │      │ (api)   │           │
│  └─────────┘      └─────────┘           │
│                        │                │
│                        ▼                │
│                   ┌─────────┐           │
│                   │ Pod C   │           │
│                   │ (db)    │           │
│                   └─────────┘           │
│                                         │
│  ✓ Web peut parler à API                │
│  ✓ API peut parler à DB                 │
│  ✗ Web ne peut PAS parler à DB          │
│  ✗ DB ne peut PAS être contacté         │
│      directement depuis Web             │
└─────────────────────────────────────────┘
```

## Concepts Fondamentaux

### 1. Ingress vs Egress

**Ingress (Entrée) :**
- Trafic **entrant** vers un pod
- Qui peut se connecter **à** ce pod ?

**Egress (Sortie) :**
- Trafic **sortant** d'un pod
- Vers où ce pod peut-il se connecter ?

```
        Ingress (↓)
            │
    ┌───────▼────────┐
    │      Pod       │
    └───────┬────────┘
            │
        Egress (↓)
```

**Exemple concret :**
- **Ingress** : Le pod "api" peut recevoir des connexions sur le port 8080 depuis les pods "web"
- **Egress** : Le pod "api" peut se connecter au pod "database" sur le port 5432

### 2. Selectors (Sélecteurs)

Les Network Policies utilisent des labels pour identifier les pods :

```yaml
# Cette policy s'applique aux pods avec le label app=api
podSelector:
  matchLabels:
    app: api
```

### 3. Types de Règles

Une Network Policy peut autoriser le trafic depuis/vers :

1. **Pods spécifiques** (via labels)
2. **Namespaces entiers** (via labels de namespace)
3. **Blocs d'adresses IP** (CIDR)

### 4. Comportement par Défaut

**Important :** Dès qu'une Network Policy sélectionne un pod, ce pod devient **isolé par défaut**.

```
Sans policy : Pod accessible par tous
Avec policy : Pod accessible uniquement selon les règles définies
```

## Structure d'une Network Policy

### Anatomie Complète

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: example-policy
  namespace: production
spec:
  # 1. À quels pods s'applique cette policy ?
  podSelector:
    matchLabels:
      app: api

  # 2. Types de trafic contrôlés
  policyTypes:
  - Ingress  # Contrôle le trafic entrant
  - Egress   # Contrôle le trafic sortant

  # 3. Règles d'entrée (Ingress)
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: web
    ports:
    - protocol: TCP
      port: 8080

  # 4. Règles de sortie (Egress)
  egress:
  - to:
    - podSelector:
        matchLabels:
          app: database
    ports:
    - protocol: TCP
      port: 5432
```

### Décomposition

```
NetworkPolicy
    │
    ├─── podSelector (À QUI s'applique la policy ?)
    │
    ├─── policyTypes (Ingress, Egress, ou les deux)
    │
    ├─── ingress[] (Règles d'entrée)
    │    ├─── from[] (Depuis où ?)
    │    └─── ports[] (Sur quels ports ?)
    │
    └─── egress[] (Règles de sortie)
         ├─── to[] (Vers où ?)
         └─── ports[] (Sur quels ports ?)
```

## Exemples de Base

### Exemple 1 : Isolation Complète (Deny All)

**Besoin :** Bloquer tout le trafic vers les pods d'un namespace.

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: deny-all
  namespace: production
spec:
  podSelector: {}  # Sélectionne TOUS les pods du namespace
  policyTypes:
  - Ingress
  - Egress
  # Aucune règle ingress/egress = tout est bloqué
```

**Résultat :**
- ❌ Aucun pod ne peut recevoir de trafic
- ❌ Aucun pod ne peut envoyer de trafic
- ⚠️ Même le DNS ne fonctionne plus !

**Usage :** Base pour construire une approche "whitelist" (autoriser explicitement).

### Exemple 2 : Autoriser DNS Uniquement

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-dns-only
  namespace: production
spec:
  podSelector: {}
  policyTypes:
  - Egress
  egress:
  # Autoriser DNS (CoreDNS dans kube-system)
  - to:
    - namespaceSelector:
        matchLabels:
          kubernetes.io/metadata.name: kube-system
    ports:
    - protocol: UDP
      port: 53
```

**Résultat :**
- ✅ Les pods peuvent résoudre les noms DNS
- ❌ Tout autre trafic sortant est bloqué

### Exemple 3 : Autoriser l'Accès à un Service Spécifique

**Besoin :** Le frontend peut accéder au backend, mais rien d'autre.

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: backend-policy
  namespace: production
spec:
  podSelector:
    matchLabels:
      app: backend
      tier: api
  policyTypes:
  - Ingress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: frontend
    ports:
    - protocol: TCP
      port: 8080
```

**Résultat :**
- ✅ Pods "frontend" peuvent se connecter au "backend" sur le port 8080
- ❌ Tout autre pod est bloqué

## Scénarios Pratiques

### Scénario 1 : Architecture Web 3-Tiers

```
┌──────────┐
│   Web    │  (frontend)
└────┬─────┘
     │ ✓ HTTP/8080
     ▼
┌──────────┐
│   API    │  (backend)
└────┬─────┘
     │ ✓ PostgreSQL/5432
     ▼
┌──────────┐
│    DB    │  (database)
└──────────┘
```

**Policy 1 : Frontend (Web)**

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: web-policy
  namespace: production
spec:
  podSelector:
    matchLabels:
      tier: frontend
  policyTypes:
  - Ingress
  - Egress
  ingress:
  # Recevoir du trafic depuis Internet (Ingress controller)
  - from:
    - namespaceSelector:
        matchLabels:
          name: ingress-nginx
    ports:
    - protocol: TCP
      port: 80
  egress:
  # Sortie vers API
  - to:
    - podSelector:
        matchLabels:
          tier: backend
    ports:
    - protocol: TCP
      port: 8080
  # Sortie vers DNS
  - to:
    - namespaceSelector:
        matchLabels:
          kubernetes.io/metadata.name: kube-system
    ports:
    - protocol: UDP
      port: 53
```

**Policy 2 : Backend (API)**

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: api-policy
  namespace: production
spec:
  podSelector:
    matchLabels:
      tier: backend
  policyTypes:
  - Ingress
  - Egress
  ingress:
  # Recevoir uniquement depuis Frontend
  - from:
    - podSelector:
        matchLabels:
          tier: frontend
    ports:
    - protocol: TCP
      port: 8080
  egress:
  # Sortie vers Database
  - to:
    - podSelector:
        matchLabels:
          tier: database
    ports:
    - protocol: TCP
      port: 5432
  # Sortie vers DNS
  - to:
    - namespaceSelector:
        matchLabels:
          kubernetes.io/metadata.name: kube-system
    ports:
    - protocol: UDP
      port: 53
```

**Policy 3 : Database**

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: db-policy
  namespace: production
spec:
  podSelector:
    matchLabels:
      tier: database
  policyTypes:
  - Ingress
  - Egress
  ingress:
  # Recevoir uniquement depuis Backend
  - from:
    - podSelector:
        matchLabels:
          tier: backend
    ports:
    - protocol: TCP
      port: 5432
  egress:
  # Sortie vers DNS uniquement
  - to:
    - namespaceSelector:
        matchLabels:
          kubernetes.io/metadata.name: kube-system
    ports:
    - protocol: UDP
      port: 53
```

### Scénario 2 : Isolation par Namespace

**Besoin :** Les pods de "dev" et "prod" ne doivent pas pouvoir communiquer entre eux.

```yaml
---
# Policy dans le namespace 'production'
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: deny-from-other-namespaces
  namespace: production
spec:
  podSelector: {}  # Tous les pods
  policyTypes:
  - Ingress
  ingress:
  # Autoriser uniquement depuis le même namespace
  - from:
    - podSelector: {}  # Tous les pods du namespace courant
---
# Policy dans le namespace 'dev'
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: deny-from-other-namespaces
  namespace: dev
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  ingress:
  - from:
    - podSelector: {}
```

**Résultat :**
- ✅ Pods dans "production" peuvent communiquer entre eux
- ✅ Pods dans "dev" peuvent communiquer entre eux
- ❌ Pods de "production" ne peuvent PAS accéder à "dev"
- ❌ Pods de "dev" ne peuvent PAS accéder à "production"

### Scénario 3 : Accès Externe Contrôlé

**Besoin :** Autoriser un pod à accéder à une API externe (ex: api.github.com).

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-external-api
  namespace: production
spec:
  podSelector:
    matchLabels:
      app: github-sync
  policyTypes:
  - Egress
  egress:
  # Autoriser GitHub API (exemple d'IP)
  - to:
    - ipBlock:
        cidr: 140.82.112.0/20  # Plage IP de GitHub
    ports:
    - protocol: TCP
      port: 443
  # Autoriser DNS
  - to:
    - namespaceSelector:
        matchLabels:
          kubernetes.io/metadata.name: kube-system
    ports:
    - protocol: UDP
      port: 53
  # Autoriser tout le trafic HTTPS (alternative)
  - to:
    - ipBlock:
        cidr: 0.0.0.0/0
        except:
        - 10.0.0.0/8      # Bloquer réseau privé
        - 172.16.0.0/12   # Bloquer réseau privé
        - 192.168.0.0/16  # Bloquer réseau privé
    ports:
    - protocol: TCP
      port: 443
```

### Scénario 4 : Monitoring et Observabilité

**Besoin :** Prometheus doit pouvoir scraper les métriques de tous les pods.

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-prometheus-scraping
  namespace: production
spec:
  podSelector:
    matchLabels:
      monitoring: "true"  # Pods qui exposent des métriques
  policyTypes:
  - Ingress
  ingress:
  # Autoriser Prometheus
  - from:
    - namespaceSelector:
        matchLabels:
          name: monitoring
      podSelector:
        matchLabels:
          app: prometheus
    ports:
    - protocol: TCP
      port: 9090  # Port des métriques
```

## Sélecteurs Avancés

### 1. Pod Selector (Sélection par Labels de Pods)

```yaml
# Autoriser depuis les pods avec app=web
- from:
  - podSelector:
      matchLabels:
        app: web
```

### 2. Namespace Selector (Sélection par Namespace)

```yaml
# Autoriser depuis tous les pods du namespace "monitoring"
- from:
  - namespaceSelector:
      matchLabels:
        name: monitoring
```

### 3. Combinaison Pod + Namespace

```yaml
# Autoriser depuis les pods "prometheus" du namespace "monitoring"
- from:
  - namespaceSelector:
      matchLabels:
        name: monitoring
    podSelector:
      matchLabels:
        app: prometheus
```

**Note :** C'est un **ET** logique (même entrée `from`).

### 4. Plusieurs Sources (OU Logique)

```yaml
# Autoriser depuis web OU depuis admin
- from:
  - podSelector:
      matchLabels:
        app: web
  - podSelector:
      matchLabels:
        app: admin
```

**Note :** C'est un **OU** logique (entrées `from` séparées).

### 5. IP Blocks (CIDR)

```yaml
# Autoriser depuis une plage d'adresses IP
- from:
  - ipBlock:
      cidr: 192.168.1.0/24
      except:
      - 192.168.1.5/32  # Exclure cette IP spécifique
```

**Usage :** Services externes, IPs de confiance, pare-feu.

## Ports et Protocoles

### Spécifier des Ports

```yaml
ports:
- protocol: TCP
  port: 8080
- protocol: TCP
  port: 8443
- protocol: UDP
  port: 53
```

### Port Range (non supporté directement)

```yaml
# ❌ Ceci n'existe pas
ports:
- protocol: TCP
  port: 8000-9000

# ✅ Solution : lister tous les ports ou ne pas spécifier de port
```

### Ne Pas Spécifier de Port

```yaml
# Autoriser TOUS les ports
- from:
  - podSelector:
      matchLabels:
        app: web
  # Pas de section 'ports' = tous les ports autorisés
```

### Protocoles Supportés

- **TCP**
- **UDP**
- **SCTP** (Stream Control Transmission Protocol)

## Patterns Courants

### Pattern 1 : Deny All + Whitelist

**Approche recommandée pour la production.**

```yaml
---
# Étape 1 : Tout bloquer par défaut
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-all
  namespace: production
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  - Egress
---
# Étape 2 : Autoriser DNS
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-dns
  namespace: production
spec:
  podSelector: {}
  policyTypes:
  - Egress
  egress:
  - to:
    - namespaceSelector:
        matchLabels:
          kubernetes.io/metadata.name: kube-system
    ports:
    - protocol: UDP
      port: 53
---
# Étape 3 : Autoriser les flux spécifiques
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-web-to-api
  namespace: production
spec:
  podSelector:
    matchLabels:
      app: api
  policyTypes:
  - Ingress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: web
    ports:
    - protocol: TCP
      port: 8080
```

### Pattern 2 : Autoriser Ingress Controller

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-ingress-controller
  namespace: production
spec:
  podSelector:
    matchLabels:
      expose: "true"  # Pods exposés publiquement
  policyTypes:
  - Ingress
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          name: ingress-nginx
    ports:
    - protocol: TCP
      port: 80
    - protocol: TCP
      port: 443
```

### Pattern 3 : Pods Stateless (Sortie Libre, Entrée Contrôlée)

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: stateless-app
  namespace: production
spec:
  podSelector:
    matchLabels:
      type: stateless
  policyTypes:
  - Ingress
  # Egress non spécifié = sortie libre
  ingress:
  - from:
    - podSelector:
        matchLabels:
          tier: frontend
```

### Pattern 4 : Autoriser Communication Intra-Namespace

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-same-namespace
  namespace: production
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  ingress:
  - from:
    - podSelector: {}  # Tous les pods du même namespace
```

## Bonnes Pratiques

### 1. Commencer par Deny All

```yaml
# Première policy à créer dans chaque namespace
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-all
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  - Egress
```

**Avantages :**
- Approche whitelist (plus sûre)
- Force à documenter les flux
- Détecte rapidement les dépendances

### 2. Toujours Autoriser le DNS

```yaml
egress:
- to:
  - namespaceSelector:
      matchLabels:
        kubernetes.io/metadata.name: kube-system
  ports:
  - protocol: UDP
    port: 53
```

**Pourquoi ?** Sans DNS, les pods ne peuvent pas résoudre les noms de services.

### 3. Utiliser des Labels Cohérents

```yaml
# Bon : labels descriptifs et cohérents
metadata:
  labels:
    app: api
    tier: backend
    environment: production
```

**Avantage :** Sélection facile dans les Network Policies.

### 4. Documenter les Policies

```yaml
metadata:
  name: api-backend-policy
  annotations:
    description: "Autorise le frontend à accéder au backend API"
    owner: "team-backend"
    flows: "frontend:80 -> backend:8080"
```

### 5. Tester les Policies Progressivement

**Étapes recommandées :**
1. Déployer dans un namespace de test
2. Tester avec `kubectl exec` et `curl`
3. Vérifier les logs des applications
4. Déployer en production

### 6. Un Namespace = Une Stratégie de Sécurité

```yaml
# production : strict
default-deny + whitelist explicite

# staging : modéré
allow intra-namespace

# dev : ouvert
pas de restrictions
```

### 7. Surveiller et Auditer

```bash
# Lister toutes les Network Policies
kubectl get networkpolicies --all-namespaces

# Voir les détails
kubectl describe networkpolicy <name> -n <namespace>
```

### 8. Combiner avec RBAC

Network Policies contrôlent le **réseau**, RBAC contrôle l'**API**.

```
RBAC : Qui peut créer/modifier des ressources ?
Network Policies : Qui peut communiquer avec qui ?
```

Les deux sont nécessaires pour une sécurité complète.

## Débogage

### Problème : Connexion Refusée / Timeout

**Symptômes :**
```bash
kubectl exec web-pod -- curl api-service:8080
# Timeout ou Connection refused
```

**Diagnostic :**

1. **Vérifier si une Network Policy s'applique :**
```bash
kubectl get networkpolicies -n production
kubectl describe pod api-pod -n production
```

2. **Vérifier les labels :**
```bash
# Labels du pod source
kubectl get pod web-pod -n production --show-labels

# Labels du pod destination
kubectl get pod api-pod -n production --show-labels
```

3. **Vérifier la Network Policy :**
```bash
kubectl describe networkpolicy api-policy -n production
```

4. **Tester sans Network Policy :**
```bash
# Supprimer temporairement la policy
kubectl delete networkpolicy api-policy -n production

# Tester la connexion
kubectl exec web-pod -- curl api-service:8080

# Recréer la policy
kubectl apply -f api-policy.yaml
```

### Problème : DNS Ne Fonctionne Pas

**Symptômes :**
```bash
kubectl exec web-pod -- nslookup api-service
# Server: 10.152.183.10
# Address: 10.152.183.10#53
#
# ;; connection timed out; no servers could be reached
```

**Solution :**
Ajouter une règle egress pour autoriser DNS :

```yaml
egress:
- to:
  - namespaceSelector:
      matchLabels:
        kubernetes.io/metadata.name: kube-system
  ports:
  - protocol: UDP
    port: 53
```

### Problème : Ingress Controller Bloqué

**Symptômes :**
L'application n'est pas accessible depuis l'extérieur.

**Solution :**
Autoriser le trafic depuis le namespace de l'Ingress Controller :

```yaml
ingress:
- from:
  - namespaceSelector:
      matchLabels:
        name: ingress-nginx  # ou le label de votre ingress
```

### Problème : Egress Vers Internet Bloqué

**Symptômes :**
Le pod ne peut pas contacter des APIs externes.

**Solutions :**

1. **Autoriser tout le trafic HTTPS :**
```yaml
egress:
- to:
  - ipBlock:
      cidr: 0.0.0.0/0
  ports:
  - protocol: TCP
    port: 443
```

2. **Autoriser des IPs spécifiques :**
```yaml
egress:
- to:
  - ipBlock:
      cidr: 140.82.112.0/20  # Exemple : GitHub
  ports:
  - protocol: TCP
    port: 443
```

### Outils de Débogage

```bash
# Tester la connectivité depuis un pod
kubectl exec -it web-pod -- curl -v api-service:8080

# Voir les logs du plugin CNI (Calico, etc.)
kubectl logs -n kube-system -l k8s-app=calico-node

# Vérifier les endpoints
kubectl get endpoints api-service -n production

# Tester avec un pod de debug
kubectl run debug --image=nicolaka/netshoot -it --rm -- bash
# Depuis ce pod, tester la connectivité
```

## Network Policies dans MicroK8s

### CNI Plugin Requis

MicroK8s nécessite un plugin CNI (Container Network Interface) pour supporter les Network Policies.

**Par défaut :** MicroK8s utilise un CNI basique qui **ne supporte pas** les Network Policies.

### Activer Calico

Calico est le plugin CNI recommandé pour les Network Policies dans MicroK8s :

```bash
# Activer Calico
microk8s enable community
microk8s enable calico

# Vérifier l'installation
microk8s kubectl get pods -n kube-system | grep calico
```

**Sortie attendue :**
```
calico-kube-controllers-xxx   1/1     Running
calico-node-xxx               1/1     Running
```

### Alternative : Cilium

```bash
# Cilium est une alternative moderne
microk8s enable cilium
```

### Vérifier le Support des Network Policies

```bash
# Créer une policy de test
cat <<EOF | microk8s kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: test-policy
  namespace: default
spec:
  podSelector:
    matchLabels:
      app: test
  policyTypes:
  - Ingress
EOF

# Vérifier
microk8s kubectl get networkpolicy
```

Si la commande fonctionne sans erreur, les Network Policies sont supportées.

### Désactiver Calico

```bash
microk8s disable calico
```

**Attention :** Désactiver Calico supprimera le support des Network Policies.

## Limitations et Considérations

### 1. Pas de Stateful Inspection

Les Network Policies sont **sans état** (stateless) pour les flux :

```
Pod A → Pod B (port 8080) : ✓ Autorisé
Pod B → Pod A (réponse)   : ✓ Autorisé automatiquement (même connexion)
```

Mais :

```
Pod B → Pod A (nouvelle connexion) : ❌ Nécessite règle explicite
```

### 2. Pas de Layer 7 (Application)

Les Network Policies opèrent aux couches 3-4 (IP, TCP/UDP), pas à la couche 7 (HTTP).

```
✓ Peut : Autoriser TCP sur port 80
✗ Ne peut pas : Autoriser uniquement GET /api/users
```

**Solution :** Utiliser un Service Mesh (Istio, Linkerd) pour le contrôle Layer 7.

### 3. Ordre d'Application

Les Network Policies sont **additives** (OR logique) :

```yaml
Policy 1 : Autoriser port 80
Policy 2 : Autoriser port 443
Résultat : Ports 80 ET 443 autorisés
```

Il n'y a **pas de priorité** entre policies. On ne peut pas avoir :
```
Policy 1 : Autoriser tout
Policy 2 : Bloquer port 22
Résultat souhaité : Tout sauf port 22 ❌ (impossible directement)
```

### 4. Performance

Les Network Policies ont un impact minimal sur les performances, mais :

- Trop de policies complexes peuvent ralentir le CNI
- Les gros clusters nécessitent un CNI performant (Cilium avec eBPF)

### 5. Pas de Logs Par Défaut

Les Network Policies ne loggent pas le trafic bloqué par défaut.

**Solution :** Utiliser les fonctionnalités de logging de votre CNI :
- Calico : `calicoctl` pour les logs
- Cilium : Hubble pour l'observabilité

## Exemples Complets

### Exemple 1 : Environnement de Production Sécurisé

```yaml
---
# 1. Tout bloquer par défaut
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-all
  namespace: production
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  - Egress
---
# 2. Autoriser DNS pour tous
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-dns
  namespace: production
spec:
  podSelector: {}
  policyTypes:
  - Egress
  egress:
  - to:
    - namespaceSelector:
        matchLabels:
          kubernetes.io/metadata.name: kube-system
    ports:
    - protocol: UDP
      port: 53
---
# 3. Frontend : recevoir depuis Ingress, sortir vers Backend
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: frontend-policy
  namespace: production
spec:
  podSelector:
    matchLabels:
      tier: frontend
  policyTypes:
  - Ingress
  - Egress
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          name: ingress-nginx
    ports:
    - protocol: TCP
      port: 80
  egress:
  - to:
    - podSelector:
        matchLabels:
          tier: backend
    ports:
    - protocol: TCP
      port: 8080
  - to:
    - namespaceSelector:
        matchLabels:
          kubernetes.io/metadata.name: kube-system
    ports:
    - protocol: UDP
      port: 53
---
# 4. Backend : recevoir depuis Frontend, sortir vers DB
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: backend-policy
  namespace: production
spec:
  podSelector:
    matchLabels:
      tier: backend
  policyTypes:
  - Ingress
  - Egress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          tier: frontend
    ports:
    - protocol: TCP
      port: 8080
  egress:
  - to:
    - podSelector:
        matchLabels:
          tier: database
    ports:
    - protocol: TCP
      port: 5432
  - to:
    - namespaceSelector:
        matchLabels:
          kubernetes.io/metadata.name: kube-system
    ports:
    - protocol: UDP
      port: 53
---
# 5. Database : recevoir uniquement depuis Backend
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: database-policy
  namespace: production
spec:
  podSelector:
    matchLabels:
      tier: database
  policyTypes:
  - Ingress
  - Egress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          tier: backend
    ports:
    - protocol: TCP
      port: 5432
  egress:
  - to:
    - namespaceSelector:
        matchLabels:
          kubernetes.io/metadata.name: kube-system
    ports:
    - protocol: UDP
      port: 53
```

### Exemple 2 : Isolation Multi-Tenant

```yaml
---
# Namespace client-a
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: isolate-tenant
  namespace: client-a
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  - Egress
  ingress:
  # Autoriser depuis le même namespace uniquement
  - from:
    - podSelector: {}
  # Autoriser depuis monitoring
  - from:
    - namespaceSelector:
        matchLabels:
          name: monitoring
  egress:
  # Sortie vers le même namespace
  - to:
    - podSelector: {}
  # Sortie vers DNS
  - to:
    - namespaceSelector:
        matchLabels:
          kubernetes.io/metadata.name: kube-system
    ports:
    - protocol: UDP
      port: 53
  # Sortie vers Internet (HTTPS)
  - to:
    - ipBlock:
        cidr: 0.0.0.0/0
        except:
        - 10.0.0.0/8
    ports:
    - protocol: TCP
      port: 443
```

## Commandes Utiles

### Gestion des Network Policies

```bash
# Lister toutes les network policies
kubectl get networkpolicies -A

# Dans un namespace spécifique
kubectl get networkpolicy -n production

# Détails d'une policy
kubectl describe networkpolicy api-policy -n production

# Format YAML
kubectl get networkpolicy api-policy -n production -o yaml

# Créer
kubectl apply -f network-policy.yaml

# Supprimer
kubectl delete networkpolicy api-policy -n production
```

### Test et Débogage

```bash
# Créer un pod de test
kubectl run test-pod --image=nicolaka/netshoot -it --rm -- bash

# Tester la connexion TCP
nc -zv api-service 8080

# Tester HTTP
curl -v http://api-service:8080

# Résolution DNS
nslookup api-service

# Vérifier les labels d'un pod
kubectl get pod my-pod --show-labels

# Voir quel(s) pod(s) une policy sélectionne
kubectl get pods -l app=api -n production
```

### Analyse

```bash
# Pods affectés par une policy
kubectl get networkpolicy api-policy -n production -o json | \
  jq '.spec.podSelector'

# Toutes les policies affectant un pod
kubectl get networkpolicies -n production -o json | \
  jq '.items[] | select(.spec.podSelector.matchLabels.app=="api")'
```

## Conclusion

Les Network Policies sont essentielles pour la sécurité réseau dans Kubernetes :

1. **Isolation** : segmenter le trafic réseau entre pods
2. **Moindre privilège** : autoriser uniquement les flux nécessaires
3. **Défense en profondeur** : couche de sécurité supplémentaire
4. **Compliance** : répondre aux exigences de sécurité

**Formule Network Policy :**
```
De (from/to) + Port + Protocole = Règle de trafic autorisé
```

**Points Clés à Retenir :**

- Par défaut, tout le trafic est autorisé dans Kubernetes
- Les Network Policies changent ce comportement pour les pods sélectionnés
- Approche recommandée : Deny All + Whitelist explicite
- Toujours autoriser DNS (port 53 UDP vers kube-system)
- Les policies sont additives (OR logique entre elles)
- Nécessite un CNI supportant les Network Policies (Calico, Cilium)
- Tester progressivement dans un environnement de non-production
- Documenter vos flux réseau

**Prochaines Étapes :**

Dans les sections suivantes, nous explorerons :
- **16.5** : Pod Security Standards (sécurité au niveau conteneur)
- **16.6** : Scan de vulnérabilités d'images
- **16.7** : Secrets management avancé

Les Network Policies, combinées avec RBAC et Pod Security, forment un système de sécurité robuste et en profondeur pour vos clusters Kubernetes.

⏭️ [Pod Security Standards](/16-securite-kubernetes/05-pod-security-standards.md)
