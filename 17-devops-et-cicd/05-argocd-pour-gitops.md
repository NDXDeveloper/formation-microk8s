🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 17.5 ArgoCD pour GitOps

## Introduction

Vous avez mis en place Git pour vos manifestes, un registry pour vos images, et des pipelines CI/CD pour automatiser les builds. Il ne manque plus qu'une pièce au puzzle : **ArgoCD**, l'outil qui va synchroniser automatiquement vos manifestes Git avec votre cluster Kubernetes.

ArgoCD est le **couteau suisse du GitOps** : il surveille votre repository Git en permanence et s'assure que l'état de votre cluster Kubernetes correspond toujours exactement à ce qui est défini dans Git. C'est la philosophie GitOps poussée à son paroxysme : **Git devient la source unique de vérité**, et ArgoCD le gardien de cette vérité.

Dans ce chapitre, nous allons découvrir comment installer et configurer ArgoCD sur MicroK8s, créer vos premières applications, et mettre en place un workflow GitOps complet et automatisé.

---

## Qu'est-ce qu'ArgoCD ?

### Définition

**ArgoCD** est un outil de **livraison continue déclarative** pour Kubernetes qui implémente les principes GitOps. C'est un contrôleur Kubernetes qui surveille en continu vos applications et compare leur état actuel avec l'état désiré défini dans Git.

**Créé par Intuit** en 2018, ArgoCD est aujourd'hui un projet gradué de la **CNCF** (Cloud Native Computing Foundation), ce qui témoigne de sa maturité et de son adoption massive.

### Les principes d'ArgoCD

ArgoCD s'appuie sur les quatre principes fondamentaux de GitOps :

1. **Déclaratif** : Tout est défini de manière déclarative dans Git
2. **Versionné** : Git garde l'historique complet
3. **Automatique** : ArgoCD synchronise automatiquement
4. **Réconciliation continue** : ArgoCD surveille et corrige les dérives

### Comment ça fonctionne ?

Le fonctionnement d'ArgoCD est simple et élégant :

```
┌─────────────────────────────────────────────────────────────┐
│                         Git Repository                      │
│  ┌───────────────────────────────────────────────────────┐  │
│  │  YAML manifests (état désiré)                         │  │
│  │  - deployments.yaml                                   │  │
│  │  - services.yaml                                      │  │
│  │  - ingress.yaml                                       │  │
│  └───────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────┘
                           ↓
                    (ArgoCD Poll)
                           ↓
┌─────────────────────────────────────────────────────────────┐
│                      ArgoCD Server                          │
│  ┌───────────────────────────────────────────────────────┐  │
│  │  Compare:                                             │  │
│  │  État désiré (Git) ≟ État actuel (K8s)                │  │
│  │                                                       │  │
│  │  Si différent → Synchroniser                          │  │
│  └───────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────┘
                           ↓
                      (kubectl apply)
                           ↓
┌─────────────────────────────────────────────────────────────┐
│                    Kubernetes Cluster                       │
│  ┌───────────────────────────────────────────────────────┐  │
│  │  Applications déployées (état actuel)                 │  │
│  │  - Pods, Services, Ingress, etc.                      │  │
│  └───────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────┘
```

**Le cycle** :
1. Vous modifiez un manifeste dans Git
2. ArgoCD détecte le changement (toutes les 3 minutes par défaut)
3. ArgoCD compare l'état désiré (Git) avec l'état actuel (Kubernetes)
4. ArgoCD applique automatiquement les changements
5. Votre cluster est maintenant synchronisé avec Git

### Avantages d'ArgoCD

**Pour les développeurs** :
- ✅ Interface web intuitive pour visualiser les déploiements
- ✅ Pas besoin d'accès kubectl direct au cluster
- ✅ Historique complet des déploiements via Git
- ✅ Rollback facile en un clic

**Pour les opérationnels** :
- ✅ Détection et correction automatique des dérives (drift detection)
- ✅ Audit complet de tous les changements
- ✅ Multi-cluster depuis une seule interface
- ✅ Gestion centralisée des configurations

**Pour la sécurité** :
- ✅ Git comme source unique de vérité
- ✅ RBAC granulaire
- ✅ Pas de credentials de cluster externes
- ✅ Validation avant déploiement

---

## Installation d'ArgoCD sur MicroK8s

### Méthode 1 : Installation standard (recommandée)

L'installation d'ArgoCD est simple et se fait via des manifestes officiels.

