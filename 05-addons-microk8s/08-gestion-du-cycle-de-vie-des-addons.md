🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 5.8 Gestion du cycle de vie des addons (enable/disable)

## Introduction

Vous savez maintenant ce que sont les addons et quels addons sont disponibles. Cette section vous apprend à **gérer** vos addons au quotidien : les activer, les désactiver, vérifier leur état, comprendre leurs dépendances, résoudre les problèmes, et appliquer les bonnes pratiques.

Considérez cette section comme le "manuel d'entretien" de vos addons : vous apprendrez non seulement à les utiliser, mais aussi à les maintenir en bon état de fonctionnement.

## Le cycle de vie d'un addon

Un addon passe par différents états au cours de son existence dans votre cluster :

### 1. État : Disponible (Available)

**Description :** L'addon existe dans le catalogue mais n'est pas installé.

**Comment le voir :**
```bash
microk8s status
```

Il apparaît dans la section "disabled".

**Ressources utilisées :** Aucune (0 CPU, 0 RAM)

### 2. État : En cours d'activation (Enabling)

**Description :** Vous venez de lancer `microk8s enable`, l'addon s'installe.

**Ce qui se passe :**
- Téléchargement des images Docker
- Création des ressources Kubernetes
- Configuration des composants
- Vérification que tout démarre correctement

**Durée :** Variable selon l'addon (10 secondes à 10 minutes)

### 3. État : Activé (Enabled)

**Description :** L'addon est installé et fonctionne.

**Comment le voir :**
```bash
microk8s status
```

Il apparaît dans la section "enabled".

**Ressources utilisées :** Variables selon l'addon

**Vérification :**
```bash
# Voir les pods de l'addon
microk8s kubectl get pods -A | grep <nom-addon>
```

### 4. État : Défaillant (Failed)

**Description :** L'addon est activé mais ne fonctionne pas correctement.

**Symptômes :**
- Pods en état CrashLoopBackOff, Error, ou ImagePullBackOff
- Services non accessibles
- Erreurs dans les logs

**Diagnostic :**
```bash
microk8s inspect
```

### 5. État : En cours de désactivation (Disabling)

**Description :** Vous venez de lancer `microk8s disable`, l'addon se désinstalle.

**Ce qui se passe :**
- Suppression des ressources Kubernetes
- Nettoyage des configurations
- Libération des ressources

**Durée :** Généralement rapide (quelques secondes)

### 6. État : Désactivé (Disabled)

**Description :** L'addon est désinstallé, retour à l'état "Disponible".

**Ressources utilisées :** Aucune

**Note :** Certains addons peuvent laisser des données (PV, ConfigMaps) même après désactivation.

## Activer un addon : Les détails

Vous connaissez déjà la commande de base, mais approfondissons.

### Syntaxe complète

```bash
microk8s enable <addon-name>[:<option1>[,<option2>...]]
```

**Décomposition :**
- `microk8s` : Le programme
- `enable` : L'action d'activation
- `<addon-name>` : Le nom de l'addon
- `:<option>` : Options de configuration (facultatif)

### Exemples d'activation

**Sans options :**
```bash
microk8s enable dns
microk8s enable dashboard
```

**Avec une option :**
```bash
microk8s enable registry:size=40Gi
microk8s enable metallb:192.168.1.100-192.168.1.110
```

**Avec plusieurs options (séparées par virgules) :**
```bash
# Exemple hypothétique
microk8s enable monitoring:storage=20Gi,retention=7d
```

### Activer plusieurs addons en une commande

Vous pouvez activer plusieurs addons simultanément :

```bash
microk8s enable dns dashboard storage registry
```

**Avantage :** Gain de temps

**Inconvénient :** Si un addon échoue, difficile de savoir lequel

**Recommandation pour débutants :** Activez un par un au début, pour bien comprendre ce que chaque addon fait.

### Activation avec confirmation

MicroK8s active les addons sans demander confirmation. Si vous voulez être sûr :

```bash
echo "Activation du dashboard..."
microk8s enable dashboard
echo "Dashboard activé !"
```

Ou dans un script :
```bash
read -p "Activer le dashboard ? (o/n) " -n 1 -r
echo
if [[ $REPLY =~ ^[Oo]$ ]]
then
    microk8s enable dashboard
fi
```

