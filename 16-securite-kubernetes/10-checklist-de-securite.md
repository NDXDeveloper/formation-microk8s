🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 16.10 Checklist de Sécurité

## Introduction

Cette checklist finale récapitule tous les éléments de sécurité que nous avons explorés dans ce chapitre. C'est un outil pratique que vous pouvez utiliser régulièrement pour évaluer et améliorer la sécurité de votre cluster Kubernetes.

**Analogie simple :** Avant de partir en voyage, vous utilisez une checklist pour ne rien oublier : passeport, billets, médicaments, etc. Cette checklist de sécurité fonctionne de la même manière : elle vous assure que vous n'avez oublié aucun élément critique pour la sécurité de votre cluster.

## Comment Utiliser Cette Checklist

### Légende des Priorités

```
🔴 CRITIQUE (P0) : À implémenter immédiatement
    - Risque de compromission majeure
    - Impact direct sur la sécurité

🟠 IMPORTANT (P1) : À implémenter dans les 2 semaines
    - Amélioration significative de la sécurité
    - Requis pour la production

🟡 RECOMMANDÉ (P2) : À implémenter dans les 1-2 mois
    - Bonnes pratiques standards
    - Défense en profondeur

🟢 OPTIONNEL (P3) : Nice to have
    - Optimisations avancées
    - Conformité stricte
```

### Niveaux de Maturité

```
Niveau 1 - Basique (0-40% complétés)
└─ Sécurité minimale, acceptable pour lab/dev

Niveau 2 - Intermédiaire (41-70% complétés)
└─ Sécurité correcte, acceptable pour staging

Niveau 3 - Avancé (71-90% complétés)
└─ Sécurité robuste, recommandé pour production

Niveau 4 - Expert (91-100% complétés)
└─ Sécurité maximale, conformité enterprise
```

### Fréquence de Vérification

```
📅 Quotidienne : Items critiques et monitoring
📅 Hebdomadaire : Revue des alertes et incidents
📅 Mensuelle : Audit complet de cette checklist
📅 Trimestrielle : Tests de sécurité approfondis
📅 Annuelle : Audit externe et certification
```

## Checklist par Domaine

### 1. Infrastructure et Cluster 🏗️

#### 1.1 Versions et Mises à Jour

- [ ] 🔴 Kubernetes est sur une version supportée (N, N-1, ou N-2)
  ```bash
  kubectl version --short
  # Version du serveur doit être >= 1.26
  ```

- [ ] 🟠 Plan de mise à jour trimestriel défini
  ```
  Prochaine mise à jour planifiée : [DATE]
  Version cible : [VERSION]
  ```

- [ ] 🟡 OS des nœuds à jour avec derniers patches de sécurité
  ```bash
  # Sur chaque nœud
  apt list --upgradable
  # Ou
  yum check-update
  ```

- [ ] 🟡 Composants addons à jour (CNI, CSI, Ingress)
  ```bash
  microk8s status
  # Vérifier les versions des addons
  ```

#### 1.2 Isolation et Hardening

- [ ] 🔴 Cluster isolé du réseau public (firewall configuré)
  ```bash
  # Vérifier les règles firewall
  sudo ufw status
  # Ou
  sudo iptables -L
  ```

- [ ] 🟠 API Server accessible uniquement depuis IPs autorisées
  ```yaml
  # /etc/kubernetes/manifests/kube-apiserver.yaml
  --authorization-mode=Node,RBAC
  ```

- [ ] 🟡 Environnements séparés (dev/staging/prod)
  ```bash
  # Clusters séparés OU namespaces isolés
  kubectl get namespaces
  ```

- [ ] 🟡 Nœuds hardened (services inutiles désactivés, SSH sécurisé)
  ```bash
  # SSH config
  cat /etc/ssh/sshd_config | grep -E 'PermitRootLogin|PasswordAuthentication'
  # Doit afficher :
  # PermitRootLogin no
  # PasswordAuthentication no
  ```

- [ ] 🟢 Chiffrement réseau entre nœuds (avec mTLS/IPSec)

#### 1.3 Sauvegardes

- [ ] 🔴 Sauvegardes automatiques configurées
  ```bash
  # Avec Velero par exemple
  velero backup get
  # Doit montrer des backups récents
  ```