#### Étape 1 : Créer le namespace

```bash
kubectl create namespace argocd
```

#### Étape 2 : Installer ArgoCD

```bash
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

Cette commande déploie :
- Le serveur ArgoCD
- Le serveur d'API
- Le repository server (pour accéder à Git)
- Le controller (pour synchroniser)
- L'interface web UI
- Redis (pour le cache)

#### Étape 3 : Vérifier l'installation

```bash
# Attendre que tous les pods soient prêts
kubectl get pods -n argocd

# Vous devriez voir quelque chose comme :
# NAME                                  READY   STATUS    RESTARTS   AGE
# argocd-application-controller-0       1/1     Running   0          2m
# argocd-server-xxx                     1/1     Running   0          2m
# argocd-repo-server-xxx                1/1     Running   0          2m
# argocd-redis-xxx                      1/1     Running   0          2m
# argocd-dex-server-xxx                 1/1     Running   0          2m
```

#### Étape 4 : Exposer l'interface web

Par défaut, ArgoCD n'est accessible que depuis l'intérieur du cluster. Plusieurs options pour l'exposer :

**Option A : Port-forward (pour tests locaux)**

```bash
kubectl port-forward svc/argocd-server -n argocd 8080:443
```

Accès : https://localhost:8080

**Option B : NodePort (pour accès depuis le réseau local)**

```bash
kubectl patch svc argocd-server -n argocd -p '{"spec": {"type": "NodePort"}}'

# Récupérer le port
kubectl get svc argocd-server -n argocd
```

Accès : https://localhost:PORT

**Option C : Ingress (pour accès externe avec nom de domaine)**

```yaml
# argocd-ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: argocd-server-ingress
  namespace: argocd
  annotations:
    cert-manager.io/cluster-issuer: letsencrypt-prod
    nginx.ingress.kubernetes.io/ssl-passthrough: "true"
    nginx.ingress.kubernetes.io/backend-protocol: "HTTPS"
spec:
  ingressClassName: nginx
  rules:
  - host: argocd.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: argocd-server
            port:
              name: https
  tls:
  - hosts:
    - argocd.example.com
    secretName: argocd-tls
```

```bash
kubectl apply -f argocd-ingress.yaml
```

Accès : https://argocd.example.com

#### Étape 5 : Récupérer le mot de passe initial

Le mot de passe administrateur initial est généré automatiquement :

```bash
# Récupérer le mot de passe
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d; echo
```

**Credentials** :
- Username : `admin`
- Password : (celui affiché par la commande ci-dessus)

#### Étape 6 : Se connecter à l'interface

1. Accédez à l'URL (selon votre méthode d'exposition)
2. Acceptez le certificat auto-signé (pour les tests locaux)
3. Connectez-vous avec `admin` et le mot de passe récupéré

🎉 **ArgoCD est installé et prêt à l'emploi !**

### Méthode 2 : Installation via Helm

Pour plus de flexibilité dans la configuration :

```bash
# Ajouter le repo Helm
helm repo add argo https://argoproj.github.io/argo-helm
helm repo update

# Installer ArgoCD
helm install argocd argo/argo-cd \
  --namespace argocd \
  --create-namespace \
  --set server.service.type=NodePort
```

### Installation du CLI ArgoCD (optionnel mais recommandé)

Le CLI ArgoCD facilite les opérations en ligne de commande :

```bash
# Linux
curl -sSL -o argocd https://github.com/argoproj/argo-cd/releases/latest/download/argocd-linux-amd64
chmod +x argocd
sudo mv argocd /usr/local/bin/

# Vérifier l'installation
argocd version
```

**Se connecter via le CLI** :

```bash
# Port-forward si nécessaire
kubectl port-forward svc/argocd-server -n argocd 8080:443 &

# Se connecter
argocd login localhost:8080

# Username: admin
# Password: (le mot de passe initial)
```

---

## Configuration initiale d'ArgoCD

### Changer le mot de passe admin

Pour la sécurité, changez le mot de passe par défaut :

**Via le CLI** :

```bash
argocd account update-password
```

**Via l'interface web** :

User Info (en haut à droite) → Update Password

### Connecter un repository Git

ArgoCD doit avoir accès à votre repository Git pour récupérer les manifestes.

#### Repository public

**Via l'interface web** :

1. Settings → Repositories
2. Connect Repo
3. Method : VIA HTTPS
4. Repository URL : https://github.com/votre-username/votre-repo.git
5. Connect

**Via le CLI** :

```bash
argocd repo add https://github.com/votre-username/votre-repo.git
```

#### Repository privé

Pour un repository privé, vous devez fournir des credentials.

**Avec Personal Access Token (recommandé)** :

```bash
argocd repo add https://github.com/votre-username/votre-repo.git \
  --username votre-username \
  --password ghp_votreTOKEN