### Vérifier que l'activation a réussi

Après avoir activé un addon, vérifiez son état :

**Méthode 1 : Status général**
```bash
microk8s status
```

Cherchez l'addon dans la liste "enabled".

**Méthode 2 : Pods de l'addon**
```bash
microk8s kubectl get pods -A
```

Cherchez les pods liés à l'addon (généralement dans un namespace spécifique).

**Méthode 3 : Logs de l'addon**
```bash
microk8s kubectl logs -n <namespace> <pod-name>
```

**Méthode 4 : Inspection complète**
```bash
microk8s inspect
```

Génère un rapport complet incluant l'état de tous les composants.

### Temps d'attente après activation

Certains addons mettent du temps à être complètement prêts. Soyez patient :

**Addons rapides (< 1 minute) :**
- dns
- rbac
- host-access
- metrics-server

**Addons moyens (1-3 minutes) :**
- dashboard
- storage
- registry
- ingress

**Addons longs (3-10 minutes) :**
- prometheus
- cert-manager
- istio
- observability

**Astuce :** Utilisez `watch` pour surveiller en temps réel :
```bash
watch microk8s kubectl get pods -A
```

Appuyez sur `Ctrl+C` pour arrêter la surveillance.

## Désactiver un addon

Désactiver un addon le supprime du cluster et libère les ressources.

### Syntaxe

```bash
microk8s disable <addon-name>
```

**Exemple :**
```bash
microk8s disable dashboard
```

### Que se passe-t-il lors de la désactivation ?

1. **Les ressources Kubernetes sont supprimées**
   - Deployments
   - Services
   - ConfigMaps
   - Secrets (parfois)

2. **Les pods sont terminés**
   - Kubernetes les arrête proprement
   - Les conteneurs se ferment

3. **Certaines données peuvent être conservées**
   - PersistentVolumes (selon la Reclaim Policy)
   - Données applicatives

4. **L'addon revient à l'état "disabled"**
   - Il peut être réactivé ultérieurement

### ⚠️ Attention : Perte de données

Désactiver certains addons peut entraîner une **perte de données** :

**Addons sans risque de perte :**
- dns (pas de données)
- dashboard (pas de données persistées)
- ingress (configuration recréable)
- metrics-server (métriques temporaires)

**Addons avec risque de perte :**
- **storage** : Les PV sont supprimés (avec les données !)
- **registry** : Vos images privées sont perdues
- **prometheus** : Historique des métriques perdu
- **observability** : Logs et traces perdus

**Avant de désactiver un addon avec données :**
1. **Sauvegardez** les données importantes
2. **Vérifiez** que vous n'en avez plus besoin
3. **Documentez** la procédure de restauration si nécessaire

### Exemple : Désactiver proprement le registry

```bash
# 1. Lister vos images
curl http://localhost:32000/v2/_catalog

# 2. Sauvegarder les images importantes
docker pull localhost:32000/my-app:1.0
docker tag localhost:32000/my-app:1.0 my-app:1.0-backup
docker save my-app:1.0-backup > my-app-backup.tar

# 3. Désactiver le registry
microk8s disable registry

# 4. Si besoin de restaurer plus tard
microk8s enable registry
docker load < my-app-backup.tar
docker tag my-app:1.0-backup localhost:32000/my-app:1.0
docker push localhost:32000/my-app:1.0
```

### Désactiver plusieurs addons

```bash
microk8s disable prometheus dashboard
```

**Ordre recommandé :** Désactivez dans l'ordre **inverse** de l'activation si possible.

**Exemple :**
```bash
# Activés dans cet ordre :
microk8s enable dns
microk8s enable storage
microk8s enable registry

# Désactiver dans l'ordre inverse :
microk8s disable registry    # Dépend de storage
microk8s disable storage
microk8s disable dns         # Addon le plus fondamental
```

### Vérifier qu'un addon est bien désactivé

```bash
microk8s status
```

L'addon doit apparaître dans "disabled".

Vérifiez aussi que les pods sont partis :
```bash
microk8s kubectl get pods -A
```

## Dépendances entre addons

Certains addons dépendent d'autres. MicroK8s gère généralement ces dépendances automatiquement.

### Dépendances automatiques

