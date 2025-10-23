🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 4.1 Anatomie d'un Manifeste YAML

## Introduction

Dans Kubernetes, tout se définit par des fichiers de configuration appelés **manifestes**. Ces fichiers utilisent le format YAML (Yet Another Markup Language), un langage de sérialisation de données lisible par l'humain. Comprendre la structure d'un manifeste YAML est fondamental pour travailler efficacement avec Kubernetes et MicroK8s.

## Qu'est-ce que YAML ?

YAML est un format de fichier qui permet de structurer des données de manière claire et lisible. Voici ses caractéristiques principales :

- **Sensible à l'indentation** : Les espaces (et non les tabulations) définissent la hiérarchie
- **Lisible** : Conçu pour être facilement compris par les humains
- **Structuré** : Organise les données sous forme de clés et de valeurs

### Règles de base YAML

```yaml
# Ceci est un commentaire
cle: valeur                    # Paire clé-valeur simple
nombre: 42                     # Les nombres n'ont pas besoin de guillemets
texte: "Hello World"          # Les chaînes peuvent être entre guillemets

# Liste (tableau)
liste:
  - element1
  - element2
  - element3

# Objet imbriqué
objet:
  propriete1: valeur1
  propriete2: valeur2
```

**⚠️ Points d'attention :**
- Utilisez **toujours des espaces**, jamais de tabulations
- L'indentation standard est de **2 espaces**
- Les tirets `-` créent des listes
- Les deux-points `:` séparent les clés des valeurs

## Structure Générale d'un Manifeste Kubernetes

Tous les manifestes Kubernetes suivent une structure de base identique, composée de **quatre sections principales** :

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: mon-application
spec:
  # Spécifications détaillées de l'objet
```

Examinons chaque section en détail.

## 1. apiVersion : La Version de l'API

```yaml
apiVersion: v1
```

Le champ `apiVersion` indique quelle version de l'API Kubernetes utiliser pour créer cet objet. Kubernetes organise ses ressources en groupes d'API.

### Versions courantes

| apiVersion | Types d'objets | Exemple |
|------------|----------------|---------|
| `v1` | Ressources de base stables | Pod, Service, ConfigMap, Secret |
| `apps/v1` | Applications | Deployment, StatefulSet, DaemonSet |
| `batch/v1` | Tâches | Job, CronJob |
| `networking.k8s.io/v1` | Réseau | Ingress, NetworkPolicy |
| `storage.k8s.io/v1` | Stockage | StorageClass |

**💡 Astuce :** Pour connaître les versions disponibles pour un type d'objet :
```bash
kubectl api-resources
```

## 2. kind : Le Type d'Objet

```yaml
kind: Pod
```

Le champ `kind` définit le **type de ressource** Kubernetes que vous créez. C'est l'équivalent d'une "classe" ou d'un "type" en programmation.

### Types d'objets principaux

**Objets de base :**
- `Pod` : L'unité de déploiement la plus petite
- `Service` : Expose des Pods sur le réseau
- `ConfigMap` : Stocke des configurations non sensibles
- `Secret` : Stocke des données sensibles

**Contrôleurs :**
- `Deployment` : Gère des Pods répliqués et leurs mises à jour
- `StatefulSet` : Pour applications avec état (bases de données)
- `DaemonSet` : Exécute un Pod sur chaque nœud
- `Job` : Exécute une tâche ponctuelle
- `CronJob` : Exécute des tâches planifiées

**Réseau et stockage :**
- `Ingress` : Routage HTTP/HTTPS
- `PersistentVolume` : Volume de stockage
- `PersistentVolumeClaim` : Demande de stockage

## 3. metadata : Les Métadonnées

```yaml
metadata:
  name: mon-application
  namespace: production
  labels:
    app: web
    environment: prod
  annotations:
    description: "Application web principale"
```

La section `metadata` contient des **informations d'identification** et de **classification** de l'objet.

### Champs essentiels

#### name (obligatoire)
Le nom unique de votre objet dans son namespace.

```yaml
metadata:
  name: mon-app
```

**Règles de nommage :**
- Lettres minuscules, chiffres et tirets uniquement
- Doit commencer et finir par une lettre ou un chiffre
- Maximum 253 caractères
- Exemples valides : `web-app`, `api-v2`, `database-01`

#### namespace (optionnel)
Le namespace permet d'organiser et d'isoler les ressources.

```yaml
metadata:
  namespace: production
