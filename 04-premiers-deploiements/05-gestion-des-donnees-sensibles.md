🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 4.5 Gestion des Données Sensibles

## Introduction

Dans la section précédente, nous avons appris à utiliser les variables d'environnement. Cependant, nous avons volontairement évité un sujet crucial : **comment gérer les informations sensibles** comme les mots de passe, les clés API, les certificats, etc.

Mettre des secrets directement dans les manifestes YAML est une **mauvaise pratique dangereuse** qui expose vos informations sensibles. Kubernetes propose une solution : les **Secrets**.

Dans cette section, nous allons apprendre à :
- Comprendre ce qu'est un Secret et pourquoi l'utiliser
- Créer et gérer des Secrets
- Utiliser les Secrets dans vos applications
- Appliquer les bonnes pratiques de sécurité

## Qu'est-ce qu'un Secret Kubernetes ?

Un **Secret** est un objet Kubernetes conçu pour stocker des informations sensibles comme :
- Mots de passe
- Tokens d'authentification
- Clés SSH
- Certificats TLS
- Clés API
- Credentials de bases de données

**Analogie :** Un Secret est comme un coffre-fort. Au lieu de laisser vos objets de valeur (mots de passe) éparpillés dans vos documents (manifestes YAML), vous les mettez dans un coffre sécurisé.

### Différence entre Secret et ConfigMap

| Aspect | ConfigMap | Secret |
|--------|-----------|--------|
| **Usage** | Configuration non sensible | Données sensibles |
| **Stockage** | Texte brut | Base64 encodé (dans etcd) |
| **Sécurité** | Aucune protection spéciale | Chiffrement optionnel au repos |
| **Visibilité** | Lisible facilement | Masqué dans certaines commandes |
| **Exemple** | Fichiers de config, URLs | Passwords, API keys |

**⚠️ Important :** Les Secrets ne sont PAS chiffrés par défaut dans etcd (la base de données de Kubernetes). Ils sont encodés en base64, ce qui n'est PAS du chiffrement. Nous verrons comment améliorer la sécurité plus loin.

## Pourquoi Utiliser des Secrets ?

### ❌ Le problème : Hardcoder les secrets

**Mauvaise pratique :**
```yaml
env:
- name: DATABASE_PASSWORD
  value: "super_secret_123"  # DANGER !
- name: API_KEY
  value: "sk-abc123xyz789"   # DANGER !
```

**Problèmes :**
- 🚨 Le secret est visible dans le fichier YAML
- 🚨 Le secret est visible dans Git si vous versionnez vos manifestes
- 🚨 Impossible de changer le secret sans modifier le code
- 🚨 Tous ceux qui ont accès au repository peuvent voir les secrets
- 🚨 Historique Git conserve les secrets même après suppression

### ✅ La solution : Utiliser des Secrets

```yaml
env:
- name: DATABASE_PASSWORD
  valueFrom:
    secretKeyRef:
      name: db-credentials
      key: password
```

**Avantages :**
- ✅ Le secret est séparé du code
- ✅ Gestion des accès via RBAC
- ✅ Peut être chiffré au repos
- ✅ Changement du secret sans modifier les manifestes
- ✅ Audit des accès possible

## Types de Secrets

Kubernetes supporte plusieurs types de Secrets selon leur usage :

| Type | Usage | Exemple |
|------|-------|---------|
| `Opaque` | Secrets génériques (par défaut) | Passwords, API keys |
| `kubernetes.io/service-account-token` | Token de compte de service | Authentification interne |
| `kubernetes.io/dockerconfigjson` | Credentials Docker registry | Pull d'images privées |
| `kubernetes.io/tls` | Certificats TLS | HTTPS, mTLS |
| `kubernetes.io/ssh-auth` | Clés SSH | Authentification SSH |
| `kubernetes.io/basic-auth` | Basic auth | Username/password |

Le type le plus courant est **Opaque**, que nous utiliserons principalement.

## Créer un Secret

Il existe plusieurs méthodes pour créer un Secret. Voyons les principales.

### Méthode 1 : Depuis la ligne de commande (Littéral)

C'est la méthode la plus simple pour créer un Secret avec quelques valeurs :

