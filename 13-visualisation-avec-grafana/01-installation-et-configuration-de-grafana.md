🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 13.1 Installation et configuration de Grafana

## Introduction

Grafana est une plateforme open-source de visualisation et d'analyse de données qui permet de créer des tableaux de bord (dashboards) interactifs et esthétiques. Dans le contexte de Kubernetes et MicroK8s, Grafana est l'outil de référence pour visualiser les métriques collectées par Prometheus.

### Pourquoi utiliser Grafana ?

- **Visualisation intuitive** : Transforme des données complexes en graphiques compréhensibles
- **Tableaux de bord personnalisables** : Création de vues adaptées à vos besoins spécifiques
- **Alertes visuelles** : Identification rapide des problèmes grâce aux couleurs et seuils
- **Multi-sources** : Peut se connecter à Prometheus, mais aussi à d'autres sources de données
- **Communauté active** : Des milliers de dashboards pré-configurés disponibles gratuitement

### Le rôle de Grafana dans votre stack de monitoring

Dans l'écosystème de monitoring Kubernetes, Grafana joue le rôle de la couche de présentation :

```
Kubernetes Cluster → Prometheus (collecte) → Grafana (visualisation) → Vous
```

Prometheus collecte et stocke les métriques, Grafana les affiche de manière lisible et exploitable.

## Prérequis

Avant d'installer Grafana, assurez-vous d'avoir :

1. **MicroK8s installé et fonctionnel**
   ```bash
   microk8s status
   ```

2. **Prometheus déjà installé et opérationnel**
   ```bash
   microk8s enable prometheus
   ```

   Grafana a besoin d'une source de données pour être utile. Prometheus est la source principale que nous utiliserons.

3. **Un accès kubectl configuré**
   ```bash
   microk8s kubectl version
   ```

## Méthodes d'installation

Il existe plusieurs façons d'installer Grafana sur MicroK8s. Nous allons explorer les trois principales méthodes.

### Méthode 1 : Installation via l'addon Prometheus (Recommandée pour débutants)

La méthode la plus simple consiste à utiliser l'addon Prometheus de MicroK8s qui inclut déjà Grafana.

#### Vérification de l'installation

Si vous avez activé l'addon Prometheus, Grafana est déjà installé ! Vérifions :

```bash
microk8s kubectl get pods -n prometheus
```

Vous devriez voir un pod Grafana dans la liste :

```
NAME                                  READY   STATUS    RESTARTS   AGE
grafana-XXXXXXXXXX-xxxxx              1/1     Running   0          10m
prometheus-XXXXXXXXXX-xxxxx           1/1     Running   0          10m
...
```

#### Vérification du service

```bash
microk8s kubectl get svc -n prometheus
```

Résultat attendu :

```
NAME                    TYPE        CLUSTER-IP       PORT(S)    AGE
grafana                 ClusterIP   10.152.183.123   3000/TCP   10m
prometheus-k8s          ClusterIP   10.152.183.124   9090/TCP   10m
```

Le service Grafana écoute sur le port 3000.

### Méthode 2 : Installation manuelle avec un manifeste YAML

Pour plus de contrôle ou pour comprendre ce qui se passe en coulisse, vous pouvez installer Grafana manuellement.

#### Création du namespace (si nécessaire)

```bash
microk8s kubectl create namespace monitoring
```

#### Création du Deployment Grafana

Créez un fichier `grafana-deployment.yaml` :

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: grafana
  namespace: monitoring
  labels:
    app: grafana
spec:
  replicas: 1
  selector:
    matchLabels:
      app: grafana
  template:
    metadata:
      labels:
        app: grafana
    spec:
      containers:
      - name: grafana
        image: grafana/grafana:latest
        ports:
        - containerPort: 3000
          name: http
        env:
        - name: GF_SECURITY_ADMIN_USER
          value: "admin"
        - name: GF_SECURITY_ADMIN_PASSWORD
          value: "admin"
        volumeMounts:
        - name: grafana-storage
          mountPath: /var/lib/grafana
      volumes:
      - name: grafana-storage
        emptyDir: {}
```

**Explication du manifeste pour débutants :**

- `replicas: 1` : Une seule instance de Grafana (suffisant pour un lab)
- `image: grafana/grafana:latest` : L'image Docker officielle de Grafana
- `containerPort: 3000` : Le port par défaut de Grafana
- `GF_SECURITY_ADMIN_USER/PASSWORD` : Identifiants de connexion par défaut
- `volumeMounts` : Permet de persister les données de Grafana
- `emptyDir: {}` : Volume temporaire (les données seront perdues si le pod redémarre)

#### Création du Service

Créez un fichier `grafana-service.yaml` :

```yaml
apiVersion: v1
kind: Service
metadata:
  name: grafana
  namespace: monitoring
