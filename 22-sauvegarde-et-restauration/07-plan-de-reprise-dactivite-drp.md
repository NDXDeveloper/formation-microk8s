🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 22.7 Plan de reprise d'activité (DRP)

## Introduction

Un **Plan de Reprise d'Activité** (DRP - Disaster Recovery Plan) est un document structuré qui décrit exactement quoi faire quand tout va mal. C'est votre plan B, votre bouée de sauvetage, votre manuel de survie pour votre infrastructure Kubernetes.

Pensez-y comme au **plan d'évacuation d'un bâtiment** : vous espérez ne jamais en avoir besoin, mais quand l'alarme incendie retentit, vous êtes content qu'il existe et que tout le monde sache quoi faire.

### Pourquoi un DRP est essentiel

Sans DRP, voici ce qui arrive lors d'un désastre :

**Scenario sans DRP** :
```
3h00 : Le serveur tombe en panne
3h15 : Panique. Où sont les sauvegardes ?
3h30 : On trouve des sauvegardes, mais lesquelles utiliser ?
4h00 : On essaie de restaurer. Échec. Pourquoi ?
5h00 : On appelle quelqu'un pour aider
6h00 : On découvre que les secrets ne sont pas sauvegardés
8h00 : On reconstruit à la main
12h00 : On découvre qu'il manque des données
Résultat : 9 heures de downtime, stress maximum, données perdues
```

**Scenario avec DRP** :
```
3h00 : Le serveur tombe en panne
3h05 : On ouvre le DRP, section "Panne serveur complète"
3h10 : On exécute le script de restauration documenté
3h30 : Le cluster est restauré, tests en cours
3h45 : Services remis en ligne
Résultat : 45 minutes de downtime, procédure calme et maîtrisée
```

### Ce qu'un DRP n'est PAS

❌ Un simple document technique ignoré dans un tiroir
❌ Une procédure écrite une fois et jamais mise à jour
❌ Une responsabilité d'une seule personne
❌ Une garantie que rien ne peut mal tourner

### Ce qu'un DRP EST

✅ Un guide pratique testé régulièrement
✅ Un document vivant mis à jour continuellement
✅ Une responsabilité partagée par toute l'équipe
✅ Un moyen de minimiser l'impact des désastres

## Concepts fondamentaux

Avant de créer votre DRP, vous devez comprendre quelques concepts clés.

### RPO (Recovery Point Objective)

**Définition simple** : Combien de données pouvez-vous vous permettre de perdre ?

**Analogie** : Imaginez que vous écrivez un roman. Si votre ordinateur plante, combien de chapitres êtes-vous prêt à réécrire ?
- Sauvegarde toutes les heures → RPO de 1 heure (vous réécrivez 1 heure de travail)
- Sauvegarde quotidienne → RPO de 24 heures (vous réécrivez une journée)

**Exemples pour MicroK8s** :
- **Blog personnel** : RPO = 24h (un article par jour, acceptable)
- **Application de notes** : RPO = 1h (données fréquentes, important)
- **Système de monitoring** : RPO = 5 minutes (métriques critiques)
- **Serveur de développement** : RPO = 1 semaine (peu critique)

**Comment définir votre RPO** :

```markdown
Question : Si je perds les X dernières heures de données, quel est l'impact ?

Impact minimal (pas grave) → RPO = 24h
Impact modéré (gênant) → RPO = 4h
Impact important (problématique) → RPO = 1h
Impact critique (inacceptable) → RPO = 15min
```

### RTO (Recovery Time Objective)

**Définition simple** : Combien de temps pouvez-vous rester hors service ?

**Analogie** : Si votre voiture tombe en panne, combien de temps pouvez-vous attendre avant d'en avoir absolument besoin ?
- Voiture de loisir → RTO = plusieurs jours
- Voiture pour aller au travail → RTO = quelques heures
- Ambulance → RTO = quelques minutes

**Exemples pour MicroK8s** :
- **Lab d'apprentissage** : RTO = 1 semaine (pas urgent)
- **Blog personnel** : RTO = 48 heures (gênant mais gérable)
- **Service interne d'équipe** : RTO = 4 heures (impact sur productivité)
- **Application critique** : RTO = 30 minutes (impact business important)

**Comment définir votre RTO** :

```markdown
Question : Combien de temps puis-je rester sans ce service ?

Plusieurs jours acceptable → RTO = 48-72h
Une journée acceptable → RTO = 8-24h
Quelques heures maximum → RTO = 2-4h
Doit être rapide → RTO = 30min-1h
Quasi immédiat → RTO = 5-15min
```

### Relation RPO/RTO et stratégie de backup

Plus vos objectifs RPO/RTO sont stricts, plus votre stratégie doit être sophistiquée :

```
RPO/RTO Relaxed (24h/48h):
→ Backups quotidiens
→ Restauration manuelle
→ Documentation simple

RPO/RTO Modéré (4h/8h):
→ Backups plusieurs fois par jour
→ Scripts de restauration semi-automatiques
→ Tests mensuels

RPO/RTO Strict (1h/2h):
→ Backups horaires + réplication
→ Restauration automatisée
→ Tests hebdomadaires
→ Haute disponibilité

RPO/RTO Critique (<15min/<30min):
→ Réplication continue
→ Failover automatique
→ Cluster multi-node HA
→ Tests quotidiens
```

### Types de désastres

Votre DRP doit couvrir différents scénarios :

