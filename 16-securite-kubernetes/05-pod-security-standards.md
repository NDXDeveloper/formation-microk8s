🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 16.5 Pod Security Standards

## Introduction

Les Pod Security Standards (PSS) définissent les règles de sécurité que doivent respecter vos pods et conteneurs. Ils contrôlent ce qu'un conteneur peut ou ne peut pas faire sur le système hôte, limitant ainsi les risques en cas de compromission.

**Analogie simple :** Imaginez un immeuble où vous louez des bureaux. Le Pod Security Standard, c'est comme le règlement de l'immeuble qui dit : "Vous pouvez utiliser votre bureau, mais vous ne pouvez pas percer les murs porteurs, modifier le système électrique ou accéder aux locaux techniques." Cela protège l'infrastructure tout en vous laissant travailler.

## Le Problème : Conteneurs Trop Privilégiés

### Risques d'un Conteneur Sans Restrictions

Par défaut, un conteneur peut :

```
┌────────────────────────────────────────┐
│         Nœud Kubernetes (Host)         │
│                                        │
│  ┌──────────────────────────────────┐  │
│  │  Conteneur Sans Restrictions     │  │
│  │                                  │  │
│  │  ✗ Peut s'exécuter en root       │  │
│  │  ✗ Peut accéder au système hôte  │  │
│  │  ✗ Peut escalader ses privilèges │  │
│  │  ✗ Peut lire tous les fichiers   │  │
│  │  ✗ Peut modifier le réseau       │  │
│  └──────────────────────────────────┘  │
│                                        │
│  ⚠️ Risque de compromission complète   │
└────────────────────────────────────────┘
```

**Conséquences d'une compromission :**
- Accès root au système hôte
- Lecture de secrets de tous les autres conteneurs
- Modification du système de fichiers de l'hôte
- Compromission d'autres pods sur le même nœud

### Avec Pod Security Standards

```
┌────────────────────────────────────────┐
│         Nœud Kubernetes (Host)         │
│                                        │
│  ┌──────────────────────────────────┐  │
│  │  Conteneur Restreint             │  │
│  │                                  │  │
│  │  ✓ Exécution en utilisateur non  │  │
│  │    privilégié                    │  │
│  │  ✓ Pas d'accès au système hôte   │  │
│  │  ✓ Capabilities minimales        │  │
│  │  ✓ Filesystem en lecture seule   │  │
│  │  ✓ Isolation réseau              │  │
│  └──────────────────────────────────┘  │
│                                        │
│  ✓ Impact limité en cas de problème    │
└────────────────────────────────────────┘
```

## Les Trois Niveaux de Sécurité

Kubernetes définit trois niveaux de sécurité pour les pods, du plus permissif au plus restrictif :

### Vue d'Ensemble

```
┌─────────────────────────────────────────────┐
│  Privileged (Privilégié)                    │
│  Aucune restriction                         │
│  ⚠️ Dangereux, uniquement pour cas spéciaux │
└─────────────────────────────────────────────┘
           │
           ▼
┌─────────────────────────────────────────────┐
│  Baseline (Base)                            │
│  Restrictions minimales                     │
│  ✓ Empêche les escalades connues            │
└─────────────────────────────────────────────┘
           │
           ▼
┌─────────────────────────────────────────────┐
│  Restricted (Restreint)                     │
│  Sécurité maximale                          │
│  ✓ Bonnes pratiques strictes                │
│  ✓ Recommandé pour la production            │
└─────────────────────────────────────────────┘
```

### 1. Privileged (Privilégié)

**Description :** Aucune restriction. Le conteneur peut tout faire.

**Autorisations :**
- ✅ Tout est autorisé
- ✅ Exécution en root
- ✅ Accès aux volumes host
- ✅ Toutes les capabilities Linux
- ✅ Conteneurs privilégiés

**Quand l'utiliser ?**
- ❌ Jamais en production standard
- ⚠️ Outils système très spécifiques (monitoring bas niveau, CNI)
- ⚠️ Développement local uniquement

**Exemple de cas d'usage :**
```yaml
# Un pod CNI qui doit configurer le réseau de l'hôte
kind: DaemonSet
name: calico-node
# Nécessite des privilèges élevés
```

### 2. Baseline (Base)

**Description :** Empêche les escalades de privilèges connues. C'est un bon équilibre pour commencer.

