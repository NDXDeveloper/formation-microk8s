🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 2.9 Configuration des alias kubectl

## Introduction

L'une des premières choses que vous remarquerez en travaillant avec MicroK8s, c'est que taper `microk8s kubectl` avant chaque commande peut devenir répétitif et fastidieux. Par exemple :

```bash
microk8s kubectl get pods
microk8s kubectl describe service mon-service
microk8s kubectl logs mon-pod
```

C'est là qu'interviennent les **alias**. Un alias est un raccourci shell qui vous permet de remplacer une longue commande par une version plus courte. Dans cette section, nous allons apprendre à configurer des alias pour simplifier considérablement votre travail quotidien avec Kubernetes.

## Pourquoi Utiliser des Alias ?

### Avantages principaux

**Gain de temps** : Réduire `microk8s kubectl get pods` à simplement `k get pods` économise de précieuses secondes, des centaines de fois par jour.

**Moins d'erreurs de frappe** : Moins de caractères à taper = moins de risques de faire une erreur.

**Productivité accrue** : Vous vous concentrez sur ce que vous voulez faire, pas sur comment le taper.

**Confort d'utilisation** : L'expérience devient plus fluide et agréable.

### Exemple concret

Sans alias :
```bash
microk8s kubectl get pods --all-namespaces
```

Avec alias :
```bash
k get pods -A
```

La différence est évidente !

## Comprendre les Shells

Avant de configurer des alias, il est important de comprendre quel **shell** vous utilisez. Le shell est l'interpréteur de commandes de votre terminal.

### Identifier votre shell

Pour savoir quel shell vous utilisez, tapez :

```bash
echo $SHELL
```

**Résultats possibles** :

- `/bin/bash` : Vous utilisez Bash (le plus courant sur Linux)
- `/bin/zsh` : Vous utilisez Zsh (par défaut sur macOS depuis Catalina)
- `/bin/fish` : Vous utilisez Fish
- Autre : Consultez la documentation de votre shell spécifique

### Fichiers de configuration par shell

Chaque shell a son propre fichier de configuration où vous placerez vos alias :

| Shell | Fichier de configuration | Emplacement |
|-------|-------------------------|-------------|
| Bash | `.bashrc` ou `.bash_profile` | `~/.bashrc` ou `~/.bash_profile` |
| Zsh | `.zshrc` | `~/.zshrc` |
| Fish | `config.fish` | `~/.config/fish/config.fish` |

**Note** : Le symbole `~` représente votre répertoire personnel (home directory).

### Bash : .bashrc ou .bash_profile ?

Sur Linux, utilisez généralement `.bashrc`. Sur macOS, utilisez `.bash_profile`.

**Pour être sûr**, vous pouvez créer les deux :

```bash
# Linux
nano ~/.bashrc

# macOS (si vous utilisez Bash)
nano ~/.bash_profile
```

Ou encore mieux, sur macOS, faites en sorte que `.bash_profile` charge `.bashrc` :

```bash
echo 'source ~/.bashrc' >> ~/.bash_profile
```

Ainsi, vous pouvez ensuite tout mettre dans `.bashrc`.

## L'Alias Principal : kubectl

### Configuration de l'alias de base

L'alias le plus important à configurer est celui qui remplace `microk8s kubectl` par simplement `kubectl`.

#### Pour Bash

Ouvrez votre fichier de configuration :

```bash
nano ~/.bashrc
```

Ajoutez cette ligne à la fin du fichier :

```bash
alias kubectl='microk8s kubectl'
```

Sauvegardez et fermez (Ctrl+O, Entrée, Ctrl+X dans nano).

Pour appliquer les changements immédiatement :

```bash
source ~/.bashrc
```

#### Pour Zsh (macOS par défaut)

Ouvrez votre fichier de configuration :

```bash
nano ~/.zshrc
```

Ajoutez cette ligne :

```bash
alias kubectl='microk8s kubectl'
```

Sauvegardez et appliquez :

```bash
source ~/.zshrc
```

#### Pour Fish

Ouvrez votre fichier de configuration :

```bash
nano ~/.config/fish/config.fish
```

Ajoutez cette ligne :

```bash
alias kubectl='microk8s kubectl'
```

Sauvegardez et appliquez :

```bash
source ~/.config/fish/config.fish
```

### Vérification

Testez votre alias :

```bash
kubectl version
```

Si cela fonctionne (affiche la version sans erreur), votre alias est correctement configuré ! 🎉

Vous pouvez maintenant utiliser :

