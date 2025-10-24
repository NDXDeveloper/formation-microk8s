🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 10.5 Configuration DNS externe et enregistrements

## Introduction

Maintenant que vous avez acquis un nom de domaine, il faut le **connecter** à votre cluster MicroK8s. C'est comme avoir une adresse postale (le domaine) : il faut maintenant indiquer au service postal (Internet) où se trouve réellement votre maison (votre serveur).

Cette connexion se fait via le **DNS** (Domain Name System). Dans ce chapitre, nous allons comprendre ce qu'est le DNS, quels types d'enregistrements existent, et comment les configurer pour que vos applications soient accessibles via votre nom de domaine.

## Qu'est-ce que le DNS ?

### Définition Simple

Le **DNS** (Domain Name System) est comme **l'annuaire téléphonique d'Internet**. Il traduit les noms de domaine faciles à retenir (comme `mon-site.com`) en adresses IP compréhensibles par les machines (comme `203.0.113.45`).

### Analogie du Carnet d'Adresses

Imaginez :
- **Vous voulez appeler votre ami** : vous cherchez "Jean Dupont" dans votre téléphone
- **Votre téléphone traduit** : "Jean Dupont" → "06 12 34 56 78"
- **Vous composez** : le numéro et l'appel se connecte

Le DNS fonctionne pareil :
- **Utilisateur tape** : `mon-site.com` dans le navigateur
- **DNS traduit** : `mon-site.com` → `203.0.113.45`
- **Navigateur se connecte** : au serveur via l'IP et affiche le site

### Le Fonctionnement du DNS

Quand quelqu'un visite `blog.mon-site.com` :

```
1. Navigateur : "Quelle est l'IP de blog.mon-site.com ?"
   ↓
2. DNS Local (cache) : "Je ne sais pas, je demande..."
   ↓
3. DNS Récursif (FAI) : "Je cherche..."
   ↓
4. Serveurs DNS Racine : "Demandez aux serveurs .com"
   ↓
5. Serveurs .com : "Demandez aux serveurs de mon-site.com"
   ↓
6. Serveurs Autoritaires (votre registrar) : "C'est 203.0.113.45 !"
   ↓
7. Réponse remonte jusqu'au navigateur
   ↓
8. Navigateur se connecte à 203.0.113.45
```

Ce processus prend généralement quelques millisecondes.

## Les Types d'Enregistrements DNS

Le DNS utilise différents types d'enregistrements pour différents usages. Voici les principaux que vous devez connaître :

### 1. Enregistrement A (Address)

**Usage** : Fait correspondre un nom de domaine à une **adresse IPv4**.

**Format** :
```
Nom              Type    Valeur
mon-site.com     A       203.0.113.45
```

**Signification** : "Quand quelqu'un demande `mon-site.com`, dirige-le vers `203.0.113.45`"

**C'est le plus important** pour connecter votre domaine à votre serveur MicroK8s !

**Exemple pratique** :
```
blog.mon-site.com    A    203.0.113.45
api.mon-site.com     A    203.0.113.45
www.mon-site.com     A    203.0.113.45
```

Tous vos sous-domaines peuvent pointer vers la même IP (celle de votre serveur MicroK8s).

### 2. Enregistrement AAAA (IPv6 Address)

**Usage** : Comme l'enregistrement A, mais pour les **adresses IPv6**.

**Format** :
```
Nom              Type    Valeur
mon-site.com     AAAA    2001:db8::1
```

**Note** : Utile seulement si votre serveur a une IPv6 publique. Pour la plupart des labs, vous n'en aurez pas besoin.

### 3. Enregistrement CNAME (Canonical Name)

**Usage** : Crée un **alias** d'un nom de domaine vers un autre.

**Format** :
```
Nom              Type     Valeur
www              CNAME    mon-site.com
```

**Signification** : "`www.mon-site.com` est un alias de `mon-site.com`"

**Important** :
- ✅ Vous pouvez faire : `www.mon-site.com CNAME → mon-site.com`
- ❌ Vous ne pouvez PAS faire : `mon-site.com CNAME → autre-domaine.com` (pas de CNAME sur un domaine racine)