**Restrictions principales :**
- ❌ Pas de conteneurs privilégiés (`privileged: true`)
- ❌ Pas d'accès aux volumes host (`hostPath`, `hostNetwork`, `hostPID`, `hostIPC`)
- ❌ Pas d'escalade de privilèges (`allowPrivilegeEscalation: false`)
- ❌ Capabilities restreintes (pas `ALL`)
- ⚠️ Peut s'exécuter en root (mais c'est déconseillé)

**Autorisations :**
- ✅ Volumes standards (emptyDir, configMap, secret, PVC)
- ✅ Ports > 1024
- ✅ Capabilities minimales nécessaires

**Quand l'utiliser ?**
- ✓ Applications standards
- ✓ Point de départ pour la migration
- ✓ Environnements de développement et staging

**Exemple :**
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: baseline-pod
spec:
  containers:
  - name: app
    image: nginx:latest
    securityContext:
      allowPrivilegeEscalation: false
      # Pas de runAsNonRoot requis
```

### 3. Restricted (Restreint)

**Description :** Sécurité maximale. Suit toutes les bonnes pratiques de hardening.

**Restrictions strictes :**
- ❌ Tout ce que Baseline interdit
- ❌ **Doit** s'exécuter en utilisateur non-root (`runAsNonRoot: true`)
- ❌ Système de fichiers racine en lecture seule
- ❌ Presque toutes les capabilities supprimées
- ❌ Pas d'accès aux ports < 1024
- ❌ seccomp activé

**Autorisations :**
- ✅ Volumes minimaux (emptyDir, configMap, secret, PVC, projected, downwardAPI)
- ✅ Ports >= 1024 uniquement

**Quand l'utiliser ?**
- ✓ **Production** (fortement recommandé)
- ✓ Applications sensibles
- ✓ Conformité stricte

**Exemple :**
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: restricted-pod
spec:
  securityContext:
    runAsNonRoot: true
    runAsUser: 1000
    seccompProfile:
      type: RuntimeDefault
  containers:
  - name: app
    image: nginx:latest
    securityContext:
      allowPrivilegeEscalation: false
      readOnlyRootFilesystem: true
      runAsNonRoot: true
      runAsUser: 1000
      capabilities:
        drop:
        - ALL
    volumeMounts:
    - name: tmp
      mountPath: /tmp
    - name: cache
      mountPath: /var/cache/nginx
    - name: run
      mountPath: /var/run
  volumes:
  - name: tmp
    emptyDir: {}
  - name: cache
    emptyDir: {}
  - name: run
    emptyDir: {}
```

## Modes d'Application

Les Pod Security Standards peuvent être appliqués selon trois modes :

### 1. Enforce (Appliquer)

**Comportement :** Rejette les pods qui ne respectent pas le standard.

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: production
  labels:
    pod-security.kubernetes.io/enforce: restricted
```

**Résultat :**
```bash
kubectl apply -f bad-pod.yaml -n production
# Error: pods "bad-pod" is forbidden: violates PodSecurity "restricted:latest"
```

**Usage :** Production, quand vous êtes sûr que tout est conforme.

### 2. Audit (Auditer)

**Comportement :** Permet la création mais enregistre une violation dans les logs d'audit.

```yaml
metadata:
  labels:
    pod-security.kubernetes.io/audit: restricted
```

**Résultat :**
- ✅ Pod créé
- 📝 Violation enregistrée dans l'audit log
- 📊 Visible dans les métriques

**Usage :** Surveiller la conformité sans bloquer.

### 3. Warn (Avertir)

**Comportement :** Permet la création mais affiche un avertissement à l'utilisateur.

```yaml
metadata:
  labels:
    pod-security.kubernetes.io/warn: restricted
```

**Résultat :**
```bash
kubectl apply -f pod.yaml -n dev
# Warning: would violate PodSecurity "restricted:latest"
# pod/my-pod created
```

**Usage :** Éduquer les utilisateurs sans bloquer le développement.

### Combinaison des Modes

Vous pouvez combiner les trois modes :

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: staging
  labels:
    # Warn sur restricted
    pod-security.kubernetes.io/warn: restricted
    # Audit sur restricted
    pod-security.kubernetes.io/audit: restricted
    # Enforce sur baseline
    pod-security.kubernetes.io/enforce: baseline
```

**Stratégie :** Enforcer baseline (minimum acceptable) tout en avertissant pour restricted (objectif).

## Security Context

