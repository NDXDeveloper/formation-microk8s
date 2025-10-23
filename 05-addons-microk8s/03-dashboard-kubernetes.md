🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 5.3 Dashboard Kubernetes (microk8s enable dashboard)

## Introduction

Le **Dashboard Kubernetes** est probablement le premier addon que vous allez vouloir activer dans MicroK8s. C'est une interface web graphique qui vous permet de visualiser et de gérer votre cluster Kubernetes sans avoir à taper des commandes dans un terminal. Pour les débutants, c'est un outil précieux qui aide à comprendre comment Kubernetes fonctionne de manière visuelle et intuitive.

## Qu'est-ce que le Dashboard Kubernetes ?

Le Dashboard Kubernetes est une **application web** qui s'exécute à l'intérieur de votre cluster et vous offre une vue complète de tout ce qui s'y passe. C'est comme le panneau de contrôle d'une voiture : au lieu de deviner ce qui se passe sous le capot, vous avez un tableau de bord avec des indicateurs clairs.

### Analogie simple

Imaginez que vous gérez un immeuble :

- **Sans Dashboard** : Vous devez monter et descendre tous les étages, frapper à chaque porte, noter sur papier qui habite où, vérifier manuellement chaque appartement, etc. C'est faisable, mais chronophage.

- **Avec Dashboard** : Vous avez un bureau central avec des écrans qui montrent en temps réel l'état de chaque appartement, qui est présent, quels équipements fonctionnent, les consommations, etc. Vous pouvez tout superviser d'un coup d'œil.

Le Dashboard Kubernetes fait exactement cela pour votre cluster : il vous donne une vue centralisée et visuelle de tout.

## Pourquoi utiliser le Dashboard ?

### Pour les débutants

Le Dashboard est particulièrement utile quand on débute avec Kubernetes :

1. **Visualisation** : Voir ses applications, pods, services représentés graphiquement aide énormément à comprendre les concepts abstraits
2. **Découverte** : Explorer l'interface permet de découvrir les différentes ressources Kubernetes de manière intuitive
3. **Débogage** : Identifier rapidement les problèmes grâce à des indicateurs visuels (rouge = problème, vert = OK)
4. **Apprentissage** : Compléter les commandes kubectl en voyant le résultat visuel immédiat

### Pour les utilisateurs avancés

Même les experts apprécient le Dashboard pour :

1. **Rapidité** : Vue d'ensemble rapide de l'état du cluster
2. **Monitoring** : Surveillance en temps réel des ressources
3. **Dépannage** : Accès rapide aux logs et événements
4. **Démonstrations** : Présenter Kubernetes à des non-techniciens

### Ce que vous pouvez faire avec le Dashboard

- **Voir** tous vos deployments, pods, services, etc.
- **Surveiller** l'utilisation des ressources (CPU, RAM)
- **Consulter** les logs des applications
- **Créer** de nouvelles ressources via des formulaires
- **Modifier** des configurations existantes
- **Supprimer** des ressources
- **Accéder** à un terminal dans vos conteneurs
- **Voir** les événements et les erreurs

## Activation du Dashboard

L'activation du Dashboard est extrêmement simple grâce à MicroK8s.

### Commande d'activation

```bash
microk8s enable dashboard
```

### Ce qui se passe lors de l'activation

Quand vous exécutez cette commande, MicroK8s va :

1. **Télécharger** les images Docker nécessaires
2. **Activer automatiquement** le Metrics Server (une dépendance)
3. **Créer** un namespace `kube-system` s'il n'existe pas déjà
4. **Déployer** le Dashboard et tous ses composants
5. **Configurer** les permissions et la sécurité
6. **Vérifier** que tout fonctionne correctement

