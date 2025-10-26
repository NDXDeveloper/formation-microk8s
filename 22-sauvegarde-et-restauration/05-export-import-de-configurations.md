🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 22.5 Export/Import de configurations

## Introduction

L'**export et l'import de configurations** sont des compétences essentielles pour gérer efficacement votre cluster Kubernetes. Cette approche vous permet de sauvegarder vos configurations, de les partager, de migrer entre clusters, ou simplement de documenter votre infrastructure.

### Pourquoi exporter les configurations ?

Imaginez ces situations courantes :

1. **Migration** : Vous voulez déplacer vos applications vers un nouveau cluster
2. **Disaster Recovery** : Votre cluster est corrompu, vous devez tout recréer
3. **Duplication** : Vous voulez créer un environnement de test identique à la production
4. **Documentation** : Vous voulez garder une trace de votre configuration
5. **Versionnage** : Vous voulez suivre les changements dans Git
6. **Partage** : Vous voulez partager votre configuration avec d'autres

Sans export/import, vous devriez tout reconfigurer manuellement à chaque fois. Avec cette technique, c'est une simple commande.

### Analogie simple

Pensez à l'export/import de configurations comme à :

- **Export** = Prendre une photo de la recette de votre plat préféré
- **Import** = Utiliser cette recette pour recréer le plat ailleurs

La "recette" ici, ce sont vos fichiers YAML qui décrivent comment votre cluster doit être configuré.

## Concepts de base

### Qu'est-ce qu'une configuration Kubernetes ?

Dans Kubernetes, tout est défini par des **ressources** décrites en YAML (ou JSON) :

- **Deployments** : Comment déployer vos applications
- **Services** : Comment exposer vos applications
- **ConfigMaps** : Variables de configuration
- **Secrets** : Données sensibles
- **PersistentVolumeClaims** : Demandes de stockage
- **Ingress** : Règles de routage HTTP/HTTPS
- Et bien d'autres...

Chaque ressource a une définition que vous pouvez **exporter** (récupérer) et **importer** (recréer).

### Les deux approches principales

Il existe deux philosophies différentes :

#### 1. Approche impérative (commandes directes)

```bash
# Créer directement avec kubectl
kubectl create deployment nginx --image=nginx
kubectl expose deployment nginx --port=80
```

**Problème** : Difficile à reproduire, pas de traçabilité.

#### 2. Approche déclarative (fichiers YAML)

```bash
# Créer à partir d'un fichier
kubectl apply -f deployment.yaml
```

**Avantage** : Reproductible, versionnable, partageable.

L'export/import favorise naturellement l'approche déclarative, qui est la meilleure pratique.

### Format YAML vs JSON

Kubernetes accepte les deux formats :

**YAML** (recommandé) :
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx
spec:
  containers:
  - name: nginx
    image: nginx:latest
```

**JSON** (plus verbeux) :
```json
{
  "apiVersion": "v1",
  "kind": "Pod",
  "metadata": {
    "name": "nginx"
  },
  "spec": {
    "containers": [{
      "name": "nginx",
      "image": "nginx:latest"
    }]
  }
}
```

Dans ce guide, nous utilisons YAML car il est plus lisible et plus courant.

## Exporter des configurations

### Exporter une ressource unique

La commande de base pour exporter est `kubectl get` avec l'option `-o yaml` :

```bash
# Exporter un deployment
kubectl get deployment nginx -o yaml > nginx-deployment.yaml

# Exporter un service
kubectl get service nginx -o yaml > nginx-service.yaml

# Exporter un configmap
kubectl get configmap app-config -o yaml > app-config.yaml
```

### Exporter toutes les ressources d'un namespace

```bash
# Exporter tous les deployments
kubectl get deployments -n production -o yaml > all-deployments.yaml

# Exporter tous les services
kubectl get services -n production -o yaml > all-services.yaml

# Exporter tous les configmaps
kubectl get configmaps -n production -o yaml > all-configmaps.yaml
```

### Exporter plusieurs types de ressources

```bash
# Exporter deployments et services en un seul fichier
kubectl get deployments,services -n production -o yaml > app-resources.yaml
```

### Exporter TOUT dans un namespace

Pour exporter la plupart des ressources d'un namespace :

```bash
# Version simple
kubectl get all -n production -o yaml > production-all.yaml

# Version plus complète (inclut plus de types)
kubectl get all,configmaps,secrets,ingress,pvc -n production -o yaml > production-complete.yaml
```

**Note** : `all` ne capture pas vraiment TOUT. Il manque certains types comme ConfigMaps, Secrets, PVCs, etc.

### Exporter avec des filtres

Exporter seulement les ressources avec un label spécifique :

```bash
# Exporter les ressources de l'app "webapp"
kubectl get all -l app=webapp -o yaml > webapp-resources.yaml