Le Security Context définit les paramètres de sécurité d'un pod ou d'un conteneur.

### Niveaux de Configuration

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: security-example
spec:
  # Security Context au niveau POD
  securityContext:
    runAsUser: 1000
    runAsGroup: 3000
    fsGroup: 2000

  containers:
  - name: app
    image: nginx
    # Security Context au niveau CONTENEUR
    securityContext:
      allowPrivilegeEscalation: false
      readOnlyRootFilesystem: true
```

**Hiérarchie :**
- Security Context du **conteneur** remplace celui du pod
- Si non spécifié au niveau conteneur, hérite du pod

### Paramètres Principaux

#### 1. runAsUser et runAsGroup

Définit l'utilisateur/groupe sous lequel le conteneur s'exécute.

```yaml
securityContext:
  runAsUser: 1000      # UID de l'utilisateur
  runAsGroup: 3000     # GID du groupe principal
```

**Par défaut :** UID 0 (root) ⚠️

**Bonne pratique :** Toujours spécifier un UID non-root (> 0).

#### 2. runAsNonRoot

Force l'exécution en tant qu'utilisateur non-root.

```yaml
securityContext:
  runAsNonRoot: true  # Kubernetes vérifie que UID != 0
```

**Comportement :**
- Si l'image définit `USER root` ou UID 0 → pod rejeté
- Recommandé en production

#### 3. fsGroup

Définit le groupe propriétaire des volumes montés.

```yaml
securityContext:
  fsGroup: 2000
```

**Usage :** Permet au conteneur d'écrire dans les volumes même avec un utilisateur non-root.

#### 4. allowPrivilegeEscalation

Empêche un processus d'obtenir plus de privilèges que son parent.

```yaml
securityContext:
  allowPrivilegeEscalation: false
```

**Important :** Devrait toujours être `false` sauf cas très spéciaux.

#### 5. readOnlyRootFilesystem

Rend le système de fichiers racine du conteneur en lecture seule.

```yaml
securityContext:
  readOnlyRootFilesystem: true
```

**Avantages :**
- Empêche les modifications non autorisées
- Force l'utilisation de volumes pour les écritures

**Exemple avec volumes temporaires :**
```yaml
spec:
  containers:
  - name: app
    securityContext:
      readOnlyRootFilesystem: true
    volumeMounts:
    - name: tmp
      mountPath: /tmp
    - name: cache
      mountPath: /var/cache
  volumes:
  - name: tmp
    emptyDir: {}
  - name: cache
    emptyDir: {}
```

#### 6. privileged

Donne au conteneur presque les mêmes privilèges que root sur l'hôte.

```yaml
securityContext:
  privileged: true  # ⚠️ DANGEREUX
```

**Ne jamais utiliser sauf :**
- Outils système très bas niveau
- Jamais en production standard

#### 7. capabilities

Les capabilities Linux permettent de donner des privilèges spécifiques sans donner root complet.

```yaml
securityContext:
  capabilities:
    drop:
    - ALL          # Supprimer toutes les capabilities
    add:
    - NET_BIND_SERVICE  # Ajouter uniquement celle-ci
```

**Capabilities courantes :**

| Capability | Description | Quand l'utiliser |
|------------|-------------|------------------|
| `NET_BIND_SERVICE` | Bind ports < 1024 | Application legacy sur port 80/443 |
| `CHOWN` | Changer ownership de fichiers | Rarement nécessaire |
| `DAC_OVERRIDE` | Bypass permissions fichiers | Éviter |
| `SETUID`/`SETGID` | Changer UID/GID | Rarement nécessaire |
| `SYS_ADMIN` | Droits admin système | ⚠️ Très dangereux |
| `ALL` | Toutes les capabilities | ❌ Jamais |

**Bonne pratique :**
```yaml
# Toujours commencer par drop ALL
capabilities:
  drop:
  - ALL
  add:
  - NET_BIND_SERVICE  # Ajouter uniquement ce qui est nécessaire
```

#### 8. seccomp (Secure Computing Mode)

Filtre les syscalls système que le conteneur peut effectuer.

```yaml
securityContext:
  seccompProfile:
    type: RuntimeDefault  # Profil par défaut du runtime
