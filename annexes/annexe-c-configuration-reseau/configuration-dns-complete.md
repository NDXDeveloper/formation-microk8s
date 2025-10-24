🔝 Retour au [Sommaire](/SOMMAIRE.md)

# Configuration DNS Complète pour MicroK8s

## Introduction

Le DNS (Domain Name System) est un élément fondamental pour faire fonctionner correctement un cluster Kubernetes. Dans le contexte de MicroK8s, il existe deux niveaux de DNS à comprendre et configurer :

1. **DNS interne** : utilisé par les pods pour communiquer entre eux au sein du cluster
2. **DNS externe** : utilisé pour accéder à vos applications depuis Internet ou votre réseau local

Ce guide vous accompagnera pas à pas dans la compréhension et la configuration de ces deux aspects.

---

## Partie 1 : DNS Interne Kubernetes (CoreDNS)

### Qu'est-ce que le DNS interne ?

Lorsque vous déployez plusieurs applications dans votre cluster Kubernetes, elles ont besoin de communiquer entre elles. Au lieu d'utiliser des adresses IP (qui peuvent changer), Kubernetes permet aux applications de se référencer par des noms, exactement comme vous utilisez "google.com" au lieu de "142.250.185.78".

### CoreDNS : Le serveur DNS de Kubernetes

CoreDNS est le serveur DNS intégré à Kubernetes qui traduit les noms de services en adresses IP. C'est comme un annuaire téléphonique pour votre cluster.

### Activation de CoreDNS dans MicroK8s

CoreDNS est un addon essentiel de MicroK8s. Pour l'activer :

```bash
microk8s enable dns
```

Cette commande va :
- Installer CoreDNS dans le namespace `kube-system`
- Configurer automatiquement tous les pods pour utiliser ce serveur DNS
- Permettre la résolution de noms au sein du cluster

### Vérifier que CoreDNS fonctionne

Après activation, vérifiez que CoreDNS est bien en cours d'exécution :

```bash
microk8s kubectl get pods -n kube-system -l k8s-app=kube-dns
```

Vous devriez voir un ou plusieurs pods CoreDNS avec le statut `Running`.

### Comment fonctionne la résolution DNS interne ?

Dans Kubernetes, chaque Service reçoit automatiquement un nom DNS suivant ce format :

```
<nom-du-service>.<namespace>.svc.cluster.local
```

**Exemple concret :**

Si vous créez un service appelé `backend` dans le namespace `production`, il sera accessible via :
- `backend` (depuis le même namespace)
- `backend.production` (depuis un autre namespace)
- `backend.production.svc.cluster.local` (nom complet - FQDN)

### Format des noms DNS internes

Kubernetes utilise une convention de nommage hiérarchique :

| Niveau | Description | Exemple |
|--------|-------------|---------|
| Service | Nom du service | `api` |
| Namespace | Espace de noms | `default` |
| Type | Toujours "svc" pour service | `svc` |
| Domaine | Domaine du cluster | `cluster.local` |

**Nom complet :** `api.default.svc.cluster.local`

### Configuration avancée de CoreDNS

La configuration de CoreDNS se trouve dans une ConfigMap. Pour la consulter :

```bash
microk8s kubectl get configmap coredns -n kube-system -o yaml
```

#### Structure de la configuration

La configuration utilise un format appelé "Corefile". Voici un exemple commenté :

```
.:53 {
    errors                    # Active les logs d'erreurs
    health {                  # Endpoint de santé
       lameduck 5s
    }
    ready                     # Endpoint de disponibilité
    kubernetes cluster.local in-addr.arpa ip6.arpa {  # Zone DNS Kubernetes
       pods insecure
       fallthrough in-addr.arpa ip6.arpa
       ttl 30
    }
    prometheus :9153          # Métriques Prometheus
    forward . /etc/resolv.conf {  # Redirection vers DNS externe
       max_concurrent 1000
    }
    cache 30                  # Cache DNS pendant 30 secondes
    loop                      # Détection de boucles
    reload                    # Rechargement automatique
    loadbalance               # Load balancing des réponses
}
```