```bash
# Créer un Secret avec des valeurs littérales
microk8s kubectl create secret generic db-credentials \
  --from-literal=username=admin \
  --from-literal=password=super_secret_123 \
  --from-literal=database=myapp_db

# Vérifier
microk8s kubectl get secret db-credentials

# Sortie :
# NAME             TYPE     DATA   AGE
# db-credentials   Opaque   3      10s
```

**Décortiquons la commande :**
- `create secret generic` : Crée un Secret de type Opaque
- `db-credentials` : Nom du Secret
- `--from-literal=key=value` : Définit une paire clé-valeur

**Avantage :** Rapide et simple
**Inconvénient :** Le secret apparaît dans l'historique du shell

### Méthode 2 : Depuis des fichiers

Pour des secrets plus complexes (certificats, clés, fichiers de config) :

```bash
# Créer des fichiers avec les secrets
echo -n 'admin' > username.txt
echo -n 'super_secret_123' > password.txt

# Créer le Secret depuis les fichiers
microk8s kubectl create secret generic db-credentials \
  --from-file=username=username.txt \
  --from-file=password=password.txt

# Nettoyer les fichiers
rm username.txt password.txt
```

**Avantage :** Pas de secret dans l'historique shell
**Inconvénient :** Nécessite de créer et supprimer des fichiers

### Méthode 3 : Depuis un manifeste YAML (avec base64)

Vous pouvez créer un Secret via un manifeste YAML, mais les valeurs doivent être encodées en base64 :

```bash
# Encoder les valeurs en base64
echo -n 'admin' | base64
# Sortie : YWRtaW4=

echo -n 'super_secret_123' | base64
# Sortie : c3VwZXJfc2VjcmV0XzEyMw==
```

Créez un fichier `db-secret.yaml` :

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: db-credentials
  namespace: default
type: Opaque
data:
  username: YWRtaW4=
  password: c3VwZXJfc2VjcmV0XzEyMw==
  database: bXlhcHBfZGI=
```

```bash
# Créer le Secret
microk8s kubectl apply -f db-secret.yaml
```

**⚠️ Attention :** L'encodage base64 N'EST PAS du chiffrement ! N'importe qui peut décoder :

```bash
echo 'YWRtaW4=' | base64 -d
# Sortie : admin
```

**Avantage :** Déclaratif, peut être versionné (avec précautions)
**Inconvénient :** Base64 n'est pas du chiffrement

### Méthode 4 : Manifeste YAML avec stringData (Recommandé pour le développement)

Pour éviter d'encoder manuellement en base64, vous pouvez utiliser `stringData` :

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: db-credentials
  namespace: default
type: Opaque
stringData:
  username: admin
  password: super_secret_123
  database: myapp_db
```

```bash
# Créer le Secret
microk8s kubectl apply -f db-secret.yaml
```

**Note :** Kubernetes encode automatiquement les valeurs en base64. Après création, si vous récupérez le Secret, vous verrez les valeurs dans `data` (encodé), pas `stringData`.

**⚠️ Important :** Ne versionnez PAS ce fichier dans Git ! Ajoutez-le à `.gitignore`.

### Comparaison des méthodes

| Méthode | Avantages | Inconvénients | Usage |
|---------|-----------|---------------|-------|
| `--from-literal` | Rapide | Historique shell | Tests rapides |
| `--from-file` | Pas d'historique | Fichiers à gérer | Certificats, clés |
| YAML + `data` | Déclaratif | Encodage manuel | CI/CD automatisé |
| YAML + `stringData` | Facile à lire | Ne pas versionner | Dev local |

## Utiliser un Secret dans un Pod

### Méthode 1 : Comme variables d'environnement

C'est la méthode la plus courante :

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: app-with-secrets
spec:
  containers:
  - name: app
    image: myapp:latest
    env:
    # Référencer une clé spécifique du Secret
    - name: DB_USERNAME
      valueFrom:
        secretKeyRef:
          name: db-credentials    # Nom du Secret
          key: username           # Clé dans le Secret

    - name: DB_PASSWORD
      valueFrom:
        secretKeyRef:
          name: db-credentials
          key: password

    - name: DB_NAME
      valueFrom:
        secretKeyRef:
          name: db-credentials
          key: database
```

**Nouveaux éléments :**

```yaml
valueFrom:
  secretKeyRef:      # Référence à un Secret
    name: db-credentials
    key: username
