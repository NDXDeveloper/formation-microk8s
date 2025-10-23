🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 5.7 Liste complète des addons disponibles

## Introduction

Vous connaissez maintenant les quatre addons essentiels (Dashboard, DNS, Registry, Storage), mais MicroK8s propose **bien plus** ! Cette section vous présente la liste complète des addons disponibles, organisés par catégorie pour faciliter votre exploration.

Considérez cette section comme un **catalogue** ou un **menu** : vous n'avez pas besoin de tout utiliser, mais il est utile de savoir ce qui existe pour pouvoir choisir les bons outils selon vos besoins.

## Comment voir la liste des addons

Pour voir tous les addons disponibles sur votre installation MicroK8s, utilisez :

```bash
microk8s status
```

Ou pour une liste plus détaillée :

```bash
microk8s status --format yaml
```

Vous verrez deux sections :
- **enabled** : Les addons actuellement activés
- **disabled** : Les addons disponibles mais non activés

### Exemple de sortie

```
microk8s is running
high-availability: no
  datastore master nodes: 127.0.0.1:19001
  datastore standby nodes: none
addons:
  enabled:
    dns                  # (core) CoreDNS
    ha-cluster           # (core) Configure high availability on the current node
  disabled:
    cert-manager         # (core) Cloud native certificate management
    dashboard            # (core) The Kubernetes dashboard
    gpu                  # (core) Automatic enablement of Nvidia CUDA
    helm                 # (core) Helm - the package manager for Kubernetes
    helm3                # (core) Helm 3 - the package manager for Kubernetes
    host-access          # (core) Allow Pods connecting to Host services smoothly
    hostpath-storage     # (core) Storage class; allocates storage from host directory
    ingress              # (core) Ingress controller for external access
    ...
```

## Organisation des addons : Core vs Community

Les addons MicroK8s sont organisés en deux dépôts principaux :

### Core (Addons officiels)

Les addons **core** sont maintenus par l'équipe MicroK8s/Canonical. Ils sont :
- Testés et validés avec chaque version de MicroK8s
- Documentés officiellement
- Supportés à long terme
- Activables directement

**Marqués dans la liste :** `(core)`

### Community (Addons communautaires)

Les addons **community** sont développés et maintenus par la communauté. Pour les utiliser, vous devez d'abord activer le dépôt communautaire :

```bash
microk8s enable community
```

Ensuite, les addons communautaires apparaissent dans votre liste et peuvent être activés.

**Marqués dans la liste :** `(community)`

## Catégories d'addons

Pour vous aider à vous y retrouver, voici les addons organisés par catégorie d'usage.

---

## 1. Addons Essentiels (Déjà vus)

Ces addons sont fondamentaux et presque toujours nécessaires.

### dns (CoreDNS)
- **Type :** Core
- **Commande :** `microk8s enable dns`
- **Utilité :** Résolution de noms DNS dans le cluster
- **Quand l'utiliser :** Toujours, pour permettre aux services de communiquer par nom
- **Dépendances :** Aucune
- **Niveau :** Débutant ⭐

**Vous l'avez déjà vu en détail dans la section 5.4.**

### dashboard
- **Type :** Core
- **Commande :** `microk8s enable dashboard`
- **Utilité :** Interface web graphique pour gérer le cluster
- **Quand l'utiliser :** Pour visualiser et gérer visuellement votre cluster
- **Dépendances :** Metrics-server (activé automatiquement)
- **Niveau :** Débutant ⭐

**Vous l'avez déjà vu en détail dans la section 5.3.**

### hostpath-storage (ou storage)
- **Type :** Core
- **Commande :** `microk8s enable storage` ou `microk8s enable hostpath-storage`
- **Utilité :** Stockage persistant local
- **Quand l'utiliser :** Pour toute application nécessitant de persister des données (bases de données, fichiers)
- **Dépendances :** Aucune
- **Niveau :** Débutant ⭐

**Vous l'avez déjà vu en détail dans la section 5.6.**

### registry
- **Type :** Core
- **Commande :** `microk8s enable registry` ou `microk8s enable registry:size=20Gi`
- **Utilité :** Registre Docker privé local
- **Quand l'utiliser :** Pour héberger vos propres images Docker
- **Dépendances :** Storage (activé automatiquement)
- **Niveau :** Débutant ⭐

