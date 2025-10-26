🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 21.10 Tests de Failover

## Introduction

Vous avez construit un cluster multi-node hautement disponible avec toutes les protections possibles : redondance, backups, stockage distribué, load balancing. Mais **comment savoir si tout cela fonctionne vraiment ?**

C'est là qu'interviennent les **tests de failover**. Dans ce chapitre final sur le multi-node, nous allons apprendre à :
- Valider que votre infrastructure est réellement résiliente
- Planifier et exécuter des tests de failover systématiques
- Mesurer les temps de récupération réels
- Identifier les faiblesses avant qu'elles ne causent des problèmes en production
- Documenter et améliorer continuellement votre résilience

**Principe fondamental :** Vous ne savez pas si votre système est résilient tant que vous ne l'avez pas cassé volontairement et vu qu'il se répare tout seul.

**Citation de Netflix :** "The best way to avoid failure is to fail constantly." - Chaos Engineering

## Qu'est-ce qu'un Test de Failover ?

### Définition

Un **test de failover** (test de basculement) consiste à simuler une panne d'un composant et à vérifier que :
1. Le système détecte la panne
2. Le système bascule automatiquement vers un composant de secours
3. Le service reste disponible (ou revient rapidement)
4. Les données restent intègres

**Analogie de l'ascenseur :**
Imaginez un immeuble avec 2 ascenseurs. Un test de failover consisterait à :
- Arrêter l'ascenseur 1 volontairement
- Vérifier que l'ascenseur 2 prend automatiquement le relais
- Mesurer combien de temps les gens doivent attendre
- S'assurer qu'aucun passager n'est bloqué

**Dans Kubernetes, c'est pareil :** On casse volontairement un composant et on vérifie que tout continue de fonctionner.

### Pourquoi Tester ?

**Raisons critiques :**

**1. Valider les hypothèses**
Vous *pensez* que votre cluster est résilient. Les tests le *prouvent*.

**2. Découvrir les faiblesses cachées**
Souvent, des problèmes n'apparaissent que lors de pannes réelles. Mieux vaut les découvrir lors d'un test contrôlé.

**3. Mesurer les temps de récupération**
Votre SLA promet un RTO de 5 minutes. Les tests vérifient que c'est réaliste.

**4. Entraîner l'équipe**
Les tests préparent votre équipe à gérer de vraies crises.

**5. Amélioration continue**
Chaque test révèle des opportunités d'amélioration.

**Statistique inquiétante :** 60% des entreprises qui déclarent avoir une infrastructure HA découvrent lors d'une vraie panne que... ça ne fonctionne pas comme prévu.

### Types de Tests

**1. Tests de composants individuels**
- Arrêt d'un nœud
- Crash d'un pod
- Saturation d'un disque

**2. Tests de cascade**
- Plusieurs composants tombent successivement
- Test de la limite de résilience

**3. Tests de charge sous panne**
- Panne pendant une période de forte charge
- Vérifie que le failover fonctionne même sous stress

**4. Tests de restauration**
- Après une panne, vérifier que tout revient à la normale
- Y compris les données

## Planification des Tests de Failover

### Quand Tester ?

**Fréquence recommandée :**

| Type de test | Fréquence | Durée typique |
|--------------|-----------|---------------|
| Test simple (1 nœud) | Mensuel | 30 min |
| Test complet | Trimestriel | 2-4 heures |
| Test de restauration | Semestriel | 4-8 heures |
| Chaos engineering | Trimestriel | Variable |

**Moments opportuns :**
- Après l'ajout d'un nouveau nœud
- Après une mise à jour majeure
- Avant un déploiement critique en production
- Lors des journées "maintenance planifiée"

**Éviter :**
- Pendant les pics de charge
- Juste avant un weekend (si quelque chose casse...)
- Sans avoir prévenu l'équipe

### Préparation

**Checklist pré-test :**

```markdown
# Checklist Test de Failover

## Avant le test
- [ ] Documentation du test préparée
- [ ] Backup récent vérifié (< 24h)
- [ ] Équipe informée (date/heure du test)
- [ ] Utilisateurs prévenus (si impact possible)
- [ ] Monitoring actif et visible
- [ ] Procédure de rollback prête
- [ ] Contact support disponible (si besoin)
- [ ] Fenêtre de temps suffisante

## Environnement
- [ ] Cluster de test isolé (recommandé) OU
- [ ] Production en période de faible charge
- [ ] Snapshots de l'état initial
- [ ] Logs activés et stockés

## Outils
- [ ] Accès SSH à tous les nœuds
- [ ] kubectl configuré
- [ ] Scripts de test prêts
- [ ] Grafana/Prometheus accessible
- [ ] Chronomètre (pour mesurer les temps)

## Plan B
- [ ] Procédure d'annulation du test définie
- [ ] Backup immédiatement accessible
- [ ] Contact escalade disponible
```

