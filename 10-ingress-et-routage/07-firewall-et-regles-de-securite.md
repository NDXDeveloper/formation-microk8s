🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 10.7 Firewall et règles de sécurité

## Introduction

Votre serveur MicroK8s est maintenant accessible depuis Internet grâce à la redirection de ports. C'est fantastique... mais aussi **potentiellement dangereux** ! Chaque seconde, des robots et des attaquants scannent Internet à la recherche de serveurs vulnérables.

Un **firewall** (pare-feu) est votre **première ligne de défense**. C'est comme la porte blindée et le système de sécurité de votre maison : il contrôle qui peut entrer et par quelle porte.

Dans ce chapitre, nous allons configurer un firewall robuste pour protéger votre serveur tout en permettant à vos applications Kubernetes de fonctionner correctement.

## Pourquoi un Firewall est Essentiel

### Le Danger d'un Serveur Exposé sans Protection

Dès que votre serveur est accessible depuis Internet, **en quelques minutes**, vous verrez des tentatives de connexion :

```
# Logs typiques d'un serveur non protégé
[2025-10-24 10:15:22] SSH login attempt from 103.45.67.89 - Failed
[2025-10-24 10:15:34] Port scan detected from 185.220.101.12
[2025-10-24 10:15:56] SSH brute force from 198.51.100.45 - 50 attempts
[2025-10-24 10:16:12] Exploit attempt on port 8080 from 203.0.113.78
```

**Sans firewall** :
- ❌ Tous les ports du serveur sont accessibles
- ❌ Services internes exposés (base de données, API Kubernetes, SSH)
- ❌ Vulnérable aux attaques automatisées
- ❌ Risque de compromission en quelques heures

**Avec firewall** :
- ✅ Seulement les ports voulus sont ouverts (80, 443)
- ✅ Services internes protégés
- ✅ Attaques bloquées automatiquement
- ✅ Logs de tentatives d'intrusion

### Analogie de la Maison

Imaginez votre serveur comme une maison :

**Sans firewall** :
- Toutes les portes et fenêtres sont ouvertes
- N'importe qui peut essayer d'entrer partout
- Les pièces privées (chambres = services internes) sont accessibles

**Avec firewall** :
- Seule la porte d'entrée principale est accessible (ports 80/443)
- Toutes les autres entrées sont verrouillées
- Un système d'alarme détecte les tentatives d'intrusion

## Les Différents Types de Firewalls

Sur Linux, il existe plusieurs solutions de firewall. Voici les principales :

### 1. UFW (Uncomplicated Firewall) ⭐ Recommandé

**Avantages** :
- ✅ Très simple à utiliser
- ✅ Commandes intuitives en anglais simple
- ✅ Configuration par défaut sécurisée
- ✅ Parfait pour les débutants
- ✅ Frontend pour iptables

**Inconvénient** :
- Moins de contrôle granulaire (suffisant pour 95% des cas)

**C'est celui que nous utiliserons dans ce tutoriel.**

### 2. iptables

**Avantages** :
- Contrôle total et granulaire
- Très puissant et flexible
- Inclus dans tous les noyaux Linux

**Inconvénients** :
- ❌ Syntaxe complexe
- ❌ Difficile pour les débutants
- ❌ Facile de faire des erreurs

**Quand l'utiliser** : Pour des configurations très avancées.

### 3. firewalld

**Avantages** :
- Interface moderne
- Gestion par zones
- Utilisé par défaut sur RHEL/CentOS

**Inconvénient** :
- Plus complexe qu'UFW
- Moins universel sur Ubuntu/Debian

### 4. nftables

**Avantages** :
- Remplaçant moderne d'iptables
- Syntaxe plus claire
- Meilleure performance

**Inconvénient** :
- Encore peu adopté
- Documentation limitée pour débutants

## Installation d'UFW

### Vérifier si UFW est Installé

Sur Ubuntu, UFW est généralement préinstallé :

```bash
sudo ufw version
```

Si installé, vous verrez :
```
ufw 0.36.2
```

