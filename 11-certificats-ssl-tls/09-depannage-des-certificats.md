🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 11.9 Dépannage des certificats

## Introduction : une approche méthodique

Les certificats SSL/TLS peuvent parfois poser problème. Un certificat qui ne s'émet pas, qui expire alors qu'il devrait être renouvelé, ou un navigateur qui affiche des erreurs de sécurité... Ces situations sont frustrantes, mais heureusement, la plupart des problèmes ont des causes bien identifiées et des solutions simples.

**Analogie** : dépanner un certificat, c'est comme diagnostiquer pourquoi votre voiture ne démarre pas. Il faut vérifier méthodiquement chaque élément de la chaîne : la batterie (ClusterIssuer), le démarreur (Certificate), l'essence (validation), jusqu'à trouver le problème.

Cette section vous guide à travers les problèmes les plus courants et leurs solutions. Nous allons voir comment :
- Diagnostiquer rapidement un problème
- Utiliser les bons outils de diagnostic
- Lire et comprendre les logs
- Résoudre les erreurs courantes
- Prévenir les problèmes futurs

## Méthodologie de dépannage

Avant de plonger dans les problèmes spécifiques, adoptons une méthodologie systématique.

### Les 5 questions essentielles

Face à un problème de certificat, posez-vous ces questions dans l'ordre :

1. **Le ClusterIssuer est-il fonctionnel ?**
   - C'est la source qui émet les certificats

2. **Le Certificate est-il créé correctement ?**
   - Vérifiez sa configuration et son état

3. **La validation (challenge) fonctionne-t-elle ?**
   - HTTP-01 : votre cluster est-il accessible depuis Internet ?
   - DNS-01 : les enregistrements DNS sont-ils créés ?

4. **Le Secret est-il créé avec le certificat ?**
   - Le certificat doit être stocké dans un Secret

5. **L'Ingress utilise-t-il le bon Secret ?**
   - Le Secret doit être correctement référencé

### Approche en entonnoir

Procédez du plus général au plus spécifique :

```
1. Cluster Kubernetes ✓
   ↓
2. Cert-Manager installé et fonctionnel ✓
   ↓
3. ClusterIssuer READY ✓
   ↓
4. Certificate créé ✓
   ↓
5. CertificateRequest générée ✓
   ↓
6. Order créé ✓
   ↓
7. Challenge(s) résolu(s) ✓
   ↓
8. Secret créé avec certificat ✓
   ↓
9. Ingress utilise le Secret ✓
   ↓
10. Certificat valide et reconnu ✓
```

À chaque étape, si vous trouvez un ❌, c'est là que se situe votre problème.

## Problème 1 : Le certificat ne s'émet jamais (READY: False)

### Symptômes

```bash
kubectl get certificate
```

```
NAME        READY   SECRET      AGE
app-tls     False   app-tls     10m
```

Le certificat reste `False` après plusieurs minutes.

### Diagnostic

#### Étape 1 : Inspecter le Certificate

```bash
kubectl describe certificate app-tls
```

Regardez la section `Status` et `Events`. Les messages d'erreur y sont visibles.

**Messages courants** :

```
Waiting for CertificateRequest "app-tls-xxxxx" to complete
```
→ Le processus est en cours, patientez.

```
Failed to create Order: 429 too many certificates already issued
```
→ Vous avez atteint les limites de taux Let's Encrypt.

```
Issuer letsencrypt-prod not found
```
→ Le ClusterIssuer n'existe pas ou est mal nommé.

#### Étape 2 : Vérifier le ClusterIssuer

```bash
kubectl get clusterissuer letsencrypt-prod
```

Doit afficher :
```
NAME                READY   AGE
letsencrypt-prod    True    30d
```

Si `READY: False` ou si le ClusterIssuer n'existe pas, c'est votre problème.

**Solution pour ClusterIssuer manquant** :

```bash
# Créer le ClusterIssuer
kubectl apply -f clusterissuer-production.yaml

# Vérifier
kubectl get clusterissuer
```

#### Étape 3 : Vérifier la CertificateRequest

```bash
kubectl get certificaterequest
```

Vous devriez voir une requête pour votre certificat :

```
NAME                APPROVED   DENIED   READY   AGE
app-tls-xxxxx       True                False   5m
```

Si DENIED est `True`, le certificat a été rejeté. Détails :

```bash
kubectl describe certificaterequest app-tls-xxxxx
```

#### Étape 4 : Vérifier l'Order

```bash
kubectl get order
```

```
NAME                    STATE     AGE
app-tls-xxxxx-yyyyy     pending   5m
```