```

**Avec clé SSH** :

```bash
# Générer une clé SSH
ssh-keygen -t ed25519 -C "argocd@example.com" -f ~/.ssh/argocd

# Ajouter la clé publique à GitHub/GitLab (Deploy Keys)
cat ~/.ssh/argocd.pub

# Ajouter le repo dans ArgoCD
argocd repo add git@github.com:votre-username/votre-repo.git \
  --ssh-private-key-path ~/.ssh/argocd
```

### Configuration du registry privé

Si vous utilisez votre registry privé MicroK8s, ArgoCD doit pouvoir y accéder.

**Créer un secret pour le registry** :

```bash
kubectl create secret docker-registry registry-credentials \
  --docker-server=localhost:32000 \
  --docker-username=admin \
  --docker-password=votremotdepasse \
  --docker-email=admin@example.com \
  -n argocd
```

**Associer le secret au service account ArgoCD** :

```bash
kubectl patch serviceaccount argocd-application-controller -n argocd \
  -p '{"imagePullSecrets": [{"name": "registry-credentials"}]}'
```

---

## Créer votre première application ArgoCD

### Structure du repository Git

Avant de créer une application, organisez votre repository Git :

```
mon-repo-gitops/
├── apps/
│   ├── production/
│   │   ├── web-app/
│   │   │   ├── deployment.yaml
│   │   │   ├── service.yaml
│   │   │   └── ingress.yaml
│   │   └── api/
│   │       ├── deployment.yaml
│   │       └── service.yaml
│   └── staging/
│       └── web-app/
│           ├── deployment.yaml
│           └── service.yaml
└── README.md
```

### Exemple de manifestes

**deployment.yaml** :

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-app
  namespace: production
  labels:
    app: web-app
spec:
  replicas: 3
  selector:
    matchLabels:
      app: web-app
  template:
    metadata:
      labels:
        app: web-app
    spec:
      containers:
      - name: web
        image: localhost:32000/web-app:v1.0.0
        ports:
        - containerPort: 8080
        resources:
          requests:
            memory: "128Mi"
            cpu: "100m"
          limits:
            memory: "256Mi"
            cpu: "500m"
```

**service.yaml** :

```yaml
apiVersion: v1
kind: Service
metadata:
  name: web-app
  namespace: production
spec:
  selector:
    app: web-app
  ports:
  - port: 80
    targetPort: 8080
  type: ClusterIP
```

### Créer l'application via l'interface web

1. **Applications** → **+ New App**

2. **Remplir les informations** :
   - **Application Name** : web-app-production
   - **Project** : default
   - **Sync Policy** : Manual (pour commencer)
   - **Repository URL** : https://github.com/votre-username/mon-repo-gitops.git
   - **Revision** : HEAD (ou main)
   - **Path** : apps/production/web-app
   - **Cluster URL** : https://kubernetes.default.svc (cluster local)
   - **Namespace** : production

3. **Create**

### Créer l'application via le CLI

```bash
argocd app create web-app-production \
  --repo https://github.com/votre-username/mon-repo-gitops.git \
  --path apps/production/web-app \
  --dest-server https://kubernetes.default.svc \
  --dest-namespace production \
  --sync-policy manual
```

### Créer l'application via un manifeste YAML

ArgoCD lui-même utilise des Custom Resource Definitions (CRD). Vous pouvez définir vos applications comme des ressources Kubernetes :

```yaml
# web-app-argocd.yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: web-app-production
  namespace: argocd
spec:
  project: default

  # Source Git
  source:
    repoURL: https://github.com/votre-username/mon-repo-gitops.git
    targetRevision: HEAD
    path: apps/production/web-app

  # Destination Kubernetes
  destination:
    server: https://kubernetes.default.svc
    namespace: production

  # Politique de synchronisation
  syncPolicy:
    automated:
      prune: true      # Supprimer les ressources obsolètes
      selfHeal: true   # Auto-correction des dérives
      allowEmpty: false
    syncOptions:
    - CreateNamespace=true
    retry:
      limit: 5
      backoff:
        duration: 5s
        factor: 2
        maxDuration: 3m
```

