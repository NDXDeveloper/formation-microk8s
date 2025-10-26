🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 25.1 Service Mesh (Istio, Linkerd)

## Introduction : Qu'est-ce qu'un Service Mesh ?

Imaginez que vous avez déployé plusieurs applications dans votre cluster Kubernetes. Ces applications communiquent entre elles : votre frontend appelle votre API, votre API interroge une base de données, et ainsi de suite. Plus votre système grandit, plus ces communications deviennent complexes.

Un **Service Mesh** (maillage de services en français) est une couche d'infrastructure dédiée qui gère toutes ces communications entre vos services. Plutôt que de gérer la sécurité, le monitoring, et la résilience dans chaque application, le Service Mesh s'en charge de manière transparente.

### Pourquoi avoir besoin d'un Service Mesh ?

Sans Service Mesh, chaque équipe de développement doit implémenter dans son code :
- La gestion des certificats TLS pour sécuriser les communications
- Les mécanismes de retry (réessai) en cas d'échec
- Le load balancing intelligent
- Le monitoring et tracing des requêtes
- Les circuit breakers pour éviter les cascades de pannes
- La gestion du trafic (canary deployments, A/B testing)

Avec un Service Mesh, toutes ces fonctionnalités sont déléguées à l'infrastructure. Vos développeurs peuvent se concentrer sur la logique métier de leurs applications.

### Le principe de fonctionnement

Un Service Mesh fonctionne selon un modèle appelé **sidecar proxy** :

1. **Un proxy est injecté automatiquement** à côté de chaque Pod de votre application
2. **Tout le trafic réseau** passe par ce proxy (on parle de proxy sidecar)
3. **Le proxy applique les règles** de sécurité, de routage, et collecte des métriques
4. **Un plan de contrôle** centralise la configuration et la gestion de tous ces proxies

```
Sans Service Mesh :
[Service A] -----> [Service B]

Avec Service Mesh :
[Service A] -> [Proxy A] -> [Proxy B] -> [Service B]
                    ↓            ↓
              [Plan de Contrôle]
```

## Les deux principaux acteurs : Istio et Linkerd

### Istio : Le couteau suisse complet

**Istio** est le Service Mesh le plus populaire et le plus riche en fonctionnalités. Développé initialement par Google, IBM et Lyft, il offre un écosystème complet.

#### Points forts d'Istio

- **Fonctionnalités exhaustives** : Istio peut tout faire en matière de gestion du trafic
- **Maturité** : Utilisé en production par de nombreuses grandes entreprises
- **Écosystème riche** : Intégrations avec de nombreux outils (Prometheus, Grafana, Jaeger, Kiali)
- **Gestion avancée du trafic** : Routage sophistiqué, fault injection, traffic mirroring
- **Sécurité robuste** : Chiffrement mTLS automatique, politiques d'autorisation granulaires

#### Points de vigilance

- **Complexité** : La courbe d'apprentissage est importante
- **Ressources** : Consomme plus de CPU et mémoire que des alternatives plus légères
- **Configuration** : Peut nécessiter beaucoup de configuration YAML

#### Composants principaux d'Istio

- **Envoy Proxy** : Le proxy sidecar utilisé (un projet CNCF très performant)
- **Istiod** : Le plan de contrôle unifié qui gère la configuration
- **Kiali** : Interface graphique pour visualiser votre Service Mesh
- **Addons** : Prometheus, Grafana, Jaeger pour l'observabilité

### Linkerd : La simplicité et la performance

**Linkerd** (prononcé "linker-dee") est le premier Service Mesh à avoir rejoint la Cloud Native Computing Foundation (CNCF). Sa philosophie : être simple, léger et ultra-rapide.

#### Points forts de Linkerd

- **Simplicité** : Installation en quelques commandes, configuration minimale
- **Performance** : Extrêmement léger et rapide, impact minimal sur la latence
- **Fiabilité** : Architecture épurée, moins de points de défaillance
- **Sécurité by default** : mTLS activé automatiquement sans configuration
- **Observabilité intégrée** : Dashboard web inclus, métriques dorées automatiques

#### Points de vigilance

- **Moins de fonctionnalités** : Volontairement plus minimaliste qu'Istio
- **Communauté** : Plus petite qu'Istio (mais en croissance)
- **Routage** : Fonctionnalités de traffic shaping plus basiques

#### Composants principaux de Linkerd

- **Linkerd2-proxy** : Un proxy Rust ultra-performant développé spécifiquement pour Linkerd
- **Control plane** : Composants de contrôle (destination, identity, proxy-injector)
- **Linkerd Dashboard** : Interface web pour la visualisation
- **CLI linkerd** : Outil en ligne de commande puissant et intuitif

## Cas d'usage concrets d'un Service Mesh

### 1. Sécurité : Chiffrement automatique mTLS

