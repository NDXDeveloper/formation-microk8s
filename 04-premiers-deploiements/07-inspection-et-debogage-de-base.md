🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 4.7 Inspection et Débogage de Base

## Introduction

Dans le monde réel, les applications ne fonctionnent pas toujours du premier coup. Un Pod ne démarre pas, un Service n'est pas accessible, une image ne se télécharge pas... Le débogage fait partie intégrante du travail avec Kubernetes.

Cette section vous apprendra une **méthodologie systématique** pour diagnostiquer et résoudre les problèmes les plus courants. Vous découvrirez les outils et techniques essentiels pour devenir autonome dans le débogage de vos applications Kubernetes.

**Objectifs :**
- Comprendre comment Kubernetes signale les problèmes
- Maîtriser les outils d'inspection essentiels
- Suivre une méthodologie de débogage efficace
- Résoudre les problèmes les plus courants

## Philosophie du Débogage

### La règle d'or

> **"Ne paniquez pas, lisez les messages d'erreur"**

Kubernetes est très bavard et fournit beaucoup d'informations. La plupart des problèmes peuvent être résolus en lisant attentivement les messages d'erreur et les événements.

### Les trois piliers du débogage Kubernetes

1. **État des ressources** (`get`, `describe`) : Où en est l'objet ?
2. **Logs** (`logs`) : Que dit l'application ?
3. **Événements** (`events`) : Qu'a fait Kubernetes ?

## Méthodologie de Débogage en 5 Étapes

Suivez cette méthodologie systématique pour résoudre efficacement les problèmes :

### Étape 1 : Identifier le problème

**Que se passe-t-il exactement ?**
- Un Pod ne démarre pas ?
- Une application est inaccessible ?
- Des erreurs dans les logs ?
- Performance dégradée ?

**Rassemblez les informations de base :**
```bash
# Quel est l'état global ?
microk8s kubectl get all

# Quels Pods ont des problèmes ?
microk8s kubectl get pods
```

### Étape 2 : Vérifier l'état des ressources

**Inspectez les ressources concernées :**
```bash
# État détaillé du Pod
microk8s kubectl describe pod <pod-name>

# État du Deployment
microk8s kubectl describe deployment <deployment-name>

# État du Service
microk8s kubectl describe service <service-name>
```

**Points d'attention :**
- Section **Status** : État actuel
- Section **Events** : Historique des actions
- Section **Conditions** : Checks de santé

### Étape 3 : Analyser les événements

**Les événements sont votre meilleure source d'information :**
```bash
# Tous les événements récents
microk8s kubectl get events --sort-by='.lastTimestamp'

# Événements d'un Pod spécifique
microk8s kubectl get events --field-selector involvedObject.name=<pod-name>

# Événements des 10 dernières minutes
microk8s kubectl get events --field-selector type=Warning
```

### Étape 4 : Examiner les logs

**Que dit l'application ?**
```bash
# Logs actuels
microk8s kubectl logs <pod-name>

# Logs du conteneur précédent (si crash)
microk8s kubectl logs <pod-name> --previous

# Suivre en temps réel
microk8s kubectl logs -f <pod-name>
```

### Étape 5 : Tester et valider

**Vérifiez que le problème est résolu :**
```bash
# L'état est-il correct maintenant ?
microk8s kubectl get pods

# L'application répond-elle ?
microk8s kubectl exec -it <pod-name> -- curl localhost:80

# Les logs sont-ils propres ?
microk8s kubectl logs <pod-name> | tail -20
```

## États des Pods et leur Signification

Comprendre les états des Pods est crucial pour le débogage.

### Tableau des états

