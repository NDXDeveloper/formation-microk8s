🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 24.3 Hébergement de Services Personnels

## Introduction

L'hébergement de services personnels avec MicroK8s vous permet de reprendre le contrôle de vos données et d'héberger vos propres applications chez vous. Au lieu de dépendre de services cloud tiers, vous créez votre propre cloud privé avec des services comme Nextcloud, Bitwarden, ou Home Assistant.

Dans ce chapitre, nous allons découvrir comment transformer votre cluster MicroK8s en un centre de services personnels sécurisé, accessible depuis n'importe où, tout en gardant vos données sous votre contrôle.

## Pourquoi Héberger Ses Propres Services ?

### Avantages de l'Auto-Hébergement

**Contrôle total de vos données** : Vos fichiers, photos, et informations sensibles restent chez vous, pas sur les serveurs d'une entreprise tierce.

**Confidentialité renforcée** : Vous n'êtes pas soumis aux politiques de confidentialité changeantes des services cloud.

**Économies à long terme** : Après l'investissement initial en matériel, les coûts sont minimes comparés aux abonnements mensuels.

**Apprentissage** : Gérer vos propres services vous fait progresser en administration système et en DevOps.

**Personnalisation** : Vous configurez vos services exactement comme vous le souhaitez, sans limitations imposées.

**Indépendance** : Vous n'êtes pas affecté si un service cloud ferme ou change ses conditions.

### Considérations Importantes

**Responsabilité** : Vous êtes responsable de la sécurité, des sauvegardes, et de la disponibilité de vos services.

**Temps requis** : L'installation initiale et la maintenance demandent du temps et des connaissances.

**Connectivité** : Vous avez besoin d'une connexion internet stable et idéalement d'une IP fixe ou d'un DNS dynamique.

**Coûts énergétiques** : Un serveur qui tourne 24/7 consomme de l'électricité.

**Risques** : Une mauvaise configuration peut exposer vos données ou votre réseau à des attaques.

## Configuration Système Recommandée

Pour héberger confortablement plusieurs services personnels :

### Configuration Minimale (1-3 services)
- **CPU** : 4 cœurs
- **RAM** : 8 GB
- **Stockage** : 100 GB (pour système et applications)
- **Stockage additionnel** : 500 GB - 1 TB (pour les données)
- **Réseau** : Connexion stable avec upload ≥ 10 Mbps

### Configuration Recommandée (3-10 services)
- **CPU** : 6-8 cœurs
- **RAM** : 16 GB
- **Stockage** : 256 GB SSD (système et applications)
- **Stockage additionnel** : 2-4 TB HDD (données)
- **Réseau** : Connexion fibre avec upload ≥ 50 Mbps

### Configuration Idéale (10+ services)
- **CPU** : 8+ cœurs
- **RAM** : 32 GB
- **Stockage** : 512 GB NVMe (système)
- **Stockage additionnel** : RAID pour redondance
- **Réseau** : Connexion fibre symétrique
- **UPS** : Onduleur pour protéger contre les coupures

## Architecture pour Services Personnels

### Composants Essentiels

1. **MicroK8s** : La base de votre infrastructure
2. **Ingress avec SSL** : Accès sécurisé à vos services
3. **Stockage persistant** : Conservation de vos données
4. **Sauvegarde automatique** : Protection contre la perte de données
5. **Monitoring** : Surveillance de l'état de vos services
6. **Authentification centralisée** : Gestion unifiée des accès (optionnel)

### Schéma Réseau

```
Internet
   ↓
[Routeur/Box Internet]
   ↓
[Firewall/NAT]
   ↓
[Serveur MicroK8s]
   ↓
[Services] ← Nextcloud, Bitwarden, etc.
```

## Préparation du Cluster

### Étape 1 : Installation des Addons Nécessaires

```bash
# DNS pour la résolution interne
microk8s enable dns

# Stockage persistant
microk8s enable hostpath-storage

# Ingress pour le routage HTTP/HTTPS
microk8s enable ingress

# Cert-Manager pour les certificats SSL
microk8s enable cert-manager

# MetalLB pour les LoadBalancers (si nécessaire)
microk8s enable metallb:192.168.1.240-192.168.1.250

# Registry privé pour vos images personnalisées (optionnel)
microk8s enable registry
```

### Étape 2 : Création du Namespace

Créez un namespace dédié pour vos services personnels :

```bash
# Créer le namespace
kubectl create namespace homelab

# Le marquer comme namespace par défaut
kubectl config set-context --current --namespace=homelab

# Ajouter des labels
kubectl label namespace homelab environment=production purpose=personal
```

### Étape 3 : Configuration du Stockage

Créez des répertoires dédiés pour les données :

```bash
# Créer des répertoires pour le stockage
sudo mkdir -p /mnt/data/homelab/{nextcloud,bitwarden,media,backups,documents}
sudo chown -R $USER:$USER /mnt/data/homelab
```

