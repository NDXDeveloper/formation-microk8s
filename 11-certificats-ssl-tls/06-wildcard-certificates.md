🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 11.6 Wildcard certificates

## Introduction : un certificat pour tous vos sous-domaines

Imaginez que vous ayez une application avec plusieurs services :
- `api.monapp.com`
- `web.monapp.com`
- `admin.monapp.com`
- `blog.monapp.com`
- `docs.monapp.com`

Jusqu'à présent, vous avez deux options :

1. **Certificat par sous-domaine** : créer un certificat séparé pour chaque service (5 certificats)
2. **Certificat SAN** : créer un seul certificat listant explicitement tous les sous-domaines

Mais que se passe-t-il si vous ajoutez un nouveau service `shop.monapp.com` ? Vous devez créer un nouveau certificat ou modifier le certificat SAN existant.

Les **certificats wildcard** offrent une troisième option, bien plus élégante : un seul certificat qui couvre automatiquement **tous les sous-domaines présents et futurs**.

**Analogie** : c'est comme avoir un pass universel pour tous les bâtiments d'un campus. Au lieu d'avoir un badge par bâtiment, vous avez un seul badge qui fonctionne partout.

## Qu'est-ce qu'un certificat wildcard ?

Un certificat **wildcard** (joker en français) utilise le symbole `*` (astérisque) pour représenter n'importe quel sous-domaine au premier niveau.

### Format

```
*.exemple.com
```

Ce certificat couvre :
- ✅ `api.exemple.com`
- ✅ `web.exemple.com`
- ✅ `admin.exemple.com`
- ✅ `n-importe-quoi.exemple.com`

Mais **ne couvre pas** :
- ❌ `exemple.com` (le domaine racine lui-même)
- ❌ `api.v2.exemple.com` (sous-domaine de second niveau)
- ❌ `autre-domaine.com`

### Certificat wildcard + domaine racine

Pour couvrir à la fois le domaine racine et tous les sous-domaines, vous devez spécifier les deux :

```yaml
dnsNames:
- exemple.com
- "*.exemple.com"
```

Cela couvre :
- ✅ `exemple.com` (racine)
- ✅ `www.exemple.com`
- ✅ `api.exemple.com`
- ✅ Tous les autres sous-domaines

## Avantages et inconvénients

### Avantages

✅ **Simplicité** : un seul certificat pour tous vos services
✅ **Flexibilité** : ajoutez de nouveaux sous-domaines sans modifier le certificat
✅ **Économie** : moins de certificats à gérer et renouveler
✅ **Automatisation** : parfait pour les environnements dynamiques
✅ **Moins de quotas** : un seul certificat Let's Encrypt au lieu de plusieurs

### Inconvénients

❌ **Configuration plus complexe** : nécessite DNS-01 avec Let's Encrypt
❌ **Accès API DNS requis** : vous devez avoir les credentials de votre fournisseur DNS
❌ **Sécurité** : si la clé privée est compromise, tous les sous-domaines sont affectés
❌ **Pas de sous-domaines de second niveau** : `*.exemple.com` ne couvre pas `api.v2.exemple.com`

### Quand utiliser un certificat wildcard ?

✅ **Situations appropriées :**
- Applications avec de nombreux sous-domaines (micro-services)
- Environnements dynamiques où de nouveaux services apparaissent régulièrement
- Architecture multi-tenant avec sous-domaines par client (`client1.monapp.com`, `client2.monapp.com`)
- SaaS avec sous-domaines dédiés

❌ **Situations inappropriées :**
- Une seule application avec un ou deux domaines → utilisez un certificat SAN classique
- Vous n'avez pas accès à l'API de votre fournisseur DNS
- Vous voulez isoler la sécurité par service

## Pourquoi DNS-01 est requis pour les wildcards Let's Encrypt

Let's Encrypt impose l'utilisation de la validation **DNS-01** pour les certificats wildcard. Pourquoi ?

