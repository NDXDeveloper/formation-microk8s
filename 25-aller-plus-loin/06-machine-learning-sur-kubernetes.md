🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 25.6 Machine Learning sur Kubernetes

## Introduction : Machine Learning et Kubernetes, pourquoi ?

### Qu'est-ce que le Machine Learning ?

Le **Machine Learning** (apprentissage automatique) est une branche de l'intelligence artificielle où les ordinateurs "apprennent" à partir de données sans être explicitement programmés pour chaque tâche.

**Analogie simple** :
```
Programmation traditionnelle :
  Si température > 30°C → Allumer climatisation
  (Règle écrite par un humain)

Machine Learning :
  Donner 1000 exemples de températures et actions
  → Le modèle apprend tout seul à décider
  → Peut s'adapter à de nouveaux cas
```

**Exemples concrets** :
- Reconnaissance d'images (détection de visages, objets)
- Traduction automatique
- Recommandations (Netflix, Spotify)
- Détection de fraude bancaire
- Maintenance prédictive (prévoir les pannes)
- Assistant vocal (Siri, Alexa)

### Le cycle de vie d'un projet ML

Un projet de Machine Learning n'est pas juste "entraîner un modèle", c'est un **pipeline complet** :

```
1. Collecte de données
   ↓
2. Préparation et nettoyage (Feature Engineering)
   ↓
3. Entraînement du modèle (Training)
   ↓
4. Évaluation et validation
   ↓
5. Déploiement en production (Serving)
   ↓
6. Monitoring des performances
   ↓
7. Ré-entraînement périodique
   (retour à l'étape 1)
```

Chaque étape peut nécessiter :
- **Des ressources différentes** (CPU, GPU, mémoire)
- **Des outils différents** (Python, Jupyter, TensorFlow, PyTorch)
- **Du scaling** (entraînement sur plusieurs GPU)

C'est là que **Kubernetes entre en jeu** !

### Pourquoi Kubernetes pour le Machine Learning ?

#### Problèmes sans Kubernetes

```
Data Scientist 1 : "J'ai besoin d'un serveur GPU pour 2h"
Admin : "Désolé, tous pris"

Data Scientist 2 : "Mon entraînement a planté après 5h"
Admin : "Il faut tout recommencer"

Data Scientist 3 : "Mon modèle fonctionne sur mon laptop mais pas en prod"
Admin : "Dependencies différentes..."

Data Scientist 4 : "Je veux comparer 10 configurations"
Admin : "Ça va prendre 20 jours en séquentiel"
```

#### Solutions avec Kubernetes

**✅ Partage de ressources** :
```yaml
# Data Scientist 1 demande un GPU pour 2h
apiVersion: batch/v1
kind: Job
metadata:
  name: training-job-1
spec:
  template:
    spec:
      containers:
      - name: training
        resources:
          limits:
            nvidia.com/gpu: 1
      restartPolicy: Never
  backoffLimit: 3
  ttlSecondsAfterFinished: 7200  # Nettoie après 2h
```

Les ressources sont libérées automatiquement !

**✅ Résilience et retry** :
```yaml
spec:
  backoffLimit: 3  # Réessaye automatiquement 3 fois
  activeDeadlineSeconds: 18000  # Timeout après 5h
```

**✅ Reproductibilité** :
```yaml
containers:
- name: training
  image: myml/training:v1.2.3  # Image Docker versionnée
  env:
  - name: RANDOM_SEED
    value: "42"  # Résultats reproductibles
```

**✅ Parallélisation** :
```yaml
# Lancer 10 entraînements en parallèle
apiVersion: batch/v1
kind: Job
metadata:
  name: hyperparameter-tuning
spec:
  parallelism: 10  # 10 configurations en même temps
  completions: 10
```

## Architecture ML sur Kubernetes

### Les trois phases principales

```
┌─────────────────────────────────────────────────────────┐
│                    PHASE 1 : TRAINING                   │
│  ┌──────────────┐    ┌──────────────┐    ┌──────────┐   │
│  │  Data Prep   │ →  │   Training   │ →  │  Model   │   │
│  │  (CPU)       │    │   (GPU)      │    │  Storage │   │
│  └──────────────┘    └──────────────┘    └──────────┘   │
└─────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────┐
│                    PHASE 2 : SERVING                    │
│  ┌──────────────┐    ┌──────────────┐    ┌──────────┐   │
│  │   Model      │ →  │  Inference   │ →  │  API     │   │
│  │   Loading    │    │  Server      │    │  Gateway │   │
│  └──────────────┘    └──────────────┘    └──────────┘   │
└─────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────┐
│                   PHASE 3 : MONITORING                  │
│  ┌──────────────┐    ┌──────────────┐    ┌──────────┐   │
│  │  Prediction  │ →  │  Model       │ →  │ Retrain  │   │
│  │  Logging     │    │  Drift Alert │    │ Trigger  │   │
│  └──────────────┘    └──────────────┘    └──────────┘   │
└─────────────────────────────────────────────────────────┘
```

### Composants typiques

#### 1. Notebooks interactifs (Jupyter)

Les data scientists adorent Jupyter pour explorer les données :

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: jupyter-notebook
spec:
  replicas: 1
  selector:
    matchLabels:
      app: jupyter
  template:
    metadata:
      labels:
        app: jupyter
    spec:
      containers:
      - name: jupyter
        image: jupyter/tensorflow-notebook:latest
        ports:
        - containerPort: 8888
        env:
        - name: JUPYTER_ENABLE_LAB
          value: "yes"
        volumeMounts:
        - name: notebooks
          mountPath: /home/jovyan/work
        - name: datasets
          mountPath: /data
      volumes:
      - name: notebooks
        persistentVolumeClaim:
          claimName: jupyter-notebooks
      - name: datasets
        persistentVolumeClaim:
          claimName: shared-datasets