Créez une StorageClass personnalisée :

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: local-storage
provisioner: microk8s.io/hostpath
reclaimPolicy: Retain
volumeBindingMode: WaitForFirstConsumer
```

### Étape 4 : Configuration de l'Accès Externe

#### Obtenir un Nom de Domaine

Pour accéder à vos services depuis l'extérieur, vous avez plusieurs options :

**Option 1 : Domaine payant** (recommandé)
- Achetez un domaine (ex: monserveur.fr) chez un registrar (OVH, Gandi, Cloudflare)
- Configuration DNS complète possible
- Sous-domaines illimités

**Option 2 : DuckDNS (gratuit)**
- Service DNS dynamique gratuit
- Format : votrenomdechoix.duckdns.org
- Idéal pour débuter

**Option 3 : No-IP ou DynDNS (gratuit/freemium)**
- Autres alternatives DNS dynamiques
- Fonctionnalités limitées en version gratuite

#### Configuration du DNS

Si vous utilisez votre propre domaine, créez ces enregistrements DNS :

```
Type    Nom              Valeur              TTL
A       @                VOTRE_IP_PUBLIQUE   300
A       *                VOTRE_IP_PUBLIQUE   300
CNAME   nextcloud        monserveur.fr       300
CNAME   bitwarden        monserveur.fr       300
CNAME   photos           monserveur.fr       300
```

**Note** : Si votre IP publique change régulièrement, configurez un client DNS dynamique.

#### Configuration du Routeur (Redirection de Ports)

Configurez votre box/routeur pour rediriger les ports vers votre serveur :

```
Port Externe    Port Interne    Protocole    IP Destination
80             80              TCP          192.168.1.100 (votre serveur)
443            443             TCP          192.168.1.100 (votre serveur)
```

**Important** : N'ouvrez QUE les ports 80 et 443. Tous les autres services passeront par l'Ingress.

### Étape 5 : Configuration des Certificats SSL

Créez un ClusterIssuer pour Let's Encrypt :

```yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-prod
spec:
  acme:
    server: https://acme-v02.api.letsencrypt.org/directory
    email: votre-email@example.com
    privateKeySecretRef:
      name: letsencrypt-prod
    solvers:
    - http01:
        ingress:
          class: nginx
```

Appliquez la configuration :

```bash
kubectl apply -f letsencrypt-issuer.yaml
```

## Services Personnels Populaires

Voici une sélection de services que vous pouvez héberger, du plus simple au plus complexe.

### 1. Nextcloud - Votre Cloud Privé

Nextcloud est une alternative à Google Drive/Dropbox. Il permet de stocker, synchroniser et partager vos fichiers.

**Fonctionnalités** :
- Stockage et synchronisation de fichiers
- Calendrier et contacts
- Édition collaborative de documents
- Galerie photos
- Applications mobiles disponibles

#### Déploiement de Nextcloud

Créez un fichier `nextcloud.yaml` :

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: nextcloud-data
  namespace: homelab
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 100Gi
  storageClassName: local-storage
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: nextcloud-db
  namespace: homelab
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 10Gi
  storageClassName: local-storage
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nextcloud-db
  namespace: homelab
spec:
  replicas: 1
  selector:
    matchLabels:
      app: nextcloud-db
  template:
    metadata:
      labels:
        app: nextcloud-db
    spec:
      containers:
      - name: mariadb
        image: mariadb:10.11
        env:
        - name: MYSQL_ROOT_PASSWORD
          value: "changeme-root-password"
        - name: MYSQL_DATABASE
          value: nextcloud
        - name: MYSQL_USER
          value: nextcloud
        - name: MYSQL_PASSWORD
          value: "changeme-user-password"
        ports:
        - containerPort: 3306
        volumeMounts:
        - name: db-data
          mountPath: /var/lib/mysql
        resources:
          requests:
            memory: "512Mi"
            cpu: "250m"
          limits:
            memory: "1Gi"
            cpu: "500m"
      volumes:
      - name: db-data
        persistentVolumeClaim:
          claimName: nextcloud-db
---
apiVersion: v1
kind: Service
metadata:
  name: nextcloud-db
  namespace: homelab
spec:
  selector:
    app: nextcloud-db
  ports:
  - protocol: TCP
    port: 3306
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nextcloud
  namespace: homelab
spec:
  replicas: 1
  selector:
    matchLabels:
      app: nextcloud
  template:
    metadata:
      labels:
        app: nextcloud
    spec:
      containers:
      - name: nextcloud
        image: nextcloud:28
        env:
        - name: MYSQL_HOST
          value: nextcloud-db
        - name: MYSQL_DATABASE
          value: nextcloud
        - name: MYSQL_USER
          value: nextcloud
        - name: MYSQL_PASSWORD
          value: "changeme-user-password"
        - name: NEXTCLOUD_TRUSTED_DOMAINS
          value: "nextcloud.monserveur.fr"
        - name: OVERWRITEPROTOCOL
          value: "https"
        ports:
        - containerPort: 80
        volumeMounts:
        - name: nextcloud-data
          mountPath: /var/www/html
        resources:
          requests:
            memory: "512Mi"
            cpu: "250m"
          limits:
            memory: "2Gi"
            cpu: "1000m"
      volumes:
      - name: nextcloud-data
        persistentVolumeClaim:
          claimName: nextcloud-data
---
apiVersion: v1
kind: Service
metadata:
  name: nextcloud
  namespace: homelab
spec:
  selector:
    app: nextcloud
  ports:
  - protocol: TCP
    port: 80
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: nextcloud
  namespace: homelab
  annotations:
    cert-manager.io/cluster-issuer: "letsencrypt-prod"
    nginx.ingress.kubernetes.io/proxy-body-size: "10G"
    nginx.ingress.kubernetes.io/proxy-request-buffering: "off"
spec:
  ingressClassName: nginx
  tls:
  - hosts:
    - nextcloud.monserveur.fr
    secretName: nextcloud-tls
  rules:
  - host: nextcloud.monserveur.fr
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: nextcloud
            port:
              number: 80
```

