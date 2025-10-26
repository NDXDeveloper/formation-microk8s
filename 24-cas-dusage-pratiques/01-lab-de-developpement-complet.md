🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 24.1 Lab de Développement Complet

## Introduction

Un lab de développement complet avec MicroK8s vous permet de créer un environnement professionnel sur votre machine personnelle, parfaitement adapté pour apprendre Kubernetes, développer des applications cloud-native, et tester vos déploiements avant de les mettre en production.

Dans ce chapitre, nous allons construire ensemble un environnement de développement complet qui intègre tous les outils essentiels dont un développeur moderne a besoin.

## Objectifs du Lab de Développement

Un lab de développement complet doit vous permettre de :

- **Développer localement** : Tester rapidement vos applications dans un environnement similaire à la production
- **Expérimenter sans risque** : Essayer de nouvelles technologies sans impact sur des environnements partagés
- **Apprendre Kubernetes** : Comprendre les concepts en pratiquant dans un environnement réel
- **Automatiser vos workflows** : Mettre en place des pipelines CI/CD pour vos projets personnels
- **Simuler des architectures complexes** : Déployer des applications multi-services (microservices)

## Architecture Recommandée

Votre lab de développement comprendra les composants suivants :

### Composants de Base

1. **MicroK8s** : Le cluster Kubernetes léger
2. **Dashboard Kubernetes** : Interface graphique pour visualiser vos ressources
3. **Registry privé** : Stocker vos images Docker localement
4. **Stockage persistant** : Conserver les données de vos applications
5. **DNS interne** : Résolution des noms de services

### Composants de Réseau

6. **Ingress Controller** : Routage HTTP/HTTPS vers vos applications
7. **MetalLB** : Load balancer pour exposer vos services
8. **Cert-Manager** : Gestion automatique des certificats SSL

### Composants d'Observabilité

9. **Prometheus** : Collecte de métriques
10. **Grafana** : Visualisation des métriques
11. **Logs centralisés** : Agrégation des logs de vos applications

### Composants DevOps

12. **ArgoCD** : Déploiement GitOps (optionnel)
13. **Helm** : Gestionnaire de packages Kubernetes

## Configuration Système Recommandée

Pour un lab de développement confortable, voici les ressources minimales recommandées :

### Configuration Minimale
- **CPU** : 4 cœurs
- **RAM** : 8 GB
- **Stockage** : 50 GB d'espace libre
- **OS** : Ubuntu 20.04+ (ou autre distribution Linux)

### Configuration Idéale
- **CPU** : 8 cœurs ou plus
- **RAM** : 16 GB ou plus
- **Stockage** : 100 GB d'espace libre (SSD recommandé)
- **OS** : Ubuntu 22.04 LTS

## Mise en Place du Lab - Étape par Étape

### Étape 1 : Installation de MicroK8s

Si MicroK8s n'est pas encore installé, commencez par l'installer :

```bash
# Installation de MicroK8s
sudo snap install microk8s --classic

# Ajouter votre utilisateur au groupe microk8s
sudo usermod -a -G microk8s $USER
sudo chown -f -R $USER ~/.kube

# Recharger les groupes
newgrp microk8s

# Vérifier le statut
microk8s status --wait-ready
```

### Étape 2 : Configuration de l'Alias kubectl

Pour plus de simplicité, configurez un alias pour kubectl :

```bash
# Ajouter l'alias dans votre .bashrc ou .zshrc
echo "alias kubectl='microk8s kubectl'" >> ~/.bashrc
source ~/.bashrc

# Ou utiliser la commande microk8s pour configurer kubectl
microk8s kubectl config view --raw > ~/.kube/config
```

### Étape 3 : Activation des Addons Essentiels

Activez les addons de base pour votre lab :

