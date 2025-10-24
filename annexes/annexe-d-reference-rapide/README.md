🔝 Retour au [Sommaire](/SOMMAIRE.md)

# Annexe D : Référence Rapide

## Introduction

Cette annexe constitue votre **guide de référence rapide** pour MicroK8s et Kubernetes. Elle regroupe les commandes, syntaxes et informations essentielles dont vous aurez besoin au quotidien.

### Objectif de cette Annexe

Cette référence rapide est conçue pour vous servir d'**aide-mémoire** pratique. Plutôt que de chercher dans la documentation complète ou de parcourir de longs tutoriels, vous trouverez ici rapidement :

✅ **Les commandes les plus utilisées**
✅ **La syntaxe correcte pour les opérations courantes**
✅ **Les options et paramètres importants**
✅ **Des exemples concrets et fonctionnels**
✅ **Les solutions aux problèmes fréquents**

### À Qui S'Adresse cette Référence ?

Cette annexe est utile pour :

- **Débutants** : Pour avoir sous la main les commandes de base
- **Utilisateurs intermédiaires** : Pour retrouver rapidement une syntaxe oubliée
- **Utilisateurs avancés** : Pour gagner du temps sur les opérations courantes
- **Administrateurs** : Pour les opérations de maintenance et dépannage

### Structure de l'Annexe

Cette référence rapide est organisée en sections thématiques :

#### 1. Commandes MicroK8s Essentielles
Les commandes spécifiques à MicroK8s pour gérer votre installation locale :
- Gestion du cluster (démarrage, arrêt, statut)
- Gestion des addons
- Configuration et maintenance
- Diagnostic et inspection

#### 2. Commandes kubectl Essentielles
Les commandes kubectl les plus utilisées au quotidien :
- Consultation des ressources
- Création et modification
- Debugging et logs
- Gestion des déploiements

#### 3. Cheat Sheet YAML
Syntaxe et exemples YAML pour toutes les ressources Kubernetes :
- Structure de base
- Pods et Deployments
- Services et Ingress
- ConfigMaps et Secrets
- Volumes et stockage
- Et bien plus...

#### 4. Troubleshooting Rapide
Solutions aux problèmes les plus courants :
- Problèmes de démarrage
- Pods qui ne démarrent pas
- Erreurs réseau
- Problèmes de stockage
- Questions de performance

#### 5. Glossaire des Termes Kubernetes
Définitions claires et accessibles des termes techniques :
- Tous les concepts Kubernetes expliqués
- Analogies et exemples
- Liens entre les concepts

### Comment Utiliser cette Référence ?

#### Méthode 1 : Navigation par Problème

**Vous avez un problème spécifique ?**
→ Consultez d'abord la section **Troubleshooting Rapide**

**Exemples :**
- "Mon pod est en CrashLoopBackOff" → Section Troubleshooting
- "Je ne sais pas quelle commande utiliser" → Section Commandes
- "Je dois écrire un Deployment YAML" → Section Cheat Sheet YAML

#### Méthode 2 : Navigation par Tâche

**Vous devez accomplir une tâche ?**
→ Trouvez la section correspondante

**Exemples :**
- "Déployer une application" → Commandes kubectl + Cheat Sheet YAML
- "Activer un addon" → Commandes MicroK8s
- "Voir les logs d'un pod" → Commandes kubectl

#### Méthode 3 : Recherche de Syntaxe

**Vous connaissez la commande mais pas la syntaxe exacte ?**
→ Utilisez la fonction de recherche (Ctrl+F / Cmd+F)

**Exemples :**
- Recherchez "scale" pour trouver comment scaler un deployment
- Recherchez "logs" pour les commandes de consultation des logs
- Recherchez "Service" pour les exemples de Services YAML

#### Méthode 4 : Apprentissage Progressif

**Vous débutez et voulez apprendre méthodiquement ?**
→ Suivez l'ordre des sections

**Parcours recommandé :**

1. **Commencez par le Glossaire** (15 min)
   - Comprenez les termes de base (Pod, Service, Deployment)
   - Familiarisez-vous avec le vocabulaire

2. **Consultez les Commandes MicroK8s** (10 min)
   - Apprenez à gérer votre installation
   - Pratiquez les commandes de base