Déployez Nextcloud :

```bash
# Remplacez les mots de passe et le domaine dans le fichier
kubectl apply -f nextcloud.yaml

# Attendez que tout soit prêt
kubectl wait --for=condition=ready pod -l app=nextcloud -n homelab --timeout=600s

# Accédez à https://nextcloud.monserveur.fr
```

**Première connexion** : Créez votre compte administrateur via l'interface web.

### 2. Bitwarden - Gestionnaire de Mots de Passe

Bitwarden (via Vaultwarden, implémentation légère) vous permet de gérer tous vos mots de passe de manière sécurisée.

**Fonctionnalités** :
- Stockage sécurisé des mots de passe
- Génération de mots de passe forts
- Extensions navigateur
- Applications mobiles
- Partage de mots de passe avec des proches

#### Déploiement de Vaultwarden

Créez un fichier `vaultwarden.yaml` :

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: vaultwarden-data
  namespace: homelab
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 5Gi
  storageClassName: local-storage
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: vaultwarden
  namespace: homelab
spec:
  replicas: 1
  selector:
    matchLabels:
      app: vaultwarden
  template:
    metadata:
      labels:
        app: vaultwarden
    spec:
      containers:
      - name: vaultwarden
        image: vaultwarden/server:latest
        env:
        - name: DOMAIN
          value: "https://bitwarden.monserveur.fr"
        - name: SIGNUPS_ALLOWED
          value: "true"  # Mettez à "false" après avoir créé vos comptes
        - name: INVITATIONS_ALLOWED
          value: "true"
        - name: WEBSOCKET_ENABLED
          value: "true"
        ports:
        - containerPort: 80
        - containerPort: 3012
        volumeMounts:
        - name: data
          mountPath: /data
        resources:
          requests:
            memory: "128Mi"
            cpu: "100m"
          limits:
            memory: "256Mi"
            cpu: "200m"
      volumes:
      - name: data
        persistentVolumeClaim:
          claimName: vaultwarden-data
---
apiVersion: v1
kind: Service
metadata:
  name: vaultwarden
  namespace: homelab
spec:
  selector:
    app: vaultwarden
  ports:
  - name: http
    protocol: TCP
    port: 80
  - name: websocket
    protocol: TCP
    port: 3012
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: vaultwarden
  namespace: homelab
  annotations:
    cert-manager.io/cluster-issuer: "letsencrypt-prod"
spec:
  ingressClassName: nginx
  tls:
  - hosts:
    - bitwarden.monserveur.fr
    secretName: vaultwarden-tls
  rules:
  - host: bitwarden.monserveur.fr
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: vaultwarden
            port:
              number: 80
```

Déployez Vaultwarden :

```bash
kubectl apply -f vaultwarden.yaml

# Vérifier le déploiement
kubectl get pods -n homelab -l app=vaultwarden

# Accéder à https://bitwarden.monserveur.fr
```

**Important** : Après avoir créé vos comptes, mettez `SIGNUPS_ALLOWED` à `false` pour sécuriser l'instance.

### 3. Jellyfin - Serveur Multimédia

Jellyfin est une alternative open-source à Plex pour organiser et streamer vos films, séries et musique.

**Fonctionnalités** :
- Streaming de vidéos et musique
- Transcodage à la volée
- Applications pour tous les appareils
- Gestion de bibliothèque multimédia
- Plusieurs utilisateurs avec profils

#### Déploiement de Jellyfin

Créez un fichier `jellyfin.yaml` :

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: jellyfin-config
  namespace: homelab
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 10Gi
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: jellyfin-cache
  namespace: homelab
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 20Gi
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: jellyfin
  namespace: homelab
spec:
  replicas: 1
  selector:
    matchLabels:
      app: jellyfin
  template:
    metadata:
      labels:
        app: jellyfin
    spec:
      containers:
      - name: jellyfin
        image: jellyfin/jellyfin:latest
        env:
        - name: TZ
          value: "Europe/Paris"
        ports:
        - containerPort: 8096
        volumeMounts:
        - name: config
          mountPath: /config
        - name: cache
          mountPath: /cache
        - name: media
          mountPath: /media
          readOnly: true
        resources:
          requests:
            memory: "1Gi"
            cpu: "500m"
          limits:
            memory: "4Gi"
            cpu: "2000m"
      volumes:
      - name: config
        persistentVolumeClaim:
          claimName: jellyfin-config
      - name: cache
        persistentVolumeClaim:
          claimName: jellyfin-cache
      - name: media
        hostPath:
          path: /mnt/data/homelab/media
          type: Directory
---
apiVersion: v1
kind: Service
metadata:
  name: jellyfin
  namespace: homelab
spec:
  selector:
    app: jellyfin
  ports:
  - protocol: TCP
    port: 8096
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: jellyfin
  namespace: homelab
  annotations:
    cert-manager.io/cluster-issuer: "letsencrypt-prod"
    nginx.ingress.kubernetes.io/proxy-body-size: "0"
spec:
  ingressClassName: nginx
  tls:
  - hosts:
    - media.monserveur.fr
    secretName: jellyfin-tls
  rules:
  - host: media.monserveur.fr
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: jellyfin
            port:
              number: 8096
```

