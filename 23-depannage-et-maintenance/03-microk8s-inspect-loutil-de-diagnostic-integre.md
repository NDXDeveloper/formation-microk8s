🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 23.3 microk8s inspect : L'Outil de Diagnostic Intégré

## Introduction

`microk8s inspect` est l'outil de diagnostic le plus puissant et le plus complet de MicroK8s. C'est votre meilleur allié lorsque vous rencontrez des problèmes complexes qui ne sont pas immédiatement évidents avec les commandes kubectl classiques. Pensez-y comme un "scanner médical complet" pour votre cluster.

> **Pour les débutants** : `microk8s inspect` fait tout le travail d'investigation à votre place. Au lieu de lancer 20 commandes différentes pour comprendre un problème, cette commande unique collecte toutes les informations pertinentes en une seule fois.

## Qu'est-ce que microk8s inspect ?

`microk8s inspect` est un outil automatisé qui :

1. **Collecte des informations** sur l'état complet de votre cluster
2. **Vérifie la santé** de tous les composants système
3. **Détecte les problèmes** courants automatiquement
4. **Génère un rapport** détaillé et structuré
5. **Crée une archive** que vous pouvez partager avec d'autres pour obtenir de l'aide

### Pourquoi utiliser microk8s inspect ?

**Avantages** :
- **Gain de temps** : Une seule commande au lieu de dizaines
- **Complet** : Capture des informations que vous n'auriez peut-être pas pensé à vérifier
- **Structuré** : Les informations sont organisées de manière logique
- **Partageable** : Vous pouvez envoyer le rapport à quelqu'un pour obtenir de l'aide
- **Automatisé** : Détecte automatiquement certains problèmes courants

**Quand l'utiliser** :
- Problèmes complexes dont la cause n'est pas évidente
- Avant de demander de l'aide sur les forums
- Pour documenter l'état du système avant/après une modification importante
- Lors d'audits ou de bilans de santé périodiques
- Quand vous voulez tout vérifier mais ne savez pas par où commencer

## Utilisation de Base

### Syntaxe Simple

La forme la plus simple de la commande :

```bash
microk8s inspect
```

C'est tout ! La commande va :
1. Analyser votre cluster
2. Collecter les informations
3. Générer un rapport
4. Créer une archive dans `/var/snap/microk8s/current/inspection-report/`

### Sortie Typique

Voici ce que vous verrez pendant l'exécution :

```
Inspecting system
Inspecting Certificates
Inspecting services
  Service snap.microk8s.daemon-cluster-agent is running
  Service snap.microk8s.daemon-containerd is running
  Service snap.microk8s.daemon-kubelite is running
  Service snap.microk8s.daemon-apiserver-kicker is running
Inspecting AppArmor configuration
Gathering system information
  Copy network configuration
  Copy processes list
  Copy snap list
  Copy VM name (or none)
  Copy disk usage information
  Copy memory usage information
  Copy server uptime
  Copy openSSL information
  Copy network setup
  Copy network routes
  Copy iptables trace
Inspecting kubernetes cluster
  Inspect kubernetes cluster
Inspecting dqlite
  Inspect dqlite

Building the report tarball
  Report tarball is at /var/snap/microk8s/common/inspection-report-20251026_143022.tar.gz
```

### Emplacement du Rapport

Par défaut, le rapport est créé dans :
```
/var/snap/microk8s/common/inspection-report-YYYYMMDD_HHMMSS.tar.gz
```

Le nom inclut la date et l'heure pour faciliter l'identification.

## Options de la Commande

### Spécifier l'Emplacement de Sortie

Vous pouvez choisir où sauvegarder le rapport :

```bash
microk8s inspect --output /tmp/mon-rapport.tar.gz
```

ou avec l'option courte :

```bash
microk8s inspect -o /tmp/mon-rapport.tar.gz
```

**Cas d'usage** : Sauvegarder le rapport dans un emplacement facilement accessible pour le transférer ou le partager.