```bash
kubectl get pods
kubectl get services
kubectl describe node
```

Au lieu de :

```bash
microk8s kubectl get pods
microk8s kubectl get services
microk8s kubectl describe node
```

## Alias Courants et Utiles

Une fois l'alias de base configuré, vous pouvez aller plus loin en créant des raccourcis pour les commandes kubectl que vous utilisez fréquemment.

### L'alias ultra-court : k

La communauté Kubernetes utilise massivement l'alias `k` pour `kubectl`. C'est devenu un standard de facto.

```bash
alias k='kubectl'
```

Maintenant vous pouvez faire :

```bash
k get pods
k get svc
k describe pod mon-pod
```

### Alias pour les commandes get

```bash
# Lister les pods
alias kgp='kubectl get pods'

# Lister les pods dans tous les namespaces
alias kgpa='kubectl get pods --all-namespaces'

# Lister les services
alias kgs='kubectl get services'

# Lister les deployments
alias kgd='kubectl get deployments'

# Lister les nodes
alias kgn='kubectl get nodes'

# Tout voir dans tous les namespaces
alias kga='kubectl get all --all-namespaces'
```

### Alias pour describe

```bash
# Décrire un pod
alias kdp='kubectl describe pod'

# Décrire un service
alias kds='kubectl describe service'

# Décrire un node
alias kdn='kubectl describe node'
```

### Alias pour les logs

```bash
# Voir les logs d'un pod
alias kl='kubectl logs'

# Suivre les logs en temps réel
alias klf='kubectl logs -f'
```

### Alias pour exec

```bash
# Exécuter une commande dans un pod
alias kex='kubectl exec -it'

# Ouvrir un shell bash dans un pod
alias ksh='kubectl exec -it -- /bin/bash'
```

### Alias pour delete

```bash
# Supprimer un pod
alias kdel='kubectl delete'

# Forcer la suppression d'un pod
alias kdelp='kubectl delete pod --grace-period=0 --force'
```

### Alias pour les namespaces

```bash
# Changer de namespace
alias kn='kubectl config set-context --current --namespace'

# Lister les namespaces
alias kgns='kubectl get namespaces'
```

### Alias pour apply et create

```bash
# Appliquer une configuration
alias ka='kubectl apply -f'

# Créer une ressource
alias kc='kubectl create'
```

### Alias pour watch

```bash
# Observer les pods en temps réel
alias kgpw='watch kubectl get pods'

# Observer tous les namespaces
alias kgpaw='watch kubectl get pods --all-namespaces'
```

## Configuration Complète Recommandée

Voici une configuration complète que vous pouvez copier-coller dans votre fichier de configuration :

### Pour Bash/Zsh

```bash
# ===== Alias MicroK8s et Kubernetes =====

# Alias de base
alias kubectl='microk8s kubectl'
alias k='kubectl'

# Get - Listing des ressources
alias kgp='kubectl get pods'
alias kgpa='kubectl get pods --all-namespaces'
alias kgpw='kubectl get pods --watch'
alias kgs='kubectl get services'
alias kgd='kubectl get deployments'
alias kgn='kubectl get nodes'
alias kgns='kubectl get namespaces'
alias kga='kubectl get all --all-namespaces'

# Describe - Description détaillée
alias kdp='kubectl describe pod'
alias kds='kubectl describe service'
alias kdd='kubectl describe deployment'
alias kdn='kubectl describe node'

# Logs
alias kl='kubectl logs'
alias klf='kubectl logs -f'
alias klt='kubectl logs --tail=100'

# Exec - Exécution dans les pods
alias kex='kubectl exec -it'
alias ksh='kubectl exec -it -- /bin/bash'

# Delete - Suppression
alias kdel='kubectl delete'
alias kdelp='kubectl delete pod'
alias kdelf='kubectl delete pod --grace-period=0 --force'

# Apply/Create
alias ka='kubectl apply -f'
alias kc='kubectl create'

# Edit
alias ke='kubectl edit'

# Namespace
alias kn='kubectl config set-context --current --namespace'

# Contexte
alias kctx='kubectl config current-context'
alias kconf='kubectl config view'

# Top - Métriques (nécessite metrics-server)
alias ktop='kubectl top'
alias ktopp='kubectl top pods'
alias ktopn='kubectl top nodes'

# Port forward
alias kpf='kubectl port-forward'

# Scale
alias kscale='kubectl scale'

# Rollout
alias kroll='kubectl rollout'
alias krollr='kubectl rollout restart'
alias krolls='kubectl rollout status'
```