```

**Fonctionnement :**
1. Kubernetes lit le Secret `db-credentials`
2. Extrait la valeur de la clé `username`
3. La décode (de base64 vers texte)
4. L'injecte comme variable d'environnement `DB_USERNAME`

### Importer toutes les clés d'un Secret

Si vous voulez importer toutes les clés d'un Secret en une fois :

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: app-all-secrets
spec:
  containers:
  - name: app
    image: myapp:latest
    envFrom:
    - secretRef:
        name: db-credentials
```

**Résultat :** Toutes les clés du Secret deviennent des variables d'environnement :
- `username` → Variable `username`
- `password` → Variable `password`
- `database` → Variable `database`

**⚠️ Attention :** Les noms de clés doivent être des noms de variables valides (pas de caractères spéciaux).

### Méthode 2 : Comme volumes (fichiers)

Pour des secrets plus complexes (certificats, fichiers de configuration) ou pour une meilleure sécurité, montez le Secret comme volume :

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: app-with-secret-volume
spec:
  containers:
  - name: app
    image: myapp:latest
    volumeMounts:
    - name: secret-volume
      mountPath: /etc/secrets      # Répertoire où monter le Secret
      readOnly: true               # Lecture seule

  volumes:
  - name: secret-volume
    secret:
      secretName: db-credentials
```

**Résultat :** Chaque clé du Secret devient un fichier :
```
/etc/secrets/
├── username  (contenu : admin)
├── password  (contenu : super_secret_123)
└── database  (contenu : myapp_db)
```

**Lecture dans l'application :**
```python
# Python
with open('/etc/secrets/username', 'r') as f:
    username = f.read()

with open('/etc/secrets/password', 'r') as f:
    password = f.read()
```

```javascript
// Node.js
const fs = require('fs');
const username = fs.readFileSync('/etc/secrets/username', 'utf8');
const password = fs.readFileSync('/etc/secrets/password', 'utf8');
```

### Monter des clés spécifiques

Vous pouvez choisir quelles clés monter et sous quels noms :

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: app-selective-secrets
spec:
  containers:
  - name: app
    image: myapp:latest
    volumeMounts:
    - name: secret-volume
      mountPath: /etc/secrets
      readOnly: true

  volumes:
  - name: secret-volume
    secret:
      secretName: db-credentials
      items:
      - key: password           # Clé dans le Secret
        path: db-password.txt   # Nom du fichier créé
      - key: username
        path: db-user.txt
```

**Résultat :**
```
/etc/secrets/
├── db-password.txt  (contenu : super_secret_123)
└── db-user.txt      (contenu : admin)
```

### Définir les permissions des fichiers

```yaml
volumes:
- name: secret-volume
  secret:
    secretName: db-credentials
    defaultMode: 0400  # Lecture seule pour le propriétaire (r--------)
```

**Valeurs courantes :**
- `0400` : Lecture seule pour le propriétaire (recommandé)
- `0440` : Lecture pour propriétaire et groupe
- `0444` : Lecture pour tous

## Exemple Complet : Application avec Base de Données

Voici un exemple réaliste d'une application web connectée à une base de données :

### 1. Créer le Secret

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: webapp-secrets
  namespace: default
type: Opaque
stringData:
  # Base de données
  db-username: webapp_user
  db-password: "P@ssw0rd!2024"
  db-host: postgres.default.svc.cluster.local
  db-port: "5432"
  db-name: webapp_production

  # API externe
  stripe-api-key: sk_live_abc123xyz789
  sendgrid-api-key: SG.abc123xyz789

  # JWT
  jwt-secret: my_super_secret_jwt_key_2024