**Niveau 1 - Incident mineur** :
- Pod qui crashe
- Service indisponible temporairement
- Erreur de configuration

**Impact** : Faible, localisé
**Fréquence** : Régulier
**Réponse** : Redémarrage, rollback simple

**Niveau 2 - Incident majeur** :
- Node qui tombe
- Application complète down
- Base de données corrompue

**Impact** : Modéré, un service complet
**Fréquence** : Occasionnel
**Réponse** : Restauration depuis backup, réparation

**Niveau 3 - Désastre local** :
- Serveur complet HS
- Disque dur défaillant
- Corruption du cluster

**Impact** : Important, tout le cluster
**Fréquence** : Rare
**Réponse** : Restauration complète, migration

**Niveau 4 - Désastre majeur** :
- Incendie du datacenter
- Catastrophe naturelle
- Cyberattaque majeure

**Impact** : Critique, perte totale
**Fréquence** : Très rare
**Réponse** : Reprise sur site distant, activation du DRP complet

## Structure d'un DRP

Voici comment structurer votre plan de reprise d'activité.

### 1. Page de garde et informations essentielles

```markdown
# Plan de Reprise d'Activité
## Cluster MicroK8s - [Nom du projet]

**Version:** 2.1
**Date de création:** 15 janvier 2025
**Dernière mise à jour:** 15 janvier 2025
**Prochain test:** 1er février 2025

**Propriétaire:** [Votre nom]
**Contacts d'urgence:**
- Propriétaire principal: [Nom] - [Email] - [Téléphone]
- Backup: [Nom] - [Email] - [Téléphone]
- Support technique: [Contact]

**Emplacement des sauvegardes:**
- Local: /mnt/backups/microk8s
- Cloud: s3://mon-bucket-backup/microk8s
- Archive: Google Drive - dossier "MicroK8s Backups"

**Accès critiques:**
- MinIO: http://minio.example.com (credentials dans 1Password)
- Velero: kubectl (config dans ~/.kube)
- Serveur: SSH user@server.example.com

**Classification:**
- RPO: 4 heures
- RTO: 8 heures
```

### 2. Vue d'ensemble du système

```markdown
## Architecture du cluster

### Infrastructure
- **Type:** MicroK8s single-node
- **OS:** Ubuntu 22.04 LTS
- **Serveur:** Dell PowerEdge / Home Lab
- **CPU:** 8 cores
- **RAM:** 32 GB
- **Stockage:** 500 GB SSD

### Applications hébergées

| Application | Namespace | Criticité | RPO | RTO |
|-------------|-----------|-----------|-----|-----|
| Blog WordPress | production | Moyenne | 24h | 8h |
| Base PostgreSQL | database | Haute | 4h | 4h |
| GitLab | devops | Haute | 2h | 4h |
| Monitoring | monitoring | Faible | 24h | 24h |

### Dépendances externes
- DNS: Cloudflare
- Certificats: Let's Encrypt via cert-manager
- Storage: MinIO (local)
- Registry: Docker Hub + registry local
```

### 3. Stratégie de sauvegarde

```markdown
## Stratégie de sauvegarde actuelle

### Backups automatiques

**Quotidien (2h00):**
- Backup complet avec Velero
- Rétention: 7 jours
- Inclut: Tous les namespaces utilisateurs + volumes
- Destination: MinIO local + S3

**Hebdomadaire (Dimanche 1h00):**
- Backup archive
- Rétention: 90 jours
- Destination: S3 uniquement

**Horaire (applications critiques):**
- GitLab: toutes les 2h
- PostgreSQL: toutes les heures (pg_dump)
- Rétention: 24 heures

### Backups manuels

**Avant maintenance:**
- Backup complet avant toute mise à jour majeure
- Rétention: Jusqu'à confirmation succès

**Configurations:**
- Export Git quotidien des manifestes
- Repository: github.com/user/k8s-configs
```

### 4. Inventaire des ressources

```markdown
## Inventaire des ressources critiques

### Namespaces

- `production`: Applications web en production
- `database`: Bases de données PostgreSQL, MySQL
- `devops`: GitLab, Jenkins, registre Docker
- `monitoring`: Prometheus, Grafana, Alertmanager

### PersistentVolumes critiques

| PV Name | Application | Taille | Données |
|---------|-------------|--------|---------|
| postgres-data-pv | PostgreSQL | 50Gi | Base de données production |
| gitlab-data-pv | GitLab | 100Gi | Repos Git, artifacts |
| wordpress-pv | WordPress | 20Gi | Média, uploads |

### Secrets critiques

- `postgres-credentials`: Accès base de données
- `gitlab-secrets`: Clés GitLab, OAuth
- `tls-certificates`: Certificats SSL wildcard
- `registry-auth`: Authentification registre Docker
- `minio-credentials`: Accès stockage backups

**Emplacement des secrets hors cluster:**
- 1Password vault: "MicroK8s Production"
- Fichier chiffré: secrets-backup.gpg (sur disque externe)
```

### 5. Procédures de restauration

C'est le cœur du DRP. Chaque scénario doit avoir une procédure détaillée.

#### Procédure 1 : Restauration d'une application unique