```

**Profils disponibles :**
- `RuntimeDefault` : Profil sécurisé par défaut (recommandé)
- `Unconfined` : Aucune restriction (déconseillé)
- `Localhost` : Profil personnalisé

**Recommandation :** Toujours utiliser `RuntimeDefault`.

#### 9. SELinux (Security-Enhanced Linux)

Options SELinux (principalement pour Red Hat/CentOS).

```yaml
securityContext:
  seLinuxOptions:
    level: "s0:c123,c456"
```

**Note :** Dépend du système d'exploitation de l'hôte.

#### 10. AppArmor

Profil AppArmor (principalement pour Ubuntu/Debian).

```yaml
metadata:
  annotations:
    container.apparmor.security.beta.kubernetes.io/app: runtime/default
```

## Application des Standards aux Namespaces

### Méthode 1 : Labels sur Namespace

La méthode standard pour appliquer les Pod Security Standards :

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: production
  labels:
    # Enforce le niveau restricted
    pod-security.kubernetes.io/enforce: restricted
    pod-security.kubernetes.io/enforce-version: latest

    # Audit et warn aussi sur restricted
    pod-security.kubernetes.io/audit: restricted
    pod-security.kubernetes.io/audit-version: latest
    pod-security.kubernetes.io/warn: restricted
    pod-security.kubernetes.io/warn-version: latest
```

### Méthode 2 : Modification d'un Namespace Existant

```bash
# Ajouter enforce restricted
kubectl label namespace production \
  pod-security.kubernetes.io/enforce=restricted

# Ajouter warn baseline
kubectl label namespace dev \
  pod-security.kubernetes.io/warn=baseline

# Voir les labels
kubectl get namespace production --show-labels
```

### Versions

Vous pouvez spécifier une version Kubernetes spécifique :

```yaml
labels:
  pod-security.kubernetes.io/enforce: restricted
  pod-security.kubernetes.io/enforce-version: v1.27  # Version spécifique
```

**Options :**
- `latest` : Dernière version (recommandé)
- `v1.XX` : Version Kubernetes spécifique
- Non spécifié : Même comportement que `latest`

## Exemples Pratiques

### Exemple 1 : Application Web Standard (Baseline)

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: webapp
  namespace: staging
spec:
  containers:
  - name: nginx
    image: nginx:1.21
    ports:
    - containerPort: 8080
    securityContext:
      allowPrivilegeEscalation: false
      runAsUser: 101  # nginx user dans l'image
```

**Conformité :** ✅ Baseline

### Exemple 2 : API Sécurisée (Restricted)

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api
  namespace: production
spec:
  replicas: 3
  selector:
    matchLabels:
      app: api
  template:
    metadata:
      labels:
        app: api
    spec:
      securityContext:
        runAsNonRoot: true
        runAsUser: 1000
        runAsGroup: 3000
        fsGroup: 2000
        seccompProfile:
          type: RuntimeDefault
      containers:
      - name: api
        image: my-api:v2.0
        ports:
        - containerPort: 8080
        securityContext:
          allowPrivilegeEscalation: false
          readOnlyRootFilesystem: true
          runAsNonRoot: true
          runAsUser: 1000
          capabilities:
            drop:
            - ALL
        volumeMounts:
        - name: tmp
          mountPath: /tmp
        - name: app-cache
          mountPath: /app/cache
      volumes:
      - name: tmp
        emptyDir: {}
      - name: app-cache
        emptyDir: {}
```

**Conformité :** ✅ Restricted

### Exemple 3 : Base de Données avec Volume Persistant

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: postgres
  namespace: production
spec:
  securityContext:
    runAsNonRoot: true
    runAsUser: 999  # postgres user
    runAsGroup: 999
    fsGroup: 999
    seccompProfile:
      type: RuntimeDefault
  containers:
  - name: postgres
    image: postgres:14
    ports:
    - containerPort: 5432
    env:
    - name: POSTGRES_PASSWORD
      valueFrom:
        secretKeyRef:
          name: db-password
          key: password
    securityContext:
      allowPrivilegeEscalation: false
      readOnlyRootFilesystem: true
      runAsNonRoot: true
      runAsUser: 999
      capabilities:
        drop:
        - ALL
    volumeMounts:
    - name: data
      mountPath: /var/lib/postgresql/data
    - name: tmp
      mountPath: /tmp
    - name: run
      mountPath: /var/run/postgresql
  volumes:
  - name: data
    persistentVolumeClaim:
      claimName: postgres-pvc
  - name: tmp
    emptyDir: {}
  - name: run
    emptyDir: {}
