🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 11.1 Introduction aux certificats

## Qu'est-ce qu'un certificat SSL/TLS ?

Imaginez que vous envoyez une lettre importante par la poste. Pour garantir que personne ne puisse lire son contenu pendant le transport, vous la placez dans une enveloppe scellée. Les certificats SSL/TLS jouent un rôle similaire sur Internet : ils protègent les communications entre votre navigateur et un serveur web en créant un canal de communication chiffré.

**SSL** (Secure Sockets Layer) et **TLS** (Transport Layer Security) sont des protocoles cryptographiques qui sécurisent les échanges de données sur Internet. TLS est en réalité la version moderne et plus sécurisée de SSL, mais on utilise encore souvent le terme "SSL" par habitude. Dans ce tutoriel, nous utiliserons les deux termes de manière interchangeable.

## Pourquoi avons-nous besoin de certificats ?

### 1. Confidentialité des données

Sans certificat, les données que vous envoyez sur Internet circulent en clair, comme une carte postale. N'importe qui interceptant la communication peut lire son contenu : mots de passe, informations bancaires, messages privés, etc.

Avec un certificat SSL/TLS, toutes ces données sont **chiffrées**. Même si quelqu'un parvient à intercepter la communication, il ne verra qu'un amas de caractères incompréhensibles.

### 2. Authentification du serveur

Comment savez-vous que vous communiquez réellement avec votre banque et non avec un imposteur ? Les certificats permettent de **vérifier l'identité** du serveur. C'est comme une carte d'identité pour un site web, délivrée par une autorité de confiance.

### 3. Intégrité des données

Les certificats garantissent également que les données n'ont pas été modifiées pendant leur transmission. Si quelqu'un tente d'altérer les données en transit, le destinataire s'en apercevra immédiatement.

## HTTP vs HTTPS : comprendre la différence

- **HTTP** (HyperText Transfer Protocol) : protocole de communication non sécurisé. Les données circulent en clair.
- **HTTPS** (HTTP Secure) : même protocole, mais avec une couche de sécurité SSL/TLS. Les données sont chiffrées.

Vous pouvez facilement identifier un site sécurisé dans votre navigateur :
- L'URL commence par `https://` au lieu de `http://`
- Un cadenas apparaît dans la barre d'adresse
- Les navigateurs modernes affichent des avertissements pour les sites non sécurisés

## Les acteurs d'un certificat

### Autorité de Certification (CA)

Une **Autorité de Certification** ou **CA** (Certificate Authority) est une organisation de confiance qui délivre les certificats. Son rôle est comparable à celui d'un notaire qui authentifie des documents officiels.

Les CA les plus connues incluent :
- Let's Encrypt (gratuit et automatisé)
- DigiCert
- GlobalSign
- Sectigo

Votre navigateur et votre système d'exploitation ont une liste pré-installée de CA de confiance. Lorsque vous visitez un site HTTPS, votre navigateur vérifie que le certificat a été signé par l'une de ces autorités.

### Certificat racine (Root Certificate)

Les CA possèdent un **certificat racine** qui sert à signer d'autres certificats. C'est le point de départ de la chaîne de confiance. Ces certificats racines sont installés par défaut dans les navigateurs et systèmes d'exploitation.

### Certificat intermédiaire

Pour des raisons de sécurité, les CA n'utilisent généralement pas directement leur certificat racine. Elles créent des **certificats intermédiaires** qui servent à signer les certificats des sites web. Cela forme une chaîne de confiance.

### Certificat serveur (End-Entity Certificate)

C'est le certificat installé sur votre serveur web, celui qui protège votre site. Il contient :
- Le nom de domaine du site (exemple : `monsite.com`)
- La clé publique
- L'identité de l'organisation (optionnel)
- La signature de la CA
- La date de validité

## Certificats auto-signés vs certificats signés par une CA

### Certificats auto-signés

Un certificat **auto-signé** est un certificat que vous créez vous-même, sans passer par une autorité de certification. C'est comme si vous signiez votre propre carte d'identité.

**Avantages :**
- Gratuit
- Création instantanée
- Parfait pour le développement et les tests

**Inconvénients :**
- Les navigateurs affichent des avertissements de sécurité
- Pas de validation par une autorité tierce
- Inadapté pour un environnement de production public

**Usage recommandé :** environnements de développement, réseaux internes, tests

### Certificats signés par une CA

Ces certificats sont émis par une autorité de certification reconnue.

