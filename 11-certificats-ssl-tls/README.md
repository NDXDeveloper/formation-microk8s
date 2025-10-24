🔝 Retour au [Sommaire](/SOMMAIRE.md)

# Chapitre 11 : Certificats SSL/TLS

## Introduction au chapitre

Bienvenue dans l'un des chapitres les plus importants de cette formation : la sécurisation de vos applications avec des certificats SSL/TLS. Dans le monde moderne d'Internet, **HTTPS n'est plus une option, c'est une nécessité**.

Que vous déployiez un blog personnel, une API, une application web d'entreprise, ou un service SaaS, vos utilisateurs s'attendent à ce que leurs données soient protégées. Les navigateurs affichent des avertissements intimidants pour les sites en HTTP, et Google pénalise les sites non sécurisés dans ses résultats de recherche. Plus important encore, les données de vos utilisateurs circulent en clair sur Internet sans HTTPS, exposées à n'importe qui pouvant intercepter le trafic.

**La bonne nouvelle** : grâce à MicroK8s et Cert-Manager, sécuriser vos applications avec HTTPS est devenu étonnamment simple. Ce qui prenait autrefois des heures de configuration manuelle et de gestion fastidieuse peut maintenant se faire en quelques lignes de YAML et de manière complètement automatisée.

## Pourquoi ce chapitre est essentiel

### Avant Kubernetes : la galère des certificats

Traditionnellement, gérer des certificats SSL/TLS était pénible :