```bash
# DNS pour la résolution de noms
microk8s enable dns

# Dashboard Kubernetes
microk8s enable dashboard

# Registry privé pour vos images
microk8s enable registry

# Stockage persistant
microk8s enable hostpath-storage

# Ingress controller pour le routage HTTP
microk8s enable ingress

# MetalLB pour les LoadBalancers
microk8s enable metallb:10.64.140.43-10.64.140.49
# Remplacez par une plage d'IPs disponible sur votre réseau local
```

**Note importante sur MetalLB** : La plage d'adresses IP doit être dans le même sous-réseau que votre machine mais ne pas être utilisée par votre routeur DHCP. Par exemple, si votre réseau est 192.168.1.0/24 et que votre DHCP distribue de 192.168.1.100 à 192.168.1.200, vous pouvez utiliser 192.168.1.240-192.168.1.250.

### Étape 4 : Installation de Cert-Manager

Pour gérer automatiquement vos certificats SSL :

```bash
# Activer cert-manager
microk8s enable cert-manager

# Attendre que cert-manager soit prêt
microk8s kubectl wait --for=condition=ready pod -l app.kubernetes.io/instance=cert-manager -n cert-manager --timeout=300s
```

### Étape 5 : Monitoring avec Prometheus et Grafana

```bash
# Activer Prometheus (inclut Grafana)
microk8s enable prometheus

# Vérifier que les pods sont démarrés
microk8s kubectl get pods -n monitoring
```

### Étape 6 : Accès au Dashboard Kubernetes

Le Dashboard Kubernetes est maintenant installé. Pour y accéder :

```bash
# Créer un token d'accès
microk8s kubectl create token default -n kube-system

# Démarrer le proxy (dans un terminal séparé)
microk8s kubectl proxy
```

Accédez ensuite au Dashboard via votre navigateur :
```
http://localhost:8001/api/v1/namespaces/kube-system/services/https:kubernetes-dashboard:/proxy/
```

Utilisez le token généré pour vous connecter.

### Étape 7 : Configuration du Registry Privé

Votre registry privé est accessible à l'adresse `localhost:32000`. Pour l'utiliser :

```bash
# Tagger une image pour votre registry
docker tag mon-app:latest localhost:32000/mon-app:latest

# Pousser l'image dans le registry
docker push localhost:32000/mon-app:latest
```

Dans vos manifestes Kubernetes, référencez les images ainsi :
```yaml
image: localhost:32000/mon-app:latest
```

## Organisation des Namespaces

Pour un lab bien organisé, créez des namespaces dédiés pour différents environnements :

```bash
# Namespace pour le développement
microk8s kubectl create namespace dev

# Namespace pour les tests
microk8s kubectl create namespace staging

# Namespace pour les expérimentations
microk8s kubectl create namespace sandbox

# Namespace pour vos outils (CI/CD, monitoring custom, etc.)
microk8s kubectl create namespace tools
```

### Définir un Namespace par Défaut

Pour ne pas avoir à spécifier `-n dev` à chaque commande :

```bash
# Définir dev comme namespace par défaut
microk8s kubectl config set-context --current --namespace=dev
```

## Exemples de Déploiements dans le Lab

### Déploiement d'une Application Web Simple

Créez un fichier `my-app.yaml` :

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-web-app
  namespace: dev
spec:
  replicas: 2
  selector:
    matchLabels:
      app: my-web-app
  template:
    metadata:
      labels:
        app: my-web-app
    spec:
      containers:
      - name: web
        image: nginx:latest
        ports:
        - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: my-web-app-service
  namespace: dev
spec:
  selector:
    app: my-web-app
  ports:
  - protocol: TCP
    port: 80
    targetPort: 80
  type: LoadBalancer
```

Déployez l'application :

```bash
microk8s kubectl apply -f my-app.yaml

# Vérifier le déploiement
microk8s kubectl get pods -n dev
microk8s kubectl get svc -n dev
```

### Configuration d'un Ingress avec SSL

Créez un fichier `ingress.yaml` :

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: my-app-ingress
  namespace: dev
  annotations:
    cert-manager.io/cluster-issuer: "letsencrypt-staging"
spec:
  ingressClassName: nginx
  tls:
  - hosts:
    - myapp.local.dev
    secretName: myapp-tls
  rules:
  - host: myapp.local.dev
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: my-web-app-service
            port:
              number: 80
```

