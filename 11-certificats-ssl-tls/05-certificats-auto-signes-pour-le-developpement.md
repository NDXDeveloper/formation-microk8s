🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 11.5 Certificats auto-signés pour le développement

## Introduction : quand Let's Encrypt n'est pas la solution

Dans la section précédente, nous avons vu comment obtenir des certificats valides avec Let's Encrypt. Mais il existe des situations où Let's Encrypt n'est pas adapté ou même impossible à utiliser :

- Votre cluster n'est **pas accessible depuis Internet** (lab personnel isolé, réseau interne)
- Vous développez en **local** sans nom de domaine
- Vous voulez tester rapidement **sans attendre** la validation Let's Encrypt
- Vous travaillez sur un **réseau d'entreprise** isolé
- Vous voulez des certificats **instantanés** pour des tests

Dans tous ces cas, les **certificats auto-signés** sont la solution idéale.

**Analogie** : c'est comme créer vous-même votre propre carte d'identité. Elle fonctionne pour vos besoins internes, mais personne d'autre ne la reconnaîtra comme officielle. Parfait pour le développement, inadapté pour la production publique !

## Qu'est-ce qu'un certificat auto-signé ?

Un certificat **auto-signé** (self-signed certificate) est un certificat que vous créez et signez vous-même, sans passer par une autorité de certification reconnue.

### Caractéristiques

**Avantages :**
- ✅ **Gratuit** (évidemment !)
- ✅ **Instantané** : création en quelques secondes
- ✅ **Pas de dépendance externe** : fonctionne complètement hors ligne
- ✅ **Pas de limite de taux** : créez autant de certificats que vous voulez
- ✅ **Pas besoin d'exposition Internet** : fonctionne sur n'importe quel réseau
- ✅ **Contrôle total** : vous maîtrisez tous les paramètres

**Inconvénients :**
- ❌ **Non reconnu par les navigateurs** : affiche un avertissement de sécurité
- ❌ **Nécessite une exception manuelle** : l'utilisateur doit accepter le certificat
- ❌ **Ne doit JAMAIS être utilisé en production publique**
- ❌ **Pas de chaîne de confiance** : vous êtes votre propre autorité

### Quand les utiliser

✅ **Situations appropriées :**
- Développement local
- Tests et CI/CD
- Environnements de staging internes
- Applications internes d'entreprise (avec distribution de la CA)
- Apprentissage et expérimentation
- PoC (Proof of Concept) rapides

❌ **À éviter absolument :**
- Sites web publics
- Applications accessibles par des clients externes
- Services de production
- Tout ce qui nécessite la confiance d'utilisateurs externes

## Configuration d'un ClusterIssuer selfSigned

La première étape consiste à créer un ClusterIssuer qui émettra des certificats auto-signés.

### Créer le manifeste

Créez un fichier nommé `clusterissuer-selfsigned.yaml` :

```yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: selfsigned-issuer
spec:
  selfSigned: {}
```

C'est tout ! La configuration est **extrêmement simple**. Pas d'URL de serveur, pas d'email, pas de configuration de challenge. Juste une déclaration que ce ClusterIssuer émettra des certificats auto-signés.

### Appliquer la configuration

```bash
kubectl apply -f clusterissuer-selfsigned.yaml
```

Résultat :

```
clusterissuer.cert-manager.io/selfsigned-issuer created
```

### Vérifier

```bash
kubectl get clusterissuer selfsigned-issuer
```

Vous devriez voir :

```
NAME                READY   AGE
selfsigned-issuer   True    5s
```

Le ClusterIssuer est immédiatement prêt à être utilisé !

## Créer un certificat auto-signé

Maintenant que nous avons notre ClusterIssuer, créons un certificat.

### Méthode 1 : Ressource Certificate explicite

Créez un fichier `certificate-selfsigned.yaml` :

```yaml
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: dev-tls-selfsigned
  namespace: default
spec:
  secretName: dev-tls-selfsigned-secret
  duration: 8760h  # 1 an (vous pouvez mettre ce que vous voulez)
  renewBefore: 720h  # 30 jours avant expiration
  issuerRef:
    name: selfsigned-issuer
    kind: ClusterIssuer
  commonName: dev.local
  dnsNames:
  - dev.local
  - "*.dev.local"  # Wildcard fonctionne parfaitement avec auto-signé !
  - app.dev.local
  - api.dev.local
```

### Comprendre les particularités

#### duration : durée libre