### Afficher la Version

Vérifier la version de l'outil inspect :

```bash
microk8s inspect --version
```

### Aide

Obtenir de l'aide sur la commande :

```bash
microk8s inspect --help
```

## Contenu du Rapport

Le rapport généré est une archive tar.gz contenant une structure de dossiers organisée. Voici ce qu'elle contient :

### Structure du Rapport

```
inspection-report/
├── README.txt                    # Instructions pour lire le rapport
├── cluster-info.txt             # Informations générales du cluster
├── version.txt                  # Versions de MicroK8s et Kubernetes
├── services/                    # État des services système
│   ├── snap.microk8s.daemon-cluster-agent.txt
│   ├── snap.microk8s.daemon-containerd.txt
│   ├── snap.microk8s.daemon-kubelite.txt
│   └── ...
├── system/                      # Informations système
│   ├── disk.txt                # Utilisation du disque
│   ├── memory.txt              # Utilisation de la mémoire
│   ├── network.txt             # Configuration réseau
│   ├── routes.txt              # Tables de routage
│   ├── iptables.txt            # Règles iptables
│   ├── processes.txt           # Liste des processus
│   └── uptime.txt              # Temps de fonctionnement
├── kubernetes/                  # État du cluster Kubernetes
│   ├── nodes.txt               # État des nœuds
│   ├── pods.txt                # État des pods
│   ├── services.txt            # Liste des services
│   ├── deployments.txt         # État des déploiements
│   ├── events.txt              # Événements récents
│   └── logs/                   # Logs des composants
│       ├── kube-apiserver.log
│       ├── kube-controller-manager.log
│       ├── kube-scheduler.log
│       └── kubelet.log
├── dqlite/                     # Informations sur la base de données Dqlite
│   └── dqlite-info.txt
└── certificates/               # État des certificats
    └── cert-info.txt
```

### Détails des Sections Importantes

#### 1. cluster-info.txt

Contient des informations de base :
```
Kubernetes version: v1.28.3
MicroK8s version: v1.28.3
Node name: ubuntu-pc
Operating system: Ubuntu 22.04.3 LTS
Kernel version: 5.15.0-88-generic
```

**Ce que ça vous dit** : Versions installées, utile pour vérifier la compatibilité ou savoir si une mise à jour est nécessaire.

#### 2. services/

Chaque fichier contient l'état d'un service MicroK8s :
```
● snap.microk8s.daemon-kubelite.service - Service for snap application microk8s.daemon-kubelite
     Loaded: loaded (/etc/systemd/system/snap.microk8s.daemon-kubelite.service; enabled; vendor preset: enabled)
     Active: active (running) since Mon 2025-10-21 10:00:00 CEST; 5 days ago
```

**Ce que ça vous dit** : Si tous les services essentiels sont en cours d'exécution. Un service "inactive" ou "failed" indique un problème majeur.

#### 3. system/disk.txt

Affiche l'utilisation du disque :
```
Filesystem      Size  Used Avail Use% Mounted on
/dev/sda1       100G   45G   50G  47% /
/dev/sda2       500G  200G  275G  42% /home
```

**Ce que ça vous dit** : Si vous manquez d'espace disque, ce qui peut causer de nombreux problèmes (pods qui ne démarrent pas, images qui ne se téléchargent pas, etc.).

#### 4. system/memory.txt

Affiche l'utilisation de la mémoire :
```
              total        used        free      shared  buff/cache   available
Mem:           16Gi       8.0Gi       2.0Gi       1.0Gi       6.0Gi       7.0Gi
Swap:         4.0Gi       0.5Gi       3.5Gi
```

**Ce que ça vous dit** : Si votre système manque de mémoire, ce qui peut provoquer des évictions de pods ou des ralentissements.

#### 5. system/network.txt

Configuration réseau :
```
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536
    inet 127.0.0.1/8 scope host lo
2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500
    inet 192.168.1.100/24 brd 192.168.1.255 scope global eth0
```

