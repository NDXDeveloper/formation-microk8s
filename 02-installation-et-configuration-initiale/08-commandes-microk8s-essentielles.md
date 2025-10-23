🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 2.8 Commandes MicroK8s essentielles (status, inspect, enable)

## Introduction

MicroK8s propose un ensemble de commandes spécifiques qui simplifient grandement la gestion d'un cluster Kubernetes. Contrairement aux commandes `kubectl` qui interagissent avec l'API Kubernetes (et qui fonctionnent avec n'importe quelle distribution Kubernetes), les commandes `microk8s` sont spécifiques à MicroK8s et permettent de gérer le cluster lui-même.

Dans cette section, nous allons découvrir les commandes essentielles que vous utiliserez quotidiennement pour :
- Vérifier l'état de votre cluster
- Diagnostiquer les problèmes
- Activer des fonctionnalités additionnelles
- Démarrer et arrêter le cluster

Ces commandes sont conçues pour être simples et intuitives, même pour les débutants.

## Architecture des Commandes MicroK8s

Toutes les commandes MicroK8s suivent cette structure :

```bash
microk8s <commande> [options] [arguments]
```

Par exemple :
```bash
microk8s status --wait-ready
microk8s enable dns
microk8s kubectl get pods
```

**Note importante** : MicroK8s encapsule également `kubectl`, vous pouvez donc exécuter `microk8s kubectl` pour toutes vos opérations Kubernetes standard.

## La Commande `microk8s status`

### Utilisation de base

La commande `status` est probablement celle que vous utiliserez le plus souvent. Elle affiche l'état actuel de votre cluster :

```bash
microk8s status
```

**Sortie type** :

```
microk8s is running
high-availability: no
  datastore master nodes: 127.0.0.1:19001
  datastore standby nodes: none
addons:
  enabled:
    ha-cluster           # (core) Configure high availability on the current node
  disabled:
    community            # (community) The community addons repository
    dashboard            # (core) The Kubernetes dashboard
    dns                  # (core) CoreDNS
    gpu                  # (core) Automatic enablement of Nvidia CUDA
    helm                 # (core) Helm - the package manager for Kubernetes
    helm3                # (core) Helm 3 - the package manager for Kubernetes
    host-access          # (core) Allow Pods connecting to Host services smoothly
    hostpath-storage     # (core) Storage class; allocates storage from host directory
    ingress              # (core) Ingress controller for external access
    kube-ovn             # (community) An advanced network fabric for Kubernetes
    mayastor             # (core) OpenEBS MayaStor
    metallb              # (core) Loadbalancer for your Kubernetes cluster
    metrics-server       # (core) K8s Metrics Server for API access to service metrics
    observability        # (core) A lightweight observability stack for logs, traces and metrics
    prometheus           # (core) Prometheus operator for monitoring and logging
    rbac                 # (core) Role-Based Access Control for authorisation
    registry             # (core) Private image registry exposed on localhost:32000
    storage              # (core) Alias to hostpath-storage add-on, deprecated
    traefik              # (community) traefik Ingress controller
```

### Interprétation de la sortie

**Première ligne** : L'état du cluster
- `microk8s is running` : Le cluster fonctionne ✅
- `microk8s is not running` : Le cluster est arrêté ❌

**Section High-Availability** : Information sur le mode de fonctionnement
- `high-availability: no` : Mode single-node (un seul serveur)
- `high-availability: yes` : Mode cluster (plusieurs serveurs)
- `datastore master nodes` : Liste des nœuds maîtres du cluster

**Section Addons** : Liste des fonctionnalités additionnelles
- `enabled` : Addons actuellement activés
- `disabled` : Addons disponibles mais non activés

Chaque addon est accompagné d'une brève description entre parenthèses.

### Options avancées

#### Attendre que le cluster soit prêt

```bash
microk8s status --wait-ready
```

Cette option fait attendre la commande jusqu'à ce que tous les composants du cluster soient opérationnels. Très utile :
- Après un démarrage
- Dans des scripts d'automatisation
- Après l'activation d'un addon

**Exemple d'utilisation dans un script** :

```bash
#!/bin/bash
microk8s start
microk8s status --wait-ready
echo "Cluster prêt, déploiement des applications..."
```

#### Timeout personnalisé

Par défaut, `--wait-ready` attend 30 secondes. Vous pouvez modifier ce délai :

```bash
microk8s status --wait-ready --timeout 60
```

Cela attend jusqu'à 60 secondes avant d'abandonner.

#### Format de sortie YAML

Pour obtenir la sortie au format YAML (utile pour le parsing automatique) :

```bash
microk8s status --format yaml
```

**Sortie exemple** :

```yaml
microk8s:
  running: true
  ready: true
  high-availability: false
addons:
  enabled:
    - ha-cluster
  disabled:
    - dashboard
    - dns
    ...
```

#### Format de sortie JSON

Pour le format JSON :

```bash
microk8s status --format json
```

Ces formats structurés sont particulièrement utiles pour intégrer MicroK8s dans des outils d'automatisation ou de monitoring.

### Cas d'usage pratiques

**Vérification rapide avant de travailler** :

```bash
microk8s status | head -1
```

Affiche uniquement la première ligne (running ou not running).

**Lister uniquement les addons activés** :

```bash
microk8s status | grep -A 100 "enabled:" | grep -B 100 "disabled:" | head -n -1
```

**Vérifier si un addon spécifique est activé** :

```bash
microk8s status | grep "dns"
```

## La Commande `microk8s inspect`

### Utilisation de base

La commande `inspect` est l'outil de diagnostic principal de MicroK8s. Elle collecte automatiquement une grande quantité d'informations sur votre cluster :

```bash
microk8s inspect
```

**Ce que fait cette commande** :

1. Collecte les informations système (OS, kernel, ressources)
2. Vérifie la configuration réseau
3. Examine les certificats et leur validité
4. Analyse les logs des composants Kubernetes
5. Vérifie l'état des services
6. Inspecte la configuration des addons
7. Compile tout dans un rapport détaillé

**Sortie** :

```
Inspecting system
Inspecting Certificates
Inspecting services
Inspecting AppArmor configuration
Gathering system information
Inspecting kubernetes cluster
Inspecting dqlite
Copy service arguments to the final report tarball
Inspecting addon configurations

Building the report tarball
  Report tarball is at /var/snap/microk8s/common/inspection-report-20250123_143022.tar.gz
```

### Analyse du rapport

Le rapport est sauvegardé dans un fichier `.tar.gz`. Pour l'examiner :

**Extraire le rapport** :

```bash
cd /tmp
cp /var/snap/microk8s/common/inspection-report-*.tar.gz .
tar -xzf inspection-report-*.tar.gz
cd inspection-report-*
```

**Contenu du rapport** :

Le répertoire extrait contient plusieurs fichiers :
- `system.txt` : Informations système (CPU, RAM, OS)
- `network.txt` : Configuration réseau
- `certificates.txt` : État des certificats
- `services/` : Logs de tous les services Kubernetes
- `addons/` : Configuration des addons activés
- `kubectl/` : Sortie de diverses commandes kubectl

### Consulter directement le rapport

Sans extraire, vous pouvez afficher le contenu directement :

```bash
microk8s inspect | less
```

Utilisez les flèches pour naviguer, tapez `/` pour rechercher, et `q` pour quitter.

### Cas d'usage de inspect

#### Diagnostic de problème

Lorsque quelque chose ne fonctionne pas :

```bash
microk8s inspect
# Examinez le rapport pour identifier la cause
```

Recherchez les mots-clés suivants dans le rapport :
- `error` ou `Error`
- `failed` ou `Failed`
- `warning` ou `Warning`
- `timeout`
- `connection refused`

#### Partage avec la communauté

Si vous avez besoin d'aide sur les forums ou GitHub :

```bash
microk8s inspect
# Partagez le fichier .tar.gz généré
```

**Attention** : Le rapport peut contenir des informations sensibles (adresses IP, chemins, etc.). Vérifiez le contenu avant de le partager publiquement.

#### Vérification de certificats

Les certificats expirent avec le temps. Pour vérifier leur état :

```bash
microk8s inspect | grep -i certificate
```

Ou examinez directement le fichier `certificates.txt` dans le rapport.

#### Analyse des logs

Pour voir les logs d'un service spécifique :

```bash
# Extraire le rapport
tar -xzf /var/snap/microk8s/common/inspection-report-*.tar.gz -C /tmp

# Lire les logs d'un service
cat /tmp/inspection-report-*/services/apiserver.log
```

### Options de la commande inspect

La commande `inspect` n'a pas beaucoup d'options, mais vous pouvez la combiner avec d'autres commandes :

**Rediriger vers un fichier texte** :

```bash
microk8s inspect > rapport-microk8s.txt
```

**Chercher un terme spécifique** :

```bash
microk8s inspect | grep -i "erreur_recherchée"
```

## La Commande `microk8s enable`

### Concept des addons

Les addons sont des fonctionnalités additionnelles que vous pouvez activer dans MicroK8s. C'est l'une des forces de MicroK8s : ajouter des fonctionnalités complexes devient aussi simple qu'une seule commande.

### Utilisation de base

Pour activer un addon :

```bash
microk8s enable <nom-addon>
```

**Exemples** :

```bash
microk8s enable dns
microk8s enable dashboard
microk8s enable ingress
```

### Lister les addons disponibles

Vous pouvez voir tous les addons disponibles avec :

```bash
microk8s status
```

Ou de manière plus spécifique :

```bash
microk8s status | grep -A 50 "addons:"
```

### Addons les plus courants

Voici les addons que vous utiliserez probablement le plus :

#### DNS (CoreDNS)

```bash
microk8s enable dns
```

**Fonction** : Active la résolution DNS à l'intérieur du cluster. Permet aux pods de se trouver par leur nom de service.

**Indispensable** : Oui, à activer en priorité !

**Utilisation** : Une fois activé, vos pods peuvent utiliser des noms comme `mon-service.mon-namespace.svc.cluster.local` au lieu d'adresses IP.

#### Dashboard

```bash
microk8s enable dashboard
```

**Fonction** : Interface web pour gérer et visualiser votre cluster Kubernetes.

**Utilisation** :
```bash
microk8s dashboard-proxy
```

Ouvre un tunnel sécurisé et affiche l'URL d'accès au dashboard.

#### Storage (hostpath-storage)

```bash
microk8s enable hostpath-storage
```

**Fonction** : Active le stockage persistant. Permet à vos applications de sauvegarder des données qui survivent aux redémarrages.

**Indispensable** : Oui, si vous déployez des bases de données ou des applications avec état.

#### Registry

```bash
microk8s enable registry
```

**Fonction** : Crée un registre d'images Docker privé, accessible sur `localhost:32000`.

**Utilisation** : Permet de stocker vos images Docker personnalisées sans utiliser Docker Hub.

#### Ingress

```bash
microk8s enable ingress
```

**Fonction** : Active le contrôleur Ingress NGINX pour exposer vos services web sur Internet.

**Utilisation** : Permet de router le trafic HTTP/HTTPS vers différentes applications selon le nom de domaine ou le chemin.

#### Metrics Server

```bash
microk8s enable metrics-server
```

**Fonction** : Collecte les métriques de ressources (CPU, RAM) des pods et nœuds.

**Utilisation** : Nécessaire pour l'autoscaling et pour la commande `kubectl top`.

#### Prometheus

```bash
microk8s enable prometheus
```

**Fonction** : Installe la stack complète de monitoring Prometheus.

**Attention** : Addon gourmand en ressources. Recommandé pour les machines avec au moins 8 Go de RAM.

#### Cert-Manager

```bash
microk8s enable cert-manager
```

**Fonction** : Gère automatiquement les certificats SSL/TLS, notamment avec Let's Encrypt.

**Utilisation** : Automatise l'obtention et le renouvellement des certificats HTTPS.

#### MetalLB

```bash
microk8s enable metallb
```

**Fonction** : Fournit un load balancer pour attribuer des adresses IP externes aux services.

**Configuration** : Vous devrez spécifier une plage d'adresses IP lors de l'activation :

```bash
microk8s enable metallb:10.64.140.43-10.64.140.49
```

### Activer plusieurs addons simultanément

Vous pouvez activer plusieurs addons en une seule commande :

```bash
microk8s enable dns dashboard storage
```

**Ordre de traitement** : Les addons sont activés dans l'ordre spécifié, ce qui peut être important si certains dépendent d'autres.

### Attendre qu'un addon soit prêt

Certains addons prennent du temps à démarrer. Pour attendre qu'ils soient complètement opérationnels :

```bash
microk8s enable dns
microk8s status --wait-ready
```

Ou dans un script :

```bash
#!/bin/bash
echo "Activation du DNS..."
microk8s enable dns

echo "Attente de la disponibilité..."
microk8s status --wait-ready

echo "DNS prêt !"
```

### Vérifier l'état d'un addon

Après l'activation, vérifiez que l'addon fonctionne :

```bash
# Vérifier le statut général
microk8s status

# Vérifier les pods de l'addon
microk8s kubectl get pods -n kube-system

# Pour les addons avec leur propre namespace
microk8s kubectl get pods -A
```

**Exemple avec le dashboard** :

```bash
microk8s enable dashboard
microk8s kubectl get pods -n kube-system | grep dashboard
```

Vous devriez voir un pod avec le nom contenant "dashboard" en état `Running`.

### Addons avec configuration

Certains addons nécessitent ou acceptent des paramètres de configuration.

#### MetalLB avec plage d'IP

```bash
microk8s enable metallb:192.168.1.240-192.168.1.250
```

#### Registry avec stockage personnalisé

```bash
microk8s enable registry:size=40Gi
```

#### Observability avec configuration

```bash
microk8s enable observability
```

Suit ensuite une série de questions interactives pour configurer l'addon.

### Dépendances entre addons

Certains addons ont des dépendances. MicroK8s les gère généralement automatiquement :

**Exemple** : Si vous activez `prometheus`, MicroK8s active automatiquement `storage` si ce n'est pas déjà fait, car Prometheus a besoin de stockage persistant.

**Recommandation** : Pour une stack complète, activez dans cet ordre :

```bash
microk8s enable dns
microk8s enable hostpath-storage
microk8s enable ingress
microk8s enable cert-manager
microk8s enable metrics-server
```

## La Commande `microk8s disable`

### Désactiver un addon

Pour désactiver un addon que vous n'utilisez plus :

```bash
microk8s disable <nom-addon>
```

**Exemples** :

```bash
microk8s disable dashboard
microk8s disable prometheus
```

**Attention** : Désactiver un addon supprime généralement toutes les ressources associées (pods, services, configurations). Les données peuvent être perdues !

### Cas d'usage

**Libérer des ressources** :

Si votre machine manque de ressources, désactivez les addons non essentiels :

```bash
microk8s disable prometheus
microk8s disable dashboard
```

**Réinitialiser un addon** :

Pour réinitialiser complètement un addon qui fonctionne mal :

```bash
microk8s disable registry
microk8s enable registry
```

**Test de différentes configurations** :

Dans un environnement de test, vous pouvez facilement activer/désactiver des addons :

```bash
microk8s disable ingress
# Testez sans ingress
microk8s enable ingress
# Testez avec ingress
```

## Autres Commandes Essentielles

### microk8s start

Démarre le cluster MicroK8s :

```bash
microk8s start
```

**Quand l'utiliser** :
- Après un arrêt du cluster
- Après le démarrage de votre machine
- Après une maintenance

**Note** : Sur certains systèmes, MicroK8s démarre automatiquement au boot.

### microk8s stop

Arrête le cluster MicroK8s :

```bash
microk8s stop
```

**Attention** : Les applications déployées dans le cluster deviennent inaccessibles.

**Quand l'utiliser** :
- Avant d'éteindre votre machine (optionnel)
- Pour libérer des ressources temporairement
- Avant une maintenance système

### microk8s kubectl

Interface vers kubectl, l'outil de ligne de commande Kubernetes :

```bash
microk8s kubectl <commande-kubectl>
```

**Exemples** :

```bash
microk8s kubectl get pods
microk8s kubectl get services
microk8s kubectl describe node
microk8s kubectl logs mon-pod
```

**Note** : Nous verrons les commandes kubectl en détail dans la section 4.6.

### microk8s config

Génère la configuration kubectl pour se connecter au cluster :

```bash
microk8s config
```

**Utilisation** : Pour exporter la configuration et l'utiliser avec un kubectl standalone :

```bash
microk8s config > ~/.kube/config
```

Après cela, vous pouvez utiliser `kubectl` directement sans le préfixe `microk8s`.

### microk8s version

Affiche les versions de MicroK8s et Kubernetes :

```bash
microk8s version
```

**Sortie exemple** :

```
MicroK8s v1.28.3 revision 5891
```

### microk8s add-node

Génère un token pour ajouter un nœud au cluster (mode multi-node) :

```bash
microk8s add-node
```

**Utilisation avancée** : Nous verrons cette commande en détail dans le chapitre 21 sur la haute disponibilité.

### microk8s join

Joint un nœud à un cluster existant :

```bash
microk8s join <token-fourni-par-add-node>
```

### microk8s remove-node

Retire un nœud du cluster :

```bash
microk8s remove-node <nom-du-noeud>
```

### microk8s refresh

Met à jour MicroK8s vers la dernière version du canal actuel :

```bash
microk8s refresh
```

**Note** : Cette commande utilise le système de snaps pour mettre à jour.

### microk8s reset

Réinitialise complètement MicroK8s à son état initial :

```bash
microk8s reset
```

**ATTENTION** : Cette commande détruit toutes les données, configurations, et déploiements ! Utilisez avec précaution.

**Utilisation** : En cas de problème majeur où vous voulez repartir de zéro.

## Combinaison de Commandes

### Scripts pratiques

**Démarrage complet avec vérification** :

```bash
#!/bin/bash
echo "Démarrage de MicroK8s..."
microk8s start

echo "Attente de la disponibilité..."
microk8s status --wait-ready

echo "État du cluster :"
microk8s kubectl get nodes

echo "✅ Cluster prêt !"
```

**Activation d'une stack complète** :

```bash
#!/bin/bash
echo "Configuration de la stack de base..."
microk8s enable dns
microk8s status --wait-ready

microk8s enable hostpath-storage
microk8s status --wait-ready

microk8s enable ingress
microk8s status --wait-ready

echo "✅ Stack de base configurée !"
```

**Vérification complète** :

```bash
#!/bin/bash
echo "=== Vérification MicroK8s ==="

echo "1. Statut :"
microk8s status | head -1

echo -e "\n2. Nœuds :"
microk8s kubectl get nodes

echo -e "\n3. Pods système :"
microk8s kubectl get pods -n kube-system

echo -e "\n4. Addons activés :"
microk8s status | grep -A 20 "enabled:"

echo -e "\n✅ Vérification terminée"
```

## Aide et Documentation

### Aide générale

Pour voir toutes les commandes disponibles :

```bash
microk8s --help
```

Ou simplement :

```bash
microk8s
```

### Aide pour une commande spécifique

Pour obtenir de l'aide sur une commande particulière :

```bash
microk8s <commande> --help
```

**Exemples** :

```bash
microk8s status --help
microk8s enable --help
microk8s kubectl --help
```

## Commandes à Mémoriser

Voici un résumé des commandes que vous utiliserez le plus souvent :

| Commande | Usage | Fréquence |
|----------|-------|-----------|
| `microk8s status` | Vérifier l'état du cluster | Quotidienne |
| `microk8s enable <addon>` | Activer une fonctionnalité | Régulière |
| `microk8s kubectl get pods` | Lister les pods | Quotidienne |
| `microk8s inspect` | Diagnostic en cas de problème | Occasionnelle |
| `microk8s start` | Démarrer le cluster | Au besoin |
| `microk8s stop` | Arrêter le cluster | Au besoin |

## Bonnes Pratiques

### Vérifications régulières

Prenez l'habitude de vérifier régulièrement :

```bash
# Au début de votre journée
microk8s status
```

### Avant chaque opération importante

```bash
# Avant de déployer
microk8s status --wait-ready
```

### Après l'activation d'un addon

```bash
# Activer
microk8s enable dns

# Vérifier
microk8s status --wait-ready
microk8s kubectl get pods -A
```

### Documentation des addons activés

Gardez trace des addons que vous avez activés :

```bash
microk8s status > configuration-cluster.txt
```

### Sauvegarde régulière de l'état

```bash
# Script de sauvegarde périodique
#!/bin/bash
DATE=$(date +%Y%m%d)
microk8s inspect
cp /var/snap/microk8s/common/inspection-report-*.tar.gz ~/backups/microk8s-$DATE.tar.gz
```

## Résolution de Problèmes avec les Commandes

### Le cluster ne démarre pas

```bash
# Tentative de démarrage
microk8s start

# Vérification
microk8s status

# Diagnostic approfondi
microk8s inspect
```

### Un addon ne fonctionne pas

```bash
# Désactiver
microk8s disable addon-problematique

# Vérifier l'arrêt
microk8s status

# Réactiver
microk8s enable addon-problematique

# Attendre la disponibilité
microk8s status --wait-ready

# Vérifier les pods
microk8s kubectl get pods -A | grep addon-problematique
```

### Commandes qui ne répondent pas

```bash
# Vérifier si MicroK8s tourne
systemctl status snap.microk8s.daemon-containerd

# Redémarrer si nécessaire
microk8s stop
microk8s start
```

## Automatisation et Scripts

### Vérification de santé automatique

```bash
#!/bin/bash
# healthcheck.sh

STATUS=$(microk8s status | head -1)

if echo "$STATUS" | grep -q "running"; then
    echo "✅ MicroK8s est opérationnel"
    exit 0
else
    echo "❌ MicroK8s ne fonctionne pas"
    echo "Tentative de redémarrage..."
    microk8s start
    microk8s status --wait-ready
    exit 1
fi
```

### Déploiement automatisé

```bash
#!/bin/bash
# setup-cluster.sh

echo "Configuration automatique du cluster..."

# Démarrage
microk8s start
microk8s status --wait-ready

# Addons de base
ADDONS="dns hostpath-storage ingress metrics-server"

for addon in $ADDONS; do
    echo "Activation de $addon..."
    microk8s enable $addon
    microk8s status --wait-ready
done

echo "✅ Configuration terminée"
microk8s status
```

## Prochaines Étapes

Maintenant que vous maîtrisez les commandes essentielles de MicroK8s, vous êtes prêt pour :

1. **Section 2.9** : Configurer les alias kubectl pour gagner en productivité
2. **Section 2.10** : Réaliser un premier diagnostic complet avec `microk8s inspect`
3. **Chapitre 3** : Découvrir les concepts Kubernetes essentiels
4. **Chapitre 4** : Effectuer vos premiers déploiements

Vous disposez maintenant de tous les outils nécessaires pour gérer efficacement votre cluster MicroK8s au quotidien ! 🎯

⏭️ [Configuration des alias kubectl](/02-installation-et-configuration-initiale/09-configuration-des-alias-kubectl.md)
