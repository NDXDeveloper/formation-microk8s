🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 3.7 Secrets

## Introduction

Dans la section précédente, nous avons appris à gérer la configuration de nos applications avec les ConfigMaps. Mais qu'en est-il des **données sensibles** comme les mots de passe, les clés API ou les certificats SSL ?

Les **Secrets** sont la solution Kubernetes pour stocker et gérer les informations confidentielles de manière plus sécurisée que les ConfigMaps.

## Le problème : données sensibles exposées

### Scénario problématique

Imaginez que vous déployez une application qui doit se connecter à une base de données :

**❌ Mauvaise approche 1 : Codé en dur dans le code**
```javascript
// database.js
const dbConfig = {
  host: "postgres.example.com",
  username: "admin",
  password: "SuperSecret123!",        // ❌ Visible dans le code source !
  database: "production"
};
```

**❌ Mauvaise approche 2 : Dans un ConfigMap**
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: db-config
data:
  DB_PASSWORD: "SuperSecret123!"      # ❌ Visible en clair !
  API_KEY: "sk_live_abc123xyz"        # ❌ Aucun chiffrement !
```

**Problèmes** :

1. **Visibilité** : Tout le monde peut voir les mots de passe
2. **Sécurité** : Les secrets sont stockés en clair
3. **Versioning** : Risque de commit dans Git par erreur
4. **Audit** : Difficile de tracer qui a accès aux secrets
5. **Rotation** : Changer un mot de passe nécessite de modifier le code ou les ConfigMaps
6. **Conformité** : Non conforme aux standards de sécurité (RGPD, PCI-DSS, etc.)

### La solution : Secrets

Les Secrets Kubernetes permettent de :
- **Séparer** les données sensibles du code et de la configuration
- **Encoder** les données (base64) pour éviter une exposition directe
- **Contrôler l'accès** via RBAC
- **Limiter la visibilité** aux Pods qui en ont réellement besoin
- **Faciliter la rotation** des credentials
- **Améliorer l'audit** et la traçabilité

## Qu'est-ce qu'un Secret ?

### Définition simple

Un **Secret** est un objet Kubernetes qui stocke des données sensibles (mots de passe, tokens, clés) de manière plus sécurisée qu'un ConfigMap classique.

### Analogie : le coffre-fort

Imaginez un **coffre-fort** dans une banque :

```
┌─────────────────────────────────────────────────┐
│              Banque (Cluster K8s)               │
│                                                 │
│  ┌──────────────────────────────────────┐       │
│  │   Coffre-fort (Secret)               │       │
│  │   🔒 Verrouillé et sécurisé          │       │
│  │                                      │       │
│  │   Contenu (encodé) :                 │       │
│  │   • Mot de passe DB                  │       │
│  │   • Clé API                          │       │
│  │   • Certificat SSL                   │       │
│  └──────────────────────────────────────┘       │
│            ↓ Accès contrôlé                     │
│  ┌──────────────────────────────────────┐       │
│  │  Pod autorisé                        │       │
│  │  Peut lire le coffre                 │       │
│  └──────────────────────────────────────┘       │
│                                                 │
│  ┌──────────────────────────────────────┐       │
│  │  Pod non autorisé                    │       │
│  │  ❌ Ne peut pas accéder              │       │
│  └──────────────────────────────────────┘       │
└─────────────────────────────────────────────────┘
```

**Caractéristiques** :
- Le coffre est **sécurisé** (encodage base64 minimum)
- Seules les personnes **autorisées** peuvent y accéder (RBAC)
- Le contenu n'est **pas visible** directement
- On peut **auditer** qui accède au coffre

### Secrets vs ConfigMaps

| Caractéristique | ConfigMap | Secret |
|----------------|-----------|--------|
| **Usage** | Configuration non sensible | Données sensibles |
| **Encodage** | Texte brut | Base64 |
| **Visibilité** | Visible par défaut | Limitée par défaut |
| **Taille max** | 1 MiB | 1 MiB |
| **Stockage etcd** | Non chiffré | Peut être chiffré |
| **RBAC** | Permissions standards | Permissions séparées |
| **Cas d'usage** | URLs, ports, flags | Mots de passe, tokens, clés |

### Important : Les Secrets ne sont pas "super sécurisés" par défaut

**⚠️ Point crucial à comprendre** :

Par défaut, les Secrets Kubernetes sont seulement **encodés en base64**, pas **chiffrés**. Cela signifie :

- Base64 n'est **pas du chiffrement** : c'est juste un encodage réversible
- Toute personne avec accès à l'API Kubernetes peut lire les Secrets
- Les Secrets sont stockés en clair dans etcd (par défaut)

**Toutefois, les Secrets offrent des avantages** :
- Séparation claire des données sensibles
- Contrôle d'accès RBAC dédié
- Possibilité d'activer le chiffrement at-rest dans etcd
- Meilleures pratiques de gestion des credentials
- Intégration avec des solutions externes (HashiCorp Vault, AWS Secrets Manager)

## Types de Secrets

Kubernetes supporte plusieurs types de Secrets selon leur usage.

### 1. Opaque (par défaut)

Le type générique pour les données arbitraires.

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: mon-secret
type: Opaque                    # Type par défaut
data:
  username: YWRtaW4=            # "admin" en base64
  password: cGFzc3dvcmQxMjM=    # "password123" en base64
```