### Installer UFW (si Nécessaire)

**Sur Ubuntu/Debian** :
```bash
sudo apt update
sudo apt install ufw
```

**Sur CentOS/RHEL** :
```bash
sudo yum install epel-release
sudo yum install ufw
```

### Vérifier le Statut

```bash
sudo ufw status
```

Par défaut, UFW est **inactif** après installation :
```
Status: inactive
```

## Principes de Configuration

### Stratégie "Deny All, Allow Specific"

La meilleure approche de sécurité :

1. **Bloquer tout** par défaut (deny all)
2. **Autoriser seulement** ce qui est nécessaire (allow specific)

C'est comme une boîte de nuit exclusive : personne n'entre sauf ceux sur la liste.

### Politique par Défaut d'UFW

UFW applique cette stratégie automatiquement :
- **Entrant (incoming)** : DENY (tout bloqué par défaut)
- **Sortant (outgoing)** : ALLOW (tout autorisé par défaut)
- **Transfert (forwarding)** : DENY (bloqué par défaut)

**Cela signifie** :
- Votre serveur peut se connecter à Internet (apt update, wget, etc.)
- Internet ne peut PAS se connecter à votre serveur (sauf règles explicites)
- Parfait pour commencer !

## Configuration de Base : Étape par Étape

### ⚠️ Avertissement Critique pour SSH

**ATTENTION** : Si vous gérez votre serveur via SSH et que vous activez UFW sans autoriser le port SSH, **vous serez bloqué et ne pourrez plus vous connecter** !

**Solution** : Toujours autoriser SSH AVANT d'activer UFW.

### Étape 1 : Autoriser SSH (Port 22)

Si vous vous connectez à distance via SSH :

```bash
sudo ufw allow 22/tcp
```

Ou plus simplement (UFW connaît les services standards) :

```bash
sudo ufw allow ssh
```

**Message de confirmation** :
```
Rules updated
Rules updated (v6)
```

**Note** : Si vous utilisez un port SSH non standard (ex: 2222), utilisez ce port :
```bash
sudo ufw allow 2222/tcp
```

### Étape 2 : Autoriser HTTP (Port 80)

Pour le trafic web non chiffré et la validation Let's Encrypt :

```bash
sudo ufw allow 80/tcp
```

Ou :
```bash
sudo ufw allow http
```

### Étape 3 : Autoriser HTTPS (Port 443)

Pour le trafic web chiffré :

```bash
sudo ufw allow 443/tcp
```

Ou :
```bash
sudo ufw allow https
```

### Étape 4 : Vérifier les Règles Avant Activation

**Important** : Vérifiez que vos règles sont correctes AVANT d'activer le firewall.

```bash
sudo ufw show added
```

Vous devriez voir :
```
Added user rules (see 'ufw status' for running firewall):
ufw allow 22/tcp
ufw allow 80/tcp
ufw allow 443/tcp
```

### Étape 5 : Activer UFW

Maintenant, activez le firewall :

```bash
sudo ufw enable
```

**Message d'avertissement** :
```
Command may disrupt existing ssh connections. Proceed with operation (y|n)?
```

Tapez `y` et validez.

**Confirmation** :
```
Firewall is active and enabled on system startup
```

Le firewall est maintenant actif et **démarrera automatiquement au reboot**.

### Étape 6 : Vérifier le Statut

```bash
sudo ufw status
```

Sortie attendue :
```
Status: active

To                         Action      From
--                         ------      ----
22/tcp                     ALLOW       Anywhere
80/tcp                     ALLOW       Anywhere
443/tcp                    ALLOW       Anywhere
22/tcp (v6)                ALLOW       Anywhere (v6)
80/tcp (v6)                ALLOW       Anywhere (v6)
443/tcp (v6)                ALLOW       Anywhere (v6)
```

**Explication** :
- Les ports 22, 80, 443 sont ouverts
- Les versions IPv4 et IPv6 sont gérées
- Tout le reste est bloqué par défaut

### Étape 7 : Vérifier le Statut Verbeux

