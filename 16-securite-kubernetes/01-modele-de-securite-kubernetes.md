🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 16.1 Modèle de Sécurité Kubernetes

## Introduction

La sécurité dans Kubernetes est un sujet vaste et crucial. Avant de plonger dans les détails techniques des chapitres suivants, il est essentiel de comprendre la philosophie générale et l'architecture de sécurité de Kubernetes. Ce chapitre vous donnera une vision d'ensemble du modèle de sécurité pour mieux appréhender les concepts avancés qui suivront.

## Pourquoi la Sécurité est Cruciale dans Kubernetes ?

Kubernetes orchestre potentiellement des dizaines, voire des centaines de conteneurs qui exécutent vos applications. Un cluster mal sécurisé peut entraîner :

- **Accès non autorisé** aux données sensibles
- **Compromission** de l'infrastructure
- **Interruption de service** (attaques DoS)
- **Fuite de données** confidentielles
- **Escalade de privilèges** permettant à un attaquant de prendre le contrôle du cluster

Même dans un environnement de laboratoire personnel comme avec MicroK8s, adopter de bonnes pratiques de sécurité dès le départ vous prépare à gérer des environnements de production.

## Les Principes Fondamentaux de la Sécurité Kubernetes

### 1. Défense en Profondeur (Defense in Depth)

Kubernetes n'a pas un seul point de sécurité, mais plusieurs couches superposées :

```
┌─────────────────────────────────────┐
│   Infrastructure & Réseau           │
├─────────────────────────────────────┤
│   Cluster Kubernetes                │
├─────────────────────────────────────┤
│   Namespaces & RBAC                 │
├─────────────────────────────────────┤
│   Pods & Containers                 │
├─────────────────────────────────────┤
│   Applications                      │
└─────────────────────────────────────┘
```

Chaque couche ajoute une protection supplémentaire. Si une couche est compromise, les autres peuvent encore protéger le système.

### 2. Principe du Moindre Privilège (Least Privilege)

Donnez uniquement les permissions minimales nécessaires à chaque composant :

- Les pods ne doivent avoir que les permissions strictement nécessaires
- Les utilisateurs ne doivent accéder qu'aux ressources dont ils ont besoin
- Les ServiceAccounts doivent être limités au strict minimum

**Analogie** : C'est comme donner une clé de bureau spécifique à un employé, plutôt que le trousseau complet de l'immeuble.

### 3. Isolation et Séparation

Kubernetes permet d'isoler les charges de travail :

- **Namespaces** : séparer logiquement les environnements (dev, test, prod)
- **Network Policies** : contrôler qui peut communiquer avec qui
- **Pod Security** : définir ce qu'un pod peut ou ne peut pas faire

### 4. Sécurité par Défaut (Secure by Default)

Les configurations par défaut devraient être sécurisées. Malheureusement, certaines installations Kubernetes peuvent être permissives par défaut. Il est donc important de :

- Réviser les configurations
- Activer les fonctionnalités de sécurité explicitement
- Durcir (hardening) le cluster

## Les Composants du Modèle de Sécurité

### Architecture de Sécurité

```
┌──────────────────────────────────────────────────────────┐
│                     API Server                           │
│  (Point d'entrée unique - Contrôle d'accès centralisé)   │
└────────────┬─────────────────────────────────────────────┘
             │
    ┌────────┴─────────┐
    │  Authentication  │  Qui êtes-vous ?
    └────────┬─────────┘
             │
    ┌────────┴─────────┐
    │  Authorization   │  Que pouvez-vous faire ?
    └────────┬─────────┘
             │
    ┌────────┴─────────┐
    │  Admission       │  Votre demande est-elle valide ?
    │  Control         │
    └──────────────────┘
```

### 1. Authentication (Authentification)

**Question : "Qui êtes-vous ?"**

L'authentification vérifie l'identité de l'utilisateur ou du service qui tente d'accéder au cluster.

**Méthodes d'authentification dans Kubernetes :**

