🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 22. Sauvegarde et Restauration

## Introduction au chapitre

Bienvenue dans l'un des chapitres les plus importants de cette formation : la **sauvegarde et la restauration**. Si vous ne deviez retenir qu'une seule chose de ce cours, ce serait celle-ci : **vos données sont précieuses, et sans sauvegarde, vous jouez à la roulette russe avec votre infrastructure**.

### Pourquoi ce chapitre est crucial

Imaginez cette situation (malheureusement trop courante) :

```
Vendredi soir, 23h45 : Vous déployez une mise à jour sur votre cluster MicroK8s
Samedi matin, 2h00 : Réveil brutal - le serveur ne répond plus
Samedi matin, 2h15 : Panique - impossible de redémarrer le cluster
Samedi matin, 2h30 : Question fatidique : "Où est ma dernière sauvegarde ?"
Samedi matin, 2h31 : Réponse terrifiante : "Euh... quelle sauvegarde ?"

Résultat : Week-end ruiné à essayer de reconstruire tout à partir de zéro.
Perte de données : 3 mois de travail.
Leçon apprise : À un prix très élevé.
```

Maintenant, imaginez le même scénario **avec une stratégie de sauvegarde solide** :

```
Vendredi soir, 23h45 : Vous déployez une mise à jour
Samedi matin, 2h00 : Le serveur ne répond plus
Samedi matin, 2h05 : Vous ouvrez votre documentation de restauration
Samedi matin, 2h10 : Vous lancez la restauration depuis le backup de la veille
Samedi matin, 3h30 : Le cluster est restauré, tout fonctionne
Samedi matin, 3h45 : Vous retournez vous coucher

Résultat : Incident géré calmement.
Perte de données : Quelques heures au maximum.
Leçon apprise : Votre stratégie de sauvegarde vaut son pesant d'or.
```

La différence entre ces deux scénarios ? **Ce chapitre.**

### Les trois vérités sur les sauvegardes

Avant d'entrer dans les détails, comprenons trois vérités fondamentales :

#### Vérité #1 : Ce n'est pas "si" mais "quand"

**La question n'est pas de savoir SI vous aurez un problème, mais QUAND.**

Les désastres arrivent à tout le monde :
- Erreur humaine : Vous supprimez accidentellement un namespace important
- Défaillance matérielle : Un disque dur lâche sans prévenir
- Bug logiciel : Une mise à jour corrompt votre cluster
- Cyberattaque : Un ransomware chiffre vos données
- Catastrophe naturelle : Inondation, incendie, panne électrique majeure

Même les géants de la tech perdent des données. La différence ? Ils ont des sauvegardes.

#### Vérité #2 : Une sauvegarde non testée n'est pas une sauvegarde

Avoir des backups ne suffit pas. Vous devez **prouver** que vous pouvez restaurer depuis ces backups.

Histoire vraie et trop commune :
```
Entreprise : "Nous avons des backups quotidiens depuis 2 ans !"
Incident : Serveur HS, restauration nécessaire
Tentative : Échec - les backups sont corrompus depuis 6 mois
Réalisation : Personne n'avait jamais testé la restauration
```

**La règle d'or** : Si vous n'avez jamais restauré depuis un backup, vous n'avez pas de backup, juste l'illusion d'en avoir.

#### Vérité #3 : Les sauvegardes sont une assurance, pas un coût

Beaucoup voient les sauvegardes comme une dépense :
- "Ça prend de l'espace disque"
- "Ça ralentit le système"
- "Je n'ai pas le temps de configurer ça"
- "Mon cluster est simple, je peux tout reconfigurer"

**Changez de perspective** : Les sauvegardes sont une **assurance**.

Vous payez une petite prime (espace disque, temps de configuration) pour vous protéger contre une perte catastrophique (des semaines de travail, des données irremplaçables, votre réputation).

Analogie simple : Vous ne conduisez pas sans assurance automobile. Pourquoi faire tourner un cluster sans "assurance données" ?