Pour voir plus de détails :

```bash
sudo ufw status verbose
```

Sortie :
```
Status: active
Logging: on (low)
Default: deny (incoming), allow (outgoing), disabled (routed)
New profiles: skip

To                         Action      From
--                         ------      ----
22/tcp                     ALLOW IN    Anywhere
80/tcp                     ALLOW IN    Anywhere
443/tcp                    ALLOW IN    Anywhere
...
```

Parfait ! Votre firewall est maintenant actif et configuré.

## Configuration pour MicroK8s

### Ports Utilisés par MicroK8s

MicroK8s utilise plusieurs ports en interne. Voici les principaux :

| Port | Service | Exposition | Action |
|------|---------|------------|--------|
| **80** | Ingress HTTP | Internet | ✅ Autoriser |
| **443** | Ingress HTTPS | Internet | ✅ Autoriser |
| 16443 | API Kubernetes | Interne | ❌ Bloquer |
| 10250 | Kubelet | Interne | ❌ Bloquer |
| 10255 | Read-only Kubelet | Interne | ❌ Bloquer |
| 25000 | Cluster agent | Interne | ❌ Bloquer |
| 12379 | Etcd/Dqlite | Interne | ❌ Bloquer |
| 10257 | kube-controller | Interne | ❌ Bloquer |
| 10259 | kube-scheduler | Interne | ❌ Bloquer |

**Bonne nouvelle** : Avec la configuration de base (seulement 80 et 443 ouverts), tous les ports internes sont déjà protégés !

### Communication Interne MicroK8s

Les pods et services Kubernetes communiquent en interne via :
- Interface `cni0` (réseau des pods)
- Interface `vxlan.calico` (si Calico est utilisé)
- Interface `lo` (localhost)

**UFW ne bloque PAS ces communications internes** par défaut. Vos applications Kubernetes fonctionneront normalement.

### Cluster Multi-Nœuds (Si Applicable)

Si vous avez un cluster multi-nœuds, vous devez autoriser la communication entre nœuds.

**Option 1** : Autoriser l'IP spécifique de chaque nœud

```bash
# Autoriser le nœud 2 depuis le nœud 1
sudo ufw allow from 192.168.1.101 to any port 16443
sudo ufw allow from 192.168.1.101 to any port 25000
sudo ufw allow from 192.168.1.101 to any port 12379
```

**Option 2** : Autoriser le sous-réseau complet

```bash
# Si tous vos nœuds sont dans 192.168.1.0/24
sudo ufw allow from 192.168.1.0/24
```

**Note** : Pour un cluster mono-nœud (cas le plus courant en lab), cela n'est pas nécessaire.

## Règles Avancées

### Autoriser un Port depuis une IP Spécifique

Pour autoriser SSH seulement depuis votre IP de bureau :

```bash
sudo ufw allow from 203.0.113.50 to any port 22
```

**Encore mieux** : autoriser seulement depuis un sous-réseau :

```bash
# Depuis votre réseau local
sudo ufw allow from 192.168.1.0/24 to any port 22
```

### Bloquer une IP Spécifique

Si vous détectez une IP malveillante :

```bash
sudo ufw deny from 198.51.100.45
```

Ou bloquer un sous-réseau complet :

```bash
sudo ufw deny from 198.51.100.0/24
```

### Limiter les Tentatives de Connexion (Rate Limiting)

Pour protéger SSH contre le brute force :

```bash
sudo ufw limit ssh
```

**Fonctionnement** :
- Si une IP fait plus de 6 tentatives de connexion en 30 secondes
- Elle sera bloquée temporairement

**Très recommandé** pour SSH !

Remplacez la règle SSH existante :
```bash
# Supprimer l'ancienne règle
sudo ufw delete allow 22/tcp

# Ajouter la règle avec limite
sudo ufw limit 22/tcp
```

### Autoriser un Service Spécifique

UFW connaît les services courants. Vous pouvez les autoriser par nom :

```bash
sudo ufw allow ftp
sudo ufw allow smtp
sudo ufw allow mysql
```

