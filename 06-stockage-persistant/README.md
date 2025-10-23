🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 6. Stockage Persistant

## Introduction à la section

Bienvenue dans cette section cruciale de notre formation MicroK8s ! Le stockage persistant est l'un des sujets les plus importants à maîtriser pour déployer des applications réelles dans Kubernetes.

Jusqu'à présent, nous avons principalement travaillé avec des applications **stateless** (sans état), comme des serveurs web ou des API REST, où les conteneurs peuvent être créés, détruits et remplacés sans conséquence. Mais dans le monde réel, la plupart des applications ont besoin de **conserver des données** :

- Une base de données doit préserver son contenu entre les redémarrages
- Une application web doit sauvegarder les fichiers uploadés par les utilisateurs
- Un système de logs doit stocker les données historiques
- Une application de traitement doit conserver ses fichiers de travail

C'est là que le **stockage persistant** entre en jeu. Cette section vous apprendra tout ce que vous devez savoir pour gérer le stockage dans Kubernetes, depuis les concepts de base jusqu'aux techniques avancées et aux exemples pratiques.

## Pourquoi le stockage est-il un défi dans Kubernetes ?

### Le problème de l'éphémérité

Kubernetes a été conçu avec un principe fondamental : les **conteneurs sont éphémères**. Cela signifie qu'ils peuvent :
- Être créés et détruits à tout moment
- Être déplacés d'un nœud à un autre
- Être remplacés en cas de problème
- Être mis à l'échelle vers le haut ou vers le bas

Cette flexibilité est excellente pour les applications stateless, mais pose un problème majeur pour les données :

```
┌───────────────────────────────────────────────────────────┐
│                SANS STOCKAGE PERSISTANT                   │
├───────────────────────────────────────────────────────────┤
│                                                           │
│  1. Pod créé avec une base de données                     │
│     └─ Données stockées dans le conteneur                 │
│                                                           │
│  2. Utilisateurs ajoutent des données                     │
│     └─ Base de données : 1000 enregistrements             │
│                                                           │
│  3. Pod redémarre (mise à jour, crash, déplacement)       │
│     └─ Conteneur détruit                                  │
│                                                           │
│  4. Nouveau pod créé                                      │
│     └─ Base de données : 0 enregistrement                 │
│                                                           │
│  ❌ Toutes les données sont perdues !                     │
│                                                           │
└───────────────────────────────────────────────────────────┘

┌───────────────────────────────────────────────────────────┐
│                AVEC STOCKAGE PERSISTANT                   │
├───────────────────────────────────────────────────────────┤
│                                                           │
│  1. Pod créé avec volume persistant                       │
│     └─ Données stockées dans le volume externe            │
│                                                           │
│  2. Utilisateurs ajoutent des données                     │
│     └─ Volume persistant : 1000 enregistrements           │
│                                                           │
│  3. Pod redémarre (mise à jour, crash, déplacement)       │
│     └─ Conteneur détruit, volume conservé                 │
│                                                           │
│  4. Nouveau pod créé, reconnecté au même volume           │
│     └─ Volume persistant : 1000 enregistrements           │
│                                                           │
│  ✅ Toutes les données sont préservées !                  │
│                                                           │
└───────────────────────────────────────────────────────────┘
```

### Les défis spécifiques du stockage dans Kubernetes

1. **Portabilité** : Comment garantir que les données restent accessibles même si un pod change de nœud ?

2. **Indépendance** : Comment séparer le cycle de vie du stockage de celui des applications ?

3. **Abstraction** : Comment permettre aux développeurs d'utiliser du stockage sans connaître les détails de l'infrastructure ?

4. **Variété** : Comment supporter différents types de stockage (local, réseau, cloud) avec une interface unifiée ?

5. **Performance** : Comment offrir des performances adaptées à chaque type d'application ?

6. **Sécurité** : Comment protéger les données sensibles et contrôler les accès ?

## Ce que vous allez apprendre dans cette section

