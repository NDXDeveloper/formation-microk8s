🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 11.2 Installation de Cert-Manager (microk8s enable cert-manager)

## La philosophie MicroK8s : simplicité avant tout

L'un des grands avantages de MicroK8s par rapport à d'autres distributions Kubernetes est la présence d'**addons** préconfigurés. Au lieu de passer des heures à installer et configurer manuellement Cert-Manager (télécharger des fichiers YAML, appliquer des manifestes complexes, configurer des CRD...), MicroK8s vous permet de l'activer en une seule commande.

C'est exactement pour ce type de cas d'usage que MicroK8s est idéal pour un lab personnel ou un environnement de développement : vous pouvez vous concentrer sur l'apprentissage et l'utilisation des outils plutôt que sur leur installation complexe.

## Prérequis

Avant d'installer Cert-Manager, assurez-vous que :

1. **MicroK8s est installé et fonctionnel**
   ```bash
   microk8s status
   ```
   Vous devriez voir que MicroK8s est en cours d'exécution.

2. **Le DNS est activé** (généralement activé par défaut)
   ```bash
   microk8s status | grep dns
   ```
   Si le DNS n'est pas activé, activez-le :
   ```bash
   microk8s enable dns
   ```

3. **Vous avez configuré l'alias kubectl** (recommandé pour plus de confort)
   ```bash
   alias kubectl='microk8s kubectl'
   ```
   Ou ajoutez cette ligne à votre `~/.bashrc` ou `~/.zshrc` pour la rendre permanente.

## Installation avec l'addon MicroK8s

### La commande magique

L'installation de Cert-Manager se fait en une seule commande :

```bash
microk8s enable cert-manager
```

C'est tout ! Lancez cette commande et observez ce qui se passe.

### Que se passe-t-il en coulisses ?

Lorsque vous exécutez cette commande, MicroK8s effectue automatiquement plusieurs opérations :

1. **Téléchargement des manifestes** : récupération des fichiers de déploiement officiels de Cert-Manager
2. **Création d'un namespace** : création d'un namespace dédié `cert-manager`
3. **Installation des CRD** : déploiement des Custom Resource Definitions nécessaires
4. **Déploiement des composants** : installation des contrôleurs et webhooks Cert-Manager
5. **Configuration des permissions** : mise en place du RBAC approprié

Tout cela se fait automatiquement, sans que vous ayez à vous en soucier.

### Durée de l'installation

L'installation prend généralement entre **30 secondes et 2 minutes**, selon :
- La vitesse de votre connexion Internet (téléchargement des images Docker)
- Les ressources disponibles sur votre machine
- La charge actuelle de votre cluster

Vous verrez des messages s'afficher pendant l'installation, indiquant la progression.

### Sortie typique de la commande

Voici à quoi ressemble une installation réussie :

```
Infer repository core for addon cert-manager
Enabling cert-manager
Deploying cert-manager
namespace/cert-manager created
customresourcedefinition.apiextensions.k8s.io/certificaterequests.cert-manager.io created
customresourcedefinition.apiextensions.k8s.io/certificates.cert-manager.io created
customresourcedefinition.apiextensions.k8s.io/challenges.acme.cert-manager.io created
customresourcedefinition.apiextensions.k8s.io/clusterissuers.cert-manager.io created
customresourcedefinition.apiextensions.k8s.io/issuers.cert-manager.io created
customresourcedefinition.apiextensions.k8s.io/orders.acme.cert-manager.io created
serviceaccount/cert-manager-cainjector created
serviceaccount/cert-manager created
serviceaccount/cert-manager-webhook created
...
deployment.apps/cert-manager-cainjector created
deployment.apps/cert-manager created
deployment.apps/cert-manager-webhook created
...
Waiting for cert-manager to be ready.
Cert-manager is ready.
Enabled cert-manager
```

Si vous voyez le message final **"Enabled cert-manager"**, félicitations ! L'installation est terminée avec succès.

## Vérification de l'installation

### 1. Vérifier le statut de l'addon

Vérifiez que l'addon est bien activé :

```bash
microk8s status
```

Dans la liste des addons, vous devriez voir :

```
cert-manager: enabled
```

### 2. Vérifier le namespace cert-manager

Cert-Manager s'installe dans son propre namespace. Vérifions qu'il existe :

```bash
kubectl get namespaces
```

Vous devriez voir le namespace `cert-manager` dans la liste :

```
NAME              STATUS   AGE
default           Active   10d
kube-system       Active   10d
kube-public       Active   10d
kube-node-lease   Active   10d
cert-manager      Active   2m
```

### 3. Vérifier les pods Cert-Manager

Les pods de Cert-Manager doivent être en cours d'exécution. Vérifions-les :

```bash
kubectl get pods -n cert-manager
```

Vous devriez voir trois pods avec le statut `Running` :