Appliquer :

```bash
kubectl apply -f web-app-argocd.yaml
```

### Synchroniser l'application

Une fois l'application créée, ArgoCD détecte qu'elle est **OutOfSync** (pas synchronisée).

**Via l'interface web** :

1. Cliquez sur l'application
2. Bouton **Sync** en haut
3. **Synchronize**

**Via le CLI** :

```bash
argocd app sync web-app-production
```

ArgoCD va alors :
1. Récupérer les manifestes depuis Git
2. Les appliquer sur le cluster
3. Marquer l'application comme **Synced**

**Vérifier le déploiement** :

```bash
kubectl get all -n production
```

---

## Synchronisation automatique

### Activer la synchronisation automatique

La vraie puissance d'ArgoCD vient de la synchronisation automatique.

**Via l'interface web** :

1. Application Details
2. App Details (bouton en haut)
3. Enable Auto-Sync

**Via le CLI** :

```bash
argocd app set web-app-production --sync-policy automated
```

**Via le manifeste** :

```yaml
spec:
  syncPolicy:
    automated:
      prune: true      # Supprimer les ressources qui ne sont plus dans Git
      selfHeal: true   # Corriger automatiquement les modifications manuelles
```

### Options de synchronisation

**prune** :
- `true` : Supprime les ressources qui n'existent plus dans Git
- `false` : Garde les ressources même si supprimées de Git

**selfHeal** :
- `true` : Si quelqu'un modifie manuellement une ressource, ArgoCD la remet dans l'état Git
- `false` : ArgoCD signale la dérive mais ne corrige pas automatiquement

**allowEmpty** :
- `false` : Empêche de synchroniser si le dossier Git est vide
- `true` : Autorise les synchronisations même si vide

### Exemple : Workflow automatique complet

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: web-app-production
  namespace: argocd
  finalizers:
  - resources-finalizer.argocd.argoproj.io  # Nettoyage lors de la suppression
spec:
  project: default

  source:
    repoURL: https://github.com/votre-username/mon-repo-gitops.git
    targetRevision: main
    path: apps/production/web-app

  destination:
    server: https://kubernetes.default.svc
    namespace: production

  syncPolicy:
    automated:
      prune: true
      selfHeal: true
      allowEmpty: false

    syncOptions:
    - CreateNamespace=true        # Créer le namespace si inexistant
    - PruneLast=true              # Supprimer les ressources à la fin
    - ApplyOutOfSyncOnly=true     # Appliquer seulement les ressources désynchronisées

    retry:
      limit: 5
      backoff:
        duration: 5s
        factor: 2
        maxDuration: 3m

  # Hooks de synchronisation
  syncPolicy:
    syncOptions:
    - CreateNamespace=true
```

---

## Gestion des environnements multiples

### Approche 1 : Applications séparées

Une application ArgoCD par environnement :

```bash
# Créer l'app staging
argocd app create web-app-staging \
  --repo https://github.com/votre-username/mon-repo-gitops.git \
  --path apps/staging/web-app \
  --dest-namespace staging \
  --sync-policy automated

# Créer l'app production
argocd app create web-app-production \
  --repo https://github.com/votre-username/mon-repo-gitops.git \
  --path apps/production/web-app \
  --dest-namespace production \
  --sync-policy manual  # Manuel pour la prod
```

### Approche 2 : App of Apps pattern

Créez une "méta-application" qui déploie toutes vos applications :

```yaml
# apps/root-app.yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: root-app
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/votre-username/mon-repo-gitops.git
    targetRevision: HEAD
    path: apps
  destination:
    server: https://kubernetes.default.svc
    namespace: argocd
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
```

Puis dans `apps/`, définissez vos applications :

```yaml
# apps/web-app-production.yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: web-app-production
  namespace: argocd
spec:
  # ... définition de l'application
```

Une seule synchronisation du `root-app` déploie toutes vos applications !

### Approche 3 : ApplicationSets

Pour des patterns répétitifs (ex: même app dans plusieurs environnements) :

```yaml
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: web-app-envs
  namespace: argocd
spec:
  generators:
  - list:
      elements:
      - env: staging
        syncPolicy: automated
      - env: production
        syncPolicy: manual

  template:
    metadata:
      name: 'web-app-{{env}}'
    spec:
      project: default
      source:
        repoURL: https://github.com/votre-username/mon-repo-gitops.git
        targetRevision: HEAD
        path: 'apps/{{env}}/web-app'
      destination:
        server: https://kubernetes.default.svc
        namespace: '{{env}}'
      syncPolicy:
        automated: '{{syncPolicy}}'
