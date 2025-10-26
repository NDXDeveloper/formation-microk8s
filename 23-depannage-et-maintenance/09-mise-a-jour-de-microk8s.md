🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 23.9 Mise à Jour de MicroK8s

## Introduction

Maintenir MicroK8s à jour est crucial pour bénéficier des dernières fonctionnalités, des corrections de bugs et surtout des correctifs de sécurité. Cependant, une mise à jour mal préparée peut causer des interruptions de service ou des problèmes de compatibilité. Cette section vous guidera à travers le processus de mise à jour en toute sécurité.

> **Pour les débutants** : Une mise à jour de MicroK8s, c'est comme mettre à jour votre téléphone. Vous obtenez de nouvelles fonctionnalités et des corrections, mais vous devez d'abord sauvegarder vos données importantes et vérifier que vos applications continueront de fonctionner. Dans Kubernetes, c'est la même chose : on prépare, on sauvegarde, on teste, puis on met à jour.

## Comprendre les Versions de MicroK8s

### Structure des Versions

MicroK8s suit le versioning de Kubernetes lui-même :

**Format** : `MAJOR.MINOR/TRACK`

**Exemple** : `1.28/stable`
- **1.28** : Version de Kubernetes
- **stable** : Canal (track) de stabilité

### Les Canaux (Tracks)

MicroK8s utilise des "canaux" pour différents niveaux de stabilité :

#### 1. latest/stable

**Définition** : La version stable la plus récente.

**Caractéristiques** :
- Version la plus à jour
- Testée et stable
- Recommandée pour la plupart des utilisateurs

**Utilisation** :
```bash
sudo snap install microk8s --classic --channel=latest/stable
```

#### 2. X.YZ/stable

**Définition** : Version stable spécifique de Kubernetes.

**Caractéristiques** :
- Version fixe (ex: 1.28, 1.29)
- Reçoit des mises à jour de sécurité
- Pas de changements majeurs
- Recommandé pour la production

**Exemples** :
```bash
# Version 1.28 stable
sudo snap install microk8s --classic --channel=1.28/stable

# Version 1.29 stable
sudo snap install microk8s --classic --channel=1.29/stable
```

#### 3. X.YZ/candidate

**Définition** : Versions candidates pour tests.

**Caractéristiques** :
- Version presque stable
- Tests finaux en cours
- Pour les utilisateurs qui veulent tester avant la stable

**Utilisation** :
```bash
sudo snap install microk8s --classic --channel=1.28/candidate
```

#### 4. X.YZ/beta

**Définition** : Versions beta pour early adopters.

**Caractéristiques** :
- Fonctionnalités nouvelles
- Peut contenir des bugs
- Pour les tests uniquement

#### 5. X.YZ/edge

**Définition** : Versions de développement.

**Caractéristiques** :
- Builds quotidiens
- Instable
- Pour les développeurs et testeurs

**Recommandation** : N'utilisez **JAMAIS** edge ou beta en production !

### Cycle de Vie des Versions

**Kubernetes a un cycle de release de 4 mois** :
- Une nouvelle version mineure tous les ~4 mois
- Chaque version est maintenue pendant ~14 mois
- Les 3 dernières versions reçoivent des correctifs

