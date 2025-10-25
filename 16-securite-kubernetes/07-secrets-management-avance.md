🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 16.7 Secrets Management Avancé

## Introduction

Les secrets (mots de passe, clés API, certificats, tokens) sont les données les plus sensibles de votre infrastructure. Une gestion inadéquate des secrets peut compromettre toute la sécurité de votre cluster. Cette section explore les solutions avancées pour gérer les secrets de manière sécurisée dans Kubernetes.

**Analogie simple :** Si votre cluster Kubernetes est une banque, les secrets sont les codes des coffres-forts. Vous ne voudriez pas écrire ces codes sur des post-it collés partout dans la banque, ni les laisser dans un tiroir non verrouillé. Le secrets management avancé, c'est avoir un système de coffres-forts ultra-sécurisés avec contrôle d'accès strict et audit.

## Le Problème : Kubernetes Secrets Natifs

### Limitations des Secrets Kubernetes de Base

Les Secrets Kubernetes natifs ont plusieurs problèmes :

```
┌─────────────────────────────────────────┐
│    Secret Kubernetes Natif              │
│                                         │
│  ⚠️ Base64 encodé (≠ chiffré)           │
│  ⚠️ Stocké en clair dans etcd           │
│  ⚠️ Visible dans les manifestes YAML    │
│  ⚠️ Pas d'audit des accès               │
│  ⚠️ Pas de rotation automatique         │
│  ⚠️ Doit être versionné avec le code    │
│  ⚠️ Difficile à partager entre équipes  │
└─────────────────────────────────────────┘
```

#### 1. Base64 n'est PAS du Chiffrement

```bash
# Créer un secret
kubectl create secret generic my-password --from-literal=password=SuperSecret123

# Le voir en base64
kubectl get secret my-password -o yaml
# data:
#   password: U3VwZXJTZWNyZXQxMjM=

# Décoder facilement
echo "U3VwZXJTZWNyZXQxMjM=" | base64 -d
# SuperSecret123
```

**⚠️ Base64 est un encodage, pas du chiffrement !** N'importe qui avec accès au cluster peut décoder.

#### 2. Stockage en Clair dans etcd

Par défaut, les secrets sont stockés **non chiffrés** dans etcd (la base de données de Kubernetes).

```
┌──────────────────────────┐
│         etcd             │
│                          │
│  secrets/default/my-pwd  │
│  {                       │
│    "password":           │
│    "SuperSecret123"      │  ← En clair !
│  }                       │
└──────────────────────────┘
```

**Risque :** Quiconque accède à etcd peut lire tous les secrets.

#### 3. Secrets dans Git

```yaml
# secret.yaml (dans Git)
apiVersion: v1
kind: Secret
metadata:
  name: database
stringData:
  password: MyDatabasePassword123  # ⚠️ En clair dans Git !
```

**Problèmes :**
- Historique Git conserve les secrets pour toujours
- Toute personne avec accès au repo voit les secrets
- Difficult de révoquer l'accès

#### 4. Pas de Rotation Automatique

```
Secret créé → 2020-01-01
              ↓
Toujours le même mot de passe → 2024-12-31
                                  ↓
                          ⚠️ Risque accru avec le temps
```

**Bonne pratique :** Rotation régulière (tous les 90 jours par exemple).

#### 5. Pas d'Audit

Kubernetes ne trace pas :
- Qui a lu quel secret ?
- Quand ?
- Depuis où ?

## Solutions Avancées

### Vue d'Ensemble

```
┌─────────────────────────────────────────────────┐
│  Solutions de Secrets Management                │
│                                                 │
│  1. Chiffrement etcd (natif K8s)                │
│     └─ Chiffre les secrets au repos             │
│                                                 │
│  2. Sealed Secrets (Bitnami)                    │
│     └─ Chiffre les secrets pour Git             │
│                                                 │
│  3. External Secrets Operator                   │
│     └─ Synchronise depuis sources externes      │
│                                                 │
│  4. HashiCorp Vault                             │
│     └─ Coffre-fort centralisé                   │
│                                                 │
│  5. Cloud Provider KMS                          │
│     └─ AWS Secrets Manager, GCP Secret Manager  │
└─────────────────────────────────────────────────┘
```