**Note** : Pour un lab local, utilisez des noms de domaine `.local` ou `.dev` et ajoutez-les à votre fichier `/etc/hosts`.

## Stack de Base de Données

### PostgreSQL avec Stockage Persistant

Créez un fichier `postgres.yaml` :

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: postgres-pvc
  namespace: dev
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 5Gi
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: postgres
  namespace: dev
spec:
  replicas: 1
  selector:
    matchLabels:
      app: postgres
  template:
    metadata:
      labels:
        app: postgres
    spec:
      containers:
      - name: postgres
        image: postgres:15
        env:
        - name: POSTGRES_DB
          value: mydb
        - name: POSTGRES_USER
          value: devuser
        - name: POSTGRES_PASSWORD
          value: devpassword
        ports:
        - containerPort: 5432
        volumeMounts:
        - name: postgres-storage
          mountPath: /var/lib/postgresql/data
      volumes:
      - name: postgres-storage
        persistentVolumeClaim:
          claimName: postgres-pvc
---
apiVersion: v1
kind: Service
metadata:
  name: postgres-service
  namespace: dev
spec:
  selector:
    app: postgres
  ports:
  - protocol: TCP
    port: 5432
    targetPort: 5432
```

### Redis pour le Cache

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: redis
  namespace: dev
spec:
  replicas: 1
  selector:
    matchLabels:
      app: redis
  template:
    metadata:
      labels:
        app: redis
    spec:
      containers:
      - name: redis
        image: redis:7
        ports:
        - containerPort: 6379
---
apiVersion: v1
kind: Service
metadata:
  name: redis-service
  namespace: dev
spec:
  selector:
    app: redis
  ports:
  - protocol: TCP
    port: 6379
    targetPort: 6379
```

## Outils de Développement Utiles

### Port-Forwarding pour le Développement Local

Pour accéder à vos services directement depuis votre machine :

```bash
# Forward le port PostgreSQL
microk8s kubectl port-forward -n dev svc/postgres-service 5432:5432

# Forward le port Redis
microk8s kubectl port-forward -n dev svc/redis-service 6379:6379

# Forward Grafana
microk8s kubectl port-forward -n monitoring svc/prometheus-grafana 3000:80
```

Vous pouvez maintenant vous connecter à PostgreSQL depuis votre machine avec `localhost:5432`.

### Exec dans un Pod pour Débogage

```bash
# Se connecter à un conteneur
microk8s kubectl exec -it -n dev <nom-du-pod> -- /bin/bash

# Exécuter une commande ponctuelle
microk8s kubectl exec -n dev <nom-du-pod> -- ls -la /app
```

### Logs en Temps Réel

```bash
# Voir les logs d'un pod
microk8s kubectl logs -n dev <nom-du-pod>

# Suivre les logs en temps réel
microk8s kubectl logs -f -n dev <nom-du-pod>

# Logs de tous les pods d'un déploiement
microk8s kubectl logs -n dev -l app=my-web-app --tail=50
```

## Workflow de Développement Typique

Voici un workflow classique dans votre lab de développement :

### 1. Développement Local
- Vous développez votre application en local avec votre IDE préféré
- Vous testez localement avec Docker ou directement

### 2. Build de l'Image
```bash
# Construire l'image Docker
docker build -t mon-app:v1.0.0 .

# Tagger pour le registry local
docker tag mon-app:v1.0.0 localhost:32000/mon-app:v1.0.0

# Pousser dans le registry
docker push localhost:32000/mon-app:v1.0.0
```

### 3. Déploiement dans Kubernetes
```bash
# Appliquer les manifestes
microk8s kubectl apply -f k8s/

# Vérifier le déploiement
microk8s kubectl get pods -n dev
microk8s kubectl describe pod -n dev <nom-du-pod>
```