Quand vous activez un addon, ses dépendances sont activées automatiquement.

**Exemple 1 : Dashboard**
```bash
microk8s enable dashboard
```
Active automatiquement :
- metrics-server (pour les graphiques de ressources)

**Exemple 2 : Registry**
```bash
microk8s enable registry
```
Active automatiquement :
- storage (pour stocker les images)

**Exemple 3 : Observability**
```bash
microk8s enable observability
```
Active automatiquement :
- dns
- storage

**Vous n'avez rien à faire, MicroK8s s'en occupe !**

### Dépendances recommandées (non automatiques)

Certains addons fonctionnent mieux avec d'autres, mais la dépendance n'est pas stricte.

**Exemple : Ingress + Cert-Manager**

Ingress fonctionne seul, mais avec cert-manager, vous obtenez des certificats HTTPS automatiques :
```bash
microk8s enable ingress
microk8s enable cert-manager  # Recommandé mais pas obligatoire
```

**Exemple : MetalLB + Ingress**

Les deux sont souvent utilisés ensemble pour exposer des applications :
```bash
microk8s enable metallb:192.168.1.100-192.168.1.110
microk8s enable ingress
```

### Vérifier les dépendances d'un addon

Il n'y a pas de commande directe pour lister les dépendances, mais :

1. **Lisez la documentation de l'addon**
   ```
   https://microk8s.io/docs/addon-<nom>
   ```

2. **Observez les messages d'activation**

   Quand vous activez un addon, MicroK8s affiche les dépendances activées automatiquement.

3. **Consultez les prérequis**

   Certains addons ont des prérequis système (GPU, drivers, etc.)

### Problème : Dépendances circulaires

Généralement, il n'y a pas de dépendances circulaires dans MicroK8s, mais si vous rencontrez un problème :

**Symptôme :** "Cannot enable A because B is not enabled, but B requires A"

**Solution :** C'est rare, mais si cela arrive, consultez la documentation de l'addon ou le forum MicroK8s.

## Réactiver un addon

Si un addon ne fonctionne pas correctement, le réactiver peut résoudre le problème.

### Procédure de réactivation

```bash
# 1. Désactiver l'addon
microk8s disable <addon>

# 2. Attendre quelques secondes
sleep 5

# 3. Réactiver l'addon
microk8s enable <addon>
```

**Exemple avec le dashboard :**
```bash
microk8s disable dashboard
sleep 5
microk8s enable dashboard
```

### Réactivation avec script

Pour automatiser :

```bash
#!/bin/bash
ADDON=$1

echo "Réactivation de $ADDON..."

echo "Désactivation..."
microk8s disable $ADDON

echo "Attente de 10 secondes..."
sleep 10

echo "Réactivation..."
microk8s enable $ADDON

echo "Vérification..."
microk8s status | grep $ADDON

echo "Fait !"
```

Utilisation :
```bash
chmod +x reactivate-addon.sh
./reactivate-addon.sh dashboard
```

### Quand réactiver un addon ?

**Situations où la réactivation aide :**
- Pods en CrashLoopBackOff sans raison apparente
- Addon partiellement fonctionnel
- Après une mise à jour de MicroK8s
- Configuration corrompue

**Situations où la réactivation n'aide pas :**
- Manque de ressources (CPU, RAM, disque)
- Problème réseau
- Dépendances manquantes
- Problème de configuration externe (DNS, firewall)

Dans ces cas, il faut diagnostiquer et résoudre le problème sous-jacent.

## Mise à jour des addons

Les addons sont mis à jour en même temps que MicroK8s.

### Comment MicroK8s met à jour les addons

Quand vous mettez à jour MicroK8s :

```bash
sudo snap refresh microk8s --channel=1.28/stable
```

Les addons sont mis à jour automatiquement vers les versions compatibles avec la nouvelle version de Kubernetes.

### Addons activés restent activés

**Important :** Quand vous mettez à jour MicroK8s, les addons déjà activés restent activés.

**Exemple :**
1. Vous avez MicroK8s 1.27 avec dns, dashboard, storage activés
2. Vous mettez à jour vers MicroK8s 1.28
3. dns, dashboard, storage restent activés (mis à jour automatiquement)

### Que faire après une mise à jour ?

**Vérifiez que tout fonctionne :**