```markdown
## Procédure: Restauration d'une application unique

**Scénario:** Une application (ex: WordPress) est corrompue ou supprimée

**Prérequis:**
- Accès kubectl au cluster
- Backup récent disponible
- 15-30 minutes

**Étapes:**

1. Identifier le dernier backup valide
   ```bash
   velero backup get | grep wordpress
   ```

2. Créer un namespace temporaire pour tests
   ```bash
   kubectl create namespace wordpress-restore-test
   ```

3. Restaurer l'application
   ```bash
   velero restore create wordpress-restore-$(date +%Y%m%d) \
     --from-backup [BACKUP_NAME] \
     --include-namespaces production \
     --include-resources deployment,service,pvc,configmap,secret \
     --selector app=wordpress \
     --namespace-mappings production:wordpress-restore-test
   ```

4. Attendre la fin de la restauration
   ```bash
   velero restore describe wordpress-restore-[DATE]
   ```

5. Vérifier que l'application démarre
   ```bash
   kubectl get pods -n wordpress-restore-test
   kubectl logs -n wordpress-restore-test [POD_NAME]
   ```

6. Tester l'accès
   ```bash
   kubectl port-forward -n wordpress-restore-test service/wordpress 8080:80
   # Ouvrir http://localhost:8080
   ```

7. Si OK, restaurer en production
   ```bash
   # Supprimer l'ancienne version
   kubectl delete all -n production -l app=wordpress

   # Restaurer en production
   velero restore create wordpress-prod-restore \
     --from-backup [BACKUP_NAME] \
     --include-namespaces production \
     --selector app=wordpress
   ```

8. Nettoyer le namespace de test
   ```bash
   kubectl delete namespace wordpress-restore-test
   ```

**Temps estimé:** 20-30 minutes

**Points d'attention:**
- Vérifier que les PVCs sont bien bound
- Vérifier les secrets (passwords)
- Tester l'accès avant de supprimer l'ancien
```

#### Procédure 2 : Restauration complète du cluster

```markdown
## Procédure: Restauration complète du cluster

**Scénario:** Le serveur complet est perdu, reconstruction nécessaire

**Prérequis:**
- Nouveau serveur ou machine disponible
- Accès aux backups (MinIO/S3)
- Fichier de credentials Velero
- 2-4 heures

**Phase 1 - Installation du système de base (30 min)**

1. Installer Ubuntu Server 22.04
   ```bash
   # Installation standard
   # Configurer le réseau
   # Mettre à jour le système
   sudo apt update && sudo apt upgrade -y
   ```

2. Installer MicroK8s
   ```bash
   sudo snap install microk8s --classic --channel=1.28/stable
   sudo usermod -aG microk8s $USER
   sudo chown -R $USER ~/.kube
   newgrp microk8s
   ```

3. Activer les addons essentiels
   ```bash
   microk8s enable dns storage
   microk8s status --wait-ready
   ```

4. Configurer kubectl
   ```bash
   mkdir -p ~/.kube
   microk8s config > ~/.kube/config
   ```

**Phase 2 - Installation de Velero (15 min)**

1. Télécharger Velero CLI
   ```bash
   wget https://github.com/vmware-tanzu/velero/releases/download/v1.12.0/velero-v1.12.0-linux-amd64.tar.gz
   tar -xvzf velero-v1.12.0-linux-amd64.tar.gz
   sudo mv velero-v1.12.0-linux-amd64/velero /usr/local/bin/
   ```

2. Récupérer le fichier de credentials
   ```bash
   # Depuis la sauvegarde sécurisée (USB, 1Password, etc.)
   cat > credentials-velero <<EOF
   [default]
   aws_access_key_id = [ACCESS_KEY]
   aws_secret_access_key = [SECRET_KEY]
   EOF
   ```

3. Installer Velero
   ```bash
   velero install \
     --provider aws \
     --plugins velero/velero-plugin-for-aws:v1.8.0 \
     --bucket velero-backups \
     --secret-file ./credentials-velero \
     --use-node-agent \
     --backup-location-config region=minio,s3ForcePathStyle="true",s3Url=http://[MINIO_URL]:9000
   ```

4. Vérifier l'accès aux backups
   ```bash
   velero backup get
   ```

**Phase 3 - Restauration (1-2h selon la taille)**

1. Identifier le backup à restaurer
   ```bash
   velero backup get
   # Choisir le plus récent en status "Completed"
   ```

2. Restaurer les ressources cluster
   ```bash
   velero restore create full-restore-$(date +%Y%m%d) \
     --from-backup [BACKUP_NAME] \
     --wait
   ```

3. Surveiller la progression
   ```bash
   watch kubectl get pods --all-namespaces
   ```

4. Vérifier les namespaces
   ```bash
   kubectl get namespaces
   ```

5. Vérifier les PVCs
   ```bash
   kubectl get pvc --all-namespaces
   ```

**Phase 4 - Vérification et tests (30 min)**

1. Vérifier chaque application critique
   ```bash
   # Pour chaque application
   kubectl get pods -n [NAMESPACE]
   kubectl logs -n [NAMESPACE] [POD]
   kubectl describe pod -n [NAMESPACE] [POD]
   ```

2. Tester la connectivité
   ```bash
   # Tester les services
   kubectl get services --all-namespaces

   # Tester les ingress
   kubectl get ingress --all-namespaces
   ```

3. Vérifier les bases de données
   ```bash
   # PostgreSQL
   kubectl exec -n database postgres-0 -- psql -U postgres -c "\l"

   # MySQL
   kubectl exec -n database mysql-0 -- mysql -u root -p[PASSWORD] -e "SHOW DATABASES;"
   ```

4. Tester l'accès externe
   ```bash
   # Vérifier DNS
   nslookup myapp.example.com

   # Tester HTTPS
   curl https://myapp.example.com
   ```

**Phase 5 - Remise en production (30 min)**

1. Réactiver les addons nécessaires
   ```bash
   microk8s enable prometheus
   microk8s enable cert-manager
   microk8s enable metallb:[IP_RANGE]
   ```

2. Mettre à jour les DNS si nécessaire
   ```bash
   # Pointer vers la nouvelle IP
   ```

3. Réactiver les sauvegardes automatiques
   ```bash
   velero schedule get
   # Vérifier que les schedules sont actifs
   ```

4. Documentation
   ```bash
   # Noter les changements
   echo "$(date): Restauration complète effectuée depuis backup [BACKUP_NAME]" >> /var/log/drp-history.log
   ```

**Temps total estimé:** 3-4 heures

**Checklist finale:**
- [ ] Tous les namespaces restaurés
- [ ] Tous les pods running
- [ ] PVCs bound
- [ ] Services accessibles
- [ ] DNS configuré
- [ ] Certificats SSL valides
- [ ] Monitoring actif
- [ ] Backups réactivés
- [ ] Documentation mise à jour
```

