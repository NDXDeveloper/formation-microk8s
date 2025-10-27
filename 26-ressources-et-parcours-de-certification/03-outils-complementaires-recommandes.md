🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 26.3 Outils Complémentaires Recommandés

## Introduction

MicroK8s et kubectl sont puissants, mais l'écosystème Kubernetes offre une multitude d'outils complémentaires qui peuvent considérablement améliorer votre productivité, faciliter votre apprentissage et enrichir votre expérience quotidienne.

Cette section vous présente les outils les plus utiles et populaires, organisés par catégorie. Vous découvrirez leurs avantages, comment les installer, et dans quels cas les utiliser. Tous ces outils sont open-source et gratuits.

### Pourquoi Utiliser des Outils Complémentaires ?

**Productivité** : Accomplir plus en moins de temps

**Visualisation** : Comprendre l'état de votre cluster en un coup d'œil

**Simplification** : Rendre des tâches complexes plus simples

**Apprentissage** : Découvrir des fonctionnalités de manière interactive

**Automatisation** : Standardiser et automatiser les workflows

### Comment Aborder cette Section

Vous n'avez pas besoin d'installer tous ces outils immédiatement. Commencez par ceux qui correspondent à vos besoins actuels, puis explorez progressivement les autres au fil de votre parcours.

## Outils d'Interface en Ligne de Commande (CLI)

### k9s - Terminal UI pour Kubernetes

**Description** : Interface TUI (Text User Interface) interactive pour gérer vos clusters Kubernetes depuis le terminal.

**Pourquoi l'utiliser ?**
- Navigation intuitive avec les flèches du clavier
- Visualisation en temps réel des ressources
- Actions rapides (delete, edit, logs, shell)
- Colorisation et mise en forme automatique
- Filtrage et recherche instantanés
- Consommation mémoire légère

**Installation**

Sur Linux :
```bash
# Via snap
sudo snap install k9s

# Via wget
wget https://github.com/derailed/k9s/releases/latest/download/k9s_Linux_amd64.tar.gz
tar xzf k9s_Linux_amd64.tar.gz
sudo mv k9s /usr/local/bin/
```

Sur macOS :
```bash
brew install k9s
```

Sur Windows :
```powershell
choco install k9s
```

**Configuration pour MicroK8s**
```bash
# Créer un alias pour utiliser directement avec MicroK8s
microk8s config > ~/.kube/config
k9s
```

**Utilisation de Base**

Lancement :
```bash
k9s
```

**Raccourcis Clavier Essentiels** :
- `:pods` - Voir tous les pods
- `:svc` - Voir tous les services
- `:deploy` - Voir tous les deployments
- `/` - Filtrer les ressources
- `l` - Voir les logs d'un pod
- `d` - Describe une ressource
- `e` - Éditer une ressource
- `s` - Shell dans un conteneur
- `ctrl+d` - Supprimer une ressource
- `?` - Aide sur tous les raccourcis
- `:q` - Quitter

**Avantages pour Débutants** :
- Exploration visuelle du cluster
- Pas besoin de mémoriser toutes les commandes kubectl
- Feedback immédiat sur l'état des ressources
- Apprentissage des relations entre objets

### kubectx et kubens - Changement de Contexte Simplifié

**Description** : Deux outils pour gérer facilement les contextes et namespaces.

**kubectx** : Switcher entre clusters/contextes
**kubens** : Switcher entre namespaces

**Pourquoi les utiliser ?**
- Commandes ultra-courtes
- Évite les erreurs de typage
- Gain de temps considérable
- Support de l'autocomplétion
- Retour au contexte/namespace précédent

**Installation**

Sur Linux :
```bash
# Via Git
sudo git clone https://github.com/ahmetb/kubectx /opt/kubectx
sudo ln -s /opt/kubectx/kubectx /usr/local/bin/kubectx
sudo ln -s /opt/kubectx/kubens /usr/local/bin/kubens
```

Sur macOS :
```bash
brew install kubectx
```

**Utilisation**

```bash
# Lister les contextes disponibles
kubectx

# Changer de contexte
kubectx microk8s

# Retour au contexte précédent
kubectx -

# Lister les namespaces
kubens

# Changer de namespace
kubens kube-system

# Retour au namespace précédent
kubens -
```

**Cas d'Usage** :
- Gérer plusieurs clusters (dev, staging, prod)
- Travailler avec différents namespaces régulièrement
- Éviter de taper `--namespace` à chaque commande

### krew - Gestionnaire de Plugins kubectl

**Description** : Le "apt" ou "brew" pour les plugins kubectl.

**Pourquoi l'utiliser ?**
- Découvrir et installer des plugins facilement
- Gestion centralisée des plugins
- Mises à jour simplifiées
- Large catalogue de plugins

