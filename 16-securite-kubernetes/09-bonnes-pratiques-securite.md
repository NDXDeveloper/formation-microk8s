🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 16.9 Bonnes Pratiques Sécurité

## Introduction

Cette section synthétise toutes les bonnes pratiques de sécurité pour Kubernetes. C'est un guide pratique et actionnable qui vous aidera à construire et maintenir un cluster sécurisé. Chaque pratique est classée par priorité et difficulté pour vous permettre de progresser étape par étape.

**Analogie simple :** Sécuriser un cluster Kubernetes, c'est comme protéger une maison. Vous ne vous contentez pas d'une seule serrure sur la porte d'entrée : vous avez des verrous, une alarme, des caméras, des détecteurs de mouvement, des lumières extérieures, etc. Chaque couche de sécurité rend l'ensemble plus robuste. C'est le principe de **défense en profondeur**.

## Principes Fondamentaux

### 1. Défense en Profondeur (Defense in Depth)

Ne comptez jamais sur une seule mesure de sécurité.

```
┌─────────────────────────────────────────┐
│  Couche 7 : Surveillance et Audit       │
│  ├─ Audit logging                       │
│  └─ Monitoring et alertes               │
├─────────────────────────────────────────┤
│  Couche 6 : Secrets Management          │
│  ├─ Chiffrement des secrets             │
│  └─ Rotation automatique                │
├─────────────────────────────────────────┤
│  Couche 5 : Scan de Vulnérabilités      │
│  ├─ Scanner les images                  │
│  └─ Politique de mise à jour            │
├─────────────────────────────────────────┤
│  Couche 4 : Pod Security                │
│  ├─ Pod Security Standards              │
│  └─ Security Context strict             │
├─────────────────────────────────────────┤
│  Couche 3 : Isolation Réseau            │
│  ├─ Network Policies                    │
│  └─ Segmentation                        │
├─────────────────────────────────────────┤
│  Couche 2 : Contrôle d'Accès            │
│  ├─ RBAC strict                         │
│  └─ ServiceAccounts dédiés              │
├─────────────────────────────────────────┤
│  Couche 1 : Infrastructure              │
│  ├─ Nœuds sécurisés                     │
│  └─ Réseau isolé                        │
└─────────────────────────────────────────┘
```

**Si une couche est compromise, les autres continuent de protéger.**

### 2. Principe du Moindre Privilège (Least Privilege)

Donnez uniquement les permissions strictement nécessaires.

```
❌ Mauvais : Accès admin pour tout le monde
✅ Bon : Chaque utilisateur/service a les permissions minimales nécessaires
```

**Application :**
- RBAC : Rôles spécifiques par fonction
- ServiceAccounts : Un SA par application
- Network Policies : Autoriser uniquement les flux nécessaires
- Pod Security : Conteneurs non-root avec capabilities minimales

### 3. Sécurité par Défaut (Secure by Default)

Les configurations par défaut doivent être sécurisées.

```yaml
# Exemple : Template de Deployment sécurisé par défaut
apiVersion: apps/v1
kind: Deployment
spec:
  template:
    spec:
      securityContext:
        runAsNonRoot: true
        seccompProfile:
          type: RuntimeDefault
      containers:
      - name: app
        securityContext:
          allowPrivilegeEscalation: false
          readOnlyRootFilesystem: true
          capabilities:
            drop:
            - ALL
```

### 4. Ne Jamais Faire Confiance, Toujours Vérifier (Zero Trust)

Même au sein du cluster, vérifiez toujours l'identité et les autorisations.

```
Requête → Authentification → Autorisation → Admission → Exécution
          (Qui es-tu ?)     (Que peux-tu ?)  (Est-ce OK ?)
```

### 5. Sécurité Observable

Si vous ne pouvez pas le voir, vous ne pouvez pas le sécuriser.

```
Logs + Métriques + Traces + Alertes = Visibilité Complète
```

## Bonnes Pratiques par Domaine

### A. Infrastructure et Cluster

#### A1. Mise à Jour Régulière ⭐⭐⭐ (Critique)

**Pourquoi :** Les vulnérabilités sont découvertes régulièrement.

```bash
# Vérifier la version actuelle
kubectl version --short

# Planifier les mises à jour
# Kubernetes : Tous les 3 mois (nouvelles versions mineures)
# MicroK8s : suivre les releases
```

