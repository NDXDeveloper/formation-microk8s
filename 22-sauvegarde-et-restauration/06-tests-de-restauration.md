🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 22.6 Tests de restauration

## Introduction

**Une sauvegarde non testée n'est pas une sauvegarde.** C'est la règle d'or de toute stratégie de sauvegarde. Vous pouvez avoir les meilleurs outils, les processus les plus automatisés, mais si vous ne testez jamais que vos restaurations fonctionnent réellement, vous découvrirez le problème au pire moment possible : quand vous en avez vraiment besoin.

Cette section vous guide à travers la mise en place de tests de restauration réguliers et efficaces pour votre cluster MicroK8s.

### Pourquoi les tests de restauration sont cruciaux

Imaginez ces scénarios :

**Scénario 1 - La mauvaise surprise** :
```
Votre serveur tombe en panne.
Vous essayez de restaurer depuis vos backups.
ERREUR : Le backup est corrompu.
ERREUR : Il manque des fichiers critiques.
ERREUR : Les secrets ne sont pas dans le backup.
Résultat : Perte de données et plusieurs jours de reconstruction manuelle.
```

**Scénario 2 - La bonne pratique** :
```
Vous testez vos restaurations tous les mois.
Vous découvrez que les volumes ne sont pas sauvegardés correctement.
Vous corrigez le problème.
Quand le serveur tombe vraiment en panne, la restauration fonctionne parfaitement.
Résultat : Remise en service en quelques heures, aucune perte de données.
```

### Ce que vous découvrez en testant

Les tests de restauration révèlent souvent :

1. **Sauvegardes incomplètes** : Des ressources manquantes (ConfigMaps, Secrets, PVCs)
2. **Erreurs de configuration** : Mauvais backup-location, credentials invalides
3. **Problèmes de dépendances** : L'ordre de restauration est important
4. **Volumes non sauvegardés** : Oubli d'activer `--default-volumes-to-fs-backup`
5. **Incompatibilités de versions** : Différences entre clusters source et destination
6. **Temps de restauration** : Découvrir qu'une restauration prend 8 heures au lieu de 2
7. **Lacunes dans la documentation** : La procédure écrite est incomplète ou erronée

### Analogie simple

Pensez aux tests de restauration comme à :

- **Exercice d'incendie** : Vous ne voulez pas découvrir que les issues de secours sont bloquées pendant un vrai incendie
- **Roue de secours** : Vous vérifiez qu'elle est gonflée avant d'avoir une crevaison
- **Plan d'évacuation** : Vous le pratiquez avant l'urgence réelle

Les tests de restauration, c'est exactement ça pour vos données : s'assurer que tout fonctionne avant le jour où vous en aurez vraiment besoin.

## Types de tests de restauration

Il existe plusieurs niveaux de tests, du plus simple au plus complet.

### 1. Test de validation de backup

**Objectif** : Vérifier que la sauvegarde s'est bien créée et est accessible.

**Ce qu'on teste** :
- Le backup existe
- Il est marqué comme "Completed"
- Il contient des données
- Il est accessible depuis le stockage

**Quand** : Après chaque sauvegarde automatique

**Durée** : 1-2 minutes

**Exemple avec Velero** :

```bash
#!/bin/bash
# validate-backup.sh

BACKUP_NAME=$1

echo "=== Validation du backup: $BACKUP_NAME ==="

# Vérifier que le backup existe
if ! velero backup get $BACKUP_NAME &>/dev/null; then
  echo "❌ ERREUR: Le backup n'existe pas"
  exit 1
fi

# Vérifier le statut
STATUS=$(velero backup describe $BACKUP_NAME --details | grep "Phase:" | awk '{print $2}')
if [ "$STATUS" != "Completed" ]; then
  echo "❌ ERREUR: Status = $STATUS (attendu: Completed)"
  exit 1
fi

# Vérifier qu'il contient des ressources
TOTAL=$(velero backup describe $BACKUP_NAME --details | grep "Total items to be backed up:" | awk '{print $6}')
if [ "$TOTAL" -lt 1 ]; then
  echo "❌ ERREUR: Aucune ressource sauvegardée"
  exit 1
fi

echo "✓ Backup valide"
echo "  Status: $STATUS"
echo "  Ressources: $TOTAL"
```

### 2. Test de restauration partielle

**Objectif** : Restaurer une seule application ou namespace pour vérifier que ça fonctionne.

**Ce qu'on teste** :
- La commande de restauration fonctionne
- Les ressources sont recréées
- Les pods démarrent correctement
- Les volumes sont restaurés avec leurs données

**Quand** : Hebdomadaire ou bi-hebdomadaire

**Durée** : 10-30 minutes

**Exemple** :