```bash
# 1. Vérifier l'état général
microk8s status

# 2. Vérifier les pods
microk8s kubectl get pods -A

# 3. Inspection complète
microk8s inspect
```

**Si un addon ne fonctionne plus après la mise à jour :**

```bash
# Réactivez-le
microk8s disable <addon>
microk8s enable <addon>
```

### Versions des addons

Les versions des addons ne sont pas affichées directement, mais vous pouvez voir les versions des images utilisées :

```bash
microk8s kubectl get pods -A -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.spec.containers[*].image}{"\n"}{end}'
```

Cela affiche tous les pods et leurs images.

## Vérifier l'état des addons

Plusieurs commandes permettent de vérifier l'état de vos addons.

### Commande status

**La plus simple :**
```bash
microk8s status
```

**Sortie formatée en YAML :**
```bash
microk8s status --format yaml
```

**Sortie formatée en JSON :**
```bash
microk8s status --format json
```

**Attendre que microk8s soit prêt :**
```bash
microk8s status --wait-ready
```

Cette commande bloque jusqu'à ce que MicroK8s soit complètement démarré. Utile dans les scripts.

### Vérifier les pods des addons

```bash
# Tous les pods, tous les namespaces
microk8s kubectl get pods -A

# Filtrer par namespace spécifique
microk8s kubectl get pods -n kube-system
microk8s kubectl get pods -n container-registry

# Afficher plus de détails
microk8s kubectl get pods -A -o wide
```

### Vérifier les services des addons

```bash
# Tous les services
microk8s kubectl get services -A

# Services d'un namespace
microk8s kubectl get services -n kube-system
```

### Logs des addons

Pour voir ce qui se passe dans un addon :

```bash
# Logs d'un pod spécifique
microk8s kubectl logs -n <namespace> <pod-name>

# Suivre les logs en temps réel
microk8s kubectl logs -n <namespace> <pod-name> -f

# Logs des conteneurs précédents (si le pod a redémarré)
microk8s kubectl logs -n <namespace> <pod-name> --previous
```

**Exemple avec CoreDNS :**
```bash
# Trouver le pod
microk8s kubectl get pods -n kube-system -l k8s-app=kube-dns

# Voir ses logs
microk8s kubectl logs -n kube-system -l k8s-app=kube-dns
```

### Décrire une ressource

Pour des détails complets sur une ressource (pod, service, deployment) :

```bash
microk8s kubectl describe pod <pod-name> -n <namespace>
```

La section "Events" en bas est particulièrement utile pour le diagnostic.

### Inspection complète

La commande la plus complète :

```bash
microk8s inspect
```

**Génère un rapport avec :**
- État de tous les services MicroK8s
- État de tous les pods
- Logs récents
- Configuration réseau
- Problèmes détectés
- Suggestions de résolution

**Très utile pour le dépannage !**

Le rapport est enregistré dans un fichier :
```
Inspection report written to /var/snap/microk8s/common/inspection-report-YYYYMMDD_HHMMSS.tar.gz
```

Vous pouvez extraire et lire ce fichier :
```bash
cd /tmp
tar -xzf /var/snap/microk8s/common/inspection-report-*.tar.gz
less inspection-report/inspection.txt
```

## Résolution des problèmes courants

### Problème 1 : Addon reste "disabled" après enable

**Symptôme :**
```bash
microk8s enable dashboard
# Commande se termine sans erreur

microk8s status
# dashboard apparaît toujours en "disabled"
```

**Causes possibles :**
1. Erreur silencieuse lors de l'activation
2. Problème de permissions
3. MicroK8s pas complètement démarré

**Solutions :**

**A. Vérifier les logs détaillés :**
```bash
sudo journalctl -u snap.microk8s.daemon-kubelite -n 100
```

**B. Réessayer avec sudo :**
```bash
sudo microk8s enable dashboard
```

**C. Redémarrer MicroK8s :**
```bash
microk8s stop
microk8s start
microk8s enable dashboard
```

### Problème 2 : Pods en CrashLoopBackOff

**Symptôme :**
```bash
microk8s kubectl get pods -A
```
```
NAMESPACE     NAME                    READY   STATUS             RESTARTS
kube-system   coredns-xxx             0/1     CrashLoopBackOff   5
```

