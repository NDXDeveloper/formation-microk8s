🔝 Retour au [Sommaire](/SOMMAIRE.md)

# Exemples de Network Policies pour MicroK8s

## Introduction

### Qu'est-ce qu'une Network Policy ?

Une Network Policy dans Kubernetes est un ensemble de règles qui contrôle comment les pods peuvent communiquer entre eux et avec d'autres points de terminaison réseau. C'est l'équivalent d'un firewall, mais au niveau des applications dans votre cluster.

**Analogie simple :** Imaginez un immeuble de bureaux. Les Network Policies sont comme les badges d'accès qui définissent quels employés peuvent entrer dans quels bureaux. Sans badge (Network Policy), tout le monde peut aller partout. Avec des badges, vous contrôlez finement les accès.

### Pourquoi utiliser des Network Policies ?

1. **Sécurité** : Limiter la surface d'attaque en cas de compromission d'un pod
2. **Isolation** : Séparer les environnements (dev, staging, production)
3. **Conformité** : Répondre aux exigences réglementaires (PCI-DSS, HIPAA)
4. **Principe du moindre privilège** : Chaque application n'accède qu'à ce dont elle a besoin
5. **Prévention de la propagation latérale** : Empêcher un attaquant de se déplacer dans le cluster

### Comportement par défaut de Kubernetes

**IMPORTANT** : Par défaut, Kubernetes autorise toute communication entre pods.

```
┌────────────┐         ┌────────────┐         ┌────────────┐
│   Pod A    │ ◄─────► │   Pod B    │ ◄─────► │   Pod C    │
│            │    ✓    │            │    ✓    │            │
└────────────┘         └────────────┘         └────────────┘
      │                                              │
      └──────────────────────►───────────────────────┘
                         ✓
        Tout est autorisé sans Network Policy
```

**Dès qu'une Network Policy sélectionne un pod**, le comportement change radicalement : tout devient interdit sauf ce qui est explicitement autorisé.

```
┌────────────┐         ┌────────────┐         ┌────────────┐
│   Pod A    │ ◄─────► │   Pod B    │    ✗    │   Pod C    │
│            │    ✓    │  (Policy)  │         │            │
└────────────┘         └────────────┘         └────────────┘
                              │
                              ✗
         Avec Network Policy : deny by default
```

### Prérequis

Les Network Policies nécessitent un plugin CNI (Container Network Interface) qui les supporte. **MicroK8s utilise Calico par défaut**, qui supporte pleinement les Network Policies.

Vérification :

```bash
# Vérifier que Calico est actif
microk8s kubectl get pods -n kube-system -l k8s-app=calico-node

# Devrait afficher des pods en status Running
```

---

## Anatomie d'une Network Policy

Voici la structure de base d'une Network Policy annotée :

```yaml
apiVersion: networking.k8s.io/v1      # Version de l'API
kind: NetworkPolicy                    # Type de ressource
metadata:
  name: exemple-policy                 # Nom de la policy
  namespace: default                   # Namespace où elle s'applique
spec:
  podSelector:                         # Quels pods sont concernés
    matchLabels:
      app: mon-app
  policyTypes:                         # Types de trafic à contrôler
  - Ingress                            # Trafic entrant
  - Egress                             # Trafic sortant
  ingress:                             # Règles pour le trafic entrant
  - from:                              # Qui peut se connecter
    - podSelector:                     # Sélection par pod
        matchLabels:
          role: frontend
    ports:                             # Sur quels ports
    - protocol: TCP
      port: 8080
  egress:                              # Règles pour le trafic sortant
  - to:                                # Vers où le pod peut se connecter
    - podSelector:
        matchLabels:
          role: database
    ports:
    - protocol: TCP
      port: 5432
```

### Composants clés

| Composant | Description | Exemple |
|-----------|-------------|---------|
| `podSelector` | Sélectionne les pods auxquels la policy s'applique | `app: backend` |
| `policyTypes` | Types de trafic contrôlé (Ingress, Egress, ou les deux) | `[Ingress, Egress]` |
| `ingress.from` | Source du trafic entrant autorisé | Pod, namespace, IP |
| `egress.to` | Destination du trafic sortant autorisé | Pod, namespace, IP |
| `ports` | Ports et protocoles autorisés | TCP 8080, UDP 53 |

---

## Exemples Progressifs - Niveau Débutant

### Exemple 1 : Bloquer tout le trafic entrant

La policy la plus simple et la plus restrictive.

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: deny-all-ingress
  namespace: default
spec:
  podSelector: {}           # {} = tous les pods du namespace
  policyTypes:
  - Ingress
  # Pas de règle ingress = tout est bloqué
```

**Ce que fait cette policy :**
- S'applique à **tous les pods** du namespace `default`
- Bloque **tout le trafic entrant**
- Le trafic sortant reste autorisé (pas dans policyTypes)

**Cas d'usage :** Point de départ pour un namespace sécurisé. Vous débloquez ensuite ce dont vous avez besoin.

**Résultat visuel :**

```
Internet ───✗───► Pod A
Pod B    ───✗───► Pod A
Pod C    ───✗───► Pod A

Pod A    ───✓───► Internet
Pod A    ───✓───► Pod B
```

### Exemple 2 : Bloquer tout le trafic sortant

L'inverse de l'exemple précédent.

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: deny-all-egress
  namespace: default
spec:
  podSelector: {}
  policyTypes:
  - Egress
  # Pas de règle egress = tout est bloqué
```

