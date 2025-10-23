🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 6.1 Concepts de stockage dans Kubernetes

## Introduction

Le stockage est l'un des aspects les plus importants et parfois les plus complexes de Kubernetes. Contrairement aux conteneurs qui sont éphémères (temporaires) par nature, certaines applications ont besoin de conserver des données de manière permanente. Pensez à une base de données : si vous perdez les données à chaque redémarrage du conteneur, votre application devient inutilisable !

Dans cette section, nous allons explorer les concepts fondamentaux du stockage dans Kubernetes, en commençant par comprendre pourquoi et comment Kubernetes gère le stockage de manière différente des applications traditionnelles.

## Pourquoi le stockage est-il particulier dans Kubernetes ?

### Le problème de l'éphémérité des conteneurs

Par défaut, les conteneurs sont conçus pour être **éphémères** :
- Ils peuvent être créés, détruits et recréés à tout moment
- Toutes les données stockées à l'intérieur d'un conteneur disparaissent lorsque celui-ci est supprimé
- Kubernetes peut déplacer vos pods d'un nœud à un autre pour optimiser les ressources

Imaginons un scénario simple : vous déployez une application web avec une base de données dans un pod. Un problème survient et Kubernetes redémarre le pod. Sans stockage persistant, toutes vos données sont perdues !

### Ce qui rend le stockage complexe

Le stockage dans Kubernetes doit résoudre plusieurs défis :

1. **Persistance** : Les données doivent survivre aux redémarrages et aux suppressions de pods
2. **Portabilité** : Un pod peut être déplacé d'un nœud à un autre, mais doit retrouver ses données
3. **Indépendance** : Le cycle de vie du stockage doit être indépendant du cycle de vie des pods
4. **Variété** : Il existe de nombreux types de stockage (disques locaux, NFS, cloud storage, etc.)

## Les deux mondes du stockage dans Kubernetes

Kubernetes distingue deux types de stockage selon leur durée de vie :

### 1. Stockage éphémère

C'est le stockage par défaut d'un conteneur. Il est :
- **Temporaire** : disparaît avec le conteneur
- **Rapide** : car il est local au nœud
- **Limité** : à l'espace disque du nœud

**Cas d'usage** :
- Fichiers de cache
- Fichiers temporaires
- Logs qui seront exportés ailleurs
- Données de configuration non critiques

### 2. Stockage persistant

C'est un stockage qui survit aux conteneurs. Il est :
- **Permanent** : les données restent même après la suppression du pod
- **Partageable** : peut être accessible depuis différents pods (selon le type)
- **Indépendant** : géré séparément du cycle de vie des applications

**Cas d'usage** :
- Bases de données
- Fichiers uploadés par les utilisateurs
- Logs à conserver
- Tout ce qui doit survivre aux redémarrages

## Les concepts clés du stockage Kubernetes

Pour bien comprendre le stockage dans Kubernetes, il faut maîtriser plusieurs concepts interconnectés. Voyons-les de manière progressive.

### Volume : le concept de base

Un **Volume** dans Kubernetes est un répertoire accessible aux conteneurs d'un pod. C'est l'abstraction la plus simple du stockage.

**Caractéristiques d'un Volume** :
- Il est défini dans la spécification du pod
- Sa durée de vie est liée au pod (pas au conteneur individuel)
- Il peut être monté dans un ou plusieurs conteneurs du même pod
- Il peut provenir de différentes sources (disque local, NFS, cloud, etc.)

**Analogie** : Pensez à un volume comme à un disque dur externe que vous branchez sur votre ordinateur. Le pod est l'ordinateur, et les conteneurs sont les applications qui peuvent utiliser ce disque.

### Volume Mount : le point de montage

Un **Volume Mount** est le lien entre un volume et un conteneur. Il définit :
- **Quel volume** sera utilisé
- **Où** dans le conteneur ce volume sera accessible (le chemin)

**Exemple conceptuel** :
```
Volume "mes-données" → monté dans le conteneur à /app/data
```