### 2. kubernetes.io/service-account-token

Créé automatiquement pour les ServiceAccounts (authentification interne).

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: default-token-xyz
type: kubernetes.io/service-account-token
```

### 3. kubernetes.io/dockerconfigjson

Pour l'authentification aux registres Docker privés.

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: regcred
type: kubernetes.io/dockerconfigjson
data:
  .dockerconfigjson: <base64-encoded-docker-config>
```

### 4. kubernetes.io/tls

Pour les certificats TLS (SSL).

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: tls-secret
type: kubernetes.io/tls
data:
  tls.crt: <base64-encoded-certificate>
  tls.key: <base64-encoded-private-key>
```

### 5. kubernetes.io/basic-auth

Pour l'authentification basique (username/password).

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: basic-auth
type: kubernetes.io/basic-auth
data:
  username: <base64-encoded-username>
  password: <base64-encoded-password>
```

### 6. kubernetes.io/ssh-auth

Pour les clés SSH.

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: ssh-key
type: kubernetes.io/ssh-auth
data:
  ssh-privatekey: <base64-encoded-ssh-key>
```

## Créer un Secret

### Encodage Base64

Les valeurs dans les Secrets doivent être encodées en base64.

**Encoder une valeur** :
```bash
# Linux/macOS
echo -n "password123" | base64
# Résultat : cGFzc3dvcmQxMjM=

echo -n "admin" | base64
# Résultat : YWRtaW4=
```

**Décoder une valeur** :
```bash
echo "cGFzc3dvcmQxMjM=" | base64 -d
# Résultat : password123
```

### Méthode 1 : Depuis des valeurs littérales (recommandé)

La méthode la plus simple et la plus sûre (pas besoin d'encoder manuellement).

```bash
microk8s kubectl create secret generic mon-secret \
  --from-literal=username=admin \
  --from-literal=password=SuperSecret123 \
  --from-literal=api-key=sk_live_abc123xyz
```

Kubernetes encode automatiquement les valeurs en base64.

### Méthode 2 : Depuis des fichiers

Si vous avez des fichiers contenant les secrets (certificats, clés, etc.).

```bash
# Créer des fichiers
echo -n "admin" > username.txt
echo -n "SuperSecret123" > password.txt

# Créer le Secret depuis les fichiers
microk8s kubectl create secret generic mon-secret \
  --from-file=username=username.txt \
  --from-file=password=password.txt

# Nettoyer les fichiers temporaires
rm username.txt password.txt
```

### Méthode 3 : Depuis un fichier YAML

**Important** : Vous devez encoder manuellement les valeurs en base64.

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: db-credentials
  namespace: default
type: Opaque
data:
  # Valeurs encodées en base64
  username: YWRtaW4=                    # admin
  password: U3VwZXJTZWNyZXQxMjM=        # SuperSecret123
  db-host: cG9zdGdyZXMuZXhhbXBsZS5jb20= # postgres.example.com
```

Appliquer :
```bash
microk8s kubectl apply -f secret.yaml
```

### Méthode 4 : Depuis un fichier YAML avec stringData