spec:
  type: NodePort
  selector:
    app: grafana
  ports:
  - port: 3000
    targetPort: 3000
    nodePort: 30300
```

**Explication :**

- `type: NodePort` : Permet d'accéder à Grafana depuis l'extérieur du cluster
- `port: 3000` : Port du service dans le cluster
- `targetPort: 3000` : Port du conteneur Grafana
- `nodePort: 30300` : Port exposé sur votre machine hôte

#### Déploiement

```bash
microk8s kubectl apply -f grafana-deployment.yaml
microk8s kubectl apply -f grafana-service.yaml
```

### Méthode 3 : Installation via Helm (Pour utilisateurs avancés)

Helm est un gestionnaire de packages pour Kubernetes qui simplifie les installations complexes.

#### Activation de l'addon Helm

```bash
microk8s enable helm3
```

#### Ajout du repository Grafana

```bash
microk8s helm3 repo add grafana https://grafana.github.io/helm-charts
microk8s helm3 repo update
```

#### Installation de Grafana

```bash
microk8s helm3 install grafana grafana/grafana \
  --namespace monitoring \
  --create-namespace \
  --set adminPassword=admin
```

Cette commande installe Grafana avec toutes les configurations recommandées par défaut.

## Accès à l'interface Grafana

Une fois Grafana installé, vous devez y accéder pour le configurer.

### Port-forwarding (Méthode simple et sécurisée)

Le port-forwarding crée un tunnel temporaire entre votre machine et le pod Grafana :

```bash
microk8s kubectl port-forward -n prometheus svc/grafana 3000:3000
```

Ou si vous avez installé dans le namespace `monitoring` :

```bash
microk8s kubectl port-forward -n monitoring svc/grafana 3000:3000
```

**Ouvrez votre navigateur à l'adresse :** `http://localhost:3000`

### Via NodePort (Si configuré)

Si vous avez utilisé un Service de type NodePort :

```bash
# Obtenez l'IP de votre nœud
ip addr show

# Accédez à Grafana
http://<IP-DE-VOTRE-MACHINE>:30300
```

### Via Ingress (Configuration avancée)

Pour une exposition plus professionnelle avec un nom de domaine, vous pouvez configurer un Ingress. Cela sera abordé plus en détail dans les chapitres suivants.

## Configuration initiale de Grafana

### Première connexion

1. **Ouvrez l'interface web** à `http://localhost:3000`

2. **Identifiants par défaut :**
   - **Username :** `admin`
   - **Password :** `admin` (ou celui que vous avez configuré)

3. **Changement de mot de passe :** Grafana vous demandera de changer le mot de passe lors de la première connexion. Choisissez un mot de passe fort.

### Interface Grafana : Vue d'ensemble

Une fois connecté, vous découvrez l'interface Grafana :

- **Panneau latéral gauche** : Menu principal de navigation
- **Home Dashboard** : Page d'accueil personnalisable
- **Search** : Recherche de dashboards existants
- **Create** : Création de nouveaux dashboards
- **Configuration** (icône engrenage) : Paramètres de Grafana

### Configuration de la langue

Par défaut, Grafana est en anglais. Pour changer la langue :

1. Cliquez sur l'icône **engrenage** (⚙️) dans le menu latéral
2. Sélectionnez **Preferences**
3. Dans **UI Theme**, vous pouvez aussi choisir entre le thème clair et sombre
4. Sauvegardez les modifications

## Configuration des sources de données (Data Sources)

Une source de données indique à Grafana où aller chercher les métriques à afficher.

### Ajout de Prometheus comme source de données

1. **Accédez aux Data Sources :**
   - Cliquez sur l'icône **engrenage** (⚙️) → **Data Sources**
   - Ou naviguez vers **Configuration → Data Sources**

2. **Ajoutez une nouvelle source :**
   - Cliquez sur **Add data source**
   - Sélectionnez **Prometheus**