```

#### 2. Jobs d'entraînement

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: train-image-classifier
spec:
  template:
    spec:
      containers:
      - name: trainer
        image: myml/image-classifier-train:v1
        command: ["python", "train.py"]
        args:
        - "--epochs=100"
        - "--batch-size=32"
        - "--learning-rate=0.001"
        resources:
          limits:
            nvidia.com/gpu: 1
            memory: "16Gi"
          requests:
            memory: "8Gi"
            cpu: "4"
        volumeMounts:
        - name: dataset
          mountPath: /data
        - name: models
          mountPath: /models
      restartPolicy: OnFailure
      volumes:
      - name: dataset
        persistentVolumeClaim:
          claimName: training-dataset
      - name: models
        persistentVolumeClaim:
          claimName: model-storage
```

#### 3. Inference Server

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: model-inference
spec:
  replicas: 3
  selector:
    matchLabels:
      app: inference
  template:
    spec:
      containers:
      - name: server
        image: tensorflow/serving:latest
        ports:
        - containerPort: 8501
        env:
        - name: MODEL_NAME
          value: "image_classifier"
        volumeMounts:
        - name: models
          mountPath: /models
          readOnly: true
      volumes:
      - name: models
        persistentVolumeClaim:
          claimName: model-storage
---
apiVersion: v1
kind: Service
metadata:
  name: inference-service
spec:
  type: LoadBalancer
  selector:
    app: inference
  ports:
  - port: 80
    targetPort: 8501
```

## GPU sur Kubernetes

### Pourquoi les GPU pour le ML ?

Les **GPU (Graphics Processing Units)** sont essentiels pour le Deep Learning car ils excellent dans les calculs parallèles.

**Analogie** :
```
CPU : 1 ouvrier très qualifié et rapide
  → Parfait pour des tâches complexes séquentielles

GPU : 1000 ouvriers moyennement qualifiés travaillant ensemble
  → Parfait pour des tâches répétitives en parallèle (matrices)
```

**Performance** :
- Entraînement d'un modèle sur CPU : 10 jours
- Même modèle sur GPU : 10 heures
- Même modèle sur 8 GPU : 2 heures

### Support GPU dans Kubernetes

#### Installation du NVIDIA Device Plugin

```bash
# Sur votre cluster MicroK8s avec GPU NVIDIA
kubectl create -f https://raw.githubusercontent.com/NVIDIA/k8s-device-plugin/v0.14.0/nvidia-device-plugin.yml

# Vérifier que les GPU sont détectés
kubectl get nodes -o json | jq '.items[].status.capacity'

# Vous devriez voir :
# "nvidia.com/gpu": "1"  (ou le nombre de GPU disponibles)
```

#### Utiliser un GPU dans un Pod

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: gpu-test
spec:
  containers:
  - name: cuda-test
    image: nvidia/cuda:11.8.0-base-ubuntu22.04
    command: ["nvidia-smi"]
    resources:
      limits:
        nvidia.com/gpu: 1  # Demander 1 GPU
```

#### Partage de GPU (MIG - Multi-Instance GPU)

Les GPU modernes (NVIDIA A100, H100) peuvent être partitionnés :

```yaml
resources:
  limits:
    nvidia.com/gpu: 1
    nvidia.com/mig-1g.10gb: 1  # 1/7 d'un A100
```

Cela permet à plusieurs petits jobs de partager un GPU coûteux.

### Scheduler intelligent pour GPU

**Time-slicing** : Partager un GPU entre plusieurs Pods :

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: time-slicing-config
  namespace: gpu-operator
data:
  any: |-
    version: v1
    sharing:
      timeSlicing:
        replicas: 4  # 4 Pods peuvent partager 1 GPU
```

## Kubeflow : La plateforme ML sur Kubernetes

### Qu'est-ce que Kubeflow ?

**Kubeflow** est une plateforme complète pour déployer des workflows de Machine Learning sur Kubernetes. C'est le "standard" pour ML sur K8s.

```
Kubeflow = Jupyter + Pipelines + Training + Serving + Monitoring
```

### Architecture Kubeflow

```
┌─────────────────────────────────────────────────────┐
│              Kubeflow Dashboard (UI)                │
└─────────────────────────────────────────────────────┘
                         │
      ┌──────────────────┼──────────────────┐
      │                  │                  │
┌─────▼──────┐  ┌────────▼────────┐  ┌──────▼─────┐
│ Notebooks  │  │   Pipelines     │  │  Training  │
│ (Jupyter)  │  │   (Argo/Tekton) │  │  Operators │
└────────────┘  └─────────────────┘  └────────────┘
                         │
      ┌──────────────────┼──────────────────┐
      │                  │                  │
┌─────▼──────┐  ┌────────▼────────┐  ┌──────▼─────┐
│  Serving   │  │   AutoML/       │  │ Monitoring │
│  (KServe)  │  │   Katib         │  │            │
└────────────┘  └─────────────────┘  └────────────┘
```

### Installation de Kubeflow

#### Option 1 : Installation complète (lourd)

```bash
# Prérequis : Cluster K8s avec 32 GB RAM minimum
# Installer kustomize
curl -s "https://raw.githubusercontent.com/kubernetes-sigs/kustomize/master/hack/install_kustomize.sh" | bash