```yaml
duration: 8760h  # 1 an
```

Contrairement à Let's Encrypt qui impose 90 jours, avec les certificats auto-signés, vous choisissez la durée que vous voulez :
- `8760h` = 1 an
- `17520h` = 2 ans
- `87600h` = 10 ans

Pour le développement, une durée longue est pratique (moins de renouvellements).

#### Wildcards sans contrainte

```yaml
dnsNames:
- "*.dev.local"
```

Avec Let's Encrypt, les wildcards nécessitent la validation DNS-01. Avec les certificats auto-signés, ils fonctionnent directement, sans aucune contrainte !

#### Noms de domaine locaux

```yaml
commonName: dev.local
dnsNames:
- dev.local
- app.dev.local
```

Vous pouvez utiliser :
- Des TLD non standards (`.local`, `.dev`, `.test`, `.internal`)
- Des noms qui ne sont pas des vrais domaines Internet
- Des adresses IP (bien que non recommandé)
- N'importe quel nom que vous voulez

### Appliquer et vérifier

```bash
kubectl apply -f certificate-selfsigned.yaml
```

Vérifiez immédiatement :

```bash
kubectl get certificate dev-tls-selfsigned
```

Résultat en quelques secondes :

```
NAME                  READY   SECRET                      AGE
dev-tls-selfsigned    True    dev-tls-selfsigned-secret   5s
```

Contrairement à Let's Encrypt qui prend 1-2 minutes, les certificats auto-signés sont **instantanés** !

### Inspecter le certificat

```bash
kubectl describe certificate dev-tls-selfsigned
```

Vous verrez :

```
Status:
  Conditions:
    Last Transition Time:  2024-01-15T10:00:00Z
    Message:               Certificate is up to date and has not expired
    Reason:                Ready
    Status:                True
    Type:                  Ready
  Not After:               2025-01-15T10:00:00Z  # Expire dans 1 an
  Not Before:              2024-01-15T10:00:00Z
```

Le certificat est créé et valide !

## Utiliser un certificat auto-signé avec Ingress

Maintenant, utilisons ce certificat pour sécuriser une application.

### Exemple complet : application de développement

```yaml
---
# Déploiement
apiVersion: apps/v1
kind: Deployment
metadata:
  name: dev-app
  namespace: default
spec:
  replicas: 1
  selector:
    matchLabels:
      app: dev-app
  template:
    metadata:
      labels:
        app: dev-app
    spec:
      containers:
      - name: app
        image: nginxdemos/hello
        ports:
        - containerPort: 80

---
# Service
apiVersion: v1
kind: Service
metadata:
  name: dev-app
  namespace: default
spec:
  selector:
    app: dev-app
  ports:
  - port: 80
    targetPort: 80

---
# Ingress avec certificat auto-signé
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: dev-app
  namespace: default
  annotations:
    cert-manager.io/cluster-issuer: "selfsigned-issuer"
spec:
  ingressClassName: public
  tls:
  - hosts:
    - app.dev.local
    secretName: dev-app-tls  # Sera créé automatiquement
  rules:
  - host: app.dev.local
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: dev-app
            port:
              number: 80
```

### Appliquer

```bash
kubectl apply -f dev-app-complete.yaml
```

### Vérifier

Le certificat est créé automatiquement grâce aux annotations :

```bash
kubectl get certificate
```

Résultat :

```
NAME          READY   SECRET        AGE
dev-app-tls   True    dev-app-tls   10s
```

### Configurer /etc/hosts (pour les tests locaux)

Si vous utilisez des noms de domaine locaux comme `app.dev.local`, ajoutez-les à votre fichier `/etc/hosts` :

```bash
# Linux/macOS
sudo nano /etc/hosts

# Windows
# Éditez C:\Windows\System32\drivers\etc\hosts en tant qu'administrateur
```

Ajoutez :

```
127.0.0.1  app.dev.local
# Ou l'IP de votre cluster MicroK8s
192.168.1.100  app.dev.local
```

### Tester

```bash
curl -k https://app.dev.local
```

L'option `-k` indique à curl d'ignorer les avertissements de certificat (nécessaire pour les certificats auto-signés).

### Accéder depuis un navigateur

Ouvrez `https://app.dev.local` dans votre navigateur.

Vous verrez un avertissement de sécurité :
- **Chrome** : "Votre connexion n'est pas privée" (NET::ERR_CERT_AUTHORITY_INVALID)
- **Firefox** : "Avertissement : risque probable de sécurité"
- **Safari** : "Ce site n'est peut-être pas sécurisé"