# Exporter les ressources de production
kubectl get all -l environment=production -o yaml > prod-resources.yaml
```

### Nettoyer les métadonnées pour le réimport

Les fichiers exportés contiennent des métadonnées gérées par Kubernetes qu'il faut souvent nettoyer :

```bash
# Exporter et nettoyer en une commande
kubectl get deployment nginx -o yaml | \
  kubectl neat > nginx-deployment-clean.yaml
```

**Note** : `kubectl neat` n'est pas installé par défaut. Installation :

```bash
kubectl krew install neat
```

Alternative sans `neat`, utiliser `yq` :

```bash
kubectl get deployment nginx -o yaml | \
  yq 'del(.metadata.resourceVersion, .metadata.uid, .metadata.creationTimestamp, .status)' \
  > nginx-deployment-clean.yaml
```

## Nettoyer manuellement les exports

### Champs à supprimer avant réimport

Voici un exemple de fichier exporté brut :

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx
  namespace: production
  uid: 12345-67890-abcdef  # À SUPPRIMER
  resourceVersion: "123456"  # À SUPPRIMER
  creationTimestamp: "2025-01-15T10:00:00Z"  # À SUPPRIMER
  generation: 1  # À SUPPRIMER
  annotations:
    deployment.kubernetes.io/revision: "1"  # Peut être gardé
    kubectl.kubernetes.io/last-applied-configuration: |  # À SUPPRIMER (très long)
      {...}
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:latest
status:  # TOUTE LA SECTION status À SUPPRIMER
  replicas: 3
  readyReplicas: 3
  conditions: [...]
```

Après nettoyage :

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx
  namespace: production
  annotations:
    deployment.kubernetes.io/revision: "1"
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:latest
```

### Script de nettoyage automatique

Créez un script `clean-export.sh` :

```bash
#!/bin/bash
# Script pour nettoyer les exports Kubernetes

INPUT_FILE=$1
OUTPUT_FILE=$2

if [ -z "$INPUT_FILE" ] || [ -z "$OUTPUT_FILE" ]; then
  echo "Usage: $0 input.yaml output.yaml"
  exit 1
fi

# Utiliser yq pour nettoyer
yq eval '
  del(.metadata.uid) |
  del(.metadata.resourceVersion) |
  del(.metadata.creationTimestamp) |
  del(.metadata.generation) |
  del(.metadata.managedFields) |
  del(.metadata.selfLink) |
  del(.metadata.annotations."kubectl.kubernetes.io/last-applied-configuration") |
  del(.status)
' "$INPUT_FILE" > "$OUTPUT_FILE"

echo "Fichier nettoyé: $OUTPUT_FILE"
```

Utilisation :

```bash
./clean-export.sh nginx-raw.yaml nginx-clean.yaml
```

## Importer des configurations

### Importer une ressource unique

```bash
# Créer la ressource depuis un fichier
kubectl apply -f nginx-deployment.yaml

# Vérifier
kubectl get deployment nginx
```

### Importer plusieurs fichiers

```bash
# Appliquer tous les fichiers YAML d'un dossier
kubectl apply -f ./configs/

# Ou spécifiquement
kubectl apply -f deployment.yaml -f service.yaml -f configmap.yaml
```

### Importer depuis un fichier multi-ressources

Si vous avez exporté plusieurs ressources dans un seul fichier :

```bash
kubectl apply -f production-all.yaml
```

Kubernetes créera toutes les ressources définies dans le fichier.

### Importer dans un namespace différent

```bash
# Créer le namespace d'abord si nécessaire
kubectl create namespace staging

# Importer en forçant le namespace
kubectl apply -f production-all.yaml -n staging
```

**Attention** : Si le fichier spécifie déjà un namespace dans `metadata.namespace`, vous devrez le modifier dans le fichier ou utiliser des outils comme `yq`.

### Modifier le namespace à l'import

```bash
# Changer le namespace dans tous les fichiers d'un coup
find ./configs -name "*.yaml" -exec sed -i 's/namespace: production/namespace: staging/g' {} \;

# Ou avec yq
yq eval '.metadata.namespace = "staging"' production-config.yaml > staging-config.yaml
kubectl apply -f staging-config.yaml
```

### Import avec validation préalable

Avant d'appliquer, vous pouvez valider :

```bash
# Mode dry-run (ne crée rien)
kubectl apply -f deployment.yaml --dry-run=client

# Afficher les différences
kubectl diff -f deployment.yaml

# Validation serveur (teste contre l'API Kubernetes)
kubectl apply -f deployment.yaml --dry-run=server
```

## Stratégies d'organisation des exports

### Organisation par namespace

```
exports/
├── production/
│   ├── deployments.yaml
│   ├── services.yaml
│   ├── configmaps.yaml
│   └── secrets.yaml
├── staging/
│   ├── deployments.yaml
│   ├── services.yaml
│   └── configmaps.yaml
└── development/
    └── all.yaml