## Solution 1 : Chiffrement etcd

### Activer le Chiffrement des Secrets dans etcd

Cette solution chiffre les secrets **au repos** dans etcd.

#### Configuration

```yaml
# /etc/kubernetes/encryption-config.yaml
apiVersion: apiserver.config.k8s.io/v1
kind: EncryptionConfiguration
resources:
  - resources:
    - secrets
    providers:
    - aescbc:
        keys:
        - name: key1
          secret: c2VjcmV0IGtleSB0aGF0IGlzIDMyIGJ5dGVzIGxvbmc=  # Base64 d'une clé 32 bytes
    - identity: {}  # Fallback pour les secrets non chiffrés
```

#### Étapes d'Activation

1. **Générer une clé de chiffrement :**

```bash
# Générer une clé aléatoire de 32 bytes
head -c 32 /dev/urandom | base64
```

2. **Créer le fichier de configuration** (ci-dessus)

3. **Modifier la configuration de l'API Server :**

```yaml
# /etc/kubernetes/manifests/kube-apiserver.yaml
spec:
  containers:
  - command:
    - kube-apiserver
    - --encryption-provider-config=/etc/kubernetes/encryption-config.yaml
    volumeMounts:
    - name: encryption-config
      mountPath: /etc/kubernetes/encryption-config.yaml
      readOnly: true
  volumes:
  - name: encryption-config
    hostPath:
      path: /etc/kubernetes/encryption-config.yaml
      type: File
```

4. **Rechiffrer les secrets existants :**

```bash
# Forcer la réécriture de tous les secrets
kubectl get secrets --all-namespaces -o json | \
  kubectl replace -f -
```

### Avantages et Limites

**Avantages :**
- ✅ Natif Kubernetes
- ✅ Chiffrement au repos
- ✅ Pas d'outil externe

**Limites :**
- ⚠️ Toujours en base64 dans les manifestes
- ⚠️ Clé de chiffrement doit être sécurisée
- ⚠️ Pas de rotation automatique
- ⚠️ Complexe à configurer

### Dans MicroK8s

MicroK8s utilise Dqlite au lieu d'etcd. Le chiffrement peut être configuré mais nécessite des étapes spécifiques.

**Recommandation :** Pour MicroK8s, préférez les solutions externes (Sealed Secrets, Vault).

## Solution 2 : Sealed Secrets

### Concept

Sealed Secrets permet de chiffrer les secrets de manière asymétrique pour les stocker en toute sécurité dans Git.

```
┌──────────────────────────────────────────┐
│        Workflow Sealed Secrets           │
│                                          │
│  1. Secret en clair                      │
│     ↓                                    │
│  2. Chiffré avec clé publique            │
│     → SealedSecret                       │
│     ↓                                    │
│  3. Stocké dans Git (sécurisé)           │
│     ↓                                    │
│  4. Appliqué au cluster                  │
│     ↓                                    │
│  5. Controller déchiffre avec clé privée │
│     → Secret Kubernetes                  │
└──────────────────────────────────────────┘
```

**Principe :** Seul le cluster peut déchiffrer les SealedSecrets.

### Installation

```bash
# Installer le controller dans le cluster
kubectl apply -f https://github.com/bitnami-labs/sealed-secrets/releases/download/v0.24.0/controller.yaml

# Vérifier l'installation
kubectl get pods -n kube-system | grep sealed-secrets

# Installer kubeseal (outil CLI)
wget https://github.com/bitnami-labs/sealed-secrets/releases/download/v0.24.0/kubeseal-linux-amd64
chmod +x kubeseal-linux-amd64
sudo mv kubeseal-linux-amd64 /usr/local/bin/kubeseal
```

### Utilisation

#### Créer un SealedSecret

```bash
# Créer un secret (ne pas l'appliquer)
kubectl create secret generic my-secret \
  --from-literal=password=SuperSecret123 \
  --dry-run=client -o yaml > secret.yaml

# Sceller le secret
kubeseal -f secret.yaml -w sealed-secret.yaml

# Le fichier sealed-secret.yaml est maintenant sûr pour Git
cat sealed-secret.yaml
```