C'est **normal et attendu** ! Cliquez sur "Avancé" puis "Continuer vers le site" (ou équivalent selon votre navigateur).

## Créer une autorité de certification personnelle (CA)

Pour une meilleure expérience de développement, vous pouvez créer votre propre **autorité de certification** (CA) et l'installer dans votre navigateur/système. Ainsi, tous les certificats signés par cette CA seront automatiquement approuvés.

### Étape 1 : Créer le certificat CA

```yaml
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: my-dev-ca
  namespace: cert-manager
spec:
  isCA: true  # C'est une CA !
  commonName: "Dev Lab CA"
  secretName: my-dev-ca-secret
  duration: 87600h  # 10 ans
  issuerRef:
    name: selfsigned-issuer
    kind: ClusterIssuer
```

Le champ **`isCA: true`** indique que ce certificat peut signer d'autres certificats.

Appliquez :

```bash
kubectl apply -f dev-ca.yaml
```

### Étape 2 : Créer un ClusterIssuer basé sur cette CA

Maintenant, créez un ClusterIssuer qui utilisera votre CA personnelle :

```yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: my-ca-issuer
spec:
  ca:
    secretName: my-dev-ca-secret
```

Appliquez :

```bash
kubectl apply -f clusterissuer-ca.yaml
```

### Étape 3 : Émettre des certificats signés par votre CA

Maintenant, tous les certificats émis avec `my-ca-issuer` seront signés par votre CA :

```yaml
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: app-signed-by-my-ca
  namespace: default
spec:
  secretName: app-ca-signed-tls
  issuerRef:
    name: my-ca-issuer  # Utilise notre CA
    kind: ClusterIssuer
  dnsNames:
  - app.dev.local
```

### Étape 4 : Extraire et installer le certificat CA

Pour que votre navigateur approuve automatiquement ces certificats, installez votre CA dans votre système.

#### Extraire le certificat CA

```bash
kubectl get secret my-dev-ca-secret -n cert-manager -o jsonpath='{.data.tls\.crt}' | base64 -d > my-dev-ca.crt
```

Vous obtenez un fichier `my-dev-ca.crt`.

#### Installer sur Linux

```bash
# Ubuntu/Debian
sudo cp my-dev-ca.crt /usr/local/share/ca-certificates/
sudo update-ca-certificates

# RHEL/CentOS
sudo cp my-dev-ca.crt /etc/pki/ca-trust/source/anchors/
sudo update-ca-trust
```

#### Installer sur macOS

```bash
sudo security add-trusted-cert -d -r trustRoot -k /Library/Keychains/System.keychain my-dev-ca.crt
```

Ou double-cliquez sur `my-dev-ca.crt` et ajoutez-le dans Trousseau d'accès avec "Toujours faire confiance".

#### Installer sur Windows

1. Double-cliquez sur `my-dev-ca.crt`
2. Cliquez sur "Installer le certificat"
3. Sélectionnez "Ordinateur local"
4. Choisissez "Placer tous les certificats dans le magasin suivant"
5. Sélectionnez "Autorités de certification racines de confiance"
6. Terminez l'installation

#### Installer dans Chrome/Edge (si nécessaire)

Si Chrome n'utilise pas le magasin système :

1. Paramètres → Confidentialité et sécurité → Sécurité
2. Gérer les certificats
3. Autorités → Importer
4. Sélectionnez `my-dev-ca.crt`
5. Cochez "Faire confiance à ce certificat pour identifier des sites Web"

#### Installer dans Firefox

Firefox utilise son propre magasin de certificats :

1. Paramètres → Vie privée et sécurité
2. Certificats → Afficher les certificats
3. Autorités → Importer
4. Sélectionnez `my-dev-ca.crt`
5. Cochez "Faire confiance à cette CA pour identifier des sites Web"

### Résultat

Après installation de votre CA, vos certificats auto-signés seront **automatiquement approuvés** ! Plus d'avertissement de sécurité dans le navigateur, vous aurez le cadenas vert.

## Certificats pour adresses IP

Vous pouvez aussi créer des certificats pour des adresses IP (bien que non recommandé) :

```yaml
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: ip-certificate
  namespace: default
spec:
  secretName: ip-tls-secret
  issuerRef:
    name: selfsigned-issuer
    kind: ClusterIssuer
  ipAddresses:
  - 192.168.1.100
  - 10.0.0.50
```