### Ce que vous allez apprendre

Ce chapitre complet couvre **tous les aspects** de la sauvegarde et restauration pour MicroK8s, du niveau débutant à expert.

#### Section 22.1 : Stratégies de sauvegarde

Avant de sauvegarder, il faut **comprendre ce qu'on sauvegarde et pourquoi**.

**Vous apprendrez** :
- Quels sont les différents types de sauvegardes (complète, incrémentale, différentielle)
- Comment choisir la bonne stratégie pour votre cas d'usage
- La règle 3-2-1 et pourquoi elle est essentielle
- Définir vos objectifs RPO (Recovery Point Objective) et RTO (Recovery Time Objective)
- Quelle fréquence de sauvegarde choisir selon vos besoins

**Analogie** : Avant de construire une maison, vous faites des plans. Cette section, ce sont vos plans pour protéger vos données.

#### Section 22.2 : Sauvegarde avec Velero

**Velero** est l'outil de référence pour sauvegarder et restaurer des clusters Kubernetes. C'est comme avoir un "bouton magique" pour capturer et restaurer tout votre cluster.

**Vous apprendrez** :
- Qu'est-ce que Velero et pourquoi c'est l'outil idéal
- Comment installer Velero sur MicroK8s
- Configurer MinIO comme stockage de backups (solution locale gratuite)
- Créer votre première sauvegarde
- Restaurer depuis une sauvegarde
- Planifier des sauvegardes automatiques

**À la fin de cette section**, vous pourrez sauvegarder et restaurer votre cluster en quelques commandes.

#### Section 22.3 : Configuration Velero

Passer du backup de base à une configuration professionnelle et optimisée.

**Vous apprendrez** :
- Configurer finement les BackupStorageLocations
- Gérer les credentials de manière sécurisée
- Configurer Restic/Kopia pour la sauvegarde des volumes
- Mettre en place des exclusions intelligentes
- Optimiser les performances
- Configurer plusieurs destinations de backup
- Sécuriser vos sauvegardes

**Cette section transforme** un système fonctionnel en système professionnel.

#### Section 22.4 : Snapshots de volumes

Les **snapshots** sont comme des "photos instantanées" de vos disques. Ultra-rapides, mais nécessitent un stockage compatible.

**Vous apprendrez** :
- La différence entre snapshots et backups par fichier
- Comment fonctionnent les snapshots (Copy-on-Write)
- Installer le support des snapshots sur MicroK8s (OpenEBS, Longhorn)
- Créer et gérer des snapshots de volumes
- Restaurer depuis un snapshot
- Utiliser les snapshots pour cloner rapidement des environnements
- Quand utiliser snapshots vs Restic/Kopia

**Important pour MicroK8s** : Par défaut, le stockage hostpath ne supporte pas les snapshots natifs. Cette section explique les alternatives et quand elles valent la peine.

#### Section 22.5 : Export/Import de configurations

Le **GitOps avant la lettre** : sauvegarder vos configurations dans des fichiers YAML versionnés.

**Vous apprendrez** :
- Exporter toutes vos ressources Kubernetes en YAML
- Nettoyer les métadonnées pour réimport
- Organiser vos exports (par namespace, par application, par type)
- Importer des configurations dans un nouveau cluster
- Migration entre clusters
- Versionner vos configurations dans Git
- Utiliser Kustomize et Helm pour la réutilisabilité
- Introduction au GitOps (ArgoCD, Flux)

**Cette approche** complète parfaitement Velero : elle capture la configuration, Velero capture les données.

#### Section 22.6 : Tests de restauration

La section la plus importante après avoir configuré vos backups.

**Vous apprendrez** :
- Pourquoi tester est absolument critique
- Les différents types de tests (validation, partiel, complet)
- Créer des environnements de test
- Méthodologie de test en 5 phases
- Planifier des tests réguliers (calendrier mensuel/trimestriel)
- Mesurer vos métriques (RTO réel, taux de réussite)
- Documenter vos tests
- Automatiser les tests avec des scripts