- [ ] 🔴 Sauvegarde testée et restauration validée
  ```
  Dernier test de restauration : [DATE]
  Résultat : [SUCCÈS/ÉCHEC]
  Durée : [TEMPS]
  ```

- [ ] 🟠 Rétention des sauvegardes définie et appliquée
  ```
  Rétention production : 90 jours
  Rétention staging : 30 jours
  Rétention dev : 7 jours
  ```

- [ ] 🟡 Sauvegardes stockées hors du cluster
  ```
  Emplacement : [S3/GCS/Azure Blob/NAS]
  ```

- [ ] 🟡 Plan de reprise d'activité (DRP) documenté
  ```
  RTO (Recovery Time Objective) : [TEMPS]
  RPO (Recovery Point Objective) : [TEMPS]
  ```

### 2. Contrôle d'Accès (RBAC) 🔐

#### 2.1 Utilisateurs et Authentification

- [ ] 🔴 RBAC activé (mode par défaut)
  ```bash
  kubectl api-versions | grep rbac
  # Doit afficher : rbac.authorization.k8s.io/v1
  ```

- [ ] 🔴 Aucun utilisateur ne possède cluster-admin sauf urgence
  ```bash
  kubectl get clusterrolebindings -o json | \
    jq '.items[] | select(.roleRef.name=="cluster-admin") | .subjects'
  # Liste doit être minimale
  ```

- [ ] 🟠 Authentification forte activée (certificats, OIDC, LDAP)
  ```
  Méthode d'authentification : [TYPE]
  MFA activée : [OUI/NON]
  ```

- [ ] 🟡 Groupes utilisés au lieu d'utilisateurs individuels
  ```bash
  kubectl get rolebindings,clusterrolebindings --all-namespaces | grep Group
  ```

#### 2.2 ServiceAccounts

- [ ] 🔴 ServiceAccount dédié pour chaque application
  ```bash
  # Vérifier qu'aucun pod n'utilise "default"
  kubectl get pods --all-namespaces -o json | \
    jq '.items[] | select(.spec.serviceAccountName=="default") | .metadata.name'
  # Liste doit être vide
  ```

- [ ] 🟠 Permissions RBAC minimales pour chaque ServiceAccount
  ```bash
  # Auditer les permissions
  kubectl auth can-i --list --as system:serviceaccount:default:my-app
  ```

- [ ] 🟡 Auto-mount des tokens désactivé si inutile
  ```yaml
  # Dans ServiceAccount ou Pod
  automountServiceAccountToken: false
  ```

#### 2.3 Audit des Permissions

- [ ] 🟠 Revue mensuelle des RoleBindings et ClusterRoleBindings
  ```
  Dernière revue : [DATE]
  Anomalies détectées : [NOMBRE]
  Actions prises : [DESCRIPTION]
  ```

- [ ] 🟡 Documentation des rôles personnalisés
  ```bash
  kubectl get roles,clusterroles --all-namespaces | grep -v "system:"
  # Chaque rôle custom doit être documenté
  ```

### 3. Sécurité des Workloads (Pods) 🛡️

#### 3.1 Pod Security Standards

- [ ] 🔴 Pod Security Standards activé sur tous les namespaces de production
  ```bash
  kubectl get namespaces -o json | \
    jq '.items[] | {name: .metadata.name, enforce: .metadata.labels["pod-security.kubernetes.io/enforce"]}'
  # Production doit avoir "restricted"
  ```

- [ ] 🟠 Niveau "restricted" appliqué en production
  ```bash
  kubectl label namespace production \
    pod-security.kubernetes.io/enforce=restricted --overwrite
  ```

- [ ] 🟡 Niveau "baseline" minimum en staging
  ```bash
  kubectl label namespace staging \
    pod-security.kubernetes.io/enforce=baseline --overwrite
  ```

#### 3.2 Security Context

- [ ] 🔴 Aucun pod ne s'exécute en root
  ```bash
  # Vérifier tous les pods
  kubectl get pods --all-namespaces -o json | \
    jq '.items[] | select(.spec.securityContext.runAsNonRoot != true) | .metadata.name'
  # Liste doit être vide en production
  ```