Cela signifie que tout fichier écrit dans `/app/data` à l'intérieur du conteneur sera en réalité stocké dans le volume "mes-données".

### Types de Volumes

Kubernetes supporte de nombreux types de volumes. Voici les plus courants pour débuter :

#### emptyDir
- Volume vide créé avec le pod
- Partagé entre tous les conteneurs du pod
- **Supprimé** quand le pod est détruit
- Utile pour le partage de données temporaires entre conteneurs

#### hostPath
- Monte un répertoire ou un fichier du nœud dans le pod
- Les données persistent même si le pod est détruit
- **Attention** : les données sont liées au nœud spécifique
- À éviter en production (sauf cas particuliers)

#### persistentVolumeClaim (PVC)
- Le type le plus important pour le stockage persistant
- Permet de demander du stockage sans connaître les détails techniques
- Nous l'étudierons en détail dans les sections suivantes

### PersistentVolume (PV) : le stockage physique

Un **PersistentVolume** (PV) représente une ressource de stockage physique dans le cluster :
- C'est un morceau de stockage réel (disque, NFS, stockage cloud, etc.)
- Il est créé par l'administrateur du cluster ou provisionné dynamiquement
- Il existe indépendamment de tout pod
- Il a une taille, des modes d'accès et des caractéristiques spécifiques

**Analogie** : Un PV est comme un espace de stockage que le propriétaire d'un immeuble (l'administrateur) met à disposition. Il existe physiquement, avec une certaine capacité et des caractéristiques précises.

### PersistentVolumeClaim (PVC) : la demande de stockage

Un **PersistentVolumeClaim** (PVC) est une demande de stockage faite par un utilisateur :
- L'application "réclame" du stockage avec certaines caractéristiques (taille, performance, etc.)
- Kubernetes trouve un PV qui correspond à la demande
- Le PVC est ensuite utilisé dans les pods comme n'importe quel volume

**Analogie** : Si le PV est l'espace de stockage disponible, le PVC est comme un bail de location. L'utilisateur demande un espace avec certaines caractéristiques, et le système lui attribue un espace correspondant.

### Le principe de séparation des responsabilités

Cette architecture en deux niveaux (PV/PVC) crée une séparation importante :

**Administrateur du cluster** :
- Crée et gère les PersistentVolumes
- Configure le stockage physique
- Définit les classes de stockage disponibles

**Développeur/Utilisateur** :
- Crée des PersistentVolumeClaims
- Utilise les PVC dans les pods
- N'a pas besoin de connaître les détails du stockage sous-jacent

Cette séparation permet de simplifier le déploiement des applications et de rendre le code plus portable.

## StorageClass : la couche d'abstraction ultime

Une **StorageClass** définit un "type" de stockage disponible dans le cluster :
- Elle décrit les caractéristiques du stockage (rapide, lent, répliqué, etc.)
- Elle permet le **provisionnement dynamique** : les PV sont créés automatiquement à la demande
- L'utilisateur demande simplement un type de stockage, sans se soucier de l'implémentation

**Exemple de StorageClasses** :
- `fast-ssd` : stockage sur disques SSD rapides
- `standard` : stockage standard pour usage général
- `backup` : stockage lent mais redondant pour les sauvegardes

### Provisionnement statique vs dynamique

**Provisionnement statique** :
1. L'administrateur crée manuellement des PV
2. L'utilisateur crée un PVC
3. Kubernetes associe le PVC à un PV existant compatible

**Provisionnement dynamique** :
1. L'utilisateur crée un PVC en spécifiant une StorageClass
2. Kubernetes crée automatiquement un PV correspondant
3. Le PVC est automatiquement lié au nouveau PV

Le provisionnement dynamique est beaucoup plus pratique et moderne, c'est ce que nous utiliserons principalement avec MicroK8s.

## Les modes d'accès (Access Modes)

Le stockage peut être accessible de différentes manières :

### ReadWriteOnce (RWO)
- Le volume peut être monté en lecture-écriture par **un seul nœud**
- C'est le mode le plus courant
- Idéal pour : bases de données, applications avec un seul pod

