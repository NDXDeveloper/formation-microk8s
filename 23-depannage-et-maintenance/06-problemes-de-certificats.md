🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 23.6 Problèmes de Certificats

## Introduction

Les certificats SSL/TLS sont essentiels pour sécuriser les communications dans Kubernetes. Ils permettent le chiffrement des données et l'authentification des services. Cependant, les problèmes de certificats sont parmi les plus frustrants à diagnostiquer, car les messages d'erreur peuvent être cryptiques et les causes multiples.

> **Pour les débutants** : Un certificat SSL/TLS est comme une carte d'identité numérique qui prouve qu'un site web ou un service est bien celui qu'il prétend être. Il permet aussi de chiffrer les communications pour que personne ne puisse les lire en transit. Dans Kubernetes, les certificats sont utilisés partout : pour accéder à l'API, pour les communications entre composants, et pour sécuriser vos applications web (HTTPS).

## Comprendre les Certificats (Bases)

### Qu'est-ce qu'un Certificat ?

Un certificat contient :
- **Une clé publique** : Partagée avec tout le monde
- **Une identité** : Le nom du service (domaine)
- **Une signature** : D'une autorité de certification (CA) qui garantit l'authenticité
- **Une date d'expiration** : Après laquelle le certificat n'est plus valide

### Types de Certificats

#### 1. Certificats Auto-signés

**Définition** : Créés et signés par vous-même, sans autorité de certification externe.

**Avantages** :
- Gratuits
- Rapides à créer
- Parfaits pour le développement et les tests

**Inconvénients** :
- Les navigateurs affichent un avertissement de sécurité
- Ne conviennent pas pour la production publique
- Doivent être explicitement approuvés

**Utilisation** : Développement local, réseaux internes

#### 2. Certificats d'une CA (Certificate Authority)

**Définition** : Signés par une autorité reconnue comme Let's Encrypt, DigiCert, etc.

**Avantages** :
- Reconnus par tous les navigateurs
- Pas d'avertissement de sécurité
- Adaptés à la production

**Inconvénients** :
- Nécessitent un domaine public
- Configuration plus complexe
- Let's Encrypt nécessite un renouvellement tous les 90 jours

**Utilisation** : Production, sites publics

### Certificats dans Kubernetes

Kubernetes utilise des certificats à plusieurs niveaux :

#### 1. Certificats du Cluster (Internes)

**Utilisés pour** :
- Communication avec l'API server
- Communication entre les composants (scheduler, controller-manager, kubelet)
- Authentification des utilisateurs

**Gestion** : MicroK8s les gère automatiquement

**Emplacement** : `/var/snap/microk8s/current/certs/`

**Durée de validité** : Généralement 1 an, renouvelés automatiquement

#### 2. Certificats d'Application (Ingress)

**Utilisés pour** :
- Sécuriser les connexions HTTPS de vos applications
- Protéger les communications avec vos services web

**Gestion** : Via cert-manager ou création manuelle

**Configuration** : Dans les ressources Ingress

## Types de Problèmes de Certificats

### Problème 1 : Certificats Expirés

Le certificat a dépassé sa date d'expiration et n'est plus valide.

### Problème 2 : Nom d'Hôte Incorrect

Le certificat est valide mais pour un nom de domaine différent.

### Problème 3 : Chaîne de Certification Incomplète

Il manque des certificats intermédiaires dans la chaîne de confiance.

### Problème 4 : Certificats Auto-signés Non Approuvés

Les clients ne font pas confiance au certificat auto-signé.

### Problème 5 : Problèmes avec Let's Encrypt

Échec de l'émission ou du renouvellement automatique.

## Problème 1 : Certificat Expiré

### Symptômes

**Dans le navigateur** :
```
NET::ERR_CERT_DATE_INVALID
Your connection is not private
```

**Avec curl** :
```bash
curl https://mon-app.example.com
# curl: (60) SSL certificate problem: certificate has expired
```