**Astuce** : Utilisez `stringData` pour éviter l'encodage manuel !

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: db-credentials
  namespace: default
type: Opaque
stringData:                        # Pas besoin d'encoder !
  username: admin
  password: SuperSecret123
  db-host: postgres.example.com
```

Kubernetes convertira automatiquement en base64 lors de l'application.

**⚠️ Attention** : `stringData` est en clair dans le fichier YAML. Ne commitez **jamais** ce fichier dans Git !

### Méthode 5 : Secret pour registry Docker

Pour pull des images depuis un registry privé.

```bash
microk8s kubectl create secret docker-registry regcred \
  --docker-server=registry.example.com \
  --docker-username=myuser \
  --docker-password=mypassword \
  --docker-email=myemail@example.com
```

### Méthode 6 : Secret TLS

Pour les certificats SSL/TLS.

```bash
microk8s kubectl create secret tls mon-tls-secret \
  --cert=path/to/cert.pem \
  --key=path/to/key.pem
```

### Voir les Secrets

```bash
# Lister tous les Secrets
microk8s kubectl get secrets

# Voir les détails (les valeurs sont cachées)
microk8s kubectl describe secret mon-secret

# Voir le Secret complet (valeurs en base64)
microk8s kubectl get secret mon-secret -o yaml

# Décoder une valeur spécifique
microk8s kubectl get secret mon-secret -o jsonpath='{.data.password}' | base64 -d
```

## Utiliser un Secret dans un Pod

### 1. Variables d'environnement : valeur unique

Injecter une seule valeur du Secret dans une variable d'environnement.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: mon-app
spec:
  containers:
  - name: app
    image: mon-application:v1
    env:
    - name: DB_USERNAME          # Nom de la variable dans le conteneur
      valueFrom:
        secretKeyRef:
          name: db-credentials   # Nom du Secret
          key: username          # Clé dans le Secret
    - name: DB_PASSWORD
      valueFrom:
        secretKeyRef:
          name: db-credentials
          key: password
```

**Dans le conteneur** :
```bash
echo $DB_USERNAME
# admin

echo $DB_PASSWORD
# SuperSecret123
```

### 2. Variables d'environnement : toutes les clés

Injecter toutes les clés d'un Secret comme variables d'environnement.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: mon-app
spec:
  containers:
  - name: app
    image: mon-application:v1
    envFrom:
    - secretRef:
        name: db-credentials     # Toutes les clés du Secret
```

### 3. Volume : fichiers

Monter les clés du Secret comme des fichiers dans le conteneur.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: mon-app
spec:
  containers:
  - name: app
    image: mon-application:v1
    volumeMounts:
    - name: secret-volume
      mountPath: /etc/secrets    # Répertoire de montage
      readOnly: true             # Lecture seule (recommandé)
  volumes:
  - name: secret-volume
    secret:
      secretName: db-credentials
```

**Résultat** : Dans `/etc/secrets/`, chaque clé du Secret devient un fichier.

```
/etc/secrets/
  ├── username    (contenu: admin)
  ├── password    (contenu: SuperSecret123)
  └── db-host     (contenu: postgres.example.com)
```

**Lire depuis l'application** :
```python
# Python exemple
with open('/etc/secrets/username', 'r') as f:
    username = f.read()

with open('/etc/secrets/password', 'r') as f:
    password = f.read()
```

### 4. Volume : fichier unique avec subPath

Monter une clé spécifique comme un fichier unique.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: mon-app
spec:
  containers:
  - name: app
    image: mon-application:v1
    volumeMounts:
    - name: secret-volume
      mountPath: /etc/db-password.txt    # Fichier spécifique
      subPath: password                  # Clé du Secret
      readOnly: true
  volumes:
  - name: secret-volume
    secret:
      secretName: db-credentials
```

### 5. Volume : permissions personnalisées

Définir les permissions des fichiers secrets.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: mon-app
spec:
  containers:
  - name: app
    image: mon-application:v1
    volumeMounts:
    - name: secret-volume
      mountPath: /etc/secrets
      readOnly: true
  volumes:
  - name: secret-volume
    secret:
      secretName: db-credentials
      defaultMode: 0400          # Permissions: lecture seule pour le propriétaire
      items:
      - key: password
        path: db-password        # Nom de fichier personnalisé
        mode: 0400
```

