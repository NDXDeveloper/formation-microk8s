🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 22.1 Stratégies de sauvegarde

## Introduction

La sauvegarde (ou **backup**) de votre cluster Kubernetes est une étape cruciale souvent négligée, mais absolument essentielle. Imaginez perdre toutes vos configurations, vos données d'applications, ou même l'accès à votre cluster après une mise à jour qui tourne mal ou une défaillance matérielle. C'est exactement ce qu'une bonne stratégie de sauvegarde permet d'éviter.

Dans ce chapitre, nous allons explorer les différentes approches pour protéger votre environnement MicroK8s, adaptées à un contexte de laboratoire personnel tout en respectant les bonnes pratiques professionnelles.

## Pourquoi sauvegarder un cluster Kubernetes ?

Avant de plonger dans les stratégies, comprenons d'abord **ce qui peut mal tourner** et pourquoi les sauvegardes sont importantes :

### Les risques courants

1. **Erreur humaine** : Suppression accidentelle d'une ressource importante avec `kubectl delete`
2. **Mise à jour problématique** : Une mise à jour de MicroK8s ou d'une application qui casse le cluster
3. **Défaillance matérielle** : Disque dur défaillant, corruption de données
4. **Corruption de l'etcd** : La base de données centrale de Kubernetes peut se corrompre
5. **Expérimentation** : Vous testez quelque chose de nouveau et voulez pouvoir revenir en arrière
6. **Besoin de migration** : Déplacer votre cluster vers une autre machine

### Ce qu'il faut protéger

Dans un cluster Kubernetes, il y a plusieurs types de données à considérer :

1. **Configuration du cluster** : Toutes les ressources Kubernetes (Pods, Services, Deployments, etc.)
2. **État du cluster** : Les données stockées dans etcd (ou Dqlite pour MicroK8s)
3. **Données applicatives** : Les volumes persistants contenant les données de vos applications
4. **Configurations système** : Les paramètres de MicroK8s lui-même
5. **Secrets et certificats** : Les informations sensibles et les certificats SSL/TLS

## Les différentes stratégies de sauvegarde

Il n'existe pas une seule "bonne" stratégie de sauvegarde universelle. Le choix dépend de vos besoins, de vos contraintes et de votre tolérance au risque. Voici les principales approches.

### 1. Stratégie déclarative avec Git (GitOps)

**Principe** : Stocker tous vos manifestes YAML dans un dépôt Git et considérer Git comme votre source de vérité.

**Comment ça fonctionne** :
- Tous vos fichiers de configuration Kubernetes sont versionnés dans Git
- Vous ne modifiez jamais directement le cluster avec `kubectl edit`
- Toute modification passe par un commit Git puis un `kubectl apply`
- En cas de problème, vous réinstallez MicroK8s et réappliquez vos manifestes

**Avantages** :
- Simple à mettre en place
- Historique complet des modifications
- Permet la collaboration
- Gratuit (avec GitHub, GitLab, etc.)
- Facilite l'Infrastructure as Code

**Limites** :
- Ne sauvegarde pas l'état interne d'etcd/Dqlite
- Ne protège pas les données dans les volumes persistants
- Ne capture pas les ressources créées manuellement ou par d'autres outils

**Adapté pour** : Environnements de développement, déploiements stateless, projets personnels simples.

### 2. Stratégie de sauvegarde complète du cluster (Snapshot etcd/Dqlite)

**Principe** : Sauvegarder régulièrement la base de données centrale qui contient tout l'état du cluster.