**Dans les logs** :
```
x509: certificate has expired or is not yet valid
```

### Diagnostic

#### Étape 1 : Vérifier la Date d'Expiration

**Avec openssl** :
```bash
# Vérifier un certificat distant
echo | openssl s_client -servername mon-app.example.com -connect mon-app.example.com:443 2>/dev/null | openssl x509 -noout -dates

# Sortie :
# notBefore=Oct 26 10:00:00 2024 GMT
# notAfter=Jan 24 10:00:00 2025 GMT
```

**Avec curl** :
```bash
curl -vI https://mon-app.example.com 2>&1 | grep -i "expire\|valid"
```

**Vérifier un certificat local** :
```bash
openssl x509 -in /path/to/cert.crt -noout -dates
```

#### Étape 2 : Vérifier les Certificats du Cluster

**Certificats MicroK8s** :
```bash
# Vérifier tous les certificats
for cert in /var/snap/microk8s/current/certs/*.crt; do
    echo "=== $cert ==="
    openssl x509 -in $cert -noout -subject -dates
    echo
done
```

**Vérifier l'API server** :
```bash
echo | openssl s_client -connect localhost:16443 2>/dev/null | openssl x509 -noout -dates
```

#### Étape 3 : Vérifier les Certificats Ingress

**Lister les secrets TLS** :
```bash
microk8s kubectl get secrets -A | grep tls
```

**Examiner un secret spécifique** :
```bash
# Extraire le certificat
microk8s kubectl get secret mon-tls-secret -n default -o jsonpath='{.data.tls\.crt}' | base64 -d | openssl x509 -noout -text

# Voir seulement les dates
microk8s kubectl get secret mon-tls-secret -n default -o jsonpath='{.data.tls\.crt}' | base64 -d | openssl x509 -noout -dates
```

### Solutions

#### Solution 1 : Renouveler Manuellement (Certificat Auto-signé)

Si vous utilisez un certificat auto-signé :

```bash
# 1. Générer une nouvelle clé privée
openssl genrsa -out tls.key 2048

# 2. Créer un certificat auto-signé (valide 365 jours)
openssl req -new -x509 -key tls.key -out tls.crt -days 365 \
  -subj "/CN=mon-app.example.com"

# 3. Créer ou mettre à jour le secret Kubernetes
microk8s kubectl create secret tls mon-tls-secret \
  --cert=tls.crt \
  --key=tls.key \
  --dry-run=client -o yaml | microk8s kubectl apply -f -
```

#### Solution 2 : Renouveler avec Cert-Manager

Si vous utilisez cert-manager :

```bash
# Forcer le renouvellement d'un certificat
microk8s kubectl annotate certificate mon-certificat -n default \
  cert-manager.io/issue-temporary-certificate="true"

# Ou supprimer le secret pour forcer la régénération
microk8s kubectl delete secret mon-tls-secret -n default

# Vérifier le statut du certificat
microk8s kubectl describe certificate mon-certificat -n default
```

#### Solution 3 : Renouveler les Certificats du Cluster

**MicroK8s gère automatiquement** les renouvellements, mais en cas de problème :

```bash
# Vérifier l'état
microk8s inspect

# En cas de certificats expirés, réinstaller (dernier recours)
sudo snap refresh microk8s --channel=<version>/stable
```

**Note** : Les certificats du cluster MicroK8s sont généralement valides 1 an et renouvelés automatiquement.

## Problème 2 : Erreur de Nom d'Hôte (Certificate Name Mismatch)

### Symptômes

**Dans le navigateur** :
```
NET::ERR_CERT_COMMON_NAME_INVALID
```

**Avec curl** :
```bash
curl https://mon-app.example.com
# curl: (60) SSL: certificate subject name 'autre-domaine.com' does not match target host name 'mon-app.example.com'
```

**Dans les logs** :
```
x509: certificate is valid for autre-domaine.com, not mon-app.example.com
```

### Diagnostic

#### Vérifier le Nom dans le Certificat

