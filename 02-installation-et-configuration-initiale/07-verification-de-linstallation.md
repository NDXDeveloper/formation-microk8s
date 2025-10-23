🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 2.7 Vérification de l'installation

## Introduction

Après avoir installé MicroK8s, il est essentiel de vérifier que tout fonctionne correctement avant de commencer à déployer des applications. Cette étape permet de s'assurer que :

- Le cluster Kubernetes est opérationnel
- Tous les composants système sont en bonne santé
- La communication réseau fonctionne
- Vous pouvez interagir avec le cluster

Cette section vous guidera à travers les différentes vérifications à effectuer, en expliquant ce que chaque commande fait et comment interpréter les résultats.

## Vérification du Statut Global

### Commande de base

La première commande à exécuter est :

```bash
microk8s status
```

Cette commande affiche l'état général de votre installation MicroK8s. Vous devriez voir une sortie similaire à :

```
microk8s is running
high-availability: no
  datastore master nodes: 127.0.0.1:19001
  datastore standby nodes: none
addons:
  enabled:
    ha-cluster           # (core) Configure high availability on the current node
  disabled:
    dashboard            # (core) The Kubernetes dashboard
    dns                  # (core) CoreDNS
    helm                 # (core) Helm - the package manager for Kubernetes
    ingress              # (core) Ingress controller for external access
    ...
```

**Interprétation** :

- **`microk8s is running`** : Votre cluster est actif et fonctionne
- **`high-availability: no`** : Vous êtes en mode single-node (normal pour un démarrage)
- **`addons`** : Liste des fonctionnalités additionnelles (nous y reviendrons plus tard)

### Attendre que le cluster soit prêt

Si vous venez juste de démarrer MicroK8s, il peut être nécessaire d'attendre que tous les services soient prêts :

```bash
microk8s status --wait-ready
```

Cette commande attend que le cluster soit complètement opérationnel avant de retourner le statut. C'est particulièrement utile dans les scripts d'automatisation ou juste après le démarrage.

**Timeout par défaut** : La commande attend jusqu'à 30 secondes. Si le cluster n'est pas prêt après ce délai, elle affiche un message d'erreur.

## Vérification des Nœuds

### Lister les nœuds du cluster

Un cluster Kubernetes est composé d'un ou plusieurs nœuds (serveurs). Pour vérifier que votre nœud est bien enregistré et opérationnel :

```bash
microk8s kubectl get nodes
```

Sortie attendue :

```
NAME               STATUS   ROLES    AGE   VERSION
votre-hostname     Ready    <none>   10m   v1.28.3
```

**Explication des colonnes** :