**Vous l'avez déjà vu en détail dans la section 5.5.**

---

## 2. Réseau et Accès Externe

Ces addons gèrent l'exposition de vos applications vers l'extérieur du cluster.

### ingress (NGINX Ingress Controller)
- **Type :** Core
- **Commande :** `microk8s enable ingress`
- **Utilité :** Contrôleur d'ingress pour router le trafic HTTP/HTTPS vers vos services
- **Quand l'utiliser :** Pour exposer plusieurs applications web sur le même port (80/443) avec des règles de routage
- **Dépendances :** Aucune
- **Niveau :** Intermédiaire ⭐⭐

**Exemple d'usage :**
- Exposer `blog.monsite.com` → service blog
- Exposer `api.monsite.com` → service api
- Tout sur le port 80/443

**Sera détaillé dans le chapitre 10.**

### metallb (Load Balancer)
- **Type :** Core
- **Commande :** `microk8s enable metallb:10.0.1.240-10.0.1.250`
- **Utilité :** Load balancer pour environnements bare-metal (sans cloud)
- **Quand l'utiliser :** Pour obtenir des adresses IP externes pour vos services de type LoadBalancer
- **Dépendances :** Aucune
- **Configuration requise :** Plage d'adresses IP disponibles sur votre réseau
- **Niveau :** Intermédiaire ⭐⭐

**Exemple d'usage :**
Sans MetalLB, un service LoadBalancer reste en `<pending>`. Avec MetalLB, il reçoit une vraie IP externe.

**Sera détaillé dans le chapitre 9.**

### host-access
- **Type :** Core
- **Commande :** `microk8s enable host-access`
- **Utilité :** Permet aux pods d'accéder facilement aux services de la machine hôte
- **Quand l'utiliser :** Quand vos applications dans Kubernetes doivent communiquer avec des services sur votre machine hôte (base de données locale, API locale, etc.)
- **Dépendances :** Aucune
- **Niveau :** Intermédiaire ⭐⭐

**Exemple d'usage :**
Votre pod peut accéder à `http://host.docker.internal:5432` pour se connecter à PostgreSQL qui tourne directement sur votre machine (hors Kubernetes).

---

## 3. Sécurité et Certificats

Ces addons gèrent la sécurité, les certificats et les accès.

### cert-manager
- **Type :** Core
- **Commande :** `microk8s enable cert-manager`
- **Utilité :** Gestion automatique des certificats SSL/TLS
- **Quand l'utiliser :** Pour obtenir automatiquement des certificats Let's Encrypt ou gérer des certificats auto-signés
- **Dépendances :** DNS
- **Niveau :** Intermédiaire ⭐⭐

**Exemple d'usage :**
Avec cert-manager + ingress, vos sites web obtiennent automatiquement des certificats HTTPS valides et renouvelés automatiquement.

**Sera détaillé dans le chapitre 11.**

### rbac
- **Type :** Core
- **Commande :** `microk8s enable rbac`
- **Utilité :** Role-Based Access Control - contrôle d'accès basé sur les rôles
- **Quand l'utiliser :** Pour définir qui peut faire quoi dans votre cluster (permissions granulaires)
- **Dépendances :** Aucune
- **Niveau :** Avancé ⭐⭐⭐

**Exemple d'usage :**
- L'utilisateur "dev" peut seulement lire les pods dans le namespace "development"
- L'utilisateur "admin" peut tout faire
- Une application peut seulement accéder à ses propres secrets

**Sera détaillé dans le chapitre 16.**

---

## 4. Monitoring et Observabilité

Ces addons vous aident à surveiller la santé et les performances de votre cluster.

### prometheus
- **Type :** Core
- **Commande :** `microk8s enable prometheus`
- **Utilité :** Stack complète de monitoring (Prometheus + Grafana + Alertmanager)
- **Quand l'utiliser :** Pour surveiller les métriques de votre cluster et applications
- **Dépendances :** Storage (activé automatiquement)
- **Niveau :** Avancé ⭐⭐⭐