### 6. ImagePullSecrets pour registries privés

Utiliser un Secret pour s'authentifier à un registry Docker privé.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: private-app
spec:
  containers:
  - name: app
    image: registry.example.com/mon-app:v1
  imagePullSecrets:              # Référence au Secret docker-registry
  - name: regcred
```

## Exemple complet : Application avec base de données

### 1. Créer le Secret pour la base de données

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: postgres-credentials
  namespace: production
type: Opaque
stringData:
  POSTGRES_USER: appuser
  POSTGRES_PASSWORD: "StrongP@ssw0rd2024"
  POSTGRES_DB: myapp_production
  POSTGRES_HOST: postgres.production.svc.cluster.local
  POSTGRES_PORT: "5432"
```

### 2. Deployment PostgreSQL utilisant le Secret

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: postgres
  namespace: production
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
        envFrom:
        - secretRef:
            name: postgres-credentials    # Toutes les variables d'env
        ports:
        - containerPort: 5432
```

### 3. Deployment Application utilisant le Secret

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: backend-api
  namespace: production
spec:
  replicas: 3
  selector:
    matchLabels:
      app: backend-api
  template:
    metadata:
      labels:
        app: backend-api
    spec:
      containers:
      - name: api
        image: mon-backend:v2.1.0
        env:
        - name: DATABASE_URL
          value: "postgresql://$(DB_USER):$(DB_PASSWORD)@$(DB_HOST):$(DB_PORT)/$(DB_NAME)"
        - name: DB_USER
          valueFrom:
            secretKeyRef:
              name: postgres-credentials
              key: POSTGRES_USER
        - name: DB_PASSWORD
          valueFrom:
            secretKeyRef:
              name: postgres-credentials
              key: POSTGRES_PASSWORD
        - name: DB_HOST
          valueFrom:
            secretKeyRef:
              name: postgres-credentials
              key: POSTGRES_HOST
        - name: DB_PORT
          valueFrom:
            secretKeyRef:
              name: postgres-credentials
              key: POSTGRES_PORT
        - name: DB_NAME
          valueFrom:
            secretKeyRef:
              name: postgres-credentials
              key: POSTGRES_DB
```

## Secrets avec certificats TLS

### Créer un Secret TLS

**Option 1 : Depuis des fichiers**
```bash
microk8s kubectl create secret tls my-tls-secret \
  --cert=certificate.crt \
  --key=private.key
```

**Option 2 : Depuis un YAML**
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: my-tls-secret
type: kubernetes.io/tls
data:
  tls.crt: LS0tLS1CRUdJTi... # Certificat encodé en base64
  tls.key: LS0tLS1CRUdJTi... # Clé privée encodée en base64
```

### Utiliser un Secret TLS dans un Ingress

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: mon-ingress
spec:
  tls:
  - hosts:
    - www.example.com
    secretName: my-tls-secret      # Référence au Secret TLS
  rules:
  - host: www.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: frontend-service
            port:
              number: 80
```

## Gestion et rotation des Secrets

### Mise à jour d'un Secret

**Méthode 1 : Éditer directement**
```bash
microk8s kubectl edit secret mon-secret
```

**Méthode 2 : Appliquer un nouveau fichier**
```bash
microk8s kubectl apply -f secret.yaml
```

**Méthode 3 : Patch**
```bash
# Encoder la nouvelle valeur
NEW_PASSWORD=$(echo -n "NewPassword456" | base64)

# Patcher le Secret
microk8s kubectl patch secret mon-secret \
  -p "{\"data\":{\"password\":\"$NEW_PASSWORD\"}}"
```

### Important : Redémarrer les Pods après modification

Comme pour les ConfigMaps, les Secrets ne se mettent pas à jour automatiquement dans les Pods existants.

**Variables d'environnement** : Jamais mises à jour (Pod doit être recréé)
**Volumes** : Mis à jour après quelques minutes, mais l'app doit recharger

```bash
# Redémarrer les Pods d'un Deployment
microk8s kubectl rollout restart deployment mon-app

# Ou forcer la suppression
microk8s kubectl delete pods -l app=mon-app
```

### Stratégie de rotation avec versioning