### ReadOnlyMany (ROX)
- Le volume peut être monté en lecture seule par **plusieurs nœuds**
- Utile pour : distribution de fichiers statiques, configuration partagée

### ReadWriteMany (RWX)
- Le volume peut être monté en lecture-écriture par **plusieurs nœuds**
- Plus complexe à mettre en place
- Nécessite un système de fichiers distribué (NFS, Ceph, etc.)
- Utile pour : applications nécessitant un stockage partagé

**Important** : Tous les types de stockage ne supportent pas tous les modes d'accès. Par exemple, un disque local ne peut être qu'en mode RWO.

## Le cycle de vie du stockage

Comprendre comment le stockage évolue dans le temps est essentiel :

### Phases d'un PersistentVolume

1. **Available** (Disponible)
   - Le PV existe et n'est lié à aucun PVC
   - Prêt à être réclamé

2. **Bound** (Lié)
   - Le PV est lié à un PVC
   - Il est utilisé par un ou plusieurs pods

3. **Released** (Libéré)
   - Le PVC a été supprimé
   - Le PV n'est plus utilisé mais peut contenir des données

4. **Failed** (Échoué)
   - Une erreur s'est produite lors de la récupération automatique

### Politiques de récupération (Reclaim Policy)

Que se passe-t-il quand un PVC est supprimé ? Cela dépend de la politique de récupération :

**Retain** (Conserver)
- Les données sont conservées
- Le PV passe en statut "Released"
- Un administrateur doit manuellement nettoyer et libérer le PV

**Delete** (Supprimer)
- Le PV et le stockage sous-jacent sont supprimés automatiquement
- Les données sont perdues définitivement
- C'est la politique par défaut pour le provisionnement dynamique

**Recycle** (Recycler) - Déprécié
- Les données sont supprimées (`rm -rf`)
- Le PV redevient disponible
- N'est plus recommandé

## Considérations importantes pour les débutants

### Perte de données

**Attention** : Le stockage persistant ne signifie pas indestructible !
- Si un PV est supprimé avec la politique "Delete", les données sont perdues
- Les erreurs de configuration peuvent causer des pertes de données
- Toujours avoir des sauvegardes pour les données critiques

### Performance

Le type de stockage impacte directement les performances :
- **Stockage local** : très rapide mais non partageable entre nœuds
- **Stockage réseau** (NFS, iSCSI) : plus lent mais partageable
- **Stockage cloud** : performances variables selon le service

### Coûts

- Le stockage persistant consomme des ressources réelles
- Dans le cloud, le stockage est souvent facturé au Go utilisé et au temps
- Pensez à nettoyer les volumes inutilisés

### Sécurité

- Les données dans les volumes peuvent être sensibles
- Kubernetes offre des mécanismes de chiffrement
- Contrôlez l'accès aux volumes avec RBAC

## Schéma récapitulatif des concepts

Voici comment tous ces concepts s'articulent ensemble :

```
┌─────────────────────────────────────────────────────────────┐
│                         Application                         │
│  ┌───────────────────────────────────────────────────────┐  │
│  │                        Pod                            │  │
│  │  ┌──────────────┐              ┌──────────────┐       │  │
│  │  │  Conteneur 1 │              │  Conteneur 2 │       │  │
│  │  │  /app/data   │              │  /logs       │       │  │
│  │  └───────┬──────┘              └───────┬──────┘       │  │
│  │          │                             │              │  │
│  │          │ Volume Mount                │              │  │
│  │          │                             │              │  │
│  │  ┌───────▼─────────────────────────────▼────────────┐ │  │
│  │  │              Volume (PVC)                        │ │  │
│  │  └──────────────────────┬───────────────────────────┘ │  │
│  └─────────────────────────┼─────────────────────────────┘  │
└────────────────────────────┼────────────────────────────────┘
                             │
                             │ Liaison (Binding)
                             │
                   ┌─────────▼──────────┐
                   │  PersistentVolume  │
                   │    (Stockage réel) │
                   └─────────┬──────────┘
                             │
                   ┌─────────▼──────────┐
                   │   StorageClass     │
                   │  (Type de stockage)│
                   └────────────────────┘
```