```

Cela génère automatiquement deux applications : `web-app-staging` et `web-app-production`.

---

## Workflow GitOps complet avec ArgoCD

Voici le workflow end-to-end, du code à la production :

### Scénario : Déployer une nouvelle version

#### 1. Développement et push du code

```bash
# Le développeur modifie le code
vim app/main.py

# Commit et push
git add app/main.py
git commit -m "Ajout de la fonctionnalité X"
git push origin main
```

#### 2. Pipeline CI/CD

Le pipeline GitLab CI ou Jenkins se déclenche :

```yaml
# .gitlab-ci.yml (extrait)
build:
  stage: build
  script:
    # Build de l'image Docker
    - docker build -t localhost:32000/web-app:$CI_COMMIT_SHA .
    - docker push localhost:32000/web-app:$CI_COMMIT_SHA

update-manifest:
  stage: deploy
  script:
    # Cloner le repo de manifestes
    - git clone https://github.com/votre-username/mon-repo-gitops.git
    - cd mon-repo-gitops

    # Mettre à jour la version de l'image
    - sed -i "s|image: localhost:32000/web-app:.*|image: localhost:32000/web-app:$CI_COMMIT_SHA|" \
        apps/production/web-app/deployment.yaml

    # Commit et push
    - git add apps/production/web-app/deployment.yaml
    - git commit -m "Update web-app to $CI_COMMIT_SHA"
    - git push origin main
```

#### 3. ArgoCD détecte et synchronise

ArgoCD détecte le changement dans le repository Git (toutes les 3 minutes) :

```
[ArgoCD] 🔍 Nouveau commit détecté dans mon-repo-gitops
[ArgoCD] 📊 Comparaison : Git vs Kubernetes
[ArgoCD] ⚠️  Application OutOfSync détectée
[ArgoCD] 🔄 Synchronisation automatique...
[ArgoCD] ✅ Application déployée avec succès
```

#### 4. Vérification

```bash
# Vérifier que les nouveaux pods sont créés
kubectl get pods -n production -w

# Vérifier l'historique ArgoCD
argocd app history web-app-production
```

**Résultat** : En quelques minutes, votre code est en production, de manière totalement automatisée et traçable !

### Avantages de ce workflow

✅ **Traçabilité complète** : Chaque changement est dans Git avec un commit explicite

✅ **Rollback facile** : `git revert` pour revenir en arrière

✅ **Pas de kubectl direct** : Les développeurs n'ont pas besoin d'accès au cluster

✅ **Audit automatique** : Git log = historique des déploiements

✅ **Self-healing** : Si quelqu'un modifie manuellement, ArgoCD corrige automatiquement

---

## Détection et correction des dérives (Drift Detection)

### Qu'est-ce qu'une dérive ?

Une **dérive** (drift) se produit quand l'état réel du cluster ne correspond plus à l'état défini dans Git.

**Causes courantes** :
- Quelqu'un exécute `kubectl edit` ou `kubectl patch` directement
- Une HPA modifie le nombre de replicas
- Une ressource est supprimée manuellement
- Un bug dans un opérateur

### Détection par ArgoCD

ArgoCD détecte les dérives en comparant constamment Git et Kubernetes.

**Dans l'interface** : L'application passe en statut **OutOfSync** avec le détail des différences.

**Via le CLI** :

```bash
# Voir le statut de sync
argocd app get web-app-production

# Voir les différences
argocd app diff web-app-production
```

### Auto-correction (Self-Heal)

Avec `selfHeal: true`, ArgoCD corrige automatiquement les dérives :

**Scénario** :

```bash
# Quelqu'un modifie manuellement le nombre de replicas
kubectl scale deployment/web-app --replicas=10 -n production

# Après quelques secondes...
# ArgoCD détecte la dérive : Git dit 3 replicas, K8s a 10
# ArgoCD re-applique automatiquement les 3 replicas de Git
# La modification manuelle est annulée
```

**Logs ArgoCD** :

```
Detected drift in Deployment production/web-app
Spec.replicas: desired=3, actual=10
Auto-healing enabled, re-syncing...
Deployment scaled back to 3 replicas
Application synced successfully
```

### Exceptions à la dérive

Certaines modifications sont légitimes et ne doivent pas être corrigées :

**Ignorer certains champs** :

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: web-app-production
  annotations:
    # Ignorer le nombre de replicas (géré par HPA)
    argocd.argoproj.io/sync-options: IgnoreDifferences={"kind":"Deployment","jsonPointers":["/spec/replicas"]}
```