**Causes possibles :**
1. Erreur de configuration
2. Ressources insuffisantes
3. Dépendances manquantes

**Solutions :**

**A. Voir les logs du pod :**
```bash
microk8s kubectl logs -n kube-system coredns-xxx
microk8s kubectl logs -n kube-system coredns-xxx --previous
```

**B. Décrire le pod :**
```bash
microk8s kubectl describe pod -n kube-system coredns-xxx
```

Regardez la section "Events" pour des indices.

**C. Vérifier les ressources :**
```bash
df -h
free -h
microk8s kubectl top nodes
```

**D. Réactiver l'addon :**
```bash
microk8s disable dns
microk8s enable dns
```

### Problème 3 : Impossible de désactiver un addon

**Symptôme :**
```bash
microk8s disable dashboard
# Commande bloque ou erreur
```

**Solutions :**

**A. Forcer l'arrêt si la commande bloque :**
Appuyez sur `Ctrl+C` et réessayez.

**B. Supprimer manuellement les ressources :**
```bash
# Trouver les ressources de l'addon
microk8s kubectl get all -n kube-system | grep dashboard

# Supprimer manuellement
microk8s kubectl delete deployment kubernetes-dashboard -n kube-system
microk8s kubectl delete service kubernetes-dashboard -n kube-system
```

**C. Redémarrer MicroK8s et réessayer :**
```bash
microk8s stop
microk8s start
microk8s disable dashboard
```

### Problème 4 : Addon consomme trop de ressources

**Symptôme :** Votre machine ralentit après avoir activé un addon.

**Diagnostic :**
```bash
microk8s kubectl top pods -A
```

**Solutions :**

**A. Limiter les ressources de l'addon (avancé) :**

Éditez le deployment de l'addon pour ajouter des resource limits :
```bash
microk8s kubectl edit deployment <addon-deployment> -n <namespace>
```

Ajoutez :
```yaml
resources:
  limits:
    cpu: 500m
    memory: 512Mi
  requests:
    cpu: 100m
    memory: 128Mi
```

**B. Désactiver l'addon si non essentiel :**
```bash
microk8s disable <addon>
```

**C. Augmenter les ressources de votre machine :**

Si vous utilisez beaucoup d'addons, vous aurez besoin de plus de RAM/CPU.

### Problème 5 : Conflits entre addons

**Symptôme :** Deux addons qui ne fonctionnent pas bien ensemble.

**Exemple courant :** Deux ingress controllers (nginx et traefik)

**Solution :** N'utilisez qu'un seul ingress controller à la fois :
```bash
# Choisir entre :
microk8s enable ingress     # NGINX
# OU
microk8s enable traefik     # Traefik

# Pas les deux !
```

### Problème 6 : Addons manquants après redémarrage système

**Symptôme :** Après un reboot de votre machine, certains addons ne sont plus actifs.

**Cause :** MicroK8s ne démarre pas automatiquement au boot du système.

**Solution permanente :**

Ajoutez MicroK8s au démarrage automatique :
```bash
# Sur les systèmes avec systemd
sudo systemctl enable snap.microk8s.daemon-kubelite.service
```

Ou, après chaque reboot, démarrez MicroK8s :
```bash
microk8s start
```

## Bonnes pratiques de gestion des addons

### 1. Documentez vos addons

Créez un fichier `ADDONS.md` dans votre projet :

```markdown
# Addons actifs sur ce cluster

## Essentiels
- dns : Résolution de noms
- dashboard : Visualisation
- storage : Persistance des données
- registry : Images privées

## Réseau
- ingress : Exposer les applications web
- cert-manager : Certificats HTTPS automatiques
- metallb:192.168.1.100-192.168.1.110 : Load balancer

## Monitoring
- prometheus : Métriques et alertes

## Notes
- Prometheus consomme ~2GB de RAM
- Le registry est configuré avec 40Gi de stockage
```

### 2. Activez progressivement

N'activez pas 10 addons d'un coup. Procédez par étapes :

**Étape 1 : Les essentiels**
```bash
microk8s enable dns
microk8s enable dashboard
microk8s enable storage
```

Testez, assurez-vous que tout fonctionne.

**Étape 2 : Selon vos besoins**
```bash
microk8s enable registry
microk8s enable ingress
```