- **Obtention manuelle** : vous deviez acheter des certificats (coût : 10-200€/an par domaine) ou passer par des processus complexes pour obtenir des certificats gratuits
- **Installation manuelle** : copier les fichiers de certificats sur chaque serveur, configurer Apache/NGINX manuellement
- **Renouvellement manuel** : chaque année (ou tous les 90 jours avec Let's Encrypt), renouveler manuellement chaque certificat
- **Risque d'oubli** : un certificat expiré = site inaccessible = clients mécontents = perte de revenus
- **Scalabilité impossible** : avec 10, 50, 100 domaines, la gestion devient cauchemardesque

### Avec Kubernetes et Cert-Manager : l'automatisation totale

Cert-Manager révolutionne complètement cette expérience :

- **Gratuit** : certificats Let's Encrypt gratuits et illimités
- **Automatique** : obtention automatique des certificats en 1-2 minutes
- **Déclaratif** : tout se configure en YAML, comme le reste de Kubernetes
- **Renouvellement automatique** : les certificats se renouvellent seuls 30 jours avant expiration
- **Zero downtime** : aucune interruption lors de l'émission ou du renouvellement
- **Scalable** : 1 certificat ou 1000, même simplicité

**Résultat** : vous configurez une fois, et vous n'avez plus à y penser. Les certificats se gèrent eux-mêmes.

## Ce que vous allez apprendre

Ce chapitre est structuré pour vous emmener de zéro à expert en gestion de certificats Kubernetes. Voici le parcours :

### Section 11.1 : Introduction aux certificats

Nous commencerons par les fondamentaux : qu'est-ce qu'un certificat SSL/TLS ? Comment fonctionne le chiffrement HTTPS ? Qu'est-ce qu'une autorité de certification (CA) ? Cette section pose les bases théoriques nécessaires, expliquées de manière simple et accessible.

### Section 11.2 : Installation de Cert-Manager

La philosophie MicroK8s dans toute sa splendeur : `microk8s enable cert-manager`. Une seule commande pour installer l'un des outils les plus puissants de l'écosystème Kubernetes. Vous verrez comment vérifier que tout est bien installé et comprendre les composants déployés.

### Section 11.3 : Configuration de Cert-Manager

Place à la configuration ! Vous apprendrez à créer des **ClusterIssuers** qui définissent comment obtenir des certificats. Nous couvrirons :
- La différence entre Issuer et ClusterIssuer
- Les environnements staging vs production de Let's Encrypt
- Les méthodes de validation HTTP-01 et DNS-01
- Les bonnes pratiques de configuration

### Section 11.4 : Certificats Let's Encrypt automatiques

C'est ici que la magie opère ! Vous verrez comment obtenir automatiquement des certificats SSL/TLS valides et reconnus par tous les navigateurs pour vos applications. Deux approches seront couvertes :
- Création explicite de ressources Certificate
- Automatisation via annotations Ingress

Vous observerez le processus d'émission en temps réel et comprendrez chaque étape.

### Section 11.5 : Certificats auto-signés pour le développement

Let's Encrypt nécessite que votre cluster soit accessible depuis Internet. Mais en développement local, ce n'est souvent pas le cas. Cette section vous montre comment créer des certificats auto-signés instantanément pour vos environnements de développement, et même comment créer votre propre autorité de certification (CA) pour éviter les avertissements de navigateur.

### Section 11.6 : Wildcard certificates

Un seul certificat pour tous vos sous-domaines ! Les certificats wildcard (`*.exemple.com`) sont parfaits pour les architectures micro-services ou les applications multi-tenant. Vous apprendrez :
- Quand utiliser des wildcards
- Comment configurer la validation DNS-01 (requise pour les wildcards)
- Configuration pour les principaux fournisseurs DNS (Cloudflare, AWS Route53, Google Cloud DNS, OVH)

### Section 11.7 : Gestion des redirections HTTPS dans Ingress

Avoir un certificat HTTPS, c'est bien. Forcer tous vos utilisateurs à l'utiliser, c'est mieux ! Cette section couvre :
- Redirection automatique HTTP → HTTPS
- HTTP Strict Transport Security (HSTS)
- Redirections www ↔ non-www
- Bonnes pratiques de sécurité

### Section 11.8 : Renouvellement automatique

Les certificats Let's Encrypt expirent tous les 90 jours. Sans renouvellement automatique, ce serait ingérable. Heureusement, Cert-Manager gère tout ! Vous apprendrez :
- Comment fonctionne le renouvellement automatique
- Les paramètres de configuration (duration, renewBefore)
- Comment surveiller vos certificats avec Prometheus
- Que faire en cas d'échec de renouvellement

### Section 11.9 : Dépannage des certificats

Même avec l'automatisation, des problèmes peuvent survenir. Cette section est votre guide de résolution de problèmes complet :
- Méthodologie de diagnostic systématique
- Les 6 problèmes les plus courants et leurs solutions
- Outils de diagnostic essentiels (kubectl, openssl, curl, dig)
- Checklist complète de vérification
- Scripts de monitoring

## Philosophie de ce chapitre

Ce chapitre a été conçu avec une philosophie claire :

### 1. Accessible aux débutants

Même si vous n'avez jamais touché à un certificat SSL de votre vie, ce chapitre vous guide pas à pas. Chaque concept est expliqué clairement, avec des analogies du monde réel quand nécessaire.

### 2. Progression pédagogique

Nous commençons par les bases théoriques, puis passons à la pratique simple, avant d'aborder les cas d'usage avancés. Chaque section s'appuie sur les précédentes.

### 3. Exemples complets et fonctionnels

Tous les exemples de code YAML sont complets et testés. Vous pouvez les copier-coller et ils fonctionneront (avec vos propres domaines, bien sûr).

### 4. Comprendre avant d'appliquer

Nous ne nous contentons pas de donner des recettes à suivre aveuglément. Chaque commande, chaque paramètre est expliqué pour que vous compreniez **pourquoi** vous faites les choses, pas seulement **comment**.

### 5. Best practices intégrées

Les bonnes pratiques de sécurité et d'opérations sont intégrées naturellement dans chaque section, pas ajoutées artificiellement à la fin.

## Prérequis pour ce chapitre

Avant de commencer ce chapitre, vous devriez avoir :

### Connaissances requises

- ✅ Compréhension de base de Kubernetes (Pods, Deployments, Services)
- ✅ Concepts d'Ingress (couverts au chapitre 10)
- ✅ Notions de base de réseau (DNS, ports HTTP/HTTPS)
- ✅ Capacité à éditer des fichiers YAML

### Infrastructure requise

- ✅ MicroK8s installé et fonctionnel
- ✅ Ingress Controller activé (`microk8s enable ingress`)
- ✅ DNS activé (`microk8s enable dns`)
- ✅ (Optionnel) Un nom de domaine que vous contrôlez
- ✅ (Optionnel) Cluster accessible depuis Internet (pour Let's Encrypt)

**Note importante** : même sans domaine ni exposition Internet, vous pourrez suivre la majorité du chapitre en utilisant des certificats auto-signés et des domaines locaux (`.local`, `.dev`, etc.).

## Structure du chapitre

Le chapitre est organisé en 9 sections progressives :

```
11. Certificats SSL/TLS (ce fichier)
│
├── 11.1 Introduction aux certificats
│   └── Théorie : SSL/TLS, HTTPS, CA, Let's Encrypt
│
├── 11.2 Installation de Cert-Manager
│   └── Pratique : microk8s enable cert-manager
│
├── 11.3 Configuration de Cert-Manager
│   └── Pratique : ClusterIssuers, HTTP-01, DNS-01
│
├── 11.4 Certificats Let's Encrypt automatiques
│   └── Pratique : Certificate, annotations Ingress
│
├── 11.5 Certificats auto-signés pour le développement
│   └── Pratique : certificats locaux, CA personnelle
│
├── 11.6 Wildcard certificates
│   └── Avancé : DNS-01, certificats pour tous les sous-domaines
│
├── 11.7 Gestion des redirections HTTPS dans Ingress
│   └── Pratique : force-ssl-redirect, HSTS
│
├── 11.8 Renouvellement automatique
│   └── Pratique : renewBefore, monitoring, alertes
│
└── 11.9 Dépannage des certificats
    └── Référence : diagnostic, résolution de problèmes
```

**Temps estimé** : 3-5 heures pour l'ensemble du chapitre (lecture + pratique)

**Niveau** : Débutant à Avancé

## Ce que vous saurez faire après ce chapitre

À la fin de ce chapitre, vous serez capable de :

✅ **Comprendre** le fonctionnement des certificats SSL/TLS et leur importance
✅ **Installer** et configurer Cert-Manager sur MicroK8s
✅ **Obtenir** automatiquement des certificats Let's Encrypt gratuits et valides
✅ **Créer** des certificats auto-signés pour le développement local
✅ **Configurer** des certificats wildcard pour plusieurs sous-domaines
✅ **Forcer** HTTPS avec redirections automatiques et HSTS
✅ **Surveiller** vos certificats et leur renouvellement
✅ **Diagnostiquer** et résoudre n'importe quel problème de certificat
✅ **Sécuriser** professionnellement n'importe quelle application Kubernetes

## Approche recommandée

Pour tirer le meilleur parti de ce chapitre :

### 1. Lisez dans l'ordre

Les sections sont conçues pour être lues séquentiellement. Ne sautez pas directement aux sections avancées sans avoir lu les fondamentaux.

### 2. Pratiquez en parallèle

Ouvrez un terminal et essayez les commandes au fur et à mesure. L'apprentissage est beaucoup plus efficace quand on pratique.

### 3. Commencez petit

Si vous n'avez pas de domaine, commencez avec les certificats auto-signés (section 11.5). Une fois à l'aise, passez à Let's Encrypt.

### 4. Utilisez staging d'abord

Quand vous testez avec Let's Encrypt, utilisez **toujours** l'environnement staging d'abord. Les limites de taux en production sont strictes.

### 5. Prenez des notes

Documentez vos configurations, notez les commandes utiles, créez vos propres scripts.

### 6. Consultez la section dépannage

Si vous rencontrez un problème, la section 11.9 contient probablement la solution. C'est une référence complète de dépannage.

## Conventions utilisées dans ce chapitre

Pour faciliter votre lecture :

### Blocs de code

```yaml
# Fichiers YAML avec commentaires explicatifs
apiVersion: cert-manager.io/v1
kind: Certificate
```

```bash
# Commandes à exécuter dans le terminal
kubectl apply -f certificate.yaml
```

### Annotations importantes

**Important** : informations critiques à ne pas manquer
**Note** : détails complémentaires utiles
**Attention** : avertissements sur des risques potentiels
**Astuce** : conseils pour gagner du temps

### Symboles

✅ Bonne pratique / fonctionnalité supportée
❌ Mauvaise pratique / fonctionnalité non supportée
⚠️ Attention requise
💡 Astuce ou conseil

### Exemples

Les exemples utilisent des domaines fictifs :
- `exemple.com`, `app.exemple.com`, `monapp.com`
- Pour vos tests, remplacez par vos vrais domaines

## Ressources complémentaires

Ce chapitre est conçu pour être autonome, mais voici des ressources pour approfondir :

### Documentation officielle

- **Cert-Manager** : https://cert-manager.io/docs/
- **Let's Encrypt** : https://letsencrypt.org/docs/
- **NGINX Ingress** : https://kubernetes.github.io/ingress-nginx/

### Outils utiles

- **SSL Labs** : https://www.ssllabs.com/ssltest/ - Tester votre configuration SSL
- **DNS Checker** : https://dnschecker.org/ - Vérifier la propagation DNS
- **HSTS Preload** : https://hstspreload.org/ - Liste HSTS des navigateurs

### Communauté

- **Cert-Manager Slack** : canal #cert-manager sur Kubernetes Slack
- **Forum Kubernetes** : discuss.kubernetes.io
- **Stack Overflow** : tag `cert-manager`

## Un dernier mot avant de commencer

La sécurité n'est pas une option dans le monde moderne. HTTPS est devenu la norme, et vos utilisateurs l'attendent. La bonne nouvelle, c'est qu'avec les outils que nous allons découvrir, sécuriser vos applications est devenu simple et automatisé.

**Cert-Manager est l'un des projets les plus matures et les plus utilisés de l'écosystème Kubernetes.** Il est utilisé en production par des milliers d'entreprises, des startups aux géants de la tech. Sa fiabilité n'est plus à démontrer.

**MicroK8s rend son installation triviale.** Une seule commande, et vous avez accès à cette puissance.

**Ce chapitre vous donne les clés.** À vous de les utiliser pour sécuriser vos applications et protéger vos utilisateurs.

Alors, prêt à plonger dans le monde des certificats SSL/TLS ? Commençons par les fondamentaux !

---

⏭️ [Introduction aux certificats](/11-certificats-ssl-tls/01-introduction-aux-certificats.md)