---

## Gestion des secrets avec ArgoCD

### Le défi des secrets

ArgoCD synchronise Git avec Kubernetes, mais comment gérer les secrets si on ne peut pas les mettre en clair dans Git ?

### Solution 1 : Sealed Secrets

**Sealed Secrets** chiffre les secrets pour pouvoir les versionner en toute sécurité.

**Installation** :

```bash
kubectl apply -f https://github.com/bitnami-labs/sealed-secrets/releases/download/v0.24.0/controller.yaml

# Installer le CLI kubeseal
wget https://github.com/bitnami-labs/sealed-secrets/releases/download/v0.24.0/kubeseal-linux-amd64 -O kubeseal
chmod +x kubeseal
sudo mv kubeseal /usr/local/bin/
```

**Utilisation** :

```bash
# Créer un secret normal (sans l'appliquer)
kubectl create secret generic db-password \
  --from-literal=password=SuperSecret123 \
  --dry-run=client -o yaml > secret.yaml

# Chiffrer le secret
kubeseal -f secret.yaml -w sealed-secret.yaml

# Le sealed secret peut être versionné dans Git
cat sealed-secret.yaml
```

Le `sealed-secret.yaml` peut être commité dans Git. Seul le cluster peut le déchiffrer !

**Dans votre application ArgoCD** :

```yaml
# apps/production/web-app/sealed-secret.yaml
apiVersion: bitnami.com/v1alpha1
kind: SealedSecret
metadata:
  name: db-password
  namespace: production
spec:
  encryptedData:
    password: AgBy3i4OJSWK+PiTySYZ... (données chiffrées)
```

### Solution 2 : External Secrets Operator

Synchronise les secrets depuis un gestionnaire externe (Vault, AWS Secrets Manager, etc.) :

```yaml
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: db-password
  namespace: production
spec:
  refreshInterval: 1h
  secretStoreRef:
    name: vault-backend
    kind: SecretStore
  target:
    name: db-password
  data:
  - secretKey: password
    remoteRef:
      key: secret/db-password
```

### Solution 3 : Helm avec values séparés

Si vous utilisez Helm avec ArgoCD :

```yaml
# Dans ArgoCD Application
spec:
  source:
    helm:
      valueFiles:
      - values-production.yaml
      - values-secrets.yaml  # Non versionné, créé manuellement
```

---

## Rollback avec ArgoCD

### Rollback via l'interface

1. Ouvrir l'application
2. **History and Rollback** (onglet)
3. Sélectionner une révision précédente
4. **Rollback**

ArgoCD synchronise vers cette révision Git.

### Rollback via le CLI

```bash
# Voir l'historique
argocd app history web-app-production

# Rollback vers une révision spécifique
argocd app rollback web-app-production 5
```

### Rollback via Git

La méthode la plus GitOps :

```bash
# Dans votre repo Git
git log --oneline

# Identifier le commit à restaurer
git revert abc123

# Push
git push origin main
```

ArgoCD détecte le changement et synchronise automatiquement.

---

## Monitoring et notifications

### Intégration avec Prometheus

ArgoCD expose des métriques Prometheus :

```yaml
apiVersion: v1
kind: Service
metadata:
  name: argocd-metrics
  namespace: argocd
  labels:
    app.kubernetes.io/name: argocd-metrics
spec:
  ports:
  - port: 8082
    protocol: TCP
    targetPort: 8082
  selector:
    app.kubernetes.io/name: argocd-application-controller
```

**Métriques disponibles** :
- `argocd_app_info` : Info sur les applications
- `argocd_app_sync_total` : Nombre de synchronisations
- `argocd_app_health_status` : Statut de santé
- `argocd_git_request_duration_seconds` : Temps de requête Git

### Dashboards Grafana

Importez des dashboards pré-configurés :
- ID 14584 : ArgoCD Application Overview
- ID 19993 : ArgoCD Operational Overview

### Notifications

Configurez des notifications pour les événements ArgoCD.

**Installation du controller de notifications** :

```bash
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj-labs/argocd-notifications/stable/manifests/install.yaml
```

**Configuration Slack** :

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: argocd-notifications-cm
  namespace: argocd