#### Procédure 3 : Restauration de base de données

```markdown
## Procédure: Restauration d'une base de données

**Scénario:** Base de données corrompue, suppression accidentelle

**Prérequis:**
- Backup de la base disponible
- Accès au cluster
- 30-60 minutes

**Pour PostgreSQL:**

1. Identifier le backup
   ```bash
   velero backup get | grep postgres
   ```

2. Mettre l'application en maintenance
   ```bash
   # Scaler à 0 les applications qui utilisent la BDD
   kubectl scale deployment webapp -n production --replicas=0
   ```

3. Créer un namespace temporaire
   ```bash
   kubectl create namespace postgres-restore
   ```

4. Restaurer PostgreSQL dans le namespace temporaire
   ```bash
   velero restore create postgres-test-restore \
     --from-backup [BACKUP_NAME] \
     --include-namespaces database \
     --namespace-mappings database:postgres-restore \
     --wait
   ```

5. Attendre que PostgreSQL démarre
   ```bash
   kubectl wait --for=condition=ready pod -l app=postgres -n postgres-restore --timeout=300s
   ```

6. Extraire un dump de la BDD restaurée
   ```bash
   POD=$(kubectl get pod -l app=postgres -n postgres-restore -o jsonpath='{.items[0].metadata.name}')

   kubectl exec -n postgres-restore $POD -- \
     pg_dump -U postgres mydb > /tmp/restored-db.sql
   ```

7. Restaurer dans la BDD de production
   ```bash
   POD_PROD=$(kubectl get pod -l app=postgres -n database -o jsonpath='{.items[0].metadata.name}')

   # Backup de sécurité
   kubectl exec -n database $POD_PROD -- \
     pg_dump -U postgres mydb > /tmp/before-restore-backup.sql

   # Drop et recréer
   kubectl exec -n database $POD_PROD -- \
     psql -U postgres -c "DROP DATABASE mydb;"
   kubectl exec -n database $POD_PROD -- \
     psql -U postgres -c "CREATE DATABASE mydb;"

   # Importer
   cat /tmp/restored-db.sql | \
     kubectl exec -i -n database $POD_PROD -- \
     psql -U postgres mydb
   ```

8. Vérifier les données
   ```bash
   kubectl exec -n database $POD_PROD -- \
     psql -U postgres mydb -c "SELECT COUNT(*) FROM users;"
   ```

9. Redémarrer les applications
   ```bash
   kubectl scale deployment webapp -n production --replicas=3
   ```

10. Nettoyer
    ```bash
    kubectl delete namespace postgres-restore
    ```

**Temps estimé:** 45-60 minutes

**Points d'attention:**
- Toujours faire un backup avant la restauration
- Vérifier l'intégrité des données après restauration
- Tester avec une requête sur les données critiques
```

### 6. Scénarios de désastre et réponses

```markdown
## Matrice des scénarios

| Scénario | Probabilité | Impact | Détection | Temps de réponse | Procédure |
|----------|-------------|--------|-----------|------------------|-----------|
| Pod crashe | Haute | Faible | Automatique | Immédiat | Auto-restart |
| Node down | Moyenne | Moyen | Monitoring | 5 min | Proc. 1 |
| Serveur HS | Faible | Élevé | Manuel | 15 min | Proc. 2 |
| Corruption BDD | Faible | Élevé | Application | 30 min | Proc. 3 |
| Disque plein | Moyenne | Moyen | Monitoring | 15 min | Proc. 4 |
| Ransomware | Très faible | Critique | Manuel | Variable | Proc. 5 |
| Incendie | Très faible | Critique | Manuel | Variable | Proc. 6 |

### Arbre de décision

```
PROBLÈME DÉTECTÉ
    │
    ├─ Application spécifique down?
    │   ├─ OUI → Proc. 1: Restauration app
    │   └─ NON → Continuer
    │
    ├─ Node accessible?
    │   ├─ NON → Proc. 2: Restauration complète
    │   └─ OUI → Continuer
    │
    ├─ Base de données affectée?
    │   ├─ OUI → Proc. 3: Restauration BDD
    │   └─ NON → Continuer
    │
    ├─ Espace disque critique?
    │   ├─ OUI → Proc. 4: Libération espace
    │   └─ NON → Continuer
    │
    └─ Cause inconnue?
        └─ Investigation approfondie