Cette section est organisée de manière progressive, du plus simple au plus avancé :

### Niveau Débutant (Sections 6.1 à 6.4)

**Section 6.1 - Concepts de stockage dans Kubernetes**
- Comprendre la différence entre stockage éphémère et persistant
- Découvrir les concepts fondamentaux (Volumes, PV, PVC, StorageClass)
- Apprendre l'architecture du stockage dans Kubernetes

**Section 6.2 - Volumes et Volume Mounts**
- Utiliser des volumes temporaires (emptyDir)
- Accéder au stockage du nœud (hostPath)
- Monter des configurations (ConfigMaps, Secrets)
- Partager des données entre conteneurs

**Section 6.3 - PersistentVolumes (PV)**
- Comprendre ce qu'est un PersistentVolume
- Créer et gérer des PV manuellement
- Découvrir les différents types de stockage
- Comprendre le cycle de vie d'un PV

**Section 6.4 - PersistentVolumeClaims (PVC)**
- Demander du stockage via des PVC
- Comprendre le processus de binding (liaison)
- Utiliser des PVC dans vos pods
- Gérer le cycle de vie des PVC

### Niveau Intermédiaire (Sections 6.5 à 6.6)

**Section 6.5 - StorageClasses**
- Découvrir le provisionnement dynamique
- Créer et configurer des StorageClasses
- Comprendre les différents provisioners
- Optimiser avec l'addon MicroK8s hostpath-storage

**Section 6.6 - Gestion des volumes persistants**
- Surveiller l'utilisation du stockage
- Effectuer des sauvegardes et restaurations
- Redimensionner les volumes (expansion)
- Migrer des données entre volumes
- Nettoyer et maintenir votre infrastructure

### Niveau Avancé (Sections 6.7 à 6.8)

**Section 6.7 - StatefulSets**
- Comprendre les applications stateful vs stateless
- Déployer des applications avec identité stable
- Gérer le stockage dédié par instance
- Maîtriser le scaling et les mises à jour

**Section 6.8 - Exemples pratiques**
- Déployer MySQL, PostgreSQL, MongoDB
- Configurer Redis pour le caching
- Mettre en place Elasticsearch
- Créer une application complète (WordPress + MySQL)

## Vue d'ensemble de l'architecture du stockage

Voici une représentation simplifiée de l'architecture complète du stockage dans Kubernetes :

```
┌────────────────────────────────────────────────────────────┐
│                    ARCHITECTURE GLOBALE                    │
│                                                            │
│  ┌────────────────────────────────────────────────────┐    │
│  │              NIVEAU APPLICATION                    │    │
│  │                                                    │    │
│  │  ┌──────┐  ┌──────┐  ┌──────┐                      │    │
│  │  │ Pod  │  │ Pod  │  │ Pod  │                      │    │
│  │  │      │  │      │  │      │                      │    │
│  │  └──┬───┘  └──┬───┘  └──┬───┘                      │    │
│  │     │         │         │                          │    │
│  └─────┼─────────┼─────────┼──────────────────────────┘    │
│        │         │         │                               │
│        │ Volume  │ Volume  │ Volume                        │
│        │ Mount   │ Mount   │ Mount                         │
│        │         │         │                               │
│  ┌─────▼─────────▼─────────▼─────────────────────────┐     │
│  │         NIVEAU ABSTRACTION                        │     │
│  │                                                   │     │
│  │  ┌─────────────────┐  ┌─────────────────┐         │     │
│  │  │      PVC        │  │      PVC        │         │     │
│  │  │  (Demande)      │  │  (Demande)      │         │     │
│  │  └────────┬────────┘  └─────────┬───────┘         │     │
│  │           │                     │                 │     │
│  │           │ Binding             │                 │     │
│  │           │                     │                 │     │
│  │  ┌────────▼────────┐  ┌─────────▼────────┐        │     │
│  │  │       PV        │  │       PV         │        │     │
│  │  │   (Ressource)   │  │   (Ressource)    │        │     │
│  │  └────────┬────────┘  └──────────┬───────┘        │     │
│  │           │                      │                │     │
│  └───────────┼──────────────────────┼────────────────┘     │
│              │                      │                      │
│  ┌───────────▼──────────────────────▼────────────────┐     │
│  │          NIVEAU PROVISIONNEMENT                   │     │
│  │                                                   │     │
│  │         ┌───────────────────┐                     │     │
│  │         │  StorageClass     │                     │     │
│  │         │  (Type/Recette)   │                     │     │
│  │         └─────────┬─────────┘                     │     │
│  │                   │                               │     │
│  │                   │ Provisioner                   │     │
│  │                   │                               │     │
│  └───────────────────┼───────────────────────────────┘     │
│                      │                                     │
│  ┌───────────────────▼───────────────────────────────┐     │
│  │           NIVEAU STOCKAGE PHYSIQUE                │     │
│  │                                                   │     │
│  │  ┌──────────┐  ┌──────────┐  ┌──────────┐         │     │
│  │  │Disque    │  │  NFS     │  │  Cloud   │         │     │
│  │  │Local     │  │  Server  │  │  Storage │         │     │
│  │  └──────────┘  └──────────┘  └──────────┘         │     │
│  │                                                   │     │
│  └───────────────────────────────────────────────────┘     │
│                                                            │
└────────────────────────────────────────────────────────────┘
```