**Cas d'usage** : Rediriger `www` vers votre domaine principal.

### 4. Enregistrement MX (Mail Exchange)

**Usage** : Indique quel serveur gère les **emails** pour ce domaine.

**Format** :
```
Nom              Type    Priorité    Valeur
mon-site.com     MX      10          mail.mon-site.com
```

**Note** : Pas nécessaire pour Kubernetes/Ingress. Utile seulement si vous gérez des emails.

### 5. Enregistrement TXT (Text)

**Usage** : Stocke du **texte arbitraire**. Utilisé pour :
- Vérification de propriété du domaine
- Configuration SPF (email)
- Validation Let's Encrypt (certificats SSL)

**Format** :
```
Nom              Type    Valeur
mon-site.com     TXT     "v=spf1 include:_spf.google.com ~all"
```

**Pour Kubernetes** : Let's Encrypt peut utiliser des enregistrements TXT pour vérifier que vous possédez le domaine.

### 6. Enregistrement NS (Name Server)

**Usage** : Indique quels **serveurs DNS** sont autoritaires pour ce domaine.

**Format** :
```
Nom              Type    Valeur
mon-site.com     NS      ns1.registrar.com
mon-site.com     NS      ns2.registrar.com
```

**Note** : Généralement configuré automatiquement par votre registrar. Ne le modifiez que si vous changez de serveurs DNS.

### 7. Enregistrement Wildcard (*)

**Usage** : Correspond à **tous les sous-domaines** non explicitement définis.

**Format** :
```
Nom              Type    Valeur
*.mon-site.com   A       203.0.113.45
```

**Signification** : Tous les sous-domaines (`test.mon-site.com`, `demo.mon-site.com`, etc.) pointent vers cette IP.

**Très utile pour Kubernetes** : évite de créer un enregistrement A pour chaque sous-domaine !

## Configuration DNS pour MicroK8s

### Étape 1 : Trouver l'IP de Votre Serveur

Avant de configurer le DNS, vous devez connaître l'**adresse IP publique** de votre serveur MicroK8s.

#### Si Votre Serveur est Chez Vous (Réseau Domestique)

Trouvez votre IP publique (celle de votre box Internet) :

**Méthode 1** : Via un site web
```bash
curl ifconfig.me
```

**Méthode 2** : Via un autre service
```bash
curl icanhazip.com
```

**Méthode 3** : Via votre navigateur
Allez sur https://www.whatismyip.com

**Note importante** : Si vous êtes derrière une box, c'est l'IP de votre box qui sera publique, pas celle de votre serveur. Vous devrez configurer du **port forwarding** (section 10.6).

#### Si Votre Serveur est dans le Cloud (VPS, Serveur Dédié)

Votre hébergeur vous a fourni une IP publique. Trouvez-la :

**Méthode 1** : Dans le panneau de contrôle de votre hébergeur
**Méthode 2** : Sur le serveur lui-même
```bash
ip addr show
# Cherchez l'interface publique (généralement eth0 ou ens3)
```

Exemple d'IP publique : `203.0.113.45`

### Étape 2 : Se Connecter à l'Interface DNS du Registrar

Connectez-vous au site de votre registrar (Namecheap, OVH, Gandi, etc.) et trouvez la section de gestion DNS.

#### Chez Namecheap

1. Connectez-vous à https://www.namecheap.com
2. Allez dans "Dashboard" → "Domain List"
3. Cliquez sur "Manage" à côté de votre domaine
4. Cliquez sur l'onglet "Advanced DNS"

#### Chez OVH

1. Connectez-vous à https://www.ovh.com/manager/
2. Allez dans "Web Cloud" → "Noms de domaine"
3. Sélectionnez votre domaine
4. Cliquez sur l'onglet "Zone DNS"
5. Cliquez sur "Modifier en mode textuel" ou "Ajouter une entrée"

#### Chez Gandi

1. Connectez-vous à https://admin.gandi.net
2. Allez dans "Noms de domaine"
3. Cliquez sur votre domaine
4. Allez dans "Enregistrements DNS"

#### Chez Cloudflare

1. Connectez-vous à https://dash.cloudflare.com
2. Sélectionnez votre domaine
3. Allez dans "DNS" → "Records"