# Clone Kubeflow manifests
git clone https://github.com/kubeflow/manifests.git
cd manifests

# Installer tout
while ! kustomize build example | kubectl apply -f -; do echo "Retrying to apply resources"; sleep 10; done

# Attendre que tout soit prêt (peut prendre 10-20 minutes)
kubectl wait --for=condition=Ready pods --all -n kubeflow --timeout=600s
```

#### Option 2 : MiniKF (pour lab/test)

```bash
# Version légère de Kubeflow pour tester
vagrant init arrikto/minikf
vagrant up
```

#### Option 3 : Composants individuels (recommandé pour débuter)

Installer uniquement ce dont vous avez besoin :

```bash
# 1. Notebooks Jupyter uniquement
kubectl apply -k "github.com/kubeflow/kubeflow/components/notebook-controller/config/overlays/kubeflow?ref=v1.8.0"

# 2. KServe pour le serving
kubectl apply -f https://github.com/kserve/kserve/releases/download/v0.11.0/kserve.yaml

# 3. Katib pour hyperparameter tuning
kubectl apply -k "github.com/kubeflow/katib/manifests/v1beta1/installs/katib-standalone?ref=v0.16.0"
```

### Kubeflow Notebooks

Interface web pour créer et gérer des Jupyter Notebooks :

```yaml
apiVersion: kubeflow.org/v1
kind: Notebook
metadata:
  name: my-ml-notebook
  namespace: kubeflow-user
spec:
  template:
    spec:
      containers:
      - name: notebook
        image: kubeflownotebookswg/jupyter-tensorflow-cuda:v1.8.0
        resources:
          requests:
            memory: "4Gi"
            cpu: "2"
          limits:
            nvidia.com/gpu: 1
        volumeMounts:
        - name: workspace
          mountPath: /home/jovyan
      volumes:
      - name: workspace
        persistentVolumeClaim:
          claimName: my-workspace
```

Accès via : `http://kubeflow.example.com/notebook/kubeflow-user/my-ml-notebook`

### Kubeflow Pipelines

Définir des workflows ML complexes comme du code :

```python
# pipeline.py
from kfp import dsl
from kfp import compiler

@dsl.component
def preprocess_data(input_data: str, output_data: dsl.Output[dsl.Dataset]):
    """Prétraitement des données"""
    import pandas as pd
    # Charger et nettoyer les données
    df = pd.read_csv(input_data)
    df_clean = df.dropna()
    df_clean.to_csv(output_data.path, index=False)

@dsl.component
def train_model(dataset: dsl.Input[dsl.Dataset], model: dsl.Output[dsl.Model]):
    """Entraînement du modèle"""
    import pandas as pd
    from sklearn.ensemble import RandomForestClassifier
    import joblib

    df = pd.read_csv(dataset.path)
    X = df.drop('target', axis=1)
    y = df['target']

    clf = RandomForestClassifier(n_estimators=100)
    clf.fit(X, y)

    joblib.dump(clf, model.path)

@dsl.component
def evaluate_model(model: dsl.Input[dsl.Model], metrics: dsl.Output[dsl.Metrics]):
    """Évaluation du modèle"""
    import joblib
    from sklearn.metrics import accuracy_score

    clf = joblib.load(model.path)
    # Évaluer sur test set
    accuracy = 0.95  # Exemple

    metrics.log_metric('accuracy', accuracy)

@dsl.pipeline(
    name='ML Training Pipeline',
    description='Pipeline complet d\'entraînement'
)
def ml_pipeline(data_path: str):
    """Pipeline principal"""
    preprocess_task = preprocess_data(input_data=data_path)
    train_task = train_model(dataset=preprocess_task.outputs['output_data'])
    evaluate_task = evaluate_model(model=train_task.outputs['model'])

# Compiler le pipeline
compiler.Compiler().compile(
    pipeline_func=ml_pipeline,
    package_path='ml_pipeline.yaml'
)
```

Déployer :
```bash
# Upload et run le pipeline via UI Kubeflow
# ou via CLI :
kfp run create --experiment-name my-experiment --pipeline-file ml_pipeline.yaml
```

### Katib : Hyperparameter Tuning

**Katib** automatise la recherche des meilleurs hyperparamètres :

```yaml
apiVersion: kubeflow.org/v1beta1
kind: Experiment
metadata:
  name: random-forest-tuning
spec:
  algorithm:
    algorithmName: random
  parallelTrialCount: 5  # 5 essais en parallèle
  maxTrialCount: 20      # 20 essais au total
  maxFailedTrialCount: 3

  objective:
    type: maximize
    goal: 0.99
    objectiveMetricName: accuracy

  parameters:
  - name: n_estimators
    parameterType: int
    feasibleSpace:
      min: "10"
      max: "200"
  - name: max_depth
    parameterType: int
    feasibleSpace:
      min: "5"
      max: "30"
  - name: learning_rate
    parameterType: double
    feasibleSpace:
      min: "0.001"
      max: "0.1"

  trialTemplate:
    primaryContainerName: training
    trialSpec:
      apiVersion: batch/v1
      kind: Job
      spec:
        template:
          spec:
            containers:
            - name: training
              image: myml/training:v1
              command:
              - "python"
              - "train.py"
              - "--n-estimators=${trialParameters.nEstimators}"
              - "--max-depth=${trialParameters.maxDepth}"
              - "--learning-rate=${trialParameters.learningRate}"
            restartPolicy: Never
```

Katib va tester automatiquement 20 combinaisons et trouver la meilleure !

## Model Serving : Déployer des modèles en production

### KServe (anciennement KFServing)