- [ ] 🔴 allowPrivilegeEscalation=false sur tous les conteneurs
  ```bash
  kubectl get pods --all-namespaces -o json | \
    jq '.items[].spec.containers[] | select(.securityContext.allowPrivilegeEscalation != false)'
  ```

- [ ] 🟠 capabilities drop ALL appliqué
  ```bash
  kubectl get pods -n production -o json | \
    jq '.items[].spec.containers[].securityContext.capabilities'
  # Doit montrer "drop: [ALL]"
  ```

- [ ] 🟡 readOnlyRootFilesystem=true quand possible
  ```bash
  # Compter les pods avec RO filesystem
  kubectl get pods --all-namespaces -o json | \
    jq '[.items[].spec.containers[] | select(.securityContext.readOnlyRootFilesystem == true)] | length'
  ```

- [ ] 🟡 seccompProfile RuntimeDefault appliqué
  ```bash
  kubectl get pods -n production -o json | \
    jq '.items[].spec.securityContext.seccompProfile'
  ```

#### 3.3 Ressources

- [ ] 🔴 Resource requests et limits définis sur tous les pods
  ```bash
  # Pods sans limits
  kubectl get pods --all-namespaces -o json | \
    jq '.items[] | select(.spec.containers[].resources.limits == null) | .metadata.name'
  # Liste doit être vide en production
  ```

- [ ] 🟠 ResourceQuotas définis par namespace
  ```bash
  kubectl get resourcequota --all-namespaces
  # Au moins un quota par namespace de production
  ```

- [ ] 🟡 LimitRanges configurés
  ```bash
  kubectl get limitrange --all-namespaces
  ```

#### 3.4 Probes et Health

- [ ] 🟠 Liveness probes configurées
  ```bash
  kubectl get pods --all-namespaces -o json | \
    jq '.items[] | select(.spec.containers[].livenessProbe == null) | .metadata.name'
  ```

- [ ] 🟠 Readiness probes configurées
  ```bash
  kubectl get pods --all-namespaces -o json | \
    jq '.items[] | select(.spec.containers[].readinessProbe == null) | .metadata.name'
  ```

### 4. Isolation Réseau 🌐

#### 4.1 Network Policies

- [ ] 🔴 Network Policies supportées (CNI compatible)
  ```bash
  # Avec MicroK8s
  microk8s status | grep -E 'calico|cilium'
  # Doit montrer que Calico ou Cilium est activé
  ```

- [ ] 🔴 Default deny policy appliquée sur production
  ```bash
  kubectl get networkpolicy -n production
  # Doit inclure une policy "default-deny-all"
  ```

- [ ] 🟠 DNS autorisé explicitement
  ```bash
  # Vérifier qu'une policy permet DNS
  kubectl get networkpolicy -n production -o yaml | grep -A 5 'port: 53'
  ```

- [ ] 🟠 Whitelist explicite des flux réseau
  ```
  Flux documentés : [NOMBRE]
  Flux autorisés : [NOMBRE]
  Dernière revue : [DATE]
  ```

#### 4.2 Isolation

- [ ] 🟠 Communication cross-namespace bloquée par défaut
  ```bash
  # Tester la connectivité
  kubectl run test -n dev --image=busybox --rm -it -- \
    wget -O- http://service.production.svc.cluster.local
  # Devrait échouer
  ```

- [ ] 🟡 Egress vers Internet limité
  ```bash
  kubectl get networkpolicy --all-namespaces -o yaml | \
    grep -A 10 'policyTypes.*Egress'
  ```

- [ ] 🟡 Pods exposés publiquement identifiés et protégés
  ```bash
  kubectl get ingress --all-namespaces
  # Vérifier chaque ingress
  ```

### 5. Images et Conteneurs 📦

#### 5.1 Scan de Vulnérabilités

- [ ] 🔴 Scanner d'images intégré dans CI/CD
  ```bash
  # Dans GitLab CI, GitHub Actions, etc.
  # Exemple :
  # trivy image --exit-code 1 --severity CRITICAL $IMAGE
  ```

- [ ] 🔴 Politique de blocage sur vulnérabilités CRITICAL
  ```
  Outil de scan : [Trivy/Grype/Snyk]
  Seuil de blocage : CRITICAL
  Exceptions documentées : [NOMBRE]
  ```