data:
  service.slack: |
    token: $slack-token

  template.app-deployed: |
    message: |
      Application {{.app.metadata.name}} deployed!
      Revision: {{.app.status.sync.revision}}

  trigger.on-deployed: |
    - when: app.status.operationState.phase in ['Succeeded']
      send: [app-deployed]
```

**Associer la notification à une application** :

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: web-app-production
  annotations:
    notifications.argoproj.io/subscribe.on-deployed.slack: prod-notifications
```

---

## Projets ArgoCD

### Organisation avec des projets

Les **projets** permettent d'organiser et de restreindre les applications.

**Créer un projet** :

```yaml
apiVersion: argoproj.io/v1alpha1
kind: AppProject
metadata:
  name: production
  namespace: argocd
spec:
  description: Applications de production

  # Repositories Git autorisés
  sourceRepos:
  - 'https://github.com/votre-org/production-apps.git'

  # Clusters de destination autorisés
  destinations:
  - namespace: 'production'
    server: https://kubernetes.default.svc
  - namespace: 'monitoring'
    server: https://kubernetes.default.svc

  # Types de ressources autorisés
  clusterResourceWhitelist:
  - group: ''
    kind: Namespace

  namespaceResourceWhitelist:
  - group: 'apps'
    kind: Deployment
  - group: ''
    kind: Service
  - group: 'networking.k8s.io'
    kind: Ingress

  # Restrictions
  roles:
  - name: dev-team
    policies:
    - p, proj:production:dev-team, applications, get, production/*, allow
    - p, proj:production:dev-team, applications, sync, production/*, allow
    groups:
    - developers
```

**Utiliser le projet** :

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: web-app
spec:
  project: production  # Utilise le projet 'production'
  # ...
```

---

## Multi-cluster avec ArgoCD

ArgoCD peut gérer plusieurs clusters depuis une seule instance.

### Ajouter un cluster

```bash
# Lister les contextes disponibles
kubectl config get-contexts

# Ajouter un cluster
argocd cluster add context-name
```

### Déployer sur un cluster spécifique

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: web-app-cluster2
spec:
  destination:
    server: https://cluster2.example.com  # URL du cluster
    namespace: production
```

---

## Bonnes pratiques ArgoCD

### 1. Structure Git claire

```
gitops-repo/
├── apps/
│   ├── base/              # Manifestes de base
│   ├── overlays/
│   │   ├── staging/       # Customizations staging
│   │   └── production/    # Customizations production
├── infrastructure/        # Resources d'infra
└── argocd/               # Définitions des apps ArgoCD
```

### 2. Sync automatique en staging, manuel en prod

```yaml
# Staging : auto
syncPolicy:
  automated:
    prune: true
    selfHeal: true

# Production : manuel
syncPolicy:
  automated: {}  # Vide = manuel
```

### 3. Toujours utiliser des versions d'images spécifiques

```yaml
# ❌ Éviter
image: nginx:latest

# ✅ Bon
image: localhost:32000/mon-app:v1.2.3
```

### 4. Séparer code et config

- **Repo 1** : Code de l'application (CI/CD build)
- **Repo 2** : Manifestes Kubernetes (ArgoCD sync)

### 5. Utiliser des health checks

```yaml
spec:
  source:
    path: apps/production
  ignoreDifferences:
  - group: apps
    kind: Deployment
    jsonPointers:
    - /spec/replicas  # Ignoré si HPA présent
```

### 6. Limiter les permissions

Utilisez des projets pour restreindre ce que chaque équipe peut faire.

### 7. Monitorer ArgoCD

- Configurez des alertes sur les sync failures
- Surveillez les métriques Prometheus
- Activez les notifications importantes

### 8. Documenter les applications

```yaml
metadata:
  name: web-app-production
  annotations:
    description: "Application web principale"
    owner: "team-platform"
    documentation: "https://wiki.example.com/web-app"
```

### 9. Tester avant la production

Utilisez des environnements de staging synchronisés automatiquement pour valider les changements.

### 10. Sauvegarder la configuration ArgoCD

```bash
# Exporter toutes les applications
argocd app list -o yaml > argocd-apps-backup.yaml

# Exporter les projets
kubectl get appprojects -n argocd -o yaml > argocd-projects-backup.yaml
```

---

## Troubleshooting ArgoCD

### Application ne se synchronise pas

**Diagnostic** :

```bash
# Vérifier le statut
argocd app get web-app-production

# Voir les logs
kubectl logs -n argocd deployment/argocd-application-controller

# Forcer une synchronisation
argocd app sync web-app-production --force
```

