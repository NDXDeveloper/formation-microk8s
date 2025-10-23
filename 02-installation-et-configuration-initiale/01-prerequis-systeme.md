🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 2.1 Prérequis système (CPU, RAM, stockage)

## Introduction

Avant d'installer MicroK8s, une question essentielle se pose : "Mon ordinateur est-il assez puissant ?" La bonne nouvelle, c'est que MicroK8s a été conçu pour être **léger et accessible**. Vous n'avez pas besoin d'un serveur de la NASA pour commencer !

Dans cette section, nous allons voir en détail les ressources nécessaires (CPU, RAM, stockage) et vous aider à évaluer si votre matériel est adapté. Spoiler : il y a de fortes chances que votre ordinateur actuel fasse parfaitement l'affaire !

## Comprendre les ressources système

Avant de parler de chiffres, comprenons ce que chaque ressource fait dans le contexte de Kubernetes.

### CPU (Processeur)

**Rôle dans Kubernetes :**
Le CPU exécute les calculs et les processus. Dans Kubernetes, cela inclut :
- Les composants du control plane (API Server, Scheduler, etc.)
- Les applications que vous déployez dans des pods
- Les processus système (kubelet, kube-proxy)

**Analogie :**
Imaginez le CPU comme le nombre de chefs dans une cuisine. Plus vous avez de chefs (cœurs CPU), plus vous pouvez préparer de plats simultanément.

**Ce qui consomme du CPU :**
- Démarrage de nouveaux pods (pics temporaires)
- Applications gourmandes en calcul
- Compilations et builds
- Monitoring intensif

### RAM (Mémoire)

**Rôle dans Kubernetes :**
La RAM stocke temporairement les données et les processus en cours d'exécution :
- L'état du cluster (dans Dqlite/etcd)
- Les composants Kubernetes en mémoire
- Vos applications et leurs données temporaires
- Les caches système

**Analogie :**
La RAM est comme le plan de travail d'une cuisine. Plus vous avez d'espace (de RAM), plus vous pouvez avoir d'ingrédients et d'ustensiles à portée de main simultanément.

**Ce qui consomme de la RAM :**
- MicroK8s lui-même (~500-800 Mo)
- Chaque pod que vous déployez
- Les addons activés (dashboard, prometheus, etc.)
- Les données en cache

### Stockage (Disque)

**Rôle dans Kubernetes :**
Le stockage conserve les données persistantes :
- L'installation de MicroK8s
- Les images de conteneurs téléchargées
- Les logs du système
- Les volumes persistants de vos applications
- Les données de vos bases de données

**Analogie :**
Le stockage est comme le garde-manger et le réfrigérateur. Vous y conservez tout ce dont vous avez besoin de manière durable.

**Ce qui consomme du stockage :**
- Installation MicroK8s : ~2-3 Go
- Images Docker : 100 Mo à plusieurs Go par image
- Logs : croissance continue si non gérés
- Vos applications et leurs données

## Les trois niveaux de configuration

Nous allons définir trois configurations types selon vos objectifs.

### Configuration Minimale : "Je veux juste essayer"

**Caractéristiques matérielles :**
- **CPU :** 2 cœurs (dual-core)
- **RAM :** 4 Go
- **Stockage :** 20 Go d'espace libre
- **Type de disque :** HDD acceptable

**Ce que vous pouvez faire :**
- Installer MicroK8s
- Déployer quelques applications simples (2-3 pods)
- Apprendre les concepts de base
- Suivre des tutoriels
- Activer 2-3 addons légers (dns, dashboard)

**Limitations :**
- Pas de déploiements gourmands
- Monitoring limité
- Peu d'applications simultanées
- Performances moyennes

**Exemples de matériel :**
- Laptop de 5-7 ans
- Raspberry Pi 4 (4 Go)
- PC bureautique basique
- Machine virtuelle minimale