**Ce schéma montre les différentes couches** :

1. **Niveau Application** : Vos pods et conteneurs qui ont besoin de stockage
2. **Niveau Abstraction** : PVC (demandes) et PV (ressources disponibles)
3. **Niveau Provisionnement** : StorageClasses qui automatisent la création
4. **Niveau Stockage Physique** : Le stockage réel (disques, NFS, cloud)

## Principes clés à retenir

Avant de plonger dans les détails techniques, gardez en tête ces principes fondamentaux :

### 1. Séparation des préoccupations

Kubernetes sépare clairement trois rôles :
- **L'administrateur** configure l'infrastructure de stockage (PV, StorageClass)
- **Le développeur** demande du stockage pour ses applications (PVC)
- **Kubernetes** fait le lien entre les deux automatiquement

Cette séparation permet aux développeurs de ne pas se soucier des détails techniques du stockage.

### 2. Déclaratif, pas impératif

Comme tout dans Kubernetes, le stockage se gère de manière déclarative :
- Vous **déclarez** ce que vous voulez ("Je veux 10Gi de stockage rapide")
- Kubernetes **réalise** votre demande automatiquement
- Vous ne gérez pas les détails de création/attachement/montage

### 3. Indépendance du cycle de vie

Le stockage a son propre cycle de vie, indépendant des applications :
- Un volume peut exister avant qu'une application l'utilise
- Un volume survit au redémarrage ou à la suppression d'un pod
- Un volume peut être réutilisé par différentes applications

### 4. Abstraction multi-niveaux

Kubernetes offre plusieurs niveaux d'abstraction :
- **Niveau bas** : Volumes simples (emptyDir, hostPath)
- **Niveau moyen** : PersistentVolumes et PersistentVolumeClaims
- **Niveau haut** : StorageClasses avec provisionnement dynamique

Vous pouvez choisir le niveau qui correspond à vos besoins.

## Cas d'usage typiques

Pour vous aider à comprendre quand utiliser le stockage persistant, voici quelques scénarios courants :

### Applications nécessitant un stockage persistant

✅ **Bases de données** (MySQL, PostgreSQL, MongoDB)
- Doivent conserver les données entre redémarrages
- Nécessitent des volumes avec haute performance

✅ **Systèmes de fichiers** (NextCloud, WordPress)
- Stockent les fichiers uploadés par les utilisateurs
- Peuvent partager le stockage entre plusieurs instances

✅ **Systèmes de logs et monitoring** (Elasticsearch, Prometheus)
- Conservent l'historique des métriques et logs
- Besoin de volumes avec beaucoup de capacité