### 4. Home Assistant - Domotique

Home Assistant vous permet de contrôler tous vos appareils connectés depuis une interface unique.

**Fonctionnalités** :
- Contrôle des lumières, thermostats, caméras
- Automatisations sophistiquées
- Support de centaines d'intégrations
- Application mobile
- Respect de la vie privée

#### Déploiement de Home Assistant

Créez un fichier `homeassistant.yaml` :

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: homeassistant-config
  namespace: homelab
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
  name: homeassistant
  namespace: homelab
spec:
  replicas: 1
  selector:
    matchLabels:
      app: homeassistant
  template:
    metadata:
      labels:
        app: homeassistant
    spec:
      hostNetwork: true  # Nécessaire pour découvrir les appareils locaux
      containers:
      - name: homeassistant
        image: homeassistant/home-assistant:latest
        env:
        - name: TZ
          value: "Europe/Paris"
        ports:
        - containerPort: 8123
        volumeMounts:
        - name: config
          mountPath: /config
        resources:
          requests:
            memory: "512Mi"
            cpu: "250m"
          limits:
            memory: "2Gi"
            cpu: "1000m"
      volumes:
      - name: config
        persistentVolumeClaim:
          claimName: homeassistant-config
---
apiVersion: v1
kind: Service
metadata:
  name: homeassistant
  namespace: homelab
spec:
  selector:
    app: homeassistant
  ports:
  - protocol: TCP
    port: 8123
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: homeassistant
  namespace: homelab
  annotations:
    cert-manager.io/cluster-issuer: "letsencrypt-prod"
    nginx.ingress.kubernetes.io/proxy-body-size: "0"
    nginx.ingress.kubernetes.io/websocket-services: "homeassistant"
spec:
  ingressClassName: nginx
  tls:
  - hosts:
    - home.monserveur.fr
    secretName: homeassistant-tls
  rules:
  - host: home.monserveur.fr
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: homeassistant
            port:
              number: 8123
```

### 5. Gitea - Serveur Git Privé

Gitea est une alternative légère à GitHub/GitLab pour héberger vos dépôts Git.

**Fonctionnalités** :
- Hébergement de dépôts Git
- Interface web élégante
- Gestion d'utilisateurs et d'organisations
- Wiki et issues
- Webhooks et CI/CD

#### Déploiement de Gitea

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: gitea-data
  namespace: homelab
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 50Gi
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: gitea-db
  namespace: homelab
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 10Gi
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: gitea-db
  namespace: homelab
spec:
  replicas: 1
  selector:
    matchLabels:
      app: gitea-db
  template:
    metadata:
      labels:
        app: gitea-db
    spec:
      containers:
      - name: postgres
        image: postgres:15
        env:
        - name: POSTGRES_DB
          value: gitea
        - name: POSTGRES_USER
          value: gitea
        - name: POSTGRES_PASSWORD
          value: "changeme-password"
        ports:
        - containerPort: 5432
        volumeMounts:
        - name: db-data
          mountPath: /var/lib/postgresql/data
      volumes:
      - name: db-data
        persistentVolumeClaim:
          claimName: gitea-db
---
apiVersion: v1
kind: Service
metadata:
  name: gitea-db
  namespace: homelab
spec:
  selector:
    app: gitea-db
  ports:
  - protocol: TCP
    port: 5432
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: gitea
  namespace: homelab
spec:
  replicas: 1
  selector:
    matchLabels:
      app: gitea
  template:
    metadata:
      labels:
        app: gitea
    spec:
      containers:
      - name: gitea
        image: gitea/gitea:latest
        env:
        - name: USER_UID
          value: "1000"
        - name: USER_GID
          value: "1000"
        - name: GITEA__database__DB_TYPE
          value: postgres
        - name: GITEA__database__HOST
          value: gitea-db:5432
        - name: GITEA__database__NAME
          value: gitea
        - name: GITEA__database__USER
          value: gitea
        - name: GITEA__database__PASSWD
          value: "changeme-password"
        ports:
        - containerPort: 3000
        - containerPort: 22
        volumeMounts:
        - name: data
          mountPath: /data
        resources:
          requests:
            memory: "256Mi"
            cpu: "200m"
          limits:
            memory: "1Gi"
            cpu: "500m"
      volumes:
      - name: data
        persistentVolumeClaim:
          claimName: gitea-data
---
apiVersion: v1
kind: Service
metadata:
  name: gitea
  namespace: homelab
spec:
  selector:
    app: gitea
  ports:
  - name: http
    protocol: TCP
    port: 3000
  - name: ssh
    protocol: TCP
    port: 22
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: gitea
  namespace: homelab
  annotations:
    cert-manager.io/cluster-issuer: "letsencrypt-prod"
spec:
  ingressClassName: nginx
  tls:
  - hosts:
    - git.monserveur.fr
    secretName: gitea-tls
  rules:
  - host: git.monserveur.fr
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: gitea
            port:
              number: 3000
```

