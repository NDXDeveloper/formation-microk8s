🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 16.2 RBAC (Contrôle d'Accès Basé sur les Rôles)

## Introduction

RBAC (Role-Based Access Control) est le mécanisme standard de contrôle d'accès dans Kubernetes. C'est l'une des fonctionnalités de sécurité les plus importantes et les plus utilisées. RBAC vous permet de définir précisément qui peut faire quoi dans votre cluster Kubernetes.

**Analogie simple :** Imaginez une entreprise où chaque employé a un badge avec des permissions spécifiques. Un développeur peut accéder aux salles de développement mais pas à la salle des serveurs. Un administrateur système peut accéder partout. RBAC fonctionne de manière similaire dans Kubernetes.

## Pourquoi RBAC est Important ?

### Scénarios sans RBAC

Sans RBAC, vous auriez deux options extrêmes :

1. **Tout ou rien** : soit vous êtes administrateur avec tous les droits, soit vous n'avez aucun accès
2. **Risques de sécurité** : impossible de limiter les permissions selon les besoins

### Avec RBAC

Vous pouvez :

- Donner à un développeur l'accès uniquement à son namespace
- Permettre à un outil de monitoring de lire les métriques sans pouvoir modifier quoi que ce soit
- Créer des comptes avec des permissions très spécifiques
- Respecter le principe du moindre privilège

## Concepts de Base de RBAC

RBAC repose sur quatre composants principaux :

```
┌─────────────┐       ┌──────────────┐
│   Sujet     │       │  Permission  │
│  (Subject)  │       │  (Role)      │
│             │       │              │
│ - User      │       │ - Role       │
│ - Group     │       │ - ClusterRole│
│ - SA        │       │              │
└──────┬──────┘       └──────┬───────┘
       │                     │
       │    ┌────────────────┘
       │    │
       └────▼────────┐
       │  Liaison    │
       │  (Binding)  │
       │             │
       │ - RoleB.    │
       │ - ClusterRB.│
       └─────────────┘
```

### Vue d'Ensemble

**RBAC = Qui + Quoi + Où**

- **Qui** : Users, Groups, ServiceAccounts (les sujets)
- **Quoi** : Roles ou ClusterRoles (les permissions)
- **Où** : Namespace ou Cluster entier (la portée)
- **Liaison** : RoleBinding ou ClusterRoleBinding (connecte le qui au quoi)

## Les Quatre Objets RBAC

### 1. Role

Un **Role** définit un ensemble de permissions **dans un namespace spécifique**.

**Structure conceptuelle :**
```yaml
Role "developer"
  Namespace: "dev"
  Permissions:
    - Lire les pods
    - Créer des deployments
    - Lire les services
```

**Exemple concret :**
```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: dev
  name: developer
rules:
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["get", "list", "watch"]
- apiGroups: ["apps"]
  resources: ["deployments"]
  verbs: ["get", "list", "create", "update"]
```

**Caractéristiques :**
- Limité à un namespace
- Définit ce qui peut être fait
- N'attribue pas de permissions (c'est le RoleBinding qui le fait)

### 2. ClusterRole

Un **ClusterRole** définit un ensemble de permissions **au niveau du cluster entier**.

**Différences avec Role :**
- S'applique à tout le cluster
- Peut contrôler des ressources cluster-wide (nodes, namespaces, etc.)
- Peut être réutilisé dans plusieurs namespaces

**Exemple :**
```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: cluster-reader
rules:
- apiGroups: [""]
  resources: ["nodes", "namespaces"]
  verbs: ["get", "list"]
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["get", "list", "watch"]
```

**Usages typiques :**
- Accès aux ressources au niveau cluster (nodes, persistent volumes)
- Rôles génériques réutilisables
- Permissions pour des outils de monitoring globaux

### 3. RoleBinding

Un **RoleBinding** attribue les permissions d'un Role à un sujet **dans un namespace**.