**Fréquence recommandée :**
- Kubernetes : Rester sur une version supportée (N, N-1, N-2)
- OS des nœuds : Patches de sécurité mensuels
- Composants (CNI, CSI, etc.) : Suivre les releases

**Pour MicroK8s :**
```bash
# Voir la version disponible
snap info microk8s

# Mettre à jour
sudo snap refresh microk8s --channel=1.28/stable
```

#### A2. Isoler le Cluster ⭐⭐ (Important)

**Réseau :**
```
┌────────────────────────────────────┐
│  Internet                          │
└──────────┬─────────────────────────┘
           │
    ┌──────▼──────┐
    │  Firewall   │  ← Bloquer tout sauf nécessaire
    └──────┬──────┘
           │
    ┌──────▼──────┐
    │  Cluster K8s│
    └─────────────┘
```

**Règles firewall minimales :**
```bash
# Autoriser seulement :
# - Port 6443 (API Server) depuis IPs autorisées
# - Port 80/443 (Ingress) depuis Internet
# - SSH depuis bastion uniquement
# Bloquer tout le reste
```

#### A3. Séparer les Environnements ⭐⭐⭐ (Critique)

```
┌─────────────────────────────────────┐
│  Cluster Dev                        │
│  └─ Sécurité permissive             │
│  └─ Données de test                 │
└─────────────────────────────────────┘

┌─────────────────────────────────────┐
│  Cluster Staging                    │
│  └─ Sécurité intermédiaire          │
│  └─ Données anonymisées             │
└─────────────────────────────────────┘

┌─────────────────────────────────────┐
│  Cluster Production                 │
│  └─ Sécurité maximale               │
│  └─ Données réelles                 │
└─────────────────────────────────────┘
```

**Ou avec un seul cluster :**
```bash
# Namespaces isolés
kubectl create namespace dev
kubectl create namespace staging
kubectl create namespace production

# Sécurité différente par namespace
kubectl label namespace production pod-security.kubernetes.io/enforce=restricted
kubectl label namespace dev pod-security.kubernetes.io/warn=baseline
```

#### A4. Sauvegardes Régulières ⭐⭐⭐ (Critique)

```bash
# Ce qu'il faut sauvegarder :
# 1. etcd (état du cluster)
# 2. Volumes persistants (données)
# 3. Configurations (manifestes YAML)
# 4. Secrets (chiffrés !)

# Fréquence :
# - Production : Quotidien + incrémental
# - Dev/Staging : Hebdomadaire
```

**Avec Velero :**
```bash
# Sauvegarde complète du namespace production
velero backup create prod-backup --include-namespaces production

# Sauvegarde planifiée
velero schedule create daily-backup --schedule="0 2 * * *"
```

#### A5. Hardening des Nœuds ⭐⭐ (Important)

**Checklist nœud Linux :**
```bash
# 1. Désactiver les services inutiles
systemctl disable cups bluetooth

# 2. Firewall local
ufw enable
ufw default deny incoming
ufw allow from 10.0.0.0/8 to any port 10250  # Kubelet

# 3. SSH sécurisé
# /etc/ssh/sshd_config
PermitRootLogin no
PasswordAuthentication no
PubkeyAuthentication yes

# 4. Audit système
apt install auditd
systemctl enable auditd

# 5. Kernel hardening
# /etc/sysctl.conf
net.ipv4.conf.all.forwarding = 1
net.ipv4.conf.all.rp_filter = 1
kernel.dmesg_restrict = 1
```

### B. Contrôle d'Accès (RBAC)

#### B1. Ne Jamais Utiliser cluster-admin ⭐⭐⭐ (Critique)

```yaml
# ❌ JAMAIS FAIRE ÇA
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: give-admin-to-everyone
subjects:
- kind: Group
  name: developers
roleRef:
  kind: ClusterRole
  name: cluster-admin  # ← DANGER !
```

**Solution :**
```yaml
# ✅ Créer des rôles spécifiques
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: developer-role
  namespace: dev
rules:
- apiGroups: ["", "apps"]
  resources: ["pods", "deployments", "services"]
  verbs: ["get", "list", "create", "update", "delete"]
```