```

Si omis, l'objet est créé dans le namespace `default`.

**Namespaces courants :**
- `default` : Namespace par défaut
- `kube-system` : Composants système Kubernetes
- `kube-public` : Ressources publiques
- Namespaces personnalisés : `dev`, `staging`, `production`, etc.

#### labels (optionnel mais recommandé)
Les labels sont des **paires clé-valeur** utilisées pour organiser et sélectionner des objets.

```yaml
metadata:
  labels:
    app: nginx
    version: "1.21"
    environment: production
    tier: frontend
```

**Utilisations des labels :**
- Sélection d'objets avec `kubectl get pods -l app=nginx`
- Association de Services aux Pods
- Organisation et filtrage des ressources
- Routage du trafic

**Bonnes pratiques de nommage :**
- Utilisez des noms descriptifs : `app`, `component`, `version`
- Soyez cohérent dans votre équipe
- Labels recommandés :
  - `app.kubernetes.io/name` : Nom de l'application
  - `app.kubernetes.io/version` : Version
  - `app.kubernetes.io/component` : Rôle (database, cache, etc.)
  - `app.kubernetes.io/managed-by` : Outil de gestion

#### annotations (optionnel)
Les annotations stockent des **métadonnées non identifiantes**, souvent utilisées par des outils externes.

```yaml
metadata:
  annotations:
    description: "Application web principale"
    contact: "equipe-dev@example.com"
    kubernetes.io/change-cause: "Mise à jour version 2.3"
```

**Différence labels vs annotations :**
- **Labels** : Pour sélectionner et organiser (limités en taille)
- **Annotations** : Pour documenter (peuvent être plus volumineuses)

## 4. spec : Les Spécifications

```yaml
spec:
  containers:
  - name: nginx
    image: nginx:1.21
    ports:
    - containerPort: 80
```

La section `spec` (spécification) définit **l'état désiré** de votre objet. Son contenu varie selon le `kind`.

### Exemple pour un Pod

```yaml
spec:
  containers:
  - name: mon-conteneur          # Nom du conteneur
    image: nginx:1.21             # Image Docker à utiliser
    ports:
    - containerPort: 80           # Port exposé par le conteneur
    env:                          # Variables d'environnement
    - name: ENV_VAR
      value: "valeur"
    resources:                    # Ressources CPU/RAM
      requests:
        memory: "64Mi"
        cpu: "250m"
      limits:
        memory: "128Mi"
        cpu: "500m"
```

### Exemple pour un Deployment

```yaml
spec:
  replicas: 3                     # Nombre de répliques
  selector:                       # Sélection des Pods à gérer
    matchLabels:
      app: nginx
  template:                       # Template des Pods
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.21
```

### Exemple pour un Service

```yaml
spec:
  selector:                       # Sélection des Pods cibles
    app: nginx
  ports:
  - protocol: TCP
    port: 80                      # Port du Service
    targetPort: 80                # Port du Pod
  type: ClusterIP                 # Type de Service
```

## Manifeste Complet : Exemple Commenté

Voici un exemple complet d'un manifeste Deployment avec tous les éléments expliqués :

```yaml
# Version de l'API pour les Deployments
apiVersion: apps/v1

# Type d'objet : un Deployment
kind: Deployment

# Métadonnées du Deployment
metadata:
  # Nom unique du Deployment
  name: webapp-nginx

  # Namespace où créer le Deployment
  namespace: production

  # Labels pour organiser et sélectionner
  labels:
    app: webapp
    component: frontend
    version: "1.0"

  # Annotations pour documenter
  annotations:
    description: "Application web basée sur Nginx"
    maintainer: "equipe-ops@example.com"

# Spécifications du Deployment
spec:
  # Nombre de répliques (Pods) à maintenir
  replicas: 3

  # Sélecteur : quels Pods ce Deployment gère
  selector:
    matchLabels:
      app: webapp
      component: frontend

  # Template : modèle pour créer les Pods
  template:
    # Métadonnées des Pods créés
    metadata:
      labels:
        app: webapp
        component: frontend
        version: "1.0"

    # Spécifications des Pods
    spec:
      # Liste des conteneurs dans le Pod
      containers:
      - name: nginx
        image: nginx:1.21-alpine

        # Ports exposés par le conteneur
        ports:
        - containerPort: 80
          name: http
          protocol: TCP

        # Variables d'environnement
        env:
        - name: ENVIRONMENT
          value: "production"

        # Ressources allouées
        resources:
          requests:
            memory: "128Mi"
            cpu: "100m"
          limits:
            memory: "256Mi"
            cpu: "200m"

        # Probes pour la santé du conteneur
        livenessProbe:
          httpGet:
            path: /
            port: 80
          initialDelaySeconds: 30
          periodSeconds: 10

        readinessProbe:
          httpGet:
            path: /
            port: 80
          initialDelaySeconds: 5
          periodSeconds: 5