**Installation**

Sur Linux et macOS :
```bash
(
  set -x; cd "$(mktemp -d)" &&
  OS="$(uname | tr '[:upper:]' '[:lower:]')" &&
  ARCH="$(uname -m | sed -e 's/x86_64/amd64/' -e 's/\(arm\)\(64\)\?.*/\1\2/' -e 's/aarch64$/arm64/')" &&
  KREW="krew-${OS}_${ARCH}" &&
  curl -fsSLO "https://github.com/kubernetes-sigs/krew/releases/latest/download/${KREW}.tar.gz" &&
  tar zxvf "${KREW}.tar.gz" &&
  ./"${KREW}" install krew
)
```

Ajouter à votre PATH dans `~/.bashrc` ou `~/.zshrc` :
```bash
export PATH="${KREW_ROOT:-$HOME/.krew}/bin:$PATH"
```

**Utilisation de Base**

```bash
# Mettre à jour la liste des plugins
kubectl krew update

# Rechercher un plugin
kubectl krew search

# Installer un plugin
kubectl krew install <plugin-name>

# Lister les plugins installés
kubectl krew list

# Mettre à jour tous les plugins
kubectl krew upgrade
```

**Plugins Recommandés via Krew** :

**kubectl tree** : Visualiser la hiérarchie des ressources
```bash
kubectl krew install tree
kubectl tree deployment nginx
```

**kubectl neat** : Nettoyer les manifestes YAML
```bash
kubectl krew install neat
kubectl get pod my-pod -o yaml | kubectl neat
```

**kubectl images** : Lister toutes les images utilisées
```bash
kubectl krew install images
kubectl images
```

**kubectl whoami** : Afficher l'identité actuelle
```bash
kubectl krew install whoami
kubectl whoami
```

**kubectl ctx** et **kubectl ns** : Alternatives à kubectx/kubens
```bash
kubectl krew install ctx
kubectl krew install ns
```

### stern - Logs Agrégés Multi-Pods

**Description** : Afficher les logs de plusieurs pods simultanément avec coloration.

**Pourquoi l'utiliser ?**
- Suivre les logs de plusieurs pods en même temps
- Coloration par pod pour la lisibilité
- Filtrage par regex
- Support des labels selectors
- Timestamps optionnels

**Installation**

Sur Linux :
```bash
wget https://github.com/stern/stern/releases/latest/download/stern_linux_amd64.tar.gz
tar xzf stern_linux_amd64.tar.gz
sudo mv stern /usr/local/bin/
```

Sur macOS :
```bash
brew install stern
```

**Utilisation**

```bash
# Logs de tous les pods d'un deployment
stern deployment-name

# Logs de tous les pods avec un label
stern -l app=nginx

# Logs dans un namespace spécifique
stern -n kube-system coredns

# Logs avec timestamps
stern -t deployment-name

# Logs depuis les 10 dernières minutes
stern --since 10m deployment-name

# Exclure certains conteneurs
stern deployment-name --exclude-container sidecar
```

**Cas d'Usage** :
- Debugging d'applications distribuées
- Monitoring en temps réel
- Corrélation de logs entre pods
- Développement avec plusieurs réplicas

### fzf - Fuzzy Finder

**Description** : Outil de recherche floue générique, très utile avec kubectl.

**Pourquoi l'utiliser ?**
- Recherche interactive rapide
- Intégration avec bash/zsh
- Améliore drastiquement l'efficacité
- Recherche floue (pas besoin de typage exact)

**Installation**

Sur Linux :
```bash
git clone --depth 1 https://github.com/junegunn/fzf.git ~/.fzf
~/.fzf/install
```

Sur macOS :
```bash
brew install fzf
$(brew --prefix)/opt/fzf/install
```

**Utilisation avec kubectl**

Créer des alias dans votre `~/.bashrc` ou `~/.zshrc` :

```bash
# Sélectionner et décrire un pod
alias kp='kubectl get pods --all-namespaces | fzf | awk "{print \$1,\$2}" | xargs -n2 kubectl describe pod -n'

# Sélectionner et voir les logs d'un pod
alias kl='kubectl get pods --all-namespaces | fzf | awk "{print \$1,\$2}" | xargs -n2 kubectl logs -n'

# Sélectionner et exec dans un pod
alias ke='kubectl get pods --all-namespaces | fzf | awk "{print \$1,\$2}" | xargs -n2 kubectl exec -it -n'
```

**Avantages** :
- Pas besoin de connaître le nom exact des ressources
- Navigation rapide et intuitive
- Réduit les erreurs de typage

## Outils de Visualisation et Dashboards