### Pour Fish

```fish
# ===== Alias MicroK8s et Kubernetes =====

# Alias de base
alias kubectl='microk8s kubectl'
alias k='kubectl'

# Get
alias kgp='kubectl get pods'
alias kgpa='kubectl get pods --all-namespaces'
alias kgs='kubectl get services'
alias kgd='kubectl get deployments'
alias kgn='kubectl get nodes'

# Logs
alias kl='kubectl logs'
alias klf='kubectl logs -f'

# Exec
alias kex='kubectl exec -it'

# Apply
alias ka='kubectl apply -f'
```

## Appliquer la Configuration

Après avoir ajouté vos alias, vous devez recharger votre fichier de configuration.

### Rechargement manuel

**Bash** :
```bash
source ~/.bashrc
```

**Zsh** :
```bash
source ~/.zshrc
```

**Fish** :
```bash
source ~/.config/fish/config.fish
```

### Rechargement automatique

Les alias seront automatiquement disponibles lors de votre prochaine ouverture de terminal.

**Astuce** : Pour ouvrir un nouveau terminal avec les alias, vous pouvez :
- Fermer et rouvrir votre terminal
- Ouvrir un nouvel onglet/fenêtre de terminal

## Vérification des Alias

Pour vérifier qu'un alias est bien configuré :

```bash
alias kubectl
```

Devrait afficher :
```
alias kubectl='microk8s kubectl'
```

Pour voir tous vos alias :

```bash
alias
```

Cette commande affiche tous les alias configurés dans votre session.

Pour voir uniquement les alias Kubernetes :

```bash
alias | grep kubectl
```

ou

```bash
alias | grep '^k'
```

## Utilisation Pratique des Alias

### Exemples concrets

**Sans alias** :
```bash
microk8s kubectl get pods --all-namespaces
microk8s kubectl describe pod mon-pod
microk8s kubectl logs mon-pod -f
microk8s kubectl exec -it mon-pod -- /bin/bash
```

**Avec alias** :
```bash
kgpa
kdp mon-pod
klf mon-pod
ksh mon-pod
```

La différence est spectaculaire !

### Workflow typique avec alias

```bash
# Vérifier l'état du cluster
k get nodes

# Lister les pods
kgp

# Voir les détails d'un pod
kdp mon-pod

# Suivre les logs
klf mon-pod

# Se connecter au pod
ksh mon-pod

# Appliquer une configuration
ka mon-deployment.yaml

# Vérifier le déploiement
kgd
```

## Alias Avancés avec Fonctions

Pour des opérations plus complexes, vous pouvez créer des fonctions shell au lieu de simples alias.

### Fonction pour voir les pods d'un namespace

**Bash/Zsh** :

```bash
# Lister les pods d'un namespace spécifique
kgpn() {
    kubectl get pods -n "$1"
}
```

Utilisation :
```bash
kgpn kube-system
kgpn default
```

### Fonction pour les logs d'un pod dans un namespace

```bash
# Logs d'un pod dans un namespace
kln() {
    kubectl logs -n "$1" "$2"
}
```

Utilisation :
```bash
kln kube-system coredns-abc123
```

### Fonction pour se connecter à un pod

```bash
# Se connecter à un pod avec choix du shell
kshell() {
    local pod="$1"
    local shell="${2:-/bin/bash}"
    kubectl exec -it "$pod" -- "$shell"
}
```

Utilisation :
```bash
kshell mon-pod              # Utilise bash par défaut
kshell mon-pod /bin/sh      # Utilise sh
```

### Fonction pour surveiller les pods

```bash
# Surveiller les pods avec auto-rafraîchissement
kwatchp() {
    watch -n 2 "kubectl get pods ${1:+--namespace $1}"
}
```

Utilisation :
```bash
kwatchp              # Tous les namespaces du contexte actuel
kwatchp kube-system  # Namespace spécifique
```

## Organisation et Maintenance

### Organiser vos alias

Pour maintenir une configuration propre, organisez vos alias par catégories :

```bash
# ===== MicroK8s =====
alias mk='microk8s'
alias mks='microk8s status'
alias mke='microk8s enable'
alias mkd='microk8s disable'

# ===== Kubectl de base =====
alias kubectl='microk8s kubectl'
alias k='kubectl'

# ===== Get =====
alias kgp='kubectl get pods'
alias kgs='kubectl get services'
# ... etc

# ===== Logs =====
alias kl='kubectl logs'
alias klf='kubectl logs -f'
# ... etc
```

### Commentaires