### Étape 3 : Configuration Minimale (Domaine Principal)

**Objectif** : Faire pointer `mon-site.com` vers votre serveur.

Créez un enregistrement A :

```
Type : A
Nom : @ (ou vide, ou mon-site.com selon le registrar)
Valeur : 203.0.113.45 (votre IP publique)
TTL : 3600 (ou "Automatic")
```

**Explication** :
- `@` représente le domaine racine (`mon-site.com`)
- L'IP est celle de votre serveur MicroK8s
- TTL (Time To Live) : durée de cache en secondes (3600 = 1 heure)

**Résultat** : `mon-site.com` → `203.0.113.45`

### Étape 4 : Configuration avec Sous-domaines

**Objectif** : Faire pointer plusieurs sous-domaines vers votre serveur.

Créez plusieurs enregistrements A :

```
Type : A
Nom : www
Valeur : 203.0.113.45
TTL : 3600

Type : A
Nom : blog
Valeur : 203.0.113.45
TTL : 3600

Type : A
Nom : api
Valeur : 203.0.113.45
TTL : 3600
```

**Résultat** :
- `www.mon-site.com` → `203.0.113.45`
- `blog.mon-site.com` → `203.0.113.45`
- `api.mon-site.com` → `203.0.113.45`

### Étape 5 : Configuration avec Wildcard (Recommandé)

**Objectif** : Tous les sous-domaines pointent automatiquement vers votre serveur.

Créez un enregistrement wildcard :

```
Type : A
Nom : *
Valeur : 203.0.113.45
TTL : 3600
```

**Avantages** :
- ✅ Pas besoin de créer un enregistrement pour chaque sous-domaine
- ✅ `test.mon-site.com`, `demo.mon-site.com`, etc. fonctionnent automatiquement
- ✅ Idéal pour Kubernetes avec de nombreux services

**Note** : Vous pouvez combiner wildcard et enregistrements spécifiques. Les enregistrements spécifiques ont la priorité.

### Étape 6 : Configuration Complète Recommandée

Voici une configuration DNS complète et optimale pour MicroK8s :

```
# Domaine principal
Type : A
Nom : @
Valeur : 203.0.113.45
TTL : 3600

# WWW (alias vers domaine principal)
Type : CNAME
Nom : www
Valeur : mon-site.com
TTL : 3600

# Wildcard pour tous les autres sous-domaines
Type : A
Nom : *
Valeur : 203.0.113.45
TTL : 3600
```

**Cette configuration permet** :
- `mon-site.com` → serveur MicroK8s
- `www.mon-site.com` → redirige vers `mon-site.com`
- Tous les autres sous-domaines → serveur MicroK8s

**C'est la configuration idéale pour Kubernetes !**

## Exemples par Registrar

### Exemple Namecheap

Interface "Advanced DNS" :

| Type | Host | Value | TTL |
|------|------|-------|-----|
| A Record | @ | 203.0.113.45 | Automatic |
| A Record | * | 203.0.113.45 | Automatic |
| CNAME Record | www | mon-site.com. | Automatic |

**Note** : Namecheap ajoute automatiquement un point final dans les CNAME.

### Exemple OVH

Zone DNS en mode textuel :

```
$TTL 3600
@       IN A     203.0.113.45
*       IN A     203.0.113.45
www     IN CNAME mon-site.com.
```

### Exemple Cloudflare

Dans "DNS" → "Records" :

| Type | Name | Content | Proxy status | TTL |
|------|------|---------|--------------|-----|
| A | @ | 203.0.113.45 | DNS only (gris) | Auto |
| A | * | 203.0.113.45 | DNS only (gris) | Auto |
| CNAME | www | mon-site.com | DNS only (gris) | Auto |

**Important avec Cloudflare** : Désactivez le proxy (nuage gris, pas orange) sinon Cloudflare interceptera le trafic.

## Propagation DNS

### Qu'est-ce que la Propagation ?

Après avoir modifié vos enregistrements DNS, ces changements ne sont pas instantanés partout dans le monde. C'est ce qu'on appelle la **propagation DNS**.

**Analogie** : C'est comme mettre à jour un annuaire téléphonique. Il faut du temps pour que tous les exemplaires dans le monde soient mis à jour.

