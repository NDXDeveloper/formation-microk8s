🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 10.6 Redirection de ports et NAT (exposition externe)

## Introduction

Votre DNS est configuré, votre domaine pointe vers votre IP publique... mais si votre serveur MicroK8s est chez vous, derrière une box Internet, **il n'est toujours pas accessible depuis l'extérieur** ! Pourquoi ? À cause du **NAT** (Network Address Translation).

Dans ce chapitre, nous allons comprendre ce qu'est le NAT, pourquoi il bloque l'accès à votre cluster, et comment configurer la **redirection de ports** (port forwarding) pour rendre vos applications Kubernetes accessibles depuis Internet.

**Note importante** : Cette section concerne principalement les installations **à domicile** (derrière une box/routeur). Si votre serveur est dans le cloud (VPS, serveur dédié), vous pouvez passer directement à la section 10.7 sur les firewalls.

## Le Problème : Réseaux Privés et NAT

### IP Privée vs IP Publique

Il existe deux types d'adresses IP :

#### IP Publique
- **Unique sur Internet** : comme une adresse postale unique
- **Routable sur Internet** : accessible depuis n'importe où
- **Exemple** : `203.0.113.45`
- **Fournie par votre FAI** (Fournisseur d'Accès Internet)
- **Coûteuse** : nombre limité d'IPv4 disponibles

#### IP Privée
- **Uniquement dans votre réseau local** : comme les numéros d'appartements dans un immeuble
- **Non routable sur Internet** : invisible depuis l'extérieur
- **Plages réservées** :
  - `192.168.0.0/16` (192.168.0.0 à 192.168.255.255)
  - `10.0.0.0/8` (10.0.0.0 à 10.255.255.255)
  - `172.16.0.0/12` (172.16.0.0 à 172.31.255.255)
- **Gratuite et illimitée** : réutilisable dans chaque réseau

### Votre Configuration Typique à Domicile

```
Internet (IP publique : 203.0.113.45)
    ↓
┌─────────────────────────────────┐
│   Box Internet / Routeur        │
│   IP publique : 203.0.113.45    │
│   IP privée : 192.168.1.1       │
└─────────────────────────────────┘
    ↓
Réseau Local (192.168.1.0/24)
    ↓
    ├─→ Ordinateur : 192.168.1.10
    ├─→ Smartphone : 192.168.1.20
    ├─→ Serveur MicroK8s : 192.168.1.100
    └─→ TV connectée : 192.168.1.30
```

**Le problème** :
- Votre serveur MicroK8s a l'IP privée `192.168.1.100`
- Votre DNS pointe vers l'IP publique `203.0.113.45` (celle de la box)
- Les requêtes arrivent à la box, mais elle ne sait pas vers quel appareil interne les envoyer !

### Qu'est-ce que le NAT ?

**NAT** (Network Address Translation) est une technique qui permet à **plusieurs appareils de partager une seule IP publique**.

**Analogie de l'immeuble** :
- **IP publique** = Adresse de l'immeuble (123 rue de la Paix)
- **Box/Routeur** = Gardien de l'immeuble
- **IP privées** = Numéros d'appartements (Apt. 1, 2, 3, etc.)

**Fonctionnement du NAT** :

**Connexions sortantes (de l'intérieur vers Internet)** :
```
1. Votre PC (192.168.1.10) veut visiter google.com
2. Box remplace l'IP source par l'IP publique (203.0.113.45)
3. Google répond à 203.0.113.45
4. Box sait que c'est pour le PC et transmet à 192.168.1.10
✅ Fonctionne automatiquement
```

**Connexions entrantes (d'Internet vers l'intérieur)** :
```
1. Quelqu'un sur Internet veut accéder à 203.0.113.45:80
2. Box reçoit la requête
3. ❓ Box ne sait pas vers quel appareil interne rediriger
4. ❌ Connexion bloquée ou rejetée
```

C'est là qu'intervient la **redirection de ports** !

## Qu'est-ce que la Redirection de Ports ?

La **redirection de ports** (port forwarding ou port mapping) est une règle que vous configurez sur votre box pour dire :

> "Quand quelqu'un accède au port X de mon IP publique, redirige vers l'appareil Y sur le port Z de mon réseau local"

**Exemple concret** :
```
Port externe : 80 (HTTP)
IP interne : 192.168.1.100 (serveur MicroK8s)
Port interne : 80
```

**Signification** : "Toutes les requêtes HTTP (port 80) qui arrivent sur mon IP publique doivent être redirigées vers le serveur MicroK8s (192.168.1.100) sur le port 80"

**Schéma avec redirection** :
```
Internet → 203.0.113.45:80 (IP publique)
    ↓
┌─────────────────────────────────┐
│   Box avec règle :              │
│   80 → 192.168.1.100:80         │
└─────────────────────────────────┘
    ↓
Serveur MicroK8s : 192.168.1.100:80
```

## Ports à Rediriger pour Ingress

Pour que NGINX Ingress Controller soit accessible depuis Internet, vous devez rediriger **deux ports** :

### Port 80 (HTTP)

**Protocole** : TCP
**Usage** : Trafic HTTP non chiffré
**Redirection** :
```
Port externe : 80
→ IP interne : 192.168.1.100 (IP de votre serveur MicroK8s)
→ Port interne : 80
```

**Nécessaire pour** :
- Accès HTTP initial (avant HTTPS)
- Validation Let's Encrypt (HTTP-01 challenge)
- Redirections HTTP → HTTPS

### Port 443 (HTTPS)

**Protocole** : TCP
**Usage** : Trafic HTTPS chiffré
**Redirection** :
```
Port externe : 443
→ IP interne : 192.168.1.100
→ Port interne : 443
```

**Nécessaire pour** :
- Accès HTTPS sécurisé
- Certificats SSL/TLS
- Production moderne (HTTPS obligatoire)

### Récapitulatif des Règles

| Port externe | Protocole | IP interne | Port interne | Usage |
|--------------|-----------|------------|--------------|-------|
| 80 | TCP | 192.168.1.100 | 80 | HTTP |
| 443 | TCP | 192.168.1.100 | 443 | HTTPS |

**C'est tout !** Seulement deux règles sont nécessaires pour un Ingress fonctionnel.

## Prérequis

Avant de configurer la redirection de ports :

### 1. Connaître l'IP Privée de Votre Serveur MicroK8s

Sur votre serveur, exécutez :

```bash
hostname -I
# ou
ip addr show
```

Cherchez l'IP dans la plage privée (généralement `192.168.x.x` ou `10.x.x.x`).

**Exemple de sortie** :
```
192.168.1.100 172.17.0.1 172.18.0.1
```

Ici, l'IP de votre serveur est `192.168.1.100`.

### 2. Attribuer une IP Fixe (Recommandé)

Par défaut, votre box attribue des IPs automatiquement via **DHCP**. Le problème : l'IP peut changer au redémarrage !

**Solution 1** : Réservation DHCP (recommandé)
- Dans l'interface de la box, réservez `192.168.1.100` pour votre serveur
- L'IP reste fixe, mais gérée par DHCP

**Solution 2** : IP statique sur le serveur
- Configurez une IP fixe directement sur le serveur
- Plus complexe mais indépendant de la box

**Pour Ubuntu/Debian avec Netplan** :

Éditez `/etc/netplan/01-netcfg.yaml` :
```yaml
network:
  version: 2
  ethernets:
    ens3:  # Remplacez par le nom de votre interface
      dhcp4: no
      addresses:
        - 192.168.1.100/24
      gateway4: 192.168.1.1
      nameservers:
        addresses:
          - 8.8.8.8
          - 8.8.4.4
```

Appliquez :
```bash
sudo netplan apply
```

### 3. Accéder à l'Interface d'Administration de Votre Box

Trouvez l'adresse de configuration de votre box :

**Méthode 1** : Passerelle par défaut
```bash
ip route | grep default
# default via 192.168.1.1 ...
```

**Méthode 2** : Adresses courantes selon les FAI

| FAI (France) | Adresse | Login par défaut |
|--------------|---------|------------------|
| Free (Freebox) | http://mafreebox.freebox.fr | Aucun (création) |
| Orange (Livebox) | http://192.168.1.1 | admin / admin |
| SFR (Box) | http://192.168.1.1 | admin / admin |
| Bouygues | http://192.168.1.254 | admin / admin |

**À l'étranger** : Généralement `192.168.0.1` ou `192.168.1.1`

Ouvrez l'adresse dans votre navigateur.

## Configuration par Type de Box

### Freebox (Free)

#### Étape 1 : Accéder à l'Interface

1. Allez sur http://mafreebox.freebox.fr
2. Si c'est la première fois, créez un compte
3. Connectez-vous

#### Étape 2 : Activer le Mode Avancé

1. Cliquez sur "Paramètres de la Freebox"
2. Sélectionnez "Mode de Configuration : Avancé"

#### Étape 3 : Configurer la Redirection

1. Allez dans "Paramètres de la Freebox"
2. Cliquez sur "Gestion des ports"
3. Cliquez sur "Ajouter une redirection"

**Pour le port 80 (HTTP)** :
```
Port de début : 80
Port de fin : 80
Protocole : TCP
IP de destination : 192.168.1.100
Port de destination : 80
Commentaire : Ingress HTTP
```

**Pour le port 443 (HTTPS)** :
```
Port de début : 443
Port de fin : 443
Protocole : TCP
IP de destination : 192.168.1.100
Port de destination : 443
Commentaire : Ingress HTTPS
```

4. Cliquez sur "Ajouter" pour chaque règle
5. Les règles sont actives immédiatement

### Livebox (Orange)

#### Étape 1 : Connexion

1. Allez sur http://192.168.1.1
2. Login : `admin`
3. Mot de passe : les 8 premiers caractères de la clé Wi-Fi (généralement collée sous la box)

#### Étape 2 : Configuration NAT/PAT

1. Allez dans "Configuration avancée" (en haut à droite)
2. Cliquez sur "NAT/PAT"
3. Cliquez sur "Ajouter une règle"

**Pour le port 80** :
```
Application/Service : Ingress HTTP (personnalisé)
Port externe : 80
Protocole : TCP
Port interne : 80
Équipement : Sélectionnez votre serveur (192.168.1.100)
```

**Pour le port 443** :
```
Application/Service : Ingress HTTPS (personnalisé)
Port externe : 443
Protocole : TCP
Port interne : 443
Équipement : Sélectionnez votre serveur (192.168.1.100)
```

4. Validez chaque règle
5. Redémarrez la Livebox si demandé

### Box SFR

#### Étape 1 : Connexion

1. Allez sur http://192.168.1.1
2. Login : `admin`
3. Mot de passe : inscrit au dos de la box

#### Étape 2 : Configuration NAT

1. Allez dans "Réseau" → "NAT"
2. Cliquez sur "Configuration NAT"
3. Activez le NAT si nécessaire
4. Cliquez sur "Ajouter une règle de redirection de port"

**Configuration** :
```
Port 80 :
- Type : TCP
- Port externe début : 80
- Port externe fin : 80
- IP serveur : 192.168.1.100
- Port interne : 80

Port 443 :
- Type : TCP
- Port externe début : 443
- Port externe fin : 443
- IP serveur : 192.168.1.100
- Port interne : 443
```

5. Enregistrez et appliquez

### Box Bouygues Telecom

#### Étape 1 : Connexion

1. Allez sur http://192.168.1.254
2. Login : `admin`
3. Mot de passe : inscrit au dos de la box

#### Étape 2 : Configuration

1. Allez dans "Services de la Bbox" → "NAT"
2. Cliquez sur "Ajouter"

**Pour chaque port** :
```
Nom : Ingress HTTP (ou HTTPS)
Protocole : TCP
Port externe : 80 (ou 443)
IP de destination : 192.168.1.100
Port interne : 80 (ou 443)
```

3. Validez et enregistrez

### Routeurs Génériques (TP-Link, Netgear, Asus, etc.)

**Principe général** :

1. Connectez-vous à l'interface web (généralement `192.168.0.1` ou `192.168.1.1`)
2. Cherchez une section nommée :
   - "Port Forwarding"
   - "Virtual Server"
   - "NAT Forwarding"
   - "Applications & Gaming" (certains routeurs)
3. Ajoutez les règles pour les ports 80 et 443

**Exemple TP-Link** :
```
Service Type : Custom
External Port : 80
Internal IP : 192.168.1.100
Internal Port : 80
Protocol : TCP
Status : Enabled
```

## Vérification de la Configuration

### Étape 1 : Vérifier les Règles dans l'Interface

Retournez dans l'interface de votre box et vérifiez que les deux règles (80 et 443) sont bien actives/activées.

### Étape 2 : Vérifier depuis l'Intérieur

Depuis votre réseau local, testez l'accès à votre serveur :

```bash
# Test HTTP
curl http://192.168.1.100

# Test HTTPS (si déjà configuré)
curl -k https://192.168.1.100
```

Si cela fonctionne, votre serveur MicroK8s répond correctement.

### Étape 3 : Vérifier depuis l'Extérieur

**Option 1** : Via mobile (4G/5G)
- Désactivez le Wi-Fi sur votre smartphone
- Ouvrez le navigateur
- Allez sur `http://votre-ip-publique.com` ou `http://votre-domaine.com`

**Option 2** : Via un VPS/serveur distant
```bash
# Sur un serveur distant
curl -I http://votre-domaine.com
```

**Option 3** : Via des outils en ligne
- https://www.yougetsignal.com/tools/open-ports/
- https://canyouseeme.org
- Entrez votre IP publique et le port 80

### Réponses Attendues

**Si la redirection fonctionne** :
- Code HTTP 404, 502, ou réponse de NGINX
- Même une erreur est bon signe : cela prouve que la connexion arrive au serveur !

**Si la redirection ne fonctionne pas** :
- Timeout (pas de réponse)
- Connection refused
- Le port apparaît comme "closed" sur les testeurs en ligne

## Dépannage des Problèmes de Redirection

### Problème 1 : Port Fermé (Closed)

**Symptôme** : Les outils en ligne disent "Port 80 is closed".

**Causes possibles** :
1. La redirection n'est pas configurée
2. La redirection est désactivée
3. L'IP interne est incorrecte
4. Le firewall de la box bloque le port

**Solutions** :
1. Vérifiez les règles dans l'interface de la box
2. Assurez-vous que les règles sont "activées"
3. Vérifiez l'IP de destination (doit être celle du serveur MicroK8s)
4. Désactivez temporairement le firewall de la box pour tester

### Problème 2 : Timeout (Pas de Réponse)

**Symptôme** : La connexion prend du temps puis timeout.

**Causes possibles** :
1. Le serveur MicroK8s n'écoute pas sur ce port
2. Le firewall du serveur bloque le port
3. NGINX Ingress Controller n'est pas démarré

**Solutions** :
1. Vérifiez que le pod Ingress est running :
   ```bash
   microk8s kubectl get pods -n ingress
   ```

2. Vérifiez que le port est ouvert sur le serveur :
   ```bash
   sudo ss -tlnp | grep :80
   sudo ss -tlnp | grep :443
   ```

3. Vérifiez le firewall du serveur (section 10.7)

### Problème 3 : Connection Refused

**Symptôme** : Erreur "Connection refused" immédiate.

**Cause** : Le port n'est pas écouté sur le serveur.

**Solutions** :
1. Vérifiez que NGINX Ingress Controller écoute sur les bons ports :
   ```bash
   microk8s kubectl get pods -n ingress
   microk8s kubectl logs -n ingress -l app.kubernetes.io/name=ingress-nginx
   ```

2. Vérifiez le DaemonSet :
   ```bash
   microk8s kubectl get daemonset -n ingress
   ```

### Problème 4 : Fonctionne en Interne mais pas en Externe

**Symptôme** : Ça marche depuis votre réseau local (192.168.1.x) mais pas depuis Internet.

**Causes possibles** :
1. Redirection de ports mal configurée
2. Firewall de la box
3. CGNAT (voir section suivante)

**Solutions** :
1. Testez depuis un réseau externe (4G, VPS)
2. Vérifiez que vous utilisez l'IP publique, pas l'IP privée
3. Vérifiez le firewall de la box

### Problème 5 : CGNAT (Carrier-Grade NAT)

**Qu'est-ce que c'est ?**
Certains FAI (notamment mobile/4G) utilisent du NAT supplémentaire. Votre "IP publique" n'est pas vraiment publique !

**Comment détecter** :
1. Trouvez votre IP publique : https://www.whatismyip.com
2. Comparez avec l'IP vue par votre box
3. Si elles sont différentes → vous êtes derrière CGNAT

**Solutions** :
- Contactez votre FAI pour obtenir une vraie IP publique (parfois payant)
- Utilisez un VPN avec port forwarding
- Utilisez Cloudflare Tunnel (voir alternatives)

## Sécurité : Bonnes Pratiques

### 1. N'Ouvrez QUE les Ports Nécessaires

❌ **Mauvais** : Ouvrir tous les ports (DMZ)
✅ **Bon** : Ouvrir uniquement 80 et 443

### 2. Firewall sur le Serveur

La redirection de ports ouvre votre serveur à Internet. **Un firewall sur le serveur est indispensable** (section 10.7).

### 3. Changez les Mots de Passe par Défaut

Si vous ne l'avez pas déjà fait, changez :
- Le mot de passe de la box
- Le mot de passe du serveur
- Les mots de passe de tous les services exposés

### 4. Surveillez les Logs

Consultez régulièrement les logs de NGINX Ingress :
```bash
microk8s kubectl logs -n ingress -l app.kubernetes.io/name=ingress-nginx -f
```

Cherchez les comportements suspects (scans de ports, attaques, etc.).

### 5. Utilisez HTTPS (TLS/SSL)

Ne laissez JAMAIS des applications sensibles en HTTP seulement. Configurez HTTPS avec Let's Encrypt (chapitre 11).

### 6. Fail2ban (Optionnel)

Pour bloquer automatiquement les IPs malveillantes :
```bash
sudo apt install fail2ban
```

Configuration de base fournie par défaut, bloque les tentatives de brute force.

## Alternatives à la Redirection de Ports

Si vous ne pouvez pas ou ne voulez pas configurer la redirection de ports :

### Option 1 : Cloudflare Tunnel (Recommandé)

**Principe** : Un tunnel sécurisé entre votre serveur et Cloudflare, sans ouvrir de ports.

**Avantages** :
- ✅ Pas besoin de redirection de ports
- ✅ Pas besoin d'IP publique
- ✅ Fonctionne derrière CGNAT
- ✅ Protection DDoS incluse
- ✅ Gratuit

**Comment** :
1. Créez un compte Cloudflare
2. Ajoutez votre domaine
3. Installez `cloudflared` sur votre serveur
4. Créez un tunnel

Documentation : https://developers.cloudflare.com/cloudflare-one/connections/connect-apps/

### Option 2 : VPN Réversible (Reverse VPN)

**Services** :
- **ngrok** : Simple mais payant pour domaines personnalisés
- **localtunnel** : Gratuit mais pas fiable pour production
- **Tailscale Funnel** : VPN mesh avec exposition publique

**Principe** : Votre serveur se connecte au service, qui expose vos applications.

### Option 3 : VPS comme Reverse Proxy

**Architecture** :
```
Internet → VPS Public → VPN → Serveur Local (MicroK8s)
```

**Étapes** :
1. Louez un petit VPS (5€/mois)
2. Configurez WireGuard/OpenVPN entre VPS et serveur local
3. Le VPS fait reverse proxy vers votre serveur via VPN

**Avantages** :
- IP publique fixe
- Fonctionne même derrière CGNAT
- Contrôle total

**Inconvénient** : Coût mensuel

### Option 4 : IPv6 (Si Disponible)

Si votre FAI fournit de l'IPv6 :
- Chaque appareil peut avoir une IPv6 publique
- Pas de NAT nécessaire
- Configuration DNS en AAAA

**Problème** : IPv6 pas encore universel, nombreux utilisateurs n'y ont pas accès.

## Tests de Connectivité

### Script de Test Complet

Créez un fichier `test-connectivity.sh` :

```bash
#!/bin/bash

DOMAIN="mon-site.com"  # Remplacez par votre domaine
PUBLIC_IP=$(curl -s ifconfig.me)

echo "======================================"
echo "Test de connectivité Ingress"
echo "======================================"
echo ""

echo "1. IP publique : $PUBLIC_IP"
echo ""

echo "2. Résolution DNS :"
dig $DOMAIN +short
echo ""

echo "3. Test port 80 (HTTP) :"
curl -I -m 5 http://$DOMAIN 2>&1 | head -5
echo ""

echo "4. Test port 443 (HTTPS) :"
curl -I -k -m 5 https://$DOMAIN 2>&1 | head -5
echo ""

echo "5. Test depuis l'extérieur (testeur en ligne) :"
echo "   https://www.yougetsignal.com/tools/open-ports/"
echo "   IP: $PUBLIC_IP"
echo "   Ports à tester: 80, 443"
echo ""

echo "======================================"
echo "Si tout fonctionne, vous devriez voir :"
echo "- DNS résout vers $PUBLIC_IP"
echo "- HTTP et HTTPS répondent (même erreur 404)"
echo "- Ports 80 et 443 ouverts en ligne"
echo "======================================"
```

Exécutez :
```bash
chmod +x test-connectivity.sh
./test-connectivity.sh
```

## Checklist Finale

Avant de passer à la suite, vérifiez que :

- [ ] Votre serveur MicroK8s a une IP privée fixe (DHCP réservation ou statique)
- [ ] La redirection du port 80 est configurée dans la box
- [ ] La redirection du port 443 est configurée dans la box
- [ ] Les règles pointent vers la bonne IP interne
- [ ] Les règles sont activées/enabled
- [ ] Le test depuis l'intérieur fonctionne (`curl http://192.168.1.100`)
- [ ] Le test depuis l'extérieur fonctionne (4G, testeur en ligne)
- [ ] Les pods Ingress sont running (`kubectl get pods -n ingress`)
- [ ] Votre DNS pointe vers votre IP publique
- [ ] Vous avez noté votre configuration pour référence future

## Points Clés à Retenir

🔑 **NAT bloque les connexions entrantes** par défaut pour des raisons de sécurité

🔑 **Redirection de ports** : dirige le trafic externe vers votre serveur interne

🔑 **Deux ports essentiels** : 80 (HTTP) et 443 (HTTPS)

🔑 **IP fixe recommandée** : évite les problèmes si l'IP change

🔑 **Testez depuis l'extérieur** : 4G, VPS, ou testeurs en ligne

🔑 **CGNAT peut bloquer** : vérifiez si votre IP publique est vraiment publique

🔑 **Alternatives existent** : Cloudflare Tunnel, VPN, VPS reverse proxy

🔑 **Sécurité importante** : n'ouvrez que le nécessaire, firewall obligatoire

## Tableau Récapitulatif des Ports

| Port | Protocole | Usage | Priorité | Exposition |
|------|-----------|-------|----------|------------|
| **80** | TCP | HTTP | ⭐⭐⭐ Essentiel | Internet |
| **443** | TCP | HTTPS | ⭐⭐⭐ Essentiel | Internet |
| 22 | TCP | SSH | ❌ Ne PAS exposer | VPN seulement |
| 6443 | TCP | Kubernetes API | ❌ Ne PAS exposer | Interne |
| 3306 | TCP | MySQL | ❌ Ne PAS exposer | Interne |

## Prochaines Étapes

Maintenant que vos ports sont redirigés et que votre serveur est accessible depuis Internet, vous devez :

1. **Configurer le firewall** (section 10.7) pour sécuriser votre serveur
2. **Créer des règles Ingress** (sections 10.8 et 10.9) pour router le trafic
3. **Ajouter des certificats SSL** (chapitre 11) pour HTTPS sécurisé

Votre serveur MicroK8s est maintenant exposé au monde ! La prochaine étape cruciale est de le sécuriser correctement avec un firewall.

---

**📚 Résumé du chapitre** : La redirection de ports (port forwarding) est nécessaire pour rendre un serveur derrière un routeur/box accessible depuis Internet. Configurez deux règles NAT sur votre box : port 80 → serveur:80 et port 443 → serveur:443. Assurez-vous que votre serveur a une IP privée fixe. Testez depuis l'extérieur (4G, testeurs en ligne). Des alternatives existent comme Cloudflare Tunnel si la redirection n'est pas possible.

⏭️ [Firewall et règles de sécurité](/10-ingress-et-routage/07-firewall-et-regles-de-securite.md)