Plutôt que de modifier un Secret existant, créez un nouveau Secret versionné.

```yaml
---
# Ancien Secret (v1)
apiVersion: v1
kind: Secret
metadata:
  name: db-credentials-v1
stringData:
  password: OldPassword123
---
# Nouveau Secret (v2)
apiVersion: v1
kind: Secret
metadata:
  name: db-credentials-v2
stringData:
  password: NewPassword456
```

Puis mettez à jour le Deployment pour pointer vers `db-credentials-v2`.

**Avantages** :
- Rollback facile (revenir à v1)
- Pas de période d'incohérence
- Historique des credentials

## Secrets externes (solutions avancées)

Pour une sécurité maximale en production, utilisez des solutions de gestion de secrets externes.

### 1. HashiCorp Vault

Vault est une solution de gestion de secrets professionnelle.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: vault-app
  annotations:
    vault.hashicorp.com/agent-inject: "true"
    vault.hashicorp.com/role: "myapp"
    vault.hashicorp.com/agent-inject-secret-database: "secret/data/database"
spec:
  containers:
  - name: app
    image: mon-app:v1
```

### 2. External Secrets Operator

Synchronise automatiquement des secrets depuis des providers externes (AWS Secrets Manager, Azure Key Vault, etc.).

```yaml
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: db-credentials
spec:
  refreshInterval: 1h
  secretStoreRef:
    name: aws-secrets-manager
    kind: SecretStore
  target:
    name: db-credentials-k8s
  data:
  - secretKey: password
    remoteRef:
      key: prod/database/password
```

### 3. Sealed Secrets

Chiffre les Secrets pour les stocker en toute sécurité dans Git.

```bash
# Créer un SealedSecret (chiffré)
microk8s kubectl create secret generic mon-secret \
  --from-literal=password=SuperSecret123 \
  --dry-run=client -o yaml | \
  kubeseal -o yaml > sealed-secret.yaml

# Le fichier sealed-secret.yaml peut être committé dans Git
```

## Bonnes pratiques de sécurité

### 1. Ne jamais committer les Secrets dans Git

**✅ Bon** :
```bash
# .gitignore
secrets/
*.secret.yaml
*-secret.yaml
```

**❌ Mauvais** :
```bash
git add secret.yaml
git commit -m "Added database password"    # ❌ JAMAIS !
```

### 2. Utiliser RBAC pour limiter l'accès

Créez des rôles qui limitent qui peut lire les Secrets.

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: production
  name: secret-reader
rules:
- apiGroups: [""]
  resources: ["secrets"]
  resourceNames: ["db-credentials"]    # Seulement ce Secret spécifique
  verbs: ["get"]
```

### 3. Principe du moindre privilège

Un Pod ne devrait avoir accès qu'aux Secrets dont il a réellement besoin.

**✅ Bon** :
```yaml
# Pod A accède seulement à db-credentials
env:
- name: DB_PASSWORD
  valueFrom:
    secretKeyRef:
      name: db-credentials
      key: password
```

**❌ Mauvais** :
```yaml
# Pod A accède à TOUS les Secrets (trop permissif)
envFrom:
- secretRef:
    name: all-secrets
```

### 4. Utiliser readOnly pour les volumes

```yaml
volumeMounts:
- name: secret-volume
  mountPath: /etc/secrets
  readOnly: true              # ✅ Empêche la modification
```

### 5. Ne pas logger les Secrets

**❌ Mauvais** :
```python
password = os.getenv('DB_PASSWORD')
print(f"Connecting with password: {password}")    # ❌ Ne pas faire !
```

**✅ Bon** :
```python
password = os.getenv('DB_PASSWORD')
print("Connecting to database...")               # ✅ Pas de secret dans les logs
```

### 6. Utiliser des ServiceAccounts dédiés

Créez des ServiceAccounts spécifiques pour chaque application.

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: backend-api-sa
  namespace: production
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: backend-api
spec:
  template:
    spec:
      serviceAccountName: backend-api-sa    # ServiceAccount dédié
      containers:
      - name: api
        image: backend-api:v1