### Documentation du Test

**Template de plan de test :**

```markdown
# Plan de Test de Failover - [DATE]

## Objectif
Valider que le cluster tolère la panne de 1 nœud worker sans interruption de service.

## Périmètre
- **Cluster** : Production (ou Test)
- **Composant testé** : Worker2 (192.168.1.21)
- **Applications impactées** : Toutes (redistribution automatique)
- **Durée estimée** : 30 minutes

## Hypothèse à Valider
Le cluster redistribue automatiquement les pods de worker2 vers les autres nœuds en moins de 2 minutes, sans perte de données.

## Procédure
1. Prendre un snapshot de l'état initial
2. Arrêter worker2 : `ssh worker2 'sudo shutdown -h now'`
3. Observer la redistribution des pods
4. Vérifier l'accessibilité des applications
5. Redémarrer worker2
6. Vérifier le retour à la normale

## Critères de Succès
- [ ] Aucune interruption de service > 30 secondes
- [ ] Tous les pods redémarrés sur d'autres nœuds
- [ ] Données intègres (vérification DB)
- [ ] Worker2 rejoint le cluster en < 5 minutes
- [ ] Aucune alerte critique non résolue

## Critères d'Échec / Rollback
- Interruption de service > 5 minutes → Annuler et investiguer
- Perte de données → Restaurer depuis backup
- Cluster instable → Annuler le test

## Équipe
- **Chef de test** : Jean Dupont
- **Observateur** : Marie Martin
- **Support** : Support Team (on-call)

## Communication
- Slack #ops : Notification début/fin de test
- Email : Rapport complet sous 24h
```

## Tests de Failover Détaillés

### Test 1 : Arrêt d'un Nœud Worker

**Objectif :** Valider que la perte d'un worker n'impacte pas le service.

**Prérequis :**
- Au moins 2 workers dans le cluster
- Applications avec replicas ≥ 2

**Procédure :**

**1. Capturer l'état initial**
```bash
# Lister tous les pods et leur nœud
microk8s kubectl get pods --all-namespaces -o wide > /tmp/pods-before.txt

# Capturer les métriques
microk8s kubectl top nodes > /tmp/metrics-before.txt

# Vérifier les applications critiques
curl -I http://myapp.example.com
```

**2. Arrêter le nœud worker ciblé**
```bash
# Sur worker2
sudo shutdown -h now

# Ou depuis le control plane
ssh worker2 'sudo shutdown -h now'
```

**Démarrer le chronomètre ! ⏱️**

**3. Observer le comportement**
```bash
# Surveiller le statut du nœud
watch -n 1 'microk8s kubectl get nodes'

# Observer la redistribution des pods
watch -n 1 'microk8s kubectl get pods --all-namespaces -o wide'
```

**Ce que vous devriez voir :**
```
# Après 30-60 secondes
NAME      STATUS     ROLES    AGE
node1     Ready      <none>   10d
node2     Ready      <none>   10d
worker1   Ready      <none>   5d
worker2   NotReady   <none>   5d    ← Marqué NotReady

# Après 1-2 minutes
# Les pods de worker2 passent en "Terminating"
# Puis sont recréés sur node1, node2 ou worker1
```

**4. Vérifier l'accessibilité des applications**
```bash
# Boucle de test (pendant 5 minutes)
for i in {1..100}; do
  curl -s -o /dev/null -w "%{http_code}\n" http://myapp.example.com
  sleep 3
done
```

**Attendu :** Tous les codes HTTP 200 (ou quelques 503 pendant max 30 secondes)

**5. Vérifier l'intégrité des données**
```bash
# Se connecter à la base de données
POD=$(microk8s kubectl get pod -l app=postgres -o jsonpath="{.items[0].metadata.name}")
microk8s kubectl exec -it $POD -- psql -U postgres -c "SELECT COUNT(*) FROM users;"

# Comparer avec le nombre avant le test
```

**6. Redémarrer le nœud**
```bash
# Sur worker2 (après 5-10 minutes)
# Le nœud redémarre

# Vérifier qu'il rejoint le cluster
microk8s kubectl get nodes
```

**7. Capturer l'état final**
```bash
microk8s kubectl get pods --all-namespaces -o wide > /tmp/pods-after.txt
microk8s kubectl top nodes > /tmp/metrics-after.txt
```

