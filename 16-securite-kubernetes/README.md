🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 16. Sécurité Kubernetes

## Introduction au Chapitre

La sécurité est l'un des aspects les plus critiques de Kubernetes. Un cluster non sécurisé peut entraîner des fuites de données, des interruptions de service, des pertes financières et des atteintes à la réputation. Ce chapitre vous guidera à travers tous les aspects essentiels de la sécurité Kubernetes, du débutant à l'expert.

**Analogie simple :** Sécuriser un cluster Kubernetes, c'est comme protéger une maison. Vous n'installez pas juste une serrure sur la porte d'entrée et vous arrêtez là. Vous installez aussi :
- Des verrous sur les fenêtres (isolation réseau)
- Une alarme (monitoring et alertes)
- Des caméras de surveillance (audit logging)
- Un coffre-fort pour vos objets de valeur (secrets management)
- Vous vérifiez régulièrement que tout fonctionne (audits de sécurité)

C'est cette approche multicouche, appelée **défense en profondeur**, que nous allons mettre en œuvre dans ce chapitre.

## Pourquoi la Sécurité est Cruciale ?

### Les Risques d'un Cluster Non Sécurisé

```
┌─────────────────────────────────────────────────┐
│  Conséquences d'une Compromission               │
│                                                 │
│  💰 Financières                                 │
│     • Rançongiciels (ransomware)                │
│     • Cryptominage malveillant                  │
│     • Amendes réglementaires (RGPD, etc.)       │
│                                                 │
│  📊 Opérationnelles                             │
│     • Interruption de service                   │
│     • Perte de données                          │
│     • Temps de récupération                     │
│                                                 │
│  🏢 Réputation                                  │
│     • Perte de confiance clients                │
│     • Image de marque détériorée                │
│     • Couverture médiatique négative            │
│                                                 │
│  ⚖️ Légales                                     │
│     • Non-conformité réglementaire              │
│     • Poursuites judiciaires                    │
│     • Responsabilité civile                     │
└─────────────────────────────────────────────────┘
```

### Incidents Réels

**Exemples de compromissions célèbres :**

1. **Tesla (2018)** - Cryptominage via dashboard Kubernetes non sécurisé
   - Cluster exposé sans authentification
   - Perte de ressources cloud
   - Données AWS exposées

2. **Capital One (2019)** - Fuite de données de 100 millions de clients
   - Mauvaise configuration firewall
   - Permissions trop larges
   - 80 millions de dollars d'amende

3. **SolarWinds (2020)** - Supply chain attack
   - Compromission du pipeline de build
   - Images Docker malveillantes
   - Impact mondial

Ces incidents auraient pu être évités ou limités avec les bonnes pratiques que nous allons étudier.

## Architecture de Sécurité Kubernetes

Kubernetes intègre plusieurs couches de sécurité qui travaillent ensemble :

```
┌────────────────────────────────────────────────────┐
│  Couche 8 : Conformité et Gouvernance              │
│  ├─ Politiques organisationnelles                  │
│  └─ Audits et certifications                       │
├────────────────────────────────────────────────────┤
│  Couche 7 : Monitoring et Audit                    │
│  ├─ Audit logging                                  │
│  ├─ Monitoring de sécurité                         │
│  └─ Détection d'anomalies                          │
├────────────────────────────────────────────────────┤
│  Couche 6 : Secrets Management                     │
│  ├─ Chiffrement des secrets                        │
│  ├─ Rotation automatique                           │
│  └─ Accès contrôlé                                 │
├────────────────────────────────────────────────────┤
│  Couche 5 : Sécurité des Images                    │
│  ├─ Scan de vulnérabilités                         │
│  ├─ Signature des images                           │
│  └─ Registry sécurisé                              │
├────────────────────────────────────────────────────┤
│  Couche 4 : Pod Security                           │
│  ├─ Pod Security Standards                         │
│  ├─ Security Context                               │
│  └─ Admission Controllers                          │
├────────────────────────────────────────────────────┤
│  Couche 3 : Isolation Réseau                       │
│  ├─ Network Policies                               │
│  ├─ Segmentation                                   │
│  └─ Service Mesh (optionnel)                       │
├────────────────────────────────────────────────────┤
│  Couche 2 : Contrôle d'Accès                       │
│  ├─ RBAC (Role-Based Access Control)               │
│  ├─ ServiceAccounts                                │
│  └─ API Server authentication                      │
├────────────────────────────────────────────────────┤
│  Couche 1 : Infrastructure                         │
│  ├─ Nœuds sécurisés (OS hardening)                 │
│  ├─ Réseau isolé                                   │
│  └─ Chiffrement en transit (TLS)                   │
└────────────────────────────────────────────────────┘
```