**Structure conceptuelle :**
```
RoleBinding "dev-team-binding"
  Namespace: dev
  Lie:
    - Sujet: User "alice"
    - Au rôle: Role "developer"
```

**Exemple :**
```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: dev-team-binding
  namespace: dev
subjects:
- kind: User
  name: alice
  apiGroup: rbac.authorization.k8s.io
- kind: User
  name: bob
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: developer
  apiGroup: rbac.authorization.k8s.io
```

**Points importants :**
- Lie un sujet (user, group, SA) à un rôle
- Limité au namespace où il est créé
- Peut référencer un Role OU un ClusterRole

### 4. ClusterRoleBinding

Un **ClusterRoleBinding** attribue les permissions d'un ClusterRole à un sujet **pour tout le cluster**.

**Exemple :**
```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: monitoring-binding
subjects:
- kind: ServiceAccount
  name: prometheus
  namespace: monitoring
roleRef:
  kind: ClusterRole
  name: cluster-reader
  apiGroup: rbac.authorization.k8s.io
```

**Caractéristiques :**
- Portée globale (tout le cluster)
- Référence toujours un ClusterRole
- Utilisé pour les permissions administratives ou cluster-wide

## Les Sujets (Subjects)

Les sujets sont les entités qui reçoivent des permissions. Il en existe trois types :

### 1. User (Utilisateur)

Représente une personne physique.

```yaml
subjects:
- kind: User
  name: alice@example.com
  apiGroup: rbac.authorization.k8s.io
```

**Important :** Kubernetes ne gère PAS les utilisateurs directement. Les utilisateurs sont gérés par :
- Certificats X.509
- Systèmes externes (OIDC, LDAP)
- Tokens d'authentification

### 2. Group (Groupe)

Représente un ensemble d'utilisateurs.

```yaml
subjects:
- kind: Group
  name: developers
  apiGroup: rbac.authorization.k8s.io
```

**Avantage :** Gérer les permissions par groupe plutôt qu'individuellement.

### 3. ServiceAccount

Représente une identité pour les applications et pods.

```yaml
subjects:
- kind: ServiceAccount
  name: my-app
  namespace: production
```

**Usage :** Les pods utilisent des ServiceAccounts pour interagir avec l'API Kubernetes.

## Les Verbes (Actions)

Les verbes définissent les actions autorisées. Voici les verbes les plus courants :

| Verbe | Description | Exemple |
|-------|-------------|---------|
| **get** | Récupérer une ressource spécifique | `kubectl get pod nginx` |
| **list** | Lister toutes les ressources d'un type | `kubectl get pods` |
| **watch** | Surveiller les changements en temps réel | Streaming des événements |
| **create** | Créer une nouvelle ressource | `kubectl create deployment` |
| **update** | Modifier une ressource existante | `kubectl edit deployment` |
| **patch** | Modifier partiellement une ressource | `kubectl patch` |
| **delete** | Supprimer une ressource | `kubectl delete pod` |
| **deletecollection** | Supprimer plusieurs ressources | `kubectl delete pods --all` |

### Verbes de Lecture (Read-Only)

```yaml
verbs: ["get", "list", "watch"]
```

Ces trois verbes ensemble donnent un accès en lecture seule.

### Verbes de Lecture-Écriture

```yaml
verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]
```

Accès complet à un type de ressource.

### Wildcard (Tout)

```yaml
verbs: ["*"]
```

**Attention :** À utiliser avec précaution ! Donne TOUTES les permissions.

## Les Ressources

Les ressources représentent les objets Kubernetes que vous voulez contrôler.

### Ressources de Base (apiGroup: "")

```yaml
resources: ["pods", "services", "configmaps", "secrets", "persistentvolumeclaims"]
```

### Ressources Apps (apiGroup: "apps")

```yaml
apiGroups: ["apps"]
resources: ["deployments", "statefulsets", "daemonsets", "replicasets"]
```

### Ressources Batch (apiGroup: "batch")

```yaml
apiGroups: ["batch"]
resources: ["jobs", "cronjobs"]
```