| État | Signification | Action |
|------|---------------|--------|
| **Pending** | En attente de scheduling | Vérifier les ressources, les volumes |
| **ContainerCreating** | Création du conteneur en cours | Normal, attendre quelques secondes |
| **Running** | Conteneur en cours d'exécution | État normal ✅ |
| **Succeeded** | Terminé avec succès (Jobs) | Normal pour les Jobs ✅ |
| **Failed** | Échec de l'exécution | Vérifier les logs et events |
| **CrashLoopBackOff** | Crash répété du conteneur | Vérifier les logs, config, resources |
| **ImagePullBackOff** | Impossible de télécharger l'image | Vérifier le nom, le registry, les credentials |
| **Error** | Erreur générale | Vérifier describe et logs |
| **Unknown** | État inconnu | Problème de communication, vérifier le Node |
| **Terminating** | En cours de suppression | Normal lors d'un delete |

### Vérifier l'état des Pods

```bash
# Vue simple
microk8s kubectl get pods

# Sortie exemple :
# NAME                     READY   STATUS             RESTARTS   AGE
# nginx-5d59d67564-7k9xm   1/1     Running            0          5m
# webapp-8f7d9c8d-x2p4l    0/1     CrashLoopBackOff   5          3m
# api-6c8b9d7f-n3k5m       0/1     ImagePullBackOff   0          2m
```

**Décodage :**
- `READY` : Conteneurs prêts / Total de conteneurs
  - `1/1` = OK ✅
  - `0/1` = Problème ❌
- `RESTARTS` : Nombre de redémarrages (si > 0, il y a eu des crashs)
- `AGE` : Temps depuis la création

## Problème 1 : Pod en "Pending"

### Symptômes

```bash
microk8s kubectl get pods
# NAME         READY   STATUS    RESTARTS   AGE
# my-app       0/1     Pending   0          2m
```

Le Pod est créé mais n'est pas assigné à un nœud.

### Diagnostic

```bash
# Détails complets
microk8s kubectl describe pod my-app
```

**Regardez la section Events en bas :**

### Causes courantes et solutions

#### Cause 1 : Ressources insuffisantes

**Message d'erreur :**
```
Events:
  Warning  FailedScheduling  0/1 nodes are available: Insufficient cpu.
```

**Signification :** Le Pod demande plus de CPU/RAM que le cluster peut fournir.

**Solution :**

```bash
# Voir l'utilisation des ressources
microk8s kubectl top nodes

# Réduire les ressources demandées
# Éditer le Deployment
microk8s kubectl edit deployment my-app

# Modifier les requests/limits
resources:
  requests:
    memory: "128Mi"  # Au lieu de 1Gi
    cpu: "100m"      # Au lieu de 1000m
```

#### Cause 2 : Volume non disponible

**Message d'erreur :**
```
Events:
  Warning  FailedMount  Unable to mount volumes: persistentvolumeclaim "data" not found
```

**Signification :** Le PVC référencé n'existe pas.

**Solution :**

```bash
# Lister les PVC
microk8s kubectl get pvc

# Créer le PVC manquant
microk8s kubectl apply -f pvc.yaml

# Ou corriger le nom dans le Pod
microk8s kubectl edit deployment my-app
```

#### Cause 3 : Node Selector incorrect

**Message d'erreur :**
```
Events:
  Warning  FailedScheduling  0/1 nodes are available: node(s) didn't match node selector.
```

**Signification :** Aucun nœud ne correspond au nodeSelector.

**Solution :**

```bash
# Voir les labels des nœuds
microk8s kubectl get nodes --show-labels

# Supprimer ou corriger le nodeSelector
microk8s kubectl edit deployment my-app
```

## Problème 2 : Pod en "CrashLoopBackOff"

### Symptômes

```bash
microk8s kubectl get pods
# NAME         READY   STATUS             RESTARTS   AGE
# my-app       0/1     CrashLoopBackOff   5          3m
```

Le conteneur démarre puis crash immédiatement, en boucle.

### Diagnostic

**Étape 1 : Voir les logs**

```bash
# Logs actuels
microk8s kubectl logs my-app

# Logs avant le dernier crash
microk8s kubectl logs my-app --previous
```

**Étape 2 : Vérifier les événements**

```bash
microk8s kubectl describe pod my-app
```