**Ce que ça vous dit** : Vos interfaces réseau et leurs adresses IP. Utile pour diagnostiquer les problèmes de connectivité.

#### 6. system/iptables.txt

Règles de pare-feu :
```
Chain INPUT (policy ACCEPT)
target     prot opt source               destination
KUBE-FIREWALL  all  --  anywhere             anywhere
...
```

**Ce que ça vous dit** : Les règles de pare-feu peuvent bloquer le trafic. Ce fichier est complexe mais peut révéler des blocages.

#### 7. kubernetes/nodes.txt

État des nœuds :
```
NAME        STATUS   ROLES    AGE   VERSION
ubuntu-pc   Ready    <none>   5d    v1.28.3
```

**Ce que ça vous dit** : Si vos nœuds sont "Ready" ou ont des problèmes.

#### 8. kubernetes/pods.txt

Liste de tous les pods :
```
NAMESPACE     NAME                                    READY   STATUS    RESTARTS   AGE
kube-system   coredns-5d78c9869d-abcde               1/1     Running   0          5d
kube-system   calico-node-xyz123                     1/1     Running   2          5d
default       mon-app-deployment-7d9f8c-pqrst        1/1     Running   0          2d
```

**Ce que ça vous dit** : Vue d'ensemble de tous les pods. Regardez les colonnes STATUS et RESTARTS pour repérer les problèmes.

#### 9. kubernetes/events.txt

Événements récents du cluster :
```
LAST SEEN   TYPE      REASON              OBJECT                MESSAGE
5m          Normal    Scheduled           pod/mon-pod           Successfully assigned default/mon-pod to ubuntu-pc
5m          Normal    Pulling             pod/mon-pod           Pulling image "nginx:1.21"
2m          Warning   BackOff             pod/autre-pod         Back-off restarting failed container
```

**Ce que ça vous dit** : Ce qui s'est passé récemment. Les événements de type "Warning" ou "Error" sont particulièrement importants.

#### 10. kubernetes/logs/

Logs des composants système Kubernetes :
- **kube-apiserver.log** : Logs de l'API server (point d'entrée de toutes les requêtes)
- **kube-controller-manager.log** : Logs du gestionnaire de contrôleurs
- **kube-scheduler.log** : Logs du scheduler (responsable du placement des pods)
- **kubelet.log** : Logs du kubelet (agent qui exécute les pods sur le nœud)

**Ce que ça vous dit** : Erreurs au niveau des composants système. Utile pour des problèmes très techniques.

#### 11. dqlite/dqlite-info.txt

Informations sur Dqlite (la base de données distribuée de MicroK8s) :
```
Dqlite cluster members:
  - 192.168.1.100:19001 (voter)

Dqlite leader: 192.168.1.100:19001
```

**Ce que ça vous dit** : État de la base de données du cluster. Important pour les clusters multi-nœuds.

#### 12. certificates/cert-info.txt

Informations sur les certificats :
```
Certificate: /var/snap/microk8s/current/certs/server.crt
  Valid from: 2025-10-21 10:00:00 UTC
  Valid until: 2026-10-21 10:00:00 UTC
  Days remaining: 365
```

**Ce que ça vous dit** : Si vos certificats sont valides ou expirés. Des certificats expirés empêchent le cluster de fonctionner.

## Comment Analyser le Rapport

Voici une méthodologie pour analyser efficacement le rapport :

### Étape 1 : Extraire l'Archive

```bash
# Se déplacer dans un dossier de travail
cd /tmp

# Extraire le rapport
tar -xzf /var/snap/microk8s/common/inspection-report-20251026_143022.tar.gz

# Entrer dans le dossier
cd inspection-report-20251026_143022
```

### Étape 2 : Lire le README

Commencez toujours par lire le fichier README :

```bash
cat README.txt
```

Il contient des instructions et un résumé du contenu.

### Étape 3 : Vérifications Prioritaires

