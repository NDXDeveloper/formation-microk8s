🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 10. Ingress et Routage

## Introduction au Chapitre

Félicitations ! Vous avez parcouru un long chemin depuis le début de cette formation. Vous savez maintenant installer MicroK8s, déployer des applications, gérer le stockage persistant, et même configurer des services réseau internes. Mais il manque encore une pièce essentielle du puzzle : **exposer vos applications au monde extérieur** de manière professionnelle et sécurisée.

C'est exactement ce que nous allons découvrir dans ce chapitre consacré à **Ingress et au routage**. Ce chapitre est l'un des plus importants de la formation, car il transforme votre cluster Kubernetes d'un environnement de développement isolé en une **plateforme de production** capable d'héberger des applications accessibles depuis Internet.

## Pourquoi ce Chapitre est Crucial

### Le Défi de l'Exposition des Applications

Jusqu'à présent, vos applications Kubernetes sont accessibles uniquement depuis l'intérieur du cluster ou via des méthodes peu pratiques (NodePort avec des ports aléatoires, port-forward temporaire, etc.). Pour une utilisation réelle, vous avez besoin de :

- **URLs professionnelles** : `https://mon-application.com` au lieu de `http://192.168.1.100:30285`
- **Certificats SSL/TLS** : HTTPS avec cadenas vert pour la sécurité et la confiance
- **Routage intelligent** : Plusieurs applications sur un seul point d'entrée
- **Gestion centralisée** : Configuration unifiée pour toutes vos applications
- **Sécurité renforcée** : Authentification, rate limiting, firewalls

### La Solution : Ingress

**Ingress** est la solution Kubernetes standard pour exposer des services HTTP et HTTPS au monde extérieur avec intelligence et élégance. C'est comme un **réceptionniste ultra-compétent** qui :

1. **Accueille** toutes les requêtes entrantes sur votre cluster
2. **Examine** l'URL demandée
3. **Dirige** chaque visiteur vers la bonne application
4. **Sécurise** les connexions avec HTTPS
5. **Optimise** les performances
6. **Protège** contre les abus

Au lieu d'avoir une IP et un port différents pour chaque application, vous avez **un seul point d'entrée** qui route intelligemment le trafic vers la bonne destination.

## Ce Que Vous Allez Apprendre

Ce chapitre est structuré comme un **parcours complet** qui vous emmène de zéro jusqu'à une configuration professionnelle d'exposition d'applications. Voici ce que nous allons couvrir :

### Phase 1 : Comprendre et Installer (Sections 10.1 à 10.3)

Nous commencerons par les **fondations** :

**Section 10.1 - Concepts d'Ingress**
- Qu'est-ce qu'un Ingress et pourquoi en avez-vous besoin ?
- Comment fonctionne un Ingress Controller ?
- Les différences avec les Services traditionnels
- Quand utiliser Ingress vs LoadBalancer vs NodePort

**Section 10.2 - Installation de NGINX Ingress Controller**
- Installation en une seule commande avec MicroK8s
- Vérification que tout fonctionne correctement
- Comprendre les composants installés
- Architecture et DaemonSets

**Section 10.3 - Configuration de NGINX Ingress Controller**
- Les trois niveaux de configuration (global, par Ingress, par Service)
- ConfigMaps pour les paramètres globaux
- Annotations pour les besoins spécifiques
- Personnalisation avancée avec snippets

### Phase 2 : Préparer l'Infrastructure (Sections 10.4 à 10.7)

Ensuite, nous mettrons en place l'**infrastructure nécessaire** pour rendre vos applications accessibles depuis Internet :

**Section 10.4 - Acquisition d'un Nom de Domaine**
- Pourquoi vous avez besoin d'un domaine
- Comment choisir et acheter un nom de domaine
- Registrars recommandés (Namecheap, OVH, Gandi, Cloudflare)
- Alternatives gratuites pour les labs (nip.io, xip.io)
- Coûts et gestion