```

## Format Multi-Documents

YAML permet de définir plusieurs objets dans un seul fichier en les séparant par `---` :

```yaml
# Premier objet : Deployment
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mon-app
spec:
  replicas: 2
  selector:
    matchLabels:
      app: mon-app
  template:
    metadata:
      labels:
        app: mon-app
    spec:
      containers:
      - name: app
        image: nginx:1.21

---
# Deuxième objet : Service
apiVersion: v1
kind: Service
metadata:
  name: mon-app-service
spec:
  selector:
    app: mon-app
  ports:
  - port: 80
    targetPort: 80
  type: ClusterIP
```

**Avantage :** Regrouper des ressources liées dans un seul fichier pour un déploiement cohérent.

## Validation et Bonnes Pratiques

### Vérifier la syntaxe YAML

Avant d'appliquer un manifeste, vous pouvez vérifier sa syntaxe :

```bash
# Validation sans création
kubectl apply --dry-run=client -f mon-manifeste.yaml

# Validation côté serveur (plus complète)
kubectl apply --dry-run=server -f mon-manifeste.yaml
```

### Bonnes pratiques générales

1. **Indentation cohérente** : Toujours 2 espaces
2. **Commentaires** : Documentez les sections complexes
3. **Noms explicites** : Utilisez des noms descriptifs
4. **Labels systématiques** : Facilitent la gestion et le débogage
5. **Versioning** : Stockez vos manifestes dans Git
6. **Organisation** : Un fichier par ressource ou regroupement logique
7. **Resources** : Définissez toujours requests et limits
8. **Namespaces** : Utilisez des namespaces pour isoler les environnements

### Erreurs courantes à éviter

❌ **Utiliser des tabulations**
```yaml
spec:
	containers:  # ← Tabulation (interdit)
```

✅ **Utiliser des espaces**
```yaml
spec:
  containers:  # ← 2 espaces (correct)
```

❌ **Oublier l'indentation**
```yaml
spec:
containers:    # ← Mauvaise indentation
- name: app
```

✅ **Respecter la hiérarchie**
```yaml
spec:
  containers:  # ← Bien indenté
  - name: app
```

❌ **Mélanger tirets et indentation**
```yaml
spec:
  containers:
    - name: app  # ← Trop indenté après le tiret
```

✅ **Aligner correctement les listes**
```yaml
spec:
  containers:
  - name: app    # ← Le tiret commence l'élément de liste
    image: nginx # ← Contenu de l'élément indenté
```

## Outils Utiles

### Éditeurs recommandés

- **VS Code** : Avec l'extension Kubernetes
- **IntelliJ IDEA** : Support YAML et Kubernetes intégré
- **Vim/Neovim** : Avec plugins YAML

### Validateurs en ligne

- [YAML Lint](http://www.yamllint.com/) : Validation syntaxe YAML
- [Kubeval](https://kubeval.com/) : Validation spécifique Kubernetes

### Commandes kubectl utiles

```bash
# Obtenir le YAML d'une ressource existante
kubectl get pod mon-pod -o yaml

# Générer un manifeste sans le créer
kubectl create deployment nginx --image=nginx --dry-run=client -o yaml

# Expliquer la structure d'une ressource
kubectl explain pod.spec.containers

# Voir la documentation complète
kubectl explain pod --recursive
```

## Récapitulatif

Un manifeste Kubernetes est composé de **quatre sections principales** :

1. **apiVersion** : Version de l'API à utiliser
2. **kind** : Type d'objet Kubernetes
3. **metadata** : Informations d'identification (nom, labels, annotations)
4. **spec** : Spécifications détaillées de l'état désiré

**Points clés à retenir :**

- YAML est sensible à l'indentation (toujours 2 espaces)
- Les labels permettent de sélectionner et d'organiser les objets
- La section spec varie selon le type d'objet
- Validez toujours vos manifestes avant de les appliquer
- Documentez avec des commentaires et annotations

Maintenant que vous comprenez l'anatomie d'un manifeste YAML, vous êtes prêt à créer vos premières ressources Kubernetes !

---

**Prochaine étape :** 4.2 Déploiement d'une application web simple

⏭️ [Déploiement d'une application web simple](/04-premiers-deploiements/02-deploiement-dune-application-web-simple.md)