**Résultat :**
```yaml
apiVersion: bitnami.com/v1alpha1
kind: SealedSecret
metadata:
  name: my-secret
  namespace: default
spec:
  encryptedData:
    password: AgBxK8z7... (très long texte chiffré)
  template:
    metadata:
      name: my-secret
```

#### Appliquer au Cluster

```bash
# Appliquer le SealedSecret
kubectl apply -f sealed-secret.yaml

# Le controller crée automatiquement le Secret
kubectl get secret my-secret
```

#### Workflow Complet

```bash
# 1. Développeur crée un secret
echo -n "SuperSecret123" | kubectl create secret generic db-password \
  --from-file=password=/dev/stdin \
  --dry-run=client -o yaml > secret.yaml

# 2. Sceller le secret
kubeseal -f secret.yaml -w sealed-secret.yaml

# 3. Committer dans Git
git add sealed-secret.yaml
git commit -m "Add database password (sealed)"
git push

# 4. GitOps (ArgoCD/Flux) déploie automatiquement
# Ou manuellement :
kubectl apply -f sealed-secret.yaml

# 5. Le secret est disponible
kubectl get secret db-password -o jsonpath='{.data.password}' | base64 -d
```

### Rotation de la Clé

Par défaut, Sealed Secrets rotationne sa clé tous les 30 jours.

```bash
# Voir les clés
kubectl get secrets -n kube-system -l sealedsecrets.bitnami.com/sealed-secrets-key=active

# Forcer une rotation manuelle
kubectl delete secret -n kube-system -l sealedsecrets.bitnami.com/sealed-secrets-key=active
kubectl rollout restart deployment -n kube-system sealed-secrets-controller
```

**Important :** Après rotation, vous devez re-sceller tous vos secrets.

### Backup de la Clé

```bash
# Sauvegarder la clé privée (critique !)
kubectl get secret -n kube-system -l sealedsecrets.bitnami.com/sealed-secrets-key=active -o yaml > sealed-secrets-key.yaml

# Restaurer sur un nouveau cluster
kubectl apply -f sealed-secrets-key.yaml
```

### Avantages et Limites

**Avantages :**
- ✅ Secrets sécurisés dans Git
- ✅ GitOps friendly
- ✅ Simple à utiliser
- ✅ Pas de dépendance externe

**Limites :**
- ⚠️ Rotation manuelle nécessaire après rotation de clé
- ⚠️ La clé privée du cluster doit être sauvegardée
- ⚠️ Pas de rotation automatique des secrets eux-mêmes

## Solution 3 : External Secrets Operator

### Concept

External Secrets Operator (ESO) synchronise des secrets depuis des sources externes vers Kubernetes.

```
┌─────────────────────────────────────────────┐
│   Sources Externes                          │
│                                             │
│   • HashiCorp Vault                         │
│   • AWS Secrets Manager                     │
│   • Azure Key Vault                         │
│   • GCP Secret Manager                      │
│   • 1Password                               │
│   • etc.                                    │
└────────────┬────────────────────────────────┘
             │
             ▼
┌────────────────────────────────────────┐
│   External Secrets Operator            │
│                                        │
│   • Récupère les secrets               │
│   • Crée des Secrets Kubernetes        │
│   • Synchronise automatiquement        │
└────────────┬───────────────────────────┘
             │
             ▼
┌────────────────────────────────────────┐
│   Secrets Kubernetes                   │
│                                        │
│   Utilisables par les pods             │
└────────────────────────────────────────┘
```

### Installation

```bash
# Avec Helm
helm repo add external-secrets https://charts.external-secrets.io
helm install external-secrets external-secrets/external-secrets -n external-secrets-system --create-namespace

# Vérifier
kubectl get pods -n external-secrets-system
```

### Configuration avec Vault (Exemple)

#### 1. Créer un SecretStore

```yaml
apiVersion: external-secrets.io/v1beta1
kind: SecretStore
metadata:
  name: vault-backend
  namespace: default
spec:
  provider:
    vault:
      server: "https://vault.example.com"
      path: "secret"
      version: "v2"
      auth:
        kubernetes:
          mountPath: "kubernetes"
          role: "my-role"
```