```

Script d'export :

```bash
#!/bin/bash
# export-by-namespace.sh

NAMESPACES="production staging development"
EXPORT_DIR="./exports"

mkdir -p $EXPORT_DIR

for ns in $NAMESPACES; do
  echo "Export du namespace: $ns"
  mkdir -p "$EXPORT_DIR/$ns"

  kubectl get deployments -n $ns -o yaml > "$EXPORT_DIR/$ns/deployments.yaml"
  kubectl get services -n $ns -o yaml > "$EXPORT_DIR/$ns/services.yaml"
  kubectl get configmaps -n $ns -o yaml > "$EXPORT_DIR/$ns/configmaps.yaml"
  kubectl get secrets -n $ns -o yaml > "$EXPORT_DIR/$ns/secrets.yaml"
done

echo "Export terminé dans: $EXPORT_DIR"
```

### Organisation par application

```
exports/
├── webapp/
│   ├── deployment.yaml
│   ├── service.yaml
│   ├── ingress.yaml
│   └── configmap.yaml
├── database/
│   ├── statefulset.yaml
│   ├── service.yaml
│   ├── pvc.yaml
│   └── secret.yaml
└── monitoring/
    └── all.yaml
```

Script d'export :

```bash
#!/bin/bash
# export-by-app.sh

APPS="webapp database monitoring"
EXPORT_DIR="./exports"

mkdir -p $EXPORT_DIR

for app in $APPS; do
  echo "Export de l'application: $app"
  mkdir -p "$EXPORT_DIR/$app"

  kubectl get all,configmaps,secrets,ingress,pvc \
    -l app=$app \
    -o yaml \
    > "$EXPORT_DIR/$app/all.yaml"
done

echo "Export terminé dans: $EXPORT_DIR"
```

### Organisation par type de ressource

```
exports/
├── deployments/
│   ├── nginx.yaml
│   ├── webapp.yaml
│   └── api.yaml
├── services/
│   ├── nginx.yaml
│   └── webapp.yaml
├── configmaps/
│   └── app-config.yaml
└── secrets/
    └── db-credentials.yaml
```

## Scripts d'export avancés

### Script d'export complet et organisé

```bash
#!/bin/bash
# full-export.sh - Export complet du cluster

EXPORT_DIR="./cluster-export-$(date +%Y%m%d-%H%M%S)"
mkdir -p "$EXPORT_DIR"

echo "=== Export du cluster MicroK8s ==="
echo "Destination: $EXPORT_DIR"

# 1. Exporter les namespaces
echo "Export des namespaces..."
kubectl get namespaces -o yaml > "$EXPORT_DIR/namespaces.yaml"

# 2. Pour chaque namespace utilisateur
NAMESPACES=$(kubectl get namespaces -o jsonpath='{.items[*].metadata.name}' | tr ' ' '\n' | grep -v 'kube-\|default')

for ns in $NAMESPACES; do
  echo "Export du namespace: $ns"
  NS_DIR="$EXPORT_DIR/namespaces/$ns"
  mkdir -p "$NS_DIR"

  # Deployments
  kubectl get deployments -n $ns -o yaml > "$NS_DIR/deployments.yaml" 2>/dev/null

  # StatefulSets
  kubectl get statefulsets -n $ns -o yaml > "$NS_DIR/statefulsets.yaml" 2>/dev/null

  # DaemonSets
  kubectl get daemonsets -n $ns -o yaml > "$NS_DIR/daemonsets.yaml" 2>/dev/null

  # Services
  kubectl get services -n $ns -o yaml > "$NS_DIR/services.yaml" 2>/dev/null

  # Ingress
  kubectl get ingress -n $ns -o yaml > "$NS_DIR/ingress.yaml" 2>/dev/null

  # ConfigMaps
  kubectl get configmaps -n $ns -o yaml > "$NS_DIR/configmaps.yaml" 2>/dev/null

  # Secrets (attention aux données sensibles !)
  kubectl get secrets -n $ns -o yaml > "$NS_DIR/secrets.yaml" 2>/dev/null

  # PVCs
  kubectl get pvc -n $ns -o yaml > "$NS_DIR/pvc.yaml" 2>/dev/null

  # ServiceAccounts
  kubectl get serviceaccounts -n $ns -o yaml > "$NS_DIR/serviceaccounts.yaml" 2>/dev/null

  # NetworkPolicies
  kubectl get networkpolicies -n $ns -o yaml > "$NS_DIR/networkpolicies.yaml" 2>/dev/null
done

# 3. Ressources au niveau cluster
echo "Export des ressources cluster..."
CLUSTER_DIR="$EXPORT_DIR/cluster-resources"
mkdir -p "$CLUSTER_DIR"