**Regardez :**
- Les **Events** : Pourquoi le conteneur s'arrête ?
- Le **Exit Code** : Code de sortie du processus

### Causes courantes et solutions

#### Cause 1 : Erreur dans l'application

**Logs montrent :**
```
Error: Cannot connect to database
Connection refused at postgres:5432
```

**Signification :** L'application ne peut pas se connecter à une dépendance.

**Solutions :**

```bash
# Vérifier que le service de base de données existe
microk8s kubectl get service postgres

# Vérifier les credentials
microk8s kubectl get secret db-credentials

# Tester la connectivité
microk8s kubectl run test --image=busybox --rm -it -- nslookup postgres
```

#### Cause 2 : Commande incorrecte

**Exit Code : 127 ou 126**
```
Events:
  Back-off restarting failed container
  Exit Code: 127
```

**Signification :** La commande spécifiée n'existe pas dans le conteneur.

**Solution :**

```bash
# Vérifier la commande dans le Deployment
microk8s kubectl get deployment my-app -o yaml | grep -A 5 command

# Tester la commande manuellement
microk8s kubectl run test --image=myapp:latest --rm -it -- /bin/sh
# Dans le conteneur, tester la commande
```

#### Cause 3 : Manque de ressources

**Logs montrent :**
```
OOMKilled
```

**Signification :** Le conteneur a été tué car il a dépassé sa limite mémoire.

**Solution :**

```bash
# Augmenter les limites mémoire
microk8s kubectl edit deployment my-app

# Modifier :
resources:
  limits:
    memory: "512Mi"  # Au lieu de 256Mi
```

#### Cause 4 : Probe mal configurée

**Events montrent :**
```
Liveness probe failed: Get http://10.1.1.5:8080/health: dial tcp 10.1.1.5:8080: connect: connection refused
```

**Signification :** La liveness probe échoue et tue le conteneur.

**Solution :**

```bash
# Vérifier la configuration de la probe
microk8s kubectl get deployment my-app -o yaml | grep -A 10 livenessProbe

# Options :
# 1. Corriger le path/port
# 2. Augmenter initialDelaySeconds
# 3. Supprimer temporairement la probe pour déboguer
```

## Problème 3 : Pod en "ImagePullBackOff"

### Symptômes

```bash
microk8s kubectl get pods
# NAME         READY   STATUS             RESTARTS   AGE
# my-app       0/1     ImagePullBackOff   0          2m
```

Kubernetes ne peut pas télécharger l'image Docker.

### Diagnostic

```bash
# Détails
microk8s kubectl describe pod my-app
```

**Events typiques :**
```
Events:
  Warning  Failed     Failed to pull image "myapp:latest": rpc error: code = Unknown desc = Error response from daemon: pull access denied for myapp, repository does not exist or may require 'docker login'
```

### Causes courantes et solutions

#### Cause 1 : Image n'existe pas

**Message :**
```
repository does not exist
```

**Vérifications :**

```bash
# Le nom de l'image est-il correct ?
microk8s kubectl get deployment my-app -o yaml | grep image:

# Erreurs courantes :
# - Faute de frappe : ngix au lieu de nginx
# - Tag inexistant : nginx:1.99 (n'existe pas)
# - Registry incorrect : docker.io/myapp au lieu de gcr.io/myapp
```

**Solution :**

```bash
# Corriger le nom de l'image
microk8s kubectl set image deployment/my-app my-app=nginx:1.21
```

#### Cause 2 : Registry privé sans credentials

**Message :**
```
pull access denied / unauthorized
```

**Solution :**

```bash
# Créer un Secret pour le registry
microk8s kubectl create secret docker-registry my-registry-secret \
  --docker-server=registry.example.com \
  --docker-username=myuser \
  --docker-password=mypassword \
  --docker-email=myemail@example.com

# Ajouter imagePullSecrets au Deployment
microk8s kubectl edit deployment my-app

# Ajouter :
spec:
  template:
    spec:
      imagePullSecrets:
      - name: my-registry-secret
```