#### 2. Créer un ExternalSecret

```yaml
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: database-credentials
  namespace: default
spec:
  refreshInterval: 1h  # Synchroniser toutes les heures
  secretStoreRef:
    name: vault-backend
    kind: SecretStore
  target:
    name: database-secret  # Nom du Secret K8s créé
    creationPolicy: Owner
  data:
  - secretKey: password      # Clé dans le Secret K8s
    remoteRef:
      key: database/prod     # Chemin dans Vault
      property: password     # Propriété dans Vault
  - secretKey: username
    remoteRef:
      key: database/prod
      property: username
```

#### 3. Le Secret est Créé Automatiquement

```bash
# ESO crée le secret
kubectl get secret database-secret

# Utiliser dans un pod
kubectl get secret database-secret -o jsonpath='{.data.password}' | base64 -d
```

### Configuration avec AWS Secrets Manager

```yaml
apiVersion: external-secrets.io/v1beta1
kind: SecretStore
metadata:
  name: aws-secrets
  namespace: default
spec:
  provider:
    aws:
      service: SecretsManager
      region: us-east-1
      auth:
        jwt:
          serviceAccountRef:
            name: external-secrets-sa
---
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: app-secrets
spec:
  refreshInterval: 15m
  secretStoreRef:
    name: aws-secrets
    kind: SecretStore
  target:
    name: app-secrets
  data:
  - secretKey: api-key
    remoteRef:
      key: prod/api-credentials
      property: api_key
```

### Avantages et Limites

**Avantages :**
- ✅ Source de vérité unique externe
- ✅ Rotation automatique possible
- ✅ Supporte de nombreux providers
- ✅ Audit centralisé
- ✅ Partage facile entre équipes

**Limites :**
- ⚠️ Dépendance à un service externe
- ⚠️ Configuration initiale complexe
- ⚠️ Coût potentiel du service externe

## Solution 4 : HashiCorp Vault

### Concept

Vault est un coffre-fort pour secrets avec fonctionnalités avancées.

```
┌───────────────────────────────────────┐
│        HashiCorp Vault                │
│                                       │
│  • Stockage centralisé                │
│  • Chiffrement                        │
│  • Rotation automatique               │
│  • Audit détaillé                     │
│  • Génération dynamique de secrets    │
│  • TTL (Time To Live)                 │
└───────────┬───────────────────────────┘
            │
            ▼
┌───────────────────────────────────────┐
│   Kubernetes (via Vault Agent)        │
│                                       │
│   Secrets injectés automatiquement    │
│   dans les pods                       │
└───────────────────────────────────────┘
```

### Installation de Vault

#### Avec Helm

```bash
# Ajouter le repo Helm
helm repo add hashicorp https://helm.releases.hashicorp.com

# Installer Vault
helm install vault hashicorp/vault \
  --set "server.dev.enabled=true" \
  --namespace vault \
  --create-namespace

# Vérifier
kubectl get pods -n vault
```

#### Mode Développement (⚠️ Pas pour la production)

```bash
# Vault en mode dev
kubectl exec -it vault-0 -n vault -- /bin/sh

# À l'intérieur du pod
vault status
vault secrets enable -path=secret kv-v2
vault kv put secret/database/config username="admin" password="SuperSecret123"
```

### Intégration avec Kubernetes

#### Méthode 1 : Vault Agent Injector

**Annotation dans le pod :**

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: app
  annotations:
    vault.hashicorp.com/agent-inject: "true"
    vault.hashicorp.com/role: "myapp"
    vault.hashicorp.com/agent-inject-secret-database: "secret/data/database/config"
    vault.hashicorp.com/agent-inject-template-database: |
      {{- with secret "secret/data/database/config" -}}
      username={{ .Data.data.username }}
      password={{ .Data.data.password }}
      {{- end }}
spec:
  serviceAccountName: app
  containers:
  - name: app
    image: my-app:latest
    command:
    - sh
    - -c
    - |
      cat /vault/secrets/database
      # Le secret est disponible dans /vault/secrets/database