**Le principe fondamental** : Une sauvegarde non testée = pas de sauvegarde.

#### Section 22.7 : Plan de reprise d'activité (DRP)

Le **DRP** (Disaster Recovery Plan) est votre manuel de survie quand tout va mal. C'est le document que vous ouvrez à 3h du matin quand le serveur est en panne.

**Vous apprendrez** :
- Qu'est-ce qu'un DRP et pourquoi c'est essentiel
- Définir vos RPO et RTO par application
- Structure complète d'un DRP (9 sections)
- Écrire des procédures détaillées étape par étape
- Gérer différents scénarios de désastre (incident mineur à catastrophe majeure)
- Organiser les contacts et l'escalade
- Tester régulièrement votre DRP
- Amélioration continue avec post-mortems

**À la fin**, vous aurez un document complet qui vous guide précisément pour chaque type d'incident.

#### Section 22.8 : Automatisation des sauvegardes

Passer d'un système qui dépend de votre mémoire à un système qui fonctionne tout seul.

**Vous apprendrez** :
- Les niveaux d'automatisation (de 20% à 95%)
- Automatiser avec Cron (planificateur Linux classique)
- Automatiser avec Systemd Timers (approche moderne)
- Utiliser les Velero Schedules (natif Kubernetes)
- Automatiser la vérification des backups
- Nettoyage automatique des anciens backups
- Notifications automatiques (email, Slack, Discord)
- Monitoring avec Prometheus et Grafana
- Orchestration complète du workflow
- Bonnes pratiques d'automatisation

**L'objectif** : Créer un système qui sauvegarde, vérifie, nettoie et alerte automatiquement pendant que vous dormez tranquille.

### Prérequis pour ce chapitre

Ce chapitre est accessible si vous avez :

**Connaissances techniques** :
- ✅ Bases de Kubernetes (Pods, Deployments, Services, PVCs)
- ✅ Utilisation de kubectl
- ✅ MicroK8s installé et fonctionnel
- ✅ Notions de ligne de commande Linux
- ✅ (Optionnel) Git pour le versionnement

**Environnement** :
- ✅ Cluster MicroK8s opérationnel
- ✅ Accès root ou sudo sur le serveur
- ✅ Espace disque disponible pour les backups (minimum 50 GB recommandé)
- ✅ (Optionnel) Compte cloud pour stockage distant (AWS S3, etc.)

**Pas de panique si vous débutez** : Ce chapitre est conçu pour être accessible. Chaque concept est expliqué avec des analogies et des exemples concrets.

### Philosophie de ce chapitre

Ce chapitre suit trois principes directeurs :

#### 1. Simplicité d'abord, complexité ensuite

Nous commençons toujours par la solution la plus simple qui fonctionne, puis nous ajoutons de la sophistication progressivement.

**Exemple** :
- Section 22.2 : Backup Velero de base (30 minutes à mettre en place)
- Section 22.3 : Configuration avancée (quelques heures)
- Section 22.8 : Orchestration complète (système professionnel)

Vous pouvez vous arrêter à n'importe quel niveau selon vos besoins.

#### 2. Pratique avant théorie

Chaque section contient des **exemples concrets** et des **commandes prêtes à utiliser**.

Vous n'aurez pas juste la théorie, vous aurez :
- Scripts complets et commentés
- Commandes exactes à exécuter
- Exemples de fichiers de configuration
- Procédures étape par étape

L'objectif : vous pouvez suivre le chapitre et avoir un système fonctionnel à la fin.

#### 3. Tests, tests, tests

Nous insistons **lourdement** sur l'importance des tests.

Chaque section inclut :
- Comment tester que ça fonctionne
- Comment vérifier que tout est OK
- Comment détecter les problèmes