**Ce que fait cette policy :**
- S'applique à **tous les pods** du namespace `default`
- Bloque **tout le trafic sortant**
- Le trafic entrant reste autorisé

**Cas d'usage :** Pods qui ne doivent jamais communiquer vers l'extérieur (ex: workers qui ne font que recevoir des jobs).

### Exemple 3 : Isolation complète (deny-all ingress et egress)

Combinaison des deux précédents.

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: deny-all-traffic
  namespace: default
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  - Egress
```

**Ce que fait cette policy :**
- Bloque **tout le trafic entrant ET sortant**
- Les pods deviennent complètement isolés

**Cas d'usage :** Mettre un namespace en quarantaine complète.

**⚠️ ATTENTION** : Cette policy bloque même la résolution DNS ! Les pods ne pourront plus rien faire.

### Exemple 4 : Autoriser tout le trafic (allow-all)

L'opposé complet - explicitement autoriser tout.

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-all-ingress
  namespace: default
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  ingress:
  - {}                      # {} vide = autoriser depuis n'importe où
```

**Ce que fait cette policy :**
- Autorise explicitement tout le trafic entrant
- Équivalent au comportement par défaut de Kubernetes

**Cas d'usage :** Rarement utilisé. Peut servir à annuler des policies plus restrictives dans certains cas spécifiques.

### Exemple 5 : Autoriser un seul pod spécifique

Premier exemple de règle ciblée.

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-from-frontend
  namespace: default
spec:
  podSelector:
    matchLabels:
      app: backend          # S'applique aux pods backend
  policyTypes:
  - Ingress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: frontend     # Autorise uniquement les pods frontend
    ports:
    - protocol: TCP
      port: 8080            # Sur le port 8080 uniquement
```

**Ce que fait cette policy :**
- S'applique aux pods avec le label `app: backend`
- Autorise uniquement les pods avec le label `app: frontend`
- Uniquement sur le port TCP 8080
- Tout autre trafic vers backend est bloqué

**Schéma :**

```
┌──────────────┐
│  Frontend    │
│ app=frontend │
└──────┬───────┘
       │ TCP:8080
       ▼ ✓
┌──────────────┐
│   Backend    │
│ app=backend  │
└──────────────┘
       ▲
       │ ✗
┌──────┴───────┐
│  Autre Pod   │
└──────────────┘
```

---

## Exemples Intermédiaires - Sélection Avancée

### Exemple 6 : Autoriser plusieurs ports

Un pod backend qui expose plusieurs services.

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: backend-multi-ports
  namespace: default
spec:
  podSelector:
    matchLabels:
      app: backend
  policyTypes:
  - Ingress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: frontend
    ports:
    - protocol: TCP
      port: 8080            # API REST
    - protocol: TCP
      port: 9090            # Métriques Prometheus
    - protocol: TCP
      port: 8081            # Health check
```

**Ce que fait cette policy :**
- Le frontend peut accéder au backend sur 3 ports différents
- Chaque port a un usage spécifique
- Tous les autres ports restent bloqués

### Exemple 7 : Autoriser depuis plusieurs sources

Backend accessible depuis plusieurs types de clients.

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: backend-multi-sources
  namespace: default
spec:
  podSelector:
    matchLabels:
      app: backend
  policyTypes:
  - Ingress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: frontend     # Source 1: Frontend
    - podSelector:
        matchLabels:
          app: admin        # Source 2: Admin panel
    - podSelector:
        matchLabels:
          app: cron-jobs    # Source 3: Jobs planifiés
    ports:
    - protocol: TCP
      port: 8080
```

**Ce que fait cette policy :**
- Le backend accepte les connexions de 3 types de pods différents
- Tous peuvent se connecter sur le port 8080
- Le opérateur `-` dans `from` signifie "OU" (OR)

**Note importante sur la syntaxe :**

```yaml
# Ceci signifie "frontend OU admin OU cron-jobs"
- from:
  - podSelector:
      matchLabels:
        app: frontend
  - podSelector:
      matchLabels:
        app: admin

# Ceci signifie "frontend ET dans namespace production"
- from:
  - podSelector:
      matchLabels:
        app: frontend
    namespaceSelector:
      matchLabels:
        env: production
```

### Exemple 8 : Sélection par namespace

Autoriser tout le trafic depuis un namespace spécifique.

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-from-namespace
  namespace: production
spec:
  podSelector:
    matchLabels:
      app: api
  policyTypes:
  - Ingress
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          name: monitoring      # Tous les pods du namespace monitoring
    ports:
    - protocol: TCP
      port: 9090                # Port métriques
```

**Prérequis** : Le namespace monitoring doit avoir le label `name: monitoring`.

Pour ajouter ce label :

```bash
microk8s kubectl label namespace monitoring name=monitoring
```

**Ce que fait cette policy :**
- L'API dans le namespace `production` accepte les connexions
- Uniquement depuis le namespace `monitoring`
- Sur le port 9090 (métriques)

**Cas d'usage :** Permettre à Prometheus de scraper les métriques de tous les services de production.

### Exemple 9 : Combiner pod et namespace selector

Autoriser uniquement un pod spécifique d'un namespace spécifique.

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-specific-pod-from-namespace
  namespace: production