Notez l'utilisation de `ipAddresses` au lieu de `dnsNames`.

**Usage** : utile uniquement pour des tests très spécifiques. Privilégiez toujours les noms de domaine.

## Renouvellement des certificats auto-signés

Cert-Manager renouvelle aussi les certificats auto-signés avant leur expiration, tout comme pour Let's Encrypt.

### Configuration du renouvellement

```yaml
spec:
  duration: 8760h  # 1 an
  renewBefore: 720h  # Renouveler 30 jours avant expiration
```

Avec ces paramètres :
- Le certificat est valide 1 an
- Il sera renouvelé automatiquement 30 jours avant expiration
- Le renouvellement est instantané (quelques secondes)

### Forcer un renouvellement

Pour forcer le renouvellement immédiat :

```bash
# Option 1 : supprimer le Secret (Cert-Manager le recréera)
kubectl delete secret dev-tls-selfsigned-secret

# Option 2 : supprimer et recréer le Certificate
kubectl delete certificate dev-tls-selfsigned
kubectl apply -f certificate-selfsigned.yaml
```

Le nouveau certificat est créé en quelques secondes.

## Cas d'usage pratiques

### 1. Environnement de développement local

```yaml
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: local-dev
  namespace: default
spec:
  secretName: local-dev-tls
  issuerRef:
    name: selfsigned-issuer
    kind: ClusterIssuer
  duration: 17520h  # 2 ans
  dnsNames:
  - "*.local.dev"
  - local.dev
```

Utilisez le wildcard pour couvrir tous vos services locaux : `api.local.dev`, `web.local.dev`, etc.

### 2. Tests automatisés (CI/CD)

```yaml
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: ci-test-cert
  namespace: ci
spec:
  secretName: ci-test-tls
  issuerRef:
    name: selfsigned-issuer
    kind: ClusterIssuer
  duration: 24h  # Certificat de courte durée pour les tests
  dnsNames:
  - test.ci.internal
```

Pour les pipelines CI/CD, des certificats de courte durée (24h) sont parfaits.

### 3. Service interne d'entreprise

```yaml
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: internal-app
  namespace: production
spec:
  secretName: internal-app-tls
  issuerRef:
    name: my-ca-issuer  # CA d'entreprise personnalisée
    kind: ClusterIssuer
  duration: 8760h
  dnsNames:
  - app.internal.company.com
```

Avec une CA d'entreprise distribuée à tous les employés.

### 4. Staging interne avant production

```yaml
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: staging-app
  namespace: staging
spec:
  secretName: staging-app-tls
  issuerRef:
    name: selfsigned-issuer
    kind: ClusterIssuer
  dnsNames:
  - staging.myapp.internal
```

Pour tester avant de déployer en production avec Let's Encrypt.

## Comparaison : auto-signé vs Let's Encrypt

| Aspect | Certificat auto-signé | Let's Encrypt |
|--------|----------------------|---------------|
| **Vitesse d'émission** | Instantané (< 5s) | 1-2 minutes |
| **Reconnaissance navigateurs** | ❌ Non | ✅ Oui |
| **Besoin Internet** | ❌ Non | ✅ Oui (pour validation) |
| **Durée de validité** | Libre (1 jour à 10 ans) | 90 jours (fixe) |
| **Wildcards** | ✅ Facile | ⚠️ DNS-01 requis |
| **Limites de taux** | ✅ Aucune | ⚠️ 50 certs/semaine |
| **Usage production publique** | ❌ Jamais | ✅ Recommandé |
| **Usage développement** | ✅ Parfait | ⚠️ Possible mais overkill |
| **Configuration** | Très simple | Moyenne |
| **Coût** | Gratuit | Gratuit |

## Combinaison des approches

Vous pouvez utiliser plusieurs ClusterIssuers simultanément :

```yaml
# Pour le développement local
- name: dev
  annotations:
    cert-manager.io/cluster-issuer: "selfsigned-issuer"

# Pour le staging interne
- name: staging
  annotations:
    cert-manager.io/cluster-issuer: "my-ca-issuer"

# Pour la production
- name: production
  annotations:
    cert-manager.io/cluster-issuer: "letsencrypt-prod"
```

Chaque environnement utilise le ClusterIssuer le plus approprié !

## Sécurité et bonnes pratiques

### 1. Jamais en production publique