**Sortie attendue :**
```
Infer repository core for addon dashboard
Enabling Kubernetes Dashboard
Infer repository core for addon metrics-server
Enabling Metrics-Server
serviceaccount/metrics-server created
clusterrole.rbac.authorization.k8s.io/system:aggregated-metrics-reader created
clusterrole.rbac.authorization.k8s.io/system:metrics-server created
rolebinding.rbac.authorization.k8s.io/metrics-server-auth-reader created
clusterrolebinding.rbac.authorization.k8s.io/metrics-server:system:auth-delegator created
clusterrolebinding.rbac.authorization.k8s.io/system:metrics-server created
service/metrics-server created
deployment.apps/metrics-server created
apiservice.apiregistration.k8s.io/v1beta1.metrics.k8s.io created
clusterrolebinding.rbac.authorization.k8s.io/microk8s-admin created
Metrics-Server is enabled
Applying manifest
serviceaccount/kubernetes-dashboard created
service/kubernetes-dashboard created
secret/kubernetes-dashboard-certs created
secret/kubernetes-dashboard-csrf created
secret/kubernetes-dashboard-key-holder created
configmap/kubernetes-dashboard-settings created
role.rbac.authorization.k8s.io/kubernetes-dashboard created
clusterrole.rbac.authorization.k8s.io/kubernetes-dashboard created
rolebinding.rbac.authorization.k8s.io/kubernetes-dashboard created
clusterrolebinding.rbac.authorization.k8s.io/kubernetes-dashboard created
deployment.apps/kubernetes-dashboard created
service/dashboard-metrics-scraper created
deployment.apps/dashboard-metrics-scraper created

If RBAC is not enabled access the dashboard using the token retrieved with:

microk8s kubectl describe secret -n kube-system microk8s-dashboard-token

Use this token in the https login UI of the kubernetes-dashboard service.

In an RBAC enabled setup (microk8s enable RBAC) you need to create a user with restricted
permissions as shown in:
https://github.com/kubernetes/dashboard/blob/master/docs/user/access-control/creating-sample-user.md

Kubernetes Dashboard is enabled
```

### Temps d'activation

L'activation du Dashboard prend généralement **30 secondes à 2 minutes**, selon votre connexion Internet pour le téléchargement initial des images.

### Vérification

Pour vérifier que le Dashboard est bien activé :

```bash
microk8s status
```

Vous devriez voir dans la liste des addons activés :
```
enabled:
  dashboard            # (core) The Kubernetes dashboard
```

### Vérifier que les pods sont prêts

```bash
microk8s kubectl get pods -n kube-system
```

Vous devriez voir quelque chose comme :
```
NAME                                         READY   STATUS    RESTARTS   AGE
kubernetes-dashboard-xxxxxxxxxx-xxxxx        1/1     Running   0          2m
dashboard-metrics-scraper-xxxxxxxxxx-xxxxx   1/1     Running   0          2m
metrics-server-xxxxxxxxxx-xxxxx              1/1     Running   0          2m
```

Tous les pods doivent être en état `Running` avec `1/1` dans la colonne `READY`.

## Accès au Dashboard

Une fois le Dashboard activé, vous devez y accéder via votre navigateur web. MicroK8s propose une commande simplifiée pour cela.

### Méthode 1 : Proxy automatique (Recommandée)

C'est la méthode la plus simple pour accéder au Dashboard :

```bash
microk8s dashboard-proxy
```

**Sortie :**
```
Checking if Dashboard is running.
Dashboard will be available at https://127.0.0.1:10443
Use the following token to login:
eyJhbGciOiJSUzI1NiIsImtpZCI6IkRFV0xKR3dFRjY3SThXcHBTZUtBUVVW...
```

Cette commande :
1. **Vérifie** que le Dashboard fonctionne
2. **Démarre** un proxy sécurisé
3. **Affiche** l'URL pour accéder au Dashboard
4. **Fournit** le token d'authentification

**Important :** Laissez cette commande en cours d'exécution dans votre terminal. Si vous fermez le terminal ou interrompez la commande (Ctrl+C), le proxy s'arrête et vous ne pourrez plus accéder au Dashboard.

### Ouvrir le Dashboard dans votre navigateur

1. **Copiez l'URL** affichée (généralement `https://127.0.0.1:10443`)
2. **Ouvrez** votre navigateur web (Chrome, Firefox, Edge, Safari, etc.)
3. **Collez** l'URL dans la barre d'adresse
4. **Acceptez** l'avertissement de sécurité (nous y reviendrons)