#### B2. Un ServiceAccount par Application ⭐⭐⭐ (Critique)

```yaml
# ❌ Mauvais : Utiliser le SA "default"
spec:
  # serviceAccountName non spécifié
  containers: ...

# ✅ Bon : SA dédié avec permissions minimales
apiVersion: v1
kind: ServiceAccount
metadata:
  name: my-app-sa
---
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: my-app-role
rules:
- apiGroups: [""]
  resources: ["configmaps"]
  resourceNames: ["my-app-config"]
  verbs: ["get"]
---
apiVersion: apps/v1
kind: Deployment
spec:
  template:
    spec:
      serviceAccountName: my-app-sa
```

#### B3. Auditer les Permissions Régulièrement ⭐⭐ (Important)

```bash
# Script d'audit mensuel
# audit-rbac.sh

# 1. Trouver les ClusterRoleBindings avec cluster-admin
kubectl get clusterrolebindings -o json | \
  jq '.items[] | select(.roleRef.name=="cluster-admin") | .metadata.name'

# 2. Lister tous les users/SA avec leurs permissions
kubectl get rolebindings,clusterrolebindings --all-namespaces -o wide

# 3. Vérifier les ServiceAccounts non utilisés
for ns in $(kubectl get ns -o jsonpath='{.items[*].metadata.name}'); do
  echo "=== Namespace: $ns ==="
  kubectl get sa -n $ns
done
```

#### B4. Utiliser des Groupes ⭐⭐ (Important)

```yaml
# ❌ Gérer individuellement
subjects:
- kind: User
  name: alice
- kind: User
  name: bob
- kind: User
  name: charlie

# ✅ Gérer par groupes
subjects:
- kind: Group
  name: developers  # Groupe géré dans LDAP/OIDC
  apiGroup: rbac.authorization.k8s.io
```

### C. Sécurité des Workloads (Pods)

#### C1. Toujours Utiliser Pod Security Standards ⭐⭐⭐ (Critique)

```yaml
# Sur TOUS les namespaces de production
apiVersion: v1
kind: Namespace
metadata:
  name: production
  labels:
    pod-security.kubernetes.io/enforce: restricted
    pod-security.kubernetes.io/audit: restricted
    pod-security.kubernetes.io/warn: restricted
```

#### C2. Jamais Root, Jamais Privileged ⭐⭐⭐ (Critique)

```yaml
# Template de sécurité pour tous vos pods
spec:
  securityContext:
    runAsNonRoot: true
    runAsUser: 1000
    runAsGroup: 3000
    fsGroup: 2000
    seccompProfile:
      type: RuntimeDefault
  containers:
  - name: app
    securityContext:
      allowPrivilegeEscalation: false
      readOnlyRootFilesystem: true
      runAsNonRoot: true
      capabilities:
        drop:
        - ALL
```

#### C3. Read-Only Root Filesystem ⭐⭐ (Important)

```yaml
spec:
  containers:
  - name: app
    securityContext:
      readOnlyRootFilesystem: true
    volumeMounts:
    # Fournir emptyDir pour les écritures temporaires
    - name: tmp
      mountPath: /tmp
    - name: cache
      mountPath: /app/cache
  volumes:
  - name: tmp
    emptyDir: {}
  - name: cache
    emptyDir: {}
```

#### C4. Resource Limits Toujours ⭐⭐⭐ (Critique)

```yaml
# Prévenir les attaques DoS
spec:
  containers:
  - name: app
    resources:
      requests:
        memory: "64Mi"
        cpu: "250m"
      limits:
        memory: "128Mi"
        cpu: "500m"
```

**Impact d'absence de limits :**
```
Pod sans limits → Peut consommer toutes les ressources du nœud
                → Fait crasher les autres pods
                → Déni de service
```

#### C5. Liveness et Readiness Probes ⭐⭐ (Important)

```yaml
# Détection automatique de problèmes
spec:
  containers:
  - name: app
    livenessProbe:
      httpGet:
        path: /healthz
        port: 8080
      initialDelaySeconds: 30
      periodSeconds: 10
    readinessProbe:
      httpGet:
        path: /ready
        port: 8080
      initialDelaySeconds: 5
      periodSeconds: 5
```

### D. Isolation Réseau