```bash
# Voir les noms pour lesquels le certificat est valide
echo | openssl s_client -servername mon-app.example.com -connect mon-app.example.com:443 2>/dev/null | openssl x509 -noout -text | grep -A1 "Subject Alternative Name"

# Sortie exemple :
# X509v3 Subject Alternative Name:
#     DNS:exemple.com, DNS:www.exemple.com
```

**Pour un certificat local** :
```bash
openssl x509 -in tls.crt -noout -text | grep -A1 "Subject Alternative Name"
```

### Solutions

#### Solution 1 : Créer un Certificat avec le Bon Nom

**Certificat auto-signé** :

```bash
# Créer une configuration pour SAN (Subject Alternative Names)
cat > req.conf <<EOF
[req]
distinguished_name = req_distinguished_name
x509_extensions = v3_req
prompt = no

[req_distinguished_name]
CN = mon-app.example.com

[v3_req]
keyUsage = keyEncipherment, dataEncipherment
extendedKeyUsage = serverAuth
subjectAltName = @alt_names

[alt_names]
DNS.1 = mon-app.example.com
DNS.2 = www.mon-app.example.com
DNS.3 = *.mon-app.example.com
EOF

# Générer le certificat
openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
  -keyout tls.key -out tls.crt -config req.conf

# Mettre à jour le secret
microk8s kubectl create secret tls mon-tls-secret \
  --cert=tls.crt \
  --key=tls.key \
  --dry-run=client -o yaml | microk8s kubectl apply -f -
```

#### Solution 2 : Corriger la Configuration Cert-Manager

**Certificate avec cert-manager** :

```yaml
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: mon-certificat
  namespace: default
spec:
  secretName: mon-tls-secret
  issuerRef:
    name: letsencrypt-prod
    kind: ClusterIssuer
  dnsNames:
  - mon-app.example.com      # Nom principal
  - www.mon-app.example.com  # Alternative
  - api.mon-app.example.com  # Autre alternative
```

Appliquer :
```bash
microk8s kubectl apply -f certificate.yaml

# Vérifier le statut
microk8s kubectl describe certificate mon-certificat
```

#### Solution 3 : Utiliser un Certificat Wildcard

Pour couvrir tous les sous-domaines :

```yaml
spec:
  dnsNames:
  - "*.mon-app.example.com"  # Wildcard pour tous les sous-domaines
  - mon-app.example.com      # Domaine racine
```

## Problème 3 : Certificats Auto-signés Bloqués

### Symptômes

**Dans le navigateur** :
```
NET::ERR_CERT_AUTHORITY_INVALID
Your connection is not private
```

**Avec curl** :
```bash
curl https://mon-app.example.com
# curl: (60) SSL certificate problem: self signed certificate
```

### Solutions

#### Solution 1 : Ignorer la Vérification (Développement Seulement)

**Avec curl** :
```bash
# -k ou --insecure ignore la vérification du certificat
curl -k https://mon-app.example.com
```

**Dans votre code (Python)** :
```python
import requests

# ATTENTION : Ne jamais utiliser en production !
response = requests.get(
    "https://mon-app.example.com",
    verify=False  # Désactive la vérification SSL
)
```

**ATTENTION** : N'utilisez JAMAIS `verify=False` ou `-k` en production !

#### Solution 2 : Ajouter le CA au Système

Pour éviter les avertissements, ajoutez le certificat CA à votre système :

**Sur Ubuntu/Debian** :
```bash
# Copier le certificat CA
sudo cp ca.crt /usr/local/share/ca-certificates/mon-ca.crt

# Mettre à jour les CA
sudo update-ca-certificates

# Vérifier
curl https://mon-app.example.com  # Devrait fonctionner sans -k
```

**Sur macOS** :
```bash
# Ajouter au trousseau
sudo security add-trusted-cert -d -r trustRoot -k /Library/Keychains/System.keychain ca.crt
```