**Verdict :**
C'est le strict minimum pour découvrir MicroK8s. Parfait pour un premier contact, mais vous serez vite limité si vous voulez faire plus.

### Configuration Recommandée : "Je veux vraiment apprendre"

**Caractéristiques matérielles :**
- **CPU :** 4 cœurs
- **RAM :** 8 Go
- **Stockage :** 50 Go d'espace libre (SSD recommandé)
- **Type de disque :** SSD fortement recommandé

**Ce que vous pouvez faire :**
- Installation complète avec tous les addons de base
- Déployer 10-15 applications simultanément
- Activer monitoring complet (Prometheus + Grafana)
- Héberger des bases de données
- Suivre toute la formation confortablement
- Faire des expérimentations sans limitation

**Avantages :**
- Expérience fluide et réactive
- Tous les addons disponibles
- Plusieurs projets simultanés
- Apprentissage sans frustration technique

**Exemples de matériel :**
- Laptop moderne (3-5 ans)
- PC desktop milieu de gamme
- Mini-PC type NUC
- MacBook de génération récente
- Machine virtuelle bien dimensionnée

**Verdict :**
C'est le sweet spot ! Cette configuration offre une expérience confortable pour apprendre et expérimenter sans limitations frustrantes. **C'est notre recommandation principale.**

### Configuration Idéale : "Je veux un vrai lab complet"

**Caractéristiques matérielles :**
- **CPU :** 6-8 cœurs (ou plus)
- **RAM :** 16 Go (ou plus)
- **Stockage :** 100 Go+ d'espace libre (SSD obligatoire)
- **Type de disque :** SSD NVMe pour performances optimales

**Ce que vous pouvez faire :**
- Tout ce que MicroK8s peut offrir
- Cluster multi-nœuds
- Nombreuses applications en production
- Stack d'observabilité complète (Prometheus, Grafana, Loki, Jaeger)
- Service mesh (Istio, Linkerd)
- Hébergement de services personnels 24/7
- Simulations de charge
- Machine learning (Kubeflow)

**Avantages :**
- Aucune limitation
- Performances excellentes
- Multi-projets complexes
- Environnement proche production
- Évolution vers multi-nœuds facile

**Exemples de matériel :**
- PC gaming moderne
- Workstation
- Serveur dédié entrée/gamme
- Mac Studio / Mac Mini M-series
- Mini-PC haut de gamme (Intel NUC i7/i9)

**Verdict :**
Configuration de rêve pour un homelab complet ! Si vous avez ce matériel disponible, vous aurez une expérience premium et pourrez explorer toutes les facettes de Kubernetes.

## Tableau récapitulatif des configurations

| Composant | Minimal | Recommandé | Idéal |
|-----------|---------|------------|-------|
| **CPU** | 2 cœurs | 4 cœurs | 6-8 cœurs |
| **RAM** | 4 Go | 8 Go | 16 Go+ |
| **Stockage libre** | 20 Go | 50 Go | 100 Go+ |
| **Type disque** | HDD ok | SSD recommandé | SSD NVMe |
| **Pods simultanés** | 5-10 | 10-20 | 30+ |
| **Addons** | 2-3 légers | Tous essentiels | Tous + avancés |
| **Cas d'usage** | Découverte | Apprentissage | Lab complet |
| **Coût** | 0€ (matériel existant) | 0-300€ | 300-800€ |

## Détails par ressource

### CPU : Comprendre les cœurs

**Architecture de base :**
- 1 cœur = 1 unité d'exécution
- Dual-core = 2 cœurs
- Quad-core = 4 cœurs
- Hexa-core = 6 cœurs
- Octa-core = 8 cœurs

**Hyperthreading / SMT :**
Certains CPU ont l'hyperthreading (Intel) ou SMT (AMD), qui double les threads logiques.

Exemple : Un CPU 4 cœurs avec hyperthreading = 4 cœurs physiques = 8 threads logiques

**Comment vérifier votre CPU :**

