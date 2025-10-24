🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 11.4 Certificats Let's Encrypt automatiques

## Introduction : la magie de l'automatisation

Maintenant que Cert-Manager est installé et configuré avec vos ClusterIssuers, nous allons voir comment obtenir automatiquement des certificats SSL/TLS pour vos applications. C'est là que la magie opère vraiment !

Imaginez que vous deviez renouveler votre carte d'identité tous les trois mois, mais qu'un service automatique s'en charge pour vous : il détecte quand elle va expirer, soumet automatiquement votre demande de renouvellement, récupère la nouvelle carte et la place dans votre portefeuille. C'est exactement ce que fait Cert-Manager avec vos certificats.

Dans cette section, nous allons découvrir deux approches pour obtenir des certificats :
1. **Approche explicite** : créer une ressource Certificate dédiée
2. **Approche automatique** : utiliser les annotations Ingress pour tout automatiser

## Comprendre le cycle de vie d'un certificat

Avant de plonger dans la pratique, comprenons ce qui se passe en coulisses :

### Le processus complet

1. **Demande** : vous créez une ressource Certificate ou un Ingress avec les bonnes annotations
2. **CertificateRequest** : Cert-Manager crée automatiquement une CertificateRequest
3. **Order** : Cert-Manager envoie une commande (Order) à Let's Encrypt
4. **Challenge** : Let's Encrypt répond avec un défi à relever (HTTP-01 ou DNS-01)
5. **Résolution** : Cert-Manager résout le défi (crée un pod temporaire pour HTTP-01)
6. **Validation** : Let's Encrypt vérifie que le défi est relevé
7. **Émission** : Let's Encrypt émet le certificat
8. **Stockage** : Cert-Manager stocke le certificat dans un Secret Kubernetes
9. **Renouvellement** : 30 jours avant l'expiration, Cert-Manager répète automatiquement le processus

Tout cela se fait **automatiquement**, sans intervention manuelle.

### Les ressources impliquées

Cert-Manager crée et gère plusieurs ressources Kubernetes :

```
Certificate (créée par vous)
    ↓
CertificateRequest (créée automatiquement)
    ↓
Order (créée automatiquement)
    ↓
Challenge (créée automatiquement)
    ↓
Secret (créé automatiquement, contient le certificat final)
```

Vous n'avez à vous préoccuper que de la première ressource (Certificate). Le reste est géré automatiquement.

## Méthode 1 : Ressource Certificate explicite

La première méthode consiste à créer une ressource **Certificate** dédiée. C'est l'approche la plus explicite et la plus flexible.

### Anatomie d'une ressource Certificate

Voici un exemple de ressource Certificate :

```yaml
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: example-com-tls
  namespace: default
spec:
  # Nom du Secret qui contiendra le certificat
  secretName: example-com-tls-secret

  # Durée de validité demandée (Let's Encrypt émet toujours pour 90 jours)
  duration: 2160h # 90 jours

  # Renouvellement 30 jours avant l'expiration
  renewBefore: 720h # 30 jours

  # Référence au ClusterIssuer à utiliser
  issuerRef:
    name: letsencrypt-staging
    kind: ClusterIssuer
    group: cert-manager.io

  # Nom commun du certificat
  commonName: example.com

  # Liste des noms DNS couverts par le certificat
  dnsNames:
  - example.com
  - www.example.com
```

### Comprendre chaque élément

#### metadata

```yaml
metadata:
  name: example-com-tls
  namespace: default
```

- **name** : nom de la ressource Certificate (choisissez un nom descriptif)
- **namespace** : namespace où créer le certificat et le Secret

#### spec.secretName

```yaml
spec:
  secretName: example-com-tls-secret
```

- **secretName** : nom du Secret Kubernetes qui contiendra le certificat
- Ce Secret sera créé automatiquement par Cert-Manager
- Vous référencerez ce nom dans votre Ingress pour utiliser le certificat

#### spec.duration et renewBefore

```yaml
  duration: 2160h # 90 jours
  renewBefore: 720h # 30 jours
```

