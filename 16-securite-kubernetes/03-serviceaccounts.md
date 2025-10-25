🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 16.3 ServiceAccounts

## Introduction

Les ServiceAccounts sont des identités spéciales dans Kubernetes, conçues pour les applications et les pods plutôt que pour les personnes. Alors que RBAC définit ce qu'on peut faire, les ServiceAccounts définissent l'identité sous laquelle on le fait.

**Analogie simple :** Imaginez un immeuble de bureaux. Les employés ont des badges personnels (comme les utilisateurs Kubernetes), mais les systèmes automatisés (climatisation, sécurité, ascenseurs) ont aussi leurs propres "identifiants" pour fonctionner. Les ServiceAccounts sont ces identifiants pour les applications.

## Pourquoi les ServiceAccounts sont Nécessaires ?

### Le Problème sans ServiceAccounts

Vos applications dans Kubernetes ont souvent besoin d'interagir avec l'API Kubernetes :

- Une application de monitoring lit les métriques des pods
- Un système de déploiement crée et met à jour des ressources
- Un job automatisé liste des services pour la découverte

**Question :** Comment ces applications s'authentifient-elles auprès de l'API Kubernetes ?

**Réponse :** Via des ServiceAccounts !

### La Solution avec ServiceAccounts

```
┌─────────────────────────────────────────┐
│  Pod (Application)                      │
│                                         │
│  ┌───────────────────────────────────┐  │
│  │ ServiceAccount Token              │  │
│  │ (injecté automatiquement)         │  │
│  └───────────────┬───────────────────┘  │
└──────────────────┼──────────────────────┘
                   │
                   ▼
         ┌─────────────────┐
         │  API Server     │
         │                 │
         │ 1. Authentifie  │
         │ 2. Autorise     │
         │ 3. Exécute      │
         └─────────────────┘
```

## Users vs ServiceAccounts

### Comparaison

| Aspect | User | ServiceAccount |
|--------|------|----------------|
| **Pour qui ?** | Personnes physiques | Applications, pods, processus |
| **Géré par** | Système externe (certificats, OIDC) | Kubernetes directement |
| **Portée** | Cluster-wide | Namespace spécifique |
| **Authentification** | Certificats, tokens externes | Tokens Kubernetes |
| **Création** | Manuel, via système externe | `kubectl create serviceaccount` |

### User (Rappel)

```yaml
subjects:
- kind: User
  name: alice@example.com
```

- Représente une personne
- Géré en dehors de Kubernetes
- Utilise des certificats ou SSO

### ServiceAccount

```yaml
subjects:
- kind: ServiceAccount
  name: my-app
  namespace: production
```

- Représente une application
- Géré par Kubernetes
- Utilise des tokens automatiques

## Concepts de Base

### Anatomie d'un ServiceAccount

Un ServiceAccount est composé de plusieurs éléments :

```
ServiceAccount
    │
    ├─── Nom (identité)
    │
    ├─── Namespace (portée)
    │
    ├─── Secrets (tokens d'authentification)
    │
    └─── Permissions (via RBAC)
```

### ServiceAccount par Défaut

Chaque namespace a automatiquement un ServiceAccount nommé "default" :

```bash
kubectl get serviceaccounts -n default
```

```
NAME      SECRETS   AGE
default   1         10d
```

**Important :** Si vous ne spécifiez pas de ServiceAccount dans un pod, il utilisera automatiquement le ServiceAccount "default" de son namespace.

## Création de ServiceAccounts

### Méthode 1 : Commande kubectl

```bash
# Créer un ServiceAccount
kubectl create serviceaccount my-app -n production

# Vérifier
kubectl get serviceaccount my-app -n production
```

### Méthode 2 : Fichier YAML

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: my-app
  namespace: production
  labels:
    app: my-app
  annotations:
    description: "ServiceAccount pour l'application my-app"