Voici les fichiers à vérifier en priorité selon les symptômes :

#### Problème : Cluster ne démarre pas

**Fichiers à vérifier** :
1. `services/` - Tous les services sont-ils "active (running)" ?
2. `system/disk.txt` - Y a-t-il assez d'espace disque ?
3. `system/memory.txt` - Y a-t-il assez de mémoire ?
4. `kubernetes/logs/kube-apiserver.log` - Y a-t-il des erreurs ?

**Commande rapide** :
```bash
# Vérifier tous les statuts de services
grep -r "Active:" services/
```

#### Problème : Pods ne démarrent pas

**Fichiers à vérifier** :
1. `kubernetes/pods.txt` - Quel est le statut exact ?
2. `kubernetes/events.txt` - Quels événements récents ?
3. `system/disk.txt` - Assez d'espace pour les images ?
4. `kubernetes/logs/kubelet.log` - Erreurs du kubelet ?

**Commande rapide** :
```bash
# Voir tous les pods non-Running
grep -v "Running" kubernetes/pods.txt

# Voir tous les événements Warning ou Error
grep -E "Warning|Error" kubernetes/events.txt
```

#### Problème : Réseau ne fonctionne pas

**Fichiers à vérifier** :
1. `system/network.txt` - Configuration des interfaces
2. `system/routes.txt` - Tables de routage
3. `system/iptables.txt` - Règles de pare-feu
4. `kubernetes/pods.txt` - État des pods réseau (CoreDNS, Calico)

**Commande rapide** :
```bash
# Vérifier l'état de CoreDNS
grep "coredns" kubernetes/pods.txt

# Vérifier les interfaces réseau
cat system/network.txt
```

#### Problème : Certificats / Accès API

**Fichiers à vérifier** :
1. `certificates/cert-info.txt` - Validité des certificats
2. `kubernetes/logs/kube-apiserver.log` - Erreurs d'authentification

**Commande rapide** :
```bash
# Vérifier les dates d'expiration
cat certificates/cert-info.txt
```

### Étape 4 : Recherche Avancée

Utilisez `grep` pour chercher des mots-clés :

```bash
# Chercher "error" dans tout le rapport
grep -r -i "error" .

# Chercher "failed" dans tout le rapport
grep -r -i "failed" .

# Chercher "warning" dans les logs Kubernetes
grep -r -i "warning" kubernetes/

# Chercher un nom de pod spécifique
grep -r "mon-pod" .
```

### Étape 5 : Comparer Avant/Après

Si vous avez généré un rapport avant un problème et un après :

```bash
# Comparer deux rapports
diff -r rapport-avant/ rapport-apres/

# Ou comparer des fichiers spécifiques
diff rapport-avant/kubernetes/pods.txt rapport-apres/kubernetes/pods.txt
```

## Scénarios Pratiques d'Utilisation

### Scénario 1 : Demander de l'Aide sur un Forum

**Situation** : Vous avez un problème complexe et voulez demander de l'aide sur un forum ou Discord Kubernetes.

**Procédure** :

1. Générer le rapport :
```bash
microk8s inspect -o ~/rapport-probleme.tar.gz
```

2. Le rapport contient des informations sensibles potentielles. Vérifiez-le avant de le partager :
```bash
cd /tmp
tar -xzf ~/rapport-probleme.tar.gz
# Parcourez les fichiers pour vérifier qu'il n'y a pas de secrets, mots de passe, etc.
```

3. Partagez le rapport sur le forum avec une description claire du problème.

**Avantage** : Les personnes qui vous aident ont toutes les informations nécessaires sans avoir à vous demander 10 fois de lancer différentes commandes.

### Scénario 2 : Diagnostic Avant Mise à Jour

**Situation** : Vous allez mettre à jour MicroK8s et voulez un état de référence.

**Procédure** :

1. Générer un rapport "avant" :
```bash
microk8s inspect -o ~/rapport-avant-maj.tar.gz
```