kubectl get clusterroles -o yaml > "$CLUSTER_DIR/clusterroles.yaml" 2>/dev/null
kubectl get clusterrolebindings -o yaml > "$CLUSTER_DIR/clusterrolebindings.yaml" 2>/dev/null
kubectl get storageclasses -o yaml > "$CLUSTER_DIR/storageclasses.yaml" 2>/dev/null
kubectl get persistentvolumes -o yaml > "$CLUSTER_DIR/persistentvolumes.yaml" 2>/dev/null

# 4. Créer un README
cat > "$EXPORT_DIR/README.md" <<EOF
# Export du cluster MicroK8s

**Date d'export:** $(date)
**Cluster:** $(kubectl config current-context)

## Structure

- \`namespaces.yaml\` - Définition des namespaces
- \`namespaces/\` - Ressources par namespace
- \`cluster-resources/\` - Ressources au niveau cluster

## Import

Pour réimporter dans un nouveau cluster:

\`\`\`bash
# 1. Créer les namespaces
kubectl apply -f namespaces.yaml

# 2. Importer les ressources de chaque namespace
kubectl apply -f namespaces/production/
kubectl apply -f namespaces/staging/

# 3. Importer les ressources cluster
kubectl apply -f cluster-resources/
\`\`\`

## Notes

- Les Secrets sont inclus (attention aux données sensibles)
- Les PersistentVolumes peuvent nécessiter une adaptation
- Vérifiez les StorageClasses avant import
EOF

echo ""
echo "Export terminé: $EXPORT_DIR"
echo "Voir README.md pour les instructions d'import"
```

Utilisation :

```bash
chmod +x full-export.sh
./full-export.sh
```

### Script d'export sélectif

Pour exporter seulement certaines applications :

```bash
#!/bin/bash
# selective-export.sh - Export sélectif

APP_LABEL=$1
EXPORT_FILE="${APP_LABEL}-export.yaml"

if [ -z "$APP_LABEL" ]; then
  echo "Usage: $0 <app-label>"
  echo "Exemple: $0 webapp"
  exit 1
fi

echo "Export de l'application avec le label: app=$APP_LABEL"

kubectl get all,configmaps,secrets,ingress,pvc \
  -l app=$APP_LABEL \
  --all-namespaces \
  -o yaml > "$EXPORT_FILE"

echo "Export créé: $EXPORT_FILE"
```

Utilisation :

```bash
./selective-export.sh webapp
```

## Cas d'usage pratiques

### Cas 1 : Migration vers un nouveau cluster

**Scénario** : Vous passez d'un ancien serveur à un nouveau.

**Étapes** :

1. **Sur l'ancien cluster** :

```bash
# Export complet
./full-export.sh

# Archiver
tar -czf cluster-backup.tar.gz cluster-export-*/
```

2. **Sur le nouveau cluster** :

```bash
# Extraire
tar -xzf cluster-backup.tar.gz
cd cluster-export-*/

# Créer les namespaces
kubectl apply -f namespaces.yaml

# Importer les ressources
for ns in namespaces/*/; do
  echo "Import de $(basename $ns)"
  kubectl apply -f "$ns"
done

# Ressources cluster
kubectl apply -f cluster-resources/
```

3. **Vérifications** :

```bash
# Vérifier les deployments
kubectl get deployments --all-namespaces

# Vérifier les services
kubectl get services --all-namespaces

# Vérifier les pods
kubectl get pods --all-namespaces
```

### Cas 2 : Créer un environnement de staging

**Scénario** : Dupliquer production vers staging.

**Script** :

```bash
#!/bin/bash
# clone-to-staging.sh

SOURCE_NS="production"
TARGET_NS="staging"

# Créer le namespace staging
kubectl create namespace $TARGET_NS --dry-run=client -o yaml | kubectl apply -f -

# Exporter production
kubectl get all,configmaps,ingress,pvc -n $SOURCE_NS -o yaml > /tmp/prod-export.yaml

# Modifier le namespace et certains paramètres
cat /tmp/prod-export.yaml | \
  sed "s/namespace: $SOURCE_NS/namespace: $TARGET_NS/g" | \
  sed "s/replicas: 3/replicas: 1/g" | \
  sed "s/cpu: \"1\"/cpu: \"500m\"/g" | \
  sed "s/memory: 2Gi/memory: 1Gi/g" \
  > /tmp/staging-import.yaml

# Nettoyer les métadonnées
yq eval '
  del(.metadata.uid) |
  del(.metadata.resourceVersion) |
  del(.metadata.creationTimestamp) |
  del(.status)
' /tmp/staging-import.yaml > staging-ready.yaml

# Importer
kubectl apply -f staging-ready.yaml

echo "Environnement staging créé depuis production"
```

### Cas 3 : Documentation as Code

**Scénario** : Garder une trace versionnée de votre infrastructure.

**Structure Git** :