- **Certificats X.509** : certificats client pour les utilisateurs
- **Tokens** : jetons bearer (comme les ServiceAccount tokens)
- **Providers externes** : OIDC, LDAP, Active Directory
- **Bootstrap tokens** : pour l'ajout de nouveaux nœuds

**Dans MicroK8s :**
MicroK8s génère automatiquement des certificats pour le cluster. Lorsque vous utilisez `microk8s kubectl`, vous êtes automatiquement authentifié avec les permissions d'administrateur.

### 2. Authorization (Autorisation)

**Question : "Que pouvez-vous faire ?"**

Une fois votre identité vérifiée, Kubernetes détermine ce que vous êtes autorisé à faire.

**Modes d'autorisation :**

- **RBAC (Role-Based Access Control)** : le plus utilisé, basé sur des rôles
- **ABAC (Attribute-Based Access Control)** : basé sur des attributs (peu utilisé)
- **Node** : autorisation spécifique pour les kubelets
- **Webhook** : délégation à un service externe

**RBAC est le standard de facto** et sera couvert en détail dans la section 16.2.

**Exemple conceptuel :**
```
Utilisateur "alice"
  → Rôle "developer"
    → Permissions : lire les pods, créer des deployments
      → Dans namespace "dev" uniquement
```

### 3. Admission Control (Contrôle d'admission)

**Question : "Votre demande est-elle valide et conforme aux politiques ?"**

Même si vous êtes authentifié et autorisé, vos requêtes passent par des contrôleurs d'admission qui peuvent :

- **Valider** : refuser une requête qui ne respecte pas les règles
- **Muter** : modifier la requête pour la rendre conforme

**Admission Controllers importants :**

- **PodSecurityAdmission** : applique les standards de sécurité des pods
- **ResourceQuota** : respecte les quotas de ressources
- **LimitRanger** : applique les limites de ressources
- **NamespaceLifecycle** : empêche de créer des ressources dans des namespaces en cours de suppression
- **ServiceAccount** : injecte automatiquement un ServiceAccount dans les pods

**Exemple pratique :**
Un admission controller peut automatiquement ajouter un label de sécurité à tous les pods créés, ou refuser un pod qui demande à s'exécuter en tant que root.

## Les Domaines de Sécurité

### 1. Sécurité du Cluster (Infrastructure)

**Composants concernés :**
- API Server
- etcd (base de données du cluster)
- Controller Manager
- Scheduler
- Kubelets (agents sur les nœuds)

**Bonnes pratiques :**
- Sécuriser l'accès à l'API Server (TLS, authentification forte)
- Chiffrer les données au repos dans etcd
- Restreindre l'accès réseau aux composants
- Maintenir les composants à jour

**Dans MicroK8s :**
MicroK8s configure automatiquement le TLS et utilise Dqlite (alternative à etcd) avec une sécurité intégrée.

### 2. Sécurité des Workloads (Pods et Conteneurs)

**Concepts clés :**

- **Security Context** : définit les privilèges d'un pod ou conteneur
- **Pod Security Standards** : niveaux de sécurité (Privileged, Baseline, Restricted)
- **Container Images** : utiliser des images de confiance et scannées
- **Runtime Security** : détecter les comportements anormaux

**Questions à se poser :**
- Le conteneur a-t-il besoin de s'exécuter en root ?
- Doit-il avoir accès au système de fichiers de l'hôte ?
- Quelles capabilities Linux sont nécessaires ?

### 3. Sécurité du Réseau

**Principes :**
- Par défaut, tous les pods peuvent communiquer entre eux
- Les **Network Policies** permettent de restreindre les communications
- Le chiffrement des communications (mTLS) avec un service mesh

**Zones de réseau :**
```
┌─────────────────────────────────────────┐
│  Internet (externe)                     │
└──────────────┬──────────────────────────┘
               │
    ┌──────────▼────────────┐
    │  Ingress/LoadBalancer │
    └──────────┬────────────┘
               │
┌──────────────▼──────────────────────────┐
│  Services (réseau interne Kubernetes)   │
└──────────────┬──────────────────────────┘
               │
    ┌──────────▼──────────┐
    │  Pods               │
    └─────────────────────┘
```