### 6. PhotoPrism - Galerie Photos Intelligente

PhotoPrism organise et indexe vos photos avec reconnaissance faciale et géolocalisation.

**Fonctionnalités** :
- Organisation automatique des photos
- Reconnaissance faciale
- Géolocalisation sur carte
- Recherche puissante
- Support RAW

#### Déploiement de PhotoPrism

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: photoprism-storage
  namespace: homelab
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 100Gi
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: photoprism-db
  namespace: homelab
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 10Gi
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: photoprism-db
  namespace: homelab
spec:
  replicas: 1
  selector:
    matchLabels:
      app: photoprism-db
  template:
    metadata:
      labels:
        app: photoprism-db
    spec:
      containers:
      - name: mariadb
        image: mariadb:10.11
        env:
        - name: MYSQL_ROOT_PASSWORD
          value: "changeme-root"
        - name: MYSQL_DATABASE
          value: photoprism
        - name: MYSQL_USER
          value: photoprism
        - name: MYSQL_PASSWORD
          value: "changeme-user"
        volumeMounts:
        - name: db-data
          mountPath: /var/lib/mysql
      volumes:
      - name: db-data
        persistentVolumeClaim:
          claimName: photoprism-db
---
apiVersion: v1
kind: Service
metadata:
  name: photoprism-db
  namespace: homelab
spec:
  selector:
    app: photoprism-db
  ports:
  - protocol: TCP
    port: 3306
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: photoprism
  namespace: homelab
spec:
  replicas: 1
  selector:
    matchLabels:
      app: photoprism
  template:
    metadata:
      labels:
        app: photoprism
    spec:
      containers:
      - name: photoprism
        image: photoprism/photoprism:latest
        env:
        - name: PHOTOPRISM_ADMIN_PASSWORD
          value: "changeme-admin"
        - name: PHOTOPRISM_DATABASE_DRIVER
          value: mysql
        - name: PHOTOPRISM_DATABASE_SERVER
          value: "photoprism-db:3306"
        - name: PHOTOPRISM_DATABASE_NAME
          value: photoprism
        - name: PHOTOPRISM_DATABASE_USER
          value: photoprism
        - name: PHOTOPRISM_DATABASE_PASSWORD
          value: "changeme-user"
        - name: PHOTOPRISM_SITE_URL
          value: "https://photos.monserveur.fr"
        ports:
        - containerPort: 2342
        volumeMounts:
        - name: storage
          mountPath: /photoprism/storage
        - name: originals
          mountPath: /photoprism/originals
        resources:
          requests:
            memory: "1Gi"
            cpu: "500m"
          limits:
            memory: "4Gi"
            cpu: "2000m"
      volumes:
      - name: storage
        persistentVolumeClaim:
          claimName: photoprism-storage
      - name: originals
        hostPath:
          path: /mnt/data/homelab/photos
---
apiVersion: v1
kind: Service
metadata:
  name: photoprism
  namespace: homelab
spec:
  selector:
    app: photoprism
  ports:
  - protocol: TCP
    port: 2342
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: photoprism
  namespace: homelab
  annotations:
    cert-manager.io/cluster-issuer: "letsencrypt-prod"
    nginx.ingress.kubernetes.io/proxy-body-size: "0"
spec:
  ingressClassName: nginx
  tls:
  - hosts:
    - photos.monserveur.fr
    secretName: photoprism-tls
  rules:
  - host: photos.monserveur.fr
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: photoprism
            port:
              number: 2342
```

## Sécurité de Votre Homelab

La sécurité est cruciale lorsque vous exposez des services sur internet.

### 1. Authentification Forte

**Utilisez des mots de passe forts** : Au minimum 16 caractères avec majuscules, minuscules, chiffres et symboles.

**Authentification à deux facteurs (2FA)** : Activez-la partout où c'est possible.

**Gestionnaire de mots de passe** : Utilisez Bitwarden pour générer et stocker des mots de passe uniques.

### 2. Authentification Centralisée avec Authelia (Optionnel)

Authelia ajoute une couche d'authentification supplémentaire devant tous vos services.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: authelia
  namespace: homelab
spec:
  replicas: 1
  selector:
    matchLabels:
      app: authelia
  template:
    metadata:
      labels:
        app: authelia
    spec:
      containers:
      - name: authelia
        image: authelia/authelia:latest
        ports:
        - containerPort: 9091
        volumeMounts:
        - name: config
          mountPath: /config
      volumes:
      - name: config
        configMap:
          name: authelia-config
```

### 3. Network Policies