Testez à nouveau.

**Étape 3 : Avancé**
```bash
microk8s enable prometheus
```

### 3. Vérifiez après chaque activation

Prenez l'habitude de vérifier systématiquement :

```bash
microk8s enable <addon>
sleep 10
microk8s status
microk8s kubectl get pods -A
```

Ne passez à l'addon suivant que quand tout fonctionne.

### 4. Créez un script de setup

Pour recréer facilement votre environnement :

**Fichier `setup-cluster.sh` :**
```bash
#!/bin/bash

echo "Configuration du cluster MicroK8s..."

echo "1. Activation des addons essentiels..."
microk8s enable dns
sleep 5
microk8s enable dashboard
sleep 10
microk8s enable storage
sleep 10
microk8s enable registry
sleep 30

echo "2. Activation du réseau..."
microk8s enable ingress
sleep 30
microk8s enable cert-manager
sleep 30

echo "3. Configuration MetalLB..."
microk8s enable metallb:192.168.1.100-192.168.1.110
sleep 10

echo "4. Vérification finale..."
microk8s status

echo "Setup terminé ! Voici l'état du cluster :"
microk8s kubectl get pods -A

echo ""
echo "Accès au dashboard :"
echo "microk8s dashboard-proxy"
```

Rendez-le exécutable et lancez-le :
```bash
chmod +x setup-cluster.sh
./setup-cluster.sh
```

### 5. Surveillez les ressources

Après avoir activé plusieurs addons, surveillez régulièrement :

```bash
# Utilisation par les nodes
microk8s kubectl top nodes

# Utilisation par les pods
microk8s kubectl top pods -A

# Espace disque
df -h

# Mémoire système
free -h
```

### 6. Sauvegardez avant de désactiver

Avant de désactiver un addon avec des données :

```bash
# Exemple avec Prometheus
# Sauvegarder les données (selon l'addon)
microk8s kubectl get pvc -n monitoring

# Puis désactiver
microk8s disable prometheus
```

### 7. Testez la résilience

Occasionnellement, testez que vos addons redémarrent correctement :

```bash
# Redémarrer MicroK8s
microk8s stop
microk8s start

# Vérifier que tout revient
microk8s status
microk8s kubectl get pods -A
```

Cela simule un redémarrage et vous assure que votre configuration est solide.

### 8. Utilisez des alias

Créez des alias pour les opérations courantes :

```bash
# Ajoutez dans votre ~/.bashrc ou ~/.zshrc
alias mk='microk8s'
alias mks='microk8s status'
alias mke='microk8s enable'
alias mkd='microk8s disable'
alias mkk='microk8s kubectl'

# Puis rechargez
source ~/.bashrc

# Utilisation :
mke dns
mks
mkk get pods -A
```

### 9. Gardez MicroK8s à jour

Mettez régulièrement à jour MicroK8s pour bénéficier des dernières versions des addons :

```bash
# Vérifier les mises à jour disponibles
snap info microk8s

# Mettre à jour (vers la même version majeure)
sudo snap refresh microk8s

# Ou changer de canal (version majeure)
sudo snap refresh microk8s --channel=1.28/stable
```

### 10. Nettoyez les addons inutilisés

Si vous ne utilisez plus un addon, désactivez-le pour libérer des ressources :

```bash
# Lister les addons actifs
microk8s status | grep enabled

# Désactiver ceux qui ne servent plus
microk8s disable <addon-inutilise>
```

## Automatisation avancée

Pour les utilisateurs qui gèrent plusieurs clusters ou réinstallent souvent MicroK8s.

### Script de configuration avec vérification

```bash
#!/bin/bash

# Fonction pour activer un addon avec vérification
enable_addon() {
    local addon=$1
    local wait_time=${2:-30}

    echo "Activation de $addon..."
    microk8s enable $addon

    echo "Attente de $wait_time secondes..."
    sleep $wait_time

    echo "Vérification..."
    if microk8s status | grep -q "$addon.*enabled"; then
        echo "✓ $addon activé avec succès"
        return 0
    else
        echo "✗ Échec de l'activation de $addon"
        return 1
    fi
}

# Activer les addons un par un
enable_addon "dns" 10
enable_addon "dashboard" 30
enable_addon "storage" 20
enable_addon "registry" 30

echo "Configuration terminée !"
microk8s status
```