```bash
#!/bin/bash
# test-partial-restore.sh

BACKUP_NAME="daily-backup-latest"
TEST_NAMESPACE="restore-test"

echo "=== Test de restauration partielle ==="

# 1. Créer un namespace de test
kubectl create namespace $TEST_NAMESPACE --dry-run=client -o yaml | kubectl apply -f -

# 2. Restaurer une application spécifique
velero restore create test-restore-$(date +%Y%m%d-%H%M%S) \
  --from-backup $BACKUP_NAME \
  --include-namespaces webapp \
  --namespace-mappings webapp:$TEST_NAMESPACE

# 3. Attendre la fin de la restauration
echo "Attente de la restauration..."
sleep 30

# 4. Vérifier que les pods démarrent
PODS=$(kubectl get pods -n $TEST_NAMESPACE --no-headers | wc -l)
RUNNING=$(kubectl get pods -n $TEST_NAMESPACE --field-selector=status.phase=Running --no-headers | wc -l)

echo "Pods restaurés: $PODS"
echo "Pods running: $RUNNING"

# 5. Nettoyage
echo "Nettoyage..."
kubectl delete namespace $TEST_NAMESPACE

if [ "$RUNNING" -gt 0 ]; then
  echo "✓ Test réussi"
  exit 0
else
  echo "❌ Test échoué"
  exit 1
fi
```

### 3. Test de restauration complète

**Objectif** : Restaurer l'intégralité d'un environnement sur un cluster vide.

**Ce qu'on teste** :
- Restauration de tous les namespaces
- Toutes les applications redémarrent
- Les données sont intactes
- Les services sont accessibles
- Le temps total de restauration

**Quand** : Mensuel ou trimestriel

**Durée** : 1-4 heures

**Exemple** :

```bash
#!/bin/bash
# test-full-restore.sh

BACKUP_NAME="weekly-full-backup"

echo "=== Test de restauration complète ==="
echo "ATTENTION: Ce test restaure TOUT sur ce cluster"
read -p "Continuer? (yes/no): " CONFIRM

if [ "$CONFIRM" != "yes" ]; then
  echo "Test annulé"
  exit 0
fi

START_TIME=$(date +%s)

# 1. Sauvegarder l'état actuel (au cas où)
echo "Sauvegarde de sécurité..."
velero backup create pre-restore-safety-backup --wait

# 2. Restaurer depuis le backup
echo "Restauration complète..."
velero restore create full-restore-test-$(date +%Y%m%d-%H%M%S) \
  --from-backup $BACKUP_NAME \
  --wait

# 3. Vérifier tous les namespaces
echo "Vérification des namespaces..."
NAMESPACES=$(kubectl get namespaces -o jsonpath='{.items[*].metadata.name}' | tr ' ' '\n' | grep -v 'kube-\|default')

for ns in $NAMESPACES; do
  echo "  Namespace: $ns"
  PODS=$(kubectl get pods -n $ns --no-headers 2>/dev/null | wc -l)
  RUNNING=$(kubectl get pods -n $ns --field-selector=status.phase=Running --no-headers 2>/dev/null | wc -l)
  echo "    Pods: $RUNNING/$PODS running"
done

# 4. Calculer le temps
END_TIME=$(date +%s)
DURATION=$((END_TIME - START_TIME))
MINUTES=$((DURATION / 60))

echo ""
echo "✓ Test terminé en $MINUTES minutes"
```

### 4. Test de récupération d'un fichier spécifique

**Objectif** : Vérifier qu'on peut récupérer des données spécifiques depuis un volume.

**Ce qu'on teste** :
- Restauration d'un volume persistant
- Accès aux fichiers
- Intégrité des données

**Quand** : Mensuel

**Durée** : 15-30 minutes

**Exemple** :

```bash
#!/bin/bash
# test-file-recovery.sh

BACKUP_NAME="database-backup"
NAMESPACE="production"
PVC_NAME="postgres-data"

echo "=== Test de récupération de fichier ==="

# 1. Créer un namespace temporaire
TEST_NS="file-recovery-test"
kubectl create namespace $TEST_NS

# 2. Restaurer le PVC
velero restore create file-recovery-$(date +%Y%m%d-%H%M%S) \
  --from-backup $BACKUP_NAME \
  --include-namespaces $NAMESPACE \
  --include-resources pvc,pv \
  --namespace-mappings $NAMESPACE:$TEST_NS

sleep 30

# 3. Créer un pod pour accéder aux données
kubectl run file-checker -n $TEST_NS \
  --image=busybox \
  --restart=Never \
  --command -- sleep 3600

# 4. Monter le volume
kubectl patch pod file-checker -n $TEST_NS -p '
{
  "spec": {
    "volumes": [{
      "name": "data",
      "persistentVolumeClaim": {"claimName": "'$PVC_NAME'"}
    }],
    "containers": [{
      "name": "file-checker",
      "volumeMounts": [{
        "name": "data",
        "mountPath": "/data"
      }]
    }]
  }
}'

sleep 10

# 5. Vérifier les fichiers
echo "Contenu du volume:"
kubectl exec -n $TEST_NS file-checker -- ls -lh /data

# 6. Nettoyage
kubectl delete namespace $TEST_NS

echo "✓ Test terminé"
```