#### Cause 3 : Problème réseau

**Message :**
```
dial tcp: lookup registry.example.com: no such host
```

**Vérifications :**

```bash
# DNS fonctionne-t-il ?
microk8s kubectl run test --image=busybox --rm -it -- nslookup registry.example.com

# Le registry est-il accessible ?
curl https://registry.example.com/v2/
```

## Problème 4 : Pod Running mais application inaccessible

### Symptômes

Le Pod est en état `Running` mais l'application ne répond pas.

```bash
microk8s kubectl get pods
# NAME         READY   STATUS    RESTARTS   AGE
# my-app       1/1     Running   0          5m

# Mais :
curl http://my-service
# Connection refused ou timeout
```

### Diagnostic par étapes

#### Étape 1 : Le conteneur écoute-t-il ?

```bash
# Accéder au Pod
microk8s kubectl exec -it my-app -- /bin/sh

# Tester depuis l'intérieur
curl localhost:8080
wget -qO- localhost:8080

# Vérifier les ports écoutés
netstat -tlnp
# ou
ss -tlnp
```

**Si ça ne fonctionne pas depuis l'intérieur du Pod :**
- L'application n'a pas démarré correctement
- L'application écoute sur le mauvais port
- Vérifier les logs : `microk8s kubectl logs my-app`

#### Étape 2 : Le Service pointe-t-il vers les bons Pods ?

```bash
# Vérifier le Service
microk8s kubectl describe service my-service

# Regarder la section Endpoints
Endpoints: 10.1.1.5:8080,10.1.1.6:8080

# Si Endpoints est vide (<none>), le sélecteur est incorrect
```

**Vérifier les labels :**

```bash
# Labels du Service selector
microk8s kubectl get service my-service -o yaml | grep -A 3 selector

# Labels des Pods
microk8s kubectl get pods --show-labels

# Les labels correspondent-ils ?
```

**Solution si les labels ne correspondent pas :**

```bash
# Option 1 : Corriger le Service
microk8s kubectl edit service my-service

# Option 2 : Corriger les labels des Pods
microk8s kubectl label pods -l app=oldname app=newname --overwrite
```

#### Étape 3 : Le port est-il correct ?

```bash
# Vérifier le Service
microk8s kubectl get service my-service -o yaml

# Comparer :
ports:
- port: 80           # Port du Service
  targetPort: 8080   # Port du conteneur

# avec le Pod :
microk8s kubectl get pod my-app -o yaml | grep containerPort

ports:
- containerPort: 8080  # Doit correspondre au targetPort
```

#### Étape 4 : Network Policies bloquent-elles le trafic ?

```bash
# Y a-t-il des Network Policies ?
microk8s kubectl get networkpolicies

# Détails d'une policy
microk8s kubectl describe networkpolicy <policy-name>
```

## Problème 5 : Logs vides ou introuvables

### Symptômes

```bash
microk8s kubectl logs my-app
# (aucune sortie)
```

### Causes et solutions

#### Cause 1 : L'application n'écrit pas sur stdout/stderr

**Vérification :**

```bash
# Accéder au Pod
microk8s kubectl exec -it my-app -- /bin/sh

# Chercher les logs dans des fichiers
ls -la /var/log/
cat /var/log/app.log
```

**Solution :** Configurer l'application pour logger sur stdout/stderr.

#### Cause 2 : Mauvais conteneur (si plusieurs)

```bash
# Lister les conteneurs du Pod
microk8s kubectl get pod my-app -o jsonpath='{.spec.containers[*].name}'

# Logs d'un conteneur spécifique
microk8s kubectl logs my-app -c container-name
```

#### Cause 3 : Le Pod vient de démarrer

```bash
# Attendre quelques secondes puis réessayer
sleep 5
microk8s kubectl logs my-app
```

## Outils de Débogage Essentiels

### 1. kubectl describe - Votre meilleur ami

**Pourquoi c'est important :**
- Montre l'historique complet (Events)
- Affiche les conditions d'erreur
- Donne le contexte complet