### Durée de Propagation

- **Minimum** : Quelques minutes (si tout va bien)
- **Typique** : 1 à 4 heures
- **Maximum** : 24 à 48 heures (cas extrêmes)

**Facteur principal** : Le **TTL** (Time To Live) que vous avez configuré.

### Pourquoi C'est Long ?

Les serveurs DNS **mettent en cache** les réponses pour améliorer les performances. Si un serveur a en cache `mon-site.com = 198.51.100.20` avec un TTL de 24h, il ne redemandera pas la vraie valeur avant 24h.

### Astuce pour Accélérer

Si vous prévoyez un changement de DNS :
1. **24-48h avant** : Réduisez le TTL à 300 (5 minutes)
2. Attendez que ce changement se propage
3. **Faites votre changement** (nouvelle IP par exemple)
4. Attendez 5-10 minutes
5. **Restaurez le TTL** à 3600 (1 heure)

## Vérification de la Configuration DNS

### Méthode 1 : Commande dig (Linux/macOS)

`dig` est l'outil de référence pour interroger le DNS.

**Vérifier un enregistrement A** :
```bash
dig mon-site.com A +short
```

Résultat attendu :
```
203.0.113.45
```

**Vérifier un sous-domaine** :
```bash
dig blog.mon-site.com A +short
```

**Vérifier le wildcard** :
```bash
dig test.mon-site.com A +short
```

**Voir les détails complets** :
```bash
dig mon-site.com A
```

### Méthode 2 : Commande nslookup (Tous OS)

`nslookup` est disponible sur Windows, Linux et macOS.

```bash
nslookup mon-site.com
```

Résultat attendu :
```
Server:  8.8.8.8
Address: 8.8.8.8#53

Non-authoritative answer:
Name:    mon-site.com
Address: 203.0.113.45
```

**Vérifier un sous-domaine** :
```bash
nslookup blog.mon-site.com
```

### Méthode 3 : Commande host (Linux/macOS)

```bash
host mon-site.com
```

Résultat :
```
mon-site.com has address 203.0.113.45
```

### Méthode 4 : Sites Web en Ligne

Si vous n'avez pas accès aux commandes :

- **DNSChecker** : https://dnschecker.org
  - Vérifie la propagation depuis plusieurs localisations dans le monde

- **What's My DNS** : https://www.whatsmydns.net
  - Visualisation mondiale de la propagation

- **DNS Propagation Checker** : https://www.whatsmydns.com

**Comment utiliser** :
1. Entrez votre nom de domaine
2. Sélectionnez le type d'enregistrement (A, CNAME, etc.)
3. Cliquez sur "Search" ou "Check"
4. Observez les résultats depuis différents serveurs DNS

### Méthode 5 : Depuis le Navigateur

**Test simple** : Ouvrez votre navigateur et tapez :
```
http://mon-site.com
```