**Ce qu'il inclut :**
- Prometheus : Collecte et stocke les métriques
- Grafana : Visualisation avec dashboards
- Alertmanager : Gestion des alertes

**Sera détaillé dans les chapitres 12-14.**

### metrics-server
- **Type :** Core
- **Commande :** `microk8s enable metrics-server`
- **Utilité :** Collecte des métriques de ressources (CPU, RAM) des pods et nodes
- **Quand l'utiliser :** Pour voir l'utilisation des ressources, nécessaire pour l'autoscaling
- **Dépendances :** Aucune
- **Niveau :** Intermédiaire ⭐⭐

**Note :** Souvent activé automatiquement avec d'autres addons (comme le dashboard).

**Exemple d'usage :**
```bash
microk8s kubectl top nodes
microk8s kubectl top pods
```

### observability
- **Type :** Core
- **Commande :** `microk8s enable observability`
- **Utilité :** Stack d'observabilité complète (Grafana, Prometheus, Loki, Tempo)
- **Quand l'utiliser :** Pour une solution d'observabilité tout-en-un (logs, métriques, traces)
- **Dépendances :** Storage, DNS
- **Niveau :** Avancé ⭐⭐⭐

**Plus complet que prometheus seul :**
- Loki : Agrégation de logs
- Tempo : Tracing distribué
- Intégration complète

---

## 5. Développement et CI/CD

Ces addons facilitent le développement et les pipelines de déploiement.

### helm / helm3
- **Type :** Core
- **Commande :** `microk8s enable helm3`
- **Utilité :** Gestionnaire de packages pour Kubernetes (comme apt ou yum pour Linux)
- **Quand l'utiliser :** Pour installer facilement des applications complexes pré-packagées
- **Dépendances :** Aucune
- **Niveau :** Intermédiaire ⭐⭐

**Exemple d'usage :**
Au lieu de créer manuellement 20 fichiers YAML pour installer WordPress, vous faites :
```bash
helm install my-wordpress bitnami/wordpress
```

**Note :** Helm3 est la version moderne, préférez-la à helm.

**Sera détaillé dans le chapitre 17.**

### registry (déjà vu)
Essentiel pour le développement, permet de pousser rapidement vos images locales.

---

## 6. Haute Disponibilité et Clustering

Ces addons sont pour les configurations avancées multi-serveurs.

### ha-cluster
- **Type :** Core
- **Commande :** `microk8s enable ha-cluster`
- **Utilité :** Configure le nœud pour participer à un cluster haute disponibilité
- **Quand l'utiliser :** Quand vous avez plusieurs serveurs et voulez un cluster résilient
- **Dépendances :** Aucune
- **Niveau :** Avancé ⭐⭐⭐

**Utilisé avec :** `microk8s add-node` pour ajouter des nœuds au cluster

**Sera détaillé dans le chapitre 21.**

---

## 7. Bases de données

Ces addons installent des bases de données prêtes à l'emploi.

### rook-ceph (Community)
- **Type :** Community
- **Commande :** `microk8s enable rook-ceph`
- **Utilité :** Système de stockage distribué Ceph via l'opérateur Rook
- **Quand l'utiliser :** Pour du stockage distribué et répliqué dans un cluster multi-node
- **Dépendances :** Storage
- **Niveau :** Expert ⭐⭐⭐⭐

**Alternative avancée à hostpath-storage pour la production.**

---

## 8. Service Mesh

Ces addons ajoutent des capacités avancées de gestion du trafic inter-services.

### istio (Community)
- **Type :** Community
- **Commande :** `microk8s enable istio`
- **Utilité :** Service mesh complet (gestion du trafic, sécurité, observabilité entre microservices)
- **Quand l'utiliser :** Pour des architectures microservices complexes nécessitant un contrôle fin du trafic
- **Dépendances :** DNS, Storage
- **Niveau :** Expert ⭐⭐⭐⭐

**Fonctionnalités :**
- Mutual TLS automatique entre services
- Circuit breaking, retry, timeout
- Traffic splitting (A/B testing, canary)
- Observabilité avancée

**Très puissant mais complexe, pour utilisateurs avancés.**

