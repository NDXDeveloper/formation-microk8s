🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 15.1 Les Trois Piliers de l'Observabilité

## Introduction à l'Observabilité

Lorsque vous déployez des applications dans un cluster Kubernetes comme MicroK8s, il ne suffit pas de savoir si elles "tournent" ou non. Vous devez comprendre **comment** elles fonctionnent, **pourquoi** elles se comportent d'une certaine manière, et **où** se trouvent les problèmes potentiels.

C'est précisément l'objectif de l'**observabilité** : rendre vos systèmes transparents et compréhensibles.

### Observabilité vs Monitoring

Avant d'aller plus loin, clarifions une distinction importante :

- **Le monitoring** consiste à surveiller des métriques connues à l'avance (CPU, mémoire, nombre de requêtes, etc.). C'est comme avoir des capteurs sur votre voiture pour surveiller la température du moteur.

- **L'observabilité** va plus loin : elle vous permet de poser des questions que vous n'aviez pas anticipées. C'est la capacité à comprendre l'état interne de votre système en observant ses sorties externes.

En d'autres termes, le monitoring vous dit "quelque chose ne va pas", tandis que l'observabilité vous aide à comprendre "pourquoi et comment le résoudre".

## Les Trois Piliers

L'observabilité repose sur trois types de données complémentaires, souvent appelés les "trois piliers" :

### 1. Les Métriques (Metrics)

#### Qu'est-ce que c'est ?

Les métriques sont des **valeurs numériques mesurées au fil du temps**. Ce sont des données quantitatives qui vous donnent une vue d'ensemble de la santé de votre système.

#### Exemples concrets

- **Utilisation CPU** : 45% d'utilisation processeur
- **Mémoire consommée** : 2,3 Go sur 4 Go disponibles
- **Nombre de requêtes HTTP** : 1250 requêtes par minute
- **Temps de réponse** : 125 millisecondes en moyenne
- **Taux d'erreur** : 0,5% des requêtes échouent
- **Nombre de pods actifs** : 15 pods en cours d'exécution

#### Caractéristiques

- **Structure fixe** : toujours le même format (nom, valeur, timestamp)
- **Agrégation facile** : vous pouvez calculer des moyennes, des sommes, des percentiles
- **Stockage efficace** : prennent peu d'espace disque
- **Idéales pour les alertes** : "Si CPU > 80%, envoyer une alerte"
- **Vision tendancielle** : permettent de voir l'évolution dans le temps

#### Dans le contexte Kubernetes

Dans votre cluster MicroK8s, les métriques vous indiquent :
- Combien de ressources chaque pod consomme
- Si un nœud est surchargé
- La latence des services
- Le débit réseau entre les pods

**Outil principal dans MicroK8s** : Prometheus (que nous avons vu au chapitre 12)

### 2. Les Logs (Journaux)

#### Qu'est-ce que c'est ?

Les logs sont des **enregistrements textuels d'événements** qui se produisent dans vos applications. Chaque ligne de log représente un événement avec un contexte détaillé.

#### Exemples concrets

```
2025-10-25 14:32:15 INFO  Utilisateur 'jean.dupont' connecté avec succès
2025-10-25 14:32:18 WARN  Connexion base de données lente : 3.2s
2025-10-25 14:32:45 ERROR Échec de paiement pour la commande #12345 : timeout API bancaire
2025-10-25 14:33:01 DEBUG Requête GET /api/products - 200 OK - 45ms
```

#### Caractéristiques

- **Non structurées ou semi-structurées** : format texte libre ou JSON
- **Riches en contexte** : contiennent des détails sur ce qui s'est passé
- **Volumineuses** : peuvent générer énormément de données
- **Idéales pour le débogage** : racontent l'histoire de ce qui s'est passé
- **Recherche et filtrage** : permettent de trouver des événements spécifiques

#### Dans le contexte Kubernetes

Les logs dans MicroK8s vous permettent de :
- Voir les messages d'erreur d'une application qui crashe
- Tracer le parcours d'une requête utilisateur
- Comprendre pourquoi un pod a redémarré
- Identifier des patterns d'erreurs répétitives

**Outils courants** :
- `kubectl logs` (natif Kubernetes)
- Stack ELK (Elasticsearch, Logstash, Kibana)
- Loki + Grafana (plus léger, que nous verrons section 15.4)

### 3. Les Traces (Traces distribuées)

#### Qu'est-ce que c'est ?