**Problème** : Vous voulez que toutes les communications entre vos services soient chiffrées, mais configurer des certificats TLS pour chaque service est fastidieux.

**Solution Service Mesh** : Le mTLS (mutual TLS) est activé automatiquement. Chaque service reçoit un certificat, et toutes les communications sont chiffrées sans modification de code.

```
Avant : [Service A] --HTTP--> [Service B] ⚠️ Non chiffré
Après :  [Service A] --mTLS--> [Service B] 🔒 Chiffré automatiquement
```

### 2. Observabilité : Visibilité complète du trafic

**Problème** : Vous ne savez pas pourquoi votre application est lente. Quelle partie de votre système pose problème ?

**Solution Service Mesh** : Chaque requête est tracée automatiquement. Vous obtenez :
- Le taux de requêtes par seconde
- La latence (p50, p95, p99)
- Le taux d'erreur
- Les dépendances entre services (qui appelle qui)

### 3. Résilience : Gestion intelligente des pannes

**Problème** : Un de vos services est instable et provoque des timeouts, ralentissant tout votre système.

**Solution Service Mesh** : Vous pouvez configurer :
- **Retry automatiques** : Réessayer les requêtes échouées
- **Timeouts** : Limiter le temps d'attente
- **Circuit breakers** : Arrêter d'envoyer du trafic vers un service défaillant

### 4. Déploiements progressifs : Canary et Blue-Green

**Problème** : Vous voulez déployer une nouvelle version de votre service, mais vous avez peur de tout casser.

**Solution Service Mesh** : Vous pouvez :
- Envoyer 5% du trafic vers la nouvelle version (canary)
- Observer les métriques
- Augmenter progressivement si tout va bien
- Revenir en arrière instantanément si nécessaire

```yaml
# Exemple conceptuel de règle de routage
apiVersion: split.smi-spec.io/v1alpha1
kind: TrafficSplit
spec:
  service: mon-service
  backends:
  - service: mon-service-v1
    weight: 95  # 95% du trafic
  - service: mon-service-v2
    weight: 5   # 5% du trafic (canary)
```

## Service Mesh et MicroK8s

### Installation sur MicroK8s

#### Option 1 : Linkerd (recommandé pour débuter)

Linkerd s'installe facilement sur MicroK8s et est idéal pour un lab personnel :

```bash
# 1. Installer le CLI Linkerd
curl -fsL https://run.linkerd.io/install | sh
export PATH=$PATH:$HOME/.linkerd2/bin

# 2. Vérifier que le cluster est prêt
linkerd check --pre

# 3. Installer Linkerd
linkerd install --crds | kubectl apply -f -
linkerd install | kubectl apply -f -

# 4. Vérifier l'installation
linkerd check

# 5. Installer le dashboard (optionnel)
linkerd viz install | kubectl apply -f -
linkerd viz dashboard
```

#### Option 2 : Istio (pour les usages avancés)

Istio est plus complexe mais offre plus de fonctionnalités :

```bash
# 1. Télécharger Istio
curl -L https://istio.io/downloadIstio | sh -
cd istio-*
export PATH=$PWD/bin:$PATH

# 2. Installation avec le profil demo
istioctl install --set profile=demo -y

# 3. Activer l'injection automatique du sidecar
kubectl label namespace default istio-injection=enabled

# 4. Installer les addons (Kiali, Prometheus, Grafana, Jaeger)
kubectl apply -f samples/addons/
```

### Considérations pour MicroK8s

**Ressources** : Un Service Mesh consomme des ressources. Pour un lab MicroK8s :
- **Linkerd** : Nécessite environ 200-300 Mo de RAM supplémentaires
- **Istio** : Nécessite environ 500-700 Mo de RAM supplémentaires

**Utilisation** : Dans un environnement de lab, un Service Mesh est surtout utile pour :
- **Apprendre** les concepts de microservices modernes
- **Expérimenter** avec le traffic management
- **Comprendre** l'observabilité avancée
- **Préparer** des architectures pour la production

## Comparaison Istio vs Linkerd

| Critère | Istio | Linkerd |
|---------|-------|---------|
| **Complexité** | Élevée | Faible |
| **Ressources (overhead)** | ~10-15% | ~1-5% |
| **Installation** | Plusieurs étapes | Très simple |
| **Fonctionnalités** | Très nombreuses | Essentielles |
| **Latence ajoutée** | ~3-5 ms | ~1-2 ms |
| **Courbe d'apprentissage** | Importante | Rapide |
| **Maturité** | Très mature | Mature |
| **Communauté** | Très grande | Moyenne |
| **Support multi-cluster** | Excellent | Bon |
| **Interface graphique** | Kiali (excellent) | Linkerd Dashboard (simple) |
| **Idéal pour** | Production complexe | Débuter, lab, prod simple |

## Concepts clés à retenir

### 1. Data Plane vs Control Plane

