🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 3.1 Pods : l'unité de base

## Introduction

Le Pod est le concept le plus fondamental de Kubernetes. Avant de déployer quoi que ce soit dans votre cluster MicroK8s, il est essentiel de bien comprendre ce qu'est un Pod et comment il fonctionne.

## Qu'est-ce qu'un Pod ?

Un **Pod** est la plus petite unité déployable dans Kubernetes. C'est un peu comme une "enveloppe" qui contient un ou plusieurs conteneurs qui doivent fonctionner ensemble.

### Analogie simple

Imaginez un Pod comme une **colocation** :
- L'appartement = le Pod
- Les colocataires = les conteneurs
- Ils partagent les mêmes ressources (cuisine, salon) = réseau et stockage partagés
- Ils vivent ensemble et communiquent facilement entre eux

## Caractéristiques principales d'un Pod

### 1. Unité atomique

Un Pod est considéré comme une unité atomique, ce qui signifie que :
- Il est créé dans son ensemble
- Il est détruit dans son ensemble
- On ne peut pas "retirer" un conteneur d'un Pod en cours d'exécution

### 2. Réseau partagé

Tous les conteneurs d'un même Pod partagent :
- **La même adresse IP** : tous les conteneurs du Pod utilisent la même IP
- **Le même namespace réseau** : ils peuvent communiquer via `localhost`
- **Les mêmes ports** : deux conteneurs dans le même Pod ne peuvent pas utiliser le même port

**Exemple concret** : Si un Pod a l'IP `10.1.5.23`, tous ses conteneurs utilisent cette IP. Un conteneur peut joindre un autre conteneur du même Pod sur `localhost:8080`.

### 3. Stockage partagé

Les conteneurs d'un même Pod peuvent partager des volumes de stockage, ce qui leur permet d'échanger des fichiers facilement.

### 4. Cycle de vie éphémère

Les Pods sont **éphémères** par nature :
- Ils peuvent être créés et détruits à tout moment
- Lorsqu'un Pod meurt, il ne "revient" pas : un nouveau Pod est créé avec une nouvelle IP
- Ils ne sont pas conçus pour être "réparés" mais remplacés

## Anatomie d'un Pod

### Les composants essentiels

Un Pod est composé de :

1. **Un ou plusieurs conteneurs** : les applications qui s'exécutent
2. **Des volumes** (optionnel) : pour le stockage partagé
3. **Une configuration réseau** : IP unique attribuée automatiquement
4. **Des métadonnées** : nom, labels, annotations

### Représentation visuelle

```
┌─────────────────────────────────────┐
│           Pod (10.1.5.23)           │
│  ┌──────────────┐  ┌──────────────┐ │
│  │ Conteneur 1  │  │ Conteneur 2  │ │
│  │   (nginx)    │  │   (sidecar)  │ │
│  │  Port: 80    │  │  Port: 9090  │ │
│  └──────────────┘  └──────────────┘ │
│           │              │          │
│           └──────┬───────┘          │
│              localhost              │
│                                     │
│  ┌──────────────────────────────┐   │
│  │    Volumes partagés          │   │
│  └──────────────────────────────┘   │
└─────────────────────────────────────┘
```

## Configuration multi-conteneurs : quand et pourquoi ?

### Un seul conteneur (cas le plus courant)

Dans 90% des cas, un Pod contient **un seul conteneur**. C'est le pattern le plus simple et le plus courant.

**Exemple** : Un Pod qui exécute une application web nginx.

### Plusieurs conteneurs (patterns avancés)

Il existe des cas où plusieurs conteneurs dans le même Pod sont nécessaires :

#### 1. Pattern Sidecar
Un conteneur auxiliaire qui assiste le conteneur principal.

**Exemple** :
- Conteneur principal : application web
- Conteneur sidecar : collecteur de logs qui envoie les logs vers un système centralisé

#### 2. Pattern Ambassador
Un conteneur qui fait office de proxy pour le conteneur principal.

**Exemple** :
- Conteneur principal : application
- Conteneur ambassador : proxy qui gère la connexion à une base de données externe

#### 3. Pattern Adapter
Un conteneur qui transforme les données du conteneur principal.

**Exemple** :
- Conteneur principal : application qui génère des logs dans un format spécifique
- Conteneur adapter : transforme les logs dans un format standard

## Manifeste YAML d'un Pod simple

Voici à quoi ressemble la définition d'un Pod en YAML :