### linkerd (Community)
- **Type :** Community
- **Commande :** `microk8s enable linkerd`
- **Utilité :** Service mesh léger, alternative plus simple à Istio
- **Quand l'utiliser :** Pour avoir un service mesh sans la complexité d'Istio
- **Dépendances :** DNS
- **Niveau :** Avancé ⭐⭐⭐

**Plus simple qu'Istio, bonne option pour commencer avec les service meshes.**

---

## 9. Serverless et Functions

Ces addons permettent d'exécuter du code "serverless" dans Kubernetes.

### knative (Community)
- **Type :** Community
- **Commande :** `microk8s enable knative`
- **Utilité :** Plateforme serverless sur Kubernetes
- **Quand l'utiliser :** Pour déployer des fonctions et applications qui scale-to-zero (pas de ressources utilisées quand pas de trafic)
- **Dépendances :** DNS, Storage, Ingress
- **Niveau :** Avancé ⭐⭐⭐

**Exemple d'usage :**
Déployez une API qui scale automatiquement de 0 à N instances selon le trafic, et revient à 0 quand pas utilisée.

### openfaas (Community)
- **Type :** Community
- **Commande :** `microk8s enable openfaas`
- **Utilité :** Framework Functions-as-a-Service
- **Quand l'utiliser :** Pour déployer facilement des fonctions serverless
- **Dépendances :** DNS, Ingress
- **Niveau :** Avancé ⭐⭐⭐

**Alternative à Knative, plus orientée "functions".**

---

## 10. GPU et Calcul Intensif

Ces addons permettent d'utiliser des GPUs dans vos pods.

### gpu
- **Type :** Core
- **Commande :** `microk8s enable gpu`
- **Utilité :** Support des GPU NVIDIA CUDA
- **Quand l'utiliser :** Pour du machine learning, calcul scientifique, rendu 3D, minage crypto
- **Dépendances :** Drivers NVIDIA installés sur l'hôte
- **Niveau :** Avancé ⭐⭐⭐

**Prérequis :**
- Carte graphique NVIDIA
- Drivers NVIDIA installés
- CUDA toolkit

**Cas d'usage :**
- Entraînement de modèles ML (TensorFlow, PyTorch)
- Inférence ML rapide
- Calculs parallèles massifs

---

## 11. Opérateurs et Outils Spécialisés

Ces addons ajoutent des fonctionnalités très spécifiques.

### kube-ovn (Community)
- **Type :** Community
- **Commande :** `microk8s enable kube-ovn`
- **Utilité :** Solution réseau avancée avec support SDN (Software Defined Network)
- **Quand l'utiliser :** Pour des besoins réseau très avancés (VLANs, QoS, network policies complexes)
- **Dépendances :** Aucune
- **Niveau :** Expert ⭐⭐⭐⭐

**Pour utilisateurs avec besoins réseau spécifiques.**

### portainer (Community)
- **Type :** Community
- **Commande :** `microk8s enable portainer`
- **Utilité :** Interface web alternative pour gérer Docker et Kubernetes
- **Quand l'utiliser :** Si vous préférez Portainer au Dashboard Kubernetes standard
- **Dépendances :** Storage
- **Niveau :** Intermédiaire ⭐⭐

**Alternative au Dashboard, plus orientée gestion d'infrastructure.**

### traefik (Community)
- **Type :** Community
- **Commande :** `microk8s enable traefik`
- **Utilité :** Reverse proxy et ingress controller moderne
- **Quand l'utiliser :** Alternative à NGINX Ingress, avec configuration automatique et dashboard intégré
- **Dépendances :** Aucune
- **Niveau :** Intermédiaire ⭐⭐

**Alternative populaire à ingress (NGINX), plus moderne et "cloud-native".**

### cilium (Community)
- **Type :** Community
- **Commande :** `microk8s enable cilium`
- **Utilité :** Solution réseau avancée basée sur eBPF
- **Quand l'utiliser :** Pour des performances réseau optimales et des network policies avancées
- **Dépendances :** Noyau Linux récent avec support eBPF
- **Niveau :** Expert ⭐⭐⭐⭐