États possibles :
- **pending** : en cours de validation
- **valid** : validé, certificat émis
- **invalid** : échec de validation
- **processing** : Let's Encrypt traite la demande

Si `invalid`, consultez les détails :

```bash
kubectl describe order app-tls-xxxxx-yyyyy
```

#### Étape 5 : Vérifier les Challenges

```bash
kubectl get challenge
```

```
NAME                              STATE     AGE
app-tls-xxxxx-yyyyy-zzzzz         pending   5m
```

États :
- **pending** : en attente de résolution
- **processing** : en cours de validation
- **valid** : validé avec succès
- **invalid** : échec de validation

Pour les détails d'un challenge :

```bash
kubectl describe challenge app-tls-xxxxx-yyyyy-zzzzz
```

Cherchez le message d'erreur dans les Events.

### Solutions courantes

#### Solution 1 : Attendre

Si tout semble correct et que le statut est `pending`, **attendez 2-3 minutes**. L'émission prend du temps.

#### Solution 2 : Limites de taux Let's Encrypt

Message : `too many certificates already issued`

**Solution** :
- Utilisez l'environnement staging pour tester
- Attendez une semaine avant de réessayer en production
- Vérifiez que vous n'avez pas créé trop de certificats par erreur

```bash
# Passer en staging temporairement
kubectl edit certificate app-tls
# Changez issuerRef.name: letsencrypt-staging
```

#### Solution 3 : Configuration DNS incorrecte

Si le challenge HTTP-01 échoue avec "connection refused" :

```bash
# Vérifier que le domaine pointe vers votre cluster
dig app.exemple.com
nslookup app.exemple.com

# Tester l'accessibilité
curl http://app.exemple.com
```

**Solution** : corrigez votre configuration DNS pour pointer vers l'IP de votre cluster.

#### Solution 4 : Port 80 non accessible

Le challenge HTTP-01 nécessite que le port 80 soit accessible depuis Internet.

**Vérifications** :
- Firewall : le port 80 est-il ouvert ?
- NAT/Port forwarding : configuré correctement ?
- Ingress Controller : fonctionne-t-il ?

```bash
# Tester depuis l'extérieur
curl http://app.exemple.com/.well-known/acme-challenge/test

# Vérifier l'Ingress Controller
kubectl get pods -n ingress
kubectl logs -n ingress <pod-ingress-controller>
```

## Problème 2 : Erreur de certificat dans le navigateur

### Symptômes

Le navigateur affiche :
- "Votre connexion n'est pas privée" (Chrome)
- "Avertissement : risque probable de sécurité" (Firefox)
- "Ce site n'est peut-être pas sécurisé" (Safari)

Code d'erreur : `NET::ERR_CERT_AUTHORITY_INVALID`, `SEC_ERROR_UNKNOWN_ISSUER`, etc.

### Diagnostic

#### Étape 1 : Vérifier quel certificat est servi

```bash
# Avec OpenSSL
echo | openssl s_client -servername app.exemple.com -connect app.exemple.com:443 2>/dev/null | openssl x509 -noout -text

# Voir l'émetteur (Issuer)
echo | openssl s_client -servername app.exemple.com -connect app.exemple.com:443 2>/dev/null | openssl x509 -noout -issuer

# Voir les domaines couverts
echo | openssl s_client -servername app.exemple.com -connect app.exemple.com:443 2>/dev/null | openssl x509 -noout -text | grep -A1 "Subject Alternative Name"
```

#### Étape 2 : Identifier le type d'erreur

**Cas 1 : Certificat auto-signé détecté**

Émetteur contient "Fake" ou "Self-signed" :
```
Issuer: CN = Fake LE Intermediate X1
```

**Cause** : vous utilisez l'environnement **staging** de Let's Encrypt.

**Solution** : passez au ClusterIssuer **production** :

```yaml
issuerRef:
  name: letsencrypt-prod  # au lieu de letsencrypt-staging
```

**Cas 2 : Le domaine ne correspond pas**

Les domaines dans le certificat ne correspondent pas à l'URL demandée.

**Exemple** :
- Certificat pour : `exemple.com`
- URL demandée : `www.exemple.com`

**Solution** : ajoutez tous les domaines nécessaires dans `dnsNames` :

```yaml
dnsNames:
- exemple.com
- www.exemple.com
```

**Cas 3 : Certificat expiré**

```
Not After : Jan 15 10:00:00 2024 GMT
```

Date dans le passé = certificat expiré.

**Solution** : forcer le renouvellement :