**Sur Windows** :
```powershell
# Importer dans le magasin de certificats
Import-Certificate -FilePath ca.crt -CertStoreLocation Cert:\LocalMachine\Root
```

#### Solution 3 : Utiliser Let's Encrypt (Production)

Pour la production, utilisez des certificats reconnus :

```bash
# Activer cert-manager
microk8s enable cert-manager

# Attendre que cert-manager soit prêt
microk8s kubectl wait --for=condition=ready pod -n cert-manager -l app=cert-manager --timeout=90s
```

Créer un ClusterIssuer pour Let's Encrypt :

```yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-prod
spec:
  acme:
    # Serveur de production Let's Encrypt
    server: https://acme-v02.api.letsencrypt.org/directory
    # Votre email
    email: votre-email@example.com
    # Secret pour stocker la clé du compte
    privateKeySecretRef:
      name: letsencrypt-prod
    # Résolution via HTTP-01
    solvers:
    - http01:
        ingress:
          class: nginx
```

Appliquer :
```bash
microk8s kubectl apply -f cluster-issuer.yaml
```

Configurer l'Ingress pour utiliser Let's Encrypt :

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: mon-ingress
  annotations:
    cert-manager.io/cluster-issuer: "letsencrypt-prod"
spec:
  ingressClassName: nginx
  tls:
  - hosts:
    - mon-app.example.com
    secretName: mon-tls-secret  # Sera créé automatiquement
  rules:
  - host: mon-app.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: mon-service
            port:
              number: 80
```

## Problème 4 : Échec Let's Encrypt / Cert-Manager

### Symptômes

- Le certificat n'est pas émis
- Le secret TLS n'est pas créé
- Dans les événements : `Failed to create Order`, `Waiting for HTTP-01 challenge`

### Diagnostic

#### Étape 1 : Vérifier Cert-Manager

```bash
# Pods cert-manager en cours d'exécution ?
microk8s kubectl get pods -n cert-manager

# Logs de cert-manager
microk8s kubectl logs -n cert-manager -l app=cert-manager
```

#### Étape 2 : Vérifier le ClusterIssuer

```bash
# Lister les issuers
microk8s kubectl get clusterissuer

# État détaillé
microk8s kubectl describe clusterissuer letsencrypt-prod
```

**Ce qu'il faut vérifier** :
- Status doit être `Ready: True`
- Pas d'erreurs dans les conditions

#### Étape 3 : Vérifier la Certificate Request

```bash
# Lister les certificats
microk8s kubectl get certificate -A

# Détails du certificat
microk8s kubectl describe certificate mon-certificat -n default
```

**Regarder** :
- **Status** : Ready, Issuing, ou Failed
- **Events** : Messages d'erreur spécifiques
- **Conditions** : État de chaque étape

#### Étape 4 : Vérifier les CertificateRequests

```bash
# Lister les demandes de certificats
microk8s kubectl get certificaterequest -A

# Détails
microk8s kubectl describe certificaterequest -n default
```

#### Étape 5 : Vérifier les Orders et Challenges

```bash
# Orders
microk8s kubectl get order -n default

# Challenges
microk8s kubectl get challenge -n default

# Détails d'un challenge
microk8s kubectl describe challenge <challenge-name> -n default
```

### Solutions Courantes

#### Solution 1 : Problème de Validation HTTP-01

**Cause** : Let's Encrypt ne peut pas atteindre votre domaine pour valider.

**Vérifications** :

```bash
# 1. Votre domaine pointe-t-il vers la bonne IP ?
nslookup mon-app.example.com

# 2. Le port 80 est-il ouvert ?
curl http://mon-app.example.com/.well-known/acme-challenge/test