**Linux :**
```bash
# Voir le nombre de cœurs
lscpu | grep "^CPU(s)"
nproc

# Voir les détails du processeur
lscpu
```

**Windows :**
```
Gestionnaire des tâches → Onglet Performance → CPU
Ou : systeminfo dans PowerShell
```

**macOS :**
```bash
sysctl -n hw.ncpu
system_profiler SPHardwareDataType
```

**Recommandations CPU :**
- **Minimum :** Tout CPU dual-core moderne (Intel i3, AMD Ryzen 3, Apple M1)
- **Recommandé :** Quad-core (Intel i5, AMD Ryzen 5, Apple M1/M2)
- **Idéal :** Hexa-core ou plus (Intel i7/i9, AMD Ryzen 7/9, Apple M1 Pro/Max)

**Note importante :** Un CPU récent dual-core sera souvent plus performant qu'un vieux quad-core. L'architecture compte autant que le nombre de cœurs !

### RAM : Planifier votre mémoire

**Décomposition de la consommation RAM :**

**MicroK8s de base (sans addons) :**
```
Composants système :      500-800 Mo
- API Server             ~150 Mo
- Scheduler              ~50 Mo
- Controller Manager     ~100 Mo
- Kubelet                ~100 Mo
- Kube-proxy             ~50 Mo
- Dqlite                 ~50-100 Mo
- Containerd             ~100 Mo
```

**Avec addons essentiels activés :**
```
DNS (CoreDNS)            +50 Mo
Dashboard                +100 Mo
Storage                  +20 Mo
Ingress (NGINX)          +100-200 Mo
Registry                 +100 Mo

Total avec essentiels :  ~1.2-1.5 Go
```

**Avec monitoring complet :**
```
Prometheus               +500-800 Mo
Grafana                  +200-300 Mo
Node-exporter            +50 Mo

Total avec monitoring :  ~2.5-3 Go pour MicroK8s
```

**Vos applications :**
Chaque pod que vous déployez consomme de la RAM. Exemples :
- Application web simple : 50-200 Mo
- Base de données (PostgreSQL, MySQL) : 200-500 Mo
- Application Java : 300-800 Mo
- Application Node.js : 100-300 Mo

**Calcul pratique :**

Configuration 8 Go RAM :
```
Système d'exploitation :  1-2 Go
MicroK8s + monitoring :   2.5-3 Go
Applications (5-10) :     2-3 Go
Marge de sécurité :       0.5-1 Go
------------------------
Total :                   ~7-8 Go
```

**Comment vérifier votre RAM :**

**Linux :**
```bash
free -h
# ou
cat /proc/meminfo
```

**Windows :**
```
Gestionnaire des tâches → Onglet Performance → Mémoire
```

**macOS :**
```bash
sysctl hw.memsize
# Ou Moniteur d'activité → Onglet Mémoire
```

**Recommandations RAM :**
- **4 Go :** Suffisant pour débuter, mais serré pour la pratique intensive
- **8 Go :** Sweet spot pour l'apprentissage et l'expérimentation
- **16 Go :** Confort maximal, aucune limitation
- **32 Go+ :** Pour des labs très avancés ou clusters multi-nœuds

**Astuce :** Si vous avez un laptop avec 8 Go et que vous voulez plus, ajouter de la RAM coûte généralement 50-100€ et transforme complètement l'expérience !

### Stockage : Types et besoins

**Types de stockage :**

**HDD (Hard Disk Drive) :**
- Technologie mécanique (plateaux rotatifs)
- Moins cher
- Plus lent (100-150 Mo/s en lecture)
- Adapté pour minimum viable

**SSD SATA (Solid State Drive) :**
- Technologie électronique (mémoire flash)
- Plus rapide (400-550 Mo/s)
- Prix abordable
- **Recommandé pour MicroK8s**