spec:
  podSelector:
    matchLabels:
      app: database
  policyTypes:
  - Ingress
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          env: production
      podSelector:
        matchLabels:
          app: backend          # ET logique
    ports:
    - protocol: TCP
      port: 5432
```

**Ce que fait cette policy :**
- La base de données accepte les connexions
- Uniquement depuis les pods avec label `app: backend`
- ET uniquement si ces pods sont dans un namespace avec label `env: production`

**Schéma :**

```
Namespace: production (env: production)
┌─────────────────────────────────────┐
│  ┌──────────┐                       │
│  │ Backend  │ ──✓──► Database       │
│  └──────────┘                       │
│  ┌──────────┐                       │
│  │ Frontend │ ──✗──► Database       │
│  └──────────┘                       │
└─────────────────────────────────────┘

Namespace: staging (env: staging)
┌─────────────────────────────────────┐
│  ┌──────────┐                       │
│  │ Backend  │ ──✗──► Database       │
│  └──────────┘        (production)   │
└─────────────────────────────────────┘
```

### Exemple 10 : Autoriser depuis une plage d'IPs (ipBlock)

Autoriser le trafic depuis des IPs externes spécifiques.

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-from-ip-range
  namespace: default
spec:
  podSelector:
    matchLabels:
      app: public-api
  policyTypes:
  - Ingress
  ingress:
  - from:
    - ipBlock:
        cidr: 203.0.113.0/24        # Réseau externe autorisé
        except:
        - 203.0.113.50/32           # Sauf cette IP spécifique
    ports:
    - protocol: TCP
      port: 443
```

**Ce que fait cette policy :**
- L'API publique accepte les connexions depuis `203.0.113.0/24`
- SAUF depuis `203.0.113.50` (IP bloquée/suspecte)
- Uniquement en HTTPS (port 443)

**Cas d'usage :**
- Autoriser un office réseau
- Autoriser un serveur de monitoring externe
- Bloquer une IP malveillante connue

**⚠️ Important** : Les `ipBlock` s'appliquent aux IPs sources **avant NAT**. Dans Kubernetes, c'est généralement l'IP du nœud, pas du pod.

---

## Exemples Avancés - Egress (Trafic Sortant)

### Exemple 11 : Autoriser DNS uniquement

Policy la plus basique pour Egress : autoriser la résolution DNS.

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-dns-only
  namespace: default
spec:
  podSelector:
    matchLabels:
      app: isolated-app
  policyTypes:
  - Egress
  egress:
  - to:
    - namespaceSelector:
        matchLabels:
          name: kube-system
      podSelector:
        matchLabels:
          k8s-app: kube-dns
    ports:
    - protocol: UDP
      port: 53
```

**Ce que fait cette policy :**
- L'application peut uniquement faire des requêtes DNS
- Vers CoreDNS dans kube-system
- Aucune autre connexion sortante n'est autorisée

**⚠️ Critique** : Sans DNS, les pods ne peuvent pas résoudre les noms de services. Toute policy Egress doit généralement inclure cette règle.

### Exemple 12 : Autoriser l'accès à la base de données

Application qui ne doit accéder qu'à sa base de données.

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: app-to-db-only
  namespace: default
spec:
  podSelector:
    matchLabels:
      app: webapp
  policyTypes:
  - Egress
  egress:
  # Règle 1 : DNS
  - to:
    - namespaceSelector:
        matchLabels:
          name: kube-system
      podSelector:
        matchLabels:
          k8s-app: kube-dns
    ports:
    - protocol: UDP
      port: 53
  # Règle 2 : Database
  - to:
    - podSelector:
        matchLabels:
          app: postgres
    ports:
    - protocol: TCP
      port: 5432
```

**Ce que fait cette policy :**
- L'application peut résoudre des DNS
- L'application peut se connecter à PostgreSQL
- Aucune autre connexion n'est autorisée (pas d'Internet, pas d'autres services)

**Cas d'usage :** Application qui ne devrait pas faire d'appels HTTP externes (APIs tierces, téléchargement de fichiers).

### Exemple 13 : Autoriser HTTPS vers Internet uniquement

Application qui doit faire des appels API externes.

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-https-external
  namespace: default
spec:
  podSelector:
    matchLabels:
      app: integration-service
  policyTypes:
  - Egress
  egress:
  # DNS
  - to:
    - namespaceSelector:
        matchLabels:
          name: kube-system
      podSelector:
        matchLabels:
          k8s-app: kube-dns
    ports:
    - protocol: UDP
      port: 53
  # HTTPS vers Internet
  - to:
    - ipBlock:
        cidr: 0.0.0.0/0           # Tout Internet
        except:                   # SAUF les réseaux privés
        - 10.0.0.0/8
        - 172.16.0.0/12
        - 192.168.0.0/16
    ports:
    - protocol: TCP
      port: 443                   # HTTPS uniquement
```

**Ce que fait cette policy :**
- Le service peut résoudre des DNS
- Le service peut faire des appels HTTPS vers Internet
- Le service NE PEUT PAS accéder aux réseaux privés (sécurité)
- Le service NE PEUT PAS utiliser HTTP (port 80 bloqué)

**Cas d'usage :**
- Service d'intégration avec APIs tierces (Stripe, SendGrid, AWS)
- Webhook callbacks
- Téléchargement de données externes

### Exemple 14 : Autoriser uniquement des domaines/IPs spécifiques

Contrôler précisément vers où une application peut se connecter.

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-specific-external-ips
  namespace: default
spec:
  podSelector:
    matchLabels:
      app: payment-service
  policyTypes:
  - Egress
  egress:
  # DNS
  - to:
    - namespaceSelector:
        matchLabels:
          name: kube-system
      podSelector:
        matchLabels:
          k8s-app: kube-dns
    ports:
    - protocol: UDP
      port: 53
  # API Stripe (exemple d'IPs)
  - to:
    - ipBlock:
        cidr: 54.187.174.169/32    # IP de l'API Stripe
    - ipBlock:
        cidr: 54.187.205.235/32    # IP de backup
    ports:
    - protocol: TCP
      port: 443
```