2. Effectuer la mise à jour :
```bash
sudo snap refresh microk8s --channel=1.29/stable
```

3. Générer un rapport "après" :
```bash
microk8s inspect -o ~/rapport-apres-maj.tar.gz
```

4. Comparer les rapports pour vérifier que tout fonctionne :
```bash
cd /tmp
mkdir compare
cd compare
tar -xzf ~/rapport-avant-maj.tar.gz
mv inspection-report-* avant
tar -xzf ~/rapport-apres-maj.tar.gz
mv inspection-report-* apres
diff -r avant/kubernetes/pods.txt apres/kubernetes/pods.txt
```

**Avantage** : Si quelque chose ne va pas après la mise à jour, vous avez un point de référence pour identifier ce qui a changé.

### Scénario 3 : Audit Périodique de Santé

**Situation** : Vous voulez vérifier régulièrement la santé de votre cluster.

**Procédure** :

1. Créer un script d'audit :
```bash
#!/bin/bash
# audit-cluster.sh

DATE=$(date +%Y%m%d)
REPORT_DIR=~/cluster-audits

mkdir -p $REPORT_DIR
microk8s inspect -o $REPORT_DIR/audit-$DATE.tar.gz

echo "Rapport d'audit généré : $REPORT_DIR/audit-$DATE.tar.gz"

# Extraire et vérifier les points critiques
cd /tmp
tar -xzf $REPORT_DIR/audit-$DATE.tar.gz
cd inspection-report-*

# Vérifier les services
echo "=== État des services ==="
grep "Active:" services/* | grep -v "active (running)"

# Vérifier les pods en erreur
echo "=== Pods en erreur ==="
grep -v "Running" kubernetes/pods.txt | grep -v "READY"

# Vérifier l'espace disque
echo "=== Utilisation disque ==="
grep "/$" system/disk.txt

# Vérifier les certificats
echo "=== Certificats ==="
grep "Days remaining" certificates/cert-info.txt
```

2. Rendre le script exécutable et le lancer :
```bash
chmod +x audit-cluster.sh
./audit-cluster.sh
```

3. Optionnel : Ajouter à une tâche cron pour automatiser :
```bash
# Éditer la crontab
crontab -e

# Ajouter une ligne pour un audit hebdomadaire tous les lundis à 9h
0 9 * * 1 /home/user/audit-cluster.sh
```

**Avantage** : Détection proactive des problèmes avant qu'ils ne deviennent critiques.

### Scénario 4 : Investigation d'un Crash

**Situation** : Votre cluster a crashé hier soir et vous voulez comprendre pourquoi.

**Procédure** :

1. Générer le rapport dès que possible :
```bash
microk8s inspect -o ~/rapport-post-crash.tar.gz
```

2. Analyser les logs pour trouver ce qui s'est passé :
```bash
cd /tmp
tar -xzf ~/rapport-post-crash.tar.gz
cd inspection-report-*

# Vérifier les redémarrages de pods
echo "=== Pods avec redémarrages ==="
awk '$4 > 0' kubernetes/pods.txt

# Chercher des erreurs Out Of Memory
echo "=== Recherche OOM ==="
grep -r "OOM" .

# Chercher des erreurs dans les logs système
echo "=== Erreurs dans les logs ==="
grep -i "error\|fatal\|panic" kubernetes/logs/*.log | tail -50
```

**Avantage** : Même si le problème est résolu (pods redémarrés), vous avez capturé l'état au moment du problème.

## Limitations et Précautions

### 1. Informations Sensibles

**Attention** : Le rapport peut contenir des informations sensibles :
- Noms d'hôtes et adresses IP
- Noms de namespaces et applications
- Configuration réseau de votre infrastructure

**Bonne pratique** : Avant de partager un rapport publiquement, anonymisez ou supprimez les informations sensibles.

### 2. Taille du Rapport

Les rapports peuvent être volumineux (plusieurs Mo), surtout si vous avez beaucoup de pods et de logs.