- [ ] 🟠 Scan de toutes les images en production mensuellement
  ```
  Dernier scan complet : [DATE]
  Images vulnérables : [NOMBRE]
  Plan de correction : [OUI/NON]
  ```

- [ ] 🟡 Scan automatique dans le registry
  ```
  Registry : [Harbor/ECR/GCR]
  Scan activé : [OUI/NON]
  ```

#### 5.2 Provenance et Intégrité

- [ ] 🔴 Images provenant uniquement de sources connues
  ```bash
  # Lister toutes les images
  kubectl get pods --all-namespaces -o json | \
    jq -r '.items[].spec.containers[].image' | sort -u
  # Vérifier chaque source
  ```

- [ ] 🟠 Tags spécifiques utilisés (pas :latest)
  ```bash
  kubectl get pods --all-namespaces -o json | \
    jq '.items[].spec.containers[] | select(.image | contains(":latest"))'
  # Liste doit être vide
  ```

- [ ] 🟡 Images signées (Cosign/Notary)
  ```bash
  # Vérifier la signature
  cosign verify --key cosign.pub $IMAGE
  ```

- [ ] 🟡 SBOM (Software Bill of Materials) généré
  ```bash
  trivy image --format cyclonedx $IMAGE > sbom.json
  ```

#### 5.3 Construction d'Images

- [ ] 🟠 Images minimales utilisées (Alpine/Distroless)
  ```bash
  # Compter les images non-minimales
  kubectl get pods --all-namespaces -o json | \
    jq -r '.items[].spec.containers[].image' | \
    grep -v -E 'alpine|distroless' | wc -l
  ```

- [ ] 🟡 Multi-stage builds utilisés
  ```dockerfile
  # Vérifier les Dockerfiles
  FROM golang:1.21 AS builder  # Stage 1
  FROM gcr.io/distroless/static  # Stage 2
  ```

- [ ] 🟡 Aucun secret dans les images
  ```bash
  trivy image --scanners secret $IMAGE
  # Aucun secret détecté
  ```

### 6. Secrets Management 🔒

#### 6.1 Stockage des Secrets

- [ ] 🔴 Aucun secret en clair dans Git
  ```bash
  # Scanner le repository
  git secrets --scan
  # Ou
  truffleHog git file://. --json
  ```

- [ ] 🔴 Solution de secrets management implémentée
  ```
  Solution : [Sealed Secrets/External Secrets/Vault]
  Date d'implémentation : [DATE]
  Coverage : [POURCENTAGE]
  ```

- [ ] 🟠 Secrets chiffrés dans etcd
  ```bash
  # Vérifier la configuration
  cat /etc/kubernetes/manifests/kube-apiserver.yaml | \
    grep encryption-provider-config
  ```

- [ ] 🟡 Rotation automatique des secrets
  ```
  Fréquence de rotation : [JOURS]
  Dernière rotation : [DATE]
  Prochaine rotation : [DATE]
  ```

#### 6.2 Accès aux Secrets

- [ ] 🔴 RBAC stricte sur l'accès aux secrets
  ```bash
  # Qui peut lister tous les secrets ?
  kubectl-who-can list secrets --all-namespaces
  # Liste doit être minimale
  ```

- [ ] 🟠 Un secret par service/composant
  ```bash
  kubectl get secrets --all-namespaces | wc -l
  # Vs nombre d'applications
  ```

- [ ] 🟡 Secrets montés en volumes (pas en env vars)
  ```yaml
  # Préférer :
  volumeMounts:
  - name: secret
    mountPath: /secrets
  # Au lieu de :
  env:
  - name: PASSWORD
    valueFrom:
      secretKeyRef: ...
  ```

### 7. Monitoring et Observabilité 📊

#### 7.1 Audit Logging

- [ ] 🔴 Audit logging activé
  ```bash
  # Vérifier les logs d'audit
  sudo ls -lh /var/log/kubernetes/audit.log
  # Fichier doit exister et être récent
  ```

- [ ] 🟠 Politique d'audit configurée (Metadata minimum)
  ```bash
  cat /etc/kubernetes/audit-policy.yaml
  # Doit contenir des règles
  ```