```

**Configuration Vault :**

```bash
# Activer Kubernetes auth
vault auth enable kubernetes

# Configurer
vault write auth/kubernetes/config \
  kubernetes_host="https://kubernetes.default.svc"

# Créer une policy
vault policy write myapp-policy - <<EOF
path "secret/data/database/config" {
  capabilities = ["read"]
}
EOF

# Créer un rôle
vault write auth/kubernetes/role/myapp \
  bound_service_account_names=app \
  bound_service_account_namespaces=default \
  policies=myapp-policy \
  ttl=24h
```

#### Méthode 2 : Vault CSI Provider

Monter les secrets comme volumes :

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: app
spec:
  serviceAccountName: app
  containers:
  - name: app
    image: my-app:latest
    volumeMounts:
    - name: secrets
      mountPath: /mnt/secrets
      readOnly: true
  volumes:
  - name: secrets
    csi:
      driver: secrets-store.csi.k8s.io
      readOnly: true
      volumeAttributes:
        secretProviderClass: vault-secrets
```

```yaml
apiVersion: secrets-store.csi.x-k8s.io/v1
kind: SecretProviderClass
metadata:
  name: vault-secrets
spec:
  provider: vault
  parameters:
    vaultAddress: "http://vault.vault.svc:8200"
    roleName: "myapp"
    objects: |
      - objectName: "database-password"
        secretPath: "secret/data/database/config"
        secretKey: "password"
```

### Secrets Dynamiques

Vault peut générer des secrets à la demande :

```bash
# Activer le moteur de base de données
vault secrets enable database

# Configurer PostgreSQL
vault write database/config/postgresql \
  plugin_name=postgresql-database-plugin \
  allowed_roles="app-role" \
  connection_url="postgresql://{{username}}:{{password}}@postgres:5432/mydb" \
  username="vault" \
  password="vault-password"

# Créer un rôle avec TTL
vault write database/roles/app-role \
  db_name=postgresql \
  creation_statements="CREATE ROLE \"{{name}}\" WITH LOGIN PASSWORD '{{password}}' VALID UNTIL '{{expiration}}';" \
  default_ttl="1h" \
  max_ttl="24h"

# Générer des credentials
vault read database/creds/app-role
# Key                Value
# ---                -----
# lease_id           database/creds/app-role/abc123
# lease_duration     1h
# username           v-app-role-xyz789
# password           A1b2C3d4E5f6
```

**Avantage :** Credentials uniques par pod, rotation automatique, révocation instantanée.

### Avantages et Limites

**Avantages :**
- ✅ Solution complète et mature
- ✅ Génération dynamique de secrets
- ✅ Rotation automatique
- ✅ Audit détaillé
- ✅ TTL et révocation
- ✅ Haute disponibilité possible

**Limites :**
- ⚠️ Complexité de déploiement et gestion
- ⚠️ Nécessite expertise
- ⚠️ Infrastructure supplémentaire
- ⚠️ Coût de licence (version enterprise)

## Solution 5 : Cloud Provider KMS

### AWS Secrets Manager

```yaml
apiVersion: external-secrets.io/v1beta1
kind: SecretStore
metadata:
  name: aws-secrets-manager
spec:
  provider:
    aws:
      service: SecretsManager
      region: us-east-1
      auth:
        jwt:
          serviceAccountRef:
            name: external-secrets
---
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: app-config
spec:
  refreshInterval: 1h
  secretStoreRef:
    name: aws-secrets-manager
    kind: SecretStore
  target:
    name: app-config
  dataFrom:
  - extract:
      key: prod/app/config  # ARN ou nom du secret
```

**Avantages :**
- ✅ Intégration native AWS
- ✅ Rotation automatique
- ✅ Chiffrement KMS
- ✅ Audit CloudTrail

### Google Secret Manager

```yaml
apiVersion: external-secrets.io/v1beta1
kind: SecretStore
metadata:
  name: gcp-secrets
spec:
  provider:
    gcpsm:
      projectID: "my-project"
      auth:
        workloadIdentity:
          clusterLocation: us-central1
          clusterName: my-cluster
          serviceAccountRef:
            name: external-secrets
---
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: app-secrets
spec:
  refreshInterval: 15m
  secretStoreRef:
    name: gcp-secrets
  target:
    name: app-secrets
  data:
  - secretKey: api-key
    remoteRef:
      key: projects/123456789/secrets/api-key/versions/latest
```