### Sous-Ressources

Certaines ressources ont des sous-ressources :

```yaml
resources: ["pods/log", "pods/exec", "pods/portforward"]
```

**Exemple :**
```yaml
# Permettre de voir les logs mais pas d'exécuter de commandes
resources: ["pods/log"]
verbs: ["get"]
```

### ResourceNames (Filtrage Spécifique)

Vous pouvez limiter l'accès à des ressources nommées spécifiquement :

```yaml
rules:
- apiGroups: [""]
  resources: ["secrets"]
  resourceNames: ["app-secret", "db-password"]
  verbs: ["get"]
```

Ici, l'accès est limité uniquement aux secrets nommés "app-secret" et "db-password".

## Combinaisons Courantes

### Rôle en Lecture Seule

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: production
  name: reader
rules:
- apiGroups: ["", "apps", "batch"]
  resources: ["*"]
  verbs: ["get", "list", "watch"]
```

**Usage :** Développeur qui a besoin de voir ce qui se passe sans pouvoir modifier.

### Rôle Développeur

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: dev
  name: developer
rules:
- apiGroups: [""]
  resources: ["pods", "services", "configmaps"]
  verbs: ["*"]
- apiGroups: ["apps"]
  resources: ["deployments", "replicasets"]
  verbs: ["*"]
- apiGroups: [""]
  resources: ["secrets"]
  verbs: ["get", "list"]  # Lecture seule pour les secrets
```

**Usage :** Développeur avec permissions complètes sauf pour les secrets.

### Rôle CI/CD

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: production
  name: deployer
rules:
- apiGroups: ["apps"]
  resources: ["deployments"]
  verbs: ["get", "list", "create", "update", "patch"]
- apiGroups: [""]
  resources: ["services"]
  verbs: ["get", "list"]
```

**Usage :** Pipeline de déploiement automatisé qui peut déployer mais pas supprimer.

## Scénarios Pratiques

### Scénario 1 : Équipe de Développement

**Besoin :** L'équipe dev doit gérer ses applications dans le namespace "dev" mais pas toucher à la production.

```yaml
---
apiVersion: v1
kind: Namespace
metadata:
  name: dev
---
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: dev
  name: dev-team-role