3. **Explorez les Commandes kubectl** (20 min)
   - Maîtrisez get, describe, logs
   - Pratiquez sur votre cluster

4. **Étudiez le Cheat Sheet YAML** (30 min)
   - Comprenez la structure des manifests
   - Créez vos premiers fichiers YAML

5. **Gardez le Troubleshooting sous la main**
   - Consultez quand vous rencontrez un problème
   - Apprenez des erreurs courantes

### Format des Exemples de Commandes

Tout au long de cette référence, vous trouverez des exemples de commandes. Voici comment les lire :

#### Exemples Simples

```bash
# Ceci est un commentaire explicatif
microk8s status
```

**À taper exactement tel quel** (sans le `#` et le commentaire).

#### Exemples avec Paramètres Variables

```bash
kubectl get pod [nom-du-pod]
```

**Remplacez `[nom-du-pod]`** par le nom réel de votre pod.

**Exemple réel :**
```bash
kubectl get pod nginx
```

#### Exemples avec Options

```bash
kubectl get pods -n production
```

`-n production` est une option qui spécifie le namespace.

#### Exemples Multi-lignes

```bash
microk8s kubectl create deployment nginx \
  --image=nginx:latest \
  --replicas=3
```

Le `\` indique que la commande continue sur la ligne suivante. Vous pouvez :
- Taper tout sur une ligne (sans les `\`)
- Ou taper ligne par ligne avec les `\`

#### Exemples YAML

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx
spec:
  containers:
  - name: nginx
    image: nginx:1.21
```

**Important :**
- Respectez l'**indentation** (2 espaces par niveau)
- Utilisez des **espaces**, jamais de tabulations
- La casse est importante (majuscules/minuscules)

### Conventions Typographiques

Dans cette référence, nous utilisons les conventions suivantes :

| Convention | Signification | Exemple |
|------------|---------------|---------|
| `code` | Commande ou code | `kubectl get pods` |
| **gras** | Terme important | **Pod**, **Service** |
| *italique* | Emphase | *très important* |
| [paramètre] | Valeur à remplacer | `[nom-du-pod]` |
| 🎯 | Information essentielle | 🎯 Commande la plus utilisée |
| ✅ | Bonne pratique | ✅ Toujours définir des ressources |
| ⚠️ | Attention | ⚠️ Cette action est destructive |
| 💡 | Astuce | 💡 Utilisez un alias pour kubectl |

### Symboles de Priorité

Les commandes et concepts sont marqués selon leur importance :

#### 🎯 Essentiel
**À connaître absolument** - Utilisé quotidiennement
```bash
🎯 kubectl get pods
🎯 kubectl logs
🎯 kubectl describe
```

#### 📌 Important
**Très utile** - Utilisé régulièrement
```bash
📌 kubectl scale
📌 kubectl rollout
```

#### 💡 Bonus
**Bon à savoir** - Pour aller plus loin
```bash
💡 kubectl top
💡 kubectl diff
```

### Organisation Visuelle

#### Sections Principales

Chaque section commence par un titre clair et une brève description.

#### Sous-sections

Les sous-sections regroupent les commandes par fonction ou catégorie.

#### Exemples Groupés

Les exemples similaires sont groupés pour faciliter la comparaison :

```bash
# Démarrer
microk8s start

# Arrêter
microk8s stop

# Redémarrer
microk8s stop && microk8s start
```

#### Tableaux de Référence

Des tableaux récapitulatifs pour une consultation rapide :

| Commande | Description | Exemple |
|----------|-------------|---------|
| `get` | Lister | `kubectl get pods` |
| `describe` | Détailler | `kubectl describe pod nginx` |
| `logs` | Voir les logs | `kubectl logs nginx` |

### Conseils d'Utilisation

#### 🖨️ Imprimez les Sections Clés

Certaines sections méritent d'être imprimées et gardées à portée de main :
- Commandes kubectl essentielles
- Troubleshooting rapide
- Cheat sheet YAML de base

#### 🔖 Marquez vos Pages

Utilisez des marque-pages (bookmarks) pour les sections que vous consultez le plus souvent.

#### ✏️ Annotez

N'hésitez pas à ajouter vos propres notes et exemples dans les marges.

#### 🔄 Pratiquez