**SSD NVMe (Non-Volatile Memory Express) :**
- Encore plus rapide (1500-3500 Mo/s ou plus)
- Prix modéré à élevé
- Idéal pour performances optimales

**Pourquoi le type de disque est important :**

Kubernetes effectue beaucoup d'opérations I/O :
- Lecture/écriture dans la base de données (Dqlite)
- Chargement des images de conteneurs
- Écriture des logs
- Gestion des volumes persistants
- Démarrage et arrêt de pods

**Impact concret :**
- HDD : Démarrage d'un pod en 5-10 secondes
- SSD SATA : Démarrage d'un pod en 2-3 secondes
- SSD NVMe : Démarrage d'un pod en 1-2 secondes

**Décomposition du stockage nécessaire :**

```
Installation MicroK8s :          2-3 Go
Système d'exploitation :         10-20 Go (déjà utilisé)
Images Docker de base :          2-5 Go
  - nginx                        ~150 Mo
  - postgres                     ~400 Mo
  - ubuntu                       ~80 Mo
  - etc.

Addons et dépendances :          3-5 Go
Logs (croissance continue) :     1-5 Go
Volumes persistants :            Variable selon usage
Marge de sécurité :              5-10 Go

TOTAL RECOMMANDÉ :               50 Go minimum
```

**Comment vérifier votre stockage :**

**Linux :**
```bash
df -h
# Voir le type de disque
lsblk
```

**Windows :**
```
Explorateur de fichiers → Ce PC
Ou : wmic diskdrive get size,model,interfacetype
```

**macOS :**
```bash
df -h
# Ou : À propos de ce Mac → Stockage
```

**Recommandations stockage :**
- **Type :** SSD fortement recommandé (SATA ou NVMe)
- **Espace libre minimum :** 20 Go (juste pour essayer)
- **Espace recommandé :** 50 Go (confortable)
- **Espace idéal :** 100 Go+ (aucune limitation)

**Astuce :** Les SSD de 250 Go coûtent aujourd'hui 30-50€. C'est l'upgrade le plus rentable pour améliorer les performances !

## Cas particuliers et considérations

### Machines virtuelles (VM)

Si vous installez MicroK8s dans une VM :

**Considérations :**
- Ajoutez 10-20% de overhead pour l'hyperviseur
- La VM elle-même consomme des ressources de l'hôte
- Les performances I/O peuvent être réduites

**Recommandations VM :**
- **CPU :** Allouez 4 cœurs minimum
- **RAM :** Allouez 8 Go minimum
- **Stockage :** 60 Go minimum
- **Disque :** Utilisez thin provisioning si possible

**Configuration VirtualBox/VMware :**
```
CPU : 4 cœurs
RAM : 8192 Mo
Disque : 60 Go dynamique (thin provisioning)
Réseau : NAT ou Bridged selon besoins
```

### Raspberry Pi

MicroK8s fonctionne sur Raspberry Pi, mais avec des limitations :

**Raspberry Pi 4 (4 Go RAM) :**
- ✅ Fonctionne
- ⚠️ Limité pour addons gourmands
- ❌ Pas adapté pour monitoring complet
- ✅ Parfait pour apprendre les bases

**Raspberry Pi 4 (8 Go RAM) :**
- ✅ Bien adapté pour apprentissage
- ✅ Peut héberger plusieurs services légers
- ⚠️ CPU ARM peut limiter certaines images
- ✅ Excellent pour cluster multi-nœuds (plusieurs Pi)

**Note :** Le Pi est excellent pour l'edge computing et l'apprentissage, mais moins pour un lab complet avec tous les addons.

### Laptop vs Desktop vs Serveur dédié

**Laptop (portable) :**
- ✅ Portable, pratique pour apprendre partout
- ✅ Consommation électrique faible
- ⚠️ Performances thermiques (throttling possible)
- ⚠️ Pas idéal pour 24/7
- **Recommandation :** Parfait pour apprentissage et développement