```bash
kubectl delete secret app-tls
```

**Cas 4 : Certificat pas encore émis**

L'Ingress utilise un certificat par défaut ou un certificat invalide car le vrai certificat n'est pas encore prêt.

**Solution** : attendez que le certificat soit `READY: True` :

```bash
kubectl get certificate app-tls
watch kubectl get certificate app-tls
```

### Solutions courantes

#### Solution 1 : Utiliser le bon ClusterIssuer

```bash
# Vérifier quel ClusterIssuer est utilisé
kubectl get certificate app-tls -o jsonpath='{.spec.issuerRef.name}'

# Si c'est staging, modifier
kubectl edit certificate app-tls
# Changez : name: letsencrypt-prod
```

#### Solution 2 : Ajouter le domaine manquant

Si vous accédez à `www.exemple.com` mais que le certificat ne couvre que `exemple.com` :

```bash
kubectl edit certificate app-tls
```

Ajoutez le domaine manquant :

```yaml
dnsNames:
- exemple.com
- www.exemple.com  # Ajoutez cette ligne
```

Le certificat sera automatiquement réémis avec les deux domaines.

#### Solution 3 : Vérifier l'Ingress utilise le bon Secret

```bash
kubectl get ingress app-ingress -o yaml | grep secretName
```

Doit correspondre au `secretName` du Certificate :

```yaml
# Dans Certificate
spec:
  secretName: app-tls

# Dans Ingress
spec:
  tls:
  - secretName: app-tls  # Doit correspondre
```

## Problème 3 : Le challenge HTTP-01 échoue

### Symptômes

```bash
kubectl describe challenge <nom>
```

Messages d'erreur :
- `Connection refused`
- `Timeout`
- `404 Not Found`
- `Invalid response`

### Diagnostic

#### Étape 1 : Vérifier que le pod solver existe

Le challenge HTTP-01 crée un pod temporaire :

```bash
kubectl get pods | grep cm-acme-http-solver
```

Vous devriez voir :

```
cm-acme-http-solver-xxxxx   1/1     Running   0          30s
```

Si le pod n'existe pas ou est en erreur :

```bash
kubectl describe pod cm-acme-http-solver-xxxxx
kubectl logs cm-acme-http-solver-xxxxx
```

#### Étape 2 : Vérifier que le Service existe

```bash
kubectl get svc | grep cm-acme-http-solver
```

#### Étape 3 : Vérifier que l'Ingress temporaire est créé

```bash
kubectl get ingress | grep cm-acme-http-solver
```

Cert-Manager crée un Ingress temporaire pour servir le fichier de challenge.

#### Étape 4 : Tester l'accès au challenge

```bash
# Récupérer le token depuis le challenge
TOKEN=$(kubectl get challenge <nom> -o jsonpath='{.spec.token}')

# Tester l'accès
curl -v http://app.exemple.com/.well-known/acme-challenge/$TOKEN
```

