🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 11.3 Configuration de Cert-Manager

## Introduction : configurer la source de certificats

Cert-Manager est maintenant installé sur votre cluster, mais il ne peut pas encore émettre de certificats. Pourquoi ? Parce qu'il ne sait pas **où** et **comment** obtenir ces certificats.

C'est comme avoir un téléphone sans carte SIM : l'appareil fonctionne, mais il ne peut pas passer d'appels tant que vous ne l'avez pas configuré avec un opérateur.

Dans cette section, nous allons configurer Cert-Manager pour qu'il puisse communiquer avec **Let's Encrypt** (ou d'autres autorités de certification) et commencer à émettre des certificats automatiquement.

## Les concepts clés : Issuer et ClusterIssuer

Cert-Manager introduit deux ressources Kubernetes essentielles pour définir les sources de certificats :

### Issuer : émetteur au niveau du namespace

Un **Issuer** est une ressource qui définit comment obtenir des certificats **dans un namespace spécifique**.

**Analogie** : c'est comme un guichet de banque dans une succursale particulière. Ce guichet ne peut servir que les clients de cette succursale.

**Caractéristiques** :
- ✅ Portée limitée à un seul namespace
- ✅ Idéal pour des équipes ou des projets isolés
- ✅ Permet des configurations différentes par namespace
- ❌ Doit être créé dans chaque namespace où vous voulez des certificats

**Exemple d'usage** : vous avez un namespace `production` et un namespace `development`, chacun avec ses propres règles d'émission de certificats.

### ClusterIssuer : émetteur au niveau du cluster

Un **ClusterIssuer** est une ressource qui définit comment obtenir des certificats **pour l'ensemble du cluster**, quel que soit le namespace.

**Analogie** : c'est comme une banque centrale qui peut servir tous les clients de toutes les succursales.

**Caractéristiques** :
- ✅ Disponible pour tous les namespaces du cluster
- ✅ Configuration centralisée
- ✅ Idéal pour un lab personnel ou une configuration uniforme
- ✅ Une seule ressource pour tout le cluster

**Exemple d'usage** : vous voulez que toutes vos applications, peu importe leur namespace, utilisent la même configuration Let's Encrypt.

### Quelle ressource choisir ?

Pour un **lab personnel ou un cluster MicroK8s simple**, nous recommandons fortement l'utilisation de **ClusterIssuer** :
- Plus simple à gérer (une seule configuration)
- Fonctionne partout dans le cluster
- Moins de duplication de configuration

Pour un **environnement multi-tenant** ou avec des équipes séparées, les **Issuers** peuvent être plus appropriés pour donner plus de flexibilité.

Dans ce tutoriel, nous utiliserons principalement des **ClusterIssuers** pour leur simplicité.

## Let's Encrypt : staging vs production

Let's Encrypt propose deux environnements différents pour émettre des certificats :

### Environnement Staging (test)

L'environnement **staging** est un environnement de test qui :

**Avantages** :
- ✅ Limites de taux très élevées (vous pouvez tester autant que vous voulez)
- ✅ Permet de valider votre configuration sans risque
- ✅ Idéal pour l'apprentissage et les tests

**Inconvénients** :
- ❌ Les certificats générés ne sont **pas reconnus** par les navigateurs
- ❌ Affiche des avertissements de sécurité (c'est normal !)
- ❌ Ne doit **jamais** être utilisé en production

**Quand l'utiliser** : toujours en premier lors de la mise en place d'une nouvelle configuration, pour valider que tout fonctionne.

### Environnement Production

L'environnement **production** est l'environnement réel qui :

**Avantages** :
- ✅ Émet des certificats **valides** reconnus par tous les navigateurs
- ✅ Aucun avertissement de sécurité
- ✅ Certificats dignes de confiance

**Inconvénients** :
- ❌ **Limites de taux strictes** : 50 certificats par domaine par semaine, 5 échecs de validation par heure
- ❌ Si vous dépassez les limites, vous êtes bloqué pendant une semaine
- ❌ Les erreurs de configuration peuvent épuiser rapidement vos quotas

**Quand l'utiliser** : uniquement après avoir validé votre configuration avec l'environnement staging.

### Stratégie recommandée

1. **Phase 1 - Tests** : créez un ClusterIssuer staging, testez votre configuration
2. **Phase 2 - Validation** : vérifiez que les certificats staging sont bien émis (même s'ils ne sont pas valides)
3. **Phase 3 - Production** : une fois que tout fonctionne avec staging, créez un ClusterIssuer production

Cette approche vous évite de gaspiller vos précieux quotas production sur des erreurs de configuration.

## Les méthodes de validation : HTTP-01 vs DNS-01

Pour prouver que vous contrôlez un domaine, Let's Encrypt vous demande de relever un **défi** (challenge). Il existe deux méthodes principales :

### HTTP-01 : validation par HTTP

Let's Encrypt vérifie que vous contrôlez le domaine en demandant à accéder à une URL spécifique sur votre serveur.

**Comment ça fonctionne** :
1. Cert-Manager crée un pod temporaire avec un fichier de challenge
2. Let's Encrypt accède à `http://votre-domaine.com/.well-known/acme-challenge/TOKEN`
3. Si le fichier est accessible, le domaine est validé

**Avantages** :
- ✅ Configuration simple, pas besoin d'accès à votre DNS
- ✅ Fonctionne avec Ingress Controller
- ✅ Recommandé pour les débutants

**Inconvénients** :
- ❌ Nécessite que votre domaine soit accessible depuis Internet sur le port 80
- ❌ Ne fonctionne pas pour les certificats wildcard (`*.exemple.com`)
- ❌ Chaque sous-domaine nécessite une validation séparée

**Prérequis** :
- Votre cluster doit être accessible depuis Internet
- Le port 80 doit être ouvert
- Ingress Controller doit être configuré

### DNS-01 : validation par DNS

Let's Encrypt vérifie que vous contrôlez le domaine en demandant de créer un enregistrement DNS spécifique.

**Comment ça fonctionne** :
1. Cert-Manager crée un enregistrement TXT dans votre zone DNS
2. Let's Encrypt interroge le DNS pour vérifier cet enregistrement
3. Si l'enregistrement est présent, le domaine est validé

**Avantages** :
- ✅ Fonctionne même si votre cluster n'est pas accessible depuis Internet
- ✅ Supporte les certificats wildcard
- ✅ Peut valider des domaines sans serveur web

**Inconvénients** :
- ❌ Configuration plus complexe
- ❌ Nécessite un accès API à votre fournisseur DNS
- ❌ Plugins DNS spécifiques requis (Cloudflare, Route53, etc.)

**Prérequis** :
- Accès API à votre fournisseur DNS
- Plugin DNS compatible avec Cert-Manager

### Quelle méthode choisir ?

Pour un **lab personnel accessible depuis Internet** : HTTP-01 (plus simple)

Pour un **cluster interne** ou si vous voulez des **certificats wildcard** : DNS-01

Dans ce tutoriel, nous nous concentrerons principalement sur **HTTP-01** car c'est la méthode la plus simple pour débuter.

## Configuration d'un ClusterIssuer Let's Encrypt Staging

Créons notre premier ClusterIssuer pour l'environnement staging de Let's Encrypt.

### Créer le manifeste YAML

Créez un fichier nommé `clusterissuer-staging.yaml` :

```yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-staging
spec:
  acme:
    # URL du serveur ACME Let's Encrypt Staging
    server: https://acme-staging-v02.api.letsencrypt.org/directory

    # Adresse email pour les notifications importantes
    # Let's Encrypt vous enverra des emails avant l'expiration des certificats
    email: votre-email@exemple.com

    # Secret où sera stockée la clé privée du compte ACME
    privateKeySecretRef:
      name: letsencrypt-staging-account-key

    # Configuration du challenge HTTP-01
    solvers:
    - http01:
        ingress:
          class: public
```

### Comprendre chaque élément

Détaillons ce manifeste ligne par ligne :

#### apiVersion et kind

```yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
```

- **apiVersion** : version de l'API Cert-Manager (v1 est la version stable actuelle)
- **kind** : type de ressource, ici ClusterIssuer (disponible dans tout le cluster)

#### metadata

```yaml
metadata:
  name: letsencrypt-staging
```

- **name** : nom unique de votre ClusterIssuer. Vous référencerez ce nom dans vos certificats.
  Choisissez un nom explicite comme `letsencrypt-staging`, `letsencrypt-prod`, etc.

#### spec.acme.server

```yaml
spec:
  acme:
    server: https://acme-staging-v02.api.letsencrypt.org/directory
```

- **server** : URL du serveur ACME Let's Encrypt
- Pour staging : `https://acme-staging-v02.api.letsencrypt.org/directory`
- Pour production : `https://acme-v02.api.letsencrypt.org/directory`

**ACME** (Automatic Certificate Management Environment) est le protocole utilisé par Let's Encrypt pour automatiser l'émission de certificats.

#### spec.acme.email

```yaml
    email: votre-email@exemple.com
```

- **email** : votre adresse email réelle
- Let's Encrypt l'utilise pour :
  - Vous notifier des problèmes avec vos certificats
  - Vous avertir avant l'expiration (si le renouvellement automatique échoue)
  - Communications importantes sur le service

⚠️ **Important** : remplacez `votre-email@exemple.com` par votre vraie adresse email !

#### spec.acme.privateKeySecretRef

```yaml
    privateKeySecretRef:
      name: letsencrypt-staging-account-key
```

- **privateKeySecretRef** : nom du Secret Kubernetes où sera stockée la clé privée de votre compte ACME
- Cette clé est générée automatiquement par Cert-Manager
- Ne la créez pas manuellement, Cert-Manager s'en charge
- Le nom peut être ce que vous voulez, mais choisissez un nom explicite

Cette clé identifie votre compte auprès de Let's Encrypt. Conservez-la précieusement (mais pas de panique, Cert-Manager la gère pour vous).

#### spec.acme.solvers

```yaml
    solvers:
    - http01:
        ingress:
          class: public
```

- **solvers** : liste des méthodes de validation disponibles
- **http01** : utilise la validation HTTP-01
- **ingress.class** : classe d'Ingress à utiliser (généralement `public` ou `nginx`)

Le solver définit comment Cert-Manager prouvera que vous contrôlez le domaine.

### Appliquer la configuration

Une fois votre fichier créé et personnalisé (n'oubliez pas l'email !), appliquez-le :

```bash
kubectl apply -f clusterissuer-staging.yaml
```

Vous devriez voir :

```
clusterissuer.cert-manager.io/letsencrypt-staging created
```

### Vérifier le ClusterIssuer

Vérifiez que votre ClusterIssuer est bien créé :

```bash
kubectl get clusterissuer
```

Résultat attendu :

```
NAME                  READY   AGE
letsencrypt-staging   True    10s
```

La colonne **READY** doit afficher `True`. Cela signifie que Cert-Manager a pu créer le compte ACME avec succès.

Pour plus de détails :

```bash
kubectl describe clusterissuer letsencrypt-staging
```

Dans la sortie, vous devriez voir :

```
Status:
  Acme:
    Last Registered Email:  votre-email@exemple.com
    Uri:                    https://acme-staging-v02.api.letsencrypt.org/acme/acct/...
  Conditions:
    Status:                True
    Type:                  Ready
```

L'URI indique que votre compte ACME a bien été créé auprès de Let's Encrypt.

## Configuration d'un ClusterIssuer Let's Encrypt Production

Une fois que vous avez validé que tout fonctionne avec staging, créez votre ClusterIssuer production.

### Créer le manifeste YAML

Créez un fichier nommé `clusterissuer-production.yaml` :

```yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-prod
spec:
  acme:
    # URL du serveur ACME Let's Encrypt Production
    server: https://acme-v02.api.letsencrypt.org/directory

    # Même adresse email que pour staging
    email: votre-email@exemple.com

    # Secret différent pour la clé production
    privateKeySecretRef:
      name: letsencrypt-prod-account-key

    # Même configuration du challenge
    solvers:
    - http01:
        ingress:
          class: public
```

**Différences avec staging** :
- URL du serveur différente (production au lieu de staging)
- Nom du ClusterIssuer différent (`letsencrypt-prod`)
- Nom du Secret différent (`letsencrypt-prod-account-key`)

### Appliquer la configuration

```bash
kubectl apply -f clusterissuer-production.yaml
```

### Vérifier

```bash
kubectl get clusterissuer
```

Vous devriez maintenant voir **les deux** ClusterIssuers :

```
NAME                  READY   AGE
letsencrypt-staging   True    5m
letsencrypt-prod      True    10s
```

Parfait ! Vous avez maintenant deux sources de certificats configurées.

## Configuration DNS-01 (avancé)

Si vous avez besoin de certificats wildcard ou si votre cluster n'est pas accessible depuis Internet, vous devez utiliser la validation DNS-01.

### Exemple avec Cloudflare

Voici un exemple de ClusterIssuer utilisant Cloudflare pour la validation DNS :

```yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-dns-prod
spec:
  acme:
    server: https://acme-v02.api.letsencrypt.org/directory
    email: votre-email@exemple.com
    privateKeySecretRef:
      name: letsencrypt-dns-account-key
    solvers:
    - dns01:
        cloudflare:
          email: votre-email@cloudflare.com
          apiTokenSecretRef:
            name: cloudflare-api-token
            key: api-token
```

### Créer le Secret pour l'API Token

Vous devez d'abord créer un Secret contenant votre token API Cloudflare :

```bash
kubectl create secret generic cloudflare-api-token \
  --from-literal=api-token=VOTRE_TOKEN_CLOUDFLARE \
  -n cert-manager
```

### Autres fournisseurs DNS

Cert-Manager supporte de nombreux fournisseurs DNS :
- Cloudflare
- AWS Route53
- Google Cloud DNS
- Azure DNS
- OVH
- Et bien d'autres...

Consultez la [documentation officielle de Cert-Manager](https://cert-manager.io/docs/configuration/acme/dns01/) pour la configuration spécifique à votre fournisseur.

## Configuration pour certificats auto-signés

Pour le développement local, vous pouvez aussi configurer un Issuer pour créer des certificats auto-signés.

### Créer un ClusterIssuer auto-signé

Créez un fichier `clusterissuer-selfsigned.yaml` :

```yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: selfsigned-issuer
spec:
  selfSigned: {}
```

C'est tout ! Aucune configuration supplémentaire n'est nécessaire.

### Appliquer

```bash
kubectl apply -f clusterissuer-selfsigned.yaml
```

Ce ClusterIssuer génèrera des certificats auto-signés instantanément, sans aucune validation externe. Parfait pour le développement !

## Issuer au niveau du namespace (optionnel)

Si vous préférez utiliser un Issuer plutôt qu'un ClusterIssuer, voici un exemple :

```yaml
apiVersion: cert-manager.io/v1
kind: Issuer
metadata:
  name: letsencrypt-staging
  namespace: default  # L'Issuer n'existe que dans ce namespace
spec:
  acme:
    server: https://acme-staging-v02.api.letsencrypt.org/directory
    email: votre-email@exemple.com
    privateKeySecretRef:
      name: letsencrypt-staging-account-key
    solvers:
    - http01:
        ingress:
          class: public
```

**Différence** : ajout du champ `namespace` dans metadata. Cet Issuer ne fonctionnera que dans le namespace `default`.

## Bonnes pratiques de configuration

### 1. Utilisez des noms explicites

Donnez des noms clairs à vos Issuers :
- ✅ `letsencrypt-staging`, `letsencrypt-prod`
- ✅ `letsencrypt-dns-wildcard`
- ❌ `issuer1`, `issuer2`, `test`

### 2. Commencez toujours avec staging

Ne sautez jamais l'étape staging. Testez d'abord, puis passez en production. Les quotas Let's Encrypt sont précieux !

### 3. Utilisez une adresse email valide

Let's Encrypt vous contactera en cas de problème. Utilisez une adresse email que vous consultez régulièrement.

### 4. Séparez les comptes staging et production

Utilisez des noms de Secrets différents pour staging et production :
- `letsencrypt-staging-account-key`
- `letsencrypt-prod-account-key`

Cela évite toute confusion et permet de garder des comptes ACME séparés.

### 5. Documentez vos choix

Ajoutez des commentaires dans vos manifestes YAML pour expliquer votre configuration, surtout si vous utilisez des solvers avancés.

### 6. Sauvegardez vos clés de compte

Les Secrets contenant vos clés de compte ACME sont importants. Incluez-les dans vos sauvegardes de cluster.

## Limites de taux Let's Encrypt

Connaître les limites vous évitera des surprises :

### Environnement Production

- **50 certificats** par domaine enregistré par semaine
- **5 certificats dupliqués** par semaine (même set de domaines)
- **5 échecs de validation** par compte, par hostname, par heure
- **300 comptes** par adresse IP par 3 heures
- **10 comptes** par adresse IP par 3 heures (en cas de validation de domain)

### Environnement Staging

Les limites sont beaucoup plus élevées, vous pouvez tester autant que vous voulez.

### Stratégies pour éviter les limites

1. **Testez avec staging** : ne passez en production qu'une fois sûr de votre configuration
2. **Groupez les sous-domaines** : utilisez des certificats SAN (Subject Alternative Names) ou wildcard
3. **Surveillez vos renouvellements** : Cert-Manager renouvelle automatiquement, mais vérifiez que ça fonctionne
4. **Évitez les recreates** : ne supprimez pas et ne recréez pas constamment vos certificats

## Dépannage des ClusterIssuers

### Le ClusterIssuer n'est pas READY

```bash
kubectl describe clusterissuer letsencrypt-staging
```

Cherchez dans les Events des messages d'erreur. Causes courantes :
- Email invalide
- Problème de connectivité avec Let's Encrypt
- Configuration du solver incorrecte

### Secret de compte non créé

Si le Secret de clé privée n'est pas créé automatiquement, vérifiez :
- Les logs du pod cert-manager
- Les permissions RBAC

```bash
kubectl logs -n cert-manager deployment/cert-manager
```

### Erreur de validation HTTP-01

Si vous obtenez des erreurs de validation HTTP-01 :
- Vérifiez que votre cluster est accessible depuis Internet
- Vérifiez que le port 80 est ouvert
- Vérifiez que l'Ingress Controller fonctionne
- Vérifiez votre configuration DNS

## Vérification de la configuration complète

Pour vérifier que tout est en ordre :

```bash
# Lister tous les ClusterIssuers
kubectl get clusterissuer

# Vérifier les détails
kubectl describe clusterissuer letsencrypt-staging
kubectl describe clusterissuer letsencrypt-prod

# Vérifier les Secrets créés
kubectl get secret -n cert-manager | grep letsencrypt
```

Vous devriez voir :
- Vos ClusterIssuers avec `READY: True`
- Les Secrets de clés de compte créés automatiquement

## Récapitulatif des fichiers créés

À ce stade, vous devriez avoir créé :

```
clusterissuer-staging.yaml      # ClusterIssuer Let's Encrypt Staging
clusterissuer-production.yaml   # ClusterIssuer Let's Encrypt Production
clusterissuer-selfsigned.yaml   # (optionnel) ClusterIssuer auto-signé
```

Et dans votre cluster :

```
3 ClusterIssuers configurés
2-3 Secrets de comptes ACME créés automatiquement
```

## Concepts clés à retenir

- **Issuer** : émetteur de certificats au niveau d'un namespace
- **ClusterIssuer** : émetteur de certificats disponible dans tout le cluster
- **ACME** : protocole utilisé par Let's Encrypt pour automatiser les certificats
- **Staging** : environnement de test Let's Encrypt (limites élevées, certificats non valides)
- **Production** : environnement réel Let's Encrypt (limites strictes, certificats valides)
- **HTTP-01** : validation par accès HTTP (simple, nécessite exposition Internet)
- **DNS-01** : validation par DNS (plus complexe, supporte wildcards)
- **Solvers** : méthodes de validation du contrôle du domaine

## Prochaines étapes

Maintenant que Cert-Manager est configuré avec vos ClusterIssuers, vous êtes prêt à obtenir vos premiers certificats ! Dans la prochaine section, nous allons :

- Créer des ressources Certificate
- Obtenir automatiquement des certificats Let's Encrypt
- Sécuriser un Ingress avec HTTPS
- Observer le processus d'émission de certificats en action

La configuration est terminée, place à l'action !

---

⏭️ [Certificats Let's Encrypt automatiques](/11-certificats-ssl-tls/04-certificats-lets-encrypt-automatiques.md)