**Pourquoi ?** Parce qu'un backup non testé ne vaut rien.

### Comment utiliser ce chapitre

#### Si vous partez de zéro

**Parcours recommandé** (2-3 jours) :

1. **Jour 1 - Fondations** :
   - Lire 22.1 (Stratégies) pour comprendre l'approche
   - Suivre 22.2 (Velero) pour installer et créer votre premier backup
   - Tester la restauration basique

2. **Jour 2 - Consolidation** :
   - Approfondir avec 22.3 (Configuration Velero)
   - Découvrir 22.5 (Export/Import) pour Git
   - Faire un premier test de restauration (22.6)

3. **Jour 3 - Automatisation** :
   - Mettre en place l'automatisation (22.8)
   - Créer un DRP minimal (22.7)
   - Valider que tout fonctionne

**Temps total estimé** : 15-20 heures pour un système complet et testé.

#### Si vous avez déjà des backups

**Parcours d'amélioration** :

1. Lisez 22.6 (Tests) et **testez vos backups existants** immédiatement
2. Évaluez votre stratégie actuelle avec 22.1
3. Améliorez la configuration avec 22.3
4. Automatisez avec 22.8
5. Créez/améliorez votre DRP avec 22.7

**Focus prioritaire** : Les tests. Si vous n'avez jamais testé une restauration, faites-le avant toute autre chose.

#### Si vous gérez un environnement critique

**Parcours professionnel** :

1. Lisez tout le chapitre pour avoir la vue d'ensemble
2. Définissez précisément vos RPO/RTO (22.1 et 22.7)
3. Implémentez une solution robuste (22.2, 22.3, 22.4)
4. Créez un DRP complet et détaillé (22.7)
5. Testez régulièrement et intensément (22.6)
6. Automatisez tout (22.8)
7. Monitoring et amélioration continue

**Temps total** : 1-2 semaines pour un système de niveau production.

### Structure des sections

Chaque section suit la même structure pour faciliter la navigation :

```
📘 Introduction
   ├─ Pourquoi cette section est importante
   ├─ Analogies et exemples concrets
   └─ Ce que vous allez apprendre

🔧 Concepts et théorie
   ├─ Explications accessibles
   ├─ Schémas et illustrations
   └─ Comparaisons et choix

💻 Pratique et implémentation
   ├─ Installation étape par étape
   ├─ Configuration commentée
   ├─ Scripts et exemples
   └─ Commandes exactes

✅ Tests et vérification
   ├─ Comment tester
   ├─ Que vérifier
   └─ Dépannage courant

📚 Bonnes pratiques
   ├─ Recommandations
   ├─ Erreurs à éviter
   └─ Optimisations

📋 Checklist récapitulative
   └─ Points clés à retenir
```

### Les outils que vous allez utiliser

Ce chapitre se concentre principalement sur quelques outils essentiels :

#### Velero
**L'outil principal de ce chapitre.**
- Développé par VMware
- Standard de facto pour Kubernetes
- Open source et mature
- Gratuit

#### MinIO
**Stockage objet pour vos backups.**
- Compatible S3
- Peut tourner localement
- Open source
- Alternative gratuite à AWS S3

#### Restic/Kopia
**Sauvegarde des volumes.**
- Intégré à Velero
- Fonctionne avec tout type de stockage
- Chiffrement natif
- Open source

#### Outils complémentaires
- **kubectl** : Interaction avec Kubernetes
- **yq/jq** : Manipulation YAML/JSON
- **Cron/Systemd** : Automatisation
- **Git** : Versionnement des configs
- **Prometheus/Grafana** : Monitoring

**Tous ces outils sont gratuits et open source.**

### À propos des exercices pratiques

**Important** : Ce chapitre ne contient volontairement pas d'exercices pratiques séparés.

**Pourquoi ?**

Parce que chaque section est **déjà un exercice pratique** en elle-même. Les exemples et commandes fournis sont conçus pour être exécutés directement sur votre système.

