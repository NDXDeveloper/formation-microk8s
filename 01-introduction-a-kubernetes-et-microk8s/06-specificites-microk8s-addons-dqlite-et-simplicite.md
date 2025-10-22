🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 1.6 Spécificités MicroK8s : addons, Dqlite et simplicité

## Introduction

Nous avons vu l'architecture générale de Kubernetes dans la section précédente. Maintenant, découvrons ce qui rend MicroK8s unique et spécial. Trois piliers distinguent MicroK8s des autres distributions Kubernetes : son **système d'addons**, **Dqlite** (sa base de données légère), et sa **philosophie de simplicité**.

Ces trois éléments ne sont pas de simples détails techniques - ils transforment radicalement l'expérience utilisateur et font de MicroK8s un outil accessible même aux débutants.

## Le système d'addons : la magie de MicroK8s

### Qu'est-ce qu'un addon ?

Un **addon** (extension en français) est une fonctionnalité supplémentaire que vous pouvez activer ou désactiver en une seule commande. C'est comme les options sur une voiture : vous choisissez celles dont vous avez besoin, quand vous en avez besoin.

### L'analogie du restaurant modulaire

Imaginez Kubernetes comme un **restaurant** :

**Kubernetes standard :**
- Vous avez une cuisine vide
- Pour servir des pizzas, vous devez acheter un four, l'installer, le configurer, trouver les recettes
- Pour servir des sushis, vous devez acheter du matériel spécialisé, apprendre à l'utiliser, etc.
- Chaque nouveau plat nécessite des heures de préparation et de configuration

**MicroK8s avec addons :**
- Vous avez une cuisine de base fonctionnelle
- Vous dites "Je veux servir des pizzas" → Un four à pizza déjà configuré apparaît, prêt à l'emploi
- Vous dites "Je veux servir des sushis" → L'équipement à sushi apparaît, parfaitement configuré
- En une commande, tout est installé et prêt à utiliser

### Pourquoi les addons sont révolutionnaires

Dans Kubernetes standard, installer des fonctionnalités comme Prometheus ou Ingress est un parcours du combattant :

**Sans addons (Kubernetes vanilla) :**
```bash
# Pour installer Prometheus, vous devriez :
1. Télécharger les manifestes YAML (trouver la bonne version)
2. Créer des namespaces
3. Configurer les ServiceAccounts et RBAC
4. Configurer les ConfigMaps
5. Créer les Deployments
6. Configurer les Services
7. Configurer le stockage persistant
8. Déboguer les problèmes de configuration
9. Ajuster les paramètres pour votre environnement

Temps estimé : 2-4 heures pour un expert, plus pour un débutant
```

**Avec les addons MicroK8s :**
```bash
microk8s enable prometheus

# C'est tout ! En 30 secondes, Prometheus est installé et configuré
```

### Comment fonctionnent les addons

Les addons MicroK8s sont des **scripts intelligents** qui :

1. **Savent** exactement quels composants installer
2. **Configurent** automatiquement ces composants pour fonctionner ensemble
3. **Vérifient** que tout fonctionne correctement
4. **Optimisent** la configuration pour MicroK8s

C'est comme avoir un expert Kubernetes personnel qui fait tout le travail de configuration pour vous.

### Commandes essentielles des addons

**Voir les addons disponibles :**
```bash
microk8s status
```
Cela affiche la liste complète des addons avec leur état (activé/désactivé).

**Activer un addon :**
```bash
microk8s enable <nom-addon>

# Exemples :
microk8s enable dns          # Active CoreDNS pour la résolution de noms
microk8s enable dashboard    # Active le tableau de bord Kubernetes
microk8s enable ingress      # Active NGINX Ingress Controller
```

**Désactiver un addon :**
```bash
microk8s disable <nom-addon>

# Exemple :
microk8s disable dashboard
```

**Activer plusieurs addons d'un coup :**
```bash
microk8s enable dns dashboard ingress storage
```

### Catalogue des addons principaux

Explorons les addons les plus importants et ce qu'ils apportent.