### 5. Test de restauration de base de données

**Objectif** : Vérifier l'intégrité d'une base de données restaurée.

**Ce qu'on teste** :
- La base de données démarre
- Les données sont présentes
- Les requêtes fonctionnent
- Aucune corruption

**Quand** : Hebdomadaire pour les bases critiques

**Durée** : 20-45 minutes

**Exemple pour PostgreSQL** :

```bash
#!/bin/bash
# test-database-restore.sh

BACKUP_NAME="postgres-backup"
NAMESPACE="database"
TEST_NS="db-restore-test"

echo "=== Test de restauration PostgreSQL ==="

# 1. Créer namespace de test
kubectl create namespace $TEST_NS

# 2. Restaurer la base de données
velero restore create postgres-restore-test \
  --from-backup $BACKUP_NAME \
  --include-namespaces $NAMESPACE \
  --namespace-mappings $NAMESPACE:$TEST_NS \
  --wait

# 3. Attendre que PostgreSQL soit prêt
echo "Attente du démarrage de PostgreSQL..."
kubectl wait --for=condition=ready pod -l app=postgres -n $TEST_NS --timeout=300s

# 4. Tests basiques
POD=$(kubectl get pod -l app=postgres -n $TEST_NS -o jsonpath='{.items[0].metadata.name}')

echo "Vérification de la connexion..."
kubectl exec -n $TEST_NS $POD -- psql -U postgres -c "SELECT version();" > /dev/null
if [ $? -eq 0 ]; then
  echo "✓ Connexion OK"
else
  echo "❌ Échec de connexion"
  kubectl delete namespace $TEST_NS
  exit 1
fi

echo "Vérification des tables..."
TABLES=$(kubectl exec -n $TEST_NS $POD -- psql -U postgres -d mydb -c "\dt" | wc -l)
echo "  Tables trouvées: $TABLES"

echo "Test d'une requête..."
kubectl exec -n $TEST_NS $POD -- psql -U postgres -d mydb -c "SELECT COUNT(*) FROM users;" > /dev/null
if [ $? -eq 0 ]; then
  echo "✓ Requête OK"
else
  echo "❌ Échec de requête"
fi

# 5. Nettoyage
kubectl delete namespace $TEST_NS

echo "✓ Test de restauration BDD terminé"
```

## Environnements de test

### Option 1 : Cluster de test dédié

**Avantage** : Isolation complète, pas de risque pour la production.

**Mise en place** :

```bash
# Installer un second MicroK8s pour les tests
# (Sur une VM ou un autre serveur)

# Configuration similaire à production
microk8s enable dns storage registry

# Connecter au même stockage de backups (MinIO)
velero install \
  --provider aws \
  --bucket velero-backups \
  --prefix test-cluster \
  --secret-file ./credentials-velero \
  ...
```

### Option 2 : Namespace de test sur le même cluster

**Avantage** : Simple, pas besoin de cluster supplémentaire.

**Mise en place** :

```bash
# Créer un namespace dédié aux tests
kubectl create namespace restore-testing

# Label pour isolation
kubectl label namespace restore-testing purpose=testing
```

### Option 3 : Snapshot du cluster avant test

**Avantage** : Permet de tester destructivement puis revenir en arrière.

**Mise en place** :

```bash
# Sur un système avec ZFS
sudo zfs snapshot mypool/microk8s@before-restore-test

# Pour revenir en arrière après le test
sudo zfs rollback mypool/microk8s@before-restore-test
```

### Option 4 : Cluster éphémère

**Avantage** : Totalement propre à chaque test, pas de pollution.

**Mise en place avec Multipass** :

```bash
#!/bin/bash
# create-test-cluster.sh

# Créer une VM temporaire
multipass launch --name test-cluster --cpus 2 --mem 4G --disk 20G

# Installer MicroK8s
multipass exec test-cluster -- sudo snap install microk8s --classic

# Installer Velero et configurer le backup-location
# ... (même config que production)

echo "Cluster de test prêt"
echo "Pour détruire: multipass delete test-cluster && multipass purge"
```

## Méthodologie de test complète

Voici une approche systématique pour tester vos restaurations.

### Phase 1 : Préparation (5 minutes)

**Checklist** :

```markdown
- [ ] Identifier le backup à tester
- [ ] Vérifier que l'environnement de test est prêt
- [ ] Documenter l'état actuel (nombre de ressources, versions)
- [ ] Préparer les outils de vérification
- [ ] Informer l'équipe (si applicable)
```