```

**Conformité :** ✅ Restricted

### Exemple 4 : Job de Maintenance (Baseline)

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: backup-job
  namespace: production
spec:
  template:
    spec:
      securityContext:
        runAsUser: 1000
        fsGroup: 2000
      containers:
      - name: backup
        image: backup-tool:latest
        command: ["/bin/sh", "-c"]
        args:
        - |
          pg_dump -h postgres -U user dbname > /backup/dump.sql
        securityContext:
          allowPrivilegeEscalation: false
          runAsUser: 1000
          capabilities:
            drop:
            - ALL
        volumeMounts:
        - name: backup
          mountPath: /backup
      volumes:
      - name: backup
        persistentVolumeClaim:
          claimName: backup-pvc
      restartPolicy: OnFailure
```

**Conformité :** ✅ Baseline (peut être amélioré vers Restricted)

## Stratégies de Migration

### Phase 1 : Audit (1-2 semaines)

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: production
  labels:
    # Pas d'enforce, seulement audit et warn
    pod-security.kubernetes.io/audit: restricted
    pod-security.kubernetes.io/warn: restricted
```

**Actions :**
1. Observer les violations dans les logs
2. Identifier les pods non conformes
3. Documenter les ajustements nécessaires

### Phase 2 : Enforce Baseline (2-4 semaines)

```yaml
metadata:
  labels:
    # Enforce baseline (plus permissif)
    pod-security.kubernetes.io/enforce: baseline
    # Continue audit/warn sur restricted
    pod-security.kubernetes.io/audit: restricted
    pod-security.kubernetes.io/warn: restricted
```

**Actions :**
1. Corriger les violations baseline
2. Tester les déploiements
3. Former les équipes

### Phase 3 : Enforce Restricted (4-8 semaines)

```yaml
metadata:
  labels:
    # Enforce restricted partout
    pod-security.kubernetes.io/enforce: restricted
    pod-security.kubernetes.io/audit: restricted
    pod-security.kubernetes.io/warn: restricted
```

**Actions :**
1. Ajuster tous les pods pour restricted
2. Vérifier les images (doivent supporter non-root)
3. Valider en production

## Adaptation des Images Docker

### Problème : Image Nécessitant Root

Beaucoup d'images Docker s'exécutent en root par défaut.

```dockerfile
# Mauvaise pratique
FROM nginx:latest
# USER par défaut est root
```

### Solution 1 : Utiliser des Images Non-Root

```dockerfile
# Bonne pratique : spécifier un USER
FROM nginx:latest

# Créer un utilisateur non-root
RUN useradd -r -u 1001 -g root nginx-user

# Ajuster les permissions
RUN chown -R 1001:0 /var/cache/nginx /var/run /var/log/nginx && \
    chmod -R g+rwx /var/cache/nginx /var/run /var/log/nginx

# Basculer vers cet utilisateur
USER 1001

# Écouter sur un port > 1024
EXPOSE 8080
```

### Solution 2 : Images Pré-configurées

Utiliser des images qui supportent déjà non-root :

```yaml
containers:
- name: nginx
  image: bitnami/nginx:latest  # Supporte runAsNonRoot
  securityContext:
    runAsNonRoot: true
```

**Images "distroless" de Google :**
```yaml
containers:
- name: app
  image: gcr.io/distroless/nodejs:latest
  # Déjà configuré pour non-root
```

### Solution 3 : Modifier l'Image Existante

```dockerfile
FROM nginx:latest

# Créer utilisateur
RUN useradd -r -u 1001 nginx-app

# Copier fichiers de configuration
COPY nginx.conf /etc/nginx/nginx.conf

# Permissions
RUN chown -R 1001:1001 /var/cache/nginx /var/run /var/log/nginx && \
    touch /var/run/nginx.pid && \
    chown 1001:1001 /var/run/nginx.pid

USER 1001

# Port non-privilégié
EXPOSE 8080
```

**Configuration nginx.conf :**
```nginx
# Écouter sur port > 1024
listen 8080;

# Pas de directives nécessitant root
# user nginx; <- Retirer cette ligne
```

## Bonnes Pratiques

### 1. Commencer Tôt

Appliquer les Pod Security Standards dès le début du projet :

```yaml
# Nouveau namespace
apiVersion: v1
kind: Namespace
metadata:
  name: new-project
  labels:
    pod-security.kubernetes.io/enforce: restricted