Les certificats auto-signés ne doivent **JAMAIS** être utilisés pour des services accessibles publiquement. Toujours utiliser Let's Encrypt ou une CA commerciale.

### 2. Documentation claire

Si vous utilisez des certificats auto-signés, documentez-le clairement :
```yaml
metadata:
  annotations:
    description: "Certificat auto-signé pour développement local uniquement"
```

### 3. Durée de vie appropriée

- **Développement** : 1-2 ans (moins de maintenance)
- **CI/CD** : 1 jour à 1 semaine (sécurité accrue)
- **Staging** : 3-6 mois (bon équilibre)

### 4. Nommage explicite

Utilisez des noms qui indiquent clairement qu'il s'agit de certificats auto-signés :
- `dev-selfsigned-tls`
- `local-dev-cert`
- `test-internal-tls`

### 5. Séparation des environnements

N'utilisez pas les mêmes certificats/CA entre développement et staging. Créez des CA distinctes si nécessaire.

### 6. Protection des CA

Si vous créez une CA personnelle, protégez son Secret :
```yaml
metadata:
  namespace: cert-manager  # Namespace restreint
  annotations:
    description: "CA racine - NE PAS SUPPRIMER"
```

Configurez RBAC pour limiter l'accès.

### 7. Rotation régulière en CI/CD

Pour les environnements de test automatisés, utilisez des certificats de courte durée qui sont renouvelés fréquemment.

## Dépannage

### Le certificat n'est pas créé

Vérifiez le ClusterIssuer :
```bash
kubectl get clusterissuer selfsigned-issuer
kubectl describe clusterissuer selfsigned-issuer
```

### Le navigateur refuse toujours le certificat

Après installation de la CA :
- Redémarrez le navigateur complètement
- Videz le cache SSL du navigateur
- Vérifiez que la CA est bien dans les autorités de confiance

### Le certificat expire trop vite

Augmentez la `duration` :
```yaml
spec:
  duration: 17520h  # 2 ans au lieu de 1
```

### La CA ne signe pas les certificats

Vérifiez que :
- Le certificat CA a bien `isCA: true`
- Le ClusterIssuer pointe vers le bon Secret
- Le Secret existe dans le bon namespace

```bash
kubectl get secret my-dev-ca-secret -n cert-manager
```

## Scripts utiles

### Script pour installer rapidement un ClusterIssuer auto-signé

```bash
#!/bin/bash
cat <<EOF | kubectl apply -f -
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: selfsigned-issuer
spec:
  selfSigned: {}
EOF
```

### Script pour créer une CA de développement complète

```bash
#!/bin/bash

# Créer le certificat CA
cat <<EOF | kubectl apply -f -
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: dev-ca
  namespace: cert-manager
spec:
  isCA: true
  commonName: "Dev Lab CA"
  secretName: dev-ca-secret
  duration: 87600h
  issuerRef:
    name: selfsigned-issuer
    kind: ClusterIssuer
EOF

# Attendre que la CA soit créée
echo "Waiting for CA certificate..."
kubectl wait --for=condition=Ready certificate/dev-ca -n cert-manager --timeout=30s

# Créer le ClusterIssuer basé sur la CA
cat <<EOF | kubectl apply -f -
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: dev-ca-issuer
spec:
  ca:
    secretName: dev-ca-secret
EOF

echo "Dev CA ClusterIssuer ready!"
```

## Concepts clés à retenir

- **Certificats auto-signés** : certificats que vous créez vous-même, sans CA externe
- **Instantanés** : création en quelques secondes, sans validation externe
- **Non reconnus** : affichent des avertissements dans les navigateurs
- **Parfaits pour le développement** : rapides, gratuits, sans contraintes
- **Jamais en production publique** : uniquement pour usage interne
- **CA personnelle** : créer votre propre autorité pour éviter les avertissements
- **Wildcards faciles** : pas besoin de DNS-01, fonctionnent directement
- **Durée libre** : vous choisissez la durée de validité

## Prochaines étapes

Maintenant que vous maîtrisez les certificats auto-signés pour le développement, nous allons explorer dans la prochaine section les certificats wildcard avec Let's Encrypt, qui permettent de couvrir tous vos sous-domaines avec un seul certificat.

Les certificats auto-signés sont un outil précieux dans votre arsenal de développement. Utilisez-les intelligemment pour accélérer votre workflow tout en sachant quand passer à des certificats valides !

---


⏭️ [Wildcard certificates](/11-certificats-ssl-tls/06-wildcard-certificates.md)