### Azure Key Vault

```yaml
apiVersion: external-secrets.io/v1beta1
kind: SecretStore
metadata:
  name: azure-keyvault
spec:
  provider:
    azurekv:
      vaultUrl: "https://my-vault.vault.azure.net"
      authType: ManagedIdentity
      identityId: "/subscriptions/.../resourceGroups/.../providers/Microsoft.ManagedIdentity/userAssignedIdentities/..."
```

## Bonnes Pratiques

### 1. Ne Jamais Committer de Secrets en Clair

```bash
# ❌ Mauvais
git add secret.yaml  # Contient des secrets en clair

# ✅ Bon
git add sealed-secret.yaml  # Secret chiffré
# Ou
git add external-secret.yaml  # Référence externe
```

**Outil de prévention :**

```bash
# Installer git-secrets
git clone https://github.com/awslabs/git-secrets
cd git-secrets
sudo make install

# Configurer dans votre repo
cd /path/to/repo
git secrets --install
git secrets --register-aws
```

### 2. Rotation Régulière

```yaml
# Politique de rotation recommandée
Secrets critiques (DB, API keys) : 90 jours
Secrets sensibles (tokens) : 180 jours
Certificats : Avant expiration
```

**Automatisation avec Vault :**

```bash
# Secret avec TTL
vault kv put -mount=secret database/prod \
  password="SuperSecret123" \
  ttl=90d
```

### 3. Principe du Moindre Privilège

```yaml
# ❌ Mauvais : Tous les secrets dans un seul Secret
apiVersion: v1
kind: Secret
metadata:
  name: all-secrets
data:
  db-password: ...
  api-key: ...
  admin-token: ...

# ✅ Bon : Un Secret par service/composant
apiVersion: v1
kind: Secret
metadata:
  name: database-credentials
data:
  password: ...
---
apiVersion: v1
kind: Secret
metadata:
  name: api-credentials
data:
  api-key: ...
```

**Avec RBAC :**

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: app-role
rules:
- apiGroups: [""]
  resources: ["secrets"]
  resourceNames: ["database-credentials"]  # Seulement ce secret
  verbs: ["get"]
```

### 4. Audit et Monitoring

```yaml
# Activer l'audit des accès aux secrets
apiVersion: audit.k8s.io/v1
kind: Policy
rules:
- level: RequestResponse
  resources:
  - group: ""
    resources: ["secrets"]
  verbs: ["get", "list", "watch"]
```

**Avec Vault :**

```bash
# Activer l'audit
vault audit enable file file_path=/vault/audit.log

# Voir les accès
vault audit list
```

### 5. Séparer les Environnements

```
┌─────────────────────────────────────┐
│  Dev                                │
│  └─ Secrets de dev (moins sensibles)│
│                                     │
│  Staging                            │
│  └─ Secrets de staging              │
│                                     │
│  Production                         │
│  └─ Secrets de prod (haute sécurité)│
└─────────────────────────────────────┘
```

**Avec namespaces :**

```bash
# Secrets dev
kubectl create secret generic db-password \
  --from-literal=password=dev123 \
  -n dev

# Secrets prod (plus sécurisé)
kubectl create secret generic db-password \
  --from-literal=password=$(vault kv get -field=password secret/prod/database) \
  -n production
```

### 6. Chiffrement en Transit

```yaml
# Toujours utiliser TLS/HTTPS
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
data:
  DATABASE_URL: "postgresql://user:pass@postgres.example.com:5432/db?sslmode=require"
  #                                                                    ^^^^^^^^^^^^^^^^
  #                                                                    TLS activé
```

### 7. Limiter l'Accès aux Secrets

```yaml
# ServiceAccount dédié
apiVersion: v1
kind: ServiceAccount
metadata:
  name: app-sa
  namespace: production
---
# Role limité
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: secret-reader
  namespace: production