- **duration** : durée de validité souhaitée (Let's Encrypt émet toujours pour 90 jours quoi qu'il arrive)
- **renewBefore** : quand commencer le renouvellement avant l'expiration
- Avec `renewBefore: 720h`, Cert-Manager renouvellera le certificat 30 jours avant son expiration

**Note** : ces champs sont optionnels. Par défaut, Cert-Manager utilise des valeurs sensées.

#### spec.issuerRef

```yaml
  issuerRef:
    name: letsencrypt-staging
    kind: ClusterIssuer
    group: cert-manager.io
```

- **name** : nom du ClusterIssuer (ou Issuer) à utiliser
- **kind** : `ClusterIssuer` ou `Issuer`
- **group** : toujours `cert-manager.io`

C'est ici que vous spécifiez quel ClusterIssuer (staging ou production) doit émettre ce certificat.

#### spec.commonName

```yaml
  commonName: example.com
```

- **commonName** : nom principal du certificat
- Généralement votre nom de domaine principal
- Optionnel si vous spécifiez dnsNames

#### spec.dnsNames

```yaml
  dnsNames:
  - example.com
  - www.example.com
```

- **dnsNames** : liste de tous les noms de domaine que le certificat doit couvrir
- Vous pouvez inclure plusieurs domaines et sous-domaines
- C'est ce qu'on appelle un certificat **SAN** (Subject Alternative Names)

**Important** : tous ces domaines doivent pointer vers votre cluster pour que la validation HTTP-01 fonctionne.

### Exemple complet : certificat staging

Créons un certificat de test avec l'environnement staging :

```yaml
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: mon-site-tls-staging
  namespace: default
spec:
  secretName: mon-site-tls-staging-secret
  issuerRef:
    name: letsencrypt-staging
    kind: ClusterIssuer
  dnsNames:
  - mon-site.exemple.com
  - www.mon-site.exemple.com
```

Enregistrez ce fichier sous `certificate-staging.yaml` et appliquez-le :

```bash
kubectl apply -f certificate-staging.yaml
```

### Observer le processus d'émission

Dès que vous créez la ressource Certificate, le processus commence. Observons-le en temps réel.

#### 1. Vérifier la ressource Certificate

```bash
kubectl get certificate
```

Vous verrez :

```
NAME                    READY   SECRET                          AGE
mon-site-tls-staging    False   mon-site-tls-staging-secret     5s
```

Au début, `READY` est `False` car le certificat n'est pas encore émis.

#### 2. Obtenir des détails

```bash
kubectl describe certificate mon-site-tls-staging
```

Dans la sortie, cherchez la section `Status` et `Events` :

```
Status:
  Conditions:
    Last Transition Time:  2024-01-15T10:00:00Z
    Message:               Issuing certificate as Secret does not exist
    Reason:                DoesNotExist
    Status:                False
    Type:                  Ready
Events:
  Type    Reason     Age   From          Message
  ----    ------     ----  ----          -------
  Normal  Issuing    10s   cert-manager  Issuing certificate as Secret does not exist
  Normal  Generated  10s   cert-manager  Stored new private key in temporary Secret resource
  Normal  Requested  10s   cert-manager  Created new CertificateRequest resource
```

Ces événements vous montrent la progression.

#### 3. Vérifier la CertificateRequest

```bash
kubectl get certificaterequest
```

Résultat :

```
NAME                            APPROVED   DENIED   READY   ISSUER                 AGE
mon-site-tls-staging-xxxxx      True                False   letsencrypt-staging    20s
```

#### 4. Vérifier l'Order

```bash
kubectl get order
```

Résultat :

```
NAME                                   STATE     AGE
mon-site-tls-staging-xxxxx-yyyyy       pending   30s
```

L'état passera de `pending` à `valid` une fois le certificat émis.

#### 5. Vérifier les Challenges

```bash
kubectl get challenge
```

Résultat :

```
NAME                                              STATE     AGE
mon-site-tls-staging-xxxxx-yyyyy-zzzzz            pending   40s
```