**Script** :

```bash
#!/bin/bash
# prepare-test.sh

BACKUP_NAME=$1
TEST_ENV=$2  # test-cluster ou test-namespace

echo "=== Préparation du test de restauration ==="
echo "Backup: $BACKUP_NAME"
echo "Environnement: $TEST_ENV"

# Vérifier le backup
velero backup describe $BACKUP_NAME

# Documenter l'état
cat > test-report-$(date +%Y%m%d-%H%M%S).md <<EOF
# Test de restauration

**Date:** $(date)
**Backup:** $BACKUP_NAME
**Environnement:** $TEST_ENV

## État du backup

\`\`\`
$(velero backup describe $BACKUP_NAME)
\`\`\`

## Tests prévus

- [ ] Restauration des ressources
- [ ] Vérification des pods
- [ ] Tests de connectivité
- [ ] Vérification des données

EOF

echo "✓ Préparation terminée"
```

### Phase 2 : Restauration (variable)

**Actions** :

1. Lancer la restauration
2. Surveiller la progression
3. Noter les erreurs
4. Mesurer le temps

**Script** :

```bash
#!/bin/bash
# execute-restore.sh

BACKUP_NAME=$1
TEST_NS=${2:-"restore-test"}

echo "=== Exécution de la restauration ==="

START=$(date +%s)

# Créer le namespace de test
kubectl create namespace $TEST_NS --dry-run=client -o yaml | kubectl apply -f -

# Restaurer
RESTORE_NAME="test-restore-$(date +%Y%m%d-%H%M%S)"
velero restore create $RESTORE_NAME \
  --from-backup $BACKUP_NAME \
  --namespace-mappings "production:$TEST_NS"

# Suivre la progression
echo "Suivi de la restauration..."
while true; do
  STATUS=$(velero restore describe $RESTORE_NAME | grep "Phase:" | awk '{print $2}')
  echo "  Status: $STATUS"

  if [ "$STATUS" = "Completed" ] || [ "$STATUS" = "Failed" ] || [ "$STATUS" = "PartiallyFailed" ]; then
    break
  fi

  sleep 10
done

END=$(date +%s)
DURATION=$((END - START))

echo ""
echo "✓ Restauration terminée en $DURATION secondes"
echo "  Status final: $STATUS"

# Afficher les détails
velero restore describe $RESTORE_NAME --details
```

### Phase 3 : Vérification (10-30 minutes)

**Checklist de vérification** :

```markdown
## Vérifications techniques

- [ ] Tous les namespaces sont présents
- [ ] Tous les deployments sont créés
- [ ] Tous les pods sont en état Running
- [ ] Tous les services sont créés
- [ ] Les PVCs sont liés
- [ ] Les volumes contiennent des données
- [ ] Les ConfigMaps sont présents
- [ ] Les Secrets sont présents
- [ ] Les Ingress sont configurés

## Vérifications fonctionnelles

- [ ] Les applications web répondent
- [ ] Les bases de données sont accessibles
- [ ] Les APIs fonctionnent
- [ ] Les données sont cohérentes
- [ ] Pas de corruption visible

## Vérifications de performance

- [ ] Temps de démarrage acceptable
- [ ] Ressources CPU/RAM normales
- [ ] Pas de redémarrages répétés
```

**Script automatisé** :