#### D1. Default Deny + Whitelist ⭐⭐⭐ (Critique)

```yaml
# Étape 1 : Bloquer tout par défaut
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-all
  namespace: production
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  - Egress
---
# Étape 2 : Autoriser DNS
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-dns
  namespace: production
spec:
  podSelector: {}
  policyTypes:
  - Egress
  egress:
  - to:
    - namespaceSelector:
        matchLabels:
          kubernetes.io/metadata.name: kube-system
    ports:
    - protocol: UDP
      port: 53
---
# Étape 3 : Autoriser les flux nécessaires
# (voir section Network Policies)
```

#### D2. Isolation par Namespace ⭐⭐⭐ (Critique)

```yaml
# Empêcher la communication cross-namespace
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: deny-cross-namespace
  namespace: production
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  ingress:
  - from:
    - podSelector: {}  # Seulement depuis le même namespace
```

#### D3. Limiter l'Accès Internet ⭐⭐ (Important)

```yaml
# Autoriser uniquement HTTPS vers IPs spécifiques
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: egress-https-only
spec:
  podSelector:
    matchLabels:
      internet-access: "true"
  policyTypes:
  - Egress
  egress:
  - to:
    - ipBlock:
        cidr: 0.0.0.0/0
        except:
        - 10.0.0.0/8
        - 172.16.0.0/12
        - 192.168.0.0/16
    ports:
    - protocol: TCP
      port: 443
```

### E. Images et Conteneurs

#### E1. Scanner Toutes les Images ⭐⭐⭐ (Critique)

```bash
# Dans le pipeline CI/CD
docker build -t my-app:latest .
trivy image --exit-code 1 --severity CRITICAL my-app:latest
docker push my-app:latest
```

**Politique :**
```
CRITICAL : Bloquer le déploiement
HIGH : Bloquer ou exception justifiée
MEDIUM : Planifier correction
LOW : Informationnel
```

#### E2. Utiliser des Images Minimales ⭐⭐ (Important)

```dockerfile
# ❌ Image complète, beaucoup de vulnérabilités
FROM ubuntu:latest
RUN apt-get update && apt-get install -y python3 ...

# ✅ Image minimale Alpine
FROM python:3.11-alpine
COPY requirements.txt .
RUN pip install -r requirements.txt
COPY app.py .
USER 1000
CMD ["python", "app.py"]

# ✅✅ Image distroless (encore mieux)
FROM gcr.io/distroless/python3:latest
COPY --from=builder /app /app
USER nonroot
ENTRYPOINT ["/app/main"]
```

#### E3. Tags Spécifiques, Jamais :latest ⭐⭐⭐ (Critique)

```yaml
# ❌ Non reproductible, dangereux
image: nginx:latest

# ✅ Version spécifique
image: nginx:1.24.0

# ✅✅ Digest SHA256 (immuable)
image: nginx@sha256:abc123def456...
```

#### E4. Signer les Images ⭐⭐ (Important)

```bash
# Avec Cosign (Sigstore)
# 1. Générer des clés
cosign generate-key-pair

# 2. Signer l'image
cosign sign --key cosign.key my-registry/my-app:v1.0.0

# 3. Vérifier avant déploiement
cosign verify --key cosign.pub my-registry/my-app:v1.0.0
```

#### E5. Pas de Secrets dans les Images ⭐⭐⭐ (Critique)

```dockerfile
# ❌ JAMAIS FAIRE ÇA
ENV DATABASE_PASSWORD=SuperSecret123
ENV API_KEY=sk-abc123xyz

# ✅ Utiliser des Secrets Kubernetes
# Dockerfile sans secrets
# Secrets injectés au runtime
```

**Vérification :**
```bash
# Scanner les images pour secrets
trivy image --scanners secret my-app:latest
```

### F. Secrets Management

#### F1. Jamais en Clair dans Git ⭐⭐⭐ (Critique)

```bash
# ❌ JAMAIS
git add secret.yaml  # Contient des secrets en clair

# ✅ Utiliser Sealed Secrets
kubeseal -f secret.yaml -w sealed-secret.yaml
git add sealed-secret.yaml

# ✅ Ou External Secrets
git add external-secret.yaml  # Référence externe
```

#### F2. Chiffrer les Secrets dans etcd ⭐⭐ (Important)

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
        secret: <base64-encoded-32-byte-key>
  - identity: {}