```
```

### 7. Tests et maintenance du DRP

```markdown
## Planning de tests

### Tests mensuels (1er de chaque mois)

**Test de validation des backups (30 min):**
- Vérifier tous les backups du mois
- Confirmer leur statut "Completed"
- Vérifier la taille (cohérence)
- Tester l'accès au stockage

**Test de restauration partielle (1h):**
- Restaurer une application non-critique
- Vérifier le fonctionnement
- Documenter le temps
- Noter les problèmes

### Tests trimestriels (15 de mars, juin, sept, déc)

**Test de restauration complète (4h):**
- Restaurer le cluster entier sur VM de test
- Valider toutes les applications
- Mesurer RTO réel
- Mettre à jour la documentation

**Drill de disaster recovery (2h):**
- Simulation sans préavis
- Chaque membre de l'équipe participe
- Chronométrer chaque étape
- Débriefing et amélioration

### Tests annuels (Janvier)

**Audit complet (1 journée):**
- Revue complète du DRP
- Test de tous les scénarios
- Mise à jour de tous les contacts
- Formation/recyclage équipe
- Validation avec stakeholders

### Maintenance continue

**Après chaque changement majeur:**
- Mise à jour du DRP
- Test de la nouvelle configuration
- Communication à l'équipe

**Hebdomadaire:**
- Vérification automatique des backups
- Revue des alertes

**Documentation:**
- Journal des tests dans `drp-test-log.md`
- Rapport après chaque test
- Actions correctives suivies
```

### 8. Contacts et escalade

```markdown
## Contacts d'urgence

### Niveau 1 - Premier contact

**Admin système principal:**
- Nom: Jean Dupont
- Email: jean.dupont@example.com
- Téléphone: +33 6 12 34 56 78
- Disponibilité: 24/7
- Responsabilités: Toutes opérations DRP

### Niveau 2 - Escalade

**Admin backup:**
- Nom: Marie Martin
- Email: marie.martin@example.com
- Téléphone: +33 6 98 76 54 32
- Disponibilité: Heures ouvrables + astreinte
- Responsabilités: Velero, stockage, backups

**Développeur lead:**
- Nom: Pierre Durand
- Email: pierre.durand@example.com
- Téléphone: +33 6 11 22 33 44
- Disponibilité: Heures ouvrables
- Responsabilités: Applications, bases de données

### Niveau 3 - Support externe

**Support hébergeur:**
- Contact: support@hosting-provider.com
- Téléphone: 01 23 45 67 89
- Contrat: #123456
- Disponibilité: 24/7

**Consultant Kubernetes:**
- Contact: expert@k8s-consulting.com
- Téléphone: +33 6 55 66 77 88
- Tarif: 200€/h
- Disponibilité: Sur demande

### Procédure d'escalade

```
Incident détecté
    ↓
Niveau 1 notifié immédiatement
    ↓
Si pas de réponse en 15 min
    ↓
Niveau 2 notifié
    ↓
Si problème non résolu en 2h
    ↓
Niveau 3 contacté
    ↓
Communication aux stakeholders
```

### Communication

**Stakeholders à informer:**
- Direction: direction@example.com
- Utilisateurs: via page status.example.com
- Équipe technique: Slack #incidents

**Templates de communication:**

**Incident détecté:**
```
INCIDENT - [Niveau] - [Timestamp]
Service affecté: [Nom]
Impact: [Description]
Actions en cours: [Liste]
Temps estimé de résolution: [Durée]
Prochaine mise à jour: [Timestamp]
```

**Incident résolu:**
```
RÉSOLU - [Timestamp]
Service: [Nom]
Durée du downtime: [Durée]
Cause: [Description]
Actions prises: [Liste]
Prévention: [Mesures]
Post-mortem: [Date]
```
```

### 9. Outils et ressources

```markdown
## Outils nécessaires

### Logiciels requis

- **kubectl**: Client Kubernetes
  - Installation: `snap install kubectl --classic`
  - Config: `~/.kube/config`

- **velero**: CLI de backup
  - Installation: Voir section 22.2
  - Config: credentials dans `/root/.velero/`

- **yq**: Manipulation YAML
  - Installation: `snap install yq`

- **jq**: Manipulation JSON
  - Installation: `apt install jq`

### Accès critiques

**Serveur principal:**
```bash
ssh admin@server.example.com
# Clé SSH: ~/.ssh/microk8s_prod
```

**MinIO (stockage backups):**
```bash
URL: http://minio.example.com:9000
User: minio-admin
Pass: [Voir 1Password]
```

**Cloud S3:**
```bash
Region: eu-west-1
Bucket: microk8s-backups
Access Key: [Voir 1Password]
```

### Documentation de référence

**Interne:**
- Architecture: `docs/architecture.md`
- Runbooks: `docs/runbooks/`
- Configurations: `git@github.com:user/k8s-configs.git`

**Externe:**
- MicroK8s: https://microk8s.io/docs
- Velero: https://velero.io/docs
- Kubernetes: https://kubernetes.io/docs

### Équipements

**Disques de secours:**
- USB 1TB: Backups mensuels offline
- Emplacement: Coffre-fort bureau
- Dernière mise à jour: [Date]

**Machine de spare:**
- Dell OptiPlex 7080
- Emplacement: Armoire serveur
- État: Prêt à l'emploi, OS préinstallé
```