```bash
#!/bin/bash
# verify-restore.sh

TEST_NS=$1

echo "=== Vérification de la restauration ==="
echo "Namespace: $TEST_NS"

ERRORS=0

# 1. Vérifier les pods
echo -e "\n1. Vérification des pods"
TOTAL_PODS=$(kubectl get pods -n $TEST_NS --no-headers 2>/dev/null | wc -l)
RUNNING_PODS=$(kubectl get pods -n $TEST_NS --field-selector=status.phase=Running --no-headers 2>/dev/null | wc -l)

echo "  Pods running: $RUNNING_PODS/$TOTAL_PODS"

if [ $RUNNING_PODS -eq $TOTAL_PODS ]; then
  echo "  ✓ Tous les pods sont en running"
else
  echo "  ❌ Des pods ne sont pas en running"
  ERRORS=$((ERRORS + 1))
  kubectl get pods -n $TEST_NS | grep -v Running
fi

# 2. Vérifier les PVCs
echo -e "\n2. Vérification des PVCs"
TOTAL_PVC=$(kubectl get pvc -n $TEST_NS --no-headers 2>/dev/null | wc -l)
BOUND_PVC=$(kubectl get pvc -n $TEST_NS --field-selector=status.phase=Bound --no-headers 2>/dev/null | wc -l)

echo "  PVCs bound: $BOUND_PVC/$TOTAL_PVC"

if [ $BOUND_PVC -eq $TOTAL_PVC ]; then
  echo "  ✓ Tous les PVCs sont liés"
else
  echo "  ❌ Des PVCs ne sont pas liés"
  ERRORS=$((ERRORS + 1))
fi

# 3. Vérifier les services
echo -e "\n3. Vérification des services"
SERVICES=$(kubectl get services -n $TEST_NS --no-headers 2>/dev/null | wc -l)
echo "  Services: $SERVICES"

if [ $SERVICES -gt 0 ]; then
  echo "  ✓ Services présents"
else
  echo "  ⚠ Aucun service trouvé"
fi

# 4. Vérifier les ConfigMaps
echo -e "\n4. Vérification des ConfigMaps"
CMS=$(kubectl get configmaps -n $TEST_NS --no-headers 2>/dev/null | wc -l)
echo "  ConfigMaps: $CMS"

# 5. Test de connectivité (si applicable)
echo -e "\n5. Test de connectivité"
if kubectl get service -n $TEST_NS webapp &>/dev/null; then
  POD=$(kubectl get pod -n $TEST_NS -l app=webapp -o jsonpath='{.items[0].metadata.name}' 2>/dev/null)
  if [ -n "$POD" ]; then
    kubectl exec -n $TEST_NS $POD -- wget -q -O- http://localhost:80 &>/dev/null
    if [ $? -eq 0 ]; then
      echo "  ✓ Application web répond"
    else
      echo "  ❌ Application web ne répond pas"
      ERRORS=$((ERRORS + 1))
    fi
  fi
fi

# Résumé
echo -e "\n=== Résumé ==="
if [ $ERRORS -eq 0 ]; then
  echo "✓ Tous les tests ont réussi"
  exit 0
else
  echo "❌ $ERRORS erreurs détectées"
  exit 1
fi
```

### Phase 4 : Nettoyage (2-5 minutes)

**Actions** :

```bash
#!/bin/bash
# cleanup-test.sh

TEST_NS=$1

echo "=== Nettoyage après test ==="

# Sauvegarder les logs si nécessaire
mkdir -p ./test-logs
kubectl get all -n $TEST_NS -o yaml > ./test-logs/final-state.yaml

# Supprimer le namespace de test
kubectl delete namespace $TEST_NS --wait

echo "✓ Nettoyage terminé"
```

### Phase 5 : Documentation (5-10 minutes)

**Compléter le rapport** :

```bash
#!/bin/bash
# finalize-report.sh

REPORT_FILE=$1
SUCCESS=$2  # "pass" ou "fail"

cat >> $REPORT_FILE <<EOF

## Résultats

**Status:** $SUCCESS
**Date de fin:** $(date)

### Observations

- Temps de restauration: XX minutes
- Pods restaurés: XX/XX
- Problèmes rencontrés:
  - [ Liste des problèmes ]

### Actions correctives

- [ Liste des actions à prendre ]

### Prochains tests

- Date du prochain test: [date]

---

**Testeur:** $(whoami)
**Signature:** _______________
EOF

echo "✓ Rapport complété: $REPORT_FILE"
```

## Plan de tests réguliers

### Calendrier recommandé

**Pour un lab personnel** :

```
Quotidien:
  - Validation automatique des backups

Hebdomadaire:
  - Test de restauration partielle (une application)

Mensuel:
  - Test de restauration complète
  - Test de restauration de base de données

Trimestriel:
  - Test de disaster recovery complet
  - Simulation de migration vers nouveau cluster

Annuel:
  - Audit complet de la stratégie de sauvegarde
  - Test avec toute l'équipe (si applicable)
```

**Pour un environnement de production** :

```
Quotidien:
  - Validation automatique des backups
  - Alertes sur échecs

Hebdomadaire:
  - Test de restauration partielle
  - Test de récupération de fichiers

Bi-hebdomadaire:
  - Test de restauration de BDD

Mensuel:
  - Test de restauration complète
  - Test de failover
  - Revue des métriques

Trimestriel:
  - Exercice de disaster recovery
  - Test de RTO/RPO
  - Mise à jour de la documentation

Annuel:
  - Audit externe (si requis)
  - Formation de l'équipe
```

### Script de planification