Pour voir la liste des services connus :
```bash
sudo less /etc/services
```

### Règles avec Plage de Ports

Pour autoriser une plage de ports :

```bash
sudo ufw allow 8000:8100/tcp
```

Cela autorise les ports TCP de 8000 à 8100.

### Règles pour UDP

Par défaut, les exemples utilisent TCP. Pour UDP :

```bash
sudo ufw allow 53/udp    # DNS
sudo ufw allow 123/udp   # NTP
```

## Gestion des Règles

### Lister Toutes les Règles avec Numéros

Pour voir les règles numérotées :

```bash
sudo ufw status numbered
```

Sortie :
```
Status: active

     To                         Action      From
     --                         ------      ----
[ 1] 22/tcp                     ALLOW IN    Anywhere
[ 2] 80/tcp                     ALLOW IN    Anywhere
[ 3] 443/tcp                    ALLOW IN    Anywhere
[ 4] 22/tcp (v6)                ALLOW IN    Anywhere (v6)
[ 5] 80/tcp (v6)                ALLOW IN    Anywhere (v6)
[ 6] 443/tcp (v6)                ALLOW IN    Anywhere (v6)
```

### Supprimer une Règle

**Méthode 1** : Par numéro (plus simple)

```bash
sudo ufw delete 3
```

Cela supprime la règle numéro 3.

**Méthode 2** : Par commande originale

```bash
sudo ufw delete allow 80/tcp
```

### Insérer une Règle à une Position Spécifique

Les règles sont évaluées dans l'ordre. Pour insérer une règle en première position :

```bash
sudo ufw insert 1 deny from 198.51.100.45
```

Cela bloque l'IP 198.51.100.45 avant toute autre règle.

### Réinitialiser UFW

Pour supprimer toutes les règles et repartir de zéro :

```bash
sudo ufw reset
```

**Attention** : Cela supprime TOUTES vos règles ! Vous devrez tout reconfigurer.

## Logging et Surveillance

### Activer les Logs

UFW peut enregistrer toutes les connexions bloquées :

```bash
sudo ufw logging on
```

Niveaux disponibles :
- `off` : Pas de logs
- `low` : Logs des paquets bloqués (défaut)
- `medium` : Logs + nouveaux paquets autorisés
- `high` : Logs + détails paquets avec limite de taux
- `full` : Tous les logs (très verbeux)

**Recommandé pour débuter** : `low` ou `medium`

```bash
sudo ufw logging medium
```

### Consulter les Logs

Les logs UFW sont écrits dans le syslog :

```bash
sudo tail -f /var/log/ufw.log
```

Ou dans le journal système :

```bash
sudo journalctl -u ufw -f
```

### Analyser les Tentatives d'Intrusion

Pour voir les tentatives de connexion bloquées :

```bash
sudo grep "UFW BLOCK" /var/log/ufw.log
```

Exemple de ligne de log :
```
Oct 24 10:15:22 server kernel: [UFW BLOCK] IN=eth0 OUT= MAC=... SRC=198.51.100.45 DST=203.0.113.1 PROTO=TCP DPT=22
```

**Lecture** :
- `SRC=198.51.100.45` : IP source (attaquant)
- `DST=203.0.113.1` : IP destination (votre serveur)
- `PROTO=TCP` : Protocole
- `DPT=22` : Port destination (SSH)

### Surveiller en Temps Réel

Pour voir les blocages en temps réel :

```bash
sudo tail -f /var/log/ufw.log | grep BLOCK
```

Vous verrez défiler les tentatives d'intrusion bloquées.

## Sécurité Renforcée

### 1. Désactiver IPv6 (Si Non Utilisé)

Si vous n'utilisez pas IPv6 :

Éditez `/etc/default/ufw` :
```bash
sudo nano /etc/default/ufw
```

Changez :
```
IPV6=yes
```

En :
```
IPV6=no
```

Rechargez UFW :
```bash
sudo ufw reload
```

### 2. Politique de Sortie Restrictive (Avancé)