✅ **Files d'attente et messaging** (RabbitMQ, Kafka)
- Persiste les messages pour la fiabilité
- Nécessite un stockage rapide et fiable

### Applications sans stockage persistant

❌ **APIs REST stateless**
- Aucune donnée à conserver
- Chaque requête est indépendante

❌ **Serveurs web statiques**
- Le contenu est dans l'image Docker
- Pas de modification de données

❌ **Workers de traitement pur**
- Lisent depuis une file, écrivent dans une autre
- Pas de stockage local nécessaire

## MicroK8s et le stockage : La simplicité avant tout

L'un des grands avantages de MicroK8s pour l'apprentissage est sa simplicité de configuration du stockage :

**Sans MicroK8s** (Kubernetes standard) :
```bash
# Configuration manuelle complexe :
1. Installer un provisioner de stockage
2. Configurer les StorageClasses
3. Créer manuellement des PersistentVolumes
4. Gérer les permissions et les chemins
5. Déboguer les problèmes de montage
```

**Avec MicroK8s** :
```bash
# Un simple addon :
microk8s enable hostpath-storage
# Et c'est tout ! Le provisionnement dynamique fonctionne.
```

MicroK8s configure automatiquement :
- ✅ Un provisioner de stockage fonctionnel
- ✅ Une StorageClass par défaut
- ✅ Le provisionnement dynamique des PV
- ✅ Les permissions et chemins appropriés

Cela vous permet de vous concentrer sur l'apprentissage des concepts plutôt que sur la configuration technique.

## Prérequis pour cette section

Avant de commencer, assurez-vous d'avoir :

### Connaissances requises
- ✅ Compréhension de base de Kubernetes (Pods, Deployments, Services)
- ✅ Familiarité avec la ligne de commande Linux
- ✅ Connaissance de base des fichiers YAML
- ✅ Compréhension du concept de conteneurs Docker

### Environnement technique
- ✅ MicroK8s installé et fonctionnel
- ✅ Au moins 20 Go d'espace disque disponible pour les volumes
- ✅ Addon hostpath-storage activé (nous le ferons ensemble)

### Vérification rapide

```bash
# Vérifier que MicroK8s fonctionne
microk8s status

# Vérifier l'espace disque disponible
df -h

# Le résultat devrait montrer au moins 20G disponibles
```

## Comment aborder cette section

Cette section est dense mais structurée de manière progressive. Voici nos recommandations :

### 1. Suivez l'ordre des sections

Les sections sont organisées du simple au complexe. Ne sautez pas d'étapes :
- Commencez par comprendre les concepts (6.1)
- Pratiquez avec les volumes simples (6.2)
- Passez aux PV et PVC (6.3, 6.4)
- Maîtrisez les StorageClasses (6.5)
- Approfondissez avec la gestion (6.6)
- Explorez les StatefulSets (6.7)
- Consolidez avec les exemples pratiques (6.8)

### 2. Prenez des notes

Le stockage comporte beaucoup de concepts interconnectés :
- Notez les définitions importantes
- Dessinez des schémas si ça vous aide
- Gardez une trace des commandes utiles

### 3. Expérimentez

La meilleure façon d'apprendre est de pratiquer :
- Testez chaque exemple fourni
- Modifiez les paramètres pour voir ce qui se passe
- Cassez les choses volontairement pour comprendre les erreurs
- MicroK8s est parfait pour l'expérimentation sans risque

### 4. Ne vous découragez pas

Le stockage est l'un des sujets les plus complexes de Kubernetes :
- C'est normal de ne pas tout comprendre immédiatement
- Revenez sur les sections précédentes si nécessaire
- Les exemples pratiques (6.8) clarifieront beaucoup de concepts

## Objectifs d'apprentissage

À la fin de cette section, vous serez capable de :