# 3. L'Ingress Controller est-il accessible ?
microk8s kubectl get svc -n ingress
```

**Solution** : Assurez-vous que :
- DNS pointe vers votre IP publique
- Port 80 est ouvert dans votre pare-feu
- Ingress Controller est accessible de l'extérieur

#### Solution 2 : Rate Limit Let's Encrypt

**Cause** : Trop de tentatives en peu de temps.

**Message d'erreur** :
```
too many certificates already issued for: example.com
```

**Solution** :
- Attendre (limite de 5 certificats/semaine pour un domaine)
- Utiliser l'environnement staging pour tester :

```yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-staging
spec:
  acme:
    # Serveur de staging (pas de limite de taux)
    server: https://acme-staging-v02.api.letsencrypt.org/directory
    email: votre-email@example.com
    privateKeySecretRef:
      name: letsencrypt-staging
    solvers:
    - http01:
        ingress:
          class: nginx
```

**Note** : Les certificats staging afficheront un avertissement dans le navigateur (c'est normal).

#### Solution 3 : Problème d'Email

**Cause** : Email invalide ou non fourni.

**Solution** : Vérifier que l'email dans le ClusterIssuer est valide :

```yaml
spec:
  acme:
    email: votre-email-valide@example.com  # Email réel requis
```

#### Solution 4 : Redémarrer Cert-Manager

```bash
# Redémarrer tous les pods cert-manager
microk8s kubectl rollout restart deployment -n cert-manager

# Ou supprimer les pods (ils redémarreront automatiquement)
microk8s kubectl delete pod -n cert-manager --all
```

#### Solution 5 : Recréer le Certificate

```bash
# Supprimer le certificat
microk8s kubectl delete certificate mon-certificat -n default

# Supprimer les ressources liées
microk8s kubectl delete certificaterequest -n default --all
microk8s kubectl delete order -n default --all
microk8s kubectl delete challenge -n default --all

# Supprimer le secret (sera recréé)
microk8s kubectl delete secret mon-tls-secret -n default

# Recréer le certificat
microk8s kubectl apply -f certificate.yaml
```

## Problème 5 : Erreurs TLS/SSL Génériques

### Symptômes Courants

#### 1. "SSL handshake failed"

**Causes possibles** :
- Versions TLS incompatibles
- Ciphers non supportés
- Configuration serveur incorrecte

**Diagnostic** :
```bash
# Tester la connexion TLS
openssl s_client -connect mon-app.example.com:443 -tls1_2

# Voir les ciphers supportés
nmap --script ssl-enum-ciphers -p 443 mon-app.example.com
```

#### 2. "Certificate chain incomplete"

**Cause** : Il manque des certificats intermédiaires.

**Diagnostic** :
```bash
# Vérifier la chaîne complète
openssl s_client -showcerts -connect mon-app.example.com:443 </dev/null

# Tester la chaîne avec SSL Labs (en ligne)
# https://www.ssllabs.com/ssltest/
```

**Solution** : Fournir la chaîne complète dans le secret :

```bash
# Concaténer le certificat et la chaîne
cat server.crt intermediate.crt > fullchain.crt

# Créer le secret avec la chaîne complète
microk8s kubectl create secret tls mon-tls-secret \
  --cert=fullchain.crt \
  --key=server.key
```

#### 3. "Protocol version mismatch"

**Cause** : Client et serveur ne supportent pas les mêmes versions de TLS.

**Solution** : Configurer les versions TLS dans l'Ingress :

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: mon-ingress
  annotations:
    nginx.ingress.kubernetes.io/ssl-protocols: "TLSv1.2 TLSv1.3"
    nginx.ingress.kubernetes.io/ssl-ciphers: "HIGH:!aNULL:!MD5"
```

## Outils de Diagnostic des Certificats

### 1. openssl

**Outil universel pour les certificats.**

```bash
# Voir le contenu d'un certificat
openssl x509 -in cert.crt -text -noout

# Vérifier un certificat et sa clé correspondent
openssl x509 -noout -modulus -in cert.crt | openssl md5
openssl rsa -noout -modulus -in key.key | openssl md5
# Les deux doivent être identiques

# Tester une connexion SSL
openssl s_client -connect domain.com:443

# Extraire le certificat d'un serveur
echo | openssl s_client -connect domain.com:443 2>/dev/null | openssl x509 > server.crt

# Vérifier la date d'expiration
openssl x509 -in cert.crt -noout -enddate

# Vérifier pour quel domaine le certificat est valide
openssl x509 -in cert.crt -noout -text | grep DNS
```