**Principe de Défense en Profondeur :** Si une couche est compromise, les autres continuent de protéger le système.

## Vue d'Ensemble du Chapitre

Ce chapitre est organisé en 10 sections progressives, du concept à la pratique :

### 16.1 Modèle de Sécurité Kubernetes

**Ce que vous apprendrez :**
- Les principes fondamentaux de la sécurité Kubernetes
- Le trio Authentication → Authorization → Admission Control
- Les domaines de sécurité (cluster, workloads, réseau, données)
- Le modèle de confiance et les frontières de sécurité

**Pourquoi c'est important :** Comprendre le modèle de sécurité global est essentiel avant de plonger dans les détails techniques.

**Durée estimée :** 30 minutes de lecture

### 16.2 RBAC (Contrôle d'Accès Basé sur les Rôles)

**Ce que vous apprendrez :**
- Les 4 objets RBAC : Role, ClusterRole, RoleBinding, ClusterRoleBinding
- Comment donner les bonnes permissions aux bons utilisateurs
- Le principe du moindre privilège en pratique
- Audit et débogage des permissions

**Pourquoi c'est important :** RBAC est la première ligne de défense pour contrôler qui peut faire quoi dans votre cluster.

**Durée estimée :** 45 minutes de lecture

### 16.3 ServiceAccounts

**Ce que vous apprendrez :**
- La différence entre Users et ServiceAccounts
- Comment créer et utiliser des ServiceAccounts
- Lier ServiceAccounts et RBAC
- Bonnes pratiques (un SA par application)

**Pourquoi c'est important :** Les ServiceAccounts sont l'identité de vos applications. Sans eux, impossible de contrôler ce que vos pods peuvent faire.

**Durée estimée :** 30 minutes de lecture

### 16.4 Network Policies

**Ce que vous apprendrez :**
- Comment isoler les communications réseau entre pods
- Default deny + whitelist explicite
- Filtrer par namespace, labels, ports
- Patterns courants (3-tiers, isolation par environnement)

**Pourquoi c'est important :** Par défaut, tous les pods peuvent communiquer entre eux. Les Network Policies permettent de segmenter le réseau comme un pare-feu.

**Durée estimée :** 45 minutes de lecture

### 16.5 Pod Security Standards

**Ce que vous apprendrez :**
- Les 3 niveaux : Privileged, Baseline, Restricted
- Security Context : runAsNonRoot, capabilities, seccomp
- Comment empêcher l'exécution en root
- Read-only filesystem et autres durcissements

**Pourquoi c'est important :** Les pods sont la surface d'attaque principale. Les sécuriser correctement limite l'impact d'une compromission.

**Durée estimée :** 60 minutes de lecture

### 16.6 Scan de Vulnérabilités d'Images

**Ce que vous apprendrez :**
- Comprendre les CVE et le scoring CVSS
- Utiliser Trivy pour scanner vos images
- Intégrer le scan dans votre pipeline CI/CD
- Corriger les vulnérabilités (mise à jour, images minimales)

**Pourquoi c'est important :** Vos conteneurs sont construits à partir d'images qui peuvent contenir des failles de sécurité. Ne déployez jamais une image non scannée en production.

**Durée estimée :** 45 minutes de lecture

### 16.7 Secrets Management Avancé

**Ce que vous apprendrez :**
- Les limitations des Secrets Kubernetes natifs
- Solutions avancées : Sealed Secrets, External Secrets, Vault
- Chiffrement des secrets au repos
- Rotation automatique et bonnes pratiques

**Pourquoi c'est important :** Les secrets (mots de passe, clés API) sont les données les plus sensibles. Une mauvaise gestion peut compromettre tout le système.

**Durée estimée :** 60 minutes de lecture

### 16.8 Audit Logging

**Ce que vous apprendrez :**
- Enregistrer toutes les actions (qui, quoi, quand)
- Les 4 niveaux d'audit : None, Metadata, Request, RequestResponse
- Configurer une politique d'audit
- Analyser les logs avec jq et centraliser avec Elasticsearch

**Pourquoi c'est important :** Sans audit, impossible de savoir ce qui s'est passé en cas d'incident. C'est votre "boîte noire" pour les enquêtes.