Cette référence n'est pas faite pour être lue de bout en bout. **Pratiquez** les commandes au fur et à mesure.

### Raccourcis et Alias Recommandés

Pour gagner du temps, nous recommandons de créer ces alias dans votre shell (à ajouter dans `~/.bashrc` ou `~/.zshrc`) :

```bash
# Alias pour MicroK8s
alias mk='microk8s'
alias mkubectl='microk8s kubectl'

# Ou encore plus court, alias kubectl directement
alias kubectl='microk8s kubectl'

# Alias kubectl courants
alias k='kubectl'
alias kgp='kubectl get pods'
alias kgs='kubectl get services'
alias kgd='kubectl get deployments'
alias kd='kubectl describe'
alias kl='kubectl logs'
alias ke='kubectl exec -it'
alias kaf='kubectl apply -f'
alias kdf='kubectl delete -f'
```

**Après avoir ajouté ces alias :**
```bash
# Recharger votre configuration
source ~/.bashrc  # ou source ~/.zshrc

# Tester
k get pods  # au lieu de kubectl get pods
kgp         # encore plus court !
```

### Complétion Automatique

Activez la complétion automatique pour kubectl (gain de temps énorme !) :

**Pour Bash :**
```bash
source <(kubectl completion bash)
echo "source <(kubectl completion bash)" >> ~/.bashrc

# Si vous utilisez l'alias 'k'
complete -F __start_kubectl k
```

**Pour Zsh :**
```bash
source <(kubectl completion zsh)
echo "source <(kubectl completion zsh)" >> ~/.zshrc

# Si vous utilisez l'alias 'k'
complete -F __start_kubectl k
```

**Avec la complétion, vous pouvez :**
```bash
kubectl get po[TAB]      # Complète en "pods"
kubectl get pods [TAB]   # Liste tous les pods disponibles
kubectl describe pod ng[TAB]  # Complète le nom du pod
```

### Ressources Complémentaires en Ligne

Cette référence rapide couvre l'essentiel. Pour aller plus loin :

#### Documentation Officielle

- **MicroK8s** : https://microk8s.io/docs
- **Kubernetes** : https://kubernetes.io/docs/
- **kubectl** : https://kubernetes.io/docs/reference/kubectl/

#### Cheat Sheets Interactives

- **Kubectl Cheat Sheet** : https://kubernetes.io/docs/reference/kubectl/cheatsheet/
- **Interactive Tutorial** : https://kubernetes.io/docs/tutorials/kubernetes-basics/

#### Communautés et Support

- **Kubernetes Slack** : https://slack.k8s.io/
- **MicroK8s Discourse** : https://discuss.kubernetes.io/c/microk8s/
- **Stack Overflow** : Tag `microk8s` ou `kubernetes`

### Mise à Jour de cette Référence

Kubernetes et MicroK8s évoluent constamment. Cette référence est basée sur :

- **MicroK8s** : Version 1.28+
- **Kubernetes** : Version 1.28+
- **Date de mise à jour** : Octobre 2024

**Pour les versions plus récentes**, consultez toujours la documentation officielle pour les nouvelles fonctionnalités.

### Comment Contribuer à vos Propres Notes

Cette référence est un point de départ. Nous vous encourageons à :

1. **Ajouter vos propres exemples** basés sur vos cas d'usage
2. **Noter les commandes que vous utilisez le plus**
3. **Documenter les solutions à vos problèmes spécifiques**
4. **Créer des scripts** pour vos tâches répétitives

**Exemple de script utile :**
```bash
#!/bin/bash
# Sauvegardé dans ~/scripts/k8s-status.sh
# Affiche un résumé rapide du cluster

echo "=== Statut MicroK8s ==="
microk8s status

echo ""
echo "=== Nœuds ==="
kubectl get nodes

echo ""
echo "=== Pods (tous namespaces) ==="
kubectl get pods -A

echo ""
echo "=== Services ==="
kubectl get svc -A

echo ""
echo "=== Utilisation Ressources ==="
kubectl top nodes 2>/dev/null || echo "Metrics-server non activé"
```

### Organisation de vos Fichiers

Créez une structure organisée pour vos manifests YAML :

