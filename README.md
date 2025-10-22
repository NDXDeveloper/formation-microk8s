# 🚀 Formation MicroK8s

![License](https://img.shields.io/badge/License-CC%20BY%204.0-blue.svg)
![MicroK8s](https://img.shields.io/badge/MicroK8s-1.31+-orange.svg)
![Kubernetes](https://img.shields.io/badge/Kubernetes-1.31+-326CE5.svg)
![Completion](https://img.shields.io/badge/Modules-26%2F26-green.svg)
![Language](https://img.shields.io/badge/Langue-Français-blue.svg)

**Un guide complet et progressif pour maîtriser MicroK8s et Kubernetes, du niveau débutant jusqu'à l'expertise.**

![MicroK8s Banner](https://blog.dftorres.ca/wp-content/uploads/sites/15/2024/04/MicroK8s.png)

---

## 📖 Table des matières

- [À propos](#-à-propos)
- [Pour qui ?](#-pour-qui-est-cette-formation)
- [Contenu](#-contenu-de-la-formation)
- [Installation](#-démarrage-rapide)
- [Utilisation](#-comment-utiliser-cette-formation)
- [Parcours d'apprentissage](#-parcours-dapprentissage-suggéré)
- [Licence](#-licence)
- [Auteur](#-auteur)

---

## 📋 À propos

Cette formation complète vous accompagne dans la découverte et la maîtrise de **MicroK8s**, la distribution Kubernetes légère et simple à déployer, particulièrement adaptée aux environnements de développement, de test et aux labs personnels.

**✨ Ce que vous trouverez ici :**
- 📚 **26 modules progressifs** structurés en 4 parties
- 🎯 **200+ concepts** expliqués de manière claire et accessible
- 🏗️ **Exemples pratiques** pour chaque fonctionnalité
- 🔧 **Configuration complète** d'un lab personnel
- 📖 **4 annexes de référence** (scripts, templates, configurations, glossaire)
- 🇫🇷 **Entièrement en français** et gratuit sous licence CC BY 4.0

**Durée estimée :** 30-40 heures • **Niveau :** Tous niveaux • **Prérequis :** Connaissances de base en Linux

---

## 👥 Pour qui est cette formation ?

### 🌱 Vous débutez avec Kubernetes ?
Cette formation est conçue pour vous accueillir et vous guider pas à pas. Chaque concept est expliqué clairement, sans supposer de connaissances préalables en orchestration de conteneurs.

### 🌿 Vous avez déjà utilisé Docker ?
Parfait ! Vous trouverez ici tout ce qu'il faut pour passer à l'étape suivante et orchestrer vos conteneurs avec Kubernetes via MicroK8s.

### 🌳 Vous connaissez Kubernetes mais pas MicroK8s ?
Découvrez comment MicroK8s simplifie le déploiement et la gestion de Kubernetes avec ses addons intégrés et sa philosophie "batteries incluses".

### 🚀 Vous êtes DevOps ou SysAdmin ?
Approfondissez vos compétences avec les modules avancés sur la haute disponibilité, le monitoring, la sécurité et les déploiements automatisés.

**Quel que soit votre niveau, cette formation s'adapte à votre rythme d'apprentissage.**

---

## 📚 Contenu de la formation

La formation est organisée en **4 parties progressives** couvrant l'ensemble de l'écosystème MicroK8s et Kubernetes.

### 🎓 Partie 1 : Fondamentaux (Modules 1-4)
**Découvrez les bases** - Installation, premiers déploiements, concepts essentiels
- Introduction à Kubernetes et MicroK8s
- Installation et configuration initiale
- Concepts Kubernetes essentiels (Pods, Deployments, Services)
- Premiers déploiements d'applications

### 🛠️ Partie 2 : Compétences Intermédiaires (Modules 5-11)
**Maîtrisez l'infrastructure** - Addons, stockage, réseau, exposition d'applications
- Addons MicroK8s (la force de MicroK8s !)
- Stockage persistant et StatefulSets
- Réseau Kubernetes et Jobs/CronJobs
- Load Balancing avec MetalLB
- Ingress et routage (exposition externe)
- Certificats SSL/TLS avec Let's Encrypt

### 🎯 Partie 3 : Niveau Avancé (Modules 12-23)
**Passez à la production** - Monitoring, sécurité, CI/CD, haute disponibilité
- Monitoring avec Prometheus et Grafana
- Alerting et notifications
- Observabilité avancée (logs, métriques, tracing)
- Sécurité Kubernetes (RBAC, Network Policies, Pod Security)
- DevOps et CI/CD (GitOps, Helm, ArgoCD)
- Gestion des ressources et autoscaling
- Multi-node et haute disponibilité avec Dqlite
- Sauvegarde et restauration
- Dépannage et maintenance

### 🚀 Partie 4 : Expertise (Modules 24-26)
**Allez plus loin** - Cas d'usage, technologies avancées, certifications
- Cas d'usage pratiques (lab complet, hébergement, démos)
- Technologies avancées (Service Mesh, Serverless, Operators)
- Ressources et parcours de certification Kubernetes

### 📖 Annexes
- Scripts d'automatisation et de maintenance
- Templates de manifestes YAML et Helm Charts
- Configuration réseau complète
- Référence rapide et glossaire

**📋 [Consulter la table des matières complète →](SOMMAIRE.md)**

---

## 🚀 Démarrage rapide

### Prérequis système

```bash
# Minimum recommandé :
- CPU : 2 cœurs
- RAM : 4 Go
- Stockage : 20 Go
- OS : Ubuntu 20.04+ / Debian 11+ / CentOS 8+ / Windows WSL2 / macOS
```

### Installation de MicroK8s

```bash
# Sur Ubuntu/Debian
sudo snap install microk8s --classic

# Ajouter votre utilisateur au groupe
sudo usermod -a -G microk8s $USER
newgrp microk8s

# Vérifier l'installation
microk8s status --wait-ready

# Configuration de l'alias kubectl (recommandé)
echo "alias kubectl='microk8s kubectl'" >> ~/.bashrc
source ~/.bashrc
```

### Cloner cette formation

```bash
git clone https://github.com/NDXDeveloper/formation-microk8s.git
cd formation-microk8s
```

### Premier déploiement

```bash
# Activer les addons essentiels
microk8s enable dns dashboard storage

# Vérifier que tout fonctionne
microk8s kubectl get all --all-namespaces
```

**🎉 Félicitations ! Vous êtes prêt à commencer votre apprentissage.**

---

## 📁 Structure du projet

```
formation-microk8s/
├── README.md
├── SOMMAIRE.md
├── LICENSE
├── 01-introduction-a-kubernetes-et-microk8s/
├── 02-installation-et-configuration-initiale/
├── 03-concepts-kubernetes-essentiels/
├── ...
├── 26-ressources-et-parcours-de-certification/
└── annexes/
    ├── annexe-a-scripts-et-automatisation/
    ├── annexe-b-templates-et-exemples/
    ├── annexe-c-configuration-reseau/
    └── annexe-d-reference-rapide/
```

---

## 🎯 Comment utiliser cette formation

### 📖 Vous débutez avec Kubernetes ?
**👉 Commencez par le [Module 1 : Introduction](01-introduction-a-kubernetes-et-microk8s/README.md)**

Suivez l'ordre des modules. Chaque concept s'appuie sur les précédents, et la progression est pensée pour vous accompagner naturellement.

### 🔧 Vous connaissez déjà Kubernetes ?
**👉 Allez directement au [Module 5 : Addons MicroK8s](05-addons-microk8s/README.md)**

Découvrez ce qui fait la force de MicroK8s : ses addons intégrés qui simplifient énormément la mise en place d'un cluster complet.

### 📚 Vous cherchez une référence rapide ?
**👉 Consultez les [Annexes](annexes/annexe-d-reference-rapide/README.md)**

Commandes essentielles, troubleshooting rapide, glossaire complet : tout ce dont vous avez besoin pour retrouver rapidement une information.

### 🏗️ Vous voulez un projet concret ?
**👉 Explorez le [Module 24 : Cas d'usage pratiques](24-cas-dusage-pratiques/README.md)**

Lab de développement complet, hébergement de services personnels, environnement de test : des architectures prêtes à l'emploi.

**💡 Conseil pratique :** Installez MicroK8s sur une machine virtuelle ou un Raspberry Pi pour expérimenter sans risque !

---

## 🗓️ Parcours d'apprentissage suggéré

| Niveau | Modules | Durée estimée | Objectifs |
|--------|---------|---------------|-----------|
| 🌱 **Découverte** | 1-4 | 6-8h | Comprendre Kubernetes et réaliser ses premiers déploiements |
| 🌿 **Opérationnel** | 5-11 | 10-12h | Maîtriser l'infrastructure (stockage, réseau, exposition) |
| 🌳 **Production** | 12-23 | 12-16h | Monitoring, sécurité, CI/CD et haute disponibilité |
| 🚀 **Expert** | 24-26 | 4-6h | Technologies avancées et préparation certifications |

**Rythme recommandé :** 1-2 modules par semaine avec pratique régulière

**💡 Astuce :** Prenez le temps d'expérimenter après chaque module. La pratique est essentielle !

---

## 🎓 Après cette formation

### Certifications Kubernetes
Cette formation vous prépare aux certifications officielles :
- **CKA** (Certified Kubernetes Administrator)
- **CKAD** (Certified Kubernetes Application Developer)
- **CKS** (Certified Kubernetes Security Specialist)

➡️ Voir le [Module 26](26-ressources-et-parcours-de-certification/README.md) pour plus de détails.

### Aller plus loin
- Contribuer à des projets open source Kubernetes
- Déployer des applications en production
- Automatiser vos infrastructures avec GitOps
- Explorer les Service Mesh et le Serverless

---

## ❓ Questions fréquentes

**Q : Dois-je obligatoirement suivre l'ordre des modules ?**
R : Pour les débutants, oui, c'est fortement recommandé. Si vous avez déjà de l'expérience, vous pouvez sauter certains modules.

**Q : Combien de temps faut-il pour terminer la formation ?**
R : Entre 30 et 40 heures réparties sur 2-3 mois à raison de 1-2 modules par semaine.

**Q : Puis-je utiliser cette formation pour enseigner ?**
R : Absolument ! La licence CC BY 4.0 vous y autorise (attribution requise).

**Q : MicroK8s vs K3s/Minikube, quelle différence ?**
R : MicroK8s se distingue par sa simplicité d'installation, ses addons intégrés et sa haute disponibilité native avec Dqlite. Voir le [Module 1.4](01-introduction-a-kubernetes-et-microk8s/04-comparaison-avec-dautres-solutions.md) pour une comparaison détaillée.

**Q : Cette formation est-elle adaptée pour la production ?**
R : Oui ! MicroK8s est utilisé en production par de nombreuses entreprises. La Partie 3 couvre spécifiquement les aspects production (monitoring, sécurité, HA).

---

## 📝 Licence

Ce projet est sous licence **Creative Commons Attribution 4.0 International (CC BY 4.0)**.

✅ Vous êtes libre de :
- **Partager** — copier et redistribuer le matériel
- **Adapter** — remixer, transformer et créer à partir du matériel
- **Usage commercial** autorisé

🔒 Aux conditions suivantes :
- **Attribution** — Vous devez créditer l'œuvre et indiquer si des modifications ont été effectuées

**Attribution recommandée :**
```
Formation MicroK8s par Nicolas DEOUX
https://github.com/NDXDeveloper/formation-microk8s
Licence CC BY 4.0
```

📄 [Lire le texte complet de la licence](LICENSE)

---

## 👨‍💻 Auteur

**Nicolas DEOUX**

Passionné par le DevOps, les technologies cloud-native et le partage de connaissances, j'ai créé cette formation pour rendre Kubernetes accessible à tous à travers MicroK8s.

**Contact :**
- 📧 Email : [NDXDev@gmail.com](mailto:NDXDev@gmail.com)
- 💼 LinkedIn : [Nicolas DEOUX](https://www.linkedin.com/in/nicolas-deoux-ab295980/)
- 🐙 GitHub : [@NDXDeveloper](https://github.com/NDXDeveloper)

---

## 🙏 Remerciements

Un grand merci à :
- La **communauté Canonical** pour MicroK8s
- La **Cloud Native Computing Foundation** (CNCF) pour Kubernetes
- Tous les contributeurs de l'écosystème Kubernetes
- Et à **vous** qui vous lancez dans cette aventure d'apprentissage ! 🎉

**Ressources qui ont inspiré cette formation :**
- [Documentation officielle MicroK8s](https://microk8s.io/docs)
- [Documentation Kubernetes](https://kubernetes.io/fr/docs/)
- [Kubernetes The Hard Way](https://github.com/kelseyhightower/kubernetes-the-hard-way)

---

<div align="center">

**🎉 Bon apprentissage avec MicroK8s ! 🎉**

[![Star on GitHub](https://img.shields.io/github/stars/NDXDeveloper/formation-microk8s?style=social)](https://github.com/NDXDeveloper/formation-microk8s)
[![Follow](https://img.shields.io/github/followers/NDXDeveloper?style=social)](https://github.com/NDXDeveloper)

**[⬆ Retour en haut](#-formation-complète-microk8s)**

*Dernière mise à jour : Octobre 2025*

</div>