**Ce que fait cette policy :**
- Le service de paiement peut uniquement accéder aux IPs Stripe
- Aucune autre connexion externe n'est autorisée

**⚠️ Limitations** :
- Les IPs publiques peuvent changer (utiliser des plages larges ou des services mesh)
- DNS n'est pas filtré par domaine, seulement par IP

**Note** : Pour un vrai filtrage par domaine, utilisez un service mesh comme Istio ou un proxy Egress.

---

## Architectures Complètes - Cas Réels

### Exemple 15 : Application 3-tiers complète

Architecture classique : Frontend → Backend → Database

**Schéma de l'architecture :**

```
Internet
   │
   ▼
┌─────────────────────┐
│  Ingress Controller │
└──────────┬──────────┘
           │
           ▼
    ┌──────────┐
    │ Frontend │  ← Utilisateurs
    └─────┬────┘
          │
          ▼
    ┌──────────┐
    │ Backend  │  ← Logic métier
    └─────┬────┘
          │
          ▼
    ┌──────────┐
    │ Database │  ← Données
    └──────────┘
```

**Policy 1 : Frontend**

```yaml
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
  # Recevoir du trafic depuis l'Ingress Controller
  - from:
    - namespaceSelector:
        matchLabels:
          name: ingress
      podSelector:
        matchLabels:
          app.kubernetes.io/name: ingress-nginx
    ports:
    - protocol: TCP
      port: 80
  egress:
  # DNS
  - to:
    - namespaceSelector:
        matchLabels:
          name: kube-system
      podSelector:
        matchLabels:
          k8s-app: kube-dns
    ports:
    - protocol: UDP
      port: 53
  # Backend API
  - to:
    - podSelector:
        matchLabels:
          tier: backend
    ports:
    - protocol: TCP
      port: 8080
```

**Policy 2 : Backend**

```yaml
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
  # Recevoir uniquement depuis Frontend
  - from:
    - podSelector:
        matchLabels:
          tier: frontend
    ports:
    - protocol: TCP
      port: 8080
  egress:
  # DNS
  - to:
    - namespaceSelector:
        matchLabels:
          name: kube-system
      podSelector:
        matchLabels:
          k8s-app: kube-dns
    ports:
    - protocol: UDP
      port: 53
  # Database
  - to:
    - podSelector:
        matchLabels:
          tier: database
    ports:
    - protocol: TCP
      port: 5432
  # APIs externes (exemple : envoi emails)
  - to:
    - ipBlock:
        cidr: 0.0.0.0/0
        except:
        - 10.0.0.0/8
        - 172.16.0.0/12
        - 192.168.0.0/16
    ports:
    - protocol: TCP
      port: 443
    - protocol: TCP
      port: 587             # SMTP pour emails
```

**Policy 3 : Database**

```yaml
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
  # Recevoir uniquement depuis Backend
  - from:
    - podSelector:
        matchLabels:
          tier: backend
    ports:
    - protocol: TCP
      port: 5432
  egress:
  # DNS uniquement
  - to:
    - namespaceSelector:
        matchLabels:
          name: kube-system
      podSelector:
        matchLabels:
          k8s-app: kube-dns
    ports:
    - protocol: UDP
      port: 53
```

**Flux de trafic autorisé :**

```
Internet → Ingress → Frontend → Backend → Database
                        │           │
                        └─ DNS ─────┘
                                    │
                        Backend ────┴──► Internet (HTTPS/SMTP)
```

**Tout autre flux est bloqué :**

```
Frontend ──✗──► Database
Internet ──✗──► Backend (direct)
Database ──✗──► Internet
```

### Exemple 16 : Microservices avec service mesh pattern

Architecture microservices avec plusieurs services interdépendants.

**Schéma :**

```
┌─────────────┐     ┌─────────────┐
│   User      │────►│   Order     │
│  Service    │     │  Service    │
└──────┬──────┘     └──────┬──────┘
       │                   │
       │    ┌──────────────┴──────┐
       │    │                     │
       ▼    ▼                     ▼
┌─────────────┐          ┌─────────────┐
│  Product    │          │  Payment    │
│  Service    │          │  Service    │
└─────────────┘          └─────────────┘
       │                        │
       └────────┬───────────────┘
                ▼
         ┌─────────────┐
         │   Database  │
         └─────────────┘
```

**Policy globale : Allow list par service**