**Durée estimée :** 45 minutes de lecture

### 16.9 Bonnes Pratiques Sécurité

**Ce que vous apprendrez :**
- Synthèse de toutes les bonnes pratiques par domaine
- Checklist de progression par phase (démarrage → expert)
- Anti-patterns à éviter absolument
- Outils recommandés et formation

**Pourquoi c'est important :** Vue d'ensemble actionnable de tout le chapitre avec priorisation claire.

**Durée estimée :** 30 minutes de lecture

### 16.10 Checklist de Sécurité

**Ce que vous apprendrez :**
- Checklist complète de 100+ items vérifiables
- Script d'audit automatisé
- Modèle de rapport d'audit
- Fréquences de vérification (quotidienne à annuelle)

**Pourquoi c'est important :** Outil pratique pour évaluer régulièrement la sécurité de votre cluster et suivre les améliorations.

**Durée estimée :** 45 minutes de lecture + utilisation continue

## Prérequis pour ce Chapitre

### Connaissances Requises

**Essentielles :**
- ✅ Concepts Kubernetes de base (Pods, Deployments, Services)
- ✅ Utilisation de kubectl
- ✅ Compréhension des manifestes YAML
- ✅ Notions de réseau (IP, ports, DNS)

**Recommandées :**
- 🔹 Expérience avec Linux/Bash
- 🔹 Notions de conteneurs Docker
- 🔹 Compréhension des permissions Unix

**Optionnelles :**
- 🔸 Cryptographie de base
- 🔸 Réseaux avancés
- 🔸 Conformité et réglementations

### Installation Nécessaire

Pour suivre ce chapitre, vous devez avoir :

```bash
# MicroK8s installé et fonctionnel
microk8s status

# kubectl configuré
kubectl version --short

# Optionnel mais recommandé :
# - Trivy (scan d'images)
# - jq (manipulation JSON)
# - kubeseal (Sealed Secrets)
```

## Progression Recommandée

### Pour les Débutants (Lab Personnel)

```
Semaine 1-2 : Fondamentaux
├─ 16.1 Modèle de sécurité
├─ 16.2 RBAC
└─ 16.3 ServiceAccounts

Semaine 3-4 : Isolation
├─ 16.4 Network Policies
└─ 16.5 Pod Security Standards

Semaine 5-6 : Protection
├─ 16.6 Scan de vulnérabilités
└─ 16.7 Secrets management

Semaine 7-8 : Observabilité
├─ 16.8 Audit logging
├─ 16.9 Bonnes pratiques
└─ 16.10 Checklist

Durée totale : 2 mois à rythme tranquille
```

### Pour les Professionnels (Production)

```
Sprint 1 (2 semaines) : Fondations
├─ 16.1-16.3 : RBAC et contrôle d'accès
└─ Implémentation : Configurer RBAC strict

Sprint 2 (2 semaines) : Isolation
├─ 16.4-16.5 : Réseau et Pods
└─ Implémentation : Network Policies + Pod Security

Sprint 3 (2 semaines) : Images et Secrets
├─ 16.6-16.7 : Scan et Secrets
└─ Implémentation : Pipeline de scan + Sealed Secrets

Sprint 4 (2 semaines) : Monitoring
├─ 16.8 : Audit logging
└─ Implémentation : Centralisation des logs

Sprint 5 (1 semaine) : Validation
├─ 16.9-16.10 : Bonnes pratiques et Checklist
└─ Audit complet et plan d'amélioration

Durée totale : 9 semaines
```

## Principes Directeurs de la Sécurité

Avant de commencer, gardez toujours à l'esprit ces principes :

### 1. Défense en Profondeur

```
Ne comptez jamais sur une seule mesure de sécurité.
Multipliez les couches de protection.

Exemple : RBAC + Network Policies + Pod Security
```

### 2. Principe du Moindre Privilège

```
Donnez uniquement les permissions strictement nécessaires.
Rien de plus, rien de moins.

Exemple : ServiceAccount avec accès à 1 ConfigMap précis
```

### 3. Sécurité par Défaut

```
Les configurations par défaut doivent être sécurisées.
Pas permissives puis durcies, mais restrictives puis assouplies.

Exemple : Default deny en Network Policies
```

### 4. Zero Trust

```
Ne jamais faire confiance implicitement.
Toujours vérifier l'identité et les autorisations.

Exemple : Authentification + Autorisation à chaque requête
```