```

### 7. Activer l'encryption at rest dans etcd

Pour MicroK8s, activez le chiffrement des Secrets dans etcd :

```yaml
# /var/snap/microk8s/current/args/kube-apiserver
--encryption-provider-config=/path/to/encryption-config.yaml
```

**fichier : encryption-config.yaml**
```yaml
apiVersion: apiserver.config.k8s.io/v1
kind: EncryptionConfiguration
resources:
  - resources:
    - secrets
    providers:
    - aescbc:
        keys:
        - name: key1
          secret: <base64-encoded-32-byte-key>
    - identity: {}
```

### 8. Rotation régulière des Secrets

Planifiez la rotation périodique des credentials :
- Mots de passe : tous les 90 jours
- Clés API : selon les recommandations du provider
- Certificats : avant expiration

### 9. Audit des accès aux Secrets

Activez l'audit logging pour tracer les accès aux Secrets.

```yaml
# audit-policy.yaml
apiVersion: audit.k8s.io/v1
kind: Policy
rules:
- level: RequestResponse
  resources:
  - group: ""
    resources: ["secrets"]
```

### 10. Utiliser des solutions externes en production

Pour la production, envisagez :
- **HashiCorp Vault**
- **AWS Secrets Manager**
- **Azure Key Vault**
- **Google Secret Manager**
- **External Secrets Operator**

## Limites et considérations

### 1. Taille maximale

Comme les ConfigMaps, les Secrets ont une limite de **1 MiB**.

Pour des données plus volumineuses (gros certificats, fichiers) :
- Divisez en plusieurs Secrets
- Utilisez des volumes externes
- Stockez dans un service externe

### 2. Base64 n'est pas du chiffrement

**Important** : Base64 est un simple encodage, pas du chiffrement !

```bash
# Décoder est trivial
echo "cGFzc3dvcmQxMjM=" | base64 -d
# password123
```

Toute personne avec accès à Kubernetes peut décoder les Secrets.

### 3. Secrets visibles dans etcd par défaut

Sans encryption at rest, les Secrets sont stockés en clair dans etcd.

**Solution** : Activez l'encryption at rest (voir bonnes pratiques).

### 4. Propagation dans les logs et describe

Les Secrets n'apparaissent pas dans les logs ou `kubectl describe`, mais :

```bash
# Le Secret complet est visible avec -o yaml
microk8s kubectl get secret mon-secret -o yaml    # ⚠️ Visible !

# Les valeurs peuvent être décodées
microk8s kubectl get secret mon-secret -o jsonpath='{.data.password}' | base64 -d
```

### 5. Variables d'environnement dans les Pods

Les variables d'environnement peuvent être visibles via `/proc`:

```bash
# Dans le Pod
cat /proc/1/environ
# Affiche toutes les variables d'env, y compris les Secrets
```

**Plus sûr** : Utilisez des volumes plutôt que des variables d'environnement.

## Commandes utiles

### Gestion des Secrets

```bash
# Créer un Secret depuis des valeurs
microk8s kubectl create secret generic mon-secret \
  --from-literal=username=admin \
  --from-literal=password=secret123

# Créer depuis un fichier
microk8s kubectl create secret generic mon-secret \
  --from-file=ssh-privatekey=~/.ssh/id_rsa

# Créer un Secret Docker registry
microk8s kubectl create secret docker-registry regcred \
  --docker-server=registry.example.com \
  --docker-username=user \
  --docker-password=pass

# Créer un Secret TLS
microk8s kubectl create secret tls tls-secret \
  --cert=cert.pem \
  --key=key.pem

# Lister les Secrets
microk8s kubectl get secrets

# Voir les détails (valeurs cachées)
microk8s kubectl describe secret mon-secret

# Voir le Secret complet (base64)
microk8s kubectl get secret mon-secret -o yaml

# Décoder une valeur
microk8s kubectl get secret mon-secret -o jsonpath='{.data.password}' | base64 -d
```

### Modification

```bash
# Éditer un Secret
microk8s kubectl edit secret mon-secret

# Appliquer un fichier modifié
microk8s kubectl apply -f secret.yaml

# Supprimer un Secret
microk8s kubectl delete secret mon-secret
```

### Débogage

```bash
# Vérifier quels Pods utilisent un Secret
microk8s kubectl get pods -o json | \
  jq '.items[] | select(.spec.volumes[]?.secret.secretName=="mon-secret") | .metadata.name'