- [ ] 🟠 Logs d'audit centralisés (Elasticsearch/Splunk)
  ```
  Destination : [SYSTÈME]
  Rétention : [JOURS]
  ```

- [ ] 🟡 Alertes sur événements critiques
  ```
  Alertes configurées : [NOMBRE]
  Dernière alerte : [DATE]
  ```

#### 7.2 Monitoring de Sécurité

- [ ] 🟠 Prometheus installé et configuré
  ```bash
  kubectl get pods -n monitoring | grep prometheus
  ```

- [ ] 🟠 Grafana avec dashboards de sécurité
  ```
  Dashboards :
  - [ ] Authentifications échouées
  - [ ] Changements RBAC
  - [ ] Vulnérabilités d'images
  - [ ] Ressources anormales
  ```

- [ ] 🟡 Runtime security installé (Falco)
  ```bash
  kubectl get pods -n falco
  ```

- [ ] 🟡 Alerting configuré (Alertmanager)
  ```bash
  kubectl get pods -n monitoring | grep alertmanager
  ```

#### 7.3 Logs et Traces

- [ ] 🟠 Logs centralisés (EFK/ELK)
  ```
  Stack : [EFK/ELK/Loki]
  Rétention : [JOURS]
  ```

- [ ] 🟡 Corrélation logs-métriques possible
  ```
  Outil : [Grafana/Kibana]
  ```

- [ ] 🟡 Distributed tracing (Jaeger/Zipkin)
  ```bash
  kubectl get pods -n tracing
  ```

### 8. Maintenance et Mise à Jour 🔄

#### 8.1 Patch Management

- [ ] 🔴 Processus de patching documenté
  ```
  Processus documenté : [OUI/NON]
  Emplacement : [URL/CHEMIN]
  ```

- [ ] 🟠 SLA de patching définis
  ```
  CRITICAL : [HEURES]
  HIGH : [JOURS]
  MEDIUM : [JOURS]
  LOW : [JOURS]
  ```

- [ ] 🟠 Images mises à jour dans les 30 jours
  ```bash
  # Âge des images
  kubectl get pods --all-namespaces -o json | \
    jq '.items[].status.containerStatuses[].imageID'
  ```

#### 8.2 Tests de Sécurité

- [ ] 🟠 Tests de sécurité mensuels
  ```
  Dernier test : [DATE]
  Outil : [kube-bench/kube-score/Polaris]
  Score : [VALEUR]
  ```

- [ ] 🟡 Pen testing trimestriel
  ```
  Dernier pen test : [DATE]
  Société : [NOM]
  Vulnérabilités critiques : [NOMBRE]
  ```

- [ ] 🟡 Bug bounty program (optionnel)
  ```
  Actif : [OUI/NON]
  Plateforme : [HACKERONE/BUGCROWD/AUTRE]
  ```

### 9. Documentation et Formation 📚

#### 9.1 Documentation

- [ ] 🟠 Architecture de sécurité documentée
  ```
  Document : [URL/CHEMIN]
  Dernière mise à jour : [DATE]
  ```

- [ ] 🟠 Runbooks d'incident documentés
  ```
  Runbooks :
  - [ ] Compromission de pod
  - [ ] Fuite de secret
  - [ ] Attaque DDoS
  - [ ] Accès non autorisé
  ```

- [ ] 🟡 Diagrammes de flux réseau
  ```
  Diagramme à jour : [OUI/NON]
  Outil : [DRAW.IO/LUCIDCHART]
  ```

#### 9.2 Formation

- [ ] 🟠 Équipe formée aux bonnes pratiques Kubernetes
  ```
  Dernière formation : [DATE]
  Participants : [NOMBRE]
  Prochaine session : [DATE]
  ```

- [ ] 🟡 Au moins un CKS dans l'équipe
  ```
  Certifications :
  - CKA : [NOMBRE]
  - CKAD : [NOMBRE]
  - CKS : [NOMBRE]
  ```

- [ ] 🟡 Sensibilisation sécurité annuelle
  ```
  Dernière session : [DATE]
  ```

### 10. Conformité et Gouvernance ⚖️

#### 10.1 Politiques