### Le problème avec HTTP-01

Rappelez-vous, HTTP-01 fonctionne ainsi :
1. Let's Encrypt demande un fichier spécifique sur `http://domaine.com/.well-known/acme-challenge/TOKEN`
2. Vous servez ce fichier depuis votre serveur
3. Let's Encrypt vérifie qu'il peut y accéder

Mais avec un wildcard `*.exemple.com`, quel sous-domaine Let's Encrypt devrait-il vérifier ?
- `test.exemple.com` ?
- `random123.exemple.com` ?
- Tous les sous-domaines possibles (impossible !) ?

C'est techniquement infaisable. On ne peut pas prouver qu'on contrôle **tous** les sous-domaines possibles avec HTTP-01.

### La solution : DNS-01

Avec DNS-01, vous prouvez que vous contrôlez **le domaine entier** en créant un enregistrement DNS spécifique :

```
_acme-challenge.exemple.com. TXT "validation-token"
```

Si vous pouvez créer cet enregistrement, c'est que vous contrôlez la zone DNS entière, donc par extension tous les sous-domaines. CQFD !

## Prérequis pour les certificats wildcard

Avant de créer un certificat wildcard avec Let's Encrypt, vous devez :

1. **Contrôler la zone DNS** de votre domaine
2. **Avoir accès à l'API** de votre fournisseur DNS (Cloudflare, Route53, Google Cloud DNS, etc.)
3. **Créer des credentials API** (token, clé d'accès, etc.)
4. **Installer le plugin DNS** correspondant dans Cert-Manager (généralement automatique)

### Fournisseurs DNS supportés

Cert-Manager supporte de nombreux fournisseurs DNS :

**Principaux fournisseurs :**
- Cloudflare
- AWS Route53
- Google Cloud DNS
- Azure DNS
- DigitalOcean
- OVH
- Gandi
- GoDaddy
- Namecheap

**Plus de 30 fournisseurs au total** : consultez la [documentation officielle](https://cert-manager.io/docs/configuration/acme/dns01/) pour la liste complète.

Si votre fournisseur n'est pas supporté, vous pouvez utiliser des webhooks personnalisés.

## Configuration avec Cloudflare (exemple complet)

Cloudflare est l'un des fournisseurs les plus populaires et les plus simples à configurer. Voyons un exemple complet.

### Étape 1 : Obtenir un token API Cloudflare

1. Connectez-vous à votre compte Cloudflare
2. Allez dans **Mon profil** → **Jetons API**
3. Cliquez sur **Créer un jeton**
4. Sélectionnez le modèle **Modifier la zone DNS**
5. Configurez les permissions :
   - **Autorisations de zone** → **DNS** → **Modifier**
   - **Ressources de zone** → **Inclure** → **Zones spécifiques** → sélectionnez votre domaine
6. Cliquez sur **Continuer le résumé** puis **Créer un jeton**
7. **Copiez le token** et conservez-le précieusement (il ne sera affiché qu'une fois)

### Étape 2 : Créer un Secret Kubernetes avec le token

Créez un Secret contenant votre token API :

```bash
kubectl create secret generic cloudflare-api-token \
  --from-literal=api-token=VOTRE_TOKEN_CLOUDFLARE \
  -n cert-manager
```

Remplacez `VOTRE_TOKEN_CLOUDFLARE` par le token que vous venez de copier.

**Important** : ce Secret doit être créé dans le namespace `cert-manager` car c'est là que Cert-Manager l'utilisera.

Vérifiez :

```bash
kubectl get secret cloudflare-api-token -n cert-manager
```

### Étape 3 : Créer un ClusterIssuer avec DNS-01

Créez un fichier `clusterissuer-letsencrypt-dns.yaml` :

```yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-dns-prod
spec:
  acme:
    # Serveur Let's Encrypt Production
    server: https://acme-v02.api.letsencrypt.org/directory

    email: votre-email@exemple.com

    # Secret pour la clé du compte ACME
    privateKeySecretRef:
      name: letsencrypt-dns-account-key

    # Configuration du solver DNS-01 avec Cloudflare
    solvers:
    - dns01:
        cloudflare:
          # Vous pouvez aussi utiliser email + api-key (ancienne méthode)
          # email: votre-email@cloudflare.com
          # apiKeySecretRef:
          #   name: cloudflare-api-key
          #   key: api-key

          # Nouvelle méthode recommandée : API Token
          apiTokenSecretRef:
            name: cloudflare-api-token
            key: api-token
```

**Note** : il existe deux méthodes d'authentification avec Cloudflare :
- **API Token** (recommandé, plus sécurisé, permissions granulaires)
- **API Key** (ancienne méthode, accès global au compte)

Utilisez toujours l'API Token si possible.

### Étape 4 : Appliquer le ClusterIssuer

```bash
kubectl apply -f clusterissuer-letsencrypt-dns.yaml
```

Vérifiez :

```bash
kubectl get clusterissuer letsencrypt-dns-prod
```

Vous devriez voir `READY: True`.

### Étape 5 : Créer un certificat wildcard

Créez un fichier `certificate-wildcard.yaml` :

```yaml
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: wildcard-monapp
  namespace: default
spec:
  secretName: wildcard-monapp-tls
  issuerRef:
    name: letsencrypt-dns-prod
    kind: ClusterIssuer
  dnsNames:
  - monapp.com          # Domaine racine
  - "*.monapp.com"      # Tous les sous-domaines
```

Les guillemets autour de `"*.monapp.com"` sont importants en YAML pour éviter des problèmes de parsing.

### Étape 6 : Appliquer et observer

```bash
kubectl apply -f certificate-wildcard.yaml
```

Observez le processus :

```bash
# Voir l'état du certificat
kubectl get certificate wildcard-monapp

# Voir les détails
kubectl describe certificate wildcard-monapp

# Voir le challenge DNS-01
kubectl get challenge
```

Le processus prend généralement 1-3 minutes (un peu plus long que HTTP-01) car il faut :
1. Créer l'enregistrement DNS
2. Attendre la propagation DNS
3. Let's Encrypt vérifie l'enregistrement
4. Émettre le certificat

### Étape 7 : Vérifier

Une fois `READY: True` :

```bash
kubectl get certificate wildcard-monapp
```

```
NAME              READY   SECRET               AGE
wildcard-monapp   True    wildcard-monapp-tls  3m
```

Le certificat est prêt ! Vous pouvez maintenant l'utiliser pour n'importe quel sous-domaine.

## Utiliser un certificat wildcard avec Ingress

Maintenant que vous avez un certificat wildcard, utilisez-le pour plusieurs services.

### Ingress pour plusieurs sous-domaines

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: multi-services
  namespace: default
spec:
  ingressClassName: public
  tls:
  - hosts:
    - api.monapp.com
    - web.monapp.com
    - admin.monapp.com
    secretName: wildcard-monapp-tls  # Un seul certificat pour tous !
  rules:
  - host: api.monapp.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: api-service
            port:
              number: 80
  - host: web.monapp.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: web-service
            port:
              number: 80
  - host: admin.monapp.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: admin-service
            port:
              number: 80
```

Un seul Secret TLS (`wildcard-monapp-tls`) est utilisé pour tous les sous-domaines !

### Ajouter de nouveaux services sans modifier le certificat

Plus tard, vous ajoutez un nouveau service `shop.monapp.com` :

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: shop-service
  namespace: default
spec:
  ingressClassName: public
  tls:
  - hosts:
    - shop.monapp.com
    secretName: wildcard-monapp-tls  # Même certificat !
  rules:
  - host: shop.monapp.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: shop-service
            port:
              number: 80
```

Aucune modification du certificat n'est nécessaire. Il couvre déjà tous les sous-domaines grâce au wildcard !

## Configuration avec AWS Route53

Si vous utilisez AWS Route53 pour votre DNS, voici la configuration.

### Étape 1 : Créer un utilisateur IAM avec permissions Route53

Dans AWS IAM, créez un utilisateur avec la politique suivante :

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "route53:GetChange",
        "route53:ChangeResourceRecordSets",
        "route53:ListResourceRecordSets"
      ],
      "Resource": [
        "arn:aws:route53:::hostedzone/*",
        "arn:aws:route53:::change/*"
      ]
    },
    {
      "Effect": "Allow",
      "Action": "route53:ListHostedZones",
      "Resource": "*"
    }
  ]
}
```

Récupérez l'**Access Key ID** et la **Secret Access Key**.

### Étape 2 : Créer le Secret Kubernetes

```bash
kubectl create secret generic aws-route53-credentials \
  --from-literal=access-key-id=VOTRE_ACCESS_KEY_ID \
  --from-literal=secret-access-key=VOTRE_SECRET_ACCESS_KEY \
  -n cert-manager