## Création de votre DRP

Maintenant que vous comprenez la structure, créons votre DRP étape par étape.

### Étape 1 : Analyse de risques

Identifiez vos risques spécifiques :

```markdown
## Analyse de risques - Mon cluster MicroK8s

| Risque | Probabilité | Impact | Mitigation actuelle | À améliorer |
|--------|-------------|--------|---------------------|-------------|
| Panne disque | Moyenne | Élevé | Backups quotidiens | RAID |
| Erreur humaine | Haute | Moyen | Git pour configs | Formation |
| Coupure électrique | Moyenne | Élevé | Aucune | UPS |
| Ransomware | Faible | Critique | Backups offline | 2FA |
| Mise à jour ratée | Moyenne | Moyen | Backup avant MAJ | Tests staging |
```

### Étape 2 : Définir RPO/RTO par application

```markdown
## Objectifs de récupération

| Application | Type | Criticité | RPO | RTO | Justification |
|-------------|------|-----------|-----|-----|---------------|
| Blog WordPress | Web | Moyenne | 24h | 8h | Trafic modéré, pas de transactions |
| GitLab | Dev | Haute | 2h | 4h | Code source critique |
| PostgreSQL prod | BDD | Critique | 1h | 2h | Données business importantes |
| Prometheus | Monitoring | Faible | 24h | 24h | Métriques reconstituables |
| Jenkins | CI/CD | Moyenne | 4h | 8h | Pipelines rejouables |
```

### Étape 3 : Documenter l'existant

```bash
#!/bin/bash
# document-current-state.sh

cat > drp-inventory-$(date +%Y%m%d).md <<EOF
# Inventaire du cluster - $(date)

## Namespaces
\`\`\`
$(kubectl get namespaces)
\`\`\`

## Applications par namespace
$(for ns in $(kubectl get namespaces -o jsonpath='{.items[*].metadata.name}' | tr ' ' '\n' | grep -v 'kube-\|default'); do
  echo "### Namespace: $ns"
  echo "\`\`\`"
  kubectl get all -n $ns
  echo "\`\`\`"
  echo ""
done)

## Volumes persistants
\`\`\`
$(kubectl get pv,pvc --all-namespaces)
\`\`\`

## Ingress
\`\`\`
$(kubectl get ingress --all-namespaces)
\`\`\`

## Configuration Velero
\`\`\`
$(velero backup-location get)
$(velero schedule get)
\`\`\`

EOF

echo "Inventaire créé: drp-inventory-$(date +%Y%m%d).md"
```

### Étape 4 : Rédiger les procédures

Utilisez ce template pour chaque procédure :

```markdown
## Procédure: [Nom]

**Déclencheurs:**
- [Condition 1]
- [Condition 2]

**Prérequis:**
- [ ] [Ressource 1]
- [ ] [Ressource 2]
- [ ] [Accès nécessaire]

**Temps estimé:** [Durée]

**Étapes détaillées:**

### Étape 1: [Nom]
**Action:**
```bash
[commande]
```

**Résultat attendu:**
```
[sortie attendue]
```

**Si échec:**
- [Action corrective]

### Étape 2: [Nom]
[...]

**Vérification finale:**
- [ ] [Check 1]
- [ ] [Check 2]

**Rollback si échec:**
[Procédure de retour arrière]
```

### Étape 5 : Tester et affiner

```bash
#!/bin/bash
# test-drp.sh

echo "=== Test du DRP - $(date) ==="

# 1. Test de validation backup
echo "1. Validation des backups..."
./scripts/validate-backups.sh
if [ $? -ne 0 ]; then
  echo "❌ Échec validation backups"
  exit 1
fi

# 2. Test de restauration partielle
echo "2. Test de restauration..."
./scripts/test-restore.sh
if [ $? -ne 0 ]; then
  echo "❌ Échec test restauration"
  exit 1
fi

# 3. Générer le rapport
./scripts/generate-drp-report.sh

echo "✓ Test DRP terminé"
```

## Automatisation du DRP

Certaines parties du DRP peuvent être automatisées pour gagner du temps.

### Scripts d'aide à la décision