- [ ] 🟠 Politique de sécurité formelle définie
  ```
  Document : [URL]
  Approuvé par : [NOM]
  Date : [DATE]
  ```

- [ ] 🟡 OPA Gatekeeper ou équivalent (politiques automatisées)
  ```bash
  kubectl get pods -n gatekeeper-system
  ```

- [ ] 🟡 Admission Controllers personnalisés si nécessaire
  ```bash
  kubectl get validatingwebhookconfigurations
  kubectl get mutatingwebhookconfigurations
  ```

#### 10.2 Audit et Certification

- [ ] 🟡 CIS Kubernetes Benchmark exécuté
  ```bash
  kube-bench run --targets master,node
  # Score > 80%
  ```

- [ ] 🟡 Conformité aux réglementations applicables
  ```
  Réglementations :
  - [ ] SOC 2
  - [ ] PCI-DSS
  - [ ] HIPAA
  - [ ] GDPR
  - [ ] ISO 27001
  ```

- [ ] 🟢 Audit externe annuel
  ```
  Dernier audit : [DATE]
  Auditeur : [SOCIÉTÉ]
  Résultat : [CONFORME/NON-CONFORME]
  ```

### 11. Incident Response 🚨

#### 11.1 Préparation

- [ ] 🔴 Plan de réponse aux incidents défini
  ```
  Document : [URL]
  Dernière révision : [DATE]
  Testé : [OUI/NON]
  ```

- [ ] 🟠 Équipe d'intervention identifiée
  ```
  Responsable sécurité : [NOM]
  Contact 24/7 : [TÉLÉPHONE/EMAIL]
  Escalade : [PROCÉDURE]
  ```

- [ ] 🟠 Canaux de communication d'urgence
  ```
  Canal principal : [SLACK/TEAMS/AUTRE]
  Canal backup : [TÉLÉPHONE]
  ```

#### 11.2 Détection

- [ ] 🟠 SIEM ou équivalent configuré
  ```
  Outil : [ELASTICSEARCH/SPLUNK/AUTRE]
  Alertes actives : [NOMBRE]
  ```

- [ ] 🟡 Honeypots ou canaries déployés
  ```
  Déployés : [OUI/NON]
  Dernière alerte : [DATE]
  ```

#### 11.3 Post-Incident

- [ ] 🟠 Processus de post-mortem défini
  ```
  Template : [URL]
  Post-mortems publics : [OUI/NON]
  ```

- [ ] 🟡 Base de connaissances des incidents
  ```
  Emplacement : [WIKI/CONFLUENCE]
  Incidents documentés : [NOMBRE]
  ```

## Score Global

### Calcul du Score

```
Score Total = (Éléments Cochés / Éléments Totaux) × 100

Pondération par priorité :
🔴 CRITIQUE : × 4
🟠 IMPORTANT : × 3
🟡 RECOMMANDÉ : × 2
🟢 OPTIONNEL : × 1
```

### Interprétation

```
0-40% : 🔴 Sécurité Insuffisante
  └─ Action requise immédiate
  └─ Ne PAS utiliser en production

41-70% : 🟠 Sécurité de Base
  └─ Acceptable pour développement/staging
  └─ Amélioration requise avant production

71-90% : 🟡 Sécurité Robuste
  └─ Acceptable pour production
  └─ Continuer les améliorations

91-100% : 🟢 Sécurité Excellence
  └─ Production haute sécurité
  └─ Maintenir le niveau
```

## Modèle de Rapport