**Conseil** : Compressez davantage ou nettoyez avant de partager :
```bash
# Créer une version allégée (sans certains logs)
cd inspection-report-*
rm -rf kubernetes/logs/
tar -czf ../rapport-light.tar.gz .
```

### 3. Instantané Ponctuel

Le rapport est un instantané à un moment T. Si le problème est intermittent, vous devrez peut-être générer plusieurs rapports à différents moments.

**Conseil** : En cas de problème intermittent, générez un rapport quand le problème se produit ET quand tout fonctionne, puis comparez.

### 4. Permissions

Vous devez avoir les permissions appropriées pour lire certaines informations système.

**Solution** : Exécutez avec les bonnes permissions ou en tant qu'utilisateur du groupe `microk8s` :
```bash
sudo usermod -a -G microk8s $USER
```

## Automatisation et Scripts Utiles

### Script 1 : Analyse Rapide

Créer un script qui extrait et analyse les points critiques :

```bash
#!/bin/bash
# quick-analysis.sh

if [ $# -eq 0 ]; then
    echo "Usage: $0 <inspection-report.tar.gz>"
    exit 1
fi

REPORT=$1
TMPDIR=$(mktemp -d)

echo "Extraction du rapport..."
tar -xzf "$REPORT" -C "$TMPDIR"
cd "$TMPDIR"/inspection-report-*

echo ""
echo "======================================"
echo "ANALYSE RAPIDE DU CLUSTER"
echo "======================================"
echo ""

echo "1. SERVICES"
echo "----------"
FAILED_SERVICES=$(grep "Active:" services/* | grep -v "active (running)" | wc -l)
if [ $FAILED_SERVICES -eq 0 ]; then
    echo "✓ Tous les services sont actifs"
else
    echo "✗ $FAILED_SERVICES service(s) en échec:"
    grep "Active:" services/* | grep -v "active (running)"
fi

echo ""
echo "2. NŒUDS"
echo "--------"
NOT_READY=$(grep -v "Ready" kubernetes/nodes.txt | grep -v "STATUS" | wc -l)
if [ $NOT_READY -eq 0 ]; then
    echo "✓ Tous les nœuds sont Ready"
else
    echo "✗ $NOT_READY nœud(s) non Ready:"
    grep -v "Ready" kubernetes/nodes.txt | grep -v "STATUS"
fi

echo ""
echo "3. PODS EN ERREUR"
echo "-----------------"
ERROR_PODS=$(grep -v "Running\|Completed" kubernetes/pods.txt | grep -v "STATUS" | wc -l)
if [ $ERROR_PODS -eq 0 ]; then
    echo "✓ Tous les pods sont Running ou Completed"
else
    echo "✗ $ERROR_PODS pod(s) en erreur:"
    grep -v "Running\|Completed" kubernetes/pods.txt | grep -v "STATUS"
fi

echo ""
echo "4. ESPACE DISQUE"
echo "----------------"
DISK_USAGE=$(grep "/$" system/disk.txt | awk '{print $5}' | sed 's/%//')
if [ $DISK_USAGE -lt 80 ]; then
    echo "✓ Espace disque OK ($DISK_USAGE%)"
else
    echo "⚠ Attention: Espace disque élevé ($DISK_USAGE%)"
fi

echo ""
echo "5. MÉMOIRE"
echo "----------"
cat system/memory.txt

echo ""
echo "6. ÉVÉNEMENTS RÉCENTS (WARNINGS/ERRORS)"
echo "---------------------------------------"
grep -E "Warning|Error" kubernetes/events.txt | tail -10

echo ""
echo "======================================"
echo "Rapport complet dans: $TMPDIR"
echo "======================================"

read -p "Supprimer le dossier temporaire ? (o/n) " -n 1 -r
echo
if [[ $REPLY =~ ^[Oo]$ ]]; then
    rm -rf "$TMPDIR"
    echo "Nettoyé."
fi
```