3. **Configuration de Prometheus :**

   - **Name :** `Prometheus` (ou un nom de votre choix)

   - **URL :** L'URL où Prometheus est accessible depuis Grafana

     Si Prometheus est dans le même cluster (cas standard) :
     ```
     http://prometheus-k8s.prometheus.svc.cluster.local:9090
     ```

     Décomposition de cette URL pour débutants :
     - `prometheus-k8s` : Nom du service Prometheus
     - `prometheus` : Namespace où Prometheus est installé
     - `svc.cluster.local` : Suffixe DNS interne de Kubernetes
     - `9090` : Port de Prometheus

   - **Access :** Sélectionnez **Server (default)**

     Cela signifie que Grafana contactera Prometheus directement depuis le serveur, et non depuis votre navigateur.

4. **Paramètres avancés (optionnels pour débutants) :**

   - **Scrape interval :** `15s` (intervalle auquel Prometheus collecte les métriques)
   - **Query timeout :** `60s` (temps maximum pour une requête)
   - **HTTP Method :** `POST` (recommandé pour les requêtes complexes)

5. **Testez la connexion :**
   - Cliquez sur **Save & Test**
   - Vous devriez voir un message vert : **"Data source is working"**

### En cas d'erreur de connexion

Si le test échoue, vérifiez :

1. **Prometheus est bien en cours d'exécution :**
   ```bash
   microk8s kubectl get pods -n prometheus
   ```

2. **Le nom du service Prometheus :**
   ```bash
   microk8s kubectl get svc -n prometheus
   ```

3. **L'URL est correcte :** Elle doit correspondre au format `http://<nom-du-service>.<namespace>.svc.cluster.local:<port>`

### Vérification de la configuration

Une fois la source de données configurée, vous pouvez tester une requête :

1. Allez dans **Explore** (icône boussole dans le menu latéral)
2. Sélectionnez **Prometheus** comme source de données
3. Dans le champ de requête, tapez : `up`
4. Cliquez sur **Run Query**

Vous devriez voir un graphique montrant l'état (1 = up, 0 = down) de tous les targets Prometheus.

## Configuration de base de Grafana

### Paramètres généraux

1. **Accédez aux paramètres :**
   - **Configuration** (engrenage) → **Preferences**

2. **Paramètres recommandés pour débutants :**

   - **Home Dashboard :** Vous pouvez définir un dashboard par défaut
   - **Timezone :** Sélectionnez votre fuseau horaire local
   - **Week start :** Définissez le premier jour de la semaine (lundi ou dimanche)

### Paramètres d'administration

Pour les administrateurs de Grafana :

1. **Accédez à :** **Configuration → Settings** (ou **Server Admin → Settings**)

2. **Paramètres importants :**

   - **Allow Sign Up :** Désactivé (false) pour éviter les inscriptions non autorisées
   - **Allow Org Creation :** Désactivé pour un lab personnel
   - **Auto Assign Org Role :** Viewer (rôle par défaut pour les nouveaux utilisateurs)

### Gestion des utilisateurs et permissions

Grafana propose trois rôles principaux :

- **Viewer :** Peut uniquement consulter les dashboards
- **Editor :** Peut créer et modifier des dashboards
- **Admin :** Contrôle total sur Grafana

Pour un lab personnel, vous serez probablement le seul utilisateur avec le rôle Admin.

Pour créer un utilisateur supplémentaire :

1. **Configuration → Users**
2. **Invite / Add user**
3. Remplissez les informations et assignez un rôle

## Organisation des dashboards

### Création de dossiers

Pour organiser vos dashboards, créez des dossiers :

1. **Dashboards → Manage** (ou **Browse**)
2. **New Folder**
3. Nommez votre dossier (exemple : "Kubernetes", "Applications", "Infrastructure")

### Organisation recommandée pour un lab Kubernetes

```
📁 Kubernetes
   └─ Cluster Overview
   └─ Node Metrics
   └─ Pod Metrics
📁 Applications
   └─ Application 1
   └─ Application 2
📁 Alerting
   └─ Critical Alerts
   └─ Warnings
```

## Configuration du stockage persistant (Optionnel mais recommandé)

Par défaut, avec `emptyDir`, les configurations de Grafana seront perdues si le pod redémarre. Pour persister les données :

### Utilisation d'un PersistentVolumeClaim

Créez un fichier `grafana-pvc.yaml` :

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: grafana-pvc
  namespace: monitoring
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 5Gi
  storageClassName: microk8s-hostpath
```

Appliquez-le :

```bash
microk8s kubectl apply -f grafana-pvc.yaml
```

Ensuite, modifiez votre Deployment Grafana pour utiliser ce PVC au lieu de `emptyDir` :

```yaml
volumes:
- name: grafana-storage
  persistentVolumeClaim:
    claimName: grafana-pvc