**8. Mesurer les temps**
```
T0 : Arrêt de worker2
T1 : Worker2 marqué NotReady (attendre 30-60s)
T2 : Premier pod en Terminating (attendre 30s)
T3 : Premier pod recréé Running (attendre 30-60s)
T4 : Tous les pods recréés (attendre 1-2 min)

Total : T4 - T0 = 2-4 minutes typiquement
```

**Résultats attendus :**
- ✅ Tous les pods redistribués
- ✅ Applications accessibles pendant toute la durée
- ✅ Aucune perte de données
- ✅ Worker2 rejoint le cluster automatiquement

**Documentation :**
```markdown
# Résultat Test Worker Failover - 2025-01-15

## Résumé
✅ SUCCÈS - Failover fonctionnel

## Métriques
- Temps avant NotReady : 45 secondes
- Temps de redistribution : 2 minutes 10 secondes
- Interruption de service : 0 seconde
- Pods perdus : 0
- Données perdues : 0

## Observations
- Pod postgres-0 a pris 3 min pour redémarrer (normal, base volumineuse)
- Aucune alerte critique
- Load balancing a bien fonctionné

## Actions
- RAS - Système conforme aux attentes
```

### Test 2 : Arrêt du Nœud Control Plane Leader

**Objectif :** Valider que la perte du leader déclenche une nouvelle élection et que le cluster reste fonctionnel.

**Difficulté :** Avancé (impacte le control plane)

**Prérequis :**
- Cluster HA avec 3+ nœuds control plane
- Backup récent

**Procédure :**

**1. Identifier le nœud leader**
```bash
# Voir les leases pour trouver le leader
microk8s kubectl get lease -n kube-system -o wide

# Ou vérifier les logs Dqlite
sudo journalctl -u snap.microk8s.daemon-k8s-dqlite | grep -i "leader"
```

**2. Arrêter le leader**
```bash
# Sur le nœud leader identifié (ex: node1)
sudo shutdown -h now
```

**Démarrer le chronomètre ! ⏱️**

**3. Observer l'élection**
```bash
# Depuis un autre nœud (node2 ou node3)
microk8s kubectl get nodes

# Essayer des commandes kubectl
microk8s kubectl get pods --all-namespaces
```

**Ce que vous devriez observer :**
- Pendant 5-10 secondes : kubectl peut être lent ou timeout
- Après 10-15 secondes : une nouvelle élection est effectuée
- Après 20-30 secondes : kubectl fonctionne normalement depuis les autres nœuds

**4. Vérifier le quorum**
```bash
# Sur node2 ou node3
microk8s status
```

```
microk8s is running
high-availability: yes
  datastore master nodes: 192.168.1.11:19001 192.168.1.12:19001
  datastore standby nodes: none
```

Node1 (le leader tombé) n'apparaît plus, mais quorum maintenu (2/3). ✅

**5. Vérifier les applications**
```bash
# Les pods continuent-ils de tourner ?
microk8s kubectl get pods --all-namespaces

# Les applications sont-elles accessibles ?
curl http://myapp.example.com
```

**6. Redémarrer l'ancien leader**
```bash
# Après 5-10 minutes, rallumer node1
# Il rejoint automatiquement comme follower
```

**7. Vérifier le retour à la normale**
```bash
microk8s status
```

```
high-availability: yes
  datastore master nodes: 192.168.1.10:19001 192.168.1.11:19001 192.168.1.12:19001
```

Les 3 nœuds sont de retour. ✅

**Résultats attendus :**
- ✅ Nouvelle élection en < 15 secondes
- ✅ kubectl fonctionne depuis les autres nœuds
- ✅ Applications continuent de tourner
- ✅ Quorum maintenu (2/3)
- ✅ Ancien leader rejoint comme follower

**Temps d'interruption kubectl :** 5-15 secondes (acceptable)
**Temps d'interruption applications :** 0 seconde ✅

### Test 3 : Perte de Quorum (2 Nœuds Control Plane)

**Objectif :** Comprendre ce qui se passe quand le quorum est perdu.

**⚠️ ATTENTION :** Test destructif ! À faire UNIQUEMENT en environnement de test.

**Prérequis :**
- Cluster de test isolé (PAS en production)
- Backup très récent

**Procédure :**

**1. État initial : 3 nœuds control plane**
```bash
microk8s kubectl get nodes
```

```
NAME    STATUS   ROLES
node1   Ready    <none>
node2   Ready    <none>
node3   Ready    <none>
```

**2. Arrêter 2 nœuds simultanément**
```bash
# Sur node1
sudo shutdown -h now &

# Sur node2 (immédiatement après)
sudo shutdown -h now &
```