### 4. Tests et Débogage
```bash
# Tester l'application
curl http://myapp.local.dev

# Vérifier les logs
microk8s kubectl logs -f -n dev -l app=mon-app

# Se connecter au pod si besoin
microk8s kubectl exec -it -n dev <nom-du-pod> -- /bin/bash
```

### 5. Itération
- Modifier le code
- Rebuild l'image avec une nouvelle version
- Update le déploiement : `microk8s kubectl rollout restart deployment/mon-app -n dev`
- Observer le rolling update

## Accès à Grafana et Prometheus

### Prometheus

Prometheus est accessible via port-forward :

```bash
microk8s kubectl port-forward -n monitoring svc/prometheus-operated 9090:9090
```

Accédez à l'interface à `http://localhost:9090`

### Grafana

```bash
# Port-forward Grafana
microk8s kubectl port-forward -n monitoring svc/prometheus-grafana 3000:80
```

Accédez à Grafana à `http://localhost:3000`

**Identifiants par défaut** :
- Username : `admin`
- Password : récupérez-le avec :
```bash
microk8s kubectl get secret -n monitoring prometheus-grafana -o jsonpath="{.data.admin-password}" | base64 --decode
```

## Bonnes Pratiques pour Votre Lab

### 1. Gestion des Ressources

Définissez toujours des limites de ressources pour éviter qu'une application monopolise tout le cluster :

```yaml
resources:
  requests:
    memory: "64Mi"
    cpu: "250m"
  limits:
    memory: "128Mi"
    cpu: "500m"
```

### 2. Utilisation de ConfigMaps et Secrets

Externalisez vos configurations :

```bash
# Créer un ConfigMap à partir d'un fichier
microk8s kubectl create configmap app-config --from-file=config.json -n dev

# Créer un Secret
microk8s kubectl create secret generic db-credentials \
  --from-literal=username=devuser \
  --from-literal=password=devpassword \
  -n dev
```

### 3. Labels Cohérents

Utilisez des labels standardisés pour faciliter la gestion :

```yaml
labels:
  app: my-app
  version: v1.0.0
  environment: dev
  team: backend
```

### 4. Organisation des Fichiers

Structurez vos manifestes de manière logique :

```
my-project/
├── k8s/
│   ├── base/
│   │   ├── deployment.yaml
│   │   ├── service.yaml
│   │   └── configmap.yaml
│   ├── dev/
│   │   └── kustomization.yaml
│   └── staging/
│       └── kustomization.yaml
├── Dockerfile
└── src/
```

### 5. Documentation

Documentez vos déploiements avec des annotations :

```yaml
metadata:
  annotations:
    description: "API backend pour l'application mobile"
    owner: "equipe-backend"
    contact: "backend@example.com"
```

## Scripts Utiles pour Votre Lab

### Script de Démarrage Rapide

Créez un fichier `start-lab.sh` :

```bash
#!/bin/bash

echo "🚀 Démarrage du lab de développement..."

# Vérifier que MicroK8s est démarré
microk8s status --wait-ready

# Vérifier que tous les addons sont actifs
echo "✅ Vérification des addons..."
microk8s status

# Démarrer le dashboard
echo "📊 Démarrage du proxy pour le Dashboard..."
microk8s kubectl proxy &

# Afficher les services exposés
echo "🌐 Services exposés :"
microk8s kubectl get svc --all-namespaces -o wide

echo "✨ Lab prêt à l'emploi !"
echo "Dashboard : http://localhost:8001/api/v1/namespaces/kube-system/services/https:kubernetes-dashboard:/proxy/"
```

### Script de Nettoyage

Créez un fichier `cleanup-lab.sh` :

```bash
#!/bin/bash

echo "🧹 Nettoyage du lab..."

# Supprimer les pods en erreur
microk8s kubectl delete pods --field-selector status.phase=Failed --all-namespaces

# Nettoyer les images non utilisées
docker system prune -f

# Afficher l'utilisation des ressources
echo "📊 Utilisation des ressources :"
microk8s kubectl top nodes
microk8s kubectl top pods --all-namespaces

echo "✅ Nettoyage terminé !"
```