Ajoutez des commentaires pour documenter vos alias :

```bash
# Lister tous les pods de tous les namespaces avec des détails
alias kgpaw='kubectl get pods --all-namespaces --watch'

# Redémarrer un deployment (utile après un changement de configmap)
alias krestart='kubectl rollout restart deployment'
```

### Fichier dédié (optionnel)

Pour une meilleure organisation, vous pouvez créer un fichier dédié à vos alias Kubernetes :

**Créer le fichier** :

```bash
nano ~/.kubectl_aliases
```

**Ajouter vos alias dans ce fichier**, puis dans votre `.bashrc` ou `.zshrc` :

```bash
# Charger les alias kubectl
if [ -f ~/.kubectl_aliases ]; then
    source ~/.kubectl_aliases
fi
```

Cette approche garde votre fichier de configuration principal propre et permet de partager facilement vos alias Kubernetes.

## Alias pour MicroK8s

N'oubliez pas les commandes MicroK8s elles-mêmes :

```bash
# Raccourcis MicroK8s
alias mk='microk8s'
alias mks='microk8s status'
alias mkstart='microk8s start'
alias mkstop='microk8s stop'
alias mke='microk8s enable'
alias mkd='microk8s disable'
alias mki='microk8s inspect'
alias mkr='microk8s reset'
```

Utilisation :
```bash
mks              # Voir le statut
mke dns          # Activer le DNS
mki              # Lancer un diagnostic
```

## Autocomplétion kubectl

En plus des alias, vous pouvez activer l'autocomplétion pour kubectl, ce qui vous permet d'appuyer sur Tab pour compléter automatiquement les commandes et les noms de ressources.

### Pour Bash

```bash
# Ajouter à ~/.bashrc
source <(kubectl completion bash)

# Si vous utilisez l'alias 'k'
complete -o default -F __start_kubectl k
```

### Pour Zsh

```bash
# Ajouter à ~/.zshrc
source <(kubectl completion zsh)

# Si vous utilisez l'alias 'k'
complete -o default -F __start_kubectl k
```

### Pour Fish

```bash
# L'autocomplétion est généralement déjà incluse dans Fish
kubectl completion fish | source
```

### Tester l'autocomplétion

Après configuration, essayez :

```bash
kubectl get po<TAB>
```

Devrait compléter en `kubectl get pods`.

```bash
k get pod mon-<TAB>
```

Devrait lister les pods commençant par "mon-".

## Partage de Configuration

### Sauvegarder vos alias

Pour sauvegarder vos alias ou les partager avec d'autres machines :

```bash
# Sauvegarder
cp ~/.bashrc ~/mes-sauvegardes/bashrc-backup

# Ou juste la partie alias
grep "alias k" ~/.bashrc > ~/kubectl-aliases.txt
```

### Synchroniser entre machines

Si vous travaillez sur plusieurs machines, vous pouvez :

1. **Utiliser un dépôt Git** :
   ```bash
   git init ~/dotfiles
   cp ~/.bashrc ~/dotfiles/
   cd ~/dotfiles
   git add .
   git commit -m "Mes configurations shell"
   git remote add origin <votre-repo>
   git push
   ```

2. **Sur une autre machine** :
   ```bash
   git clone <votre-repo> ~/dotfiles
   ln -s ~/dotfiles/bashrc ~/.bashrc
   source ~/.bashrc
   ```

### Dotfiles managers

Des outils comme **chezmoi**, **yadm**, ou **dotbot** permettent de gérer facilement vos fichiers de configuration entre plusieurs machines.

## Problèmes Courants et Solutions

### Alias non reconnu après configuration

**Problème** : Vous avez ajouté l'alias mais il ne fonctionne pas.

**Solutions** :
1. Rechargez votre configuration :
   ```bash
   source ~/.bashrc  # ou ~/.zshrc
   ```
2. Vérifiez que vous avez modifié le bon fichier
3. Vérifiez qu'il n'y a pas d'erreurs de syntaxe :
   ```bash
   bash -n ~/.bashrc
   ```

### Conflit d'alias

**Problème** : Un alias existe déjà et est écrasé.

**Solution** : Vérifiez les alias existants avant d'en créer :
```bash
alias | grep "^k="
```

Si un alias existe déjà, choisissez un nom différent.

### Alias ne fonctionne que dans le terminal actuel

**Problème** : L'alias disparaît quand vous fermez le terminal.

**Solution** : Vous avez probablement créé l'alias directement dans le terminal au lieu de l'ajouter au fichier de configuration.