#### Addons Fondamentaux (Essential)

**dns (CoreDNS)**
- **Rôle :** Résolution de noms à l'intérieur du cluster
- **Pourquoi :** Permet aux pods de se trouver par nom au lieu d'adresses IP
- **Exemple :** Au lieu de `http://10.1.2.3`, vous utilisez `http://mon-service`
- **Activation :** `microk8s enable dns`

**storage (hostpath-storage)**
- **Rôle :** Stockage persistant pour vos données
- **Pourquoi :** Les conteneurs sont éphémères, ce addon permet de garder les données
- **Exemple :** Bases de données, fichiers uploadés par les utilisateurs
- **Activation :** `microk8s enable hostpath-storage`

**dashboard**
- **Rôle :** Interface web graphique pour gérer Kubernetes
- **Pourquoi :** Alternative visuelle à kubectl, idéale pour débuter
- **Exemple :** Voir tous vos pods, déploiements, services dans un navigateur
- **Activation :** `microk8s enable dashboard`

**ingress (NGINX Ingress Controller)**
- **Rôle :** Routage HTTP/HTTPS vers vos applications
- **Pourquoi :** Exposer plusieurs applications sur le même port (80/443)
- **Exemple :** `blog.monsite.com` → Application blog, `api.monsite.com` → API
- **Activation :** `microk8s enable ingress`

#### Addons Réseau

**metallb (Load Balancer)**
- **Rôle :** Services de type LoadBalancer dans un environnement bare-metal
- **Pourquoi :** Obtenir des IPs externes pour vos services (comme dans le cloud)
- **Configuration :** Nécessite un pool d'adresses IP
- **Activation :** `microk8s enable metallb:10.64.140.43-10.64.140.49`

**cert-manager**
- **Rôle :** Gestion automatique des certificats SSL/TLS
- **Pourquoi :** HTTPS automatique avec Let's Encrypt
- **Exemple :** Certificats SSL gratuits et renouvelés automatiquement
- **Activation :** `microk8s enable cert-manager`

#### Addons Monitoring et Observabilité

**prometheus**
- **Rôle :** Collecte et stockage de métriques
- **Pourquoi :** Surveiller la santé et les performances du cluster
- **Exemple :** CPU, mémoire, requêtes par seconde
- **Activation :** `microk8s enable prometheus`

**observability (Stack complet)**
- **Rôle :** Prometheus + Grafana + Loki + Tempo
- **Pourquoi :** Observabilité complète (métriques, logs, traces)
- **Exemple :** Dashboards, alertes, analyse de logs
- **Activation :** `microk8s enable observability`

#### Addons Développement

**registry**
- **Rôle :** Registre d'images Docker privé
- **Pourquoi :** Héberger vos propres images sans Docker Hub
- **Exemple :** `localhost:32000/mon-app:v1`
- **Activation :** `microk8s enable registry`

**helm3**
- **Rôle :** Gestionnaire de packages Kubernetes
- **Pourquoi :** Installer des applications complexes facilement
- **Exemple :** `helm install wordpress bitnami/wordpress`
- **Activation :** `microk8s enable helm3`

#### Addons Avancés

**istio**
- **Rôle :** Service mesh (maillage de services)
- **Pourquoi :** Communication sécurisée et observable entre microservices
- **Complexité :** Avancé
- **Activation :** `microk8s enable istio`

**knative**
- **Rôle :** Plateforme serverless sur Kubernetes
- **Pourquoi :** Déployer des fonctions et applications serverless
- **Activation :** `microk8s enable knative`

**kubeflow**
- **Rôle :** Machine Learning sur Kubernetes
- **Pourquoi :** Entraîner et déployer des modèles ML
- **Activation :** `microk8s enable kubeflow`

### Exemple de workflow avec addons

Imaginons que vous voulez héberger une application web avec monitoring :