```

```bash
microk8s kubectl apply -f webapp-secrets.yaml
```

### 2. Créer le Deployment

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: webapp
  namespace: default
spec:
  replicas: 3
  selector:
    matchLabels:
      app: webapp
  template:
    metadata:
      labels:
        app: webapp
    spec:
      containers:
      - name: app
        image: mywebapp:1.0.0
        ports:
        - containerPort: 8080

        # Variables d'environnement depuis le Secret
        env:
        # Base de données
        - name: DB_USERNAME
          valueFrom:
            secretKeyRef:
              name: webapp-secrets
              key: db-username

        - name: DB_PASSWORD
          valueFrom:
            secretKeyRef:
              name: webapp-secrets
              key: db-password

        - name: DB_HOST
          valueFrom:
            secretKeyRef:
              name: webapp-secrets
              key: db-host

        - name: DB_PORT
          valueFrom:
            secretKeyRef:
              name: webapp-secrets
              key: db-port

        - name: DB_NAME
          valueFrom:
            secretKeyRef:
              name: webapp-secrets
              key: db-name

        # APIs externes
        - name: STRIPE_API_KEY
          valueFrom:
            secretKeyRef:
              name: webapp-secrets
              key: stripe-api-key

        - name: SENDGRID_API_KEY
          valueFrom:
            secretKeyRef:
              name: webapp-secrets
              key: sendgrid-api-key

        - name: JWT_SECRET
          valueFrom:
            secretKeyRef:
              name: webapp-secrets
              key: jwt-secret

        # Variables non sensibles (depuis le manifeste)
        - name: ENVIRONMENT
          value: "production"
        - name: LOG_LEVEL
          value: "info"

        resources:
          requests:
            memory: "256Mi"
            cpu: "200m"
          limits:
            memory: "512Mi"
            cpu: "500m"
```

## Secrets pour Images Docker Privées

Si vos images Docker sont dans un registry privé, vous avez besoin d'un Secret spécial de type `dockerconfigjson`.

### Créer un Secret Docker Registry

```bash
# Méthode 1 : Via kubectl
microk8s kubectl create secret docker-registry my-registry-secret \
  --docker-server=registry.example.com \
  --docker-username=myuser \
  --docker-password=mypassword \
  --docker-email=myemail@example.com
```

### Utiliser le Secret pour pull d'images

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: private-app
spec:
  replicas: 2
  selector:
    matchLabels:
      app: private-app
  template:
    metadata:
      labels:
        app: private-app
    spec:
      # Référence au Secret docker-registry
      imagePullSecrets:
      - name: my-registry-secret

      containers:
      - name: app
        image: registry.example.com/myapp:1.0.0  # Image privée
        ports:
        - containerPort: 8080
```

**Note :** `imagePullSecrets` est au niveau du Pod spec, pas du conteneur.

## Secrets TLS pour HTTPS

Pour des certificats TLS (HTTPS), utilisez le type `kubernetes.io/tls` :

```bash
# Créer un Secret TLS depuis des fichiers
microk8s kubectl create secret tls my-tls-secret \
  --cert=path/to/tls.crt \
  --key=path/to/tls.key
```

**Ou via YAML :**

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: my-tls-secret
type: kubernetes.io/tls
stringData:
  tls.crt: |
    -----BEGIN CERTIFICATE-----
    MIIDXTCCAkWgAwIBAgIJAKJ...
    -----END CERTIFICATE-----
  tls.key: |
    -----BEGIN PRIVATE KEY-----
    MIIEvQIBADANBgkqhkiG9w0...
    -----END PRIVATE KEY-----
```

**Utilisation dans un Ingress :**

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: my-ingress
spec:
  tls:
  - hosts:
    - example.com
    secretName: my-tls-secret  # Référence au Secret TLS
  rules:
  - host: example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: my-service
            port:
              number: 80
```

Nous verrons les certificats en détail dans le chapitre 11.

## Gérer les Secrets

### Lister les Secrets

```bash
# Lister tous les Secrets
microk8s kubectl get secrets

# Dans un namespace spécifique
microk8s kubectl get secrets -n production

# Avec plus de détails
microk8s kubectl get secrets -o wide
```

### Voir les détails d'un Secret

```bash
# Description du Secret
microk8s kubectl describe secret db-credentials

# Sortie :
# Name:         db-credentials
# Namespace:    default
# Labels:       <none>
# Annotations:  <none>
# Type:  Opaque
# Data
# ====
# password:  16 bytes
# username:  5 bytes
```

**Note :** Par défaut, `describe` ne montre PAS les valeurs, seulement leur taille.

### Voir le contenu d'un Secret (décodé)

```bash
# Obtenir le Secret en YAML
microk8s kubectl get secret db-credentials -o yaml

# Voir une clé spécifique (encodée en base64)
microk8s kubectl get secret db-credentials -o jsonpath='{.data.password}'

# Décoder une clé
microk8s kubectl get secret db-credentials -o jsonpath='{.data.password}' | base64 -d
```

**⚠️ Attention :** Ces commandes exposent le secret ! Utilisez-les avec précaution.

### Modifier un Secret

```bash
# Éditer directement (ouvre l'éditeur)
microk8s kubectl edit secret db-credentials