**Utilisation :**

```bash
# Pour un Pod
microk8s kubectl describe pod my-app

# Pour un Deployment
microk8s kubectl describe deployment my-app

# Pour un Service
microk8s kubectl describe service my-service

# Pour un Node
microk8s kubectl describe node
```

**Sections importantes :**
1. **Events** (en bas) : Historique chronologique
2. **Status/State** : État actuel
3. **Conditions** : Checks de santé (Ready, ContainersReady, etc.)

### 2. kubectl get events - Vue d'ensemble

```bash
# Tous les événements récents
microk8s kubectl get events --sort-by='.lastTimestamp'

# Derniers événements (10)
microk8s kubectl get events --sort-by='.lastTimestamp' | tail -10

# Événements de type Warning
microk8s kubectl get events --field-selector type=Warning

# Événements d'un Pod spécifique
microk8s kubectl get events --field-selector involvedObject.name=my-app

# Événements en temps réel
microk8s kubectl get events --watch
```

### 3. kubectl logs - Que dit l'application ?

```bash
# Logs de base
microk8s kubectl logs my-app

# Suivre en temps réel
microk8s kubectl logs -f my-app

# Logs avec timestamps
microk8s kubectl logs my-app --timestamps

# 100 dernières lignes
microk8s kubectl logs my-app --tail=100

# Depuis 1 heure
microk8s kubectl logs my-app --since=1h

# Logs du conteneur précédent (crash)
microk8s kubectl logs my-app --previous

# Tous les Pods d'un Deployment
microk8s kubectl logs -l app=my-app

# Sauvegarder les logs
microk8s kubectl logs my-app > app-logs.txt
```

### 4. kubectl exec - Entrer dans le conteneur

```bash
# Shell interactif
microk8s kubectl exec -it my-app -- /bin/sh
microk8s kubectl exec -it my-app -- /bin/bash

# Commande unique
microk8s kubectl exec my-app -- ls /app
microk8s kubectl exec my-app -- cat /etc/config/app.conf
microk8s kubectl exec my-app -- env

# Tester la connectivité
microk8s kubectl exec my-app -- ping google.com
microk8s kubectl exec my-app -- curl http://other-service
microk8s kubectl exec my-app -- nslookup my-service
```

**Commandes utiles dans un conteneur :**

```bash
# Voir les processus
ps aux

# Ports en écoute
netstat -tlnp
ss -tlnp

# Variables d'environnement
env | sort

# Tester HTTP
curl localhost:8080
wget -qO- localhost:8080

# DNS
nslookup my-service
```

### 5. kubectl port-forward - Accès direct

```bash
# Tunnel vers un Pod
microk8s kubectl port-forward my-app 8080:80

# Tunnel vers un Service
microk8s kubectl port-forward service/my-service 8080:80

# Utiliser un port local aléatoire
microk8s kubectl port-forward my-app :80

# Accès : http://localhost:8080
```

### 6. kubectl top - Utilisation des ressources

```bash
# Activer metrics-server si pas déjà fait
microk8s enable metrics-server

# Utilisation des Nodes
microk8s kubectl top nodes

# Utilisation des Pods
microk8s kubectl top pods

# Trier par CPU
microk8s kubectl top pods --sort-by=cpu

# Trier par mémoire
microk8s kubectl top pods --sort-by=memory

# Avec détails des conteneurs
microk8s kubectl top pods --containers
```

## Techniques de Débogage Avancées

### 1. Créer un Pod de test

Pour tester la connectivité réseau ou DNS :

```bash
# Pod temporaire avec busybox
microk8s kubectl run test --image=busybox --rm -it --restart=Never -- sh

# Dans le Pod :
# Tester DNS
nslookup my-service
nslookup my-service.default.svc.cluster.local

# Tester connectivité
wget -qO- http://my-service:80

# Pod avec plus d'outils réseau
microk8s kubectl run netshoot --image=nicolaka/netshoot --rm -it --restart=Never -- bash

# Dans le Pod :
curl http://my-service
dig my-service
traceroute my-service
```

