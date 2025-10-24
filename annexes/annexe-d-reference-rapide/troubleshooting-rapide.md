🔝 Retour au [Sommaire](/SOMMAIRE.md)

# Troubleshooting Rapide - MicroK8s

## Introduction

Le dépannage (troubleshooting) est une compétence essentielle pour travailler avec Kubernetes. Ce guide vous fournit des solutions rapides aux problèmes les plus courants rencontrés avec MicroK8s.

### Comment Utiliser ce Guide

1. **Identifiez le symptôme** : Quel est le problème visible ?
2. **Trouvez la section correspondante** : Utilisez la table des matières
3. **Suivez les étapes de diagnostic** : Dans l'ordre proposé
4. **Appliquez la solution** : Une fois la cause identifiée

### Structure des Solutions

Chaque problème suit ce format :

```
🔴 Symptôme : Description du problème visible
🔍 Diagnostic : Commandes pour identifier la cause
💡 Cause(s) : Raisons possibles
✅ Solution : Étapes pour résoudre
```

---

## Table des Matières Rapide

**MicroK8s Ne Démarre Pas**
- [MicroK8s ne démarre pas après installation](#microk8s-ne-démarre-pas)
- [MicroK8s bloqué en "starting"](#microk8s-bloqué-en-starting)
- [Erreur "Permission denied"](#erreur-permission-denied)

**Problèmes de Pods**
- [Pod en état Pending](#pod-en-état-pending)
- [Pod en CrashLoopBackOff](#pod-en-crashloopbackoff)
- [Pod en ImagePullBackOff](#pod-en-imagepullbackoff)
- [Pod en Error](#pod-en-état-error)
- [Pod redémarre constamment](#pod-redémarre-constamment)

**Problèmes Réseau**
- [Services inaccessibles](#services-inaccessibles)
- [DNS ne fonctionne pas](#dns-ne-fonctionne-pas)
- [Ingress ne répond pas](#ingress-ne-répond-pas)
- [Pas de connectivité entre pods](#pas-de-connectivité-entre-pods)

**Problèmes de Stockage**
- [PVC en état Pending](#pvc-en-état-pending)
- [Volume non monté](#volume-non-monté)
- [Problèmes d'espace disque](#problèmes-despace-disque)

**Problèmes de Performance**
- [Cluster lent](#cluster-lent)
- [Mémoire insuffisante](#mémoire-insuffisante)

---

## Problèmes MicroK8s

### MicroK8s Ne Démarre Pas

#### 🔴 Symptôme

```bash
microk8s status
# microk8s is not running
```

#### 🔍 Diagnostic

```bash
# Vérifier l'état du service
sudo systemctl status snap.microk8s.daemon-kubelite

# Voir les logs système
sudo journalctl -u snap.microk8s.daemon-kubelite -n 50

# Vérifier l'espace disque
df -h
```

#### 💡 Causes Possibles

1. **Manque de ressources** (RAM, CPU, disque)
2. **Port déjà utilisé** (16443, 10250)
3. **Conflit avec autre installation Kubernetes**
4. **Corruption de fichiers de configuration**

#### ✅ Solutions

**Solution 1 : Redémarrage simple**
```bash
sudo microk8s stop
sudo microk8s start
microk8s status --wait-ready
```

**Solution 2 : Vérifier les ressources**
```bash
# Mémoire disponible
free -h
# Doit avoir au moins 2GB disponible

# CPU disponible
lscpu | grep "CPU(s)"
# Doit avoir au moins 2 cores

# Espace disque
df -h /var/snap/microk8s
# Doit avoir au moins 20GB disponible
```

**Solution 3 : Vérifier les ports**
```bash
# Vérifier si le port 16443 est libre
sudo netstat -tulpn | grep 16443

# Si occupé, identifier le processus et l'arrêter
sudo lsof -i :16443
```

**Solution 4 : Réinstallation propre**
```bash
# Sauvegarder vos configs importantes d'abord !
microk8s reset --destroy-storage
sudo snap remove microk8s
sudo snap install microk8s --classic
sudo usermod -a -G microk8s $USER
newgrp microk8s
```

### MicroK8s Bloqué en "Starting"

#### 🔴 Symptôme

```bash
microk8s status
# microk8s is starting
# (reste bloqué pendant plusieurs minutes)
```

#### 🔍 Diagnostic

```bash
# Voir ce qui se passe
sudo journalctl -u snap.microk8s.daemon-kubelite -f

# Vérifier les pods système
microk8s kubectl get pods -n kube-system

# Rapport de diagnostic complet
microk8s inspect
```

#### 💡 Causes Possibles

1. **Pods système ne démarrent pas**
2. **Problème de réseau CNI**
3. **Manque de ressources**

#### ✅ Solutions

**Solution 1 : Attendre plus longtemps**
```bash
# Parfois il faut juste être patient (5-10 minutes)
microk8s status --wait-ready --timeout 600
```

**Solution 2 : Forcer le redémarrage**
```bash
sudo snap restart microk8s
microk8s status --wait-ready
```

**Solution 3 : Reset complet**
```bash
microk8s reset
microk8s start
```

### Erreur "Permission Denied"

#### 🔴 Symptôme

```bash
microk8s kubectl get pods
# Error: permission denied
```

#### 🔍 Diagnostic

```bash
# Vérifier les groupes de l'utilisateur
groups

# Le groupe "microk8s" doit apparaître
```

#### ✅ Solution

```bash
# Ajouter l'utilisateur au groupe
sudo usermod -a -G microk8s $USER

# Changer les permissions du dossier kube
sudo chown -f -R $USER ~/.kube

# Recharger les groupes (ou se déconnecter/reconnecter)
newgrp microk8s

# Vérifier
microk8s kubectl get nodes
```

---

## Problèmes de Pods

### Pod en État Pending

#### 🔴 Symptôme

```bash
kubectl get pods
# NAME    READY   STATUS    RESTARTS   AGE
# myapp   0/1     Pending   0          5m
```

#### 🔍 Diagnostic

```bash
# Voir les détails du pod
kubectl describe pod myapp

# Regarder les événements (Events section en bas)
kubectl get events --sort-by='.lastTimestamp'

# Vérifier les ressources disponibles
kubectl top nodes
kubectl describe nodes
```

#### 💡 Causes Possibles

1. **Ressources insuffisantes** (CPU, RAM)
2. **Pas de nœud disponible**
3. **Volume PVC non disponible**
4. **Contraintes de scheduling non satisfaites** (nodeSelector, affinity)
5. **ImagePullBackOff caché**

#### ✅ Solutions

**Cause 1 : Ressources insuffisantes**

```bash
# Vérifier les ressources
kubectl describe pod myapp | grep -A 5 "Requests"

# Solutions :
# a) Réduire les requests
kubectl edit pod myapp
# Ou modifier le deployment/fichier YAML

# b) Scaler à la baisse d'autres applications
kubectl scale deployment autre-app --replicas=1

# c) Ajouter plus de ressources au nœud
```

**Cause 2 : PVC non disponible**

```bash
# Vérifier les PVC
kubectl get pvc

# Si PVC en Pending
kubectl describe pvc nom-pvc

# Solution : Activer le stockage
microk8s enable hostpath-storage

# Ou créer un PV manuellement
```

**Cause 3 : Contraintes de scheduling**

```bash
# Voir si des contraintes bloquent
kubectl describe pod myapp | grep -A 10 "Events"

# Solution : Retirer ou modifier les contraintes
kubectl edit pod myapp
# Supprimer nodeSelector, affinity, ou taints/tolerations
```

### Pod en CrashLoopBackOff

#### 🔴 Symptôme

```bash
kubectl get pods
# NAME    READY   STATUS             RESTARTS   AGE
# myapp   0/1     CrashLoopBackOff   5          3m
```

Le pod démarre, crash, redémarre, crash à nouveau, etc.

#### 🔍 Diagnostic

```bash
# Logs actuels
kubectl logs myapp

# Logs du conteneur précédent (avant le crash)
kubectl logs myapp --previous

# Détails du pod
kubectl describe pod myapp

# Se connecter si le pod reste vivant assez longtemps
kubectl exec -it myapp -- sh
```

#### 💡 Causes Possibles

1. **Application crash au démarrage**
2. **Configuration incorrecte** (variables d'env, ConfigMap)
3. **Dépendance manquante** (base de données non accessible)
4. **Liveness probe trop agressive**
5. **Commande de démarrage incorrecte**

#### ✅ Solutions

**Solution 1 : Vérifier les logs**

```bash
# Voir l'erreur exacte
kubectl logs myapp --previous

# Exemples d'erreurs courantes :
# - "Connection refused" → Service dépendant non disponible
# - "Config file not found" → ConfigMap manquant
# - "Permission denied" → Problème de droits
# - "Port already in use" → Conflit de ports
```

**Solution 2 : Vérifier la configuration**

```bash
# ConfigMap existe ?
kubectl get configmap

# Secret existe ?
kubectl get secret

# Variables d'environnement correctes ?
kubectl describe pod myapp | grep -A 20 "Environment"
```

**Solution 3 : Désactiver temporairement les probes**

```yaml
# Éditer le pod/deployment
kubectl edit deployment myapp

# Commenter ou supprimer temporairement :
# livenessProbe:
#   httpGet:
#     path: /health
#     port: 8080
```

**Solution 4 : Tester la commande localement**

```bash
# Si l'image est accessible
docker run -it --entrypoint sh your-image:tag

# Tester manuellement la commande
./start.sh
```

### Pod en ImagePullBackOff

#### 🔴 Symptôme

```bash
kubectl get pods
# NAME    READY   STATUS             RESTARTS   AGE
# myapp   0/1     ImagePullBackOff   0          2m
```

#### 🔍 Diagnostic

```bash
# Détails du pod
kubectl describe pod myapp

# Chercher la section "Events"
# Exemple : "Failed to pull image"

# Vérifier l'image spécifiée
kubectl get pod myapp -o jsonpath='{.spec.containers[*].image}'
```

#### 💡 Causes Possibles

1. **Image n'existe pas** (nom incorrect, tag incorrect)
2. **Registry privé sans authentification**
3. **Problème de connectivité réseau**
4. **Rate limit du registry**

#### ✅ Solutions

**Solution 1 : Vérifier le nom de l'image**

```bash
# Voir l'image exacte utilisée
kubectl describe pod myapp | grep "Image:"

# Vérifier que l'image existe
docker pull your-image:tag

# Corriger si erreur de nom/tag
kubectl set image deployment/myapp container-name=correct-image:tag
```

**Solution 2 : Registry privé**

```bash
# Créer un secret pour le registry
kubectl create secret docker-registry regcred \
  --docker-server=registry.example.com \
  --docker-username=user \
  --docker-password=password \
  --docker-email=user@example.com

# Ajouter au pod/deployment
kubectl edit deployment myapp

# Ajouter :
spec:
  template:
    spec:
      imagePullSecrets:
      - name: regcred
```

**Solution 3 : Utiliser le registry local**

```bash
# Activer le registry MicroK8s
microk8s enable registry

# Tag et push l'image localement
docker tag your-image:tag localhost:32000/your-image:tag
docker push localhost:32000/your-image:tag

# Utiliser l'image locale
kubectl set image deployment/myapp container-name=localhost:32000/your-image:tag
```

### Pod en État Error

#### 🔴 Symptôme

```bash
kubectl get pods
# NAME    READY   STATUS   RESTARTS   AGE
# myapp   0/1     Error    0          1m
```

#### 🔍 Diagnostic

```bash
# Logs du pod
kubectl logs myapp

# Détails
kubectl describe pod myapp

# Exit code
kubectl get pod myapp -o jsonpath='{.status.containerStatuses[0].state.terminated.exitCode}'
```

#### 💡 Exit Codes Courants

| Exit Code | Signification | Cause probable |
|-----------|---------------|----------------|
| 0 | Succès | Normal (pour un Job) |
| 1 | Erreur générale | Bug applicatif |
| 2 | Mauvaise utilisation | Commande incorrecte |
| 126 | Commande non exécutable | Permissions |
| 127 | Commande non trouvée | Path incorrect |
| 137 | SIGKILL | OOMKilled (mémoire) |
| 139 | SIGSEGV | Segmentation fault |
| 143 | SIGTERM | Arrêt gracieux |

#### ✅ Solutions

**Exit Code 137 (OOMKilled)**

```bash
# Le pod a été tué par manque de mémoire
# Augmenter les limites
kubectl edit deployment myapp

# Augmenter :
resources:
  limits:
    memory: "1Gi"  # Augmenter
```

**Exit Code 1 (Erreur application)**

```bash
# Vérifier les logs pour l'erreur exacte
kubectl logs myapp

# Corriger le code ou la configuration
```

**Exit Code 127 (Commande non trouvée)**

```bash
# Vérifier la commande dans le pod
kubectl describe pod myapp | grep -A 5 "Command"

# Corriger la commande ou le path
```

### Pod Redémarre Constamment

#### 🔴 Symptôme

```bash
kubectl get pods
# NAME    READY   STATUS    RESTARTS   AGE
# myapp   1/1     Running   15         5m
```

Le compteur RESTARTS augmente continuellement.

#### 🔍 Diagnostic

```bash
# Vérifier les restarts
kubectl get pod myapp

# Voir pourquoi
kubectl describe pod myapp | grep -A 10 "Last State"

# Vérifier les probes
kubectl describe pod myapp | grep -A 5 "Liveness"
kubectl describe pod myapp | grep -A 5 "Readiness"

# Logs
kubectl logs myapp
```

#### 💡 Causes Possibles

1. **Liveness probe échoue** (application trop lente)
2. **Application crash périodiquement**
3. **Fuite mémoire** (OOMKilled)
4. **Dépendance intermittente**

#### ✅ Solutions

**Solution 1 : Probe trop stricte**

```yaml
# Augmenter les délais
livenessProbe:
  httpGet:
    path: /health
    port: 8080
  initialDelaySeconds: 60  # Augmenter de 30 à 60
  periodSeconds: 20        # Augmenter de 10 à 20
  timeoutSeconds: 5        # Augmenter de 1 à 5
  failureThreshold: 5      # Augmenter de 3 à 5
```

**Solution 2 : Vérifier les ressources**

```bash
# Surveiller l'utilisation
kubectl top pod myapp

# Si près des limites, augmenter
kubectl edit deployment myapp
```

**Solution 3 : Désactiver temporairement la probe**

```bash
kubectl edit deployment myapp
# Commenter livenessProbe pour tester
```

---

## Problèmes Réseau

### Services Inaccessibles

#### 🔴 Symptôme

```bash
curl http://my-service
# curl: (7) Failed to connect
```

#### 🔍 Diagnostic

```bash
# Le service existe ?
kubectl get svc my-service

# Il a des endpoints (pods backend) ?
kubectl get endpoints my-service

# Si vide, c'est que les sélecteurs ne matchent pas
kubectl describe svc my-service

# Comparer avec les labels des pods
kubectl get pods --show-labels
```

#### ✅ Solutions

**Solution 1 : Pas d'endpoints**

```bash
# Vérifier le sélecteur du service
kubectl describe svc my-service | grep Selector

# Vérifier les labels des pods
kubectl get pods --show-labels

# Ils doivent correspondre !
# Corriger soit le service, soit les pods
kubectl edit svc my-service
# ou
kubectl edit deployment my-app
```

**Solution 2 : Mauvais port**

```bash
# Vérifier les ports
kubectl describe svc my-service

# Port du service vs port du conteneur
kubectl get pod my-pod -o jsonpath='{.spec.containers[*].ports}'

# Corriger si nécessaire
kubectl edit svc my-service
```

**Solution 3 : Service type incorrect**

```bash
# Pour accès externe, utiliser NodePort ou LoadBalancer
kubectl edit svc my-service

# Changer :
spec:
  type: LoadBalancer  # ou NodePort
```

### DNS Ne Fonctionne Pas

#### 🔴 Symptôme

```bash
# Depuis un pod
nslookup my-service
# Server can't find my-service: NXDOMAIN
```

#### 🔍 Diagnostic

```bash
# CoreDNS actif ?
microk8s status | grep dns

# Pods CoreDNS running ?
kubectl get pods -n kube-system -l k8s-app=kube-dns

# Service DNS existe ?
kubectl get svc -n kube-system kube-dns

# Test DNS depuis un pod
kubectl run test --image=busybox:1.28 --rm -it --restart=Never -- nslookup kubernetes.default
```

#### ✅ Solutions

**Solution 1 : Activer DNS**

```bash
microk8s enable dns

# Attendre que les pods démarrent
kubectl wait --for=condition=ready pod -l k8s-app=kube-dns -n kube-system --timeout=120s
```

**Solution 2 : Redémarrer CoreDNS**

```bash
kubectl rollout restart deployment coredns -n kube-system

# Vérifier
kubectl get pods -n kube-system -l k8s-app=kube-dns
```

**Solution 3 : Vérifier la configuration**

```bash
# Voir la config CoreDNS
kubectl get configmap coredns -n kube-system -o yaml

# Si corrompue, désactiver/réactiver
microk8s disable dns
microk8s enable dns
```

### Ingress Ne Répond Pas

#### 🔴 Symptôme

```bash
curl http://myapp.example.com
# curl: (7) Failed to connect
# ou
# 404 Not Found
```

#### 🔍 Diagnostic

```bash
# Ingress Controller actif ?
microk8s status | grep ingress

# Pods Ingress running ?
kubectl get pods -n ingress

# Service Ingress a une IP externe ?
kubectl get svc -n ingress

# Ingress resource existe ?
kubectl get ingress

# Détails de l'Ingress
kubectl describe ingress my-ingress
```

#### ✅ Solutions

**Solution 1 : Activer Ingress**

```bash
microk8s enable ingress

# Attendre
kubectl wait --for=condition=ready pod -l app.kubernetes.io/name=ingress-nginx -n ingress --timeout=120s
```

**Solution 2 : Vérifier l'Ingress Resource**

```bash
kubectl describe ingress my-ingress

# Vérifier :
# 1. Host correct
# 2. Service backend existe
# 3. Path correct

# Corriger si nécessaire
kubectl edit ingress my-ingress
```

**Solution 3 : Vérifier le Service Backend**

```bash
# Le service référencé existe ?
kubectl get svc backend-service

# Il a des endpoints ?
kubectl get endpoints backend-service

# Si vide, problème de sélecteur (voir section Services)
```

**Solution 4 : Vérifier DNS/Réseau**

```bash
# DNS pointe vers votre IP ?
nslookup myapp.example.com

# Port forwarding configuré (80, 443) ?
sudo iptables -L -n | grep -E "80|443"

# Firewall autorise ?
sudo ufw status
```

### Pas de Connectivité Entre Pods

#### 🔴 Symptôme

```bash
# Depuis pod frontend
curl http://backend-service:8080
# curl: (7) Failed to connect
```

#### 🔍 Diagnostic

```bash
# Pods actifs ?
kubectl get pods

# Service backend existe ?
kubectl get svc backend-service

# DNS fonctionne ?
kubectl exec frontend-pod -- nslookup backend-service

# Network Policies bloquent ?
kubectl get networkpolicies

# Calico fonctionne ?
kubectl get pods -n kube-system -l k8s-app=calico-node
```

#### ✅ Solutions

**Solution 1 : Service manquant**

```bash
# Créer le service
kubectl expose deployment backend --port=8080

# Vérifier
kubectl get svc backend
```

**Solution 2 : Network Policy trop restrictive**

```bash
# Lister les policies
kubectl get networkpolicies

# Voir laquelle bloque
kubectl describe networkpolicy policy-name

# Désactiver temporairement pour tester
kubectl delete networkpolicy policy-name

# Si ça fonctionne, corriger la policy
```

**Solution 3 : Calico en erreur**

```bash
# Vérifier Calico
kubectl get pods -n kube-system -l k8s-app=calico-node

# Si problème, redémarrer
kubectl rollout restart daemonset calico-node -n kube-system
```

---

## Problèmes de Stockage

### PVC en État Pending

#### 🔴 Symptôme

```bash
kubectl get pvc
# NAME       STATUS    VOLUME   CAPACITY   ACCESS MODES
# data-pvc   Pending
```

#### 🔍 Diagnostic

```bash
# Détails du PVC
kubectl describe pvc data-pvc

# PV disponibles ?
kubectl get pv

# StorageClass existe ?
kubectl get storageclass

# Addon storage activé ?
microk8s status | grep storage
```

#### ✅ Solutions

**Solution 1 : Activer le stockage**

```bash
microk8s enable hostpath-storage

# Vérifier
kubectl get storageclass
kubectl get pvc  # Devrait passer à Bound
```

**Solution 2 : Pas de PV correspondant**

```bash
# Créer un PV manuellement
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv-data
spec:
  capacity:
    storage: 5Gi
  accessModes:
  - ReadWriteOnce
  storageClassName: standard
  hostPath:
    path: /mnt/data
EOF

# Le PVC devrait se binder automatiquement
```

**Solution 3 : StorageClass incorrecte**

```bash
# Vérifier le storageClassName dans le PVC
kubectl get pvc data-pvc -o yaml | grep storageClassName

# Changer pour correspondre à un existant
kubectl edit pvc data-pvc
```

### Volume Non Monté

#### 🔴 Symptôme

```bash
# Pod démarre mais application plante
kubectl logs myapp
# Error: cannot access /data: no such file or directory
```

#### 🔍 Diagnostic

```bash
# Vérifier les volumes du pod
kubectl describe pod myapp | grep -A 10 "Volumes"

# PVC bound ?
kubectl get pvc

# VolumeMount correct ?
kubectl get pod myapp -o yaml | grep -A 5 volumeMounts
```

#### ✅ Solutions

**Solution 1 : PVC non bound**

Voir section "PVC en État Pending" ci-dessus.

**Solution 2 : VolumeMount incorrect**

```bash
# Vérifier le chemin
kubectl describe pod myapp | grep -A 10 "Mounts"

# Corriger le deployment
kubectl edit deployment myapp

# S'assurer que :
# volumeMounts.mountPath = chemin attendu par l'app
# volumes.persistentVolumeClaim.claimName = nom correct du PVC
```

### Problèmes d'Espace Disque

#### 🔴 Symptôme

```bash
# Pods evicted
kubectl get pods
# NAME    READY   STATUS    RESTARTS   AGE
# myapp   0/1     Evicted   0          5m
```

#### 🔍 Diagnostic

```bash
# Espace disque sur le nœud
df -h

# Voir le seuil d'éviction
kubectl describe node | grep -A 5 "Allocatable"

# Images Docker prenant de la place
microk8s ctr images ls

# Volumes
du -sh /var/snap/microk8s/common/var/lib/kubelet/pods/*
```

#### ✅ Solutions

**Solution 1 : Nettoyer les images**

```bash
# Lister les images
microk8s ctr images ls

# Supprimer les images inutilisées
microk8s ctr images rm IMAGE_NAME

# Ou toutes les images non utilisées
microk8s ctr images prune
```

**Solution 2 : Nettoyer les pods terminated**

```bash
# Supprimer les pods Completed
kubectl delete pods --field-selector=status.phase==Succeeded -A

# Supprimer les pods Failed
kubectl delete pods --field-selector=status.phase==Failed -A

# Supprimer les pods Evicted
kubectl get pods -A | grep Evicted | awk '{print $2, $1}' | xargs -n2 kubectl delete pod -n
```

**Solution 3 : Nettoyer les logs**

```bash
# Logs de conteneurs
sudo sh -c 'find /var/snap/microk8s/common/var/log/pods/ -type f -name "*.log" -delete'

# Logs système
sudo journalctl --vacuum-time=7d
```

**Solution 4 : Augmenter l'espace**

```bash
# Ajouter un disque ou étendre le volume existant
# Puis déplacer /var/snap/microk8s vers le nouveau disque
```

---

## Problèmes de Performance

### Cluster Lent

#### 🔴 Symptôme

- Commandes kubectl lentes
- Pods démarrent lentement
- Applications répondent lentement

#### 🔍 Diagnostic

```bash
# Utilisation des ressources
kubectl top nodes
kubectl top pods -A --sort-by=cpu
kubectl top pods -A --sort-by=memory

# I/O disque
iostat -x 2 10

# Nombre de pods
kubectl get pods -A | wc -l

# Événements récents
kubectl get events -A --sort-by='.lastTimestamp' | tail -20
```

#### ✅ Solutions

**Solution 1 : Trop de pods**

```bash
# Scaler à la baisse les déploiements non critiques
kubectl scale deployment non-critical-app --replicas=1

# Supprimer les ressources inutilisées
kubectl delete all -l environment=test
```

**Solution 2 : Ressources surchargées**

```bash
# Identifier les gros consommateurs
kubectl top pods -A --sort-by=cpu | head -10
kubectl top pods -A --sort-by=memory | head -10

# Réduire les replicas ou les ressources
kubectl scale deployment big-app --replicas=2
```

**Solution 3 : I/O disque saturé**

```bash
# Passer à un SSD si sur HDD
# Ou utiliser un StorageClass plus performant

# Nettoyer les logs (voir section Espace Disque)
```

### Mémoire Insuffisante

#### 🔴 Symptôme

```bash
# Pods OOMKilled
kubectl get pods
# NAME    READY   STATUS      RESTARTS   AGE
# myapp   0/1     OOMKilled   5          2m
```

#### 🔍 Diagnostic

```bash
# Mémoire du système
free -h

# Utilisation par pod
kubectl top pods -A --sort-by=memory

# Limites définies
kubectl describe pod myapp | grep -A 5 "Limits"

# Événements OOM
kubectl get events | grep OOM
```

#### ✅ Solutions

**Solution 1 : Augmenter les limites**

```bash
# Éditer le deployment
kubectl edit deployment myapp

# Augmenter les limites
resources:
  limits:
    memory: "1Gi"  # Au lieu de 512Mi
  requests:
    memory: "512Mi"
```

**Solution 2 : Optimiser l'application**

- Identifier les fuites mémoire
- Optimiser le code
- Réduire la taille des caches

**Solution 3 : Scaler horizontalement**

```bash
# Au lieu d'augmenter la mémoire, augmenter les replicas
kubectl scale deployment myapp --replicas=3

# Avec moins de mémoire par pod
```

**Solution 4 : Ajouter de la RAM au système**

```bash
# Vérifier la RAM totale
free -h

# Ajouter plus de RAM physique
# Ou augmenter la VM si dans une VM
```

---

## Outils de Diagnostic

### microk8s inspect

Génère un rapport complet de diagnostic.

```bash
# Créer un rapport
microk8s inspect

# Résultat : /var/snap/microk8s/common/inspection-report-TIMESTAMP.tar.gz

# Extraire et analyser
tar -xzf inspection-report-*.tar.gz
cd inspection-report-*/

# Contient :
# - Logs de tous les services
# - État des pods système
# - Configuration réseau
# - Snapshots des ressources
```

### kubectl debug

Créer un pod de debug.

```bash
# Pod temporaire avec outils réseau
kubectl run debug --image=nicolaka/netshoot --rm -it --restart=Never -- bash

# Depuis ce pod, tester :
# DNS
nslookup kubernetes.default

# Connectivité
curl http://backend-service:8080

# Résolution d'IP
dig backend-service.default.svc.cluster.local
```

### Logs et Événements

```bash
# Événements récents (très utile !)
kubectl get events --sort-by='.lastTimestamp' | tail -20

# Événements d'un namespace
kubectl get events -n production --sort-by='.lastTimestamp'

# Événements d'une ressource
kubectl describe pod myapp | grep -A 10 Events

# Logs MicroK8s
sudo journalctl -u snap.microk8s.daemon-kubelite -n 100

# Logs en temps réel
sudo journalctl -u snap.microk8s.daemon-kubelite -f
```

### Tests de Connectivité

```bash
# Test DNS depuis l'hôte
nslookup example.com

# Test connectivité pod-to-pod
kubectl run test-source --image=busybox --rm -it -- wget -O- http://backend-service:8080

# Test depuis un pod existant
kubectl exec frontend-pod -- curl http://backend-service:8080

# Port-forward pour test local
kubectl port-forward svc/backend-service 8080:8080
curl http://localhost:8080
```

---

## Checklist de Dépannage Générale

Quand quelque chose ne fonctionne pas, suivez cette checklist :

### 1. Identifier le Problème

```bash
# Statut général
microk8s status

# Pods problématiques
kubectl get pods -A | grep -v Running | grep -v Completed

# Ressources système
kubectl top nodes
kubectl top pods -A
```

### 2. Collecter les Informations

```bash
# Logs du pod
kubectl logs problematic-pod

# Logs précédents (si crashé)
kubectl logs problematic-pod --previous

# Détails complets
kubectl describe pod problematic-pod

# Événements récents
kubectl get events --sort-by='.lastTimestamp' | tail -20
```

### 3. Vérifications Standard

```bash
# ✓ Addons essentiels actifs ?
microk8s status

# ✓ Pods système running ?
kubectl get pods -n kube-system

# ✓ DNS fonctionne ?
kubectl run test --image=busybox:1.28 --rm -it -- nslookup kubernetes.default

# ✓ Ressources suffisantes ?
kubectl top nodes

# ✓ Espace disque ?
df -h
```

### 4. Recherche d'Erreurs Communes

```bash
# Dans les logs
kubectl logs pod-name | grep -i error
kubectl logs pod-name | grep -i fail
kubectl logs pod-name | grep -i exception

# Dans les événements
kubectl get events | grep -i warning
kubectl get events | grep -i error

# Dans les descriptions
kubectl describe pod pod-name | grep -i error
```

### 5. Tests d'Isolation

```bash
# Le problème est-il spécifique à :
# - Un pod ? → Supprimer et recréer
# - Un déploiement ? → Rollback
# - Un namespace ? → Vérifier les quotas/limits
# - Le cluster ? → Redémarrer MicroK8s
```

---

## Solutions d'Urgence

### Tout Redémarrer

```bash
# Redémarrage complet de MicroK8s
microk8s stop
microk8s start
microk8s status --wait-ready

# Ou via snap
sudo snap restart microk8s
```

### Redémarrer un Deployment

```bash
# Force un nouveau rollout
kubectl rollout restart deployment/app-name

# Supprimer tous les pods (ils seront recréés)
kubectl delete pods -l app=app-name
```

### Reset Partiel

```bash
# Reset sans détruire le stockage
microk8s reset

# Réactiver les addons nécessaires
microk8s enable dns
microk8s enable ingress
```

### Reset Complet

```bash
# ⚠️ ATTENTION : Perte de toutes les données !

# Sauvegarder ce qui est important
kubectl get all -A -o yaml > backup.yaml

# Reset total
microk8s reset --destroy-storage

# Ou réinstallation
sudo snap remove microk8s
sudo snap install microk8s --classic
```

---

## Prévention des Problèmes

### Bonnes Pratiques

1. **Toujours définir des ressources**
```yaml
resources:
  requests:
    memory: "256Mi"
    cpu: "250m"
  limits:
    memory: "512Mi"
    cpu: "500m"
```

2. **Toujours définir des health checks**
```yaml
livenessProbe:
  httpGet:
    path: /health
    port: 8080
readinessProbe:
  httpGet:
    path: /ready
    port: 8080
```

3. **Utiliser des versions d'images spécifiques**
```yaml
# ✗ Éviter
image: nginx:latest

# ✓ Préférer
image: nginx:1.21.3
```

4. **Tester avant de déployer**
```bash
kubectl apply -f deployment.yaml --dry-run=client
kubectl diff -f deployment.yaml
```

5. **Surveiller régulièrement**
```bash
# Script de monitoring quotidien
kubectl top nodes
kubectl get pods -A | grep -v Running | grep -v Completed
kubectl get events --sort-by='.lastTimestamp' | tail -20
df -h
```

### Maintenance Préventive

```bash
# Hebdomadaire
# Nettoyer les pods terminated
kubectl delete pods --field-selector=status.phase==Succeeded -A
kubectl delete pods --field-selector=status.phase==Failed -A

# Mensuel
# Nettoyer les images
microk8s ctr images prune

# Nettoyer les logs
sudo journalctl --vacuum-time=30d

# Trimestriel
# Mise à jour MicroK8s
sudo snap refresh microk8s --channel=latest/stable

# Backup complet
kubectl get all -A -o yaml > backup-$(date +%Y%m%d).yaml
microk8s inspect
```

---

## Quand Demander de l'Aide

Si après avoir suivi ce guide vous êtes toujours bloqué :

### 1. Rassembler les Informations

```bash
# Créer un rapport complet
microk8s inspect

# Sauvegarder les configs problématiques
kubectl get pod problematic-pod -o yaml > pod-config.yaml
kubectl describe pod problematic-pod > pod-describe.txt
kubectl logs problematic-pod > pod-logs.txt
```

### 2. Où Demander de l'Aide

- **Forum MicroK8s** : https://discuss.kubernetes.io/
- **GitHub Issues** : https://github.com/canonical/microk8s/issues
- **Stack Overflow** : Tag `microk8s`
- **Kubernetes Slack** : Canal #microk8s

### 3. Informations à Fournir

- Version de MicroK8s : `microk8s version`
- Système d'exploitation : `cat /etc/os-release`
- Description du problème
- Ce que vous avez déjà essayé
- Logs et fichiers de configuration (anonymisés)
- Rapport d'inspection

---

## Résumé - Commandes Essentielles de Dépannage

```bash
# État général
microk8s status
kubectl get pods -A
kubectl get events --sort-by='.lastTimestamp'

# Diagnostic d'un pod
kubectl describe pod [name]
kubectl logs [name]
kubectl logs [name] --previous

# Ressources
kubectl top nodes
kubectl top pods -A
df -h
free -h

# Réseau
kubectl get svc
kubectl get endpoints
kubectl get ingress

# Rapport complet
microk8s inspect

# Logs système
sudo journalctl -u snap.microk8s.daemon-kubelite -n 100

# Redémarrages
kubectl rollout restart deployment/[name]
microk8s stop && microk8s start
sudo snap restart microk8s
```

---

## Conclusion

Le troubleshooting est une compétence qui s'améliore avec la pratique. Les points clés à retenir :

✅ **Commencez par les logs** : `kubectl logs` et `kubectl describe`
✅ **Vérifiez les événements** : `kubectl get events`
✅ **Testez méthodiquement** : Isoler le problème
✅ **Utilisez les outils** : `microk8s inspect`, `kubectl top`
✅ **Documentez** : Notez les solutions qui fonctionnent

**Gardez ce guide à portée de main** : Il vous fera gagner un temps précieux !

Bon dépannage ! 🔧

⏭️ [Glossaire des termes Kubernetes](/annexes/annexe-d-reference-rapide/glossaire-des-termes-kubernetes.md)