```bash
#!/bin/bash
# drp-assistant.sh - Assistant interactif DRP

echo "=== Assistant DRP ==="
echo "Quel est le problème rencontré ?"
echo ""
echo "1) Une application ne répond plus"
echo "2) Le serveur est inaccessible"
echo "3) Une base de données est corrompue"
echo "4) Perte de données (suppression accidentelle)"
echo "5) Le cluster entier est down"
echo "6) Autre / Je ne sais pas"
echo ""

read -p "Choix (1-6): " CHOICE

case $CHOICE in
  1)
    echo ""
    echo "→ Procédure recommandée: Restauration d'application"
    echo "   Fichier: procedures/restore-single-app.md"
    echo ""
    read -p "Quelle application? " APP
    echo "Backup le plus récent pour $APP:"
    velero backup get | grep $APP | head -5
    echo ""
    read -p "Lancer la procédure? (y/n) " CONFIRM
    if [ "$CONFIRM" = "y" ]; then
      ./procedures/restore-single-app.sh $APP
    fi
    ;;

  2)
    echo ""
    echo "→ SITUATION CRITIQUE"
    echo "   Procédure: Restauration complète"
    echo "   Temps estimé: 3-4 heures"
    echo "   Fichier: procedures/full-cluster-restore.md"
    echo ""
    echo "Contacts d'urgence:"
    cat contacts.txt
    echo ""
    echo "Étapes initiales:"
    echo "1. Préparer nouveau serveur"
    echo "2. Installer MicroK8s"
    echo "3. Installer Velero"
    echo "4. Lancer restauration"
    ;;

  3)
    echo ""
    echo "→ Procédure recommandée: Restauration BDD"
    echo "   Fichier: procedures/restore-database.md"
    echo ""
    read -p "Quelle BDD? (postgres/mysql/mongodb) " DB
    ./procedures/restore-database.sh $DB
    ;;

  4)
    echo ""
    echo "→ Deux options:"
    echo "   A) Restaurer depuis backup récent"
    echo "   B) Récupérer fichiers spécifiques"
    echo ""
    read -p "Choix (A/B): " OPTION
    if [ "$OPTION" = "A" ]; then
      velero backup get
    else
      ./procedures/recover-specific-files.sh
    fi
    ;;

  5)
    echo ""
    echo "→ URGENCE MAXIMALE"
    echo "   RTO: 2-4 heures"
    echo "   Contacter immédiatement:"
    cat contacts.txt | head -5
    echo ""
    echo "Procédure: procedures/disaster-recovery-full.md"
    ;;

  6)
    echo ""
    echo "→ Diagnostic nécessaire"
    echo "   Exécution des vérifications..."
    ./scripts/diagnose.sh
    ;;
esac
```

### Monitoring et alertes

```yaml
# alertmanager-drp-config.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: alertmanager-drp
  namespace: monitoring
data:
  alertmanager.yml: |
    route:
      receiver: 'drp-team'
      routes:
      - match:
          severity: critical
        receiver: 'drp-escalation'

    receivers:
    - name: 'drp-team'
      email_configs:
      - to: 'team@example.com'
        send_resolved: true

    - name: 'drp-escalation'
      email_configs:
      - to: 'oncall@example.com'
        headers:
          Priority: urgent
      webhook_configs:
      - url: 'https://hooks.slack.com/services/YOUR/WEBHOOK'
```

### Dashboards de santé

```yaml
# grafana-drp-dashboard.json
{
  "dashboard": {
    "title": "DRP Health Status",
    "panels": [
      {
        "title": "Backup Success Rate",
        "targets": [{
          "expr": "sum(velero_backup_success_total) / sum(velero_backup_total) * 100"
        }]
      },
      {
        "title": "Time Since Last Backup",
        "targets": [{
          "expr": "(time() - velero_backup_last_successful_timestamp{schedule=\"daily\"}) / 3600"
        }]
      },
      {
        "title": "Backup Size Trend",
        "targets": [{
          "expr": "velero_backup_size_bytes"
        }]
      },
      {
        "title": "RTO Simulation",
        "targets": [{
          "expr": "avg(velero_restore_duration_seconds)"
        }]
      }
    ]
  }
}
```

## Amélioration continue

Un DRP n'est jamais terminé. Il doit évoluer constamment.

### Processus d'amélioration

```markdown
## Cycle d'amélioration continue

1. **Test** → Identifier les faiblesses
2. **Analyse** → Comprendre les causes
3. **Correction** → Améliorer le DRP
4. **Documentation** → Mettre à jour les procédures
5. **Formation** → Partager les learnings
6. **Retour au test** → Valider les améliorations
```

### Post-mortem après incident

```markdown
# Template Post-Mortem

**Incident:** [Description courte]
**Date:** [Date]
**Durée:** [Durée totale]
**Impact:** [Description impact]

## Chronologie

| Heure | Événement |
|-------|-----------|
| 14:00 | Incident détecté |
| 14:05 | Équipe notifiée |
| 14:15 | Diagnostic démarré |
| 14:30 | Restauration lancée |
| 16:00 | Service restauré |
| 16:30 | Validation complète |

## Cause racine

[Description détaillée de la cause]

## Ce qui a bien fonctionné

- [Point 1]
- [Point 2]

## Ce qui n'a pas fonctionné

- [Point 1] → Action: [Solution]
- [Point 2] → Action: [Solution]

## Actions correctives

| Action | Responsable | Date cible | Status |
|--------|-------------|------------|--------|
| [Action 1] | [Nom] | [Date] | [ ] |
| [Action 2] | [Nom] | [Date] | [ ] |

## Mises à jour du DRP

- [ ] Procédure X mise à jour
- [ ] Contact Y ajouté
- [ ] Script Z créé

**Prochaine revue:** [Date]
```

### Métriques de performance du DRP