**Très performant, mais complexe à configurer.**

### mayastor (Community)
- **Type :** Community
- **Commande :** `microk8s enable mayastor`
- **Utilité :** Solution de stockage haute performance
- **Quand l'utiliser :** Pour du stockage distribué très rapide (SSD/NVMe)
- **Dépendances :** Hardware spécifique
- **Niveau :** Expert ⭐⭐⭐⭐

**Pour environnements nécessitant des performances de stockage maximales.**

### minio (Community)
- **Type :** Community
- **Commande :** `microk8s enable minio`
- **Utilité :** Stockage d'objets compatible S3
- **Quand l'utiliser :** Pour héberger un service de stockage d'objets (comme AWS S3) localement
- **Dépendances :** Storage
- **Niveau :** Intermédiaire ⭐⭐

**Exemple d'usage :**
Stocker des fichiers, images, vidéos avec une API compatible S3.

### argocd (Community)
- **Type :** Community
- **Commande :** `microk8s enable argocd`
- **Utilité :** Outil GitOps pour déploiement continu
- **Quand l'utiliser :** Pour automatiser vos déploiements depuis Git (GitOps)
- **Dépendances :** Ingress
- **Niveau :** Avancé ⭐⭐⭐

**Principe GitOps :**
Votre Git est la source de vérité, ArgoCD synchronise automatiquement votre cluster avec Git.

**Sera mentionné dans le chapitre 17.**

### fluentd (Community)
- **Type :** Community
- **Commande :** `microk8s enable fluentd`
- **Utilité :** Collecteur et agrégateur de logs
- **Quand l'utiliser :** Pour centraliser tous les logs de votre cluster
- **Dépendances :** Storage
- **Niveau :** Avancé ⭐⭐⭐

**Alternative ou complément à Loki (dans observability).**

### jaeger (Community)
- **Type :** Community
- **Commande :** `microk8s enable jaeger`
- **Utilité :** Système de tracing distribué
- **Quand l'utiliser :** Pour tracer les requêtes à travers vos microservices et identifier les goulots d'étranglement
- **Dépendances :** Storage
- **Niveau :** Avancé ⭐⭐⭐

**Pour debugger les performances dans les architectures microservices.**

---

## 12. Autres Addons Utiles

### kubeflow (Community)
- **Type :** Community
- **Commande :** `microk8s enable kubeflow`
- **Utilité :** Plateforme complète pour le Machine Learning
- **Quand l'utiliser :** Pour développer, entraîner et déployer des modèles ML sur Kubernetes
- **Dépendances :** Storage, GPU (recommandé)
- **Niveau :** Expert ⭐⭐⭐⭐

**Stack complète pour le ML : notebooks, pipelines, serving, etc.**

### multus (Community)
- **Type :** Community
- **Commande :** `microk8s enable multus`
- **Utilité :** Permet d'attacher plusieurs interfaces réseau à un pod
- **Quand l'utiliser :** Pour des cas d'usage réseau très spécifiques (NFV, télécoms)
- **Dépendances :** Aucune
- **Niveau :** Expert ⭐⭐⭐⭐

**Cas d'usage spécialisés, rarement nécessaire.**

---

## Comment choisir ses addons ?

Voici un guide de décision pour vous aider à choisir.

### Pour un lab de développement basique

**Activez ces addons :**
```bash
microk8s enable dns
microk8s enable dashboard
microk8s enable storage
microk8s enable registry
```

**Vous avez tout ce qu'il faut pour développer !**

### Pour héberger des applications web

**Ajoutez :**
```bash
microk8s enable ingress
microk8s enable cert-manager
microk8s enable metallb:192.168.1.100-192.168.1.110
```

**Vous pouvez maintenant exposer vos sites web avec HTTPS !**

### Pour du monitoring

**Ajoutez :**
```bash
microk8s enable prometheus
```

Ou pour une solution complète :
```bash
microk8s enable observability
```

**Vous avez maintenant des dashboards Grafana et des alertes !**

### Pour apprendre Kubernetes en profondeur