### 2. Debugging d'un Pod qui crash au démarrage

Si le Pod crash trop vite pour déboguer :

```bash
# Option 1 : Remplacer la commande par un sleep
microk8s kubectl run my-app-debug --image=myapp:latest -- sleep 3600

# Puis accéder et tester
microk8s kubectl exec -it my-app-debug -- /bin/sh

# Option 2 : Copier le Deployment et modifier
microk8s kubectl get deployment my-app -o yaml > my-app-debug.yaml

# Éditer my-app-debug.yaml :
# - Changer le nom
# - Remplacer command par: ["sleep", "3600"]
# - Commenter livenessProbe et readinessProbe

microk8s kubectl apply -f my-app-debug.yaml
```

### 3. Comparer les configurations

```bash
# Comparer un Pod qui fonctionne vs un qui ne fonctionne pas
microk8s kubectl get pod working-pod -o yaml > working.yaml
microk8s kubectl get pod broken-pod -o yaml > broken.yaml

diff working.yaml broken.yaml
```

### 4. Vérifier les quotas et limits

```bash
# ResourceQuotas dans le namespace
microk8s kubectl get resourcequota

# Détails
microk8s kubectl describe resourcequota

# LimitRanges
microk8s kubectl get limitrange
microk8s kubectl describe limitrange
```

### 5. Inspecter le scheduling

```bash
# Pourquoi un Pod n'est pas schedulé ?
microk8s kubectl get pod my-app -o yaml | grep -A 10 status

# Voir les conditions
microk8s kubectl get pod my-app -o jsonpath='{.status.conditions[*]}' | jq

# Vérifier les taints sur les Nodes
microk8s kubectl describe nodes | grep -i taint
```

## Check-list de Débogage Rapide

Quand quelque chose ne fonctionne pas, suivez cette check-list :

### ☑️ Étape 1 : État général

```bash
# Vue d'ensemble
microk8s kubectl get all

# Focus sur les Pods
microk8s kubectl get pods

# Questions :
# - Quel est le STATUS ?
# - READY est-il à x/x ?
# - Y a-t-il des RESTARTS ?
```

### ☑️ Étape 2 : Événements

```bash
# Événements récents
microk8s kubectl get events --sort-by='.lastTimestamp' | tail -20

# Événements du Pod
microk8s kubectl describe pod <pod-name> | grep -A 20 Events
```

### ☑️ Étape 3 : Logs

```bash
# Logs actuels
microk8s kubectl logs <pod-name>

# Si crash : logs précédents
microk8s kubectl logs <pod-name> --previous
```

### ☑️ Étape 4 : Configuration

```bash
# Image correcte ?
microk8s kubectl get pod <pod-name> -o jsonpath='{.spec.containers[0].image}'

# Variables d'environnement ?
microk8s kubectl get pod <pod-name> -o jsonpath='{.spec.containers[0].env}'

# Ressources suffisantes ?
microk8s kubectl describe pod <pod-name> | grep -A 5 Requests
```

### ☑️ Étape 5 : Réseau

```bash
# Service existe ?
microk8s kubectl get service

# Endpoints corrects ?
microk8s kubectl get endpoints <service-name>

# Labels correspondent ?
microk8s kubectl get pods --show-labels
microk8s kubectl get service <service-name> -o yaml | grep -A 3 selector
```

### ☑️ Étape 6 : Tests

```bash
# Depuis l'intérieur du Pod
microk8s kubectl exec -it <pod-name> -- curl localhost:8080

# Depuis un Pod test
microk8s kubectl run test --image=busybox --rm -it -- wget -qO- http://<service-name>
```

## Commandes de Diagnostic Utiles

### Obtenir des informations JSON/YAML