Rendez les scripts exécutables :
```bash
chmod +x start-lab.sh cleanup-lab.sh
```

## Dépannage Courant dans le Lab

### Problème : Les Pods Restent en Pending

**Cause** : Ressources insuffisantes

**Solution** :
```bash
# Vérifier les ressources disponibles
microk8s kubectl describe node

# Réduire les replicas ou les limites de ressources
microk8s kubectl scale deployment/mon-app --replicas=1 -n dev
```

### Problème : Impossible d'Accéder au Service

**Cause** : Configuration réseau ou DNS

**Solution** :
```bash
# Vérifier que CoreDNS fonctionne
microk8s kubectl get pods -n kube-system -l k8s-app=kube-dns

# Tester la résolution DNS depuis un pod
microk8s kubectl run -it --rm debug --image=busybox --restart=Never -- nslookup mon-service.dev.svc.cluster.local
```

### Problème : Le Registry Privé ne Fonctionne Pas

**Solution** :
```bash
# Vérifier que le registry est actif
microk8s kubectl get pods -n container-registry

# Redémarrer le registry
microk8s disable registry
microk8s enable registry
```

## Sauvegarde de Votre Lab

Pour sauvegarder votre configuration :

```bash
# Créer un répertoire de backup
mkdir -p ~/microk8s-backup

# Exporter tous les manifestes du namespace dev
microk8s kubectl get all -n dev -o yaml > ~/microk8s-backup/dev-resources.yaml

# Exporter les ConfigMaps et Secrets
microk8s kubectl get configmaps -n dev -o yaml > ~/microk8s-backup/dev-configmaps.yaml
microk8s kubectl get secrets -n dev -o yaml > ~/microk8s-backup/dev-secrets.yaml

# Sauvegarder les PVC
microk8s kubectl get pvc -n dev -o yaml > ~/microk8s-backup/dev-pvc.yaml
```

## Mise à Jour du Lab

Pour garder votre lab à jour :

```bash
# Mettre à jour MicroK8s
sudo snap refresh microk8s

# Vérifier la version
microk8s version

# Redémarrer MicroK8s si nécessaire
microk8s stop
microk8s start
```

## Optimisations pour les Performances

### Désactiver les Addons Non Utilisés

```bash
# Lister les addons actifs
microk8s status

# Désactiver un addon
microk8s disable <addon-name>
```

### Limiter les Ressources de Prometheus

Si Prometheus consomme trop de ressources, vous pouvez ajuster sa configuration :

```bash
# Éditer la configuration de Prometheus
microk8s kubectl edit configmap prometheus-server -n monitoring
```

Réduisez la rétention des données :
```yaml
storage:
  tsdb:
    retention.time: 7d  # Au lieu de 15d par défaut
```

## Conclusion

Vous disposez maintenant d'un lab de développement complet et professionnel avec MicroK8s. Cet environnement vous permet de :

✅ Développer et tester vos applications localement
✅ Apprendre Kubernetes dans des conditions réelles
✅ Expérimenter avec différentes technologies
✅ Préparer vos déploiements production
✅ Monitorer et observer vos applications
✅ Automatiser vos workflows de développement

**Prochaines étapes** : Explorez les autres chapitres de cette formation pour aller encore plus loin avec des cas d'usage spécifiques, l'automatisation CI/CD, et les concepts avancés de Kubernetes.

N'hésitez pas à personnaliser ce lab selon vos besoins spécifiques. L'avantage de MicroK8s est sa légèreté et sa flexibilité : vous pouvez activer ou désactiver des addons en fonction de vos projets du moment.

**Bon développement ! 🚀**

⏭️ [Environnement de test automatisé](/24-cas-dusage-pratiques/02-environnement-de-test-automatise.md)