Les traces suivent le **parcours complet d'une requête** à travers tous les services qu'elle traverse. C'est comme un GPS qui enregistre tous les points de passage d'un trajet.

#### Exemple concret

Imaginons qu'un utilisateur effectue un achat sur votre site e-commerce :

```
Trace ID: abc123xyz
|
├─ [API Gateway] Réception requête POST /checkout - 450ms
   |
   ├─ [Service Auth] Vérification token utilisateur - 50ms
   |
   ├─ [Service Panier] Récupération articles - 80ms
   |  |
   |  └─ [Base de données] Requête SQL SELECT - 65ms
   |
   ├─ [Service Paiement] Traitement carte bancaire - 280ms
   |  |
   |  ├─ [API Bancaire externe] Autorisation paiement - 250ms
   |  └─ [Base de données] Enregistrement transaction - 15ms
   |
   └─ [Service Email] Envoi confirmation - 40ms
```

#### Caractéristiques

- **Vue end-to-end** : montre le chemin complet d'une requête
- **Hiérarchie parent-enfant** : chaque étape est liée à la précédente
- **Timing précis** : temps passé à chaque étape
- **Contexte distribué** : fonctionne à travers plusieurs services
- **Identification des goulots** : repère facilement où le temps est perdu

#### Dans le contexte Kubernetes

Les traces dans MicroK8s vous aident à :
- Comprendre les dépendances entre vos microservices
- Identifier quel service ralentit l'ensemble
- Déboguer des erreurs qui apparaissent seulement dans certains parcours
- Optimiser les performances en trouvant les appels inutiles

**Outils courants** : Jaeger, Zipkin, Tempo (que nous verrons section 15.5)

## Pourquoi Trois Piliers ? La Complémentarité

Chaque pilier répond à des questions différentes :

### Scénario : Votre Application est Lente

1. **Les métriques** vous alertent : "Le temps de réponse moyen est passé de 100ms à 3000ms"
   - Elles identifient **QUAND** le problème a commencé
   - Elles quantifient **L'AMPLEUR** du problème

2. **Les logs** vous donnent des indices : "Erreur: Connection timeout to database after 2.5s"
   - Ils expliquent **QUEL** composant est affecté
   - Ils fournissent des **MESSAGES D'ERREUR** précis

3. **Les traces** montrent le chemin : "La requête passe 2.8s à attendre une réponse du service de base de données"
   - Elles révèlent **OÙ** exactement se trouve le goulot
   - Elles visualisent **COMMENT** les services interagissent

### Tableau Comparatif

| Aspect | Métriques | Logs | Traces |
|--------|-----------|------|--------|
| **Type de données** | Numériques | Textuelles | Structurées (spans) |
| **Volume** | Faible | Élevé | Moyen |
| **Rétention typique** | Longue (mois/années) | Moyenne (jours/semaines) | Courte (jours) |
| **Question** | "Combien ?" | "Quoi ?" | "Où et comment ?" |
| **Utilisation** | Alertes, dashboards | Débogage, audit | Performance, dépendances |
| **Coût stockage** | Faible | Élevé | Moyen |
| **Recherche** | Rapide (agrégats) | Lente (full-text) | Moyenne (indexée) |

## Comment les Utiliser Ensemble

### Workflow d'Investigation Typique

1. **Alerte déclenchée** (via métriques) : "Taux d'erreur > 5%"

2. **Identification du périmètre** (via métriques) :
   - Quel service est affecté ?
   - Depuis quand ?
   - Quelle est l'ampleur ?

3. **Recherche des erreurs** (via logs) :
   - Quels sont les messages d'erreur exacts ?
   - Y a-t-il des patterns communs ?
   - Quelles sont les stack traces ?

4. **Analyse du flux** (via traces) :
   - Quel est le parcours des requêtes en échec ?
   - Où se produit l'erreur dans la chaîne d'appels ?
   - Y a-t-il des dépendances problématiques ?

5. **Résolution et vérification** :
   - Corriger le problème identifié
   - Vérifier les métriques pour confirmer le retour à la normale
   - Garder les logs pour analyse post-mortem

## Architecture d'Observabilité dans MicroK8s

Voici comment ces trois piliers s'intègrent dans votre cluster MicroK8s :