```

### 2. Stratégie par Environnement

```yaml
# Développement : permissif
namespace: dev
labels:
  pod-security.kubernetes.io/warn: baseline

# Staging : intermédiaire
namespace: staging
labels:
  pod-security.kubernetes.io/enforce: baseline
  pod-security.kubernetes.io/warn: restricted

# Production : strict
namespace: production
labels:
  pod-security.kubernetes.io/enforce: restricted
```

### 3. Toujours Drop ALL Capabilities

```yaml
securityContext:
  capabilities:
    drop:
    - ALL
    # add: seulement si vraiment nécessaire
```

### 4. Read-Only Root Filesystem

```yaml
securityContext:
  readOnlyRootFilesystem: true

# Fournir emptyDir pour les écritures temporaires
volumeMounts:
- name: tmp
  mountPath: /tmp
```

### 5. Seccomp par Défaut

```yaml
securityContext:
  seccompProfile:
    type: RuntimeDefault
```

### 6. Documenter les Exceptions

Si vous devez faire une exception :

```yaml
metadata:
  annotations:
    security-exception: "Nécessite NET_BIND_SERVICE pour legacy app"
    approved-by: "security-team"
    approved-date: "2024-01-15"
    review-date: "2024-07-15"
```

### 7. Tester les Déploiements

```bash
# Tester dans un namespace temporaire avec restricted
kubectl create namespace test-restricted
kubectl label namespace test-restricted \
  pod-security.kubernetes.io/enforce=restricted

# Déployer votre application
kubectl apply -f app.yaml -n test-restricted

# Si ça fonctionne, vous êtes conforme !
```

### 8. Utiliser des Scanners

```bash
# kubesec : analyser la sécurité d'un manifeste
kubesec scan pod.yaml

# kube-score : évaluation de la configuration
kube-score score pod.yaml

# polaris : audit de sécurité
polaris audit --format=pretty
```

## Débogage

### Erreur : Pod Rejeté

**Message :**
```
Error: pods "my-pod" is forbidden: violates PodSecurity "restricted:latest":
allowPrivilegeEscalation != false (container "app" must set
securityContext.allowPrivilegeEscalation=false)
```

**Solution :**
```yaml
securityContext:
  allowPrivilegeEscalation: false  # Ajouter cette ligne
```

### Erreur : RunAsNonRoot

**Message :**
```
Error: pods "my-pod" is forbidden: violates PodSecurity "restricted:latest":
runAsNonRoot != true (pod or container "app" must set
securityContext.runAsNonRoot=true)
```

**Solution :**
```yaml
spec:
  securityContext:
    runAsNonRoot: true
    runAsUser: 1000  # UID > 0
  containers:
  - name: app
    securityContext:
      runAsNonRoot: true
```

### Erreur : Capabilities

**Message :**
```
Error: pods "my-pod" is forbidden: violates PodSecurity "restricted:latest":
unrestricted capabilities (container "app" must set
securityContext.capabilities.drop=["ALL"])
```

**Solution :**
```yaml
securityContext:
  capabilities:
    drop:
    - ALL
```

### Erreur : Seccomp

**Message :**
```
Error: pods "my-pod" is forbidden: violates PodSecurity "restricted:latest":
seccompProfile (pod or container "app" must set
securityContext.seccompProfile.type to "RuntimeDefault" or "Localhost")
```

**Solution :**
```yaml
spec:
  securityContext:
    seccompProfile:
      type: RuntimeDefault
```

### Voir les Détails d'une Violation

```bash
# Voir les événements
kubectl get events -n production --sort-by='.lastTimestamp'

# Logs de l'API server (si accessible)
kubectl logs -n kube-system kube-apiserver-xxx | grep PodSecurity
```

### Tester un Pod sans le Créer

```bash
# Dry-run pour voir si le pod serait accepté
kubectl apply -f pod.yaml --dry-run=server -n production
```

## Pod Security Standards dans MicroK8s

### Activation

Par défaut, MicroK8s supporte les Pod Security Standards (disponible depuis Kubernetes 1.23+).

```bash
# Vérifier la version
microk8s kubectl version --short

# Les PSS sont activés par défaut si version >= 1.23
```

### Configuration d'un Namespace

```bash
# Créer un namespace avec enforce restricted
microk8s kubectl create namespace secure-app

# Ajouter les labels
microk8s kubectl label namespace secure-app \
  pod-security.kubernetes.io/enforce=restricted \
  pod-security.kubernetes.io/audit=restricted \
  pod-security.kubernetes.io/warn=restricted