```
infrastructure/
├── README.md
├── production/
│   ├── webapp/
│   │   ├── deployment.yaml
│   │   ├── service.yaml
│   │   └── ingress.yaml
│   └── database/
│       ├── statefulset.yaml
│       └── service.yaml
├── staging/
│   └── ...
└── scripts/
    ├── export.sh
    └── apply.sh
```

**Workflow** :

```bash
# 1. Exporter régulièrement
./scripts/export.sh

# 2. Commiter dans Git
git add .
git commit -m "Update cluster configuration"
git push

# 3. Sur un autre cluster, récupérer et appliquer
git pull
./scripts/apply.sh
```

### Cas 4 : Rollback après une mauvaise configuration

**Scénario** : Une modification a cassé votre application.

```bash
# 1. Exporter l'état actuel (cassé)
kubectl get deployment webapp -n production -o yaml > webapp-broken.yaml

# 2. Restaurer depuis un export précédent (dans Git)
git checkout HEAD~1 production/webapp/deployment.yaml
kubectl apply -f production/webapp/deployment.yaml

# 3. Vérifier
kubectl rollout status deployment/webapp -n production

# 4. Si OK, commiter le rollback
git add production/webapp/deployment.yaml
git commit -m "Rollback webapp deployment"
```

### Cas 5 : Partager une configuration

**Scénario** : Partager votre configuration d'application avec un collègue.

```bash
# Exporter seulement les ressources nécessaires
kubectl get deployment,service,configmap,ingress \
  -l app=myapp \
  -o yaml > myapp-config.yaml

# Nettoyer les données sensibles
sed -i '/password:/d' myapp-config.yaml

# Partager le fichier
# Votre collègue peut maintenant faire:
kubectl apply -f myapp-config.yaml
```

## Gestion des Secrets lors de l'export

Les Secrets contiennent des données sensibles (mots de passe, clés API, certificats). Il faut faire attention lors de l'export.

### Exporter des Secrets (approche prudente)

```bash
# Exporter SANS les données
kubectl get secrets -o yaml | \
  yq 'del(.items[].data)' \
  > secrets-template.yaml
```

Cela crée un template où vous pourrez remplir les valeurs plus tard.

### Chiffrer les Secrets exportés

Si vous devez exporter des secrets avec leurs valeurs :

```bash
# Exporter
kubectl get secrets -n production -o yaml > secrets.yaml

# Chiffrer avec GPG
gpg --encrypt --recipient votre-email@exemple.com secrets.yaml

# Résultat : secrets.yaml.gpg (chiffré)
rm secrets.yaml  # Supprimer la version non chiffrée

# Pour déchiffrer plus tard
gpg --decrypt secrets.yaml.gpg > secrets.yaml
```

### Alternative : Sealed Secrets

**Sealed Secrets** est un outil qui permet de chiffrer des Secrets de manière sûre pour Git.

```bash
# Installation du controller
kubectl apply -f https://github.com/bitnami-labs/sealed-secrets/releases/download/v0.24.0/controller.yaml

# Installation du CLI
wget https://github.com/bitnami-labs/sealed-secrets/releases/download/v0.24.0/kubeseal-linux-amd64
sudo install -m 755 kubeseal-linux-amd64 /usr/local/bin/kubeseal

# Utilisation
kubectl create secret generic mysecret \
  --from-literal=password=SuperSecret123 \
  --dry-run=client -o yaml | \
  kubeseal -o yaml > mysealedsecret.yaml

# Vous pouvez commiter mysealedsecret.yaml en toute sécurité
# Le controller déchiffrera automatiquement dans le cluster
kubectl apply -f mysealedsecret.yaml
```

### Exclure les Secrets des exports automatiques

Dans vos scripts, vous pouvez exclure les Secrets :

```bash
# Exporter tout SAUF les secrets
kubectl get all,configmaps,ingress,pvc -n production -o yaml > export-no-secrets.yaml
```

## Outils avancés pour l'export/import

### kubectl kustomize

**Kustomize** permet de personnaliser des configurations sans modifier les fichiers originaux.

**Structure** :

```
app/
├── base/
│   ├── deployment.yaml
│   ├── service.yaml
│   └── kustomization.yaml
├── overlays/
│   ├── production/
│   │   ├── kustomization.yaml
│   │   └── replicas.yaml
│   └── staging/
│       ├── kustomization.yaml
│       └── replicas.yaml
```

**base/kustomization.yaml** :

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - deployment.yaml
  - service.yaml
```

**overlays/production/kustomization.yaml** :

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
bases:
  - ../../base
patchesStrategicMerge:
  - replicas.yaml
namespace: production
```

**Utilisation** :

```bash
# Générer et appliquer pour production
kubectl apply -k overlays/production/

# Générer et appliquer pour staging
kubectl apply -k overlays/staging/
```

### Helm Charts

**Helm** est un gestionnaire de packages pour Kubernetes.