## Dans MicroK8s : la simplicité avant tout

MicroK8s simplifie considérablement la gestion du stockage grâce à son système d'addons :

- **Addon hostpath-storage** : Active automatiquement un provisionnement dynamique simple
- **StorageClass par défaut** : Configurée automatiquement
- **Pas de configuration complexe** : Tout fonctionne "out of the box"

C'est l'un des grands avantages de MicroK8s pour un environnement de lab ou de développement !

## Résumé des concepts clés

Avant de passer à la pratique, retenons l'essentiel :

1. **Volume** : Un répertoire accessible dans un pod (concept de base)

2. **PersistentVolume (PV)** : Une ressource de stockage réelle dans le cluster (créée par l'admin)

3. **PersistentVolumeClaim (PVC)** : Une demande de stockage faite par l'utilisateur (créée par le développeur)

4. **StorageClass** : Un type de stockage avec provisionnement dynamique (définie par l'admin)

5. **Volume Mount** : Le lien entre un volume et un chemin dans un conteneur

6. **Access Modes** : Comment le stockage peut être partagé (RWO, ROX, RWX)

7. **Reclaim Policy** : Que faire du stockage quand le PVC est supprimé (Retain, Delete)

## Questions à se poser avant de choisir un stockage

Lorsque vous devez ajouter du stockage à votre application, posez-vous ces questions :

1. **Les données doivent-elles survivre aux redémarrages ?**
   - Non → emptyDir suffit
   - Oui → Utiliser un PVC

2. **Quelle quantité de stockage nécessaire ?**
   - Définira la taille du PVC

3. **Combien de pods accéderont au stockage ?**
   - Un seul → RWO
   - Plusieurs (lecture seule) → ROX
   - Plusieurs (lecture-écriture) → RWX (plus complexe)

4. **Les performances sont-elles critiques ?**
   - Oui → Choisir une StorageClass rapide (SSD)
   - Non → StorageClass standard

5. **Que se passe-t-il si je supprime l'application ?**
   - Garder les données → Reclaim Policy "Retain"
   - Supprimer les données → Reclaim Policy "Delete"

## Prochaines étapes

Maintenant que vous comprenez les concepts fondamentaux du stockage dans Kubernetes, nous allons pouvoir :

- Explorer en détail les **Volumes et Volume Mounts** (section 6.2)
- Apprendre à créer et gérer des **PersistentVolumes** (section 6.3)
- Maîtriser les **PersistentVolumeClaims** (section 6.4)
- Configurer des **StorageClasses** (section 6.5)
- Déployer des applications avec stockage persistant

Chaque concept sera illustré par des exemples pratiques concrets avec MicroK8s.

## Conclusion

Le stockage dans Kubernetes peut sembler complexe au premier abord, mais il repose sur des concepts logiques et bien structurés. La séparation entre PV, PVC et StorageClass permet une grande flexibilité tout en maintenant une simplicité d'utilisation pour les développeurs.

Avec MicroK8s, la complexité de configuration est largement réduite grâce aux addons préconfigurés. Vous pouvez vous concentrer sur l'essentiel : faire en sorte que vos applications conservent leurs données de manière fiable.

Dans les prochaines sections, nous passerons à la pratique et vous verrez que, une fois les concepts compris, l'utilisation du stockage dans Kubernetes devient naturelle et intuitive.

---

**Points clés à retenir** :
- Les conteneurs sont éphémères, le stockage persistant est essentiel pour les données importantes
- Le système PV/PVC sépare les responsabilités entre admin et développeur
- Les StorageClasses permettent le provisionnement dynamique automatique
- MicroK8s simplifie énormément la configuration avec son addon hostpath-storage
- Toujours penser aux sauvegardes, même avec du stockage persistant !

⏭️ [Volumes et Volume Mounts](/06-stockage-persistant/02-volumes-et-volume-mounts.md)