```yaml
---
# User Service : point d'entrée
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: user-service-policy
  namespace: microservices
spec:
  podSelector:
    matchLabels:
      service: user
  policyTypes:
  - Ingress
  - Egress
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          name: ingress
    ports:
    - protocol: TCP
      port: 8080
  egress:
  - to:
    - podSelector:
        matchLabels:
          k8s-app: kube-dns
    ports:
    - protocol: UDP
      port: 53
  - to:
    - podSelector:
        matchLabels:
          service: product
    ports:
    - protocol: TCP
      port: 8080
  - to:
    - podSelector:
        matchLabels:
          service: order
    ports:
    - protocol: TCP
      port: 8080

---
# Order Service
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: order-service-policy
  namespace: microservices
spec:
  podSelector:
    matchLabels:
      service: order
  policyTypes:
  - Ingress
  - Egress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          service: user
    ports:
    - protocol: TCP
      port: 8080
  egress:
  - to:
    - podSelector:
        matchLabels:
          k8s-app: kube-dns
    ports:
    - protocol: UDP
      port: 53
  - to:
    - podSelector:
        matchLabels:
          service: payment
    ports:
    - protocol: TCP
      port: 8080
  - to:
    - podSelector:
        matchLabels:
          service: product
    ports:
    - protocol: TCP
      port: 8080

---
# Product Service
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: product-service-policy
  namespace: microservices
spec:
  podSelector:
    matchLabels:
      service: product
  policyTypes:
  - Ingress
  - Egress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          service: user
    - podSelector:
        matchLabels:
          service: order
    ports:
    - protocol: TCP
      port: 8080
  egress:
  - to:
    - podSelector:
        matchLabels:
          k8s-app: kube-dns
    ports:
    - protocol: UDP
      port: 53
  - to:
    - podSelector:
        matchLabels:
          service: database
    ports:
    - protocol: TCP
      port: 5432

---
# Payment Service
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: payment-service-policy
  namespace: microservices
spec:
  podSelector:
    matchLabels:
      service: payment
  policyTypes:
  - Ingress
  - Egress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          service: order
    ports:
    - protocol: TCP
      port: 8080
  egress:
  - to:
    - podSelector:
        matchLabels:
          k8s-app: kube-dns
    ports:
    - protocol: UDP
      port: 53
  - to:
    - podSelector:
        matchLabels:
          service: database
    ports:
    - protocol: TCP
      port: 5432
  # API externe de paiement
  - to:
    - ipBlock:
        cidr: 0.0.0.0/0
        except:
        - 10.0.0.0/8
        - 172.16.0.0/12
        - 192.168.0.0/16
    ports:
    - protocol: TCP
      port: 443

---
# Database : ultra sécurisée
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: database-policy
  namespace: microservices
spec:
  podSelector:
    matchLabels:
      service: database
  policyTypes:
  - Ingress
  - Egress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          service: product
    - podSelector:
        matchLabels:
          service: payment
    ports:
    - protocol: TCP
      port: 5432
  egress:
  - to:
    - podSelector:
        matchLabels:
          k8s-app: kube-dns
    ports:
    - protocol: UDP
      port: 53
```

---

## Exemples de Monitoring et Observabilité

### Exemple 17 : Permettre à Prometheus de scraper toutes les métriques

Autoriser Prometheus à collecter les métriques de tous les services.

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-prometheus-scraping
  namespace: production
spec:
  podSelector:
    matchLabels:
      prometheus.io/scrape: "true"    # Tous les pods exposant des métriques
  policyTypes:
  - Ingress
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          name: monitoring
      podSelector:
        matchLabels:
          app: prometheus
    ports:
    - protocol: TCP
      port: 9090        # Port métriques standard
    - protocol: TCP
      port: 8080        # Port alternatif
```

**Déploiement type :**

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-app
  labels:
    app: my-app
    prometheus.io/scrape: "true"    # Active le scraping
spec:
  containers:
  - name: app
    image: my-app:latest
    ports:
    - containerPort: 9090
      name: metrics
```

### Exemple 18 : Isoler le monitoring

Le namespace monitoring peut scraper tout, mais rien ne peut accéder au monitoring.

```yaml
---
# Dans le namespace monitoring : deny all ingress depuis l'extérieur
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: monitoring-isolation
  namespace: monitoring
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  ingress:
  # Permettre la communication interne au namespace monitoring
  - from:
    - podSelector: {}

---
# Prometheus peut sortir vers tous les namespaces pour scraper
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: prometheus-egress
  namespace: monitoring
spec:
  podSelector:
    matchLabels:
      app: prometheus
  policyTypes:
  - Egress
  egress:
  # DNS
  - to:
    - namespaceSelector:
        matchLabels:
          name: kube-system
      podSelector:
        matchLabels:
          k8s-app: kube-dns
    ports:
    - protocol: UDP
      port: 53
  # Scraping vers tous les namespaces
  - to:
    - namespaceSelector: {}       # Tous les namespaces
    ports:
    - protocol: TCP
      port: 9090
    - protocol: TCP
      port: 8080
```

**Résultat :**
- Prometheus peut scraper n'importe quel pod exposant des métriques
- Aucun pod ne peut se connecter directement à Prometheus
- Accès à Grafana contrôlé séparément

---

## Exemples de Sécurité Multi-Environnements

### Exemple 19 : Isolation entre Dev, Staging, Production

Empêcher toute communication inter-environnements.