### Kubernetes Dashboard

**Description** : Interface web officielle Kubernetes (déjà disponible via addon MicroK8s).

**Activation avec MicroK8s**
```bash
microk8s enable dashboard
```

**Avantages** :
- Interface officielle et maintenue
- Vue d'ensemble complète du cluster
- Actions CRUD via interface graphique
- Logs et shell intégrés
- Monitoring des ressources

**Accès au Dashboard**
```bash
# Obtenir le token d'accès
microk8s kubectl describe secret -n kube-system microk8s-dashboard-token

# Forward du port
microk8s kubectl port-forward -n kube-system service/kubernetes-dashboard 10443:443
```

Accéder via : https://localhost:10443

### Lens - The Kubernetes IDE

**Description** : IDE desktop gratuit pour Kubernetes, souvent appelé "le VSCode de Kubernetes".

**Pourquoi l'utiliser ?**
- Interface graphique moderne et intuitive
- Support multi-cluster
- Visualisation en temps réel
- Terminal intégré
- Édition de ressources
- Monitoring et métriques intégrés
- Extensions disponibles
- Gratuit pour usage personnel

**Installation**

Télécharger depuis : https://k8slens.dev/

Disponible pour Windows, macOS et Linux.

**Configuration avec MicroK8s**

Lens détecte automatiquement les clusters dans votre kubeconfig :
```bash
# S'assurer que la config MicroK8s est accessible
microk8s config > ~/.kube/config
```

Lancer Lens, et votre cluster MicroK8s apparaîtra automatiquement.

**Fonctionnalités Principales** :
- Explorer toutes les ressources visuellement
- Logs en temps réel avec recherche
- Shell dans les conteneurs
- Port-forwarding en un clic
- Graphiques de métriques
- Helm charts intégrés
- Support RBAC

**Idéal Pour** :
- Débutants cherchant une interface visuelle
- Gestion multi-cluster
- Debugging visuel
- Présentation et démonstrations

### Octant - Interface Web Local

**Description** : Dashboard web développé par VMware, focus sur le développement.

**Pourquoi l'utiliser ?**
- Interface moderne et réactive
- Visualisation des relations entre ressources
- Plugins extensibles
- Mode développeur-friendly
- Aucune installation serveur requise

**Installation**

Sur Linux :
```bash
wget https://github.com/vmware-tanzu/octant/releases/latest/download/octant_Linux-64bit.deb
sudo dpkg -i octant_Linux-64bit.deb
```

Sur macOS :
```bash
brew install octant
```

**Lancement**
```bash
octant
```

Ouvre automatiquement http://localhost:7777

**Avantages** :
- Visualisation graphique des applications
- Suivi des changements en temps réel
- Resource viewer détaillé
- Navigation intuitive

### Kui - Terminal GUI Hybride

**Description** : Terminal intelligent avec capacités graphiques pour Kubernetes.

**Pourquoi l'utiliser ?**
- Combine CLI et GUI
- Tableaux interactifs
- Visualisation dans le terminal
- Exécution de commandes kubectl normales
- Meilleure lisibilité des outputs

**Installation**

Télécharger depuis : https://github.com/kubernetes-sigs/kui

Disponible pour toutes les plateformes.

**Utilisation** :
- Lancer Kui
- Taper des commandes kubectl normales
- Output enrichi avec tableaux, couleurs, liens
- Cliquer sur les ressources pour plus de détails

## Outils de Développement et CI/CD

### Skaffold - Développement Continu

**Description** : Outil de workflow de développement local pour Kubernetes.

**Pourquoi l'utiliser ?**
- Build, push, deploy automatiques
- Hot reload pendant le développement
- Support multi-environnement
- Intégration CI/CD facile
- Détecte les changements de code

**Installation**

Sur Linux :
```bash
curl -Lo skaffold https://storage.googleapis.com/skaffold/releases/latest/skaffold-linux-amd64
sudo install skaffold /usr/local/bin/
```

Sur macOS :
```bash
brew install skaffold
```

**Concepts de Base**

Créer un `skaffold.yaml` :
```yaml
apiVersion: skaffold/v4beta6
kind: Config
build:
  artifacts:
  - image: my-app
    context: .
deploy:
  kubectl:
    manifests:
    - k8s/*.yaml
```

**Commandes Principales**

```bash
# Développement continu
skaffold dev

# Build et deploy une fois
skaffold run

# Debugging
skaffold debug

# Cleanup
skaffold delete
```

**Cas d'Usage** :
- Développement d'applications sur Kubernetes
- Tests rapides de modifications
- Pipeline de développement standardisé
- Hot reload pour les développeurs

### Tilt - Productivité Développeur