```

#### F3. Rotation Régulière ⭐⭐ (Important)

```
Secrets critiques : 90 jours
Secrets sensibles : 180 jours
Certificats : Avant expiration
```

**Automatisation avec External Secrets :**
```yaml
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
spec:
  refreshInterval: 1h  # Synchronisation automatique
```

#### F4. Principle of Least Privilege ⭐⭐⭐ (Critique)

```yaml
# ✅ RBAC strict sur les secrets
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: app-role
rules:
- apiGroups: [""]
  resources: ["secrets"]
  resourceNames: ["app-secret"]  # Seulement CE secret
  verbs: ["get"]
```

### G. Monitoring et Audit

#### G1. Activer l'Audit Logging ⭐⭐⭐ (Critique)

```yaml
# Politique d'audit minimale
apiVersion: audit.k8s.io/v1
kind: Policy
rules:
- level: Metadata
  verbs: ["create", "update", "patch", "delete"]
  resources:
  - group: ""
  - group: "apps"
  - group: "rbac.authorization.k8s.io"
```

#### G2. Centraliser les Logs ⭐⭐ (Important)

```
Cluster → Fluentd/Filebeat → Elasticsearch/Splunk
                           → Alertes automatiques
```

#### G3. Monitoring de Sécurité ⭐⭐ (Important)

**Métriques à surveiller :**
```
• Authentications échouées
• Changements RBAC
• Créations/suppressions de pods
• Accès aux secrets
• Modifications de Network Policies
• Pods crashant fréquemment
• Consommation anormale de ressources
```

**Alertes Prometheus :**
```yaml
- alert: UnauthorizedAPIAccess
  expr: apiserver_audit_event_total{verb!~"get|list|watch",response_code=~"401|403"} > 10
  for: 5m
  annotations:
    summary: "Multiple unauthorized access attempts detected"
```

#### G4. Runtime Security ⭐⭐ (Important)

```bash
# Installer Falco pour détecter les comportements suspects
kubectl apply -f https://raw.githubusercontent.com/falcosecurity/deploy-kubernetes/main/kubernetes/falco/templates/daemonset.yaml
```

**Falco détecte :**
- Shells dans les conteneurs
- Fichiers sensibles modifiés
- Connexions réseau suspectes
- Escalades de privilèges

### H. Maintenance et Mise à Jour

#### H1. Patch Management ⭐⭐⭐ (Critique)

```
Plan de patching :
• Vulnérabilités CRITICAL : Sous 24h
• Vulnérabilités HIGH : Sous 7 jours
• Vulnérabilités MEDIUM : Sous 30 jours
• Mises à jour Kubernetes : Chaque trimestre
```

#### H2. Tests de Sécurité Réguliers ⭐⭐ (Important)

```bash
# Scan mensuel du cluster
# 1. Scanner les images en production
kubectl get pods --all-namespaces -o jsonpath='{range .items[*]}{.spec.containers[*].image}{"\n"}{end}' | sort -u | while read image; do
  trivy image "$image"
done

# 2. Audit RBAC
kubectl-who-can list secrets --all-namespaces

# 3. Vérifier les Network Policies
kubectl get networkpolicies --all-namespaces

# 4. Audit des configurations
kube-bench run --targets master,node
```

#### H3. Disaster Recovery ⭐⭐⭐ (Critique)

```bash
# Plan DR :
# 1. Sauvegardes testées régulièrement
velero restore create --from-backup prod-backup --dry-run