```
┌─────────────────────────────────────────────────────┐
│                   Vos Applications                  │
│              (Pods dans MicroK8s)                   │
└─────────────────────────────────────────────────────┘
         │              │              │
         │ métriques    │ logs         │ traces
         ↓              ↓              ↓
┌─────────────┐  ┌─────────────┐  ┌─────────────┐
│  Prometheus │  │    Loki     │  │   Jaeger    │
│             │  │     ou      │  │             │
│  (Collecte  │  │     EFK     │  │  (Collecte  │
│  métriques) │  │  (Collecte  │  │   traces)   │
└─────────────┘  │    logs)    │  └─────────────┘
         │       └─────────────┘         │
         │              │                │
         └──────────────┴────────────────┘
                        ↓
                ┌─────────────┐
                │   Grafana   │
                │             │
                │ (Tableau de │
                │    bord     │
                │  unifié)    │
                └─────────────┘
```

## Bonnes Pratiques pour Débutants

### Pour les Métriques

1. **Commencez simple** : surveillez CPU, mémoire, et temps de réponse
2. **Utilisez des dashboards existants** : ne réinventez pas la roue
3. **Définissez des seuils réalistes** : évitez trop d'alertes (alert fatigue)
4. **Gardez un historique** : les données passées aident à comprendre les tendances

### Pour les Logs

1. **Structurez vos logs** : utilisez le format JSON quand possible
2. **Incluez du contexte** : ajoutez des IDs de requête, noms d'utilisateurs, etc.
3. **Utilisez des niveaux** : DEBUG, INFO, WARN, ERROR
4. **Attention au volume** : ne loguez pas tout en production
5. **Centralisez** : tous les logs au même endroit

### Pour les Traces

1. **Commencez petit** : tracez d'abord les flux critiques
2. **Échantillonnage** : ne tracez pas 100% du trafic (coûteux)
3. **Propagez le contexte** : assurez-vous que les IDs de trace passent entre services
4. **Nommez bien vos spans** : utilisez des noms descriptifs

## Correlation : Le Pouvoir de la Combinaison

Le véritable pouvoir vient de la **corrélation** entre les trois piliers :

### Exemple de Corrélation

Imaginez que vous voyez une métrique anormale :
```
Latence API : +300% à 14:32
```

Avec la corrélation :

1. **Métriques → Logs** :
   - Timestamp de la métrique → Recherche logs au même moment
   - Trouve : "Database connection pool exhausted at 14:32"

2. **Logs → Traces** :
   - ID de requête dans le log → Recherche trace correspondante
   - Visualise : Requête bloquée pendant 8s sur l'acquisition d'une connexion DB

3. **Traces → Métriques** :
   - Span database montre 8s → Retour aux métriques
   - Confirme : Pool de connexions DB saturé à 100% à 14:32

**Résultat** : Problème identifié en quelques minutes au lieu de plusieurs heures de débogage !

## Ce Qu'il Faut Retenir

🔍 **Les trois piliers sont complémentaires**, pas interchangeables :
- **Métriques** = Vue d'ensemble quantitative
- **Logs** = Détails et contexte
- **Traces** = Parcours et relations

📊 **Chaque pilier a ses forces** :
- Métriques : idéales pour les alertes et les tendances
- Logs : parfaits pour le débogage détaillé
- Traces : essentielles pour comprendre les systèmes distribués

🔗 **La corrélation multiplie leur valeur** :
- Utilisez-les ensemble pour résoudre les problèmes plus rapidement
- Liez-les par des identifiants communs (trace ID, request ID)

🎯 **Pour un lab MicroK8s** :
- Commencez par les métriques (Prometheus)
- Ajoutez les logs (Loki est léger et s'intègre bien)
- Les traces viennent en dernier (plus complexes à mettre en place)

🚀 **Prochaines étapes** :
- Dans les sections suivantes, nous verrons comment implémenter concrètement chaque pilier dans votre cluster MicroK8s
- Nous apprendrons à les configurer et à les utiliser efficacement

---

## Conclusion

L'observabilité n'est pas un luxe, c'est une **nécessité** pour gérer des applications modernes, même dans un lab personnel. Les trois piliers travaillent ensemble pour vous donner une vision complète de ce qui se passe dans votre cluster MicroK8s.

Avec une bonne observabilité, vous passerez moins de temps à chercher des problèmes et plus de temps à améliorer vos applications. C'est un investissement qui paie très rapidement, surtout quand les choses tournent mal à 3h du matin ! 😊

Dans les prochaines sections, nous plongerons dans l'implémentation pratique de chacun de ces piliers.

⏭️ [Métriques custom dans vos applications](/15-observabilite-avancee/02-metriques-custom-dans-vos-applications.md)