rules:
- apiGroups: [""]
  resources: ["secrets"]
  resourceNames: ["app-secret"]  # Uniquement ce secret
  verbs: ["get"]
---
# RoleBinding
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: app-secret-binding
  namespace: production
subjects:
- kind: ServiceAccount
  name: app-sa
roleRef:
  kind: Role
  name: secret-reader
  apiGroup: rbac.authorization.k8s.io
```

### 8. Scanner les Secrets dans les Images

```bash
# Avec Trivy
trivy image --scanners secret my-app:latest

# Avec truffleHog
docker run --rm -it trufflesecurity/trufflehog:latest \
  docker --image my-app:latest
```

### 9. Utiliser des Init Containers

Charger les secrets avant le démarrage de l'application :

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: app
spec:
  initContainers:
  - name: secret-loader
    image: vault:latest
    command:
    - sh
    - -c
    - |
      # Récupérer le secret depuis Vault
      vault kv get -field=password secret/database > /secrets/db-password
    volumeMounts:
    - name: secrets
      mountPath: /secrets
  containers:
  - name: app
    image: my-app:latest
    volumeMounts:
    - name: secrets
      mountPath: /secrets
      readOnly: true
  volumes:
  - name: secrets
    emptyDir: {}
```

### 10. Documentation

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: api-credentials
  annotations:
    description: "Credentials for external API"
    owner: "team-backend"
    rotation-schedule: "Every 90 days"
    last-rotation: "2024-01-15"
    next-rotation: "2024-04-15"
    source: "HashiCorp Vault: secret/prod/api"
```

## Comparaison des Solutions

| Solution | Complexité | Coût | Rotation Auto | GitOps | Audit | Recommandé Pour |
|----------|------------|------|---------------|--------|-------|-----------------|
| **etcd Encryption** | Moyenne | Gratuit | ❌ | ❌ | ❌ | Amélioration de base |
| **Sealed Secrets** | Faible | Gratuit | ❌ | ✅ | ❌ | Petites équipes, GitOps |
| **External Secrets** | Moyenne | Variable | ✅ | ✅ | ✅ | Intégration cloud |
| **Vault** | Élevée | Gratuit/Payant | ✅ | ⚠️ | ✅ | Grandes organisations |
| **Cloud KMS** | Faible | Payant | ✅ | ✅ | ✅ | Si déjà sur le cloud |

## Implémentation dans MicroK8s

### Sealed Secrets (Recommandé pour Commencer)

```bash
# Installation
kubectl apply -f https://github.com/bitnami-labs/sealed-secrets/releases/download/v0.24.0/controller.yaml

# Vérification
microk8s kubectl get pods -n kube-system | grep sealed-secrets

# Installer kubeseal
wget https://github.com/bitnami-labs/sealed-secrets/releases/download/v0.24.0/kubeseal-linux-amd64
chmod +x kubeseal-linux-amd64
sudo mv kubeseal-linux-amd64 /usr/local/bin/kubeseal

# Utilisation
echo -n "SuperSecret123" | kubectl create secret generic my-secret \
  --from-file=password=/dev/stdin \
  --dry-run=client -o yaml | \
  kubeseal -o yaml > sealed-secret.yaml

microk8s kubectl apply -f sealed-secret.yaml
```

### External Secrets Operator

```bash
# Installer avec Helm
microk8s enable helm3

# Ajouter le repo
microk8s helm3 repo add external-secrets https://charts.external-secrets.io

# Installer
microk8s helm3 install external-secrets \
  external-secrets/external-secrets \
  -n external-secrets-system \
  --create-namespace

# Vérifier
microk8s kubectl get pods -n external-secrets-system
```

### Vault en Mode Dev

```bash
# Installer Vault
microk8s helm3 repo add hashicorp https://helm.releases.hashicorp.com

microk8s helm3 install vault hashicorp/vault \
  --set "server.dev.enabled=true" \
  --namespace vault \
  --create-namespace

# Accéder à Vault
microk8s kubectl exec -it vault-0 -n vault -- /bin/sh
vault status
```

## Workflow Recommandé

### Pour une Petite Équipe

```
1. Développement local
   └─ Secrets en variables d'environnement locales