Le Challenge crée un pod temporaire pour répondre à la validation HTTP-01 :

```bash
kubectl get pods | grep cm-acme-http-solver
```

Vous verrez un pod temporaire :

```
cm-acme-http-solver-xxxxx   1/1     Running   0          45s
```

Ce pod sert le fichier de challenge à Let's Encrypt.

#### 6. Attendre l'émission (1-2 minutes)

Après environ 1-2 minutes, vérifiez à nouveau :

```bash
kubectl get certificate
```

Si tout s'est bien passé :

```
NAME                    READY   SECRET                          AGE
mon-site-tls-staging    True    mon-site-tls-staging-secret     2m
```

`READY: True` signifie que le certificat est émis et prêt à être utilisé !

#### 7. Vérifier le Secret créé

```bash
kubectl get secret mon-site-tls-staging-secret
```

Résultat :

```
NAME                          TYPE                DATA   AGE
mon-site-tls-staging-secret   kubernetes.io/tls   2      2m
```

Le Secret contient deux données :
- `tls.crt` : le certificat
- `tls.key` : la clé privée

Pour voir le contenu (encodé en base64) :

```bash
kubectl describe secret mon-site-tls-staging-secret
```

### Inspecter le certificat

Vous pouvez extraire et inspecter le certificat :

```bash
# Extraire le certificat
kubectl get secret mon-site-tls-staging-secret -o jsonpath='{.data.tls\.crt}' | base64 -d > cert.crt

# Afficher les informations du certificat
openssl x509 -in cert.crt -text -noout
```

Vous verrez les détails du certificat, notamment :
- L'émetteur (Fake LE Intermediate X1 pour staging)
- Les domaines couverts
- La date d'expiration (90 jours)

### Passer en production

Une fois que vous avez validé que tout fonctionne avec staging, créez le certificat production.

Créez `certificate-production.yaml` :

```yaml
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: mon-site-tls-prod
  namespace: default
spec:
  secretName: mon-site-tls-prod-secret
  issuerRef:
    name: letsencrypt-prod  # Utilisez le ClusterIssuer production
    kind: ClusterIssuer
  dnsNames:
  - mon-site.exemple.com
  - www.mon-site.exemple.com
```

Appliquez :

```bash
kubectl apply -f certificate-production.yaml
```

Le processus est identique, mais cette fois le certificat sera valide et reconnu par tous les navigateurs !

## Méthode 2 : Annotations Ingress (recommandé)

La deuxième méthode est encore plus simple : laisser Cert-Manager créer automatiquement les certificats à partir de vos ressources Ingress.

### Principe

Au lieu de créer explicitement une ressource Certificate, vous ajoutez simplement des **annotations** à votre Ingress. Cert-Manager détecte ces annotations et crée automatiquement les ressources Certificate nécessaires.

### Avantages

- ✅ Plus simple : tout est dans une seule ressource (Ingress)
- ✅ Moins de fichiers à gérer
- ✅ Configuration centralisée
- ✅ Mise à jour automatique si l'Ingress change

### Exemple d'Ingress avec certificat automatique

Voici un Ingress complet qui déclenche l'émission automatique d'un certificat :

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: mon-application
  namespace: default
  annotations:
    # Indique à Cert-Manager de créer un certificat
    cert-manager.io/cluster-issuer: "letsencrypt-staging"
spec:
  ingressClassName: public
  tls:
  - hosts:
    - mon-app.exemple.com
    secretName: mon-app-tls-secret  # Secret qui sera créé automatiquement
  rules:
  - host: mon-app.exemple.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: mon-application
            port:
              number: 80
```

### Comprendre l'Ingress

#### Les annotations Cert-Manager

```yaml
annotations:
  cert-manager.io/cluster-issuer: "letsencrypt-staging"
```

- **cert-manager.io/cluster-issuer** : indique quel ClusterIssuer utiliser
- Cert-Manager détecte cette annotation et crée automatiquement une ressource Certificate

Alternatives :
- `cert-manager.io/issuer: "nom-issuer"` : si vous utilisez un Issuer au lieu d'un ClusterIssuer

#### La section tls

```yaml
tls:
- hosts:
  - mon-app.exemple.com
  secretName: mon-app-tls-secret