```bash
#!/bin/bash
# schedule-tests.sh

# Créer des CronJobs pour automatiser les tests

cat <<EOF | kubectl apply -f -
---
# Validation quotidienne des backups
apiVersion: batch/v1
kind: CronJob
metadata:
  name: daily-backup-validation
  namespace: velero
spec:
  schedule: "0 3 * * *"  # 3h du matin
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: validator
            image: bitnami/kubectl:latest
            command:
            - /bin/bash
            - -c
            - |
              # Valider le dernier backup
              LATEST=\$(velero backup get --output=name | sort | tail -1)
              STATUS=\$(velero backup describe \$LATEST | grep "Phase:" | awk '{print \$2}')

              if [ "\$STATUS" != "Completed" ]; then
                echo "❌ ALERTE: Backup \$LATEST en échec!"
                # Envoyer une notification (webhook, email, etc.)
                exit 1
              fi

              echo "✓ Backup validé: \$LATEST"
          restartPolicy: OnFailure
---
# Test de restauration hebdomadaire
apiVersion: batch/v1
kind: CronJob
metadata:
  name: weekly-restore-test
  namespace: velero
spec:
  schedule: "0 2 * * 0"  # Dimanche à 2h
  jobTemplate:
    spec:
      template:
        spec:
          serviceAccountName: velero
          containers:
          - name: restore-tester
            image: bitnami/kubectl:latest
            command:
            - /bin/bash
            - -c
            - |
              # Test de restauration partielle
              echo "=== Test de restauration hebdomadaire ==="

              # Créer namespace de test
              kubectl create namespace restore-test-weekly

              # Restaurer une application
              LATEST=\$(velero backup get --output=name | sort | tail -1)
              velero restore create weekly-test-\$(date +%Y%m%d) \\
                --from-backup \$LATEST \\
                --include-namespaces webapp \\
                --namespace-mappings webapp:restore-test-weekly

              # Attendre et vérifier
              sleep 60
              PODS=\$(kubectl get pods -n restore-test-weekly --field-selector=status.phase=Running --no-headers | wc -l)

              # Nettoyage
              kubectl delete namespace restore-test-weekly

              echo "✓ Test terminé - \$PODS pods ont démarré"
          restartPolicy: OnFailure
EOF

echo "✓ Tests automatiques planifiés"
```

## Métriques et KPIs

### Métriques à suivre

**Temps de restauration (RTO - Recovery Time Objective)** :

```bash
#!/bin/bash
# measure-rto.sh

echo "=== Mesure du RTO ==="

BACKUP_NAME=$1
START=$(date +%s)

# Restauration
velero restore create rto-test \
  --from-backup $BACKUP_NAME \
  --wait

END=$(date +%s)
RTO=$((END - START))
RTO_MINUTES=$((RTO / 60))

echo "RTO mesuré: $RTO_MINUTES minutes"

# Enregistrer dans un fichier
echo "$(date),$BACKUP_NAME,$RTO_MINUTES" >> rto-metrics.csv

# Si dépasse l'objectif, alerter
RTO_OBJECTIVE=120  # 2 heures
if [ $RTO_MINUTES -gt $RTO_OBJECTIVE ]; then
  echo "⚠ ALERTE: RTO dépasse l'objectif ($RTO_OBJECTIVE min)"
fi
```

**Taux de réussite des restaurations** :

```bash
#!/bin/bash
# calculate-success-rate.sh

TOTAL=$(velero restore get --output=json | jq '.items | length')
COMPLETED=$(velero restore get --output=json | jq '[.items[] | select(.status.phase=="Completed")] | length')

SUCCESS_RATE=$((COMPLETED * 100 / TOTAL))

echo "Taux de réussite: $SUCCESS_RATE%"
echo "  Completed: $COMPLETED"
echo "  Total: $TOTAL"

if [ $SUCCESS_RATE -lt 95 ]; then
  echo "⚠ ALERTE: Taux de réussite trop faible"
fi
```

**Intégrité des données** :

```bash
#!/bin/bash
# verify-data-integrity.sh

echo "=== Vérification de l'intégrité des données ==="

# Créer un checksum avant backup
kubectl exec -n production postgres-0 -- \
  psql -U postgres -d mydb -c "SELECT COUNT(*), SUM(id) FROM users;" \
  > /tmp/before.txt

# Restaurer dans un namespace de test
# ... (restauration)

# Créer un checksum après restore
kubectl exec -n restore-test postgres-0 -- \
  psql -U postgres -d mydb -c "SELECT COUNT(*), SUM(id) FROM users;" \
  > /tmp/after.txt

# Comparer
if diff /tmp/before.txt /tmp/after.txt; then
  echo "✓ Données identiques"
else
  echo "❌ Différence détectée dans les données"
fi
```

### Dashboard de suivi

Créez un fichier de tracking :

```csv
Date,Type de test,Backup,Durée (min),Résultat,Notes
2025-01-15,Validation,daily-backup,1,Pass,
2025-01-14,Partiel,daily-backup,15,Pass,
2025-01-10,Complet,weekly-backup,120,Pass,
2025-01-08,BDD,postgres-backup,25,Fail,Timeout connexion
```

Script de génération de rapport :