**Causes courantes** :
- Erreur dans les manifestes YAML
- Problème de connexion au repository Git
- Ressources avec des champs immutables modifiés
- Problèmes de permissions RBAC

### ArgoCD ne détecte pas les changements Git

**Vérifier la configuration du webhook** :

Si vous voulez une synchronisation immédiate au lieu d'attendre 3 minutes, configurez un webhook Git.

**Dans GitLab/GitHub** : Settings → Webhooks
- URL : `https://argocd.example.com/api/webhook`
- Secret : Token ArgoCD

### Problèmes de certificats

```bash
# Désactiver la vérification SSL (développement seulement)
argocd repo add https://github.com/user/repo.git --insecure-skip-server-verification
```

### Out of Sync mais rien ne change

Certaines ressources ont des champs calculés qui diffèrent toujours :

```yaml
# Ignorer ces différences
spec:
  ignoreDifferences:
  - group: apps
    kind: Deployment
    jsonPointers:
    - /spec/replicas
```

---

## Récapitulatif

### Points clés

**ArgoCD, c'est** :
- L'implémentation de référence de GitOps
- La synchronisation automatique Git → Kubernetes
- La détection et correction des dérives
- Une interface web intuitive pour visualiser vos déploiements

**Installation** :
- Simple avec les manifestes officiels
- Exposition via Port-forward, NodePort ou Ingress
- CLI optionnel mais recommandé

**Applications ArgoCD** :
- Définies via l'interface, CLI ou manifestes YAML
- Synchronisation manuelle ou automatique
- Self-healing et pruning configurables
- Rollback facile vers n'importe quelle révision Git

**Workflow GitOps** :
1. Code pushed to Git
2. CI/CD builds and updates manifests
3. ArgoCD detects changes
4. ArgoCD syncs cluster automatically
5. Monitoring validates deployment

**Bonnes pratiques** :
- Structure Git claire
- Auto-sync en staging, manuel en prod
- Versions d'images spécifiques
- Projets pour organiser et sécuriser
- Notifications configurées
- Backup régulier

### Bénéfices immédiats

Avec ArgoCD :
- ✅ Déploiements automatiques et fiables
- ✅ Rollback en quelques clics
- ✅ Audit complet dans Git
- ✅ Pas de kubectl direct nécessaire
- ✅ Auto-correction des modifications manuelles
- ✅ Visualisation claire de l'état du cluster

---

## Conclusion : Le GitOps complet

Vous avez maintenant tous les éléments d'une infrastructure GitOps moderne :

```
┌────────────┐     ┌──────────┐     ┌──────────┐     ┌──────────┐
│    Git     │ --> │   CI/CD  │ --> │ Registry │ --> │  ArgoCD  │
│ (manifests)│     │ (build)  │     │ (images) │     │  (sync)  │
└────────────┘     └──────────┘     └──────────┘     └──────────┘
                                                            ↓
                                                     ┌──────────┐
                                                     │   K8s    │
                                                     │ Cluster  │
                                                     └──────────┘
```

**Le cycle complet** :
1. **Git** : Source de vérité pour le code et les configs
2. **CI/CD** : Build, test, et push des images
3. **Registry** : Stockage sécurisé des images
4. **ArgoCD** : Synchronisation automatique avec K8s
5. **Monitoring** : Validation et alertes

C'est le **GitOps parfait** : déclaratif, versionné, automatique et auto-correctif !

---

## Prochaines étapes

Maintenant que vous maîtrisez l'ensemble de la chaîne DevOps :
- Git pour versionner (17.2)
- Registry pour stocker (17.3)
- CI/CD pour automatiser (17.4)
- ArgoCD pour synchroniser (17.5)

Vous êtes prêt pour :

1. **Section 17.6** : Helm pour packager vos applications
2. **Section 17.7** : Créer vos propres Helm Charts
3. **Section 17.8** : Stratégies de déploiement avancées (Blue-Green, Canary)

**Félicitations !** Vous avez construit une infrastructure DevOps complète sur MicroK8s, avec les mêmes outils et pratiques utilisés en production par les plus grandes organisations !

**Prêt à packager vos applications avec Helm ?** Passons à la section 17.6 : Helm - gestionnaire de packages !

⏭️ [Helm : gestionnaire de packages](/17-devops-et-cicd/06-helm-gestionnaire-de-packages.md)