### 5. Fail Securely

```
En cas d'erreur, échouer de manière sécurisée.
Mieux vaut bloquer que laisser passer.

Exemple : Si l'admission controller échoue, rejeter le pod
```

### 6. Séparation des Privilèges

```
Ne pas tout concentrer sur un seul compte.
Distribuer les responsabilités.

Exemple : Admin cluster ≠ Admin application
```

### 7. Auditabilité

```
Tout doit pouvoir être tracé et vérifié.
Qui a fait quoi, quand ?

Exemple : Audit logging complet
```

### 8. Simplicité

```
Un système complexe est plus difficile à sécuriser.
Privilégiez les solutions simples et compréhensibles.

Exemple : RBAC simple plutôt que règles alambiquées
```

## Ce que Ce Chapitre N'Est Pas

Pour éviter les mauvaises attentes :

❌ **Pas un guide de hacking** - Nous ne verrons pas comment attaquer Kubernetes
❌ **Pas exhaustif** - La sécurité évolue constamment, nous couvrons l'essentiel
❌ **Pas uniquement théorique** - Chaque concept est accompagné d'exemples pratiques
❌ **Pas spécifique à un cloud** - Les principes s'appliquent partout (on-premise, cloud)

✅ **C'est un guide pratique** pour sécuriser Kubernetes du niveau débutant à avancé
✅ **C'est orienté MicroK8s** mais applicable à tout Kubernetes
✅ **C'est progressif** avec des checkpoints de validation
✅ **C'est actionnable** avec des commandes et configurations concrètes

## Glossaire des Termes Essentiels

Voici les termes clés que vous rencontrerez dans ce chapitre :

| Terme | Définition Courte |
|-------|-------------------|
| **RBAC** | Role-Based Access Control - Contrôle d'accès par rôles |
| **ServiceAccount** | Identité pour les applications (vs Users pour humains) |
| **Network Policy** | Règles de pare-feu entre pods |
| **Pod Security** | Contraintes de sécurité sur les conteneurs |
| **CVE** | Common Vulnerabilities and Exposures - Identifiant de vulnérabilité |
| **Secrets** | Objet Kubernetes pour stocker données sensibles |
| **Audit Log** | Enregistrement de toutes les actions dans le cluster |
| **Admission Controller** | Validation/modification des requêtes vers l'API |
| **Defense in Depth** | Défense en profondeur - Multiple couches de sécurité |
| **Least Privilege** | Moindre privilège - Permissions minimales nécessaires |
| **Zero Trust** | Ne jamais faire confiance implicitement |

## Structure des Sections

Chaque section de ce chapitre suit une structure cohérente :

```
1. Introduction
   └─ Analogie simple pour comprendre le concept

2. Le Problème
   └─ Pourquoi cette sécurité est nécessaire

3. Concepts Fondamentaux
   └─ Théorie accessible

4. Configuration Pratique
   └─ Exemples concrets avec YAML et commandes

5. Exemples Réels
   └─ Cas d'usage pratiques

6. Bonnes Pratiques
   └─ Comment bien faire

7. Débogage
   └─ Problèmes courants et solutions

8. Spécificités MicroK8s
   └─ Particularités de MicroK8s

9. Conclusion
   └─ Points clés à retenir
```

## Comment Tirer le Meilleur de ce Chapitre

### Pour les Lecteurs Débutants

1. **Lisez dans l'ordre** - Les sections se construisent les unes sur les autres
2. **Testez dans un lab** - N'ayez pas peur d'expérimenter avec MicroK8s
3. **Prenez des notes** - Documentez vos configurations
4. **Ne sautez pas les bonnes pratiques** - Elles évitent les erreurs courantes
5. **Utilisez la checklist finale** - C'est votre guide de progression

### Pour les Professionnels

1. **Scannez rapidement** - Identifiez les gaps dans votre cluster actuel
2. **Priorisez** - Commencez par les items CRITIQUES de la checklist
3. **Automatisez** - Intégrez dans vos pipelines CI/CD dès le début
4. **Documentez** - Créez des runbooks pour votre équipe
5. **Formez** - Partagez les connaissances avec vos collègues

### Exercice Préparatoire

Avant de commencer, faites un état des lieux de votre cluster :