### 2. curl

**Tester rapidement les certificats d'un endpoint HTTPS.**

```bash
# Voir les détails SSL
curl -vI https://mon-app.example.com

# Ignorer les erreurs de certificat (test seulement)
curl -k https://mon-app.example.com

# Tester avec un CA personnalisé
curl --cacert ca.crt https://mon-app.example.com

# Temps de connexion SSL
curl -w "%{time_connect} %{time_starttransfer}\n" -o /dev/null -s https://mon-app.example.com
```

### 3. kubectl cert-manager plugin

**Plugin kubectl pour cert-manager.**

Installation :
```bash
# Télécharger
wget https://github.com/cert-manager/cert-manager/releases/latest/download/kubectl-cert_manager-linux-amd64.tar.gz

# Extraire
tar -xzf kubectl-cert_manager-linux-amd64.tar.gz

# Installer
sudo mv kubectl-cert_manager /usr/local/bin/

# Utiliser
microk8s kubectl cert-manager status certificate mon-certificat -n default
microk8s kubectl cert-manager renew mon-certificat -n default
```

### 4. Online Tools

**SSL Labs Server Test** : https://www.ssllabs.com/ssltest/
- Analyse complète de la configuration SSL
- Note de sécurité (A+, A, B, etc.)
- Identifie les vulnérabilités

**Certificate Decoder** : https://www.sslshopper.com/certificate-decoder.html
- Décode et affiche le contenu d'un certificat
- Vérifie la chaîne de certification

## Workflows de Dépannage

### Workflow 1 : Certificat Expiré

```bash
# 1. Identifier le problème
curl https://mon-app.example.com
# Erreur : certificate has expired

# 2. Vérifier la date d'expiration
microk8s kubectl get secret mon-tls-secret -o jsonpath='{.data.tls\.crt}' | base64 -d | openssl x509 -noout -dates

# 3a. Si cert-manager : forcer le renouvellement
microk8s kubectl delete secret mon-tls-secret
# cert-manager le recréera automatiquement

# 3b. Si manuel : créer un nouveau certificat
openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
  -keyout tls.key -out tls.crt \
  -subj "/CN=mon-app.example.com"

# 4. Mettre à jour le secret
microk8s kubectl create secret tls mon-tls-secret \
  --cert=tls.crt --key=tls.key \
  --dry-run=client -o yaml | microk8s kubectl apply -f -

# 5. Vérifier
curl https://mon-app.example.com
```

### Workflow 2 : Let's Encrypt Ne Fonctionne Pas

```bash
# 1. Vérifier cert-manager
microk8s kubectl get pods -n cert-manager
# Tous doivent être Running

# 2. Vérifier le ClusterIssuer
microk8s kubectl describe clusterissuer letsencrypt-prod
# Status doit être Ready

# 3. Vérifier le Certificate
microk8s kubectl describe certificate mon-certificat
# Regarder Events pour les erreurs

# 4. Vérifier les Challenges
microk8s kubectl get challenge
microk8s kubectl describe challenge <challenge-name>
# Identifier le problème (souvent HTTP-01 challenge)

# 5. Vérifier l'accessibilité
curl http://mon-app.example.com/.well-known/acme-challenge/test
# Doit être accessible de l'extérieur

# 6. Si bloqué, supprimer et recréer
microk8s kubectl delete certificate mon-certificat
microk8s kubectl delete secret mon-tls-secret
microk8s kubectl apply -f certificate.yaml

# 7. Surveiller la progression
microk8s kubectl get certificate -w
```

### Workflow 3 : Nom d'Hôte Incorrect