#### Personnalisation courante : ajouter des domaines personnalisés

Si vous voulez que CoreDNS résolve des domaines internes spécifiques de votre entreprise :

```bash
microk8s kubectl edit configmap coredns -n kube-system
```

Ajoutez une section comme celle-ci :

```
monentreprise.local:53 {
    errors
    cache 30
    forward . 10.0.0.1        # IP de votre DNS d'entreprise
}
```

Après modification, CoreDNS se rechargera automatiquement.

### Résolution DNS pour les Pods

Chaque pod reçoit automatiquement un fichier `/etc/resolv.conf` configuré pour utiliser CoreDNS :

```
nameserver 10.152.183.10      # IP du service CoreDNS
search default.svc.cluster.local svc.cluster.local cluster.local
options ndots:5
```

**Explication :**
- `nameserver` : l'adresse IP du service CoreDNS
- `search` : domaines de recherche automatique (permet d'utiliser des noms courts)
- `ndots:5` : si le nom a moins de 5 points, essayer d'abord les domaines de recherche

### Politiques DNS des Pods

Vous pouvez personnaliser le comportement DNS d'un pod avec le champ `dnsPolicy` :

| Politique | Description | Cas d'usage |
|-----------|-------------|-------------|
| `ClusterFirst` | Utilise CoreDNS (par défaut) | Usage standard |
| `Default` | Hérite du DNS du nœud | Besoin d'accès direct au DNS de l'hôte |
| `None` | Pas de configuration automatique | Configuration DNS entièrement personnalisée |
| `ClusterFirstWithHostNet` | Pour pods avec `hostNetwork: true` | Pods réseau hôte nécessitant CoreDNS |

**Exemple de configuration personnalisée :**

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: custom-dns-pod
spec:
  dnsPolicy: "None"
  dnsConfig:
    nameservers:
      - 8.8.8.8
      - 8.8.4.4
    searches:
      - mondomaine.local
    options:
      - name: ndots
        value: "2"
  containers:
  - name: app
    image: nginx
```

### Dépannage du DNS interne

#### Tester la résolution DNS depuis un pod

Créez un pod de test temporaire :

```bash
microk8s kubectl run dnstest --image=busybox:1.28 --rm -it --restart=Never -- sh
```

À l'intérieur du pod, testez la résolution :

```bash
# Tester un service
nslookup kubernetes.default

# Tester un domaine externe
nslookup google.com

# Vérifier le fichier resolv.conf
cat /etc/resolv.conf
```

#### Problèmes courants et solutions

**Problème : Les pods ne peuvent pas résoudre les noms de services**

Vérifications :
1. CoreDNS est-il actif ?
   ```bash
   microk8s kubectl get pods -n kube-system -l k8s-app=kube-dns
   ```

2. Le service CoreDNS existe-t-il ?
   ```bash
   microk8s kubectl get svc -n kube-system kube-dns
   ```

3. Y a-t-il des erreurs dans les logs CoreDNS ?
   ```bash
   microk8s kubectl logs -n kube-system -l k8s-app=kube-dns
   ```

**Problème : Résolution lente**

Solutions :
- Augmenter la taille du cache dans la ConfigMap CoreDNS (passer de `cache 30` à `cache 300`)
- Augmenter le nombre de réplicas CoreDNS pour la charge
- Vérifier les métriques de performance via Prometheus

**Problème : Les domaines externes ne sont pas résolus**

Vérification :
```bash
# Depuis un pod de test
nslookup google.com
```

Si ça ne fonctionne pas, vérifiez la section `forward` dans la ConfigMap CoreDNS. Elle doit pointer vers un DNS fonctionnel.

---

## Partie 2 : DNS Externe - Accéder à vos applications

### Vue d'ensemble

Pour que vos applications dans MicroK8s soient accessibles depuis l'extérieur du cluster via des noms de domaine (comme `monapp.mondomaine.com`), vous devez configurer plusieurs éléments :

1. **Nom de domaine** : acheter ou configurer un domaine
2. **Enregistrements DNS** : pointer le domaine vers votre cluster
3. **Ingress Controller** : router les requêtes vers les bonnes applications
4. **Réseau** : configurer votre routeur/firewall

### Scénarios d'utilisation

#### Scénario 1 : Lab local privé (pas d'accès Internet)

Vous voulez accéder à vos applications uniquement depuis votre réseau local.

**Solution :** Utiliser un fichier `/etc/hosts` local ou un serveur DNS local (comme Pi-hole, dnsmasq).

#### Scénario 2 : Accès depuis Internet

Vous voulez que vos applications soient accessibles depuis Internet avec un vrai nom de domaine.

**Solution :** Acheter un domaine et configurer des enregistrements DNS publics.

#### Scénario 3 : Accès hybride

Certaines applications accessibles depuis Internet, d'autres uniquement en local.

**Solution :** Combiner DNS public et Split-DNS (DNS interne différent du DNS externe).

---

## Configuration DNS pour Lab Local (Sans accès Internet)

### Méthode 1 : Fichier /etc/hosts

La solution la plus simple pour un ordinateur unique.

**Sur Linux/macOS :**

```bash
sudo nano /etc/hosts
```

**Sur Windows :**

Ouvrir en tant qu'administrateur :
```
C:\Windows\System32\drivers\etc\hosts
```

**Ajouter vos entrées :**

```
192.168.1.100  monapp.local
192.168.1.100  api.local
192.168.1.100  dashboard.local
```

Remplacez `192.168.1.100` par l'adresse IP de votre machine MicroK8s.

**Avantages :** Simple et immédiat
**Inconvénients :** Doit être répété sur chaque machine, pas de wildcards

### Méthode 2 : Serveur DNS local (dnsmasq)

Pour un réseau local avec plusieurs machines.

**Installation de dnsmasq (sur Ubuntu) :**

```bash
sudo apt update
sudo apt install dnsmasq
```

**Configuration de base :**

Éditez `/etc/dnsmasq.conf` :

```bash
# Écouter sur toutes les interfaces
interface=eth0

# Définir le domaine local
domain=lab.local

# Serveur DNS upstream (pour Internet)
server=8.8.8.8
server=8.8.4.4

# Fichier d'hôtes personnalisés
addn-hosts=/etc/dnsmasq.hosts

# Cache DNS
cache-size=1000
```

**Créer le fichier d'hôtes :**

```bash
sudo nano /etc/dnsmasq.hosts
```

Contenu :

```
192.168.1.100  monapp.lab.local
192.168.1.100  api.lab.local
192.168.1.100  dashboard.lab.local
```

**Redémarrer dnsmasq :**

```bash
sudo systemctl restart dnsmasq
```

**Configurer les clients :**

Sur chaque machine du réseau local, configurer le serveur DNS vers l'IP de votre machine dnsmasq (192.168.1.100).

### Méthode 3 : Pi-hole (Solution complète)

Pi-hole est un serveur DNS avec interface web qui bloque aussi les publicités.

**Installation (sur Raspberry Pi ou VM Ubuntu) :**

```bash
curl -sSL https://install.pi-hole.net | bash
```

**Configuration via l'interface web :**

1. Accéder à `http://[IP-pihole]/admin`
2. Aller dans "Local DNS" → "DNS Records"
3. Ajouter vos entrées personnalisées

**Exemple d'entrées :**

| Domaine | IP |
|---------|-----|
| `monapp.lab.local` | `192.168.1.100` |
| `*.lab.local` (wildcard) | `192.168.1.100` |

**Configurer les clients :**

Dans les paramètres réseau de chaque machine, définir le serveur DNS principal vers l'IP du Pi-hole.

---

## Configuration DNS pour Accès Internet

### Étape 1 : Acquisition d'un nom de domaine

#### Registrars recommandés

- **Namecheap** : bon rapport qualité/prix
- **Gandi** : respectueux de la vie privée
- **OVH** : français, bon support
- **Google Domains** / **Cloudflare Registrar** : interface simple

#### Choix du domaine

Pour un lab/test, privilégiez :
- Extensions peu chères : `.xyz`, `.site`, `.online`
- Sous-domaine d'un domaine existant : `lab.mondomaine.com`

### Étape 2 : Configuration des enregistrements DNS

Une fois le domaine acheté, accédez à la console de gestion DNS de votre registrar.

#### Types d'enregistrements DNS

| Type | Usage | Exemple |
|------|-------|---------|
| **A** | Pointe vers une adresse IPv4 | `monapp.com` → `203.0.113.50` |
| **AAAA** | Pointe vers une adresse IPv6 | `monapp.com` → `2001:db8::1` |
| **CNAME** | Alias vers un autre domaine | `www.monapp.com` → `monapp.com` |
| **TXT** | Informations textuelles | Vérification domaine, SPF, DKIM |
| **MX** | Serveurs de messagerie | Configuration email |
| **NS** | Serveurs de noms | Délégation DNS |

#### Configuration de base

Supposons que votre IP publique soit `203.0.113.50`.

**Enregistrements à créer :**

```
Type    Nom                TTL     Valeur
A       @                  3600    203.0.113.50
A       www                3600    203.0.113.50
A       api                3600    203.0.113.50
A       app1               3600    203.0.113.50
A       app2               3600    203.0.113.50
```

- `@` représente le domaine racine (exemple.com)
- `www`, `api`, etc. sont des sous-domaines
- TTL (Time To Live) en secondes : 3600 = 1 heure

#### Configuration avec Wildcard

Pour simplifier, utilisez un enregistrement wildcard :

```
Type    Nom                TTL     Valeur
A       @                  3600    203.0.113.50
A       *                  3600    203.0.113.50
```

Le wildcard `*` signifie que **tous** les sous-domaines (api.exemple.com, app.exemple.com, n-importe-quoi.exemple.com) pointent vers la même IP.

**Avantage :** Vous pouvez créer autant d'Ingress que vous voulez sans toucher au DNS.

### Étape 3 : IP dynamique - Utiliser DynamicDNS (DDNS)

Si votre fournisseur d'accès Internet change régulièrement votre IP publique, vous avez besoin d'un service DDNS.

#### Services DDNS populaires

- **DuckDNS** : gratuit et simple
- **No-IP** : gratuit avec limitations
- **Dynu** : gratuit
- **OVH DynHost** : si domaine chez OVH

#### Configuration avec DuckDNS

**1. Créer un compte sur duckdns.org**

**2. Obtenir un sous-domaine gratuit :**
   - Exemple : `monlab.duckdns.org`

**3. Récupérer votre token API**

**4. Installer un client DDNS sur votre machine MicroK8s :**

```bash
# Créer le script de mise à jour
sudo nano /usr/local/bin/update-duckdns.sh
```

Contenu du script :

```bash
#!/bin/bash
echo url="https://www.duckdns.org/update?domains=monlab&token=VOTRE-TOKEN&ip=" | curl -k -o /var/log/duckdns.log -K -
```

**5. Rendre le script exécutable :**

```bash
sudo chmod +x /usr/local/bin/update-duckdns.sh
```

**6. Configurer une tâche cron pour mise à jour automatique :**

```bash
crontab -e
```

Ajouter :

```
*/5 * * * * /usr/local/bin/update-duckdns.sh
```

Cela met à jour l'IP toutes les 5 minutes.

#### Alternative : Utiliser un conteneur DDNS

Déployer un conteneur dans MicroK8s qui gère DDNS :

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: ddns-updater
spec:
  replicas: 1
  selector:
    matchLabels:
      app: ddns
  template:
    metadata:
      labels:
        app: ddns
    spec:
      containers:
      - name: ddns-updater
        image: qmcgaw/ddns-updater
        env:
        - name: PERIOD
          value: "5m"
        - name: CONFIG
          value: |
            {
              "settings": [
                {
                  "provider": "duckdns",
                  "domain": "monlab.duckdns.org",
                  "token": "VOTRE-TOKEN"
                }
              ]
            }
```

### Étape 4 : Vérification de la configuration DNS

#### Outils en ligne de commande

**Sous Linux/macOS :**

```bash
# Test de résolution
dig monapp.exemple.com

# Version simplifiée
nslookup monapp.exemple.com

# Avec un serveur DNS spécifique
dig @8.8.8.8 monapp.exemple.com
```

**Sous Windows :**

```cmd
nslookup monapp.exemple.com
```

#### Outils web

- **WhatsMyDNS** (whatsmydns.net) : vérifier la propagation mondiale
- **DNSChecker** (dnschecker.org) : propagation DNS
- **MXToolbox** : diagnostic DNS complet

#### Temps de propagation

Après modification des enregistrements DNS :
- **Local** : quelques minutes
- **Mondial** : 24 à 48 heures (selon le TTL)

Pour accélérer localement :
```bash
# Vider le cache DNS local (Linux)
sudo systemd-resolve --flush-caches

# macOS
sudo dscacheutil -flushcache

# Windows (en admin)
ipconfig /flushdns
```

---

## Intégration DNS avec Ingress

### Principe

Le flux complet :
1. Un utilisateur tape `monapp.exemple.com` dans son navigateur
2. Le DNS résout ce nom vers votre IP publique (ex: 203.0.113.50)
3. La requête arrive sur votre routeur/firewall
4. Le routeur redirige le port 80/443 vers votre machine MicroK8s
5. L'Ingress Controller reçoit la requête
6. L'Ingress Controller lit le header `Host: monapp.exemple.com`
7. L'Ingress Controller route vers le bon Service selon la configuration

### Configuration Ingress avec nom de domaine

**Exemple d'Ingress simple :**

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: mon-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  ingressClassName: public
  rules:
  - host: monapp.exemple.com
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

**Avec plusieurs applications :**

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: multi-app-ingress
spec:
  ingressClassName: public
  rules:
  - host: app1.exemple.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: app1-service
            port:
              number: 80
  - host: app2.exemple.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: app2-service
            port:
              number: 80
  - host: api.exemple.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: api-service
            port:
              number: 8080
```

### Routage par chemin (path-based routing)

Vous pouvez aussi router en fonction du chemin d'URL :

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: path-based-ingress
spec:
  ingressClassName: public
  rules:
  - host: exemple.com
    http:
      paths:
      - path: /app1
        pathType: Prefix
        backend:
          service:
            name: app1-service
            port:
              number: 80
      - path: /app2
        pathType: Prefix
        backend:
          service:
            name: app2-service
            port:
              number: 80
      - path: /api
        pathType: Prefix
        backend:
          service:
            name: api-service
            port:
              number: 8080
```

Accès :
- `exemple.com/app1` → app1-service
- `exemple.com/app2` → app2-service
- `exemple.com/api` → api-service

---

## Redirection de Ports et NAT

### Configuration du routeur/box Internet

Pour que les requêtes Internet atteignent votre cluster MicroK8s, vous devez configurer la redirection de ports (Port Forwarding).

#### Ports à rediriger

| Port | Protocole | Service | Destination |
|------|-----------|---------|-------------|
| 80 | TCP | HTTP | IP_MICROK8S:80 |
| 443 | TCP | HTTPS | IP_MICROK8S:443 |

#### Procédure générale

1. **Accéder à l'interface de votre box/routeur**
   - Généralement via `192.168.1.1` ou `192.168.0.1`
   - Identifiants : consulter la documentation de votre FAI

2. **Trouver la section "Redirection de ports" ou "NAT"**
   - Noms possibles : Port Forwarding, NAT, Virtual Server

3. **Créer les règles :**

**Règle 1 - HTTP :**
- Port externe : 80
- Port interne : 80
- Protocole : TCP
- IP destination : 192.168.1.100 (IP de votre machine MicroK8s)

**Règle 2 - HTTPS :**
- Port externe : 443
- Port interne : 443
- Protocole : TCP
- IP destination : 192.168.1.100

4. **Sauvegarder et redémarrer le routeur si nécessaire**

#### IP statique locale

Pour éviter que l'IP interne de votre machine MicroK8s change, configurez une IP statique (ou réservation DHCP).

**Méthode 1 : Réservation DHCP sur le routeur**
- Associer l'adresse MAC de votre machine MicroK8s à une IP fixe

**Méthode 2 : IP statique sur la machine**

Sur Ubuntu, éditez `/etc/netplan/00-installer-config.yaml` :

```yaml
network:
  version: 2
  ethernets:
    eth0:
      addresses:
        - 192.168.1.100/24
      gateway4: 192.168.1.1
      nameservers:
        addresses:
          - 8.8.8.8
          - 8.8.4.4
```

Appliquer :

```bash
sudo netplan apply
```

### Configuration Firewall

#### UFW (Uncomplicated Firewall - Ubuntu)

Autoriser les ports HTTP et HTTPS :

```bash
# Installer UFW si nécessaire
sudo apt install ufw

# Autoriser SSH (important pour ne pas se bloquer !)
sudo ufw allow 22/tcp

# Autoriser HTTP et HTTPS
sudo ufw allow 80/tcp
sudo ufw allow 443/tcp

# Activer le firewall
sudo ufw enable

# Vérifier le statut
sudo ufw status
```

#### Firewalld (CentOS/RHEL)

```bash
# Autoriser HTTP et HTTPS
sudo firewall-cmd --permanent --add-service=http
sudo firewall-cmd --permanent --add-service=https

# Recharger la configuration
sudo firewall-cmd --reload

# Vérifier
sudo firewall-cmd --list-all
```

#### Firewall Windows (WSL2)

Si vous utilisez MicroK8s dans WSL2 sur Windows :

```powershell
# En tant qu'administrateur PowerShell
New-NetFirewallRule -DisplayName "MicroK8s HTTP" -Direction Inbound -Protocol TCP -LocalPort 80 -Action Allow
New-NetFirewallRule -DisplayName "MicroK8s HTTPS" -Direction Inbound -Protocol TCP -LocalPort 443 -Action Allow
```

---

## DNS pour environnements spécifiques

### Multi-node : DNS et Load Balancing

Avec un cluster MicroK8s multi-node, vous avez plusieurs options DNS :

#### Option 1 : Round-Robin DNS

Créer plusieurs enregistrements A pointant vers différents nœuds :

```
A    exemple.com    203.0.113.50    (nœud 1)
A    exemple.com    203.0.113.51    (nœud 2)
A    exemple.com    203.0.113.52    (nœud 3)
```

Le DNS retournera les adresses dans un ordre différent à chaque requête (load balancing basique).

**Inconvénient :** Pas de détection de panne automatique.

#### Option 2 : MetalLB avec IP virtuelle

Utiliser MetalLB pour obtenir une seule IP virtuelle partagée :

```bash
microk8s enable metallb:203.0.113.100-203.0.113.100
```

Pointer le DNS vers cette IP unique :

```
A    exemple.com    203.0.113.100
```

MetalLB gère automatiquement le basculement entre nœuds.

#### Option 3 : Load Balancer externe (HAProxy, Nginx)

Déployer un load balancer externe devant vos nœuds :

```
                    ┌─────────┐
    DNS ──────────> │ HAProxy │
                    └─────────┘
                         │
            ┌────────────┼────────────┐
            │            │            │
         Node 1       Node 2       Node 3
     (MicroK8s)   (MicroK8s)   (MicroK8s)
```

Le DNS pointe vers le load balancer, qui distribue vers les nœuds.

### Split-DNS : Accès différent interne vs externe

Configuration où :
- Depuis Internet → accès via DNS public
- Depuis réseau local → accès via DNS local (plus rapide, pas de sortie/entrée Internet)

**Exemple d'architecture :**

```
┌─────────────────────────────────────────────┐
│ DNS Public (exemple.com)                    │
│ A    exemple.com    203.0.113.50 (IP pub)   │
└─────────────────────────────────────────────┘
                  │
                  │ Internet
                  │
        ┌─────────▼─────────┐
        │   Routeur/NAT     │
        └─────────┬─────────┘
                  │
                  │ Réseau local
                  │
┌─────────────────▼─────────────────────────┐
│ DNS Local (dnsmasq/Pi-hole)               │
│ A    exemple.com    192.168.1.100         │
└───────────────────────────────────────────┘
```

**Configuration dnsmasq pour split-DNS :**

```
# /etc/dnsmasq.conf
address=/exemple.com/192.168.1.100
address=/app1.exemple.com/192.168.1.100
address=/app2.exemple.com/192.168.1.100
```

Les machines du réseau local utilisent dnsmasq comme DNS et obtiennent l'IP locale directement.

---

## Dépannage DNS Externe

### Problème : Le domaine ne résout pas

**Vérifications :**

1. **Propagation DNS :**
   - Vérifier sur whatsmydns.net
   - Attendre jusqu'à 48h pour propagation mondiale

2. **Enregistrements corrects :**
   ```bash
   dig exemple.com
   # Vérifier que l'IP retournée est la bonne
   ```

3. **Cache DNS local :**
   - Vider le cache (voir commandes plus haut)

4. **Serveurs DNS utilisés :**
   ```bash
   # Voir quel DNS votre machine utilise
   cat /etc/resolv.conf

   # Tester avec un DNS spécifique
   dig @8.8.8.8 exemple.com
   dig @1.1.1.1 exemple.com
   ```

### Problème : Le domaine résout mais la page ne charge pas

**Vérifications :**

1. **Redirection de ports :**
   - Vérifier que les ports 80/443 sont bien redirigés
   - Tester depuis l'extérieur avec un téléphone en 4G

2. **Firewall :**
   ```bash
   sudo ufw status
   # Vérifier que 80/443 sont autorisés
   ```

3. **Ingress Controller actif :**
   ```bash
   microk8s kubectl get pods -n ingress
   ```

4. **Service LoadBalancer :**
   ```bash
   microk8s kubectl get svc -n ingress
   # Vérifier qu'il y a une EXTERNAL-IP
   ```

5. **Test depuis l'intérieur du réseau :**
   ```bash
   curl -H "Host: exemple.com" http://192.168.1.100
   ```

### Problème : HTTPS ne fonctionne pas

**Vérifications :**

1. **Certificat installé :**
   ```bash
   microk8s kubectl get certificates
   microk8s kubectl describe certificate mon-cert
   ```

2. **Port 443 redirigé :**
   - Vérifier la configuration du routeur

3. **Test du certificat :**
   ```bash
   openssl s_client -connect exemple.com:443 -servername exemple.com
   ```

### Outil de diagnostic complet

Script de diagnostic DNS/réseau :

```bash
#!/bin/bash
DOMAIN="exemple.com"
LOCAL_IP="192.168.1.100"

echo "=== Test de résolution DNS ==="
dig $DOMAIN

echo -e "\n=== Test depuis différents serveurs DNS ==="
dig @8.8.8.8 $DOMAIN +short
dig @1.1.1.1 $DOMAIN +short

echo -e "\n=== Test de connectivité HTTP ==="
curl -I http://$DOMAIN

echo -e "\n=== Test de connectivité HTTPS ==="
curl -I https://$DOMAIN

echo -e "\n=== Test local ==="
curl -I http://$LOCAL_IP

echo -e "\n=== Vérification firewall ==="
sudo ufw status

echo -e "\n=== Vérification Ingress ==="
microk8s kubectl get ingress --all-namespaces

echo -e "\n=== Vérification Services LoadBalancer ==="
microk8s kubectl get svc --all-namespaces | grep LoadBalancer
```

---

## Bonnes Pratiques DNS

### Sécurité

1. **Ne jamais exposer CoreDNS à Internet**
   - CoreDNS doit rester interne au cluster

2. **Utiliser DNSSEC si possible**
   - Active l'authentification des réponses DNS
   - Vérifier le support chez votre registrar

3. **Limiter les enregistrements wildcard en production**
   - Préférer des enregistrements explicites pour la sécurité

4. **Monitoring des requêtes DNS**
   - Activer les logs CoreDNS
   - Surveiller les requêtes anormales

### Performance

1. **Utiliser des TTL appropriés**
   - Développement : 300 secondes (5 min)
   - Production stable : 3600 secondes (1h)
   - Migration : 60 secondes (1 min)

2. **Augmenter le cache CoreDNS**
   ```
   cache 300
   ```

3. **Optimiser le nombre de réplicas CoreDNS**
   ```bash
   microk8s kubectl scale deployment coredns -n kube-system --replicas=3
   ```

4. **Utiliser des DNS rapides pour le forwarding**
   - Google (8.8.8.8, 8.8.4.4)
   - Cloudflare (1.1.1.1, 1.0.0.1)
   - Quad9 (9.9.9.9)

### Organisation

1. **Convention de nommage claire**
   ```
   [environnement]-[application]-[fonction].[domaine]
   prod-api-users.exemple.com
   dev-frontend-dashboard.exemple.com
   ```

2. **Documentation**
   - Maintenir un fichier listant tous les sous-domaines utilisés
   - Documenter les redirections spéciales

3. **Automatisation**
   - Utiliser External-DNS pour synchronisation automatique
   - Infrastructure as Code (Terraform) pour les enregistrements DNS

### Monitoring

1. **Métriques à surveiller :**
   - Temps de réponse DNS
   - Taux d'échec de résolution
   - Taux de cache hit/miss

2. **Alertes recommandées :**
   - CoreDNS pods non disponibles
   - Augmentation anormale des requêtes DNS
   - Échecs de résolution > 1%

---

## Résumé et Checklist

### Checklist DNS Interne

- [ ] CoreDNS activé (`microk8s enable dns`)
- [ ] CoreDNS pods en Running
- [ ] Service kube-dns accessible
- [ ] Test de résolution depuis un pod réussi
- [ ] Configuration CoreDNS personnalisée si nécessaire
- [ ] Cache DNS configuré (300 secondes recommandé)

### Checklist DNS Externe (Lab local)

- [ ] Fichier `/etc/hosts` configuré OU
- [ ] Serveur DNS local (dnsmasq/Pi-hole) installé et configuré
- [ ] Tous les clients pointent vers le bon serveur DNS
- [ ] Tests de résolution réussis depuis les clients

### Checklist DNS Externe (Internet)

- [ ] Nom de domaine acheté ou sous-domaine DDNS créé
- [ ] Enregistrements DNS configurés (A ou wildcard)
- [ ] Propagation DNS vérifiée (whatsmydns.net)
- [ ] IP publique connue (ipify.org)
- [ ] DDNS configuré si IP dynamique
- [ ] Redirection de ports configurée (80, 443)
- [ ] Firewall configuré
- [ ] Tests depuis l'extérieur réussis
- [ ] Ingress configuré avec les bons hostnames

### Outils Recommandés

| Outil | Usage | Commande |
|-------|-------|----------|
| **dig** | Résolution DNS détaillée | `dig exemple.com` |
| **nslookup** | Résolution DNS simple | `nslookup exemple.com` |
| **curl** | Test HTTP/HTTPS | `curl -I https://exemple.com` |
| **whatsmydns.net** | Propagation mondiale | Via navigateur web |
| **dnschecker.org** | Vérification multi-DNS | Via navigateur web |

---

## Conclusion

La configuration DNS est un élément fondamental pour utiliser MicroK8s efficacement, que ce soit pour un lab local ou pour exposer des services sur Internet.

**Points clés à retenir :**

1. **DNS interne (CoreDNS)** : automatique avec MicroK8s, essentiel pour la communication inter-pods
2. **DNS externe local** : simple avec `/etc/hosts` ou dnsmasq pour un lab privé
3. **DNS externe Internet** : nécessite un domaine, des enregistrements DNS, et une redirection de ports
4. **Ingress** : le lien entre le DNS et vos applications dans Kubernetes
5. **Monitoring** : surveillez les performances et la disponibilité DNS

Avec une configuration DNS correcte, vos applications MicroK8s deviennent facilement accessibles et professionnelles, que ce soit pour l'apprentissage, le développement ou même de petites applications en production.

N'hésitez pas à commencer simple (fichier hosts) et à évoluer progressivement vers des solutions plus complexes selon vos besoins !

⏭️ [Configuration firewall](/annexes/annexe-c-configuration-reseau/configuration-firewall.md)
