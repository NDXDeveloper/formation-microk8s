🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 2.5 Installation sur Windows (WSL2)

## Introduction

Vous êtes sur Windows et voulez utiliser MicroK8s ? Excellente nouvelle : grâce à **WSL2** (Windows Subsystem for Linux 2), vous pouvez faire tourner un véritable Linux à l'intérieur de Windows avec d'excellentes performances !

Cette section vous guidera à travers l'installation complète de WSL2, puis de MicroK8s. À la fin, vous aurez un cluster Kubernetes fonctionnel directement accessible depuis Windows, sans dual-boot ni machine virtuelle compliquée.

## Comprendre WSL2

### Qu'est-ce que WSL2 ?

**WSL2** est une technologie Microsoft qui permet d'exécuter un **véritable noyau Linux** directement dans Windows. C'est comme avoir Linux et Windows qui tournent simultanément sur la même machine.

**Analogie simple :**
Imaginez que votre ordinateur est une maison avec deux appartements :
- **Appartement Windows** : Votre système Windows habituel
- **Appartement Linux** : Un véritable système Linux
- **Couloir commun** : Vous pouvez facilement passer de l'un à l'autre

Les deux systèmes cohabitent, partagent les ressources (CPU, RAM, réseau) et peuvent communiquer entre eux.

### WSL1 vs WSL2 : Pourquoi WSL2 ?

| Aspect | WSL1 | WSL2 |
|--------|------|------|
| **Noyau Linux** | Émulé | Véritable noyau Linux |
| **Performances** | Moyennes | Excellentes (90-95% du natif) |
| **Compatibilité** | Limitée | Complète |
| **Docker/Kubernetes** | ❌ Non compatible | ✅ Compatible |
| **Appels système** | Traduits | Natifs |

**Pour MicroK8s : WSL2 est obligatoire !** WSL1 ne fonctionnera pas.

### Avantages de WSL2 pour MicroK8s

✅ **Pas besoin de dual-boot** : Windows et Linux simultanément
✅ **Performances excellentes** : Proche du Linux natif
✅ **Intégration profonde** : Accès aux fichiers Windows depuis Linux et vice-versa
✅ **Réseau partagé** : Les applications sont accessibles depuis Windows
✅ **Pas de VM lourde** : Plus léger que VirtualBox ou VMware
✅ **Développement fluide** : Outils Windows + environnement Linux

## Prérequis Windows

### Version Windows requise

**Windows 10 :**
- Version **2004** (Build **19041**) ou supérieure
- Toutes les éditions : Home, Pro, Enterprise, Education

**Windows 11 :**
- Toutes les versions ✅

### Vérifier votre version Windows

**Méthode 1 : Commande winver**
1. Appuyez sur `Windows + R`
2. Tapez : `winver`
3. Appuyez sur Entrée

Vous verrez une fenêtre avec :
```
Version 22H2 (Build OS 19045.xxxx)
```

**Le Build doit être ≥ 19041** pour WSL2.

**Méthode 2 : PowerShell**
```powershell
# Ouvrir PowerShell et taper :
[System.Environment]::OSVersion.Version
```

### Configuration matérielle requise

**Obligatoire :**
- Processeur 64 bits
- Virtualisation activée dans le BIOS (Intel VT-x ou AMD-V)
- 4 Go RAM minimum (8 Go recommandé)

**Recommandé :**
- 8 Go RAM ou plus
- SSD pour meilleures performances
- 50 Go d'espace disque libre

### Vérifier la virtualisation

**Dans le Gestionnaire des tâches :**
1. Ouvrir le Gestionnaire des tâches (`Ctrl + Shift + Esc`)
2. Onglet "Performance"
3. Cliquer sur "CPU"
4. Regarder "Virtualisation" en bas

**Statut attendu :** "Activé" ✅

**Si "Désactivé" :**
Vous devez l'activer dans le BIOS (voir section troubleshooting).