### Avertissement de sécurité

Quand vous accédez au Dashboard, votre navigateur affichera probablement un avertissement de sécurité disant que la connexion n'est pas privée ou que le certificat n'est pas valide.

**C'est normal !** Le Dashboard utilise un certificat auto-signé pour le HTTPS. Dans un environnement de développement local, c'est parfaitement acceptable.

**Comment contourner l'avertissement :**
- **Chrome** : Cliquez sur "Avancé" puis "Continuer vers le site (dangereux)"
- **Firefox** : Cliquez sur "Avancé" puis "Accepter le risque et poursuivre"
- **Edge** : Cliquez sur "Détails" puis "Accéder à la page web"
- **Safari** : Cliquez sur "Afficher les détails" puis "Visiter ce site web"

### Authentification avec le Token

Une fois sur la page du Dashboard, vous verrez un écran de connexion avec deux options :
1. **Kubeconfig** (fichier de configuration)
2. **Token** (jeton d'authentification)

**Choisissez l'option Token**, puis :

1. **Copiez** le long token affiché dans votre terminal (celui qui commence par `eyJhbGc...`)
2. **Collez-le** dans le champ "Enter token"
3. **Cliquez** sur "Sign in"

Et voilà ! Vous êtes dans le Dashboard Kubernetes.

### Méthode 2 : Port-forward manuel (Alternative)

Si la méthode du proxy automatique ne fonctionne pas, vous pouvez utiliser un port-forward manuel :

```bash
microk8s kubectl port-forward -n kube-system service/kubernetes-dashboard 10443:443
```

Puis accédez à `https://localhost:10443` dans votre navigateur.

Pour obtenir le token manuellement :

```bash
microk8s kubectl describe secret -n kube-system microk8s-dashboard-token
```

Cherchez la ligne qui commence par `token:` et copiez la longue chaîne de caractères.

## Interface du Dashboard

Une fois connecté, vous découvrez l'interface du Dashboard Kubernetes. Prenons le temps de comprendre ce que vous voyez.

### Vue d'ensemble (Overview)

Par défaut, vous arrivez sur la vue d'ensemble qui montre :

- **Workloads** (Charges de travail) : Vos applications en cours d'exécution
- **Services** : Les points d'accès réseau
- **Config and Storage** : Configurations et stockage
- **Namespace** : Le contexte actuel (en haut à droite)

### Navigation dans la barre latérale

La barre latérale gauche organise les ressources par catégories :

#### 1. Workloads (Charges de travail)
- **Deployments** : Vos applications déployées
- **ReplicaSets** : Ensembles de réplicas de pods
- **Pods** : Les conteneurs en cours d'exécution
- **Jobs** : Tâches ponctuelles
- **CronJobs** : Tâches planifiées
- **DaemonSets** : Services qui tournent sur chaque nœud
- **StatefulSets** : Applications avec état

#### 2. Discovery and Load Balancing
- **Services** : Points d'entrée réseau
- **Ingresses** : Règles de routage HTTP/HTTPS

#### 3. Config and Storage
- **ConfigMaps** : Configurations non sensibles
- **Secrets** : Données sensibles (mots de passe, tokens)
- **PersistentVolumeClaims** : Demandes de stockage

#### 4. Cluster
- **Namespaces** : Espaces de noms logiques
- **Nodes** : Serveurs physiques ou virtuels
- **Persistent Volumes** : Volumes de stockage
- **Storage Classes** : Classes de stockage
- **Roles** : Rôles de sécurité
- Et plus encore...

### Comprendre les couleurs et icônes

Le Dashboard utilise un code couleur intuitif :

- **Vert** ✓ : Tout va bien, la ressource est en état normal
- **Jaune** ⚠ : Avertissement, attention requise
- **Rouge** ✗ : Erreur, problème nécessitant intervention
- **Gris** ○ : En cours, en attente, ou désactivé

## Utiliser le Dashboard : Cas d'usage courants

### 1. Visualiser vos applications

**Navigation :** Workloads > Deployments

Vous voyez la liste de tous vos déploiements avec :
- Le **nom** du deployment
- Le **namespace** où il se trouve
- Le nombre de **pods** (ex: 2/2 = 2 pods souhaités, 2 pods en cours)
- L'**âge** du deployment
- Une **icône de statut** (vert = OK, rouge = problème)

**En cliquant sur un deployment**, vous accédez aux détails :
- Spécifications (image utilisée, ressources, variables d'environnement)
- Pods associés
- Événements récents
- Possibilité d'éditer, de scaler ou de supprimer

### 2. Inspecter un pod

**Navigation :** Workloads > Pods

La liste des pods vous montre :
- **Nom** du pod
- **Status** (Running, Pending, Failed, etc.)
- **Restarts** : Nombre de redémarrages (si élevé, il y a un problème)
- **CPU et Memory** : Utilisation des ressources
- **Node** : Sur quel serveur il tourne

**En cliquant sur un pod**, vous pouvez :
- **Voir les logs** : Bouton "Logs" en haut à droite
- **Ouvrir un terminal** : Bouton "Exec" pour un shell dans le conteneur
- **Voir les détails** : Configuration, état, événements
- **Supprimer le pod** : Si vous voulez le redémarrer

### 3. Consulter les logs

Les logs sont essentiels pour comprendre ce qui se passe dans vos applications.

**Pour voir les logs d'un pod :**
1. Allez dans Workloads > Pods
2. Cliquez sur le pod qui vous intéresse
3. Cliquez sur l'icône **"Logs"** en haut à droite (📄)

Vous verrez les sorties console de votre application en temps réel. C'est l'équivalent de `docker logs` mais pour Kubernetes.

**Astuces :**
- Utilisez le champ de recherche pour filtrer les logs
- Cochez "Auto-refresh" pour voir les nouveaux logs automatiquement
- Utilisez "Download" pour télécharger les logs

### 4. Surveiller les ressources

**Navigation :** Cluster > Nodes > [Cliquez sur votre nœud]

Vous verrez des graphiques montrant :
- **CPU** : Utilisation du processeur en pourcentage
- **Memory** : Utilisation de la RAM en Go
- **Pods** : Nombre de pods en cours d'exécution

Ces métriques sont fournies par le **Metrics Server** qui a été activé automatiquement avec le Dashboard.

### 5. Créer une ressource

Le Dashboard permet de créer des ressources de deux manières :

**Méthode 1 : Via formulaire**
- Cliquez sur le bouton **"+" (Create)** en haut à droite
- Choisissez "Create from form"
- Remplissez les champs (nom, image, ports, etc.)
- Cliquez sur "Deploy"

**Méthode 2 : Via YAML**
- Cliquez sur le bouton **"+" (Create)**
- Choisissez "Create from input"
- Collez votre manifeste YAML
- Cliquez sur "Upload"

Pour les débutants, le formulaire est plus simple. Pour les utilisateurs avancés, le YAML offre plus de contrôle.

### 6. Modifier une ressource

1. Trouvez la ressource (deployment, service, etc.)
2. Cliquez dessus pour voir les détails
3. Cliquez sur l'icône **"Edit"** (crayon) en haut à droite
4. Modifiez le YAML directement dans l'éditeur
5. Cliquez sur "Update"

**Attention :** Soyez prudent quand vous éditez du YAML. Une erreur de syntaxe peut casser votre application.

### 7. Scaler une application

Pour augmenter ou diminuer le nombre de réplicas d'un deployment :

1. Allez dans Workloads > Deployments
2. Cliquez sur votre deployment
3. Cliquez sur l'icône **"Scale"** (⚖) en haut à droite
4. Entrez le nouveau nombre de réplicas souhaité
5. Cliquez sur "Scale"

Vous verrez en temps réel de nouveaux pods se créer (ou des pods existants se terminer).

### 8. Supprimer une ressource

1. Trouvez la ressource à supprimer
2. Cliquez dessus
3. Cliquez sur l'icône **"Delete"** (poubelle) en haut à droite
4. Confirmez la suppression

**Attention :** La suppression est immédiate et irréversible. Assurez-vous de supprimer la bonne ressource !

## Comprendre les Namespaces dans le Dashboard

En haut à droite du Dashboard, vous verrez un **sélecteur de namespace**. Par défaut, il affiche souvent "default".

### Qu'est-ce qu'un namespace ?

Un namespace est comme un dossier qui organise vos ressources Kubernetes. C'est une façon de séparer logiquement différentes applications ou environnements.

### Namespaces courants

- **default** : Namespace par défaut pour vos applications
- **kube-system** : Composants système de Kubernetes (dont le Dashboard lui-même)
- **kube-public** : Ressources publiques accessibles à tous
- **kube-node-lease** : Informations de santé des nœuds

### Changer de namespace

Pour voir les ressources d'un autre namespace :
1. Cliquez sur le sélecteur de namespace en haut à droite
2. Sélectionnez le namespace désiré (ou "All namespaces" pour tout voir)

**Exemple :** Pour voir le Dashboard lui-même, sélectionnez le namespace "kube-system" et allez dans Workloads > Deployments. Vous verrez `kubernetes-dashboard`.

## Sécurité et Bonnes Pratiques

### Le Dashboard en production

**Important :** Le Dashboard Kubernetes tel qu'activé par MicroK8s est conçu pour le **développement et l'apprentissage**, pas pour la production.

En production, vous devriez :
- Utiliser des certificats SSL valides (pas auto-signés)
- Implémenter une authentification forte (OIDC, LDAP)
- Restreindre l'accès réseau au Dashboard
- Utiliser RBAC pour limiter les permissions
- Désactiver le Dashboard si vous ne l'utilisez pas

### Protéger votre token

Le token d'accès au Dashboard est une clé très puissante. Avec ce token, on peut contrôler tout le cluster.

**Bonnes pratiques :**
- **Ne partagez jamais** votre token
- **Ne le committez jamais** dans Git ou un système de contrôle de version
- **Régénérez-le** si vous pensez qu'il a été compromis
- **Fermez** votre session Dashboard quand vous avez fini

### Accès depuis l'extérieur (réseau local)

Par défaut, le Dashboard n'est accessible que depuis `localhost` (127.0.0.1). C'est une mesure de sécurité.

Si vous voulez y accéder depuis un autre ordinateur sur votre réseau local (par exemple, accéder au Dashboard de votre serveur depuis votre laptop), vous devrez :

1. **Modifier le proxy** pour écouter sur toutes les interfaces
2. **Configurer votre firewall** pour autoriser le port
3. **Utiliser l'adresse IP** de votre serveur au lieu de localhost

**Exemple :**
```bash
microk8s kubectl port-forward -n kube-system service/kubernetes-dashboard 10443:443 --address 0.0.0.0
```

Puis accédez à `https://[IP-de-votre-serveur]:10443`

**⚠️ Attention :** Cela expose le Dashboard sur votre réseau local. Assurez-vous de bien protéger votre réseau et votre token.

## Dépannage

### Le Dashboard ne démarre pas

**Symptôme :** Les pods restent en état "Pending" ou "CrashLoopBackOff"

**Solutions :**
1. Vérifiez les logs du pod :
   ```bash
   microk8s kubectl logs -n kube-system [nom-du-pod-dashboard]
   ```

2. Vérifiez les ressources disponibles :
   ```bash
   microk8s kubectl describe nodes
   ```

3. Réinstallez le Dashboard :
   ```bash
   microk8s disable dashboard
   microk8s enable dashboard
   ```

### Impossible de se connecter au Dashboard

**Symptôme :** Le navigateur affiche "Connection refused" ou "Unable to connect"

**Solutions :**
1. Vérifiez que le proxy est bien en cours d'exécution
2. Vérifiez l'URL (doit être `https://`, pas `http://`)
3. Essayez un autre navigateur
4. Vérifiez que le port n'est pas déjà utilisé :
   ```bash
   netstat -an | grep 10443
   ```

### Le token ne fonctionne pas

**Symptôme :** "Invalid token" après avoir collé le token

**Solutions :**
1. Assurez-vous d'avoir copié le token en entier (très long)
2. Vérifiez qu'il n'y a pas d'espaces au début ou à la fin
3. Générez un nouveau token :
   ```bash
   microk8s kubectl describe secret -n kube-system microk8s-dashboard-token
   ```

### Les métriques ne s'affichent pas

**Symptôme :** Pas de graphiques CPU/Memory dans le Dashboard

**Solution :** Le Metrics Server n'est peut-être pas prêt. Attendez quelques minutes, puis vérifiez :
```bash
microk8s kubectl get pods -n kube-system | grep metrics-server
```

Le pod doit être en état "Running". Si ce n'est pas le cas, consultez ses logs.

### Le Dashboard est lent

**Causes possibles :**
- Trop de ressources dans le cluster
- Ordinateur avec peu de RAM
- Problèmes réseau

**Solutions :**
- Filtrez par namespace au lieu de tout afficher
- Fermez les onglets de logs en temps réel
- Augmentez les ressources de votre machine si possible

## Alternatives et Compléments au Dashboard

Le Dashboard Kubernetes est excellent, mais il existe d'autres outils de visualisation :

### Lens (Gratuit)
Une application desktop qui offre une interface encore plus riche que le Dashboard web. Très populaire dans la communauté.

### K9s (Gratuit)
Une interface en ligne de commande interactive pour Kubernetes. Comme un Dashboard dans le terminal. Parfait si vous aimez le terminal mais voulez une interface plus conviviale que kubectl.

### Portainer (Freemium)
Une interface web complète pour gérer Docker et Kubernetes. Plus orientée vers la gestion multi-cluster.

### Octant (Gratuit)
Un outil de visualisation développé par VMware, plus axé sur le développement.

Cependant, pour débuter et pour un lab personnel, le Dashboard intégré à MicroK8s est largement suffisant et a l'avantage d'être déjà là, prêt à l'emploi.

## Récapitulatif

Le Dashboard Kubernetes est un outil essentiel pour :

✅ **Visualiser** votre cluster de manière intuitive
✅ **Surveiller** l'état de vos applications en temps réel
✅ **Déboguer** rapidement en accédant aux logs
✅ **Gérer** vos ressources via une interface graphique
✅ **Apprendre** Kubernetes de manière visuelle

**Commandes clés à retenir :**
```bash
# Activer le Dashboard
microk8s enable dashboard

# Accéder au Dashboard
microk8s dashboard-proxy

# Vérifier l'état
microk8s status

# Désactiver si nécessaire
microk8s disable dashboard
```

## Ce que vous avez appris

Dans cette section, vous avez découvert :
- Ce qu'est le Dashboard Kubernetes et pourquoi c'est utile
- Comment activer le Dashboard avec MicroK8s
- Comment accéder au Dashboard depuis votre navigateur
- Comment naviguer dans l'interface
- Les cas d'usage courants (visualiser, surveiller, déboguer)
- Les bonnes pratiques de sécurité
- Comment résoudre les problèmes courants

## Prochaine étape

Maintenant que vous avez une interface graphique pour votre cluster, il est temps d'activer une fonctionnalité encore plus fondamentale : le **DNS** (Domain Name System).

Le DNS est ce qui permet à vos applications de communiquer entre elles en utilisant des noms plutôt que des adresses IP. C'est un addon essentiel que nous allons découvrir dans la prochaine section (5.4).

Avant de passer à la suite, prenez le temps d'explorer le Dashboard. Naviguez dans les différentes sections, regardez ce qui s'y trouve. Plus vous serez familier avec cette interface, plus vous serez à l'aise avec Kubernetes !

---

**Astuce pratique :** Gardez un onglet du Dashboard ouvert pendant que vous travaillez sur Kubernetes. C'est très utile pour voir immédiatement l'effet de vos commandes kubectl. Vous tapez une commande dans le terminal, vous rafraîchissez le Dashboard, et vous voyez le changement. C'est une excellente façon d'apprendre !

⏭️ [DNS (CoreDNS - microk8s enable dns)](/05-addons-microk8s/04-dns-coredns.md)