```
NAME                                       READY   STATUS    RESTARTS   AGE
cert-manager-7d4c5d8d9f-xk2lp             1/1     Running   0          2m
cert-manager-cainjector-5c5695b8f7-9nhqc  1/1     Running   0          2m
cert-manager-webhook-5f57f59ffc-kq8zt     1/1     Running   0          2m
```

**Note** : les noms exacts des pods seront différents sur votre système (les suffixes aléatoires changent), mais vous devriez avoir ces trois types de pods.

Si les pods sont en état `ContainerCreating` ou `Pending`, patientez quelques instants supplémentaires, le téléchargement des images Docker peut prendre du temps.

### 4. Vérifier les CRD (Custom Resource Definitions)

Cert-Manager introduit de nouvelles ressources Kubernetes personnalisées. Vérifions qu'elles sont bien installées :

```bash
kubectl get crd | grep cert-manager
```

Vous devriez voir plusieurs CRD :

```
certificaterequests.cert-manager.io
certificates.cert-manager.io
challenges.acme.cert-manager.io
clusterissuers.cert-manager.io
issuers.cert-manager.io
orders.acme.cert-manager.io
```

Ces ressources personnalisées sont ce qui permet à Cert-Manager de fonctionner. Nous les utiliserons dans les prochaines sections.

### 5. Vérifier les services

Cert-Manager crée également des services Kubernetes :

```bash
kubectl get services -n cert-manager
```

Vous devriez voir :

```
NAME                   TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)    AGE
cert-manager           ClusterIP   10.152.183.101   <none>        9402/TCP   3m
cert-manager-webhook   ClusterIP   10.152.183.102   <none>        443/TCP    3m
```

### 6. Vérifier les logs (optionnel)

Si vous voulez voir ce que fait Cert-Manager, vous pouvez consulter ses logs :

```bash
kubectl logs -n cert-manager deployment/cert-manager
```

Vous devriez voir des logs indiquant que le contrôleur démarre correctement, sans erreurs critiques.

## Comprendre les composants installés

Cert-Manager est composé de trois composants principaux qui travaillent ensemble :

### 1. cert-manager (le contrôleur principal)