**Section 10.5 - Configuration DNS Externe**
- Comprendre le DNS (Domain Name System)
- Types d'enregistrements (A, AAAA, CNAME, wildcard)
- Configuration étape par étape selon votre registrar
- Propagation DNS et vérification
- Wildcard DNS pour plusieurs sous-domaines

**Section 10.6 - Redirection de Ports et NAT**
- Comprendre le NAT et les réseaux privés
- Configuration du port forwarding sur votre box/routeur
- Ports à ouvrir (80 pour HTTP, 443 pour HTTPS)
- Tests et vérification depuis l'extérieur
- Alternatives si le port forwarding n'est pas possible (Cloudflare Tunnel)

**Section 10.7 - Firewall et Règles de Sécurité**
- Pourquoi un firewall est essentiel
- Installation et configuration d'UFW (Uncomplicated Firewall)
- Règles de base pour protéger votre serveur
- Rate limiting et protection contre les attaques
- Outils complémentaires (Fail2ban)

### Phase 3 : Créer les Règles de Routage (Sections 10.8 à 10.10)

Une fois l'infrastructure en place, nous créerons des **règles de routage intelligentes** :

**Section 10.8 - Routage par Nom d'Hôte**
- Principe du host-based routing
- Créer des règles pour différents sous-domaines
- Exemples : `blog.mon-site.com`, `api.mon-site.com`, `admin.mon-site.com`
- Gestion de plusieurs applications sur différents domaines
- Bonnes pratiques et organisation

**Section 10.9 - Routage par Chemin**
- Principe du path-based routing
- Créer des règles pour différents chemins
- Exemples : `mon-site.com/blog`, `mon-site.com/api`, `mon-site.com/admin`
- Réécriture d'URL (rewrite-target)
- Combinaison avec le routage par hôte
- Applications SPA (React, Vue, Angular)

**Section 10.10 - Middleware et Annotations**
- Qu'est-ce qu'un middleware et à quoi sert-il ?
- Annotations pour personnaliser le comportement
- Redirections HTTPS automatiques
- Authentification HTTP Basic
- CORS pour les APIs
- Rate limiting et sécurité
- Optimisation des performances (compression, timeouts)
- WebSockets et connexions persistantes

### Phase 4 : Exemples Concrets (Section 10.11)

Enfin, nous mettrons tout ensemble avec des **exemples pratiques complets** :

**Section 10.11 - Exemples Pratiques d'Ingress**
- Application web simple avec routage basique
- API REST avec CORS et rate limiting
- Interface admin sécurisée avec authentification
- Architecture microservices complète
- Application de chat avec WebSockets
- Environnements multiples (dev, staging, prod)

## À Qui s'Adresse ce Chapitre

Ce chapitre est conçu pour être **accessible à tous** :

### Pour les Débutants
- Explications claires avec des analogies simples
- Progression étape par étape sans sauter de concepts
- Pas de connaissances réseau avancées requises
- Chaque commande est expliquée et commentée

### Pour les Utilisateurs Intermédiaires
- Approfondissement des concepts Kubernetes
- Configurations avancées et cas d'usage réels
- Bonnes pratiques de production
- Dépannage et optimisation

### Pour Tous
- Configurations testées et éprouvées
- Exemples réutilisables et templates
- Références et documentation complètes
- Solutions aux problèmes courants

## Prérequis

Avant de commencer ce chapitre, vous devriez avoir :

✅ **MicroK8s installé et fonctionnel** (Chapitre 2)
✅ **Compréhension de base de Kubernetes** (Pods, Services, Deployments - Chapitre 3)
✅ **Au moins une application déployée** pour tester l'exposition
✅ **Accès à votre serveur** (physique ou SSH)
✅ **Droits administrateur** sur votre machine/serveur

**Optionnel mais utile** :
- Un nom de domaine (ou utilisation de services gratuits comme nip.io)
- Accès à l'interface de votre box Internet (pour port forwarding)
- Une carte de crédit pour acheter un domaine si souhaité

## Durée Estimée

Ce chapitre est dense mais structuré de manière progressive :