```

## Vérification de l'installation

Pour vérifier que tout fonctionne correctement :

### 1. Vérifiez que le pod Grafana est en cours d'exécution

```bash
microk8s kubectl get pods -n prometheus
# ou
microk8s kubectl get pods -n monitoring
```

Résultat attendu : `STATUS = Running`, `READY = 1/1`

### 2. Vérifiez les logs Grafana

```bash
microk8s kubectl logs -n prometheus deployment/grafana
```

Vous ne devriez voir aucune erreur critique.

### 3. Testez la connexion à Prometheus

Dans l'interface Grafana, allez dans **Explore** et exécutez une requête simple comme `up`. Si vous voyez des résultats, tout fonctionne !

## Résolution des problèmes courants

### Problème : Impossible de se connecter à Grafana

**Causes possibles :**

1. Le port-forwarding n'est pas actif
   - **Solution :** Relancez la commande `kubectl port-forward`

2. Le pod Grafana n'est pas en cours d'exécution
   - **Vérification :** `microk8s kubectl get pods -n prometheus`
   - **Solution :** Redémarrez le pod si nécessaire

### Problème : "Data source is not working" lors du test Prometheus

**Causes possibles :**

1. URL incorrecte
   - **Solution :** Vérifiez le nom du service et le namespace avec `kubectl get svc -n prometheus`

2. Prometheus n'est pas en cours d'exécution
   - **Vérification :** `microk8s kubectl get pods -n prometheus`

3. Problème réseau entre Grafana et Prometheus
   - **Test :** Depuis un pod Grafana, testez la connectivité :
     ```bash
     microk8s kubectl exec -n prometheus deployment/grafana -- wget -O- http://prometheus-k8s.prometheus.svc.cluster.local:9090/api/v1/query?query=up
     ```

### Problème : Grafana est lent ou ne répond pas

**Causes possibles :**

1. Ressources insuffisantes
   - **Solution :** Augmentez les ressources (CPU/RAM) allouées au pod Grafana

2. Trop de dashboards ou requêtes complexes
   - **Solution :** Optimisez vos requêtes PromQL

## Configuration de sécurité de base

### Changement du mot de passe admin

Si vous ne l'avez pas fait lors de la première connexion :

1. **Cliquez sur votre profil** (en bas à gauche)
2. **Change password**
3. Entrez l'ancien et le nouveau mot de passe

### Désactivation de l'inscription publique

Par défaut, Grafana peut permettre aux utilisateurs de s'inscrire. Pour désactiver cela (recommandé pour un lab) :

Modifiez les variables d'environnement dans le Deployment :

```yaml
env:
- name: GF_USERS_ALLOW_SIGN_UP
  value: "false"
```

### Activation de HTTPS (Optionnel)

Pour sécuriser la connexion à Grafana, vous pouvez configurer HTTPS via un Ingress avec Cert-Manager (voir chapitre 11).

## Prochaines étapes

Maintenant que Grafana est installé et configuré, vous êtes prêt pour :

- **Chapitre 13.2 :** Connexion Grafana-Prometheus (approfondir la configuration)
- **Chapitre 13.3 :** Importer et utiliser des dashboards Kubernetes pré-configurés
- **Chapitre 13.4 :** Créer vos propres dashboards personnalisés

## Récapitulatif

Dans ce chapitre, vous avez appris à :

✅ Comprendre le rôle de Grafana dans l'écosystème de monitoring
✅ Installer Grafana sur MicroK8s (trois méthodes)
✅ Accéder à l'interface web de Grafana
✅ Configurer Prometheus comme source de données
✅ Effectuer les configurations de base de Grafana
✅ Organiser vos dashboards avec des dossiers
✅ Résoudre les problèmes courants
✅ Sécuriser votre installation Grafana

Grafana est maintenant opérationnel et prêt à visualiser les métriques de votre cluster Kubernetes !

---

**Note pour les débutants :** Ne vous inquiétez pas si tout ne fait pas encore sens immédiatement. Au fur et à mesure que vous utiliserez Grafana et créerez des dashboards, les concepts deviendront plus clairs. L'important est d'avoir une installation fonctionnelle pour commencer à explorer.

⏭️ [Connexion Grafana-Prometheus](/13-visualisation-avec-grafana/02-connexion-grafana-prometheus.md)