```bash
#!/bin/bash
# generate-report.sh

cat <<EOF
# Rapport de tests de restauration

**Période:** $(date -d '30 days ago' +%Y-%m-%d) à $(date +%Y-%m-%d)

## Statistiques

\`\`\`
$(awk -F',' 'NR>1 {
  total++;
  if($5=="Pass") pass++
}
END {
  print "Total tests: " total;
  print "Réussis: " pass;
  print "Taux de réussite: " (pass*100/total) "%"
}' test-tracking.csv)
\`\`\`

## Tests par type

\`\`\`
$(awk -F',' 'NR>1 {count[$2]++} END {for (type in count) print type ": " count[type]}' test-tracking.csv)
\`\`\`

## Durée moyenne

\`\`\`
$(awk -F',' 'NR>1 {
  if($2=="Complet") {sum+=$4; count++}
}
END {
  if(count>0) print "Restauration complète: " (sum/count) " minutes"
}' test-tracking.csv)
\`\`\`

## Problèmes identifiés

$(awk -F',' 'NR>1 && $5=="Fail" {print "- " $1 ": " $6}' test-tracking.csv)

EOF
```

## Troubleshooting pendant les tests

### Problème 1 : Restauration bloquée

**Symptômes** : La restauration reste en "InProgress" indéfiniment.

**Diagnostic** :

```bash
# Vérifier les logs Velero
kubectl logs -n velero deployment/velero

# Vérifier les logs du restore
velero restore logs restore-name

# Vérifier les events
kubectl get events -n target-namespace --sort-by='.lastTimestamp'
```

**Solutions courantes** :

```bash
# Supprimer et recommencer
velero restore delete problematic-restore
velero restore create new-attempt --from-backup same-backup

# Vérifier les ressources du node
kubectl top nodes

# Vérifier si c'est un problème de PVC
kubectl get pvc -n target-namespace
```

### Problème 2 : Pods en CrashLoopBackOff

**Diagnostic** :

```bash
# Voir les logs du pod
kubectl logs -n test-namespace pod-name --previous

# Décrire le pod
kubectl describe pod -n test-namespace pod-name

# Vérifier les volumes
kubectl get pvc -n test-namespace
```

**Solutions** :

```bash
# Vérifier les ConfigMaps/Secrets
kubectl get configmaps,secrets -n test-namespace

# Vérifier les dépendances (BDD, services externes)
kubectl get services -n test-namespace

# Augmenter les ressources temporairement
kubectl patch deployment app-name -n test-namespace -p '
{
  "spec": {
    "template": {
      "spec": {
        "containers": [{
          "name": "container-name",
          "resources": {
            "requests": {
              "memory": "512Mi",
              "cpu": "500m"
            }
          }
        }]
      }
    }
  }
}'
```

### Problème 3 : Données manquantes après restauration

**Diagnostic** :

```bash
# Vérifier si les volumes ont été restaurés
velero restore describe restore-name --details | grep "Restic"

# Vérifier le contenu d'un volume
kubectl run -n test-namespace debug-pod --rm -it --image=busybox -- sh
# Dans le pod:
ls -la /data
```

**Solutions** :

```bash
# Vérifier que --default-volumes-to-fs-backup était activé
velero backup describe backup-name

# Relancer avec restauration des volumes explicite
velero restore create new-restore \
  --from-backup backup-name \
  --restore-volumes=true
```

### Problème 4 : Erreurs de permissions

**Diagnostic** :

```bash
# Vérifier les ServiceAccounts
kubectl get serviceaccounts -n test-namespace

# Vérifier les Roles/RoleBindings
kubectl get roles,rolebindings -n test-namespace
```

**Solutions** :

```bash
# Restaurer aussi les ressources RBAC
velero restore create rbac-restore \
  --from-backup backup-name \
  --include-resources serviceaccounts,roles,rolebindings

# Ou créer manuellement
kubectl create serviceaccount app-sa -n test-namespace
```

## Documentation des tests

### Template de rapport de test

```markdown
# Rapport de test de restauration

**Date:** [date]
**Testeur:** [nom]
**Type de test:** [Validation/Partiel/Complet/BDD]

## Informations du backup

- **Nom:** [backup-name]
- **Date de création:** [date]
- **Taille:** [size]
- **Ressources:** [count]

## Environnement de test

- **Cluster:** [test/production]
- **Namespace:** [namespace]
- **Version MicroK8s:** [version]

## Procédure suivie

1. [Étape 1]
2. [Étape 2]
3. [Étape 3]

## Résultats

### Métriques

- **Durée de restauration:** [XX] minutes
- **Pods restaurés:** [XX/XX]
- **PVCs liés:** [XX/XX]
- **Services créés:** [XX]

### Tests fonctionnels

- [ ] Application web accessible
- [ ] Base de données répond
- [ ] Données intactes
- [ ] Pas de corruption

## Problèmes rencontrés

1. [Problème 1] - [Résolution]
2. [Problème 2] - [Résolution]

## Actions correctives

- [ ] [Action 1]
- [ ] [Action 2]

## Conclusion

**Status:** ✓ PASS / ❌ FAIL

**Recommandations:**
- [Recommandation 1]
- [Recommandation 2]

**Prochain test:** [date]

---

**Signature:** _______________
```