**Créer un Chart depuis des manifestes existants** :

```bash
# Créer la structure
helm create mychart

# Placer vos manifestes dans templates/
cp deployment.yaml mychart/templates/
cp service.yaml mychart/templates/

# Paramétrer avec values.yaml
# Puis installer
helm install myapp mychart/
```

### yq - Le couteau suisse YAML

**yq** est un outil CLI puissant pour manipuler YAML.

```bash
# Installation
wget https://github.com/mikefarah/yq/releases/latest/download/yq_linux_amd64
sudo mv yq_linux_amd64 /usr/local/bin/yq
sudo chmod +x /usr/local/bin/yq

# Exemples d'utilisation

# Changer une valeur
yq eval '.spec.replicas = 5' deployment.yaml

# Supprimer un champ
yq eval 'del(.status)' deployment.yaml

# Merger plusieurs fichiers
yq eval-all 'select(fileIndex == 0) * select(fileIndex == 1)' file1.yaml file2.yaml

# Filtrer par label
yq eval 'select(.metadata.labels.app == "webapp")' all-resources.yaml
```

### kubeval - Validation de manifestes

**kubeval** valide la syntaxe de vos fichiers YAML.

```bash
# Installation
wget https://github.com/instrumenta/kubeval/releases/latest/download/kubeval-linux-amd64.tar.gz
tar xf kubeval-linux-amd64.tar.gz
sudo mv kubeval /usr/local/bin/

# Validation
kubeval deployment.yaml

# Valider tous les fichiers d'un dossier
kubeval configs/*.yaml
```

## Automatisation avec GitOps

Le GitOps pousse l'export/import à l'extrême : **Git devient la source de vérité** pour votre cluster.

### Principe GitOps

1. Toutes les configurations sont dans Git
2. Tout changement passe par Git (PR, review)
3. Un outil synchronise automatiquement Git → Cluster

### ArgoCD (outil GitOps populaire)

**Installation** :

```bash
# Créer le namespace
kubectl create namespace argocd

# Installer ArgoCD
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

# Exposer l'interface
kubectl port-forward svc/argocd-server -n argocd 8080:443
```

**Configuration** :

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: myapp
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/votre-user/k8s-configs
    targetRevision: main
    path: production/myapp
  destination:
    server: https://kubernetes.default.svc
    namespace: production
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
```

Avec cette configuration, ArgoCD :
- Surveille votre repo Git
- Applique automatiquement les changements
- Répare le cluster si modifié manuellement

### Flux (alternative à ArgoCD)

**Installation** :

```bash
# Installer Flux CLI
curl -s https://fluxcd.io/install.sh | sudo bash

# Bootstrap Flux sur votre cluster
flux bootstrap github \
  --owner=votre-user \
  --repository=k8s-configs \
  --path=clusters/microk8s \
  --personal
```

Flux crée automatiquement la structure dans votre repo Git.

## Bonnes pratiques

### 1. Versionner TOUT dans Git

```
k8s-configs/
├── .gitignore  # Ignorer les secrets non chiffrés
├── README.md
├── clusters/
│   ├── production/
│   │   ├── apps/
│   │   ├── infrastructure/
│   │   └── README.md
│   └── staging/
│       └── ...
└── scripts/
    ├── export.sh
    └── apply.sh
```

**.gitignore** :

```
# Ne jamais commiter ces fichiers
secrets-raw.yaml
*-secret.yaml
*.gpg
credentials.txt

# Autoriser les secrets chiffrés
!*-sealedsecret.yaml
```

### 2. Séparer configuration et secrets

```
production/
├── deployments/
│   └── webapp.yaml
├── services/
│   └── webapp.yaml
├── configmaps/
│   └── app-config.yaml
└── secrets/  # Dossier séparé, avec .gitignore
    └── db-credentials.yaml
```

### 3. Documenter vos configurations

Chaque application devrait avoir un README :

```markdown
# Webapp

## Description
Application web principale

## Déploiement
\`\`\`bash
kubectl apply -f deployments/webapp.yaml
kubectl apply -f services/webapp.yaml
\`\`\`

## Configuration
- Replicas: 3
- Image: webapp:v1.2.3
- Port: 8080

## Secrets requis
- `db-credentials` : Username et password pour PostgreSQL

## Dépendances
- PostgreSQL database
- Redis cache
```

### 4. Utiliser des labels cohérents

Tous vos manifestes devraient avoir des labels standards :

```yaml
metadata:
  labels:
    app: webapp
    version: v1.2.3
    environment: production
    component: frontend
    managed-by: helm
```

Cela facilite les exports sélectifs :

```bash
kubectl get all -l app=webapp,environment=production -o yaml
```

### 5. Tester avant d'appliquer