# Voir les variables d'env d'un Pod
microk8s kubectl exec mon-pod -- env | grep -i password

# Voir les fichiers secrets montés
microk8s kubectl exec mon-pod -- ls -la /etc/secrets
microk8s kubectl exec mon-pod -- cat /etc/secrets/password
```

## Dépannage

### Problème 1 : Secret non trouvé

**Symptôme** :
```
Error: secret "mon-secret" not found
```

**Vérifications** :
```bash
# Vérifier que le Secret existe
microk8s kubectl get secret mon-secret

# Vérifier le bon namespace
microk8s kubectl get secret mon-secret -n production

# Lister tous les Secrets
microk8s kubectl get secrets --all-namespaces | grep mon-secret
```

### Problème 2 : Pod en erreur à cause d'une clé manquante

**Symptôme** :
```
Pod cannot start: secret "mon-secret" does not contain key "api-key"
```

**Vérification** :
```bash
# Voir les clés disponibles dans le Secret
microk8s kubectl get secret mon-secret -o jsonpath='{.data}'
```

**Solution** : Ajouter la clé manquante au Secret.

### Problème 3 : Valeur décodée incorrecte

**Symptôme** : La valeur décodée contient des caractères étranges.

**Cause** : Mauvais encodage base64 (souvent un `\n` en trop).

```bash
# ❌ Mauvais (ajoute un newline)
echo "password123" | base64

# ✅ Bon (pas de newline)
echo -n "password123" | base64
```

### Problème 4 : Permission denied sur les fichiers secrets

**Symptôme** : L'application ne peut pas lire les fichiers secrets.

**Solution** : Ajuster les permissions du volume.

```yaml
volumes:
- name: secret-volume
  secret:
    secretName: mon-secret
    defaultMode: 0644    # rw-r--r--
```

### Problème 5 : Secret modifié mais Pods pas mis à jour

**Solution** : Redémarrer les Pods.

```bash
microk8s kubectl rollout restart deployment mon-app
```

## Secrets vs ConfigMaps : quand utiliser quoi ?

### Utilisez un Secret pour :

- ✅ Mots de passe
- ✅ Tokens d'API
- ✅ Clés privées (SSH, TLS)
- ✅ Certificats
- ✅ Credentials de base de données
- ✅ Clés de chiffrement
- ✅ Tokens OAuth
- ✅ Credentials Docker registry

### Utilisez un ConfigMap pour :

- ✅ URLs d'API
- ✅ Ports
- ✅ Feature flags
- ✅ Fichiers de configuration (nginx.conf, etc.)
- ✅ Variables d'environnement non sensibles
- ✅ Paramètres de l'application
- ✅ Timeouts, limites, etc.

## Résumé des points clés

- Les **Secrets** stockent des données sensibles de manière plus sécurisée que les ConfigMaps
- Par défaut, les Secrets sont **encodés en base64** (pas chiffrés)
- **Base64 n'est pas du chiffrement** : toute personne avec accès à K8s peut décoder
- Les Secrets s'utilisent comme les ConfigMaps : **variables d'environnement** ou **volumes**
- **Plusieurs types** : Opaque, TLS, Docker, SSH, etc.
- **Taille maximum** : 1 MiB
- Utilisez **RBAC** pour contrôler l'accès aux Secrets
- Pour la production, activez **l'encryption at rest** dans etcd
- Envisagez des solutions externes : **Vault**, **AWS Secrets Manager**, etc.
- **Ne jamais** committer des Secrets dans Git
- **Rotation régulière** des credentials recommandée

## Prochaines étapes

Maintenant que vous maîtrisez les Secrets, vous êtes prêt à approfondir :
- **Le stockage persistant** : comment gérer les volumes et les données qui doivent survivre au redémarrage des Pods
- **Les PersistentVolumes et PersistentVolumeClaims** : pour persister les données
- **Les StatefulSets** : pour déployer des applications avec état nécessitant des identités et du stockage stables

Les Secrets sont essentiels pour déployer des applications sécurisées en production. Combinés avec les bonnes pratiques de sécurité, ils constituent une base solide pour gérer les données sensibles dans Kubernetes !

⏭️ [Premiers Déploiements](/04-premiers-deploiements/README.md)