Utilisation :
```bash
chmod +x quick-analysis.sh
./quick-analysis.sh ~/rapport-probleme.tar.gz
```

### Script 2 : Comparaison de Deux Rapports

```bash
#!/bin/bash
# compare-reports.sh

if [ $# -ne 2 ]; then
    echo "Usage: $0 <rapport1.tar.gz> <rapport2.tar.gz>"
    exit 1
fi

REPORT1=$1
REPORT2=$2
TMPDIR=$(mktemp -d)

echo "Extraction des rapports..."
mkdir -p "$TMPDIR/report1" "$TMPDIR/report2"
tar -xzf "$REPORT1" -C "$TMPDIR/report1"
tar -xzf "$REPORT2" -C "$TMPDIR/report2"

cd "$TMPDIR"
R1=$(ls report1/)
R2=$(ls report2/)

echo ""
echo "======================================"
echo "COMPARAISON DE RAPPORTS"
echo "======================================"
echo ""

echo "1. DIFFÉRENCES DANS LES PODS"
echo "----------------------------"
diff "report1/$R1/kubernetes/pods.txt" "report2/$R2/kubernetes/pods.txt" || echo "Aucune différence"

echo ""
echo "2. DIFFÉRENCES DANS LES SERVICES"
echo "--------------------------------"
diff "report1/$R1/kubernetes/services.txt" "report2/$R2/kubernetes/services.txt" || echo "Aucune différence"

echo ""
echo "3. DIFFÉRENCES DANS LES ÉVÉNEMENTS"
echo "----------------------------------"
echo "Nouveaux événements dans le rapport 2:"
comm -13 <(sort "report1/$R1/kubernetes/events.txt") <(sort "report2/$R2/kubernetes/events.txt")

echo ""
echo "======================================"
echo "Rapports dans: $TMPDIR"
echo "======================================"

read -p "Supprimer les fichiers temporaires ? (o/n) " -n 1 -r
echo
if [[ $REPLY =~ ^[Oo]$ ]]; then
    rm -rf "$TMPDIR"
fi
```

## Intégration avec d'Autres Outils

### Avec kubectl

`microk8s inspect` complète kubectl, ne le remplace pas :

```bash
# D'abord un diagnostic rapide avec kubectl
microk8s kubectl get pods
microk8s kubectl describe pod mon-pod

# Si le problème n'est pas clair, générer un rapport complet
microk8s inspect
```

### Avec Monitoring (Prometheus/Grafana)

Le rapport inspect capture un instantané, tandis que le monitoring capture l'historique :

- **Utilisez inspect pour** : Diagnostic ponctuel, état actuel complet
- **Utilisez monitoring pour** : Tendances dans le temps, alertes proactives

**Workflow combiné** :
1. Prometheus détecte une anomalie et envoie une alerte
2. Vous générez un rapport `microk8s inspect` pour investigation détaillée
3. Vous consultez les dashboards Grafana pour voir l'historique

### Avec les Logs Centralisés

Si vous avez mis en place une stack ELK ou Loki :

- **Logs centralisés** : Historique complet des logs application
- **microk8s inspect** : Logs système et état du cluster

**Workflow combiné** :
1. Problème détecté dans les logs applicatifs (Loki/ELK)
2. Générer `microk8s inspect` pour voir si c'est un problème d'infrastructure
3. Corréler les timestamps entre les deux sources

## Bonnes Pratiques

### 1. Générer Régulièrement des Rapports

Même si tout fonctionne, générez des rapports périodiquement :

```bash
# Par exemple, un rapport hebdomadaire
microk8s inspect -o ~/audits/rapport-$(date +%Y-%m-%d).tar.gz
```

**Pourquoi** : Avoir un historique pour comparer quand un problème survient.

### 2. Documenter les Changements

Quand vous faites des modifications importantes, générez un rapport avant et après :

```bash
# Avant
microk8s inspect -o ~/rapport-avant-changement.tar.gz

# Faire les modifications...

# Après
microk8s inspect -o ~/rapport-apres-changement.tar.gz
```

