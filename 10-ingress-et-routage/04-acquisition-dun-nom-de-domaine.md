🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 10.4 Acquisition d'un nom de domaine

## Introduction

Pour que vos applications Kubernetes soient accessibles depuis Internet avec des URLs professionnelles comme `mon-site.com` ou `api.mon-entreprise.fr`, vous avez besoin d'un **nom de domaine**. C'est une étape essentielle pour passer d'un environnement de test local à une exposition publique.

Dans ce chapitre, nous allons voir ce qu'est un nom de domaine, pourquoi vous en avez besoin pour utiliser pleinement Ingress, comment en acquérir un, et quelles alternatives existent pour les environnements de développement.

## Pourquoi un Nom de Domaine est Nécessaire

### Le Problème des Adresses IP

Sans nom de domaine, vos applications ne sont accessibles que via des adresses IP :
- `http://192.168.1.100` - Difficile à mémoriser
- `http://203.0.113.45` - Pas professionnel
- `http://45.67.89.123:30080` - Encore pire avec un port

**Problèmes** :
- 🚫 Impossible à mémoriser
- 🚫 Pas professionnel pour un service public
- 🚫 L'IP peut changer (reconnexion, changement de FAI)
- 🚫 Pas de certificat SSL/TLS possible (HTTPS)
- 🚫 Pas de routage Ingress par nom d'hôte

### Avec un Nom de Domaine