**3. Observer depuis node3**
```bash
# Node3 détecte la perte de quorum
microk8s status
```

```
microk8s is running
high-availability: no (QUORUM LOST)
  datastore master nodes: 192.168.1.12:19001
```

**4. Tenter des commandes kubectl**
```bash
# Lecture : fonctionne
microk8s kubectl get pods

# Écriture : ÉCHOUE
microk8s kubectl run test --image=nginx
```

```
Error: cluster is read-only (quorum lost)
```

**Le cluster est en mode READ-ONLY.** Les pods existants continuent de tourner, mais impossible de créer/modifier/supprimer des ressources.

**5. Restaurer le quorum**
```bash
# Rallumer node1 OU node2 (1 suffit)
# Après ~2 minutes, le quorum est restauré

microk8s status
```

```
high-availability: yes
  datastore master nodes: 192.168.1.11:19001 192.168.1.12:19001
```

Quorum restauré (2/3). ✅ Cluster redevient opérationnel.

**Leçons apprises :**
- ❌ Perte de quorum = cluster en lecture seule
- ✅ Les applications continuent de tourner
- ✅ Restauration possible en rallumant 1 nœud
- 💡 **Importance de toujours avoir 3+ nœuds control plane**

### Test 4 : Saturation d'un Disque

**Objectif :** Valider que Kubernetes gère correctement un nœud avec disque plein.

**Procédure :**

**1. Créer un gros fichier pour saturer le disque**
```bash
# Sur worker1
df -h /  # Noter l'espace libre

# Créer un fichier pour remplir à 90%
sudo fallocate -l 10G /tmp/bigfile

df -h /  # Vérifier
```

**2. Observer le comportement de Kubernetes**
```bash
# Après quelques minutes, Kubernetes détecte la pression disque
microk8s kubectl describe node worker1
```

```
Conditions:
  Type             Status
  ----             ------
  DiskPressure     True   ← Kubernetes a détecté le problème
```

**3. Vérifier que les nouveaux pods évitent ce nœud**
```bash
# Créer un nouveau deployment
microk8s kubectl create deployment test --image=nginx --replicas=5

# Vérifier où les pods sont placés
microk8s kubectl get pods -o wide
```

Les 5 pods devraient être sur node1, node2, node3, worker2 - **AUCUN sur worker1**. ✅

**4. Nettoyer**
```bash
sudo rm /tmp/bigfile
```

Après quelques minutes, worker1 repasse en `DiskPressure: False`.

**Résultats attendus :**
- ✅ Kubernetes détecte la saturation en < 5 minutes
- ✅ Nouveaux pods ne sont pas schedulés sur le nœud plein
- ✅ Pods existants continuent de tourner
- ✅ Alertes déclenchées (Prometheus)

### Test 5 : Crash d'un Pod

**Objectif :** Valider que Kubernetes redémarre automatiquement les pods crashés.

**Procédure :**

**1. Identifier un pod à crasher**
```bash
microk8s kubectl get pods -l app=web
```

```
NAME                   READY   STATUS
web-7d8f9c5b4-abc123   1/1     Running
```

**2. Tuer le processus principal dans le pod**
```bash
microk8s kubectl exec web-7d8f9c5b4-abc123 -- killall -9 nginx
```

**Démarrer le chronomètre ! ⏱️**

**3. Observer le redémarrage**
```bash
watch -n 1 'microk8s kubectl get pods -l app=web'
```

```
# Immédiatement après le kill
NAME                   READY   STATUS    RESTARTS
web-7d8f9c5b4-abc123   0/1     Error     0

# Après quelques secondes
web-7d8f9c5b4-abc123   0/1     CrashLoopBackOff  1

# Puis
web-7d8f9c5b4-abc123   1/1     Running   1
```

**4. Mesurer le temps de redémarrage**
Typiquement : 10-30 secondes

**5. Vérifier que les autres replicas ont pris le relais**
```bash
# Si vous avez 3 replicas, les 2 autres ont continué à servir le trafic
curl http://myapp.example.com  # Devrait fonctionner pendant toute la durée
```

**Résultats attendus :**
- ✅ Pod redémarré automatiquement
- ✅ RESTARTS incrémenté (visible dans kubectl get pods)
- ✅ Service non interrompu (grâce aux autres replicas)

### Test 6 : Panne Réseau (Partition)

**Objectif :** Simuler une panne réseau et observer le comportement.

**⚠️ Test avancé** - Risque de casser la communication cluster

**Procédure :**