```

- **hosts** : liste des domaines à inclure dans le certificat
- **secretName** : nom du Secret où stocker le certificat (Cert-Manager le créera)

#### Les rules

```yaml
rules:
- host: mon-app.exemple.com
  http:
    paths:
    - path: /
      pathType: Prefix
      backend:
        service:
          name: mon-application
          port:
            number: 80
```

Configuration standard d'Ingress qui route le trafic vers votre service.

**Important** : le domaine dans `tls.hosts` et dans `rules.host` doit correspondre !

### Scénario complet : déployer une application avec HTTPS

Mettons tout ensemble pour déployer une application web complète avec HTTPS automatique.

#### 1. Déploiement de l'application

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: hello-world
  namespace: default
spec:
  replicas: 2
  selector:
    matchLabels:
      app: hello-world
  template:
    metadata:
      labels:
        app: hello-world
    spec:
      containers:
      - name: hello-world
        image: nginxdemos/hello
        ports:
        - containerPort: 80
```

#### 2. Service

```yaml
apiVersion: v1
kind: Service
metadata:
  name: hello-world
  namespace: default
spec:
  selector:
    app: hello-world
  ports:
  - protocol: TCP
    port: 80
    targetPort: 80
```

#### 3. Ingress avec certificat automatique

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: hello-world
  namespace: default
  annotations:
    cert-manager.io/cluster-issuer: "letsencrypt-staging"
spec:
  ingressClassName: public
  tls:
  - hosts:
    - hello.exemple.com
    secretName: hello-world-tls
  rules:
  - host: hello.exemple.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: hello-world
            port:
              number: 80
```

#### 4. Appliquer tout

Enregistrez les trois manifestes dans un fichier `hello-world-complete.yaml` et appliquez :

```bash
kubectl apply -f hello-world-complete.yaml
```

#### 5. Observer la création automatique

Cert-Manager détecte le nouvel Ingress et crée automatiquement une ressource Certificate :

```bash
kubectl get certificate
```

Vous verrez :

```
NAME                READY   SECRET              AGE
hello-world-tls     False   hello-world-tls     10s
```

Après 1-2 minutes :

```
NAME                READY   SECRET              AGE
hello-world-tls     True    hello-world-tls     2m
```

Le certificat est émis, et votre application est accessible en HTTPS !

### Vérifier que tout fonctionne

#### Tester avec curl

```bash
curl -k https://hello.exemple.com
```

L'option `-k` ignore la validation du certificat (nécessaire pour staging).

#### Tester avec un navigateur

Accédez à `https://hello.exemple.com` dans votre navigateur.

**Avec staging** : vous verrez un avertissement de sécurité (c'est normal, le certificat n'est pas de confiance).

**Avec production** : pas d'avertissement, le cadenas vert apparaît !

## Certificat pour plusieurs domaines (SAN)

Vous pouvez obtenir un seul certificat couvrant plusieurs domaines :

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: multi-domain
  annotations:
    cert-manager.io/cluster-issuer: "letsencrypt-staging"
spec:
  ingressClassName: public
  tls:
  - hosts:
    - app1.exemple.com
    - app2.exemple.com
    - www.exemple.com
    secretName: multi-domain-tls
  rules:
  - host: app1.exemple.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: app1
            port:
              number: 80
  - host: app2.exemple.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: app2
            port:
              number: 80
```

Un seul certificat couvrira les trois domaines.

## Certificats séparés pour différents domaines

Vous pouvez aussi avoir plusieurs sections `tls` dans un Ingress :

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: multi-cert
  annotations:
    cert-manager.io/cluster-issuer: "letsencrypt-staging"
spec:
  ingressClassName: public
  tls:
  - hosts:
    - app1.exemple.com
    secretName: app1-tls
  - hosts:
    - app2.exemple.com
    secretName: app2-tls
  rules:
  - host: app1.exemple.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: app1
            port:
              number: 80
  - host: app2.exemple.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: app2
            port:
              number: 80
```