```bash
# YAML complet d'un Pod
microk8s kubectl get pod my-app -o yaml

# JSON complet
microk8s kubectl get pod my-app -o json

# Extraire un champ spécifique (JSONPath)
microk8s kubectl get pod my-app -o jsonpath='{.status.phase}'
microk8s kubectl get pod my-app -o jsonpath='{.status.containerStatuses[0].restartCount}'
microk8s kubectl get pod my-app -o jsonpath='{.status.podIP}'
```

### Inspecter les ressources

```bash
# Toutes les ressources d'un type
microk8s kubectl get pods -o wide
microk8s kubectl get services -o wide
microk8s kubectl get nodes -o wide

# Avec les labels
microk8s kubectl get pods --show-labels

# Ressources système
microk8s kubectl get all -n kube-system
```

### Diagnostic réseau

```bash
# DNS interne fonctionne ?
microk8s kubectl run test --image=busybox --rm -it -- nslookup kubernetes.default

# Connectivité inter-pods
microk8s kubectl run test --image=nicolaka/netshoot --rm -it -- bash
# Dans le conteneur :
ping <pod-ip>
curl http://<service-name>
```

## Problèmes Courants - Solutions Rapides

### Pod en "ErrImagePull"

```bash
# Vérifier le nom de l'image
microk8s kubectl get deployment my-app -o jsonpath='{.spec.template.spec.containers[0].image}'

# Corriger si nécessaire
microk8s kubectl set image deployment/my-app my-app=nginx:1.21
```

### "Back-off restarting failed container"

```bash
# Voir les logs du crash
microk8s kubectl logs <pod-name> --previous

# Vérifier le Exit Code
microk8s kubectl describe pod <pod-name> | grep "Exit Code"
```

### Service n'a pas d'Endpoints

```bash
# Vérifier les Endpoints
microk8s kubectl get endpoints my-service

# Si vide, problème de selector
# Comparer les labels
microk8s kubectl get service my-service -o yaml | grep -A 3 selector
microk8s kubectl get pods --show-labels
```

### "OOMKilled" (Out Of Memory)

```bash
# Augmenter la limite mémoire
microk8s kubectl set resources deployment my-app \
  --limits=memory=512Mi \
  --requests=memory=256Mi
```

### ConfigMap ou Secret non trouvé

```bash
# Vérifier qu'il existe
microk8s kubectl get configmap
microk8s kubectl get secret

# Créer si manquant
microk8s kubectl create configmap my-config --from-literal=key=value
```

## Récapitulatif

Dans cette section, vous avez appris :

✅ Une **méthodologie systématique** de débogage en 5 étapes
✅ Comprendre les **états des Pods** et leur signification
✅ Diagnostiquer les problèmes courants : Pending, CrashLoopBackOff, ImagePullBackOff
✅ Utiliser les **outils essentiels** : describe, logs, events, exec, port-forward
✅ Appliquer des **techniques avancées** de débogage
✅ Suivre une **check-list rapide** pour résoudre efficacement les problèmes

**Points clés :**

- **Ne paniquez pas** : Lisez les messages d'erreur
- Les **Events** sont votre meilleure source d'information
- `kubectl describe` montre l'historique complet
- Les **logs** révèlent ce que fait l'application
- Testez **depuis l'intérieur** du Pod avec `exec`
- Suivez une **méthodologie systématique**

**Réflexe de débogage :**
1. `kubectl get pods` → Quel est l'état ?
2. `kubectl describe pod` → Que disent les Events ?
3. `kubectl logs` → Que dit l'application ?

Avec ces outils et cette méthodologie, vous êtes maintenant équipé pour diagnostiquer et résoudre la majorité des problèmes que vous rencontrerez dans Kubernetes !

---

**Fin de la Partie 1 - Fondamentaux**

Vous avez maintenant une base solide pour travailler avec Kubernetes et MicroK8s. La Partie 2 abordera les compétences intermédiaires : addons, stockage persistant, réseau avancé, et plus encore.

⏭️ [Addons MicroK8s - Le Cœur de la Simplicité](/05-addons-microk8s/README.md)