**Si le DNS est configuré correctement** :
- Le navigateur essaiera de se connecter à votre serveur
- Vous verrez soit votre application, soit une erreur de connexion (normal si le firewall/port forwarding n'est pas configuré)

**Si le DNS n'est pas encore propagé** :
- Vous verrez "This site can't be reached" ou "DNS_PROBE_FINISHED_NXDOMAIN"

## Spécificités selon l'Hébergement

### Serveur à Domicile (Derrière une Box)

**Problème** : Votre IP publique change régulièrement (IP dynamique).

**Solution 1** : DynDNS (DNS Dynamique)
- Services : No-IP, DuckDNS, FreeDNS
- Installez un client qui met à jour l'IP automatiquement
- Votre domaine principal pointe vers le nom DynDNS

**Solution 2** : Cloudflare avec script
- Script qui met à jour l'enregistrement A via l'API Cloudflare
- S'exécute régulièrement (cron) pour détecter les changements d'IP

### Serveur VPS/Cloud

**Avantage** : IP fixe et stable.

**Configuration** : Simple et directe, pas de complications.

### Plusieurs Serveurs (Load Balancing)

**Si vous avez plusieurs serveurs** :

Option 1 : Plusieurs enregistrements A
```
mon-site.com    A    203.0.113.45
mon-site.com    A    203.0.113.46
mon-site.com    A    203.0.113.47
```

Le DNS fera du **round-robin** (rotation) entre les IPs.

Option 2 : Utiliser un vrai load balancer
- Configuration plus avancée, hors scope pour un lab

## Cas Spéciaux et Configurations Avancées

### Utiliser Cloudflare comme DNS

Cloudflare offre des serveurs DNS gratuits, ultra-rapides et avec des fonctionnalités de sécurité.

**Avantages** :
- ✅ DNS ultra-rapide (1.1.1.1)
- ✅ Protection DDoS
- ✅ Cache et CDN
- ✅ Analytics gratuit
- ✅ SSL flexible

**Étapes** :
1. Créez un compte sur https://cloudflare.com
2. Ajoutez votre domaine
3. Cloudflare scanne vos enregistrements DNS actuels
4. Changez les nameservers chez votre registrar vers ceux de Cloudflare
5. Gérez vos DNS depuis Cloudflare

**Note** : Si vous utilisez le proxy Cloudflare (nuage orange), désactivez-le pour Kubernetes.

### Configuration pour Let's Encrypt (Préparation)

Pour obtenir des certificats SSL automatiques (chapitre 11), Let's Encrypt peut utiliser deux méthodes :

**HTTP-01 Challenge** : Aucune configuration DNS spéciale nécessaire.

**DNS-01 Challenge** : Nécessite de créer des enregistrements TXT.
- Plus complexe mais fonctionne avec des wildcards
- Nous verrons cela au chapitre 11

### IPv6 (Optionnel)

Si votre serveur a une IPv6 :

```
Type : AAAA
Nom : @
Valeur : 2001:db8::1
TTL : 3600

Type : AAAA
Nom : *
Valeur : 2001:db8::1
TTL : 3600
```

**Note** : IPv6 est optionnel. IPv4 suffit pour commencer.

## Dépannage des Problèmes DNS

### Problème 1 : DNS Ne Résout Pas (NXDOMAIN)

**Symptôme** :
```bash
dig mon-site.com
# NXDOMAIN (domain does not exist)
```

**Causes possibles** :
- Les enregistrements n'ont pas encore été créés
- Faute de frappe dans le nom de domaine
- Le domaine n'est pas activé

**Solutions** :
1. Vérifiez que vous avez bien sauvegardé les enregistrements DNS
2. Attendez quelques minutes
3. Vérifiez l'orthographe du domaine

### Problème 2 : Ancienne IP Encore en Cache

**Symptôme** : Vous avez changé l'IP mais `dig` retourne encore l'ancienne.

**Cause** : Cache DNS local ou chez votre FAI.

**Solutions** :
1. Videz le cache DNS local :
   ```bash
   # Linux
   sudo systemd-resolve --flush-caches

   # macOS
   sudo dscacheutil -flushcache

   # Windows
   ipconfig /flushdns
   ```

2. Utilisez un serveur DNS public pour tester :
   ```bash
   dig @8.8.8.8 mon-site.com
   dig @1.1.1.1 mon-site.com
   ```

3. Attendez la propagation complète

### Problème 3 : Propagation Très Lente

**Symptôme** : Plus de 24h et certains endroits ne voient pas le changement.

**Causes** :
- TTL très élevé (86400 = 24h)
- Problème chez le registrar

**Solutions** :
1. Vérifiez le TTL actuel :
   ```bash
   dig mon-site.com | grep "IN.*A"
   ```
2. Réduisez le TTL à 300 pour les futurs changements
3. Contactez le support du registrar si le problème persiste

### Problème 4 : Wildcard Ne Fonctionne Pas

**Symptôme** : `mon-site.com` fonctionne mais pas `test.mon-site.com`.

**Causes** :
- Enregistrement wildcard mal configuré
- Enregistrement spécifique qui prend le dessus

**Solutions** :
1. Vérifiez la configuration :
   ```bash
   dig test.mon-site.com A +short
   ```
2. Assurez-vous d'avoir créé `*.mon-site.com` et pas `*`
3. Supprimez les enregistrements conflictuels

### Problème 5 : CNAME Loop (Boucle)

**Symptôme** : Erreur "CNAME loop detected".

**Cause** : CNAME qui pointe vers lui-même ou forme une boucle.

**Exemple de boucle** :
```
www  CNAME  blog.mon-site.com
blog CNAME  www.mon-site.com
```

**Solution** : Cassez la boucle, faites pointer vers un enregistrement A final.

## Vérification Finale : Checklist

Avant de passer à la section suivante, vérifiez que :

- [ ] Vous avez créé un enregistrement A pour `@` (domaine principal)
- [ ] Vous avez créé un wildcard `*` ou des enregistrements A pour vos sous-domaines
- [ ] `dig mon-site.com +short` retourne votre IP
- [ ] `dig test.mon-site.com +short` retourne votre IP (si wildcard)
- [ ] `nslookup mon-site.com` fonctionne
- [ ] Les serveurs DNS publics (8.8.8.8, 1.1.1.1) résolvent correctement
- [ ] Le site https://dnschecker.org montre une propagation complète
- [ ] Le TTL est raisonnable (3600 ou moins)

## Configuration DNS : Tableau Récapitulatif

| Registrar | Interface | Difficulté | Documentation |
|-----------|-----------|------------|---------------|
| **Namecheap** | Intuitive | ⭐⭐ | Excellente |
| **OVH** | Complète | ⭐⭐⭐ | Bonne |
| **Gandi** | Simple | ⭐⭐ | Excellente |
| **Cloudflare** | Moderne | ⭐⭐ | Excellente |
| **Google Domains** | Épurée | ⭐ | Bonne |

## Points Clés à Retenir

🔑 **DNS traduit** les noms de domaine en adresses IP

🔑 **Enregistrement A** : le plus important, fait correspondre domaine → IP

🔑 **Wildcard (*)** : tous les sous-domaines pointent automatiquement vers votre IP

🔑 **Configuration minimale** : `@` (domaine) et `*` (wildcard) en enregistrements A

🔑 **Propagation DNS** : 1-4h typiquement, jusqu'à 48h maximum

🔑 **Vérification** : utilisez `dig`, `nslookup`, ou https://dnschecker.org

🔑 **TTL faible** : facilite les changements futurs (300-3600 secondes)

🔑 **Cloudflare** : option excellente pour DNS rapide et sécurisé

## Commandes de Référence Rapide

```bash
# Vérifier un enregistrement A
dig mon-site.com A +short

# Vérifier depuis un DNS spécifique
dig @8.8.8.8 mon-site.com

# Vérifier tous les types d'enregistrements
dig mon-site.com ANY

# Vérifier avec nslookup
nslookup mon-site.com

# Vérifier avec host
host mon-site.com

# Vider le cache DNS local
sudo systemd-resolve --flush-caches  # Linux
sudo dscacheutil -flushcache          # macOS
ipconfig /flushdns                    # Windows

# Tracer le chemin de résolution DNS
dig +trace mon-site.com
```

## Prochaines Étapes

Maintenant que votre DNS est configuré et que votre domaine pointe vers votre serveur, vous devez :

1. **Configurer le port forwarding** (section 10.6) si vous êtes derrière une box
2. **Configurer le firewall** (section 10.7) pour autoriser le trafic
3. **Créer vos règles Ingress** (sections 10.8 et 10.9) pour router les requêtes
4. **Ajouter des certificats SSL** (chapitre 11) pour HTTPS

Votre domaine est maintenant correctement configuré pour pointer vers votre cluster MicroK8s. La prochaine étape consiste à rendre ce cluster accessible depuis Internet !

---

**📚 Résumé du chapitre** : Le DNS traduit les noms de domaine en adresses IP. Configurez un enregistrement A pour `@` (domaine principal) et un wildcard `*` pour tous les sous-domaines, tous pointant vers l'IP publique de votre serveur MicroK8s. La propagation DNS prend 1-4h typiquement. Vérifiez avec `dig` ou https://dnschecker.org. Une fois propagé, votre domaine est prêt à être utilisé avec Ingress.

⏭️ [Redirection de ports et NAT (exposition externe)](/10-ingress-et-routage/06-redirection-de-ports-et-nat.md)