```
~/kubernetes/
├── apps/
│   ├── production/
│   ├── staging/
│   └── development/
├── configs/
│   ├── configmaps/
│   └── secrets/
├── infrastructure/
│   ├── ingress/
│   ├── storage/
│   └── monitoring/
└── scripts/
    ├── deploy.sh
    ├── backup.sh
    └── status.sh
```

### Workflow Recommandé

Pour travailler efficacement avec cette référence :

#### Workflow de Développement

1. **Conception** : Consultez le Cheat Sheet YAML
2. **Création** : Écrivez vos manifests
3. **Validation** : `kubectl apply --dry-run`
4. **Déploiement** : `kubectl apply -f`
5. **Vérification** : `kubectl get`, `kubectl describe`
6. **Monitoring** : `kubectl logs`, `kubectl top`
7. **Debugging** : Section Troubleshooting si problème

#### Workflow de Maintenance

1. **Vérification** : `microk8s status`
2. **Mise à jour** : `sudo snap refresh microk8s`
3. **Nettoyage** : Supprimer les ressources inutilisées
4. **Backup** : Sauvegarder les configs importantes
5. **Monitoring** : Surveiller les ressources

### Points d'Attention Importants

Avant de plonger dans les sections suivantes, gardez à l'esprit :

#### ⚠️ Sécurité

- **Jamais de secrets en clair** dans les fichiers YAML versionnés
- **Toujours utiliser des Secrets** Kubernetes pour les données sensibles
- **RBAC activé** pour contrôler les accès
- **Network Policies** pour isoler les applications

#### ⚠️ Production

- **Toujours tester** sur dev/staging avant production
- **Définir des ressources** (requests/limits) sur tous les pods
- **Configurer des probes** (liveness, readiness)
- **Avoir une stratégie de backup**

#### ⚠️ Performance

- **Ne pas surcharger** un seul nœud
- **Monitorer** les ressources régulièrement
- **Nettoyer** les images et logs inutilisés
- **Scaler** en fonction de la charge

### Prêt à Commencer ?

Maintenant que vous savez comment utiliser cette référence rapide, vous pouvez :

1. **Parcourir les sections** qui vous intéressent
2. **Pratiquer les commandes** sur votre cluster MicroK8s
3. **Consulter le Troubleshooting** quand vous rencontrez un problème
4. **Revenir régulièrement** pour rafraîchir vos connaissances

---

## Navigation Rapide

Voici les liens directs vers chaque section principale :

### 📋 Sections de cette Annexe

1. **[Commandes MicroK8s Essentielles](#commandes-microk8s-essentielles)**
   - Gestion du cluster
   - Addons
   - Configuration
   - Diagnostic

2. **[Commandes kubectl Essentielles](#commandes-kubectl-essentielles)**
   - Consultation (get, describe)
   - Création et modification (create, apply, edit)
   - Debugging (logs, exec, port-forward)
   - Déploiements (scale, rollout)

3. **[Cheat Sheet YAML](#cheat-sheet-yaml)**
   - Pods
   - Deployments
   - Services
   - Ingress
   - ConfigMaps & Secrets
   - Volumes

4. **[Troubleshooting Rapide](#troubleshooting-rapide)**
   - Problèmes MicroK8s
   - Problèmes de Pods
   - Problèmes réseau
   - Problèmes de stockage
   - Performance

5. **[Glossaire Kubernetes](#glossaire-kubernetes)**
   - A à Z des termes
   - Définitions et exemples
   - Concepts clés

---

## À Retenir

Cette référence rapide est votre **compagnon quotidien** pour travailler avec MicroK8s et Kubernetes.

**Les 5 choses à retenir :**

1. 🎯 **Consultez d'abord le Troubleshooting** en cas de problème
2. 📖 **Utilisez le Glossaire** pour comprendre les termes
3. ⌨️ **Pratiquez les commandes** régulièrement
4. 📝 **Référez-vous au Cheat Sheet** pour écrire vos YAML
5. 🔖 **Gardez cette référence accessible** à tout moment

**Conseil final :** Imprimez ou ajoutez cette page à vos favoris. Vous y reviendrez souvent !

---

Passons maintenant aux sections pratiques ! 🚀

⏭️ [Commandes microk8s essentielles](/annexes/annexe-d-reference-rapide/commandes-microk8s-essentielles.md)