```bash
# Toujours utiliser --dry-run first
kubectl apply -f config.yaml --dry-run=client

# Vérifier les différences
kubectl diff -f config.yaml

# Appliquer progressivement
kubectl apply -f deployments/ --dry-run=server
kubectl apply -f services/ --dry-run=server
kubectl apply -f configmaps/ --dry-run=server

# Si tout est OK
kubectl apply -f .
```

### 6. Automatiser les exports réguliers

Créez un CronJob qui exporte régulièrement :

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: cluster-export
  namespace: default
spec:
  schedule: "0 2 * * *"  # Tous les jours à 2h
  jobTemplate:
    spec:
      template:
        spec:
          serviceAccountName: cluster-exporter
          containers:
          - name: exporter
            image: bitnami/kubectl:latest
            command:
            - /bin/bash
            - -c
            - |
              DATE=$(date +%Y%m%d)
              EXPORT_DIR="/exports/$DATE"
              mkdir -p $EXPORT_DIR

              # Export
              kubectl get all --all-namespaces -o yaml > $EXPORT_DIR/all.yaml
              kubectl get configmaps --all-namespaces -o yaml > $EXPORT_DIR/configmaps.yaml
              kubectl get ingress --all-namespaces -o yaml > $EXPORT_DIR/ingress.yaml

              echo "Export terminé: $EXPORT_DIR"
            volumeMounts:
            - name: exports
              mountPath: /exports
          volumes:
          - name: exports
            persistentVolumeClaim:
              claimName: export-storage
          restartPolicy: OnFailure
```

### 7. Maintenir un changelog

Gardez une trace des changements importants :

```markdown
# Changelog Cluster Production

## 2025-01-15
- Ajout de l'application monitoring (Prometheus + Grafana)
- Mise à jour webapp: v1.2.3 → v1.3.0
- Ajout de NetworkPolicy pour la base de données

## 2025-01-10
- Migration de staging vers nouveau serveur
- Augmentation replicas webapp: 2 → 3
- Configuration SSL avec cert-manager

## 2025-01-05
- Déploiement initial du cluster
- Applications: webapp, database, redis
```

## Troubleshooting

### Problème : Import échoue avec "already exists"

**Cause** : La ressource existe déjà dans le cluster.

**Solutions** :

```bash
# Option 1: Utiliser --force pour recréer
kubectl apply -f config.yaml --force

# Option 2: Supprimer puis recréer
kubectl delete -f config.yaml
kubectl apply -f config.yaml

# Option 3: Utiliser replace au lieu d'apply
kubectl replace -f config.yaml
```

### Problème : Namespace incorrect après import

**Cause** : Le fichier spécifie un namespace qui n'existe pas ou diffère.

**Solution** :

```bash
# Créer le namespace d'abord
kubectl create namespace production

# Ou forcer le namespace
kubectl apply -f config.yaml -n production
```

### Problème : Secrets non décodés

**Cause** : Les secrets sont en base64 dans Kubernetes.

**Solution** :

```bash
# Décoder un secret
kubectl get secret mysecret -o jsonpath='{.data.password}' | base64 --decode

# Encoder pour un nouveau secret
echo -n "MonMotDePasse" | base64
# Résultat: TW9uTW90RGVQYXNzZQ==
```

### Problème : PVCs ne se lient pas

**Cause** : StorageClass ou PV différent sur le nouveau cluster.

**Solution** :

```bash
# Vérifier les StorageClasses disponibles
kubectl get storageclass

# Modifier le fichier PVC pour utiliser la bonne StorageClass
yq eval '.spec.storageClassName = "microk8s-hostpath"' pvc.yaml | kubectl apply -f -
```

### Problème : Trop de manifestes à gérer

**Solution** : Utilisez Kustomize ou Helm.

```bash
# Avec Kustomize
cat > kustomization.yaml <<EOF
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - deployment.yaml
  - service.yaml
  - configmap.yaml
  - ingress.yaml
EOF

kubectl apply -k .
```

## Scripts de référence

### Script d'export intelligent

```bash
#!/bin/bash
# smart-export.sh - Export intelligent avec nettoyage

set -e

NAMESPACE=${1:-"default"}
OUTPUT_DIR="./export-${NAMESPACE}-$(date +%Y%m%d-%H%M%S)"

mkdir -p "$OUTPUT_DIR"

echo "=== Export du namespace: $NAMESPACE ==="

# Fonction de nettoyage
clean_yaml() {
  local input=$1
  local output=$2

  yq eval '
    del(.metadata.uid) |
    del(.metadata.resourceVersion) |
    del(.metadata.creationTimestamp) |
    del(.metadata.generation) |
    del(.metadata.managedFields) |
    del(.metadata.selfLink) |
    del(.metadata.annotations."kubectl.kubernetes.io/last-applied-configuration") |
    del(.status)
  ' "$input" > "$output"
}

# Export des ressources
RESOURCES="deployments statefulsets daemonsets services ingresses configmaps secrets persistentvolumeclaims"