```bash
# 1. Identifier le problème
curl https://wrong-domain.example.com
# Erreur : certificate is valid for autre-domain.com

# 2. Vérifier le certificat actuel
microk8s kubectl get secret mon-tls-secret -o jsonpath='{.data.tls\.crt}' | base64 -d | openssl x509 -noout -text | grep DNS

# 3a. Avec cert-manager : corriger la Certificate
cat > certificate.yaml <<EOF
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: mon-certificat
spec:
  secretName: mon-tls-secret
  issuerRef:
    name: letsencrypt-prod
    kind: ClusterIssuer
  dnsNames:
  - wrong-domain.example.com  # Ajouter le bon domaine
  - www.wrong-domain.example.com
EOF

microk8s kubectl apply -f certificate.yaml

# 3b. Manuel : créer un nouveau certificat avec SAN
# (voir Solution 1 du Problème 2)

# 4. Vérifier
curl https://wrong-domain.example.com
```

## Bonnes Pratiques

### 1. Utiliser Cert-Manager en Production

✅ **Recommandé** :
```bash
microk8s enable cert-manager
```

**Pourquoi** :
- Renouvellement automatique
- Gestion centralisée
- Intégration Let's Encrypt
- Évite les certificats expirés

### 2. Configurer le Renouvellement Automatique

```yaml
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: mon-certificat
spec:
  # Renouveler 30 jours avant l'expiration
  renewBefore: 720h  # 30 jours
  secretName: mon-tls-secret
  issuerRef:
    name: letsencrypt-prod
    kind: ClusterIssuer
  dnsNames:
  - mon-app.example.com
```

### 3. Surveiller les Expirations

**Script de surveillance** :

```bash
#!/bin/bash
# check-cert-expiry.sh

# Vérifier tous les secrets TLS
microk8s kubectl get secrets --all-namespaces -o json | \
jq -r '.items[] | select(.type=="kubernetes.io/tls") |
  .metadata.namespace + "/" + .metadata.name + " " + .data."tls.crt"' | \
while read ns_name cert; do
    namespace=$(echo $ns_name | cut -d'/' -f1)
    name=$(echo $ns_name | cut -d'/' -f2)

    expiry=$(echo $cert | base64 -d | openssl x509 -noout -enddate | cut -d'=' -f2)
    expiry_epoch=$(date -d "$expiry" +%s)
    now_epoch=$(date +%s)
    days_left=$(( ($expiry_epoch - $now_epoch) / 86400 ))

    echo "$namespace/$name: $days_left jours restants"

    if [ $days_left -lt 30 ]; then
        echo "  ⚠️  ATTENTION: Expire bientôt!"
    fi
done
```

### 4. Utiliser Staging pour les Tests

```yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-staging
spec:
  acme:
    server: https://acme-staging-v02.api.letsencrypt.org/directory
    # Pas de limite de taux, parfait pour tester
```

### 5. Documenter les Certificats

Gardez une trace de :
- Quels domaines utilisent quels certificats
- Dates d'expiration
- Méthode de renouvellement (auto/manuel)
- Contacts responsables

**Exemple de documentation** :

```markdown
## Certificats

| Domaine | Type | Expiration | Renouvellement | Responsable |
|---------|------|------------|----------------|-------------|
| app.example.com | Let's Encrypt | Auto | Cert-Manager | DevOps |
| admin.example.com | Auto-signé | 2026-01-01 | Manuel | Admin |
| api.example.com | Let's Encrypt | Auto | Cert-Manager | DevOps |
```

### 6. Séparer Dev et Prod

**Développement** : Certificats auto-signés
```bash
# Rapide et facile
openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
  -keyout dev-tls.key -out dev-tls.crt \
  -subj "/CN=dev.example.local"
```

**Production** : Let's Encrypt via cert-manager
```yaml
issuerRef:
  name: letsencrypt-prod
  kind: ClusterIssuer
```

### 7. Sauvegarder les Certificats Importants