**Description** : Alternative à Skaffold, focus sur l'expérience développeur.

**Pourquoi l'utiliser ?**
- Interface web pour suivre les builds
- Très rapide (builds incrémentaux)
- Configuration en Python (Tiltfile)
- Live update sans rebuild complet
- Excellent pour le dev local

**Installation**

Sur Linux :
```bash
curl -fsSL https://raw.githubusercontent.com/tilt-dev/tilt/master/scripts/install.sh | bash
```

Sur macOS :
```bash
brew install tilt
```

**Utilisation de Base**

Créer un `Tiltfile` :
```python
docker_build('my-app', '.')
k8s_yaml('kubernetes.yaml')
k8s_resource('my-app', port_forwards=8000)
```

Lancer :
```bash
tilt up
```

Interface web disponible sur http://localhost:10350

**Avantages** :
- Feedback visuel en temps réel
- Gestion multi-service facile
- Fast feedback loop
- Extensions disponibles

### ArgoCD - GitOps pour Kubernetes

**Description** : Outil de déploiement continu suivant les principes GitOps.

**Pourquoi l'utiliser ?**
- Git comme source unique de vérité
- Déploiements déclaratifs
- Rollback facile
- Audit trail complet
- Interface web et CLI

**Installation sur MicroK8s**

```bash
# Créer namespace
microk8s kubectl create namespace argocd

# Installer ArgoCD
microk8s kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

# Accéder à l'interface
microk8s kubectl port-forward svc/argocd-server -n argocd 8080:443

# Obtenir le mot de passe admin
microk8s kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d
```

**Concepts Clés** :
- Application : Définition de ce qui doit être déployé
- Projet : Groupement logique d'applications
- Repository : Où se trouve le code source
- Sync : Synchronisation Git → Cluster

**Cas d'Usage** :
- Production deployments
- Multi-environment management
- Automatisation CD
- Audit et compliance

## Outils de Configuration et Templating

### Helm - Package Manager

**Description** : Gestionnaire de packages pour Kubernetes (le "apt" de Kubernetes).

**Pourquoi l'utiliser ?**
- Déploiements reproductibles
- Templating YAML
- Gestion des versions
- Rollback facile
- Large catalogue de charts

**Installation**

Sur Linux :
```bash
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
```

Sur macOS :
```bash
brew install helm
```

**Configuration avec MicroK8s**
```bash
# Créer la config kubectl
microk8s config > ~/.kube/config

# Helm utilisera automatiquement cette config
helm version
```

**Commandes Essentielles**

```bash
# Ajouter un repository
helm repo add bitnami https://charts.bitnami.com/bitnami

# Rechercher un chart
helm search repo nginx

# Installer un chart
helm install my-nginx bitnami/nginx

# Lister les releases
helm list

# Mettre à jour une release
helm upgrade my-nginx bitnami/nginx

# Rollback
helm rollback my-nginx 1

# Désinstaller
helm uninstall my-nginx
```

**Cas d'Usage** :
- Déploiement d'applications complexes
- Gestion de configurations multi-environnement
- Partage de packages réutilisables
- Standards de déploiement

### Kustomize - Customisation Native

**Description** : Outil de customisation de configuration Kubernetes sans templates.