**Sans addons (Kubernetes standard) :**
```bash
# Plusieurs heures de travail :
1. Installer manuellement DNS
2. Configurer le stockage
3. Installer NGINX Ingress (télécharger, configurer)
4. Installer Cert-Manager (manifestes, CRDs)
5. Installer Prometheus (opérateur, configuration)
6. Installer Grafana
7. Configurer les dashboards
8. Déboguer tous les problèmes d'intégration
```

**Avec addons MicroK8s :**
```bash
# 5 minutes de travail :
microk8s enable dns storage ingress cert-manager observability

# Tout est installé, configuré et fonctionnel !
```

### Personnalisation des addons

Certains addons acceptent des paramètres lors de l'activation :

```bash
# MetalLB avec plage d'IPs personnalisée
microk8s enable metallb:192.168.1.240-192.168.1.250

# Registry avec taille de stockage
microk8s enable registry:size=20Gi

# GPU support pour NVIDIA
microk8s enable gpu
```

### Liste complète des addons

Pour voir tous les addons disponibles dans votre version de MicroK8s :

```bash
microk8s status

# Ou pour plus de détails
microk8s enable --help
```

Au moment de l'écriture, MicroK8s propose **plus de 50 addons** couvrant presque tous les besoins imaginables !

## Dqlite : la base de données simplifiée

### Le problème avec etcd

Nous avons vu qu'etcd est la base de données standard de Kubernetes. C'est un excellent choix, mais il a quelques inconvénients pour les petits déploiements :

**Complexité de la haute disponibilité avec etcd :**
- Nécessite un nombre impair de nœuds (3, 5, 7...)
- Configuration complexe pour le clustering
- Gestion des certificats entre les nœuds
- Résolution de problèmes de quorum difficile
- Consommation de ressources significative

**Pour un cluster à 3 nœuds :**
```
Configuration etcd classique :
- 3 instances etcd (une par nœud)
- Configuration du clustering etcd
- Gestion des certificats TLS entre tous les membres
- Synchronisation et élection de leader
- ~300-500 Mo RAM par instance etcd

Résultat : Complexe et gourmand en ressources
```

### Dqlite : l'alternative élégante

**Dqlite** est une base de données développée par Canonical spécifiquement pour simplifier les déploiements distribués.

#### Qu'est-ce que Dqlite ?

Dqlite = **SQLite distribué**

- **SQLite** : Base de données légère, fichier unique, ultra-rapide
- **Distribué** : Peut se répliquer sur plusieurs nœuds
- **Raft** : Utilise l'algorithme de consensus Raft pour la cohérence

#### Les avantages de Dqlite

**1. Simplicité extrême**
```bash
# Créer un cluster haute disponibilité MicroK8s :
# Sur le premier nœud
microk8s add-node

# Sur les autres nœuds
microk8s join <token-du-premier-nœud>

# C'est tout ! Dqlite gère automatiquement la réplication
```

**2. Légèreté**
- Empreinte mémoire minuscule (~50-100 Mo)
- Pas de processus séparé gourmand
- Intégré directement dans MicroK8s

**3. Haute disponibilité simplifiée**
- Pas besoin de nombre impair de nœuds (2, 3, 4... tous fonctionnent)
- Configuration automatique du clustering
- Pas de gestion manuelle de certificats
- Récupération automatique en cas de panne

**4. Performance**
- Très rapide pour les petits et moyens clusters
- Latence faible
- Parfait pour les déploiements edge

#### Comparaison etcd vs Dqlite