Par défaut, tout le trafic sortant est autorisé. Pour plus de sécurité :

```bash
sudo ufw default deny outgoing
```

**Attention** : Vous devrez ensuite autoriser explicitement chaque service sortant :

```bash
sudo ufw allow out 53        # DNS
sudo ufw allow out 80        # HTTP
sudo ufw allow out 443       # HTTPS
sudo ufw allow out 123       # NTP
```

**Recommandé seulement** pour les environnements très sécurisés.

### 3. Protection Contre les Scans de Ports

Ajoutez des règles dans `/etc/ufw/before.rules` pour bloquer les scans :

```bash
sudo nano /etc/ufw/before.rules
```

Ajoutez avant la ligne `*filter` :

```
# Bloquer les scans de ports
-A ufw-before-input -p tcp --tcp-flags ALL NONE -j DROP
-A ufw-before-input -p tcp ! --syn -m state --state NEW -j DROP
-A ufw-before-input -p tcp --tcp-flags ALL ALL -j DROP
```

Rechargez :
```bash
sudo ufw reload
```

### 4. Limiter les Nouvelles Connexions

Pour protéger contre les attaques DDoS simples, limitez les connexions HTTP/HTTPS :

```bash
sudo ufw limit 80/tcp
sudo ufw limit 443/tcp
```

**Note** : Cela peut affecter les performances pour les sites à fort trafic légitime.

## Configuration Recommandée pour MicroK8s

Voici la configuration complète recommandée :

```bash
# 1. Réinitialiser (si UFW déjà configuré)
sudo ufw --force reset

# 2. Définir les politiques par défaut
sudo ufw default deny incoming
sudo ufw default allow outgoing

# 3. Autoriser SSH avec rate limiting
sudo ufw limit 22/tcp comment 'SSH with rate limiting'

# 4. Autoriser HTTP et HTTPS pour Ingress
sudo ufw allow 80/tcp comment 'Ingress HTTP'
sudo ufw allow 443/tcp comment 'Ingress HTTPS'

# 5. Activer le logging (niveau medium)
sudo ufw logging medium

# 6. Activer UFW
sudo ufw --force enable

# 7. Vérifier le statut
sudo ufw status verbose
```

**Résultat final** :
```
Status: active
Logging: on (medium)
Default: deny (incoming), allow (outgoing), disabled (routed)

To                         Action      From
--                         ------      ----
22/tcp                     LIMIT IN    Anywhere                   # SSH with rate limiting
80/tcp                     ALLOW IN    Anywhere                   # Ingress HTTP
443/tcp                    ALLOW IN    Anywhere                   # Ingress HTTPS
```

## Dépannage

### Problème 1 : Plus d'Accès SSH

**Symptôme** : Vous ne pouvez plus vous connecter en SSH.

**Cause** : SSH n'a pas été autorisé avant d'activer UFW.

**Solution** :
1. Si vous avez un accès physique ou console :
   ```bash
   sudo ufw allow 22/tcp
   ```

2. Sinon, désactivez temporairement UFW :
   ```bash
   sudo ufw disable
   ```
   Puis ajoutez la règle SSH et réactivez.

### Problème 2 : Ingress Ne Répond Plus

**Symptôme** : Vos applications ne sont plus accessibles via HTTP/HTTPS.

**Cause** : Ports 80/443 bloqués.

**Solution** :
```bash
sudo ufw status | grep -E '80|443'
```

Si les ports ne sont pas listés :
```bash
sudo ufw allow 80/tcp
sudo ufw allow 443/tcp
```

### Problème 3 : MicroK8s Ne Fonctionne Plus

**Symptôme** : Erreurs de communication entre composants MicroK8s.

**Cause** : Interfaces réseau ou règles trop restrictives.

**Solution** :
Ajoutez une règle pour autoriser le trafic sur l'interface loopback :
```bash
sudo ufw allow in on lo
sudo ufw allow out on lo
```

### Problème 4 : UFW Ne Démarre Pas au Boot