**Comment ça fonctionne** :
- MicroK8s utilise Dqlite (une variante légère d'etcd)
- Vous créez des snapshots de cette base de données
- En cas de problème, vous restaurez le snapshot

**Avantages** :
- Capture l'état complet du cluster à un instant T
- Récupération rapide possible
- Inclut toutes les ressources, même celles créées dynamiquement

**Limites** :
- Les snapshots peuvent être volumineux
- Nécessite un arrêt du cluster pour une restauration complète
- La restauration peut être complexe si la version de MicroK8s a changé

**Adapté pour** : Environnements de production, clusters avec beaucoup de ressources dynamiques.

### 3. Stratégie de sauvegarde des volumes persistants

**Principe** : Sauvegarder séparément les données contenues dans vos PersistentVolumes.

**Comment ça fonctionne** :
- Identifier les volumes contenant des données critiques (bases de données, fichiers utilisateurs, etc.)
- Mettre en place des sauvegardes régulières de ces volumes
- Utiliser des outils comme `rsync`, `tar`, ou des snapshots de système de fichiers

**Avantages** :
- Protège les données applicatives les plus importantes
- Peut être combiné avec d'autres stratégies
- Sauvegarde sélective possible (ne sauvegarder que ce qui est important)

**Limites** :
- Ne sauvegarde pas la configuration du cluster
- Nécessite une planification manuelle pour chaque volume
- Peut nécessiter l'arrêt de l'application pour garantir la cohérence

**Adapté pour** : Applications avec état (bases de données, systèmes de fichiers, etc.).

### 4. Stratégie hybride (Recommandée)

**Principe** : Combiner plusieurs approches pour une protection optimale.

**Exemple de stratégie hybride** :
1. **Configuration dans Git** : Tous les manifestes YAML versionnés
2. **Snapshots réguliers** : Sauvegarde de Dqlite une fois par semaine
3. **Sauvegarde des volumes** : Backup quotidien des données critiques (bases de données)
4. **Documentation** : Notes sur les configurations non-standard et les addons activés

**Avantages** :
- Protection maximale
- Flexibilité de restauration (restauration sélective ou complète)
- Adaptable aux différents besoins

**Limites** :
- Plus complexe à mettre en place
- Nécessite plus d'espace de stockage
- Demande plus de maintenance

**Adapté pour** : Environnements de production, projets critiques, apprentissage avancé.

## La règle 3-2-1 appliquée à Kubernetes

La règle **3-2-1** est un principe fondamental en matière de sauvegarde :

- **3 copies** de vos données : L'original + 2 sauvegardes
- **2 supports différents** : Par exemple, disque dur + NAS, ou disque local + cloud
- **1 copie hors site** : Idéalement dans un autre lieu physique ou dans le cloud

### Application pratique pour votre lab MicroK8s :

1. **Copie originale** : Votre cluster MicroK8s en fonctionnement
2. **Première sauvegarde** : Sur un disque dur externe ou un autre disque de la même machine
3. **Deuxième sauvegarde** : Dans le cloud (Google Drive, Dropbox, S3) ou sur un NAS

**Exemple concret** :
```
- Original : Cluster MicroK8s sur /var/snap/microk8s
- Backup 1 : Snapshots quotidiens sur /mnt/backup-local
- Backup 2 : Synchronisation hebdomadaire vers un stockage cloud
```

## Fréquence de sauvegarde : trouver le bon équilibre

La fréquence de vos sauvegardes dépend de votre **RTO** (Recovery Time Objective) et **RPO** (Recovery Point Objective).

### Définitions simples :

- **RPO** (Recovery Point Objective) : Quelle quantité de données pouvez-vous vous permettre de perdre ?
  - Si RPO = 24h, vous acceptez de perdre jusqu'à 24h de travail

- **RTO** (Recovery Time Objective) : Combien de temps pouvez-vous tolérer d'être hors service ?
  - Si RTO = 2h, vous devez pouvoir restaurer en moins de 2h

### Recommandations par type d'environnement :

#### Lab personnel / Apprentissage
- **Snapshots Dqlite** : Hebdomadaire ou avant chaque expérimentation majeure
- **Manifestes Git** : À chaque modification
- **Volumes persistants** : Hebdomadaire pour les données importantes
- **RPO acceptable** : 1 semaine
- **RTO acceptable** : 1 journée

#### Environnement de développement actif
- **Snapshots Dqlite** : Quotidien
- **Manifestes Git** : À chaque commit (automatique)
- **Volumes persistants** : Quotidien
- **RPO acceptable** : 24 heures
- **RTO acceptable** : 4 heures

#### Services personnels en production
- **Snapshots Dqlite** : Quotidien avec rétention d'une semaine
- **Manifestes Git** : En continu (automatique)
- **Volumes persistants** : Plusieurs fois par jour
- **RPO acceptable** : 4-12 heures
- **RTO acceptable** : 2 heures

## Rétention des sauvegardes

Ne gardez pas toutes les sauvegardes indéfiniment, mais adoptez une stratégie de **rétention intelligente** :

### Stratégie GFS (Grandfather-Father-Son)

C'est une approche classique et efficace :

- **Quotidien (Son)** : Gardez les 7 derniers jours
- **Hebdomadaire (Father)** : Gardez les 4 dernières semaines
- **Mensuel (Grandfather)** : Gardez les 6 ou 12 derniers mois

**Exemple pratique** :
```
Lundi 1er Janvier : Backup quotidien (garder 7 jours)
Dimanche 7 Janvier : Backup hebdomadaire (garder 4 semaines)
31 Janvier : Backup mensuel (garder 12 mois)
```

### Stratégie simplifiée pour un lab

Si la stratégie GFS est trop complexe pour votre besoin, voici une alternative simple :

- **7 sauvegardes** rotatives (une par jour de la semaine)
- **1 sauvegarde mensuelle** archivée
- **Suppression automatique** des sauvegardes de plus de 90 jours

## Où stocker vos sauvegardes ?

### Options locales

1. **Disque dur externe**
   - Avantages : Rapide, contrôle total, pas de coût récurrent
   - Inconvénients : Risque de défaillance, pas de protection contre le vol/incendie

2. **NAS (Network Attached Storage)**
   - Avantages : Centralisé, souvent avec RAID, accessible réseau
   - Inconvénients : Coût initial, vulnérable aux mêmes risques que le serveur (incendie, etc.)

3. **Autre machine sur le même réseau**
   - Avantages : Utilise du matériel existant
   - Inconvénients : Dépendance réseau, pas vraiment "hors site"

### Options cloud

1. **Stockage objet (S3, Backblaze B2, Wasabi)**
   - Avantages : Très fiable, hors site, économique pour grandes quantités
   - Inconvénients : Nécessite une connexion internet, coût variable

2. **Services cloud classiques (Google Drive, Dropbox, OneDrive)**
   - Avantages : Simple à utiliser, intégration facile
   - Inconvénients : Limites de stockage gratuit, moins adapté aux gros volumes

3. **Solutions dédiées Kubernetes (Velero avec S3)**
   - Avantages : Conçues pour Kubernetes, sauvegarde complète
   - Inconvénients : Configuration plus complexe, coût de stockage

### Recommandation pour débuter

Pour un lab personnel, une approche simple et efficace :

1. **Sauvegarde locale** : Disque externe ou partition dédiée
2. **Sauvegarde cloud** : Compte cloud gratuit (Google Drive, Dropbox) pour les configurations
3. **Git** : GitHub/GitLab gratuit pour tous les manifestes

## Tests de restauration

**La règle d'or** : Une sauvegarde non testée n'est pas une sauvegarde fiable !

### Pourquoi tester ?

- Vérifier que la sauvegarde est complète
- S'assurer que le processus de restauration fonctionne
- Se familiariser avec la procédure avant une vraie crise
- Identifier les problèmes pendant qu'il est encore temps de les corriger

### Recommandations de tests

- **Test complet** : Au moins une fois par trimestre
- **Test partiel** : Mensuel (restauration d'une seule application)
- **Vérification** : Hebdomadaire (vérifier que les sauvegardes s'exécutent bien)

### Que tester ?

1. Restaurer une application simple depuis une sauvegarde
2. Vérifier que les données sont intactes
3. Tester le temps nécessaire à la restauration
4. Documenter les étapes et les problèmes rencontrés

## Documentation de votre stratégie

Créez un document simple qui décrit votre stratégie de sauvegarde. Ce document devrait inclure :

### Éléments essentiels

1. **Que sauvegarde-t-on ?**
   - Liste des composants sauvegardés
   - Emplacement des sauvegardes

2. **Quand ?**
   - Fréquence de chaque type de sauvegarde
   - Horaires programmés

3. **Où ?**
   - Emplacements de stockage
   - Méthode d'accès

4. **Comment restaurer ?**
   - Procédure étape par étape pour chaque type de restauration
   - Commandes exactes à utiliser

5. **Contact et ressources**
   - Liens vers documentation
   - Notes spécifiques à votre environnement

### Exemple de documentation simple

```markdown
# Stratégie de sauvegarde - MicroK8s Lab Personnel

## Sauvegardes configurées

1. Manifestes YAML : Git (GitHub)
   - Fréquence : À chaque modification
   - Emplacement : https://github.com/mon-user/microk8s-configs

2. Snapshot Dqlite : Hebdomadaire
   - Fréquence : Tous les dimanches à 2h00
   - Emplacement : /mnt/backup/dqlite/
   - Rétention : 4 semaines

3. Volumes PostgreSQL : Quotidien
   - Fréquence : Tous les jours à 3h00
   - Emplacement : /mnt/backup/postgres/
   - Rétention : 7 jours

## Procédure de restauration rapide

### Restauration depuis Git
1. Réinstaller MicroK8s
2. git clone https://github.com/mon-user/microk8s-configs
3. kubectl apply -f configs/

### Restauration snapshot Dqlite
1. sudo snap stop microk8s
2. Copier le snapshot vers /var/snap/microk8s/common/
3. sudo snap start microk8s

## Dernier test de restauration : 15 janvier 2025 ✓
```

## Automatisation des sauvegardes

Pour que votre stratégie soit efficace sur le long terme, **l'automatisation est essentielle**. Vous ne voulez pas avoir à penser à faire une sauvegarde manuelle chaque jour.

### Outils d'automatisation simples

1. **Cron** : Le planificateur de tâches Linux classique
   - Parfait pour des scripts simples
   - Intégré à tous les systèmes Linux

2. **Systemd timers** : Alternative moderne à cron
   - Plus de contrôle et de flexibilité
   - Meilleure intégration système

3. **Kubernetes CronJobs** : Pour des tâches dans le cluster
   - Natif à Kubernetes
   - Idéal pour sauvegarder des données applicatives

### Principe général

Quelle que soit la méthode choisie, le principe reste le même :

1. Créer un script de sauvegarde
2. Tester le script manuellement
3. Programmer son exécution automatique
4. Vérifier régulièrement que ça fonctionne
5. Recevoir des notifications en cas d'échec

## Surveillance et alertes

Une bonne stratégie de sauvegarde inclut un système de **surveillance** pour s'assurer que tout fonctionne correctement.

### Points de surveillance

1. **Succès/échec de la sauvegarde** : L'opération s'est-elle terminée sans erreur ?
2. **Taille de la sauvegarde** : Est-elle cohérente avec les précédentes ?
3. **Durée d'exécution** : Le processus prend-il plus de temps que d'habitude ?
4. **Espace disque disponible** : Y a-t-il assez de place pour la prochaine sauvegarde ?
5. **Âge de la dernière sauvegarde** : La dernière sauvegarde date-t-elle de trop longtemps ?

### Notifications

Configurez des alertes pour être prévenu en cas de problème :

- **Email** : Pour les erreurs importantes
- **Slack/Discord** : Pour un suivi quotidien
- **Logs système** : Pour le diagnostic détaillé
- **Dashboard** : Pour une vue d'ensemble (via Grafana si disponible)

## Sécurité des sauvegardes

Vos sauvegardes contiennent des informations sensibles et doivent être protégées.

### Bonnes pratiques de sécurité

1. **Chiffrement** :
   - Chiffrez les sauvegardes, surtout si elles contiennent des secrets
   - Utilisez des outils comme `gpg` ou `age` pour le chiffrement

2. **Contrôle d'accès** :
   - Limitez qui peut accéder aux sauvegardes
   - Utilisez des permissions fichiers restrictives (chmod 600)

3. **Stockage sécurisé des clés** :
   - Ne stockez jamais les clés de chiffrement avec les sauvegardes
   - Utilisez un gestionnaire de mots de passe

4. **Validation d'intégrité** :
   - Créez des checksums (SHA256) de vos sauvegardes
   - Vérifiez l'intégrité régulièrement

5. **Protection contre les ransomwares** :
   - Sauvegardez sur des supports qui peuvent être déconnectés
   - Utilisez le versioning pour pouvoir revenir en arrière
   - Activez l'immuabilité si votre stockage le supporte

## Cas particuliers et considérations

### Sauvegarder un cluster multi-node

Pour un cluster avec plusieurs nœuds :
- Sauvegardez la configuration de tous les nœuds
- Documentez la topologie du cluster
- Testez la restauration d'un nœud individuel

### Sauvegarder avec des applications stateful

Pour des bases de données ou applications avec état :
- Utilisez les outils de backup natifs de l'application (pg_dump, mysqldump)
- Assurez la cohérence des données (arrêtez l'application si nécessaire)
- Testez la restauration des données, pas seulement des volumes

### Considérations de conformité

Même pour un lab personnel :
- Attention aux données personnelles (RGPD)
- Ne stockez pas de vraies données de production
- Anonymisez les données de test si nécessaire

## Évolution de votre stratégie

Votre stratégie de sauvegarde doit évoluer avec votre cluster :

### Signes qu'il faut améliorer votre stratégie

- Vous hébergez maintenant des services "critiques" (serveur mail personnel, etc.)
- Vous avez perdu des données récemment
- Le temps de restauration est trop long
- Vous avez ajouté beaucoup de nouvelles applications
- Votre cluster a grandi (multi-node)

### Étapes d'amélioration progressive

1. **Débutant** : Manifestes dans Git + backup manuel occasionnel
2. **Intermédiaire** : + Snapshots automatisés hebdomadaires
3. **Avancé** : + Sauvegarde des volumes + tests réguliers
4. **Expert** : + Solution complète type Velero + monitoring + disaster recovery

## Conclusion

Une stratégie de sauvegarde efficace pour MicroK8s ne doit pas être complexe, mais elle doit être :

- **Planifiée** : Savoir quoi sauvegarder, quand et où
- **Automatisée** : Ne pas dépendre de votre mémoire
- **Testée** : Vérifier régulièrement qu'elle fonctionne
- **Documentée** : Pour pouvoir restaurer même en situation de stress
- **Évolutive** : S'adapter à l'évolution de vos besoins

**Retenez ceci** : Il vaut mieux une stratégie simple mais appliquée régulièrement qu'une stratégie complexe jamais mise en œuvre.

Dans les prochaines sections, nous verrons comment mettre en pratique ces stratégies avec des outils concrets comme Velero, et comment automatiser l'ensemble du processus.

---

**Points clés à retenir** :

✅ Plusieurs types de données à protéger : configuration, état, volumes, secrets
✅ Stratégie hybride recommandée : Git + snapshots + sauvegarde volumes
✅ Règle 3-2-1 : 3 copies, 2 supports, 1 hors site
✅ Tester vos sauvegardes régulièrement
✅ Automatiser pour garantir la régularité
✅ Documenter la procédure de restauration
✅ Chiffrer et sécuriser vos sauvegardes

⏭️ [Sauvegarde avec Velero](/22-sauvegarde-et-restauration/02-sauvegarde-avec-velero.md)