```

**Création :**
```bash
kubectl apply -f serviceaccount.yaml
```

### Méthode 3 : Inline dans un Deployment

Vous pouvez créer le ServiceAccount en même temps que d'autres ressources :

```yaml
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: my-app
  namespace: production
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
  namespace: production
spec:
  replicas: 1
  selector:
    matchLabels:
      app: my-app
  template:
    metadata:
      labels:
        app: my-app
    spec:
      serviceAccountName: my-app  # Référence le SA créé au-dessus
      containers:
      - name: app
        image: my-app:latest
```

## Utilisation dans les Pods

### Spécifier un ServiceAccount

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-pod
  namespace: production
spec:
  serviceAccountName: my-app  # Important !
  containers:
  - name: app
    image: nginx:latest
```

### ServiceAccount par Défaut

Si vous ne spécifiez rien :

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-pod
spec:
  # serviceAccountName non spécifié
  # → utilise automatiquement "default"
  containers:
  - name: app
    image: nginx:latest
```

**Équivalent à :**
```yaml
spec:
  serviceAccountName: default
```

### Désactiver l'Auto-Mount du Token

Par défaut, Kubernetes monte automatiquement le token du ServiceAccount dans chaque pod. Vous pouvez désactiver ce comportement :

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: no-api-access
automountServiceAccountToken: false  # Désactive l'auto-mount
```

Ou au niveau du pod :

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-pod
spec:
  serviceAccountName: my-app
  automountServiceAccountToken: false  # Désactive pour ce pod uniquement
  containers:
  - name: app
    image: nginx:latest
```

**Quand désactiver ?**
- Application qui n'a pas besoin d'accéder à l'API Kubernetes
- Améliore la sécurité (principe du moindre privilège)

## Tokens et Authentification

### Token Automatique

Quand un pod utilise un ServiceAccount, Kubernetes monte automatiquement un token à l'intérieur du pod :

```
Emplacement dans le pod :
/var/run/secrets/kubernetes.io/serviceaccount/
    ├── token          (jeton JWT)
    ├── ca.crt         (certificat CA)
    └── namespace      (nom du namespace)
```

### Voir le Token dans un Pod

```bash
# Entrer dans un pod
kubectl exec -it my-pod -n production -- sh

# Afficher le token
cat /var/run/secrets/kubernetes.io/serviceaccount/token

# Afficher le namespace
cat /var/run/secrets/kubernetes.io/serviceaccount/namespace
```

### Structure du Token

Le token est un JWT (JSON Web Token) qui contient :

```
{
  "iss": "kubernetes/serviceaccount",
  "sub": "system:serviceaccount:production:my-app",
  "aud": ["https://kubernetes.default.svc"],
  "exp": 1735689600,
  ...
}
```

- **sub** : identité complète (namespace + nom du SA)
- **exp** : date d'expiration
- **aud** : audience (API server)

### Utiliser le Token

Votre application peut utiliser ce token pour s'authentifier auprès de l'API :

```bash
# Variable avec le token
TOKEN=$(cat /var/run/secrets/kubernetes.io/serviceaccount/token)

# Requête vers l'API
curl -H "Authorization: Bearer $TOKEN" \
     --cacert /var/run/secrets/kubernetes.io/serviceaccount/ca.crt \
     https://kubernetes.default.svc/api/v1/namespaces/production/pods
```

**Note :** Les SDK Kubernetes (client-go, etc.) gèrent tout cela automatiquement.

## ServiceAccounts et RBAC

### Lier ServiceAccount et Permissions

Un ServiceAccount sans permissions ne peut rien faire. Il faut le lier à un Role via un RoleBinding.

### Exemple Complet : Application de Monitoring

**Étape 1 : Créer le ServiceAccount**

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: monitoring-app
  namespace: monitoring
```

**Étape 2 : Créer un Role avec Permissions**

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: pod-reader
rules:
- apiGroups: [""]
  resources: ["pods", "nodes"]
  verbs: ["get", "list", "watch"]