**KServe** est le standard pour servir des modèles ML sur Kubernetes.

#### Architecture KServe

```
┌──────────────────────────────────────────────┐
│              Ingress / Gateway               │
└──────────────────┬───────────────────────────┘
                   │
     ┌─────────────┼─────────────┐
     │             │             │
┌────▼────┐   ┌────▼────┐   ┌────▼───┐
│ Model A │   │ Model B │   │Model C │
│ (v1)    │   │ (v1)    │   │ (v2)   │
└─────────┘   └─────────┘   └────────┘
```

#### InferenceService

```yaml
apiVersion: serving.kserve.io/v1beta1
kind: InferenceService
metadata:
  name: sklearn-iris
spec:
  predictor:
    sklearn:
      storageUri: "gs://kfserving-examples/models/sklearn/iris"
      resources:
        requests:
          memory: "1Gi"
          cpu: "1"
        limits:
          memory: "2Gi"
          cpu: "2"
```

KServe crée automatiquement :
- Un Deployment avec autoscaling
- Un Service
- Des routes Ingress
- Monitoring Prometheus

#### Canary Deployment avec KServe

Déployer progressivement une nouvelle version :

```yaml
apiVersion: serving.kserve.io/v1beta1
kind: InferenceService
metadata:
  name: sklearn-iris
spec:
  predictor:
    sklearn:
      storageUri: "gs://models/sklearn/iris/v1"
      # 90% du trafic sur v1
  canaryTrafficPercent: 10
  canaryPredictor:
    sklearn:
      storageUri: "gs://models/sklearn/iris/v2"
      # 10% du trafic sur v2 (canary)
```

Si v2 fonctionne bien, augmentez progressivement le trafic.

### TensorFlow Serving

Solution spécialisée pour modèles TensorFlow :

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: tensorflow-serving
spec:
  replicas: 3
  selector:
    matchLabels:
      app: tf-serving
  template:
    metadata:
      labels:
        app: tf-serving
    spec:
      containers:
      - name: tensorflow-serving
        image: tensorflow/serving:latest
        ports:
        - containerPort: 8501
          name: http
        - containerPort: 8500
          name: grpc
        env:
        - name: MODEL_NAME
          value: "my_model"
        volumeMounts:
        - name: model-storage
          mountPath: /models/my_model
        resources:
          requests:
            memory: "2Gi"
            cpu: "2"
          limits:
            memory: "4Gi"
            cpu: "4"
        livenessProbe:
          httpGet:
            path: /v1/models/my_model
            port: 8501
          initialDelaySeconds: 30
          periodSeconds: 10
        readinessProbe:
          httpGet:
            path: /v1/models/my_model
            port: 8501
          initialDelaySeconds: 10
          periodSeconds: 5
      volumes:
      - name: model-storage
        persistentVolumeClaim:
          claimName: models-pvc
```

#### Client Python pour TensorFlow Serving

```python
import requests
import json

# Préparer les données
data = {
    "instances": [[5.1, 3.5, 1.4, 0.2]]
}

# Appeler le modèle
response = requests.post(
    'http://tf-serving-service/v1/models/my_model:predict',
    json=data
)

# Résultat
prediction = response.json()
print(f"Prédiction : {prediction['predictions']}")
```

### Triton Inference Server (NVIDIA)

Solution haute performance pour GPU :

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: triton-server
spec:
  replicas: 2
  selector:
    matchLabels:
      app: triton
  template:
    spec:
      containers:
      - name: triton
        image: nvcr.io/nvidia/tritonserver:23.10-py3
        ports:
        - containerPort: 8000
          name: http
        - containerPort: 8001
          name: grpc
        - containerPort: 8002
          name: metrics
        resources:
          limits:
            nvidia.com/gpu: 1
        volumeMounts:
        - name: models
          mountPath: /models
      volumes:
      - name: models
        persistentVolumeClaim:
          claimName: models-pvc
```

Triton supporte TensorFlow, PyTorch, ONNX, TensorRT simultanément !

## Entraînement distribué

### Pourquoi l'entraînement distribué ?

Pour des modèles très grands ou des datasets massifs, un seul GPU ne suffit pas :

```
Dataset : 1 TB
Temps sur 1 GPU : 30 jours
Temps sur 8 GPU : 4 jours
Temps sur 64 GPU : 12 heures
```

### PyTorch Distributed avec TorchElastic

```yaml
apiVersion: kubeflow.org/v1
kind: PyTorchJob
metadata:
  name: pytorch-distributed
spec:
  pytorchReplicaSpecs:
    Master:
      replicas: 1
      restartPolicy: OnFailure
      template:
        spec:
          containers:
          - name: pytorch
            image: pytorch/pytorch:2.0.0-cuda11.8-cudnn8-runtime
            command:
            - python
            - train.py
            resources:
              limits:
                nvidia.com/gpu: 1

    Worker:
      replicas: 7  # 7 workers + 1 master = 8 GPU
      restartPolicy: OnFailure
      template:
        spec:
          containers:
          - name: pytorch
            image: pytorch/pytorch:2.0.0-cuda11.8-cudnn8-runtime
            command:
            - python
            - train.py
            resources:
              limits:
                nvidia.com/gpu: 1
```

Le code d'entraînement :

```python
# train.py
import torch
import torch.distributed as dist
from torch.nn.parallel import DistributedDataParallel as DDP

def setup():
    """Initialiser le groupe de processus distribué"""
    dist.init_process_group(backend='nccl')
    torch.cuda.set_device(int(os.environ['LOCAL_RANK']))

def train():
    setup()

    # Créer le modèle
    model = MyModel().cuda()

    # Wrapper pour distribution
    model = DDP(model)

    # Entraînement normal
    for epoch in range(100):
        for batch in dataloader:
            loss = model(batch)
            loss.backward()
            optimizer.step()

if __name__ == '__main__':
    train()
```

