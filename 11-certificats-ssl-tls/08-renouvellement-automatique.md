🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 11.8 Renouvellement automatique

## Introduction : le problème des certificats expirés

Les certificats SSL/TLS ont une **durée de vie limitée**. Cette limitation n'est pas un bug, c'est une fonctionnalité de sécurité !

Imaginez que la clé privée d'un certificat soit compromise (volée, leaked, etc.). Si ce certificat est valide pendant 10 ans, l'attaquant peut se faire passer pour votre site pendant 10 ans. Mais si le certificat n'est valide que 90 jours, la fenêtre d'attaque est considérablement réduite.

**Le problème** : gérer manuellement le renouvellement de dizaines ou centaines de certificats tous les 90 jours est :
- Chronophage
- Sujet aux erreurs humaines
- Source potentielle de pannes (certificat oublié qui expire)

**La solution** : Cert-Manager gère le renouvellement **automatiquement**, sans intervention humaine. C'est l'une de ses fonctionnalités les plus précieuses !

**Analogie** : c'est comme avoir un service qui renouvelle automatiquement votre carte de transport avant qu'elle n'expire, sans que vous ayez à y penser.

## Comprendre le cycle de vie d'un certificat

### Timeline d'un certificat Let's Encrypt

Voici ce qui se passe avec un certificat Let's Encrypt typique :

```
Jour 0 : Émission du certificat
│
│  [Période valide : 90 jours]
│
Jour 60 : Cert-Manager commence à surveiller le renouvellement
│         (renewBefore = 30 jours par défaut)
│
Jour 60-90 : Fenêtre de renouvellement
│            Cert-Manager tentera de renouveler le certificat
│
Jour 90 : Expiration du certificat
│         ❌ Si non renouvelé : site inaccessible !
```

### Mécanisme de renouvellement

Cert-Manager vérifie régulièrement (toutes les heures par défaut) tous les certificats :

1. **Vérification** : est-ce que le certificat approche de sa date d'expiration ?
2. **Calcul** : `date_expiration - renewBefore` < date actuelle ?
3. **Déclenchement** : si oui, démarrer le processus de renouvellement
4. **Renouvellement** : même processus que pour l'émission initiale (challenge, validation, émission)
5. **Mise à jour** : remplacement du certificat dans le Secret
6. **Rechargement** : l'Ingress Controller recharge automatiquement le nouveau certificat

Tout cela se passe **automatiquement**, de manière transparente, **sans interruption de service**.

## Configuration du renouvellement

### Paramètres de base

Les paramètres de renouvellement sont définis dans la ressource Certificate :

```yaml
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: monapp-tls
  namespace: default
spec:
  secretName: monapp-tls-secret
  issuerRef:
    name: letsencrypt-prod
    kind: ClusterIssuer

  # Durée de validité demandée
  duration: 2160h  # 90 jours (90 * 24 heures)

  # Quand commencer le renouvellement
  renewBefore: 720h  # 30 jours avant expiration (30 * 24 heures)

  dnsNames:
  - monapp.com
  - www.monapp.com
```

### duration : durée de validité

```yaml
duration: 2160h  # 90 jours
```

- **Définition** : durée de validité souhaitée pour le certificat
- **Format** : en heures (h) ou minutes (m)
- **Let's Encrypt** : émet toujours des certificats de 90 jours, quelle que soit votre demande
- **Auto-signé** : respecte votre durée demandée (vous pouvez mettre 1 an, 10 ans, etc.)

**Exemples** :
- `2160h` = 90 jours
- `8760h` = 1 an (365 jours)
- `720h` = 30 jours

**Valeur par défaut** : 2160h (90 jours) si non spécifié

### renewBefore : quand renouveler

```yaml
renewBefore: 720h  # 30 jours avant expiration
```

- **Définition** : combien de temps **avant l'expiration** Cert-Manager doit commencer à tenter le renouvellement
- **Format** : en heures (h) ou minutes (m)
- **Calcul** : renouvellement déclenché quand `temps_restant < renewBefore`