```markdown
# Rapport d'Audit Sécurité Kubernetes

**Cluster :** [NOM]
**Date :** [DATE]
**Auditeur :** [NOM]

## Résumé Exécutif

- Score global : [XX]%
- Niveau de maturité : [1/2/3/4]
- Vulnérabilités critiques : [NOMBRE]
- Recommandations prioritaires : [NOMBRE]

## Détails par Catégorie

### 1. Infrastructure et Cluster
- Éléments cochés : XX/YY (ZZ%)
- Problèmes critiques :
  1. [DESCRIPTION]
  2. [DESCRIPTION]

### 2. Contrôle d'Accès (RBAC)
- Éléments cochés : XX/YY (ZZ%)
- Problèmes critiques :
  1. [DESCRIPTION]

[...répéter pour chaque catégorie...]

## Plan d'Action Prioritaire

### À faire cette semaine (🔴 CRITIQUE)
1. [ ] [ACTION 1]
2. [ ] [ACTION 2]

### À faire ce mois (🟠 IMPORTANT)
1. [ ] [ACTION 1]
2. [ ] [ACTION 2]

### À faire ce trimestre (🟡 RECOMMANDÉ)
1. [ ] [ACTION 1]
2. [ ] [ACTION 2]

## Tendances

| Trimestre | Score | Évolution |
|-----------|-------|-----------|
| Q4 2023   | 65%   | -         |
| Q1 2024   | 72%   | +7%       |
| Q2 2024   | 78%   | +6%       |

## Prochaines Étapes

1. [ÉTAPE 1]
2. [ÉTAPE 2]

**Prochain audit :** [DATE]
```

## Automatisation de la Checklist

### Script d'Audit Automatisé

```bash
#!/bin/bash
# audit-security.sh

echo "=== Audit de Sécurité Kubernetes ==="
echo "Date: $(date)"
echo ""

# 1. Vérifier la version
echo "📋 Version Kubernetes"
kubectl version --short

# 2. Vérifier RBAC
echo ""
echo "📋 Audit RBAC"
echo "Utilisateurs avec cluster-admin:"
kubectl get clusterrolebindings -o json | \
  jq '.items[] | select(.roleRef.name=="cluster-admin") | .subjects'

# 3. Vérifier Pod Security
echo ""
echo "📋 Pod Security Standards"
kubectl get namespaces -o json | \
  jq '.items[] | {name: .metadata.name, enforce: .metadata.labels["pod-security.kubernetes.io/enforce"]}'

# 4. Vérifier les images
echo ""
echo "📋 Images avec :latest"
kubectl get pods --all-namespaces -o json | \
  jq -r '.items[].spec.containers[] | select(.image | contains(":latest")) | .image' | \
  sort -u

# 5. Vérifier les Network Policies
echo ""
echo "📋 Network Policies"
kubectl get networkpolicies --all-namespaces

# 6. Vérifier l'audit logging
echo ""
echo "📋 Audit Logging"
if [ -f /var/log/kubernetes/audit.log ]; then
  echo "✅ Audit log présent"
  echo "Taille: $(du -h /var/log/kubernetes/audit.log | cut -f1)"
else
  echo "❌ Audit log absent"
fi

# 7. Score final
echo ""
echo "=== Analyse complète avec kube-bench ==="
kube-bench run --targets master,node

echo ""
echo "=== Fin de l'audit ==="
```

**Utilisation :**
```bash
chmod +x audit-security.sh
./audit-security.sh > audit-report-$(date +%Y%m%d).txt
```

### Dashboard de Sécurité

```yaml
# Prometheus rules pour métriques de sécurité
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: kubernetes-security-metrics
spec:
  groups:
  - name: security
    interval: 30s
    rules:
    - record: security:pods_without_limits:count
      expr: count(kube_pod_container_resource_limits{resource="memory"} == 0)

    - record: security:pods_as_root:count
      expr: count(kube_pod_container_security_context_run_as_root == 1)

    - record: security:images_with_latest:count
      expr: count(kube_pod_container_image =~ ".*:latest$")

    - record: security:score:percentage
      expr: |
        100 - (
          (security:pods_without_limits:count * 10) +
          (security:pods_as_root:count * 20) +
          (security:images_with_latest:count * 5)
        )
```

## Fréquence de Revue

### Quotidienne 📅

```bash
# Quick check
- [ ] Alertes monitoring (5 min)
- [ ] Scan rapide des logs d'audit (10 min)
- [ ] Vérifier les backups (2 min)
```

### Hebdomadaire 📅

```bash
# Weekly review
- [ ] Revue des alertes de la semaine (20 min)
- [ ] Nouveaux pods déployés (10 min)
- [ ] Changements RBAC (10 min)
- [ ] Incidents de sécurité (si applicable) (30 min)
```

### Mensuelle 📅

```bash
# Monthly audit
- [ ] Checklist complète (2-3 heures)
- [ ] Scan de toutes les images
- [ ] Audit RBAC complet
- [ ] Revue des Network Policies
- [ ] Mise à jour de la documentation
- [ ] Rapport mensuel
```