**Activez progressivement :**
1. Les essentiels (dns, dashboard, storage, registry)
2. Le réseau (ingress, metallb)
3. La sécurité (cert-manager, rbac)
4. Le monitoring (prometheus)
5. Le CI/CD (helm, argocd)
6. Les concepts avancés (ha-cluster, service mesh)

**Prenez votre temps, un addon à la fois !**

### Pour la production (attention)

**MicroK8s est excellent pour dev/test, mais pour la production :**
- Utilisez un cluster multi-node
- Activez ha-cluster
- Utilisez du stockage distribué (pas hostpath)
- Configurez RBAC
- Mettez en place monitoring et alerting
- Considérez un service mesh si microservices complexes

**Ou considérez d'autres distributions Kubernetes pour la production à grande échelle.**

---

## Bonnes pratiques pour gérer les addons

### 1. N'activez que ce dont vous avez besoin

Chaque addon consomme des ressources (CPU, RAM, stockage). Sur une machine modeste :
- Limitez-vous aux essentiels
- Ajoutez au fur et à mesure

### 2. Documentez ce qui est activé

Gardez une trace des addons activés et pourquoi :
```bash
# Créez un fichier addons.txt
echo "dns - résolution de noms" >> cluster-addons.txt
echo "dashboard - visualisation" >> cluster-addons.txt
echo "storage - persistance des données" >> cluster-addons.txt
```

### 3. Testez dans l'ordre

Activez les addons un par un, testez que tout fonctionne, puis passez au suivant.

### 4. Surveillez les ressources

Après avoir activé un addon, vérifiez l'impact :
```bash
microk8s kubectl top nodes
microk8s kubectl top pods -A
```

### 5. Lisez la documentation

Pour les addons avancés, lisez la documentation officielle avant d'activer :
```
https://microk8s.io/docs/addon-[nom-addon]
```

### 6. Sauvegardez votre configuration