- **Lecture complète** : 4-5 heures
- **Pratique et configuration** : 3-4 heures
- **Maîtrise complète** : 1-2 jours avec expérimentation

**Conseil** : Prenez votre temps. Il est préférable de bien comprendre chaque section avant de passer à la suivante. Vous pouvez faire des pauses entre les sections.

## Organisation du Chapitre

Le chapitre suit une **progression logique et naturelle** :

```
Fondations (10.1-10.3)
    ↓
Infrastructure (10.4-10.7)
    ↓
Routage (10.8-10.9)
    ↓
Optimisation (10.10)
    ↓
Pratique (10.11)
```

Chaque section s'appuie sur les précédentes, créant une **montée en compétence progressive** sans rupture brutale de difficulté.

## Ce Que Vous Saurez Faire à la Fin

À l'issue de ce chapitre, vous serez capable de :

🎯 **Exposer des applications Kubernetes** de manière professionnelle avec des URLs propres

🎯 **Configurer des noms de domaine** et gérer les enregistrements DNS

🎯 **Sécuriser votre infrastructure** avec firewalls et redirections de ports

🎯 **Router intelligemment le trafic** selon les noms d'hôte ou les chemins d'URL

🎯 **Optimiser les performances** avec compression, timeouts et mise en cache

🎯 **Protéger vos applications** avec authentification, rate limiting et CORS

🎯 **Débugger et résoudre** les problèmes de routage et de connectivité

🎯 **Créer des configurations** adaptées à différents cas d'usage (APIs, sites web, admin)

## Philosophie de ce Chapitre

Nous avons construit ce chapitre autour de **quatre principes fondamentaux** :

### 1. Apprentissage Progressif

Nous ne vous jetons pas dans le grand bain. Chaque concept est introduit au bon moment, dans le bon ordre. Vous construisez vos connaissances brique par brique.

### 2. Pratique Réelle

Tous les exemples sont **testés et fonctionnels**. Pas de configurations théoriques qui ne marchent jamais dans la vraie vie. Tout ce que vous verrez ici fonctionne réellement.

### 3. Explications Claires

Chaque concept difficile est accompagné d'**analogies simples** et de **schémas textuels**. Vous comprendrez non seulement le "comment" mais aussi le "pourquoi".

### 4. Autonomie

À la fin de ce chapitre, vous ne serez pas dépendant d'exemples copiés-collés. Vous comprendrez assez bien les concepts pour **adapter et créer vos propres configurations**.

## Points d'Attention

Quelques éléments importants à garder en tête :

### 🔴 Sécurité Avant Tout

Exposer des applications sur Internet comporte des risques. Ce chapitre met un **accent fort sur la sécurité** :
- Configuration de firewalls
- Authentification et autorisation
- Rate limiting et protection anti-abus
- HTTPS obligatoire en production

**Ne sautez pas les sections sur la sécurité**, même si elles semblent moins excitantes que le routage.

### 🔴 Patience avec le DNS

La propagation DNS peut prendre du temps (de quelques minutes à 48 heures dans le pire cas). **Ne paniquez pas** si votre domaine ne fonctionne pas immédiatement après configuration. Nous vous montrerons comment vérifier que tout est correctement configuré.

### 🔴 Tests Recommandés

À chaque étape, nous vous encourageons à **tester** que tout fonctionne avant de passer à la suite. C'est plus facile de débugger un problème à la fois que d'avoir 10 problèmes accumulés à la fin.

### 🔴 Environnements Différents

Votre environnement (box Internet, FAI, hébergeur) peut différer des exemples. Nous couvrons les cas les plus courants, mais vous devrez peut-être adapter certaines configurations. C'est normal et c'est une excellente opportunité d'apprentissage !

## Ressources Complémentaires

En complément de ce chapitre, voici des ressources utiles :

**Documentation officielle** :
- NGINX Ingress Controller : https://kubernetes.github.io/ingress-nginx/
- Kubernetes Ingress : https://kubernetes.io/docs/concepts/services-networking/ingress/