### Export/Import de configuration

Malheureusement, MicroK8s ne propose pas de commande native pour exporter/importer la configuration des addons, mais vous pouvez créer un script :

**Exporter :**
```bash
#!/bin/bash
# export-addons.sh

echo "# Liste des addons actifs" > addons-backup.txt
microk8s status --format yaml | grep "enabled:" -A 100 | grep "name:" | sed 's/.*name: //' >> addons-backup.txt

echo "Configuration des addons exportée dans addons-backup.txt"
cat addons-backup.txt
```

**Importer :**
```bash
#!/bin/bash
# import-addons.sh

if [ ! -f "addons-backup.txt" ]; then
    echo "Fichier addons-backup.txt non trouvé"
    exit 1
fi

while IFS= read -r addon; do
    if [ ! -z "$addon" ] && [ "$addon" != "#"* ]; then
        echo "Activation de $addon..."
        microk8s enable $addon
        sleep 15
    fi
done < addons-backup.txt

echo "Import terminé !"
```

## Récapitulatif

La gestion du cycle de vie des addons comprend :

✅ **Activation** : `microk8s enable <addon>`
✅ **Désactivation** : `microk8s disable <addon>`
✅ **Vérification** : `microk8s status`
✅ **Surveillance** : `microk8s kubectl get pods -A`
✅ **Diagnostic** : `microk8s inspect`
✅ **Réactivation** : disable puis enable
✅ **Documentation** : Gardez une trace de ce qui est activé
✅ **Automatisation** : Créez des scripts pour reproduire votre setup

**Commandes essentielles à retenir :**
```bash
# État général
microk8s status

# Activer
microk8s enable <addon>

# Désactiver
microk8s disable <addon>

# Voir les pods
microk8s kubectl get pods -A

# Diagnostic complet
microk8s inspect

# Réactiver (si problème)
microk8s disable <addon> && sleep 5 && microk8s enable <addon>
```

## Ce que vous avez appris

Dans cette section, vous avez découvert :
- Le cycle de vie complet d'un addon (de disponible à activé à désactivé)
- Comment activer et désactiver des addons correctement
- La gestion des dépendances entre addons
- Comment vérifier l'état des addons de multiples façons
- Les problèmes courants et leurs solutions
- Les bonnes pratiques pour gérer vos addons au quotidien
- Comment automatiser la configuration de votre cluster
- Les stratégies de sauvegarde et de restauration

## Conclusion du Chapitre 5

**Félicitations !** Vous avez terminé le chapitre 5 sur les addons MicroK8s. Vous maîtrisez maintenant :

1. ✅ La **philosophie** des addons (section 5.1)
2. ✅ La **commande enable** (section 5.2)
3. ✅ Les **4 addons essentiels** : Dashboard, DNS, Registry, Storage (sections 5.3-5.6)
4. ✅ La **liste complète** des addons disponibles (section 5.7)
5. ✅ La **gestion du cycle de vie** des addons (section 5.8)

Votre cluster MicroK8s est maintenant configuré et vous savez comment l'enrichir et le gérer au quotidien.

## Prochaines étapes

Les chapitres suivants vont approfondir des thématiques spécifiques :
- **Chapitre 6** : Stockage persistant en détail (PV, PVC, StatefulSets)
- **Chapitre 7** : Réseau Kubernetes interne
- **Chapitre 8** : Tâches planifiées (Jobs et CronJobs)
- **Chapitre 9** : Load balancing avec MetalLB
- **Chapitre 10** : Ingress et routage HTTP/HTTPS
- **Chapitre 11** : Certificats SSL/TLS avec Cert-Manager

Prenez le temps d'expérimenter avec ce que vous avez appris. Créez des applications de test, activez et désactivez des addons, cassez et réparez. C'est en pratiquant que vous deviendrez vraiment à l'aise !

---

**Astuce finale :** Créez un "journal de bord" de votre cluster MicroK8s. Notez les addons que vous activez, pourquoi, et les problèmes rencontrés avec leurs solutions. Dans quelques mois, vous aurez une ressource précieuse personnalisée pour référence rapide. Votre futur vous remerciera !

⏭️ [Stockage Persistant](/06-stockage-persistant/README.md)