C'est le cerveau de Cert-Manager. Il :
- Surveille les ressources Certificate, Issuer et ClusterIssuer
- Gère le cycle de vie des certificats (création, renouvellement)
- Communique avec les autorités de certification (Let's Encrypt, etc.)
- Met à jour les Secrets Kubernetes avec les certificats obtenus

### 2. cert-manager-webhook

Le webhook valide et transforme les ressources Cert-Manager avant qu'elles ne soient créées dans le cluster. C'est un composant de sécurité qui :
- Valide la syntaxe des ressources
- Applique des valeurs par défaut
- Empêche la création de ressources mal configurées

### 3. cert-manager-cainjector

Le CA Injector injecte automatiquement les certificats d'autorité de certification dans les ressources Kubernetes qui en ont besoin, notamment :
- Les webhooks personnalisés
- Les API Services
- Les ressources qui nécessitent une validation TLS

**Pour un débutant**, vous n'avez pas besoin de vous préoccuper des détails de ces composants. Sachez simplement qu'ils travaillent ensemble pour automatiser la gestion de vos certificats.

## Version de Cert-Manager installée

Pour connaître la version de Cert-Manager installée par MicroK8s :

```bash
kubectl get deployment cert-manager -n cert-manager -o jsonpath='{.spec.template.spec.containers[0].image}'
```

Cela affichera quelque chose comme :

```
quay.io/jetstack/cert-manager-controller:v1.13.3
```

Le numéro de version (ici `v1.13.3`) vous indique quelle version de Cert-Manager est déployée. MicroK8s utilise généralement une version stable et récente.

## Namespace cert-manager : un espace isolé

Cert-Manager fonctionne dans son propre **namespace** (`cert-manager`), ce qui signifie :

- **Isolation** : ses ressources sont séparées de vos applications
- **Sécurité** : vous pouvez limiter l'accès à ce namespace via RBAC
- **Organisation** : tous les composants de Cert-Manager sont regroupés au même endroit
- **Facilité de gestion** : vous pouvez inspecter ou supprimer Cert-Manager sans affecter vos autres applications

Cependant, Cert-Manager peut gérer des certificats pour **tous les namespaces** de votre cluster, pas seulement le sien. C'est son rôle : être un service centralisé de gestion de certificats.

## Désactivation de l'addon (si nécessaire)

Si pour une raison quelconque vous souhaitez désactiver Cert-Manager, utilisez :

```bash
microk8s disable cert-manager
```

Cela supprimera tous les composants de Cert-Manager. **Attention** : si vous avez des certificats gérés par Cert-Manager, ils cesseront d'être renouvelés automatiquement après la désactivation.

Vous pourrez toujours réactiver l'addon plus tard avec `microk8s enable cert-manager`.

## Dépannage courant

### Les pods ne démarrent pas (status: Pending ou ContainerCreating)

**Causes possibles** :
- Téléchargement des images Docker en cours (patientez)
- Ressources système insuffisantes
- Problème de réseau

**Solution** :
1. Attendez quelques minutes
2. Vérifiez les événements du pod :
   ```bash
   kubectl describe pod <nom-du-pod> -n cert-manager
   ```
3. Vérifiez les logs :
   ```bash
   kubectl logs <nom-du-pod> -n cert-manager
   ```

### Erreur "webhook is not ready"

Si vous voyez des erreurs concernant le webhook juste après l'installation, c'est normal. Le webhook peut prendre quelques secondes à se stabiliser. Patientez 30-60 secondes et réessayez.

### Les CRD ne sont pas créées

Si `kubectl get crd | grep cert-manager` ne retourne rien, essayez de désactiver puis réactiver l'addon :

```bash
microk8s disable cert-manager
microk8s enable cert-manager
```

## Ressources créées par l'addon

Voici un récapitulatif de tout ce qui a été créé dans votre cluster :

| Type de ressource | Nombre | Emplacement | Description |
|-------------------|--------|-------------|-------------|
| Namespace | 1 | - | `cert-manager` |
| Deployment | 3 | cert-manager | Les trois composants principaux |
| Service | 2 | cert-manager | Services pour le contrôleur et le webhook |
| ServiceAccount | 3 | cert-manager | Comptes de service pour les pods |
| ClusterRole | Plusieurs | - | Permissions au niveau du cluster |
| ClusterRoleBinding | Plusieurs | - | Liaisons des rôles aux comptes de service |
| CustomResourceDefinition | 6 | - | Nouvelles ressources Kubernetes |
| ValidatingWebhookConfiguration | 1 | - | Configuration du webhook de validation |
| MutatingWebhookConfiguration | 1 | - | Configuration du webhook de mutation |

Toutes ces ressources travaillent ensemble pour fournir la fonctionnalité de gestion automatique des certificats.

## Consommation de ressources

Cert-Manager est relativement léger. Voici la consommation typique :

- **CPU** : ~10-50m (millicores) au repos, peut augmenter lors du traitement de certificats
- **Mémoire** : ~150-300 MB au total pour les trois pods
- **Stockage** : négligeable (quelques MB pour les images Docker)

Ces ressources sont largement acceptables même pour un lab personnel modeste.

## Pourquoi trois pods ?

Vous vous demandez peut-être pourquoi Cert-Manager nécessite trois pods différents. Voici une analogie simple :

Imaginez une chaîne de montage pour fabriquer des voitures :
- **Un ouvrier** (cert-manager) assemble la voiture
- **Un contrôleur qualité** (webhook) vérifie que tout est conforme avant l'assemblage
- **Un magasinier** (cainjector) distribue les pièces aux bons endroits

Chaque pod a un rôle spécifique et complémentaire. Ensemble, ils forment un système robuste et sécurisé.

## Vérification finale : test de disponibilité

Pour être sûr que Cert-Manager est prêt à être utilisé, exécutez cette commande :

```bash
kubectl get deployment -n cert-manager
```

Les trois deployments doivent afficher `READY 1/1` et `AVAILABLE 1` :

```
NAME                      READY   UP-TO-DATE   AVAILABLE   AGE
cert-manager              1/1     1            1           5m
cert-manager-cainjector   1/1     1            1           5m
cert-manager-webhook      1/1     1            1           5m
```

Si c'est le cas, **Cert-Manager est opérationnel** et prêt à gérer vos certificats !

## Ce que vous pouvez faire maintenant

Maintenant que Cert-Manager est installé, vous pouvez :

✅ Créer des **Issuers** pour définir comment obtenir des certificats
✅ Obtenir des **certificats Let's Encrypt** automatiquement
✅ Créer des **certificats auto-signés** pour le développement
✅ Gérer le **renouvellement automatique** des certificats
✅ Sécuriser vos **Ingress** avec HTTPS

Mais ne vous précipitez pas ! Nous allons découvrir tout cela pas à pas dans les prochaines sections.

## Concepts clés à retenir

- **Addon MicroK8s** : installation simplifiée en une commande
- **Trois composants** : cert-manager (contrôleur), webhook (validation), cainjector (injection)
- **Namespace dédié** : cert-manager fonctionne dans son propre namespace
- **CRD** : nouvelles ressources Kubernetes pour gérer les certificats
- **Léger** : consommation de ressources très raisonnable
- **Vérification** : toujours vérifier que les pods sont en état `Running`

## Prochaines étapes

Dans la section suivante, nous allons configurer Cert-Manager pour qu'il puisse communiquer avec Let's Encrypt et commencer à émettre des certificats. Vous allez découvrir les concepts d'**Issuer** et **ClusterIssuer**, qui sont au cœur du fonctionnement de Cert-Manager.

L'installation est terminée, et c'était la partie la plus facile ! Maintenant, passons à la configuration et à l'utilisation concrète.

---

⏭️ [Configuration de Cert-Manager](/11-certificats-ssl-tls/03-configuration-de-cert-manager.md)