- **Data Plane** : Les proxies sidecars qui gèrent le trafic réel entre vos services
- **Control Plane** : Les composants qui configurent et pilotent le data plane

### 2. mTLS (Mutual TLS)

Le chiffrement bidirectionnel où le client et le serveur s'authentifient mutuellement avec des certificats. Dans un Service Mesh, c'est géré automatiquement.

### 3. Circuit Breaking

Mécanisme qui empêche l'envoi de requêtes vers un service défaillant, permettant au système de "respirer" et au service de récupérer.

### 4. Traffic Shifting

La capacité de rediriger le trafic de manière granulaire entre différentes versions d'un service, basée sur des pourcentages ou des règles.

### 5. Observability (Métriques dorées)

Les quatre métriques essentielles surveillées automatiquement :
- **Latency** : Combien de temps prend une requête
- **Traffic** : Combien de requêtes par seconde
- **Errors** : Quel pourcentage de requêtes échouent
- **Saturation** : À quel point le système est chargé

## Quand utiliser un Service Mesh ?

### ✅ Vous DEVRIEZ envisager un Service Mesh si :

- Vous avez **plus de 5-10 microservices** qui communiquent entre eux
- Vous avez besoin de **sécurité forte** (mTLS, autorisation fine)
- Vous voulez de **l'observabilité avancée** sans modifier votre code
- Vous faites des **déploiements complexes** (canary, blue-green)
- Vous avez des **exigences de conformité** (chiffrement, audit)

### ❌ Vous ne devriez PAS utiliser un Service Mesh si :

- Vous avez **1-3 applications simples**
- Vous êtes **limité en ressources** (petit cluster)
- Votre équipe **débute avec Kubernetes**
- Vous avez une **architecture monolithique**
- Vous voulez de la **simplicité avant tout**

## Alternatives plus légères

Si un Service Mesh complet semble trop lourd pour vos besoins, considérez :

### 1. Linkerd (version minimale)

Installer uniquement le core de Linkerd sans les addons de visualisation.

### 2. NGINX Service Mesh

Plus simple qu'Istio, basé sur NGINX. Bon compromis fonctionnalités/simplicité.

### 3. Consul Connect

Service Mesh de HashiCorp, bien intégré avec leur écosystème.

### 4. Traefik Mesh

Très simple, basé sur le populaire reverse proxy Traefik.

### 5. Solutions applicatives

Pour des besoins basiques, vous pouvez implémenter :
- TLS avec des certificats Cert-Manager
- Observabilité avec Prometheus + Grafana
- Tracing avec Jaeger directement dans vos apps

## Conclusion

Un Service Mesh est un outil puissant qui simplifie la gestion de microservices complexes en externalisant les préoccupations de communication, de sécurité et d'observabilité hors du code applicatif.

### Recommandations pour MicroK8s

**Pour apprendre et expérimenter** :
- Commencez avec **Linkerd** : simple, rapide à installer, documentation claire
- Installez-le sur un namespace dédié pour tester sans risque
- Déployez une application exemple (Linkerd fournit des démos excellentes)

**Pour aller plus loin** :
- Une fois à l'aise avec Linkerd, testez **Istio** pour découvrir ses fonctionnalités avancées
- Comparez les performances et la consommation de ressources
- Explorez Kiali pour visualiser votre Service Mesh Istio

**Pour la production** :
- Évaluez vos besoins réels avant d'adopter un Service Mesh
- Testez en profondeur dans un environnement de staging
- Documentez votre configuration et formez votre équipe

### Points clés à retenir

1. Un Service Mesh **n'est pas obligatoire** pour faire du Kubernetes
2. Il apporte de la **valeur surtout à partir de 5-10 microservices**
3. **Linkerd est idéal pour débuter** : simple, léger, efficace
4. **Istio est puissant pour les cas avancés** : riche mais complexe
5. L'overhead en ressources est **réel** : planifiez en conséquence
6. La sécurité (mTLS) et l'observabilité sont les **bénéfices immédiats**
7. Les fonctionnalités de **traffic management sont puissantes** pour les déploiements

### Prochaines étapes

Après avoir exploré les Service Mesh, vous pourriez vous intéresser à :
- **25.2 Serverless avec Knative** : Exécuter des fonctions événementielles sur Kubernetes
- **25.3 Operators et Custom Resources** : Étendre Kubernetes avec vos propres ressources
- **15.5 Tracing distribué avec Jaeger** : Approfondir l'observabilité avec ou sans Service Mesh

---

**Ressources complémentaires** :

- Documentation officielle Linkerd : https://linkerd.io/docs/
- Documentation officielle Istio : https://istio.io/docs/
- Service Mesh Interface (SMI) : Standard pour les Service Mesh
- CNCF Service Mesh Landscape : Vue d'ensemble de l'écosystème

⏭️ [Serverless avec Knative](/25-aller-plus-loin/02-serverless-avec-knative.md)