# Remplacer depuis un fichier
microk8s kubectl apply -f db-secret.yaml

# Supprimer et recréer
microk8s kubectl delete secret db-credentials
microk8s kubectl create secret generic db-credentials --from-literal=...
```

**⚠️ Important :** Modifier un Secret ne redémarre PAS automatiquement les Pods. Vous devez :

```bash
# Forcer le redémarrage des Pods
microk8s kubectl rollout restart deployment webapp
```

### Supprimer un Secret

```bash
# Supprimer un Secret
microk8s kubectl delete secret db-credentials

# Depuis un fichier
microk8s kubectl delete -f db-secret.yaml
```

**⚠️ Attention :** Si des Pods utilisent ce Secret, ils ne pourront plus démarrer !

## Bonnes Pratiques de Sécurité

### 1. Ne jamais versionner les Secrets dans Git

**❌ Mauvais :**
```bash
git add db-secret.yaml
git commit -m "Add database secrets"  # DANGER !
```

**✅ Bon :**
```bash
# .gitignore
*-secret.yaml
secrets/
*.key
*.pem
```

**Alternatives :**
- Créer les Secrets manuellement dans chaque environnement
- Utiliser des outils de gestion de secrets (Vault, Sealed Secrets)
- Utiliser le chiffrement Git (git-crypt, SOPS)

### 2. Utiliser des Secrets différents par environnement

Ne partagez JAMAIS les mêmes secrets entre dev, staging et production :

```yaml
# dev-secrets.yaml
stringData:
  db-password: "dev_password_123"

# prod-secrets.yaml
stringData:
  db-password: "P@ssw0rd_Pr0d!#2024_$uper$ecure"
```

### 3. Principe du moindre privilège

Limitez l'accès aux Secrets via RBAC :

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: secret-reader
  namespace: production
rules:
- apiGroups: [""]
  resources: ["secrets"]
  resourceNames: ["webapp-secrets"]  # Secret spécifique uniquement
  verbs: ["get", "list"]
```

### 4. Rotation régulière des secrets

Changez vos secrets régulièrement :
- Mots de passe : tous les 90 jours
- API keys : tous les 6 mois
- Certificats : avant expiration

```bash
# Exemple de rotation
microk8s kubectl create secret generic db-credentials-new \
  --from-literal=password=new_password

# Mettre à jour le Deployment pour utiliser le nouveau Secret
# Puis supprimer l'ancien
microk8s kubectl delete secret db-credentials-old
```

### 5. Audit des accès aux Secrets

Activez l'audit logging pour tracer qui accède aux Secrets :

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

### 6. Chiffrer les Secrets au repos

Par défaut, les Secrets sont encodés en base64 dans etcd, pas chiffrés. Pour une vraie sécurité, activez le chiffrement au repos (configuration avancée du cluster).

### 7. Utiliser des volumes plutôt que des variables d'environnement

**Pourquoi ?**
- Les variables d'environnement peuvent être loguées
- Elles sont visibles dans `ps`, `/proc`, etc.
- Moins de contrôle sur les permissions

**✅ Plus sécurisé :**
```yaml
volumeMounts:
- name: secrets
  mountPath: /etc/secrets
  readOnly: true

volumes:
- name: secrets
  secret:
    secretName: db-credentials
    defaultMode: 0400  # Lecture seule, propriétaire uniquement
```

### 8. Ne pas logger les secrets

**❌ Mauvais :**
```python
password = os.getenv('DB_PASSWORD')
print(f"Connecting with password: {password}")  # DANGER !
```

**✅ Bon :**
```python
password = os.getenv('DB_PASSWORD')
print("Connecting to database...")  # Pas de secret dans les logs
```

### 9. Valider les secrets au démarrage

```python
import os
import sys

required_secrets = ['DB_PASSWORD', 'API_KEY', 'JWT_SECRET']

for secret in required_secrets:
    if not os.getenv(secret):
        print(f"FATAL: Missing required secret: {secret}")
        sys.exit(1)
```

### 10. Utiliser des outils externes pour les secrets critiques

Pour la production, considérez des outils spécialisés :
- **HashiCorp Vault** : Gestion centralisée des secrets
- **Sealed Secrets** : Chiffrement des secrets pour Git
- **External Secrets Operator** : Synchronisation avec des providers externes (AWS Secrets Manager, Azure Key Vault, etc.)