**Avantages :**
- Reconnus et approuvés par tous les navigateurs
- Pas d'avertissements de sécurité
- Confiance établie automatiquement

**Inconvénients :**
- Peuvent être payants (sauf Let's Encrypt)
- Processus de validation requis
- Renouvellement périodique nécessaire

**Usage recommandé :** tous les sites web en production, applications publiques

## Let's Encrypt : la révolution des certificats gratuits

**Let's Encrypt** est une autorité de certification gratuite, automatisée et ouverte, lancée en 2015. Elle a révolutionné l'adoption du HTTPS en rendant les certificats SSL/TLS :
- Totalement gratuits
- Faciles à obtenir
- Automatisables
- Renouvelables automatiquement

Let's Encrypt a pour mission de sécuriser l'ensemble du Web en supprimant les barrières financières et techniques à l'adoption du HTTPS.

## Durée de validité et renouvellement

Les certificats ont une **durée de validité limitée** pour des raisons de sécurité :
- Certificats Let's Encrypt : **90 jours**
- Certificats commerciaux : généralement 1 an (maximum)

Cette durée limitée peut sembler contraignante, mais elle présente des avantages importants :
- Limite l'impact en cas de compromission d'une clé privée
- Encourage l'automatisation des processus
- Permet de révoquer plus facilement des certificats problématiques

Le **renouvellement automatique** est donc essentiel, surtout avec Let's Encrypt. Heureusement, des outils comme Cert-Manager (que nous verrons dans les prochaines sections) gèrent ce renouvellement de manière totalement transparente.

## Types de certificats

### Certificat à validation de domaine (DV - Domain Validation)

Le type le plus simple et le plus courant. La CA vérifie uniquement que vous contrôlez le nom de domaine.

- **Validation** : vérification automatique via DNS ou HTTP
- **Délivrance** : quelques minutes
- **Usage** : blogs, sites personnels, applications web
- **Exemple** : Let's Encrypt, la plupart des certificats gratuits

### Certificat à validation d'organisation (OV - Organization Validation)

La CA vérifie l'existence légale de votre organisation en plus du contrôle du domaine.

- **Validation** : vérification manuelle de documents officiels
- **Délivrance** : quelques jours
- **Usage** : sites d'entreprises, portails clients
- **Informations** : nom de l'organisation visible dans le certificat

### Certificat à validation étendue (EV - Extended Validation)

Le niveau de validation le plus strict. Enquête approfondie sur l'organisation.

- **Validation** : processus rigoureux et documenté
- **Délivrance** : plusieurs jours à semaines
- **Usage** : banques, sites de commerce électronique à haute sécurité
- **Affichage** : certains navigateurs affichent le nom de l'organisation en vert (fonctionnalité en déclin)

### Certificats Wildcard

Un certificat **wildcard** (joker) protège un domaine principal et tous ses sous-domaines au premier niveau.

- **Format** : `*.exemple.com`
- **Couvre** : `exemple.com`, `www.exemple.com`, `blog.exemple.com`, `api.exemple.com`
- **Ne couvre pas** : `sous.domaine.exemple.com` (sous-domaine de second niveau)
- **Usage** : pratique pour les applications avec plusieurs sous-domaines

## Les certificats dans Kubernetes

### Pourquoi avons-nous besoin de certificats dans Kubernetes ?

Dans un cluster Kubernetes, vous déployez des applications web accessibles via Internet ou votre réseau interne. Pour sécuriser ces applications, vous avez besoin de certificats SSL/TLS.

Sans Kubernetes, vous installeriez manuellement les certificats sur chaque serveur web. Avec Kubernetes, vous devez gérer les certificats de manière déclarative et automatisée, en accord avec la philosophie "Infrastructure as Code".

### Les défis spécifiques à Kubernetes

1. **Provisionnement automatique** : créer et installer automatiquement des certificats pour de nouveaux services
2. **Renouvellement automatique** : renouveler les certificats avant leur expiration sans intervention manuelle
3. **Gestion centralisée** : gérer tous les certificats depuis un point central
4. **Scalabilité** : gérer potentiellement des dizaines ou centaines de certificats
5. **Secrets Kubernetes** : stocker les certificats de manière sécurisée dans des Secrets

### La solution : Cert-Manager

**Cert-Manager** est un contrôleur Kubernetes natif qui automatise la gestion des certificats. C'est l'outil de référence pour gérer les certificats SSL/TLS dans Kubernetes.

Fonctionnalités principales :
- Obtention automatique de certificats auprès de Let's Encrypt
- Renouvellement automatique avant expiration
- Support de plusieurs CA (Let's Encrypt, HashiCorp Vault, etc.)
- Création de certificats auto-signés pour le développement
- Gestion des certificats via des ressources Kubernetes (CRD)

Avec MicroK8s, l'installation de Cert-Manager est simplifiée grâce à l'addon intégré que nous découvrirons dans les sections suivantes.

## Schéma de fonctionnement simplifié

Voici comment fonctionne la sécurisation d'une application web dans Kubernetes avec des certificats :

1. **Déploiement** : vous déployez votre application dans un Pod
2. **Service** : vous créez un Service pour exposer l'application
3. **Ingress** : vous créez une ressource Ingress pour router le trafic HTTP/HTTPS
4. **Certificat** : Cert-Manager obtient automatiquement un certificat Let's Encrypt
5. **Secret** : le certificat est stocké dans un Secret Kubernetes
6. **Configuration** : l'Ingress Controller utilise ce certificat pour chiffrer les connexions
7. **Renouvellement** : Cert-Manager renouvelle automatiquement le certificat avant expiration

Tout ce processus est déclaratif et automatisé. Vous n'avez qu'à créer quelques manifestes YAML.

## Bonnes pratiques de sécurité

### 1. Toujours utiliser HTTPS en production

Tout site ou application accessible publiquement devrait utiliser HTTPS, même pour du contenu qui semble "non sensible". Les moteurs de recherche favorisent les sites HTTPS, et les navigateurs avertissent les utilisateurs pour les sites HTTP.

### 2. Protéger les clés privées

La **clé privée** est le composant le plus sensible d'un certificat. Si quelqu'un y a accès, il peut se faire passer pour votre site. Dans Kubernetes :
- Les clés privées sont stockées dans des Secrets
- Ne jamais les commiter dans Git
- Limiter l'accès via RBAC

### 3. Automatiser le renouvellement

Ne comptez jamais sur des interventions manuelles pour renouveler des certificats. Les certificats expirés causent des interruptions de service. Avec Cert-Manager, le renouvellement est automatique et fiable.

### 4. Utiliser des certificats valides en production

Les certificats auto-signés sont parfaits pour le développement, mais doivent être remplacés par des certificats signés par une CA en production, même pour des applications internes.

### 5. Surveiller les expirations

Bien que le renouvellement soit automatisé, mettez en place une surveillance pour être alerté en cas de problème. Cert-Manager expose des métriques Prometheus pour cela.

## Concepts clés à retenir

- **SSL/TLS** : protocoles qui sécurisent les communications sur Internet
- **HTTPS** : HTTP avec chiffrement SSL/TLS
- **Certificat** : document électronique qui prouve l'identité d'un site et permet le chiffrement
- **CA (Autorité de Certification)** : organisation qui délivre et signe les certificats
- **Let's Encrypt** : CA gratuite et automatisée qui a démocratisé HTTPS
- **Certificat auto-signé** : certificat créé soi-même, utile pour le développement
- **Wildcard** : certificat qui couvre un domaine et tous ses sous-domaines
- **Cert-Manager** : outil Kubernetes qui automatise la gestion des certificats
- **Renouvellement automatique** : processus essentiel pour éviter les expirations

## Ce que nous verrons dans les prochaines sections

Maintenant que vous comprenez les fondamentaux des certificats SSL/TLS, nous allons explorer leur mise en œuvre pratique dans MicroK8s :

- **Section 11.2** : installation et configuration de Cert-Manager avec l'addon MicroK8s
- **Section 11.3** : configuration détaillée de Cert-Manager
- **Section 11.4** : obtention automatique de certificats Let's Encrypt pour vos applications
- **Section 11.5** : création de certificats auto-signés pour vos environnements de développement
- **Section 11.6** : configuration de certificats wildcard
- **Section 11.7** : mise en place de redirections HTTPS automatiques
- **Section 11.8** : gestion du renouvellement automatique
- **Section 11.9** : dépannage des problèmes courants liés aux certificats

La bonne nouvelle, c'est que MicroK8s et Cert-Manager rendent tout ce processus étonnamment simple. En quelques commandes et manifestes YAML, vous aurez des certificats SSL/TLS professionnels et automatiquement renouvelés pour toutes vos applications.

---

⏭️ [Installation de Cert-Manager (microk8s enable cert-manager)](/11-certificats-ssl-tls/02-installation-de-cert-manager.md)