rules:
- apiGroups: ["", "apps", "batch"]
  resources: ["*"]
  verbs: ["*"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: dev-team-binding
  namespace: dev
subjects:
- kind: Group
  name: developers
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: dev-team-role
  apiGroup: rbac.authorization.k8s.io
```

**Résultat :** Les développeurs ont tous les droits dans "dev" mais aucun accès ailleurs.

### Scénario 2 : Monitoring Read-Only

**Besoin :** Prometheus doit lire les métriques de tout le cluster sans pouvoir modifier quoi que ce soit.

```yaml
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: prometheus
  namespace: monitoring
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: prometheus-reader
rules:
- apiGroups: [""]
  resources: ["nodes", "nodes/metrics", "services", "endpoints", "pods"]
  verbs: ["get", "list", "watch"]
- apiGroups: ["apps"]
  resources: ["deployments", "daemonsets", "statefulsets"]
  verbs: ["get", "list", "watch"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: prometheus-binding
subjects:
- kind: ServiceAccount
  name: prometheus
  namespace: monitoring
roleRef:
  kind: ClusterRole
  name: prometheus-reader
  apiGroup: rbac.authorization.k8s.io
```

**Résultat :** Prometheus peut surveiller tout le cluster en lecture seule.

### Scénario 3 : Application avec Accès Limité

**Besoin :** Une application a besoin de lister les pods dans son propre namespace.

```yaml
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: my-app
  namespace: production
---
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: production
  name: app-role
rules:
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["get", "list"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: app-binding
  namespace: production
subjects:
- kind: ServiceAccount
  name: my-app
  namespace: production
roleRef:
  kind: Role
  name: app-role
  apiGroup: rbac.authorization.k8s.io
```

**Utilisation dans le Pod :**
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-app
  namespace: production
spec:
  serviceAccountName: my-app
  containers:
  - name: app
    image: my-app:latest
```

## Portée et Combinaisons

### Tableau de Portée

| Binding Type | Role Type | Portée | Usage |
|--------------|-----------|--------|-------|
| RoleBinding | Role | Namespace spécifique | Standard |
| RoleBinding | ClusterRole | Namespace spécifique | Réutilisation |
| ClusterRoleBinding | ClusterRole | Tout le cluster | Admin/Global |

### RoleBinding avec ClusterRole

**Astuce :** Vous pouvez utiliser un RoleBinding pour lier un ClusterRole dans un namespace spécifique.

```yaml
# ClusterRole réutilisable
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: generic-reader
rules:
- apiGroups: [""]
  resources: ["pods", "services"]
  verbs: ["get", "list", "watch"]
---
# Utilisation dans le namespace "dev"
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: dev-reader-binding
  namespace: dev
subjects:
- kind: User
  name: alice
roleRef:
  kind: ClusterRole  # Note: ClusterRole, pas Role
  name: generic-reader
```

**Avantage :** Définir une fois, réutiliser partout.

## ClusterRoles Prédéfinis

Kubernetes fournit des ClusterRoles prédéfinis très utiles :

### 1. cluster-admin

```bash
# Donne TOUS les droits sur TOUT le cluster
# ⚠️ À utiliser avec extrême prudence
```

**Permissions :** Accès complet à toutes les ressources.

### 2. admin

**Permissions :** Accès complet dans un namespace (via RoleBinding).
- Lire et écrire la plupart des ressources
- Gérer les roles et rolebindings dans le namespace

### 3. edit

**Permissions :** Lire et écrire les ressources dans un namespace.
- Créer/modifier des pods, services, deployments
- Ne peut PAS modifier les roles et rolebindings

### 4. view

**Permissions :** Lecture seule dans un namespace.
- Voir les ressources
- Ne peut PAS voir les secrets

### Utilisation des Rôles Prédéfinis

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: alice-edit
  namespace: dev
subjects:
- kind: User
  name: alice
roleRef:
  kind: ClusterRole
  name: edit  # Rôle prédéfini
  apiGroup: rbac.authorization.k8s.io
```

## Commandes kubectl Utiles

### Voir les Rôles

```bash
# Lister tous les roles dans un namespace
kubectl get roles -n dev

# Lister tous les ClusterRoles
kubectl get clusterroles

# Détails d'un role
kubectl describe role developer -n dev

# Détails d'un ClusterRole
kubectl describe clusterrole view
```

### Voir les Bindings

```bash
# RoleBindings dans un namespace
kubectl get rolebindings -n dev

# ClusterRoleBindings
kubectl get clusterrolebindings

# Détails
kubectl describe rolebinding dev-team-binding -n dev
```

### Vérifier les Permissions

```bash
# Vérifier si vous pouvez faire une action
kubectl auth can-i create deployments -n dev

# Vérifier pour un autre utilisateur
kubectl auth can-i list pods -n dev --as alice

# Vérifier pour un ServiceAccount
kubectl auth can-i get secrets -n production --as system:serviceaccount:production:my-app

# Lister toutes les permissions d'un utilisateur
kubectl auth can-i --list -n dev --as alice
```

### Créer des Ressources RBAC

```bash
# Créer un Role depuis un fichier
kubectl create -f developer-role.yaml

# Créer un RoleBinding en ligne de commande
kubectl create rolebinding alice-edit \
  --clusterrole=edit \
  --user=alice \
  --namespace=dev

# Créer un ClusterRoleBinding
kubectl create clusterrolebinding prometheus-reader \
  --clusterrole=view \
  --serviceaccount=monitoring:prometheus
```

## Bonnes Pratiques RBAC

### 1. Principe du Moindre Privilège

**Mauvais :**
```yaml
# Ne faites pas ça !
verbs: ["*"]
resources: ["*"]
```

**Bon :**
```yaml
# Soyez spécifique
verbs: ["get", "list", "watch"]
resources: ["pods", "services"]
```

### 2. Utiliser des Namespaces

Séparez vos environnements :
```
dev → permissions larges
staging → permissions modérées
production → permissions restreintes
```

### 3. ServiceAccounts Dédiés

Créez un ServiceAccount par application :

```yaml
# Mauvais : réutiliser le ServiceAccount par défaut
serviceAccountName: default

# Bon : ServiceAccount dédié
serviceAccountName: my-app
```

### 4. Groupes plutôt qu'Utilisateurs Individuels

```yaml
# Moins maintenable
subjects:
- kind: User
  name: alice
- kind: User
  name: bob
- kind: User
  name: charlie

# Plus maintenable
subjects:
- kind: Group
  name: developers
```

### 5. Rôles Réutilisables

Créez des ClusterRoles génériques et liez-les avec des RoleBindings :

```yaml
# Définir une fois
kind: ClusterRole
name: generic-developer

# Réutiliser partout
kind: RoleBinding → namespace: dev
kind: RoleBinding → namespace: staging
```

### 6. Documenter les Permissions

```yaml
metadata:
  name: developer
  annotations:
    description: "Permet aux développeurs de gérer leurs applications"
    owner: "team-platform"
```

### 7. Auditer Régulièrement

```bash
# Qui a accès au namespace production ?
kubectl get rolebindings -n production -o wide

# Quels sont les ClusterRoleBindings sensibles ?
kubectl get clusterrolebindings | grep cluster-admin
```

### 8. Éviter cluster-admin

```yaml
# ⚠️ Ne donnez jamais cluster-admin sauf nécessité absolue
roleRef:
  kind: ClusterRole
  name: cluster-admin  # Dangereux !
```

## Débogage RBAC

### Problème : "Forbidden"

```bash
Error from server (Forbidden): pods is forbidden:
User "alice" cannot list resource "pods" in API group ""
in the namespace "production"
```

**Diagnostic :**

1. Vérifier l'authentification :
```bash
kubectl auth whoami
```

2. Vérifier les permissions :
```bash
kubectl auth can-i list pods -n production --as alice
```

3. Chercher les RoleBindings :
```bash
kubectl get rolebindings -n production
kubectl get clusterrolebindings | grep alice
```

4. Examiner le rôle :
```bash
kubectl describe role <role-name> -n production
```

### Problème : Permissions Manquantes

**Symptôme :** Une action spécifique échoue.

**Solution :**
```bash
# Tester l'action
kubectl auth can-i create deployments -n dev --as alice

# Si "no", ajouter la permission
kubectl edit role developer -n dev

# Ajouter :
- apiGroups: ["apps"]
  resources: ["deployments"]
  verbs: ["create"]
```

### Problème : Trop de Permissions

**Symptôme :** Un utilisateur peut faire des choses qu'il ne devrait pas.

**Solution :**
```bash
# Lister toutes les permissions
kubectl auth can-i --list -n dev --as alice

# Identifier le binding problématique
kubectl get rolebindings -n dev -o yaml | grep alice

# Modifier ou supprimer
kubectl delete rolebinding problematic-binding -n dev
```

## RBAC dans MicroK8s

### Configuration par Défaut

MicroK8s active RBAC par défaut. Lorsque vous utilisez `microk8s kubectl`, vous avez automatiquement les permissions cluster-admin.

### Créer des Utilisateurs pour le Développement

Pour tester RBAC dans votre lab :

1. **Créer un certificat client** (représente un utilisateur)
2. **Créer un Role et RoleBinding**
3. **Tester avec le nouveau contexte**

**Note :** Les détails de création d'utilisateurs sortent du cadre de ce chapitre, mais sachez que c'est possible.

### Tester RBAC

Vous pouvez simuler des utilisateurs sans créer de certificats :

```bash
# Simuler l'utilisateur alice
microk8s kubectl auth can-i list pods -n dev --as alice

# Tester une action
microk8s kubectl get pods -n dev --as alice
```

## Agrégation de ClusterRoles

Fonctionnalité avancée : créer des ClusterRoles en agrégeant d'autres ClusterRoles.

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: monitoring-all
aggregationRule:
  clusterRoleSelectors:
  - matchLabels:
      rbac.monitoring: "true"
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: monitoring-pods
  labels:
    rbac.monitoring: "true"
rules:
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["get", "list"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: monitoring-services
  labels:
    rbac.monitoring: "true"
rules:
- apiGroups: [""]
  resources: ["services"]
  verbs: ["get", "list"]
```

Le ClusterRole "monitoring-all" contiendra automatiquement toutes les règles des ClusterRoles avec le label `rbac.monitoring: "true"`.

## Exemples Complets

### Exemple 1 : Environnement Multi-Équipes

```yaml
---
# Namespace pour l'équipe A
apiVersion: v1
kind: Namespace
metadata:
  name: team-a
---
# Role pour l'équipe A
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: team-a
  name: team-a-full-access
rules:
- apiGroups: ["", "apps", "batch"]
  resources: ["*"]
  verbs: ["*"]
---
# Binding pour l'équipe A
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: team-a-binding
  namespace: team-a
subjects:
- kind: Group
  name: team-a-developers
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: team-a-full-access
  apiGroup: rbac.authorization.k8s.io
---
# L'équipe A peut VOIR les ressources de l'équipe B (read-only)
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: team-a-view-team-b
  namespace: team-b
subjects:
- kind: Group
  name: team-a-developers
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: view
  apiGroup: rbac.authorization.k8s.io
```

### Exemple 2 : Application avec Auto-Discovery

```yaml
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: discovery-app
  namespace: production
---
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: production
  name: discovery-role
rules:
# L'application peut découvrir les autres services
- apiGroups: [""]
  resources: ["services", "endpoints"]
  verbs: ["get", "list", "watch"]
# L'application peut lire sa propre configuration
- apiGroups: [""]
  resources: ["configmaps"]
  resourceNames: ["discovery-app-config"]
  verbs: ["get"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: discovery-binding
  namespace: production
subjects:
- kind: ServiceAccount
  name: discovery-app
  namespace: production
roleRef:
  kind: Role
  name: discovery-role
  apiGroup: rbac.authorization.k8s.io
```

## Conclusion

RBAC est un système puissant mais simple une fois les concepts de base compris :

1. **Role/ClusterRole** : définit les permissions (QUOI)
2. **RoleBinding/ClusterRoleBinding** : attribue les permissions (QUI)
3. **Subjects** : utilisateurs, groupes ou ServiceAccounts (À QUI)
4. **Verbes** : actions autorisées (ACTIONS)
5. **Ressources** : objets Kubernetes concernés (SUR QUOI)

**Formule RBAC :**
```
QUI (Subject) + PEUT FAIRE (Verbs) + QUOI (Resources) = Permissions
```

**Points Clés à Retenir :**

- RBAC est le mécanisme standard de contrôle d'accès dans Kubernetes
- Utilisez toujours le principe du moindre privilège
- Les namespaces isolent naturellement les permissions
- Les ClusterRoles sont réutilisables avec des RoleBindings
- Les rôles prédéfinis (view, edit, admin) couvrent la plupart des besoins
- Testez avec `kubectl auth can-i` avant de donner des permissions
- Documentez vos rôles et faites des audits réguliers

**Prochaines Étapes :**

Dans les sections suivantes, nous explorerons :
- **16.3** : ServiceAccounts (identités pour les applications)
- **16.4** : Network Policies (isolation réseau)
- **16.5** : Pod Security Standards (sécurité des conteneurs)

RBAC est la fondation de la sécurité Kubernetes. Maîtriser RBAC vous permettra de construire des clusters sécurisés et bien organisés.

⏭️ [ServiceAccounts](/16-securite-kubernetes/03-serviceaccounts.md)