Avec un nom de domaine, vous obtenez :
- ✅ `https://mon-site.com` - Simple et mémorisable
- ✅ `https://api.mon-site.com` - Sous-domaines pour différents services
- ✅ `https://blog.mon-site.com` - Organisation claire
- ✅ Certificats SSL/TLS automatiques (Let's Encrypt)
- ✅ Routage Ingress intelligent par nom d'hôte
- ✅ Crédibilité et professionnalisme

### Lien avec Ingress

Rappelez-vous : **Ingress route le trafic en fonction du nom de domaine** demandé dans l'URL. Sans nom de domaine, pas de routage intelligent !

Exemple de règle Ingress :
```yaml
spec:
  rules:
  - host: blog.mon-site.com      # Nécessite un nom de domaine !
    http:
      paths:
      - path: /
        backend:
          service:
            name: blog-service
```

## Qu'est-ce qu'un Nom de Domaine ?

### Définition Simple

Un **nom de domaine** est l'adresse lisible par les humains qui permet d'accéder à un site web. C'est comme une **étiquette facile à retenir** collée sur une adresse IP complexe.

**Analogie** :
- L'adresse IP = le numéro GPS exact d'une maison (latitude/longitude)
- Le nom de domaine = l'adresse postale lisible (123 rue de la Paix, Paris)

### Anatomie d'un Nom de Domaine

Prenons l'exemple : `blog.mon-site.com`

```
blog.mon-site.com
 │    │      │
 │    │      └─ TLD (Top-Level Domain) - Extension
 │    └──────── Domaine principal
 └───────────── Sous-domaine
```

**Composants** :
1. **TLD** (`.com`, `.fr`, `.org`, `.net`) : L'extension du domaine
2. **Domaine principal** (`mon-site`) : Le nom que vous choisissez et achetez
3. **Sous-domaine** (`blog`, `api`, `www`) : Préfixes que vous créez gratuitement après l'achat

### Les Niveaux de Domaine

- **Domaine de premier niveau (TLD)** : `.com`, `.fr`, `.org`, `.io`, etc.
- **Domaine de second niveau** : `mon-site.com` (celui que vous achetez)
- **Domaine de troisième niveau** : `blog.mon-site.com` (sous-domaines illimités)

Quand vous achetez `mon-site.com`, vous pouvez créer autant de sous-domaines que vous voulez **gratuitement** :
- `www.mon-site.com`
- `api.mon-site.com`
- `admin.mon-site.com`
- `test.mon-site.com`
- etc.

## Les Différentes Options

### 1. Domaine Payant (Recommandé pour la Production)

**Avantages** :
- ✅ Nom personnalisé et professionnel
- ✅ Contrôle total
- ✅ Certificats SSL reconnus
- ✅ Peut être revendu
- ✅ Sous-domaines illimités
- ✅ Support technique

**Inconvénients** :
- 💰 Coût annuel (10-20€/an en général)
- ⏱️ Processus d'achat et de configuration

**Quand l'utiliser** :
- Site web professionnel
- Application en production
- Service public
- Entreprise ou projet sérieux

### 2. Domaine Gratuit (Pour Tester)

**Services de domaines gratuits** :
- Freenom : `.tk`, `.ml`, `.ga`, `.cf`, `.gq`
- Autres services : `.pp.ua`, `.us.to`

**Avantages** :
- ✅ Gratuit
- ✅ Bon pour apprendre et tester
- ✅ Fonctionne comme un vrai domaine

**Inconvénients** :
- ❌ Extensions peu professionnelles
- ❌ Peuvent être révoqués sans préavis
- ❌ Limitations de durée (souvent 1 an max)
- ❌ Mauvaise réputation SEO
- ❌ Souvent bloqués par certains systèmes

**Quand l'utiliser** :
- Apprentissage et tests
- Labs personnels
- Démonstrations temporaires
- **Jamais pour de la production sérieuse**

### 3. Services DNS Dynamiques (Pour Labs Locaux)

Pour les environnements de développement **non accessibles publiquement**, vous pouvez utiliser des services qui mappent des IPs vers des noms de domaine automatiquement.

#### nip.io (Recommandé pour les Labs)

**Principe** : Le nom de domaine contient l'IP, et nip.io résout automatiquement vers cette IP.

**Exemples** :
- `192.168.1.100.nip.io` → résout vers `192.168.1.100`
- `app.192.168.1.100.nip.io` → résout vers `192.168.1.100`
- `10-0-0-1.nip.io` → résout vers `10.0.0.1` (tirets au lieu de points)

**Avantages** :
- ✅ Gratuit et sans inscription
- ✅ Fonctionne immédiatement
- ✅ Parfait pour les tests locaux
- ✅ Supporte les sous-domaines
- ✅ Compatible avec Let's Encrypt

**Inconvénients** :
- ❌ Pas professionnel (contient l'IP dans le nom)
- ❌ Dépend d'un service externe
- ❌ Seulement pour développement/test

**Cas d'usage** :
```yaml
spec:
  rules:
  - host: mon-app.192.168.1.100.nip.io
    http:
      paths:
      - path: /
        backend:
          service:
            name: mon-service
```

#### xip.io (Alternative à nip.io)

Fonctionne de manière similaire à nip.io :
- `192.168.1.100.xip.io`
- `app.192.168.1.100.xip.io`

#### sslip.io (Autre Alternative)

Même principe :
- `192-168-1-100.sslip.io`
- `app-192-168-1-100.sslip.io`

### 4. Résolution Locale (Fichier hosts)

Pour des tests purement locaux sans connexion Internet, vous pouvez modifier le fichier `hosts` de votre machine.

**Sur Linux/macOS** : `/etc/hosts`
**Sur Windows** : `C:\Windows\System32\drivers\etc\hosts`

Ajoutez des lignes comme :
```
192.168.1.100   mon-site.local
192.168.1.100   api.mon-site.local
192.168.1.100   blog.mon-site.local
```

**Avantages** :
- ✅ Gratuit et totalement local
- ✅ Aucune dépendance externe
- ✅ Parfait pour développement isolé

**Inconvénients** :
- ❌ Fonctionne seulement sur votre machine
- ❌ À configurer sur chaque poste de travail
- ❌ Pas de certificats Let's Encrypt possibles

## Comment Choisir un Nom de Domaine

### Critères de Sélection

1. **Court et Mémorisable**
   - ✅ `monsite.com`
   - ❌ `mon-super-site-web-de-demonstration.com`

2. **Facile à Épeler**
   - ✅ `blog-tech.com`
   - ❌ `bl0g-t3kn0l0gy.com`

3. **Éviter les Caractères Spéciaux**
   - ✅ `mon-site.com` (tirets OK)
   - ❌ `mon_site.com` (underscores problématiques)

4. **Extension Appropriée**
   - `.com` : usage général, international
   - `.fr` : France, confiance locale
   - `.org` : organisations, associations
   - `.io` : technologie, startups
   - `.dev` : développement, tech
   - `.app` : applications

5. **Disponibilité**
   - Vérifiez que le nom n'est pas déjà pris
   - Vérifiez qu'il n'y a pas de marque déposée

6. **Budget**
   - `.com` : 10-15€/an
   - `.fr` : 8-12€/an
   - `.io` : 30-40€/an
   - `.dev` : 12-20€/an

### Outils de Vérification

Avant d'acheter, vérifiez la disponibilité :
- **Namecheap Domain Search** : https://www.namecheap.com
- **GoDaddy Domain Search** : https://www.godaddy.com
- **OVH Domaines** : https://www.ovhcloud.com
- **Google Domains** : https://domains.google (si disponible dans votre région)

## Où Acheter un Nom de Domaine

### Registrars Populaires et Fiables

#### 1. Namecheap (Recommandé)
- **Site** : https://www.namecheap.com
- **Prix** : 10-15€/an (.com)
- **Avantages** :
  - Interface simple
  - Bon support client
  - WHOIS privacy gratuit (protection des données)
  - Pas de frais cachés
  - DNS gratuit inclus

#### 2. OVH (Français)
- **Site** : https://www.ovhcloud.com
- **Prix** : 8-12€/an (.com, .fr)
- **Avantages** :
  - Entreprise française
  - Support en français
  - Bon rapport qualité/prix
  - Nombreuses extensions

#### 3. Gandi (Français)
- **Site** : https://www.gandi.net
- **Prix** : 12-18€/an
- **Avantages** :
  - Éthique et transparent
  - Interface claire
  - Support excellent
  - "No bullshit" policy

#### 4. Google Domains
- **Site** : https://domains.google
- **Prix** : 12€/an environ
- **Avantages** :
  - Simple et intégré Google
  - WHOIS privacy inclus
  - DNS rapide
- **Note** : Service en cours de transfert vers Squarespace

#### 5. Cloudflare Registrar
- **Site** : https://www.cloudflare.com/products/registrar/
- **Prix** : Prix coûtant (10-12€/an)
- **Avantages** :
  - Pas de marge ajoutée
  - DNS ultra-rapide Cloudflare
  - Sécurité incluse
- **Inconvénient** : Nécessite déjà d'utiliser Cloudflare

### Registrars à Éviter

❌ **GoDaddy** : Pratiques commerciales agressives, upsells constants, interface confuse
❌ **Registrars inconnus** : Risque de perte de domaine, support inexistant
❌ **Offres "gratuites" suspectes** : Souvent des arnaques ou avec conditions cachées

## Processus d'Achat d'un Domaine

### Étape 1 : Rechercher la Disponibilité

1. Allez sur le site du registrar (ex: Namecheap)
2. Tapez le nom souhaité dans la barre de recherche : `mon-site.com`
3. Consultez les résultats :
   - ✅ Disponible → Vous pouvez l'acheter
   - ❌ Pris → Essayez une autre extension ou un autre nom

### Étape 2 : Sélectionner le Domaine

1. Ajoutez le domaine au panier
2. Choisissez la durée (1, 2, 5 ans)
   - **Conseil** : 1 an pour commencer, sauf si vous êtes sûr du projet

### Étape 3 : Refuser les Options Inutiles

Les registrars proposent souvent des options supplémentaires :
- **WHOIS Privacy** : ✅ Prenez-le (souvent gratuit)
- **Email hosting** : ❌ Pas nécessaire pour Kubernetes
- **Website builder** : ❌ Vous avez Kubernetes !
- **SSL certificate** : ❌ Let's Encrypt est gratuit (chapitre 11)
- **Domain protection** : ❌ Rarement utile

### Étape 4 : Créer un Compte et Payer

1. Créez un compte sur le registrar
2. Entrez vos informations de paiement
3. Procédez au paiement (carte bancaire, PayPal)

**Prix attendu** : 10-20€/an selon l'extension.

### Étape 5 : Vérification Email

Après l'achat :
1. Vous recevrez un email de confirmation
2. **Important** : Vérifiez votre email pour valider le domaine (obligatoire ICANN)
3. Cliquez sur le lien de validation dans l'email

⚠️ **Attention** : Si vous ne validez pas l'email dans les 15 jours, votre domaine peut être suspendu !

### Étape 6 : Accéder à la Gestion DNS

Une fois le domaine acheté :
1. Connectez-vous au panneau de contrôle du registrar
2. Trouvez la section "DNS Management" ou "Gestion DNS"
3. Vous êtes prêt pour la configuration (section 10.5)

## Coûts à Prévoir

### Coût Initial (Première Année)

| Extension | Prix annuel | Usage recommandé |
|-----------|-------------|------------------|
| `.com` | 10-15€ | Général, international |
| `.fr` | 8-12€ | France, local |
| `.org` | 12-15€ | Associations, organisations |
| `.net` | 10-15€ | Technique, réseau |
| `.io` | 30-40€ | Tech, startups |
| `.dev` | 12-20€ | Développement |
| `.app` | 15-20€ | Applications |
| `.tech` | 40-50€ | Technologie |

### Renouvellement

**Important** : Le prix de renouvellement peut être différent du prix initial !

Exemple courant :
- Première année : 9.99€ (prix promotionnel)
- Renouvellement : 14.99€/an (prix normal)

**Conseil** : Vérifiez toujours le prix de renouvellement avant d'acheter.

### Coûts Annuels

Pour un projet typique :
```
Domaine .com :        12€/an
WHOIS privacy :       Gratuit (généralement)
DNS hosting :         Gratuit (inclus)
Certificat SSL :      Gratuit (Let's Encrypt)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Total :               12€/an
```

**C'est tout !** Pas besoin de payer pour autre chose.

## Alternatives pour Environnements de Développement

Si vous ne voulez pas acheter de domaine pour un lab de développement :

### Option 1 : Services DNS Dynamiques (Recommandé)

Utilisez **nip.io** ou **xip.io** :

```yaml
# Gratuit, sans inscription, fonctionne immédiatement
host: mon-app.192.168.1.100.nip.io
```

**Parfait pour** :
- Apprentissage Kubernetes
- Labs personnels
- Tests locaux
- Démonstrations internes

### Option 2 : Fichier hosts Local

Éditez `/etc/hosts` :
```
192.168.1.100   mon-site.local
192.168.1.100   api.mon-site.local
```

**Parfait pour** :
- Tests complètement hors ligne
- Développement isolé
- Pas de dépendance externe

### Option 3 : Sous-domaine d'un Domaine Existant

Si vous avez déjà un domaine, utilisez un sous-domaine pour vos tests :
```
lab.mon-domaine-existant.com
```

**Avantages** :
- Utilise votre domaine existant
- Peut obtenir des certificats Let's Encrypt
- Plus professionnel que nip.io

## Conseils et Bonnes Pratiques

### 1. Protection de la Vie Privée (WHOIS Privacy)

Quand vous achetez un domaine, vos informations personnelles (nom, adresse, email, téléphone) sont publiques dans la base de données WHOIS.

**Solution** : Activez **WHOIS Privacy Protection** (souvent gratuit).
- Vos vraies informations sont masquées
- Le registrar fournit des informations proxy

### 2. Renouvellement Automatique

Activez le renouvellement automatique pour ne jamais perdre votre domaine :
- Évite les oublis
- Empêche le vol de domaine après expiration
- Une carte valide doit être enregistrée

### 3. Notifications Email

Assurez-vous de recevoir les emails du registrar :
- Notifications d'expiration
- Rappels de renouvellement
- Alertes de sécurité

### 4. Sécurité du Compte

- ✅ Utilisez un mot de passe fort et unique
- ✅ Activez l'authentification à deux facteurs (2FA) si disponible
- ✅ Ne partagez jamais vos identifiants
- ✅ Vérifiez régulièrement votre compte

### 5. Transfert de Domaine

Si vous n'êtes pas satisfait de votre registrar, vous pouvez transférer votre domaine :
- Possible après 60 jours
- Coûte généralement le prix d'une année supplémentaire
- Conserve votre historique et vos enregistrements DNS

### 6. Planification Long Terme

Si votre projet est sérieux :
- Achetez le domaine pour plusieurs années (réduction de prix)
- Achetez plusieurs extensions (`.com`, `.fr`, `.io`) pour protéger votre marque
- Enregistrez des variantes avec/sans tirets

## Ce Qui Vient Ensuite

Une fois votre domaine acquis, vous devrez :

1. **Configurer les DNS** (section 10.5) pour pointer vers votre cluster MicroK8s
2. **Configurer le routage réseau** (sections 10.6 et 10.7) pour rendre votre cluster accessible
3. **Créer des règles Ingress** (sections 10.8 et 10.9) pour router le trafic
4. **Ajouter des certificats SSL** (chapitre 11) pour HTTPS

## Tableau Récapitulatif des Options

| Option | Coût | Professionnalisme | Facilité | Usage recommandé |
|--------|------|-------------------|----------|------------------|
| **Domaine payant** | 10-20€/an | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐ | Production |
| **Domaine gratuit** | Gratuit | ⭐⭐ | ⭐⭐⭐⭐ | Tests temporaires |
| **nip.io / xip.io** | Gratuit | ⭐ | ⭐⭐⭐⭐⭐ | Labs, développement |
| **Fichier hosts** | Gratuit | ⭐ | ⭐⭐⭐ | Tests locaux |

## Points Clés à Retenir

🔑 **Un nom de domaine est essentiel** pour utiliser pleinement Ingress avec routage par nom d'hôte

🔑 **10-20€/an** pour un domaine professionnel (.com, .fr)

🔑 **Sous-domaines illimités** gratuits après achat du domaine principal

🔑 **Registrars recommandés** : Namecheap, OVH, Gandi, Cloudflare

🔑 **WHOIS Privacy** : à activer pour protéger vos données personnelles

🔑 **Pour les labs** : utilisez nip.io (gratuit, immédiat, sans inscription)

🔑 **Vérifiez l'email** après achat pour valider le domaine

🔑 **Renouvellement automatique** : activez-le pour ne jamais perdre votre domaine

## Cas d'Usage Pratiques

### Pour Apprendre Kubernetes (Lab Personnel)
**Recommandation** : nip.io
```
Coût : Gratuit
Configuration : Immédiate
Exemple : mon-app.192.168.1.100.nip.io
```

### Pour un Projet Personnel Sérieux
**Recommandation** : Domaine .com ou .fr
```
Coût : 10-15€/an
Registrar : Namecheap ou OVH
Exemple : mon-projet.com
```

### Pour une Entreprise ou Startup
**Recommandation** : Domaine .com + .fr + .io
```
Coût : 30-60€/an (plusieurs extensions)
Registrar : Cloudflare ou Gandi
Protection : WHOIS privacy + 2FA
```

### Pour une Association
**Recommandation** : Domaine .org ou .fr
```
Coût : 12-15€/an
Registrar : OVH ou Gandi
Exemple : mon-asso.org
```

## Conclusion

L'acquisition d'un nom de domaine est une étape simple mais fondamentale pour exposer vos applications Kubernetes de manière professionnelle. Pour un lab personnel, les services comme nip.io suffisent largement. Pour un projet sérieux, investir 10-15€/an dans un domaine .com ou .fr est un choix judicieux.

Une fois votre domaine en main (ou votre solution alternative choisie), vous êtes prêt à passer à la configuration DNS, qui fera le lien entre votre nom de domaine et votre cluster MicroK8s !

---

**📚 Résumé du chapitre** : Un nom de domaine est nécessaire pour utiliser Ingress avec routage par nom d'hôte et obtenir des certificats SSL. Pour la production, achetez un domaine (10-20€/an) chez un registrar fiable comme Namecheap ou OVH. Pour les labs, utilisez nip.io (gratuit, immédiat). Activez WHOIS privacy et le renouvellement automatique pour protéger votre domaine.

⏭️ [Configuration DNS externe et enregistrements](/10-ingress-et-routage/05-configuration-dns-externe-et-enregistrements.md)