# 2. Runbook de restauration documenté
# 3. RTO (Recovery Time Objective) : 4h
# 4. RPO (Recovery Point Objective) : 1h
# 5. Tests DR trimestriels
```

## Checklist de Sécurité par Phase

### Phase 1 : Démarrage (Semaine 1-2) - Essentiel

- [ ] Kubernetes à jour (version supportée)
- [ ] RBAC activé
- [ ] Pas d'utilisation de cluster-admin
- [ ] ServiceAccount par application
- [ ] Pod Security Standards (baseline minimum)
- [ ] Secrets en base64 (pas en clair dans Git)
- [ ] Resource limits sur tous les pods
- [ ] Images de sources connues

**Niveau de sécurité : Base acceptable**

### Phase 2 : Consolidation (Semaine 3-6) - Important

- [ ] Network Policies (default deny)
- [ ] Audit logging activé
- [ ] Scan des images dans CI/CD
- [ ] Images minimales (Alpine/Distroless)
- [ ] Read-only root filesystem
- [ ] Sealed Secrets ou External Secrets
- [ ] Monitoring de base (Prometheus)
- [ ] Sauvegardes automatiques

**Niveau de sécurité : Production acceptable**

### Phase 3 : Avancé (Semaine 7-12) - Optimal

- [ ] Pod Security Standards (restricted)
- [ ] Chiffrement etcd
- [ ] Signature des images
- [ ] HashiCorp Vault ou Cloud KMS
- [ ] SIEM centralisé (Elasticsearch/Splunk)
- [ ] Runtime security (Falco)
- [ ] Admission controllers personnalisés
- [ ] Pen testing régulier

**Niveau de sécurité : Production haute sécurité**

### Phase 4 : Excellence (Continu) - Best-in-Class

- [ ] Service Mesh (mTLS)
- [ ] Zero Trust Architecture
- [ ] Chaos Engineering
- [ ] Bug Bounty Program
- [ ] Certification (CIS Benchmarks)
- [ ] Conformité (SOC 2, ISO 27001)

**Niveau de sécurité : Enterprise grade**

## Checklist par Environnement

### Développement (Dev)

```yaml
Priorités :
✅ RBAC de base
✅ Resource limits
⚠️ Network Policies (optionnel)
⚠️ Pod Security (baseline)
⚠️ Scan images (warnings)

Objectif : Apprentissage et rapidité
```

### Pré-production (Staging)

```yaml
Priorités :
✅ RBAC complet
✅ Resource limits
✅ Network Policies
✅ Pod Security (baseline → restricted)
✅ Scan images (bloquer CRITICAL)
✅ Audit logging
✅ Monitoring

Objectif : Réplique de production
```

### Production

```yaml
Priorités :
✅✅✅ TOUT de la phase 2 minimum
✅✅✅ Pod Security (restricted)
✅✅✅ Scan images (bloquer CRITICAL+HIGH)
✅✅✅ Secrets management avancé
✅✅✅ Audit logging complet
✅✅✅ Monitoring et alertes
✅✅✅ Sauvegardes testées
✅✅✅ Plan DR

Objectif : Sécurité maximale
```

## Anti-Patterns à Éviter

### ❌ Anti-Pattern 1 : Privilèges Excessifs

```yaml
# JAMAIS faire ça
securityContext:
  privileged: true
  allowPrivilegeEscalation: true
```

### ❌ Anti-Pattern 2 : Pas de Limites

```yaml
# JAMAIS déployer sans limits
spec:
  containers:
  - name: app
    # resources: non spécifié ← DANGER
```

### ❌ Anti-Pattern 3 : Secrets Hardcodés

```yaml
# JAMAIS
env:
- name: DB_PASSWORD
  value: "SuperSecret123"  # ← En clair !
```

### ❌ Anti-Pattern 4 : Images non Scannées

```bash
# JAMAIS déployer sans scanner
docker build -t app:latest .
docker push app:latest  # ← Pas de scan !
kubectl apply -f deployment.yaml
```

### ❌ Anti-Pattern 5 : Tout en Root

```dockerfile
# JAMAIS
FROM ubuntu
# USER non spécifié → root par défaut
```

### ❌ Anti-Pattern 6 : :latest Everywhere

```yaml
# JAMAIS
image: nginx:latest  # ← Non reproductible
```

### ❌ Anti-Pattern 7 : Pas d'Audit

```yaml
# JAMAIS en production sans audit
# Comment savoir qui a fait quoi ?
```

### ❌ Anti-Pattern 8 : Pas de Sauvegardes

```bash
# "On verra plus tard..."
# Puis : crash du cluster
# Résultat : Tout est perdu
```

### ❌ Anti-Pattern 9 : Un Seul Namespace

```yaml
# JAMAIS tout mélanger
# dev + staging + prod dans "default"
# ← Impossible d'isoler
```

### ❌ Anti-Pattern 10 : Pas de Monitoring

```yaml
# "Ça fonctionne, pas besoin de surveiller"
# Puis : Compromission non détectée pendant des mois
```

## Outils Recommandés

### Analyse et Audit

```bash
# 1. kube-bench - CIS Kubernetes Benchmark
kubectl apply -f https://raw.githubusercontent.com/aquasecurity/kube-bench/main/job.yaml