**Exemples** :
- `720h` = 30 jours (recommandé pour Let's Encrypt)
- `360h` = 15 jours
- `168h` = 7 jours

**Valeur par défaut** : 720h (30 jours) si non spécifié

### Pourquoi 30 jours avant ?

Pour Let's Encrypt, la recommandation est de renouveler **30 jours avant expiration** (donc à J+60 pour un certificat de 90 jours). Pourquoi ?

1. **Marge de sécurité** : si le premier renouvellement échoue, il reste 30 jours pour résoudre le problème
2. **Multiples tentatives** : Cert-Manager réessaiera plusieurs fois pendant cette période
3. **Résolution de problèmes** : vous avez le temps d'intervenir si nécessaire
4. **Limites de taux** : évite de se retrouver bloqué par les limites Let's Encrypt en dernière minute

**Ne mettez jamais** `renewBefore` trop court (< 7 jours) pour Let's Encrypt !

### Exemple complet avec paramètres explicites

```yaml
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: webapp-tls
  namespace: production
spec:
  secretName: webapp-tls-secret

  # Émet un certificat de 90 jours
  duration: 2160h

  # Commence à renouveler 30 jours avant expiration
  renewBefore: 720h

  issuerRef:
    name: letsencrypt-prod
    kind: ClusterIssuer

  dnsNames:
  - webapp.exemple.com
  - www.webapp.exemple.com
```

## Observer le renouvellement

### Vérifier l'état d'un certificat

Pour voir quand un certificat va expirer et être renouvelé :

```bash
kubectl describe certificate webapp-tls
```

Dans la section `Status`, cherchez :

```yaml
Status:
  Conditions:
    Last Transition Time:  2024-01-15T10:00:00Z
    Message:               Certificate is up to date and has not expired
    Reason:                Ready
    Status:                True
    Type:                  Ready
  Not After:               2024-04-15T10:00:00Z   # Date d'expiration
  Not Before:              2024-01-15T10:00:00Z   # Date de début
  Renewal Time:            2024-03-16T10:00:00Z   # Date de renouvellement prévu
```

Les champs importants :

- **Not After** : date d'expiration du certificat actuel
- **Not Before** : date à partir de laquelle le certificat est valide
- **Renewal Time** : date à laquelle Cert-Manager commencera le renouvellement

Dans cet exemple :
- Certificat valide du 15 janvier au 15 avril (90 jours)
- Renouvellement prévu le 16 mars (30 jours avant expiration)

### Calculer le temps restant

Pour voir combien de jours reste avant expiration :

```bash
# Extraire la date d'expiration
expiration=$(kubectl get certificate webapp-tls -o jsonpath='{.status.notAfter}')

# Calculer les jours restants (Linux/macOS)
echo $(( ($(date -d "$expiration" +%s) - $(date +%s)) / 86400 )) jours restants

# Alternative avec outils communs
kubectl get certificate webapp-tls -o jsonpath='{.status.notAfter}' | xargs -I {} date -d {} +%s | awk -v now=$(date +%s) '{print int(($1-now)/86400)" jours restants"}'
```

### Lister tous les certificats et leurs expirations

```bash
kubectl get certificate --all-namespaces -o custom-columns=\
NAMESPACE:.metadata.namespace,\
NAME:.metadata.name,\
READY:.status.conditions[0].status,\
EXPIRATION:.status.notAfter,\
RENEWAL:.status.renewalTime
```

Résultat :

```
NAMESPACE    NAME           READY   EXPIRATION              RENEWAL
default      webapp-tls     True    2024-04-15T10:00:00Z   2024-03-16T10:00:00Z
production   api-tls        True    2024-05-20T14:30:00Z   2024-04-20T14:30:00Z
staging      test-tls       True    2024-03-10T08:15:00Z   2024-02-09T08:15:00Z
```

## Le processus de renouvellement en détail

Lorsque la date de renouvellement arrive, voici ce qui se passe :

### Étape 1 : Détection

Cert-Manager détecte que le certificat doit être renouvelé :

```bash
kubectl logs -n cert-manager deployment/cert-manager | grep -i renew
```

Vous verrez des logs comme :

```
I1215 10:00:00.123456 certificate_controller.go:123] cert-manager/controller/certificates "msg"="Certificate must be re-issued" "reason"="Renewing"
```

### Étape 2 : Création d'une nouvelle CertificateRequest

Cert-Manager crée automatiquement une nouvelle CertificateRequest :

```bash
kubectl get certificaterequest
```

Vous verrez une nouvelle requête avec un suffixe différent :

```
NAME                  APPROVED   DENIED   READY   ISSUER              AGE
webapp-tls-abcde      True              False   letsencrypt-prod    10s
webapp-tls-xyz12      True              True    letsencrypt-prod    30d  # Ancienne
```

### Étape 3 : Challenge et validation

Le processus de validation recommence (HTTP-01 ou DNS-01), identique à l'émission initiale :

```bash
kubectl get challenge
kubectl get order
```

### Étape 4 : Émission du nouveau certificat

Let's Encrypt émet le nouveau certificat (valide 90 jours supplémentaires).

### Étape 5 : Mise à jour du Secret

Cert-Manager met à jour le Secret avec le nouveau certificat :

```bash
# Observer les changements du Secret
kubectl get secret webapp-tls-secret -o yaml

# Voir l'âge du Secret (sera récent après renouvellement)
kubectl get secret webapp-tls-secret
```

### Étape 6 : Rechargement par l'Ingress Controller

L'Ingress Controller (NGINX, Traefik, etc.) détecte automatiquement le changement du Secret et recharge le nouveau certificat **sans interruption de service**.

Aucun redémarrage de pod n'est nécessaire !

## Renouvellement forcé manuel

Parfois, vous voulez forcer le renouvellement immédiat d'un certificat (tests, résolution de problème, etc.).

### Méthode 1 : Supprimer le Secret

La méthode la plus simple :

```bash
kubectl delete secret webapp-tls-secret
```

Cert-Manager détecte immédiatement que le Secret a disparu et crée automatiquement un nouveau certificat.

**Attention** : pendant quelques secondes, votre site peut être inaccessible en HTTPS.

### Méthode 2 : Modifier l'annotation

Ajoutez une annotation pour forcer le renouvellement :

```bash
kubectl annotate certificate webapp-tls cert-manager.io/issue-temporary-certificate="true" --overwrite
```

Puis supprimez l'annotation :

```bash
kubectl annotate certificate webapp-tls cert-manager.io/issue-temporary-certificate-
```

### Méthode 3 : Supprimer et recréer le Certificate

```bash
# Sauvegarder le manifeste
kubectl get certificate webapp-tls -o yaml > webapp-tls-backup.yaml

# Supprimer
kubectl delete certificate webapp-tls

# Recréer immédiatement
kubectl apply -f webapp-tls-backup.yaml
```

Le nouveau certificat sera émis en 1-2 minutes.

### Méthode 4 : Utiliser cmctl (outil CLI de Cert-Manager)

Si vous avez installé `cmctl` :

```bash
# Forcer le renouvellement
cmctl renew webapp-tls

# Renouveler tous les certificats d'un namespace
cmctl renew --all --namespace production
```

## Échec de renouvellement : que faire ?

Le renouvellement automatique peut échouer pour plusieurs raisons. Heureusement, Cert-Manager réessaie automatiquement.

### Détecter un échec

Vérifiez l'état du certificat :

```bash
kubectl describe certificate webapp-tls
```

Si le renouvellement a échoué, vous verrez :

```yaml
Status:
  Conditions:
    Last Transition Time:  2024-03-16T10:00:00Z
    Message:               Failed to renew certificate: challenge failed
    Reason:                Failed
    Status:                False
    Type:                  Ready
```

### Causes courantes d'échec

#### 1. Problème réseau / DNS

Le domaine ne pointe plus vers votre cluster :

```bash
# Vérifier le DNS
dig webapp.exemple.com
nslookup webapp.exemple.com

# Vérifier l'accessibilité
curl http://webapp.exemple.com/.well-known/acme-challenge/test
```

**Solution** : corrigez votre DNS ou votre configuration réseau.

#### 2. Limites de taux Let's Encrypt

Vous avez épuisé vos quotas :

```bash
kubectl describe certificaterequest <nom>
```

Cherchez : "too many certificates already issued"

**Solution** : attendez une semaine ou utilisez l'environnement staging pour tester.

#### 3. Ingress Controller défaillant

L'Ingress Controller ne répond pas :

```bash
kubectl get pods -n ingress
kubectl logs -n ingress <ingress-controller-pod>
```

**Solution** : redémarrez l'Ingress Controller ou vérifiez sa configuration.

#### 4. Credentials API expirés (pour DNS-01)

Vos credentials DNS ont expiré ou sont invalides :

```bash
kubectl logs -n cert-manager deployment/cert-manager | grep -i dns
```

**Solution** : mettez à jour le Secret avec de nouveaux credentials.

#### 5. Challenge timeout

La validation prend trop de temps :

```bash
kubectl describe challenge <nom>
```

**Solution** : vérifiez la connectivité réseau et les performances DNS.

### Tentatives automatiques de Cert-Manager

Cert-Manager ne se décourage pas facilement ! Il réessaie automatiquement selon cette stratégie :

1. **Premier échec** : réessai après 1 heure
2. **Deuxième échec** : réessai après 2 heures
3. **Troisième échec** : réessai après 4 heures
4. **Échecs suivants** : réessai toutes les 8 heures

Avec `renewBefore: 720h` (30 jours), Cert-Manager a **beaucoup de temps** pour réussir le renouvellement.

### Surveiller les échecs

Consultez les événements Kubernetes :

```bash
# Événements récents du Certificate
kubectl describe certificate webapp-tls | grep -A 10 Events

# Tous les événements du namespace
kubectl get events --sort-by='.lastTimestamp' | grep certificate
```

## Monitoring et alertes

### Métriques Prometheus

Cert-Manager expose plusieurs métriques importantes pour Prometheus :

#### 1. Date d'expiration

```
certmanager_certificate_expiration_timestamp_seconds{name="webapp-tls",namespace="default"}
```

Timestamp Unix de la date d'expiration.

#### 2. Temps jusqu'à expiration (en secondes)

Calculé avec PromQL :

```promql
certmanager_certificate_expiration_timestamp_seconds - time()
```

Résultat en secondes. Pour avoir des jours :

```promql
(certmanager_certificate_expiration_timestamp_seconds - time()) / 86400
```

#### 3. État du certificat

```
certmanager_certificate_ready_status{name="webapp-tls",namespace="default"}
```

- `1` = certificat prêt (READY)
- `0` = certificat non prêt (problème)

#### 4. Date du dernier renouvellement

```
certmanager_certificate_renewal_timestamp_seconds{name="webapp-tls"}
```

### Créer des alertes Prometheus

Exemple d'alertes pour surveiller les certificats :

```yaml
groups:
- name: certificates
  rules:

  # Alerte si certificat expire dans moins de 7 jours
  - alert: CertificateExpiringSoon
    expr: |
      (certmanager_certificate_expiration_timestamp_seconds - time()) / 86400 < 7
    for: 1h
    labels:
      severity: warning
    annotations:
      summary: "Certificat {{ $labels.name }} expire bientôt"
      description: "Le certificat expire dans {{ $value | humanizeDuration }}"

  # Alerte critique si certificat expire dans moins de 2 jours
  - alert: CertificateExpiringSoonCritical
    expr: |
      (certmanager_certificate_expiration_timestamp_seconds - time()) / 86400 < 2
    for: 15m
    labels:
      severity: critical
    annotations:
      summary: "🚨 Certificat {{ $labels.name }} expire très bientôt !"
      description: "Le certificat expire dans {{ $value | humanizeDuration }}"

  # Alerte si certificat n'est pas READY
  - alert: CertificateNotReady
    expr: |
      certmanager_certificate_ready_status == 0
    for: 1h
    labels:
      severity: warning
    annotations:
      summary: "Certificat {{ $labels.name }} n'est pas prêt"
      description: "Vérifiez le statut avec kubectl describe certificate {{ $labels.name }}"

  # Alerte si renouvellement échoue de manière répétée
  - alert: CertificateRenewalFailed
    expr: |
      (certmanager_certificate_expiration_timestamp_seconds - time()) / 86400 < 30
      and certmanager_certificate_ready_status == 0
    for: 24h
    labels:
      severity: critical
    annotations:
      summary: "Échec de renouvellement pour {{ $labels.name }}"
      description: "Le certificat approche de l'expiration mais n'est pas renouvelé"
```

### Dashboard Grafana

Créez un dashboard Grafana pour visualiser vos certificats :

**Panels recommandés** :

1. **Table des certificats avec dates d'expiration**
   ```promql
   certmanager_certificate_expiration_timestamp_seconds
   ```

2. **Jours restants avant expiration**
   ```promql
   (certmanager_certificate_expiration_timestamp_seconds - time()) / 86400
   ```

3. **Certificats non prêts**
   ```promql
   count(certmanager_certificate_ready_status == 0)
   ```

4. **Timeline des renouvellements**
   ```promql
   changes(certmanager_certificate_renewal_timestamp_seconds[30d])
   ```

Il existe des dashboards Grafana pré-configurés pour Cert-Manager dans la communauté.

### Script de monitoring simple

Si vous n'avez pas Prometheus, utilisez un script simple :

```bash
#!/bin/bash
# check-certificates.sh

echo "=== État des certificats ==="
echo ""

for cert in $(kubectl get certificate -A -o jsonpath='{range .items[*]}{.metadata.namespace}{":"}{.metadata.name}{"\n"}{end}'); do
  namespace=$(echo $cert | cut -d: -f1)
  name=$(echo $cert | cut -d: -f2)

  expiration=$(kubectl get certificate $name -n $namespace -o jsonpath='{.status.notAfter}')
  ready=$(kubectl get certificate $name -n $namespace -o jsonpath='{.status.conditions[?(@.type=="Ready")].status}')

  if [ -n "$expiration" ]; then
    days_left=$(( ($(date -d "$expiration" +%s) - $(date +%s)) / 86400 ))

    if [ "$ready" = "True" ]; then
      status="✅"
    else
      status="❌"
    fi

    if [ $days_left -lt 7 ]; then
      echo "$status $namespace/$name - ⚠️  EXPIRE BIENTÔT: $days_left jours"
    else
      echo "$status $namespace/$name - OK: $days_left jours"
    fi
  else
    echo "❓ $namespace/$name - Pas encore émis"
  fi
done

echo ""
echo "=== Fin du rapport ==="
```

Utilisez-le :

```bash
chmod +x check-certificates.sh
./check-certificates.sh
```

Ajoutez-le à un cron pour surveillance régulière :

```bash
# Tous les jours à 9h
0 9 * * * /path/to/check-certificates.sh | mail -s "Rapport certificats" admin@exemple.com
```

## Renouvellement et haute disponibilité

### Cert-Manager en HA

Si vous avez plusieurs réplicas de Cert-Manager (pour la haute disponibilité), une seule instance gère les renouvellements grâce à un **leader election**.

Vérifiez quelle instance est le leader :

```bash
kubectl logs -n cert-manager deployment/cert-manager | grep -i leader
```

Vous verrez des messages comme :

```
I1215 10:00:00 leaderelection.go:243] successfully acquired lease cert-manager/cert-manager-controller
```

### Renouvellement sans interruption

Le renouvellement d'un certificat se fait **sans aucune interruption de service** :

1. Le nouveau certificat est émis
2. Le Secret est mis à jour avec le nouveau certificat ET l'ancienne clé (coexistence temporaire)
3. L'Ingress Controller recharge la configuration
4. Le basculement est instantané

Vos utilisateurs ne verront jamais d'erreur de certificat.

## Gestion des certificats multiples

Si vous avez de nombreux certificats, automatisez leur surveillance.

### Politique de renouvellement uniforme

Utilisez les mêmes paramètres pour tous les certificats :

```yaml
# Définir des valeurs par défaut dans tous vos manifestes
duration: 2160h      # 90 jours
renewBefore: 720h    # 30 jours avant
```

Ou utilisez un template Helm / Kustomize pour standardiser.

### Renouvellement en masse

Si vous devez forcer le renouvellement de tous les certificats (par exemple, après un incident) :

```bash
# Méthode 1 : supprimer tous les Secrets TLS
kubectl get secrets --all-namespaces -o json | \
  jq -r '.items[] | select(.type=="kubernetes.io/tls") | "\(.metadata.namespace) \(.metadata.name)"' | \
  while read ns name; do
    kubectl delete secret $name -n $ns
  done

# Méthode 2 : avec cmctl
cmctl renew --all --all-namespaces
```

**Attention** : cela peut déclencher de nombreuses requêtes vers Let's Encrypt. Faites-le uniquement si nécessaire.

## Bonnes pratiques

### 1. Utilisez les valeurs par défaut (renewBefore = 30 jours)

Ne changez `renewBefore` que si vous avez une raison spécifique :

```yaml
# ✅ Bon - utilise les valeurs par défaut sensées
spec:
  issuerRef:
    name: letsencrypt-prod
    kind: ClusterIssuer
  dnsNames:
  - monapp.com

# ❌ Mauvais - valeur trop courte
spec:
  renewBefore: 168h  # 7 jours - trop risqué !
```

### 2. Surveillez activement vos certificats

Mettez en place du monitoring avec Prometheus ou au minimum un script de vérification.

### 3. Testez le renouvellement avec staging

Avant de déployer en production, testez le renouvellement avec l'environnement staging :

```yaml
issuerRef:
  name: letsencrypt-staging
  kind: ClusterIssuer
duration: 720h    # 30 jours au lieu de 90
renewBefore: 360h # 15 jours avant
```

Avec ces valeurs, vous verrez un renouvellement en 15 jours au lieu de 60.

### 4. Alertez-vous bien avant l'expiration

Configurez des alertes à plusieurs niveaux :
- Warning à 14 jours
- Critical à 7 jours
- Emergency à 2 jours

### 5. Documentez les certificats critiques

Identifiez vos certificats les plus critiques et documentez-les :

```yaml
metadata:
  annotations:
    description: "Certificat production principal - critique"
    contact: "equipe-devops@exemple.com"
    runbook: "https://wiki.exemple.com/cert-renewal"
```

### 6. Sauvegardez vos clés de compte ACME

Les Secrets contenant vos clés de compte ACME sont précieux. Incluez-les dans vos backups :

```bash
kubectl get secret letsencrypt-prod-account-key -n cert-manager -o yaml > backup-acme-key.yaml
```

### 7. Ne forcez pas le renouvellement sans raison

Le renouvellement automatique fonctionne bien. Ne le forcez manuellement qu'en cas de nécessité (compromission, tests, etc.).

### 8. Planifiez les maintenances

Si vous devez intervenir sur votre cluster (maintenance réseau, migration), assurez-vous que ce n'est pas pendant une fenêtre de renouvellement.

## Cas particuliers

### Certificats avec durée personnalisée (auto-signés)

Pour les certificats auto-signés, vous contrôlez la durée :

```yaml
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: dev-cert
spec:
  secretName: dev-tls
  issuerRef:
    name: selfsigned-issuer
    kind: ClusterIssuer
  duration: 8760h       # 1 an
  renewBefore: 720h     # 30 jours avant (même stratégie)
  dnsNames:
  - "*.dev.local"
```

Le renouvellement fonctionne de la même manière.

### Certificats qui ne doivent jamais expirer (déconseillé)

Si vous voulez vraiment un certificat de très longue durée (non recommandé) :

```yaml
duration: 87600h  # 10 ans
renewBefore: 8760h # Renouveler 1 an avant
```

**Attention** : c'est une mauvaise pratique de sécurité. N'utilisez cela que pour des cas très spécifiques (labs isolés, tests, etc.).

## Dépannage du renouvellement

### Le certificat n'est pas renouvelé

```bash
# 1. Vérifier l'état du certificat
kubectl describe certificate webapp-tls

# 2. Vérifier les logs Cert-Manager
kubectl logs -n cert-manager deployment/cert-manager | grep webapp-tls

# 3. Vérifier s'il y a des CertificateRequests en cours
kubectl get certificaterequest

# 4. Vérifier les Challenges/Orders
kubectl get challenge
kubectl get order

# 5. Forcer le renouvellement manuel
kubectl delete secret webapp-tls-secret
```

### Le Secret n'est pas mis à jour

Vérifiez les permissions RBAC de Cert-Manager :

```bash
kubectl auth can-i update secrets --as=system:serviceaccount:cert-manager:cert-manager -n default
```

Devrait retourner `yes`.

### L'Ingress n'utilise pas le nouveau certificat

L'Ingress Controller doit être redémarré :

```bash
# NGINX Ingress Controller
kubectl rollout restart deployment -n ingress nginx-ingress-controller

# Ou juste attendre (rechargement automatique après quelques minutes)
```

## Logs de renouvellement

Pour suivre un renouvellement en temps réel :

```bash
# Surveiller les logs en continu
kubectl logs -n cert-manager deployment/cert-manager -f | grep -i renew

# Surveiller l'état du Certificate
watch kubectl get certificate webapp-tls

# Surveiller les événements
kubectl get events --watch | grep certificate
```

## Concepts clés à retenir

- **Renouvellement automatique** : Cert-Manager gère tout, sans intervention manuelle
- **renewBefore** : délai avant expiration pour déclencher le renouvellement (défaut: 30 jours)
- **duration** : durée de validité demandée (Let's Encrypt: toujours 90 jours)
- **Tentatives multiples** : Cert-Manager réessaie automatiquement en cas d'échec
- **Sans interruption** : le renouvellement se fait sans downtime
- **Monitoring essentiel** : surveillez vos certificats avec Prometheus ou scripts
- **Marge de sécurité** : toujours renouveler bien avant l'expiration

## Prochaines étapes

Dans la prochaine et dernière section de ce chapitre, nous allons explorer le dépannage des problèmes courants liés aux certificats. Vous apprendrez à diagnostiquer et résoudre les problèmes d'émission, de renouvellement, et de configuration.

Le renouvellement automatique est la pierre angulaire d'une gestion moderne des certificats. Configurez-le correctement, surveillez-le activement, et vos certificats ne vous causeront jamais de problèmes !

---


⏭️ [Dépannage des certificats](/11-certificats-ssl-tls/09-depannage-des-certificats.md)