**Outils en ligne** :
- Vérification DNS : https://dnschecker.org
- Test de ports ouverts : https://www.yougetsignal.com/tools/open-ports/
- Vérification SSL : https://www.ssllabs.com/ssltest/

**Communautés** :
- Forum MicroK8s : https://discuss.kubernetes.io/c/microk8s
- Stack Overflow : Tag `kubernetes-ingress`

## Structure des Sections

Chaque section de ce chapitre suit une **structure cohérente** :

1. **Introduction** : Vue d'ensemble du sujet
2. **Concepts théoriques** : Explications avec analogies
3. **Configuration pratique** : Étapes détaillées avec commandes
4. **Vérification** : Comment tester que ça fonctionne
5. **Dépannage** : Solutions aux problèmes courants
6. **Bonnes pratiques** : Recommandations pour la production
7. **Points clés** : Résumé des éléments essentiels

Cette structure vous permet de :
- **Comprendre** avant d'agir
- **Suivre** des instructions claires
- **Vérifier** votre travail
- **Résoudre** les problèmes rapidement
- **Approfondir** quand vous le souhaitez

## Conventions Utilisées

Pour faciliter votre lecture, nous utilisons des conventions visuelles :

**Commandes à exécuter** :
```bash
microk8s kubectl get pods
```

**Fichiers de configuration** :
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
```

**Résultats attendus** :
```
NAME          READY   STATUS    RESTARTS   AGE
mon-pod       1/1     Running   0          5m
```

**Points importants** :
⚠️ Attention / Avertissement
✅ Recommandation / Bonne pratique
❌ À éviter / Erreur courante
🔑 Point clé à retenir
💡 Astuce / Conseil

## Un Mot d'Encouragement

Ce chapitre peut sembler intimidant au premier abord, avec ses 11 sections couvrant DNS, réseaux, sécurité, et routage. C'est normal de se sentir un peu dépassé !

Mais rappelez-vous : **des millions de personnes ont réussi avant vous**, et vous allez y arriver aussi. Nous avons décomposé chaque concept complexe en étapes simples et compréhensibles.

**Prenez votre temps**. Si une section vous semble difficile, relisez-la, faites des pauses, testez chaque commande. L'apprentissage n'est pas une course.

**Expérimentez**. N'ayez pas peur de tester, de casser des choses, de recommencer. C'est comme ça qu'on apprend vraiment. Votre environnement MicroK8s se remet facilement d'erreurs.

**Posez des questions**. Si quelque chose n'est pas clair, consultez la documentation, cherchez en ligne, demandez de l'aide sur les forums. La communauté Kubernetes est accueillante et prête à aider.

## Ce Qui Vous Attend au Chapitre 11

Une fois que vous aurez maîtrisé Ingress et le routage, vous serez prêt pour le **Chapitre 11 : Certificats SSL/TLS**. Là, nous ajouterons la cerise sur le gâteau en configurant des certificats HTTPS automatiques avec Let's Encrypt, pour que vos applications soient non seulement accessibles mais aussi **totalement sécurisées**.

## Êtes-vous Prêt ?

Vous avez maintenant une vue d'ensemble complète de ce qui vous attend dans ce chapitre. Vous comprenez :
- **Pourquoi** Ingress est important
- **Ce que** vous allez apprendre
- **Comment** le chapitre est organisé
- **Où** vous allez arriver

Il est temps de commencer cette aventure passionnante qui transformera votre cluster Kubernetes en une plateforme de production capable d'héberger des applications web réelles, accessibles et sécurisées !

Prenez une profonde inspiration, assurez-vous d'avoir du temps devant vous, et plongeons dans la **Section 10.1 : Concepts d'Ingress**.

**Bonne chance et bon apprentissage ! 🚀**

---

**Note** : N'oubliez pas de sauvegarder vos configurations au fur et à mesure. Un simple dossier Git avec vos manifestes YAML vous évitera bien des tracas en cas de problème. Nous vous rappellerons cette bonne pratique tout au long du chapitre.

⏭️ [Concepts d'Ingress](/10-ingress-et-routage/01-concepts-dingress.md)