Devrait retourner une chaîne de validation (pas 404, pas d'erreur).

### Solutions courantes

#### Solution 1 : Domaine ne pointe pas vers le cluster

```bash
# Vérifier le DNS
dig app.exemple.com

# Doit pointer vers l'IP publique de votre cluster
# Si ce n'est pas le cas, mettez à jour votre DNS
```

#### Solution 2 : Firewall bloque le port 80

Le port 80 DOIT être accessible depuis Internet pour HTTP-01.

**Vérifications** :
- Firewall du serveur : `sudo ufw status` ou `sudo firewall-cmd --list-all`
- Firewall cloud (AWS Security Group, GCP Firewall Rules, etc.)
- Router/Box : port forwarding configuré

**Solution** : ouvrez le port 80 en entrée.

#### Solution 3 : Ingress Controller ne fonctionne pas

```bash
# Vérifier l'Ingress Controller
kubectl get pods -n ingress

# Vérifier les logs
kubectl logs -n ingress deployment/nginx-ingress-controller

# Redémarrer si nécessaire
kubectl rollout restart deployment -n ingress nginx-ingress-controller
```

#### Solution 4 : Classe d'Ingress incorrecte

Vérifiez que la classe d'Ingress correspond :

```yaml
# Dans ClusterIssuer
solvers:
- http01:
    ingress:
      class: public  # Doit correspondre

# Dans votre Ingress principal
spec:
  ingressClassName: public  # Doit correspondre
```

#### Solution 5 : Problème de résolution DNS depuis Let's Encrypt

Parfois, Let's Encrypt ne résout pas correctement votre domaine (propagation DNS en cours).

**Solution** :
- Attendez 10-15 minutes que le DNS se propage
- Vérifiez avec plusieurs serveurs DNS :

```bash
dig @8.8.8.8 app.exemple.com
dig @1.1.1.1 app.exemple.com
```

## Problème 4 : Le challenge DNS-01 échoue

### Symptômes

Message d'erreur dans le challenge :
- `DNS record not found`
- `NXDOMAIN`
- `Unauthorized`
- `Invalid credentials`

### Diagnostic

#### Étape 1 : Vérifier les credentials API

Les credentials DNS sont-ils corrects et valides ?

```bash
# Vérifier que le Secret existe
kubectl get secret cloudflare-api-token -n cert-manager

# Vérifier son contenu (sans afficher la clé)
kubectl describe secret cloudflare-api-token -n cert-manager
```

#### Étape 2 : Tester les credentials manuellement

**Pour Cloudflare** :

```bash
# Extraire le token
TOKEN=$(kubectl get secret cloudflare-api-token -n cert-manager -o jsonpath='{.data.api-token}' | base64 -d)

# Tester l'API
curl -X GET "https://api.cloudflare.com/client/v4/user/tokens/verify" \
     -H "Authorization: Bearer $TOKEN" \
     -H "Content-Type: application/json"
```

Devrait retourner `"status":"active"`.

#### Étape 3 : Vérifier que l'enregistrement DNS est créé

Pendant le challenge, un enregistrement TXT doit être créé :

```bash
# Récupérer le domaine du challenge
DOMAIN=$(kubectl get certificate app-tls -o jsonpath='{.spec.dnsNames[0]}')

# Vérifier l'enregistrement TXT
dig _acme-challenge.$DOMAIN TXT
nslookup -type=TXT _acme-challenge.$DOMAIN
```

Si l'enregistrement n'existe pas, Cert-Manager n'a pas réussi à le créer.

### Solutions courantes

#### Solution 1 : Credentials expirés ou invalides

**Solution** : régénérez les credentials et mettez à jour le Secret :

```bash
# Supprimer l'ancien
kubectl delete secret cloudflare-api-token -n cert-manager

# Créer le nouveau avec le bon token
kubectl create secret generic cloudflare-api-token \
  --from-literal=api-token=NOUVEAU_TOKEN \
  -n cert-manager
```

#### Solution 2 : Permissions insuffisantes

Le token API doit avoir les bonnes permissions.

**Pour Cloudflare**, le token doit avoir :
- Permission : **Zone - DNS - Edit**
- Ressources : **Zone spécifique** (votre domaine)

**Solution** : recréez le token avec les bonnes permissions.

#### Solution 3 : Zone DNS incorrecte

Vérifiez que vous utilisez la bonne zone DNS :

```bash
# Pour Cloudflare, lister vos zones
curl -X GET "https://api.cloudflare.com/client/v4/zones" \
     -H "Authorization: Bearer $TOKEN" \
     -H "Content-Type: application/json"
```

#### Solution 4 : Propagation DNS lente

Le DNS peut prendre du temps à se propager.

**Solution** : augmenter le timeout dans le ClusterIssuer :

```yaml
solvers:
- dns01:
    cloudflare:
      apiTokenSecretRef:
        name: cloudflare-api-token
        key: api-token
    # Augmenter le timeout
    cnameStrategy: Follow
    webhookTimeout: 120s  # 2 minutes au lieu de 60s
```

## Problème 5 : Le renouvellement ne se fait pas

### Symptômes

Le certificat approche de l'expiration mais reste `READY: True` sans renouvellement :

```bash
kubectl get certificate app-tls
```

```
NAME      READY   SECRET     AGE
app-tls   True    app-tls    85d
```

Le certificat a 85 jours, il devrait être renouvelé à J+60 (avec `renewBefore: 720h`).

### Diagnostic

#### Étape 1 : Vérifier la configuration de renouvellement

```bash
kubectl get certificate app-tls -o yaml | grep -A2 renewBefore
```

Devrait afficher :

```yaml
renewBefore: 720h
```

Si absent, les valeurs par défaut sont utilisées (normalement OK).

#### Étape 2 : Vérifier la date de renouvellement prévue

```bash
kubectl describe certificate app-tls | grep "Renewal Time"
```

```
Renewal Time:  2024-03-16T10:00:00Z
```

Si cette date est dépassée et que le certificat n'a pas été renouvelé, il y a un problème.

#### Étape 3 : Vérifier les logs Cert-Manager

```bash
kubectl logs -n cert-manager deployment/cert-manager | grep app-tls | grep -i renew
```

Cherchez des erreurs ou des messages indiquant pourquoi le renouvellement n'a pas lieu.

### Solutions courantes

#### Solution 1 : Forcer le renouvellement

Si le renouvellement ne se déclenche pas automatiquement :

```bash
# Méthode simple : supprimer le Secret
kubectl delete secret app-tls

# Cert-Manager recréera automatiquement le certificat
# Vérifier après 1-2 minutes
kubectl get certificate app-tls
```

#### Solution 2 : Cert-Manager ne fonctionne pas

Vérifiez que Cert-Manager tourne correctement :

```bash
# Vérifier les pods
kubectl get pods -n cert-manager

# Tous doivent être Running
# Si ce n'est pas le cas, redémarrez
kubectl rollout restart deployment -n cert-manager cert-manager
```

#### Solution 3 : Problème de permissions

Cert-Manager doit pouvoir mettre à jour les Secrets :

```bash
# Vérifier les permissions
kubectl auth can-i update secrets --as=system:serviceaccount:cert-manager:cert-manager -n default
```

Doit retourner `yes`.

## Problème 6 : Boucle de redirection infinie

### Symptômes

Le navigateur affiche :
- "ERR_TOO_MANY_REDIRECTS"
- "Trop de redirections"
- La page ne charge jamais

### Diagnostic

#### Tester avec curl

```bash
curl -I http://app.exemple.com
```

Si vous voyez une succession de `301` ou `302`, c'est une boucle.

### Solutions

#### Solution 1 : Redirection HTTPS sans certificat

Vous avez activé `force-ssl-redirect` mais pas configuré de certificat.

**Solution** : vérifiez que l'Ingress a bien une section `tls` :

```yaml
metadata:
  annotations:
    nginx.ingress.kubernetes.io/force-ssl-redirect: "true"
spec:
  tls:  # Cette section DOIT exister
  - hosts:
    - app.exemple.com
    secretName: app-tls
```

#### Solution 2 : Load balancer externe mal configuré

Si vous êtes derrière un load balancer (AWS ELB, GCP Load Balancer), il peut terminer le SSL et communiquer avec Ingress en HTTP, causant une boucle.

**Solution** : configurez Ingress pour accepter le header X-Forwarded-Proto :

```yaml
annotations:
  nginx.ingress.kubernetes.io/configuration-snippet: |
    if ($http_x_forwarded_proto = 'http') {
      return 301 https://$host$request_uri;
    }
```

## Outils de diagnostic essentiels

### 1. Commandes kubectl de base

```bash
# Vue d'ensemble
kubectl get certificate,certificaterequest,order,challenge

# Détails d'un Certificate
kubectl describe certificate <nom>

# État du ClusterIssuer
kubectl get clusterissuer
kubectl describe clusterissuer <nom>

# Logs Cert-Manager
kubectl logs -n cert-manager deployment/cert-manager

# Logs en temps réel
kubectl logs -n cert-manager deployment/cert-manager -f

# Événements récents
kubectl get events --sort-by='.lastTimestamp' | grep -i cert
```

### 2. OpenSSL : analyser les certificats

```bash
# Inspecter un certificat distant
echo | openssl s_client -servername app.exemple.com -connect app.exemple.com:443 2>/dev/null | openssl x509 -noout -text

# Voir seulement l'émetteur
echo | openssl s_client -servername app.exemple.com -connect app.exemple.com:443 2>/dev/null | openssl x509 -noout -issuer

# Voir les dates de validité
echo | openssl s_client -servername app.exemple.com -connect app.exemple.com:443 2>/dev/null | openssl x509 -noout -dates

# Voir les domaines couverts (SAN)
echo | openssl s_client -servername app.exemple.com -connect app.exemple.com:443 2>/dev/null | openssl x509 -noout -text | grep -A1 "Subject Alternative Name"

# Extraire et analyser un certificat depuis un Secret
kubectl get secret app-tls -o jsonpath='{.data.tls\.crt}' | base64 -d | openssl x509 -noout -text
```

### 3. Curl : tester les connexions

```bash
# Tester HTTPS (ignorer erreurs de certificat)
curl -k https://app.exemple.com

# Voir les headers de redirection
curl -I http://app.exemple.com

# Suivre les redirections
curl -L http://app.exemple.com

# Verbose (voir la négociation SSL)
curl -v https://app.exemple.com

# Tester le challenge HTTP-01
curl http://app.exemple.com/.well-known/acme-challenge/test
```

### 4. Dig/NSLookup : vérifier le DNS

```bash
# Résolution DNS basique
dig app.exemple.com
nslookup app.exemple.com

# Vérifier un enregistrement TXT (pour DNS-01)
dig _acme-challenge.app.exemple.com TXT
nslookup -type=TXT _acme-challenge.app.exemple.com

# Utiliser un serveur DNS spécifique
dig @8.8.8.8 app.exemple.com
dig @1.1.1.1 app.exemple.com

# Trace complète de la résolution
dig +trace app.exemple.com
```

### 5. cmctl : outil CLI Cert-Manager

Si vous avez installé `cmctl` :

```bash
# Vérifier l'état d'un certificat
cmctl status certificate app-tls

# Forcer un renouvellement
cmctl renew app-tls

# Inspecter les problèmes
cmctl inspect secret app-tls
```

Installation de cmctl :

```bash
# Linux
OS=$(go env GOOS); ARCH=$(go env GOARCH); curl -sSL -o cmctl.tar.gz https://github.com/cert-manager/cert-manager/releases/download/v1.13.0/cmctl-$OS-$ARCH.tar.gz
tar xzf cmctl.tar.gz
sudo mv cmctl /usr/local/bin

# Ou avec kubectl plugin
kubectl krew install cert-manager
```

## Checklist de dépannage complète

Utilisez cette checklist pour diagnostiquer méthodiquement n'importe quel problème :

### ✅ Niveau 1 : Infrastructure

- [ ] MicroK8s fonctionne : `microk8s status`
- [ ] Cert-Manager est installé : `microk8s status | grep cert-manager`
- [ ] Pods Cert-Manager fonctionnent : `kubectl get pods -n cert-manager`
- [ ] Pas d'erreurs dans les logs : `kubectl logs -n cert-manager deployment/cert-manager | grep -i error`

### ✅ Niveau 2 : Configuration de base

- [ ] ClusterIssuer existe : `kubectl get clusterissuer`
- [ ] ClusterIssuer est READY : `kubectl get clusterissuer <nom>`
- [ ] Certificate est créé : `kubectl get certificate`
- [ ] Configuration Certificate correcte : `kubectl get certificate <nom> -o yaml`

### ✅ Niveau 3 : Réseau et DNS

- [ ] Domaine pointe vers le cluster : `dig <domaine>`
- [ ] Port 80 accessible (pour HTTP-01) : `curl http://<domaine>`
- [ ] Port 443 accessible : `curl -k https://<domaine>`
- [ ] DNS propagé partout : `dig @8.8.8.8 <domaine>` et `dig @1.1.1.1 <domaine>`

### ✅ Niveau 4 : Validation

- [ ] CertificateRequest créée : `kubectl get certificaterequest`
- [ ] Order créé : `kubectl get order`
- [ ] Challenge(s) créé(s) : `kubectl get challenge`
- [ ] Challenge(s) résolu(s) : `kubectl describe challenge <nom>`
- [ ] Pod solver fonctionne (HTTP-01) : `kubectl get pods | grep cm-acme`
- [ ] Enregistrement DNS créé (DNS-01) : `dig _acme-challenge.<domaine> TXT`

### ✅ Niveau 5 : Certificat et Secret

- [ ] Secret créé : `kubectl get secret <secret-name>`
- [ ] Secret contient tls.crt et tls.key : `kubectl describe secret <secret-name>`
- [ ] Certificat valide : `kubectl get secret <secret-name> -o jsonpath='{.data.tls\.crt}' | base64 -d | openssl x509 -noout -text`
- [ ] Domaines corrects dans le certificat : voir "Subject Alternative Name"
- [ ] Certificat pas expiré : voir "Not After"

### ✅ Niveau 6 : Ingress

- [ ] Ingress existe : `kubectl get ingress`
- [ ] Ingress référence le bon Secret : `kubectl get ingress <nom> -o yaml | grep secretName`
- [ ] Ingress Controller fonctionne : `kubectl get pods -n ingress`
- [ ] Pas d'erreurs Ingress Controller : `kubectl logs -n ingress <pod>`

### ✅ Niveau 7 : Test final

- [ ] HTTPS accessible : `curl https://<domaine>`
- [ ] Certificat reconnu par navigateur
- [ ] Pas d'avertissement de sécurité
- [ ] Redirection HTTP → HTTPS fonctionne : `curl -I http://<domaine>`

## Problèmes spécifiques Let's Encrypt

### Rate Limit : 50 certificats par semaine

**Message** : `too many certificates already issued`

**Solution** :
- Utilisez staging pour les tests
- Utilisez des certificats wildcard pour économiser vos quotas
- Attendez une semaine avant de réessayer
- Vérifiez que vous n'avez pas créé de certificats en boucle par erreur

### Failed Validation Limit : 5 échecs par heure

**Message** : `too many failed authorizations recently`

**Solution** :
- Attendez 1 heure avant de réessayer
- Corrigez le problème de validation avant de réessayer
- Utilisez staging pour tester vos correctifs

### Problèmes de connectivité Let's Encrypt

Si Let's Encrypt est temporairement inaccessible :

```bash
# Vérifier la connectivité
curl -I https://acme-v02.api.letsencrypt.org/directory

# Vérifier les status de Let's Encrypt
# https://letsencrypt.status.io/
```

## Guide de résolution rapide

### Problème : "Certificate READY: False après 10 minutes"

```bash
# 1. Voir le problème exact
kubectl describe certificate <nom>

# 2. Vérifier le challenge
kubectl get challenge
kubectl describe challenge <nom-du-challenge>

# 3. Si challenge HTTP-01, tester l'accès
curl http://<domaine>/.well-known/acme-challenge/test

# 4. Vérifier les logs
kubectl logs -n cert-manager deployment/cert-manager | tail -50

# 5. Si bloqué, supprimer et recréer
kubectl delete certificate <nom>
kubectl apply -f <fichier-certificate>.yaml
```

### Problème : "Certificat expiré"

```bash
# 1. Vérifier l'expiration
kubectl describe certificate <nom> | grep "Not After"

# 2. Forcer le renouvellement
kubectl delete secret <secret-name>

# 3. Attendre 2 minutes et vérifier
watch kubectl get certificate <nom>
```

### Problème : "Avertissement dans le navigateur"

```bash
# 1. Vérifier quel certificat est servi
echo | openssl s_client -servername <domaine> -connect <domaine>:443 2>/dev/null | openssl x509 -noout -issuer

# 2. Si "Fake LE", c'est staging
kubectl edit certificate <nom>
# Changez : name: letsencrypt-prod

# 3. Attendre le renouvellement ou forcer
kubectl delete secret <secret-name>
```

## Scripts utiles pour le monitoring

### Script : vérifier tous les certificats

```bash
#!/bin/bash
# check-all-certs.sh

echo "=== Vérification des certificats ==="
echo ""

for ns in $(kubectl get namespaces -o jsonpath='{.items[*].metadata.name}'); do
  certs=$(kubectl get certificate -n $ns -o jsonpath='{.items[*].metadata.name}' 2>/dev/null)

  if [ -n "$certs" ]; then
    echo "Namespace: $ns"
    for cert in $certs; do
      ready=$(kubectl get certificate $cert -n $ns -o jsonpath='{.status.conditions[?(@.type=="Ready")].status}')
      expiration=$(kubectl get certificate $cert -n $ns -o jsonpath='{.status.notAfter}')

      if [ "$ready" = "True" ]; then
        echo "  ✅ $cert - READY"
      else
        echo "  ❌ $cert - NOT READY"
        kubectl describe certificate $cert -n $ns | grep -A5 "Message:"
      fi
    done
    echo ""
  fi
done
```

### Script : tester l'accessibilité d'un domaine

```bash
#!/bin/bash
# test-domain.sh

DOMAIN=$1

if [ -z "$DOMAIN" ]; then
  echo "Usage: $0 <domaine>"
  exit 1
fi

echo "=== Test de $DOMAIN ==="
echo ""

echo "1. Résolution DNS:"
dig +short $DOMAIN
echo ""

echo "2. Test HTTP (port 80):"
curl -s -o /dev/null -w "Status: %{http_code}\n" http://$DOMAIN
echo ""

echo "3. Test HTTPS (port 443):"
curl -s -o /dev/null -w "Status: %{http_code}\n" -k https://$DOMAIN
echo ""

echo "4. Certificat HTTPS:"
echo | openssl s_client -servername $DOMAIN -connect $DOMAIN:443 2>/dev/null | openssl x509 -noout -subject -issuer -dates
echo ""

echo "5. Test redirection HTTP → HTTPS:"
curl -s -o /dev/null -w "Status: %{http_code}, Redirect: %{redirect_url}\n" http://$DOMAIN
```

## Bonnes pratiques pour éviter les problèmes

### 1. Testez toujours avec staging d'abord

```yaml
# Phase 1 : staging
issuerRef:
  name: letsencrypt-staging

# Phase 2 : production (après validation)
issuerRef:
  name: letsencrypt-prod
```

### 2. Utilisez des valeurs de renouvellement sensées

```yaml
duration: 2160h      # 90 jours
renewBefore: 720h    # 30 jours avant
```

### 3. Surveillez proactivement

Mettez en place du monitoring (Prometheus) ou au minimum des scripts de vérification réguliers.

### 4. Documentez vos certificats

```yaml
metadata:
  annotations:
    description: "Certificat production pour l'API principale"
    contact: "devops@exemple.com"
    created-by: "Jean Dupont"
    runbook: "https://wiki.exemple.com/certs"
```

### 5. Sauvegardez vos configurations

```bash
# Sauvegarder tous les Certificates
kubectl get certificate --all-namespaces -o yaml > backup-certificates.yaml

# Sauvegarder les ClusterIssuers
kubectl get clusterissuer -o yaml > backup-clusterissuers.yaml

# Sauvegarder les Secrets (chiffré !)
kubectl get secret -o yaml > backup-secrets-encrypted.yaml
```

### 6. Testez régulièrement le renouvellement

Créez des certificats de test avec durées courtes pour valider que le renouvellement fonctionne :

```yaml
duration: 720h       # 30 jours
renewBefore: 360h    # 15 jours avant
```

Vous verrez un renouvellement en 15 jours au lieu de 60.

### 7. Gardez Cert-Manager à jour

```bash
# Vérifier la version actuelle
kubectl get deployment cert-manager -n cert-manager -o jsonpath='{.spec.template.spec.containers[0].image}'

# Mettre à jour MicroK8s (inclut Cert-Manager)
microk8s stop
snap refresh microk8s
microk8s start
```

## Ressources additionnelles

### Documentation officielle

- Cert-Manager : https://cert-manager.io/docs/
- Let's Encrypt : https://letsencrypt.org/docs/
- NGINX Ingress : https://kubernetes.github.io/ingress-nginx/

### Outils en ligne

- SSL Labs : https://www.ssllabs.com/ssltest/ (tester votre config SSL)
- Let's Encrypt Status : https://letsencrypt.status.io/
- DNS Checker : https://dnschecker.org/ (vérifier la propagation DNS)
- HSTS Preload : https://hstspreload.org/

### Communauté

- Cert-Manager Slack : https://cert-manager.io/docs/contributing/
- Kubernetes Slack : channel #cert-manager
- GitHub Issues : https://github.com/cert-manager/cert-manager/issues

## Concepts clés à retenir

- **Méthodologie systématique** : suivre l'ordre Certificate → Order → Challenge → Secret
- **Logs sont vos amis** : `kubectl describe` et `kubectl logs` révèlent presque tous les problèmes
- **Staging d'abord** : toujours tester avec staging avant production
- **Patience** : l'émission prend 1-3 minutes, ne paniquez pas immédiatement
- **DNS est critique** : la majorité des problèmes viennent de configurations DNS incorrectes
- **Monitoring proactif** : surveillez vos certificats, ne les laissez pas expirer
- **Documentation** : documentez vos certificats et leurs configurations

## Conclusion du chapitre 11

Vous avez maintenant une compréhension complète de la gestion des certificats SSL/TLS avec MicroK8s et Cert-Manager :

1. **Bases** : comprendre les certificats et leur importance
2. **Installation** : Cert-Manager en une commande avec MicroK8s
3. **Configuration** : ClusterIssuers pour Let's Encrypt et autres CAs
4. **Automatisation** : obtention automatique de certificats Let's Encrypt
5. **Développement** : certificats auto-signés pour vos environnements de dev
6. **Wildcards** : un certificat pour tous vos sous-domaines
7. **Redirections** : forcer HTTPS avec HSTS
8. **Renouvellement** : tout est automatique, sans intervention
9. **Dépannage** : résoudre n'importe quel problème méthodiquement

Avec ces connaissances, vous êtes capable de **sécuriser professionnellement** n'importe quelle application Kubernetes avec des certificats SSL/TLS valides et automatiquement renouvelés.

**Rappelez-vous** : 99% du temps, Cert-Manager fait tout le travail pour vous. Mais quand un problème survient, vous avez maintenant tous les outils pour le résoudre rapidement !

---

**Fin du chapitre 11 - Certificats SSL/TLS**

**Prochaine section** : [Chapitre 12 - Monitoring avec Prometheus](../12-monitoring-prometheus/12.1-introduction-prometheus.md)

⏭️ [Monitoring avec Prometheus](/12-monitoring-avec-prometheus/README.md)