**Exemple** (au moment de l'écriture) :
- **1.29** : Version actuelle (latest)
- **1.28** : Toujours maintenue
- **1.27** : Toujours maintenue
- **1.26** : Fin de vie bientôt
- **1.25** : Plus de support

**Implication** : Vous devez mettre à jour au moins une fois par an pour rester dans les versions supportées.

## Vérifier la Version Actuelle

### Voir la Version Installée

```bash
# Version de MicroK8s
microk8s version

# Sortie typique :
# MicroK8s v1.28.3 revision 5891
# Kubernetes v1.28.3
```

### Voir le Canal Actuel

```bash
# Informations snap
snap info microk8s | grep tracking

# Sortie :
# tracking:     1.28/stable
```

### Voir les Versions Disponibles

```bash
# Lister tous les canaux disponibles
snap info microk8s

# Sortie partielle :
# channels:
#   latest/stable:    v1.29.0  2024-01-15 (6238) 248MB classic
#   latest/candidate: v1.29.0  2024-01-10 (6234) 248MB classic
#   1.29/stable:      v1.29.0  2024-01-15 (6238) 248MB classic
#   1.28/stable:      v1.28.5  2024-01-10 (6229) 242MB classic
#   1.27/stable:      v1.27.9  2024-01-08 (6222) 237MB classic
```

## Préparation Avant Mise à Jour

### Étape 1 : Sauvegarder les Données Importantes

#### 1.1 Sauvegarder les Configurations

**Exporter toutes les ressources importantes** :

```bash
# Créer un répertoire pour les backups
mkdir -p ~/microk8s-backup-$(date +%Y%m%d)
cd ~/microk8s-backup-$(date +%Y%m%d)

# Exporter tous les namespaces
microk8s kubectl get namespaces -o yaml > namespaces.yaml

# Pour chaque namespace important, exporter les ressources
for ns in default production staging; do
    echo "Backup du namespace $ns"
    mkdir -p $ns

    # Deployments
    microk8s kubectl get deployments -n $ns -o yaml > $ns/deployments.yaml

    # Services
    microk8s kubectl get services -n $ns -o yaml > $ns/services.yaml

    # ConfigMaps
    microk8s kubectl get configmaps -n $ns -o yaml > $ns/configmaps.yaml

    # Secrets
    microk8s kubectl get secrets -n $ns -o yaml > $ns/secrets.yaml

    # PersistentVolumeClaims
    microk8s kubectl get pvc -n $ns -o yaml > $ns/pvc.yaml

    # Ingress
    microk8s kubectl get ingress -n $ns -o yaml > $ns/ingress.yaml
done

# PersistentVolumes (cluster-wide)
microk8s kubectl get pv -o yaml > persistentvolumes.yaml

# StorageClasses
microk8s kubectl get storageclass -o yaml > storageclasses.yaml
```

#### 1.2 Sauvegarder les Données des Volumes

```bash
# Lister les PVC importants
microk8s kubectl get pvc --all-namespaces

# Pour chaque PVC critique, sauvegarder les données
# Exemple pour une base de données PostgreSQL
microk8s kubectl exec -n production postgres-0 -- pg_dumpall -U postgres > postgres-backup.sql

# Exemple pour des fichiers
microk8s kubectl exec -n production app-pod -- tar czf - /data > data-backup.tar.gz
```

#### 1.3 Sauvegarder la Configuration MicroK8s

```bash
# Configuration des addons activés
microk8s status --format yaml > microk8s-status.yaml

# Copier les certificats (optionnel)
sudo cp -r /var/snap/microk8s/current/certs ~/microk8s-backup-$(date +%Y%m%d)/certs-backup
```

### Étape 2 : Documenter l'État Actuel

```bash
# Créer un rapport de l'état du cluster
cat > cluster-state.txt <<EOF
=== État du Cluster Avant Mise à Jour ===
Date: $(date)
Version: $(microk8s version)
Canal: $(snap info microk8s | grep tracking)

=== Nœuds ===
$(microk8s kubectl get nodes -o wide)

=== Pods ===
$(microk8s kubectl get pods --all-namespaces)

=== Services ===
$(microk8s kubectl get services --all-namespaces)

=== PVC ===
$(microk8s kubectl get pvc --all-namespaces)

=== Addons Activés ===
$(microk8s status | grep enabled)

=== Utilisation Ressources ===
Nœuds:
$(microk8s kubectl top nodes 2>/dev/null || echo "Metrics server non disponible")

Pods:
$(microk8s kubectl top pods --all-namespaces 2>/dev/null || echo "Metrics server non disponible")
EOF

cat cluster-state.txt
```

### Étape 3 : Vérifier la Compatibilité

#### 3.1 Consulter les Notes de Version

**Important** : Lisez TOUJOURS les release notes avant de mettre à jour !

```bash
# Ouvrir les release notes de Kubernetes
# https://kubernetes.io/docs/setup/release/notes/

# Chercher :
# - Breaking changes
# - Deprecated APIs
# - Known issues
```

**Éléments critiques à vérifier** :
- APIs dépréciées que vous utilisez
- Changements dans les versions d'addons
- Problèmes connus avec votre configuration

#### 3.2 Tester les Manifests avec kubectl

```bash
# Dry-run pour vérifier la compatibilité
microk8s kubectl apply --dry-run=server -f mon-deployment.yaml

# Si erreurs, corriger avant la mise à jour
```

#### 3.3 Vérifier les Versions d'Addons

Certains addons peuvent avoir des incompatibilités :

```bash
# Voir les versions des addons
microk8s kubectl get deployments --all-namespaces -o wide
```

### Étape 4 : Planifier la Fenêtre de Maintenance

**Recommandations** :
- Planifier pendant les heures creuses
- Prévenir les utilisateurs à l'avance
- Prévoir 1-2 heures (première fois)
- Avoir un plan de rollback
- Être prêt à intervenir en cas de problème

## Processus de Mise à Jour

### Méthode 1 : Mise à Jour Automatique (Recommandée)

#### Mise à Jour vers la Dernière Version du Canal Actuel

**C'est la méthode la plus sûre** : MicroK8s se met à jour automatiquement dans le même canal.

```bash
# Rafraîchir MicroK8s
sudo snap refresh microk8s

# Sortie :
# microk8s (1.28/stable) v1.28.5 from Canonical✓ refreshed
```

**Ce qui se passe** :
1. Le snap télécharge la nouvelle version
2. MicroK8s s'arrête gracieusement
3. La nouvelle version est installée
4. MicroK8s redémarre automatiquement
5. Les pods redémarrent progressivement

**Durée typique** : 2-5 minutes

#### Vérifier que la Mise à Jour est Terminée

```bash
# Attendre que MicroK8s soit prêt
microk8s status --wait-ready

# Vérifier la nouvelle version
microk8s version

# Vérifier que tous les pods système sont Running
microk8s kubectl get pods -n kube-system

# Vérifier vos applications
microk8s kubectl get pods --all-namespaces
```

### Méthode 2 : Changement de Canal (Mise à Jour Majeure)

**Pour passer à une version majeure** (ex: 1.28 → 1.29) :

#### Étape 1 : Changer de Canal

```bash
# Passer au canal de la nouvelle version
sudo snap refresh microk8s --channel=1.29/stable
```

**Attention** : Ceci effectue une mise à jour majeure. Testez d'abord dans un environnement de développement !

#### Étape 2 : Attendre la Fin de la Mise à Jour

```bash
# Surveiller le processus
watch microk8s status

# Ou attendre la fin
microk8s status --wait-ready
```

#### Étape 3 : Vérifier le Cluster

```bash
# Version
microk8s version

# Nœuds
microk8s kubectl get nodes

# Pods système
microk8s kubectl get pods -n kube-system

# Vos applications
microk8s kubectl get pods --all-namespaces
```

### Méthode 3 : Mise à Jour avec Contrôle Manuel

**Pour plus de contrôle** sur le timing :

#### Option A : Désactiver les Mises à Jour Automatiques

```bash
# Empêcher les mises à jour automatiques
sudo snap refresh --hold=forever microk8s

# Pour réactiver plus tard
sudo snap refresh --unhold microk8s
```

#### Option B : Mise à Jour à un Moment Précis

```bash
# Vérifier d'abord les mises à jour disponibles
snap refresh --list

# Effectuer la mise à jour manuellement quand vous êtes prêt
sudo snap refresh microk8s
```

## Mise à Jour d'un Cluster Multi-Nœuds

**Stratégie** : Mettre à jour les nœuds un par un.

### Étape 1 : Mettre à Jour le Premier Nœud (Control Plane)

```bash
# Sur le nœud control plane
sudo snap refresh microk8s

# Attendre qu'il soit prêt
microk8s status --wait-ready

# Vérifier
microk8s kubectl get nodes
```

### Étape 2 : Mettre à Jour les Worker Nodes

**Pour chaque worker node** :

```bash
# Sur le worker node
# 1. Drainer le nœud (évacuer les pods)
microk8s kubectl drain <node-name> --ignore-daemonsets --delete-emptydir-data

# 2. Mettre à jour
sudo snap refresh microk8s

# 3. Attendre que le nœud soit prêt
microk8s status --wait-ready

# 4. Remettre le nœud en service
microk8s kubectl uncordon <node-name>

# 5. Vérifier que les pods reviennent
microk8s kubectl get pods --all-namespaces -o wide | grep <node-name>
```

### Étape 3 : Valider le Cluster

```bash
# Tous les nœuds doivent être Ready
microk8s kubectl get nodes

# Tous les pods doivent être Running
microk8s kubectl get pods --all-namespaces
```

## Problèmes Courants et Solutions

### Problème 1 : MicroK8s Ne Redémarre Pas

**Symptômes** :
```bash
microk8s status
# microk8s is not running
```

**Diagnostic** :

```bash
# Voir les logs snap
sudo journalctl -u snap.microk8s.daemon-kubelite -n 100

# Vérifier l'état des services
snap services microk8s
```

**Solutions** :

#### Solution 1 : Redémarrer Manuellement

```bash
# Arrêter MicroK8s
microk8s stop

# Attendre quelques secondes
sleep 10

# Démarrer MicroK8s
microk8s start

# Vérifier
microk8s status --wait-ready
```

#### Solution 2 : Réinstaller le Snap

Si le redémarrage manuel échoue :

```bash
# Sauvegarder d'abord les données !

# Désinstaller
sudo snap remove microk8s

# Réinstaller avec la version voulue
sudo snap install microk8s --classic --channel=1.28/stable

# Restaurer les configurations si nécessaire
```

### Problème 2 : Pods Ne Redémarrent Pas

**Symptômes** :
```bash
microk8s kubectl get pods
# NAME      READY   STATUS             RESTARTS   AGE
# mon-pod   0/1     ImagePullBackOff   0          5m
```

**Diagnostic** :

```bash
# Voir les détails
microk8s kubectl describe pod mon-pod

# Logs
microk8s kubectl logs mon-pod
```

**Causes courantes** :
- Image non disponible
- Problème de registre
- Erreur de configuration

**Solutions** :

```bash
# Forcer la récréation du pod
microk8s kubectl delete pod mon-pod

# Ou redéployer
microk8s kubectl rollout restart deployment mon-deployment
```

### Problème 3 : APIs Dépréciées

**Symptômes** :
```bash
microk8s kubectl apply -f deployment.yaml
# Warning: resource deployments/v1beta1 is deprecated
# Error: unable to recognize "deployment.yaml": no matches for kind "Deployment" in version "apps/v1beta1"
```

**Cause** : Votre manifeste utilise une API dépréciée.

**Solution** : Mettre à jour les manifests.

**Avant (v1beta1)** :
```yaml
apiVersion: apps/v1beta1
kind: Deployment
```

**Après (v1)** :
```yaml
apiVersion: apps/v1
kind: Deployment
```

**Outils pour identifier les APIs dépréciées** :

```bash
# pluto - outil pour détecter les APIs dépréciées
# Installation
wget https://github.com/FairwindsOps/pluto/releases/download/v5.19.0/pluto_5.19.0_linux_amd64.tar.gz
tar -xzf pluto_5.19.0_linux_amd64.tar.gz
sudo mv pluto /usr/local/bin/

# Scan du cluster
pluto detect-all-in-cluster

# Scan de fichiers
pluto detect-files -d ./manifests/
```

### Problème 4 : Addons Ne Fonctionnent Plus

**Symptômes** :
Après mise à jour, certains addons sont cassés.

**Diagnostic** :

```bash
# Vérifier l'état des addons
microk8s status

# Logs des pods des addons
microk8s kubectl get pods -n kube-system
microk8s kubectl logs -n kube-system <pod-addon>
```

**Solution** :

```bash
# Désactiver l'addon
microk8s disable <addon>

# Attendre quelques secondes
sleep 5

# Réactiver l'addon
microk8s enable <addon>

# Vérifier
microk8s kubectl get pods -n kube-system
```

**Exemple avec DNS** :

```bash
microk8s disable dns
sleep 5
microk8s enable dns
microk8s kubectl wait --for=condition=ready pod -n kube-system -l k8s-app=kube-dns --timeout=60s
```

### Problème 5 : Performance Dégradée

**Symptômes** :
- Cluster lent après mise à jour
- Pods qui mettent du temps à démarrer

**Diagnostic** :

```bash
# Utilisation des ressources
microk8s kubectl top nodes
microk8s kubectl top pods --all-namespaces

# Vérifier les logs système
sudo journalctl -u snap.microk8s.daemon-kubelite -n 100
```

**Solutions** :

```bash
# Redémarrer MicroK8s
microk8s stop
microk8s start

# Nettoyer les images non utilisées
microk8s ctr images rm $(microk8s ctr images ls -q)

# Redémarrer le système (dernier recours)
sudo reboot
```

## Rollback (Retour en Arrière)

Si la mise à jour cause des problèmes, vous pouvez revenir en arrière.

### Méthode 1 : Revert du Snap

**Revenir à la version précédente** :

```bash
# Voir les révisions disponibles
snap list --all microk8s

# Sortie :
# Name      Version   Rev   Tracking      Publisher     Notes
# microk8s  v1.29.0   6238  1.29/stable   canonical✓    classic
# microk8s  v1.28.5   6229  1.28/stable   canonical✓    classic,disabled

# Revenir à une révision précédente
sudo snap revert microk8s --revision 6229

# Vérifier
microk8s version
```

**Attention** : Ceci revient à l'ancienne version binaire, mais ne restaure pas nécessairement vos données.

### Méthode 2 : Changement de Canal

```bash
# Revenir à l'ancien canal
sudo snap refresh microk8s --channel=1.28/stable

# Vérifier
microk8s version
```

### Méthode 3 : Restauration Complète

Si tout est cassé, restaurez depuis votre backup :

```bash
# 1. Désinstaller MicroK8s
sudo snap remove microk8s --purge

# 2. Réinstaller la version stable précédente
sudo snap install microk8s --classic --channel=1.28/stable

# 3. Attendre que le cluster démarre
microk8s status --wait-ready

# 4. Restaurer les configurations
cd ~/microk8s-backup-YYYYMMDD

# Restaurer les namespaces
microk8s kubectl apply -f namespaces.yaml

# Restaurer les ressources par namespace
for ns in default production staging; do
    echo "Restauration du namespace $ns"
    microk8s kubectl apply -f $ns/
done

# 5. Réactiver les addons
microk8s enable dns
microk8s enable hostpath-storage
# ... autres addons selon votre configuration
```

## Validation Post-Mise à Jour

Après une mise à jour réussie, vérifiez systématiquement :

### Checklist de Validation

#### 1. Version et État

```bash
# Version correcte ?
microk8s version

# Cluster Running ?
microk8s status

# Nœuds Ready ?
microk8s kubectl get nodes
```

#### 2. Composants Système

```bash
# Tous les pods système Running ?
microk8s kubectl get pods -n kube-system

# Aucun pod en erreur ?
microk8s kubectl get pods --all-namespaces | grep -v Running | grep -v Completed
```

#### 3. Addons

```bash
# Addons actifs ?
microk8s status | grep enabled

# Tester DNS
microk8s kubectl run test --image=busybox --rm -it --restart=Never -- nslookup kubernetes.default
```

#### 4. Applications

```bash
# Vos applications Running ?
microk8s kubectl get pods --all-namespaces

# Services accessibles ?
microk8s kubectl get services --all-namespaces

# Tester un endpoint
curl http://votre-service/health
```

#### 5. Stockage

```bash
# PVC toujours Bound ?
microk8s kubectl get pvc --all-namespaces

# Données intactes ?
# Vérifier dans un pod avec volume monté
microk8s kubectl exec -it <pod-avec-volume> -- ls -la /data
```

#### 6. Réseau

```bash
# Connectivité pod-to-pod ?
POD1=$(microk8s kubectl get pod -n default -o name | head -1)
POD2=$(microk8s kubectl get pod -n default -o name | tail -1)
microk8s kubectl exec $POD1 -- ping -c 3 $(microk8s kubectl get $POD2 -o jsonpath='{.status.podIP}')
```

#### 7. Ingress (si applicable)

```bash
# Ingress Controller Running ?
microk8s kubectl get pods -n ingress

# Tester l'accès externe
curl -v http://votre-domaine.com
```

### Script de Validation Automatique

```bash
#!/bin/bash
# validate-upgrade.sh

echo "=== Validation Post-Mise à Jour ==="
echo ""

# Couleurs
GREEN='\033[0;32m'
RED='\033[0;31m'
NC='\033[0m' # No Color

function check() {
    if [ $? -eq 0 ]; then
        echo -e "${GREEN}✓${NC} $1"
    else
        echo -e "${RED}✗${NC} $1"
        return 1
    fi
}

# Version
echo "Version:"
microk8s version
echo ""

# Status
microk8s status --wait-ready
check "MicroK8s Running"

# Nœuds
NODES_READY=$(microk8s kubectl get nodes | grep -c " Ready ")
NODES_TOTAL=$(microk8s kubectl get nodes | grep -c "<none>")
if [ "$NODES_READY" -eq "$NODES_TOTAL" ]; then
    check "Tous les nœuds Ready ($NODES_READY/$NODES_TOTAL)"
else
    echo -e "${RED}✗${NC} Nœuds Ready: $NODES_READY/$NODES_TOTAL"
fi

# Pods système
SYSTEM_PODS_RUNNING=$(microk8s kubectl get pods -n kube-system | grep -c "Running")
SYSTEM_PODS_TOTAL=$(microk8s kubectl get pods -n kube-system | grep -v NAME | wc -l)
if [ "$SYSTEM_PODS_RUNNING" -eq "$SYSTEM_PODS_TOTAL" ]; then
    check "Tous les pods système Running ($SYSTEM_PODS_RUNNING/$SYSTEM_PODS_TOTAL)"
else
    echo -e "${RED}✗${NC} Pods système Running: $SYSTEM_PODS_RUNNING/$SYSTEM_PODS_TOTAL"
fi

# Pods application
APP_PODS_ISSUES=$(microk8s kubectl get pods --all-namespaces | grep -v "Running\|Completed" | grep -v NAME | wc -l)
if [ "$APP_PODS_ISSUES" -eq 0 ]; then
    check "Aucun pod en erreur"
else
    echo -e "${RED}✗${NC} $APP_PODS_ISSUES pod(s) en erreur"
    microk8s kubectl get pods --all-namespaces | grep -v "Running\|Completed"
fi

# DNS
microk8s kubectl run test-dns-$$ --image=busybox --rm -it --restart=Never -- nslookup kubernetes.default > /dev/null 2>&1
check "DNS fonctionne"

# PVC
PVC_NOT_BOUND=$(microk8s kubectl get pvc --all-namespaces | grep -v Bound | grep -v NAME | wc -l)
if [ "$PVC_NOT_BOUND" -eq 0 ]; then
    check "Tous les PVC Bound"
else
    echo -e "${RED}✗${NC} $PVC_NOT_BOUND PVC non Bound"
fi

echo ""
echo "=== Validation Terminée ==="
```

**Utilisation** :

```bash
chmod +x validate-upgrade.sh
./validate-upgrade.sh
```

## Bonnes Pratiques

### 1. Tester d'Abord dans un Environnement de Dev

❌ **Ne faites JAMAIS** :
```bash
# Mise à jour directe en production
sudo snap refresh microk8s --channel=1.29/stable
```

✅ **Faites TOUJOURS** :
```bash
# 1. Tester dans dev d'abord
# 2. Valider les applications
# 3. Corriger les problèmes
# 4. Puis mettre à jour production
```

### 2. Suivre un Canal Stable Spécifique

✅ **Recommandé pour production** :
```bash
sudo snap refresh microk8s --channel=1.28/stable
```

**Pourquoi** :
- Contrôle sur les versions
- Pas de surprises
- Temps de préparer les mises à jour majeures

### 3. Maintenir une Cadence de Mise à Jour

**Recommandation** :
- **Mises à jour mineures** : Mensuel (correctifs de sécurité)
- **Mises à jour majeures** : Tous les 6-12 mois
- Ne pas rester sur une version plus de 12 mois

### 4. Sauvegarder Avant Chaque Mise à Jour

**Systématiquement** :
```bash
# Script de backup automatique
cat > backup-before-upgrade.sh <<'EOF'
#!/bin/bash
BACKUP_DIR=~/microk8s-backup-$(date +%Y%m%d-%H%M%S)
mkdir -p $BACKUP_DIR

echo "Backup dans $BACKUP_DIR"

# État actuel
microk8s version > $BACKUP_DIR/version.txt
microk8s status --format yaml > $BACKUP_DIR/status.yaml

# Ressources
for ns in $(microk8s kubectl get ns -o jsonpath='{.items[*].metadata.name}'); do
    mkdir -p $BACKUP_DIR/$ns
    microk8s kubectl get all,cm,secrets,pvc,ingress -n $ns -o yaml > $BACKUP_DIR/$ns/all.yaml
done

echo "Backup terminé: $BACKUP_DIR"
EOF

chmod +x backup-before-upgrade.sh
./backup-before-upgrade.sh
```

### 5. Planifier et Communiquer

**Avant la mise à jour** :
1. Annoncer la maintenance à l'avance
2. Choisir une fenêtre appropriée
3. Préparer un plan de rollback
4. Assigner des responsabilités

**Template d'annonce** :
```
Objet: Maintenance MicroK8s - [DATE]

Bonjour,

Une mise à jour de MicroK8s est planifiée :
- Date : [Date et heure]
- Durée estimée : 30 minutes
- Impact : Interruption brève des services
- Version actuelle : 1.28.5
- Version cible : 1.29.0

Actions :
- Backup effectué : Oui
- Tests préalables : Effectués en dev
- Plan de rollback : Prêt

Contact en cas de problème : [Contact]
```

### 6. Surveiller Après la Mise à Jour

**Les 24 premières heures** :
```bash
# Surveiller les ressources
watch -n 30 microk8s kubectl top nodes
watch -n 30 microk8s kubectl top pods --all-namespaces

# Surveiller les logs
microk8s kubectl logs -f -n kube-system -l k8s-app=kube-dns

# Vérifier les événements
watch microk8s kubectl get events --all-namespaces --sort-by='.lastTimestamp'
```

### 7. Documenter la Mise à Jour

**Tenir un journal** :
```markdown
# Mise à Jour MicroK8s - 2025-10-26

## Informations
- Version source : 1.28.5
- Version cible : 1.29.0
- Date/Heure : 2025-10-26 14:00
- Responsable : [Nom]

## Préparation
- [x] Backup effectué
- [x] Tests en dev réussis
- [x] Release notes lues
- [x] Équipe prévenue

## Exécution
- 14:00 - Début de la mise à jour
- 14:02 - Snap refresh lancé
- 14:05 - MicroK8s redémarré
- 14:07 - Validation en cours
- 14:10 - Mise à jour terminée

## Problèmes Rencontrés
- Pod DNS lent à redémarrer (résolu après 2 min)

## Validation
- [x] Tous les pods Running
- [x] DNS fonctionne
- [x] Applications accessibles
- [x] Stockage intact

## Notes
- RAS - Mise à jour fluide
- Durée totale : 10 minutes
```

### 8. Automatiser Quand Possible

**Mise à jour automatique des patches** :

```bash
# Pour recevoir automatiquement les correctifs de sécurité
# (reste dans le même canal)
sudo snap set microk8s auto-refresh=true
```

**Mais bloquer les mises à jour majeures** :
```bash
# Rester sur 1.28/stable, pas de saut vers 1.29
sudo snap refresh microk8s --channel=1.28/stable
```

## Résumé

### Commandes Essentielles

```bash
# Vérifier la version actuelle
microk8s version
snap info microk8s | grep tracking

# Voir les versions disponibles
snap info microk8s

# Mise à jour dans le canal actuel
sudo snap refresh microk8s

# Changement de canal
sudo snap refresh microk8s --channel=1.29/stable

# Rollback
sudo snap revert microk8s
sudo snap revert microk8s --revision <num>

# Validation
microk8s status --wait-ready
microk8s kubectl get nodes
microk8s kubectl get pods --all-namespaces
```

### Workflow de Mise à Jour

```
1. PRÉPARATION
   ├── Backup des données
   ├── Documentation de l'état actuel
   ├── Lecture des release notes
   └── Tests en environnement de dev

2. EXÉCUTION
   ├── Fenêtre de maintenance
   ├── sudo snap refresh microk8s
   └── Attendre la fin (5-10 min)

3. VALIDATION
   ├── Vérifier la version
   ├── Vérifier les nœuds
   ├── Vérifier les pods
   ├── Tester les applications
   └── Surveiller 24h

4. ROLLBACK (si problème)
   ├── sudo snap revert microk8s
   └── Restaurer depuis backup
```

### Canaux Recommandés

| Environnement | Canal Recommandé | Raison |
|---------------|------------------|--------|
| Production | `1.XX/stable` | Stable, prévisible, contrôlé |
| Staging | `latest/stable` | Tester les nouvelles versions |
| Développement | `latest/stable` ou `candidate` | Tester en avance |
| Test/Lab | `beta` ou `edge` | Tester les nouvelles fonctionnalités |

### Points Clés à Retenir

1. **TOUJOURS sauvegarder** avant une mise à jour
2. **Tester en dev** avant production
3. **Lire les release notes** pour éviter les surprises
4. **Utiliser un canal stable spécifique** en production (ex: 1.28/stable)
5. **Planifier et communiquer** la maintenance
6. **Valider complètement** après la mise à jour
7. **Avoir un plan de rollback** prêt
8. **Mettre à jour régulièrement** (ne pas rester trop en retard)

### Fréquence de Mise à Jour Recommandée

- **Correctifs de sécurité** : Dès disponibles (automatique dans le canal)
- **Mises à jour mineures** : Mensuel
- **Mises à jour majeures** : 1-2 fois par an
- **Maximum sans mise à jour** : 12 mois

**Prochaine étape** : Section 23.10 - Nettoyage et maintenance

---


⏭️ [Nettoyage et maintenance](/23-depannage-et-maintenance/10-nettoyage-et-maintenance.md)