2. Git
   └─ Sealed Secrets (chiffrés)

3. CI/CD
   └─ Décrypte et déploie les Sealed Secrets

4. Cluster
   └─ Controller Sealed Secrets crée les Secrets K8s
```

### Pour une Grande Organisation

```
1. Développement local
   └─ Variables d'environnement ou Vault dev

2. Git
   └─ ExternalSecrets manifests (pas de secrets)

3. Vault / Cloud KMS
   └─ Source de vérité centralisée

4. Cluster
   └─ External Secrets Operator synchronise depuis Vault

5. Applications
   └─ Utilisent les Secrets K8s créés automatiquement
```

## Checklist Secrets Management

Avant de déployer en production :

- [ ] Secrets jamais en clair dans Git
- [ ] Solution de chiffrement en place (Sealed Secrets minimum)
- [ ] Rotation des secrets planifiée (90-180 jours)
- [ ] RBAC stricte sur les secrets
- [ ] Audit des accès aux secrets activé
- [ ] Secrets séparés par environnement
- [ ] TLS/HTTPS partout
- [ ] Scanner les images pour secrets exposés
- [ ] Backup des clés de chiffrement
- [ ] Documentation des secrets (sans les valeurs)
- [ ] Plan de révocation en cas de compromission
- [ ] Formation de l'équipe

## Scénario de Compromission

### Si un Secret est Compromis

```
1. IDENTIFICATION
   └─ Quel secret ? Quelle portée ?

2. RÉVOCATION IMMÉDIATE
   └─ Révoquer le secret compromis
   └─ Générer un nouveau secret

3. ROTATION
   └─ Mettre à jour dans Vault/KMS
   └─ External Secrets synchronise automatiquement
   └─ Ou redéployer les Sealed Secrets

4. INVESTIGATION
   └─ Comment le secret a-t-il été compromis ?
   └─ Audit logs pour voir les accès

5. AMÉLIORATION
   └─ Corriger la faille
   └─ Améliorer les processus
```

**Exemple avec Vault :**

```bash
# Révoquer le lease d'un secret dynamique
vault lease revoke database/creds/app-role/abc123

# Regénérer un nouveau secret
vault kv put secret/database/prod password="NewSuperSecret456"

# External Secrets synchronise automatiquement
# (selon refreshInterval configuré)
```

## Conclusion

Le secrets management est crucial pour la sécurité :

1. **Ne jamais** stocker de secrets en clair dans Git
2. **Chiffrer** les secrets au repos et en transit
3. **Rotate** régulièrement (90-180 jours)
4. **Auditer** les accès aux secrets
5. **Limiter** l'accès selon le principe du moindre privilège

**Formule Secrets Management :**
```
Sécurité = Chiffrement + Rotation + Audit + Moindre Privilège
```

**Points Clés à Retenir :**

- Base64 ≠ Chiffrement
- Kubernetes Secrets natifs ont des limitations importantes
- **Sealed Secrets** : Solution simple pour GitOps
- **External Secrets** : Intégration avec sources externes (Vault, AWS, etc.)
- **Vault** : Solution complète pour grandes organisations
- Rotation automatique quand possible
- RBAC stricte sur les secrets
- Audit de tous les accès
- Séparer dev/staging/production
- Scanner les images pour secrets exposés

**Recommandations par Contexte :**

- **Lab personnel / Apprentissage** : Sealed Secrets
- **Petite équipe / Startup** : Sealed Secrets + External Secrets
- **Moyenne entreprise** : External Secrets + Cloud KMS
- **Grande entreprise** : HashiCorp Vault + External Secrets

**Prochaines Étapes :**

Dans les sections suivantes, nous explorerons :
- **16.8** : Audit logging
- **16.9** : Bonnes pratiques de sécurité
- **16.10** : Checklist de sécurité complète

Le secrets management, combiné avec toutes les autres couches de sécurité (RBAC, Network Policies, Pod Security, Scan d'images), complète votre stratégie de défense en profondeur pour Kubernetes.

⏭️ [Audit logging](/16-securite-kubernetes/08-audit-logging.md)