## Limitations des Secrets Kubernetes

### Taille maximale

- **1 MB** par Secret
- Limité par la taille maximale des objets etcd

**Si vous dépassez :**
- Divisez en plusieurs Secrets
- Utilisez un volume externe (PV)
- Utilisez une solution externe (Vault)

### Pas de rotation automatique

Kubernetes ne fait pas de rotation automatique des Secrets. Vous devez :
- Implémenter votre propre processus
- Utiliser des outils externes
- Redémarrer manuellement les Pods après rotation

### Base64 n'est pas du chiffrement

**Important :** Les Secrets sont encodés en base64, pas chiffrés. N'importe qui avec accès à etcd peut les lire.

**Mitigation :**
- Activer le chiffrement au repos dans etcd
- Limiter l'accès à etcd
- Utiliser des outils de chiffrement (Sealed Secrets, SOPS)

### Les Secrets sont en mémoire

Les Secrets montés comme volumes sont stockés dans tmpfs (RAM), pas sur disque. Cela signifie :
- ✅ Plus sécurisé (pas écrit sur disque)
- ❌ Consomme de la RAM
- ❌ Perdu au redémarrage (mais recréé automatiquement)

## Debugging des Secrets

### Le Pod ne démarre pas

**Symptôme :**
```bash
microk8s kubectl get pods
# NAME    READY   STATUS              RESTARTS   AGE
# app     0/1     CreateContainerConfigError   0   30s
```

**Vérifications :**

```bash
# Voir les événements
microk8s kubectl describe pod app

# Chercher des messages comme :
# Error: secret "db-credentials" not found
```

**Causes courantes :**
- Le Secret n'existe pas
- Nom du Secret incorrect
- Secret dans un autre namespace
- Clé inexistante dans le Secret

**Solutions :**

```bash
# Vérifier si le Secret existe
microk8s kubectl get secret db-credentials

# Vérifier le namespace
microk8s kubectl get secret db-credentials -n production

# Voir les clés disponibles
microk8s kubectl get secret db-credentials -o jsonpath='{.data}' | jq
```

### Le Secret existe mais la valeur est vide

```bash
# Accéder au conteneur
microk8s kubectl exec -it app -- sh

# Vérifier la variable
echo $DB_PASSWORD
# Si vide, problème de référence

# Vérifier le fichier (si monté comme volume)
cat /etc/secrets/password
```

**Vérification :**
```bash
# Le Secret contient-il la clé ?
microk8s kubectl get secret db-credentials -o yaml

# La clé est-elle bien référencée ?
microk8s kubectl get pod app -o yaml | grep -A 5 secretKeyRef
```

### Erreur de décodage base64

Si vous créez un Secret manuellement avec base64 :

```bash
# Tester le décodage
echo 'YWRtaW4=' | base64 -d
# Doit afficher : admin

# Si erreur, le base64 est malformé
```

**Solution :** Utilisez `stringData` au lieu de `data` pour éviter l'encodage manuel.

## Récapitulatif

Dans cette section, vous avez appris à :

✅ Comprendre l'importance de **ne jamais hardcoder les secrets**
✅ Créer des Secrets via plusieurs méthodes (CLI, fichiers, YAML)
✅ Utiliser les Secrets comme **variables d'environnement**
✅ Monter les Secrets comme **volumes (fichiers)**
✅ Gérer les Secrets pour **Docker registries** et **certificats TLS**
✅ Appliquer les **bonnes pratiques de sécurité**
✅ Déboguer les problèmes de Secrets

**Points clés :**

- Les **Secrets** séparent les données sensibles du code
- Utilisez `secretKeyRef` pour les variables d'environnement
- Utilisez les **volumes** pour plus de sécurité
- **Ne versionnez JAMAIS** les Secrets dans Git
- L'encodage **base64 n'est PAS du chiffrement**
- Activez le **chiffrement au repos** pour la production
- Utilisez des **outils externes** (Vault, Sealed Secrets) pour plus de sécurité
- **Rotation régulière** des secrets

La sécurité des secrets est critique. Prenez le temps d'implémenter ces pratiques dès le début de vos projets Kubernetes.

---

**Prochaine section :** 4.6 Commandes kubectl essentielles

⏭️ [Commandes kubectl essentielles](/04-premiers-deploiements/06-commandes-kubectl-essentielles.md)