**Pourquoi l'utiliser ?**
- Intégré à kubectl (depuis 1.14)
- Pas de templating, juste des overlays
- Gestion de variants (dev, staging, prod)
- Composition de ressources
- DRY (Don't Repeat Yourself)

**Déjà Inclus**
```bash
# Kustomize est intégré à kubectl
microk8s kubectl kustomize --help
```

**Structure Typique**

```
├── base/
│   ├── deployment.yaml
│   ├── service.yaml
│   └── kustomization.yaml
└── overlays/
    ├── dev/
    │   └── kustomization.yaml
    └── prod/
        └── kustomization.yaml
```

**Utilisation**

```bash
# Voir le résultat sans appliquer
microk8s kubectl kustomize overlays/dev/

# Appliquer
microk8s kubectl apply -k overlays/dev/
```

**Cas d'Usage** :
- Environnements multiples
- Éviter la duplication de YAML
- Approche déclarative pure
- Alternative plus simple à Helm

### yq - YAML Processor

**Description** : jq pour YAML - traitement en ligne de commande de fichiers YAML.

**Pourquoi l'utiliser ?**
- Parser et manipuler YAML facilement
- Scripts d'automatisation
- Extraction de données
- Modifications programmatiques
- Conversion YAML ↔ JSON

**Installation**

Sur Linux :
```bash
wget https://github.com/mikefarah/yq/releases/latest/download/yq_linux_amd64
sudo mv yq_linux_amd64 /usr/local/bin/yq
sudo chmod +x /usr/local/bin/yq
```

Sur macOS :
```bash
brew install yq
```

**Exemples d'Utilisation**

```bash
# Lire une valeur
yq eval '.spec.replicas' deployment.yaml

# Modifier une valeur
yq eval '.spec.replicas = 5' -i deployment.yaml

# Merge de fichiers
yq eval-all 'select(fileIndex == 0) * select(fileIndex == 1)' base.yaml overlay.yaml

# Conversion YAML vers JSON
yq eval -o json deployment.yaml

# Filtrer des ressources
yq eval 'select(.kind == "Deployment")' all-resources.yaml
```

**Cas d'Usage** :
- Scripting et automatisation
- Pipeline CI/CD
- Manipulation de manifestes
- Extraction d'informations

## Outils de Test et Validation

### kubeval - Validation de Manifestes

**Description** : Valide vos manifestes YAML Kubernetes contre les schémas officiels.

**Pourquoi l'utiliser ?**
- Détection d'erreurs avant déploiement
- Validation de syntaxe
- Vérification des champs obligatoires
- Support de versions multiples
- CI/CD integration

**Installation**

Sur Linux :
```bash
wget https://github.com/instrumenta/kubeval/releases/latest/download/kubeval-linux-amd64.tar.gz
tar xf kubeval-linux-amd64.tar.gz
sudo mv kubeval /usr/local/bin
```

Sur macOS :
```bash
brew install kubeval
```

**Utilisation**

```bash
# Valider un fichier
kubeval deployment.yaml

# Valider plusieurs fichiers
kubeval *.yaml

# Valider pour une version K8s spécifique
kubeval --kubernetes-version 1.28.0 deployment.yaml

# Strict mode
kubeval --strict deployment.yaml
```

**Cas d'Usage** :
- Pre-commit hooks
- Pipeline CI
- Vérification locale avant apply
- Learning (comprendre les erreurs)

### kubeconform - Alternative Moderne à kubeval

**Description** : Outil de validation moderne, plus rapide et maintenu activement.

**Pourquoi l'utiliser ?**
- Plus rapide que kubeval
- Support CRDs
- Activement maintenu
- Meilleure gestion des versions

**Installation**

Sur Linux :
```bash
wget https://github.com/yannh/kubeconform/releases/latest/download/kubeconform-linux-amd64.tar.gz
tar xf kubeconform-linux-amd64.tar.gz
sudo mv kubeconform /usr/local/bin/
```

**Utilisation**

```bash
# Validation simple
kubeconform deployment.yaml

# Tous les fichiers d'un répertoire
kubeconform k8s/*.yaml

# Avec CRDs
kubeconform -schema-location default -schema-location 'https://raw.githubusercontent.com/...' deployment.yaml
```

### kube-score - Analyse de Meilleures Pratiques

**Description** : Analyse vos manifestes et suggère des améliorations selon les best practices.

**Pourquoi l'utiliser ?**
- Recommandations de sécurité
- Optimisation des ressources
- Best practices Kubernetes
- Production readiness check
- Scoring détaillé

**Installation**

Sur Linux :
```bash
wget https://github.com/zegl/kube-score/releases/latest/download/kube-score_linux_amd64
chmod +x kube-score_linux_amd64
sudo mv kube-score_linux_amd64 /usr/local/bin/kube-score
```

Sur macOS :
```bash
brew install kube-score
```

**Utilisation**

```bash
# Analyser un manifeste
kube-score score deployment.yaml

# Analyser tous les manifestes
kube-score score k8s/*.yaml

# Sortie JSON
kube-score score --output-format json deployment.yaml

# Ignorer certains checks
kube-score score --ignore-test pod-networkpolicy deployment.yaml
```

**Ce qu'il vérifie** :
- Resource requests/limits
- Probes (readiness, liveness)
- Security contexts
- Network policies
- Labels recommandés
- Et beaucoup plus

### Polaris - Audit de Configuration

**Description** : Outil d'audit pour identifier les problèmes de configuration.

**Pourquoi l'utiliser ?**
- Dashboard visuel
- Scoring par workload
- Checks de sécurité
- Checks de fiabilité
- Checks d'efficacité

**Installation**

```bash
# Via kubectl
microk8s kubectl apply -f https://github.com/FairwindsOps/polaris/releases/latest/download/dashboard.yaml

# Port forward pour accès
microk8s kubectl port-forward --namespace polaris svc/polaris-dashboard 8080:80
```

Accès via : http://localhost:8080

**CLI Installation**

```bash
# Linux
wget https://github.com/FairwindsOps/polaris/releases/latest/download/polaris_linux_amd64.tar.gz
tar xzf polaris_linux_amd64.tar.gz
sudo mv polaris /usr/local/bin/

# macOS
brew install fairwinds/tap/polaris
```

**Utilisation CLI**

```bash
# Audit de fichiers locaux
polaris audit --audit-path ./k8s/

# Audit du cluster
polaris audit

# Générer un rapport HTML
polaris audit --format html > report.html
```

## Outils de Sécurité

### Trivy - Scanner de Vulnérabilités

**Description** : Scanner de sécurité pour conteneurs et Kubernetes.

**Pourquoi l'utiliser ?**
- Scan d'images Docker
- Détection de CVEs
- Scan de manifestes Kubernetes
- Scan de configurations IaC
- Facile à intégrer en CI/CD

**Installation**

Sur Linux :
```bash
wget https://github.com/aquasecurity/trivy/releases/latest/download/trivy_Linux-64bit.deb
sudo dpkg -i trivy_Linux-64bit.deb
```

Sur macOS :
```bash
brew install trivy
```

**Utilisation**

```bash
# Scanner une image
trivy image nginx:latest

# Scanner une image dans le registre MicroK8s
trivy image localhost:32000/my-app:latest

# Scanner des manifestes K8s
trivy config k8s/

# Scanner le cluster
trivy k8s --report summary
```

**Cas d'Usage** :
- CI/CD pipeline security
- Audit régulier des images
- Compliance et CVE tracking
- Prevention de déploiement d'images vulnérables

### kube-bench - CIS Benchmark

**Description** : Vérifie que Kubernetes est configuré selon les CIS Kubernetes Benchmarks.

**Pourquoi l'utiliser ?**
- Audit de sécurité automatique
- Standards de l'industrie (CIS)
- Recommandations actionnables
- Compliance tracking

**Installation et Utilisation**

```bash
# Exécuter en job Kubernetes
microk8s kubectl apply -f https://raw.githubusercontent.com/aquasecurity/kube-bench/main/job.yaml

# Voir les résultats
microk8s kubectl logs -f job/kube-bench
```

**Interprétation** :
- PASS : Configuration conforme
- FAIL : Configuration non-conforme
- WARN : Vérification manuelle requise
- INFO : Informations contextuelles

### Falco - Runtime Security

**Description** : Détection d'anomalies et menaces en temps réel.

**Pourquoi l'utiliser ?**
- Détection de comportements suspects
- Audit d'activité en temps réel
- Alerting sur incidents de sécurité
- Compliance monitoring

**Nota** : Plus avancé, pour environnements de production.

## Outils de Networking et Debugging

### kubectl-debug - Debugging Avancé

**Description** : Plugin pour déboguer des pods sans modifier leur image.

**Installation via krew**

```bash
kubectl krew install debug
```

**Utilisation**

```bash
# Déboguer un pod avec des outils
kubectl debug mypod -it --image=busybox

# Déboguer avec une image spécifique
kubectl debug mypod -it --image=nicolaka/netshoot

# Copier un pod pour debugging
kubectl debug mypod -it --copy-to=mypod-debug --container=mycontainer -- sh
```

**Images de Debug Recommandées** :
- `busybox` : Léger, outils basiques
- `nicolaka/netshoot` : Outils réseau complets
- `alpine` : Léger avec package manager

### Netshoot - Swiss Army Knife Réseau

**Description** : Container Docker avec tous les outils réseau dont vous avez besoin.

**Pourquoi l'utiliser ?**
- Debugging réseau complet
- Tous les outils en un seul endroit
- Parfait pour troubleshooting

**Lancer un Pod Netshoot**

```bash
microk8s kubectl run tmp-shell --rm -i --tty --image nicolaka/netshoot
```

**Outils Inclus** :
- curl, wget
- netcat (nc)
- nslookup, dig
- tcpdump
- nmap
- iperf, iperf3
- traceroute
- Et beaucoup d'autres

**Cas d'Usage** :
- Tester la connectivité réseau
- Debugger DNS
- Analyser le trafic
- Tester les Network Policies

## IDE et Éditeurs

### Visual Studio Code + Extensions

**Extensions Kubernetes Recommandées** :

**Kubernetes** (by Microsoft)
- Gestion de cluster
- IntelliSense pour YAML
- Helm support
- Deploy depuis l'IDE

**YAML** (by Red Hat)
- Validation YAML
- Autocomplétion
- Schema validation

**Docker** (by Microsoft)
- Build et push d'images
- Gestion de conteneurs
- Integration avec registres

**GitLens**
- Git superpuissant
- Idéal pour GitOps workflows

**Remote - Containers**
- Développement dans conteneurs
- Environnements consistants

**Installation des Extensions**

Dans VSCode :
1. Ouvrir la palette de commandes (Ctrl+Shift+P)
2. Chercher "Extensions: Install Extensions"
3. Rechercher et installer les extensions

**Snippets Kubernetes**

Créer `.vscode/settings.json` dans votre projet :
```json
{
  "yaml.schemas": {
    "kubernetes": "*.yaml"
  },
  "yaml.customTags": [
    "!encrypted/pkcs1-oaep scalar",
    "!vault scalar"
  ]
}
```

### IntelliJ IDEA / PyCharm + Kubernetes Plugin

**Pour les utilisateurs JetBrains** :

**Kubernetes Plugin**
- Support natif Kubernetes
- Autocomplétion YAML
- Deploy et debug
- Logs intégrés

**Installation** :
1. Preferences → Plugins
2. Rechercher "Kubernetes"
3. Installer et redémarrer

## Outils de Documentation

### kubectl-graph - Visualisation de Relations

**Description** : Génère des graphes des relations entre ressources Kubernetes.

**Installation**

```bash
kubectl krew install graph
```

**Utilisation**

```bash
# Graphe d'un deployment
kubectl graph deployment my-app

# Tous les workloads
kubectl graph all

# Export en format DOT
kubectl graph deployment my-app > graph.dot
```

### kubectl-tree - Hiérarchie de Ressources

**Description** : Affiche la hiérarchie parent-enfant des ressources.

**Installation**

```bash
kubectl krew install tree
```

**Utilisation**

```bash
# Arbre d'un deployment
kubectl tree deployment nginx

# Avec plus de détails
kubectl tree deployment nginx --all-namespaces
```

**Output Exemple** :
```
NAMESPACE  NAME                           READY  STATUS   AGE
default    Deployment/nginx               -              1d
default    └─ReplicaSet/nginx-abc123      -              1d
default      ├─Pod/nginx-abc123-xyz       True   Running  1d
default      ├─Pod/nginx-abc123-def       True   Running  1d
default      └─Pod/nginx-abc123-ghi       True   Running  1d
```

## Outils de Productivité et Automatisation

### Aliases Kubectl Essentiels

Ajouter à votre `~/.bashrc` ou `~/.zshrc` :

```bash
# Alias de base
alias k='kubectl'
alias mk='microk8s kubectl'

# Get resources
alias kgp='kubectl get pods'
alias kgs='kubectl get services'
alias kgd='kubectl get deployments'
alias kgn='kubectl get nodes'

# Describe
alias kdp='kubectl describe pod'
alias kds='kubectl describe service'

# Logs
alias kl='kubectl logs'
alias klf='kubectl logs -f'

# Delete
alias kdel='kubectl delete'

# Apply
alias ka='kubectl apply -f'

# Exec
alias kex='kubectl exec -it'

# All namespaces
alias kgpa='kubectl get pods --all-namespaces'
alias kgsa='kubectl get services --all-namespaces'

# Watch
alias kgpw='watch kubectl get pods'
```

### oh-my-zsh + Kubectl Plugin

Si vous utilisez zsh avec oh-my-zsh :

**Activation du Plugin**

Dans `~/.zshrc` :
```bash
plugins=(... kubectl)
```

**Avantages** :
- Aliases préconfigurés
- Autocomplétion améliorée
- Prompt avec contexte K8s actuel

### Bash/Zsh Completion

**Pour Bash**

```bash
# Ajouter à ~/.bashrc
source <(kubectl completion bash)
source <(microk8s kubectl completion bash)

# Avec alias
complete -o default -F __start_kubectl k
```

**Pour Zsh**

```bash
# Ajouter à ~/.zshrc
source <(kubectl completion zsh)
source <(microk8s kubectl completion zsh)

# Avec alias
compdef __start_kubectl k
```

### Scripts Utiles

**Redémarrer tous les pods d'un deployment**

```bash
#!/bin/bash
# restart-deployment.sh
kubectl rollout restart deployment $1
```

**Nettoyer les pods Evicted**

```bash
#!/bin/bash
# cleanup-evicted.sh
kubectl get pods --all-namespaces --field-selector 'status.phase=Failed' -o json | \
  kubectl delete -f -
```

**Vérifier la santé du cluster**

```bash
#!/bin/bash
# check-health.sh
echo "=== Nodes ==="
kubectl get nodes

echo -e "\n=== System Pods ==="
kubectl get pods -n kube-system

echo -e "\n=== Resources ==="
kubectl top nodes
kubectl top pods --all-namespaces
```

## Outils pour Environnement Multi-Cluster

### kubeswitch - Context Switching Avancé

**Description** : Alternative avancée à kubectx avec recherche floue.

**Installation**

```bash
# Via script
curl -sL https://raw.githubusercontent.com/danielfoehrKn/kubeswitch/master/hack/install/install.sh | bash
```

**Utilisation**

```bash
# Switch interactif
switch

# Avec recherche floue
switch <partial-name>

# Historique
switch -
```

### kubecm - Kubeconfig Manager

**Description** : Gestion avancée de fichiers kubeconfig.

**Installation**

```bash
# Linux
wget https://github.com/sunny0826/kubecm/releases/latest/download/kubecm_linux_amd64.tar.gz
tar -zxvf kubecm_linux_amd64.tar.gz
sudo mv kubecm /usr/local/bin/
```

**Utilisation**

```bash
# Lister les contextes
kubecm list

# Merger des kubeconfig
kubecm add -f new-cluster.yaml

# Supprimer un contexte
kubecm delete context-name

# Switch interactif
kubecm switch
```

## Conclusion et Recommandations

### Pour Débutants - Kit de Démarrage

**Essentiels** (installer en premier) :
1. **k9s** - Interface TUI intuitive
2. **kubectx/kubens** - Navigation rapide
3. **stern** - Logs multi-pods
4. **Helm** - Déploiements standards

**Optionnels mais utiles** :
5. **VSCode + Extensions** - Développement confortable
6. **Lens** - Interface graphique complète

### Pour Niveau Intermédiaire

Ajoutez à votre boîte à outils :
1. **krew** + plugins essentiels
2. **Skaffold ou Tilt** - Développement local
3. **kubeval/kubeconform** - Validation
4. **Trivy** - Sécurité
5. **yq** - Manipulation YAML

### Pour Niveau Avancé

Complétez avec :
1. **ArgoCD** - GitOps
2. **Polaris/kube-score** - Best practices
3. **Falco** - Runtime security
4. **Custom scripts** - Automatisation

### Stratégie d'Adoption

**Phase 1 - Exploration** (Semaine 1-2)
- Installer k9s et explorer votre cluster
- Configurer les aliases kubectl
- Essayer Lens pour visualisation

**Phase 2 - Développement** (Semaine 3-4)
- Configurer VSCode avec extensions
- Essayer Skaffold pour le dev local
- Mettre en place Helm pour les déploiements

**Phase 3 - Production** (Mois 2-3)
- Intégrer la validation (kubeval/kubeconform)
- Mettre en place le scanning de sécurité (Trivy)
- Adopter GitOps (ArgoCD)

**Phase 4 - Optimisation** (Mois 3+)
- Automatisation avec scripts
- Monitoring avancé
- Best practices enforcement

### Maintenance des Outils

**Mises à Jour Régulières**

```bash
# Homebrew (macOS)
brew update && brew upgrade

# krew plugins
kubectl krew upgrade

# Snap (Linux)
sudo snap refresh
```

**Nettoyage Périodique**

```bash
# Supprimer les anciens plugins krew
kubectl krew uninstall <unused-plugin>

# Nettoyer les anciennes versions
brew cleanup
```

### Ressources et Documentation

**Sites Officiels des Outils** :
- k9s : https://k9scli.io/
- Helm : https://helm.sh/
- Skaffold : https://skaffold.dev/
- ArgoCD : https://argo-cd.readthedocs.io/
- Lens : https://k8slens.dev/

**Collections d'Outils** :
- Awesome Kubernetes Tools : https://github.com/ramitsurana/awesome-kubernetes
- CNCF Landscape : https://landscape.cncf.io/

**Comparaisons et Reviews** :
- Reddit r/kubernetes
- CNCF Blog
- DevOps newsletters

### Points Clés à Retenir

1. **N'installez pas tout d'un coup** - Progressez graduellement
2. **Chaque outil a sa place** - Utilisez le bon outil pour le bon job
3. **Gardez votre toolbox à jour** - Les outils évoluent rapidement
4. **Partagez vos découvertes** - La communauté bénéficie de votre expérience
5. **Automatisez votre setup** - Scripts d'installation pour reproductibilité
6. **Documentez vos workflows** - Pour vous et votre équipe

### Prochaines Étapes

Continuez votre parcours avec :
- **26.4 Formations avancées** - Approfondir vos compétences
- **26.5 Certifications Kubernetes** - Valider votre expertise
- **26.6 Roadmap d'apprentissage** - Planifier votre progression

L'écosystème Kubernetes est riche et en constante évolution. Ces outils vous accompagneront dans votre voyage, du premier pod au cluster de production sophistiqué. Explorez, expérimentez, et trouvez les outils qui correspondent le mieux à votre style de travail !

⏭️ [Formations avancées](/26-ressources-et-parcours-de-certification/04-formations-avancees.md)