**Compétences théoriques** :
- ✅ Expliquer pourquoi le stockage persistant est nécessaire
- ✅ Décrire l'architecture du stockage dans Kubernetes
- ✅ Différencier volumes, PV, PVC et StorageClass
- ✅ Comprendre le provisionnement statique vs dynamique
- ✅ Connaître les différents types de volumes

**Compétences pratiques** :
- ✅ Créer et utiliser des volumes dans des pods
- ✅ Configurer des PersistentVolumes et PersistentVolumeClaims
- ✅ Utiliser les StorageClasses pour le provisionnement dynamique
- ✅ Gérer le cycle de vie du stockage (création, sauvegarde, restauration)
- ✅ Déployer des StatefulSets pour applications stateful
- ✅ Déployer des bases de données (MySQL, PostgreSQL, MongoDB, etc.)

**Compétences opérationnelles** :
- ✅ Surveiller l'utilisation du stockage
- ✅ Effectuer des sauvegardes et restaurations
- ✅ Diagnostiquer et résoudre les problèmes de stockage
- ✅ Optimiser les performances du stockage
- ✅ Appliquer les bonnes pratiques de sécurité

## Progression recommandée

Voici une estimation du temps nécessaire pour chaque section :

| Section | Durée estimée | Niveau |
|---------|---------------|--------|
| 6.1 - Concepts | 30-45 min | Débutant |
| 6.2 - Volumes | 45-60 min | Débutant |
| 6.3 - PersistentVolumes | 60-90 min | Débutant |
| 6.4 - PersistentVolumeClaims | 60-90 min | Débutant |
| 6.5 - StorageClasses | 60-90 min | Intermédiaire |
| 6.6 - Gestion | 90-120 min | Intermédiaire |
| 6.7 - StatefulSets | 90-120 min | Avancé |
| 6.8 - Exemples pratiques | 120-180 min | Avancé |

**Durée totale estimée** : 8 à 13 heures réparties sur plusieurs sessions

**Conseil** : Prévoyez des pauses régulières et ne forcez pas si vous êtes fatigué. Il vaut mieux comprendre profondément que survoler rapidement.

## Ressources complémentaires

Pour approfondir vos connaissances :

**Documentation officielle** :
- [Kubernetes Storage Documentation](https://kubernetes.io/docs/concepts/storage/)
- [MicroK8s Storage Guide](https://microk8s.io/docs/addon-hostpath-storage)

**Concepts clés à rechercher** :
- Container Storage Interface (CSI)
- Volume Snapshots
- Storage Capacity Tracking
- Volume Health Monitoring

## Mot de conclusion de l'introduction

Le stockage persistant est un pilier fondamental de Kubernetes. C'est ce qui permet de passer d'un simple système d'orchestration de conteneurs à une plateforme complète capable d'héberger n'importe quelle application, même les plus exigeantes comme les bases de données.

Cette section peut sembler intimidante au premier abord avec tous ses concepts (Volumes, PV, PVC, StorageClass, StatefulSets...), mais ne vous inquiétez pas : nous avons structuré le contenu de manière progressive et accessible. Chaque concept est expliqué clairement avec de nombreux exemples.

L'objectif n'est pas de vous transformer en expert du stockage en quelques heures, mais de vous donner les bases solides nécessaires pour :
- Comprendre comment fonctionne le stockage dans Kubernetes
- Être capable de déployer des applications avec stockage persistant
- Savoir où chercher quand vous rencontrez un problème
- Avoir les fondations pour approfondir par vous-même

**Conseil final** : Gardez MicroK8s ouvert pendant votre lecture et n'hésitez pas à tester les commandes au fur et à mesure. La pratique est le meilleur moyen d'ancrer les connaissances.

Maintenant, plongeons dans les concepts du stockage Kubernetes !

---

**Prêt à commencer ?** Direction la section 6.1 pour découvrir les concepts fondamentaux du stockage dans Kubernetes.

⏭️ [Concepts de stockage dans Kubernetes](/06-stockage-persistant/01-concepts-de-stockage-dans-kubernetes.md)