```bash
# 1. Quelle version de Kubernetes ?
kubectl version --short

# 2. RBAC activé ?
kubectl api-versions | grep rbac

# 3. Combien de namespaces ?
kubectl get namespaces

# 4. Combien de pods en production ?
kubectl get pods -n production 2>/dev/null | wc -l

# 5. Y a-t-il des Network Policies ?
kubectl get networkpolicies --all-namespaces

# 6. Audit logging activé ?
ls -l /var/log/kubernetes/audit.log 2>/dev/null

# 7. Scanner d'images installé ?
which trivy
```

Notez les résultats. À la fin du chapitre, vous referez cet état des lieux et verrez vos progrès !

## Ressources Complémentaires

### Avant de Commencer

- **Documentation Kubernetes Security** : https://kubernetes.io/docs/concepts/security/
- **CNCF Security Whitepaper** : https://www.cncf.io/wp-content/uploads/2020/08/CNCF_Kubernetes_Security_Whitepaper.pdf
- **CIS Kubernetes Benchmark** : https://www.cisecurity.org/benchmark/kubernetes

### Pendant le Chapitre

- **Kubernetes Slack** : #kubernetes-security channel
- **Stack Overflow** : Tag [kubernetes-security]
- **GitHub Discussions** : github.com/kubernetes/kubernetes/discussions

### Après le Chapitre

- **Certifications** : CKA, CKAD, CKS (Certified Kubernetes Security Specialist)
- **Livres** : "Kubernetes Security" par Liz Rice et Michael Hausenblas
- **Podcasts** : TWIMLAI Kubernetes Security episodes

## Engagement et Objectifs

### Qu'allez-vous Accomplir ?

À la fin de ce chapitre, vous serez capable de :

✅ **Comprendre** le modèle de sécurité Kubernetes de bout en bout
✅ **Configurer** RBAC avec le principe du moindre privilège
✅ **Implémenter** des Network Policies pour isoler les workloads
✅ **Sécuriser** vos pods avec les Pod Security Standards
✅ **Scanner** vos images pour détecter les vulnérabilités
✅ **Gérer** les secrets de manière sécurisée
✅ **Auditer** toutes les actions dans votre cluster
✅ **Évaluer** la sécurité de votre cluster avec la checklist
✅ **Améliorer** continuellement votre posture de sécurité

### Votre Cluster Avant vs Après

```
┌─────────────────────────────────────────────────┐
│  AVANT                                          │
├─────────────────────────────────────────────────┤
│  • Tout le monde est admin                      │
│  • Pods peuvent tout faire                      │
│  • Aucune isolation réseau                      │
│  • Images non scannées                          │
│  • Secrets en clair dans Git                    │
│  • Pas d'audit                                  │
│  • Vulnérabilités inconnues                     │
│                                                 │
│  Niveau de sécurité : 🔴 Critique               │
└─────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────┐
│  APRÈS                                          │
├─────────────────────────────────────────────────┤
│  ✅ RBAC strict par rôle                        │
│  ✅ Pods avec permissions minimales             │
│  ✅ Network Policies en place                   │
│  ✅ Images scannées automatiquement             │
│  ✅ Secrets chiffrés et rotés                   │
│  ✅ Audit logging centralisé                    │
│  ✅ Monitoring de sécurité actif                │
│                                                 │
│  Niveau de sécurité : 🟢 Robuste                │
└─────────────────────────────────────────────────┘
```

## Prêt à Commencer ?

La sécurité peut sembler intimidante au début, mais comme tout dans Kubernetes, elle se construit étape par étape. Chaque section de ce chapitre vous rapprochera d'un cluster sécurisé et résilient.

**Rappelez-vous :**
- 🎯 La perfection n'existe pas, mais l'amélioration continue est possible
- 🔒 Chaque couche de sécurité ajoutée renforce l'ensemble
- 📚 La documentation est votre meilleure amie
- 🤝 La communauté est là pour vous aider
- 🚀 Commencez simple et complexifiez progressivement

### Mot de la Fin

> "La sécurité n'est pas un produit, mais un processus."
> — Bruce Schneier

Ce chapitre vous donne les outils et les connaissances pour construire et maintenir ce processus. À vous de jouer !

---

**Prochaine étape :** 16.1 Modèle de Sécurité Kubernetes →

**Temps estimé pour le chapitre complet :** 6-8 heures de lecture + implémentation progressive

**Bonne chance et bon apprentissage ! 🔐🚀**

---

⏭️ [Modèle de sécurité Kubernetes](/16-securite-kubernetes/01-modele-de-securite-kubernetes.md)