```bash
#!/bin/bash
# drp-metrics.sh

cat > drp-metrics-$(date +%Y%m).md <<EOF
# Métriques DRP - $(date +%B\ %Y)

## Indicateurs clés

**Backups:**
- Total effectués: $(velero backup get | wc -l)
- Taux de réussite: $(echo "scale=2; $(velero backup get | grep Completed | wc -l) * 100 / $(velero backup get | wc -l)" | bc)%
- Taille moyenne: $(velero backup get -o json | jq '[.items[].status.backupSizeBytes] | add / length / 1024 / 1024 / 1024' | xargs printf "%.2f GB\n")

**Tests:**
- Tests effectués: $(cat drp-test-log.csv | grep $(date +%Y-%m) | wc -l)
- Tests réussis: $(cat drp-test-log.csv | grep $(date +%Y-%m) | grep Pass | wc -l)
- RTO moyen: $(cat drp-test-log.csv | grep $(date +%Y-%m) | awk -F',' '{sum+=$4; n++} END {print sum/n}') min

**Incidents:**
- Incidents: $(cat incident-log.csv | grep $(date +%Y-%m) | wc -l)
- DRP activé: $(cat incident-log.csv | grep $(date +%Y-%m) | grep "DRP activated" | wc -l)
- Temps moyen de résolution: [À calculer]

## Objectifs vs Réalisé

| Métrique | Objectif | Réalisé | Status |
|----------|----------|---------|--------|
| RPO | 4h | [Calculé] | [✓/✗] |
| RTO | 8h | [Mesuré] | [✓/✗] |
| Tests/mois | 2 | $(cat drp-test-log.csv | grep $(date +%Y-%m) | wc -l) | [✓/✗] |
| Taux réussite backups | 98% | [Calculé] | [✓/✗] |

## Recommandations

- [Recommandation 1]
- [Recommandation 2]

EOF
```

## Checklist de mise en œuvre

Pour créer et maintenir votre DRP :

```markdown
## Phase 1 : Création (1-2 semaines)

- [ ] Analyser les risques spécifiques
- [ ] Définir RPO/RTO pour chaque application
- [ ] Documenter l'architecture actuelle
- [ ] Inventorier toutes les ressources
- [ ] Lister les dépendances
- [ ] Identifier les contacts clés
- [ ] Rédiger les procédures principales
- [ ] Créer les scripts d'automatisation
- [ ] Préparer les templates

## Phase 2 : Validation (1 semaine)

- [ ] Relecture par l'équipe
- [ ] Test des procédures critiques
- [ ] Validation des accès et credentials
- [ ] Vérification des backups
- [ ] Test de restauration partielle
- [ ] Corrections et ajustements

## Phase 3 : Déploiement (1 semaine)

- [ ] Formation de l'équipe
- [ ] Distribution du DRP
- [ ] Configuration des alertes
- [ ] Mise en place du monitoring
- [ ] Communication aux stakeholders
- [ ] Premier test grandeur nature

## Phase 4 : Maintenance continue

**Mensuel:**
- [ ] Test de validation
- [ ] Revue des métriques
- [ ] Mise à jour si nécessaire

**Trimestriel:**
- [ ] Test de restauration complète
- [ ] Drill d'équipe
- [ ] Revue approfondie du document

**Annuel:**
- [ ] Audit complet
- [ ] Révision des RPO/RTO
- [ ] Formation/recyclage
- [ ] Mise à jour majeure
```

## Conclusion

Un Plan de Reprise d'Activité solide est votre meilleure assurance contre les désastres. C'est la différence entre quelques heures de downtime organisé et plusieurs jours de panique et de perte de données.

### Les piliers d'un bon DRP

✅ **Complet** : Couvre tous les scénarios de disaster
✅ **Clair** : Procédures étape par étape, sans ambiguïté
✅ **Testé** : Validé régulièrement en conditions réelles
✅ **Accessible** : Disponible même si le cluster est down
✅ **À jour** : Maintenu à jour avec l'infrastructure
✅ **Partagé** : Connu et compris par toute l'équipe

### Erreurs à éviter

❌ **DRP théorique jamais testé** : Découvrir qu'il ne fonctionne pas pendant un vrai incident
❌ **Documentation obsolète** : Procédures qui ne correspondent plus à la réalité
❌ **Une seule personne sait** : Point de défaillance unique
❌ **Pas de maintenance** : Le DRP n'évolue pas avec l'infra
❌ **Trop complexe** : Impossible à suivre sous stress
❌ **Stocké uniquement en ligne** : Inaccessible quand on en a besoin

### Vos premiers pas

Si vous n'avez pas encore de DRP :

1. **Commencez simple** : Document Word de 5 pages avec les procédures essentielles
2. **Testez-le rapidement** : Validation de base dans la semaine
3. **Itérez** : Améliorez après chaque test
4. **Partagez** : Mettez-le sur Google Drive, GitHub, USB

Si vous avez déjà un DRP :

1. **Testez-le maintenant** : Trouvez les failles avant qu'elles ne comptent
2. **Mettez-le à jour** : Reflète-t-il vraiment votre infra actuelle ?
3. **Automatisez** : Scripts pour gagner du temps
4. **Formez** : Tout le monde doit savoir

### La règle d'or

**Le meilleur moment pour créer/tester votre DRP était hier.
Le deuxième meilleur moment est maintenant.**

N'attendez pas le désastre pour réaliser que vous n'êtes pas préparé.

---

**Checklist finale DRP** :

- [ ] DRP écrit et structuré
- [ ] Procédures détaillées pour chaque scénario
- [ ] RPO/RTO définis et documentés
- [ ] Contacts d'urgence à jour
- [ ] Accès et credentials documentés
- [ ] Scripts d'automatisation créés
- [ ] Premier test effectué
- [ ] Équipe formée
- [ ] Planning de maintenance établi
- [ ] Backups offsite configurés
- [ ] Monitoring et alertes en place
- [ ] Post-mortem template prêt

**N'oubliez jamais : Le but du DRP n'est pas d'empêcher les désastres, mais de s'assurer que vous pouvez vous en remettre rapidement et efficacement.**

⏭️ [Automatisation des sauvegardes](/22-sauvegarde-et-restauration/08-automatisation-des-sauvegardes.md)