**Notre approche** :
1. Vous lisez la théorie
2. Vous suivez les exemples concrets
3. Vous testez sur votre cluster
4. Vous adaptez à vos besoins

C'est un apprentissage par la pratique directe plutôt que par des exercices théoriques.

### Avertissements et précautions

#### ⚠️ Testez sur un environnement de lab d'abord

Si vous gérez un cluster de production, **ne testez PAS directement en production**.

**Recommandation** :
1. Installez un second MicroK8s de test
2. Pratiquez toutes les opérations
3. Une fois à l'aise, appliquez en production
4. Toujours faire un backup avant de tester une restauration

#### ⚠️ Les sauvegardes contiennent des données sensibles

Vos backups contiennent :
- Secrets Kubernetes (mots de passe, clés API)
- Certificats SSL/TLS
- Données applicatives

**Sécurisez-les** :
- Chiffrez les backups
- Contrôlez les accès
- Ne les partagez pas publiquement
- Stockez-les dans un endroit sûr

#### ⚠️ L'espace disque

Les backups prennent de la place, surtout si vous avez beaucoup de volumes.

**Planifiez** :
- Minimum 50 GB d'espace pour les backups
- Plus si vous avez des volumes volumineux
- Surveillez l'utilisation
- Mettez en place une rotation

#### ⚠️ La restauration peut écraser des données

Une restauration **remplace** les ressources existantes.

**Soyez prudent** :
- Comprenez ce que vous faites
- Testez d'abord dans un namespace temporaire
- Ayez toujours un backup de l'état actuel
- Documentez vos actions

### Messages clés à retenir

Avant d'entrer dans les détails techniques, retenez ces principes fondamentaux :

1. **Les désastres arrivent à tout le monde** - Pas de "si" mais "quand"

2. **Une sauvegarde non testée n'est pas une sauvegarde** - Testez régulièrement

3. **La simplicité est une vertu** - Commencez simple, complexifiez si nécessaire

4. **L'automatisation est essentielle** - Vous oublierez, un script jamais

5. **Documentez tout** - Future vous remerciera présent vous

6. **3-2-1 est votre ami** - 3 copies, 2 supports, 1 hors site

7. **RPO et RTO guident vos choix** - Définissez-les clairement

8. **Le backup n'est que la moitié** - La restauration est aussi importante

### Motivation finale

Configurer des sauvegardes n'est pas le travail le plus excitant. Ce n'est pas aussi cool que de déployer une nouvelle application brillante ou de mettre en place un cluster Kubernetes complexe.

**Mais c'est l'un des investissements les plus importants que vous ferez.**

Pensez-y comme à une **assurance peace-of-mind** :
- Vous pouvez expérimenter sans peur
- Vous pouvez dormir tranquille
- Vous pouvez gérer les incidents calmement
- Vous protégez des mois de travail

Un jour, vous serez content d'avoir lu ce chapitre. Peut-être même très, très content.

### Prêt à commencer ?

Ce chapitre est long (8 sections), mais chaque section est autonome. Vous pouvez :
- Le lire d'un bout à l'autre
- Piocher les sections qui vous intéressent
- Y revenir comme référence

**Commençons par le commencement** : Comprendre les stratégies de sauvegarde dans la section 22.1.

Mais avant, une dernière chose...

### La règle d'or (à ne jamais oublier)

> **"Le meilleur moment pour configurer vos sauvegardes était hier. Le deuxième meilleur moment est maintenant."**

N'attendez pas le désastre pour réaliser que vous auriez dû avoir des backups.

**N'attendez pas. Commencez maintenant.**

---

*Vous êtes maintenant prêt à plonger dans le monde de la sauvegarde et restauration Kubernetes. Direction la section 22.1 pour découvrir les stratégies de sauvegarde !*

⏭️ [Stratégies de sauvegarde](/22-sauvegarde-et-restauration/01-strategies-de-sauvegarde.md)