Limitez la communication entre les pods :

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: deny-all-ingress
  namespace: homelab
spec:
  podSelector: {}
  policyTypes:
  - Ingress
---
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-ingress-to-web
  namespace: homelab
spec:
  podSelector:
    matchLabels:
      type: web-service
  policyTypes:
  - Ingress
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          name: ingress-nginx
    ports:
    - protocol: TCP
      port: 80
```

### 4. Fail2Ban pour Bloquer les Attaques

Installez Fail2Ban sur votre serveur hôte pour bloquer les IPs malveillantes :

```bash
sudo apt install fail2ban

# Configuration pour NGINX
sudo cat > /etc/fail2ban/jail.local <<EOF
[nginx-limit-req]
enabled = true
filter = nginx-limit-req
logpath = /var/log/nginx/error.log
maxretry = 5
bantime = 3600
EOF

sudo systemctl restart fail2ban
```

### 5. Mise à Jour Régulière

Gardez vos services à jour :

```bash
# Script de mise à jour automatique
kubectl set image deployment/nextcloud nextcloud=nextcloud:latest -n homelab
kubectl rollout status deployment/nextcloud -n homelab
```

### 6. Audit de Sécurité

Scannez régulièrement vos images :

```bash
# Avec Trivy
trivy image nextcloud:latest

# Scanner toutes les images du namespace
for image in $(kubectl get pods -n homelab -o jsonpath='{range .items[*]}{.spec.containers[*].image}{"\n"}{end}' | sort -u); do
  echo "Scanning $image..."
  trivy image $image
done
```

## Sauvegarde Automatique

La sauvegarde est essentielle pour protéger vos données personnelles.

### Stratégie de Sauvegarde 3-2-1

- **3** copies de vos données
- Sur **2** supports différents
- Avec **1** copie hors site

### Script de Sauvegarde Automatique

Créez un CronJob pour sauvegarder régulièrement :

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: backup-nextcloud
  namespace: homelab
spec:
  schedule: "0 3 * * *"  # Tous les jours à 3h du matin
  jobTemplate:
    spec:
      template:
        spec:
          restartPolicy: OnFailure
          containers:
          - name: backup
            image: alpine:latest
            command:
            - sh
            - -c
            - |
              apk add --no-cache rsync

              # Sauvegarde des données Nextcloud
              rsync -av /data/nextcloud/ /backups/nextcloud-$(date +%Y%m%d)/

              # Garder seulement les 7 derniers backups
              cd /backups
              ls -t | tail -n +8 | xargs rm -rf

              echo "Backup completed at $(date)"
            volumeMounts:
            - name: data
              mountPath: /data
              readOnly: true
            - name: backups
              mountPath: /backups
          volumes:
          - name: data
            persistentVolumeClaim:
              claimName: nextcloud-data
          - name: backups
            hostPath:
              path: /mnt/data/homelab/backups
```

### Sauvegarde des Bases de Données

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: backup-databases
  namespace: homelab
spec:
  schedule: "0 2 * * *"
  jobTemplate:
    spec:
      template:
        spec:
          restartPolicy: OnFailure
          containers:
          - name: backup
            image: postgres:15
            command:
            - sh
            - -c
            - |
              # Backup PostgreSQL
              pg_dump -h postgres-service -U gitea gitea > /backups/gitea-$(date +%Y%m%d).sql

              # Compression
              gzip /backups/gitea-$(date +%Y%m%d).sql

              # Rotation
              find /backups -name "gitea-*.sql.gz" -mtime +30 -delete
            env:
            - name: PGPASSWORD
              value: "changeme-password"
            volumeMounts:
            - name: backups
              mountPath: /backups
          volumes:
          - name: backups
            hostPath:
              path: /mnt/data/homelab/backups/databases
```

### Sauvegarde Externe

Pour une protection optimale, envoyez vos sauvegardes vers un stockage externe :

```bash
# Vers un serveur distant via rsync
rsync -avz --delete /mnt/data/homelab/backups/ user@backup-server:/backups/homelab/

# Vers un cloud (Backblaze B2, AWS S3, etc.)
# Avec rclone
rclone sync /mnt/data/homelab/backups/ b2:my-bucket/homelab-backups/
```

## Monitoring de Vos Services

### Dashboard Personnalisé avec Homepage

Homepage est un dashboard élégant pour accéder à tous vos services :

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: homepage
  namespace: homelab
spec:
  replicas: 1
  selector:
    matchLabels:
      app: homepage
  template:
    metadata:
      labels:
        app: homepage
    spec:
      containers:
      - name: homepage
        image: ghcr.io/gethomepage/homepage:latest
        ports:
        - containerPort: 3000
        volumeMounts:
        - name: config
          mountPath: /app/config
      volumes:
      - name: config
        configMap:
          name: homepage-config
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: homepage-config
  namespace: homelab
data:
  services.yaml: |
    - Stockage:
        - Nextcloud:
            href: https://nextcloud.monserveur.fr
            description: Cloud personnel
            icon: nextcloud.png

    - Sécurité:
        - Bitwarden:
            href: https://bitwarden.monserveur.fr
            description: Gestionnaire de mots de passe
            icon: bitwarden.png

    - Média:
        - Jellyfin:
            href: https://media.monserveur.fr
            description: Serveur multimédia
            icon: jellyfin.png
        - PhotoPrism:
            href: https://photos.monserveur.fr
            description: Galerie photos
            icon: photoprism.png

    - Développement:
        - Gitea:
            href: https://git.monserveur.fr
            description: Serveur Git
            icon: gitea.png

    - Domotique:
        - Home Assistant:
            href: https://home.monserveur.fr
            description: Contrôle maison
            icon: home-assistant.png
---
apiVersion: v1
kind: Service
metadata:
  name: homepage
  namespace: homelab
spec:
  selector:
    app: homepage
  ports:
  - protocol: TCP
    port: 3000
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: homepage
  namespace: homelab
  annotations:
    cert-manager.io/cluster-issuer: "letsencrypt-prod"
spec:
  ingressClassName: nginx
  tls:
  - hosts:
    - home.monserveur.fr
    secretName: homepage-tls
  rules:
  - host: home.monserveur.fr
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: homepage
            port:
              number: 3000
```