```

### Tester

```bash
# Créer un pod non conforme
cat <<EOF | microk8s kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: bad-pod
  namespace: secure-app
spec:
  containers:
  - name: nginx
    image: nginx
EOF

# Devrait être rejeté avec un message explicite
```

### Désactiver (Non Recommandé)

Les Pod Security Standards sont au niveau de l'API et ne peuvent pas être "désactivés" dans MicroK8s. Pour éviter les restrictions :

```bash
# Ne pas mettre de labels sur le namespace
# Ou utiliser 'privileged'
kubectl label namespace my-ns \
  pod-security.kubernetes.io/enforce=privileged
```

## Checklist de Conformité Restricted

Voici une checklist pour vérifier qu'un pod est conforme au niveau **Restricted** :

### Au niveau Pod (spec.securityContext)

- [ ] `runAsNonRoot: true`
- [ ] `runAsUser` défini (valeur > 0)
- [ ] `seccompProfile.type: RuntimeDefault`

### Au niveau Conteneur (spec.containers[].securityContext)

- [ ] `allowPrivilegeEscalation: false`
- [ ] `runAsNonRoot: true`
- [ ] `capabilities.drop: [ALL]`
- [ ] `readOnlyRootFilesystem: true` (recommandé)

### Volumes

- [ ] Pas de `hostPath`
- [ ] Pas de `hostNetwork: true`
- [ ] Pas de `hostPID: true`
- [ ] Pas de `hostIPC: true`
- [ ] Seulement volumes autorisés : emptyDir, configMap, secret, PVC, projected, downwardAPI

### Ports

- [ ] Conteneurs écoutent sur ports >= 1024

### Général

- [ ] Pas de `privileged: true`
- [ ] Image supporte l'exécution non-root

### Exemple de Pod Conforme

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: fully-compliant
  namespace: production
spec:
  # ✓ Pod security context
  securityContext:
    runAsNonRoot: true
    runAsUser: 1000
    runAsGroup: 3000
    fsGroup: 2000
    seccompProfile:
      type: RuntimeDefault

  containers:
  - name: app
    image: my-app:latest
    ports:
    - containerPort: 8080  # ✓ Port >= 1024

    # ✓ Container security context
    securityContext:
      allowPrivilegeEscalation: false
      runAsNonRoot: true
      runAsUser: 1000
      readOnlyRootFilesystem: true
      capabilities:
        drop:
        - ALL

    # ✓ Volumes pour écritures temporaires
    volumeMounts:
    - name: tmp
      mountPath: /tmp
    - name: cache
      mountPath: /app/cache

  # ✓ Volumes autorisés
  volumes:
  - name: tmp
    emptyDir: {}
  - name: cache
    emptyDir: {}
```

## Conclusion

Les Pod Security Standards sont essentiels pour sécuriser vos conteneurs et pods :

1. **Trois niveaux** : Privileged, Baseline, Restricted
2. **Restricted pour la production** : suivez toutes les bonnes pratiques
3. **Trois modes** : enforce (bloquer), audit (logger), warn (avertir)
4. **Security Context** : définit les paramètres de sécurité
5. **Migration progressive** : audit → baseline → restricted

**Formule Pod Security :**
```
Pod Security Standard + Security Context = Conteneur Sécurisé
```

**Points Clés à Retenir :**

- Utilisez **Restricted** en production
- Toujours `runAsNonRoot: true`
- Toujours `allowPrivilegeEscalation: false`
- Toujours `capabilities.drop: [ALL]`
- Toujours `seccompProfile.type: RuntimeDefault`
- Utilisez `readOnlyRootFilesystem: true` quand possible
- Adaptez vos images Docker pour supporter non-root
- Testez avec `--dry-run=server`
- Migrez progressivement (audit → baseline → restricted)

**Prochaines Étapes :**

Dans les sections suivantes, nous explorerons :
- **16.6** : Scan de vulnérabilités d'images
- **16.7** : Secrets management avancé
- **16.8** : Audit logging

Les Pod Security Standards, combinés avec RBAC, ServiceAccounts et Network Policies, créent une défense en profondeur robuste pour vos applications Kubernetes.

⏭️ [Scan de vulnérabilités d'images](/16-securite-kubernetes/06-scan-de-vulnerabilites-dimages.md)