for resource in $RESOURCES; do
  echo "Export: $resource"
  kubectl get $resource -n $NAMESPACE -o yaml > "$OUTPUT_DIR/${resource}-raw.yaml" 2>/dev/null || continue

  if [ -s "$OUTPUT_DIR/${resource}-raw.yaml" ]; then
    clean_yaml "$OUTPUT_DIR/${resource}-raw.yaml" "$OUTPUT_DIR/${resource}.yaml"
    rm "$OUTPUT_DIR/${resource}-raw.yaml"
  fi
done

# Créer un fichier all-in-one
echo "Création du fichier all.yaml..."
cat $OUTPUT_DIR/*.yaml > "$OUTPUT_DIR/all.yaml" 2>/dev/null

# README
cat > "$OUTPUT_DIR/README.md" <<EOF
# Export du namespace: $NAMESPACE

Date: $(date)

## Fichiers

$(ls -1 *.yaml | sed 's/^/- /')

## Import

Pour importer dans un nouveau cluster:

\`\`\`bash
# Créer le namespace
kubectl create namespace $NAMESPACE

# Importer tout
kubectl apply -f all.yaml -n $NAMESPACE

# Ou importer sélectivement
kubectl apply -f deployments.yaml -n $NAMESPACE
kubectl apply -f services.yaml -n $NAMESPACE
\`\`\`
EOF

echo ""
echo "✓ Export terminé: $OUTPUT_DIR"
ls -lh "$OUTPUT_DIR"
```

### Script d'import avec validation

```bash
#!/bin/bash
# smart-import.sh - Import avec validation

set -e

IMPORT_DIR=$1
NAMESPACE=${2:-"default"}

if [ -z "$IMPORT_DIR" ] || [ ! -d "$IMPORT_DIR" ]; then
  echo "Usage: $0 <import-directory> [namespace]"
  exit 1
fi

echo "=== Import dans le namespace: $NAMESPACE ==="

# Créer le namespace si nécessaire
kubectl create namespace $NAMESPACE --dry-run=client -o yaml | kubectl apply -f -

# Ordre d'import (important !)
ORDER="configmaps secrets persistentvolumeclaims services deployments statefulsets daemonsets ingresses"

for resource in $ORDER; do
  FILE="$IMPORT_DIR/${resource}.yaml"

  if [ -f "$FILE" ]; then
    echo "Import: $resource"

    # Dry-run d'abord
    echo "  Validation..."
    kubectl apply -f "$FILE" -n $NAMESPACE --dry-run=server

    # Import réel
    echo "  Application..."
    kubectl apply -f "$FILE" -n $NAMESPACE

    echo "  ✓ OK"
  fi
done

echo ""
echo "=== Vérification ==="
kubectl get all -n $NAMESPACE

echo ""
echo "✓ Import terminé"
```

## Conclusion

L'export et l'import de configurations sont des compétences fondamentales pour gérer efficacement un cluster Kubernetes. Ces techniques vous permettent de :

### Récapitulatif des avantages

✅ **Sauvegarder** : Protection contre les pertes de configuration
✅ **Migrer** : Déplacer facilement vers un nouveau cluster
✅ **Dupliquer** : Créer des environnements de test rapidement
✅ **Documenter** : Garder une trace de votre infrastructure
✅ **Versionner** : Suivre l'évolution dans Git
✅ **Partager** : Collaborer efficacement
✅ **Automatiser** : GitOps pour une gestion continue

### Progression recommandée

**Niveau débutant** :
- Export/import manuel avec kubectl
- Organisation basique des fichiers
- Versionnage dans Git

**Niveau intermédiaire** :
- Scripts automatisés d'export/import
- Nettoyage des métadonnées
- Gestion des secrets avec Sealed Secrets
- Utilisation de yq et kubeval

**Niveau avancé** :
- GitOps avec ArgoCD ou Flux
- Kustomize ou Helm pour la réutilisabilité
- CI/CD complet
- Infrastructure as Code mature

### Prochaines étapes

Dans la section suivante (22.6), nous aborderons les tests de restauration, essentiels pour s'assurer que vos exports et sauvegardes fonctionnent réellement en cas de besoin.

---

**Checklist Export/Import** :

- [ ] Maîtriser kubectl get -o yaml
- [ ] Créer des scripts d'export personnalisés
- [ ] Nettoyer les métadonnées avant import
- [ ] Organiser les exports de manière cohérente
- [ ] Versionner les configurations dans Git
- [ ] Gérer les Secrets de manière sécurisée
- [ ] Tester régulièrement les imports
- [ ] Documenter les configurations
- [ ] Automatiser avec CronJobs ou GitOps
- [ ] Établir un workflow d'export/import régulier

⏭️ [Tests de restauration](/22-sauvegarde-et-restauration/06-tests-de-restauration.md)