Cela créera deux certificats séparés, un pour chaque domaine.

## Renouvellement automatique

Le grand avantage de Cert-Manager : le renouvellement automatique !

### Comment ça fonctionne

- Cert-Manager surveille tous les certificats
- 30 jours avant l'expiration (par défaut), il lance automatiquement le processus de renouvellement
- Le nouveau certificat remplace l'ancien dans le Secret
- Aucune interruption de service

### Surveiller les renouvellements

Pour voir quand un certificat sera renouvelé :

```bash
kubectl describe certificate mon-site-tls-prod
```

Cherchez :

```
Status:
  Not After:  2024-04-15T10:00:00Z
  Not Before: 2024-01-15T10:00:00Z
  Renewal Time: 2024-03-16T10:00:00Z  # Date de renouvellement prévue
```

### Forcer un renouvellement manuel

Si vous voulez forcer le renouvellement immédiatement :

```bash
# Supprimer le Secret (Cert-Manager le recréera)
kubectl delete secret mon-site-tls-prod-secret

# Ou supprimer et recréer le Certificate
kubectl delete certificate mon-site-tls-prod
kubectl apply -f certificate-production.yaml
```

**Attention** : ne faites cela que si nécessaire (tests, problèmes). Le renouvellement automatique fonctionne très bien.

## Annotations avancées

Cert-Manager supporte plusieurs annotations utiles :

### Spécifier la durée du certificat

```yaml
annotations:
  cert-manager.io/cluster-issuer: "letsencrypt-prod"
  cert-manager.io/duration: "2160h"  # 90 jours
  cert-manager.io/renew-before: "360h"  # 15 jours avant expiration
```

### Utiliser plusieurs ClusterIssuers

Vous pouvez avoir plusieurs Ingress avec des ClusterIssuers différents :

```yaml
# Ingress 1 - staging
annotations:
  cert-manager.io/cluster-issuer: "letsencrypt-staging"

# Ingress 2 - production
annotations:
  cert-manager.io/cluster-issuer: "letsencrypt-prod"
```

### Spécifier le type de challenge

Par défaut, Cert-Manager utilise le solver défini dans le ClusterIssuer, mais vous pouvez le surcharger :

```yaml
annotations:
  cert-manager.io/cluster-issuer: "letsencrypt-prod"
  acme.cert-manager.io/http01-edit-in-place: "true"
```

## Dépannage des certificats

### Le certificat n'est jamais READY

Causes possibles et solutions :

#### 1. Problème DNS

**Symptôme** : le challenge échoue avec "connection refused" ou "timeout"

**Solution** : vérifiez que votre domaine pointe bien vers votre cluster :

```bash
dig mon-app.exemple.com
nslookup mon-app.exemple.com
```

#### 2. Port 80 non accessible

**Symptôme** : erreur de validation HTTP-01

**Solution** : vérifiez que le port 80 est ouvert et accessible depuis Internet :

```bash
curl http://mon-app.exemple.com/.well-known/acme-challenge/test
```

#### 3. Ingress Controller non configuré

**Symptôme** : le pod cm-acme-http-solver ne reçoit pas de trafic

**Solution** : vérifiez que votre Ingress Controller fonctionne :

```bash
kubectl get pods -n ingress
kubectl logs -n ingress <ingress-controller-pod>
```

#### 4. Quota Let's Encrypt dépassé

**Symptôme** : erreur "too many certificates already issued"

**Solution** : vous avez atteint les limites de taux. Attendez une semaine ou utilisez staging.

#### 5. Configuration du solver incorrecte

**Symptôme** : le challenge n'est jamais créé

**Solution** : vérifiez votre ClusterIssuer :

```bash
kubectl describe clusterissuer letsencrypt-staging
```

### Consulter les logs Cert-Manager

En cas de problème, les logs de Cert-Manager sont précieux :

```bash
kubectl logs -n cert-manager deployment/cert-manager
```

Cherchez des messages d'erreur ou des avertissements.

### Événements Kubernetes