### TensorFlow Distributed avec TFJob

```yaml
apiVersion: kubeflow.org/v1
kind: TFJob
metadata:
  name: tensorflow-distributed
spec:
  tfReplicaSpecs:
    Chief:
      replicas: 1
      template:
        spec:
          containers:
          - name: tensorflow
            image: tensorflow/tensorflow:2.13.0-gpu
            command:
            - python
            - train.py
            resources:
              limits:
                nvidia.com/gpu: 1

    Worker:
      replicas: 3
      template:
        spec:
          containers:
          - name: tensorflow
            image: tensorflow/tensorflow:2.13.0-gpu
            command:
            - python
            - train.py
            resources:
              limits:
                nvidia.com/gpu: 1

    PS:  # Parameter Server
      replicas: 2
      template:
        spec:
          containers:
          - name: tensorflow
            image: tensorflow/tensorflow:2.13.0-gpu
            command:
            - python
            - train.py
```

## MLOps : DevOps pour le Machine Learning

### Qu'est-ce que le MLOps ?

**MLOps** = ML + DevOps = Bonnes pratiques DevOps appliquées au Machine Learning

```
DevOps traditionnel :
Code → Build → Test → Deploy → Monitor

MLOps :
Data + Code → Train → Validate → Deploy Model → Monitor Performance
    ↑                                                      ↓
    └──────────────── Retrain si drift ────────────────────┘
```

### Versioning des modèles

#### MLflow pour le tracking

```python
import mlflow
import mlflow.sklearn
from sklearn.ensemble import RandomForestClassifier

# Démarrer une expérience
mlflow.set_experiment("iris-classification")

# Logger un run
with mlflow.start_run():
    # Hyperparamètres
    n_estimators = 100
    max_depth = 10
    mlflow.log_param("n_estimators", n_estimators)
    mlflow.log_param("max_depth", max_depth)

    # Entraîner le modèle
    clf = RandomForestClassifier(
        n_estimators=n_estimators,
        max_depth=max_depth
    )
    clf.fit(X_train, y_train)

    # Métriques
    accuracy = clf.score(X_test, y_test)
    mlflow.log_metric("accuracy", accuracy)

    # Sauvegarder le modèle
    mlflow.sklearn.log_model(clf, "model")

    print(f"Model logged with accuracy: {accuracy}")
```

MLflow sur Kubernetes :

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mlflow-server
spec:
  replicas: 1
  selector:
    matchLabels:
      app: mlflow
  template:
    spec:
      containers:
      - name: mlflow
        image: python:3.9
        command:
        - sh
        - -c
        - |
          pip install mlflow psycopg2-binary boto3
          mlflow server \
            --backend-store-uri postgresql://user:pass@postgres:5432/mlflow \
            --default-artifact-root s3://mlflow-artifacts/ \
            --host 0.0.0.0 \
            --port 5000
        ports:
        - containerPort: 5000
        env:
        - name: AWS_ACCESS_KEY_ID
          valueFrom:
            secretKeyRef:
              name: aws-credentials
              key: access-key-id
        - name: AWS_SECRET_ACCESS_KEY
          valueFrom:
            secretKeyRef:
              name: aws-credentials
              key: secret-access-key
```

### Model Registry

Gérer le cycle de vie des modèles :

```
Development → Staging → Production → Archived

Model v1.0 : Production
Model v1.1 : Staging (en test)
Model v2.0 : Development
Model v0.9 : Archived
```

```python
import mlflow
from mlflow.tracking import MlflowClient

client = MlflowClient()

# Enregistrer un modèle
model_uri = "runs:/<run-id>/model"
mv = mlflow.register_model(model_uri, "iris-classifier")

# Transition vers staging
client.transition_model_version_stage(
    name="iris-classifier",
    version=mv.version,
    stage="Staging"
)

# Après validation, transition vers production
client.transition_model_version_stage(
    name="iris-classifier",
    version=mv.version,
    stage="Production"
)
```

### CI/CD pour ML

Pipeline GitOps pour ML :

```yaml
# .github/workflows/ml-pipeline.yml
name: ML Training Pipeline

on:
  push:
    branches: [ main ]
  pull_request:
    branches: [ main ]

jobs:
  train:
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v3

    - name: Set up Python
      uses: actions/setup-python@v4
      with:
        python-version: '3.9'

    - name: Install dependencies
      run: |
        pip install -r requirements.txt

    - name: Run data validation
      run: |
        python validate_data.py

    - name: Train model
      run: |
        python train.py

    - name: Evaluate model
      run: |
        python evaluate.py

    - name: Check performance threshold
      run: |
        python check_metrics.py --min-accuracy 0.90

    - name: Push model to registry
      if: github.ref == 'refs/heads/main'
      run: |
        python push_model.py

    - name: Deploy to staging
      if: github.ref == 'refs/heads/main'
      run: |
        kubectl apply -f k8s/staging/
```

### Monitoring de modèles en production

#### Détection de drift

Le **drift** est la dégradation des performances d'un modèle dans le temps :

```
Modèle entraîné en 2023 :
  Accuracy sur données 2023 : 95%

1 an plus tard (2024) :
  Accuracy sur données 2024 : 75%  ← DRIFT !

Pourquoi ? Les données ont changé :
  - Nouvelles tendances
  - Changements de comportement utilisateurs
  - Évolution du marché