**Desktop (PC bureau) :**
- ✅ Meilleures performances thermiques
- ✅ Plus facile à upgrader (RAM, SSD)
- ✅ Peut tourner 24/7
- ✅ Bon rapport performance/prix
- **Recommandation :** Excellent pour lab permanent

**Serveur dédié / Mini-PC :**
- ✅ Conçu pour tourner 24/7
- ✅ Silencieux (souvent fanless)
- ✅ Faible consommation
- ✅ Rack-mountable (serveurs)
- 💰 Plus cher
- **Recommandation :** Idéal pour homelab sérieux

### Consommation électrique

**Coûts annuels approximatifs (24/7) :**

Avec un prix du kWh à 0.20€ (moyenne France) :

- **Laptop moderne :** 20-40W → 35-70€/an
- **Mini-PC :** 15-30W → 26-52€/an
- **PC Desktop :** 50-150W → 87-262€/an
- **Raspberry Pi 4 :** 5-8W → 8-14€/an
- **Serveur dédié :** 40-100W → 70-175€/an

**Optimisation :**
- Utilisez du matériel moderne (plus efficace)
- Considérez les mini-PC pour 24/7
- Les Pi sont imbattables en consommation

## Évaluer votre matériel actuel

### Checklist d'évaluation

Répondez à ces questions pour savoir si votre matériel convient :

**1. Combien de cœurs CPU avez-vous ?**
- [ ] 2 cœurs → Configuration minimale possible
- [ ] 4 cœurs → Configuration recommandée ✅
- [ ] 6+ cœurs → Configuration idéale ⭐

**2. Combien de RAM avez-vous ?**
- [ ] 4 Go → Minimum viable
- [ ] 8 Go → Recommandé ✅
- [ ] 16 Go+ → Idéal ⭐

**3. Quel type de disque avez-vous ?**
- [ ] HDD → Fonctionnel mais lent
- [ ] SSD SATA → Recommandé ✅
- [ ] SSD NVMe → Optimal ⭐

**4. Combien d'espace libre avez-vous ?**
- [ ] 20-40 Go → Juste suffisant
- [ ] 50-80 Go → Confortable ✅
- [ ] 100 Go+ → Excellent ⭐

**Interprétation :**
- **Tous verts (✅) ou étoiles (⭐) :** Parfait, vous êtes prêt !
- **Quelques minimums :** Vous pouvez commencer, mais envisagez des upgrades
- **Que des minimums :** Possible mais limité, considérez un upgrade ou une autre machine

### Scénarios courants

**Scénario 1 : "J'ai un laptop de 2018 avec i5, 8 Go RAM, SSD 256 Go"**
✅ **Verdict :** Excellent ! Configuration recommandée, vous pouvez suivre toute la formation confortablement.

**Scénario 2 : "J'ai un vieux PC de 2015 avec 4 cœurs, 4 Go RAM, HDD 500 Go"**
⚠️ **Verdict :** Possible pour débuter, mais :
- Envisagez d'ajouter 4 Go de RAM (priorité 1)
- Un SSD de 250 Go transformerait l'expérience (priorité 2)
- Coût upgrade : ~80€

**Scénario 3 : "J'ai un MacBook M1 avec 8 Go RAM, SSD 256 Go"**
✅ **Verdict :** Parfait ! Les puces M1/M2 sont excellentes pour MicroK8s, même avec "seulement" 8 Go.

**Scénario 4 : "J'ai un Raspberry Pi 4 8 Go"**
✅ **Verdict :** Bien adapté pour apprentissage et petits projets. Limitations sur addons gourmands.

**Scénario 5 : "J'ai un PC gaming récent : Ryzen 7, 16 Go RAM, SSD NVMe 1 To"**
⭐ **Verdict :** Configuration de rêve ! Vous pourrez tout faire sans aucune limitation.

## Upgrades recommandés

Si votre matériel est limite, voici les upgrades par ordre d'impact :