## Étape 1 : Activer WSL et la plateforme de machine virtuelle

Nous allons activer les fonctionnalités Windows nécessaires pour WSL2.

### Option A : Installation automatique (Windows 11 et Windows 10 récent)

**C'est la méthode la plus simple !**

1. **Ouvrir PowerShell en tant qu'administrateur**
   - Clic droit sur le menu Démarrer
   - Cliquer sur "Terminal (Admin)" ou "Windows PowerShell (Admin)"
   - Ou chercher "PowerShell" dans le menu, clic droit → "Exécuter en tant qu'administrateur"

2. **Installer WSL avec une commande**
   ```powershell
   wsl --install
   ```

**Ce qui se passe :**
Cette commande fait TOUT automatiquement :
- Active WSL
- Active la plateforme de machine virtuelle
- Installe WSL2
- Télécharge et installe Ubuntu (distribution par défaut)

**Sortie attendue :**
```
Installation en cours...
La fonctionnalité demandée a été activée avec succès.
Une modification du système nécessite un redémarrage.
```

3. **Redémarrer l'ordinateur**
   ```powershell
   Restart-Computer
   ```

4. **Après le redémarrage**
   - Ubuntu se lancera automatiquement
   - On vous demandera de créer un utilisateur Linux
   - Passez à l'Étape 2 (Configuration Ubuntu)

### Option B : Installation manuelle (Windows 10 ancien)

Si `wsl --install` ne fonctionne pas, utilisez cette méthode :

**1. Activer WSL**
```powershell
# Dans PowerShell Admin
dism.exe /online /enable-feature /featurename:Microsoft-Windows-Subsystem-Linux /all /norestart
```

**2. Activer la plateforme de machine virtuelle**
```powershell
dism.exe /online /enable-feature /featurename:VirtualMachinePlatform /all /norestart
```

**3. Redémarrer l'ordinateur**
```powershell
Restart-Computer
```

**4. Télécharger et installer le package de mise à jour WSL2**
- Visitez : https://aka.ms/wsl2kernel
- Téléchargez le fichier `.msi`
- Exécutez l'installateur
- Suivez les instructions

**5. Définir WSL2 comme version par défaut**
```powershell
wsl --set-default-version 2
```

## Étape 2 : Installer Ubuntu dans WSL2

### Via Microsoft Store (Méthode recommandée)

**Si vous avez utilisé `wsl --install`, Ubuntu est déjà installé !** Passez directement à "Configuration initiale d'Ubuntu".

**Sinon :**

1. **Ouvrir Microsoft Store**
   - Chercher "Microsoft Store" dans le menu Démarrer
   - Ou appuyer sur `Windows + S` et chercher "Store"

2. **Chercher Ubuntu**
   - Dans la barre de recherche : "Ubuntu"
   - Plusieurs versions disponibles :
     - **Ubuntu** (dernière version, recommandé)
     - Ubuntu 22.04 LTS
     - Ubuntu 24.04 LTS

3. **Installer Ubuntu**
   - Cliquer sur la distribution choisie
   - Cliquer sur "Obtenir" ou "Installer"
   - Attendre le téléchargement (~500 Mo)

4. **Lancer Ubuntu**
   - Cliquer sur "Ouvrir" dans le Store
   - Ou chercher "Ubuntu" dans le menu Démarrer

### Configuration initiale d'Ubuntu

La première fois que vous lancez Ubuntu, vous devez le configurer :

**1. Patienter pendant l'installation**
```
Installing, this may take a few minutes...
```
Attendez 1-2 minutes.

**2. Créer votre compte utilisateur Linux**
```
Enter new UNIX username:
```
- Tapez votre nom d'utilisateur (lettres minuscules, pas d'espaces)
- Exemple : `john`, `marie`, `admin`