Ajoutez-le dans `~/.bashrc` ou `~/.zshrc` pour qu'il soit permanent.

### Problème de permissions

**Problème** : Erreur lors de l'exécution de l'alias.

**Solution** : Vérifiez que MicroK8s fonctionne :
```bash
microk8s status
```

Et que vous avez les bonnes permissions :
```bash
groups | grep microk8s
```

### Shell non supporté

**Problème** : Vous utilisez un shell différent.

**Solution** : Consultez la documentation de votre shell pour savoir où placer les alias. La syntaxe de base est généralement la même.

## Bonnes Pratiques

### Choisir des noms mnémoniques

- Utilisez des abréviations logiques : `kgp` pour "kubectl get pods"
- Gardez une cohérence : tous les alias "get" commencent par `kg`
- Évitez les noms trop courts qui pourraient entrer en conflit

### Documenter vos alias

```bash
# ===== Alias personnalisés =====
# kgpl : Get pods with labels displayed
alias kgpl='kubectl get pods --show-labels'

# kgps : Get pods sorted by creation time
alias kgps='kubectl get pods --sort-by=.metadata.creationTimestamp'
```

### Ne pas surcharger

- Commencez par les alias de base
- Ajoutez progressivement ceux que vous utilisez réellement
- Supprimez ceux que vous n'utilisez jamais

### Tester avant de sauvegarder

Créez d'abord l'alias temporairement dans votre terminal :

```bash
alias test_alias='kubectl get pods'
test_alias  # Tester
```

Si cela fonctionne, ajoutez-le à votre fichier de configuration.

## Alias Communautaires Populaires

La communauté Kubernetes a développé des collections d'alias très complètes.

### kubectl-aliases

Le projet **kubectl-aliases** sur GitHub propose plus de 800 alias générés automatiquement :

https://github.com/ahmetb/kubectl-aliases

Exemples d'alias de cette collection :
- `k` = `kubectl`
- `kg` = `kubectl get`
- `kgpo` = `kubectl get pods`
- `kgsvc` = `kubectl get svc`
- `kdpo` = `kubectl describe pod`

**Installation** :

```bash
# Télécharger le fichier
wget https://raw.githubusercontent.com/ahmetb/kubectl-aliases/master/.kubectl_aliases

# Sourcer dans votre configuration
echo "[ -f ~/.kubectl_aliases ] && source ~/.kubectl_aliases" >> ~/.bashrc
source ~/.bashrc
```

**Attention** : Cette collection est très complète mais peut être overwhelming pour les débutants. Commencez par les alias de base de ce tutoriel.

## Récapitulatif des Alias Essentiels

Voici les alias que vous devriez absolument configurer pour commencer :

```bash
# Les 10 alias essentiels
alias kubectl='microk8s kubectl'  # Le plus important !
alias k='kubectl'                 # Raccourci ultra-court
alias kgp='kubectl get pods'      # Lister les pods
alias kgpa='kubectl get pods -A' # Pods tous namespaces
alias kl='kubectl logs'           # Voir les logs
alias klf='kubectl logs -f'       # Suivre les logs
alias kdp='kubectl describe pod' # Décrire un pod
alias ka='kubectl apply -f'       # Appliquer config
alias kex='kubectl exec -it'      # Exec dans pod
alias kgn='kubectl get nodes'     # Lister les nodes
```

Copiez ces 10 lignes dans votre `~/.bashrc` ou `~/.zshrc`, et vous verrez immédiatement la différence !

## Prochaines Étapes

Maintenant que vos alias sont configurés et que vous pouvez travailler efficacement avec kubectl, vous êtes prêt pour :

1. **Section 2.10** : Réaliser un premier diagnostic complet du cluster
2. **Chapitre 3** : Explorer les concepts Kubernetes essentiels
3. **Chapitre 4** : Effectuer vos premiers déploiements avec vos nouveaux alias

Avec ces alias en place, votre productivité va considérablement augmenter. Vous passerez moins de temps à taper des commandes et plus de temps à apprendre et expérimenter avec Kubernetes ! ⚡

## Références

- [kubectl Cheat Sheet officiel](https://kubernetes.io/docs/reference/kubectl/cheatsheet/)
- [kubectl-aliases sur GitHub](https://github.com/ahmetb/kubectl-aliases)
- Documentation shell : `man bash`, `man zsh`

⏭️ [Premier diagnostic du cluster avec microk8s inspect](/02-installation-et-configuration-initiale/10-premier-diagnostic-du-cluster.md)