```

#### Prometheus pour monitoring ML

```yaml
apiVersion: v1
kind: ServiceMonitor
metadata:
  name: model-serving-metrics
spec:
  selector:
    matchLabels:
      app: inference
  endpoints:
  - port: metrics
    interval: 30s
```

Métriques à surveiller :

```python
from prometheus_client import Counter, Histogram, Gauge

# Compteur de prédictions
predictions_total = Counter(
    'model_predictions_total',
    'Total number of predictions',
    ['model_name', 'model_version']
)

# Latence
prediction_latency = Histogram(
    'model_prediction_latency_seconds',
    'Prediction latency in seconds',
    ['model_name']
)

# Accuracy en temps réel (si ground truth disponible)
model_accuracy = Gauge(
    'model_accuracy',
    'Current model accuracy',
    ['model_name', 'model_version']
)

# Dans votre code de serving
def predict(data):
    start = time.time()

    # Faire la prédiction
    prediction = model.predict(data)

    # Métriques
    latency = time.time() - start
    prediction_latency.labels(model_name='iris').observe(latency)
    predictions_total.labels(
        model_name='iris',
        model_version='v1.2'
    ).inc()

    return prediction
```

#### Alertes sur drift

```yaml
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: model-drift-alerts
spec:
  groups:
  - name: ml-models
    interval: 30s
    rules:
    # Alerte si accuracy descend sous 85%
    - alert: ModelAccuracyDegraded
      expr: model_accuracy < 0.85
      for: 5m
      labels:
        severity: warning
      annotations:
        summary: "Model accuracy below threshold"
        description: "Model {{ $labels.model_name }} accuracy is {{ $value }}"

    # Alerte si latence trop élevée
    - alert: ModelLatencyHigh
      expr: histogram_quantile(0.95, rate(model_prediction_latency_seconds_bucket[5m])) > 1.0
      for: 5m
      labels:
        severity: warning
      annotations:
        summary: "Model latency too high"
        description: "95th percentile latency is {{ $value }}s"
```

## Stockage de données pour ML

### Défis du stockage ML

Les datasets ML sont souvent :
- **Très volumineux** (téraoctets, pétaoctets)
- **Nombreux fichiers** (millions d'images)
- **Accès intensif** (lecture rapide pour entraînement)

### Solutions de stockage

#### 1. S3 / Object Storage

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: training-with-s3
spec:
  containers:
  - name: trainer
    image: myml/trainer:v1
    env:
    - name: AWS_ACCESS_KEY_ID
      valueFrom:
        secretKeyRef:
          name: aws-creds
          key: access-key
    - name: AWS_SECRET_ACCESS_KEY
      valueFrom:
        secretKeyRef:
          name: aws-creds
          key: secret-key
    command:
    - python
    - train.py
    - --data-path=s3://my-bucket/datasets/imagenet/
```

#### 2. PVC avec StorageClass rapide

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: training-dataset
spec:
  accessModes:
  - ReadWriteMany  # Accès simultané par plusieurs Pods
  storageClassName: fast-ssd
  resources:
    requests:
      storage: 500Gi
```

#### 3. Pachyderm pour versioning de données

**Pachyderm** = Git pour les données

```bash
# Installer Pachyderm
helm repo add pachyderm https://helm.pachyderm.com
helm install pachyderm pachyderm/pachyderm

# Créer un repository de données
pachctl create repo images

# Ajouter des données (versionné)
pachctl put file images@master:/cat1.jpg -f cat1.jpg
pachctl put file images@master:/cat2.jpg -f cat2.jpg

# Créer un pipeline qui réagit aux changements
pachctl create pipeline -f pipeline.json
```

Pipeline Pachyderm :

```json
{
  "pipeline": {
    "name": "preprocess"
  },
  "input": {
    "pfs": {
      "repo": "images",
      "glob": "/*.jpg"
    }
  },
  "transform": {
    "cmd": [ "python", "preprocess.py" ],
    "image": "myml/preprocess:v1"
  }
}
```

## Alternatives légères pour MicroK8s

### MLflow seul (sans Kubeflow)

Pour un lab, Kubeflow est souvent trop lourd. Utilisez juste MLflow :

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mlflow
spec:
  replicas: 1
  selector:
    matchLabels:
      app: mlflow
  template:
    spec:
      containers:
      - name: mlflow
        image: python:3.9
        command:
        - sh
        - -c
        - |
          pip install mlflow
          mlflow server --host 0.0.0.0 --port 5000
        ports:
        - containerPort: 5000
        volumeMounts:
        - name: mlflow-data
          mountPath: /mlflow
      volumes:
      - name: mlflow-data
        persistentVolumeClaim:
          claimName: mlflow-pvc
---
apiVersion: v1
kind: Service
metadata:
  name: mlflow
spec:
  selector:
    app: mlflow
  ports:
  - port: 80
    targetPort: 5000
  type: LoadBalancer
```

### Jupyter + TensorFlow/PyTorch simple

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: ml-workspace
spec:
  containers:
  - name: jupyter
    image: jupyter/tensorflow-notebook:latest
    ports:
    - containerPort: 8888
    resources:
      limits:
        nvidia.com/gpu: 1  # Si GPU disponible
        memory: "8Gi"
      requests:
        memory: "4Gi"
        cpu: "2"
    volumeMounts:
    - name: workspace
      mountPath: /home/jovyan/work
  volumes:
  - name: workspace
    persistentVolumeClaim:
      claimName: jupyter-workspace