**3. Définir votre mot de passe**
```
New password:
```
- Tapez un mot de passe
- **Important :** En Linux, rien ne s'affiche quand vous tapez le mot de passe (c'est normal !)
- Retapez le même mot de passe pour confirmer

**4. Confirmation**
```
Installation successful!
```

Vous voyez maintenant le terminal Ubuntu :
```
username@computername:~$
```

**Félicitations, vous êtes dans Linux ! 🎉**

### Vérifier que vous êtes en WSL2

**Dans PowerShell Windows :**
```powershell
wsl --list --verbose
```

**Sortie attendue :**
```
  NAME            STATE           VERSION
* Ubuntu          Running         2
```

**VERSION doit être "2"** ✅

**Si VERSION est "1" :**
```powershell
# Convertir en WSL2
wsl --set-version Ubuntu 2
```

## Étape 3 : Mettre à jour Ubuntu

Dans le terminal Ubuntu, mettez à jour le système :

```bash
# Mettre à jour la liste des paquets
sudo apt update

# Mettre à jour les paquets installés
sudo apt upgrade -y
```

**Note :** `sudo` vous demandera votre mot de passe Linux (celui créé à l'étape 2).

**Durée :** 2-5 minutes selon le nombre de mises à jour.

## Étape 4 : Installer MicroK8s dans Ubuntu (WSL2)

Maintenant que Ubuntu est prêt, installons MicroK8s !

### Installation de MicroK8s

```bash
# Installer MicroK8s
sudo snap install microk8s --classic
```

**Sortie attendue :**
```
microk8s (1.28/stable) v1.28.3 from Canonical✓ installed
```

**Durée :** 2-5 minutes.

### Ajouter votre utilisateur au groupe MicroK8s

```bash
# Ajouter votre utilisateur au groupe
sudo usermod -a -G microk8s $USER

# Changer les permissions
mkdir -p ~/.kube
sudo chown -R $USER ~/.kube

# Appliquer les changements
su - $USER
```

**Note :** La commande `su - $USER` vous demandera votre mot de passe.

### Vérifier l'installation

```bash
# Vérifier le statut
microk8s status --wait-ready
```

**Sortie attendue :**
```
microk8s is running
```

✅ Si vous voyez "microk8s is running", c'est bon !

```bash
# Vérifier le nœud
microk8s kubectl get nodes
```

**Sortie attendue :**
```
NAME                STATUS   ROLES    AGE   VERSION
computername        Ready    <none>   30s   v1.28.3
```

**STATUS = Ready** → Parfait ! ✅

## Étape 5 : Configuration pour WSL2

WSL2 a quelques spécificités. Configurons MicroK8s pour qu'il fonctionne parfaitement.

### Créer l'alias kubectl

```bash
# Ajouter l'alias à .bashrc
echo "alias kubectl='microk8s kubectl'" >> ~/.bashrc
source ~/.bashrc

# Tester
kubectl get nodes
```

### Configurer la mémoire WSL2 (Important !)

Par défaut, WSL2 peut utiliser jusqu'à 50% de votre RAM totale. Limitons cela.

**Dans Windows, créer/éditer le fichier `.wslconfig` :**

1. **Ouvrir l'Explorateur de fichiers Windows**
2. **Aller dans votre dossier utilisateur**
   - `C:\Users\VotreNom\`
   - Ou tapez `%USERPROFILE%` dans la barre d'adresse

3. **Créer le fichier `.wslconfig`**
   - Clic droit → Nouveau → Document texte
   - Nommer : `.wslconfig` (avec le point au début)
   - Si Windows ne permet pas le point, utiliser Notepad et "Enregistrer sous" en mettant le nom entre guillemets : `".wslconfig"`

4. **Éditer le fichier avec Notepad**
   ```ini
   [wsl2]
   memory=4GB
   processors=2
   swap=2GB
   ```

**Adapter selon votre RAM :**
- **8 Go RAM total** : `memory=4GB`
- **16 Go RAM total** : `memory=8GB`
- **32 Go RAM total** : `memory=12GB`

5. **Redémarrer WSL2**

   Dans PowerShell :
   ```powershell
   wsl --shutdown
   ```

   Puis relancer Ubuntu depuis le menu Démarrer.

### Activer le démarrage automatique de MicroK8s (optionnel)

Par défaut, MicroK8s ne démarre pas automatiquement dans WSL2.

**Option 1 : Démarrage manuel**
```bash
# Démarrer MicroK8s manuellement quand nécessaire
microk8s start
```

**Option 2 : Script de démarrage automatique**

Créer un script qui démarre MicroK8s au lancement d'Ubuntu :

```bash
# Créer le script
cat << 'EOF' >> ~/.bashrc

# Auto-start MicroK8s si pas déjà démarré
if ! microk8s status >/dev/null 2>&1; then
    echo "Démarrage de MicroK8s..."
    microk8s start
fi
EOF

# Recharger
source ~/.bashrc
```

**Note :** Cette option consommera des ressources même si vous n'utilisez pas MicroK8s.

## Étape 6 : Accéder à MicroK8s depuis Windows

### Accéder aux services via localhost

Les services MicroK8s sont automatiquement accessibles depuis Windows via `localhost` !

**Exemple :**
```bash
# Dans Ubuntu/WSL2
kubectl create deployment nginx --image=nginx
kubectl expose deployment nginx --port=80 --type=NodePort
kubectl get service nginx
```

```
NAME    TYPE       CLUSTER-IP      PORT(S)
nginx   NodePort   10.152.183.xx   80:31234/TCP
```

**Dans Windows :**
- Ouvrir un navigateur
- Aller sur `http://localhost:31234`
- Vous voyez nginx ! ✅

### Installer kubectl sur Windows (optionnel)

Vous pouvez aussi utiliser kubectl directement depuis PowerShell Windows :

**Méthode 1 : Via Chocolatey**
```powershell
# Installer Chocolatey (si pas déjà installé)
Set-ExecutionPolicy Bypass -Scope Process -Force
[System.Net.ServicePointManager]::SecurityProtocol = [System.Net.ServicePointManager]::SecurityProtocol -bor 3072
iex ((New-Object System.Net.WebClient).DownloadString('https://community.chocolatey.org/install.ps1'))

# Installer kubectl
choco install kubernetes-cli
```

**Méthode 2 : Téléchargement manuel**
1. Télécharger : https://kubernetes.io/docs/tasks/tools/install-kubectl-windows/
2. Ajouter au PATH Windows

**Configurer kubectl Windows pour utiliser MicroK8s :**

Dans Ubuntu/WSL2 :
```bash
# Générer la config
microk8s config > /mnt/c/Users/VotreNom/.kube/config
```

Dans PowerShell Windows :
```powershell
kubectl get nodes
# Fonctionne depuis Windows ! ✅
```

## Étape 7 : Intégration Windows ↔ Linux

### Accéder aux fichiers Windows depuis Ubuntu

Les disques Windows sont montés dans `/mnt/` :

```bash
# Disque C:
cd /mnt/c/

# Dossier Users
cd /mnt/c/Users/

# Votre dossier personnel Windows
cd /mnt/c/Users/VotreNom/
```

**Exemple : Copier un fichier Windows vers Linux**
```bash
cp /mnt/c/Users/VotreNom/Documents/app.yaml ~/
```

### Accéder aux fichiers Ubuntu depuis Windows

**Dans l'Explorateur Windows, tapez :**
```
\\wsl$\Ubuntu\home\votre-user\
```

Ou :
```
\\wsl$\Ubuntu\
```

Vous pouvez créer un raccourci ou épingler ce dossier.

**Via PowerShell/Terminal :**
```powershell
# Ouvrir le dossier home Ubuntu depuis Windows
explorer.exe \\wsl$\Ubuntu\home\votre-user\
```

### Exécuter des commandes Linux depuis Windows

```powershell
# Depuis PowerShell Windows
wsl microk8s kubectl get pods

# Ou entrer dans Ubuntu
wsl
```

### Ouvrir VS Code dans WSL2

Si vous utilisez Visual Studio Code :

```bash
# Dans Ubuntu, depuis le dossier de votre projet
code .
```

VS Code s'ouvrira dans Windows mais travaillera dans l'environnement Linux ! Très pratique pour le développement.

## Test de déploiement complet

Testons que tout fonctionne avec un déploiement réel.

### Dans le terminal Ubuntu

```bash
# Créer un déploiement
kubectl create deployment hello --image=gcr.io/google-samples/hello-app:1.0

# Exposer le service
kubectl expose deployment hello --port=8080 --type=NodePort

# Récupérer le port
kubectl get service hello
```

**Sortie :**
```
NAME    TYPE       PORT(S)
hello   NodePort   8080:30123/TCP
```

### Dans Windows

**Navigateur :** `http://localhost:30123`

Vous devriez voir :
```
Hello, world!
Version: 1.0.0
Hostname: hello-xxxxx
```

✅ **Ça fonctionne !** Windows peut accéder à votre application dans Kubernetes/WSL2.

### Nettoyer

```bash
kubectl delete service hello
kubectl delete deployment hello
```

## Commandes WSL2 utiles

### Gestion de WSL2 depuis PowerShell

```powershell
# Lister les distributions installées
wsl --list --verbose

# Arrêter WSL2 (toutes les distributions)
wsl --shutdown

# Arrêter une distribution spécifique
wsl --terminate Ubuntu

# Démarrer Ubuntu
wsl -d Ubuntu

# Définir Ubuntu comme distribution par défaut
wsl --set-default Ubuntu

# Voir la version WSL
wsl --version
```

### Gestion de MicroK8s dans WSL2

```bash
# Démarrer MicroK8s
microk8s start

# Arrêter MicroK8s
microk8s stop

# Statut
microk8s status

# Redémarrer
microk8s stop && microk8s start
```

## Troubleshooting : Problèmes spécifiques à WSL2

### Problème : "Virtualisation non activée"

**Message d'erreur lors de l'installation WSL :**
```
Please enable the Virtual Machine Platform Windows feature and ensure virtualization is enabled in the BIOS.
```

**Solution : Activer la virtualisation dans le BIOS**

Les étapes varient selon le fabricant :

**Pour la plupart des PC :**
1. Redémarrer l'ordinateur
2. Appuyer sur `F2`, `F10`, `Del` ou `Esc` au démarrage (dépend du fabricant)
3. Chercher "Virtualization Technology" ou "Intel VT-x" ou "AMD-V"
4. Mettre sur "Enabled"
5. Sauvegarder et quitter (F10 généralement)

**Marques courantes :**
- **HP** : F10 → Configuration → Virtualisation
- **Dell** : F2 → Advanced → Virtualization
- **Lenovo** : F1 ou F2 → Configuration → Intel VT
- **ASUS** : F2 ou Del → Advanced → CPU Configuration

### Problème : WSL2 est lent ou consomme trop de RAM

**Solution 1 : Limiter la RAM via .wslconfig**

Créer `C:\Users\VotreNom\.wslconfig` :
```ini
[wsl2]
memory=4GB
processors=2
localhostForwarding=true
```

Puis :
```powershell
wsl --shutdown
```

**Solution 2 : Compacter le disque virtuel**

Avec le temps, le disque virtuel WSL2 grossit. Le compacter :

```powershell
# Arrêter WSL2
wsl --shutdown

# Ouvrir diskpart
diskpart

# Dans diskpart
select vdisk file="C:\Users\VotreNom\AppData\Local\Packages\CanonicalGroupLimited.Ubuntu_xxxxx\LocalState\ext4.vhdx"
compact vdisk
exit
```

### Problème : "Cannot connect to the Docker daemon"

WSL2 utilise containerd, pas Docker. C'est normal !

**Si vraiment besoin de Docker :**
```bash
# Installer Docker dans WSL2 (optionnel)
curl -fsSL https://get.docker.com -o get-docker.sh
sudo sh get-docker.sh
```

Mais pour MicroK8s, containerd suffit.

### Problème : MicroK8s ne démarre pas après redémarrage Windows

**Cause :** WSL2 ne démarre pas automatiquement.

**Solution :**
```bash
# Dans Ubuntu
microk8s start
```

Ou créer un script de démarrage automatique (voir Étape 5).

### Problème : "wsl --install" ne fonctionne pas

**Sur Windows 10 ancien :**

Utilisez la méthode manuelle (Option B de l'Étape 1) ou mettez à jour Windows :
```powershell
# Vérifier les mises à jour
Start-Process ms-settings:windowsupdate
```

### Problème : Impossible d'accéder aux services NodePort depuis Windows

**Solution 1 : Vérifier le firewall Windows**
```powershell
# Désactiver temporairement pour tester
Set-NetFirewallProfile -Profile Domain,Public,Private -Enabled False

# Réactiver après test
Set-NetFirewallProfile -Profile Domain,Public,Private -Enabled True
```

**Solution 2 : Utiliser l'IP WSL2**

Trouver l'IP de WSL2 :
```bash
# Dans Ubuntu
ip addr show eth0 | grep inet
```

Utiliser cette IP depuis Windows : `http://172.x.x.x:30xxx`

### Problème : "This application does not support WSL1"

**Cause :** Vous êtes en WSL1, pas WSL2.

**Solution :**
```powershell
# Convertir en WSL2
wsl --set-version Ubuntu 2

# Attendre la conversion (peut prendre 5-10 minutes)

# Vérifier
wsl --list --verbose
```

### Problème : WSL2 démarre très lentement

**Causes possibles :**
- Antivirus qui scanne WSL2
- Trop peu de RAM allouée
- Disque dur plein

**Solutions :**
```powershell
# Exclure WSL2 de l'antivirus Windows Defender
Add-MpPreference -ExclusionPath "C:\Users\VotreNom\AppData\Local\Packages\CanonicalGroupLimited.Ubuntu*"

# Libérer de l'espace disque
# Désinstaller programmes inutiles
# Nettoyer avec l'outil Windows
```

## Optimisations et bonnes pratiques

### 1. Utiliser Windows Terminal

**Windows Terminal** est bien meilleur que le terminal par défaut :

1. Installer depuis Microsoft Store : "Windows Terminal"
2. Configurer Ubuntu comme profil par défaut
3. Personnaliser (thèmes, raccourcis)

### 2. Intégration VS Code

```bash
# Installer l'extension WSL dans VS Code
# Puis depuis Ubuntu :
code .
```

VS Code s'ouvrira avec l'extension WSL activée automatiquement.

### 3. Utiliser Docker Desktop (optionnel)

Docker Desktop s'intègre parfaitement avec WSL2 :
- Installer Docker Desktop pour Windows
- Activer l'intégration WSL2 dans les paramètres
- Utiliser docker depuis Ubuntu/WSL2

### 4. Créer des scripts de démarrage

**Script PowerShell pour démarrer MicroK8s :**

Créer `start-microk8s.ps1` :
```powershell
# Démarrer WSL2 et MicroK8s
wsl -d Ubuntu -u root -- microk8s start
Write-Host "MicroK8s démarré !"
```

### 5. Sauvegarder votre environnement WSL2

**Exporter Ubuntu :**
```powershell
wsl --export Ubuntu C:\Backups\ubuntu-backup.tar
```

**Importer ailleurs ou restaurer :**
```powershell
wsl --import Ubuntu C:\WSL\Ubuntu C:\Backups\ubuntu-backup.tar
```

## Performances WSL2 vs Linux natif

Pour votre information :

| Aspect | Linux Natif | WSL2 |
|--------|-------------|------|
| **CPU** | 100% | 95-98% |
| **RAM** | 100% | 90-95% |
| **I/O disque** | 100% | 80-90% |
| **Réseau** | 100% | 95-98% |
| **Démarrage** | Rapide | Très rapide |

WSL2 offre d'excellentes performances, très proches du natif !

## Désinstallation

Si vous devez tout désinstaller :

### Désinstaller MicroK8s

```bash
# Dans Ubuntu
microk8s stop
sudo snap remove microk8s
```

### Désinstaller Ubuntu WSL2

```powershell
# Dans PowerShell Admin
wsl --unregister Ubuntu
```

⚠️ **Attention :** Cela supprime TOUTES les données Ubuntu !

### Désactiver WSL2

```powershell
# Désactiver WSL
dism.exe /online /disable-feature /featurename:Microsoft-Windows-Subsystem-Linux
dism.exe /online /disable-feature /featurename:VirtualMachinePlatform

# Redémarrer
Restart-Computer
```

## Checklist de vérification finale

**✅ Windows :**
- [ ] Version ≥ Build 19041 (`winver`)
- [ ] Virtualisation activée (Gestionnaire des tâches)
- [ ] WSL2 installé (`wsl --version`)

**✅ Ubuntu :**
- [ ] Ubuntu installé dans WSL2 (`wsl -l -v` → VERSION = 2)
- [ ] Système à jour (`sudo apt update && upgrade`)
- [ ] Utilisateur Linux créé

**✅ MicroK8s :**
- [ ] MicroK8s installé (`microk8s version`)
- [ ] Status running (`microk8s status`)
- [ ] Node Ready (`kubectl get nodes`)

**✅ Fonctionnement :**
- [ ] Pods système en Running (`kubectl get pods -A`)
- [ ] Déploiement test nginx réussi
- [ ] Accès depuis Windows via localhost fonctionne

**✅ Configuration :**
- [ ] .wslconfig créé pour limiter RAM
- [ ] Alias kubectl configuré
- [ ] Intégration fichiers Windows ↔ Linux fonctionne

**Si toutes les cases cochées : Installation complète et fonctionnelle ! 🎉**

## Prochaines étapes

Votre installation MicroK8s sur Windows (WSL2) est maintenant complète !

**Ce que vous pouvez faire :**
1. Activer des addons : `microk8s enable dns dashboard storage`
2. Explorer le Dashboard Kubernetes
3. Déployer vos applications
4. Continuer le tutoriel !

**Particularités WSL2 à retenir :**
- WSL2 = véritable Linux dans Windows
- Performances excellentes (90-95% du natif)
- Accès bidirectionnel fichiers Windows ↔ Linux
- Services accessibles via localhost depuis Windows
- .wslconfig pour gérer la RAM
- `wsl --shutdown` pour redémarrer proprement

**Vous avez maintenant le meilleur des deux mondes : la puissance de Kubernetes avec le confort de Windows ! 🚀**

---

**Points clés à retenir :**
- **WSL2 requis** (pas WSL1) : `wsl --set-default-version 2`
- **Build Windows ≥ 19041** pour WSL2
- **Virtualisation doit être activée** dans le BIOS
- **Installation simple** : `wsl --install` fait tout automatiquement
- **Ubuntu recommandé** comme distribution Linux
- **Limiter RAM** via .wslconfig : `memory=4GB`
- **Services accessibles** via localhost depuis Windows
- **Fichiers partagés** : `/mnt/c/` et `\\wsl$\Ubuntu\`
- **Performances excellentes** : 90-95% du natif
- **Windows Terminal** recommandé pour meilleure expérience

⏭️ [Installation sur macOS](/02-installation-et-configuration-initiale/06-installation-sur-macos.md)