```bash
# Exporter un certificat et sa clé
microk8s kubectl get secret mon-tls-secret -o yaml > mon-tls-backup.yaml

# Restaurer
microk8s kubectl apply -f mon-tls-backup.yaml
```

**Attention** : Les clés privées sont sensibles, sécurisez les backups !

## Checklist de Dépannage Certificats

### Vérifications de Base

- [ ] Le certificat existe-t-il ? (`kubectl get secret mon-tls-secret`)
- [ ] Est-il référencé dans l'Ingress ?
- [ ] Est-il expiré ? (`openssl x509 -in cert.crt -noout -dates`)
- [ ] Le nom d'hôte correspond-il ? (`openssl x509 -in cert.crt -noout -text | grep DNS`)

### Si Cert-Manager

- [ ] Cert-manager est-il Running ? (`kubectl get pods -n cert-manager`)
- [ ] Le ClusterIssuer est-il Ready ? (`kubectl describe clusterissuer`)
- [ ] Le Certificate est-il Ready ? (`kubectl describe certificate`)
- [ ] Y a-t-il des Challenges bloqués ? (`kubectl get challenge`)

### Si Let's Encrypt

- [ ] Le domaine est-il accessible publiquement ?
- [ ] Le port 80 est-il ouvert ?
- [ ] Le DNS pointe-t-il vers la bonne IP ?
- [ ] Avez-vous atteint les limites de taux ?

### Problèmes Courants

- [ ] Email valide dans le ClusterIssuer ?
- [ ] `ingressClassName: nginx` dans l'Ingress ?
- [ ] Annotation `cert-manager.io/cluster-issuer` présente ?
- [ ] Le service backend fonctionne-t-il ?

## Messages d'Erreur Courants et Solutions

| Erreur | Cause Probable | Solution |
|--------|---------------|----------|
| `certificate has expired` | Certificat expiré | Renouveler le certificat |
| `certificate is valid for X, not Y` | Mauvais nom d'hôte | Créer certificat avec bon SAN |
| `self signed certificate` | Cert auto-signé non approuvé | Ajouter CA ou utiliser Let's Encrypt |
| `certificate signed by unknown authority` | CA non reconnue | Ajouter CA au système |
| `Waiting for HTTP-01 challenge` | Domaine inaccessible | Vérifier DNS, firewall, ports |
| `too many certificates issued` | Limite Let's Encrypt | Attendre ou utiliser staging |
| `unable to get local issuer certificate` | Chaîne incomplète | Fournir fullchain.crt |

## Résumé

### Commandes Essentielles

```bash
# Vérifier un certificat distant
echo | openssl s_client -connect domain.com:443 2>/dev/null | openssl x509 -noout -dates

# Vérifier un certificat local
openssl x509 -in cert.crt -noout -text

# Vérifier un secret Kubernetes
kubectl get secret mon-tls-secret -o jsonpath='{.data.tls\.crt}' | base64 -d | openssl x509 -noout -dates

# État cert-manager
kubectl get certificate,certificaterequest,order,challenge -A

# Logs cert-manager
kubectl logs -n cert-manager -l app=cert-manager

# Forcer renouvellement
kubectl delete secret mon-tls-secret
```

### Points Clés

1. **Certificats expirés** : La cause n°1 des problèmes - utilisez cert-manager pour l'auto-renouvellement
2. **Nom d'hôte** : Doit correspondre exactement (utilisez SAN pour plusieurs noms)
3. **Let's Encrypt** : Nécessite accès HTTP-01 depuis internet
4. **Auto-signés** : OK pour dev, pas pour production
5. **Cert-manager** : Simplifie grandement la gestion
6. **Monitoring** : Surveillez les dates d'expiration
7. **Staging** : Testez avec Let's Encrypt staging d'abord

**Prochaine étape** : Section 23.7 - Problèmes de stockage

---


⏭️ [Problèmes de stockage](/23-depannage-et-maintenance/07-problemes-de-stockage.md)