```

### FastAPI pour serving simple

Au lieu de KServe complexe, un simple FastAPI :

```python
# serve.py
from fastapi import FastAPI
import joblib
import numpy as np
from pydantic import BaseModel

app = FastAPI()

# Charger le modèle au démarrage
model = joblib.load('model.pkl')

class PredictionRequest(BaseModel):
    features: list[float]

@app.post("/predict")
def predict(request: PredictionRequest):
    """Endpoint de prédiction"""
    features = np.array(request.features).reshape(1, -1)
    prediction = model.predict(features)

    return {
        "prediction": int(prediction[0])
    }

@app.get("/health")
def health():
    """Health check"""
    return {"status": "ok"}
```

Dockerfile :

```dockerfile
FROM python:3.9-slim

WORKDIR /app

COPY requirements.txt .
RUN pip install -r requirements.txt

COPY serve.py model.pkl ./

CMD ["uvicorn", "serve:app", "--host", "0.0.0.0", "--port", "8000"]
```

Déploiement :

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: model-api
spec:
  replicas: 3
  selector:
    matchLabels:
      app: model-api
  template:
    spec:
      containers:
      - name: api
        image: myregistry/model-api:v1
        ports:
        - containerPort: 8000
        resources:
          requests:
            memory: "512Mi"
            cpu: "500m"
          limits:
            memory: "1Gi"
            cpu: "1000m"
        livenessProbe:
          httpGet:
            path: /health
            port: 8000
          initialDelaySeconds: 10
          periodSeconds: 10
```

## Bonnes pratiques ML sur Kubernetes

### 1. Séparer entraînement et serving

```
Cluster Training (GPU) :
  - Jobs batch d'entraînement
  - Notebooks Jupyter
  - Hyperparameter tuning

Cluster Serving (CPU ou petit GPU) :
  - Inference servers
  - APIs de prédiction
  - Monitoring
```

Ou au minimum, des namespaces séparés :

```bash
kubectl create namespace ml-training
kubectl create namespace ml-serving
```

### 2. Utiliser des Resource Quotas

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: ml-training-quota
  namespace: ml-training
spec:
  hard:
    requests.nvidia.com/gpu: "4"  # Max 4 GPU simultanés
    requests.memory: "128Gi"
    requests.cpu: "32"
```

### 3. Limiter les durées de jobs

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: training-job
spec:
  activeDeadlineSeconds: 86400  # Max 24h
  ttlSecondsAfterFinished: 3600  # Nettoie après 1h
  backoffLimit: 1  # Ne réessaye qu'une fois
```

### 4. Checkpointing

Sauvegarder régulièrement l'état :

```python
# Dans votre code d'entraînement
for epoch in range(100):
    train_epoch()

    # Checkpoint tous les 10 epochs
    if epoch % 10 == 0:
        torch.save({
            'epoch': epoch,
            'model_state_dict': model.state_dict(),
            'optimizer_state_dict': optimizer.state_dict(),
            'loss': loss,
        }, f'/checkpoints/checkpoint_epoch_{epoch}.pt')
```

Volume pour checkpoints :

```yaml
volumes:
- name: checkpoints
  persistentVolumeClaim:
    claimName: training-checkpoints
```

### 5. Monitoring des ressources

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: training
  annotations:
    prometheus.io/scrape: "true"
    prometheus.io/port: "8000"
spec:
  containers:
  - name: trainer
    image: myml/trainer:v1
    # Exporter des métriques custom
    env:
    - name: PROMETHEUS_PORT
      value: "8000"
```

## Exemple complet : Pipeline de classification d'images

### Architecture

```
1. Upload Images → S3
2. Preprocessing Job → Nettoie et augmente
3. Training Job (GPU) → Entraîne ResNet50
4. Evaluation → Valide sur test set
5. Model Registry → Enregistre si accuracy > 90%
6. Serving → Déploie avec KServe
7. Monitoring → Prometheus + Grafana
```

### 1. Preprocessing Job

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: image-preprocessing
spec:
  template:
    spec:
      containers:
      - name: preprocess
        image: myml/image-preprocess:v1
        command:
        - python
        - preprocess.py
        args:
        - --input-path=s3://mybucket/raw-images/
        - --output-path=s3://mybucket/processed-images/
        - --augmentation=true
        resources:
          requests:
            memory: "4Gi"
            cpu: "2"
      restartPolicy: OnFailure
```

### 2. Training Job

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: resnet50-training
spec:
  template:
    spec:
      containers:
      - name: training
        image: myml/resnet-training:v1
        command:
        - python
        - train.py
        args:
        - --data-path=s3://mybucket/processed-images/
        - --model-name=resnet50
        - --epochs=50
        - --batch-size=32
        resources:
          limits:
            nvidia.com/gpu: 2
            memory: "32Gi"
          requests:
            memory: "16Gi"
            cpu: "8"
        env:
        - name: MLFLOW_TRACKING_URI
          value: "http://mlflow-service:5000"
        volumeMounts:
        - name: checkpoints
          mountPath: /checkpoints
      volumes:
      - name: checkpoints
        persistentVolumeClaim:
          claimName: training-checkpoints
      restartPolicy: OnFailure
  backoffLimit: 2
  activeDeadlineSeconds: 172800  # 48h max
```

### 3. Serving

```yaml
apiVersion: serving.kserve.io/v1beta1
kind: InferenceService
metadata:
  name: image-classifier
spec:
  predictor:
    pytorch:
      storageUri: "s3://mybucket/models/resnet50-v1/"
      resources:
        requests:
          memory: "2Gi"
          cpu: "1"
        limits:
          memory: "4Gi"
          cpu: "2"
      env:
      - name: WORKERS_PER_CORE
        value: "2"
  scaleTarget: 10  # Target 10 requêtes par pod
  scaleMetric: concurrency