```

### Étape 3 : Créer le ClusterIssuer

```yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-dns-route53
spec:
  acme:
    server: https://acme-v02.api.letsencrypt.org/directory
    email: votre-email@exemple.com
    privateKeySecretRef:
      name: letsencrypt-dns-route53-key
    solvers:
    - dns01:
        route53:
          region: eu-west-1  # Votre région AWS
          accessKeyID: VOTRE_ACCESS_KEY_ID  # Ou référence à un Secret
          secretAccessKeySecretRef:
            name: aws-route53-credentials
            key: secret-access-key
```

### Méthode alternative : utiliser les rôles IAM (pour EKS)

Si votre cluster est sur AWS EKS, vous pouvez utiliser IAM Roles for Service Accounts (IRSA) :

```yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-dns-route53
spec:
  acme:
    server: https://acme-v02.api.letsencrypt.org/directory
    email: votre-email@exemple.com
    privateKeySecretRef:
      name: letsencrypt-dns-route53-key
    solvers:
    - dns01:
        route53:
          region: eu-west-1
          # Pas besoin de credentials, utilise le rôle IAM du pod
```

Plus sécurisé car pas de credentials statiques !

## Configuration avec Google Cloud DNS

Pour Google Cloud DNS :

### Étape 1 : Créer un compte de service

Dans Google Cloud Console :
1. Allez dans **IAM & Admin** → **Comptes de service**
2. Créez un nouveau compte de service
3. Accordez le rôle **DNS Administrator**
4. Créez une clé JSON pour ce compte de service
5. Téléchargez le fichier JSON

### Étape 2 : Créer le Secret

```bash
kubectl create secret generic gcp-dns-admin \
  --from-file=key.json=chemin/vers/votre/cle.json \
  -n cert-manager