**1. Bloquer le réseau sur un nœud**
```bash
# Sur worker2, bloquer tout le trafic (sauf SSH pour garder l'accès)
sudo iptables -A INPUT -p tcp --dport 22 -j ACCEPT
sudo iptables -A OUTPUT -p tcp --sport 22 -j ACCEPT
sudo iptables -A INPUT -j DROP
sudo iptables -A OUTPUT -j DROP
```

**2. Observer depuis les autres nœuds**
```bash
microk8s kubectl get nodes
```

Après ~1 minute, worker2 passe en `NotReady`.

**3. Vérifier que les pods sont redistribués**
Même comportement que le test 1 (arrêt de nœud).

**4. Restaurer le réseau**
```bash
# Sur worker2
sudo iptables -F  # Flush toutes les règles
```

Worker2 redevient `Ready` et rejoint le cluster.

**Résultats attendus :**
- ✅ Nœud isolé marqué NotReady
- ✅ Pods redistribués
- ✅ Restauration automatique une fois le réseau rétabli

### Test 7 : Restauration Complète depuis Backup

**Objectif :** Valider que vous pouvez reconstruire le cluster depuis zéro.

**Durée :** 4-8 heures (test complet)

**Procédure :**

**1. Créer un "snapshot" de l'état actuel**
```bash
# Backup complet (vu au chapitre 21.8)
sudo /usr/local/bin/backup-cluster-manual.sh
```

**2. Détruire le cluster**
```bash
# Sur chaque nœud
microk8s stop
sudo snap remove microk8s --purge
```

**Cluster complètement détruit ! 💥**

**3. Reconstruire depuis zéro**
```bash
# Sur chaque nœud
sudo snap install microk8s --classic --channel=1.28/stable

# Former le cluster HA
# (voir chapitre 21.4)
```

**4. Restaurer depuis le backup**
```bash
# Copier le backup
scp backup-20250115.tar.gz node1:/tmp/

# Extraire et restaurer
# (voir chapitre 21.8 - section Restauration)
```

**5. Valider la restauration**
```bash
# Toutes les applications sont-elles revenues ?
microk8s kubectl get all --all-namespaces

# Les données sont-elles intègres ?
# Vérifier la base de données

# Les Ingress fonctionnent-ils ?
curl http://myapp.example.com
```

**6. Mesurer le temps total de restauration**
```
T0 : Début de la reconstruction
T1 : Cluster HA formé (45 min)
T2 : Addons restaurés (15 min)
T3 : Applications restaurées (30 min)
T4 : Données restaurées (1-2h)
T5 : Validation complète (30 min)

Total : 3-4 heures
```

