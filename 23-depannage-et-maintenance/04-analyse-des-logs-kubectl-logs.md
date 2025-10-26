🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 23.4 Analyse des Logs (kubectl logs)

## Introduction

Les logs sont la fenêtre vers l'intérieur de vos applications. Ils vous disent ce qui se passe réellement dans vos conteneurs : les requêtes reçues, les erreurs rencontrées, les opérations effectuées. Savoir lire et analyser les logs est une compétence essentielle pour tout utilisateur de Kubernetes.

> **Pour les débutants** : Les logs sont comme le journal de bord d'une application. Tout ce que votre application affiche sur la console (stdout/stderr) est capturé par Kubernetes et accessible via `kubectl logs`. C'est votre première source d'information pour comprendre pourquoi quelque chose ne fonctionne pas.

## Comprendre les Logs dans Kubernetes

### Qu'est-ce qu'un Log ?

Dans Kubernetes, un log est :
- Tout ce qui est écrit sur **stdout** (sortie standard)
- Tout ce qui est écrit sur **stderr** (sortie d'erreur)
- Par votre application qui s'exécute dans un conteneur

**Exemple concret** :

```python
# Dans votre application Python
print("Serveur démarré sur le port 8080")  # → stdout → log Kubernetes
logging.error("Impossible de se connecter à la base de données")  # → stderr → log Kubernetes
```

```javascript
// Dans votre application Node.js
console.log("Requête GET reçue");  // → stdout → log Kubernetes
console.error("Erreur: Token invalide");  // → stderr → log Kubernetes
```

### Où sont Stockés les Logs ?

Kubernetes ne stocke PAS les logs indéfiniment. Voici comment ça fonctionne :

1. **Sur le nœud** : Les logs sont d'abord écrits sur le disque du nœud dans `/var/log/pods/`
2. **Rotation** : Les logs sont automatiquement "tournés" (rotation) pour éviter de remplir le disque
3. **Durée de rétention** : Par défaut, seuls les logs récents sont conservés (généralement quelques jours)
4. **Après redémarrage** : Les logs du conteneur précédent sont conservés temporairement

**Implication importante** : Si vous avez besoin de conserver les logs longtemps, vous devez mettre en place une solution de logs centralisés (ELK, Loki, etc.). `kubectl logs` n'est que pour le diagnostic immédiat.

### Types de Logs Accessibles

Vous pouvez accéder aux logs de :
- **Pods** (conteneurs individuels)
- **Deployments** (via leurs pods)
- **StatefulSets** (via leurs pods)
- **DaemonSets** (via leurs pods)
- **Jobs** et **CronJobs** (via leurs pods)

## kubectl logs : La Commande de Base

### Syntaxe Fondamentale

```bash
microk8s kubectl logs <nom-du-pod>
```

**Exemple simple** :
```bash
microk8s kubectl logs nginx-deployment-7d9f8c-abcde
```

Cela affiche tous les logs du pod depuis son démarrage.

### Spécifier le Namespace

Si votre pod n'est pas dans le namespace `default` :

```bash
microk8s kubectl logs <nom-du-pod> -n <namespace>
```

**Exemple** :
```bash
microk8s kubectl logs prometheus-server-abc123 -n monitoring
```

### Pour un Pod Multi-Conteneurs

Si un pod contient plusieurs conteneurs, vous devez spécifier lequel :

```bash
microk8s kubectl logs <nom-du-pod> -c <nom-conteneur>
```

**Exemple** :
```bash
# Pod avec un conteneur nginx et un conteneur sidecar
microk8s kubectl logs mon-pod -c nginx
microk8s kubectl logs mon-pod -c sidecar
```

**Comment savoir les noms des conteneurs ?**
```bash
microk8s kubectl get pod mon-pod -o jsonpath='{.spec.containers[*].name}'
```

## Options Essentielles de kubectl logs

### 1. Suivre les Logs en Temps Réel (-f, --follow)

**Description** : Affiche les nouveaux logs au fur et à mesure qu'ils arrivent (comme `tail -f`).

```bash
microk8s kubectl logs -f <nom-du-pod>
```

**Quand l'utiliser** :
- Surveiller une application en cours d'exécution
- Observer le comportement pendant un test
- Voir les logs d'une application qui vient de démarrer

**Exemple** :
```bash
# Surveiller les requêtes entrantes sur un serveur web
microk8s kubectl logs -f nginx-pod

# Dans un autre terminal, faire des requêtes
curl http://mon-service/
```

**Astuce** : Appuyez sur `Ctrl+C` pour arrêter le suivi.

### 2. Voir les Logs du Conteneur Précédent (--previous)

**Description** : Si un pod a crash et redémarré, voir les logs juste avant le crash.

```bash
microk8s kubectl logs <nom-du-pod> --previous
```

ou la forme courte :
```bash
microk8s kubectl logs <nom-du-pod> -p
```

**Pourquoi c'est crucial** : Les logs actuels d'un pod redémarré ne contiennent que ce qui s'est passé APRÈS le redémarrage. Pour comprendre POURQUOI il a crashé, vous devez voir les logs du conteneur précédent.

**Scénario typique** :
```bash
# Votre pod est en CrashLoopBackOff
microk8s kubectl get pods
# NAME      READY   STATUS             RESTARTS   AGE
# mon-pod   0/1     CrashLoopBackOff   5          10m

# Les logs actuels montrent le démarrage actuel (peu utile)
microk8s kubectl logs mon-pod

# Les logs précédents montrent l'erreur qui a causé le crash
microk8s kubectl logs mon-pod --previous
# Error: Cannot connect to database at postgres:5432
# Connection refused
```

**Note** : Si le pod n'a jamais redémarré, cette commande ne retournera rien.

### 3. Limiter le Nombre de Lignes (--tail)

**Description** : N'afficher que les N dernières lignes.

```bash
microk8s kubectl logs <nom-du-pod> --tail=<nombre>
```

**Exemples** :
```bash
# Les 50 dernières lignes
microk8s kubectl logs mon-pod --tail=50

# Les 10 dernières lignes en temps réel
microk8s kubectl logs -f mon-pod --tail=10

# Une seule ligne (la plus récente)
microk8s kubectl logs mon-pod --tail=1
```

**Quand l'utiliser** :
- Pods avec beaucoup de logs (éviter d'afficher des milliers de lignes)
- Vérification rapide du dernier message
- Performance : charge moins le système

### 4. Logs Depuis un Certain Temps (--since)

**Description** : Afficher uniquement les logs depuis une durée spécifiée.

```bash
microk8s kubectl logs <nom-du-pod> --since=<durée>
```

**Formats de durée** :
- `s` : secondes
- `m` : minutes
- `h` : heures

**Exemples** :
```bash
# Logs des 5 dernières minutes
microk8s kubectl logs mon-pod --since=5m

# Logs des 2 dernières heures
microk8s kubectl logs mon-pod --since=2h

# Logs des 30 dernières secondes
microk8s kubectl logs mon-pod --since=30s
```

**Cas d'usage** :
```bash
# Un problème est survenu il y a 10 minutes, voir ce qui s'est passé
microk8s kubectl logs mon-pod --since=15m

# Combiner avec follow pour voir les nouveaux logs depuis un point précis
microk8s kubectl logs -f mon-pod --since=5m
```

### 5. Logs Depuis une Date Précise (--since-time)

**Description** : Afficher les logs depuis une date/heure spécifique (format RFC3339).

```bash
microk8s kubectl logs <nom-du-pod> --since-time=<timestamp>
```

**Format** : `YYYY-MM-DDTHH:MM:SSZ`

**Exemple** :
```bash
microk8s kubectl logs mon-pod --since-time=2025-10-26T14:30:00Z
```

**Cas d'usage** : Quand vous savez exactement à quel moment un problème est survenu.

### 6. Ajouter les Timestamps (--timestamps)

**Description** : Préfixer chaque ligne de log avec son timestamp.

```bash
microk8s kubectl logs <nom-du-pod> --timestamps
```

**Sortie sans timestamps** :
```
Application started
Listening on port 8080
GET /api/users
POST /api/users
```

**Sortie avec timestamps** :
```
2025-10-26T14:30:00.123456789Z Application started
2025-10-26T14:30:00.234567890Z Listening on port 8080
2025-10-26T14:30:05.345678901Z GET /api/users
2025-10-26T14:30:08.456789012Z POST /api/users
```

**Quand l'utiliser** :
- Corréler les logs avec des événements externes
- Comprendre la séquence temporelle d'événements
- Déboguer des problèmes de timing ou de race conditions

### 7. Limiter les Bytes (--limit-bytes)

**Description** : Limiter la quantité de données retournées (en bytes).

```bash
microk8s kubectl logs <nom-du-pod> --limit-bytes=1000
```

**Cas d'usage** : Quand les logs sont vraiment très volumineux et que vous voulez juste un aperçu.

**Note** : Moins utilisé que `--tail`, mais utile pour éviter de surcharger votre terminal.

## Combinaisons d'Options Utiles

### Scénario 1 : Déboguer un Crash Récent

**Objectif** : Voir les derniers logs avant un crash qui vient de se produire.

```bash
microk8s kubectl logs mon-pod --previous --tail=100 --timestamps
```

**Explication** :
- `--previous` : Logs du conteneur qui a crashé
- `--tail=100` : Seulement les 100 dernières lignes (pour ne pas tout afficher)
- `--timestamps` : Pour voir quand chaque événement s'est produit

### Scénario 2 : Surveiller une Application en Production

**Objectif** : Suivre en temps réel les logs récents.

```bash
microk8s kubectl logs -f mon-pod --since=10m --timestamps
```

**Explication** :
- `-f` : Suivi en temps réel
- `--since=10m` : Commence par montrer les 10 dernières minutes
- `--timestamps` : Pour voir l'heure de chaque événement

### Scénario 3 : Chercher un Problème Survenu à une Heure Précise

**Objectif** : Examiner les logs autour de 14h30.

```bash
microk8s kubectl logs mon-pod --since-time=2025-10-26T14:25:00Z --timestamps | head -200
```

**Explication** :
- `--since-time` : Commence à 14h25 (5 minutes avant)
- `--timestamps` : Pour voir les heures exactes
- `| head -200` : Limite à 200 lignes pour ne pas tout afficher

### Scénario 4 : Vérification Rapide

**Objectif** : Voir rapidement si tout va bien.

```bash
microk8s kubectl logs mon-pod --tail=20
```

**Explication** :
- `--tail=20` : Juste les 20 dernières lignes pour un aperçu rapide

## Filtrage et Recherche dans les Logs

### Utiliser grep pour Filtrer

`grep` est votre meilleur ami pour chercher dans les logs.

#### Recherche Simple

```bash
# Chercher le mot "error"
microk8s kubectl logs mon-pod | grep error

# Chercher "error" (insensible à la casse)
microk8s kubectl logs mon-pod | grep -i error

# Chercher plusieurs termes
microk8s kubectl logs mon-pod | grep -E "error|warning|fail"
```

#### Recherche avec Contexte

```bash
# Afficher 3 lignes avant et après chaque occurrence
microk8s kubectl logs mon-pod | grep -C 3 error

# Afficher 5 lignes avant
microk8s kubectl logs mon-pod | grep -B 5 error

# Afficher 5 lignes après
microk8s kubectl logs mon-pod | grep -A 5 error
```

**Exemple pratique** :
```bash
# Trouver toutes les erreurs de connexion avec contexte
microk8s kubectl logs mon-pod | grep -C 3 "connection refused"
```

#### Inverser la Recherche

```bash
# Afficher tout SAUF les lignes contenant "debug"
microk8s kubectl logs mon-pod | grep -v debug

# Afficher tout sauf "info" et "debug"
microk8s kubectl logs mon-pod | grep -v -E "info|debug"
```

#### Compter les Occurrences

```bash
# Combien d'erreurs ?
microk8s kubectl logs mon-pod | grep -c error

# Combien de requêtes GET ?
microk8s kubectl logs mon-pod | grep -c "GET /"
```

#### Recherche avec Coloration

```bash
# Mettre en évidence les occurrences trouvées
microk8s kubectl logs mon-pod | grep --color=always -E "error|warning|$"
```

### Utiliser awk pour Traiter les Logs

`awk` est puissant pour extraire des colonnes ou filtrer selon des critères complexes.

#### Extraire des Colonnes Spécifiques

```bash
# Si vos logs ont un format : timestamp level message
# Extraire uniquement le level et le message (colonnes 2 et 3+)
microk8s kubectl logs mon-pod | awk '{print $2, $3, $4, $5}'

# Extraire les timestamps (colonne 1)
microk8s kubectl logs mon-pod --timestamps | awk '{print $1}'
```

#### Filtrer par Valeur

```bash
# Afficher seulement les lignes où la colonne 2 est "ERROR"
microk8s kubectl logs mon-pod | awk '$2 == "ERROR"'

# Afficher les requêtes avec un code de réponse >= 400
microk8s kubectl logs nginx-pod | awk '$9 >= 400'
```

#### Agréger des Données

```bash
# Compter les occurrences de chaque niveau de log
microk8s kubectl logs mon-pod | awk '{print $2}' | sort | uniq -c

# Exemple de sortie :
#   150 INFO
#    23 WARNING
#     5 ERROR
```

### Utiliser sed pour Transformer

```bash
# Remplacer des chaînes pour faciliter la lecture
microk8s kubectl logs mon-pod | sed 's/com.myapp.module/Module/g'

# Extraire seulement les lignes entre deux motifs
microk8s kubectl logs mon-pod | sed -n '/START/,/END/p'
```

### Sauvegarder les Logs Filtrés

```bash
# Sauvegarder toutes les erreurs dans un fichier
microk8s kubectl logs mon-pod | grep -i error > erreurs.log

# Sauvegarder les logs avec timestamps
microk8s kubectl logs mon-pod --timestamps > logs-complets.log

# Sauvegarder les logs des 5 dernières minutes
microk8s kubectl logs mon-pod --since=5m > logs-recents.log
```

## Logs de Plusieurs Pods

### Logs d'un Deployment/ReplicaSet

Pour voir les logs de tous les pods d'un deployment :

```bash
# Utiliser un label selector
microk8s kubectl logs -l app=nginx

# Avec suivi en temps réel
microk8s kubectl logs -f -l app=nginx

# Tous les conteneurs des pods matchés
microk8s kubectl logs -l app=nginx --all-containers=true
```

**Exemple complet** :
```bash
# Votre deployment a 3 réplicas
microk8s kubectl get pods -l app=nginx
# NAME                     READY   STATUS    RESTARTS   AGE
# nginx-deployment-abc123  1/1     Running   0          1h
# nginx-deployment-def456  1/1     Running   0          1h
# nginx-deployment-ghi789  1/1     Running   0          1h

# Voir les logs de tous en même temps
microk8s kubectl logs -l app=nginx --all-containers=true --prefix=true
```

**Note** : L'option `--prefix=true` ajoute le nom du pod avant chaque ligne, pour savoir d'où vient le log.

### Logs en Parallèle (kubetail/stern)

Pour une meilleure expérience avec plusieurs pods, utilisez des outils tiers :

#### stern (recommandé)

**Installation** :
```bash
# Sur Ubuntu/Debian
wget https://github.com/stern/stern/releases/download/v1.28.0/stern_1.28.0_linux_amd64.tar.gz
tar -xzf stern_1.28.0_linux_amd64.tar.gz
sudo mv stern /usr/local/bin/

# Ou avec snap
sudo snap install stern
```

**Utilisation** :
```bash
# Voir les logs de tous les pods dont le nom commence par "nginx"
stern nginx

# Avec un label selector
stern -l app=nginx

# Avec namespace
stern -n production nginx

# Avec coloration par pod
stern --color always nginx
```

**Avantages de stern** :
- Colore différemment chaque pod
- Suit automatiquement les nouveaux pods
- Préfixe chaque ligne avec le nom du pod
- Accepte des regex pour les noms de pods

## Patterns de Logs et Interprétation

### Reconnaître les Formats de Logs Courants

#### Format Standard (Application Simple)

```
2025-10-26 14:30:00 INFO Server started on port 8080
2025-10-26 14:30:05 INFO Received GET request for /api/users
2025-10-26 14:30:06 ERROR Failed to connect to database: connection timeout
```

**Structure** : `timestamp level message`

#### Format JSON (Application Moderne)

```json
{"timestamp":"2025-10-26T14:30:00Z","level":"INFO","message":"Server started","port":8080}
{"timestamp":"2025-10-26T14:30:05Z","level":"INFO","message":"Request received","method":"GET","path":"/api/users"}
{"timestamp":"2025-10-26T14:30:06Z","level":"ERROR","message":"Database error","error":"connection timeout"}
```

**Avantage** : Facilement parsable par des outils (jq, outils de logs centralisés)

**Parser avec jq** :
```bash
# Extraire seulement les erreurs
microk8s kubectl logs mon-pod | jq -r 'select(.level=="ERROR") | .message'

# Extraire un champ spécifique
microk8s kubectl logs mon-pod | jq -r '.timestamp + " " + .message'
```

#### Format Apache/Nginx (Logs Serveur Web)

```
192.168.1.100 - - [26/Oct/2025:14:30:05 +0000] "GET /api/users HTTP/1.1" 200 1234
192.168.1.101 - - [26/Oct/2025:14:30:06 +0000] "POST /api/users HTTP/1.1" 500 89
```

**Structure** : `IP - - [timestamp] "method path protocol" status size`

**Extraire les erreurs (codes >= 400)** :
```bash
microk8s kubectl logs nginx-pod | awk '$9 >= 400'
```

### Identifier les Problèmes Courants dans les Logs

#### 1. Erreurs de Connexion

**Patterns à chercher** :
```bash
microk8s kubectl logs mon-pod | grep -i "connection refused\|timeout\|connection reset"
```

**Exemples de messages** :
```
ERROR: Connection refused to postgres:5432
ERROR: Read timeout after 30s
ERROR: Connection reset by peer
```

**Ce que ça signifie** :
- **Connection refused** : Le service cible n'est pas disponible ou pas à cette adresse
- **Timeout** : Le service est trop lent à répondre ou injoignable
- **Connection reset** : Connexion interrompue brutalement

**Actions** :
1. Vérifier que le service cible existe : `microk8s kubectl get service postgres`
2. Vérifier que le service est joignable : `microk8s kubectl exec mon-pod -- nc -zv postgres 5432`
3. Vérifier les Network Policies

#### 2. Erreurs d'Authentification

**Patterns à chercher** :
```bash
microk8s kubectl logs mon-pod | grep -i "unauthorized\|forbidden\|authentication failed\|invalid token"
```

**Exemples de messages** :
```
ERROR: 401 Unauthorized
ERROR: Authentication failed: invalid credentials
ERROR: Forbidden: insufficient permissions
ERROR: Invalid token or expired
```

**Actions** :
1. Vérifier les secrets : `microk8s kubectl get secrets`
2. Vérifier que les secrets sont bien montés dans le pod
3. Vérifier les variables d'environnement : `microk8s kubectl exec mon-pod -- env`

#### 3. Erreurs de Configuration

**Patterns à chercher** :
```bash
microk8s kubectl logs mon-pod | grep -i "config\|configuration\|missing\|not found"
```

**Exemples de messages** :
```
ERROR: Configuration file not found: /app/config.yaml
ERROR: Missing required environment variable: DATABASE_URL
ERROR: Invalid configuration: port must be between 1 and 65535
```

**Actions** :
1. Vérifier les ConfigMaps : `microk8s kubectl get configmap`
2. Vérifier le montage des volumes
3. Vérifier les variables d'environnement

#### 4. Erreurs de Ressources

**Patterns à chercher** :
```bash
microk8s kubectl logs mon-pod | grep -i "out of memory\|OOM\|killed\|no space left"
```

**Exemples de messages** :
```
ERROR: Cannot allocate memory
FATAL: Out of memory
ERROR: write /tmp/data.tmp: no space left on device
```

**Actions** :
1. Vérifier l'utilisation des ressources : `microk8s kubectl top pod mon-pod`
2. Vérifier les limites : `microk8s kubectl describe pod mon-pod`
3. Augmenter les limits si nécessaire

#### 5. Stack Traces et Exceptions

**Exemple Python** :
```
Traceback (most recent call last):
  File "app.py", line 42, in handle_request
    data = json.loads(request_body)
json.decoder.JSONDecodeError: Expecting value: line 1 column 1 (char 0)
```

**Exemple Java** :
```
Exception in thread "main" java.lang.NullPointerException
    at com.myapp.Service.processRequest(Service.java:123)
    at com.myapp.Controller.handleRequest(Controller.java:45)
```

**Comment lire** :
1. Identifiez le **type d'exception** (JSONDecodeError, NullPointerException)
2. Lisez le **message** (ce qui explique l'erreur)
3. Regardez la **stack trace** (où l'erreur s'est produite)
4. Remontez la chaîne d'appels pour comprendre le contexte

**Extraire les stack traces complètes** :
```bash
# Les stack traces s'étendent sur plusieurs lignes
microk8s kubectl logs mon-pod | grep -A 20 "Traceback\|Exception"
```

## Logs Multi-Conteneurs et Init Containers

### Pods avec Plusieurs Conteneurs

Pour un pod avec plusieurs conteneurs :

```bash
# Lister les conteneurs du pod
microk8s kubectl get pod mon-pod -o jsonpath='{.spec.containers[*].name}'
# Sortie : app sidecar

# Logs du conteneur principal
microk8s kubectl logs mon-pod -c app

# Logs du sidecar
microk8s kubectl logs mon-pod -c sidecar

# Logs de tous les conteneurs (interleaved)
microk8s kubectl logs mon-pod --all-containers=true
```

**Astuce** : Si vous ne spécifiez pas `-c` pour un pod multi-conteneurs, vous aurez une erreur vous demandant de choisir.

### Init Containers

Les init containers s'exécutent AVANT les conteneurs principaux. Si un pod ne démarre pas, vérifiez les init containers :

```bash
# Voir si le pod a des init containers
microk8s kubectl get pod mon-pod -o jsonpath='{.spec.initContainers[*].name}'

# Logs d'un init container
microk8s kubectl logs mon-pod -c init-database

# Logs d'un init container précédent (si échec et relance)
microk8s kubectl logs mon-pod -c init-database --previous
```

**Scénario typique** :
```bash
# Pod bloqué en Init:0/1
microk8s kubectl get pods
# NAME      READY   STATUS     RESTARTS   AGE
# mon-pod   0/1     Init:0/1   0          5m

# Vérifier les logs de l'init container
microk8s kubectl logs mon-pod -c init-migrations
# Running database migrations...
# ERROR: Cannot connect to database
```

## Techniques Avancées d'Analyse

### 1. Analyse de Performance

**Compter les requêtes par seconde** :
```bash
# Pour des logs avec timestamps
microk8s kubectl logs nginx-pod --timestamps | awk '{print $1}' | cut -d'T' -f2 | cut -d':' -f1-2 | uniq -c

# Exemple de sortie :
#  145 14:30
#  189 14:31
#  167 14:32
```

**Identifier les requêtes lentes** :
```bash
# Si vos logs incluent le temps de réponse
microk8s kubectl logs mon-pod | awk '$NF ~ /[0-9]+ms/ && $NF+0 > 1000'
```

### 2. Corrélation Temporelle

**Comparer les logs de deux pods** :
```bash
# Générer deux fichiers avec timestamps
microk8s kubectl logs pod1 --timestamps > pod1.log
microk8s kubectl logs pod2 --timestamps > pod2.log

# Fusionner et trier par timestamp
sort -m pod1.log pod2.log | less
```

### 3. Analyse Statistique

**Compter les types d'événements** :
```bash
# Compter par niveau de log
microk8s kubectl logs mon-pod | awk '{print $2}' | sort | uniq -c | sort -rn

# Exemple de sortie :
#  1250 INFO
#   145 WARNING
#    23 ERROR
#     2 FATAL
```

**Identifier les erreurs les plus fréquentes** :
```bash
microk8s kubectl logs mon-pod | grep ERROR | sort | uniq -c | sort -rn | head -10
```

### 4. Détection d'Anomalies

**Trouver les pics d'erreurs** :
```bash
# Regrouper les erreurs par minute
microk8s kubectl logs mon-pod --timestamps | grep ERROR | cut -d'T' -f2 | cut -d':' -f1-2 | uniq -c

# Comparer avec le nombre total de logs par minute
microk8s kubectl logs mon-pod --timestamps | cut -d'T' -f2 | cut -d':' -f1-2 | uniq -c
```

### 5. Export pour Analyse Externe

**Exporter vers CSV** :
```bash
# Logs JSON vers CSV
microk8s kubectl logs mon-pod | jq -r '[.timestamp, .level, .message] | @csv' > logs.csv
```

**Exporter avec horodatage pour Excel** :
```bash
microk8s kubectl logs mon-pod --timestamps > logs-$(date +%Y%m%d-%H%M%S).txt
```

## Automatisation et Scripts

### Script 1 : Surveiller les Erreurs en Continu

```bash
#!/bin/bash
# watch-errors.sh

POD_NAME=$1
NAMESPACE=${2:-default}

if [ -z "$POD_NAME" ]; then
    echo "Usage: $0 <pod-name> [namespace]"
    exit 1
fi

echo "Surveillance des erreurs dans $POD_NAME (namespace: $NAMESPACE)"
echo "Appuyez sur Ctrl+C pour arrêter"
echo "=========================================="

microk8s kubectl logs -f $POD_NAME -n $NAMESPACE | grep --line-buffered -i -E "error|exception|fatal|fail" | while read line; do
    echo "[$(date '+%Y-%m-%d %H:%M:%S')] $line"
done
```

**Utilisation** :
```bash
chmod +x watch-errors.sh
./watch-errors.sh mon-pod production
```

### Script 2 : Extraire et Analyser les Erreurs

```bash
#!/bin/bash
# analyze-errors.sh

POD_NAME=$1
NAMESPACE=${2:-default}
OUTPUT_FILE="errors-analysis-$(date +%Y%m%d-%H%M%S).txt"

if [ -z "$POD_NAME" ]; then
    echo "Usage: $0 <pod-name> [namespace]"
    exit 1
fi

echo "Analyse des erreurs de $POD_NAME"
echo "=================================" > $OUTPUT_FILE
echo "" >> $OUTPUT_FILE

# Obtenir tous les logs
LOGS=$(microk8s kubectl logs $POD_NAME -n $NAMESPACE)

# Compter les erreurs
ERROR_COUNT=$(echo "$LOGS" | grep -i error | wc -l)
echo "Nombre total d'erreurs: $ERROR_COUNT" >> $OUTPUT_FILE
echo "" >> $OUTPUT_FILE

# Top 10 des erreurs les plus fréquentes
echo "Top 10 des erreurs les plus fréquentes:" >> $OUTPUT_FILE
echo "---------------------------------------" >> $OUTPUT_FILE
echo "$LOGS" | grep -i error | sort | uniq -c | sort -rn | head -10 >> $OUTPUT_FILE
echo "" >> $OUTPUT_FILE

# Première et dernière erreur
echo "Première erreur:" >> $OUTPUT_FILE
echo "$LOGS" | grep -i error | head -1 >> $OUTPUT_FILE
echo "" >> $OUTPUT_FILE

echo "Dernière erreur:" >> $OUTPUT_FILE
echo "$LOGS" | grep -i error | tail -1 >> $OUTPUT_FILE
echo "" >> $OUTPUT_FILE

# Erreurs avec stack traces
echo "Erreurs avec stack traces:" >> $OUTPUT_FILE
echo "$LOGS" | grep -A 10 -i "exception\|traceback" >> $OUTPUT_FILE

echo "Analyse terminée. Résultat dans: $OUTPUT_FILE"
cat $OUTPUT_FILE
```

### Script 3 : Comparer les Logs Avant/Après

```bash
#!/bin/bash
# compare-logs.sh

POD_NAME=$1
NAMESPACE=${2:-default}
BEFORE_FILE="logs-before.txt"
AFTER_FILE="logs-after.txt"

echo "Capture des logs 'avant'..."
microk8s kubectl logs $POD_NAME -n $NAMESPACE > $BEFORE_FILE

echo "Attente... Effectuez votre action maintenant (ex: redémarrage, changement de config)"
read -p "Appuyez sur Enter quand vous êtes prêt à capturer les logs 'après'..."

echo "Capture des logs 'après'..."
microk8s kubectl logs $POD_NAME -n $NAMESPACE > $AFTER_FILE

echo "Différences:"
diff $BEFORE_FILE $AFTER_FILE

echo ""
echo "Nouveaux logs:"
comm -13 <(sort $BEFORE_FILE) <(sort $AFTER_FILE)
```

### Script 4 : Alertes par Email

```bash
#!/bin/bash
# alert-on-errors.sh

POD_NAME=$1
NAMESPACE=${2:-default}
EMAIL="admin@example.com"
ERROR_THRESHOLD=5

# Compter les erreurs des 5 dernières minutes
ERROR_COUNT=$(microk8s kubectl logs $POD_NAME -n $NAMESPACE --since=5m | grep -i error | wc -l)

if [ $ERROR_COUNT -gt $ERROR_THRESHOLD ]; then
    SUBJECT="ALERTE: $ERROR_COUNT erreurs détectées dans $POD_NAME"
    BODY="Le pod $POD_NAME (namespace: $NAMESPACE) a généré $ERROR_COUNT erreurs dans les 5 dernières minutes.

Dernières erreurs:
$(microk8s kubectl logs $POD_NAME -n $NAMESPACE --since=5m | grep -i error | tail -10)"

    echo "$BODY" | mail -s "$SUBJECT" $EMAIL
    echo "Alerte envoyée à $EMAIL"
fi
```

**Configuration cron** :
```bash
# Vérifier toutes les 5 minutes
*/5 * * * * /path/to/alert-on-errors.sh mon-pod production
```

## Bonnes Pratiques

### 1. Format de Logs Structurés

**Mauvais exemple** (logs non structurés) :
```
Server started
User John connected
Error occurred
```

**Bon exemple** (logs structurés) :
```json
{"timestamp":"2025-10-26T14:30:00Z","level":"INFO","component":"server","message":"Server started","port":8080}
{"timestamp":"2025-10-26T14:30:05Z","level":"INFO","component":"auth","message":"User connected","user":"John"}
{"timestamp":"2025-10-26T14:30:06Z","level":"ERROR","component":"database","message":"Connection failed","error":"timeout"}
```

**Avantages** :
- Facilement parsable
- Facilite les recherches
- Compatible avec les outils de logs centralisés

### 2. Niveaux de Log Appropriés

Utilisez les bons niveaux de sévérité :

- **DEBUG** : Informations de débogage détaillées (ne pas utiliser en production)
- **INFO** : Informations générales (démarrage, arrêt, événements normaux)
- **WARNING** : Quelque chose d'anormal mais non critique
- **ERROR** : Erreur qui nécessite attention
- **FATAL/CRITICAL** : Erreur catastrophique, l'application va s'arrêter

**Exemple** :
```python
import logging

logging.debug("Valeur de la variable x: 42")  # DEBUG
logging.info("Utilisateur connecté")          # INFO
logging.warning("Pool de connexions presque plein")  # WARNING
logging.error("Impossible de se connecter à la base")  # ERROR
logging.critical("Out of memory, arrêt de l'application")  # FATAL
```

### 3. Contexte dans les Logs

**Mauvais** :
```
Error: Connection failed
```

**Bon** :
```json
{
  "level": "ERROR",
  "message": "Database connection failed",
  "error": "connection timeout after 30s",
  "database": "postgres",
  "host": "db.example.com",
  "port": 5432,
  "user": "app_user",
  "timestamp": "2025-10-26T14:30:06Z"
}
```

**Pourquoi** : Le contexte permet de déboguer sans avoir à ajouter d'autres logs ou relancer.

### 4. Éviter les Logs Sensibles

**Ne JAMAIS logger** :
- Mots de passe
- Tokens d'authentification
- Numéros de carte de crédit
- Données personnelles sensibles

**Mauvais** :
```
User login: username=john, password=secret123
```

**Bon** :
```
User login successful: username=john
```

### 5. Rotation et Rétention

Si vous avez des applications très verbeuses :

**Dans votre configuration de logging (ex: Python)** :
```python
import logging
from logging.handlers import RotatingFileHandler

handler = RotatingFileHandler(
    'app.log',
    maxBytes=10*1024*1024,  # 10MB
    backupCount=5
)
```

**Note** : Dans Kubernetes, la rotation est généralement gérée au niveau du nœud, pas de l'application.

### 6. Utiliser des Request IDs

Pour tracer une requête à travers plusieurs services :

```json
{"request_id":"abc-123","service":"api","message":"Request received"}
{"request_id":"abc-123","service":"database","message":"Query executed"}
{"request_id":"abc-123","service":"api","message":"Response sent"}
```

**Recherche** :
```bash
microk8s kubectl logs mon-pod | grep "abc-123"
```

## Troubleshooting des Logs

### Problème 1 : Pas de Logs

**Symptôme** :
```bash
microk8s kubectl logs mon-pod
# (aucune sortie)
```

**Causes possibles** :
1. Le pod n'a pas encore démarré
2. L'application n'écrit rien sur stdout/stderr
3. L'application écrit dans des fichiers au lieu de stdout

**Solutions** :
```bash
# Vérifier l'état du pod
microk8s kubectl get pod mon-pod

# Vérifier les événements
microk8s kubectl describe pod mon-pod

# Vérifier si l'application écrit dans des fichiers
microk8s kubectl exec mon-pod -- ls -la /var/log/
```

### Problème 2 : Logs Tronqués

**Symptôme** : Vous ne voyez pas tous les logs.

**Cause** : Kubernetes ne garde que les logs récents par défaut.

**Solutions** :
```bash
# Utiliser --tail pour voir plus de lignes
microk8s kubectl logs mon-pod --tail=1000

# Mettre en place des logs centralisés pour conserver l'historique
```

### Problème 3 : "previous terminated container has not been created"

**Symptôme** :
```bash
microk8s kubectl logs mon-pod --previous
# Error: previous terminated container "app" in pod "mon-pod" not found
```

**Cause** : Le pod n'a jamais redémarré, il n'y a donc pas de conteneur précédent.

**Solution** : Utilisez simplement `kubectl logs mon-pod` sans `--previous`.

### Problème 4 : Trop de Logs

**Symptôme** : Des milliers de lignes s'affichent et votre terminal est surchargé.

**Solutions** :
```bash
# Limiter avec --tail
microk8s kubectl logs mon-pod --tail=100

# Filtrer immédiatement
microk8s kubectl logs mon-pod | grep error

# Sauvegarder dans un fichier pour analyse
microk8s kubectl logs mon-pod > logs.txt
```

### Problème 5 : Logs d'un Pod qui a Disparu

**Symptôme** : Le pod a été supprimé et vous n'avez plus accès aux logs.

**Prévention** :
- Mettre en place des logs centralisés (ELK, Loki)
- Sauvegarder régulièrement les logs importants
- Augmenter la rétention au niveau du nœud

**Si c'est trop tard** : Vous ne pouvez pas récupérer les logs d'un pod supprimé sans logs centralisés.

## Intégration avec les Logs Centralisés

### Pourquoi des Logs Centralisés ?

`kubectl logs` a des limitations :
- Logs perdus après suppression du pod
- Pas d'historique long terme
- Difficile de chercher dans les logs de tous les pods
- Pas d'agrégation ou visualisation avancée

**Solutions** :
- **ELK Stack** (Elasticsearch, Logstash, Kibana)
- **Loki + Grafana** (plus léger, similaire à Prometheus pour les logs)
- **Fluentd/Fluent Bit** (collecteurs de logs)

### Workflow Hybride

En attendant des logs centralisés, combinez :

1. **kubectl logs** pour le diagnostic immédiat
2. **Sauvegarde manuelle** des logs importants
3. **Scripts d'archivage** automatiques

**Exemple de script d'archivage** :
```bash
#!/bin/bash
# archive-logs.sh

NAMESPACE="production"
OUTPUT_DIR="~/logs-archive/$(date +%Y%m%d)"

mkdir -p $OUTPUT_DIR

# Archiver les logs de tous les pods
for pod in $(microk8s kubectl get pods -n $NAMESPACE -o name); do
    POD_NAME=$(echo $pod | cut -d'/' -f2)
    echo "Archivage de $POD_NAME..."
    microk8s kubectl logs $pod -n $NAMESPACE > "$OUTPUT_DIR/$POD_NAME.log"
done

echo "Logs archivés dans $OUTPUT_DIR"
```

## Checklist : Analyse de Logs

Voici une checklist pour analyser efficacement les logs :

- [ ] **Identifier le pod** : Connaître le nom exact du pod à analyser
- [ ] **Vérifier l'état** : `kubectl get pod` - Le pod est-il Running ?
- [ ] **Logs actuels** : `kubectl logs <pod>` - Que se passe-t-il maintenant ?
- [ ] **Logs précédents** : `kubectl logs <pod> --previous` - En cas de redémarrage
- [ ] **Timestamps** : Ajouter `--timestamps` pour comprendre la chronologie
- [ ] **Chercher les erreurs** : `| grep -i error`
- [ ] **Contexte des erreurs** : `| grep -C 3 error`
- [ ] **Stack traces** : `| grep -A 10 "Exception\|Traceback"`
- [ ] **Patterns spécifiques** : Connection refused, timeout, out of memory, etc.
- [ ] **Corrélation** : Comparer avec les événements (`kubectl get events`)
- [ ] **Sauvegarder** : Si nécessaire, exporter les logs pour analyse détaillée
- [ ] **Documenter** : Noter les erreurs trouvées et les actions prises

## Résumé

### Commandes Essentielles à Retenir

| Commande | Utilité |
|----------|---------|
| `microk8s kubectl logs <pod>` | Voir les logs d'un pod |
| `microk8s kubectl logs -f <pod>` | Suivre en temps réel |
| `microk8s kubectl logs <pod> --previous` | Logs avant crash |
| `microk8s kubectl logs <pod> --tail=100` | Dernières 100 lignes |
| `microk8s kubectl logs <pod> --since=5m` | Logs des 5 dernières minutes |
| `microk8s kubectl logs <pod> --timestamps` | Avec horodatage |
| `microk8s kubectl logs <pod> -c <conteneur>` | Pod multi-conteneurs |
| `microk8s kubectl logs -l app=nginx` | Tous les pods d'un label |

### Patterns de Filtrage Courants

| Recherche | Commande |
|-----------|----------|
| Erreurs | `\| grep -i error` |
| Erreurs avec contexte | `\| grep -C 3 -i error` |
| Warnings et erreurs | `\| grep -E "warning\|error"` |
| Exclure debug | `\| grep -v debug` |
| Compter erreurs | `\| grep -c error` |
| Stack traces | `\| grep -A 10 "Exception"` |

### Points Clés

1. **Les logs sont temporaires** - Mettez en place des logs centralisés pour l'historique long terme
2. **Utilisez --previous** - Essentiel après un crash
3. **Ajoutez des timestamps** - Pour comprendre la chronologie
4. **Filtrez intelligemment** - grep, awk, sed sont vos amis
5. **Contexte est roi** - Ne vous contentez pas d'une ligne d'erreur, regardez autour
6. **Automatisez** - Créez des scripts pour les analyses répétitives
7. **Logs structurés** - JSON facilite énormément l'analyse

**Prochaine étape** : Section 23.5 - Problèmes réseau courants

---


⏭️ [Problèmes réseau courants](/23-depannage-et-maintenance/05-problemes-reseau-courants.md)