# 2. kubesec - Analyse des manifestes
kubesec scan pod.yaml

# 3. kube-score - Score de qualité
kube-score score deployment.yaml

# 4. Polaris - Audit complet
polaris audit --format=pretty

# 5. kubectl-who-can - Vérifier les permissions
kubectl-who-can delete pods -n production

# 6. kubetap - Debug réseau
kubectl tap deployment/api -n production

# 7. kube-ps1 - Prompt avec contexte
# Affiche le cluster/namespace actuel dans le prompt
```

### Scan d'Images

```bash
# 1. Trivy (recommandé)
trivy image nginx:latest

# 2. Grype
grype nginx:latest

# 3. Snyk
snyk container test nginx:latest

# 4. Clair (avec Harbor)
# Intégré dans Harbor registry
```

### Secrets Management

```bash
# 1. Sealed Secrets
kubeseal -f secret.yaml

# 2. External Secrets Operator
kubectl apply -f external-secret.yaml

# 3. HashiCorp Vault
vault kv get secret/data

# 4. SOPS (Mozilla)
sops -e secret.yaml
```

### Monitoring Sécurité

```bash
# 1. Falco - Runtime security
kubectl apply -f falco-daemonset.yaml

# 2. Prometheus + Alertmanager
# Métriques et alertes

# 3. Grafana
# Visualisation

# 4. Audit2rbac
# Générer des policies RBAC depuis audit logs
```

## Formation et Sensibilisation

### Pour les Développeurs

**Formation requise :**
1. RBAC de base
2. Secrets management
3. Security Context
4. Network Policies de base
5. Scan d'images

**Documentation à fournir :**
- Templates de manifestes sécurisés
- Guide de déploiement
- Checklist pré-déploiement

### Pour les Ops/SRE

**Formation requise :**
1. Sécurité complète Kubernetes
2. Audit logging et analyse
3. Incident response
4. Disaster recovery

**Certifications recommandées :**
- CKA (Certified Kubernetes Administrator)
- CKS (Certified Kubernetes Security Specialist)

### Pour les Managers

**Sensibilisation :**
1. Coûts d'une compromission
2. Importance de la sécurité dès le début
3. Budget pour outils de sécurité
4. Temps nécessaire pour correctifs

## Plan d'Amélioration Continue

### Mensuel

```bash
# Checklist mensuelle
- [ ] Scan de toutes les images en prod
- [ ] Audit des permissions RBAC
- [ ] Revue des logs d'audit
- [ ] Vérification des Network Policies
- [ ] Mise à jour des images vulnérables
- [ ] Test des sauvegardes
```

### Trimestriel

```bash
# Checklist trimestrielle
- [ ] Mise à jour Kubernetes
- [ ] Pen testing externe
- [ ] Revue complète de sécurité
- [ ] Test du plan DR
- [ ] Formation d'équipe
- [ ] Mise à jour de la documentation
```

### Annuel

```bash
# Checklist annuelle
- [ ] Certification CIS Benchmark
- [ ] Audit de conformité (SOC 2, ISO)
- [ ] Revue de l'architecture
- [ ] Évaluation des outils
- [ ] Budget sécurité année suivante
```

## Réponse aux Incidents

### Procédure Standard

```
1. DÉTECTION
   └─ Alerte automatique ou découverte manuelle

2. TRIAGE
   └─ Sévérité : P0 (Critique) à P4 (Info)
   └─ Impact : Production vs Dev
   └─ Portée : Cluster entier vs pod isolé

3. CONFINEMENT
   └─ Isoler les ressources affectées
   └─ Network Policies pour bloquer
   └─ Scale down si nécessaire

4. INVESTIGATION
   └─ Audit logs : qui, quoi, quand
   └─ Logs applicatifs
   └─ Métriques système

5. ÉRADICATION
   └─ Supprimer la menace
   └─ Patcher les vulnérabilités
   └─ Changer les secrets compromis