```

### Étape 3 : Créer le ClusterIssuer

```yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-dns-gcp
spec:
  acme:
    server: https://acme-v02.api.letsencrypt.org/directory
    email: votre-email@exemple.com
    privateKeySecretRef:
      name: letsencrypt-dns-gcp-key
    solvers:
    - dns01:
        cloudDNS:
          project: votre-projet-gcp
          serviceAccountSecretRef:
            name: gcp-dns-admin
            key: key.json
```

## Configuration avec OVH

OVH est un fournisseur européen populaire :

### Étape 1 : Créer des credentials API OVH

1. Allez sur https://eu.api.ovh.com/createToken/
2. Configurez les droits :
   - **GET** sur `/domain/zone/*`
   - **POST** sur `/domain/zone/*`
   - **DELETE** sur `/domain/zone/*`
3. Récupérez :
   - Application Key
   - Application Secret
   - Consumer Key

### Étape 2 : Créer le Secret

```bash
kubectl create secret generic ovh-credentials \
  --from-literal=application-key=VOTRE_APP_KEY \
  --from-literal=application-secret=VOTRE_APP_SECRET \
  --from-literal=consumer-key=VOTRE_CONSUMER_KEY \
  -n cert-manager
```

### Étape 3 : Créer le ClusterIssuer

```yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-dns-ovh
spec:
  acme:
    server: https://acme-v02.api.letsencrypt.org/directory
    email: votre-email@exemple.com
    privateKeySecretRef:
      name: letsencrypt-dns-ovh-key
    solvers:
    - dns01:
        webhook:
          groupName: acme.ovh.com
          solverName: ovh
          config:
            endpoint: ovh-eu  # ou ovh-ca, ovh-us selon votre compte
            applicationKeySecretRef:
              name: ovh-credentials
              key: application-key
            applicationSecretSecretRef:
              name: ovh-credentials
              key: application-secret
            consumerKeySecretRef:
              name: ovh-credentials
              key: consumer-key
```

**Note** : OVH nécessite l'installation d'un webhook supplémentaire. Consultez la documentation du projet [cert-manager-webhook-ovh](https://github.com/baarde/cert-manager-webhook-ovh).

## Certificats wildcard avec auto-signé

Bonne nouvelle : les certificats wildcard auto-signés sont **beaucoup plus simples** !

Pas besoin de DNS-01, pas de configuration complexe :

```yaml
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: dev-wildcard
  namespace: default
spec:
  secretName: dev-wildcard-tls
  issuerRef:
    name: selfsigned-issuer  # ClusterIssuer auto-signé
    kind: ClusterIssuer
  duration: 8760h  # 1 an
  dnsNames:
  - "*.dev.local"
  - dev.local
```

Appliquez et le certificat est créé instantanément ! Parfait pour le développement.

## Wildcard de second niveau

Un wildcard comme `*.exemple.com` ne couvre pas les sous-domaines de second niveau comme `api.v2.exemple.com`.

### Solution 1 : Wildcard de second niveau

Vous pouvez créer un wildcard pour le second niveau :

```yaml
dnsNames:
- "*.v2.exemple.com"
```

Cela couvre :
- ✅ `api.v2.exemple.com`
- ✅ `web.v2.exemple.com`
- ❌ `api.exemple.com` (premier niveau)

### Solution 2 : Plusieurs wildcards

Combinez plusieurs wildcards dans le même certificat :

```yaml
dnsNames:
- exemple.com
- "*.exemple.com"
- "*.v2.exemple.com"
- "*.staging.exemple.com"
```

Cela couvre :
- Le domaine racine
- Tous les sous-domaines de premier niveau
- Tous les sous-domaines sous `v2.exemple.com`
- Tous les sous-domaines sous `staging.exemple.com`

## Cas d'usage pratiques

### 1. Architecture micro-services

```yaml
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: microservices-wildcard
  namespace: production
spec:
  secretName: microservices-tls
  issuerRef:
    name: letsencrypt-dns-prod
    kind: ClusterIssuer
  dnsNames:
  - monapp.com
  - "*.monapp.com"
```

Couvre tous vos micro-services :
- `auth.monapp.com`
- `users.monapp.com`
- `orders.monapp.com`
- `payments.monapp.com`
- ... et tous les futurs services

### 2. Multi-tenant SaaS

```yaml
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: saas-wildcard
  namespace: production
spec:
  secretName: saas-tls
  issuerRef:
    name: letsencrypt-dns-prod
    kind: ClusterIssuer
  dnsNames:
  - "*.maplateforme.com"
```

Chaque client a son sous-domaine :
- `client1.maplateforme.com`
- `client2.maplateforme.com`
- `entrepriseA.maplateforme.com`

### 3. Environnements multiples

```yaml
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: multi-env-wildcard
  namespace: default
spec:
  secretName: multi-env-tls
  issuerRef:
    name: letsencrypt-dns-prod
    kind: ClusterIssuer
  dnsNames:
  - monapp.com
  - "*.monapp.com"
  - "*.staging.monapp.com"
  - "*.dev.monapp.com"
```

Couvre tous les environnements :
- Production : `api.monapp.com`, `web.monapp.com`
- Staging : `api.staging.monapp.com`, `web.staging.monapp.com`
- Développement : `api.dev.monapp.com`, `web.dev.monapp.com`

## Renouvellement des certificats wildcard

Le renouvellement fonctionne exactement comme pour les certificats normaux :
- 30 jours avant expiration, Cert-Manager lance le renouvellement
- Il recrée l'enregistrement DNS `_acme-challenge`
- Let's Encrypt valide et émet un nouveau certificat
- Le Secret est mis à jour automatiquement

Aucune action manuelle requise !

## Dépannage DNS-01

### L'enregistrement DNS n'est pas créé

Vérifiez les logs de Cert-Manager :

```bash
kubectl logs -n cert-manager deployment/cert-manager | grep DNS
```

Causes possibles :
- Credentials API incorrects
- Permissions insuffisantes
- Zone DNS incorrecte

### Timeout de propagation DNS

La propagation DNS peut prendre du temps. Par défaut, Cert-Manager attend 60 secondes.

Si vous avez des problèmes de propagation lente, augmentez le timeout :

```yaml
solvers:
- dns01:
    cloudflare:
      apiTokenSecretRef:
        name: cloudflare-api-token
        key: api-token
    # Attendre 2 minutes au lieu de 1
    cnameStrategy: Follow
    webhookTimeout: 120s
```

### Vérifier manuellement l'enregistrement DNS

Pendant le challenge, vérifiez que l'enregistrement est bien créé :

```bash
dig _acme-challenge.exemple.com TXT
# ou
nslookup -type=TXT _acme-challenge.exemple.com
```

Vous devriez voir un enregistrement TXT avec le token de validation.

### Problème de zone DNS

Assurez-vous que le domaine utilisé correspond à une zone DNS que vous contrôlez :

```bash
# Liste des zones (exemple Cloudflare)
curl -X GET "https://api.cloudflare.com/client/v4/zones" \
     -H "Authorization: Bearer VOTRE_TOKEN" \
     -H "Content-Type: application/json"
```

## Sécurité des credentials API

### Bonnes pratiques

1. **Permissions minimales** : donnez uniquement les permissions DNS nécessaires
2. **Rotation régulière** : changez les tokens/clés régulièrement
3. **Secrets Kubernetes** : ne commitez jamais les credentials dans Git
4. **RBAC** : limitez l'accès au namespace cert-manager
5. **Audit** : surveillez l'utilisation de vos API

### Utiliser des Secrets externes (avancé)

Pour plus de sécurité, utilisez un gestionnaire de secrets externe comme :
- HashiCorp Vault
- AWS Secrets Manager
- Google Secret Manager
- Azure Key Vault

Avec External Secrets Operator, vous synchronisez automatiquement les secrets :

```yaml
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: cloudflare-api-token
  namespace: cert-manager
spec:
  refreshInterval: 1h
  secretStoreRef:
    name: vault-backend
    kind: SecretStore
  target:
    name: cloudflare-api-token
  data:
  - secretKey: api-token
    remoteRef:
      key: cloudflare/api-token
```

## Monitoring et alertes

### Métriques Prometheus

Cert-Manager expose des métriques pour les certificats wildcard :

```
certmanager_certificate_expiration_timestamp_seconds{name="wildcard-monapp"}
certmanager_certificate_renewal_timestamp_seconds{name="wildcard-monapp"}
```

### Créer des alertes

Exemple d'alerte Prometheus pour les wildcards :

```yaml
groups:
- name: certificates
  rules:
  - alert: WildcardCertificateExpiringSoon
    expr: |
      certmanager_certificate_expiration_timestamp_seconds{name=~".*wildcard.*"} - time() < 604800
    for: 1h
    labels:
      severity: warning
    annotations:
      summary: "Certificat wildcard {{ $labels.name }} expire bientôt"
      description: "Le certificat expire dans moins de 7 jours"
```

## Comparaison : wildcard vs certificats multiples

| Aspect | Certificat Wildcard | Certificats Multiples |
|--------|--------------------|-----------------------|
| **Nombre de certificats** | 1 | N (un par domaine) |
| **Flexibilité** | Haute (nouveaux domaines gratuits) | Moyenne (faut ajouter au certificat) |
| **Configuration** | Complexe (DNS-01) | Simple (HTTP-01) |
| **Prérequis** | Accès API DNS | Exposition HTTP |
| **Sécurité** | Si compromis, tout est affecté | Isolation par certificat |
| **Quotas Let's Encrypt** | 1 certificat | N certificats |
| **Gestion** | Centralisée | Distribuée |
| **Coût** | Minimal | Plus élevé (plus de renouvellements) |

## Bonnes pratiques

### 1. Utilisez staging d'abord

Même avec DNS-01, testez d'abord avec Let's Encrypt staging :

```yaml
spec:
  acme:
    server: https://acme-staging-v02.api.letsencrypt.org/directory
```

### 2. Documentez vos credentials

Documentez où et comment obtenir les credentials API pour chaque fournisseur DNS.

### 3. Protégez les Secrets

Limitez l'accès aux Secrets contenant les credentials API :

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: cert-manager-secrets-reader
  namespace: cert-manager
rules:
- apiGroups: [""]
  resources: ["secrets"]
  verbs: ["get", "list"]
  resourceNames: ["cloudflare-api-token", "aws-route53-credentials"]
```

### 4. Un wildcard par environnement

Ne mélangez pas production et staging dans le même wildcard :

```yaml
# ❌ Mauvais - tout dans un seul certificat
dnsNames:
- "*.monapp.com"
- "*.staging.monapp.com"

# ✅ Bon - certificats séparés
---
# Production
dnsNames:
- "*.monapp.com"
---
# Staging
dnsNames:
- "*.staging.monapp.com"
```

### 5. Monitoring actif

Surveillez les renouvellements de wildcards car ils couvrent beaucoup de services.

### 6. Plan B

Ayez toujours un certificat de secours ou un moyen de basculer rapidement en cas de problème avec le wildcard.

## Concepts clés à retenir

- **Wildcard** : un certificat qui couvre tous les sous-domaines (`*.exemple.com`)
- **DNS-01 requis** : Let's Encrypt impose DNS-01 pour les wildcards
- **Accès API DNS** : vous devez avoir les credentials de votre fournisseur DNS
- **Un niveau uniquement** : `*.exemple.com` ne couvre pas `*.api.exemple.com`
- **Flexibilité** : ajoutez de nouveaux sous-domaines sans modifier le certificat
- **Auto-signés simples** : les wildcards auto-signés ne nécessitent pas DNS-01
- **Staging d'abord** : toujours tester avec l'environnement staging

## Prochaines étapes

Dans la prochaine section, nous allons voir comment configurer des redirections HTTPS automatiques pour forcer tous vos utilisateurs à utiliser des connexions sécurisées. Nous explorerons aussi comment gérer les cas particuliers et optimiser votre configuration SSL/TLS.

Les certificats wildcard sont un outil puissant pour simplifier la gestion de nombreux sous-domaines. Bien que leur configuration soit plus complexe que les certificats classiques, l'investissement en vaut la peine pour les applications à grande échelle !

---


⏭️ [Gestion des redirections HTTPS dans Ingress](/11-certificats-ssl-tls/07-gestion-des-redirections-https-dans-ingress.md)