### Priority 1 : RAM (si < 8 Go)

**Impact :** ⭐⭐⭐⭐⭐ (Maximum)

Passer de 4 Go à 8 Go transforme complètement l'expérience.

**Coût :** 30-60€ pour 4-8 Go de RAM DDR4
**Difficulté :** Facile (2 minutes sur desktop, 5-10 minutes sur laptop)
**ROI :** Excellent

### Priority 2 : SSD (si HDD)

**Impact :** ⭐⭐⭐⭐ (Très élevé)

Le passage HDD → SSD rend tout 3-5x plus rapide.

**Coût :** 30-50€ pour SSD SATA 250 Go
**Difficulté :** Moyenne (nécessite clonage ou réinstallation)
**ROI :** Excellent

### Priority 3 : Plus de stockage

**Impact :** ⭐⭐⭐ (Moyen)

Si vous êtes limité en espace.

**Coût :** 60-100€ pour SSD 500 Go ou 1 To
**Difficulté :** Facile si second disque, moyenne si remplacement
**ROI :** Bon si vous êtes vraiment limité

### Priority 4 : CPU

**Impact :** ⭐⭐ (Faible à moyen)

Souvent implique changement de machine.

**Coût :** Variable (200-1000€ selon solution)
**Difficulté :** Difficile (nouveau PC ou upgrade processeur)
**ROI :** Moyen (investissement important)

**Note :** Ne changez le CPU que si tout le reste est déjà optimal.

## Recommandations finales

### Vous pouvez commencer si vous avez :

**Minimum absolument nécessaire :**
- 2 cœurs CPU
- 4 Go RAM
- 20 Go stockage libre

**Pour une bonne expérience :**
- 4 cœurs CPU
- 8 Go RAM
- 50 Go stockage libre (SSD)

### Ne vous arrêtez pas au matériel !

**Message important :** Ne laissez pas le matériel être un frein à votre apprentissage !

- Un laptop de 5 ans avec 4 Go RAM peut faire tourner MicroK8s
- Vous n'avez pas besoin du dernier MacBook Pro pour apprendre
- L'important est de **commencer** et de **pratiquer**
- Vous pouvez upgrader progressivement si nécessaire

**Rappel :** Les plus grands experts Kubernetes ont commencé avec du matériel modeste. Ce qui compte, c'est votre motivation et votre pratique !

## Conclusion

Maintenant vous savez exactement ce dont vous avez besoin pour installer MicroK8s :

**Configuration minimale :** 2 cœurs, 4 Go RAM, 20 Go stockage
**Configuration recommandée :** 4 cœurs, 8 Go RAM, 50 Go SSD ✅
**Configuration idéale :** 6+ cœurs, 16 Go RAM, 100 Go SSD NVMe ⭐

**La bonne nouvelle :** La plupart des ordinateurs modernes (moins de 5 ans) dépassent déjà la configuration recommandée !

Dans la section suivante, nous verrons les systèmes d'exploitation supportés et comment préparer votre système pour l'installation.

---

**Points clés à retenir :**
- **Configuration minimale :** 2 cœurs, 4 Go RAM, 20 Go → Découverte
- **Configuration recommandée :** 4 cœurs, 8 Go RAM, 50 Go SSD → Apprentissage confortable ✅
- **Configuration idéale :** 6+ cœurs, 16 Go RAM, 100 Go+ SSD → Lab complet
- MicroK8s consomme ~500-800 Mo de RAM de base
- Avec monitoring complet : ~2.5-3 Go
- SSD recommandé pour performances (3-5x plus rapide que HDD)
- Upgrades prioritaires : RAM puis SSD
- Ne laissez pas le matériel freiner votre apprentissage !
- La plupart des ordinateurs modernes conviennent parfaitement

⏭️ [Systèmes d'exploitation supportés](/02-installation-et-configuration-initiale/02-systemes-dexploitation-supportes.md)