```

**Étape 3 : Lier le ServiceAccount au Role**

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: monitoring-app-binding
subjects:
- kind: ServiceAccount
  name: monitoring-app
  namespace: monitoring
roleRef:
  kind: ClusterRole
  name: pod-reader
  apiGroup: rbac.authorization.k8s.io
```

**Étape 4 : Utiliser dans un Deployment**

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: monitoring-app
  namespace: monitoring
spec:
  replicas: 1
  selector:
    matchLabels:
      app: monitoring
  template:
    metadata:
      labels:
        app: monitoring
    spec:
      serviceAccountName: monitoring-app  # Utilise notre SA
      containers:
      - name: monitor
        image: monitoring-app:latest
```

### Vérifier les Permissions

```bash
# Vérifier ce que peut faire le ServiceAccount
kubectl auth can-i list pods \
  --as=system:serviceaccount:monitoring:monitoring-app

# Lister toutes les permissions
kubectl auth can-i --list \
  --as=system:serviceaccount:monitoring:monitoring-app
```

## Exemples Pratiques

### Exemple 1 : Application Lecture Seule

**Besoin :** Une application doit lire les ConfigMaps et Secrets de son namespace.

```yaml
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: config-reader
  namespace: production
---
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: config-reader-role
  namespace: production
rules:
- apiGroups: [""]
  resources: ["configmaps", "secrets"]
  verbs: ["get", "list"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: config-reader-binding
  namespace: production
subjects:
- kind: ServiceAccount
  name: config-reader
roleRef:
  kind: Role
  name: config-reader-role
  apiGroup: rbac.authorization.k8s.io
---
apiVersion: v1
kind: Pod
metadata:
  name: app
  namespace: production
spec:
  serviceAccountName: config-reader
  containers:
  - name: app
    image: my-app:latest
```

### Exemple 2 : Job de Backup

**Besoin :** Un CronJob doit lister et exporter des ressources.

```yaml
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: backup-job
  namespace: admin
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: backup-reader
rules:
- apiGroups: ["", "apps", "batch"]
  resources: ["*"]
  verbs: ["get", "list"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: backup-job-binding
subjects:
- kind: ServiceAccount
  name: backup-job
  namespace: admin
roleRef:
  kind: ClusterRole
  name: backup-reader
  apiGroup: rbac.authorization.k8s.io
---
apiVersion: batch/v1
kind: CronJob
metadata:
  name: backup
  namespace: admin
spec:
  schedule: "0 2 * * *"  # Tous les jours à 2h
  jobTemplate:
    spec:
      template:
        spec:
          serviceAccountName: backup-job
          containers:
          - name: backup
            image: backup-tool:latest
            command:
            - /bin/sh
            - -c
            - |
              # Le script peut maintenant lister toutes les ressources
              kubectl get all --all-namespaces -o yaml > /backup/cluster.yaml
          restartPolicy: OnFailure
```

### Exemple 3 : Application avec Service Discovery

**Besoin :** Une application doit découvrir d'autres services dynamiquement.

```yaml
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: service-discovery
  namespace: production
---
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: service-discovery-role
  namespace: production
rules:
- apiGroups: [""]
  resources: ["services", "endpoints"]
  verbs: ["get", "list", "watch"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: service-discovery-binding
  namespace: production
subjects:
- kind: ServiceAccount
  name: service-discovery
roleRef:
  kind: Role
  name: service-discovery-role
  apiGroup: rbac.authorization.k8s.io
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api-gateway
  namespace: production
spec:
  replicas: 2
  selector:
    matchLabels:
      app: api-gateway
  template:
    metadata:
      labels:
        app: api-gateway
    spec:
      serviceAccountName: service-discovery
      containers:
      - name: gateway
        image: api-gateway:latest
        env:
        - name: KUBERNETES_NAMESPACE
          valueFrom:
            fieldRef:
              fieldPath: metadata.namespace
```

### Exemple 4 : CI/CD Deployer

**Besoin :** Un pipeline CI/CD doit déployer des applications.

```yaml
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: ci-deployer
  namespace: cicd
---
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: deployer-role
  namespace: production
rules:
- apiGroups: ["apps"]
  resources: ["deployments", "replicasets"]
  verbs: ["get", "list", "create", "update", "patch"]
- apiGroups: [""]
  resources: ["services", "configmaps"]
  verbs: ["get", "list", "create", "update"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: ci-deployer-binding
  namespace: production
subjects:
- kind: ServiceAccount
  name: ci-deployer
  namespace: cicd
roleRef:
  kind: Role
  name: deployer-role
  apiGroup: rbac.authorization.k8s.io
```

## Gestion des ServiceAccounts

### Lister les ServiceAccounts

```bash
# Dans un namespace spécifique
kubectl get serviceaccounts -n production

# Dans tous les namespaces
kubectl get serviceaccounts --all-namespaces

# Format détaillé
kubectl get serviceaccount my-app -n production -o yaml
```

### Détails d'un ServiceAccount

```bash
kubectl describe serviceaccount my-app -n production
```

**Sortie :**
```
Name:                my-app
Namespace:           production
Labels:              app=my-app
Annotations:         <none>
Image pull secrets:  <none>
Mountable secrets:   <none>
Tokens:              <none>
Events:              <none>
```

### Supprimer un ServiceAccount

```bash
kubectl delete serviceaccount my-app -n production
```

**Attention :** Les pods utilisant ce ServiceAccount ne pourront plus s'authentifier auprès de l'API.

### Modifier un ServiceAccount

```bash
# Éditer directement
kubectl edit serviceaccount my-app -n production

# Ou via patch
kubectl patch serviceaccount my-app -n production \
  -p '{"metadata":{"annotations":{"description":"Updated SA"}}}'
```

## Image Pull Secrets

Les ServiceAccounts peuvent être liés à des secrets pour tirer des images de registres privés.

### Créer un Secret de Registry

```bash
kubectl create secret docker-registry my-registry-secret \
  --docker-server=registry.example.com \
  --docker-username=user \
  --docker-password=password \
  --docker-email=user@example.com \
  -n production
```

### Lier le Secret au ServiceAccount

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: my-app
  namespace: production
imagePullSecrets:
- name: my-registry-secret
```

### Utilisation

Maintenant, tous les pods utilisant ce ServiceAccount pourront tirer des images du registry privé :

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: app
  namespace: production
spec:
  serviceAccountName: my-app  # Hérite des imagePullSecrets
  containers:
  - name: app
    image: registry.example.com/my-private-image:latest
```

## Bonnes Pratiques

### 1. Un ServiceAccount par Application

**Mauvais :**
```yaml
# Plusieurs applications partagent le même SA
serviceAccountName: shared-sa
```

**Bon :**
```yaml
# Chaque application a son propre SA
serviceAccountName: app-1-sa
serviceAccountName: app-2-sa
```

**Avantages :**
- Permissions granulaires
- Meilleure traçabilité (audit)
- Isolation en cas de compromission

### 2. Principe du Moindre Privilège

**Mauvais :**
```yaml
# Permissions trop larges
verbs: ["*"]
resources: ["*"]
```

**Bon :**
```yaml
# Permissions spécifiques aux besoins
verbs: ["get", "list"]
resources: ["configmaps", "secrets"]
```

### 3. Ne Pas Utiliser le ServiceAccount "default"

**Mauvais :**
```yaml
# Utilise le SA par défaut
# serviceAccountName non spécifié
```

**Bon :**
```yaml
# SA dédié avec permissions appropriées
serviceAccountName: my-app-sa
```

**Pourquoi ?**
- Le SA "default" peut avoir des permissions par défaut
- Plus difficile à auditer
- Viole le principe de séparation

### 4. Désactiver l'Auto-Mount si Inutile

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: no-api-access
automountServiceAccountToken: false
```

**Quand l'utiliser ?**
- Application qui ne communique jamais avec l'API Kubernetes
- Conteneurs statiques (nginx, apache)
- Réduit la surface d'attaque

### 5. Documenter les ServiceAccounts

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: monitoring-app
  annotations:
    description: "ServiceAccount pour l'application de monitoring"
    owner: "team-platform"
    permissions: "Lecture seule des pods et nodes"
    last-review: "2024-01-15"
```

### 6. Limiter la Portée

**Préférer Role à ClusterRole quand possible :**

```yaml
# Bon : permissions limitées au namespace
kind: Role
metadata:
  namespace: production

# Attention : permissions cluster-wide
kind: ClusterRole
```

### 7. Audit Régulier

```bash
# Lister tous les ServiceAccounts
kubectl get sa --all-namespaces

# Vérifier les permissions
kubectl auth can-i --list \
  --as=system:serviceaccount:production:my-app

# Trouver les SA avec ClusterRoleBinding
kubectl get clusterrolebindings -o json | \
  jq '.items[] | select(.subjects[]?.kind=="ServiceAccount")'
```

## Débogage

### Problème : Pod ne Peut Pas Accéder à l'API

**Symptômes :**
```
Error: Unauthorized
403 Forbidden: User "system:serviceaccount:production:my-app"
cannot list resource "pods"
```

**Diagnostic :**

1. Vérifier le ServiceAccount du pod :
```bash
kubectl get pod my-pod -n production -o jsonpath='{.spec.serviceAccountName}'
```

2. Vérifier les permissions :
```bash
kubectl auth can-i list pods \
  --as=system:serviceaccount:production:my-app
```

3. Chercher les bindings :
```bash
kubectl get rolebindings,clusterrolebindings -A | grep my-app
```

**Solution :**
Créer ou modifier le RoleBinding/ClusterRoleBinding approprié.

### Problème : Token Non Monté

**Symptômes :**
```
Error: unable to load in-cluster configuration
```

**Diagnostic :**
```bash
# Vérifier dans le pod
kubectl exec my-pod -n production -- ls /var/run/secrets/kubernetes.io/serviceaccount/
```

**Causes possibles :**
1. `automountServiceAccountToken: false` configuré
2. Le ServiceAccount n'existe pas

**Solution :**
```yaml
spec:
  serviceAccountName: my-app
  automountServiceAccountToken: true  # Explicitement activé
```

### Problème : Mauvais ServiceAccount

**Symptômes :**
Le pod utilise "default" au lieu de votre SA personnalisé.

**Diagnostic :**
```bash
kubectl get pod my-pod -o jsonpath='{.spec.serviceAccountName}'
```

**Solution :**
Vérifier l'orthographe et l'existence du SA :
```bash
kubectl get sa my-app -n production
```

Si absent, créer le SA d'abord.

### Problème : Image Pull Failed

**Symptômes :**
```
Failed to pull image: unauthorized
```

**Diagnostic :**
```bash
# Vérifier les imagePullSecrets du SA
kubectl get sa my-app -n production -o jsonpath='{.imagePullSecrets}'
```

**Solution :**
Ajouter le secret au ServiceAccount :
```yaml
imagePullSecrets:
- name: my-registry-secret
```

## ServiceAccounts dans MicroK8s

### Utilisation Standard

MicroK8s gère les ServiceAccounts de manière standard. Toutes les commandes et configurations de ce chapitre fonctionnent :

```bash
# Créer un SA
microk8s kubectl create serviceaccount my-app -n default

# Lister
microk8s kubectl get serviceaccounts
```

### Token API et Projected Volumes

MicroK8s moderne (1.21+) utilise des tokens avec expiration (Token Request API) au lieu des anciens tokens secrets permanents.

**Avantages :**
- Tokens avec durée de vie limitée
- Rotation automatique
- Plus sécurisé

**Note :** C'est transparent pour vos applications.

## Commandes Utiles

### Création et Gestion

```bash
# Créer un ServiceAccount
kubectl create serviceaccount my-app -n production

# Obtenir les détails
kubectl get serviceaccount my-app -n production -o yaml

# Décrire
kubectl describe serviceaccount my-app -n production

# Supprimer
kubectl delete serviceaccount my-app -n production
```

### Vérification des Permissions

```bash
# Vérifier une action spécifique
kubectl auth can-i get pods \
  --as=system:serviceaccount:production:my-app

# Lister toutes les permissions
kubectl auth can-i --list \
  --as=system:serviceaccount:production:my-app \
  -n production
```

### Inspection des Pods

```bash
# Voir le SA d'un pod
kubectl get pod my-pod -o jsonpath='{.spec.serviceAccountName}'

# Voir si le token est monté
kubectl exec my-pod -- ls -la /var/run/secrets/kubernetes.io/serviceaccount/

# Lire le token
kubectl exec my-pod -- cat /var/run/secrets/kubernetes.io/serviceaccount/token
```

### Recherche et Audit

```bash
# Tous les SA dans le cluster
kubectl get serviceaccounts --all-namespaces

# SA avec bindings
kubectl get rolebindings,clusterrolebindings -A \
  -o json | jq '.items[] | select(.subjects[]?.kind=="ServiceAccount")'

# Pods utilisant un SA spécifique
kubectl get pods --all-namespaces \
  -o jsonpath='{range .items[?(@.spec.serviceAccountName=="my-app")]}{.metadata.namespace}/{.metadata.name}{"\n"}{end}'
```

## Évolution et Versions

### Anciennes Versions (< 1.21)

Les ServiceAccounts créaient automatiquement un Secret permanent avec le token.

```bash
kubectl get secrets
```
```
NAME                  TYPE                                  DATA   AGE
my-app-token-xyz123   kubernetes.io/service-account-token   3      1d
```

### Versions Modernes (>= 1.21)

Utilisation de la Token Request API avec projected volumes :

```yaml
volumes:
- name: kube-api-access
  projected:
    sources:
    - serviceAccountToken:
        expirationSeconds: 3607
        path: token
    - configMap:
        items:
        - key: ca.crt
          path: ca.crt
        name: kube-root-ca.crt
    - downwardAPI:
        items:
        - fieldRef:
            fieldPath: metadata.namespace
          path: namespace
```

**Avantages :**
- Tokens avec expiration
- Rotation automatique
- Binding à l'audience
- Plus sécurisé

## Conclusion

Les ServiceAccounts sont l'identité des applications dans Kubernetes :

1. **Identité** : chaque application doit avoir son propre ServiceAccount
2. **Authentification** : le token permet de s'authentifier auprès de l'API
3. **Autorisation** : RBAC définit ce que le ServiceAccount peut faire
4. **Isolation** : un ServiceAccount par application pour la sécurité
5. **Simplification** : gestion automatique par Kubernetes

**Formule ServiceAccount :**
```
ServiceAccount + RBAC = Permissions d'Application
```

**Points Clés à Retenir :**

- ServiceAccounts = identités pour applications/pods
- Différent des Users (pour les humains)
- Automatiquement injecté dans les pods via un token
- Doit être lié à un Role/ClusterRole via un Binding pour avoir des permissions
- Un ServiceAccount par application (principe du moindre privilège)
- Désactiver l'auto-mount si l'application n'utilise pas l'API
- Préférer des SA dédiés au SA "default"
- Auditer régulièrement les permissions

**Prochaines Étapes :**

Dans les sections suivantes, nous explorerons :
- **16.4** : Network Policies (isolation réseau entre pods)
- **16.5** : Pod Security Standards (sécurité au niveau conteneur)
- **16.6** : Scan de vulnérabilités d'images

Les ServiceAccounts combinés avec RBAC forment la base de la sécurité des applications dans Kubernetes. Maîtriser ces concepts vous permet de construire des systèmes sécurisés et bien isolés.

⏭️ [Network Policies](/16-securite-kubernetes/04-network-policies.md)