```

### 4. Client d'inférence

```python
import requests
import json
from PIL import Image
import base64
import io

def classify_image(image_path, api_url):
    """Classifier une image via l'API"""
    # Charger l'image
    with open(image_path, 'rb') as f:
        image_bytes = f.read()

    # Encoder en base64
    image_b64 = base64.b64encode(image_bytes).decode('utf-8')

    # Préparer la requête
    data = {
        "instances": [{
            "image": image_b64
        }]
    }

    # Appeler l'API
    response = requests.post(
        f"{api_url}/v1/models/image-classifier:predict",
        json=data
    )

    # Parser la réponse
    predictions = response.json()
    return predictions

# Utilisation
result = classify_image(
    'cat.jpg',
    'http://image-classifier.default.example.com'
)
print(f"Classe prédite : {result['predictions'][0]['class']}")
print(f"Confiance : {result['predictions'][0]['confidence']}")
```

## Conclusion

Le **Machine Learning sur Kubernetes** transforme la manière dont les modèles ML sont développés, déployés et maintenus. Kubernetes apporte au ML ce qu'il a apporté aux applications traditionnelles : **scalabilité, résilience, et gestion efficace des ressources**.

### Points clés à retenir

1. **ML sur K8s** = Résoudre les défis de ressources, reproducibilité, scaling
2. **GPU essentiels** pour Deep Learning, bien supportés par Kubernetes
3. **Kubeflow** = Plateforme complète mais lourde, alternatives légères existent
4. **Three phases** : Training (GPU) → Serving (CPU/petit GPU) → Monitoring
5. **MLOps** = DevOps adapté au ML, essentiel pour la production
6. **Model Serving** : KServe, TF Serving, Triton selon vos besoins
7. **Distributed Training** : Nécessaire pour gros modèles et datasets
8. **Monitoring crucial** : Détecter le drift, surveiller les performances

### Recommandations pour MicroK8s

**Pour apprendre le ML sur K8s** :
- Commencez sans GPU (CPU suffit pour débuter)
- Jupyter notebook simple pour expérimenter
- MLflow pour tracking (plus léger que Kubeflow)
- FastAPI pour serving (plus simple que KServe)

**Si vous avez un GPU** :
- Installez le NVIDIA Device Plugin
- Testez l'entraînement de petits modèles
- Explorez PyTorch/TensorFlow sur GPU
- Mesurez les gains de performance

**Pour aller en production** :
- Investissez dans Kubeflow ou alternatives (Metaflow, ZenML)
- Implémentez CI/CD pour ML
- Monitoring obligatoire (drift detection)
- Model registry pour versioning

**Limitations de MicroK8s pour ML** :
- GPU unique (pas de multi-GPU distribué facilement)
- Ressources limitées pour gros modèles
- Pas idéal pour production à grande échelle
- Parfait pour learning, prototyping, edge ML

### Évolution du ML sur Kubernetes

**Tendances actuelles** :
- **LLMs (Large Language Models)** : GPT, LLaMA sur K8s
- **AutoML** : Automatisation complète (Katib, Ray)
- **Edge ML** : Modèles légers déployés en edge avec K3s
- **ML sur WASM** : WebAssembly pour ML ultra-portable
- **Federated Learning** : Entraînement distribué préservant la vie privée

**Outils émergents** :
- **Ray** : Framework distribué pour ML et RL
- **Seldon Core** : Alternative à KServe
- **BentoML** : Packaging et déploiement de modèles
- **ZenML** : Pipeline ML/MLOps moderne
- **Weights & Biases** : Expérimentation et collaboration ML

### Quand utiliser Kubernetes pour ML ?

**✅ OUI si** :
- Équipe ML de 3+ personnes
- Besoin de partager des GPU
- Multiples projets ML simultanés
- Besoin de reproductibilité
- Déploiement de modèles en production
- Entraînement distribué nécessaire

**❌ NON si** :
- Projet ML solo et exploratoire
- Budget serré (GPU cloud)
- Modèles simples (scikit-learn)
- Déploiement occasionnel
- Pas de compétences DevOps/K8s

### Alternatives à K8s pour ML

- **Vertex AI** (Google) : ML managé
- **SageMaker** (AWS) : Plateforme ML complète
- **Azure ML** : Solution Microsoft
- **Databricks** : Pour big data + ML
- **Paperspace Gradient** : GPU cloud simple
- **Lambda Labs** : GPU cloud abordable

### Prochaines étapes

Après avoir exploré le ML sur Kubernetes, vous pourriez vous intéresser à :
- **25.5 Edge computing** : Déployer des modèles ML en edge
- **19. Scaling et Autoscaling** : Optimiser les ressources ML
- **22. Sauvegarde et Restauration** : Protéger vos modèles et données

---

**Ressources complémentaires** :

- Kubeflow Documentation : https://www.kubeflow.org/docs/
- KServe : https://kserve.github.io/website/
- MLflow : https://mlflow.org/docs/
- PyTorch on Kubernetes : https://pytorch.org/ecosystem/
- TensorFlow on Kubernetes : https://www.tensorflow.org/tfx
- NVIDIA GPU Operator : https://docs.nvidia.com/datacenter/cloud-native/gpu-operator/
- ML Engineering Book : https://www.mlebook.com/
- Made With ML : https://madewithml.com/

⏭️ [Tendances et évolutions](/25-aller-plus-loin/07-tendances-et-evolutions.md)