### Uptime Kuma - Monitoring de Disponibilité

Surveillez la disponibilité de tous vos services :

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: uptime-kuma
  namespace: homelab
spec:
  replicas: 1
  selector:
    matchLabels:
      app: uptime-kuma
  template:
    metadata:
      labels:
        app: uptime-kuma
    spec:
      containers:
      - name: uptime-kuma
        image: louislam/uptime-kuma:latest
        ports:
        - containerPort: 3001
        volumeMounts:
        - name: data
          mountPath: /app/data
      volumes:
      - name: data
        persistentVolumeClaim:
          claimName: uptime-kuma-data
```

## Optimisation des Performances

### 1. Caching avec Redis

Ajoutez un cache Redis pour accélérer vos applications :

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: redis
  namespace: homelab
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
        image: redis:7-alpine
        ports:
        - containerPort: 6379
        resources:
          requests:
            memory: "128Mi"
            cpu: "100m"
          limits:
            memory: "256Mi"
            cpu: "200m"
---
apiVersion: v1
kind: Service
metadata:
  name: redis
  namespace: homelab
spec:
  selector:
    app: redis
  ports:
  - protocol: TCP
    port: 6379
```

Configurez Nextcloud pour utiliser Redis :

```bash
kubectl exec -it -n homelab <nextcloud-pod> -- \
  php occ config:system:set redis host --value="redis"
kubectl exec -it -n homelab <nextcloud-pod> -- \
  php occ config:system:set redis port --value="6379"
kubectl exec -it -n homelab <nextcloud-pod> -- \
  php occ config:system:set memcache.locking --value="\OC\Memcache\Redis"
```

### 2. Tuning NGINX Ingress

Optimisez l'Ingress pour de meilleures performances :

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: nginx-configuration
  namespace: ingress-nginx
data:
  proxy-body-size: "0"
  proxy-buffering: "off"
  proxy-request-buffering: "off"
  client-max-body-size: "10G"
  proxy-connect-timeout: "600"
  proxy-send-timeout: "600"
  proxy-read-timeout: "600"
```

### 3. SSD pour les Données Fréquemment Accédées

Utilisez un SSD pour les bases de données et les caches :

```bash
# Déplacer la base de données vers un SSD
sudo mkdir -p /mnt/ssd/databases
sudo chown -R $USER:$USER /mnt/ssd/databases

# Mettre à jour les PV pour pointer vers le SSD
```

## Gestion de la Bande Passante

Si votre connexion est limitée, optimisez l'utilisation de la bande passante.

### Limitation de Bande Passante par Service

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: bandwidth-limited-service
  annotations:
    kubernetes.io/ingress-bandwidth: "10M"
    kubernetes.io/egress-bandwidth: "10M"
spec:
  containers:
  - name: app
    image: myapp:latest
```

### Compression NGINX

Activez la compression dans l'Ingress :

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: my-service
  annotations:
    nginx.ingress.kubernetes.io/enable-compression: "true"
    nginx.ingress.kubernetes.io/compression-types: "text/plain text/css application/json application/javascript text/xml application/xml"
```

## Maintenance et Bonnes Pratiques

### Script de Maintenance Hebdomadaire

```bash
#!/bin/bash

echo "🔧 Starting weekly maintenance..."

# 1. Nettoyage des images Docker
echo "Cleaning Docker images..."
docker system prune -af

# 2. Vérification de l'espace disque
echo "Checking disk space..."
df -h

# 3. Vérification des pods
echo "Checking pod status..."
kubectl get pods --all-namespaces | grep -v Running

# 4. Vérification des certificats
echo "Checking certificate expiration..."
kubectl get certificates --all-namespaces

# 5. Mise à jour des images
echo "Updating container images..."
kubectl set image deployment/nextcloud nextcloud=nextcloud:latest -n homelab
kubectl set image deployment/vaultwarden vaultwarden=vaultwarden/server:latest -n homelab

# 6. Redémarrage des services si nécessaire
echo "Checking for pods requiring restart..."
kubectl rollout restart deployment/nextcloud -n homelab