```yaml
---
# Policy dans le namespace production
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: production-isolation
  namespace: production
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  - Egress
  ingress:
  # Autoriser seulement depuis production
  - from:
    - podSelector: {}         # Pods du même namespace
    - namespaceSelector:
        matchLabels:
          env: production     # Autres namespaces production
  egress:
  # DNS
  - to:
    - namespaceSelector:
        matchLabels:
          name: kube-system
      podSelector:
        matchLabels:
          k8s-app: kube-dns
    ports:
    - protocol: UDP
      port: 53
  # Seulement vers production
  - to:
    - namespaceSelector:
        matchLabels:
          env: production
  # Internet si nécessaire
  - to:
    - ipBlock:
        cidr: 0.0.0.0/0
        except:
        - 10.0.0.0/8
        - 172.16.0.0/12
        - 192.168.0.0/16
    ports:
    - protocol: TCP
      port: 443

---
# Même chose pour staging
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: staging-isolation
  namespace: staging
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  - Egress
  ingress:
  - from:
    - podSelector: {}
    - namespaceSelector:
        matchLabels:
          env: staging
  egress:
  - to:
    - namespaceSelector:
        matchLabels:
          name: kube-system
      podSelector:
        matchLabels:
          k8s-app: kube-dns
    ports:
    - protocol: UDP
      port: 53
  - to:
    - namespaceSelector:
        matchLabels:
          env: staging
  - to:
    - ipBlock:
        cidr: 0.0.0.0/0
        except:
        - 10.0.0.0/8
        - 172.16.0.0/12
        - 192.168.0.0/16
    ports:
    - protocol: TCP
      port: 443

---
# Dev : plus permissif
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: dev-isolation
  namespace: dev
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  ingress:
  - from:
    - podSelector: {}
    - namespaceSelector:
        matchLabels:
          env: dev
  # Pas de restriction Egress en dev pour faciliter le développement
```

**Labels nécessaires sur les namespaces :**

```bash
microk8s kubectl label namespace production env=production
microk8s kubectl label namespace staging env=staging
microk8s kubectl label namespace dev env=dev
```

### Exemple 20 : Quarantaine d'un pod compromis

Isoler immédiatement un pod suspect.

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: quarantine-compromised-pod
  namespace: production
spec:
  podSelector:
    matchLabels:
      security: quarantine      # Label à appliquer manuellement
  policyTypes:
  - Ingress
  - Egress
  # Aucune règle = tout bloqué
```

**Pour mettre un pod en quarantaine :**

```bash
# Identifier le pod compromis
microk8s kubectl get pods -n production

# Ajouter le label de quarantaine
microk8s kubectl label pod suspicious-pod-xyz security=quarantine -n production
```

**Le pod devient immédiatement isolé** :
- Plus aucune connexion entrante
- Plus aucune connexion sortante
- Le pod continue de tourner mais ne peut plus communiquer

**Pour lever la quarantaine :**

```bash
microk8s kubectl label pod suspicious-pod-xyz security- -n production
```

---

## Patterns Avancés

### Exemple 21 : Default Deny avec Whitelist

Approche recommandée pour la production : tout bloquer par défaut, puis autoriser explicitement.

```yaml
---
# Étape 1 : Bloquer tout (deny-all baseline)
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
# Étape 2 : Autoriser DNS pour tous
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-dns-all
  namespace: production
spec:
  podSelector: {}
  policyTypes:
  - Egress
  egress:
  - to:
    - namespaceSelector:
        matchLabels:
          name: kube-system
      podSelector:
        matchLabels:
          k8s-app: kube-dns
    ports:
    - protocol: UDP
      port: 53

---
# Étape 3 : Autoriser explicitement chaque flux nécessaire
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-frontend-to-backend
  namespace: production
spec:
  podSelector:
    matchLabels:
      tier: backend
  policyTypes:
  - Ingress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          tier: frontend
    ports:
    - protocol: TCP
      port: 8080

# Continuer pour chaque flux...
```

**Ordre d'application important** :
1. D'abord `default-deny-all` (tout bloquer)
2. Puis `allow-dns-all` (base nécessaire)
3. Puis les policies spécifiques pour chaque service

### Exemple 22 : Multi-port avec ports nommés

Utiliser des ports nommés pour plus de clarté.

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: backend-named-ports
  namespace: default
spec:
  podSelector:
    matchLabels:
      app: backend
  policyTypes:
  - Ingress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: frontend
    ports:
    - protocol: TCP
      port: http              # Port nommé (défini dans le pod)
    - protocol: TCP
      port: metrics           # Port nommé
    - protocol: TCP
      port: health            # Port nommé
```

**Définition des ports nommés dans le pod :**

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: backend
  labels:
    app: backend
spec:
  containers:
  - name: app
    image: backend:latest
    ports:
    - containerPort: 8080
      name: http
    - containerPort: 9090
      name: metrics
    - containerPort: 8081
      name: health
```

**Avantage** : Si vous changez les numéros de ports, vous n'avez pas besoin de modifier les Network Policies.

### Exemple 23 : Label selector avec expressions

Sélection avancée avec opérateurs.

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: complex-selector
  namespace: default
spec:
  podSelector:
    matchExpressions:
    - key: app
      operator: In
      values:
      - backend
      - api
    - key: version
      operator: NotIn
      values:
      - v1-deprecated
    - key: environment
      operator: Exists          # Le label doit exister, valeur importe peu
  policyTypes:
  - Ingress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          tier: frontend
```