**Symptôme** : Le firewall est inactif après un redémarrage.

**Solution** :
```bash
sudo systemctl enable ufw
sudo systemctl start ufw
```

### Problème 5 : Règles Non Appliquées

**Symptôme** : Vous ajoutez des règles mais elles ne fonctionnent pas.

**Solution** :
1. Vérifiez que UFW est actif :
   ```bash
   sudo ufw status
   ```

2. Si inactif, activez-le :
   ```bash
   sudo ufw enable
   ```

3. Rechargez les règles :
   ```bash
   sudo ufw reload
   ```

## Test de Sécurité

### Scanner Votre Serveur depuis l'Extérieur

Utilisez des outils en ligne pour scanner votre serveur :

1. **Nmap Online** : https://pentest-tools.com/network-vulnerability-scanning/tcp-port-scanner-online-nmap
   - Entrez votre IP publique
   - Scannez les ports courants
   - Seuls 80, 443 (et éventuellement 22) devraient être ouverts

2. **YouGetSignal** : https://www.yougetsignal.com/tools/open-ports/
   - Testez individuellement les ports

3. **Depuis un VPS ou serveur distant** :
   ```bash
   nmap -p- votre-ip-publique
   ```

**Résultat attendu** :
```
PORT    STATE    SERVICE
22/tcp  open     ssh
80/tcp  open     http
443/tcp open     https
All other ports: filtered
```

### Vérifier les Règles Actives

```bash
sudo iptables -L -n -v
```

Cela montre les règles iptables sous-jacentes utilisées par UFW.

## Outils Complémentaires de Sécurité

### 1. Fail2ban

**Fail2ban** analyse les logs et bannit automatiquement les IPs suspectes.

**Installation** :
```bash
sudo apt install fail2ban
```

**Configuration de base** :
```bash
sudo cp /etc/fail2ban/jail.conf /etc/fail2ban/jail.local
sudo nano /etc/fail2ban/jail.local
```

Activez la protection SSH :
```ini
[sshd]
enabled = true
port = ssh
filter = sshd
logpath = /var/log/auth.log
maxretry = 3
bantime = 3600
```

Démarrez :
```bash
sudo systemctl enable fail2ban
sudo systemctl start fail2ban
```

Fail2ban bannira automatiquement les IPs après 3 tentatives SSH échouées.

### 2. PSAD (Port Scan Attack Detector)

Détecte les scans de ports et les attaques :

```bash
sudo apt install psad
```

PSAD analyse les logs UFW et peut bannir automatiquement les IPs qui scannent votre serveur.

### 3. ClamAV (Antivirus)

Pour scanner les fichiers et détecter les malwares :

```bash
sudo apt install clamav clamav-daemon
sudo freshclam  # Mise à jour des signatures
```

Scan manuel :
```bash
sudo clamscan -r /home
```

## Bonnes Pratiques de Sécurité

### 1. Principe du Moindre Privilège

N'ouvrez **que** les ports strictement nécessaires. Si un service n'a pas besoin d'être public, ne l'exposez pas.

### 2. Surveillance Régulière

Consultez les logs régulièrement :
```bash
# Une fois par semaine
sudo grep "UFW BLOCK" /var/log/ufw.log | tail -100
```

### 3. Mise à Jour du Firewall

Après chaque changement de service ou d'application :
- Revoyez vos règles UFW
- Supprimez les règles obsolètes
- Testez que tout fonctionne

### 4. Documentation

Documentez vos règles dans un fichier :
```bash
# /root/firewall-rules.txt
# Port 22: SSH (limité contre brute force)
# Port 80: Ingress HTTP
# Port 443: Ingress HTTPS
```

### 5. Backup de la Configuration

Sauvegardez votre configuration UFW :
```bash
sudo cp /etc/ufw/user.rules /root/ufw-backup-$(date +%Y%m%d).rules
```

### 6. Tests Après Modification

Après chaque modification :
1. Testez l'accès SSH
2. Testez l'accès HTTP/HTTPS
3. Vérifiez les logs

### 7. Plan B