### 3. Nettoyer les Anciens Rapports

Les rapports peuvent occuper de l'espace. Nettoyez régulièrement :

```bash
# Supprimer les rapports de plus de 30 jours
find ~/audits -name "*.tar.gz" -mtime +30 -delete
```

### 4. Sécuriser les Rapports

Les rapports contiennent des informations sensibles :

```bash
# Restreindre les permissions
chmod 600 ~/rapport-probleme.tar.gz

# Chiffrer si nécessaire
gpg -c ~/rapport-probleme.tar.gz
```

### 5. Combiner avec d'Autres Diagnostics

`microk8s inspect` ne remplace pas tout :

```bash
# 1. inspect pour l'état global
microk8s inspect

# 2. kubectl pour les détails spécifiques
microk8s kubectl describe pod mon-pod

# 3. logs pour l'activité en temps réel
microk8s kubectl logs -f mon-pod
```

## Résolution de Problèmes avec inspect

### Problème : La commande échoue

**Symptôme** :
```
Error: Permission denied
```

**Solution** :
```bash
# Vérifier que vous êtes dans le groupe microk8s
groups

# Si non, ajouter votre utilisateur
sudo usermod -a -G microk8s $USER
newgrp microk8s

# Ou exécuter avec sudo
sudo microk8s inspect
```

### Problème : Rapport trop volumineux

**Symptôme** : Le fichier .tar.gz fait plusieurs centaines de Mo.

**Solution** :
```bash
# Extraire le rapport
tar -xzf rapport-probleme.tar.gz
cd inspection-report-*

# Supprimer les logs les plus volumineux
rm kubernetes/logs/kubelet.log

# Recréer une archive plus légère
tar -czf ../rapport-light.tar.gz .
```

### Problème : Impossible de trouver le rapport

**Symptôme** : Vous ne savez pas où le rapport a été créé.

**Solution** :
```bash
# La commande affiche toujours le chemin en fin d'exécution
# Ou chercher les rapports récents
find /var/snap/microk8s/common/ -name "inspection-report-*.tar.gz" -mtime -1
```

## Résumé et Checklist

### Quand utiliser microk8s inspect ?

- [ ] Problème complexe dont la cause n'est pas évidente
- [ ] Avant de demander de l'aide sur les forums/Discord
- [ ] Avant et après une mise à jour majeure
- [ ] Pour des audits de santé périodiques
- [ ] Après un crash ou incident majeur
- [ ] Quand vous voulez une vue d'ensemble complète

### Workflow recommandé

1. **Générer le rapport** :
   ```bash
   microk8s inspect -o ~/mon-rapport.tar.gz
   ```

2. **Extraire l'archive** :
   ```bash
   tar -xzf ~/mon-rapport.tar.gz
   cd inspection-report-*
   ```

3. **Analyser les sections clés** :
   - Services (tous actifs ?)
   - Pods (tous Running ?)
   - Events (Warnings/Errors ?)
   - System (disque/mémoire suffisants ?)

4. **Chercher les erreurs** :
   ```bash
   grep -r -i "error\|warning\|failed" .
   ```

5. **Documenter et corriger** :
   - Noter les problèmes trouvés
   - Appliquer les corrections
   - Générer un nouveau rapport pour valider

### Points clés à retenir

- `microk8s inspect` est votre outil de diagnostic le plus complet
- Il génère un rapport structuré avec toutes les informations système et Kubernetes
- C'est un instantané ponctuel, pas un monitoring continu
- Utilisez-le en complément de `kubectl`, pas en remplacement
- Générez des rapports régulièrement pour avoir un historique
- Faites attention aux informations sensibles avant de partager

**Prochaine étape** : Section 23.4 - Analyse des logs (kubectl logs)

---


⏭️ [Analyse des logs (kubectl logs)](/23-depannage-et-maintenance/04-analyse-des-logs-kubectl-logs.md)