**Ce que fait cette policy :**
- S'applique aux pods où :
  - `app` est `backend` OU `api`
  - `version` n'est PAS `v1-deprecated`
  - Le label `environment` existe (avec n'importe quelle valeur)

**Opérateurs disponibles :**
- `In` : valeur dans la liste
- `NotIn` : valeur pas dans la liste
- `Exists` : label existe
- `DoesNotExist` : label n'existe pas

---

## Débogage et Validation

### Tester une Network Policy

#### Méthode 1 : Pods de test temporaires

```bash
# Créer un pod source
microk8s kubectl run test-source --image=nicolaka/netshoot -it --rm

# Dans le pod, tester la connectivité
curl http://backend-service:8080
wget -O- http://backend-service:8080
nc -zv backend-service 8080
```

#### Méthode 2 : Depuis un pod existant

```bash
# Tester depuis un pod existant
microk8s kubectl exec -it frontend-pod -- curl http://backend-service:8080

# Avec timeout pour voir les blocages
microk8s kubectl exec -it frontend-pod -- curl -m 5 http://backend-service:8080
```

#### Méthode 3 : Vérifier les logs Calico

```bash
# Voir les paquets bloqués
microk8s kubectl logs -n kube-system -l k8s-app=calico-node --tail=100 | grep -i denied

# Ou
microk8s kubectl logs -n kube-system -l k8s-app=calico-node --tail=100 | grep -i drop
```

### Commandes de Diagnostic

```bash
# Lister toutes les Network Policies
microk8s kubectl get networkpolicies --all-namespaces

# Détails d'une policy
microk8s kubectl describe networkpolicy [nom] -n [namespace]

# Voir quelles policies s'appliquent à un pod
microk8s kubectl get networkpolicies -n default -o yaml | grep -A 10 "podSelector"

# Vérifier les labels d'un pod
microk8s kubectl get pod [nom] --show-labels

# Vérifier les labels d'un namespace
microk8s kubectl get namespace [nom] --show-labels
```

### Erreurs Courantes

#### Erreur 1 : DNS ne fonctionne pas

**Symptôme :** Les pods ne peuvent pas résoudre les noms

**Cause :** Policy Egress sans autorisation DNS

**Solution :** Toujours inclure cette règle dans les policies Egress :

```yaml
egress:
- to:
  - namespaceSelector:
      matchLabels:
        name: kube-system
    podSelector:
      matchLabels:
        k8s-app: kube-dns
  ports:
  - protocol: UDP
    port: 53
```

#### Erreur 2 : Label selector ne matche pas

**Symptôme :** La policy ne semble pas s'appliquer

**Cause :** Les labels du pod ne correspondent pas au `podSelector`

**Solution :** Vérifier les labels :

```bash
# Voir les labels du pod
microk8s kubectl get pod my-pod --show-labels

# Voir la policy
microk8s kubectl get networkpolicy my-policy -o yaml
```

#### Erreur 3 : Namespace sans label

**Symptôme :** `namespaceSelector` ne fonctionne pas

**Cause :** Le namespace n'a pas le label requis

**Solution :**

```bash
# Vérifier les labels
microk8s kubectl get namespace my-namespace --show-labels

# Ajouter le label
microk8s kubectl label namespace my-namespace name=my-namespace
```

---

## Outils de Visualisation

### Calico Enterprise / Tigera

Calico propose des outils de visualisation des Network Policies (version payante).

### kubectl-neat

Nettoyer les YAMLs pour mieux lire :

```bash
# Installation
kubectl krew install neat

# Utilisation
microk8s kubectl get networkpolicy my-policy -o yaml | kubectl neat
```

### Network Policy Editor (Web UI)

Éditeur visuel pour créer des Network Policies :
- https://editor.cilium.io/ (Cilium)
- https://networkpolicy.io/

### kubectl-trace

Tracer les paquets réseau :

```bash
# Voir les connexions d'un pod
kubectl trace run pod-name -e 'kprobe:tcp_v4_connect { printf("%s\n", comm); }'
```

---

## Bonnes Pratiques

### 1. Commencer Simple

Ne créez pas 50 policies complexes dès le début. Approche progressive :

1. **Phase 1 :** Déployer sans policies (tout fonctionne)
2. **Phase 2 :** Ajouter une policy `deny-all` sur un namespace de test
3. **Phase 3 :** Débloquer progressivement chaque flux nécessaire
4. **Phase 4 :** Appliquer à la production

### 2. Documenter les Flux

Maintenir un diagramme des flux autorisés :

```
Frontend → Backend (TCP 8080)
Backend → Database (TCP 5432)
Backend → Internet (TCP 443) [APIs externes]
Prometheus → * (TCP 9090) [Métriques]
```

### 3. Utiliser des Labels Cohérents

Convention de nommage claire :

```yaml
labels:
  app: backend              # Application
  tier: backend             # Couche architecture
  version: v2.1             # Version
  env: production           # Environnement
  team: api-team            # Équipe responsable
```

### 4. Tester en Staging d'abord

Ne jamais appliquer une nouvelle policy directement en production :

```bash
# Staging d'abord
microk8s kubectl apply -f policy.yaml -n staging

# Valider pendant 1 semaine

# Puis production
microk8s kubectl apply -f policy.yaml -n production
```

### 5. Monitoring des Policies

Surveiller les connexions bloquées :

```bash
# Alerter si trop de paquets bloqués
# → Peut indiquer une policy trop restrictive ou une attaque
```

### 6. Versionner les Policies

Stocker dans Git avec le reste de votre infra :

```
repo/
  kubernetes/
    network-policies/
      production/
        frontend-policy.yaml
        backend-policy.yaml
        database-policy.yaml
      staging/
        ...
```

### 7. Utiliser des Outils d'Audit

```bash
# Vérifier qu'aucun namespace de production n'est sans policy
for ns in $(microk8s kubectl get ns -l env=production -o name); do
  count=$(microk8s kubectl get networkpolicies -n ${ns##*/} --no-headers | wc -l)
  if [ $count -eq 0 ]; then
    echo "WARNING: ${ns##*/} has no Network Policies"
  fi
done
```

---

## Templates Réutilisables

### Template : Allow DNS

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-dns
  namespace: NAMESPACE
spec:
  podSelector: {}
  policyTypes:
  - Egress
  egress:
  - to:
    - namespaceSelector:
        matchLabels:
          name: kube-system
      podSelector:
        matchLabels:
          k8s-app: kube-dns
    ports:
    - protocol: UDP
      port: 53
```

### Template : Allow Monitoring

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-monitoring
  namespace: NAMESPACE
spec:
  podSelector:
    matchLabels:
      monitoring: enabled
  policyTypes:
  - Ingress
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          name: monitoring
    ports:
    - protocol: TCP
      port: 9090
```

### Template : Deny All

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-all
  namespace: NAMESPACE
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  - Egress
```

### Template : Service to Service

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: SERVICE_TARGET-allow-from-SERVICE_SOURCE
  namespace: NAMESPACE
spec:
  podSelector:
    matchLabels:
      app: SERVICE_TARGET
  policyTypes:
  - Ingress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: SERVICE_SOURCE
    ports:
    - protocol: TCP
      port: PORT
```

---

## Checklist de Sécurité Network Policies

### Avant le Déploiement

- [ ] Les flux nécessaires sont documentés
- [ ] Les labels sont cohérents sur tous les pods
- [ ] Les namespaces ont les labels requis
- [ ] Les policies incluent l'autorisation DNS
- [ ] Les policies ont été testées en staging
- [ ] Un plan de rollback existe

### Après le Déploiement

- [ ] Toutes les applications fonctionnent normalement
- [ ] Les logs ne montrent pas de connexions légitimes bloquées
- [ ] Les métriques montrent un trafic normal
- [ ] La documentation est à jour
- [ ] L'équipe est formée sur les nouvelles policies

### Audit Régulier (mensuel)

- [ ] Revoir toutes les policies actives
- [ ] Vérifier que les exceptions temporaires ont été retirées
- [ ] Analyser les logs de blocage
- [ ] Mettre à jour selon les nouveaux services
- [ ] Tester les plans de reprise

---

## Résumé

### Points Clés

1. **Default Behavior** : Sans policy, tout est autorisé
2. **Dès qu'une policy s'applique** : tout devient interdit sauf explicit allow
3. **Toujours inclure DNS** dans les policies Egress
4. **Commencer simple** : deny-all puis whitelist progressive
5. **Tester systématiquement** avant production

### Tableau Récapitulatif des Exemples

| Exemple | Type | Cas d'usage |
|---------|------|-------------|
| 1-4 | Basique | Deny/Allow global |
| 5-10 | Intermédiaire | Sélection ciblée |
| 11-14 | Egress | Contrôle sortant |
| 15-16 | Architecture | Application complète |
| 17-18 | Monitoring | Observabilité |
| 19-20 | Multi-env | Isolation environnements |
| 21-23 | Avancé | Patterns production |

### Commandes Essentielles

```bash
# Lister
microk8s kubectl get networkpolicies -A

# Détails
microk8s kubectl describe networkpolicy [nom] -n [namespace]

# Appliquer
microk8s kubectl apply -f policy.yaml

# Supprimer
microk8s kubectl delete networkpolicy [nom] -n [namespace]

# Tester
microk8s kubectl exec -it [pod] -- curl [target]
```

---

## Pour Aller Plus Loin

### Documentation Officielle

- [Kubernetes Network Policies](https://kubernetes.io/docs/concepts/services-networking/network-policies/)
- [Calico Network Policy](https://docs.tigera.io/calico/latest/network-policy/)

### Outils Recommandés

- **kube-network-policy-viewer** : visualiser les policies
- **kubectl-np-viewer** : plugin kubectl
- **Cilium Hubble** : observabilité réseau

### Ressources Avancées

- **Service Mesh (Istio, Linkerd)** : contrôle plus fin avec mTLS
- **OPA (Open Policy Agent)** : policies d'admission
- **Falco** : détection d'intrusion

### Certification

**Certified Kubernetes Security Specialist (CKS)** couvre en profondeur les Network Policies.

---

## Conclusion

Les Network Policies sont un outil puissant pour sécuriser votre cluster MicroK8s. Bien qu'elles puissent sembler complexes au début, elles deviennent rapidement naturelles avec la pratique.

**Rappelez-vous :**

1. La sécurité est un processus itératif
2. Commencez simple, complexifiez progressivement
3. Documentez tout
4. Testez avant de déployer
5. Surveillez après déploiement

Avec des Network Policies bien conçues, vous transformez votre cluster d'un réseau "flat" où tout communique avec tout, en une architecture segmentée et sécurisée où chaque application n'a accès qu'à ce dont elle a vraiment besoin.

**Prochain pas :** Implémenter la première policy sur votre lab MicroK8s !

Bon courage dans la sécurisation de vos applications Kubernetes ! 🔒

⏭️ [Annexe D : Référence Rapide](/annexes/annexe-d-reference-rapide/README.md)