Gardez toujours un accès de secours :
- Console physique
- IPMI/iLO/iDRAC
- VNC via le panneau de contrôle de l'hébergeur

## Commandes de Référence Rapide

```bash
# Installation
sudo apt install ufw

# Configuration de base
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw limit 22/tcp
sudo ufw allow 80/tcp
sudo ufw allow 443/tcp

# Gestion
sudo ufw enable               # Activer
sudo ufw disable              # Désactiver
sudo ufw reload               # Recharger
sudo ufw reset                # Réinitialiser

# Consultation
sudo ufw status               # Statut simple
sudo ufw status verbose       # Statut détaillé
sudo ufw status numbered      # Avec numéros de règles
sudo ufw show added           # Règles ajoutées

# Ajout de règles
sudo ufw allow 8080/tcp       # Autoriser un port
sudo ufw deny 3306/tcp        # Bloquer un port
sudo ufw limit ssh            # Limite de taux
sudo ufw allow from 192.168.1.0/24  # Autoriser un sous-réseau

# Suppression de règles
sudo ufw delete 3             # Supprimer par numéro
sudo ufw delete allow 80/tcp  # Supprimer par commande

# Logs
sudo ufw logging on           # Activer les logs
sudo ufw logging medium       # Niveau medium
sudo tail -f /var/log/ufw.log # Voir les logs

# Debug
sudo iptables -L -n -v        # Voir les règles iptables
sudo ufw show raw             # Configuration brute
```

## Checklist de Sécurité Firewall

Avant de passer à la suite, vérifiez que :

- [ ] UFW est installé et actif (`sudo ufw status`)
- [ ] SSH est autorisé (avec rate limiting si possible)
- [ ] Ports 80 et 443 sont ouverts pour Ingress
- [ ] Tous les autres ports sont bloqués par défaut
- [ ] Les logs sont activés (niveau medium)
- [ ] Vous pouvez vous connecter en SSH
- [ ] Vous pouvez accéder à vos applications web
- [ ] Vous avez testé depuis l'extérieur (scan de ports)
- [ ] Votre configuration est documentée
- [ ] Vous avez un backup de la configuration

## Points Clés à Retenir

🔑 **Firewall = première ligne de défense** contre les attaques

🔑 **UFW = simple et efficace** pour la majorité des besoins

🔑 **Stratégie "deny all, allow specific"** : bloquer par défaut, autoriser le nécessaire

🔑 **Trois ports essentiels** : 22 (SSH), 80 (HTTP), 443 (HTTPS)

🔑 **Rate limiting sur SSH** : protection contre le brute force

🔑 **Toujours autoriser SSH avant d'activer** le firewall !

🔑 **Logs activés** : pour surveiller les tentatives d'intrusion

🔑 **Tests réguliers** : scanner votre serveur depuis l'extérieur

## Prochaines Étapes

Maintenant que votre serveur est sécurisé avec un firewall, vous êtes prêt à :

1. **Créer des règles de routage Ingress** (section 10.8) pour diriger le trafic vers vos applications
2. **Configurer le routage par chemin** (section 10.9) pour des URLs plus complexes
3. **Ajouter des certificats SSL/TLS** (chapitre 11) pour sécuriser les communications HTTPS

Votre serveur MicroK8s est maintenant accessible depuis Internet ET protégé. La combinaison redirection de ports + firewall + HTTPS (prochain chapitre) offre une sécurité solide pour vos applications !

---

**📚 Résumé du chapitre** : Un firewall est essentiel pour protéger un serveur exposé sur Internet. UFW est simple et efficace : bloquez tout par défaut (deny incoming), autorisez uniquement SSH (avec rate limiting), HTTP (80) et HTTPS (443). Activez les logs pour surveiller les tentatives d'intrusion. Testez régulièrement votre configuration avec des scans de ports. Le firewall est votre première ligne de défense contre les attaques automatisées.

⏭️ [Règles de routage par nom d'hôte](/10-ingress-et-routage/08-regles-de-routage-par-nom-dhote.md)