### 4. Sécurité des Données

**Préoccupations :**
- **Secrets** : stockage sécurisé des informations sensibles (mots de passe, clés API)
- **ConfigMaps** : ne jamais y stocker de données sensibles
- **Volumes** : chiffrement des données persistantes
- **Logs** : ne pas logger d'informations sensibles

**Kubernetes Secrets :**
- Stockés dans etcd (idéalement chiffrés au repos)
- Montés dans les pods comme volumes ou variables d'environnement
- Ne sont PAS chiffrés par défaut dans etcd (à configurer)

### 5. Sécurité des Accès (RBAC)

Contrôle qui peut faire quoi dans le cluster :

- **Users** : personnes physiques
- **ServiceAccounts** : identités pour les pods et applications
- **Roles** : ensemble de permissions dans un namespace
- **ClusterRoles** : ensemble de permissions au niveau du cluster
- **RoleBindings** : lie un rôle à un utilisateur/ServiceAccount
- **ClusterRoleBindings** : lie un ClusterRole à un utilisateur/ServiceAccount

## Le Modèle de Confiance (Trust Model)

### Frontières de Confiance

Kubernetes définit plusieurs frontières de confiance :

```
┌─────────────────────────────────────────┐
│  Cluster (Frontière de confiance 1)     │
│                                         │
│  ┌────────────────────────────────────┐ │
│  │ Namespace (Frontière 2)            │ │
│  │                                    │ │
│  │  ┌──────────────────────────────┐  │ │
│  │  │ Pod (Frontière 3)            │  │ │
│  │  │                              │  │ │
│  │  │  ┌───────────────────────┐   │  │ │
│  │  │  │ Container (Frontière 4│   │  │ │
│  │  │  └───────────────────────┘   │  │ │
│  │  └──────────────────────────────┘  │ │
│  └────────────────────────────────────┘ │
└─────────────────────────────────────────┘
```

### Principes de Confiance

1. **Ne jamais faire confiance au réseau** : supposer que le réseau peut être compromis
2. **Vérifier toujours** : authentifier et autoriser chaque requête
3. **Isoler les charges sensibles** : utiliser des namespaces et network policies
4. **Audit** : logger toutes les actions importantes

## Menaces Courantes et Mitigations

### 1. Accès Non Autorisé à l'API Server

**Menace :** Un attaquant accède à l'API Server et peut contrôler le cluster.

**Mitigation :**
- Authentification forte
- RBAC strict
- Limiter l'exposition réseau de l'API
- Utiliser des certificats

### 2. Compromission d'un Conteneur

**Menace :** Un attaquant exploite une vulnérabilité dans une application et compromet un conteneur.

**Mitigation :**
- Exécuter les conteneurs sans privilèges
- Scanner les images pour les vulnérabilités
- Appliquer des Pod Security Standards
- Limiter les capabilities

### 3. Escalade de Privilèges

**Menace :** Passer d'un accès limité à des privilèges administrateur.

**Mitigation :**
- RBAC bien configuré
- Pod Security Standards
- Pas d'exécution en root
- ServiceAccounts avec permissions minimales

### 4. Fuite de Données Sensibles

**Menace :** Des secrets ou données confidentielles sont exposés.

**Mitigation :**
- Chiffrer les Secrets dans etcd
- Ne pas committer de secrets dans Git
- Utiliser des gestionnaires de secrets externes (Vault)
- Limiter l'accès aux Secrets via RBAC

### 5. Attaque par Déni de Service (DoS)

**Menace :** Consommation excessive de ressources rendant le cluster inutilisable.

**Mitigation :**
- Resource Quotas par namespace
- Limits sur les pods
- Network Policies pour limiter le trafic
- Rate limiting sur l'API Server

## Niveaux de Sécurité

Kubernetes définit des niveaux de sécurité pour les pods (Pod Security Standards) :

### 1. Privileged (Privilégié)