echo "✅ Maintenance completed!"
```

### Checklist Mensuelle

**Sécurité** :
- [ ] Vérifier les mises à jour de sécurité disponibles
- [ ] Scanner les vulnérabilités des images
- [ ] Vérifier les logs pour activités suspectes
- [ ] Tester les sauvegardes

**Performance** :
- [ ] Vérifier l'utilisation des ressources
- [ ] Optimiser les requêtes lentes
- [ ] Nettoyer les anciennes données

**Mises à jour** :
- [ ] Mettre à jour MicroK8s
- [ ] Mettre à jour les applications
- [ ] Mettre à jour le système d'exploitation

## Dépannage Courant

### Service Inaccessible de l'Extérieur

**Vérifications** :

```bash
# 1. Vérifier que le pod fonctionne
kubectl get pods -n homelab

# 2. Vérifier le service
kubectl get svc -n homelab

# 3. Vérifier l'Ingress
kubectl get ingress -n homelab
kubectl describe ingress <nom> -n homelab

# 4. Vérifier le certificat SSL
kubectl get certificate -n homelab
kubectl describe certificate <nom> -n homelab

# 5. Tester depuis l'intérieur du cluster
kubectl run -it --rm debug --image=curlimages/curl --restart=Never -- \
  curl -v http://nextcloud.homelab.svc.cluster.local

# 6. Vérifier les règles firewall
sudo iptables -L -n
```

### Problèmes de Performance

```bash
# Vérifier l'utilisation des ressources
kubectl top nodes
kubectl top pods -n homelab

# Identifier les goulots d'étranglement
kubectl describe pod <nom> -n homelab

# Augmenter les ressources si nécessaire
kubectl set resources deployment/<nom> \
  --limits=cpu=2000m,memory=4Gi \
  --requests=cpu=1000m,memory=2Gi \
  -n homelab
```

### Espace Disque Plein

```bash
# Identifier les gros consommateurs
du -sh /mnt/data/homelab/*

# Nettoyer les anciennes sauvegardes
find /mnt/data/homelab/backups -type f -mtime +30 -delete

# Nettoyer Docker
docker system prune -af --volumes

# Nettoyer les logs Kubernetes
sudo journalctl --vacuum-time=7d
```

## Coûts et Économies

### Comparaison des Coûts

**Services Cloud (par an)** :
- Dropbox Plus (2TB) : ~120€
- Google One (2TB) : ~100€
- LastPass Premium : ~40€
- Plex Pass : ~40€
- **Total** : ~300€/an

**Auto-hébergement (coûts uniques + annuels)** :
- Mini PC ou NUC : 300-600€ (unique)
- Disque dur 4TB : 100€ (unique)
- Électricité (~50W 24/7) : ~80€/an
- Domaine : ~15€/an
- **Total première année** : ~500-800€
- **Années suivantes** : ~95€/an

**Économies après 2 ans** : ~500€ et vous possédez votre matériel !

## Évolution de Votre Homelab

### Ajouter de Nouveaux Services

Le processus est toujours le même :

1. Chercher l'image Docker officielle
2. Créer les manifestes YAML (Deployment, Service, Ingress)
3. Configurer le stockage persistant
4. Déployer et tester
5. Configurer les sauvegardes

### Services Supplémentaires Populaires

**Wikijs** : Wiki personnel pour documenter vos projets

**FreshRSS** : Agrégateur de flux RSS

**Paperless-ngx** : Gestion de documents numérisés

**Firefly III** : Gestion de finances personnelles

**Mealie** : Gestion de recettes de cuisine

**Bookstack** : Documentation et wiki

**Tandoor** : Planification de repas

**Stirling-PDF** : Manipulation de PDFs

## Conclusion

Vous disposez maintenant d'un guide complet pour héberger vos propres services personnels avec MicroK8s. Cet environnement vous permet de :

✅ Reprendre le contrôle de vos données personnelles
✅ Économiser sur les abonnements cloud
✅ Personnaliser vos services selon vos besoins
✅ Apprendre les technologies cloud et Kubernetes
✅ Créer un cloud privé sécurisé et fiable

**Points clés à retenir** :

- **Sécurisez d'abord** : SSL, mots de passe forts, mises à jour régulières
- **Sauvegardez tout** : Stratégie 3-2-1, automatisation des backups
- **Commencez petit** : Déployez un service à la fois
- **Documentez** : Prenez des notes sur votre configuration
- **Surveillez** : Monitoring et alertes pour détecter les problèmes
- **Optimisez progressivement** : Améliorez au fil du temps

**Prochaines étapes** :

1. Choisissez 1-2 services prioritaires
2. Déployez-les en suivant ce guide
3. Testez pendant quelques semaines
4. Ajoutez progressivement d'autres services
5. Affinez la sécurité et les sauvegardes

**Ressources complémentaires** :
- r/selfhosted sur Reddit : communauté active
- awesome-selfhosted sur GitHub : liste de services
- Documentation officielle de chaque service

**Bon hébergement ! 🏠**

⏭️ [Learning playground Kubernetes](/24-cas-dusage-pratiques/04-learning-playground-kubernetes.md)