**Résultats attendus :**
- ✅ Cluster reconstruit fonctionnel
- ✅ Toutes les applications restaurées
- ✅ Données intègres
- ✅ RTO respecté (si < 4h était l'objectif)

**Document les écarts :**
- Quoi n'a pas été restauré ?
- Quelles difficultés rencontrées ?
- Comment améliorer la procédure ?

## Métriques de Failover

### Temps de Récupération (RTO)

**Recovery Time Objective = Temps maximum acceptable pour restaurer le service**

**Mesures à prendre :**
```
RTO_nœud = Temps pour que le cluster redistribue les pods après panne d'un nœud
RTO_pod = Temps pour redémarrer un pod crashé
RTO_leader = Temps pour élire un nouveau leader Dqlite
RTO_complet = Temps pour restaurer depuis backup complet
```

**Benchmarks typiques pour un cluster MicroK8s 3 nœuds :**
| Événement | RTO typique | RTO cible |
|-----------|-------------|-----------|
| Crash d'un pod | 10-30s | < 1min |
| Panne d'un worker | 1-2 min | < 3min |
| Panne leader Dqlite | 10-20s | < 30s |
| Saturation disque (détection) | 3-5 min | < 5min |
| Restauration depuis backup | 3-4h | < 4h |

### Perte de Données (RPO)

**Recovery Point Objective = Perte de données maximale acceptable**

**Mesures à prendre :**
```
RPO = Fréquence des backups

Exemples :
- Backup quotidien → RPO = 24h
- Backup horaire → RPO = 1h
- Réplication synchrone → RPO = 0 (aucune perte)
```

**Pour Longhorn avec 3 replicas :** RPO = 0 (réplication synchrone)
**Pour backups Velero :** RPO = fréquence du backup (ex: 1h)

### Taux de Disponibilité (Uptime)

**Formule :**
```
Uptime (%) = ((Temps total - Downtime) / Temps total) × 100
```

**Calcul après tests :**
```
Période de test : 1 mois (720 heures)
Downtime cumulé :
- Test failover worker : 0s (service maintenu)
- Test failover leader : 15s
- Panne imprévue : 5 minutes = 300s
- Maintenance : 10 minutes = 600s

Total downtime : 915 secondes = 15.25 minutes

Uptime = ((720×60 - 15.25) / (720×60)) × 100 = 99.96%
```

**SLA standards :**
- 99% = 7h20min/mois
- 99.9% = 43min/mois
- **99.95% = 21min/mois** ← Votre résultat
- 99.99% = 4min/mois

### Tableau de Bord des Tests

**Maintenir un suivi :**

```markdown
# Historique Tests de Failover - Q1 2025

| Date | Type | Durée | Downtime | Succès | Notes |
|------|------|-------|----------|--------|-------|
| 2025-01-15 | Worker failover | 30min | 0s | ✅ | RAS |
| 2025-02-10 | Leader failover | 45min | 12s | ✅ | Légère latence kubectl |
| 2025-03-05 | Disk saturation | 1h | 0s | ✅ | Alerte bien déclenchée |
| 2025-03-20 | Restauration complète | 4h | N/A | ✅ | Procédure améliorée |

## Statistiques Q1
- Tests réalisés : 4
- Succès : 4 (100%)
- Downtime cumulé : 12s
- RTO moyen : 2min 30s
- Uptime Q1 : 99.98%
```

## Automatisation des Tests

### Scripts de Test

**Script de test automatique :**

```bash
#!/bin/bash
# test-failover-auto.sh

# Configuration
TARGET_NODE="worker2"
TEST_NAME="Worker Failover Test"
TIMESTAMP=$(date +%Y%m%d-%H%M%S)
REPORT_FILE="/tmp/failover-test-$TIMESTAMP.txt"

echo "=== $TEST_NAME - $TIMESTAMP ===" | tee $REPORT_FILE

# 1. État initial
echo "État initial :" | tee -a $REPORT_FILE
microk8s kubectl get nodes | tee -a $REPORT_FILE
microk8s kubectl get pods --all-namespaces -o wide > /tmp/pods-before-$TIMESTAMP.txt

# 2. Vérifier l'accessibilité avant
echo "Test accessibilité avant panne..." | tee -a $REPORT_FILE
HTTP_BEFORE=$(curl -s -o /dev/null -w "%{http_code}" http://myapp.example.com)
echo "HTTP Status avant : $HTTP_BEFORE" | tee -a $REPORT_FILE

# 3. Arrêter le nœud
echo "Arrêt de $TARGET_NODE..." | tee -a $REPORT_FILE
START_TIME=$(date +%s)
ssh $TARGET_NODE 'sudo shutdown -h now' &

# 4. Surveiller la redistribution
echo "Surveillance de la redistribution..." | tee -a $REPORT_FILE
sleep 30  # Attendre que le nœud soit NotReady

# Attendre que tous les pods soient Running
TIMEOUT=300  # 5 minutes max
ELAPSED=0
while [ $ELAPSED -lt $TIMEOUT ]; do
  PENDING=$(microk8s kubectl get pods --all-namespaces --field-selector=status.phase!=Running,status.phase!=Succeeded --no-headers | wc -l)

  if [ $PENDING -eq 0 ]; then
    echo "Tous les pods sont Running" | tee -a $REPORT_FILE
    break
  fi

  echo "Pods en attente : $PENDING" | tee -a $REPORT_FILE
  sleep 10
  ELAPSED=$((ELAPSED + 10))
done

END_TIME=$(date +%s)
RECOVERY_TIME=$((END_TIME - START_TIME))

echo "Temps de récupération : ${RECOVERY_TIME}s" | tee -a $REPORT_FILE

# 5. Vérifier l'accessibilité après
echo "Test accessibilité après redistribution..." | tee -a $REPORT_FILE
HTTP_AFTER=$(curl -s -o /dev/null -w "%{http_code}" http://myapp.example.com)
echo "HTTP Status après : $HTTP_AFTER" | tee -a $REPORT_FILE

# 6. Comparer avant/après
microk8s kubectl get pods --all-namespaces -o wide > /tmp/pods-after-$TIMESTAMP.txt

# 7. Générer le rapport
echo "" | tee -a $REPORT_FILE
echo "=== RÉSULTAT ===" | tee -a $REPORT_FILE

if [ "$HTTP_AFTER" == "200" ] && [ $RECOVERY_TIME -lt 180 ]; then
  echo "✅ TEST RÉUSSI" | tee -a $REPORT_FILE
  echo "- Service accessible" | tee -a $REPORT_FILE
  echo "- Récupération en ${RECOVERY_TIME}s (< 3min)" | tee -a $REPORT_FILE
else
  echo "❌ TEST ÉCHOUÉ" | tee -a $REPORT_FILE
  [ "$HTTP_AFTER" != "200" ] && echo "- Service inaccessible" | tee -a $REPORT_FILE
  [ $RECOVERY_TIME -ge 180 ] && echo "- Récupération trop lente (${RECOVERY_TIME}s)" | tee -a $REPORT_FILE
fi

echo "" | tee -a $REPORT_FILE
echo "Rapport complet : $REPORT_FILE"

# 8. Envoyer notification
# curl -X POST https://hooks.slack.com/... -d "Test failover : voir $REPORT_FILE"
```

**Rendre exécutable et tester :**
```bash
chmod +x test-failover-auto.sh
./test-failover-auto.sh
```

### Intégration CI/CD

**GitLab CI / GitHub Actions pour tests mensuels :**

```yaml
# .gitlab-ci.yml
failover-test:
  stage: test
  only:
    - schedules  # Exécuter uniquement sur schedule mensuel
  script:
    - ./scripts/test-failover-auto.sh
  artifacts:
    paths:
      - /tmp/failover-test-*.txt
    expire_in: 3 months
  allow_failure: false  # Fail le pipeline si test échoue
```

**Schedule mensuel dans GitLab :**
Settings → CI/CD → Schedules → New Schedule (1er de chaque mois)

### Chaos Engineering avec Chaos Mesh

**Installation de Chaos Mesh :**

```bash
# Ajouter le repo Helm
microk8s helm3 repo add chaos-mesh https://charts.chaos-mesh.org

# Installer
microk8s helm3 install chaos-mesh chaos-mesh/chaos-mesh \
  --namespace chaos-mesh \
  --create-namespace \
  --set chaosDaemon.runtime=containerd \
  --set chaosDaemon.socketPath=/var/snap/microk8s/common/run/containerd.sock
```

**Exemple de Chaos Experiment - Tuer un pod aléatoire :**

```yaml
# pod-kill-experiment.yaml
apiVersion: chaos-mesh.org/v1alpha1
kind: PodChaos
metadata:
  name: pod-kill-test
  namespace: default
spec:
  action: pod-kill
  mode: one  # Tuer 1 pod aléatoire
  selector:
    namespaces:
      - production
    labelSelectors:
      app: web
  scheduler:
    cron: '@every 1h'  # Toutes les heures
```

```bash
microk8s kubectl apply -f pod-kill-experiment.yaml
```

**Chaos Mesh tue automatiquement un pod web toutes les heures** → Force votre système à être résilient !

## Amélioration Continue

### Analyse Post-Test

**Questions à se poser après chaque test :**

1. **Le test s'est-il passé comme prévu ?**
   - Oui → Système conforme
   - Non → Investiguer les écarts

2. **Les temps mesurés sont-ils acceptables ?**
   - RTO < objectif → ✅
   - RTO > objectif → Identifier les goulots

3. **Y a-t-il eu des surprises ?**
   - Comportements inattendus ?
   - Erreurs non anticipées ?

4. **Les alertes ont-elles fonctionné ?**
   - Alertes déclenchées ?
   - Notifications reçues ?
   - Faux positifs ?

5. **La documentation était-elle à jour ?**
   - Procédures correctes ?
   - Runbooks utiles ?

6. **L'équipe était-elle prête ?**
   - Communication efficace ?
   - Rôles clairs ?

### Boucle d'Amélioration

```
┌─────────────────────────────────────────────┐
│       CYCLE D'AMÉLIORATION CONTINUE         │
└─────────────────────────────────────────────┘

1. PLANIFIER
   └─► Définir objectifs du test
       Préparer procédures

2. EXÉCUTER
   └─► Lancer le test
       Collecter métriques

3. ANALYSER
   └─► Comparer résultats vs objectifs
       Identifier écarts

4. AMÉLIORER
   └─► Corriger faiblesses
       Mettre à jour procédures
       ┌────────────┐
       │ Retour à 1 │
       └────────────┘
```

### Actions d'Amélioration Typiques

**Suite à des tests :**

**Problème détecté :** RTO de 5 minutes (objectif 3 min)
**Action :** Augmenter le nombre de replicas de 2 à 3
**Résultat :** RTO réduit à 2 minutes ✅

**Problème détecté :** Alerte non déclenchée lors saturation disque
**Action :** Configurer alerte Prometheus à 80% (au lieu de 90%)
**Résultat :** Alerte précoce ✅

**Problème détecté :** Procédure de restauration incomplète
**Action :** Mettre à jour documentation avec étapes manquantes
**Résultat :** Restauration plus fluide au prochain test ✅

### Partage de Connaissances

**Post-mortem blameless :**
Après chaque test (surtout les échecs), organiser une réunion pour :
- Débriefer ce qui s'est passé
- Identifier les améliorations
- Partager les apprentissages
- **SANS chercher de coupable** (culture blameless)

**Documentation vivante :**
- Wiki interne avec retours d'expérience
- Runbooks mis à jour après chaque test
- Vidéos de tests pour formation

## Checklist Finale de Résilience

**Avant de considérer votre cluster "production-ready" :**

```markdown
# Checklist Résilience Cluster Kubernetes

## Infrastructure
- [ ] 3+ nœuds control plane
- [ ] Nœuds répartis sur différentes zones (racks, alim, etc.)
- [ ] UPS installé
- [ ] Network bonding configuré

## Stockage
- [ ] Longhorn avec 3 replicas minimum
- [ ] Backups automatiques quotidiens
- [ ] Backups testés (restauration réussie)
- [ ] Snapshots configurés

## Applications
- [ ] Toutes apps critiques avec replicas ≥ 2
- [ ] PodDisruptionBudgets configurés
- [ ] Anti-affinity sur apps critiques
- [ ] Health checks (liveness + readiness) partout

## Monitoring
- [ ] Prometheus + Grafana actifs
- [ ] Alertes critiques configurées
- [ ] Slack/Email notifications fonctionnelles
- [ ] Dashboards de résilience créés

## Tests
- [ ] Test worker failover : ✅ SUCCÈS
- [ ] Test leader failover : ✅ SUCCÈS
- [ ] Test saturation disque : ✅ SUCCÈS
- [ ] Test restauration backup : ✅ SUCCÈS
- [ ] Tests mensuels programmés
- [ ] Documentation à jour

## Procédures
- [ ] Runbooks à jour
- [ ] Plan de reprise d'activité (DRP) écrit
- [ ] Contacts d'escalade définis
- [ ] Équipe formée

## Métriques
- [ ] RTO mesuré et documenté
- [ ] RPO défini et respecté
- [ ] Uptime > 99.9% sur 3 mois
- [ ] Historique des tests maintenu

## Résultat
- [ ] ✅ Cluster prêt pour la production
- [ ] ⚠️ Amélioration nécessaire (détails: ___)
- [ ] ❌ Non prêt (actions: ___)
```

## Points Clés à Retenir

1. **Tester = Valider** : Vous ne savez pas si ça marche tant que vous n'avez pas testé
2. **Fréquence** : Tests mensuels minimum (plus au début)
3. **Documentation** : Tout doit être documenté (avant, pendant, après)
4. **Métriques** : RTO, RPO, Uptime doivent être mesurés
5. **Automatisation** : Scripts pour répéter les tests facilement
6. **Amélioration continue** : Chaque test = opportunité d'améliorer
7. **Culture blameless** : Chercher à comprendre, pas à blâmer
8. **Tests variés** : Nœuds, pods, disques, réseau, restauration
9. **Environnement de test** : Idéalement isolé de la production
10. **Chaos engineering** : Passer au niveau supérieur avec des pannes aléatoires

## Conclusion du Chapitre 21

**Félicitations !** Vous avez complété le chapitre 21 sur le multi-node et la haute disponibilité. Vous avez appris à :

- Comprendre l'architecture multi-node (21.1)
- Maîtriser Dqlite et la HA (21.2)
- Ajouter des nœuds (21.3)
- Configurer un cluster HA complet (21.4)
- Ajouter des workers (21.5)
- Implémenter le load balancing (21.6)
- Gérer le stockage distribué (21.7)
- Sauvegarder le control plane (21.8)
- Concevoir des stratégies de redondance (21.9)
- **Tester et valider la résilience (21.10)** ✅

Votre cluster est maintenant :
- **Hautement disponible** (tolère les pannes)
- **Résilient** (récupération automatique)
- **Évolutif** (ajout de nœuds facile)
- **Sauvegardé** (protection contre les catastrophes)
- **Testé** (vous SAVEZ qu'il fonctionne)

**Vous êtes maintenant capable de gérer un cluster Kubernetes de qualité production !** 🎉🚀

---

**Bravo !** Vous maîtrisez maintenant les tests de failover et la validation de résilience. Vous savez non seulement construire un cluster résilient, mais aussi prouver qu'il l'est vraiment !

⏭️ [Sauvegarde et Restauration](/22-sauvegarde-et-restauration/README.md)