- **Description** : aucune restriction
- **Usage** : tests, développement local
- **Risques** : accès complet au système hôte

### 2. Baseline (Base)

- **Description** : restrictions minimales mais empêche les escalades connues
- **Usage** : applications standard sans besoins spéciaux
- **Restrictions** : pas de conteneurs privilégiés, pas d'accès hostPath, etc.

### 3. Restricted (Restreint)

- **Description** : sécurité maximale, bonnes pratiques strictes
- **Usage** : environnements de production, applications sensibles
- **Restrictions** : pas de root, capabilities minimales, pas d'accès host, etc.

**Recommandation :** Commencez par "Baseline" et passez à "Restricted" pour la production.

## Sécurité dans MicroK8s

### Caractéristiques de Sécurité

MicroK8s intègre plusieurs fonctionnalités de sécurité :

1. **TLS automatique** : communication chiffrée entre composants
2. **Isolation par défaut** : les snapd confinement
3. **RBAC activé** : contrôle d'accès par défaut
4. **Dqlite** : alternative sécurisée à etcd

### Bonnes Pratiques pour Votre Lab

Même dans un environnement de laboratoire :

1. **Activez RBAC** : créez des utilisateurs avec permissions limitées
2. **Utilisez des Namespaces** : séparez vos projets
3. **Configurez Network Policies** : pratiquez l'isolation réseau
4. **Scannez vos images** : habituez-vous à vérifier les vulnérabilités
5. **Chiffrez les Secrets** : adoptez les bonnes pratiques

## Évolution du Modèle de Sécurité

La sécurité Kubernetes évolue constamment :

- **Pod Security Policy (PSP)** : déprécié depuis Kubernetes 1.21, supprimé en 1.25
- **Pod Security Admission (PSA)** : remplace PSP, plus simple
- **KMS (Key Management Service)** : chiffrement des secrets au repos
- **Service Mesh** : sécurité réseau avancée (mTLS)

## Checklist de Sécurité (Aperçu)

Voici un aperçu des points de sécurité que nous explorerons dans les chapitres suivants :

- [ ] RBAC configuré avec principe du moindre privilège
- [ ] Pod Security Standards appliqués
- [ ] Network Policies définies
- [ ] Secrets chiffrés au repos
- [ ] Images scannées pour vulnérabilités
- [ ] Audit logging activé
- [ ] TLS/HTTPS partout
- [ ] Resource quotas et limits
- [ ] ServiceAccounts avec permissions minimales
- [ ] Mises à jour régulières

## Conclusion

Le modèle de sécurité Kubernetes repose sur plusieurs piliers :

1. **Authentification** : vérifier l'identité
2. **Autorisation** : contrôler les permissions (RBAC)
3. **Admission Control** : valider et modifier les requêtes
4. **Isolation** : namespaces, network policies, security contexts
5. **Chiffrement** : TLS, secrets, données au repos

La sécurité n'est pas un état mais un processus continu. Chaque couche de sécurité ajoute une protection supplémentaire selon le principe de défense en profondeur.

Dans les chapitres suivants, nous approfondirons chacun de ces aspects avec des configurations concrètes et des exemples pratiques adaptés à MicroK8s.

---

**Points Clés à Retenir :**

- La sécurité Kubernetes est multi-couches (défense en profondeur)
- Trois étapes clés : Authentication → Authorization → Admission Control
- Appliquez toujours le principe du moindre privilège
- Isolez vos workloads (namespaces, network policies)
- MicroK8s intègre des fonctionnalités de sécurité par défaut
- La sécurité est un processus, pas une destination

**Prochaines Étapes :**

Dans les sections suivantes, nous allons explorer en détail :
- **16.2** : RBAC (contrôle d'accès basé sur les rôles)
- **16.3** : ServiceAccounts
- **16.4** : Network Policies
- **16.5** : Pod Security Standards

⏭️ [RBAC (contrôle d'accès basé sur les rôles)](/16-securite-kubernetes/02-rbac-controle-dacces-base-sur-les-roles.md)