### Script de génération de rapport

```bash
#!/bin/bash
# generate-test-report.sh

BACKUP_NAME=$1
TEST_TYPE=$2
RESULT=$3  # pass/fail

REPORT_FILE="test-report-$(date +%Y%m%d-%H%M%S).md"

cat > $REPORT_FILE <<EOF
# Rapport de test de restauration

**Date:** $(date)
**Testeur:** $(whoami)
**Type de test:** $TEST_TYPE
**Résultat:** $RESULT

## Backup testé

\`\`\`
$(velero backup describe $BACKUP_NAME)
\`\`\`

## État du cluster après restauration

### Namespaces
\`\`\`
$(kubectl get namespaces)
\`\`\`

### Pods
\`\`\`
$(kubectl get pods --all-namespaces | grep -v kube-system)
\`\`\`

### Services
\`\`\`
$(kubectl get services --all-namespaces | grep -v kube-system)
\`\`\`

## Logs de restauration

\`\`\`
$(velero restore logs latest --namespace velero)
\`\`\`

## Conclusion

**Status:** $RESULT

---
EOF

echo "Rapport généré: $REPORT_FILE"
```

## Checklist complète de test

```markdown
# Checklist de test de restauration

## Avant le test

- [ ] Choisir le backup à tester
- [ ] Préparer l'environnement de test
- [ ] Vérifier que le backup est Completed
- [ ] Documenter l'état actuel
- [ ] Prévenir l'équipe (si applicable)

## Pendant le test

- [ ] Démarrer le chronomètre
- [ ] Lancer la restauration
- [ ] Surveiller les logs Velero
- [ ] Noter toute erreur ou warning
- [ ] Vérifier la progression régulièrement

## Après restauration

- [ ] Vérifier tous les namespaces
- [ ] Compter les pods (total vs running)
- [ ] Vérifier les PVCs (total vs bound)
- [ ] Tester les services
- [ ] Vérifier les données
- [ ] Tester la connectivité
- [ ] Mesurer le temps total

## Vérifications fonctionnelles

- [ ] Application web répond
- [ ] API accessible
- [ ] Base de données fonctionne
- [ ] Requêtes SQL OK
- [ ] Fichiers présents dans les volumes
- [ ] Pas de corruption détectée

## Nettoyage

- [ ] Sauvegarder les logs
- [ ] Exporter l'état final
- [ ] Supprimer le namespace de test
- [ ] Nettoyer les ressources temporaires

## Documentation

- [ ] Compléter le rapport de test
- [ ] Enregistrer les métriques
- [ ] Noter les problèmes rencontrés
- [ ] Lister les actions correctives
- [ ] Planifier le prochain test
- [ ] Archiver tous les documents

## Suivi

- [ ] Implémenter les corrections
- [ ] Mettre à jour la documentation
- [ ] Informer l'équipe des résultats
- [ ] Planifier le test suivant
```

## Conclusion

Les tests de restauration sont le pilier d'une stratégie de sauvegarde efficace. Sans tests réguliers, vous travaillez avec un faux sentiment de sécurité.

### Points clés à retenir

✅ **Tester régulièrement** : Minimum mensuel, idéalement hebdomadaire
✅ **Varier les tests** : Validation, partiel, complet, spécifique
✅ **Documenter tout** : Chaque test doit générer un rapport
✅ **Mesurer les métriques** : RTO, taux de réussite, intégrité
✅ **Automatiser** : Scripts et CronJobs pour tests récurrents
✅ **Corriger rapidement** : Chaque problème détecté = action immédiate
✅ **Former l'équipe** : Tout le monde doit savoir restaurer

### Les 3 règles d'or

1. **Une sauvegarde non testée n'est pas une sauvegarde**
2. **Testez avant d'en avoir besoin**
3. **Documentez comme si quelqu'un d'autre devait restaurer**

### Prochaines étapes

Dans la section suivante (22.7), nous verrons comment créer un plan de reprise d'activité (DRP - Disaster Recovery Plan) complet qui intègre tous les aspects de sauvegarde et de restauration dans une stratégie globale.

---

**Checklist finale** :

- [ ] Définir une fréquence de tests adaptée
- [ ] Créer des scripts de test automatisés
- [ ] Mettre en place un environnement de test
- [ ] Établir des métriques de référence (RTO, RPO)
- [ ] Documenter la procédure de test
- [ ] Former les personnes concernées
- [ ] Planifier les tests dans un calendrier
- [ ] Créer des alertes sur échecs
- [ ] Maintenir un journal des tests
- [ ] Réviser et améliorer régulièrement

⏭️ [Plan de reprise d'activité (DRP)](/22-sauvegarde-et-restauration/07-plan-de-reprise-dactivite-drp.md)