Si vous avez un setup complexe avec de nombreux addons, notez :
- Quels addons sont activés
- Leurs configurations (paramètres passés lors de l'activation)
- Les personnalisations apportées

Ainsi, vous pouvez recréer votre environnement facilement.

---

## Désactiver les addons

Si vous n'avez plus besoin d'un addon ou voulez libérer des ressources :

```bash
microk8s disable <nom-addon>
```

**Exemple :**
```bash
microk8s disable prometheus
```

**⚠️ Attention :**
- Désactiver un addon supprime les ressources associées
- Les données peuvent être perdues (selon l'addon)
- Vérifiez les dépendances (certains addons dépendent d'autres)

**Ordre de désactivation :**
Désactivez dans l'ordre inverse de l'activation si vous voulez tout nettoyer :
```bash
# Activés dans cet ordre :
microk8s enable dns
microk8s enable storage
microk8s enable registry

# Désactivez dans l'ordre inverse :
microk8s disable registry
microk8s disable storage
microk8s disable dns
```

---

## Addons communautaires : Comment y accéder

Pour utiliser les addons communautaires, activez d'abord le dépôt :

```bash
microk8s enable community
```

**Sortie attendue :**
```
Infer repository community for addon community
Cloning into '/var/snap/microk8s/common/addons/community'...
Community repository is now enabled
```

Ensuite, `microk8s status` montrera aussi les addons communautaires dans la liste.

**Exemple :**
```bash
microk8s enable community
microk8s enable argocd
```

---

## Versions et compatibilité

Les addons évoluent avec MicroK8s. Selon votre version :
- Certains addons peuvent ne pas être disponibles
- Les noms peuvent légèrement changer
- Les options de configuration peuvent évoluer

**Vérifier votre version :**
```bash
microk8s version
```

**Pour la liste à jour des addons de votre version :**
```bash
microk8s status
```

Ou consultez la documentation officielle :
```
https://microk8s.io/docs/addons
```

---

## Addons expérimentaux

Certains addons sont marqués comme "experimental" ou "beta". Cela signifie :
- Ils peuvent avoir des bugs
- L'API peut changer
- Ils peuvent être retirés dans une future version

**Pour un environnement de production, évitez les addons expérimentaux.**

Pour un lab d'apprentissage, c'est l'occasion de tester les nouvelles technologies !

---

## Créer ses propres addons

Pour les utilisateurs très avancés, il est possible de créer ses propres addons MicroK8s.

Un addon est essentiellement :
- Un script qui s'exécute lors de l'activation (`enable`)
- Un script qui s'exécute lors de la désactivation (`disable`)
- Des manifestes YAML à déployer

Si vous avez un ensemble d'outils que vous déployez souvent, créer un addon personnalisé peut vous faire gagner du temps.

**Ressource :**
```
https://microk8s.io/docs/how-to-build-an-addon
```

---

## Récapitulatif des addons essentiels

Voici un aide-mémoire des addons que vous utiliserez probablement le plus :

### Les 4 fondamentaux (déjà vus en détail)
```bash
microk8s enable dns          # Communication par noms
microk8s enable dashboard    # Interface graphique
microk8s enable storage      # Persistance des données
microk8s enable registry     # Images Docker privées
```

### Pour exposer des applications web
```bash
microk8s enable ingress      # Routage HTTP/HTTPS
microk8s enable cert-manager # Certificats SSL automatiques
microk8s enable metallb      # Load balancer externe
```

### Pour le monitoring
```bash
microk8s enable prometheus   # Métriques et dashboards
```

### Pour le développement avancé
```bash
microk8s enable helm3        # Gestionnaire de packages
```

### Pour la haute disponibilité
```bash
microk8s enable ha-cluster   # Cluster multi-node
```

---

## Tableau récapitulatif

Voici un tableau synthétique pour vous aider à choisir :

| Addon | Niveau | Usage | Priorité |
|-------|--------|-------|----------|
| **dns** | ⭐ Débutant | Communication services | ★★★★★ Essentiel |
| **dashboard** | ⭐ Débutant | Visualisation | ★★★★★ Essentiel |
| **storage** | ⭐ Débutant | Persistance données | ★★★★★ Essentiel |
| **registry** | ⭐ Débutant | Images privées | ★★★★☆ Très utile |
| **ingress** | ⭐⭐ Intermédiaire | Exposer applications web | ★★★★☆ Très utile |
| **cert-manager** | ⭐⭐ Intermédiaire | Certificats HTTPS | ★★★☆☆ Utile |
| **metallb** | ⭐⭐ Intermédiaire | Load balancer | ★★★☆☆ Utile |
| **prometheus** | ⭐⭐⭐ Avancé | Monitoring | ★★★☆☆ Utile |
| **helm3** | ⭐⭐ Intermédiaire | Gestionnaire packages | ★★★☆☆ Utile |
| **rbac** | ⭐⭐⭐ Avancé | Sécurité/Permissions | ★★☆☆☆ Situationnel |
| **ha-cluster** | ⭐⭐⭐ Avancé | Haute disponibilité | ★★☆☆☆ Situationnel |
| **gpu** | ⭐⭐⭐ Avancé | Calcul GPU | ★☆☆☆☆ Spécialisé |
| **istio** | ⭐⭐⭐⭐ Expert | Service mesh | ★☆☆☆☆ Spécialisé |

---

## Ce que vous avez appris

Dans cette section, vous avez découvert :
- La liste complète des addons MicroK8s
- La différence entre addons core et community
- Les catégories d'addons (réseau, sécurité, monitoring, etc.)
- L'utilité de chaque addon principal
- Comment choisir les addons selon vos besoins
- Comment activer le dépôt communautaire
- Les bonnes pratiques de gestion des addons

## Prochaine étape

Maintenant que vous connaissez tous les addons disponibles, la section suivante (5.8) vous montrera comment **gérer le cycle de vie des addons** : activer, désactiver, mettre à jour, vérifier l'état, et résoudre les problèmes.

Vous aurez alors une maîtrise complète du système d'addons MicroK8s !

---

**Conseil pratique :** Créez un document personnel listant les addons que vous utilisez régulièrement avec des notes sur leur utilité pour vos projets. Cela vous servira de référence rapide et vous évitera de chercher à chaque fois ce que fait chaque addon. Votre "cheat sheet" personnalisée !

⏭️ [Gestion du cycle de vie des addons (enable/disable)](/05-addons-microk8s/08-gestion-du-cycle-de-vie-des-addons.md)