- **NAME** : Nom de votre nœud (généralement le hostname de votre machine)
- **STATUS** : État du nœud
  - `Ready` : Le nœud est opérationnel (c'est ce que vous voulez voir)
  - `NotReady` : Le nœud a un problème
- **ROLES** : Rôle du nœud (peut être vide pour MicroK8s en mode simple)
- **AGE** : Depuis combien de temps le nœud existe
- **VERSION** : Version de Kubernetes

### Obtenir des détails sur un nœud

Pour voir plus d'informations sur votre nœud :

```bash
microk8s kubectl describe node
```

Cette commande affiche une grande quantité d'informations, incluant :

- L'architecture (CPU, OS)
- Les ressources disponibles (CPU, mémoire, stockage)
- Les pods en cours d'exécution
- Les événements récents

Pour les débutants, la partie la plus importante est la section **Conditions** qui indique la santé du nœud :

```
Conditions:
  Type             Status
  ----             ------
  MemoryPressure   False
  DiskPressure     False
  PIDPressure      False
  Ready            True
```

Tous les états doivent être `False` sauf `Ready` qui doit être `True`.

## Vérification des Composants Système

### Les pods système

Kubernetes lui-même utilise des pods pour faire fonctionner ses composants internes. Vérifions qu'ils fonctionnent tous correctement :

```bash
microk8s kubectl get pods -n kube-system
```

L'option `-n kube-system` indique que nous voulons voir les pods dans le namespace (espace de noms) système de Kubernetes.

Sortie typique :

```
NAME                                      READY   STATUS    RESTARTS   AGE
calico-kube-controllers-77bd7c5b-8xk9m   1/1     Running   0          15m
calico-node-xbz4p                        1/1     Running   0          15m
```

**Explication des colonnes** :

- **NAME** : Nom du pod
- **READY** : Nombre de conteneurs prêts / nombre total de conteneurs
  - `1/1` signifie que tous les conteneurs du pod sont prêts
  - `0/1` ou `1/2` indiquerait un problème
- **STATUS** : État du pod
  - `Running` : Le pod fonctionne normalement (parfait !)
  - `Pending` : Le pod attend de démarrer (normal pendant quelques secondes)
  - `CrashLoopBackOff` : Le pod redémarre en boucle (problème !)
  - `Error` : Le pod a rencontré une erreur
- **RESTARTS** : Nombre de fois que le pod a redémarré
  - `0` est idéal
  - Un nombre élevé (>5) peut indiquer un problème
- **AGE** : Depuis combien de temps le pod existe

### Les services système

Vérifions maintenant les services Kubernetes :

```bash
microk8s kubectl get services -n kube-system
```

Sortie attendue :

```
NAME       TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)                  AGE
kube-dns   ClusterIP   10.152.183.10   <none>        53/UDP,53/TCP,9153/TCP   20m
```

Vous devriez au minimum voir le service `kube-dns` (ou `coredns` selon la version), qui gère la résolution DNS à l'intérieur du cluster.

## Vérification de la Version

### Version de MicroK8s et Kubernetes

Pour connaître les versions installées :

```bash
microk8s version
```

Sortie exemple :

```
MicroK8s v1.28.3 revision 5891
```

Vous pouvez également vérifier la version du serveur Kubernetes :

```bash
microk8s kubectl version --short
```

Sortie exemple :

```
Client Version: v1.28.3
Server Version: v1.28.3
```

**Note** : Il est normal que les versions client et serveur soient identiques dans MicroK8s.

## Vérification de la Connectivité

### Test de communication avec l'API Kubernetes

Pour vérifier que vous pouvez communiquer avec l'API Kubernetes :

```bash
microk8s kubectl cluster-info
```

Sortie attendue :

```
Kubernetes control plane is running at https://127.0.0.1:16443

To further debug and diagnose cluster problems, use 'kubectl cluster-info dump'.
```

Cela confirme que :
- L'API Kubernetes est accessible
- Elle écoute sur le port 16443 (port par défaut de MicroK8s)

### Test avec un pod de diagnostic

Créons un pod simple pour tester que Kubernetes peut bien démarrer des conteneurs :

```bash
microk8s kubectl run test-nginx --image=nginx --restart=Never
```

Cette commande crée un pod nommé `test-nginx` utilisant l'image nginx.

Vérifiez que le pod démarre correctement :

```bash
microk8s kubectl get pod test-nginx
```

Vous devriez voir :

```
NAME         READY   STATUS    RESTARTS   AGE
test-nginx   1/1     Running   0          30s
```

Si le statut est `Running`, félicitations ! Kubernetes peut bien créer et exécuter des pods.

**Nettoyage** : Supprimez le pod de test :

```bash
microk8s kubectl delete pod test-nginx
```

## Inspection Approfondie avec microk8s inspect

MicroK8s dispose d'un outil de diagnostic intégré très puissant :

```bash
microk8s inspect
```

Cette commande effectue une série de vérifications automatiques et génère un rapport complet. Elle vérifie :

- La configuration réseau
- Les certificats
- Les services système
- Les logs
- La configuration des addons

**Sortie** : La commande génère un fichier de rapport (généralement dans `/var/snap/microk8s/common/`). Le chemin exact est affiché à la fin de l'exécution.

**Utilisation** : Cette commande est particulièrement utile pour le dépannage ou pour partager des informations de diagnostic avec la communauté si vous rencontrez un problème.

Pour voir le rapport immédiatement sans le sauvegarder dans un fichier :

```bash
microk8s inspect | less
```

L'outil `less` permet de naviguer dans le rapport avec les flèches haut/bas. Appuyez sur `q` pour quitter.

## Vérification de la Configuration kubectl

### Afficher la configuration

MicroK8s gère automatiquement la configuration de kubectl. Pour voir cette configuration :

```bash
microk8s kubectl config view
```

Cette commande affiche :
- L'adresse du serveur API
- Les certificats utilisés pour l'authentification
- Le contexte actuel (quel cluster vous utilisez)

**Note pour les débutants** : Ne vous inquiétez pas si vous ne comprenez pas tout dans cette sortie. L'important est qu'elle existe et ne contienne pas d'erreurs.

### Vérifier le contexte actuel

```bash
microk8s kubectl config current-context
```

Cela affiche le contexte actuellement utilisé, généralement `microk8s`.

## Vérification des Ressources Système

### Ressources disponibles

Pour voir les ressources (CPU, mémoire) disponibles dans votre cluster :

```bash
microk8s kubectl top nodes
```

**Note** : Cette commande peut ne pas fonctionner immédiatement car elle nécessite le Metrics Server. Si vous obtenez une erreur, ce n'est pas grave pour l'instant. Nous verrons comment activer cette fonctionnalité dans les sections suivantes.

### Espace disque

Vérifiez que vous avez suffisamment d'espace disque :

**Sur Linux** :

```bash
df -h /var/snap/microk8s
```

**Sur macOS** (dans la VM) :

```bash
multipass exec microk8s-vm -- df -h
```

Assurez-vous d'avoir au moins 10-15 Go d'espace libre pour fonctionner confortablement.

## Tests de Santé Complets

### Vérification complète en une seule commande

Voici un script simple qui effectue toutes les vérifications importantes :

```bash
#!/bin/bash

echo "=== Vérification du statut MicroK8s ==="
microk8s status

echo -e "\n=== Vérification des nœuds ==="
microk8s kubectl get nodes

echo -e "\n=== Vérification des pods système ==="
microk8s kubectl get pods -n kube-system

echo -e "\n=== Vérification des services système ==="
microk8s kubectl get services -n kube-system

echo -e "\n=== Information du cluster ==="
microk8s kubectl cluster-info

echo -e "\n=== Version ==="
microk8s version

echo -e "\n✅ Vérification terminée !"
```

Vous pouvez sauvegarder ce script dans un fichier `verify-microk8s.sh`, le rendre exécutable et le lancer :

```bash
chmod +x verify-microk8s.sh
./verify-microk8s.sh
```

## Interpréter les Résultats

### Tout va bien si...

✅ `microk8s status` affiche "microk8s is running"
✅ Tous les nœuds sont en état `Ready`
✅ Tous les pods système sont en état `Running` avec READY à `1/1`
✅ Le nombre de RESTARTS est faible (0-2)
✅ `kubectl cluster-info` retourne l'adresse de l'API
✅ Vous pouvez créer et supprimer un pod de test sans erreur

### Problèmes courants

#### MicroK8s ne démarre pas

**Symptôme** : `microk8s status` indique que MicroK8s n'est pas en cours d'exécution

**Solutions** :
- Essayez de démarrer explicitement : `microk8s start`
- Vérifiez les logs : `microk8s inspect`
- Sur macOS, vérifiez que la VM Multipass fonctionne : `multipass list`

#### Nœud en état NotReady

**Symptôme** : `kubectl get nodes` montre le nœud avec STATUS `NotReady`

**Solutions** :
- Attendez quelques minutes, le nœud peut encore initialiser
- Vérifiez les pods système : il y a probablement un pod en erreur
- Redémarrez MicroK8s : `microk8s stop` puis `microk8s start`

#### Pods en CrashLoopBackOff

**Symptôme** : Des pods système sont en erreur et redémarrent constamment

**Solutions** :
- Vérifiez les logs du pod : `microk8s kubectl logs <nom-du-pod> -n kube-system`
- Vérifiez les ressources disponibles (RAM, CPU)
- Consultez `microk8s inspect` pour plus de détails

#### Impossible de contacter l'API

**Symptôme** : Les commandes kubectl retournent des erreurs de connexion

**Solutions** :
- Vérifiez que MicroK8s est bien démarré : `microk8s status`
- Sur Linux, vérifiez le pare-feu : le port 16443 doit être accessible
- Réinitialisez la configuration : `microk8s kubectl config view`

## Commandes de Diagnostic Supplémentaires

### Voir les événements du cluster

Les événements Kubernetes enregistrent tout ce qui se passe dans le cluster :

```bash
microk8s kubectl get events --all-namespaces --sort-by='.lastTimestamp'
```

Cette commande affiche les événements récents, triés par date. C'est très utile pour comprendre ce qui s'est passé récemment dans le cluster.

### Voir les logs d'un composant système

Si un composant système pose problème, consultez ses logs :

```bash
microk8s kubectl logs <nom-du-pod> -n kube-system
```

Remplacez `<nom-du-pod>` par le nom du pod que vous souhaitez inspecter (obtenu avec `get pods`).

### Vérification des certificats

Les certificats sont cruciaux pour la sécurité de Kubernetes. Pour vérifier leur état :

```bash
microk8s inspect | grep -i certificate
```

## Vérifications Spécifiques par Système d'Exploitation

### Linux

Sur Linux, vérifiez également :

```bash
# Vérifier que les services snap sont actifs
systemctl list-units --type=service | grep snap.microk8s

# Vérifier les permissions
ls -la /var/snap/microk8s/common/
```

### macOS

Sur macOS, vérifiez la VM Multipass :

```bash
# État de la VM
multipass list

# Informations sur la VM MicroK8s
multipass info microk8s-vm

# Ressources allouées
multipass get local.microk8s-vm.cpus
multipass get local.microk8s-vm.memory
```

### Windows (WSL2)

Sur Windows avec WSL2 :

```bash
# Vérifier WSL
wsl --status

# Vérifier la distribution utilisée
wsl --list --verbose

# Dans WSL, vérifier systemd
systemctl status snapd
```

## Checklist de Vérification

Utilisez cette checklist pour vous assurer que tout fonctionne :

- [ ] `microk8s status` affiche "running"
- [ ] Nœud(s) en état `Ready`
- [ ] Tous les pods système en état `Running`
- [ ] Aucun pod avec un nombre élevé de restarts
- [ ] `kubectl cluster-info` fonctionne
- [ ] Capable de créer un pod de test
- [ ] Capable de supprimer un pod de test
- [ ] `microk8s inspect` ne montre aucune erreur critique
- [ ] Suffisamment d'espace disque disponible
- [ ] Version de Kubernetes affichée correctement

Si tous ces éléments sont cochés, votre installation MicroK8s est validée et prête à l'emploi ! ✅

## Documenter Votre Installation

Il est recommandé de noter quelques informations sur votre installation :

- **Date d'installation** : _______________
- **Version de MicroK8s** : _______________
- **Version de Kubernetes** : _______________
- **Système d'exploitation** : _______________
- **Ressources allouées** :
  - CPU : _______________
  - RAM : _______________
  - Stockage : _______________

Ces informations seront utiles pour :
- Le dépannage futur
- Les mises à jour
- Le partage avec la communauté si vous avez besoin d'aide

## Prochaines Étapes

Maintenant que votre installation MicroK8s est vérifiée et fonctionnelle, vous êtes prêt à :

1. **Section 2.8** : Découvrir les commandes MicroK8s essentielles
2. **Section 2.9** : Configurer les alias kubectl pour plus de confort
3. **Chapitre 3** : Apprendre les concepts Kubernetes essentiels
4. **Chapitre 4** : Réaliser vos premiers déploiements

Félicitations ! Vous avez maintenant un cluster Kubernetes pleinement vérifié et opérationnel. Vous êtes prêt pour l'aventure Kubernetes ! 🚀

⏭️ [Commandes MicroK8s essentielles (status, inspect, enable)](/02-installation-et-configuration-initiale/08-commandes-microk8s-essentielles.md)