6. RÉCUPÉRATION
   └─ Restaurer depuis backup si nécessaire
   └─ Redéployer proprement
   └─ Vérifier l'intégrité

7. POST-MORTEM
   └─ Cause racine
   └─ Actions correctives
   └─ Amélioration des processus
   └─ Mise à jour de la documentation
```

### Commandes d'Urgence

```bash
# Isoler un pod compromis
kubectl label pod compromised-pod quarantine=true
kubectl apply -f - <<EOF
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: quarantine
spec:
  podSelector:
    matchLabels:
      quarantine: "true"
  policyTypes:
  - Ingress
  - Egress
  # Pas de règles = deny all
EOF

# Révoquer l'accès d'un utilisateur
kubectl delete rolebinding user-alice-binding
kubectl delete clusterrolebinding user-alice-cluster-binding

# Forcer rotation des secrets
kubectl delete secret compromised-secret
kubectl create secret generic compromised-secret --from-literal=...

# Examiner les logs d'audit
grep "compromised-user" /var/log/kubernetes/audit.log | jq .

# Snapshot de l'état actuel
kubectl get all,cm,secret,sa,role,rolebinding --all-namespaces -o yaml > cluster-snapshot.yaml
```

## Ressources

### Documentation Officielle

- **Kubernetes Security** : https://kubernetes.io/docs/concepts/security/
- **CIS Kubernetes Benchmark** : https://www.cisecurity.org/benchmark/kubernetes
- **NSA/CISA Kubernetes Hardening Guide** : https://media.defense.gov/2022/Aug/29/2003066362/-1/-1/0/CTR_KUBERNETES_HARDENING_GUIDANCE_1.2_20220829.PDF
- **OWASP Kubernetes Top 10** : https://owasp.org/www-project-kubernetes-top-ten/

### Outils et Projets

- **Trivy** : https://github.com/aquasecurity/trivy
- **Falco** : https://falco.org/
- **OPA Gatekeeper** : https://open-policy-agent.github.io/gatekeeper/
- **Sealed Secrets** : https://github.com/bitnami-labs/sealed-secrets
- **External Secrets** : https://external-secrets.io/

### Formations et Certifications

- **CKA** : Certified Kubernetes Administrator
- **CKAD** : Certified Kubernetes Application Developer
- **CKS** : Certified Kubernetes Security Specialist
- **Kubernetes Security (Essentials)** : Linux Foundation (LFS260)
- **Kubernetes Security Specialization** : Coursera

## Conclusion

La sécurité Kubernetes n'est pas une destination mais un voyage continu :

```
Sécurité =
  Culture (Sensibilisation) +
  Processus (Bonnes pratiques) +
  Outils (Automatisation) +
  Monitoring (Vigilance)
```

**Les 10 Commandements de la Sécurité Kubernetes :**

1. **Tu garderas ton cluster à jour** - Patches réguliers
2. **Tu n'utiliseras point cluster-admin** - Moindre privilège
3. **Tu scanneras tes images** - Pas de vulnérabilités CRITICAL
4. **Tu n'exécuteras point en root** - runAsNonRoot: true
5. **Tu isoleras tes workloads** - Network Policies
6. **Tu chiffreras tes secrets** - Sealed Secrets minimum
7. **Tu auditeras tout** - Audit logging activé
8. **Tu limiteras les ressources** - Requests et Limits
9. **Tu sauvegarderas régulièrement** - Backups testés
10. **Tu monitoreras constamment** - Alertes configurées

**Points Clés à Retenir :**

- La sécurité est une **responsabilité partagée**
- Commencez **simple** et améliorez progressivement
- **Automatisez** autant que possible
- **Testez** régulièrement vos défenses
- **Formez** votre équipe continuellement
- **Documentez** tout
- Préparez-vous au **pire scénario**

**Prochaines Étapes :**

Dans la section suivante, nous verrons :
- **16.10** : Checklist de sécurité complète et finale

La sécurité Kubernetes demande de la vigilance et de l'amélioration continue. En suivant ces bonnes pratiques, vous construirez des clusters résilients et sécurisés. N'oubliez pas : **la sécurité parfaite n'existe pas, mais la sécurité suffisante est atteignable avec discipline et méthode**.

⏭️ [Checklist de sécurité](/16-securite-kubernetes/10-checklist-de-securite.md)