Les événements donnent beaucoup d'informations :

```bash
# Événements du Certificate
kubectl describe certificate mon-site-tls

# Événements du CertificateRequest
kubectl describe certificaterequest <nom>

# Événements du Challenge
kubectl describe challenge <nom>
```

### Commandes de diagnostic

Voici un ensemble de commandes utiles pour diagnostiquer :

```bash
# Vue d'ensemble
kubectl get certificate,certificaterequest,order,challenge

# Détails d'un certificat
kubectl describe certificate <nom>

# Logs du solver HTTP-01
kubectl logs <cm-acme-http-solver-pod>

# Vérifier le Secret
kubectl get secret <secret-name> -o yaml

# Logs Cert-Manager
kubectl logs -n cert-manager deployment/cert-manager | grep -i error
```

## Bonnes pratiques

### 1. Commencez toujours avec staging

Testez d'abord avec `letsencrypt-staging`, validez que tout fonctionne, puis passez à `letsencrypt-prod`.

### 2. Utilisez des annotations Ingress

Pour la plupart des cas, les annotations Ingress sont plus simples que les ressources Certificate explicites.

### 3. Nommage cohérent des Secrets

Utilisez un schéma de nommage clair :
- `<nom-app>-tls` pour les certificats production
- `<nom-app>-tls-staging` pour les certificats staging

### 4. Un certificat SAN pour plusieurs sous-domaines

Si vous avez plusieurs sous-domaines d'une même application, utilisez un seul certificat SAN plutôt que plusieurs certificats séparés.

### 5. Surveillez les expirations

Bien que le renouvellement soit automatique, mettez en place une surveillance (avec Prometheus/Grafana) pour être alerté en cas de problème.

### 6. Gardez vos Secrets précieusement

Les Secrets contenant les certificats doivent être sauvegardés et protégés par RBAC.

### 7. Documentation

Documentez quel ClusterIssuer est utilisé pour quelle application, surtout si vous utilisez plusieurs environnements.

## Métriques et monitoring

Cert-Manager expose des métriques Prometheus pour surveiller vos certificats :

```
certmanager_certificate_expiration_timestamp_seconds : date d'expiration
certmanager_certificate_ready_status : état READY du certificat
certmanager_certificate_renewal_timestamp_seconds : date de dernier renouvellement
```

Nous verrons comment les utiliser avec Prometheus et Grafana dans les chapitres suivants.

## Résumé des deux approches

| Aspect | Ressource Certificate | Annotations Ingress |
|--------|----------------------|---------------------|
| Complexité | Moyenne | Simple |
| Fichiers | 2 (Certificate + Ingress) | 1 (Ingress seul) |
| Flexibilité | Élevée | Moyenne |
| Cas d'usage | Certificats avancés, non liés à Ingress | Applications web standard |
| Recommandation | Cas spécifiques | Usage général |

## Concepts clés à retenir

- **Certificate** : ressource Kubernetes représentant un certificat SSL/TLS
- **Émission automatique** : Cert-Manager gère tout le cycle de vie
- **Annotations Ingress** : méthode simple pour automatiser les certificats
- **Secret TLS** : stockage sécurisé du certificat et de la clé privée
- **Challenge** : mécanisme de validation du contrôle du domaine
- **Renouvellement automatique** : 30 jours avant expiration par défaut
- **Staging d'abord** : toujours tester avec staging avant production

## Prochaines étapes

Maintenant que vous savez obtenir des certificats Let's Encrypt automatiquement, nous allons explorer dans les prochaines sections :

- Création de certificats auto-signés pour le développement local
- Configuration de certificats wildcard pour couvrir tous vos sous-domaines
- Mise en place de redirections HTTPS automatiques
- Gestion avancée du renouvellement
- Résolution de problèmes courants

Vous avez maintenant les bases pour sécuriser toutes vos applications avec HTTPS gratuitement et automatiquement !

---


⏭️ [Certificats auto-signés pour le développement](/11-certificats-ssl-tls/05-certificats-auto-signes-pour-le-developpement.md)