### Trimestrielle 📅

```bash
# Quarterly deep dive
- [ ] Pen testing
- [ ] Tests de DR
- [ ] Mise à jour Kubernetes
- [ ] Formation équipe
- [ ] Revue de l'architecture
- [ ] Audit CIS Benchmark
```

### Annuelle 📅

```bash
# Yearly assessment
- [ ] Audit externe
- [ ] Certification (SOC 2, ISO, etc.)
- [ ] Revue stratégique de sécurité
- [ ] Budget sécurité
- [ ] Roadmap sécurité
```

## Outils pour l'Audit

### Scan et Analyse

```bash
# kube-bench - CIS Benchmark
kube-bench run

# kube-score - Score de qualité
kube-score score *.yaml

# kubesec - Scan de sécurité
kubesec scan pod.yaml

# Polaris - Audit complet
polaris audit --format=pretty

# Trivy - Scan d'images
trivy image nginx:latest

# kube-hunter - Pen testing
kube-hunter --remote <cluster-ip>
```

### Monitoring

```bash
# kubectl-who-can
kubectl-who-can delete pods

# kubectl-trace
kubectl-trace run node/worker-1 -e 'open("/etc/passwd")'

# rbac-lookup
rbac-lookup alice@company.com

# Popeye - Cluster sanitizer
popeye
```

## Conclusion

Cette checklist est un outil vivant qui doit évoluer avec :

1. **Votre maturité** : Commencez par les items 🔴 CRITIQUES
2. **Vos besoins** : Adaptez selon votre contexte (dev/prod, taille, industrie)
3. **Les menaces** : Mettez à jour selon les nouvelles vulnérabilités
4. **La technologie** : Ajoutez les nouveautés Kubernetes

### Prochaines Actions

```
Étape 1 : Évaluation initiale
└─ Compléter cette checklist (2-3h)
└─ Calculer votre score
└─ Identifier les gaps critiques

Étape 2 : Plan d'action
└─ Prioriser par risque et effort
└─ Assigner les responsabilités
└─ Définir les deadlines

Étape 3 : Exécution
└─ Implémenter les corrections
└─ Valider les changements
└─ Documenter

Étape 4 : Vérification
└─ Re-exécuter la checklist
└─ Mesurer l'amélioration
└─ Ajuster le plan

Étape 5 : Routine
└─ Intégrer dans le processus quotidien
└─ Automatiser ce qui peut l'être
└─ Amélioration continue
```

### Message Final

```
La sécurité n'est pas une destination, c'est un voyage.
Cette checklist est votre carte routière.

Commencez aujourd'hui, améliorez continuellement.
Chaque case cochée rend votre cluster plus sûr.

Bonne chance ! 🚀🔒
```

---

**Dernière mise à jour :** [25/10/2025]
**Version de la checklist :** 1.0
**Compatible avec :** Kubernetes 1.26+, MicroK8s 1.26+

---

## Ressources Complémentaires

### Documentation

- **Kubernetes Security Best Practices** : https://kubernetes.io/docs/concepts/security/
- **CIS Kubernetes Benchmark** : https://www.cisecurity.org/benchmark/kubernetes
- **NSA Kubernetes Hardening Guide** : https://media.defense.gov/2022/Aug/29/2003066362/-1/-1/0/CTR_KUBERNETES_HARDENING_GUIDANCE_1.2_20220829.PDF
- **OWASP Kubernetes Top 10** : https://owasp.org/www-project-kubernetes-top-ten/

### Communauté

- **CNCF Slack** : #kubernetes-security
- **Reddit** : r/kubernetes
- **Stack Overflow** : [kubernetes] tag

### Support

En cas de doute sur un item de cette checklist, n'hésitez pas à :
1. Consulter les sections détaillées (16.1 à 16.9)
2. Lire la documentation Kubernetes officielle
3. Demander de l'aide à la communauté
4. Faire appel à un expert en sécurité Kubernetes

**La sécurité est l'affaire de tous. Chaque action compte. 🛡️**

⏭️ [DevOps et CI/CD](/17-devops-et-cicd/README.md)