| Aspect | etcd | Dqlite |
|--------|------|--------|
| **Complexité setup** | Élevée | Très faible |
| **Mémoire** | 300-500 Mo/nœud | 50-100 Mo |
| **HA Setup** | Complexe (impair) | Simple (n'importe) |
| **Configuration** | Manuelle | Automatique |
| **Certificats** | Gestion manuelle | Gestion automatique |
| **Performance** | Excellente (large scale) | Excellente (small-medium) |
| **Cas d'usage** | Grands clusters (100+) | Petits-moyens (1-20) |
| **Apprentissage** | Difficile | Facile |

#### Architecture avec Dqlite

**Cluster MicroK8s 3 nœuds avec Dqlite :**

```
┌─────────────┐     ┌─────────────┐     ┌─────────────┐
│   Node 1    │─────│   Node 2    │─────│   Node 3    │
│  (Leader)   │     │ (Follower)  │     │ (Follower)  │
│             │     │             │     │             │
│  Dqlite DB  │◄───►│  Dqlite DB  │◄───►│  Dqlite DB  │
│  (Master)   │ Sync│  (Replica)  │ Sync│  (Replica)  │
└─────────────┘     └─────────────┘     └─────────────┘

Réplication automatique
Si Node 1 tombe → Node 2 ou 3 devient leader automatiquement
```

**Fonctionnement :**
1. Un nœud est élu leader (automatiquement)
2. Toutes les écritures passent par le leader
3. Les données sont répliquées sur les followers
4. Si le leader tombe, un nouveau leader est élu en secondes
5. Aucune intervention humaine nécessaire

#### Quand Dqlite peut atteindre ses limites

Dqlite est excellent pour la plupart des usages, mais a ses limites :

**Utilisez etcd si :**
- Très grand cluster (50+ nœuds)
- Trafic d'écriture extrêmement élevé
- Nécessite les performances maximales absolues
- Standard industriel requis contractuellement

**Dqlite est parfait pour :**
- Labs personnels (notre cas !)
- Petites productions (5-20 nœuds)
- Edge computing
- Environnements à ressources limitées
- Apprentissage et développement

**Note :** Pour un lab personnel, Dqlite est non seulement suffisant, mais supérieur à etcd en termes de simplicité !

#### Migration entre Dqlite et etcd

MicroK8s permet de basculer entre Dqlite et etcd si nécessaire :

```bash
# Activer etcd (remplace Dqlite)
microk8s enable etcd

# Revenir à Dqlite
microk8s disable etcd
```

Mais pour 99% des usages de lab et petites productions, Dqlite est le meilleur choix.

## La philosophie de simplicité de MicroK8s

Au-delà des addons et Dqlite, MicroK8s incarne une **philosophie de simplicité** qui imprègne toute sa conception.

### 1. Installation en une commande

**La promesse :**
```bash
sudo snap install microk8s --classic

# En une commande, vous avez :
# ✓ Kubernetes complet et fonctionnel
# ✓ Tous les composants configurés
# ✓ Cluster prêt à l'emploi
# ✓ Networking configuré
# ✓ Sécurité en place
```

**Comparaison avec Kubernetes standard :**
- Kubernetes avec kubeadm : 20-30 commandes minimum
- Kubernetes from scratch : 100+ commandes
- MicroK8s : 1 commande

### 2. Zéro configuration initiale

MicroK8s fonctionne **immédiatement après l'installation** :

```bash
# Installer
sudo snap install microk8s --classic

# Attendre 30 secondes

# Utiliser
microk8s kubectl get nodes
# NAME      STATUS   READY
# node1     Ready    <30s
```

Pas de :
- Configuration de réseau à faire
- Certificats à générer
- Fichiers de configuration à éditer
- Étapes post-installation compliquées

### 3. Commandes unifiées

Toutes les opérations MicroK8s utilisent le préfixe `microk8s` :

```bash
microk8s status      # État du cluster
microk8s enable      # Activer des fonctionnalités
microk8s disable     # Désactiver des fonctionnalités
microk8s kubectl     # Utiliser kubectl
microk8s add-node    # Ajouter un nœud
microk8s inspect     # Diagnostiquer des problèmes
```

**Cohérence** : Vous savez toujours quelle commande utiliser.

### 4. Diagnostic intégré

L'outil `microk8s inspect` est un **diagnostic automatique** :

```bash
microk8s inspect

# Génère un rapport complet :
# - État de tous les composants
# - Logs des services
# - Configuration réseau
# - Utilisation des ressources
# - Problèmes potentiels détectés
# - Suggestions de résolution
```

C'est comme avoir un mécanicien expert qui examine votre voiture et vous dit exactement ce qui ne va pas.

### 5. Gestion automatique

MicroK8s s'occupe de nombreuses tâches automatiquement :

**Mises à jour :**
```bash
# Snap gère les mises à jour automatiquement
# Ou manuellement :
sudo snap refresh microk8s
```

**Redémarrages :**
- Si un composant crash, il redémarre automatiquement
- Survit aux redémarrages de la machine
- Démarre automatiquement au boot

**Nettoyage :**
- Gestion automatique des images inutilisées
- Rotation des logs
- Maintenance du système

### 6. Isolation via Snap

L'utilisation de Snap apporte des avantages :

**Confinement :**
- MicroK8s est isolé du reste du système
- Pas de pollution de votre système avec des dépendances
- Désinstallation propre garantie

**Rollback :**
```bash
# Si une mise à jour pose problème
sudo snap revert microk8s
# Revient instantanément à la version précédente
```

**Canaux :**
```bash
# Choisir la version de Kubernetes
sudo snap install microk8s --classic --channel=1.28/stable
sudo snap install microk8s --classic --channel=latest/edge
```

### 7. Multi-plateforme simplifié

MicroK8s fonctionne partout avec la même expérience :

**Linux :**
```bash
sudo snap install microk8s --classic
```

**Windows (WSL2) :**
```bash
# Dans WSL2 Ubuntu
sudo snap install microk8s --classic
```

**macOS (via Multipass) :**
```bash
# Multipass crée une VM Linux automatiquement
multipass launch --name microk8s-vm --mem 4G --disk 40G
multipass exec microk8s-vm -- sudo snap install microk8s --classic
```

Même commandes, même comportement, partout.

### 8. Documentation accessible

La documentation MicroK8s est conçue pour être accessible :

- Tutoriels pas-à-pas pour débutants
- Cas d'usage pratiques
- Exemples fonctionnels
- Troubleshooting clair
- Communauté active

### 9. Pas de surprises

MicroK8s suit le **principe de moindre surprise** :

**Ce que vous attendez :**
```bash
microk8s enable ingress
# → Ingress est installé et fonctionne
```

**Ce qui se passe réellement :**
```bash
microk8s enable ingress
# → Ingress est installé, configuré, et fonctionne
# Pas d'étapes cachées, pas de configuration manuelle, ça marche
```

### 10. Production-ready par défaut

Contrairement à d'autres outils "simples", MicroK8s ne fait pas de compromis sur la qualité :

**Ce qui est inclus :**
- Sécurité (RBAC, Network Policies)
- Isolation des ressources
- Haute disponibilité (avec Dqlite)
- Monitoring et logging
- Conformité Kubernetes

**Ce qui n'est pas sacrifié :**
- Performance
- Stabilité
- Standards
- Sécurité

## Comparaison de l'expérience utilisateur

### Installation de Prometheus : un cas concret

**Kubernetes standard (sans outils additionnels) :**
```bash
# 1. Télécharger les manifestes
wget https://github.com/prometheus-operator/...

# 2. Créer le namespace
kubectl create namespace monitoring

# 3. Appliquer les CRDs
kubectl apply -f prometheus-operator-crds.yaml

# 4. Créer les ServiceAccounts
kubectl apply -f service-accounts.yaml

# 5. Configurer RBAC
kubectl apply -f cluster-roles.yaml
kubectl apply -f cluster-role-bindings.yaml

# 6. Créer les ConfigMaps
kubectl apply -f prometheus-config.yaml
kubectl apply -f alertmanager-config.yaml

# 7. Déployer l'opérateur
kubectl apply -f prometheus-operator.yaml

# 8. Créer l'instance Prometheus
kubectl apply -f prometheus-instance.yaml

# 9. Configurer le stockage
kubectl apply -f persistent-volume-claim.yaml

# 10. Exposer les services
kubectl apply -f prometheus-service.yaml

# 11. Déboguer les problèmes...
kubectl logs -n monitoring prometheus-operator-xxx
kubectl describe pod -n monitoring prometheus-xxx

# Temps total : 2-4 heures pour un expert
# Pour un débutant : frustration garantie
```

**MicroK8s avec addon :**
```bash
microk8s enable prometheus

# Temps total : 30 secondes
# Niveau requis : débutant total
# Résultat : Prometheus fonctionnel et accessible
```

**La différence :**
- 10 étapes vs 1 commande
- 2-4 heures vs 30 secondes
- Expert requis vs accessible à tous
- Risque d'erreur élevé vs zéro risque

C'est toute la philosophie MicroK8s en action !

## Les trois piliers en synergie

Ces trois spécificités ne fonctionnent pas isolément - elles se renforcent mutuellement :

### Addons + Simplicité
Les addons rendent simple ce qui est complexe. Un débutant peut installer un stack complet d'observabilité en une commande.

### Dqlite + Simplicité
La haute disponibilité devient accessible. Plus besoin d'être un expert réseau pour avoir un cluster robuste.

### Addons + Dqlite
Les addons peuvent tirer parti de Dqlite pour leur propre haute disponibilité, simplifiant encore plus les choses.

**Résultat :** Une expérience utilisateur fluide du début à la fin.

## Limitations honnêtes

Pour être complet, mentionnons les quelques limitations :

### Dépendance à Snap

**Avantage :** Installation simple, mises à jour automatiques
**Inconvénient :** Snap n'est pas disponible sur toutes les distributions Linux

**Solutions :**
- Utiliser une distribution qui supporte Snap (Ubuntu, Debian, Fedora, etc.)
- Installer snapd sur votre distribution
- Utiliser MicroK8s dans une VM si vraiment nécessaire

### Confinement Snap

Le confinement Snap peut parfois compliquer l'accès à certains chemins du système.

**Solutions :**
- MicroK8s utilise `--classic` qui relâche le confinement
- La plupart des usages ne sont pas affectés
- Documentation claire pour les cas spéciaux

### Addons vs Flexibilité totale

Les addons sont préconfigurés, ce qui est génial pour commencer, mais peut limiter la personnalisation avancée.

**Solutions :**
- Vous pouvez toujours installer manuellement à la place d'un addon
- Les addons sont un point de départ, pas une prison
- Pour les usages avancés, installation standard disponible

## Conclusion

Les spécificités de MicroK8s - addons, Dqlite, et philosophie de simplicité - ne sont pas de simples fonctionnalités. Elles représentent une **vision différente** de ce que Kubernetes devrait être :

**Kubernetes traditionnel :**
- "Voici un outil puissant, apprenez à vous en servir"
- Courbe d'apprentissage abrupte
- Barrière à l'entrée élevée

**MicroK8s :**
- "Kubernetes devrait être accessible à tous"
- Courbe d'apprentissage douce
- Barrière à l'entrée minimale

**Sans sacrifier :**
- La puissance
- La conformité
- La qualité
- Les standards

MicroK8s prouve qu'on peut avoir **le beurre et l'argent du beurre** : simplicité ET puissance. C'est ce qui en fait l'outil parfait pour un lab personnel et pour l'apprentissage.

Dans la section suivante, nous verrons les cas d'usage pratiques et les objectifs de cette formation, pour comprendre concrètement ce que vous pourrez accomplir avec votre lab MicroK8s !

---

**Points clés à retenir :**
- **Addons** : 50+ fonctionnalités activables en une commande
- **Dqlite** : Alternative légère à etcd, HA simplifiée
- **Simplicité** : Installation en une commande, configuration zéro
- Addons transforment des heures de travail en une commande
- Dqlite rend la haute disponibilité accessible aux débutants
- Philosophie : Kubernetes devrait être simple ET puissant
- microk8s inspect : diagnostic automatique intégré
- Snap : mises à jour automatiques, isolation, rollback facile
- Production-ready dès l'installation
- Pas de compromis sur la qualité malgré la simplicité

⏭️ [Cas d'usage et objectifs de la formation](/01-introduction-a-kubernetes-et-microk8s/07-cas-dusage-et-objectifs-de-la-formation.md)