```yaml
apiVersion: v1                    # Version de l'API Kubernetes
kind: Pod                         # Type de ressource : Pod
metadata:                         # Métadonnées du Pod
  name: mon-premier-pod           # Nom unique du Pod
  labels:                         # Labels pour identifier le Pod
    app: nginx
    environnement: dev
spec:                             # Spécification du Pod
  containers:                     # Liste des conteneurs
  - name: nginx-container         # Nom du conteneur
    image: nginx:1.25             # Image Docker à utiliser
    ports:                        # Ports exposés
    - containerPort: 80           # Port sur lequel le conteneur écoute
```

### Explication ligne par ligne

- **apiVersion: v1** : Indique quelle version de l'API Kubernetes utiliser. Pour les Pods, c'est toujours `v1`.

- **kind: Pod** : Spécifie le type de ressource. Ici, nous créons un Pod.

- **metadata** : Section contenant les informations d'identification du Pod
  - **name** : Nom unique du Pod dans son namespace
  - **labels** : Paires clé-valeur pour organiser et sélectionner les ressources

- **spec** : Spécification technique du Pod (ce qu'il contient)
  - **containers** : Liste des conteneurs à exécuter dans le Pod
    - **name** : Nom du conteneur (unique dans le Pod)
    - **image** : Image Docker à télécharger et exécuter
    - **ports** : Ports que le conteneur expose

## Exemple avec plusieurs conteneurs

Voici un Pod avec deux conteneurs (pattern sidecar) :

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-multi-conteneurs
  labels:
    app: web-with-logger
spec:
  containers:
  # Conteneur principal
  - name: application-web
    image: nginx:1.25
    ports:
    - containerPort: 80
    volumeMounts:                 # Montage d'un volume
    - name: logs-partages
      mountPath: /var/log/nginx

  # Conteneur sidecar (collecteur de logs)
  - name: log-collector
    image: busybox:1.36
    command: ['sh', '-c']
    args:
    - while true; do
        echo "Collecte des logs...";
        sleep 30;
      done
    volumeMounts:                 # Accès au même volume
    - name: logs-partages
      mountPath: /logs

  # Définition du volume partagé
  volumes:
  - name: logs-partages
    emptyDir: {}                  # Volume temporaire
```

Dans cet exemple :
- Le conteneur `application-web` écrit ses logs dans `/var/log/nginx`
- Le conteneur `log-collector` peut lire ces logs depuis `/logs`
- Les deux conteneurs partagent le volume `logs-partages`
- Ils communiquent via `localhost` car ils sont dans le même Pod

## États d'un Pod

Un Pod passe par différents états durant son cycle de vie :

### Les phases principales

1. **Pending** (En attente)
   - Le Pod a été accepté par Kubernetes
   - Les images de conteneurs sont en cours de téléchargement
   - Le Pod attend d'être planifié sur un nœud

2. **Running** (En cours d'exécution)
   - Le Pod a été attribué à un nœud
   - Tous les conteneurs ont été créés
   - Au moins un conteneur est en cours d'exécution

3. **Succeeded** (Réussi)
   - Tous les conteneurs du Pod se sont terminés avec succès
   - Le Pod ne sera pas redémarré

4. **Failed** (Échoué)
   - Tous les conteneurs se sont terminés
   - Au moins un conteneur s'est terminé en erreur

5. **Unknown** (Inconnu)
   - L'état du Pod ne peut pas être déterminé
   - Généralement dû à un problème de communication avec le nœud

### Diagramme des transitions

```
    ┌─────────┐
    │ Pending │
    └────┬────┘
         │
         v
    ┌─────────┐
    │ Running │
    └────┬────┘
         │
    ┌────┴────────┐
    v             v
┌──────────┐  ┌────────┐
│Succeeded │  │ Failed │
└──────────┘  └────────┘
```

## Conditions d'un Pod

En plus de la phase, un Pod possède des **conditions** qui donnent plus de détails :

- **PodScheduled** : Le Pod a été assigné à un nœud
- **ContainersReady** : Tous les conteneurs du Pod sont prêts
- **Initialized** : Tous les init containers ont été exécutés avec succès
- **Ready** : Le Pod peut recevoir du trafic

## Politique de redémarrage (Restart Policy)

Kubernetes peut automatiquement redémarrer les conteneurs d'un Pod selon la politique définie :

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-avec-restart-policy
spec:
  restartPolicy: Always    # Always, OnFailure, ou Never
  containers:
  - name: mon-conteneur
    image: nginx:1.25
```

### Les trois options

1. **Always** (par défaut)
   - Redémarre toujours le conteneur, même s'il se termine avec succès
   - Utilisé pour les applications qui doivent toujours tourner

2. **OnFailure**
   - Redémarre uniquement si le conteneur se termine en erreur
   - Utilisé pour les tâches qui doivent réussir

3. **Never**
   - Ne redémarre jamais le conteneur
   - Utilisé pour les tâches ponctuelles

## Limitations importantes des Pods

### 1. Ne pas créer des Pods directement

**Bonne pratique** : Dans la pratique, on ne crée presque **jamais** de Pods directement. On utilise des contrôleurs de plus haut niveau comme :
- **Deployments** (le plus courant)
- **StatefulSets**
- **DaemonSets**
- **Jobs**

**Raison** : Ces contrôleurs gèrent automatiquement :
- Le redémarrage des Pods en cas de panne
- La mise à l'échelle
- Les mises à jour progressives

### 2. Pas de garantie de persistance

Si un Pod meurt :
- Il ne revient pas "à la vie"
- Un nouveau Pod est créé avec une **nouvelle IP**
- Toutes les données non stockées sur un volume persistant sont perdues

### 3. Un Pod = une seule machine

Un Pod s'exécute toujours sur **un seul nœud**. On ne peut pas répartir les conteneurs d'un même Pod sur plusieurs machines.

## Bonnes pratiques

### 1. Un conteneur par Pod (sauf exception)

Privilégiez les Pods avec un seul conteneur, sauf si vous avez une raison valable d'utiliser plusieurs conteneurs (patterns sidecar, ambassador, adapter).

### 2. Utilisez des labels

Les labels permettent d'organiser et de sélectionner vos Pods facilement :

```yaml
metadata:
  labels:
    app: mon-application
    version: v1.0
    environnement: production
    equipe: backend
```

### 3. Nommez vos conteneurs explicitement

Donnez des noms descriptifs à vos conteneurs pour faciliter le débogage :

```yaml
containers:
- name: nginx-webserver      # Clair et descriptif
  image: nginx:1.25
```

### 4. Spécifiez toujours une version d'image

Évitez d'utiliser le tag `latest` :

```yaml
# ❌ Mauvais
image: nginx:latest

# ✅ Bon
image: nginx:1.25.3
```

### 5. Ne créez pas de Pods "nus"

Utilisez toujours un Deployment ou un autre contrôleur plutôt que de créer des Pods directement.

## Pourquoi les Pods sont-ils conçus ainsi ?

### Design pour la modularité

Les Pods permettent de :
- **Grouper logiquement** des conteneurs qui doivent travailler ensemble
- **Partager des ressources** facilement (réseau, stockage)
- **Faciliter la communication** entre conteneurs liés

### Design pour la scalabilité

Kubernetes peut :
- Créer plusieurs copies du même Pod pour gérer plus de charge
- Distribuer les Pods sur différents nœuds
- Remplacer rapidement les Pods défaillants

### Design pour la résilience

Si un conteneur plante :
- Kubernetes peut le redémarrer automatiquement
- Si le nœud entier tombe, les Pods sont recréés sur un autre nœud

## Résumé des points clés

- Un **Pod** est la plus petite unité déployable dans Kubernetes
- Il contient **un ou plusieurs conteneurs** qui partagent le réseau et le stockage
- Les conteneurs d'un Pod communiquent via **localhost**
- Chaque Pod a une **IP unique** dans le cluster
- Les Pods sont **éphémères** : ils peuvent être créés et détruits à tout moment
- On ne crée **presque jamais** de Pods directement, on utilise des **Deployments**
- Un Pod s'exécute sur **un seul nœud** à la fois

## Prochaines étapes

Maintenant que vous comprenez ce qu'est un Pod, vous êtes prêt à découvrir :
- **Les Deployments** : comment déployer et gérer plusieurs Pods de manière robuste
- **Les ReplicaSets** : comment maintenir un nombre désiré de Pods en fonctionnement
- **Les Services** : comment exposer vos Pods et permettre la communication entre eux

Les Pods sont la fondation de tout déploiement Kubernetes. Bien les comprendre vous permettra de maîtriser tous les concepts avancés par la suite !

⏭️ [Deployments et ReplicaSets](/03-concepts-kubernetes-essentiels/02-deployments-et-replicasets.md)
